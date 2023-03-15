​																						Mybatis整合spring

基于springboot整合mybatis来进行说明 纯注解

使用@MapperScan注解扫描mybatis的mapper接口

```java
//@Import是spring的注解，给容器中注入一个组件
@Import(MapperScannerRegistrar.class)
public @interface MapperScan {


```



@Import(MapperScannerRegistrar.class)给容器中导入MapperScannerRegistrar，这个类实现了ImportBeanDefinitionRegistrar接口，可以在接口定义的方法registerBeanDefinitions()中手动注册bean到容器中

```java
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware {

  private ResourceLoader resourceLoader;

  /**
   * {@inheritDoc}
   */
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    //获取到MapperScan注解的属性值
    AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    //创建路径扫描器，下面的一大段都是将MapperScan 中的属性设置到ClassPathMapperScanner ，做的就是一个set操作
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    // this check is needed in Spring 3.1
    if (resourceLoader != null) {
      //设置资源加载器，作用：扫描指定包下的class文件
      scanner.setResourceLoader(resourceLoader);
    }

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      scanner.setAnnotationClass(annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      scanner.setMarkerInterface(markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
      scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
      scanner.setMapperFactoryBean(BeanUtils.instantiateClass(mapperFactoryBeanClass));
    }

    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));

    List<String> basePackages = new ArrayList<String>();
    for (String pkg : annoAttrs.getStringArray("value")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }
    for (String pkg : annoAttrs.getStringArray("basePackages")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }
    for (Class<?> clazz : annoAttrs.getClassArray("basePackageClasses")) {
      basePackages.add(ClassUtils.getPackageName(clazz));
    }
    //注册过滤器，作用：什么类型的mapper将会留下来
    scanner.registerFilters();
    //扫描指定包
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }
```



ClassPathMapperScanner.doScan()：

```java
public class ClassPathBeanDefinitionScanner extends ClassPathScanningCandidateComponentProvider {
    public Set<BeanDefinitionHolder> doScan(String... basePackages) {
      //调用父类的doScan()：将basePackages下的class都封装成BeanDefinitionHolder，并注册进Spring容器的BeanDefinition
      Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

      if (beanDefinitions.isEmpty()) {
        logger.warn("No MyBatis mapper was found in '" + Arrays.toString(basePackages) + "' package. Please check your configuration.");
      } else {
        //继续对BeanDefinition做一些处理，额外设置一下属性
        processBeanDefinitions(beanDefinitions);
      }

      return beanDefinitions;
   }
```

ClassPathScanningCandidateComponentProvider.doScan()：

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet();
    String[] var3 = basePackages;
    int var4 = basePackages.length;

  	//遍历basePackages进行扫描
    for(int var5 = 0; var5 < var4; ++var5) {
        String basePackage = var3[var5];
      	//找出匹配的类
        Set<BeanDefinition> candidates = this.findCandidateComponents(basePackage);
        Iterator var8 = candidates.iterator();

        while(var8.hasNext()) {
            BeanDefinition candidate = (BeanDefinition)var8.next();
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                this.postProcessBeanDefinition((AbstractBeanDefinition)candidate, beanName);
            }

            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition)candidate);
            }

            if (this.checkCandidate(beanName, candidate)) {
              	//封装成BeanDefinitionHolder对象
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
              	//将BeanDefinition对象注入spring的BeanDefinitionMap中，后续getBean时，就是从BeanDefinitionMap获取对应的BeanDefinition对象，取出其属性进行实例化Bean
                this.registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }

    return beanDefinitions;
}
```



processBeanDefinitions()：

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
  GenericBeanDefinition definition;
  //遍历BeanDefinitionHolder
  for (BeanDefinitionHolder holder : beanDefinitions) {
    //取出BeanDefinitionHolder中的BeanDefinition
    definition = (GenericBeanDefinition) holder.getBeanDefinition();

    if (logger.isDebugEnabled()) {
      logger.debug("Creating MapperFactoryBean with name '" + holder.getBeanName() 
        + "' and '" + definition.getBeanClassName() + "' mapperInterface");
    }

    // 设置definition构造器的输入参数为definition.getBeanClassName()，这里就是Mapper接口
		// 那么在getBean实例化时，通过反射调用构造器实例化时要将这个参数传进去
    //definition.getBeanClassName()这个类型就是MapperFactoryBean.getObject()返回的对象的类型
    definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
    //修改definition对应的class
    //这里是一招偷梁换柱。注意区分：把类交给class管理(@Component等)和把对象(@Bean)交给class管理，前者对象实例化的方式我们不能干预，后者我们可以根据自己的需求实例化对象
    //spring是根据BeanDefinition的beanClass属性进行实例化的（利用反射），所以，BeanDefinition的beanClass是哪个类的Class，spring就实例化该Class对于的bean，这里的beanClass本来应该是Mapper接口，由于mybatis无法干预spring根据BeanDefinition实例化对象的过程，所以只能使用FactoryBean的机制。如果BeanDefinition的beanClass是FactoryBean类型，spring并不会仅仅把FactoryBean加入容器中，最重要的是将FactoryBean.getObject()返回的对象加入到容器中，使用FactoryBean将mapper加入到容器中，mybatis可以控制bean的创建过程，此时，就可以根据自己的需求对bean做一些改造
    
    
    // 看过Spring源码的都知道，getBean()返回的就是BeanDefinitionHolder中beanClass属性对应的实例
    definition.setBeanClass(this.mapperFactoryBean.getClass());
		
    definition.getPropertyValues().add("addToConfig", this.addToConfig);

    boolean explicitFactoryUsed = false;
    if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
      definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
      explicitFactoryUsed = true;
    } else if (this.sqlSessionFactory != null) {
      //往definition设置属性值sqlSessionFactory，那么在MapperFactoryBean实例化后，进行属性赋值时populateBean(beanName, mbd, instanceWrapper);，会通过反射调用sqlSessionFactory的set方法进行赋值
		 //也就是在MapperFactoryBean创建实例后，要调用setSqlSessionFactory方法将sqlSessionFactory注入进MapperFactoryBean实例
      definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
      explicitFactoryUsed = true;
    }

    if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
      if (explicitFactoryUsed) {
        logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
      }
      definition.getPropertyValues().add("sqlSessionTemplate", new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
      explicitFactoryUsed = true;
    } else if (this.sqlSessionTemplate != null) {
      if (explicitFactoryUsed) {
        logger.warn("Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
      }
      definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
      explicitFactoryUsed = true;
    }

    if (!explicitFactoryUsed) {
      if (logger.isDebugEnabled()) {
        logger.debug("Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
      }
      //设置
      definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    }
  }
}
```

instantiate()：通过构造器创建实例

```java
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
   
  	// 没有覆盖，直接使用反射实例化即可
   if (bd.getMethodOverrides().isEmpty()) {
      Constructor<?> constructorToUse;
      synchronized (bd.constructorArgumentLock) {
				 // 获取已经解析的构造函数
         constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
         if (constructorToUse == null) {
           	// 如果为 null，从 class 中解析获取，并设置
            final Class<?> clazz = bd.getBeanClass();
            if (clazz.isInterface()) {
               throw new BeanInstantiationException(clazz, "Specified class is an interface");
            }
            try {
               if (System.getSecurityManager() != null) {
                  constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
                     @Override
                     public Constructor<?> run() throws Exception {
                        return clazz.getDeclaredConstructor((Class[]) null);
                     }
                  });
               }
               else {
                 	//利用反射获取构造器
                  constructorToUse = clazz.getDeclaredConstructor((Class[]) null);
               }
               bd.resolvedConstructorOrFactoryMethod = constructorToUse;
            }
            catch (Throwable ex) {
               throw new BeanInstantiationException(clazz, "No default constructor found", ex);
            }
         }
      }
     	//// 通过BeanUtils直接使用构造器对象实例化bean
      return BeanUtils.instantiateClass(constructorToUse);
   }
   else {
      // 有需要覆盖的方法，使用CGLIB创建子类对象
      return instantiateWithMethodInjection(bd, beanName, owner);
   }
}
```

看到没，是通过**bd.getBeanClass();从**BeanDefinition中获取beanClass属性，然后通过反射实例化Bean，如上，所有的Mapper接口扫描封装成的BeanDefinition的beanClass都设置成了MapperFactoryBean，我们知道在Spring初始化的最后，会获取所有的BeanDefinition，并通过getBean创建所有的实例注入进Spring容器，那么意思就是说，在getBean时，创建的的所有Mapper对象是MapperFactoryBean这个对象了。



MapperFactoryBean是FactoryBean类型的bean

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

  private Class<T> mapperInterface;

  private boolean addToConfig = true;

  public MapperFactoryBean() {
    //intentionally empty 
  }
  
  public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }
```









