Eureka

基于Hoxton.SR1版本

```java
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
    
<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
</dependencies>
```



eureka-server

- 服务注册

- 服务发现

- 服务下架   client--主动   服务下架和服务剔除最终调用的是同一个方法  server集群间会进行同步

- 服务剔除   client--被动（eureka server维持一个定时任务扫描服务注册表，判断服务心跳续约是否过期） eureka server判断心跳过期  server集群间不进行同步

- 心跳续约

- 自我保护机制

- 服务注册表

- 集群同步 （PeerAwareInstanceRegistryImpl#renew负责所有eureka server的集群同步）



eureka server也是一个mvc架构，controller层使用jersey实现    

注册中心需要与客户端进行交互 接受请求 返回响应 -- mvc架构(不是springmvc（基于servlet），而是通过jersey（基于过滤器）实现的)



# Eureka server自动配置原理

springboot项目使用eureka注册中心

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }

}
```



查看@EnableEurekaServer的类定义

```java
@Import({EurekaServerMarkerConfiguration.class})
public @interface EnableEurekaServer {
}
```

@EnableEurekaServer给容器里导入了一个bean == EurekaServerMarkerConfiguration

```java
public class EurekaServerMarkerConfiguration {

   @Bean
   public Marker eurekaServerMarkerBean() {
      return new Marker();
   }

   class Marker {

   }

}
```

EurekaServerMarkerConfiguration给容器里导入了一个Marker(标记) bean  这个Marker的作用就是一个标记 标记用户有没有使用@EnableEurekaServer这个注解



在spring-cloud-netflix-eureka-server的META-INF/spring.factories中 导入了EurekaServerAutoConfiguration自动配置类

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```







![eureka server自动配置原理](images/eureka server自动配置原理.png)



```java
@Configuration(proxyBeanMethods = false)
//导入配置类
@Import(EurekaServerInitializerConfiguration.class)

@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)  //当前配置类是否生效
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
      InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {
    //省略部分代码
 
  
  	//server端后台管理界面接口
    @Bean
    @ConditionalOnProperty(
        prefix = "eureka.dashboard",
        name = {"enabled"},
        matchIfMissing = true
    )
    public EurekaController eurekaController() {
        return new EurekaController(this.applicationInfoManager);
    }

  
    
    //jersey框架基于过滤器 注册过滤器到容器中
    @Bean
	public FilterRegistrationBean<?> jerseyFilterRegistration(
			javax.ws.rs.core.Application eurekaJerseyApp) {
		FilterRegistrationBean<Filter> bean = new FilterRegistrationBean<Filter>();
		bean.setFilter(new ServletContainer(eurekaJerseyApp));
		bean.setOrder(Ordered.LOWEST_PRECEDENCE);
		bean.setUrlPatterns(
				Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));

		return bean;
	}
    
  @Bean
	public javax.ws.rs.core.Application jerseyApplication(Environment environment,
			ResourceLoader resourceLoader) {

		ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(
				false, environment);
		/**
		 *  includeFilters：允许过滤的条件 指定需要扫描的内容（此处内容是Component注解）
		 *
		 *  代表只有被includeFilters内匹配的注解才可以被扫描解析
		 */
		provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
		provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));
	
		Set<Class<?>> classes = new HashSet<>();
		for (String basePackage : EUREKA_PACKAGES) {
			Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
			for (BeanDefinition bd : beans) {
				Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
						resourceLoader.getClassLoader());
				classes.add(cls);
			}
		}
		Map<String, Object> propsAndFeatures = new HashMap<>();
		propsAndFeatures.put(
				ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
				EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

		DefaultResourceConfig rc = new DefaultResourceConfig(classes);
		rc.setPropertiesAndFeatures(propsAndFeatures);

		return rc;
	}

    

```



导入了配置类@Import(EurekaServerInitializerConfiguration.class)

```java
public class EurekaServerInitializerConfiguration implements ServletContextAware, SmartLifecycle, Ordered {
  	
  	//实现LifeCycle的生命周期方法
  	public void start() {
      new Thread(() -> {
        try {
          //初始化
          eurekaServerBootstrap.contextInitialized(EurekaServerInitializerConfiguration.this.servletContext);
          log.info("Started Eureka Server");

          publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
          EurekaServerInitializerConfiguration.this.running = true;
          publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
        }
        catch (Exception ex) {
          // Help!
          log.error("Could not initialize Eureka servlet context", ex);
        }
      }).start();
	}
  
  
}
```



```java
public class EurekaServerBootstrap {
	public void contextInitialized(ServletContext context) {
		try {
			//打印了一行日志
			initEurekaEnvironment();
			//
			initEurekaServerContext();

			context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
		}
		catch (Throwable e) {
			log.error("Cannot bootstrap eureka server :", e);
			throw new RuntimeException("Cannot bootstrap eureka server :", e);
		}
	}
	
  protected void initEurekaServerContext() throws Exception {
		JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);
		XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), XStream.PRIORITY_VERY_HIGH);

		if (isAws(this.applicationInfoManager.getInfo())) {
			this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig, this.eurekaClientConfig, this.registry,
					this.applicationInfoManager);
			this.awsBinder.start();
		}

		EurekaServerContextHolder.initialize(this.serverContext);

		log.info("Initialized server context");

		//从其他eureka server节点同步信息
		int registryCount = this.registry.syncUp();
      
    //启动服务剔除任务类
		this.registry.openForTraffic(this.applicationInfoManager, registryCount);

		// Register all monitoring statistics.
		EurekaMonitors.registerAllStats();
	}
	
}
```





PeerAwareInstanceRegistryImpl#openForTraffic

```java
    public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
        this.expectedNumberOfRenewsPerMin = count * 2;
        this.numberOfRenewsPerMinThreshold = (int)((double)this.expectedNumberOfRenewsPerMin * this.serverConfig.getRenewalPercentThreshold());
        logger.info("Got {} instances from neighboring DS node", count);
        logger.info("Renew threshold is: {}", this.numberOfRenewsPerMinThreshold);
        this.startupTime = System.currentTimeMillis();
        if (count > 0) {
            this.peerInstancesTransferEmptyOnStartup = false;
        }

        DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
        boolean isAws = Name.Amazon == selfName;
        if (isAws && this.serverConfig.shouldPrimeAwsReplicaConnections()) {
            logger.info("Priming AWS connections for all replicas..");
            this.primeAwsReplicas(applicationInfoManager);
        }

        logger.info("Changing status to UP");
        applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
        super.postInit();
    }
```



AbstractInstanceRegistry#postInit

```java
  protected void postInit() {
     this.renewsLastMin.start();
     if (this.evictionTaskRef.get() != null) {
       ((EvictionTask)this.evictionTaskRef.get()).cancel();
     }

     this.evictionTaskRef.set(new EvictionTask());
		 //启动服务剔除定时任务类
     this.evictionTimer.schedule((TimerTask)this.evictionTaskRef.get(), this.serverConfig.getEvictionIntervalTimerInMs(), this.serverConfig.getEvictionIntervalTimerInMs());
  }
```













# 整体架构

**InstanceRegistry  extends PeerAwareInstanceRegistryImpl extends AbstractInstanceRegistry**  责任链模式

## InstanceRegistry

spring cloud提供的 用来整合netflix eureka server，主要是发布事件

```java
EurekaInstanceRegisteredEvent：注册事件
EurekaInstanceRenewedEvent：心跳续约时间
EurekaInstanceCanceledEvent：服务下架和服务剔除事件

```

我们可以在spring应用中编写监听器 监听这些事件

## PeerAwareInstanceRegistryImpl 

netflix提供的 用来的做集群同步操作的



## AbstractInstanceRegistry

netflix提供的 真正用来实现服务注册 心跳续约 的业务类



# 服务注册

**入口在eureke-core包下的 com.netflix.eureka.resources.ApplicationResource#addInstance**     

```java
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info, @HeaderParam("x-netflix-discovery-replication") String isReplication) {
 				//省略部分代码
        this.registry.register(info, "true".equals(isReplication));
        return Response.status(204).build();
    }
}
```

Resource相当于springmvc的Controller  这里我们只关注业务层



**InstanceRegistry  extends PeerAwareInstanceRegistryImpl extends AbstractInstanceRegistry**

org.springframework.cloud.netflix.eureka.server.InstanceRegistry.register()：这个类是springcloud写的

```java
@Override
public void register(InstanceInfo info, int leaseDuration, boolean isReplication) {
    //info：服务注册信息
   //发布监听
   handleRegistration(info, leaseDuration, isReplication);
   //调用父类的监听方法(此处才是netflix的代码)
   super.register(info, leaseDuration, isReplication);
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#register

```java
public void register(final InstanceInfo info, final boolean isReplication) {
    //默认心跳续约过期时间
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        //读取客户端传递归来的心跳续约时间
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    //服务注册
    super.register(info, leaseDuration, isReplication);
    //服务注册成功之后进行集群信息同步
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```



com.netflix.eureka.registry.AbstractInstanceRegistry#register()：真正的服务注册实现

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    //registrant：服务注册信息 eureka client注册时传递的信息
    try {
        read.lock();
        //private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry;
        //外层map的key是服务名  内层map的key是实例id  一个服务可以有多个实例
        //Lease：租债器
        //拿到微服务组
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
        REGISTER.increment(isReplication);
        //判断微服务组是否存在  
        //客户端对于注册请求有超时和重试机制 如果请求超时 将会进行重试 
        if (gMap == null) {
            //如果为空就创建一个新的
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }
        //拿到微服务组中的某个已经存在的实例信息
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        //existingLease.getHolder()：租债器里面的实例对象 
        // Retain the last dirty timestamp without overwriting it, if there is already a lease
        //不为空说明冲突了（同一个实例发送了多个注册请求或者客户端重连）
        if (existingLease != null && (existingLease.getHolder() != null)) {
            //获取此实例最后一次操作时间
            Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
            //客户端传递过来的时间戳
            Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

            // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
            // InstanceInfo instead of the server local copy.
            //如果冲突了 哪个时间戳比较新就用哪个   看谁后发的注册请求  同一个实例 由于有超时和重试机制 后发的注册请求会覆盖先发的
            //如果两个时间戳相等 使用客户端新传递过来的实例信息
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                        " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = existingLease.getHolder();
            }
        } else {
            //没有冲突则重新计算触发自我保护阈值
            // The lease does not exist and hence it is a new registration
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                   	//预计需要接受心跳续约的客户端数量 客户端注册时加1 服务下架时减1
                    this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                    //重新计算触发自我保护阈值
                    updateRenewsPerMinThreshold();
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
        //创建一个租债器
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        if (existingLease != null) {
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
        //将包含实例信息的租债器保存到微服务组中
        gMap.put(registrant.getId(), lease);
        synchronized (recentRegisteredQueue) {
            //将当前实例的注册事件放到最新注册的queue里面，方便查询最近的注册事件
            recentRegisteredQueue.add(new Pair<Long, String>(
                    System.currentTimeMillis(),
                    registrant.getAppName() + "(" + registrant.getId() + ")"));
        }
      
        InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }

        InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);
        if (InstanceStatus.UP.equals(registrant.getStatus())) {
            lease.serviceUp();
        }
        //设置应用实例的操作类型 为添加 供客户端做增量更新使用
        registrant.setActionType(ActionType.ADDED);
        //将当前注册操作添加到最近租约变更记录队列 供客户端做增量更新使用
        recentlyChangedQueue.add(new RecentlyChangedItem(lease));
        //更新租约的过期时间
        registrant.setLastUpdatedTimestamp();
        //使读写缓存无效
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})",
                registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
    } finally {
        read.unlock();
    }
}
```

Lease：租债器 装饰eureka client实例

```java
public class Lease<T> {

    enum Action {
        Register, Cancel, Renew
    };

    //默认心跳续约时间
    public static final int DEFAULT_DURATION_IN_SECS = 90;

    private T holder;
    //服务剔除时间戳
    private long evictionTimestamp;
    //服务注册时间戳
    private long registrationTimestamp;
    //服务上线时间戳
    private long serviceUpTimestamp;
    // Make it volatile so that the expiration task would see this quicker
    //续约后的服务过期时间
    private volatile long lastUpdateTimestamp;
    private long duration;
    
    //心跳续约
    public void renew() {
          //当前时间+续约时间
        lastUpdateTimestamp = System.currentTimeMillis() + duration;

    }

    //服务剔除
    public void cancel() {
        if (evictionTimestamp <= 0) {
            evictionTimestamp = System.currentTimeMillis();
        }
    }

    //标记服务上线
    public void serviceUp() {
        if (serviceUpTimestamp == 0) {
            serviceUpTimestamp = System.currentTimeMillis();
        }
    }
    
    //additionalLeaseMs：
    public boolean isExpired(long additionalLeaseMs) {
        //剔除时间大于0 或 当前时间 > 续约后的服务过期时间 + 续约时间 + additionalLeaseMs
        //这个地方 eureka承认有错误 应该是是剔除时间大于0 或 当前时间 > 续约后的服务过期时间 + additionalLeaseMs
        //lastUpdateTimestamp本身就代表续约后的服务过期时间
        return (evictionTimestamp > 0 || System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs));
    }
```



# 心跳续约

对于心跳续约，当 Eureka-Client 收到 404 响应后，会重新发起 注册请求



入口：com.netflix.eureka.resources.InstanceResource#renewLease()

```java
@PUT
public Response renewLease(
        @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
        @QueryParam("overriddenstatus") String overriddenStatus,
        @QueryParam("status") String status,
        @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
    //true表示来自集群内其他节点的同步
    boolean isFromReplicaNode = "true".equals(isReplication);
    boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);

  	//续租失败，返回404，EurekaClient端收到404后会发起注册请求
    if (!isSuccess) {
        logger.warn("Not Found (Renew): {} - {}", app.getName(), id);
        // Eureka-Client 收到 404 响应后，会重新发起 注册
        return Response.status(Status.NOT_FOUND).build();
    }
 
    Response response;
    if (lastDirtyTimestamp != null && serverConfig.shouldSyncWhenTimestampDiffers()) {
        response = this.validateDirtyTimestamp(Long.valueOf(lastDirtyTimestamp), isFromReplicaNode);
        // Store the overridden status since the validation found out the node that replicates wins
        if (response.getStatus() == Response.Status.NOT_FOUND.getStatusCode()
                && (overriddenStatus != null)
                && !(InstanceStatus.UNKNOWN.name().equals(overriddenStatus))
                && isFromReplicaNode) {
            registry.storeOverriddenStatusIfRequired(app.getAppName(), id, InstanceStatus.valueOf(overriddenStatus));
        }
    } else {
        response = Response.ok().build();
    }
    logger.debug("Found (Renew): {} - {}; reply status={}", app.getName(), id, response.getStatus());
    return response;
}
```

com.netflix.eureka.resources.InstanceResource#validateDirtyTimestamp

```java
private Response validateDirtyTimestamp(Long lastDirtyTimestamp,
                                        boolean isReplication) {
    //根据服务名和实例id拿到实例信息
    InstanceInfo appInfo = registry.getInstanceByAppAndId(app.getName(), id, false);
    if (appInfo != null) {
        //lastDirtyTimestamp：客户端传过来的该实例最后一次发送心跳续约的时间戳
        //appInfo.getLastDirtyTimestamp()：eureka server保存到的该实例最后一次发送心跳续约的时间戳
        if ((lastDirtyTimestamp != null) && 
            (!lastDirtyTimestamp.equals(appInfo.getLastDirtyTimestamp()))) {
            Object[] args = {id, appInfo.getLastDirtyTimestamp(), lastDirtyTimestamp, isReplication};
			//客户端传递过来的lastDirtyTimestamp比服务端保存的实例的lastDirtyTimestamp大 （正常情况下应该是相等的）
            if (lastDirtyTimestamp > appInfo.getLastDirtyTimestamp()) {
                return Response.status(Status.NOT_FOUND).build();
            } else if (appInfo.getLastDirtyTimestamp() > lastDirtyTimestamp) {
                // In the case of replication, send the current instance info in the registry for the
                // replicating node to sync itself with this one.
                if (isReplication) {
                    return Response.status(Status.CONFLICT).entity(appInfo).build();
                } else {
                    return Response.ok().build();
                }
            }
        }

    }
    return Response.ok().build();
}
```



具体的业务代码：org.springframework.cloud.netflix.eureka.server.InstanceRegistry#renew

```java
public boolean renew(final String appName, final String serverId,
      boolean isReplication) {
   log("renew " + appName + " serverId " + serverId + ", isReplication {}"
         + isReplication);
   List<Application> applications = getSortedApplications();
   //根据客户端传递过来的服务名称和实例id 双层for循环拿到对应的实例信息
   for (Application input : applications) {
      if (input.getName().equals(appName)) {
         InstanceInfo instance = null;
         for (InstanceInfo info : input.getInstances()) {
            if (info.getId().equals(serverId)) {
               instance = info;
               break;
            }
         }
         //发布监听
         publishEvent(new EurekaInstanceRenewedEvent(this, appName, serverId,
               instance, isReplication));
         break;
      }
   }
   //父类的心跳续约方法
   return super.renew(appName, serverId, isReplication);
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#renew

```java
public boolean renew(final String appName, final String id, final boolean isReplication) {
    //调用父类的心跳续约方法
    if (super.renew(appName, id, isReplication)) {
        //集群同步
        replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
        return true;
    }
    return false;
}
```



com.netflix.eureka.registry.AbstractInstanceRegistry#renew

```java
public boolean renew(String appName, String id, boolean isReplication) {
    RENEW.increment(isReplication);
    //根据服务名称拿到微服务组
    Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
    Lease<InstanceInfo> leaseToRenew = null;
    //根据微服务id拿到实例信息
    if (gMap != null) {
        leaseToRenew = gMap.get(id);
    }
    if (leaseToRenew == null) {
        RENEW_NOT_FOUND.increment(isReplication);
        logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
        return false;
    } else {
        InstanceInfo instanceInfo = leaseToRenew.getHolder();
        if (instanceInfo != null) {
            // touchASGCache(instanceInfo.getASGName());
            InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                    instanceInfo, leaseToRenew, isReplication);
            if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                logger.info("Instance status UNKNOWN possibly due to deleted override for instance {}"
                        + "; re-register required", instanceInfo.getId());
                RENEW_NOT_FOUND.increment(isReplication);
                return false;
            }
            //应用实例的状态与覆盖状态不相等，使用覆盖状态覆盖应用实例的状态
            if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                logger.info(
                        "The instance status {} is different from overridden instance status {} for instance {}. "
                                + "Hence setting the status to overridden status", instanceInfo.getStatus().name(),
                                instanceInfo.getOverriddenStatus().name(),
                                instanceInfo.getId());
                instanceInfo.setStatusWithoutDirty(overriddenInstanceStatus);

            }
        }
        //增加每分钟续约次数 自我保护机制会用到
        renewsLastMin.increment();
        //设置续约后的过期时间
        leaseToRenew.renew();
        return true;
    }
}
```

# 集群同步

集群同步原理：

- Eureka-Server 集群不区分**主从节点**或者 **Primary & Secondary 节点**，所有节点**相同角色( 也就是没有角色 )，完全对等**。
- Eureka-Client 可以向**任意** Eureka-Server 发起任意**读写**操作
- eureka client选择任意一个Eureka-Server发起服务注册 心跳续约 服务下架 服务状态更新 状态删除操作，eureka server 成功执行这些操作后 会循环eureka server集群内每个节点，将操作复制到另外的 Eureka-Server 以达到**最终一致性**。注意，Eureka-Server 保证AP。

eureka集群有3个节点 server1 server2 server3 

client1选择server2发起注册请求 server2处理完client1的注册请求 （注意 此时并未返回success）会进行集群同步操作 遍历eureka server列表 向server1和server3发出同样的client1注册请求 同时 isReplication为true 表示本次注册请求来自于集群同步 这样server1和server3接收到来自server2的注册请求后 不会再次进行集群同步 从而造成死循环



com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#replicateToPeers负责所有的集群同步

服务注册 心跳续约 服务下架 服务状态更新 状态删除

```java
private void replicateToPeers(Action action, String appName, String id,
                              InstanceInfo info /* optional */,
                              InstanceStatus newStatus /* optional */, boolean isReplication) {
    //isReplication：是否来自于集群同步 判断这个请求来自于eureka client还是eureka server（同步）
    //action：操作类型
    Stopwatch tracer = action.getTimer().start();
    try {
        if (isReplication) {
            numberOfReplicationsLastMin.increment();
        }
        // If it is a replication already, do not replicate again as this will create a poison replication
        if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
            //当集群只有单个节点或当前操作（服务注册 心跳续约 服务下架 服务状态更新 状态删除）来自于集群同步 返回
            return;
        }
        //遍历所有的eureka server节点
        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // If the url represents this host, do not replicate to yourself.
            //判断当前节点是否为自己
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            //向其他的eureka server发出相同的操作请求
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } finally {
        tracer.stop();
    }
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#replicateInstanceActionsToPeers()

```java
private void replicateInstanceActionsToPeers(Action action, String appName,
                                             String id, InstanceInfo info, InstanceStatus newStatus,
                                             PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry = null;
        CurrentRequestVersion.set(Version.V2);
        //根据不同的操作 执行对应的动作
        switch (action) {
            case Cancel:
                node.cancel(appName, id);
                break;
            case Heartbeat:
                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                break;
            case Register:
                node.register(info);
                break;
            case StatusUpdate:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                break;
            case DeleteStatusOverride:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.deleteStatusOverride(appName, id, infoFromRegistry);
                break;
        }
    } catch (Throwable t) {
        logger.error("Cannot replicate information to {} for action {}", node.getServiceUrl(), action.name(), t);
    }
}
```



# 服务剔除

在EurekaServerAutoConfiguration中会导入EurekaServerInitializerConfiguration进行eureka server的一些初始化操作

初始化eureka server的配置

初始化eureka server context(集群同步注册信息、启动一些定时器（服务剔除、自我保护机制监听）、初始化自我保护机制的阈值)

**服务剔除集群间不会进行同步** 只进行本地剔除

如果想要自我保护机制正常运行 建议客户端的心跳续约间隔与服务端的相同



EurekaServerAutoConfiguration自动配置类导入了EurekaServerInitializerConfiguration

```java
public class EurekaServerInitializerConfiguration
		implements ServletContextAware, SmartLifecycle, Ordered {
		//实现了SmartLifecycle的生命周期方法 spring在启动过程中会进行回调
        public void start() {
           //创建一个新的线程来执行操作 并不会影响main线程
           new Thread(() -> {
              try {
                 //初始化eureka server上下文
                 eurekaServerBootstrap.contextInitialized(
                       EurekaServerInitializerConfiguration.this.servletContext);
                 log.info("Started Eureka Server");
								//发布事件
                 publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
                 EurekaServerInitializerConfiguration.this.running = true;
                 publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
              }
              catch (Exception ex) {
                 // Help!
                 log.error("Could not initialize Eureka servlet context", ex);
              }
           }).start();
        }
        
       }
}
```

org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#contextInitialized

```java
public void contextInitialized(ServletContext context) {
   try {
      //读取配置文件 初始化eureka环境
      initEurekaEnvironment();
      //初始化eureka 上下文   去集群其他节点同步注册表  初始化服务剔除定时任务
      initEurekaServerContext();

      context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
   }
   catch (Throwable e) {
      log.error("Cannot bootstrap eureka server :", e);
      throw new RuntimeException("Cannot bootstrap eureka server :", e);
   }
}
```



org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#initEurekaServerContext

```java
protected void initEurekaServerContext() throws Exception {
   // For backward compatibility
   JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
         XStream.PRIORITY_VERY_HIGH);
   XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
         XStream.PRIORITY_VERY_HIGH);

   if (isAws(this.applicationInfoManager.getInfo())) {
      this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
            this.eurekaClientConfig, this.registry, this.applicationInfoManager);
      this.awsBinder.start();
   }

   EurekaServerContextHolder.initialize(this.serverContext);

   log.info("Initialized server context");

   // Copy registry from neighboring eureka node
   //从其他eureka server节点同步集群信息 返回同步到的客户端数量
   int registryCount = this.registry.syncUp();
    //初始化自我保护阈值 初始化服务剔除定时任务
   this.registry.openForTraffic(this.applicationInfoManager, registryCount);

   // Register all monitoring statistics.
   EurekaMonitors.registerAllStats();
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#syncUp

```java
public int syncUp() {
    // Copy entire entry from neighboring DS node
    int count = 0;

    //serverConfig.getRegistrySyncRetries()：读取配置的重试次数
    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        if (i > 0) {
            try {
                //重试等待时间
                Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
            } catch (InterruptedException e) {
                logger.warn("Interrupted during registry transfer..");
                break;
            }
        }
        //获取集群注册信息 此时的eureka sever 相对于其他eureka server节点来说 就是一个eureka client
        Applications apps = eurekaClient.getApplications();
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                try {
                    if (isRegisterable(instance)) {
                        //拿到注册信息后 注册到本地
                        register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                        count++;
                    }
                } catch (Throwable t) {
                    logger.error("During DS init copy", t);
                }
            }
        }
    }
    return count;
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#openForTraffic()

省去了org.springframework.cloud.netflix.eureka.server.InstanceRegistry#openForTraffic()的调用 没干啥事

```java
public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
    // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
    //预计需要接受心跳续约的客户端数量 就是从集群同步到的客户端数量
    this.expectedNumberOfClientsSendingRenews = count;
    //重新计算触发自我保护机制阈值
    updateRenewsPerMinThreshold();
    logger.info("Got {} instances from neighboring DS node", count);
    logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
    this.startupTime = System.currentTimeMillis();
    if (count > 0) {
        this.peerInstancesTransferEmptyOnStartup = false;
    }
    DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
    boolean isAws = Name.Amazon == selfName;
    if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
        logger.info("Priming AWS connections for all replicas..");
        primeAwsReplicas(applicationInfoManager);
    }
    logger.info("Changing status to UP");
    applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
    //创建线程 执行服务剔除定时任务
    super.postInit();
}
```

com.netflix.eureka.registry.AbstractInstanceRegistry#postInit

```java
protected void postInit() {
    renewsLastMin.start();
    if (evictionTaskRef.get() != null) {
        evictionTaskRef.get().cancel();
    }
    evictionTaskRef.set(new EvictionTask());
  	//添加定时任务
    evictionTimer.schedule(evictionTaskRef.get(),
            serverConfig.getEvictionIntervalTimerInMs(),
            serverConfig.getEvictionIntervalTimerInMs());
}
```

## 服务剔除任务类

定时清除长时间没有发送心跳续约的客户端  默认**60s**执行一次

```java
class EvictionTask extends TimerTask {

    private final AtomicLong lastExecutionNanosRef = new AtomicLong(0l);

    @Override
    public void run() {
        try {
            //获取补偿时间毫秒数 计算公式 = 当前时间 - 最后任务执行时间 - 任务执行频率
            long compensationTimeMs = getCompensationTimeMs();
            logger.info("Running the evict task with compensationTime {}ms", compensationTimeMs);
           //执行服务剔除方法
            evict(compensationTimeMs);
        } catch (Throwable e) {
            logger.error("Could not run the evict task", e);
        }
    }

}
```



com.netflix.eureka.registry.AbstractInstanceRegistry#evict(long)

```java
public void evict(long additionalLeaseMs) {
    logger.debug("Running the evict task");
	  //如果触发了自我保护机制 就不进行服务剔除
    //isLeaseExpirationEnabled()：判断是否打开（打开自我保护机制后就有可能触发）和触发服务保护机制
    if (!isLeaseExpirationEnabled()) {
        logger.debug("DS: lease expiration is currently disabled.");
        return;
    }

    //遍历服务注册表 调用租债器的isExpired()方法 判断服务是否过期
    //需要剔除的节点信息保存在expiredLeases
    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
    for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
        Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
        if (leaseMap != null) {
            for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                Lease<InstanceInfo> lease = leaseEntry.getValue();
              	//过期了
                if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                    expiredLeases.add(lease);
                }
            }
        }
    }

    //eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Eureka Server 会将这些实例保护起来，让这些实例不会过期  即有15%的节点在15分钟之内没有进行心跳续约
    //取自我保护机制的阈值和需要剔除的节点数量中小的那一个（先尽量不触发自我保护机制）
    
    //拿到eureka server所有的注册节点数量
    int registrySize = (int) getLocalRegistrySize();
    
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    //如果清理租约数量 > evictionLimit 就会触发自我保护机制
    int evictionLimit = registrySize - registrySizeThreshold;
    //计算 最大允许清理租约数量（不触发自我保护机制）
    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
    if (toEvict > 0) {
        logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);

        Random random = new Random(System.currentTimeMillis());
        //开始剔除节点
        for (int i = 0; i < toEvict; i++) {
            //随机剔除
            int next = i + random.nextInt(expiredLeases.size() - i);
            Collections.swap(expiredLeases, i, next);
            Lease<InstanceInfo> lease = expiredLeases.get(i);

            String appName = lease.getHolder().getAppName();
            String id = lease.getHolder().getId();
            EXPIRED.increment();
            logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
            //服务剔除
            internalCancel(appName, id, false);
        }
    }
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#isLeaseExpirationEnabled

当最近一分钟心跳次数( `renewsLastMin` ) **小于** `numberOfRenewsPerMinThreshold` 时，并且开启自动保护模式开关( `eureka.enableSelfPreservation = true` 默认为true) 时，**触发自动保护机制，不再自动过期租约**

```java
public boolean isLeaseExpirationEnabled() {
  	//如果禁止开启自我保护机制
    if (!isSelfPreservationModeEnabled()) {
        // The self preservation mode is disabled, hence allowing the instances to expire.
        return true;
    }
    //getNumOfRenewsInLastMin：最近一分钟的心跳次数
    //numberOfRenewsPerMinThreshold：触发自我保护机制阈值 一分钟内eureka server需要接收到多少个心跳续约当前eureka servr节点才算正常
    
    return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
}
```

com.netflix.eureka.lease.Lease#isExpired(long)

😈**注意**：在不考虑 `additionalLeaseMs` 参数的情况下，租约过期时间比预期多了**一个** `duration`，原因在于 `renew()` 方法错误的设置 `lastUpdateTimestamp = System.currentTimeMillis() + duration`，正确的设置应该是 `lastUpdateTimestamp = System.currentTimeMillis()` 。

```java
public boolean isExpired(long additionalLeaseMs) {
    return (evictionTimestamp > 0 || System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs));
}
public void renew() {
   lastUpdateTimestamp = System.currentTimeMillis() + duration;
    
  //  正确的设置应该是 `lastUpdateTimestamp = System.currentTimeMillis()
}
```



org.springframework.cloud.netflix.eureka.server.InstanceRegistry#internalCancel()

```java
@Override
protected boolean internalCancel(String appName, String id, boolean isReplication) {
   handleCancelation(appName, id, isReplication);
   return super.internalCancel(appName, id, isReplication);
}
```

com.netflix.eureka.registry.AbstractInstanceRegistry#internalCancel()

```java
protected boolean internalCancel(String appName, String id, boolean isReplication) {
    try {
        read.lock();
        //增加取消注册次数到监控
        CANCEL.increment(isReplication);
        Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> leaseToCancel = null;
        if (gMap != null) {
        	  //剔除租约映射 返回剔除的租约
            leaseToCancel = gMap.remove(id);
        }
        synchronized (recentCanceledQueue) {
            //添加到最近取消注册的调试队列 用于 Eureka-Server 运维界面的显示，无实际业务逻辑使用
            /**
            * 最近取消注册的调试队列
            * key ：添加时的时间戳
            * value ：字符串 = 应用名(应用实例信息编号)
            **/
            recentCanceledQueue.add(new Pair<Long, String>(System.currentTimeMillis(), appName + "(" + id + ")"));
        }
        //移除应用实例覆盖状态映射
        InstanceStatus instanceStatus = overriddenInstanceStatusMap.remove(id);
        if (instanceStatus != null) {
            logger.debug("Removed instance id {} from the overridden map which has value {}", id, instanceStatus.name());
        }
        //租约不存在
        if (leaseToCancel == null) {
            //添加取消租约不存在到监控
            CANCEL_NOT_FOUND.increment(isReplication);
            logger.warn("DS: Registry: cancel failed because Lease is not registered for: {}/{}", appName, id);
            return false;
        } else {
            //更新剔除时间
            leaseToCancel.cancel();
            InstanceInfo instanceInfo = leaseToCancel.getHolder();
            String vip = null;
            String svip = null;
            if (instanceInfo != null) {
                //设置当前服务剔除的操作类型 供客户端做增量更新使用
                instanceInfo.setActionType(ActionType.DELETED);
                 //将当前服务剔除或服务下架操作添加到最近租约变更记录队列 供客户端做增量更新使用
                recentlyChangedQueue.add(new RecentlyChangedItem(leaseToCancel));
                instanceInfo.setLastUpdatedTimestamp();
                vip = instanceInfo.getVIPAddress();
                svip = instanceInfo.getSecureVipAddress();
            }
            //使指定key对应的读写缓存失效
            invalidateCache(appName, vip, svip);
            logger.info("Cancelled instance {}/{} (replication={})", appName, id, isReplication);
            return true;
        }
    } finally {
        read.unlock();
    }
}
```



# 服务下架

入口：com.netflix.eureka.resources.InstanceResource#cancelLease

```

    @DELETE
    public Response cancelLease(@HeaderParam("x-netflix-discovery-replication") String isReplication) {
        try {
            boolean isSuccess = this.registry.cancel(this.app.getName(), this.id, "true".equals(isReplication));
            if (isSuccess) {
                logger.debug("Found (Cancel): {} - {}", this.app.getName(), this.id);
                return Response.ok().build();
            } else {
                logger.info("Not Found (Cancel): {} - {}", this.app.getName(), this.id);
                return Response.status(Status.NOT_FOUND).build();
            }
        } catch (Throwable var3) {
            logger.error("Error (cancel): {} - {}", new Object[]{this.app.getName(), this.id, var3});
            return Response.serverError().build();
        }
    }
```



org.springframework.cloud.netflix.eureka.server.InstanceRegistry#cancel

```java
@Override
public boolean cancel(String appName, String serverId, boolean isReplication) {
   handleCancelation(appName, serverId, isReplication);
   return super.cancel(appName, serverId, isReplication);
}
```

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#cancel

```java
public boolean cancel(final String appName, final String id,
                      final boolean isReplication) {
    if (super.cancel(appName, id, isReplication)) {
        replicateToPeers(Action.Cancel, appName, id, null, null, isReplication);
        synchronized (lock) {
            if (this.expectedNumberOfClientsSendingRenews > 0) {
                //更新预计需要接受心跳续约的客户端数量
                this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews - 1;
                //重新计算触发自我保护机制阈值
                updateRenewsPerMinThreshold();
            }
        }
        return true;
    }
    return false;
}
```

com.netflix.eureka.registry.AbstractInstanceRegistry#cancel()：最终调用的和服务剔除是同一个方法

```java
public boolean cancel(String appName, String id, boolean isReplication) {
    return internalCancel(appName, id, isReplication);
}
```





# 自我保护

eureka server短时间内（默认15分钟）发现大量心跳续约（15%）过期，就会触发自我保护机制 （根据客户端一段时间内的心跳续约来判断）

eureka server开启了自我保护机制后 就不会进行服务剔除了

**服务注册、 服务下架 、15分钟自动计算 、eureka server初始化 会重新计算触发自我保护机制的阈值   （注：服务剔除不会重新计算）**

注意：生产环境的服务注册、服务下架的情况较少 （即有机会重新计算触发自我保护机制的阈值也少） 这时，默认15分钟执行一次的定时任务就很重要了

 当最近一分钟心跳次数( `renewsLastMin` ) **小于** `numberOfRenewsPerMinThreshold` 时，并且开启自动保护模式开关( `eureka.enableSelfPreservation = true` 默认为true) 时，**触发自动保护机制，不再自动过期租约**

```java
public boolean isLeaseExpirationEnabled() {
  	//禁止自我保护机制
    if (!isSelfPreservationModeEnabled()) {
        // The self preservation mode is disabled, hence allowing the instances to expire.
        return true;
    }
    //getNumOfRenewsInLastMin：最近一分钟的心跳次数
    //numberOfRenewsPerMinThreshold：触发自我保护机制的阈值 一分钟内eureka server需要接收到多少个心跳续约当前eureka servr节点才算正常
    return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
}
```

在服务注册或服务下架时 会重新计算触发自我保护机制阈值

```java
protected void updateRenewsPerMinThreshold() {
    //numberOfRenewsPerMinThreshold：触发自我保护机制阈值 计算出来的
    //expectedNumberOfClientsSendingRenews:预计需要接受心跳续约的客户端数量 根据注册的客户端节点数量实时变化
    //serverConfig.getExpectedClientRenewalIntervalSeconds()：客户端需要多长时间发送一次心跳给客户端(服务端配置)
    //serverConfig.getRenewalPercentThreshold()：eureka server期望从客户端接收到的心跳续约的最低百分比
    this.numberOfRenewsPerMinThreshold = (int) (this.expectedNumberOfClientsSendingRenews
            * (60.0 / serverConfig.getExpectedClientRenewalIntervalSeconds())//60.0：1分钟
            * serverConfig.getRenewalPercentThreshold());
}
```







PeerAwareInstanceRegistryImpl#init   

```java
public void init(PeerEurekaNodes peerEurekaNodes) throws Exception {
    this.numberOfReplicationsLastMin.start();
    this.peerEurekaNodes = peerEurekaNodes;
    this.initializedResponseCache();
  	//每隔15分钟计算自我保护的阈值
    this.scheduleRenewalThresholdUpdateTask();
    this.initRemoteRegionRegistry();

    try {
        Monitors.registerObject(this);
    } catch (Throwable var3) {
        logger.warn("Cannot register the JMX monitor for the InstanceRegistry :", var3);
    }

}

 private void scheduleRenewalThresholdUpdateTask() {
        this.timer.schedule(new TimerTask() {
            public void run() {
              	//计算自我保护的阈值
                PeerAwareInstanceRegistryImpl.this.updateRenewalThreshold();
            }
        }, (long)this.serverConfig.getRenewalThresholdUpdateIntervalMs(), (long)this.serverConfig.getRenewalThresholdUpdateIntervalMs());
    }
```









默认每15分钟执行一次的定时任务 注意 这里的expectedNumberOfClientsSendingRenews是去其他eureka server上拿的 跟其他三种类型不一样

com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#updateRenewalThreshold

```java
private void updateRenewalThreshold() {
    try {
        //计算应用实例数
        Applications apps = eurekaClient.getApplications();
        int count = 0;
        
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                if (this.isRegisterable(instance)) {
                    ++count;
                }
            }
        }
        synchronized (lock) {
            // Update threshold only if the threshold is greater than the
            // current expected threshold or if self preservation is disabled.
            //如果应用实例数（即最新的客户端数量） > eureka server期望从客户端接收到的心跳续约的最低百分比 * 预计需要接受心跳续约的客户端数量
            //或者没有禁止自我保护机制    就重新计算自我保护机制阈值   如果重新计算，自动保护机制会每次定时执行后失效。
            
            if ((count) > (serverConfig.getRenewalPercentThreshold() * expectedNumberOfClientsSendingRenews)
                    || (!this.isSelfPreservationModeEnabled())) {
                //应用实例数 赋值给 预计需要接受心跳续约的客户端数量
                this.expectedNumberOfClientsSendingRenews = count;
                //重新计算触发自我保护机制阈值
                updateRenewsPerMinThreshold();
            }
        }
        logger.info("Current renewal threshold is : {}", numberOfRenewsPerMinThreshold);
    } catch (Throwable e) {
        logger.error("Cannot update renewal threshold", e);
    }
}
```



# 服务发现（服务端缓存架构）

## 读缓存

客户端获取注册表

入口：com.netflix.eureka.resources.ApplicationsResource#getContainers

com.netflix.eureka.registry.ResponseCacheImpl#getGZIP

```java
public byte[] getGZIP(Key key) {
    Value payload = getValue(key, shouldUseReadOnlyResponseCache);
    if (payload == null) {
        return null;
    }
    return payload.getGzipped();
}
```

com.netflix.eureka.registry.ResponseCacheImpl#getValue

```java
Value getValue(final Key key, boolean useReadOnlyCache) {
    Value payload = null;
    try {
        //useReadOnlyCache：是否打开只读缓存 默认打开
        if (useReadOnlyCache) {
            //先从只读缓存里获取数据
            final Value currentPayload = readOnlyCacheMap.get(key);
            if (currentPayload != null) {
                payload = currentPayload;
            } else {
                //只读缓存没有 再从读写缓存里获取数据
                payload = readWriteCacheMap.get(key);
                //并放一份到只读缓存中
                readOnlyCacheMap.put(key, payload);
            }
        } else {
            payload = readWriteCacheMap.get(key);
        }
    } catch (Throwable t) {
        logger.error("Cannot get value for key : {}", key, t);
    }
    return payload;
}
```



注册信息的三层容器架构    不能保证强一致性(AP)  保证最终一致性 与zk(CP)区别 

只读缓存：ConcurrentHashMap   读操作远多于写操作

读写缓存：guava  给真实数据降压  默认过期时间3分钟

真实数据 ：ConsurrentHashMap   真实数据的操作要加锁 服务注册、服务下架、服务剔除加读锁      客户端做服务发现（eureka client获取eureka server上的注册信息）时加写锁



客户端获取注册信息流程：

先从**只读缓存**获取 只读缓存没有 就去**读写缓存**（google guava实现）获取 ，读写缓存创建时注册了**监听器**，如果读写缓存没有 会执行监听器的逻辑 自动加载**真实数据**到读写缓存 当从读写缓存获取到数据后 会向只读缓存put一份



```java
public class ResponseCacheImpl implements ResponseCache {
    //创建ResponseCacheImpl对象的时候会初始化只读缓存和读写缓存(基于guava实现)
    ResponseCacheImpl(EurekaServerConfig serverConfig, ServerCodecs serverCodecs, AbstractInstanceRegistry registry) {
        this.serverConfig = serverConfig;
        this.serverCodecs = serverCodecs;
        this.shouldUseReadOnlyResponseCache = serverConfig.shouldUseReadOnlyResponseCache();
        this.registry = registry;

        long responseCacheUpdateIntervalMs = serverConfig.getResponseCacheUpdateIntervalMs();
        this.readWriteCacheMap =
               //初始化读写缓存 初始化大小 过期时间（默认3分钟）CacheBuilder.newBuilder().initialCapacity(serverConfig.getInitialCapacityOfResponseCache())
                        .expireAfterWrite(serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS)
                        .removalListener(new RemovalListener<Key, Value>() {
                            //缓存的移除通知
                            @Override
                            public void onRemoval(RemovalNotification<Key, Value> notification) {
                                Key removedKey = notification.getKey();
                                if (removedKey.hasRegions()) {
                                    Key cloneWithNoRegions = removedKey.cloneWithoutRegions();
                                    regionSpecificKeys.remove(cloneWithNoRegions, removedKey);
                                }
                            }
                        })
                        .build(new CacheLoader<Key, Value>() {
                            //缓存构建监听器
                            //当从读写缓存get不到数据的时候(记录不存在)，会进入到这里
                            @Override
                            public Value load(Key key) throws Exception {
                                if (key.hasRegions()) {
                                    Key cloneWithNoRegions = key.cloneWithoutRegions();
                                    regionSpecificKeys.put(cloneWithNoRegions, key);
                                }
                                Value value = generatePayload(key);
                                return value;
                            }
                        });
		//shouldUseReadOnlyResponseCache：是否开启只读缓存
        if (shouldUseReadOnlyResponseCache) {
            timer.schedule(getCacheUpdateTask(),
                    new Date(((System.currentTimeMillis() / responseCacheUpdateIntervalMs) * responseCacheUpdateIntervalMs)
                            + responseCacheUpdateIntervalMs),
                    responseCacheUpdateIntervalMs);
        }

        try {
            Monitors.registerObject(this);
        } catch (Throwable e) {
            logger.warn("Cannot register the JMX monitor for the InstanceRegistry", e);
        }
    }

}
```

com.netflix.eureka.registry.ResponseCacheImpl#generatePayload()：构建

```java
private Value generatePayload(Key key) {
    Stopwatch tracer = null;
    try {
        String payload;
        switch (key.getEntityType()) {
            case Application:
                boolean isRemoteRegionRequested = key.hasRegions();
				        //全量获取
                if (ALL_APPS.equals(key.getName())) {
                    if (isRemoteRegionRequested) {
                        //云服务 省略
                        tracer = serializeAllAppsWithRemoteRegionTimer.start();
                        payload = getPayLoad(key, registry.getApplicationsFromMultipleRegions(key.getRegions()));
                    } else {
                        //走这里
                        tracer = serializeAllAppsTimer.start();
                        payload = getPayLoad(key, registry.getApplications());
                    }
                    
                } else if (ALL_APPS_DELTA.equals(key.getName())) {
                    //增量获取
                    if (isRemoteRegionRequested) {
                        tracer = serializeDeltaAppsWithRemoteRegionTimer.start();
                        versionDeltaWithRegions.incrementAndGet();
                        versionDeltaWithRegionsLegacy.incrementAndGet();
                        payload = getPayLoad(key,
                                registry.getApplicationDeltasFromMultipleRegions(key.getRegions()));
                    } else {
                        tracer = serializeDeltaAppsTimer.start();
                        versionDelta.incrementAndGet();
                        versionDeltaLegacy.incrementAndGet();
                        payload = getPayLoad(key, registry.getApplicationDeltas());
                    }
                } else {
                    tracer = serializeOneApptimer.start();
                    payload = getPayLoad(key, registry.getApplication(key.getName()));
                }
                break;
            case VIP:
            case SVIP:
                tracer = serializeViptimer.start();
                payload = getPayLoad(key, getApplicationsForVip(key, registry));
                break;
            default:
                logger.error("Unidentified entity type: {} found in the cache key.", key.getEntityType());
                payload = "";
                break;
        }
        return new Value(payload);
    } finally {
        if (tracer != null) {
            tracer.stop();
        }
    }
}
```

## **全量拉取**

com.netflix.eureka.registry.AbstractInstanceRegistry#getApplications()  

```java
public Applications getApplications() {
    boolean disableTransparentFallback = serverConfig.disableTransparentFallbackToOtherRegion();
    if (disableTransparentFallback) {
        //本地获取
        return getApplicationsFromLocalRegionOnly();
    } else {
        return getApplicationsFromAllRemoteRegions();  // Behavior of falling back to remote region can be disabled.
    }
}
```

com.netflix.eureka.registry.AbstractInstanceRegistry#getApplicationsFromLocalRegionOnly

```java
@Override
public Applications getApplicationsFromLocalRegionOnly() {
    return getApplicationsFromMultipleRegions(EMPTY_STR_ARRAY);
}
```

com.netflix.eureka.registry.AbstractInstanceRegistry#getApplicationsFromMultipleRegions

```java
public Applications getApplicationsFromMultipleRegions(String[] remoteRegions) {

    boolean includeRemoteRegion = null != remoteRegions && remoteRegions.length != 0;

    logger.debug("Fetching applications registry with remote regions: {}, Regions argument {}",
            includeRemoteRegion, remoteRegions);

    if (includeRemoteRegion) {
        GET_ALL_WITH_REMOTE_REGIONS_CACHE_MISS.increment();
    } else {
        GET_ALL_CACHE_MISS.increment();
    }
    Applications apps = new Applications();
    apps.setVersion(1L);
    //registry：真实数据
    for (Entry<String, Map<String, Lease<InstanceInfo>>> entry : registry.entrySet()) {
        Application app = null;

        if (entry.getValue() != null) {
            for (Entry<String, Lease<InstanceInfo>> stringLeaseEntry : entry.getValue().entrySet()) {
                Lease<InstanceInfo> lease = stringLeaseEntry.getValue();
                if (app == null) {
                    app = new Application(lease.getHolder().getAppName());
                }
                app.addInstance(decorateInstanceInfo(lease));
            }
        }
        if (app != null) {
            apps.addApplication(app);
        }
    }
    if (includeRemoteRegion) {
        for (String remoteRegion : remoteRegions) {
            RemoteRegionRegistry remoteRegistry = regionNameVSRemoteRegistry.get(remoteRegion);
            if (null != remoteRegistry) {
                Applications remoteApps = remoteRegistry.getApplications();
                for (Application application : remoteApps.getRegisteredApplications()) {
                    if (shouldFetchFromRemoteRegistry(application.getName(), remoteRegion)) {
                        logger.info("Application {}  fetched from the remote region {}",
                                application.getName(), remoteRegion);

                        Application appInstanceTillNow = apps.getRegisteredApplications(application.getName());
                        if (appInstanceTillNow == null) {
                            appInstanceTillNow = new Application(application.getName());
                            apps.addApplication(appInstanceTillNow);
                        }
                        for (InstanceInfo instanceInfo : application.getInstances()) {
                            appInstanceTillNow.addInstance(instanceInfo);
                        }
                    } else {
                        logger.debug("Application {} not fetched from the remote region {} as there exists a "
                                        + "whitelist and this app is not in the whitelist.",
                                application.getName(), remoteRegion);
                    }
                }
            } else {
                logger.warn("No remote registry available for the remote region {}", vb );
            }
        }
    }
    apps.setAppsHashCode(apps.getReconcileHashCode());
    return apps;
}
```



## 增量拉取

com.netflix.eureka.registry.AbstractInstanceRegistry#getApplicationDeltas

```java
public Applications getApplicationDeltas() {
    GET_ALL_CACHE_MISS_DELTA.increment();
    Applications apps = new Applications();
    apps.setVersion(responseCache.getVersionDelta().get());
    Map<String, Application> applicationInstancesMap = new HashMap<String, Application>();
    try {
        write.lock();
        //最近租约变更记录队列 默认保存3分钟之内改变过状态的微服务实例
        //这个队列里的节点 默认过期时间为3分钟
        Iterator<RecentlyChangedItem> iter = this.recentlyChangedQueue.iterator();
        logger.debug("The number of elements in the delta queue is : {}",
                this.recentlyChangedQueue.size());
        while (iter.hasNext()) {
            Lease<InstanceInfo> lease = iter.next().getLeaseInfo();
            InstanceInfo instanceInfo = lease.getHolder();
            logger.debug(
                    "The instance id {} is found with status {} and actiontype {}",
                    instanceInfo.getId(), instanceInfo.getStatus().name(), instanceInfo.getActionType().name());
            Application app = applicationInstancesMap.get(instanceInfo
                    .getAppName());
            if (app == null) {
                app = new Application(instanceInfo.getAppName());
                applicationInstancesMap.put(instanceInfo.getAppName(), app);
                apps.addApplication(app);
            }
            app.addInstance(new InstanceInfo(decorateInstanceInfo(lease)));
        }

        boolean disableTransparentFallback = serverConfig.disableTransparentFallbackToOtherRegion();

        if (!disableTransparentFallback) {
            Applications allAppsInLocalRegion = getApplications(false);

            for (RemoteRegionRegistry remoteRegistry : this.regionNameVSRemoteRegistry.values()) {
                Applications applications = remoteRegistry.getApplicationDeltas();
                for (Application application : applications.getRegisteredApplications()) {
                    Application appInLocalRegistry =
                            allAppsInLocalRegion.getRegisteredApplications(application.getName());
                    if (appInLocalRegistry == null) {
                        apps.addApplication(application);
                    }
                }
            }
        }

        Applications allApps = getApplications(!disableTransparentFallback);
        //同步增量数据与全量数据的hashcode
        apps.setAppsHashCode(allApps.getReconcileHashCode());
        return apps;
    } finally {
        write.unlock();
    }
}
```



清除  最近租约变更记录队列  里3分钟没有变更的实例信息  默认每30s执行一次

```java
private TimerTask getDeltaRetentionTask() {
    return new TimerTask() {

        @Override
        public void run() {
            Iterator<RecentlyChangedItem> it = recentlyChangedQueue.iterator();
            while (it.hasNext()) {
                if (it.next().getLastUpdateTime() <
                				//serverConfig.getRetentionTimeInMSInDeltaQueue()：默认180s
                        System.currentTimeMillis() - serverConfig.getRetentionTimeInMSInDeltaQueue()) {
                    it.remove();
                } else {
                    break;
                }
            }
        }

    };
}
```





## 改缓存

**只读缓存**的数据只能来源于读写缓存 而且没有提供手动更新的api，

只读缓存：1、通过定时任务去更新（初始化只读缓存时注册了一个定时任务） 默认每30s执行一次 同步读写缓存的数据到只读缓存

2、从只读缓存获取没有 从读写缓存获取到后 会向只读缓存放一份（invalidateCache()只会改读写缓存，不会改只读缓存，此时只读缓存只能依赖定时任务更新）

定时更新 会导致数据不一致



**读写缓存**的数据修改就是 如果缓存中不包含key对应的记录，就会触发监听器去真实数据中加载数据 然后放到读写缓存中

```java
private TimerTask getCacheUpdateTask() {
    return new TimerTask() {
        @Override
        public void run() {
            logger.debug("Updating the client cache from response cache");
            //遍历只读缓存
            for (Key key : readOnlyCacheMap.keySet()) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Updating the client cache from response cache for key : {} {} {} {}",
                            key.getEntityType(), key.getName(), key.getVersion(), key.getType());
                }
                try {
                    CurrentRequestVersion.set(key.getVersion());
                    Value cacheValue = readWriteCacheMap.get(key);
                    Value currentCacheValue = readOnlyCacheMap.get(key);
                    //如果同一个key 只读缓存对应的value和读写缓存不一样 就用读写缓存的数据覆盖只读缓存的数据
                    if (cacheValue != currentCacheValue) {
                        readOnlyCacheMap.put(key, cacheValue);
                    }
                } catch (Throwable th) {
                    logger.error("Error while updating the client cache from response cache for key {}", key.toStringCompact(), th);
                }
            }
        }
    };
}
```

**服务注册、服务下架、服务剔除** 都会调用invalidateCache()更新**读写缓存 ** 此时是已经修改了真实数据（修改真实数据需要加锁） 再更新读写缓存

com.netflix.eureka.registry.AbstractInstanceRegistry#invalidateCache()：使某些key对应的缓存无效

```java
private void invalidateCache(String appName, @Nullable String vipAddress, @Nullable String secureVipAddress) {
    // invalidate cache
    responseCache.invalidate(appName, vipAddress, secureVipAddress);
}
```





## 客户端服务发现数据延迟

理论上最大的读数据延迟： 定时任务30s（更新只读缓存） + 客户端缓存30s + ribbon缓存30s



有客户端挂掉时的延迟

90s * 2                                                                                                									60s                                              30s                    30s

清理未续约节点超时时间，默认90s（*2是因为eureka的租债器类那个bug）         服务剔除任务间隔          客户端缓存           ribbon缓存

ribbon缓存：ribbon默认每30s（从eureka client缓存）更新使用的服务注册信息，只保存状态为 UP 的服务。







# 客户端增量获取 全量获取



增量入口：com.netflix.eureka.resources.ApplicationsResource#getContainerDifferential

全量入口：com.netflix.eureka.resources.ApplicationsResource#getContainers



无论是全量获取还是增量获取 最后都会调用com.netflix.eureka.registry.ResponseCache#getGZIP方法















