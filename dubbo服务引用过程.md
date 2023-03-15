从源码的角度分析服务引用过程。



前面服务暴露过程的文章讲解到，服务引用有两种方式，一种就是直连，也就是直接指定服务的地址来进行引用，这种方式更多的时候被用来做服务测试，不建议在生产环境使用这样的方法，因为直连不适合服务治理，dubbo本身就是一个服务治理的框架，提供了很多服务治理的功能。所以更多的时候，我们都不会选择绕过注册中心，而是通过注册中心的方式来进行服务引用。

大致可以分为三个步骤：

1. 配置加载
2. 创建invoker
3. 创建服务接口代理类



# 引用起点

dubbo服务的引用起点就类似于bean加载。dubbo中有一个类ReferenceBean，它实现了FactoryBean接口，继承了ReferenceConfig，所以ReferenceBean作为dubbo中能生产对象的工厂Bean，而我们要引用服务，也就是要有一个该服务的对象。

服务引用被触发有两个时机：

- Spring 容器调用 ReferenceBean 的 afterPropertiesSet 方法时引用服务（饿汉式）
- 在 ReferenceBean 对应的服务被注入到其他类中时引用（懒汉式）

默认情况下，Dubbo 使用懒汉式引用服务。如果需要使用饿汉式，可通过配置 `<dubbo:reference>` 的 init 属性开启。

因为ReferenceBean实现了FactoryBean接口的getObject()方法，所以在加载bean的时候，会调用ReferenceBean的getObject()方法

```java
public class ReferenceBean<T> extends ReferenceConfig<T> implements FactoryBean, ApplicationContextAware, InitializingBean, DisposableBean {
	public Object getObject() {
        return get();
    }

}
```

这个get方法是ReferenceConfig的get()方法

```java
public class ReferenceConfig<T> extends AbstractReferenceConfig {
	 public synchronized T get() {
        // 检查并且更新配置
        checkAndUpdateSubConfigs();
        // 如果被销毁，则抛出异常
        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        // 检测 代理对象ref 是否为空，为空则通过 init 方法创建
        if (ref == null) {
            // 入口  用于处理配置，以及调用 createProxy 生成代理类
            init();
        }
        return ref;  // Invoke代理
    }
}
```

关于checkAndUpdateSubConfigs()方法前一篇文章已经讲了，我就不再讲述。这里关注init方法。该方法也是处理各类配置的开始。

# 配置加载（1）

ReferenceConfig#init

```java
private void init() {
    // 如果已经初始化过，则结束
    if (initialized) {
        return;
    }
    // 本地存根合法性校验
    checkStubAndLocal(interfaceClass);
    // mock合法性校验
    checkMock(interfaceClass);
    // 用来存放配置
    Map<String, String> map = new HashMap<String, String>();
    // 存放这是消费者侧
    map.put(SIDE_KEY, CONSUMER_SIDE);
    // 添加 协议版本、发布版本，时间戳 等信息到 map 中
    appendRuntimeParameters(map);
    // 如果是泛化调用
    if (!ProtocolUtils.isGeneric(getGeneric())) {
        // 获得版本号
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            // 设置版本号
            map.put(REVISION_KEY, revision);

        }
        // 获得所有方法
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            logger.warn("No method found in service interface " + interfaceClass.getName());
            map.put(METHODS_KEY, ANY_VALUE);
        } else {
            // 把所有方法签名拼接起来放入map
            map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), COMMA_SEPARATOR));
        }
    }
    // 加入服务接口名称
    map.put(INTERFACE_KEY, interfaceName);
    // 添加metrics、application、module、consumer、protocol的所有信息到map
    appendParameters(map, metrics);
    appendParameters(map, application);
    appendParameters(map, module);
    // remove 'default.' prefix for configs from ConsumerConfig
    // appendParameters(map, consumer, Constants.DEFAULT_KEY);
    appendParameters(map, consumer);
    appendParameters(map, this);

    Map<String, Object> attributes = null;
    if (CollectionUtils.isNotEmpty(methods)) {
        attributes = new HashMap<String, Object>();
        // 遍历方法配置
        for (MethodConfig methodConfig : methods) {
            // 把方法配置加入map
            appendParameters(map, methodConfig, methodConfig.getName());
            // 生成重试的配置key
            String retryKey = methodConfig.getName() + ".retry";
            // 如果map中已经有该配置，则移除该配置
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                // 如果配置为false，也就是不重试，则设置重试次数为0次
                if ("false".equals(retryValue)) {
                    map.put(methodConfig.getName() + ".retries", "0");
                }
            }
            // 设置异步配置
            attributes.put(methodConfig.getName(), convertMethodConfig2AsyncInfo(methodConfig));
        }
    }
    // 获取服务消费者 ip 地址
    String hostToRegistry = ConfigUtils.getSystemProperty(DUBBO_IP_TO_REGISTRY);
    // 如果为空，则获取本地ip
    if (StringUtils.isEmpty(hostToRegistry)) {
        hostToRegistry = NetUtils.getLocalHost();
    } else if (isInvalidLocalHost(hostToRegistry)) {
        throw new IllegalArgumentException("Specified invalid registry ip from property:" + DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
    }
    // 设置消费者ip
    map.put(REGISTER_IP_KEY, hostToRegistry);

    // 创建代理对象
    ref = createProxy(map);
    // 生产服务key
    String serviceKey = URL.buildKey(interfaceName, group, version);
    // 根据服务名，ReferenceConfig，代理类构建 ConsumerModel，
    // 并将 ConsumerModel 存入到 ApplicationModel 中
    ApplicationModel.initConsumerModel(serviceKey, buildConsumerModel(serviceKey, attributes));
    // 设置初始化标志为true
    initialized = true;
}
```

该方法大致分为以下几个步骤：

1. 检测本地存根和mock合法性。
2. 添加协议版本、发布版本，时间戳、metrics、application、module、consumer、protocol等的所有信息到 map 中
3. 单独处理方法配置，设置重试次数配置以及设置该方法对异步配置信息。
4. 添加消费者ip地址到map
5. 创建代理对象
6. 生成ConsumerModel存入到 ApplicationModel 中

在这里处理配置到逻辑比较清晰。下面就是看ReferenceConfig的createProxy()方法。

## 创建invoker

ReferenceConfig#createProxy

```java
private T createProxy(Map<String, String> map) {
    // 根据配置检查是否为本地调用
    if (shouldJvmRefer(map)) {
        // 生成url，protocol使用的是injvm://
        URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
        // 利用InjvmProtocol 的 refer 方法生成 InjvmInvoker 实例
        invoker = REF_PROTOCOL.refer(interfaceClass, url);
        if (logger.isInfoEnabled()) {
            logger.info("Using injvm service " + interfaceClass.getName());
        }
    } else {
        // 如果url不为空，则用户可能想进行直连来调用
        // 为什么会有urls，因为可以在@Reference的url属性中配置多个url，可以是点对点的服务地址，也可以是注册中心的地址
        urls.clear(); // reference retry init will add url to urls, lead to OOM
        // @Reference中指定了url属性 直连
        if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
            // 当需要配置多个 url 时，可用分号进行分割，这里会进行切分
            String[] us = SEMICOLON_SPLIT_PATTERN.split(url); // 用;号切分
            if (us != null && us.length > 0) {
                // 遍历所有的url
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (StringUtils.isEmpty(url.getPath())) {
                        // 设置接口全限定名为 url 路径
                        url = url.setPath(interfaceName);
                    }
                    // 检测 url 协议是否为 registry，若是，表明用户想使用指定的注册中心
                    // 如果是注册中心地址，则在url中添加一个refer参数
                    if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        // 将 map 转换为查询字符串，并作为 refer 参数的值添加到 url 中
                        urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                    } else {

                        // 如果是服务地址
                        // 有可能url中配置了参数，map中表示的服务消费者消费服务时的参数，所以需要合并
                        // 合并 url，移除服务提供者的一些配置（这些配置来源于用户配置的 url 属性），
                        // 比如线程池相关配置。并保留服务提供者的部分配置，比如版本，group，时间戳等
                        // 最后将合并后的配置设置为 url 查询字符串中。
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else { // assemble URL from register center's configuration
            // @Reference中的protocol属性表示使用哪个协议调用服务，如果不是本地调用协议injvm://，则把注册中心地址找出来
            // 对于injvm://协议已经在之前的逻辑中就已经生成invoke了
            // if protocols not injvm checkRegistry
            if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())){
                // 校验注册中心
                checkRegistry();
                // 加载注册中心地址
                List<URL> us = loadRegistries(false);
                if (CollectionUtils.isNotEmpty(us)) {
                    // 遍历所有的注册中心
                    for (URL u : us) {
                        // 生成监控url
                        URL monitorUrl = loadMonitor(u);
                        if (monitorUrl != null) {
                            // 加入监控中心url的配置
                            map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                        }
                        // 对于注册中心地址都添加REFER_KEY
                        urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                    }
                }
                // 如果urls为空，则抛出异常
                if (urls.isEmpty()) {
                    throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                }
            }
        }

        // 如果只有一个url则直接refer得到一个invoker
        if (urls.size() == 1) {
            // RegistryProtocol.refer() 或者 DubboProtocol.refer()
            invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
            // MockClusterInvoker-->FailoverClusterInvoker-->RegistryDirectory
            //                                                          --->RegistryDirectory$InvokerDelegate-->ListenerInvokerWrapper-->ProtocolFilterWrapper$CallbackRegistrationInvoker-->ConsumerContextFilter-->FutureFilter-->MonitorFilter-->AsyncToSyncInvoker-->DubboInvoker
            //                                                          --->RegistryDirectory$InvokerDelegate-->ListenerInvokerWrapper-->ProtocolFilterWrapper$CallbackRegistrationInvoker-->ConsumerContextFilter-->FutureFilter-->MonitorFilter-->AsyncToSyncInvoker-->DubboInvoker
        } else {
            // 如果有多个url
            // 1. 根据每个url，refer得到对应的invoker
            // 2. 如果这多个urls中存在注册中心url，则把所有invoker整合为RegistryAwareClusterInvoker，该Invoker在调用时，会查看所有Invoker中是否有默认的，如果有则使用默认的Invoker，如果没有，则使用第一个Invoker
            // 2. 如果这多个urls中不存在注册中心url，则把所有invoker整合为FailoverCluster

            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null; // 用来记录urls中最后一个注册中心url
            // 遍历所有的注册中心url
            for (URL url : urls) {
                // 通过 refprotocol 调用 refer 构建 Invoker，
                // refprotocol 会在运行时根据 url 协议头加载指定的 Protocol 实例，并调用实例的 refer 方法
                // 把生成的Invoker加入到集合中
                invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                // 如果是注册中心的协议
                if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    // 则设置registryURL
                    registryURL = url; // use last registry url
                }
            }

            // 如果存在注册中心地址 优先用注册中心的url
            if (registryURL != null) { // registry url is available
                // use RegistryAwareCluster only when register's CLUSTER is available
                // 只有当注册中心链接可用的时候，采用RegistryAwareCluster
                URL u = registryURL.addParameter(CLUSTER_KEY, RegistryAwareCluster.NAME);
                // StaticDirectory表示静态服务目录，里面的invokers是不会变的, 生成一个RegistryAwareCluster
                // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
                // 由集群进行多个invoker合并 RegistryAwareClusterInvoker
                invoker = CLUSTER.join(new StaticDirectory(u, invokers));
            } else { // not a registry url, must be direct invoke.
                // 如果不存在注册中心地址, 生成一个FailoverClusterInvoker
                // 直接进行合并
                invoker = CLUSTER.join(new StaticDirectory(invokers));
            }
        }
    }
    // 如果需要核对该服务是否可用，并且该服务不可用
    if (shouldCheck() && !invoker.isAvailable()) {
        throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
    }
    if (logger.isInfoEnabled()) {
        logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
    }
    /**
     * @since 2.7.0
     * ServiceData Store
     */
    // 元数据中心服务
    MetadataReportService metadataReportService = null;
    // 加载元数据服务，如果成功
    if ((metadataReportService = getMetadataReportService()) != null) {
        // 生成url
        URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
        // 把消费者配置加入到元数据中心中
        metadataReportService.publishConsumer(consumerURL);
    }
    // create service proxy
    // 创建服务代理
    return (T) PROXY_FACTORY.getProxy(invoker);
}
```

该方法的大致逻辑可用分为以下几步：

1. 如果是本地调用，则直接使用InjvmProtocol 的 refer 方法生成 Invoker 实例。
2. 如果不是本地调用，但是是选择直连的方式来进行调用，则分割配置的多个url。如果协议是配置是registry，则表明用户想使用指定的注册中心，配置url后将url保存到urls里面，否则就合并url，并且保存到urls。
3. 如果是通过注册中心来进行调用，则先校验所有的注册中心，然后加载注册中心的url，遍历每个url，加入监控中心url配置，最后把每个url保存到urls。
4. 针对urls集合的数量，如果是单注册中心，直接引用RegistryProtocol 的 refer 构建 Invoker 实例，如果是多注册中心，则对每个url都生成Invoker，利用集群进行多个Invoker合并。
5. 最终输出一个invoker。



Invoker 是 Dubbo 的核心模型，代表一个可执行体。在服务提供方，Invoker 用于调用服务提供类。在服务消费方，Invoker 用于执行远程调用。Invoker 是由 Protocol 实现类构建而来。关于这几个接口的定义介绍可以参考《dubbo源码解析（十九）远程调用——开篇》，Protocol 实现类有很多，下面会分析 RegistryProtocol 和 DubboProtocol，我们可以看到上面的源码中讲到，当只有一个注册中心的时候，会直接使用RegistryProtocol。所以先来看看RegistryProtocol的refer()方法。

### RegistryProtocol生成invoker

```java
public class RegistryProtocol implements Protocol {
	    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {

        // 从registry://的url中获取对应的注册中心，比如zookeeper， 默认为dubbo，dubbo提供了自带的注册中心实现
        // url由 registry:// 改变为---> zookeeper://
        url = URLBuilder.from(url)
                .setProtocol(url.getParameter(REGISTRY_KEY, DEFAULT_REGISTRY))
                .removeParameter(REGISTRY_KEY)
                .build();

        // 拿到注册中心实现，ZookeeperRegistry
        Registry registry = registryFactory.getRegistry(url);

        // 下面这个代码，通过过git历史提交记录是用来解决SimpleRegistry不可用的问题
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        // qs表示 queryString, 表示url中的参数，表示消费者引入服务时所配置的参数
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));

        // group="a,b" or group="*"
        // https://dubbo.apache.org/zh/docs/v2.7/user/examples/group-merger/
        String group = qs.get(GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                // group有多个值，这里的cluster为MergeableCluster
                // 如果有多个组，或者组配置为*，则使用MergeableCluster，并调用 doRefer 继续执行服务引用逻辑
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        // 只有一个组或者没有组配置，则直接执行doRefer
        // 这里的cluster是cluster的Adaptive对象,扩展点
        return doRefer(cluster, registry, type, url);
    }
}
```

上面的逻辑比较简单，如果是注册服务中心，则直接创建代理。如果不是，先处理组配置，根据组配置来决定Cluster的实现方式，然后调用doRefer方法。



RegistryProtocol#doRefer

```java
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
    // RegistryDirectory表示动态服务目录，会和注册中心的数据保持同步
    // type表示一个服务对应一个RegistryDirectory，url表示注册中心地址
    // 在消费端，最核心的就是RegistryDirectory
    RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
    // 设置注册中心
    directory.setRegistry(registry);
    //设置协议
    directory.setProtocol(protocol);


    // all attributes of REFER_KEY
    // 引入服务所配置的参数
    Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());

    // 消费者url
    URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
    // 注册服务消费者，在 consumers 目录下新节点
    if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
        directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));

        // 注册简化后的消费url
        registry.register(directory.getRegisteredConsumerUrl());
    }

    // 构造路由链,路由链会在引入服务时按路由条件进行过滤
    // 路由链是动态服务目录中的一个属性，通过路由链可以过滤某些服务提供者
    directory.buildRouterChain(subscribeUrl);

    // 服务目录需要订阅的几个路径
    // 当前应用所对应的动态配置目录：/dubbo/config/dubbo/dubbo-demo-consumer-application.configurators
    // 当前所引入的服务的动态配置目录：/dubbo/config/dubbo/org.apache.dubbo.demo.DemoService:1.1.1:g1.configurators
    // 当前所引入的服务的提供者目录：/dubbo/org.apache.dubbo.demo.DemoService/providers
    // 当前所引入的服务的老版本动态配置目录：/dubbo/org.apache.dubbo.demo.DemoService/configurators
    // 当前所引入的服务的老版本路由器目录：/dubbo/org.apache.dubbo.demo.DemoService/routers
    directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
            PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));

    // 利用传进来的cluster，join得到invoker, MockClusterWrapper
    /**
     * 创建RegistryProtocol的时候会进行依赖注入，注入进来的是Cluster的Adaptive类
     */
    // 一个注册中心可能有多个服务提供者，因此这里需要将多个服务提供者合并为一个，生成一个invoker
    Invoker invoker = cluster.join(directory);
    // 在服务提供者处注册消费者
    ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
    return invoker;
}
```

该方法大致可以分为以下步骤：

1. 创建一个 RegistryDirectory 实例，然后生成服务者消费者链接。
2. 向注册中心进行注册。
3. 紧接着订阅 providers、configurators、routers 等节点下的数据。完成订阅后，RegistryDirectory 会收到这几个节点下的子节点信息。
4. 由于一个服务可能部署在多台服务器上，这样就会在 providers 产生多个节点，这个时候就需要 Cluster 将多个服务节点合并为一个，并生成一个 Invoker。关于 RegistryDirectory 和 Cluster，可以看我前面写的一些文章介绍





### DubboProtocol生成invoker

首先还是从DubboProtocol的refer()开始，先调用父类AbstractProtocol#refer方法

```java
public abstract class AbstractProtocol implements Protocol {
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        // 异步转同步Invoker , type是接口，url是服务地址
        // DubboInvoker是异步的，而AsyncToSyncInvoker会封装为同步的
        return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
    }
    
}
```



DubboProtocol实现了protocolBindingRefer方法

```java
public class DubboProtocol extends AbstractProtocol {
	public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {

        optimizeSerialization(url);

        // create rpc invoker.
        // clients很重要，为什么一个DubboInvoker会有多个clients，为了提高效率，因为每个client和server之间都会有一个socket, 多个client连的是同一个server
        // 在DubboInvoker发送请求时会轮询clients去发送数据
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        // 把该invoker放入集合
        invokers.add(invoker);

        return invoker;
    }

}
```

创建DubboInvoker比较简单，调用了构造方法，这里主要讲这么生成ExchangeClient，也就是getClients方法。

DubboProtocol#getClients

```java
private ExchangeClient[] getClients(URL url) {
    // whether to share connection
    // 获取client    new NettyClient()  socket
    // DUbboInvoker.invoke 3NettyClient
    // 一个连接是否对于一个服务
    boolean useShareConnect = false;
    // 获得url中连接共享的配置 默认为0
    // connections表示对当前服务提供者建立connections个socket连接
    // 消费者应用引用了两个服务A和B，这两个服务都部署在了应用C上，如果connections为2，那么消费者应用会与应用C建立4个Socket连接
    int connections = url.getParameter(CONNECTIONS_KEY, 0);

    List<ReferenceCountExchangeClient> shareClients = null;

    // if not configured, connection is shared, otherwise, one connection for one service
    // 如果为0，则是共享类，并且连接数为1
    // 如果没有配置connections，那么则取shareConnectionsStr（默认为1），表示共享socket连接个数
    // 消费者应用引用了两个服务A和B，这两个服务都部署在了应用C上，如果shareConnectionsStr为2，那么消费者应用会与应用C建立2个Socket连接
    if (connections == 0) {
        useShareConnect = true;

        /**
         * The xml configuration should have a higher priority than properties.
         */
        String shareConnectionsStr = url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
        connections = Integer.parseInt(StringUtils.isBlank(shareConnectionsStr) ? ConfigUtils.getProperty(SHARE_CONNECTIONS_KEY,
                DEFAULT_SHARE_CONNECTIONS) : shareConnectionsStr);
        //获得共享客户端对象
        shareClients = getSharedClient(url, connections);
    }

    /**
     *
     * 为什么不直接是NettyClient 因为dubbo协议还支持mina
     */
    // 创建数组
    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        // 如果使用共享的，则利用shareClients
        if (useShareConnect) {
            clients[i] = shareClients.get(i);

        } else {
            // 不然就初始化，在初始化client时会去连接服务端
            clients[i] = initClient(url);
        }
    }

    return clients;
}
```

如果是共享客户端，则调用getSharedClient

DubboProtocol#getSharedClient

```java
private List<ReferenceCountExchangeClient> getSharedClient(URL url, int connectNum) {
    // 这个方法返回的是可以共享的client，要么已经生成过了，要么需要重新生成
    // 对于已经生成过的client,都会存在referenceClientMap中，key为所调用的服务IP+PORT

    String key = url.getAddress();
    // 从集合中取出客户端对象
    List<ReferenceCountExchangeClient> clients = referenceClientMap.get(key);

    // 根据当前引入的服务对应的ip+port，看看是否已经存在clients了，
    if (checkClientCanUse(clients)) {
        // 如果每个client都可用，那就对每个client的计数+1，表示这些client被引用了多少次
        batchClientRefIncr(clients);
        return clients;
    }

    locks.putIfAbsent(key, new Object());
    synchronized (locks.get(key)) {
        clients = referenceClientMap.get(key);
        // dubbo check
        if (checkClientCanUse(clients)) {
            batchClientRefIncr(clients);
            return clients;
        }

        // connectNum must be greater than or equal to 1
        // 至少一个
        connectNum = Math.max(connectNum, 1);

        // If the clients is empty, then the first initialization is
        if (CollectionUtils.isEmpty(clients)) {
            // 如果clients为空，则按指定的connectNum生成client
            clients = buildReferenceCountExchangeClientList(url, connectNum);
            referenceClientMap.put(key, clients);
        } else {
            // 如果clients不为空，则遍历这些client，对于不可用的client，则重新生成一个client
            for (int i = 0; i < clients.size(); i++) {
                ReferenceCountExchangeClient referenceCountExchangeClient = clients.get(i);
                // If there is a client in the list that is no longer available, create a new one to replace him.
                if (referenceCountExchangeClient == null || referenceCountExchangeClient.isClosed()) {
                    clients.set(i, buildReferenceCountExchangeClient(url));
                    continue;
                }

                referenceCountExchangeClient.incrementAndGetCount();
            }
        }

        /**
         * I understand that the purpose of the remove operation here is to avoid the expired url key
         * always occupying this memory space.
         */
        locks.remove(key);

        return clients;
    }
}
```

该方法比较简单，先访问缓存，若缓存未命中，则通过 initClient 方法创建新的 ExchangeClient 实例，并将该实例传给 ReferenceCountExchangeClient 构造方法创建一个带有引用计数功能的 ExchangeClient 实例



DubboProtocol#initClient 初始化客户端

```java
private ExchangeClient initClient(URL url) {

    // client type setting.
    // 拿设置的client，默认为netty3
    String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));

    // 编码方式
    url = url.addParameter(CODEC_KEY, DubboCodec.NAME);

    // enable heartbeat by default
    // 心跳， 默认60 * 1000,60秒一个心跳
    url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));

    // BIO is not allowed since it has severe performance issue.
    // 如果没有指定的client扩展，则抛异常
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported client type: " + str + "," +
                " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
    }

    ExchangeClient client;
    try {
        // connection should be lazy
        // 是否需要延迟连接，，默认不开启
        //服务与引入时先不建立连接 调用服务时才建立连接
        if (url.getParameter(LAZY_CONNECT_KEY, false)) {
            // 创建延迟连接的客户端
            client = new LazyConnectExchangeClient(url, requestHandler);

        } else {
            // client是在refer的时候生成的，这个时候就已经建立好连接了？
            // 答案是就是会去建立连接，也是能够理解了，只有连接建立好了才有client和server之分
            // 先建立连接，在调用方法时再基于这个连接去发送数据
            client = Exchangers.connect(url, requestHandler);  // connect
        }

    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
    }

    return client;
}
```

initClient 方法首先获取用户配置的客户端类型，最新版本已经改为默认 netty4。然后设置用户心跳配置，然后检测用户配置的客户端类型是否存在，不存在则抛出异常。最后根据 lazy 配置决定创建什么类型的客户端。这里的 LazyConnectExchangeClient 代码并不是很复杂，该类会在 request 方法被调用时通过 Exchangers 的 connect 方法创建 ExchangeClient 客户端。下面我们分析一下 Exchangers 的 connect 方法。

```java
public class Exchangers {
	  public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");

        // 得到一个HeaderExchanger去connect
        return getExchanger(url).connect(url, handler);
    }

}
```

getExchanger 会通过 SPI 加载 HeaderExchangeClient 实例，这个方法比较简单。接下来分析 HeaderExchangeClient 的connect的实现。

```java
public class HeaderExchanger implements Exchanger {

    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        // 用传输层连接返回的client 创建对应的信息交换客户端，默认开启心跳检测
        // 利用NettyTransporter去connect
        // 为什么在connect和bind时都是DecodeHandler，解码，解的是把InputStream解析成AppResponse对象
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }
}
```

Transporters的connect()

```java
public class Transporters {
	 public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        ChannelHandler handler;
        if (handlers == null || handlers.length == 0) {
            handler = new ChannelHandlerAdapter();
        } else if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        // 调用Transporter的实现类对象的connect方法。
        // 例如实现NettyTransporter，则调用NettyTransporter的connect，并且返回相应的client
        return getTransporter().connect(url, handler);
    }

}
```

其中获得自适应拓展类，该类会在运行时根据客户端类型加载指定的 Transporter 实现类。若用户未配置客户端类型，则默认加载 NettyTransporter，并调用该类的 connect 方法。假设是netty4的实现

```java
public class NettyTransporter implements Transporter {

    public static final String NAME = "netty";

    @Override
    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }
}
```

到这里为止，DubboProtocol生成invoker过程也结束了。再回到createProxy方法的最后一句代码，根据invoker创建服务代理对象。



### AbstractProxyFactory .getProxy()

为服务接口生成代理对象。有了代理对象，即可进行远程调用。首先来看AbstractProxyFactory 的 getProxy()方法。

```java
public abstract class AbstractProxyFactory implements ProxyFactory {

    @Override
    public <T> T getProxy(Invoker<T> invoker) throws RpcException {
        return getProxy(invoker, false);
    }
    
    public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
        Class<?>[] interfaces = null;
        // 获得需要代理的接口
        String config = invoker.getUrl().getParameter(INTERFACES);
        if (config != null && config.length() > 0) {
            // 根据逗号把每个接口分割开
            String[] types = COMMA_SPLIT_PATTERN.split(config);
            if (types != null && types.length > 0) {
                // 创建接口类型数组
                interfaces = new Class<?>[types.length + 2];
                // 第一个放invoker的服务接口
                interfaces[0] = invoker.getInterface();
                // 第二个位置放回声测试服务的接口类
                interfaces[1] = EchoService.class;
                // 其他接口循环放入
                for (int i = 0; i < types.length; i++) {
                    // TODO can we load successfully for a different classloader?.
                    interfaces[i + 2] = ReflectUtils.forName(types[i]);
                }
            }
        }
        // 如果接口为空，就是config为空，则是回声测试
        if (interfaces == null) {
            interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
        }
        // 如果是泛化服务，那么在代理的接口集合中加入泛化服务类型
        if (!GenericService.class.isAssignableFrom(invoker.getInterface()) && generic) {
            int len = interfaces.length;
            Class<?>[] temp = interfaces;
            interfaces = new Class<?>[len + 1];
            System.arraycopy(temp, 0, interfaces, 0, len);
            interfaces[len] = com.alibaba.dubbo.rpc.service.GenericService.class;
        }
        // 获得代理
        return getProxy(invoker, interfaces);
    }

    public abstract <T> T getProxy(Invoker<T> invoker, Class<?>[] types);
}
```

可以看到第二个getProxy方法其实就是获取 interfaces 数组，调用到第三个getProxy方法时，该getProxy是个抽象方法，由子类来实现，我们还是默认它的代理实现方式为Javassist。所以可以看JavassistProxyFactory的getProxy方法。



```java
public class JavassistProxyFactory extends AbstractProxyFactory {

    @Override
    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        // 生成 Proxy 子类（Proxy 是抽象类）。并调用 Proxy 子类的 newInstance 方法创建 Proxy 实例
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
}
```

我们重点看Proxy的getProxy方法。

```java
public abstract class Proxy {
	 public static Proxy getProxy(Class<?>... ics) {
        // 获得Proxy的类加载器来进行生成代理类
        return getProxy(ClassUtils.getClassLoader(Proxy.class), ics);
    }
     public static Proxy getProxy(ClassLoader cl, Class<?>... ics) {
        // 最大的代理接口数限制是65535
        if (ics.length > MAX_PROXY_COUNT) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        StringBuilder sb = new StringBuilder();
        // 遍历代理接口，获取接口的全限定名并以分号分隔连接成字符串
        for (int i = 0; i < ics.length; i++) {
            // 获得类名
            String itf = ics[i].getName();
            // 判断是否为接口
            if (!ics[i].isInterface()) {
                throw new RuntimeException(itf + " is not a interface.");
            }

            Class<?> tmp = null;
            try {
                // 获得与itf对应的Class对象
                tmp = Class.forName(itf, false, cl);
            } catch (ClassNotFoundException e) {
            }
            // 如果通过类名获得的类型跟ics中的类型不一样，则抛出异常
            if (tmp != ics[i]) {
                throw new IllegalArgumentException(ics[i] + " is not visible from class loader");
            }
            // 拼接接口全限定名，分隔符为 ;
            sb.append(itf).append(';');
        }

        // use interface class name list as key.
        // 使用拼接后的接口名作为 key
        String key = sb.toString();

        // get cache by class loader.
        final Map<String, Object> cache;
        // 把该类加载器加到本地缓存
        synchronized (PROXY_CACHE_MAP) {
            // 通过类加载器获得缓存
            cache = PROXY_CACHE_MAP.computeIfAbsent(cl, k -> new HashMap<>());
        }

        Proxy proxy = null;
        synchronized (cache) {
            do {
                // 从缓存中获取 Reference<Proxy> 实例
                Object value = cache.get(key);
                // 如果缓存中存在，则直接返回代理对象
                if (value instanceof Reference<?>) {
                    proxy = (Proxy) ((Reference<?>) value).get();
                    if (proxy != null) {
                        return proxy;
                    }
                }
                // 是等待生成的类型，则等待  并发控制，保证只有一个线程可以进行后续操作
                if (value == PENDING_GENERATION_MARKER) {
                    try {
                        // 其他线程在此处进行等待
                        cache.wait();
                    } catch (InterruptedException e) {
                    }
                } else {
                    //放置标志位到缓存中，并跳出 while 循环进行后续操作
                    cache.put(key, PENDING_GENERATION_MARKER);
                    break;
                }
            }
            while (true);
        }
        // AtomicLong自增生成代理类类名后缀id，防止冲突
        long id = PROXY_CLASS_COUNTER.getAndIncrement();
        String pkg = null;
        ClassGenerator ccp = null, ccm = null;
        try {
            // 创建 ClassGenerator 对象
            ccp = ClassGenerator.newInstance(cl);

            Set<String> worked = new HashSet<>();
            List<Method> methods = new ArrayList<>();

            for (int i = 0; i < ics.length; i++) {
                // 判断是否为public 检测接口访问级别是否为 protected 或 privete
                if (!Modifier.isPublic(ics[i].getModifiers())) {

                    // 获得该类的包名
                    String npkg = ics[i].getPackage().getName();
                    if (pkg == null) {
                        pkg = npkg;
                    } else {
                        // 非 public 级别的接口必须在同一个包下，否者抛出异常
                        if (!pkg.equals(npkg)) {
                            throw new IllegalArgumentException("non-public interfaces from different packages");
                        }
                    }
                }
                // 把接口加入到ccp的mInterfaces中
                ccp.addInterface(ics[i]);
                // 遍历每个类的方法
                for (Method method : ics[i].getMethods()) {
                    // 获得方法描述 这个方法描述是自定义：
                    // 例如：int do(int arg1) => "do(I)I"
                    // 例如：void do(String arg1,boolean arg2) => "do(Ljava/lang/String;Z)V"
                    String desc = ReflectUtils.getDesc(method);
                    // 如果方法描述字符串已在 worked 中，则忽略。考虑这种情况，
                    // A 接口和 B 接口中包含一个完全相同的方法
                    if (worked.contains(desc)) {
                        continue;
                    }
                    if (ics[i].isInterface() && Modifier.isStatic(method.getModifiers())) {
                        continue;
                    }
                    // 如果集合中不存在，则加入该描述
                    worked.add(desc);

                    int ix = methods.size();
                    // 获得方法返回类型
                    Class<?> rt = method.getReturnType();
                    // 获得方法参数类型
                    Class<?>[] pts = method.getParameterTypes();
                    // 新建一句代码
                    // 例如Object[] args = new Object[参数数量】
                    StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
                    // 每一个参数都生成一句代码
                    // 例如args[0] = ($w)$1;
                    // 例如 Object ret = handler.invoke(this, methods[3], args);
                    for (int j = 0; j < pts.length; j++) {
                        // 生成 args[1...N] = ($w)$1...N;
                        code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(";");
                    }
                    // 生成 InvokerHandler 接口的 invoker 方法调用语句，如下：
                    // Object ret = handler.invoke(this, methods[1...N], args);
                    code.append(" Object ret = handler.invoke(this, methods[").append(ix).append("], args);");
                    // 如果方法不是void类型
                    // 则拼接 return ret;
                    if (!Void.TYPE.equals(rt)) {
                        // 生成返回语句，形如 return (java.lang.String) ret;
                        code.append(" return ").append(asArgument(rt, "ret")).append(";");
                    }

                    methods.add(method);
                    // 添加方法名、访问控制符、参数列表、方法代码等信息到 ClassGenerator 中
                    ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
                }
            }

            if (pkg == null) {
                pkg = PACKAGE_NAME;
            }

            // create ProxyInstance class.
            // 构建接口代理类名称：pkg + ".proxy" + id，比如 org.apache.dubbo.proxy0
            String pcn = pkg + ".proxy" + id;
            ccp.setClassName(pcn);
            // 添加静态字段Method[] methods
            ccp.addField("public static java.lang.reflect.Method[] methods;");
            // 生成 private java.lang.reflect.InvocationHandler handler;
            ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
            // 添加实例对象InvokerInvocationHandler hanler，添加参数为InvokerInvocationHandler的构造器
            // 为接口代理类添加带有 InvocationHandler 参数的构造方法，比如：
            // porxy0(java.lang.reflect.InvocationHandler arg0) {
            //     handler=$1;
            // }
            ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
            // 添加默认无参构造器
            ccp.addDefaultConstructor();
            // 使用toClass方法生成对应的字节码
            Class<?> clazz = ccp.toClass();
            clazz.getField("methods").set(null, methods.toArray(new Method[0]));

            // create Proxy class.
            // 生成的字节码对象为服务接口的代理对象 构建 Proxy 子类名称，比如 Proxy1，Proxy2 等
            String fcn = Proxy.class.getName() + id;
            ccm = ClassGenerator.newInstance(cl);
            ccm.setClassName(fcn);
            ccm.addDefaultConstructor();
            ccm.setSuperClass(Proxy.class);
            // 为 Proxy 的抽象方法 newInstance 生成实现代码，形如：
            // public Object newInstance(java.lang.reflect.InvocationHandler h) {
            //     return new org.apache.dubbo.proxy0($1);
            // }
            ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
            Class<?> pc = ccm.toClass();
            // 生成 Proxy 实现类
            proxy = (Proxy) pc.newInstance();
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeException(e.getMessage(), e);
        } finally {
            // release ClassGenerator
            // 重置类构造器
            if (ccp != null) {
                ccp.release();
            }
            if (ccm != null) {
                ccm.release();
            }
            synchronized (cache) {
                if (proxy == null) {
                    cache.remove(key);
                } else {
                    cache.put(key, new WeakReference<Proxy>(proxy));
                }
                // 唤醒其他等待线程

                cache.notifyAll();
            }
        }
        return proxy;
    }

}
```

代码比较多，大致可以分为以下几步：

1. 对接口进行校验，检查是否是一个接口，是否不能被类加载器加载。
2. 做并发控制，保证只有一个线程可以进行后续的代理生成操作。
3. 创建cpp，用作为服务接口生成代理类。首先对接口定义以及包信息进行处理。
4. 对接口的方法进行处理，包括返回类型，参数类型等。最后添加方法名、访问控制符、参数列表、方法代码等信息到 ClassGenerator 中。
5. 创建接口代理类的信息，比如名称，默认构造方法等。
6. 生成接口代理类。
7. 创建ccm，ccm 则是用于为 org.apache.dubbo.common.bytecode.Proxy 抽象类生成子类，主要是实现 Proxy 类的抽象方法。
8. 设置名称、创建构造方法、添加方法
9. 生成 Proxy 实现类。
10. 释放资源
11. 创建弱引用，写入缓存，唤醒其他线程。



到这里，接口代理类生成后，服务引用也就结束了。







