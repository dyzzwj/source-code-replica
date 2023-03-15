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
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
</dependencies>
```





# 1、eureka client自动配置

spring-cloud-netflix-eureka-client包下的MEAT-INF/spring.factories中导入了自动配置类 org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration



CloudEurekaClient是springcloud对netflix提供的EurekaClient的实现

org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration.RefreshableEurekaClientConfiguration#eurekaClient

```java
@Bean(
    destroyMethod = "shutdown"
)
@ConditionalOnMissingBean(
    value = {EurekaClient.class},
    search = SearchStrategy.CURRENT
)
@org.springframework.cloud.context.config.annotation.RefreshScope
@Lazy
public EurekaClient eurekaClient(ApplicationInfoManager manager, EurekaClientConfig config, EurekaInstanceConfig instance, @Autowired(required = false) HealthCheckHandler healthCheckHandler) {
    ApplicationInfoManager appManager;
    if (AopUtils.isAopProxy(manager)) {
        appManager = (ApplicationInfoManager)ProxyUtils.getTargetObject(manager);
    } else {
        appManager = manager;
    }

    CloudEurekaClient cloudEurekaClient = new CloudEurekaClient(appManager, config, this.optionalArgs, this.context);
    cloudEurekaClient.registerHealthCheck(healthCheckHandler);
    return cloudEurekaClient;
}
```

CloudEurekaClient在构造方法中调用父类的构造方法

```java
public CloudEurekaClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs<?> args, ApplicationEventPublisher publisher) {
    super(applicationInfoManager, config, args);
    this.cacheRefreshedCount = new AtomicLong(0L);
    this.eurekaHttpClient = new AtomicReference();
    this.applicationInfoManager = applicationInfoManager;
    this.publisher = publisher;
    this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class, "eurekaTransport");
    ReflectionUtils.makeAccessible(this.eurekaTransportField);
}
```



com.netflix.discovery.DiscoveryClient#DiscoveryClient

```java
public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args, EndpointRandomizer randomizer) {
    this(applicationInfoManager, config, args, new Provider<BackupRegistry>() {
        //备用eureka server地址
        //只有在eureka client从配置的eureka sever中获取不到注册信息时，会尝试从备用eureka server处获取
        //备用eureka server需单独配置
        //只有在从备用server上获取信息时才会调用get()方法
        private volatile BackupRegistry backupRegistryInstance;

        @Override
        public synchronized BackupRegistry get() {
            if (backupRegistryInstance == null) {
                String backupRegistryClassName = config.getBackupRegistryImpl();
                if (null != backupRegistryClassName) {
                    try {
                        backupRegistryInstance = (BackupRegistry) Class.forName(backupRegistryClassName).newInstance();
                        logger.info("Enabled backup registry of type {}", backupRegistryInstance.getClass());
                    } catch (InstantiationException e) {
                        logger.error("Error instantiating BackupRegistry.", e);
                    } catch (IllegalAccessException e) {
                        logger.error("Error instantiating BackupRegistry.", e);
                    } catch (ClassNotFoundException e) {
                        logger.error("Error instantiating BackupRegistry.", e);
                    }
                }

                if (backupRegistryInstance == null) {
                    logger.warn("Using default backup registry implementation which does not do anything.");
                    backupRegistryInstance = new NotImplementedRegistryImpl();
                }
            }

            return backupRegistryInstance;
        }
    }, randomizer);
}
```

com.netflix.discovery.DiscoveryClient#DiscoveryClient

```java
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
    //....省略部分代码

    //指示此客户端是否从eureka server获取服务注册表信息
 

    //如果服务注册和服务发现都关闭了 直接返回
    if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
        logger.info("Client configured to neither register nor query for data.");
        scheduler = null;
        heartbeatExecutor = null;
        cacheRefreshExecutor = null;
        eurekaTransport = null;
        instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, this.getApplications().size());

        return;  // no need to setup up an network tasks and we are done
    }

    try {
        // default size of 2 - 1 each for heartbeat and cacheRefresh
        //用来管理心跳续约定时任务和缓存刷新定时任务
        scheduler = Executors.newScheduledThreadPool(2,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-%d")
                        .setDaemon(true)
                        .build());

        //心跳续约定时任务
        heartbeatExecutor = new ThreadPoolExecutor(
                1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

        //缓存刷新定时任务  -- 注册表信息缓存
        cacheRefreshExecutor = new ThreadPoolExecutor(
                1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

        eurekaTransport = new EurekaTransport();
        scheduleServerEndpointTask(eurekaTransport, args);

        AzToRegionMapper azToRegionMapper;
        if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
            azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
        } else {
            azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
        }
        if (null != remoteRegionsToFetch.get()) {
            azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
        }
        instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
    } catch (Throwable e) {
        throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
    }
	//如果开启了服务发现并且从eureka server上拉取注册表信息失败的话，就从备用eureka server上拉取服务注册表
    //fetchRegistry()：从eureka server上拉取服务注册表信息
    if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        fetchRegistryFromBackup();
    }

    // call and execute the pre registration handler before all background tasks (inc registration) is started
    if (this.preRegistrationHandler != null) {
        this.preRegistrationHandler.beforeRegistration();
    }

    if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
        try {
            //register()：服务注册
            if (!register() ) {
                throw new IllegalStateException("Registration error at startup. Invalid server response.");
            }
        } catch (Throwable th) {
            logger.error("Registration error at startup: {}", th.getMessage());
            throw new IllegalStateException(th);
        }
    }

    // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
    //初始化服务续约和缓存刷新定时任务
    initScheduledTasks();

    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register timers", e);
    }
    //省略部分代码......
}
```

## 1.1initScheduledTasks:初始化心跳续约和服务发现（缓存刷新）定时任务

com.netflix.discovery.DiscoveryClient#initScheduledTasks()：初始化服务续约和缓存刷新定时任务

```java
private void initScheduledTasks() {
    //允许服务发现
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
        int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
        int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        //定时刷新缓存（服务发现）任务 CacheRefreshThread
        scheduler.schedule(
                new TimedSupervisorTask(
                        "cacheRefresh",
                        scheduler,
                        cacheRefreshExecutor,
                        registryFetchIntervalSeconds,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new CacheRefreshThread()
                ),
                registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }

    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

        //心跳续约任务 HeartbeatThread
        scheduler.schedule(
                new TimedSupervisorTask(
                        "heartbeat",
                        scheduler,
                        heartbeatExecutor,
                        renewalIntervalInSecs,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new HeartbeatThread()
                ),
                renewalIntervalInSecs, TimeUnit.SECONDS);

        //实例化InstanceInfoReplicator对象
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize
		
        //监听器 用来监听作为eureka client的自身的状态变化
        statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
            @Override
            public String getId() {
                return "statusChangeListener";
            }

            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {
                if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                        InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                    // log at warn level if DOWN was involved
                    logger.warn("Saw local status change event {}", statusChangeEvent);
                } else {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                }
                //状态变化时notify方法会被执行，此时上报最新状态到Eureka server
                instanceInfoReplicator.onDemandUpdate();
            }
        };

        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            //注册监听器
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }
		    //服务注册
        instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```

### 1.1.1心跳续约任务类

默认每30s执行一次

```java
private class HeartbeatThread implements Runnable {

    public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}
```

com.netflix.discovery.DiscoveryClient#renew()：

```java
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        //发送心跳续约请求
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        //服务端找不到该实例 就进行服务注册
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            //进行服务注册
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```



### 1.1.2服务发现任务类

服务发现即eureka client向eureka server拉取服务注册表信息 分为全量拉取和增量拉取

默认30s执行一次

```java
class CacheRefreshThread implements Runnable {
    public void run() {
        refreshRegistry();
    }
}
```

com.netflix.discovery.DiscoveryClient#refreshRegistry()

```java
void refreshRegistry() {
    try {
        //云服务相关的 略过
        
		
        //服务发现 分为全量拉取和部分拉取
        boolean success = fetchRegistry(remoteRegionsModified);
        if (success) {
            //注册信息的应用实例数
            registrySize = localRegionApps.get().size();
            //设置 最后获取注册信息时间
            lastSuccessfulRegistryFetchTimestamp = System.currentTimeMillis();
        }
    } catch (Throwable e) {
        logger.error("Cannot fetch registry from server", e);
    }
}
```

com.netflix.discovery.DiscoveryClient#fetchRegistry

```java
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    //forceFullRegistryFetch：是否强制全量拉取
    Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

    try {
    
        //拿到本地的eureka注册表信息缓存
        Applications applications = getApplications();
		    //如果禁止了增量获取 或 vip地址不为空 或强制使用全量拉取 或本地缓存为空 或本地缓存的注册表信息为空 
        if (clientConfig.shouldDisableDelta()
                || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                || forceFullRegistryFetch
                || (applications == null)
                || (applications.getRegisteredApplications().size() == 0)
                || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
        {   
            //全量拉取
            getAndStoreFullRegistry();
        } else {
            //增量拉取
            getAndUpdateDelta(applications);
        }
        //拿到服务端返回的hashcode 设置给本地缓存
        //该变量用于校验增量获取的注册信息和 Eureka-Server 全量的注册信息是否一致
        applications.setAppsHashCode(applications.getReconcileHashCode());
        logTotalInstances();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
        return false;
    } finally {
        if (tracer != null) {
            tracer.stop();
        }
    }

    //触发 CacheRefreshedEvent 事件。目前 Eureka 未提供默认的该事件监听器
  	//ribbon提供了一个ServerListUpdater（EurekaNotificationServerListUpdater）会监听CacheRefreshedEvent事件，更新ribbon本地的ServerList
    //你可以实现自定义的事件监听器监听 CacheRefreshedEvent 事件，以达到持久化最新的注册信息到存储器( 例如，本地文件 )，通过这样的方式，配合实现 BackupRegistry 接口读取存储器。 
    onCacheRefreshed();

    // Update remote status based on refreshed data held in the cache
    //更新本地缓存的的当前应用实例在eureka server的状态
    updateInstanceRemoteStatus();

    // registry was fetched successfully, so return true
    return true;
}
```



### 1.1.3全量拉取和增量拉取

**服务发现即eureka client向eureka server拉取服务注册表信息 分为全量拉取和增量拉取**

eureka client启动时，首先进行一次全量获取 将注册

全量拉取：拉取eureka server上所有的注册表信息

com.netflix.discovery.DiscoveryClient#getAndStoreFullRegistry

```java
private void getAndStoreFullRegistry() throws Throwable {
    long currentUpdateGeneration = fetchRegistryGeneration.get();

    logger.info("Getting all instance registry info from the eureka server");

    Applications apps = null;
    //clientConfig.getRegistryRefreshSingleVipAddress()：当前eureka客户端是否只对某些服务的注册信息感兴趣
    EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null          						  //获取所有服务的注册信息
            ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
              //获取某几个服务的注册信息
            : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        apps = httpResponse.getEntity();
    }
    logger.info("The response status is {}", httpResponse.getStatusCode());

    if (apps == null) {
        logger.error("The application is null for some reason. Not storing this information");
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        localRegionApps.set(this.filterAndShuffle(apps));
        logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
    } else {
        logger.warn("Not updating applications as another thread is updating it already");
    }
}
```

增量拉取：拉取eureka server上最近（默认三分钟之内）更新的注册表信息

```java
private void getAndUpdateDelta(Applications applications) throws Throwable {
    long currentUpdateGeneration = fetchRegistryGeneration.get();

    Applications delta = null;
    EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        delta = httpResponse.getEntity();
    }

    if (delta == null) {
        //如果服务端返回的增量信息为空 就进行全量拉取
        getAndStoreFullRegistry();
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        logger.debug("Got delta update with apps hashcode {}", delta.getAppsHashCode());
        String reconcileHashCode = "";
        if (fetchRegistryUpdateLock.tryLock()) {
            try {
                //同步增量数据到本地
                updateDelta(delta);
                //拿到本地缓存加上获取到的增量数据的hashcode
                reconcileHashCode = getReconcileHashCode(applications);
            } finally {
                fetchRegistryUpdateLock.unlock();
            }
        } else {
            logger.warn("Cannot acquire update lock, aborting getAndUpdateDelta");
        }
        //比较本地缓存和获取到的增量数据的hashcode   和 服务端返回的全部数据的hashcode
        //如果 本地缓存 + 增量数据  的hashcode = 服务端返回的全部数据的hashcoe  就说明数据同步正确 
        //反之 则说明本地 和 服务端 注册表数据  不一致    就进行一次全量拉取
        if (!reconcileHashCode.equals(delta.getAppsHashCode()) || clientConfig.shouldLogDeltaDiff()) {
            //全量拉取
            reconcileAndLogDifference(delta, reconcileHashCode);  // this makes a remoteCall
        }
    } else {
        logger.warn("Not updating application delta as another thread is updating it already");
        logger.debug("Ignoring delta update with apps hashcode {}, as another thread is updating it already", delta.getAppsHashCode());
    }
}
```

com.netflix.discovery.DiscoveryClient#updateDelta()：同步增量数据

```java
private void updateDelta(Applications delta) {
    int deltaCount = 0;
    //delta：增量数据
    for (Application app : delta.getRegisteredApplications()) {
        for (InstanceInfo instance : app.getInstances()) {
            Applications applications = getApplications();
            //云服务 略过
            String instanceRegion = instanceRegionChecker.getInstanceRegion(instance);
            if (!instanceRegionChecker.isLocalRegion(instanceRegion)) {
                //本地缓存
                Applications remoteApps = remoteRegionVsApps.get(instanceRegion);
                if (null == remoteApps) {
                    remoteApps = new Applications();
                    remoteRegionVsApps.put(instanceRegion, remoteApps);
                }
                applications = remoteApps;
            }

            ++deltaCount;
            //增量数据对应的操作 添加？删除？修改？
            //根据不同的操作类型对本地缓存执行对应的操作
            if (ActionType.ADDED.equals(instance.getActionType())) {
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp == null) {
                    applications.addApplication(app);
                }
                logger.debug("Added instance {} to the existing apps in region {}", instance.getId(), instanceRegion);
                applications.getRegisteredApplications(instance.getAppName()).addInstance(instance);
            } else if (ActionType.MODIFIED.equals(instance.getActionType())) {
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp == null) {
                    applications.addApplication(app);
                }
                logger.debug("Modified instance {} to the existing apps ", instance.getId());

                applications.getRegisteredApplications(instance.getAppName()).addInstance(instance);

            } else if (ActionType.DELETED.equals(instance.getActionType())) {
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp != null) {
                    logger.debug("Deleted instance {} to the existing apps ", instance.getId());
                    existingApp.removeInstance(instance);
                    /*
                     * We find all instance list from application(The status of instance status is not only the status is UP but also other status)
                     * if instance list is empty, we remove the application.
                     */
                    if (existingApp.getInstancesAsIsFromEureka().isEmpty()) {
                        applications.removeApplication(existingApp);
                    }
                }
            }
        }
    }
    logger.debug("The total number of instances fetched by the delta processor : {}", deltaCount);
	
    //版本号
    getApplications().setVersion(delta.getVersion());
    getApplications().shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());

    for (Applications applications : remoteRegionVsApps.values()) {
        applications.setVersion(delta.getVersion());
        applications.shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());
    }
}
```

### 1.1.4vip地址

vip地址：当前eureka client是否对单一服务的注册信息感兴趣

可以通过配置 指定用户微服务只拉取订单微服务的注册信息 不拉取物流服务的注册信息

# 2、服务注册

com.netflix.discovery.DiscoveryClient#register();

在DiscoveryClient的构造方法中会进行一次服务注册

```java
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
        //发送注册请求
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
        throw e;
    }
    if (logger.isInfoEnabled()) {
        logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
    }
    return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
}
```



com.netflix.discovery.shared.transport.jersey.AbstractJerseyEurekaHttpClient#register

使用jerseyclient发起http请求，进行服务注册

http://localhost:8080/eureka/apps/微服务名

```java
public EurekaHttpResponse<Void> register(InstanceInfo info) {
    String urlPath = "apps/" + info.getAppName();
    ClientResponse response = null;
    try {
        //发送http请求进行服务注册
        Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
        addExtraHeaders(resourceBuilder);
        response = resourceBuilder
                .header("Accept-Encoding", "gzip")
                .type(MediaType.APPLICATION_JSON_TYPE)
                .accept(MediaType.APPLICATION_JSON)
                .post(ClientResponse.class, info);
        return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
    } finally {
        if (logger.isDebugEnabled()) {
            logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                    response == null ? "N/A" : response.getStatus());
        }
        if (response != null) {
            response.close();
        }
    }
}
```

## 2.1eureka client服务注册的两个条件（定时任务）

- 配置 `eureka.registration.enabled = true`，这是Eureka-Client 向 Eureka-Server 发起注册应用实例的**开关**。
- InstanceInfo 在 Eureka-Client 和 Eureka-Server 数据不一致。



在com.netflix.discovery.DiscoveryClient#initScheduledTasks()的方法中 除了注册心跳续约和服务发现定时任务类，还注册了应用实例状态变更监听器

```java
private void initScheduledTasks() {
    //省略部分代码...
	
    //开启了服务注册
    if (clientConfig.shouldRegisterWithEureka()) {
        //创建应用实例信息复制器
        // InstanceInfo replicator
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize
				//创建应用实例状态变更监听器
        statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
            @Override
            public String getId() {
                return "statusChangeListener";
            }

            //监听StatusChangeEvent事件
            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {
                if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                        InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                    // log at warn level if DOWN was involved
                    logger.warn("Saw local status change event {}", statusChangeEvent);
                } else {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                }
                instanceInfoReplicator.onDemandUpdate();
            }
        };
		
        //注册应用实例状态变更监听器
        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }
		    //开启应用实例信息复制器 定时检查InstanceInfo的状态是否发生变化，若发生变化 发起注册
        instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```



```java
public class ApplicationInfoManager {
  //设置实例状态 并回调监听器
	public synchronized void setInstanceStatus(InstanceStatus status) {
        InstanceStatus next = instanceStatusMapper.map(status);
        if (next == null) {
            return;
        }

        InstanceStatus prev = instanceInfo.setStatus(next);
        if (prev != null) {
            for (StatusChangeListener listener : listeners.values()) {
                try {
                    listener.notify(new StatusChangeEvent(prev, next));
                } catch (Exception e) {
                    logger.warn("failed to notify listener: {}", listener.getId(), e);
                }
            }
        }
    }
    
    public void registerStatusChangeListener(StatusChangeListener listener) {
        listeners.put(listener.getId(), listener);
    }

}
```

com.netflix.discovery.DiscoveryClient#shutdown()和com.netflix.discovery.DiscoveryClient#refreshInstanceInfo里会调用setInstanceStatus()，

- com.netflix.discovery.DiscoveryClient#shutdown()：应用关闭时执行 设置状态为下线

- com.netflix.discovery.DiscoveryClient#refreshInstanceInfo()：关注应用实例信息的 `hostName` 、 `ipAddr` 、 `dataCenterInfo` 属性和renewalIntervalInSecs` 、 `durationInSecs属性的变化。

  

```java
class InstanceInfoReplicator implements Runnable {
    
    private final DiscoveryClient discoveryClient;
    /**
     * 应用实例信息
     */
    private final InstanceInfo instanceInfo;
    /**
     * 定时执行频率，单位：秒
     */
    private final int replicationIntervalSeconds;
    /**
     * 定时执行器
     */
    private final ScheduledExecutorService scheduler;
    /**
     * 定时执行任务的 Future
     */
    private final AtomicReference<Future> scheduledPeriodicRef;
    /**
     * 是否开启调度
     */
    private final AtomicBoolean started;
    
    public void start(int initialDelayMs) {
        if (started.compareAndSet(false, true)) {
            instanceInfo.setIsDirty();  // for initial register
            //提交任务 执行run()
            Future next = scheduler.schedule(this, initialDelayMs, TimeUnit.SECONDS);
            //在 InstanceInfoReplicator#onDemandUpdate() 方法会看到具体用途。
            scheduledPeriodicRef.set(next);
        }
    }
    
    //线程任务逻辑
    //定时检查InstanceInfo的状态是否发生变化，若发生变化 发起注册
    public void run() {
        try {
            //刷新应用实例信息
            discoveryClient.refreshInstanceInfo();
			     //判断应用实例信息是否数据不一致
            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            
            if (dirtyTimestamp != null) {
                //不一致
                //注册
                discoveryClient.register();
                //重置应用实例信息的状态
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
             // 提交任务，并设置该任务的 Future  通过这样的方式，不断循环定时执行任务。
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
    
}
    
```

实例信息

```java
public class InstanceInfo {
  
  private volatile boolean isInstanceInfoDirty = false;
	private volatile Long lastDirtyTimestamp;
    
    public synchronized void setIsDirty() {
        isInstanceInfoDirty = true;
        lastDirtyTimestamp = System.currentTimeMillis();
    }
    
    public synchronized void unsetIsDirty(long unsetDirtyTimestamp) {
        if (lastDirtyTimestamp <= unsetDirtyTimestamp) {
            isInstanceInfoDirty = false;
        } else {
        }
    }
     public synchronized Long isDirtyWithTime() {
        if (isInstanceInfoDirty) {
            return lastDirtyTimestamp;
        } else {
            return null;
        }
    }
}
```











# 3、服务下架



```java
public class EurekaClientAutoConfiguration {
    @Bean(destroyMethod = "shutdown")
    @ConditionalOnMissingBean(value = EurekaClient.class,
          search = SearchStrategy.CURRENT)
    public EurekaClient eurekaClient(ApplicationInfoManager manager,
          EurekaClientConfig config) {
       return new CloudEurekaClient(manager, config, this.optionalArgs,
             this.context);
    }
}
```

destroyMethod指定销毁方法







客户端主动发起服务下架请求 比如：关闭spring容器

com.netflix.discovery.DiscoveryClient#shutdown

```java
//@PreDestroy指定组件的销毁方法 spring容器会在容器关闭时回调单例组件的销毁方法
@PreDestroy
@Override
public synchronized void shutdown() {
    if (isShutdown.compareAndSet(false, true)) {
        logger.info("Shutting down DiscoveryClient ...");

        if (statusChangeListener != null && applicationInfoManager != null) {
            applicationInfoManager.unregisterStatusChangeListener(statusChangeListener.getId());
        }
		    //关闭服务发现和心跳续约定时任务
        cancelScheduledTasks();

        //如果开启了服务注册并且服务停止时需要服务主动进行下架两个开关
        if (applicationInfoManager != null
                && clientConfig.shouldRegisterWithEureka()
                && clientConfig.shouldUnregisterOnShutdown()) {
            applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
            //服务下架
            unregister();
        }

        if (eurekaTransport != null) {
            eurekaTransport.shutdown();
        }

        heartbeatStalenessMonitor.shutdown();
        registryStalenessMonitor.shutdown();

        logger.info("Completed shut down of DiscoveryClient");
    }
}
```

com.netflix.discovery.DiscoveryClient#unregister()：服务下架

```java
void unregister() {
    // It can be null if shouldRegisterWithEureka == false
    if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
        try {
            logger.info("Unregistering ...");
            //发送服务下架请求
            EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
            logger.info(PREFIX + "{} - deregister  status: {}", appPathIdentifier, httpResponse.getStatusCode());
        } catch (Exception e) {
            logger.error(PREFIX + "{} - de-registration failed{}", appPathIdentifier, e.getMessage(), e);
        }
    }
}
```



# 4、心跳续约

定时任务

```java
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        //发送心跳续约请求
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        //服务续约失败 就进行服务注册
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            //进行服务注册
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```





# 5、服务发现

见1.1.2和1.2.3



# 6、@EnableEurekaClient与@@EnableDiscoveryClient区别

- 在注册中心为Eureka时，可以使用@EnableEurekaClient也可以使用@EnableDiscoveryClient
- 注册中心为不为Eureka时，比如consul、zookeeper等，只能使用@EnableDiscoveryClient







