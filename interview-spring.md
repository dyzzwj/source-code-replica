## springmvc执行顺序：

1、将request和response绑定到当前线程

2、检查文件上传

3、根据request获取由HandlerMethod和拦截器链组成的处理器执行器链HandlerExecutionChain

4、根据HandlerMethod获取对应的HandlerAdaptor（适配器模式）

5、执行HandlerInterceptor.preHandle()

6、使用HandlerAdaptor执行HandlerMethod，返回ModelAndView

- 参数解析器
- 方法返回值处理器

7、执行HandlerInterceptor.postHandler()方法，此时response body已经写出去了

8、根据ModelAndView处理分发结果，视图渲染

9、执行HandlerInterceptor.afterCompletion()方法



## 核心概念

IOC（控制反转、依赖注入）和AOP（面向切面）

**IOC：创建和查找依赖对象的控制权交给了容器，由容器进行注入组合对象**



**AOP：把与核心业务逻辑无关的代码全部抽取出来，放置到某个地方集中管理，然后在具体运行时，再由容器动态织入这些共有代码**



##spring中的设计模式

- 工厂模式：BeanFactory就是简单工厂模式的体现，用来创建对象的实例；
- 单例模式：Bean默认为单例模式。
- 代理模式：Spring的AOP功能用到了JDK的动态代理和CGLIB字节码生成技术；
- 模板方法：用来解决代码重复的问题。比如. RestTemplate, JmsTemplate, JpaTemplate。
- 观察者模式：定义对象键一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知被制动更新，如Spring中listener的实现–ApplicationListener。
- 策略模式





## BeanFactory和FactoryBean 







## BeanFactory和ApplicationContext

BeanFactory：是Spring里面最底层的接口，包含了各种Bean的定义，读取bean配置文档，管理bean的加载、实例化，控制bean的生命周期，维护bean之间的依赖关系。

ApplicationContext接口作为BeanFactory的派生，除了提供BeanFactory所具有的功能外，还提供了更完整的框架功能：

- 继承MessageSource，因此支持国际化。
- 统一的资源文件访问方式。
- 提供在监听器中注册bean的事件。
- 同时加载多个配置文件。
- 载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层



## 依赖注入

构造器注入

setter方法注入

属性注入



## 循环依赖

- 只支持单例，不支持多例

- 只支持属性和setter循环依赖，不支持构造器循环依赖



A中注入B，B中注入A

假如先进行A的生命周期，把A添加到正在被创建的bean集合中，创建完A对象后，为A初始化一个具有提前完成对象生命周期的工厂（此时工厂并没有执行）并把工厂添加到三级缓存中，在对A进行属性赋值时，需要注入B，发现容器中还没有B，就去进行B的生命周期，创建完B对象后，对B对象进行属性赋值，需要注入A，容器中此时还没有A，（此时对象A还没有完成生命周期），就去进行A的生命周期，先根据A的beanName尝试从一级缓存（单例池）中获取，发现单例池中没有并且A在正在被创建的bean集合中，就根据A的beanName去二级缓存中获取，二级缓存中也没有并且允许提前引用，就根据A的beanName去三级缓存里获取单例工厂，A的单例工厂是存在的，就调用单例工厂提前完成A的生命周期(aop)，

```
AbstractAutoProxyCreator#getEarlyBeanReference() : 提前完成bean的aop，把beanName添加到earlyProxyReferences集合中
  AbstractAutoProxyCreator#postProcessAfterInitialization()中判断beanName不在earlyProxyReferences集合中，才会去进行代理
```

然后把完全体的A放到二级缓存，把A的对象工厂从三级缓存中移除。

B的生命周期完成之后，继续A的属性赋值，然后执行A的初始化方法，最后调用AbstractAutoProxyCreator#postProcessAfterInitialization判断是否需要aop（此时不会进行aop了）





#### Spring循环依赖三级缓存是否可以去掉第三级缓存？

**如果要使用`二级缓存`解决`循环依赖`，意味着Bean在`构造`完后就要创建`代理对象`，而不是在属性赋值和初始化后在创建代理对象**

| 说明                  |                                                              |
| --------------------- | ------------------------------------------------------------ |
| singletonObjects      | 第一级缓存，存放可用的`成品Bean`。                           |
| earlySingletonObjects | 第二级缓存，存放`半成品的Bean`，`半成品的Bean`是已创建对象并完成代理（如需要），但是未注入属性和初始化。 |
| singletonFactories    | 第三级缓存，存的是`Bean工厂对象`，用来生成`半成品的Bean`并放入到二级缓存中。用以解决循环依赖。 |

我们假设现在有这样的场景 AService 依赖 BService，BService 依赖 AService，BService依赖CService,CService依赖AService。

1、如果AService未被代理，那么可以不要二级缓存也能是现在循环依赖

2、如果AService被代理，那么必须使用三级缓存

BService注入AService，调用singletonFactory.getObject()生成AService的代理对象，放入二级缓存

CService注入AService，如果没有二级缓存，那么还会再调用一次singletonFactory.getObject()又生成新的AService代理对象，这样，AService就不是单例的。







## 给容器中注册组件

- @ComponentScan + @Component

- @Bean + @Configuration

- @Import快速导入

  ​	1、@Import(要导入到容器中的组件)：容器自动注册这个组件，组件id默认是全类名

  ​	2、@Import(ImportSelector)：ImportSelector 返回需要导入的组件的全类名数组

  ​	3、@Import(ImportBeanDefinitionRegistrar)：ImportBeanDefinitionRegistrar 手动注册bean到容器中

- FactoryBean





## BeanPostProcessor的执行顺序

第一次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean()

```java
//为aop做准备
//实际执行 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);

```

第二次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean()

```java
//推断构造方法
//实际执行SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()
instanceWrapper = createBeanInstance(beanName, mbd, args);
```

第三次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean()

```java
//先找到需要注入的属性信息(@Autowired、@Value、@Resource)并缓存 以便后面注入时使用
//MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()
applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
```

第四次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean()

```java
//getEarlyBeanReference():判断是否需要aop 为循环依赖提前暴露工厂 
//实际执行 SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

第五次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean()

```java
//是否进行属性注入 postProcessAfterInstantiation()返回false == 不进行属性注入
InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
```

第六次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean()

```java
//属性注入
InstantiationAwareBeanPostProcessor.postProcessPropertyValues()
```

第七次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean()

```java
//BeanPostProcessor.postProcessBeforeInitialization  处理jsr规范提供的声明周期注解
//CommonAnnotationBeanPostProcessor在此处执行,处理@PostConstruct @PreDestory注解
wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
//spring的声明周期方法在之后执行，但是是直接写在代码里了，没有通过BeanPostProcessor进行调用
```

```java
/**
 * 调用spring的生命周期方法
 * 1、实现InitializingBean
 * 2、@Bean(init = "",destory = "")
 */
```



第八次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean()

```java
//BeanPostProcessor.postProcessAfterInitialization
//aop实现：AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator
wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
```







推断构造方法

为循环依赖做准备

属性注入

初始化方法(@PostConstruct、InitializingBean、Bean(initMethod))

aop









## bean的生命周期

创建（包括属性赋值）：

- ​	单实例 容器启动的时候创建对象
- ​	多实例 每次获取的时候创建对象

 

BeanPostProcessor.postProcessBeforeInitialization

初始化：对象创建完成，并赋好值，调用初始化方法

BeanPostProcessor.postProcessAfterInitialization

 

销毁：

- ​	单实例 容器关闭的时候

- ​	多实例 容器不会管理多实例bean，不会调用销毁方法

  

①@Bean(initMethod,destroyMethod);

②通过让bean实现InitializingBean定义初始化逻辑，实现DisposableBean定义销毁逻辑（代码有侵入）

​	注：Spring不推荐使用InitializationBean 来调用其初始化方法，因为它不必要地将代码耦合到Spring

③使用JSR250规范提供的注解：spring中提供了CommonAnnotationBeanPostProcessor负责支持@PostConstruct和@PreDestroy

​	@PostConstruct：在bean创建并完成属性赋值后，执行初始化逻辑

​	@PreDestroy：在容器销毁bean之前执行销毁逻辑

④BeanPostProcessor：bean的后置处理器，在bean初始化前后进行一些处理工作

​	postProcessBeforeInitialization：在bean初始化之前调用

​	postProcessAfterInitialization：在bean初始化之后调用





初始化顺序：

@PostConstruct

InitializingBean

@Bean(initMethod)





## Spring自动装配方式

- no：默认的方式是不进行自动装配的，通过手工设置ref属性来进行装配bean。

- byName：通过bean的名称进行自动装配，如果一个bean的 property 与另一bean 的name 相同，就进行自动装配。

- byType：通过参数的数据类型进行自动装配。

- constructor：利用构造函数进行装配，并且构造函数的参数通过byType进行装配。

  

## @Autowired

①默认优先按照类型去容器中找对应的组件；

**②如果根据类型找到多个类型相同的组件，再将属性的名称作为组件的id去容器中查找，如果根据属性名称找不到，则报错；如果根据类型找不到组件，则报错**

③@Qualifier 使用@Qualifier指定需要装配的组件id，而不是属性名

④@Autowired自动装配默认一定要完成组件注入（容器中可能没有这个组件），没有就会报错，可以使用@Autowired(required = false)

⑤@Primary，让spring进行自动装配的时候，默认使用首选的Bean，也可以继续使用@Qualifier知道需要装配的bean的名字 

优先级： @Qualifier > @Primary





## spring aop



`AbstractAutoProxyCreator`



- jdk动态代理
- cglib代理

将增强逻辑封装成拦截器链







### spring切面5种通知类型

前置通知

后置通知

环绕通知

返回通知

异常通知





## spring事务

`TransactionInterceptor`



### spring事务传播行为

**支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRED：** 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **TransactionDefinition.PROPAGATION_SUPPORTS：** 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **TransactionDefinition.PROPAGATION_MANDATORY：** 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

**不支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRES_NEW：** 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NOT_SUPPORTED：** 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NEVER：** 以非事务方式运行，如果当前存在事务，则抛出异常。

**其他情况：**

- **TransactionDefinition.PROPAGATION_NESTED：** 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。



## spring事物失效场景

1、数据库引擎不支持事务

2、bean没有被 Spring 管理

3、方法不是 public 的

4、方法用final或static修饰

final导致事物失效：spring事务底层使用了aop，也就是通过jdk动态代理或者cglib，帮我们生成了代理类，在代理类中实现的事务功能。但如果某个方法用final修饰了，那么在它的代理类中，就无法重写该方法，而添加事务功能。

5、方法内部调用

6、多线程调用

7、设置错误的传播行为

8、try catch

9、抛出非RuntimeException异常

spring事务，默认情况下只会回滚`RuntimeException`（运行时异常）和`Error`（错误），对于普通的Exception（非运行时异常），它不会回滚。

10、自定义了rollbackFor



## spring监听器

- 实现ApplicationListener接口并添加到容器中 由ApplicationListenerDetector后置处理器处理，添加到监听器列表
- 方法上加@EventListener注解   由EventListenerMethodProcessor处理（bean工厂的后置处理器）



事件多播器派发事件时，会根据事件类型找监听该事件的监听器（事件泛型）

