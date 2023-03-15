# 核心组件 

Route



Predicate



Filter



yaml文件里以spring.cloud.gateway开头的配置绑定到GateWayProperties上    

RouteDefinition

FilterDefinition

PredicateDefinition



GatewayProperties



```java
@ConfigurationProperties("spring.cloud.gateway")
@Validated
public class GatewayProperties {

   /**
    * List of Routes
    */
   @NotNull
   @Valid
   private List<RouteDefinition> routes = new ArrayList<>();

   /**
    * List of filter definitions that are applied to every route.
    */
    //这里的filter定义会被应用到每一个route上
   private List<FilterDefinition> defaultFilters = new ArrayList<>();
    
}
```



RoutePredicateFactory 是所有 predicate factory 的顶级接口，职责就是生产 Predicate

GatewayFilterFactory 职责就是生产 GatewayFilter。

RouteDefinitionLocator：获取RouteDefinition

```
RouteDefinitionRepository
CompositeRouteDefinitionLocator
PropertiesRouteDefinitionLocator
DiscoveryClientRouteDefinitionLocator
```





org.springframework.cloud.gateway.route.RouteDefinitionRouteLocator#combinePredicates

PredicateDefinition 对象转换成 AfterRoutePredicateFactory.Config 对象 

RouteLocator ：获取Route











RouteDefinitionRouteLocator 是RouteLocator的实现类：从RouteDefinitionLocator获取RouteDefinition 转换成Route



CompositeRouteLocator：组合多种 RouteLocator 的实现类，为 RoutePredicateHandlerMapping 提供统一入口访问路由



RoutePredicateFactory：路由谓语工厂





DispatcherHandler   根究路径匹配HandlerMapping

RoutePredicateHandlerMapping   根据predicate匹配Route 返回FilteringWebHandler

FilteringWebHandler 创建DefaultGatewayFilterChain





# gateway请求处理流程



1、DispatcherHandler.handle()  

2、遍历所有的HandlerMapping

3、调用handleMapping的getHandler方法   RoutePredicateHandlerMapping#getHandlerInternal

4、匹配Route RoutePredicateHandlerMapping.lookupRoute() 遍历所有Route，调用Route的Predicate的进行匹配，成功匹配到一个即返回      给ServerWebExchange添加一个属性 GATEWAY_ROUTE_ATTR ，值为 匹配的 Route                       最终返回给DispatcherHandler一个FilteringWebHandler

5、DispatcherHandler遍历HandlerMapping返回的Handler，遍历所有的HandlerAdapter，调用support方法判断是否支持当前Handler，支持的话调用HandlerAdapter.handle()方法调用Handler

其中SimpleHandlerAdapter支持FilteringWebHandler 由SimpleHandlerAdapter调用FilteringWebHandler 的handler方法

6、 FilteringWebHandler.handler()

- 获取在RoutePredicateHandlerMapping.getHandlerInternal中设置的Route
- 拿到该Route的GatewayFilter列表
- 将globalFilters全局过滤器加到GatewayFilter列表中（globalFilters会有一个适配到gatewayFilter的过程  由GatewayFilterAdapter完成）
- 排序
- 使用排完序的GatewayFilter数组创建过滤器链 DefaultGatewayFilterChain
- 执行过滤器器链
- 返回执行结果HandlerResult给SimpleHandlerAdapter



7、SimpleHandlerAdapter拿到HandlerResult返回给DispatcherHandler



8、DispatcherHandler拿到所有支持当前HandlerResult的HandlerResultHandler（调用support()判断是否支持）

遍历HandlerResultHandler 执行handleResult()方法















