springcloud(原理)



# Eureka

Eureka是AP：保证可用性和分区容错性

Zookeeper是CP，保证强一致性和分区容错性



Eureka客户端在启动时，首先会创建一个定时任务，定时向服务端发送心跳信息，服务端会对客户端心跳做出响应，如果响应状态码为404时，表示服务端没有该客户端的服务信息，那么客户端则会向服务端发送注册请求，注册信息包括服务名、ip、端口、唯一实例ID等信息。



客户端拉取服务端服务信息是通过一个定时任务定时拉取的，每次拉取后刷新本地已保存的信息，需要使用时直接从本地直接获取。

注册中心收到注册信息后会判断是否是其他注册中心同步的信息还是客户端注册的信息，如果是客户端注册的信息，那么他将会将该客户端信息同步到其他注册中心去；否则收到信息后不作任何操作



**Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Eureka Server 会将这些实例保护起来，让这些实例不会过期，**但是在保护期内如果服刚好这个服务提供者非正常下线了，此时服务消费者就会拿到一个无效的服务实例，此时会调用失败，对于这个问题需要服务消费者端要有一些容错机制，如重试，断路器等。



## Ribbon

负载均衡

IRule：根据规则从服务列表中选取一个要访问的服务

IPing：检查服务列表中服务的状态

ServerList：存储服务列表 有静态列表和动态列表

ServerListFilter：该接口允许过滤配置或动态获取的具有所需特性的服务器列表

ServerListUpdater：

IClientConfig：定义各种配置信息，用来初始化ribbon客户端和负载均衡器

ILoadBalancer:



ribbon重试机制



## OpenFeign

`openfeign`则是`spring cloud`在`feign`的基础上支持了`spring mvc`的注解，如`@RequesMapping`、`@GetMapping`、`@PostMapping`等。`openfeign`还实现与`Ribbon`的整合。

声明式调用

动态代理



## openFeign自动装配

1、EnableFeignClients注解

```java
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
    
}
```

给spring容器中导入FeignClientsRegistrar

```java
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar,
      ResourceLoaderAware, EnvironmentAware {
          
    @Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
        //注册默认配置
		registerDefaultConfiguration(metadata, registry);
        
        //注册FeignClients
        /**
        EnableFeignClients 注入 FeignClientsRegistrar，FeignClientsRegistrar 开启自动扫描，将包下 			        @FeignClient 标注的接口包装成 FeignClientFactoryBean 对象，最终通过 Feign.Builder 生成该接口的代理对象。而        Feign.Builder 的默认配置是 FeignClientsConfiguration，是在 FeignAutoConfiguration 自动注入的。
        */
		registerFeignClients(metadata, registry);
	}
   
```



## 











## Zuul

过滤器

pre route post



## Hystrix

线程池隔离

信号量隔离



##spring cloud config



## spring cloud bus

## Spring cloud config 



# spring cloud alibaba

## nacos



## sentinel



## Seata