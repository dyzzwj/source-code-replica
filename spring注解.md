spring注解版



# **1、给容器中注册组件**

1、包扫描+组件标注注解（@Controller、@Service、@Repository、@Component） 导入自定义的组件

2、@Bean 导入第三方包里面的组件

3、@Import() 快速给容器中导入组件

①要导入到容器中的组件，容器就会自动注册这个组件，组件id默认是全类名

②ImportSelector：返回需要导入的组件的全类名数组

③ImportBeanDefinitionRegistrar：手动注册bean到容器中

4、使用spring提供的FactoryBean（工厂Bean）

 默认获取到的是工厂bean调用getObject()创建的对象

 要获取工厂bean本身，需要给id前面加一个&

 注：使用FactoryBean注册的bean只有在获取的时候才会创建bean（调用工厂的getObject()）

 

 

## 如何把自己创建的对象放到spring容器中？

1、@Bean

2、FactoryBean

3、applicationContext.getBeanFactory().registerSingleton();//不推荐



# **2、bean的生命周期**

我们可以自定义初始化和销毁方法，容器在bean进行到对应的生命周期的时候来调用我们自定义的初始化和销毁方法

创建（包括属性赋值）：

单实例 容器启动的时候创建对象

多实例 每次获取的时候创建对象

 

BeanPostProcessor.postProcessBeforeInitialization

初始化：对象创建完成，并赋好值，调用初始化方法

BeanPostProcessor.postProcessAfterInitialization

 

销毁：

单实例 容器关闭的时候

多实例 容器不会管理多实例bean，不会调用销毁方法

 

①@Bean(initMethod,destroyMethod);

②通过让bean实现InitializingBean定义初始化逻辑，实现DisposableBean定义销毁逻辑

注：Spring不推荐使用InitializationBean 来调用其初始化方法，因为它不必要地将代码耦合到Spring

③使用JSR250规范提供的注解：spring中提供了CommonAnnotationBeanPostProcessor负责支持@PostConstruct和@PreDestroy

 

@PostConstruct：在bean创建并完成属性赋值后，执行初始化逻辑

@PreDestroy：在容器销毁bean之前执行销毁逻辑

 

④BeanPostProcessor：bean的后置处理器，在bean初始化前后进行一些处理工作

postProcessBeforeInitialization：在bean初始化之前调用

postProcessAfterInitialization：在bean初始化之后调用

 

扩展：BeanPostProcessor原理

ApplicationContextAwareProcessor：注入ApplicationContext对象

AsyncAnnotationBeanPostProcessor：处理@Async注解

InitDestroyAnnotationBeanPostProcessor：

AutowiredAnnotationBeanPostProcessor：处理@Autowired注解







# **3、bean创建流程**

AbstractBeanFactory.getBean()->doGetBean()->createBean()->resolveBeforeInstantiation();doCreateBean()->

 

doGetBean()

doGetBean{

Object sharedInstance = getSingleton();//先从缓存中获取 BeanPostProcessor在此处返回，所有的BeanPostProcessor在registerBeanPostProcessors()就实例化并保存在容器中

If(sharedInstance == nulkl){

createBean();

}

}

 

 

doCreateBean:

{

//属性赋值

populateBean();

//执行初始化方法

initializeBean();

}

 

initializeBean{

invokeAwareMethods();

applyBeanPostProcessorsBeforeInitialization();//自定义bean生命周期的方式③在此处由spring提供的CommonAnnotationBeanPostProcessor进行支持

invokeInitMethods();//自定义bean生命周期的方式①和方式②是在此处被执行的

applyBeanPostProcessorsAfterInitialization();

}





# **4、@Scope**

 

与@Bean配合使用

singleton：单实例 ioc容器启动的时候就创建对象放到ioc容器中，以后每次获取直接到iod容器中拿

prototype：多实例 ioc容器启动的时候不回去创建对象，而是每次获取的时候才会创建对象

 

# 5、**@Value**

使用@Value赋值：

①基本数据类型

②spEL表达式  #{}

③${} 取出配置文件中的值（在运行的环境变量里的值）

 

# **6、@PropertySource**

使用@PropertySoucre读取外部配置文件的k/v保存到运行的环境变量中

 

# 7、**自动装配（@Autowired）**

spring利用依赖注入（DI），完成对IOC容器中各个组件的依赖关系赋值

 

## **1、@Autowired：自动注入**

①默认优先按照类型去容器中找对应的组件；

②如果找到多个类型相同的组件，再将属性的名称作为组件的id去容器中查找，找不到就报错；如果根据类型找不到组件，则直接报错

③@Qualifier 使用@Qualifier指定需要装配的组件id，而不是属性名

④@Autowired自动装配默认一定要完成组件注入（容器中可能没有这个组件），没有就会报错，可以使用@Autowired(required = false)指定不能注入的时候不报错

⑤@Primary，让spring进行自动装配的时候，默认使用首选的Bean，也可以继续使用@Qualifier知道需要装配的bean的名字 优先级： @Qualifier > @Primary

 

## **2、@Resource、@Inject**

spring还支持使用@Resource(JSR250)和@Reject(JSR330) java规范的注解

@Resource：可以实现和@Autowired一样的自动装配功能**默认是按照组件名称**进行装配的。
但不能支持@Primary、@Qualifier和@Autowired的required=false  

@Inject：

需要导入javax.inject的包。支持@Primary，没有@Autowired的required=false

 

AutowiredAnnotationBeanPostProcessor：解析完成自动装配 @Value、@Autowired、@Inject

 

CommonAnnotationBeanPostProcessor：解析@Resource、@PostConstruct、@PreDestroy

 

 

## 3、**@Autowired使用位置**

@Autowired可以使用在构造器、方法、参数、属性位置

①方法

spring容器创建当前对象，就会调用方法，完成赋值

方法使用的参数，从ioc容器中获取，进行注入

@Component
public class Boss {

  private Car car;

  public Car getCar() {
    return car;
  }
  @Autowired

//spring容器创建当前对象，就会调用方法，完成赋值，方法使用的参数，从ioc容器中获取，进行注入
  public void setCar(Car car) {
    this.car = car;
  }
}

②构造器

加在ioc容器中的组件，容器启动会调用无参构造创建对象，再进行初始化赋值

如果组件只有一个有参构造器，这个有参构造器的@Autowired可以省略（不管是标注在构造器上还是参数上）,默认不写@Autowired

@Component
public class Boss {
  //构造器要用的组件，从ioc容器中获取
  @Autowired
  public Boss(Car car){
    this.car = car;
    System.*out*.println("Boss 有参构造...");
  }
  private Car car;

 

 

@Component
public class Boss {

  //构造器要用的组件，从ioc容器中获取

  public Boss(@Autowired Car car){
    this.car = car;
    System.*out*.println("Boss 有参构造...");
  }

  private Car car;

 

 

@Component
public class Boss {

  //构造器要用的组件，从ioc容器中获取

  public Boss(Car car){
    this.car = car;
    System.*out*.println("Boss 有参构造...");
  }

  private Car car;

 



③参数位置

@Bean标注的方法创建对象的时候，方法参数的值从容器中获取，@Autowired可省略，默认不写@Autowired

//@Bean标注的方法创建对象的时候，方法参数的值从容器中获取，@Autowired可省略
@Bean
public Color color(Car car){
  Color color = new Color();
  color.setCar(car);
  return color;
}

 

④属性位置



## 4、**注入spring底层组件**

自定义组件需要使用spring底层组件，如ApplicationContext、BeanFactory

自定义组件只需实现xxxxAware接口，spring在创建自定义组件的时候，会调用接口中对应的方法注入组件：Aware

 

实现原理：
①在初始化Bean的initializeBean方法中执行invokeAwareMethods方法

②通过后置处理器ApplicationContextAwareProcessor来实现的，它实现了BeanPostProcessor接口



# **8、AOP**

 

AOP【动态代理】

指在程序运行期间动态的将某段代码切入到指定方法指定位置进行运行的编程方式

 

1、导入aop模块

Spring aop 【spring-aspects】

 

2、定义一个业务逻辑类（MathCaculate），在业务逻辑类的目标方法（Mathcaculate.div()）运行的时候将日志进行打印（方法执行之前，方法执行之后、方法出现异常...）



3、定义一个日志切面类（LogAspects），切面类里的方法需要动态感知业务逻辑类运行到哪里

通知方法：

前置通知（@Before）：在目标方法运行之前执行

后置通知（@After）：在目标方法运行结束之后执行（包括异常退出和正常结束）

返回通知（@AfterReturning）:在目标方法正常返回之后执行

异常通知（@AfterThrowing）:在目标方法执行过程中出现异常后执行
	环绕通知（@Around）：动态代理，手动推进目标方法执行（joinPoint.procced()）

 

4、给切面类的目标方法标注何时何地运行（通知注解）

5、将切面类和业务逻辑类（目标方法所在类）都加入到容器中

6、告诉spring哪个类是切面类（给切面类加一个注解@Aspect【告诉spring当前类是一个切面类】）

7、给配置类加@EnableAspectJAutoProxy【开启基于直接的aop模式】

 

@Aspect
public class LogAspect {

  */****
\*   ** 抽取公共的切入点表达式**
\*   ** 1.本类引用 @Before("pointCut()")**
\*   ** 2.外部类引用 @After("com.dyzwj.aop.LogAspect.pointCut()")**
\*   **/**
\*  @Pointcut("execution(public int com.dyzwj.aop.MathCaculate.*(..))")
  public void pointCut(){}
  */****
\*   ** 在目标方法之前切入 切入点表达式（指定在哪个位置切入）**
\*   **/**
\*  @Before("pointCut()")
  public void logStart(JoinPoint joinPoint){
    Object[] joinPointArgs = joinPoint.getArgs();
    System.*out*.println(joinPoint.getSignature() + "运行...参数是：{"+ Arrays.*asList*(joinPointArgs) +"}");

  }
  @After("com.dyzwj.aop.LogAspect.pointCut()")
  public void logEnd(JoinPoint joinPoint){
    System.*out*.println(joinPoint.getSignature() + "结束...");
  }

  */****
\*   ** JoinPoint一定要出现在参数列表的第一位**
\*   *** ***@param\*** *joinPoint**
\*   *** ***@param\*** *result**
\*   **/**
\*  @AfterReturning(value = "pointCut()",returning = "result")
  public void logReturn(JoinPoint joinPoint,Object result){
    System.*out*.println(joinPoint.getSignature().getName() + "返回...结果是：{"+ result +"}");
  }

  @AfterThrowing(value = "pointCut()",throwing = "exception")
  public void logException(JoinPoint joinPoint,Exception exception){
    System.*out*.println(joinPoint.getSignature().getName() + "异常...异常原因：{"+ exception +"}");
  }
}

 

 

总结：

三步：

1、将业务逻辑组件和切面类都加入到容器中，告诉spring哪个是切面类（@Aspect）

2、在切面类上的每一个通知方法上标注通知注解，告诉spring何时何地运行（切入点表达式）

3、开启基于注解的aop模式【@EnableAspectJAutoProxy】



# 9、@Conditional



 

除了自己自定义Condition之外，Spring还提供了很多Condition给我们用

@ConditionalOnClass ： classpath中存在该类时起效 

@ConditionalOnMissingClass ： classpath中不存在该类时起效 

@ConditionalOnBean ： DI容器中存在该类型Bean时起效 

@ConditionalOnMissingBean ： DI容器中不存在该类型Bean时起效 

@ConditionalOnSingleCandidate ： DI容器中该类型Bean只有一个或@Primary的只有一个时起效 

@ConditionalOnExpression ： SpEL表达式结果为true时 

@ConditionalOnProperty ： 参数设置或者值一致时起效 

@ConditionalOnResource ： 指定的文件存在时起效 

@ConditionalOnJndi ： 指定的JNDI存在时起效 

@ConditionalOnJava ： 指定的Java版本存在时起效 

@ConditionalOnWebApplication ： Web应用环境下起效 

@ConditionalOnNotWebApplication ： 非Web应用环境下起效

 

 

 

@Conditional定义

@Retention(RetentionPolicy.RUNTIME)  

@Target(ElementType.TYPE, ElementType.METHOD)  

public @interface Conditional{  

  Class<? extends Condition>[] value();

}  

 

@Conditional注解主要用在以下位置：

1、类级别可以放在注标识有@Component（包含@Configuration）的类上

2、作为一个meta-annotation,组成自定义注解

3、方法级别可以放在标识由@Bean的方法上

 

注：如果一个@Configuration的类标记了@Conditional，所有标识了@Bean的方法和@Import注解导入的相关类将遵从这些条件。





# 10、@Bean



\- value：name属性的别名，在不需要其他属性时使用，也就是说value 就是默认值

\- name：此bean 的名称，或多个名称，主要的bean的名称加别名。如果未指定，则bean的名称是带注解方法的名称。如果指定了，方法的名称就会忽略，如果没有其他属性声明的话，bean的名称和别名可能通过value属性配置

\- autowire ：此注解的方法表示自动装配的类型，默认值为No，默认表示不通过自动装配

\- initMethod：指定bean的初始化方法

\- destoryMethod: 指定容器关闭时单实例bean的销毁方法





## **1、@Scope**

在Spring中对于bean的默认处理都是单例的，我们通过上下文容器.getBean方法拿到bean，并对其进行实例化，这个实例化的过程其实只进行一次，即多次getBean 获取的对象都是同一个对象，也就相当于这个bean的实例在IOC容器中是共享（单例）的，对于所有的bean请求来讲都可以共享此bean。

详见@Scope

 

## **2、@Lazy**

@Lazy : 表明一个bean 是否延迟加载，可以作用在方法上，表示这个方法返回的bean被延迟加载，只有在第一次获取这个类的时才会被创建；可以作用在@Component (或者由@Component 作为原注解) 注释的类上，表明这个类中所有的bean 都被延迟加载。

如果没有@Lazy注释，或者@Lazy 被设置为false，那么该bean 就会被饥饿加载（容器启动的时候就被加载到容器中）；除了上面两种作用域，@Lazy 还可以作用在@Autowired和@Inject注释的属性上，在这种情况下，它将为该字段创建一个惰性代理，作为使用ObjectFactory或Provider的默认方法。

 

## 3、**DependsOn** 

指当前bean所依赖的bean。任何指定的所依赖的bean都能保证在此bean创建之前由IOC容器创建。在bean没有通过属性或构造函数参数显式依赖于另一个bean的情况下很少使用，可能直接使用在任何直接或者间接使用 Component 或者Bean 注解表明的类上。来看一下具体的用法。

## 4、**@Primary** 

指示当多个候选者有资格自动装配依赖项时，应优先考虑bean。此注解在语义上就等同于在Spring XML中定义的bean 元素的primary属性。









