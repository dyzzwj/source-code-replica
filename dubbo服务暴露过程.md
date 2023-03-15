从源码的角度分析服务暴露过程。

# 服务暴露过程

服务暴露过程大致可分为三个部分：

1. 前置工作，主要用于检查参数，组装 URL。
2. 导出服务，包含暴露服务到本地 (JVM)，和暴露服务到远程两个过程。
3. 向注册中心注册服务，用于服务发现



## 暴露起点

Spring中有一个ApplicationListener接口，其中定义了一个onApplicationEvent()方法，在当容器内发生对应的事件时，此方法会被触发。

Dubbo中ServiceBean类实现了该接口，并且实现了onApplicationEvent方法：

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean,
        ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware,
        ApplicationEventPublisherAware {
            
      //监听ContextRefreshedEvent
	  public void onApplicationEvent(ContextRefreshedEvent event) {
        // 当前服务没有被导出并且没有卸载，才导出服务
        if (!isExported() && !isUnexported()) {
            if (logger.isInfoEnabled()) {
                logger.info("The service ready on spring started. service: " + getInterface());
            }
            // 服务导出（服务注册）
            export();
        }
      }        
}
```

只要服务没有被暴露并且服务没有被取消暴露，就暴露服务。执行export方法。接下来就是跟官网的时序图有关，对照着时序图来看下面的过程

![dubbo服务暴露](images/dubbo服务暴露.webp)

我会在下面的小标题加上时序图的每一步操作范围，比如下面要讲到的前置工作其实就是时序图中的1:export()，那我会在标题括号里面写1。

其中服务暴露的第八步已经没有了。



## 前置工作（1）

前置工作主要包含两个部分，分别是配置检查，以及 URL 装配。在暴露服务之前，Dubbo 需要检查用户的配置是否合理，或者为用户补充缺省配置。配置检查完成后，接下来需要根据这些配置组装 URL。在 Dubbo 中，URL 的作用十分重要。Dubbo 使用 URL 作为配置载体，所有的拓展点都是通过 URL 获取配置



在调用export方法之后，执行的是ServiceConfig中的export方法。

```java
public class ServiceConfig<T> extends AbstractServiceConfig {
	    public synchronized void export() {
        //检查并且更新配置  在此之前从properties中和注解中获取的配置都是优先级比较低的 需要获取优先级更高的配置去覆盖
        checkAndUpdateSubConfigs();

        // 检查服务是否需要导出
        if (!shouldExport()) {
            return;
        }

        // 检查是否需要延迟发布 则延迟delay时间后暴露服务
        if (shouldDelay()) {
            DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
        } else {
            // 导出服务
            doExport();
        }
    }
}
```

可以看到首先做的就是对配置的检查和更新，执行的是ServiceConfig中的checkAndUpdateSubConfigs方法。然后检测是否应该暴露，如果不应该暴露，则直接结束，然后检测是否配置了延迟加载，如果是，则使用定时器来实现延迟加载的目的。

### 配置检查

checkAndUpdateSubConfigs

```java
 public class ServiceConfig<T> extends AbstractServiceConfig {
      public void checkAndUpdateSubConfigs() {
        // ServiceConfig中的某些属性如果是空的，那么就从ProviderConfig、ModuleConfig、ApplicationConfig中获取
        // 补全ServiceConfig中的属性   application -> module -> provider -> service
        completeCompoundConfigs();

        // Config Center should always being started first.
        // 从配置中心获取配置，包括应用配置和全局配置
        // 把获取到的配置放入到Environment中的externalConfigurationMap和appExternalConfigurationMap中
        // 并刷新所有的XxConfig的属性（除开ServiceConfig），刷新的意思就是优先级更高的配置覆盖XxConfig中的属性
        startConfigCenter();

        // 检测 provider 是否为空，为空则新建一个，并通过系统变量为其初始化
        checkDefault();
        // 检查protocols是否为空
        checkProtocol();
        // 检查application是否为空
        checkApplication();

        // if protocol is not injvm checkRegistry
        // 如果protocol不是只有injvm协议，表示服务调用不是只在本机jvm里面调用，那就需要用到注册中心
        if (!isOnlyInJvm()) {
            checkRegistry();
        }

        // 刷新ServiceConfig
        this.refresh();

        // 如果配了metadataReportConfig，那么就刷新配置
        checkMetadataReport();

        // 服务接口名不能为空，否则抛出异常
        if (StringUtils.isEmpty(interfaceName)) {
            throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
        }

        // 当前服务对应的实现类是一个GenericService，表示没有特定的接口
        if (ref instanceof GenericService) {
            // 设置interfaceClass为GenericService
            interfaceClass = GenericService.class;

            if (StringUtils.isEmpty(generic)) {
                // 设置generic = true
                generic = Boolean.TRUE.toString();
            }
        } else {
            // 加载接口
            try {
                interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                        .getContextClassLoader());
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            // 刷新MethodConfig，并判断MethodConfig中对应的方法在接口中是否存在 对 interfaceClass，以及 <dubbo:method> 标签中的必要字段进行检查
            checkInterfaceAndMethods(interfaceClass, methods);
            // 实现类是不是该接口类型
            checkRef();
            generic = Boolean.FALSE.toString();
        }
        //  stub local一样都是配置本地存根 local和stub一样，不建议使用了
        if (local != null) {
            // 如果本地存根为true，则存根类为interfaceName + "Local"
            if (Boolean.TRUE.toString().equals(local)) {
                local = interfaceName + "Local";
            }
            // 加载本地存根类
            Class<?> localClass;
            try {
                localClass = ClassUtils.forNameWithThreadContextClassLoader(local);
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            if (!interfaceClass.isAssignableFrom(localClass)) {
                throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
            }
        }
        // 本地存根
        if (stub != null) {
            // 如果本地存根为true，则存根类为interfaceName + "Stub"
            if (Boolean.TRUE.toString().equals(stub)) {
                stub = interfaceName + "Stub";
            }
            Class<?> stubClass;
            try {
                stubClass = ClassUtils.forNameWithThreadContextClassLoader(stub);
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            if (!interfaceClass.isAssignableFrom(stubClass)) {
                throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
            }
        }
        // 检查local和stub
        checkStubAndLocal(interfaceClass);
        // 检查mock
        checkMock(interfaceClass);
    }
     
     
 }

```

可以看到，该方法中是对各类配置的校验，并且更新部分配置。其中检查的细节我就不展开，因为服务暴露整个过程才是本文重点。

在经过shouldExport()和shouldDelay()两个方法检测后，会执行ServiceConfig的doExport()方法





### 服务导出



```java
public class ServiceConfig<T> extends AbstractServiceConfig {
	   protected synchronized void doExport() {
        //当前服务以及被取消了 就不能再导出了 
        // 如果调用不暴露的方法，则unexported值为true
        if (unexported) {
            throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
        }
        // 已经导出了，就不再导出了
        if (exported) {
            return;
        }
        exported = true;

        if (StringUtils.isEmpty(path)) {
            path = interfaceName;
        }
        //导出url
        doExportUrls();
    }

}
```

该方法就是对于服务是否暴露在一次校验，然后会执行ServiceConfig的doExportUrls()方法，对于多协议多注册中心暴露服务进行支持。

```java
public class ServiceConfig<T> extends AbstractServiceConfig {
    
      private void doExportUrls() {
        // registryURL 表示一个注册中心
        //得到url 注册服务也是一个服务 所以也会有对应的URL 通过调用该url 完成服务注册
        List<URL> registryURLs = loadRegistries(true);
        // 遍历 protocols，并在每个协议下暴露服务
        for (ProtocolConfig protocolConfig : protocols) {
            // 以path、group、version来作为服务唯一性确定的key
            // pathKey = group/contextpath/path:version
            // 例子：myGroup/user/org.apache.dubbo.demo.DemoService:1.0.1
            String pathKey = URL.buildKey(getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version);

            // ProviderModel中存在服务提供者访问路径，实现类，接口，以及接口中的各个方法对应的ProviderMethodModel
            // ProviderMethodModel表示某一个方法，方法名，所属的服务的，
            ProviderModel providerModel = new ProviderModel(pathKey, ref, interfaceClass);

            // ApplicationModel表示应用中有哪些服务提供者和引用了哪些服务
            ApplicationModel.initProviderModel(pathKey, providerModel);

            // 重点
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
}
```

从该方法可以看到：

- loadRegistries()方法是加载注册中心链接。
- 服务的唯一性是通过path、group、version一起确定的。
- doExportUrlsFor1Protocol()方法开始组装URL。

#### 获取注册中心url

loadRegistries

```java
public class ServiceConfig<T> extends AbstractServiceConfig {
	 protected List<URL> loadRegistries(boolean provider) {
        // check && override if necessary
        List<URL> registryList = new ArrayList<URL>();
        // 如果registries为空，直接返回空集合
        if (CollectionUtils.isNotEmpty(registries)) {
            // 遍历注册中心配置集合registries
            for (RegistryConfig config : registries) {
                // 获得地址
                String address = config.getAddress();
                // 如果注册中心没有配地址，则地址为0.0.0.0
                if (StringUtils.isEmpty(address)) {
                    address = ANYHOST_VALUE;
                }
                // 如果注册中心的地址不是"N/A"
                if (!RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
                    Map<String, String> map = new HashMap<String, String>();
                    // 把application中的参数放入map中，注意，map中的key是没有prefix的
                    appendParameters(map, application);
                    // 把config中的参数放入map中，注意，map中的key是没有prefix的
                    // config是RegistryConfig，表示注册中心
                    appendParameters(map, config);
                    // 此处path值固定为RegistryService.class.getName()，因为现在是在加载注册中心
                    map.put(PATH_KEY, RegistryService.class.getName());
                    // 把dubbo的版本信息和pid放入map中
                    appendRuntimeParameters(map);

                    // 如果map中如果没有protocol，那么默认为dubbo
                    if (!map.containsKey(PROTOCOL_KEY)) {
                        map.put(PROTOCOL_KEY, DUBBO_PROTOCOL);
                    }

                    // 解析得到 URL 列表，address 可能包含多个注册中心 ip，因此解析得到的是一个列表 地址+参数
                    List<URL> urls = UrlUtils.parseURLs(address, map);
                    // 遍历URL 列表
                    for (URL url : urls) {
                        // 将 URL 协议头设置为 registry
                        url = URLBuilder.from(url)
                                .addParameter(REGISTRY_KEY, url.getProtocol())
                                .setProtocol(REGISTRY_PROTOCOL)
                                .build();
                        // 到此为止，url的内容大概为：
                        // registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-provider&dubbo=2.0.2&pid=269936&registry=zookeeper&timestamp=1584886077813
                        // 该url表示：使用registry协议调用org.apache.dubbo.registry.RegistryService服务
                        // 参数为application=dubbo-demo-annotation-provider&dubbo=2.0.2&pid=269936&registry=zookeeper&timestamp=1584886077813

                        // 这里是服务提供者和服务消费者区别的逻辑
                        // 如果是服务提供者，获取register的值，如果为false，表示该服务不注册到注册中心
                        // 如果是服务消费者，获取subscribe的值，如果为false，表示该引入的服务不订阅注册中心中的数据
                        if ((provider && url.getParameter(REGISTER_KEY, true))
                                || (!provider && url.getParameter(SUBSCRIBE_KEY, true))) {
                            registryList.add(url);
                        }
                    }
                }
            }
        }
        return registryList;
    }

}
```



#### 根据协议暴露服务

doExportUrlsFor1Protocol

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    // protocolConfig表示某个协议，registryURLs表示所有的注册中心

    // 如果配置的某个协议，没有配置name，那么默认为dubbo
    String name = protocolConfig.getName();
    if (StringUtils.isEmpty(name)) {
        name = DUBBO;
    }

    // 这个map表示服务url的参数
    Map<String, String> map = new HashMap<String, String>();
    map.put(SIDE_KEY, PROVIDER_SIDE);

    appendRuntimeParameters(map);

    // 监控中心参数
    appendParameters(map, metrics);
    // 应用相关参数
    appendParameters(map, application);
    // 模块相关参数
    appendParameters(map, module);
    // remove 'default.' prefix for configs from ProviderConfig
    // appendParameters(map, provider, Constants.DEFAULT_KEY);

    // 提供者相关参数
    appendParameters(map, provider);

    // 协议相关参数
    appendParameters(map, protocolConfig);

    // 服务本身相关参数
    appendParameters(map, this);

    // 服务中某些方法参数 如果method的配置列表不为空
    if (CollectionUtils.isNotEmpty(methods)) {
        // 遍历method配置列表
        for (MethodConfig method : methods) {
            // 某个方法的配置参数，注意有prefix
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";

            // 如果某个方法配置存在xx.retry=false，则改成xx.retry=0
            // 添加 MethodConfig 对象的字段信息到 map 中，键 = 方法名.属性名。
            // 比如存储 <dubbo:method name="sayHello" retries="2"> 对应的 MethodConfig，
            // 键 = sayHello.retries，map = {"sayHello.retries": 2, "xxx": "yyy"}
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                // 如果retryValue为false，则不重试，设置值为0
                if (Boolean.FALSE.toString().equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            // 获得ArgumentConfig列表
            List<ArgumentConfig> arguments = method.getArguments();
            if (CollectionUtils.isNotEmpty(arguments)) {
                // 遍历当前方法配置中的参数配置
                for (ArgumentConfig argument : arguments) {

                    // 如果配置了type，则遍历当前接口的所有方法，然后找到方法名和当前方法名相等的方法，可能存在多个
                    // 如果配置了index,则看index对应位置的参数类型是否等于type,如果相等，则向map中存入argument对象中的参数
                    // 如果没有配置index，那么则遍历方法所有的参数类型，等于type则向map中存入argument对象中的参数
                    // 如果没有配置type,但配置了index,则把对应位置的argument放入map
                    // convert argument type
                    // 检测 type 属性是否为空，或者空串
                    if (argument.getType() != null && argument.getType().length() > 0) {
                        // 利用反射获取该服务的所有方法集合
                        Method[] methods = interfaceClass.getMethods();
                        // visit all methods
                        if (methods != null && methods.length > 0) {
                            // 遍历所有方法
                            for (int i = 0; i < methods.length; i++) {
                                //方法名
                                String methodName = methods[i].getName();
                                // target the method, and get its signature
                                // 找到目标方法
                                if (methodName.equals(method.getName())) {
                                    // 通过反射获取目标方法的参数类型数组 argtypes
                                    Class<?>[] argtypes = methods[i].getParameterTypes();
                                    // one callback in the method
                                    // 如果下标不为-1
                                    if (argument.getIndex() != -1) {
                                        // 检测 argType 的名称与 ArgumentConfig 中的 type 属性是否一致
                                        if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                            //  添加 ArgumentConfig 字段信息到 map 中
                                            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                        } else {
                                            // 不一致，则抛出异常
                                            throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                        }
                                    } else {
                                        // multiple callbacks in the method
                                        // 遍历参数类型数组 argtypes，查找 argument.type 类型的参数
                                        for (int j = 0; j < argtypes.length; j++) {
                                            Class<?> argclazz = argtypes[j];
                                            if (argclazz.getName().equals(argument.getType())) {
                                                // 如果找到，则添加 ArgumentConfig 字段信息到 map 中
                                                appendParameters(map, argument, method.getName() + "." + j);
                                                if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                    throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    } else if (argument.getIndex() != -1) {
                        // 用户未配置 type 属性，但配置了 index 属性，且 index != -1，则直接添加到map
                        appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                        // 抛出异常
                        throw new IllegalArgumentException("Argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                    }

                }
            }
        } // end of methods for
    }
    // 如果是泛化调用，则在map中设置generic和methods
    if (ProtocolUtils.isGeneric(generic)) {
        map.put(GENERIC_KEY, generic);
        map.put(METHODS_KEY, ANY_VALUE);
    } else {
        // 获得版本号
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            // 放入map
            map.put(REVISION_KEY, revision);
        }

        // 通过接口对应的Wrapper，拿到接口中所有的方法名字
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        // 如果为空，则告警
        if (methods.length == 0) {
            logger.warn("No method found in service interface " + interfaceClass.getName());
            map.put(METHODS_KEY, ANY_VALUE);
        } else {
            // 否则加入方法集合
            map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }

    // Token是为了防止服务被消费者直接调用（伪造http请求）
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(TOKEN_KEY, token);
        }
    }

    // export service
    // 通过该host和port访问该服务
    String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = this.findConfigedPorts(protocolConfig, name, map);
    // 服务url
    URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);
    // url：http://192.168.40.17:80/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&bean.name=ServiceBean:org.apache.dubbo.demo.DemoService&bind.ip=192.168.40.17&bind.port=80&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=285072&release=&side=provider&timestamp=1585206500409

    // —————————————————————————————————————分割线———————————————————————————————————————


    // 加载 ConfiguratorFactory，并生成 Configurator 实例，判断是否有该协议的实现存在
    // 可以通过ConfiguratorFactory，对服务url再次进行配置
    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        // 通过实例配置 url
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(SCOPE_KEY); // scope可能为null，remote, local,none
    // don't export when none is configured
    // 如果scope为none,则不会进行任何的服务导出，既不会远程，也不会本地
    if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

        // export to local if the config is not remote (export to remote only when config is remote)
        // scope != remote，暴露到本地
        if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
            // 如果scope不是remote,则会进行本地导出，会把当前url的protocol改为injvm，然后进行导出
            exportLocal(url);
        }
        // export to remote if the config is not local (export to local only when config is local)
        // 如果scope不是local,则会进行远程导出
        if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
            // 如果注册中心集合不为空
            if (CollectionUtils.isNotEmpty(registryURLs)) {
                // 如果有注册中心，则将服务注册到注册中心
                for (URL registryURL : registryURLs) {

                    //if protocol is only injvm ,not register
                    // 如果是injvm，则不需要进行注册中心注册
                    if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                        continue;
                    }

                    // 该服务是否是动态，对应zookeeper上表示是否是临时节点，对应dubbo中的功能就是静态服务
                    url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));

                    // 拿到监控中心地址
                    URL monitorUrl = loadMonitor(registryURL);

                    // 当前服务连接哪个监控中心
                    if (monitorUrl != null) {
                        // 添加监视器配置
                        url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                    }

                    // 服务的register参数，如果为true，则表示要注册到注册中心
                    if (logger.isInfoEnabled()) {
                        if (url.getParameter(REGISTER_KEY, true)) {
                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                        } else {
                            logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                        }
                    }

                    // For providers, this is used to enable custom proxy to generate invoker
                    // 服务使用的动态代理机制，如果为空则使用javassit
                    String proxy = url.getParameter(PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        // 添加代理方式到注册中心到url
                        registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                    }

                    // 基于registryUrl生成一个当前服务接口的代理对象   所以生成的invoker的url其protocol是registry协议
                    // 使用代理生成一个Invoker，Invoker表示服务提供者的代理，可以使用Invoker的invoke方法执行服务
                    // 对应的url为 registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-annotation-provider&dubbo=2.0.2&export=http%3A%2F%2F192.168.40.17%3A80%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddubbo-demo-annotation-provider%26bean.name%3DServiceBean%3Aorg.apache.dubbo.demo.DemoService%26bind.ip%3D192.168.40.17%26bind.port%3D80%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26methods%3DsayHello%26pid%3D19472%26release%3D%26side%3Dprovider%26timestamp%3D1585207994860&pid=19472&registry=zookeeper&timestamp=1585207994828
                    // 这个Invoker中包括了服务的实现者、服务接口类、服务的注册地址（针对当前服务的，参数export指定了当前服务）
                    // 此invoker表示一个可执行的服务，调用invoker的invoke()方法即可执行服务,同时此invoker也可用来导出
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));


                    // DelegateProviderMetaDataInvoker也表示服务提供者，包括了Invoker和服务的配置
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    /**
                     * ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
                     */
                    // 使用特定的协议来对服务进行导出，这里的协议为RegistryProtocol，导出成功后得到一个Exporter
                    // 1. 先使用RegistryProtocol进行服务注册
                    // 2. 注册完了之后，使用DubboProtocol进行导出
                    // 到此为止做了哪些事情？ ServiceBean.export()-->刷新ServiceBean的参数-->得到注册中心URL和协议URL-->遍历每个协议URL-->组成服务URL-->生成可执行服务Invoker-->导出服务
                    //RegistryProtocol.export
                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    // 加入到暴露者集合中
                    exporters.add(exporter);
                }
            } else {
                // 没有配置注册中心时，也会导出服务
                // 不存在注册中心，则仅仅暴露服务，不会记录暴露到地址
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }


                Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = protocol.export(wrapperInvoker);
                exporters.add(exporter);
            }


            /**
             * @since 2.7.0
             * ServiceData Store
             */
            // 根据服务url，讲服务的元信息存入元数据中心
            MetadataReportService metadataReportService = null;
            // 如果元数据中心服务不为空，则发布该服务，也就是在元数据中心记录url中到部分配置
            if ((metadataReportService = getMetadataReportService()) != null) {
                metadataReportService.publishProvider(url);
            }
        }
    }
    this.urls.add(url);
}
```

先看分割线上面部分，就是组装URL的全过程，我觉得大致可以分为一下步骤：

1. 它把metrics、application、module、provider、protocol等所有配置都放入map中，
2. 针对method都配置，先做签名校验，先找到该服务是否有配置的方法存在，然后该方法签名是否有这个参数存在，都核对成功才将method的配置加入map。
3. 将泛化调用、版本号、method或者methods、token等信息加入map
4. 获得服务暴露地址和端口号，利用map内数据组装成URL。

##### 创建invoker（2，3）

暴露到远程的源码直接看doExportUrlsFor1Protocol()方法分割线下半部分。当生成暴露者的时候，服务已经暴露，接下来会细致的分析这暴露内部的过程。可以发现无论暴露到本地还是远程，都会通过代理工厂创建invoker。这个时候就走到了上述时序图的ProxyFactory。首先我们来看看invoker如何诞生的。Invoker 是由 ProxyFactory 创建而来，Dubbo 默认的 ProxyFactory 实现类是 JavassistProxyFactory。JavassistProxyFactory中有一个getInvoker()方法

###### 获取invoker方法

getInvoker()

```java
public class JavassistProxyFactory extends AbstractProxyFactory {	
	 public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {

        // 如果现在被代理的对象proxy本身就是一个已经被代理过的对象，那么则取代理类的Wrapper，否则取type（接口）的Wrapper
        // Wrapper是针对某个类或某个接口的包装类，通过wrapper对象可以更方便的去执行某个类或某个接口的方法
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);

        // proxy是服务实现类
        // type是服务接口
        // url是一个注册中心url
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {

                // 执行proxy的method方法
                // 执行的proxy实例的方法
                // 如果没有wrapper，则要通过原生的反射技术去获取Method对象，然后执行
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
}
```

可以看到，该方法就是创建了一个匿名的Invoker类对象，在doInvoke()方法中调用wrapper.invokeMethod()方法。Wrapper 是一个抽象类，仅可通过 getWrapper(Class) 方法创建子类。在创建 Wrapper 子类的过程中，子类代码生成逻辑会对 getWrapper 方法传入的 Class 对象进行解析，拿到诸如类方法，类成员变量等信息。以及生成 invokeMethod 方法代码和其他一些方法代码。代码生成完毕后，通过 Javassist 生成 Class 对象，最后再通过反射创建 Wrapper 实例。那么我们先来看看getWrapper()方法：

```java
public abstract class Wrapper {
	public static Wrapper getWrapper(Class<?> c) {
        // 判断c是否继承 ClassGenerator.DC.class ，如果是，则拿到父类，避免重复包装
        while (ClassGenerator.isDynamicClass(c)) // can not wrapper on dynamic class.
        {
            c = c.getSuperclass();
        }
        // 如果类为object类型
        if (c == Object.class) {
            return OBJECT_WRAPPER;
        }

        //从缓存中拿
        Wrapper ret = WRAPPER_MAP.get(c);
        // 如果缓存里面没有该对象，则新建一个wrapper
        if (ret == null) {
            //创建Wrapper
            ret = makeWrapper(c);
            //写入缓存
            WRAPPER_MAP.put(c, ret);
        }
        return ret;
    }

}
```

该方法只是对Wrapper 做了缓存。主要的逻辑在makeWrapper()。

###### makeWrapper()

```java
private static Wrapper makeWrapper(Class<?> c) {
    // 如果c是私有类，则抛出异常
    if (c.isPrimitive()) {
        throw new IllegalArgumentException("Can not create wrapper for primitive type: " + c);
    }

    // 获得类名
    String name = c.getName();
    // 获得类加载器
    ClassLoader cl = ClassUtils.getClassLoader(c);
    // 设置属性的方法第一行public void setPropertyValue(Object o, String n, Object v){
    StringBuilder c1 = new StringBuilder("public void setPropertyValue(Object o, String n, Object v){ ");
    // 获得属性的方法第一行 public Object getPropertyValue(Object o, String n){
    StringBuilder c2 = new StringBuilder("public Object getPropertyValue(Object o, String n){ ");
    // 执行方法的第一行
    StringBuilder c3 = new StringBuilder("public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws " + InvocationTargetException.class.getName() + "{ ");
    // 添加每个方法中被调用对象的类型转换的代码
    c1.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
    c2.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
    c3.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");

    Map<String, Class<?>> pts = new HashMap<>(); // <property name, property types>
    Map<String, Method> ms = new LinkedHashMap<>(); // <method desc, Method instance>
    List<String> mns = new ArrayList<>(); // method names.
    List<String> dmns = new ArrayList<>(); // declaring method names.

    // get all public field.
    // 遍历每个public的属性，放入setPropertyValue和getPropertyValue方法中
    for (Field f : c.getFields()) {
        //属性名
        String fn = f.getName();
        //熟悉类型
        Class<?> ft = f.getType();
        // 排除有static 和 transient修饰的属性
        if (Modifier.isStatic(f.getModifiers()) || Modifier.isTransient(f.getModifiers())) {
            continue;
        }

        c1.append(" if( $2.equals(\"").append(fn).append("\") ){ w.").append(fn).append("=").append(arg(ft, "$3")).append("; return; }");
        c2.append(" if( $2.equals(\"").append(fn).append("\") ){ return ($w)w.").append(fn).append("; }");
        pts.put(fn, ft);
    }

    Method[] methods = c.getMethods();
    // get all public method.
    boolean hasMethod = hasMethods(methods);
    // 在invokeMethod方法中添加try的代码
    if (hasMethod) {
        c3.append(" try{");
        // 遍历方法
        for (Method m : methods) {
            //ignore Object's method.
            // 忽律Object的方法
            if (m.getDeclaringClass() == Object.class) {
                continue;
            }
            // 判断方法名和方法参数长度
            String mn = m.getName();

            c3.append(" if( \"").append(mn).append("\".equals( $2 ) ");
            // 方法参数长度
            int len = m.getParameterTypes().length;
            // 判断方法参数长度代码
            c3.append(" && ").append(" $3.length == ").append(len);

            boolean override = false;

            for (Method m2 : methods) {
                // 若相同方法名存在多个，增加参数类型数组的比较判断
                if (m != m2 && m.getName().equals(m2.getName())) {
                    override = true;
                    break;
                }
            }
            if (override) {
                if (len > 0) {
                    for (int l = 0; l < len; l++) {
                        c3.append(" && ").append(" $3[").append(l).append("].getName().equals(\"")
                                .append(m.getParameterTypes()[l].getName()).append("\")");
                    }
                }
            }

            c3.append(" ) { ");
            // 如果返回类型是void，则return null，如果不是，则返回对应参数类型
            if (m.getReturnType() == Void.TYPE) {
                c3.append(" w.").append(mn).append('(').append(args(m.getParameterTypes(), "$4")).append(");").append(" return null;");
            } else {
                c3.append(" return ($w)w.").append(mn).append('(').append(args(m.getParameterTypes(), "$4")).append(");");
            }

            c3.append(" }");

            mns.add(mn);
            if (m.getDeclaringClass() == c) {
                dmns.add(mn);
            }
            ms.put(ReflectUtils.getDesc(m), m);
        }
        c3.append(" } catch(Throwable e) { ");
        c3.append("     throw new java.lang.reflect.InvocationTargetException(e); ");
        c3.append(" }");
    }

    c3.append(" throw new " + NoSuchMethodException.class.getName() + "(\"Not found method \\\"\"+$2+\"\\\" in class " + c.getName() + ".\"); }");

    // deal with get/set method.

    // 处理get set方法
    Matcher matcher;
    for (Map.Entry<String, Method> entry : ms.entrySet()) {
        String md = entry.getKey();
        Method method = entry.getValue();
        if ((matcher = ReflectUtils.GETTER_METHOD_DESC_PATTERN.matcher(md)).matches()) {
            String pn = propertyName(matcher.group(1));
            c2.append(" if( $2.equals(\"").append(pn).append("\") ){ return ($w)w.").append(method.getName()).append("(); }");
            pts.put(pn, method.getReturnType());
        } else if ((matcher = ReflectUtils.IS_HAS_CAN_METHOD_DESC_PATTERN.matcher(md)).matches()) {
            String pn = propertyName(matcher.group(1));
            c2.append(" if( $2.equals(\"").append(pn).append("\") ){ return ($w)w.").append(method.getName()).append("(); }");
            pts.put(pn, method.getReturnType());
        } else if ((matcher = ReflectUtils.SETTER_METHOD_DESC_PATTERN.matcher(md)).matches()) {
            Class<?> pt = method.getParameterTypes()[0];
            String pn = propertyName(matcher.group(1));
            c1.append(" if( $2.equals(\"").append(pn).append("\") ){ w.").append(method.getName()).append("(").append(arg(pt, "$3")).append("); return; }");
            pts.put(pn, pt);
        }
    }
    c1.append(" throw new " + NoSuchPropertyException.class.getName() + "(\"Not found property \\\"\"+$2+\"\\\" field or setter method in class " + c.getName() + ".\"); }");
    c2.append(" throw new " + NoSuchPropertyException.class.getName() + "(\"Not found property \\\"\"+$2+\"\\\" field or setter method in class " + c.getName() + ".\"); }");

    // make class
    long id = WRAPPER_CLASS_COUNTER.getAndIncrement();
    ClassGenerator cc = ClassGenerator.newInstance(cl);
    cc.setClassName((Modifier.isPublic(c.getModifiers()) ? Wrapper.class.getName() : c.getName() + "$sw") + id);
    cc.setSuperClass(Wrapper.class);
    // 增加无参构造器
    cc.addDefaultConstructor();
    // 添加属性
    cc.addField("public static String[] pns;"); // property name array.
    cc.addField("public static " + Map.class.getName() + " pts;"); // property type map.
    cc.addField("public static String[] mns;"); // all method name array.
    cc.addField("public static String[] dmns;"); // declared method name array.
    for (int i = 0, len = ms.size(); i < len; i++) {
        cc.addField("public static Class[] mts" + i + ";");
    }
    // 添加属性相关的方法
    cc.addMethod("public String[] getPropertyNames(){ return pns; }");
    cc.addMethod("public boolean hasProperty(String n){ return pts.containsKey($1); }");
    cc.addMethod("public Class getPropertyType(String n){ return (Class)pts.get($1); }");
    cc.addMethod("public String[] getMethodNames(){ return mns; }");
    cc.addMethod("public String[] getDeclaredMethodNames(){ return dmns; }");
    cc.addMethod(c1.toString());
    cc.addMethod(c2.toString());
    cc.addMethod(c3.toString());
    System.out.println("wrapper类：");
    System.out.println(c1.toString());
    System.out.println(c2.toString());
    System.out.println(c3.toString());

    try {
        // 生成字节码
        Class<?> wc = cc.toClass();
        // setup static field.
        // 反射，设置静态变量的值
        wc.getField("pts").set(null, pts);
        wc.getField("pns").set(null, pts.keySet().toArray(new String[0]));
        wc.getField("mns").set(null, mns.toArray(new String[0]));
        wc.getField("dmns").set(null, dmns.toArray(new String[0]));
        int ix = 0;
        for (Method m : ms.values()) {
            wc.getField("mts" + ix++).set(null, m.getParameterTypes());
        }
        // 创建对象并且返回
        return (Wrapper) wc.newInstance();
    } catch (RuntimeException e) {
        throw e;
    } catch (Throwable e) {
        throw new RuntimeException(e.getMessage(), e);
    } finally {
        cc.release();
        ms.clear();
        mns.clear();
        dmns.clear();
    }
}
```

该方法有点长，大致可以分为几个步骤：

1. 初始化了c1、c2、c3、pts、ms、mns、dmns变量，向 c1、c2、c3 中添加方法定义和类型转换代码。
2. 为 public 级别的字段生成条件判断取值与赋值代码
3. 为定义在当前类中的方法生成判断语句，和方法调用语句。
4. 处理 getter、setter 以及以 is/has/can 开头的方法。处理方式是通过正则表达式获取方法类型（get/set/is/...），以及属性名。之后为属性名生成判断语句，然后为方法生成调用语句。
5. 通过 ClassGenerator 为刚刚生成的代码构建 Class 类，并通过反射创建对象。ClassGenerator 是 Dubbo 自己封装的，该类的核心是 toClass() 的重载方法 toClass(ClassLoader, ProtectionDomain)，该方法通过 javassist 构建 Class



#### 服务暴露

服务暴露分为暴露到本地 (JVM)，和暴露到远程。doExportUrlsFor1Protocol()方法分割线下半部分就是服务暴露的逻辑。根据scope的配置分为：

- scope = none，不暴露服务
- scope != remote，暴露到本地
- scope != local，暴露到远程

##### 暴露到本地（4）

导出本地执行的是ServiceConfig中的exportLocal()方法。

exportLocal

```java
public class ServiceConfig<T> extends AbstractServiceConfig {
    private void exportLocal(URL url) {
        // 生成本地的url,分别把协议改为injvm，设置host和port
        URL local = URLBuilder.from(url)
                .setProtocol(LOCAL_PROTOCOL)
                .setHost(LOCALHOST_VALUE)
                .setPort(0)
                .build();
        // 通过代理工程创建invoker
        // 再调用export方法进行暴露服务，生成Exporter
        //InjvmProtocol.export
        Exporter<?> exporter = protocol.export(
                PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local));
        // 把生成的暴露者加入集合
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
    }
}
```





###### InjvmProtocol.export（5）

```java
public class InjvmProtocol extends AbstractProtocol implements Protocol {
	public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        // 创建InjvmExporter 并且返回
        return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
    }

}
```

该方法只是创建了一个InjvmExporter，因为暴露到本地，所以在同一个jvm中。所以不需要其他操作。

##### 暴露到远程

暴露到远程的逻辑要比本地复杂的多，它大致可以分为服务暴露和服务注册两个过程。先来看看服务暴露。我们知道dubbo有很多协议实现，在doExportUrlsFor1Protocol()方法分割线下半部分中，生成了Invoker后，就需要调用protocol 的 export()方法，很多人会认为这里的export()就是配置中指定的协议实现中的方法，但这里是不对的。因为暴露到远程后需要进行服务注册，而RegistryProtocol的 export()方法就是实现了服务暴露和服务注册两个过程。所以这里的export()调用的是RegistryProtocol的 export()。

```java
public class RegistryProtocol implements Protocol {
	    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        // 导出服务
        // registry://   ---> RegistryProtocol
        // zookeeper://  ---> ZookeeperRegistry
        // dubbo://      ---> DubboProtocol

        // 获得注册中心的url registry://xxx?xx=xx&registry=zookeeper ---> zookeeper://xxx?xx=xx     表示注册中心
        URL registryUrl = getRegistryUrl(originInvoker); // zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=dubbo-demo-provider-application&dubbo=2.0.2&export=dubbo%3A%2F%2F192.168.40.17%3A20880%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application%3Ddubbo-demo-provider-application%26bean.name%3DServiceBean%3Aorg.apache.dubbo.demo.DemoService%26bind.ip%3D192.168.40.17%26bind.port%3D20880%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dorg.apache.dubbo.demo.DemoService%26logger%3Dlog4j%26methods%3DsayHello%26pid%3D27656%26release%3D2.7.0%26side%3Dprovider%26timeout%3D3000%26timestamp%3D1590735956489&logger=log4j&pid=27656&release=2.7.0&timestamp=1590735956479
        // 得到服务提供者url，表示服务提供者
        URL providerUrl = getProviderUrl(originInvoker); // dubbo://192.168.40.17:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-provider-application&bean.name=ServiceBean:org.apache.dubbo.demo.DemoService&bind.ip=192.168.40.17&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&logger=log4j&methods=sayHello&pid=27656&release=2.7.0&side=provider&timeout=3000&timestamp=1590735956489

        // Subscribe the override data

        // overrideSubscribeUrl是老版本的动态配置监听url，表示了需要监听的服务以及监听的类型（configurators， 这是老版本上的动态配置）
        // 在服务提供者url的基础上，生成一个overrideSubscribeUrl，协议为provider://，增加参数category=configurators&check=false
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);

        // 一个overrideSubscribeUrl对应一个OverrideListener，用来监听变化事件，监听到overrideSubscribeUrl的变化后，
        // OverrideListener就会根据变化进行相应处理，具体处理逻辑看OverrideListener的实现
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        // 把监听器添加到集合
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);


        // 在这个方法里会利用providerConfigurationListener和serviceConfigurationListener去重写providerUrl
        // providerConfigurationListener表示应用级别的动态配置监听器，providerConfigurationListener是RegistyProtocol的一个属性
        // serviceConfigurationListener表示服务级别的动态配置监听器，serviceConfigurationListener是在每暴露一个服务时就会生成一个
        // 这两个监听器都是新版本中的监听器
        // 新版本监听的zk路径是：
        // 服务： /dubbo/config/dubbo/org.apache.dubbo.demo.DemoService.configurators节点的内容
        // 应用： /dubbo/config/dubbo/dubbo-demo-provider-application.configurators节点的内容
        // 注意，要和配置中心的路径区分开来，配置中心的路径是：
        // 应用：/dubbo/config/dubbo/org.apache.dubbo.demo.DemoService/dubbo.properties节点的内容
        // 全局：/dubbo/config/dubbo/dubbo.properties节点的内容
        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);

        // export invoker
        // 服务暴露 根据动态配置重写了providerUrl之后，就会调用DubboProtocol或HttpProtocol去进行导出服务了
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // url to registry
        // 得到注册中心-ZookeeperRegistry
        final Registry registry = getRegistry(originInvoker);

        // 得到存入到注册中心去的providerUrl,会对服务提供者url中的参数进行简化（删除一些不需要的参数）
        final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);

        // 将当前服务提供者Invoker，以及该服务对应的注册中心地址，以及简化后的服务url存入ProviderConsumerRegTable
        ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                registryUrl, registeredProviderUrl);


        // ————————————————————————————————分割线——————————————————————————————————————


        //to judge if we need to delay publish
        //是否需要注册到注册中心
        boolean register = providerUrl.getParameter(REGISTER_KEY, true);
        if (register) {
            // 注册服务，把简化后的服务提供者url注册到registryUrl中去
            register(registryUrl, registeredProviderUrl);
            // 设置reg为true，表示服务注册过
            providerInvokerWrapper.setReg(true);
        }

        // 针对老版本的动态配置，需要把overrideSubscribeListener绑定到overrideSubscribeUrl上去进行监听
        // 兼容老版本的配置修改，利用overrideSubscribeListener去监听旧版本的动态配置变化
        // 监听overrideSubscribeUrl   provider://192.168.40.17:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-annotation-provider&bean.name=ServiceBean:org.apache.dubbo.demo.DemoService&bind.ip=192.168.40.17&bind.port=20880&category=configurators&check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=416332&release=&side=provider&timestamp=1585318241955
        // 那么新版本的providerConfigurationListener和serviceConfigurationListener是在什么时候进行订阅的呢？在这两个类构造的时候
        // Deprecated! Subscribe to override rules in 2.6.x or before.
        // 老版本监听的zk路径是：/dubbo/org.apache.dubbo.demo.DemoService/configurators/override://0.0.0.0/org.apache.dubbo.demo.DemoService?category=configurators&compatible_config=true&dynamic=false&enabled=true&timeout=6000
        // 监听的是路径的内容，不是节点的内容
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        // 设置注册中心url
        exporter.setRegisterUrl(registeredProviderUrl);
        // 设置override数据订阅的url
        exporter.setSubscribeUrl(overrideSubscribeUrl);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<>(exporter);
    }

}
```

从代码上看，我用分割线分成两部分，分别是服务暴露和服务注册。该方法的逻辑大致分为以下几个步骤：

1. 获得服务提供者的url，再通过override数据重新配置url，然后执行doLocalExport()进行服务暴露。
2. 加载注册中心实现类，向注册中心注册服务。
3. 向注册中心进行订阅 override 数据。
4. 创建并返回 DestroyableExporter



服务暴露先调用的是RegistryProtocol的doLocalExport()方法

```java
public class RegistryProtocol implements Protocol {
	 private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
        String key = getCacheKey(originInvoker);

        return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
            Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
            // protocol属性的值是哪来的，是在SPI中注入进来的，是一个代理类
            // 这里实际利用的就是DubboProtocol或HttpProtocol去export  NettyServer
            // 为什么需要ExporterChangeableWrapper？方便注销已经被导出的服务
            //invokerDelegate里的URL是服务提供者的url，最终会调用DubboProtocol或HttpProtocol
            return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
        });
    }

}
```

这里的逻辑比较简单，主要是在这里根据不同的协议配置，调用不同的protocol实现。跟暴露到本地的时候实现InjvmProtocol一样。我这里假设配置选用的是dubbo协议，来继续下面的介绍。



DubboProtocol#export

```java
public class DubboProtocol extends AbstractProtocol {
 public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl();

        // export service.
        //  得到服务key  group+"/"+serviceName+":"+serviceVersion+":"+port
        String key = serviceKey(url);
        // 构造一个Exporter进行服务导出
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        // 加入到集合
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
        // 如果是本地存根事件而不是回调服务
        if (isStubSupportEvent && !isCallbackservice) {
            // 获得本地存根的方法
            String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
            // 如果为空，则抛出异常
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }

            } else {
                // 服务的stub方法
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

        // 开启NettyServer
        openServer(url);  //请求--->invocation--->服务key--->exporterMap.get(key)--->exporter--->invoker--->invoker.invoke(invocation)-->执行服务

        // 特殊的一些序列化机制，比如kryo提供了注册机制来注册类，提高序列化和反序列化的速度
        optimizeSerialization(url);

        return exporter;
    }

}
```



该方法的逻辑大致分为以下几个步骤：

1、处理本地存根

2、启动服务器



###### DubboProtocol的openServer()

《dubbo源码解析（二十四）远程调用——dubbo协议》中的（三）DubboProtocol有openServer()相关的源码分析， 不过该文章中的代码是2.6.x的代码，最新的版本中加入了 **DCL**。其中reset方法则是重置服务器的一些配置。例如在同一台机器上（单网卡），同一个端口上仅允许启动一个服务器实例。若某个端口上已有服务器实例，此时则调用 reset 方法重置服务器的一些配置。主要来看其中的createServer()方法。





###### DubboProtocol的createServer()

《dubbo源码解析（二十四）远程调用——dubbo协议》中的（三）DubboProtocol有createServer()相关的源码分析，其中最新版本的默认远程通讯服务端实现方式已经改为netty4。该方法大致可以分为以下几个步骤：

1. 对服务端远程通讯服务端实现方式配置是否支持的检测
2. 创建服务器实例，也就是调用bind()方法
3. 对服务端远程通讯客户端实现方式配置是否支持的检测



###### Exchangers的bind()

可以参考《dubbo源码解析（十）远程通信——Exchange层》中的（二十一）Exchangers。

```java
public class Exchangers {
	  public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        // codec表示协议编码方式
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        // 通过url得到HeaderExchanger， 利用HeaderExchanger进行bind，将得到一个HeaderExchangeServer
        return getExchanger(url).bind(url, handler);
    }

}
```



###### HeaderExchanger的bind()

可以参考《dubbo源码解析（十）远程通信——Exchange层》（十六）HeaderExchanger，其中bind()方法做了大致以下步骤：

1. 创建HeaderExchangeHandler
2. 创建DecodeHandler
3. Transporters.bind()，创建服务器实例。
4. 创建HeaderExchangeServer



其中HeaderExchangeHandler、DecodeHandler、HeaderExchangeServer可以参考《dubbo源码解析（十）远程通信——Exchange层》中的讲解。



```java
public class Transporters {
	    public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handlers == null || handlers.length == 0) {
            throw new IllegalArgumentException("handlers == null");
        }

        // 如果bind了多个handler，那么当有一个连接过来时，会循环每个handler去处理连接
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }

        // 调用Transporter的实现类对象的bind方法。
        // 调用NettyTransporter去绑定，Transporter表示网络传输层
        //org.apache.dubbo.remoting.transport.netty4.NettyTransporter.bind
        return getTransporter().bind(url, handler);
    }

}
```

getTransporter() 方法获取的 Transporter 是在运行时动态创建的，类名为 TransporterAdaptive，也就是自适应拓展类。TransporterAdaptive 会在运行时根据传入的 URL 参数决定加载什么类型的 Transporter，默认为基于Netty4的实现。假设是 NettyTransporter 的 bind 方法



###### NettyTransporter的bind()（6）

可以参考《dubbo源码解析（十七）远程通信——Netty4》的（六）NettyTransporter。

```java
public class NettyTransporter implements Transporter {
	public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }
}
```



###### NettyServer的构造方法（7）

可以参考《dubbo源码解析（十七）远程通信——Netty4》的（五）NettyServer

```java
public class NettyServer extends AbstractServer implements Server {
	 public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        // you can customize name and type of client thread pool by THREAD_NAME_KEY and THREADPOOL_KEY in CommonConstants.
        // the handler will be warped: MultiMessageHandler->HeartbeatHandler->handler
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
    }
}
```

调用的是父类AbstractServer构造方法

###### AbstractServer的构造方法（7）

可以参考《dubbo源码解析（九）远程通信——Transport层》的（三）AbstractServer中的构造方法。

服务器实例创建完以后，就是开启服务器了，AbstractServer中的doOpen是抽象方法，还是拿netty4来讲解，也就是看NettyServer的doOpen()的方法。

###### NettyServer的doOpen()

可以参考《dubbo源码解析（十七）远程通信——Netty4》中的（五）NettyServer中的源码分析。这里执行完成后，服务器被开启，服务也暴露出来了。接下来就是讲解服务注册的内容。

##### 服务注册（9）

dubbo服务注册并不是必须的，因为dubbo支持直连的方式就可以绕过注册中心。直连的方式很多时候用来做服务测试。

回过头去看一下RegistryProtocol的 export()方法的分割线下面部分。其中服务注册先调用的是register()方法。

###### RegistryProtocol的register()

```java
public class RegistryProtocol implements Protocol {
	public void register(URL registryUrl, URL registeredProviderUrl) {
        // 获取 Registry
        Registry registry = registryFactory.getRegistry(registryUrl);

        //FailbackRegistry.register
        registry.register(registeredProviderUrl);
    }

}
```

所以服务注册大致可以分为两步：

1. 获得注册中心实例
2. 注册服务

获得注册中心首先执行的是AbstractRegistryFactory的getRegistry()方法

###### AbstractRegistryFactory的getRegistry()

可以参考《dubbo源码解析（三）注册中心——开篇》的（七）support包下的AbstractRegistryFactory中的源码解析。大概的逻辑就是先从缓存中取，如果没有命中，则创建注册中心实例，这里的createRegistry()是一个抽象方法，具体的实现逻辑由子类完成，假设这里使用zookeeper作为注册中心，则调用的是ZookeeperRegistryFactory的createRegistry()。

###### ZookeeperRegistryFactory的createRegistry()

```java
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {
    @Override
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }

}
```

就是创建了一个ZookeeperRegistry，执行了ZookeeperRegistry的构造方法。

###### ZookeeperRegistry的构造方法

可以参考《dubbo源码解析（七）注册中心——zookeeper》的（一）ZookeeperRegistry中的源码分析。大致的逻辑可以分为以下几个步骤：

1. 创建zookeeper客户端
2. 添加监听器

主要看ZookeeperTransporter的connect方法，因为当connect方法执行完后，注册中心创建过程就结束了。首先执行的是AbstractZookeeperTransporter的connect方法。

###### AbstractZookeeperTransporter的connect()

```java
public abstract class AbstractZookeeperTransporter implements ZookeeperTransporter {
    public ZookeeperClient connect(URL url) {
        ZookeeperClient zookeeperClient;
        // 获得所有url地址
        List<String> addressList = getURLBackupAddress(url);
        // The field define the zookeeper server , including protocol, host, port, username, password
        // 从缓存中查找可用的客户端，如果有，则直接返回
        if ((zookeeperClient = fetchAndUpdateZookeeperClientCache(addressList)) != null && zookeeperClient.isConnected()) {
        logger.info("find valid zookeeper client from the cache for address: " + url);
        return zookeeperClient;
        }
        // avoid creating too many connections， so add lock
        synchronized (zookeeperClientMap) {
        if ((zookeeperClient = fetchAndUpdateZookeeperClientCache(addressList)) != null && zookeeperClient.isConnected()) {
        logger.info("find valid zookeeper client from the cache for address: " + url);
        return zookeeperClient;
        }
        // 创建客户端
        zookeeperClient = createZookeeperClient(toClientURL(url));
        logger.info("No valid zookeeper client found from cache, therefore create a new client for url. " + url);
        // 加入缓存
        writeToClientMap(addressList, zookeeperClient);
        }
        return zookeeperClient;
    }
}
```



看上面的源码，主要是执行了createZookeeperClient()方法，而该方法是一个抽象方法，由子类实现，这里是CuratorZookeeperTransporter的createZookeeperClient()

###### CuratorZookeeperTransporter的createZookeeperClient()

```java
public class CuratorZookeeperTransporter extends AbstractZookeeperTransporter {
    @Override
    public ZookeeperClient createZookeeperClient(URL url) {
        //创建CuratorZookeeperClient
        return new CuratorZookeeperClient(url);
    }
}
```

这里就是执行了CuratorZookeeperClient的构造方法。

###### CuratorZookeeperClient的构造方法

可以参考《dubbo源码解析（十八）远程通信——Zookeeper》的（四）CuratorZookeeperClient中的源码分析，其中逻辑主要用于创建和启动 CuratorFramework 实例，基本都是调用Curator框架的API。

创建完注册中心的实例后，我们就要进行注册服务了。也就是调用的是FailbackRegistry的register()方法。

###### FailbackRegistry的register()

可以参考《dubbo源码解析（三）注册中心——开篇》的（六）support包下的FailbackRegistry中的源码分析。可以看到关键是执行了doRegister()方法，该方法是抽象方法，由子类完成。这里因为假设是zookeeper，所以执行的是ZookeeperRegistry的doRegister()。

###### ZookeeperRegistry的doRegister()

可以参考《dubbo源码解析（七）注册中心——zookeeper》的（一）ZookeeperRegistry中的源代码，可以看到逻辑就是调用Zookeeper 客户端创建服务节点。节点路径由 toUrlPath 方法生成。而这里create方法执行的是AbstractZookeeperClient的create() 方法

###### AbstractZookeeperClient的create()

可以参考《dubbo源码解析（十八）远程通信——Zookeeper》的（二）AbstractZookeeperClient中的源代码分析。createEphemeral()和createPersistent()是抽象方法，具体实现由子类完成，也就是CuratorZookeeperClient类。代码逻辑比较简单。我就不再赘述。到这里为止，服务也就注册完成。

关于向注册中心进行订阅 override 数据的规则在最新版本有一些大变动，跟2.6.x及以前的都不一样。所以这部分内容在新特性中去讲解

















