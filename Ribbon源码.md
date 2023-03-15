​				



​                                                                                   Ribbon源码

```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```







客户端调用与负载均衡

基于RestTemplate的拦截器机制



ribbon作用：1、负载均衡 2、微服务名 -> ip:port

# 自动配置

spring-cloud-netflix-ribbon包下META-INF/spring.factories照片那个引入了RibbonAutoConfiguration自动配置类

```java
@RibbonClients
@AutoConfigureBefore({LoadBalancerAutoConfiguration.class, AsyncLoadBalancerAutoConfiguration.class})
@EnableConfigurationProperties({RibbonEagerLoadProperties.class, ServerIntrospectorProperties.class})
public class RibbonAutoConfiguration {

}
```

RibbonAutoConfiguration中导入了LoadBalancerAutoConfiguration

```java
@ConditionalOnClass({RestTemplate.class})
@ConditionalOnBean({LoadBalancerClient.class})
@EnableConfigurationProperties({LoadBalancerRetryProperties.class})
public class LoadBalancerAutoConfiguration {
    
    //注入所有带有@LoadBalanced注解的RestTemplate类型的bean到restTemplates集合中
    @LoadBalanced
    @Autowired(
        required = false
    )
    private List<RestTemplate> restTemplates = Collections.emptyList();
   
    @ConditionalOnMissingClass({"org.springframework.retry.support.RetryTemplate"})
    static class LoadBalancerInterceptorConfig {
        	//给容器中注入负载均衡拦截器
            @Bean
            public LoadBalancerInterceptor ribbonInterceptor(LoadBalancerClient loadBalancerClient, LoadBalancerRequestFactory requestFactory) {
                return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
            }
        
			      //给容器中导入RestTemplateCustomizer
            @Bean
            @ConditionalOnMissingBean
            public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
                return (restTemplate) -> {
                    //自己添加拦截器并不会让ribbon的拦截器失效
                    List<ClientHttpRequestInterceptor> list = new ArrayList(restTemplate.getInterceptors());
                    list.add(loadBalancerInterceptor);
                    restTemplate.setInterceptors(list);
                };
            }
    }
    
    
  @Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            //给所有带有@LoadBalanced注解的RestTemplate类型的bean进行自定义配置 （添加拦截器）
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);
				}
			}
		});
	}
}
```



```java
@Import(RibbonClientConfigurationRegistrar.class)
public @interface RibbonClients {}
```



保存各ribbon client对应的配置  在后面创建RibbonClient对应的ApplicationContext，并注册所有可用的Configuration配置类

```java
public class RibbonClientConfigurationRegistrar implements ImportBeanDefinitionRegistrar {

   @Override
   public void registerBeanDefinitions(AnnotationMetadata metadata,
         BeanDefinitionRegistry registry) {
       //1、@RibbonClients注解 
       //@RibbonClients(        value = {                @RibbonClient(name = "user-service",configuration = UserRibbonConfig.class),               @RibbonClient(name = "order-service",configuration = OderRibbonConfig.class)        } )
      Map<String, Object> attrs = metadata.getAnnotationAttributes(
            RibbonClients.class.getName(), true);
      //1.1 value是RibbonClient[]，遍历针对具体的RibbonClient配置的configuration配置类，并注册
      if (attrs != null && attrs.containsKey("value")) {
         AnnotationAttributes[] clients = (AnnotationAttributes[]) attrs.get("value");
         for (AnnotationAttributes client : clients) {
            registerClientConfiguration(registry, getClientName(client),
                  client.get("configuration"));
         }
      }
        // 1.2 找到@RibbonClients注解的defaultConfiguration，即默认配置
       // @RibbonClients(         defaultConfiguration = MyRibbonConfig.class)
        //     注册成以default.Classname.RibbonClientSpecification为名的RibbonClientSpecification
      if (attrs != null && attrs.containsKey("defaultConfiguration")) {
         String name;
         if (metadata.hasEnclosingClass()) {
            name = "default." + metadata.getEnclosingClassName();
         } else {
            name = "default." + metadata.getClassName();
         }
         registerClientConfiguration(registry, name,
               attrs.get("defaultConfiguration"));
      }
       // 2、@RibbonClient注解
       //@RibbonClient(name = "user",configuration = UserRibbonConfig.class)
        // 注册某个具体Ribbon Client的configuration配置类
      Map<String, Object> client = metadata.getAnnotationAttributes(
            RibbonClient.class.getName(), true);
      String name = getClientName(client);
      if (name != null) {
         registerClientConfiguration(registry, name, client.get("configuration"));
      }
   }
```













# LoadBalancerInterceptor



ribbon拦截RestTemplate的拦截器



LoadBalancerInterceptor

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

   private LoadBalancerClient loadBalancer;

   private LoadBalancerRequestFactory requestFactory;

   public LoadBalancerInterceptor(LoadBalancerClient loadBalancer,
         LoadBalancerRequestFactory requestFactory) {
      this.loadBalancer = loadBalancer;
      this.requestFactory = requestFactory;
   }

   public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
      // for backwards compatibility
      this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
   }

    //拦截RestTemplate
   @Override
   public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
         final ClientHttpRequestExecution execution) throws IOException {
      //请求全路径 http://user-service/getUser
      final URI originalUri = request.getURI();
      //微服务名 user-service
      String serviceName = originalUri.getHost();
      return this.loadBalancer.execute(serviceName,
            this.requestFactory.createRequest(request, body, execution));
   }

}
```

org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#execute()

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
    return this.execute(serviceId, (LoadBalancerRequest)request, (Object)null);
}
```

org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#execute()

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
    //根据微服务名拿负载均衡器
    ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
    //进行负载均衡 通过指定的负载均衡器，返回一个服务器地址
    Server server = this.getServer(loadBalancer, hint);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    } else {
        RibbonLoadBalancerClient.RibbonServer ribbonServer = new RibbonLoadBalancerClient.RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
      	//执行请求
        return this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request);
    }
}
```

## LoadBalancer

负载均衡器 一个微服务名字对应一个负载均衡器

不同的负载均衡器对应不同的SpingContext

职能：

维护Sever列表的数量（新增、更新、删除）

维护Server列表的状态



org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#getLoadBalancer

```java
protected ILoadBalancer getLoadBalancer(String serviceId) {
    return this.clientFactory.getLoadBalancer(serviceId);
}
```

org.springframework.cloud.netflix.ribbon.SpringClientFactory#getLoadBalancer

```java
public ILoadBalancer getLoadBalancer(String name) {
    return (ILoadBalancer)this.getInstance(name, ILoadBalancer.class);
}
```

### SpringClientFactory#getInstance

会给每个ribbon client创建一个独立的spring应用上下文ApplicationContext，并在其中加载对应的配置及ribbon核心接口的实现类

```java
//name代表当前的ribbon客户端 type代表要获取实例类型 如IClient、IRule
public <C> C getInstance(String name, Class<C> type) {
	  //返回一个DynamicServerListLoadBalancer
    // 先从父类NamedContextFactory中直接从客户端对应的ApplicationContext中获取实例
    // 如果没有就根据IClientConfig中的配置找到具体的实现类，并通过反射初始化后放到Client对应的ApplicationContext中
    C instance = super.getInstance(name, type);
    if (instance != null) {
        return instance;
    } else {
        IClientConfig config = (IClientConfig)this.getInstance(name, IClientConfig.class);
        return instantiateWithConfig(this.getContext(name), type, config);
    }
}
```

org.springframework.cloud.context.named.NamedContextFactory#getInstance()

```java
public <T> T getInstance(String name, Class<T> type) {
    //根据微服务名拿到对应的上下文
    //可以针对不同的微服务调用 配置隔离 自定义不同的负载均衡组件 比如负载均衡策略、服务列表   
    AnnotationConfigApplicationContext context = this.getContext(name);
    return BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context, type).length > 0 ? context.getBean(type) : null;
}
```

org.springframework.cloud.context.named.NamedContextFactory#getContext

```java
// 如果包含Client上下文直接返回
// 如果不包含，调用createContext(name)，并放入contexts集合
protected AnnotationConfigApplicationContext getContext(String name) {
    if (!this.contexts.containsKey(name)) {
        synchronized(this.contexts) {
            if (!this.contexts.containsKey(name)) {
                this.contexts.put(name, this.createContext(name));
            }
        }
    }

    return (AnnotationConfigApplicationContext)this.contexts.get(name);
}
```

### createContext()：构建父子容器

org.springframework.cloud.context.named.NamedContextFactory#createContext

```java
// 创建名为name的RibbonClient的ApplicationContext上下文
protected AnnotationConfigApplicationContext createContext(String name) {
    //创建一个新的容器
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    //1、用户指定的配置 只对某个ribbon client生效 注册专门为当前RibbonClient指定的configuration配置类，@RibbonClient注解  @RibbonClients注解
    if (this.configurations.containsKey(name)) {
        for (Class<?> configuration : this.configurations.get(name)
             .getConfiguration()) {
            context.register(configuration);
        }
    }
    
    // 2、spring 的默认配置 将为所有RibbonClient的configuration配置类注册到ApplicationContext
    //configurations：RibbonAutoConfiguration、RibbonEurekaAutoConfiguration两个配置类
    //如果注册中心是nacos，configurations就是RibbonAutoConfiguration、RibbonNacosAutoConfiguration两个配置类
  	
    for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
        if (entry.getKey().startsWith("default.")) {
            for (Class<?> configuration : entry.getValue().getConfiguration()) {
                context.register(configuration);
            }
        }
    }
    
    // 3、注册defaultConfigType，即ribbon的默认配置类 RibbonClientConfiguration
  	//RibbonClientConfiguration向容器里注入默认的IRule、IPing、ServerList、ServerListUpdater、ILoadBalancer、ServerListFilter
    //defaultConfigType:RibbonClientConfiguration
    //如果整合了ribbonfeign，defaultConfigType就是FeignClientsConfiguration
    context.register(PropertyPlaceholderAutoConfiguration.class,
                     this.defaultConfigType);
    // 添加 ribbon.client.name=具体RibbonClient name的enviroment配置	 
    context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
        this.propertySourceName,
        Collections.<String, Object>singletonMap(this.propertyName, name)));
    if (this.parent != null) {
        //将当前容器设置为新创建容器的父容器 这样可以使得当前创建的子ApplicationContext可以使用父上下文中的Bean
        context.setParent(this.parent);
        context.setClassLoader(this.parent.getClassLoader());
    }
    context.setDisplayName(generateDisplayName(name));
    //刷新容器 根据上面两个配置类刷新容器
    context.refresh();
    return context;
}


```



### DynamicServerListLoadBalancer

在上一步骤刷新容器的过程中 会创建一个ILoadBalancer类型的bean--DynamicServerListLoadBalancer

最后get到的LoadBalancer实现类是DynamicServerListLoadBalancer

```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping, ServerList<T> serverList, ServerListFilter<T> filter, ServerListUpdater serverListUpdater) {
    //父类中会调用initWithConfig()方法
    super(clientConfig, rule, ping);
    this.isSecure = false;
    this.useTunnel = false;
    this.serverListUpdateInProgress = new AtomicBoolean(false);
    //重点： eureka和nacos更新本地缓存的方式在这里定义
    this.updateAction = new NamelessClass_1();
    this.serverListImpl = serverList;
    this.filter = filter;
    this.serverListUpdater = serverListUpdater;
    if (filter instanceof AbstractServerListFilter) {
        ((AbstractServerListFilter)filter).setLoadBalancerStats(this.getLoadBalancerStats());
    }

    this.restOfInit(clientConfig);
}
```



com.netflix.loadbalancer.BaseLoadBalancer#initWithConfig

```java
void initWithConfig(IClientConfig clientConfig, IRule rule, IPing ping) {
    this.config = clientConfig;
    String clientName = clientConfig.getClientName();
    this.name = clientName;
    int pingIntervalTime = Integer.parseInt(""
            + clientConfig.getProperty(
                    CommonClientConfigKey.NFLoadBalancerPingInterval,
                    Integer.parseInt("30")));
    int maxTotalPingTime = Integer.parseInt(""
            + clientConfig.getProperty(
                    CommonClientConfigKey.NFLoadBalancerMaxTotalPingTime,
                    Integer.parseInt("2")));

    setPingInterval(pingIntervalTime);
    setMaxTotalPingTime(maxTotalPingTime);
    //设置默认rule  ZoneAvoidanceRule 轮训
    setRule(rule);
    //设置ping  NIWSDiscoveryPing 从注册中心获取实例状态(up/down)
    setPing(ping);
    setLoadBalancerStats(new LoadBalancerStats(clientName));
    rule.setLoadBalancer(this);
    if (ping instanceof AbstractLoadBalancerPing) {
        ((AbstractLoadBalancerPing) ping).setLoadBalancer(this);
    }
   
}
```





DynamicServerListLoadBalancer.restOfInit

```java
void restOfInit(IClientConfig clientConfig) {
   
    boolean primeConnection = this.isEnablePrimingConnections();
    this.setEnablePrimingConnections(false);
   	//如果服务注册和发现是eureka，定时去eureka server拉取
    //有两种更新策略  
    //一种是定时去eureka server拉取  --> PollingServerListUpdater   默认
    //一种是基于Eureka服务事件通知的方式更新 --> EurekaNotificationServerListUpdater
    //如果服务注册和发现是nacos 则定时去nacos sever上拉取服务列表
    //PollingServerListUpdater#start
    this.enableAndInitLearnNewServersFeature();
    //拿当前DynamicServerListLoadBalancer对应的微服务列表 即根据微服务名 拿 ip+port列表
    //DiscoveryEnabledNIWSServerList：从Eureka client缓存的服务列表获取ServerList
    //StaticServerList：静态文件中直接配置的ServerList
    //ConfigurationBasedServerList：从配置文件参数listOfServers获取ServerList，多个用逗号分隔
    this.updateListOfServers();
    if (primeConnection && this.getPrimeConnections() != null) {
        this.getPrimeConnections().primeConnections(this.getReachableServers());
    }
	
    this.setEnablePrimingConnections(primeConnection);
    LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
}
```



enableAndInitLearnNewServersFeature

```java
public void enableAndInitLearnNewServersFeature() {
    LOGGER.info("Using serverListUpdater {}", serverListUpdater.getClass().getSimpleName());
  	//EurekaNotificationServerListUpdater#start
  	//PollingServerListUpdater#start  默认
    serverListUpdater.start(updateAction); //updateAction就是去注册中心获取当前服务名对应的服务列表
}
```



### ServerListUpdater

#### PollingServerListUpdater

PollingServerListUpdater#start

```java
public synchronized void start(final UpdateAction updateAction) {
    if (isActive.compareAndSet(false, true)) {
        final Runnable wrapperRunnable = new Runnable() {
            @Override
            public void run() {
                if (!isActive.get()) {
                    if (scheduledFuture != null) {
                        scheduledFuture.cancel(true);
                    }
                    return;
                }
                try {
                    updateAction.doUpdate();
                    lastUpdated = System.currentTimeMillis();
                } catch (Exception e) {
                    logger.warn("Failed one update cycle", e);
                }
            }
        };
				
      	//每30s去注册中心获取服务列表
        scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                wrapperRunnable,
                initialDelayMs,
                refreshIntervalMs,
                TimeUnit.MILLISECONDS
        );
    } else {
        logger.info("Already active, no-op");
    }
}
```



#### EurekaNotificationServerListUpdater

EurekaNotificationServerListUpdater#start

```java
public synchronized void start(final ServerListUpdater.UpdateAction updateAction) {
    if (this.isActive.compareAndSet(false, true)) {
      	//定义一个eureka事件监听器
        this.updateListener = new EurekaEventListener() {
            public void onEvent(EurekaEvent event) {
                //判断事件类型 感兴趣的事件
                if (event instanceof CacheRefreshedEvent) {
                    
                    if (!EurekaNotificationServerListUpdater.this.updateQueued.compareAndSet(false, true)) {
                        EurekaNotificationServerListUpdater.logger.info("an update action is already queued, returning as no-op");
                        return;
                    }

                    if (!EurekaNotificationServerListUpdater.this.refreshExecutor.isShutdown()) {
                        try {
                            EurekaNotificationServerListUpdater.this.refreshExecutor.submit(new Runnable() {
                                public void run() {
                                    try {
                                        updateAction.doUpdate();
                                        EurekaNotificationServerListUpdater.this.lastUpdated.set(System.currentTimeMillis());
                                    } catch (Exception var5) {
                                        EurekaNotificationServerListUpdater.logger.warn("Failed to update serverList", var5);
                                    } finally {
                                        EurekaNotificationServerListUpdater.this.updateQueued.set(false);
                                    }

                                }
                            });
                        } catch (Exception var3) {
                            EurekaNotificationServerListUpdater.logger.warn("Error submitting update task to executor, skipping one round of updates", var3);
                            EurekaNotificationServerListUpdater.this.updateQueued.set(false);
                        }
                    } else {
                        EurekaNotificationServerListUpdater.logger.debug("stopping EurekaNotificationServerListUpdater, as refreshExecutor has been shut down");
                        EurekaNotificationServerListUpdater.this.stop();
                    }
                }

            }
        };
        if (this.eurekaClient == null) {
            this.eurekaClient = (EurekaClient)this.eurekaClientProvider.get();
        }

        if (this.eurekaClient == null) {
            logger.error("Failed to register an updateListener to eureka client, eureka client is null");
            throw new IllegalStateException("Failed to start the updater, unable to register the update listener due to eureka client being null.");
        }
				//注册监听器   监听CacheRefreshedEvent事件
        this.eurekaClient.registerEventListener(this.updateListener);
    } else {
        logger.info("Update listener already registered, no-op");
    }

}
```





### ServerList

com.netflix.loadbalancer.DynamicServerListLoadBalancer#updateListOfServers

```java
public void updateListOfServers() {
    List<T> servers = new ArrayList();
    if (this.serverListImpl != null) {
        //serverList：DiscoveryEnabledNIWSServerList（eureka）
        servers = this.serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}", this.getIdentifier(), servers);
        if (this.filter != null) {
            servers = this.filter.getFilteredListOfServers((List)servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}", this.getIdentifier(), servers);
        }
    }

    this.updateAllServerList((List)servers);
}
```



#### DiscoveryEnabledNIWSServerList

- DiscoveryEnabledNIWSServerList：从Eureka client缓存的服务列表，来获取ServerList





com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList#getUpdatedListOfServers

```java
public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
    return this.obtainServersViaDiscovery();
}
```

com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList#obtainServersViaDiscovery

```java
private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
    List<DiscoveryEnabledServer> serverList = new ArrayList();
    if (this.eurekaClientProvider != null && this.eurekaClientProvider.get() != null) {
        EurekaClient eurekaClient = (EurekaClient)this.eurekaClientProvider.get();
        if (this.vipAddresses != null) {
            String[] var3 = this.vipAddresses.split(",");
            int var4 = var3.length;

            for(int var5 = 0; var5 < var4; ++var5) {
                String vipAddress = var3[var5];
                //去eureka server上拿实例信息
                List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, this.isSecure, this.targetRegion);
                Iterator var8 = listOfInstanceInfo.iterator();

                while(var8.hasNext()) {
                    InstanceInfo ii = (InstanceInfo)var8.next();
                    if (ii.getStatus().equals(InstanceStatus.UP)) {
                        if (this.shouldUseOverridePort) {
                            if (logger.isDebugEnabled()) {
                                logger.debug("Overriding port on client name: " + this.clientName + " to " + this.overridePort);
                            }

                            InstanceInfo copy = new InstanceInfo(ii);
                            if (this.isSecure) {
                                ii = (new Builder(copy)).setSecurePort(this.overridePort).build();
                            } else {
                                ii = (new Builder(copy)).setPort(this.overridePort).build();
                            }
                        }

                        DiscoveryEnabledServer des = this.createServer(ii, this.isSecure, this.shouldUseIpAddr);
                        serverList.add(des);
                    }
                }

                if (serverList.size() > 0 && this.prioritizeVipAddressBasedServers) {
                    break;
                }
            }
        }

        return serverList;
    } else {
        logger.warn("EurekaClient has not been initialized yet, returning an empty list");
        return new ArrayList();
    }
}
```



com.netflix.discovery.DiscoveryClient#getInstancesByVipAddress()

```java
@Singleton
public class DiscoveryClient implements EurekaClient {
	 public List<InstanceInfo> getInstancesByVipAddress(String vipAddress, boolean secure, @Nullable String region) {
        if (vipAddress == null) {
            throw new IllegalArgumentException("Supplied VIP Address cannot be null");
        } else {
            Applications applications;
            if (this.instanceRegionChecker.isLocalRegion(region)) {
                applications = (Applications)this.localRegionApps.get();
            } else {
                applications = (Applications)this.remoteRegionVsApps.get(region);
                if (null == applications) {
                    logger.debug("No applications are defined for region {}, so returning an empty instance list for vip address {}.", region, vipAddress);
                    return Collections.emptyList();
                }
            }

            return !secure ? applications.getInstancesByVirtualHostName(vipAddress) : applications.getInstancesBySecureVirtualHostName(vipAddress);
       
}
```







#### ConfigurationBasedServerList

ConfigurationBasedServerList：从配置文件参数listOfServers获取ServerList，多个用逗号分隔

```java
public class ConfigurationBasedServerList extends AbstractServerList<Server>  {

   private IClientConfig clientConfig;
      
   @Override
   public List<Server> getInitialListOfServers() {
       return getUpdatedListOfServers();
   }

   @Override
   public List<Server> getUpdatedListOfServers() {
        String listOfServers = clientConfig.get(CommonClientConfigKey.ListOfServers);
        return derive(listOfServers);
   }

   @Override
   public void initWithNiwsConfig(IClientConfig clientConfig) {
       this.clientConfig = clientConfig;
   }
   
   protected List<Server> derive(String value) {
       List<Server> list = Lists.newArrayList();
      if (!Strings.isNullOrEmpty(value)) {
         for (String s: value.split(",")) {
            list.add(new Server(s.trim()));
         }
      }
        return list;
   }
}
```



#### StaticServerList

StaticServerList：静态ServerList创建包含不变的服务实例

```java
public class StaticServerList<T extends Server> implements ServerList<T> {
    private final List<T> servers;

    public StaticServerList(T... servers) {
        this.servers = Arrays.asList(servers);
    }

    public List<T> getInitialListOfServers() {
        return this.servers;
    }

    public List<T> getUpdatedListOfServers() {
        return this.servers;
    }
}
```



负载均衡器通过ServerList来获取服务列表   由具体的注册中心中间件去实现ServerList

如果服务注册和发现是eureka，则注册事件监听器  监听eureka的服务列表变更事件 eureka每次更新完本地缓存 会发布事件监听
如果服务注册和发现是nacos 则定时去nacos sever上拉取服务列表

​        

### IPing

DynamicServerListLoadBalancer extends BaseLoadBalancer 



```java
public BaseLoadBalancer() {
    this.name = DEFAULT_NAME;
    this.ping = null;
    setRule(DEFAULT_RULE);
    setupPingTask();
    lbStats = new LoadBalancerStats(DEFAULT_NAME);
}
```



```java
void setupPingTask() {
    if (canSkipPing()) {
        return;
    }
    if (lbTimer != null) {
        lbTimer.cancel();
    }
    lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,
            true);
    //提交定时任务  每10s执行一次 
    lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
    //强制迅速ping一次
    forceQuickPing();
}
```

定时任务类

```java
class PingTask extends TimerTask {
    public void run() {
        try {
           new Pinger(pingStrategy).runPinger();
        } catch (Exception e) {
            logger.error("LoadBalancer [{}]: Error pinging", name, e);
        }
    }
}
```

com.netflix.loadbalancer.BaseLoadBalancer.Pinger#runPinger

```java
public void runPinger() throws Exception {
        if (!pingInProgress.compareAndSet(false, true)) { 
            return; // Ping in progress - nothing to do
        }
        
        // we are "in" - we get to Ping

        Server[] allServers = null;
        boolean[] results = null;

        Lock allLock = null;
        Lock upLock = null;

        try {
            /*
             * The readLock should be free unless an addServer operation is
             * going on...
             */
            allLock = allServerLock.readLock();
            allLock.lock();
            allServers = allServerList.toArray(new Server[allServerList.size()]);
            allLock.unlock();

            int numCandidates = allServers.length;
            //执行ping策略
            results = pingerStrategy.pingServers(ping, allServers);

            final List<Server> newUpList = new ArrayList<Server>();
            final List<Server> changedServers = new ArrayList<Server>();

            for (int i = 0; i < numCandidates; i++) {
                boolean isAlive = results[i];
                Server svr = allServers[i];
                boolean oldIsAlive = svr.isAlive();
								//设置服务状态
                svr.setAlive(isAlive);
								//比较原来和现在最新的状态
                if (oldIsAlive != isAlive) {
                    changedServers.add(svr);
                    logger.debug("LoadBalancer [{}]:  Server [{}] status changed to {}", 
                      name, svr.getId(), (isAlive ? "ALIVE" : "DEAD"));
                }

                if (isAlive) {
                    newUpList.add(svr);
                }
            }
            upLock = upServerLock.writeLock();
            upLock.lock();
            upServerList = newUpList;
            upLock.unlock();

            notifyServerStatusChangeListener(changedServers);
        } finally {
            pingInProgress.set(false);
        }
    }
}
```

执行具体的ping逻辑

我们可以实现IPing接口 自定义ping逻辑

```java
private static class SerialPingStrategy implements IPingStrategy {

    @Override
    public boolean[] pingServers(IPing ping, Server[] servers) {
        int numCandidates = servers.length;
        boolean[] results = new boolean[numCandidates];

        logger.debug("LoadBalancer:  PingTask executing [{}] servers configured", numCandidates);
		//遍历微服务对应的列表
        for (int i = 0; i < numCandidates; i++) {
            results[i] = false; /* Default answer is DEAD. */
            try {
                
                if (ping != null) {
                    //执行ping 判断是否存活
                    results[i] = ping.isAlive(servers[i]);
                }
            } catch (Exception e) {
                logger.error("Exception while pinging Server: '{}'", servers[i], e);
            }
        }
        return results;
    }
}
```

IPing实现类：

```
PingConstant：不去ping 返回常量
NoOpPing：不去ping 直接返回true
NIWSDiscoveryPing：根据eureka client缓存的服务状态 UP DOWN   如果和eureka整合 默认使用此实现 
DummyPing：不去ping 直接返回true
PingUrl：去ping指定url

```

#### NIWSDiscoveryPing

```java
public class NIWSDiscoveryPing extends AbstractLoadBalancerPing {
    BaseLoadBalancer lb = null;

    public NIWSDiscoveryPing() {
    }

    public BaseLoadBalancer getLb() {
        return this.lb;
    }

    public void setLb(BaseLoadBalancer lb) {
        this.lb = lb;
    }

    public boolean isAlive(Server server) {
        boolean isAlive = true;
        if (server != null && server instanceof DiscoveryEnabledServer) {
            DiscoveryEnabledServer dServer = (DiscoveryEnabledServer)server;
          	//获取eureka client本地缓存的服务实例信息
            InstanceInfo instanceInfo = dServer.getInstanceInfo();
            if (instanceInfo != null) {
                InstanceInfo.InstanceStatus status = instanceInfo.getStatus();
                if (status != null) {
                    isAlive = status.equals(InstanceStatus.UP);
                }
            }
        }

        return isAlive;
    }

    public void initWithNiwsConfig(IClientConfig clientConfig) {
    }
}
```



#### PingUrl

```java
public boolean isAlive(Server server) {
    String urlStr = "";
    if (this.isSecure) {
        urlStr = "https://";
    } else {
        urlStr = "http://";
    }

    urlStr = urlStr + server.getId();
    urlStr = urlStr + this.getPingAppendString();
    boolean isAlive = false;
    HttpClient httpClient = new DefaultHttpClient();
    HttpUriRequest getRequest = new HttpGet(urlStr);
    String content = null;

    try {
        //发请求
        HttpResponse response = httpClient.execute(getRequest);
        content = EntityUtils.toString(response.getEntity());
      	//响应码为200
        isAlive = response.getStatusLine().getStatusCode() == 200;
        if (this.getExpectedContent() != null) {
            LOGGER.debug("content:" + content);
            if (content == null) {
                isAlive = false;
            } else if (content.equals(this.getExpectedContent())) {
                isAlive = true;
            } else {
                isAlive = false;
            }
        }
    } catch (IOException var11) {
        var11.printStackTrace();
    } finally {
        getRequest.abort();
    }

    return isAlive;
}
```







```java

class EurekaRibbonClientConfiguration{
    @Bean
    @ConditionalOnMissingBean
    public IPing ribbonPing(IClientConfig config) {
       if (this.propertiesFactory.isSet(IPing.class, serviceId)) {
          return this.propertiesFactory.get(IPing.class, config, serviceId);
       }
       NIWSDiscoveryPing ping = new NIWSDiscoveryPing();
       ping.initWithNiwsConfig(config);
       return ping;
    }

    @Bean
    @ConditionalOnMissingBean
    public ServerList<?> ribbonServerList(IClientConfig config,
          Provider<EurekaClient> eurekaClientProvider) {
       if (this.propertiesFactory.isSet(ServerList.class, serviceId)) {
          return this.propertiesFactory.get(ServerList.class, config, serviceId);
       }
       DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(
             config, eurekaClientProvider);
       DomainExtractingServerList serverList = new DomainExtractingServerList(
             discoveryServerList, config, this.approximateZoneFromHostname);
       return serverList;
    }
}
```



### 总结getLoadBalancer()

根据不同的微服务名（服务提供者）构建spring容器 配置隔离

构建负载均衡器

读取当前eureka client的配置缓存

定时刷新ribbon 缓存配置（30s）

定时ping微服务提供者（10s）



## Server



RibbonLoadBalancerClient#execute()

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
    //获取服务id对应的负载均衡器
    ILoadBalancer loadBalancer = this.getLoadBalancer(serviceId);
  	//根据IRule从ServerList中选择一个实例
    Server server = this.getServer(loadBalancer, hint);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    } else {
        RibbonServer ribbonServer = new RibbonServer(serviceId, server, this.isSecure(server, serviceId), this.serverIntrospector(serviceId).getMetadata(server));
        return this.execute(serviceId, (ServiceInstance)ribbonServer, (LoadBalancerRequest)request);
    }
}
```



RibbonLoadBalancerClient#getServer()

拿到该微服务名对应的服务列表 做负载均衡

```java
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    return loadBalancer == null ? null : loadBalancer.chooseServer(hint != null ? hint : "default");
}
```

com.netflix.loadbalancer.BaseLoadBalancer#chooseServer

```java
public Server chooseServer(Object key) {
    if (this.counter == null) {
        this.counter = this.createCounter();
    }

    this.counter.increment();
    if (this.rule == null) {
        return null;
    } else {
        try {
            return this.rule.choose(key);
        } catch (Exception var3) {
            logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", new Object[]{this.name, key, var3});
            return null;
        }
    }
}
```



### IRule

根据负载均衡算法选择一个server

#### RoundRobinRule：轮询

```java
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        log.warn("no load balancer");
        return null;
    }

    Server server = null;
    int count = 0;
    while (server == null && count++ < 10) {
      	//获取可达的ServerList
        List<Server> reachableServers = lb.getReachableServers();
        List<Server> allServers = lb.getAllServers();
        int upCount = reachableServers.size();
        int serverCount = allServers.size();

        if ((upCount == 0) || (serverCount == 0)) {
            log.warn("No up servers available from load balancer: " + lb);
            return null;
        }
				//自增
        int nextServerIndex = incrementAndGetModulo(serverCount);
        server = allServers.get(nextServerIndex);

        if (server == null) {
            /* Transient. */
            Thread.yield();
            continue;
        }

        if (server.isAlive() && (server.isReadyToServe())) {
            return (server);
        }
        server = null;
    }

    if (count >= 10) {
        log.warn("No available alive servers after 10 tries from load balancer: "
                + lb);
    }
    return server;
}
```



#### RandomRule：随机

```java
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        return null;
    }
    Server server = null;

    while (server == null) {
        if (Thread.interrupted()) {
            return null;
        }
     		//获取可达的ServerList
        List<Server> upList = lb.getReachableServers();
        List<Server> allList = lb.getAllServers();

        int serverCount = allList.size();
        if (serverCount == 0) {
            /*
             * No servers. End regardless of pass, because subsequent passes
             * only get more restrictive.
             */
            return null;
        }
				
      	//生成serverCount以内的随机数
        int index = chooseRandomInt(serverCount);
        server = upList.get(index);

        if (server == null) {
            
            Thread.yield();
            continue;
        }
				
        if (server.isAlive()) {
            return (server);
        }

        server = null;
        Thread.yield();
    }

    return server;

}
```



#### AvailabilityFilteringRule

AvailabilityFilteringRule：先随机获取一个，然后过滤掉由于多次访问故障而处于断路器状态的服务，或者并发的连接数量超过阈值的服务，然后对剩余的服务列表按照轮询策略进行访问；

```java
public Server choose(Object key) {
    int count = 0;
  	//先随机获取一个Server
    Server server = roundRobinRule.choose(key);
  	//最多重试10次
    while (count++ <= 10) {
      	//AvailabilityPredicate#apply：判断该Server是否符合条件
        if (predicate.apply(new PredicateKey(server))) {
            return server;
        }
        server = roundRobinRule.choose(key);
    }
    return super.choose(key);
}
```



AvailabilityPredicate#apply

```java
public boolean apply(@Nullable PredicateKey input) {
    LoadBalancerStats stats = getLBStats();
    if (stats == null) {
        return true;
    }
  	//是否该跳过该Server
    return !shouldSkipServer(stats.getSingleServerStat(input.getServer()));
}


private boolean shouldSkipServer(ServerStats stats) {  
  	//如果多次访问故障而处于断路器状态 
    if ((CIRCUIT_BREAKER_FILTERING.get() && stats.isCircuitBreakerTripped()) 
       			//或者并发的连接数量超过阈值的服务
            || stats.getActiveRequestsCount() >= activeConnectionsLimit.get()) {
        return true;
    }
    return false;
}
```



#### WeightedResponseTimeRule

- WeightedResponseTimeRule：根据平均响应时间计算所有服务的权重，响应时间越快的服务权重越大被选中的概率越大。刚启动时如果统计信息不足，则使用RoundRobinRule（轮询）策略，等统计信息足够，会切换到WeightedResponseTimeRule；




 

```java
public Server choose(ILoadBalancer lb, Object key) {
    if (lb == null) {
        return null;
    }
    Server server = null;

    while (server == null) {
        // 存储权重的集合，该List中每个权重所处的位置对应了负载均衡器维护实例清单中所有实例在清单中的位置
        List<Double> currentWeights = accumulatedWeights;
        if (Thread.interrupted()) {
            return null;
        }
        List<Server> allList = lb.getAllServers();

        int serverCount = allList.size();

        if (serverCount == 0) {
            return null;
        }

        int serverIndex = 0;

       
      	//currentWeights保存的是每个Server对应的权重 下标
      	// 获取最后一个实例的权重（保证如果currentWeights里有元素，那么一定会选出一个Server）
        double maxTotalWeight = currentWeights.size() == 0 ? 0 : currentWeights.get(currentWeights.size() - 1); 
   
        // 如果最后一个实例的权重小于0.001，则采用父类实现的现象轮询的策略
        if (maxTotalWeight < 0.001d || serverCount != currentWeights.size()) {
            server =  super.choose(getLoadBalancer(), key);
            if(server == null) {
                return server;
            }
        } else {
            // 产生一个[0,maxTotalWeight]的随机值
            double randomWeight = random.nextDouble() * maxTotalWeight;
            // pick the server index based on the randomIndex
            int n = 0;
          //遍历权重列表，比较权重值与随机数的大小，如果权重值大于随机数，就拿当前权重列表的索引值去服务实例表获取具体的实例。
            for (Double d : currentWeights) {
              	//只要大于等于最后一个Server的权重就行
                if (d >= randomWeight) {
                    serverIndex = n;
                    break;
                } else {
                    n++;
                }
            }

            server = allList.get(serverIndex);
        }

        if (server == null) {
            /* Transient. */
            Thread.yield();
            continue;
        }

        if (server.isAlive()) {
            return (server);
        }

        // Next.
        server = null;
    }
    return server;
}
```



WeightedResponseTimeRule创建会调用initialize初始化方法

```java
void initialize(ILoadBalancer lb) {        
    if (serverWeightTimer != null) {
        serverWeightTimer.cancel();
    }
    serverWeightTimer = new Timer("NFLoadBalancer-serverWeightTimer-"
            + name, true);
    
  	//启动一个定时任务，用来为每个服务实例计算权重，默认30秒执行一次，调用DynamicServerWeighTask方法，向下找
    serverWeightTimer.schedule(new DynamicServerWeightTask(), 0,
            serverWeightTaskTimerInterval);
    // do a initial run
    ServerWeight sw = new ServerWeight();
    sw.maintainWeights();

    Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
        public void run() {
            logger
                    .info("Stopping NFLoadBalancer-serverWeightTimer-"
                            + name);
            serverWeightTimer.cancel();
        }
    }));
}
```



```java
class DynamicServerWeightTask extends TimerTask {
    public void run() {
        ServerWeight serverWeight = new ServerWeight();
        try {
            serverWeight.maintainWeights();
        } catch (Exception e) {
            logger.error("Error running DynamicServerWeightTask for {}", name, e);
        }
    }
}
```



```java
class ServerWeight {
		
  	/*该函数主要分为两个步骤
      1 根据LoadBalancerStats中记录的每个实例的统计信息，累计所有实例的平均响应时间，
      得到总的平均响应时间totalResponseTime，该值用于后面的计算。
      2 为负载均衡器中维护的实例清单逐个计算权重（从第一个开始），计算规则为：
      weightSoFar+totalResponseTime-实例平均相应时间，其中weightSoFar初始化为0，并且
      每计算好一个权重需要累加到weightSoFar上供下一次计算使用。
      示例：4个实例A、B、C、D，它们的平均响应时间为10,40,80,100，所以总的响应时间为
      230，每个实例的权重为总响应时间与实例自身的平均响应时间的差的累积所得，所以实例A
      B，C，D的权重分别为：
      A：230-10=220
      B：220+230-40=410
      C：410+230-80=560
      D：560+230-100=690
      需要注意的是，这里的权重值只是表示各实例权重区间的上限，并非某个实例的优先级，所以不
      是数值越大被选中的概率就越大。而是由实例的权重区间来决定选中的概率和优先级。
      A：[0,220]
      B：(220,410]
      C：(410,560]
      D：(560,690)
      实际上每个区间的宽度就是：总的平均响应时间-实例的平均响应时间，所以实例的平均响应时间越短
      ，权重区间的宽度越大，而权重区间宽度越大被选中的概率就越大。
    */
    public void maintainWeights() {
        ILoadBalancer lb = getLoadBalancer();
        if (lb == null) {
            return;
        }
        
        if (!serverWeightAssignmentInProgress.compareAndSet(false,  true))  {
            return; 
        }
        
        try {
        
            AbstractLoadBalancer nlb = (AbstractLoadBalancer) lb;
            LoadBalancerStats stats = nlb.getLoadBalancerStats();
            if (stats == null) {
                // no statistics, nothing to do
                return;
            }
            double totalResponseTime = 0;
            // find maximal 95% response time
            for (Server server : nlb.getAllServers()) {
                //获取Server的状态信息
                ServerStats ss = stats.getSingleServerStat(server);
              	//获取所有server的平均响应时间并累加
                totalResponseTime += ss.getResponseTimeAvg();
            }
            // weight for each server is (sum of responseTime of all servers - responseTime)
            // so that the longer the response time, the less the weight and the less likely to be chosen
            Double weightSoFar = 0.0;
            
            //weightSoFar+totalResponseTime-实例平均相应时间
            List<Double> finalWeights = new ArrayList<Double>();
            for (Server server : nlb.getAllServers()) {
                ServerStats ss = stats.getSingleServerStat(server);
                double weight = totalResponseTime - ss.getResponseTimeAvg();
                weightSoFar += weight;
                finalWeights.add(weightSoFar);   
            }
            setWeights(finalWeights);
        } catch (Exception e) {
            logger.error("Error calculating server weights", e);
        } finally {
            serverWeightAssignmentInProgress.set(false);
        }

    }
}

void setWeights(List<Double> weights) {
        this.accumulatedWeights = weights;
}
```



#### RetryRule

- RetryRule：先按照RoundRobinRule（轮询）策略获取服务，如果获取服务失败则在指定时间内进行重试，获取可用的服务；

```java
//具备重试机制的实例选择功能
public class RetryRule extends AbstractLoadBalancerRule {
    //默认使用RoundRobinRule实例
    IRule subRule = new RoundRobinRule();
    //阈值为500ms
    long maxRetryMillis = 500;
 
    public RetryRule() {
    }
 
    public RetryRule(IRule subRule) {
        this.subRule = (subRule != null) ? subRule : new RoundRobinRule();
    }
 
    public RetryRule(IRule subRule, long maxRetryMillis) {
        this.subRule = (subRule != null) ? subRule : new RoundRobinRule();
        this.maxRetryMillis = (maxRetryMillis > 0) ? maxRetryMillis : 500;
    }
 
    public void setRule(IRule subRule) {
        this.subRule = (subRule != null) ? subRule : new RoundRobinRule();
    }
 
    public IRule getRule() {
        return subRule;
    }
 
    public void setMaxRetryMillis(long maxRetryMillis) {
        if (maxRetryMillis > 0) {
            this.maxRetryMillis = maxRetryMillis;
        } else {
            this.maxRetryMillis = 500;
        }
    }
 
    public long getMaxRetryMillis() {
        return maxRetryMillis;
    }
 
    
    
    @Override
    public void setLoadBalancer(ILoadBalancer lb) {        
        super.setLoadBalancer(lb);
        subRule.setLoadBalancer(lb);
    }
 
 
    public Server choose(ILoadBalancer lb, Object key) {
        long requestTime = System.currentTimeMillis();
        long deadline = requestTime + maxRetryMillis;
 
        Server answer = null;
 
        answer = subRule.choose(key);
 
        if (((answer == null) || (!answer.isAlive()))
                && (System.currentTimeMillis() < deadline)) {
 
            InterruptTask task = new InterruptTask(deadline
                    - System.currentTimeMillis());
            //反复重试
            while (!Thread.interrupted()) {
                //选择实例
                answer = subRule.choose(key);
                //500ms内没选择到就返回null
                if (((answer == null) || (!answer.isAlive()))
                        && (System.currentTimeMillis() < deadline)) {
                    /* pause and retry hoping it's transient */
                    Thread.yield();
                } 
                else //若能选择到实例，就返回
                {
                    break;
                }
            }
 
            task.cancel();
        }
 
        if ((answer == null) || (!answer.isAlive())) {
            return null;
        } else {
            return answer;
        }
    }
 
    @Override
    public Server choose(Object key) {
        return choose(getLoadBalancer(), key);
    }
 
    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
    }
}
```



#### BestAvailableRule

- BestAvailableRule：会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务；

```java
public Server choose(Object key) {
    if (loadBalancerStats == null) {
        return super.choose(key);
    }
    List<Server> serverList = getLoadBalancer().getAllServers();
    int minimalConcurrentConnections = Integer.MAX_VALUE;
    long currentTime = System.currentTimeMillis();
    Server chosen = null;
    for (Server server: serverList) {
        ServerStats serverStats = loadBalancerStats.getSingleServerStat(server);
      	//滤掉由于多次访问故障而处于断路器跳闸状态
        if (!serverStats.isCircuitBreakerTripped(currentTime)) {
            int concurrentConnections = serverStats.getActiveRequestsCount(currentTime);
          	//找连接数最小的Server
            if (concurrentConnections < minimalConcurrentConnections) {
                minimalConcurrentConnections = concurrentConnections;
                chosen = server;
            }
        }
    }
    if (chosen == null) {
        return super.choose(key);
    } else {
        return chosen;
    }
}
```



#### ZoneAvoidanceRule

- ZoneAvoidanceRule：复合判断Server所在区域的性能和Server的可用性选择服务器，在没有Zone的情况下是类似轮询的算法；















# RibbonLoadBalancerClient



org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#execute

```java
public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
    Server server = null;
    if (serviceInstance instanceof RibbonLoadBalancerClient.RibbonServer) {
        server = ((RibbonLoadBalancerClient.RibbonServer)serviceInstance).getServer();
    }

    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    } else {
        RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);
        RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

        try {
            //执行请求
            T returnVal = request.apply(serviceInstance);
            statsRecorder.recordStats(returnVal);
            return returnVal;
        } catch (IOException var8) {
            statsRecorder.recordStats(var8);
            throw var8;
        } catch (Exception var9) {
            statsRecorder.recordStats(var9);
            ReflectionUtils.rethrowRuntimeException(var9);
            return null;
        }
    }
}
```



# 配置隔离

@RibbonClients(

​	@RibbonClient(name = "user-service",configuration = UserServiceRibbonConfiguration.class)

)



注：UserServiceRibbonConfiguration不能放在能被spring扫描到的包下





# 服务列表缓存

## ribbon缓存

ribbon默认30s刷新一次服务列表，服务列表来源是erueka client缓存在本地的服务端列表

构造DynamicServerListLoadBalancer对象时

```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping, ServerList<T> serverList, ServerListFilter<T> filter, ServerListUpdater serverListUpdater) {
				//完成初始化
        this.restOfInit(clientConfig);
    }
```



```java
void restOfInit(IClientConfig clientConfig) {
  	//启动定时更新服务列表定时任务
    this.enableAndInitLearnNewServersFeature();
   
}
```



```java
public void enableAndInitLearnNewServersFeature() {
        this.serverListUpdater.start(this.updateAction);
    }

```



```java
public class PollingServerListUpdater implements ServerListUpdater {
		
   public PollingServerListUpdater(IClientConfig clientConfig) {
     	  //getRefreshIntervalMs：获取刷新服务列表的间隔
        this(LISTOFSERVERS_CACHE_UPDATE_DELAY, getRefreshIntervalMs(clientConfig));
   }
  private static long getRefreshIntervalMs(IClientConfig clientConfig) {
    		//默认30s 可通过配置项ribbon.ServerListRefreshInterval: 30000指定
        return (long)(Integer)clientConfig.get(CommonClientConfigKey.ServerListRefreshInterval, LISTOFSERVERS_CACHE_REPEAT_INTERVAL);
  }
  
   public synchronized void start(final UpdateAction updateAction) {
        if (this.isActive.compareAndSet(false, true)) {
            Runnable wrapperRunnable = new Runnable() {
                public void run() {
                    if (!PollingServerListUpdater.this.isActive.get()) {
                        if (PollingServerListUpdater.this.scheduledFuture != null) {
                            PollingServerListUpdater.this.scheduledFuture.cancel(true);
                        }

                    } else {
                        try {
                            updateAction.doUpdate();
                            PollingServerListUpdater.this.lastUpdated = System.currentTimeMillis();
                        } catch (Exception var2) {
                            PollingServerListUpdater.logger.warn("Failed one update cycle", var2);
                        }

                    }
                }
            };
          	//启动定时任务 并指定多久执行一次
            this.scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(wrapperRunnable, this.initialDelayMs, this.refreshIntervalMs, TimeUnit.MILLISECONDS);
        } else {
            logger.info("Already active, no-op");
        }

    }
}
```



```java
class NamelessClass_1 implements UpdateAction {
 
    public void doUpdate() {
        DynamicServerListLoadBalancer.this.updateListOfServers();
    }
}
```



```java
  public void updateListOfServers() {
        List<T> servers = new ArrayList();
        if (this.serverListImpl != null) {
          	//获取服务列表
            servers = this.serverListImpl.getUpdatedListOfServers();
            LOGGER.debug("List of Servers for {} obtained from Discovery client: {}", this.getIdentifier(), servers);
            if (this.filter != null) {
                servers = this.filter.getFilteredListOfServers((List)servers);
                LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}", this.getIdentifier(), servers);
            }
        }

        this.updateAllServerList((List)servers);
    }
```



```java
public class DiscoveryEnabledNIWSServerList extends AbstractServerList<DiscoveryEnabledServer> {
	public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
        return this.obtainServersViaDiscovery();
  }
   private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
        List<DiscoveryEnabledServer> serverList = new ArrayList();
        if (this.eurekaClientProvider != null && this.eurekaClientProvider.get() != null) {
            EurekaClient eurekaClient = (EurekaClient)this.eurekaClientProvider.get();
            if (this.vipAddresses != null) {
                String[] var3 = this.vipAddresses.split(",");
                int var4 = var3.length;

                for(int var5 = 0; var5 < var4; ++var5) {
                    String vipAddress = var3[var5];
                  	//并没有去注册中心获取服务列表 而是从eureka client的缓存获取服务列表
                    List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, this.isSecure, this.targetRegion);
                    Iterator var8 = listOfInstanceInfo.iterator();

                    while(var8.hasNext()) {
                        InstanceInfo ii = (InstanceInfo)var8.next();
                        if (ii.getStatus().equals(InstanceStatus.UP)) {
                            if (this.shouldUseOverridePort) {
                                if (logger.isDebugEnabled()) {
                                    logger.debug("Overriding port on client name: " + this.clientName + " to " + this.overridePort);
                                }

                                InstanceInfo copy = new InstanceInfo(ii);
                                if (this.isSecure) {
                                    ii = (new Builder(copy)).setSecurePort(this.overridePort).build();
                                } else {
                                    ii = (new Builder(copy)).setPort(this.overridePort).build();
                                }
                            }

                            DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, this.isSecure, this.shouldUseIpAddr);
                            des.setZone(DiscoveryClient.getZone(ii));
                            serverList.add(des);
                        }
                    }

                    if (serverList.size() > 0 && this.prioritizeVipAddressBasedServers) {
                        break;
                    }
                }
            }

            return serverList;
        } else {
            logger.warn("EurekaClient has not been initialized yet, returning an empty list");
            return new ArrayList();
        }
    }
}

```



```java
@Singleton
public class DiscoveryClient implements EurekaClient {
	 public List<InstanceInfo> getInstancesByVipAddress(String vipAddress, boolean secure, @Nullable String region) {
        if (vipAddress == null) {
            throw new IllegalArgumentException("Supplied VIP Address cannot be null");
        } else {
            Applications applications;
            if (this.instanceRegionChecker.isLocalRegion(region)) {
                applications = (Applications)this.localRegionApps.get();
            } else {
                applications = (Applications)this.remoteRegionVsApps.get(region);
                if (null == applications) {
                    logger.debug("No applications are defined for region {}, so returning an empty instance list for vip address {}.", region, vipAddress);
                    return Collections.emptyList();
                }
            }

            return !secure ? applications.getInstancesByVirtualHostName(vipAddress) : applications.getInstancesBySecureVirtualHostName(vipAddress);
       
}
```





# 重试机制















