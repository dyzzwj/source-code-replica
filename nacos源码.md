nacos

版本 spring-cloud-alibaba-2.2.0







# nacos注册中心自动配置





## 服务注册自动配置

spring-cloud-alibaba-nacos-discovery模块META-INF/spring.factories中，引入了自动配置类NacosServiceRegistryAutoConfiguration

```java
public class NacosServiceRegistryAutoConfiguration {

  
  //注入NacosServiceRegistry
	@Bean
	public NacosServiceRegistry nacosServiceRegistry(
			NacosDiscoveryProperties nacosDiscoveryProperties) {
		return new NacosServiceRegistry(nacosDiscoveryProperties);
	}
  
  
  //注入NacosAutoServiceRegistration
  @Bean
	@ConditionalOnBean(AutoServiceRegistrationProperties.class)
	public NacosAutoServiceRegistration nacosAutoServiceRegistration(
			NacosServiceRegistry registry,
			AutoServiceRegistrationProperties autoServiceRegistrationProperties,
			NacosRegistration registration) {
    //构造器传入的就是上面注入的NacosServiceRegistry
		return new NacosAutoServiceRegistration(registry,
				autoServiceRegistrationProperties, registration);
	}
  
  
}
```





NacosAutoServiceRegistration继承了AbstractAutoServiceRegistration抽象类（spring-cloud-common模块定义）

```java
public class NacosAutoServiceRegistration
      extends AbstractAutoServiceRegistration<Registration> {
  
}
```



AbstractAutoServiceRegistration实现了ApplicationListener接口，监听WebServerInitializedEvent事件

```java
public abstract class AbstractAutoServiceRegistration<R extends Registration> implements AutoServiceRegistration, ApplicationContextAware, ApplicationListener<WebServerInitializedEvent> {
 
   //Web容器初始化完成触发WebServerInitializedEvent事件
   public void onApplicationEvent(WebServerInitializedEvent event) {
        this.bind(event);
   }
  
  
   public void bind(WebServerInitializedEvent event) {
        ApplicationContext context = event.getApplicationContext();
        if (!(context instanceof ConfigurableWebServerApplicationContext) || !"management".equals(((ConfigurableWebServerApplicationContext)context).getServerNamespace())) {
            this.port.compareAndSet(0, event.getWebServer().getPort());
            this.start();
        }
    }
  
   public void start() {
        if (!this.isEnabled()) {
            if (logger.isDebugEnabled()) {
                logger.debug("Discovery Lifecycle disabled. Not starting");
            }

        } else {
            if (!this.running.get()) {
                this.context.publishEvent(new InstancePreRegisteredEvent(this, this.getRegistration()));
              	//注册
                this.register();
                if (this.shouldRegisterManagement()) {
                    this.registerManagement();
                }

                this.context.publishEvent(new InstanceRegisteredEvent(this, this.getConfiguration()));
                this.running.compareAndSet(false, true);
            }

        }
    }

    protected void register() {
      
      	//实际调用NacosServiceRegistry#register
        this.serviceRegistry.register(this.getRegistration());
    }
   
  
}
```







```java
public class NacosServiceRegistry implements ServiceRegistry<Registration> {

   private static final Logger log = LoggerFactory.getLogger(NacosServiceRegistry.class);

   private final NacosDiscoveryProperties nacosDiscoveryProperties;

   private final NamingService namingService;

   public NacosServiceRegistry(NacosDiscoveryProperties nacosDiscoveryProperties) {
      this.nacosDiscoveryProperties = nacosDiscoveryProperties;
      this.namingService = nacosDiscoveryProperties.namingServiceInstance();
   }

   @Override
   public void register(Registration registration) {

      if (StringUtils.isEmpty(registration.getServiceId())) {
         log.warn("No service to register for nacos client...");
         return;
      }

      String serviceId = registration.getServiceId();
      String group = nacosDiscoveryProperties.getGroup();

      Instance instance = getNacosInstanceFromRegistration(registration);

      try {
         //注册
         namingService.registerInstance(serviceId, group, instance);
         log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
               instance.getIp(), instance.getPort());
      }
      catch (Exception e) {
         log.error("nacos registry, {} register failed...{},", serviceId,
               registration.toString(), e);
         // rethrow a RuntimeException if the registration is failed.
         // issue : https://github.com/alibaba/spring-cloud-alibaba/issues/1132
         rethrowRuntimeException(e);
      }
   }
```







说明：

**Nacos既支持CP，也支持AP  与之对应 Eureka - AP  Zookeeper - CP**





## 服务发现自动配置









## 服务注册

com.alibaba.nacos.naming.controllers.InstanceController#register





nacos支持两种模式

AP:Distro协议  临时节点

CP: Raft协议   持久化到磁盘  持久节点



cp协议的注册表更新（增删改） 

1、判断当前你节点是否是leader

2、如果非leader，将注册请求转发到集群的leader

3、如果是leader，leader节点先更新到本地内存（同步）和磁盘（异步）

4、leader利用CountDownLatch实现raft协议节点数据同步 必须集群半数以上节点 >= (peers.size / 2   + 1) 写入成功 才返回给客户端成功









服务注册表结构

<img src="images/nacos注册表结构.png" alt="nacos注册表结构" style="zoom:50%;" />









服务注册



![nacos服务注册](images/nacos服务注册.png)













# nacos配置中心自动配置

spring-cloud-alibaba-nacos-config模块META-INF/spring.factories中，引入了自动配置类NacosConfigBootstrapConfiguration和NacosConfigAutoConfiguration

```properties
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
com.alibaba.cloud.nacos.NacosConfigBootstrapConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.alibaba.cloud.nacos.NacosConfigAutoConfiguration,\
com.alibaba.cloud.nacos.endpoint.NacosConfigEndpointAutoConfiguration
org.springframework.boot.diagnostics.FailureAnalyzer=\
com.alibaba.cloud.nacos.diagnostics.analyzer.NacosConnectionFailureAnalyzer
org.springframework.boot.env.PropertySourceLoader=\
com.alibaba.cloud.nacos.parser.NacosJsonPropertySourceLoader,\
com.alibaba.cloud.nacos.parser.NacosXmlPropertySourceLoader
```



```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnProperty(
    name = {"spring.cloud.nacos.config.enabled"},
    matchIfMissing = true
)
public class NacosConfigBootstrapConfiguration {
    public NacosConfigBootstrapConfiguration() {
    }

    // spring.cloud.nacos.config配置
    @Bean
    @ConditionalOnMissingBean
    public NacosConfigProperties nacosConfigProperties() {
        return new NacosConfigProperties();
    }

    // 管理ConfigService
    @Bean
    @ConditionalOnMissingBean
    public NacosConfigManager nacosConfigManager(NacosConfigProperties nacosConfigProperties) {
        return new NacosConfigManager(nacosConfigProperties);
    }

    // PropertySourceBootstrapConfiguration会加载NacosPropertySourceLocator提供的配置
    @Bean
    public NacosPropertySourceLocator nacosPropertySourceLocator(NacosConfigManager nacosConfigManager) {
        return new NacosPropertySourceLocator(nacosConfigManager);
    }
}
```

























































更新注册表  copyonwrite





心跳续约

com.alibaba.nacos.naming.controllers.InstanceController#beat

临时节点才有心跳续约

临时节点启动时会注册一个心跳续约定时任务







集群同步

com.alibaba.nacos.naming.controllers.DistroController#onSyncDatum



服务剔除







服务下架

com.alibaba.nacos.naming.controllers.InstanceController#deregister



服务发现

com.alibaba.nacos.naming.controllers.InstanceController#list





naocs配置模型

- **Namspace（Tenant）：命名空间（租户）**，默认命名空间是public。一个命名空间可以包含多个Group，在Nacos源码里有些变量是tenant租户，和命名空间是一个东西。

- **Group：组**，默认分组是DEFAULT_GROUP。一个组可以包含多个dataId。

- **DataId：译为数据id，在nacos中DataId代表一整个配置文件，是配置的最小单位**。



Nacos客户端查询配置

- 优先failover本地配置
- 其次调用服务端查询/v1/cs/configs
- 兜底使用snapshot本地配置，每次成功调用服务端/v1/cs/configs获取配置，都会同步更新snapshot配置



nacos服务端配置存储

- 内存：Nacos每个节点都在内存里缓存了配置，但是**只包含配置的md5**（缓存配置文件太多了），所以**内存级别的配置只能用于比较配置是否发生了变更，只用于客户端长轮询配置等场景**。

- 文件系统：文件系统配置来源于数据库写入的配置。如果是集群启动或mysql单机启动，服务端会以本地文件系统的配置响应客户端查询。

- 数据库：所有写数据都会先写入数据库。只有当以derby数据源（-DembeddedStorage=true）单机（-Dnacos.standalone=true）启动时，客户端的查询配置请求会实时查询derby数据库。







**对于写请求，Nacos会将数据先更新到数据库，之后异步写入所有节点的文件系统并更新内存**。







数据库选择

配置文件主要存储在config_info表中。Nacos有三个配置项与数据源的选择有关：

- application.properties中的spring.datasource.platform配置项，默认为空，可以配置为mysql
- -Dnacos.standalone，true代表单机启动，false代表集群启动，默认false
- -DembeddedStorage，true代表使用嵌入式存储derby数据源，false代表不使用derby数据源，默认false

这块感觉比较乱，通过伪代码的方式理一下。主要是spring.datasource.platform在默认为空的场景下，**满足条件集群启动且-DembeddedStorage=false（默认false），还是会选择mysql数据源**。也就是说，**集群启动，如果没特殊配置，Nacos会使用MySQL数据源**。

```sh
// 指定数据源为mysql，直接返回mysql
if (mysql) {
	return mysql;
}
// 如果单机部署，没指定数据源，使用derby
if (standalone) {
	return derby;
} else {
    // 如果集群部署且指定使用嵌入式存储，使用derby
	if (embeddedStorage) {
		return derby;
	} else {
	    // 集群部署，默认使用mysql存储
		return mysql;
	}
}
```





客户端查询配置，对于derby数据源单机部署，会查询derby数据源中的配置；对于mysql数据源或集群部署，会查询本地文件系统中的配置



配置发布

![nacos配置发布](/Users/sirzheng/king/repo/source-code/images/nacos配置发布.jpg)





**nacos2.0配置中心**

2.x配置中心主要的改动在于**引进长连接代替了短连接长轮询**。

**客户端**改动点：

**改动点一：**

1.x每3000个CacheData，客户端会开启一个**LongPollingRunnable**长轮询任务；2.x每3000个CacheData，客户端会开启一个RpcClient，每个RpcClient与服务端建立一个长连接。

**改动点二：**

客户端增加定时全量拉取配置的逻辑。

在1.x中，Nacos配置中心通过长轮询的方式更新客户端配置，对于客户端来说只有配置推送；

在2.x中支持客户端定时同步配置，所以2.x属于推拉结合的方式。

**拉：\**每5分钟，客户端会对全量CacheData发起配置监听请求\**ConfigBatchListenRequest**，如果配置md5发生变更，会同步收到变更配置项，发起**ConfigQuery**请求查询实时配置。

**推**：服务端配置变更，会发送**ConfigChangeNotifyRequest**请求给与当前节点建立长连接的客户端通知配置变更项。

**服务端**改动点：

**改动点一：**

由于2.x使用长连接代替长轮询，监听请求ConfigBatchListenRequest不会被服务端hold住，会立即返回。服务端只是将监听关系保存在内存中，方便后续通知。

**groupKey和connectionId的映射关系，方便后续通过变更配置项找到对应客户端长连接**；connectionId和groupKey的映射关系，只是为了控制台展示。这些关系保存在服务端的**ConfigChangeListenContext**单例中。

**改动点二**：

对应改动点一，1.x需要通过groupKey找到仍然在进行长轮询的客户端AsyncContext；2.x是通过groupKey找到connectionId，再通过connectionId找到长连接，发送ConfigChangeNotifyRequest通知客户端配置变更。











