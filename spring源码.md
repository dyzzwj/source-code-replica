		spring源码

spring注入模型（需提供setter方法）：

no

by type

by name

constructor





spring自动注入利用java的内省机制完成的



byType和byName就算没有提供属性 但提供了setter方法，setter方法依然会执行

byType和byName是提供了属性的话必须提供该属性的setter方法，通过setter注入的

@Autowired注入提供了属性的话无需提供setter，通过反射注入



spring依赖注入（@Autowired）：无需提供setter方法，使用反射注入 Field.set

构造器注入

setter注入



bean的生命周期 容器的生命周期

lookup



对象和bean的区别



如何将自己创建的对象放入spring容器中（被spring所管理）





FactoryBean和普通bean的区别



```java
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AutowiredConfig.class);
AutowiredConfig autowiredConfig = applicationContext.getBean(AutowiredConfig.class);
System.out.println(autowiredConfig);
```

AutowiredConfig上是否加@Configuration，打印结果不一样

加@Configuration：com.dyzwj.springautowired.config.AutowiredConfig$$EnhancerBySpringCGLIB$$f3fe349@46292372

不加@Configuration：com.dyzwj.springautowired.config.AutowiredConfig@c0b41d6







为什么AnnotationConfigApplicationContext需要手动注册配置类?





代理模式：

①静态代理

继承实现

聚合模式

②动态代理

jdk动态代理（基于聚合）

cglib动态代理





InvocationHandler三大参数



BeanDefinition

autowiredCandidate 是否作为自动装配的候选bean

scan --> parser BeanDefinition  -> validate -> bean实例化（生命周期）

BeanDefinition接口的两个父接口



spring源码编译

gradle配置 build and run 和 run test using ===> idea

在gradle界面找到spring-core，执行complieTestJava

新建一个moudle，看能否使用spring注解如@Configuration

如果不能使用，是因为spring-context没有编译，找到spring-context模块，执行test任务







BeanDefinition

RootBeanDefinition：

①作为父bd

②作为真实bd

③不能作为子bd（RootBeanDefinition.setParentName()会抛异常）

ChildBeanDefinition：

只能作为子bd（必须指定父bd），不能作为一个单独的真实bd

GenericBeanDefinition：可以作为父bd，可以作为子bd，可以作为真实bd



```java
RootBeanDefinition rootBeanDefinition = new RootBeanDefinition();
//属性赋值
rootBeanDefinition.getPropertyValues().addPropertyValue("",)
//构构造器参数赋值
rootBeanDefinition.getConstructorArgumentValues().addGenericArgumentValue();
```





```java
/**
 * BeanDefinition集成体系
 * 1、RootBeanDefinition
 *
 * 2、ChildBeanDefinition 可以继承RootBeanDefinition的数据
 *
 * 3、GenericBeanDefinition xml配置的<bean></bean>
    RootBeanDefinition和ChildBeanDefinition可以用GenericBeanDefinition来代替,效果相同
 *
 * 4、AnnotatedGenericBeanDefinition 以@Configuration注解标记的会解析为AnnotatedGenericBeanDefinition
 *
 * 5、ConfigurationClassBeanDefinition 以@Bean注解标记的会解析为ConfigurationClassBeanDefinition
 *
 * 6、ScannedGenericBeanDefinition 以@Component注解标记的会解析为ScannedGenericBeanDefinition
 *
 *   BeanDefinitionBuilder 可以使用BeanDefinitionBuilder来构建BeanDefinition
 *
 */
```



```
AnnotatedGenericBeanDefinition：applicationContext.register()
ScannedGenericBeanDefinition：@Component
ConfigurationClassBeanDefinition：@Bean
GenericBeanDefinition：xml配置的<bean></bean>
```

bean工程的后置处理器有两种类型，作用是来修改bean工厂

BeanFactoryProcessor

BeanDefinitionRegistryPostProcessor





BeanFactoryProcessor.postProcessBeanFactory()：完成bean扫描后执行，可以修改bean工厂（修改bd、手动注册bean）



BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry()：在bean扫描之前执行



**ConfigurationClassPostProcessor** implements BeanDefinitionRegistryPostProcessor：完成bean扫描，class -> BeanDefinition->add beanDefinitionMap



类扫描 **ClassPathBeanDefinitionScanner.scan()**

ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider

```
TypeFilter
```



```java
public AnnotationConfigApplicationContext() {
   //初始化一个bdReader，读取一个配置类（@Cobfiguration）,生成一个AnnotatedGenericBeanDefinition
   //注册spring内部的后置处理器的beanDefinition
   //ConfigurationClassPostProcessor
   //AutowiredAnnotationBeanPostProcessor
   //CommonAnnotationBeanPostProcessor
   this.reader = new AnnotatedBeanDefinitionReader(this)
   //初始化一个用来动态扫描的scanner
   this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```



**mybatis的mapper扫描**：

@MapperScan ==>  @Import(MapperScannerRegistrar) == > registerBeanDefinitions()中向beanDefinitionMap put了一个MapperScannerConfigurer的beanDefinition

MapperScannerConfigurer









**ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry**





**ConfigurationClassPostProcessor.postProcessBeanFactory()**：判断配置类是否需要进行代理（是否加了@Configuration注解）









# ConfigurationClassPostProcessor

解析@Configuration注解

解析@ComponentScan注解，并完成扫描   

解析@Import注解

解析@Bean注解



1、先执行postProcessBeanDefinitionRegistry





2、再执行postProcessBeanFactory





```java
//AppConfig是配置类（不一定加了@Configuration注解）
AnnotationConfigApplicationContext annotationConfigApplicationContext =
      new AnnotationConfigApplicationContext(AppConfig.class);

System.out.println(annotationConfigApplicationContext.getBean(AppConfig.class));

```

配置类上加@Configuration和不加@Configuration是有区别的

加了@Configuration：System.out.println(annotationConfigApplicationContext.getBean(AppConfig.class));

打印结果：com.dyzwj.config.AppConfig$$EnhancerBySpringCGLIB$$68859aff@f40c25   //**cglib代理对象**



不加@Configuration： System.out.println(annotationConfigApplicationContext.getBean(AppConfig.class));

打印结果：com.dyzwj.config.AppConfig@14f9431



**总结：配置类不一定要加@Configuration，但@Configuration能让spring为配置类生成代理对象**





```java
@ComponentScan("com.dyzwj")
public class AppConfig {
   
   @Bean
   public Demo demo(){
      System.out.println("demo init");
      return new Demo();
   }
   @Bean
   public Test1 test1(){
      demo();
      return new Test1();
   }
   
}
```

对于上述代码，不加@Configuration会有一个问题：

spring在启动过程中，"demo init"会打印两次（即Demo对象会被实例化两次）

配置类上加上@Configuration可以解决该问题



**spring中cglib动态代理实现：**

**BeanMethodInterceptor** implements MethodInterceptor



**spring循环依赖**

1、正在创建



循环引用条件：单例非构造方法注入（setter、@Autowired）

set方法注入可以使用三级缓存解决

构造器注入无法解决循环依赖

设置是否支持循环依赖，默认为true即支持

**annotationConfigApplicationContext.setAllowCircularReferences();**





BeanPostProcessor：干预bean的实例化和初始化过程，不同的时机执行不同类型的BeanPostProcessor

属性注入、生命周期回调、aop、推断构造方法

**三级缓存**      --  解决了循环依赖

```
singletonObjects 一级
singletonFactories 二级
earlySingletonObjects 三级 
```



aop

切点表达式：

execution：支持最小粒度是方法级别（包括返回类型、方法名、参数个数、参数类型 ）

within：支持最小粒度是类级别

this：代理对象

target：目标对象



spring使用jdk代理和cglib代理区别

class A implements I{

}

针对实现了I接口的A类，spring会使用默认的jdk代理，此时生成的代理对象类型是I，而不是A       	I proxy = ac.getBean(I.class);拿到代理对象

注意：spring并不会为A类创建对象并放到容器中，放到容器中的是为A类生成的代理对象（实现了I接口）



class A{

}

针对未实现接口的A类，spring使用Cglib进行代理，此时生成的代理对象类型是A 	A proxy = ac.getBean(A.class);拿到代理对象

注意：spring并不会为A类创建对象并放到容器中，放到容器中的是为A类生成的代理对象（继承了A类）



class A implements I{

}

针对实现了I接口的A类，spring会使用默认的jdk代理，但我们可以强制spring使用Cglib进行代理

此时生成的代理对象继承了A类并实现了I接口，此时如何从容器总拿到代理对象呢



spring代理对象在哪产生的

java合成类、合成方法





推断构造方法：AutowiredAnnotationBeanPostProcessor



```java
public class Test3 {
   Class clazz;

   public Test3(Class clazz){
      System.out.println(clazz.getSimpleName());
   }
}

AnnotationConfigApplicationContext ac =
    new AnnotationConfigApplicationContext();
ac.register(AppConfig.class);
GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
beanDefinition.setBeanClass(Test3.class);
beanDefinition.getConstructorArgumentValues().addGenericArgumentValue("com.dyzwj.Test2");
ac.registerBeanDefinition("test3",beanDefinition);
ac.refresh();
ac.getBean(Test3.class);


```

spring构造器注入

考虑因素：

```java
是否自动注入
构造器参数个数（是否手动提供默认无参构造）
构造器参数类型（接口、实现类）
是否@Autowired（@Autowired的个数）
是否required

程序员是否手动添加构造器参数（程序员手动添加的参数其类型是否需要进行转换 String ==> Class，程序员手动添加的构造器参数类型和构造器形参是否匹配 == TypeConverter)


```





beanDefinition合并





springmvc   5.0.6

手写springmvc

文件路径、类路径







springmvc零xml配置原理

servlet3.0规范

扩展springmvc 实现WebMvcConfigurer



外部tomcat和嵌入式tomcat

tomcat.addContext()：只会去初始化一个context的资源目录 并不会加载web的生命周期

tomcat.addWebapp



springmvc源码


```
处理请求的入口：DispatcherServlet.doDispatch
```

springmvc大概流程：

1、扫描指定路径

2、判断有没有@Controller或@RequestMapping

3、扫描方法

4.判断方法上有没有注解

5、把路径与method绑定



```
/**
 * handlerMapping：会有多个  ？？？  因为与多种注册Controller的方式
 *     1、@Controller + @RequestMapping  --- 常规方式   --  RequestMappingHandlerMapping
 *     2、编写一个类 实现 Controller或HttpRequestHandler接口   ---早期    --  BeanNameUrlHandlerMapping  以		beanName声明的url的方式
 *
 *  每个HandlerMapping都维护一个自己的映射关系集合
 */
 
 
 springboot默认的静态资源映射就是通过编写自定义HandlerMapping实现的  --  四个目录 --- 
 如何响应静态资源 通过response.getWriter().write()写到浏览器 
 
 
 
HandlerMapping
HandlerAdapter




```

springmvc初始化controller流程  --- AbstractHandlerMethodMapping.afterPropertiesSet()

controller调用流程 

参数赋值 HandlerMethodArgumentResolver接口(有优先级)   扩展参数赋值

父子容器

视图渲染  ----  针对不同类型的返回值类型做处理---HandlerMethodReturnValueHandler(接口)

如何解析@ResponseBody     由RequestResponseBodyMethodProcessor来判断@ResponseBody   ----   响应数据----消息转换（HttpMessageConverter）



视图名称解析  --- ViewResolver  接口









spring

第三次调用后置处理器

在此处会先把需要注入的位置（属性或方法）找到并缓存  注意InjectElement。。。的关系





属性注入 @Autowired注入流程   根据类型找到多个，再根据名字

List<I>注入







springboot自动配置原理 @SpringbootApplication -- @EnableAutoConfiguration -- @Import(AutoConfigurationImportSelector.class)

```
AutoConfigurationImportSelector implements ImportSelector  ==> String[] selectImports()方法
selectImports():返回一个String数组，里面包含需要被spring容器加载的类的全路径名

AutoConfigurationImportSelector.selectImports()：会去classpath下找META-INF/spring.factories文件
```



如何开发starter





springboot整合springmvc



DispatcherServlet(的父类FrameworkServlet)实现了ApplicationContextAware接口  注意：只有



**springboot是把DispatcherServlet放到spring容器中**   

注意：只有DispatcherServlet在spring容器中 spring在创建DispatcherServlet的时候才会调用setApplicationContextAware()注入applicationContext





**springmvc是把spring容器放到DispatcherServlet**  (DispatcherServlet并不在spring容器中，但它持有spring容器的引用，可以从容器里拿)

new DispatcherServlet(applicationContext);  不会调用setApplicationContextAware()v 



springboot配置springmvc







索引是排好序的快速查找数据结构

双模糊('%zwj%')可以使用覆盖索引解决



重要的BeanPostProcessor：

CommonAnnotationBeanPostProcessor

处理@Resource

处理@PostContruct、@PreDestroy





AutowiredAnnotationBeanPostProcessor

处理@Autowired

处理InitializingBean、DisposableBean的实现类

处理@Bean指定了initMethod、destroyMethod









spring扩展点

BeanFactoryPostProcessor

BeanDefinitionRegistryPostProcessor

ImportBeanDefinitionRegistrar

BeanPostProcessor：

FactoryBean：





spring循环依赖



场景：

构造器的循环依赖(构造器注入)

Field属性的循环依赖（set注入）







三级缓存

一级：单例池   经历过完整生命周期的bean

二级：early

三级：singletonFactory  对象实例化后立即放入三级缓存  只是一个java对象  还没完成属性赋值及后续的一系列生命周期 





为什么构造器的循环依赖spring无法解决   因为对象根本没有实例化（没有引用） 无法放到三级缓存

