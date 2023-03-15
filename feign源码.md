

​																											feign源码

基于Hoxton.SR1版本

```xml
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
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```



声明式服务调用 负载均衡基于ribbon实现  服务调用基于动态代理



# 自动配置

spring-cloud-openfeign-core模块的META-INF/spring.factories文件导入了FeignRibbonClientAutoConfiguration自动配置类

FeignRibbonClientAutoConfiguration配置类给容器中导入了DefaultFeignLoadBalancedConfiguration配置类

```java
@Import({ HttpClientFeignLoadBalancedConfiguration.class,
		OkHttpFeignLoadBalancedConfiguration.class,
		DefaultFeignLoadBalancedConfiguration.class })
public class FeignRibbonClientAutoConfiguration {
  
}
```



DefaultFeignLoadBalancedConfiguration配置类给容器中注入了一个Client类型的bean--LoadBalancerFeignClient

```java
@Configuration(proxyBeanMethods = false)
class DefaultFeignLoadBalancedConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
			SpringClientFactory clientFactory) {
		return new LoadBalancerFeignClient(new Client.Default(null, null), cachingFactory,
				clientFactory);
	}

}
```



使用@EnableFeignClients开启feign，向spring容器中导入了FeignClientsRegistrar

```java
@Import({FeignClientsRegistrar.class})
public @interface EnableFeignClients {
    
}
```

## FeignClientsRegistrar（spring扫描阶段）

FeignClientsRegistrar实现了ImportBeanDefinitionRegistrar接口，spring在初始化ImportBeanDefinitionRegistrar过程中，会调用其registerBeanDefinitions方法，手动注册bean到BeanDefinition

```java
class FeignClientsRegistrar
      implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {
    
    //ImportBeanDefinitionRegistrar：回调registerBeanDefinitions方法
    //ResourceLoaderAware：注入ResourceLoader，读取类路径下的文件
    //EnvironmentAware：注入Environment
  public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
     //AnnotationMetadata metadata：标注@Import(ImportBeanDefinitionRegistrar)的类的元数据信息
        
    //注册配置
		registerDefaultConfiguration(metadata, registry);
    //注册FeignClient
		registerFeignClients(metadata, registry);
	}

	private void registerDefaultConfiguration(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		Map<String, Object> defaultAttrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName(), true);
		//如果存在自定义配置 则进行配置
		if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
			String name;
			if (metadata.hasEnclosingClass()) {
				name = "default." + metadata.getEnclosingClassName();
			}
			else {
				name = "default." + metadata.getClassName();
			}
			registerClientConfiguration(registry, name,
					defaultAttrs.get("defaultConfiguration"));
		}
	}

	public void registerFeignClients(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
        //类路径扫描器
		ClassPathScanningCandidateComponentProvider scanner = getScanner();
		scanner.setResourceLoader(this.resourceLoader);

		Set<String> basePackages;
		
    //拿到@EnableFeignClients注解的属性
		Map<String, Object> attrs = metadata
				.getAnnotationAttributes(EnableFeignClients.class.getName());
		AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
				FeignClient.class);
		final Class<?>[] clients = attrs == null ? null
				: (Class<?>[]) attrs.get("clients");
		if (clients == null || clients.length == 0) {
            //一般不会配置 走这里
            //指定要扫描的注解
			scanner.addIncludeFilter(annotationTypeFilter);
			basePackages = getBasePackages(metadata);
		}
		else {
      //此处针对的场景是@EnableFeignClients{clients={UserFeignClient.class,OrderFeignClient.class}}
      //就只会扫描这些类 不是扫描整个类路径  
			final Set<String> clientClasses = new HashSet<>();
			basePackages = new HashSet<>();
			for (Class<?> clazz : clients) {
				basePackages.add(ClassUtils.getPackageName(clazz));
				clientClasses.add(clazz.getCanonicalName());
			}
			AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
				@Override
				protected boolean match(ClassMetadata metadata) {
          //处理内部类
					String cleaned = metadata.getClassName().replaceAll("\\$", ".");
					return clientClasses.contains(cleaned);
				}
			};
			scanner.addIncludeFilter(
					new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
		}

		for (String basePackage : basePackages) {
      //扫描 拿到所有类上有@FeignClient注解的类对应的BeanDefinition
			Set<BeanDefinition> candidateComponents = scanner
					.findCandidateComponents(basePackage);
			for (BeanDefinition candidateComponent : candidateComponents) {
				if (candidateComponent instanceof AnnotatedBeanDefinition) {
					// verify annotated class is an interface
					AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
					AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();

					Map<String, Object> attributes = annotationMetadata
							.getAnnotationAttributes(
									FeignClient.class.getCanonicalName());

					String name = getClientName(attributes);
          //针对不同的feign client注册配置  配置隔离
					registerClientConfiguration(registry, name,
							attributes.get("configuration"));
					//注册feign client
					registerFeignClient(registry, annotationMetadata, attributes);
				}
			}
		}
	}
```



org.springframework.cloud.openfeign.FeignClientsRegistrar#registerClientConfiguration

```java
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
      Object configuration) {
   BeanDefinitionBuilder builder = BeanDefinitionBuilder
         .genericBeanDefinition(FeignClientSpecification.class);
   builder.addConstructorArgValue(name);
   builder.addConstructorArgValue(configuration);
    //注册beanDefinition
   registry.registerBeanDefinition(
         name + "." + FeignClientSpecification.class.getSimpleName(),
         builder.getBeanDefinition());
}
```





org.springframework.cloud.openfeign.FeignClientsRegistrar#registerFeignClient

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
      AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
   String className = annotationMetadata.getClassName();
    //通过FactoryBean注册feign client
   BeanDefinitionBuilder definition = BeanDefinitionBuilder
         .genericBeanDefinition(FeignClientFactoryBean.class);
   validate(attributes);
   //设置factoryBean的属性
   definition.addPropertyValue("url", getUrl(attributes));
   definition.addPropertyValue("path", getPath(attributes));
   String name = getName(attributes);
   definition.addPropertyValue("name", name);
   String contextId = getContextId(attributes);
   definition.addPropertyValue("contextId", contextId);
   definition.addPropertyValue("type", className);
   definition.addPropertyValue("decode404", attributes.get("decode404"));
   definition.addPropertyValue("fallback", attributes.get("fallback"));
   definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
   definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

   String alias = contextId + "FeignClient";
   AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

   boolean primary = (Boolean) attributes.get("primary"); // has a default, won't be
           
   beanDefinition.setPrimary(primary);

   String qualifier = getQualifier(attributes);
   if (StringUtils.hasText(qualifier)) {
      alias = qualifier;
   }

   BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
         new String[] { alias });
   //注册到BeabFactory
   BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

## FeignClientFactoryBean（bean实例化阶段）

```java
class FeignClientFactoryBean implements FactoryBean<Object>, InitializingBean, ApplicationContextAware {
    @Override
	public Object getObject() throws Exception {
		return getTarget();
	}
	
	<T> T getTarget() {
		FeignContext context = this.applicationContext.getBean(FeignContext.class);
    //创建子容器 配置feign 设置日志、重试策略、错误code解析、超时时间、拦截器
		Feign.Builder builder = feign(context); // ==> NamedContextFactory#getInstance  创建容器
		//如果同时配置了url和name属性 优先使用url
		if (!StringUtils.hasText(this.url)) {
            //没有配置url属性   @FeignClient(url="")
            //http://producer-hello
			if (!this.name.startsWith("http")) {
				this.url = "http://" + this.name;
			}
			else {
				this.url = this.name;
			}
            // http://producer-hello/hello
			this.url += cleanPath();
            //走ribbon负载均衡的逻辑
			return (T) loadBalance(builder, context,
					new HardCodedTarget<>(this.type, this.name, this.url));
		}
    //如果直接配置了url，后面调用时就不会走负载均衡 而是直接调用
		if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
			this.url = "http://" + this.url;
		}
		String url = this.url + cleanPath();
		Client client = getOptional(context, Client.class);
		if (client != null) {
			if (client instanceof LoadBalancerFeignClient) {
				// not load balancing because we have a url,
				// but ribbon is on the classpath, so unwrap
				client = ((LoadBalancerFeignClient) client).getDelegate();
			}
			if (client instanceof FeignBlockingLoadBalancerClient) {
				// not load balancing because we have a url,
				// but Spring Cloud LoadBalancer is on the classpath, so unwrap
				client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
			}
			builder.client(client);
		}
		Targeter targeter = get(context, Targeter.class);
		return (T) targeter.target(this, builder, context,
				new HardCodedTarget<>(this.type, this.name, url));
	}
}	
```



### feign子容器

feign()

```java
protected Feign.Builder feign(FeignContext context) {
   FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
   Logger logger = loggerFactory.create(this.type);

   // @formatter:off
   Feign.Builder builder = get(context, Feign.Builder.class)
         // required values
         .logger(logger)
         .encoder(get(context, Encoder.class))
         .decoder(get(context, Decoder.class))
         .contract(get(context, Contract.class));
   // @formatter:on

   configureFeign(context, builder);

   return builder;
}
```

get()

```java
protected <T> T get(FeignContext context, Class<T> type) {
   T instance = context.getInstance(this.name, type);
   if (instance == null) {
      throw new IllegalStateException("No bean found of type " + type + " for "
            + this.name);
   }
   return instance;
}
```



org.springframework.cloud.context.named.NamedContextFactory#getInstance

```java
public <T> T getInstance(String name, Class<T> type) {
    AnnotationConfigApplicationContext context = this.getContext(name);
    return BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context, type).length > 0 ? context.getBean(type) : null;
}
```



getContext

```java
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



feign(context)最终会调用到这里 NamedContextFactory#createContext   **注意与ribbon的区别**

```java
protected AnnotationConfigApplicationContext createContext(String name) {
   AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    
    
    //default.com.dyzwj.consumerhellohystrixfeign.ConsumerHelloHystrixFeignApplication -> {FeignClientSpecification@5694} "FeignClientSpecification{name='default.com.dyzwj.consumerhellohystrixfeign.ConsumerHelloHystrixFeignApplication', configuration=[]}"
    //producer-hello -> {FeignClientSpecification@5695} "FeignClientSpecification{name='producer-hello', configuration=[]}"
   if (this.configurations.containsKey(name)) {
      for (Class<?> configuration : this.configurations.get(name)
            .getConfiguration()) {
         context.register(configuration);
      }
   }
   for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
      if (entry.getKey().startsWith("default.")) {
         for (Class<?> configuration : entry.getValue().getConfiguration()) {
            context.register(configuration);
         }
      }
   }
   context.register(PropertyPlaceholderAutoConfiguration.class,
         this.defaultConfigType);
   context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
         this.propertySourceName,
         Collections.<String, Object> singletonMap(this.propertyName, name)));
   if (this.parent != null) {
      // Uses Environment from parent as well as beans
      context.setParent(this.parent);
   }
   context.setDisplayName(generateDisplayName(name));
    //刷新
   context.refresh();
   return context;
}
```



org.springframework.cloud.openfeign.FeignClientFactoryBean#loadBalance

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
      HardCodedTarget<T> target) {
    //从上下文中根据类型拿Client 发起请求的客户端
    //LoadBalancerFeignClient
   Client client = getOptional(context, Client.class);
   if (client != null) {
      builder.client(client);
       //如果feign整合了hystrix  返回HystrixTargeter 见自动配置
      Targeter targeter = get(context, Targeter.class);
      return targeter.target(this, builder, context, target);
   }

   throw new IllegalStateException(
         "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
}
```

org.springframework.cloud.openfeign.HystrixTargeter#target

```java
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
      FeignContext context, Target.HardCodedTarget<T> target) {
    //有没有跟hystrix整合 == 有没有开启hystrix
   if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
       //没有跟hystrix整合
      return feign.target(target);
   }
    
    //跟hystrix整合
   feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
   String name = StringUtils.isEmpty(factory.getContextId()) ? factory.getName()
         : factory.getContextId();
   SetterFactory setterFactory = getOptional(name, context, SetterFactory.class);
   if (setterFactory != null) {
      builder.setterFactory(setterFactory);
   }
   //拿到fallback类名
   Class<?> fallback = factory.getFallback();
   if (fallback != void.class) {
       //走这里
      return targetWithFallback(name, context, target, builder, fallback);
   }
   Class<?> fallbackFactory = factory.getFallbackFactory();
   if (fallbackFactory != void.class) {
      return targetWithFallbackFactory(name, context, target, builder,
            fallbackFactory);
   }

   return feign.target(target);
}
```



### 跟hystrix整合

org.springframework.cloud.openfeign.HystrixTargeter#targetWithFallback

```java
private <T> T targetWithFallback(String feignClientName, FeignContext context,
                         Target.HardCodedTarget<T> target,
                         HystrixFeign.Builder builder, Class<?> fallback) {
   T fallbackInstance = getFromContext("fallback", feignClientName, context, fallback, target.type());
   return builder.target(target, fallbackInstance);
}
```

feign.hystrix.HystrixFeign.Builder#target(feign.Target<T>, T)

```java
public <T> T target(Target<T> target, T fallback) {
    return this.build(fallback != null ? new feign.hystrix.FallbackFactory.Default(fallback) : null).newInstance(target);
}
```

feign.hystrix.HystrixFeign.Builder#build()

```java
Feign build(final FallbackFactory<?> nullableFallbackFactory) {
    //设置InvocationHandler
    super.invocationHandlerFactory(new InvocationHandlerFactory() {
        public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
            return new HystrixInvocationHandler(target, dispatch, Builder.this.setterFactory, nullableFallbackFactory);
        }
    });
    super.contract(new HystrixDelegatingContract(this.contract));
    return super.build();
}
```

feign.ReflectiveFeign#newInstance

```java
public <T> T newInstance(Target<T> target) {
    //方法名和MethodHandler的映射关系
    //HelloFeignClient#user(String) -> {SynchronousMethodHandler@6083} 
    //HelloFeignClient#hello(Map) -> {SynchronousMethodHandler@6085} 
    Map<String, MethodHandler> nameToHandler = this.targetToHandlersByName.apply(target);
    //方法和MethodHandler的映射关系
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList();
    Method[] var5 = target.type().getMethods();
    int var6 = var5.length;

    for(int var7 = 0; var7 < var6; ++var7) {
        Method method = var5[var7];
        if (method.getDeclaringClass() != Object.class) {
            if (Util.isDefault(method)) {
                DefaultMethodHandler handler = new DefaultMethodHandler(method);
                defaultMethodHandlers.add(handler);
                methodToHandler.put(method, handler);
            } else {
                methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
            }
        }
    }
	
    //HystrixInvocationHandler 
    //feign client的接口调用会调用到HystrixInvocationHandler#invoke()
    InvocationHandler handler = this.factory.create(target, methodToHandler);
    //调用Jdk动态代理生成代理对象
    T proxy = Proxy.newProxyInstance(target.type().getClassLoader(), new Class[]{target.type()}, handler);
    Iterator var12 = defaultMethodHandlers.iterator();
	
    //把接口的默认方法绑定到代理对象
    while(var12.hasNext()) {
        DefaultMethodHandler defaultMethodHandler = (DefaultMethodHandler)var12.next();
        defaultMethodHandler.bindTo(proxy);
    }
	//返回代理对象
    return proxy;
}
```



#### HystrixInvocationHandler

feign.hystrix.HystrixInvocationHandler#invoke

```java
public Object invoke(Object proxy, final Method method, final Object[] args) throws Throwable {
    if (!"equals".equals(method.getName())) {
        if ("hashCode".equals(method.getName())) {
            return this.hashCode();
        } else if ("toString".equals(method.getName())) {
            return this.toString();
        } else {
            //我们写的方法
            HystrixCommand<Object> hystrixCommand = new HystrixCommand<Object>((Setter)this.setterMethodMap.get(method)) {
                protected Object run() throws Exception {
                    try {
                        //后续逻辑见没有跟hystrix整合的代码
                        return ((MethodHandler)HystrixInvocationHandler.this.dispatch.get(method)).invoke(args);
                    } catch (Exception var2) {
                        throw var2;
                    } catch (Throwable var3) {
                        throw (Error)var3;
                    }
                }

                protected Object getFallback() {
                    if (HystrixInvocationHandler.this.fallbackFactory == null) {
                        return super.getFallback();
                    } else {
                        try {
                            Object fallback = HystrixInvocationHandler.this.fallbackFactory.create(this.getExecutionException());
                            Object result = ((Method)HystrixInvocationHandler.this.fallbackMethodMap.get(method)).invoke(fallback, args);
                            if (HystrixInvocationHandler.this.isReturnsHystrixCommand(method)) {
                                return ((HystrixCommand)result).execute();
                            } else if (HystrixInvocationHandler.this.isReturnsObservable(method)) {
                                return ((Observable)result).toBlocking().first();
                            } else if (HystrixInvocationHandler.this.isReturnsSingle(method)) {
                                return ((Single)result).toObservable().toBlocking().first();
                            } else if (HystrixInvocationHandler.this.isReturnsCompletable(method)) {
                                ((Completable)result).await();
                                return null;
                            } else {
                                return result;
                            }
                        } catch (IllegalAccessException var3) {
                            throw new AssertionError(var3);
                        } catch (InvocationTargetException var4) {
                            throw new AssertionError(var4.getCause());
                        }
                    }
                }
            };
            if (this.isReturnsHystrixCommand(method)) {
                return hystrixCommand;
            } else if (this.isReturnsObservable(method)) {
                return hystrixCommand.toObservable();
            } else if (this.isReturnsSingle(method)) {
                return hystrixCommand.toObservable().toSingle();
            } else {
                return this.isReturnsCompletable(method) ? hystrixCommand.toObservable().toCompletable() : hystrixCommand.execute();
            }
        }
    } else {
        try {
            Object otherHandler = args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
            return this.equals(otherHandler);
        } catch (IllegalArgumentException var5) {
            return false;
        }
    }
}
```



### 没有跟hystrix整合

feign.Feign.Builder#target(feign.Target<T>)

```java
public <T> T target(Target<T> target) {
    return this.build().newInstance(target);
}
```



feign.ReflectiveFeign#newInstance

```java
public <T> T newInstance(Target<T> target) {
    //方法名 == MethodHandler 方法处理器
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    //方法 == MethodHandler的映射
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();
	  //遍历被代理接口(UserFeignClient)的所有方法
    for (Method method : target.type().getMethods()) {
        //如果方法继承自Object类 什么都不做
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if (Util.isDefault(method)) {
        //如果是接口的默认方法
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        //我们写的feign client的方法一般走这里
        //(Feign.configKey(target.type(), method)：生成方法名 UserFeignClient#hello()
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    //使用工厂创建代理逻辑
    InvocationHandler handler = factory.create(target, methodToHandler);
    //进行代理
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);

    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      //将默认方法绑定到代理对象 （接口的默认方法还是需要对象来调用）
      defaultMethodHandler.bindTo(proxy);
    }
    //返回给FactoryBean的getObject() 
    return proxy;
  }
```

feign.InvocationHandlerFactory.Default.create(是个内部类)

```java
static final class Default implements InvocationHandlerFactory {

  @Override
  public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
    //FeignInvocationHandler实现了InvocationHandler接口 实现其invoke()方法
    return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
  }
}
```



#### FeignInvocationHandler 

```java
static class FeignInvocationHandler implements InvocationHandler {

  private final Target target;
  private final Map<Method, MethodHandler> dispatch;

  FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
    this.target = checkNotNull(target, "target");
    this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
  }

   //调用feign client的方法时 会走到这里
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if ("equals".equals(method.getName())) {
      try {
        Object otherHandler =
            args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
        return equals(otherHandler);
      } catch (IllegalArgumentException e) {
        return false;
      }
    } else if ("hashCode".equals(method.getName())) {
      return hashCode();
    } else if ("toString".equals(method.getName())) {
      return toString();
    }
	//我们写的方法一般走这里
    return dispatch.get(method).invoke(args);
  }
```

feign.SynchronousMethodHandler#invoke

```java
public Object invoke(Object[] argv) throws Throwable {
  //请求模板 封装请求信息
  RequestTemplate template = buildTemplateFromArgs.create(argv);
  //feign的重试配置信息
  Options options = findOptions(argv);
  //重试机制 非线程安全 每次都会新创建一个重试器
  Retryer retryer = this.retryer.clone();
  while (true) {
    try {
      //执行请求并解码
      return executeAndDecode(template, options);
    } catch (RetryableException e) {
      try {
        //统计重试次数  feign的重试机制
        retryer.continueOrPropagate(e);
      } catch (RetryableException th) {
        Throwable cause = th.getCause();
        if (propagationPolicy == UNWRAP && cause != null) {
          throw cause;
        } else {
          throw th;
        }
      }
      if (logLevel != Logger.Level.NONE) {
        logger.logRetry(metadata.configKey(), logLevel);
      }
      continue;
    }
  }
}
```

feign.SynchronousMethodHandler#executeAndDecode

```java
Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
  //Feign的拓展点  对Request进行处理
  Request request = targetRequest(template);

  if (logLevel != Logger.Level.NONE) {
    logger.logRequest(metadata.configKey(), logLevel, request);
  }

  Response response;
  long start = System.nanoTime();
  try {
    //执行请求
    response = client.execute(request, options);
  } catch (IOException e) {
    if (logLevel != Logger.Level.NONE) {
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
    }
    throw errorExecuting(request, e);
  }
  long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

  //处理响应
  boolean shouldClose = true;
  try {
    if (Response.class == metadata.returnType()) {
      if (response.body() == null) {
        return response;
      }
      if (response.body().length() == null ||
          response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
        shouldClose = false;
        return response;
      }
      // Ensure the response body is disconnected
      byte[] bodyData = Util.toByteArray(response.body().asInputStream());
      return response.toBuilder().body(bodyData).build();
    }
    if (response.status() >= 200 && response.status() < 300) {
      if (void.class == metadata.returnType()) {
        return null;
      } else {
        //处理响应，映射成对象
        Object result = decode(response);
        shouldClose = closeAfterDecode;
        return result;
      }
    } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
      Object result = decode(response);
      shouldClose = closeAfterDecode;
      return result;
    } else {
      throw errorDecoder.decode(metadata.configKey(), response);
    }
  } catch (IOException e) {
    if (logLevel != Logger.Level.NONE) {
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
    }
    throw errorReading(request, response, e);
  } finally {
    if (shouldClose) {
      ensureClosed(response.body());
    }
  }
}
```

springcloud-openfeign-core包下的META-INF/spring.factories导入了FeignRibbonClientAutoConfiguration自动配置类 向容器中添加了LoadBalancerFeignClient

org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute

```java
public Response execute(Request request, Request.Options options) throws IOException {
   try {
      //拿到请求uri http://user-service/getUser?name=111
      URI asUri = URI.create(request.url());
       //拿到微服务名 user-service
      String clientName = asUri.getHost();
      URI uriWithoutHost = cleanUrl(request.url(), clientName);
      //构建负载均衡器
      FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
            this.delegate, request, uriWithoutHost);

      IClientConfig requestConfig = getClientConfig(options, clientName);
      //执行负载均衡逻辑
      return lbClient(clientName)
            .executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
   }
   catch (ClientException e) {
      IOException io = findIOException(e);
      if (io != null) {
         throw io;
      }
      throw new RuntimeException(e);
   }
}
```

com.netflix.client.AbstractLoadBalancerAwareClient#executeWithLoadBalancer

```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    //创建LoadBalancerCommand 设置ribbon的重试策略
    LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

    try {
        return command.submit(
            new ServerOperation<T>() {
                @Override
                public Observable<T> call(Server server) {
                    //server：selectServer()的结果
                    //最终的uri地址  http://192.168.78.96:9090/getUser?name=111
                    URI finalUri = reconstructURIWithServer(server, request.getUri());
                    S requestForServer = (S) request.replaceUri(finalUri);
                    try {
                        //FeignLoadBalancer#execute：执行请求
                        return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                    } 
                    catch (Exception e) {
                        return Observable.error(e);
                    }
                }
            })
            .toBlocking()
            .single();
    } catch (Exception e) {
        Throwable t = e.getCause();
        if (t instanceof ClientException) {
            throw (ClientException) t;
        } else {
            throw new ClientException(e);
        }
    }
    
}
```



submit()：选择一台服务器  负载均衡

```java
public Observable<T> submit(final ServerOperation<T> operation) {
    final ExecutionInfoContext context = new ExecutionInfoContext();
    
    if (listenerInvoker != null) {
        try {
            listenerInvoker.onExecutionStart();
        } catch (AbortExecutionException e) {
            return Observable.error(e);
        }
    }
    //获取重试相同实例的次数
    final int maxRetrysSame = retryHandler.getMaxRetriesOnSameServer();
    // 重试其他实例的最大重试次数，不包括首次所选的server
    final int maxRetrysNext = retryHandler.getMaxRetriesOnNextServer();

    Observable<T> o = 
             //选择具体的Server进行调用
            (server == null ? selectServer() : Observable.just(server))
            .concatMap(new Func1<Server, Observable<T>>() {
                @Override
                public Observable<T> call(Server server) {
                    context.setServer(server);
                    //获取这个server调用监控记录，用于各种统计和LoadBalanceRule的筛选server处理
                    final ServerStats stats = loadBalancerContext.getServerStats(server);
                    
                    //获取本次server调用的回调入口，用于重试同一实例的重试回调
                    Observable<T> o = Observable
                            .just(server)
                            .concatMap(new Func1<Server, Observable<T>>() {
                                @Override
                                public Observable<T> call(final Server server) {
                                    
                                    //lambda表达式在这里调用
                                    return operation.call(server).doOnEach(new Observer<T>() {
                                       
                                    });
                                }
                            });
                    //设置同一实例的重试
                    if (maxRetrysSame > 0) 
                        o = o.retry(retryPolicy(maxRetrysSame, true));
                    return o;
                }
            });
    //重试其他实例的最大重试次数，不包括首次所选的server
    if (maxRetrysNext > 0 && server == null) 
        o = o.retry(retryPolicy(maxRetrysNext, false));
    //异常回调
    return o.onErrorResumeNext(new Func1<Throwable, Observable<T>>() {
        @Override
        public Observable<T> call(Throwable e) {
            return Observable.error(e);
        }
    });
}
```





```java
private Func2<Integer, Throwable, Boolean> retryPolicy(final int maxRetrys, final boolean same) {
  	//same=false:重试其他实例
  	//same=true:重试同一实例
    return new Func2<Integer, Throwable, Boolean>() {
        @Override
        public Boolean call(Integer tryCount, Throwable e) {
            if (e instanceof AbortExecutionException) {
                return false;
            }
						
          	//
            if (tryCount > maxRetrys) {
                return false;
            }
            
            if (e.getCause() != null && e instanceof RuntimeException) {
                e = e.getCause();
            }
            
            return retryHandler.isRetriableException(e, same);
        }
    };
}
```







com.netflix.loadbalancer.reactive.LoadBalancerCommand#selectServer

```java
private Observable<Server> selectServer() {
    return Observable.create(new OnSubscribe<Server>() {
        @Override
        public void call(Subscriber<? super Server> next) {
            try {
                Server server = loadBalancerContext.getServerFromLoadBalancer(loadBalancerURI, loadBalancerKey);   
                next.onNext(server);
                next.onCompleted();
            } catch (Exception e) {
                next.onError(e);
            }
        }
    });
}
```



com.netflix.loadbalancer.LoadBalancerContext#getServerFromLoadBalancer

```java
public Server getServerFromLoadBalancer(@Nullable URI original, @Nullable Object loadBalancerKey) throws ClientException {
    String host = null;
    int port = -1;
    if (original != null) {
        host = original.getHost();
    }
    if (original != null) {
        Pair<String, Integer> schemeAndPort = deriveSchemeAndPortFromPartialUri(original);        
        port = schemeAndPort.second();
    }

    // Various Supported Cases
    // The loadbalancer to use and the instances it has is based on how it was registered
    // In each of these cases, the client might come in using Full Url or Partial URL
    ILoadBalancer lb = getLoadBalancer();
    if (host == null) {
        // Partial URI or no URI Case
        // well we have to just get the right instances from lb - or we fall back
        if (lb != null){
            //选择一个服务
            Server svc = lb.chooseServer(loadBalancerKey);
            if (svc == null){
                throw new ClientException(ClientException.ErrorType.GENERAL,
                        "Load balancer does not have available server for client: "
                                + clientName);
            }
            host = svc.getHost();
            if (host == null){
                throw new ClientException(ClientException.ErrorType.GENERAL,
                        "Invalid Server for :" + svc);
            }
            logger.debug("{} using LB returned Server: {} for request {}", new Object[]{clientName, svc, original});
            return svc;
        } else {
            // No Full URL - and we dont have a LoadBalancer registered to
            // obtain a server
            // if we have a vipAddress that came with the registration, we
            // can use that else we
            // bail out
            if (vipAddresses != null && vipAddresses.contains(",")) {
                throw new ClientException(
                        ClientException.ErrorType.GENERAL,
                        "Method is invoked for client " + clientName + " with partial URI of ("
                        + original
                        + ") with no load balancer configured."
                        + " Also, there are multiple vipAddresses and hence no vip address can be chosen"
                        + " to complete this partial uri");
            } else if (vipAddresses != null) {
                try {
                    Pair<String,Integer> hostAndPort = deriveHostAndPortFromVipAddress(vipAddresses);
                    host = hostAndPort.first();
                    port = hostAndPort.second();
                } catch (URISyntaxException e) {
                    throw new ClientException(
                            ClientException.ErrorType.GENERAL,
                            "Method is invoked for client " + clientName + " with partial URI of ("
                            + original
                            + ") with no load balancer configured. "
                            + " Also, the configured/registered vipAddress is unparseable (to determine host and port)");
                }
            } else {
                throw new ClientException(
                        ClientException.ErrorType.GENERAL,
                        this.clientName
                        + " has no LoadBalancer registered and passed in a partial URL request (with no host:port)."
                        + " Also has no vipAddress registered");
            }
        }
    } else {
        // Full URL Case
        // This could either be a vipAddress or a hostAndPort or a real DNS
        // if vipAddress or hostAndPort, we just have to consult the loadbalancer
        // but if it does not return a server, we should just proceed anyways
        // and assume its a DNS
        // For restClients registered using a vipAddress AND executing a request
        // by passing in the full URL (including host and port), we should only
        // consult lb IFF the URL passed is registered as vipAddress in Discovery
        boolean shouldInterpretAsVip = false;

        if (lb != null) {
            shouldInterpretAsVip = isVipRecognized(original.getAuthority());
        }
        if (shouldInterpretAsVip) {
            Server svc = lb.chooseServer(loadBalancerKey);
            if (svc != null){
                host = svc.getHost();
                if (host == null){
                    throw new ClientException(ClientException.ErrorType.GENERAL,
                            "Invalid Server for :" + svc);
                }
                logger.debug("using LB returned Server: {} for request: {}", svc, original);
                return svc;
            } else {
                // just fall back as real DNS
                logger.debug("{}:{} assumed to be a valid VIP address or exists in the DNS", host, port);
            }
        } else {
            // consult LB to obtain vipAddress backed instance given full URL
            //Full URL execute request - where url!=vipAddress
            logger.debug("Using full URL passed in by caller (not using load balancer): {}", original);
        }
    }
    // end of creating final URL
    if (host == null){
        throw new ClientException(ClientException.ErrorType.GENERAL,"Request contains no HOST to talk to");
    }
    // just verify that at this point we have a full URL

    return new Server(host, port);
}
```





feign.Retryer.Default#continueOrPropagate

```java
public void continueOrPropagate(RetryableException e) {
  if (attempt++ >= maxAttempts) {
    throw e;
  }

  long interval;
  if (e.retryAfter() != null) {
    interval = e.retryAfter().getTime() - currentTimeMillis();
    if (interval > maxPeriod) {
      interval = maxPeriod;
    }
    if (interval < 0) {
      return;
    }
  } else {
    interval = nextMaxInterval();
  }
  try {
    Thread.sleep(interval);
  } catch (InterruptedException ignored) {
    Thread.currentThread().interrupt();
    throw e;
  }
  sleptForMillis += interval;
}
```





springcloud-openfeign-core包下的META-INF/spring.factories导入了FeignRibbonClientAutoConfiguration自动配置类



```java

@Import({ HttpClientFeignLoadBalancedConfiguration.class,
      OkHttpFeignLoadBalancedConfiguration.class,
      DefaultFeignLoadBalancedConfiguration.class })
public class FeignRibbonClientAutoConfiguration {


}
```

FeignRibbonClientAutoConfiguration导入了DefaultFeignLoadBalancedConfiguration

```java
@Configuration(proxyBeanMethods = false)
class DefaultFeignLoadBalancedConfiguration {

   @Bean
   @ConditionalOnMissingBean
   public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
         SpringClientFactory clientFactory) {
      return new LoadBalancerFeignClient(new Client.Default(null, null), cachingFactory,
            clientFactory);
   }

}
```







# feign、ribbon的重试  超时

## feign重试机制

### 

feign.SynchronousMethodHandler#invoke

```java
public Object invoke(Object[] argv) throws Throwable {
  RequestTemplate template = buildTemplateFromArgs.create(argv);
   //feign的重试配置信息  每次请求都会创建一个新的重试配置信息 默认为NEVER_RETRY
  Retryer retryer = this.retryer.clone();
  while (true) {
    try {
        //执行请求
      return executeAndDecode(template);
    } catch (RetryableException e) {
      //捕获重试异常  feign的重试策略
      retryer.continueOrPropagate(e);
      if (logLevel != Logger.Level.NONE) {
        logger.logRetry(metadata.configKey(), logLevel);
      }
      continue;
    }
  }
}
```







## ribbon重试机制

ribbon常用配置

```java
ribbon:
  OkToRetryOnAllOperations: true #对所有操作请求都进行重试,默认false
  ReadTimeout: 1000   #请求处理超时时间，默认值5000
  ConnectTimeout: 3000 #请求连接的超时时间，默认值2000
  MaxAutoRetries: 1    #对当前实例的重试次数，默认0
  MaxAutoRetriesNextServer: 0 # 重试其他实例的最大重试次数，不包括首次所选的server
，默认1

```



```java
public DefaultLoadBalancerRetryHandler(IClientConfig clientConfig) {
    this.retrySameServer = (Integer)clientConfig.get(CommonClientConfigKey.MaxAutoRetries, 0);
    this.retryNextServer = (Integer)clientConfig.get(CommonClientConfigKey.MaxAutoRetriesNextServer, 1);
    this.retryEnabled = (Boolean)clientConfig.get(CommonClientConfigKey.OkToRetryOnAllOperations, false);
}
```

> retrySameServer：重试相同实例，对应MaxAutoRetries
> retryNextServer：重试其他实例的最大重试次数，对应MaxAutoRetriesNextServer，不包括首次所选的server
> retryEnabled：重试所有操作，对应OkToRetryOnAllOperations



```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    //创建command 封装了重试策略
    LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

    try {
        return command.submit(
            new ServerOperation<T>() {
                @Override
                public Observable<T> call(Server server) {
                    URI finalUri = reconstructURIWithServer(server, request.getUri());
                    S requestForServer = (S) request.replaceUri(finalUri);
                    try {
                        return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                    } 
                    catch (Exception e) {
                        return Observable.error(e);
                    }
                }
            })
            .toBlocking()
            .single();
    } catch (Exception e) {
        Throwable t = e.getCause();
        if (t instanceof ClientException) {
            throw (ClientException) t;
        } else {
            throw new ClientException(e);
        }
    }
    
}
```



com.netflix.client.AbstractLoadBalancerAwareClient#buildLoadBalancerCommand

```java
   protected LoadBalancerCommand<T> buildLoadBalancerCommand(final S request, final IClientConfig config) {
   //获取重试策略
   RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, config);
   LoadBalancerCommand.Builder<T> builder = LoadBalancerCommand.<T>builder()
         .withLoadBalancerContext(this)
     			//设置重试策略
         .withRetryHandler(handler)
         .withLoadBalancerURI(request.getUri());
   customizeLoadBalancerCommandBuilder(request, config, builder);
   return builder.build();
}
```

org.springframework.cloud.openfeign.ribbon.FeignLoadBalancer#getRequestSpecificRetryHandler

```java
public RequestSpecificRetryHandler getRequestSpecificRetryHandler(
      RibbonRequest request, IClientConfig requestConfig) {
    
   //如果OkToRetryOnAllOperations配置为true 则所有的请求都进行重试 默认为false
   if (this.ribbon.isOkToRetryOnAllOperations()) {
      return new RequestSpecificRetryHandler(true, true, this.getRetryHandler(),
            requestConfig);
   }
   //如果OkToRetryOnAllOperations配置为false
    
   //如果不是get请求 就关闭重试
   if (!request.toRequest().method().equals("GET")) {
      //构造器前两个boolean参数：
      //okToRetryOnConnectErrors：连接错误是否重试
      //okToRetryOnAllErrors：是否所有错误都重试
      return new RequestSpecificRetryHandler(true, false, this.getRetryHandler(),
            requestConfig);
   }
    //如果是get请求 就开启重试
   else {
      return new RequestSpecificRetryHandler(true, true, this.getRetryHandler(),
            requestConfig);
   }
}
```

com.netflix.loadbalancer.reactive.LoadBalancerCommand#submit

```java
public Observable<T> submit(final ServerOperation<T> operation) {
    final ExecutionInfoContext context = new ExecutionInfoContext();
    
    if (listenerInvoker != null) {
        try {
            listenerInvoker.onExecutionStart();
        } catch (AbortExecutionException e) {
            return Observable.error(e);
        }
    }
    //重试相同实例
    final int maxRetrysSame = retryHandler.getMaxRetriesOnSameServer();
    //重试下一实例
    final int maxRetrysNext = retryHandler.getMaxRetriesOnNextServer();


    Observable<T> o = 
             //选择具体的Server进行调用
            (server == null ? selectServer() : Observable.just(server))
            .concatMap(new Func1<Server, Observable<T>>() {
                @Override
            
                public Observable<T> call(Server server) {
                    context.setServer(server);
                    //获取这个server调用监控记录，用于各种统计和LoadBalanceRule的筛选server处理
                    final ServerStats stats = loadBalancerContext.getServerStats(server);
                    
                     //获取本次server调用的回调入口，用于重试同一实例的重试回调
                    Observable<T> o = Observable
                            .just(server)
                            .concatMap(new Func1<Server, Observable<T>>() {
                                @Override
                                public Observable<T> call(final Server server) {
                                    
                                    //lambda表达式在这里调用
                                    return operation.call(server).doOnEach(new Observer<T>() {
                                       
                                    });
                                }
                            });
                    //同一实例的重试
                    if (maxRetrysSame > 0) 
                        o = o.retry(retryPolicy(maxRetrysSame, true));
                    return o;
                }
            });
    //重试其他实例的最大重试次数
    if (maxRetrysNext > 0 && server == null) 
        o = o.retry(retryPolicy(maxRetrysNext, false));
    //异常回调
    return o.onErrorResumeNext(new Func1<Throwable, Observable<T>>() {
        @Override
        public Observable<T> call(Throwable e) {
            return Observable.error(e);
        }
    });
}
```

com.netflix.loadbalancer.reactive.LoadBalancerCommand#retryPolicy

```java
private Func2<Integer, Throwable, Boolean> retryPolicy(final int maxRetrys, final boolean same) {
    return new Func2<Integer, Throwable, Boolean>() {
        @Override
        public Boolean call(Integer tryCount, Throwable e) {
            if (e instanceof AbortExecutionException) {
                return false;
            }
						//判断是否继续重试
            if (tryCount > maxRetrys) {
                return false;
            }
            
            if (e.getCause() != null && e instanceof RuntimeException) {
                e = e.getCause();
            }
            
            //进行异常处理
            //同一实例 same为true
            //下一实例 same为false
            return retryHandler.isRetriableException(e, same);
        }
    };
}
```



com.netflix.client.RequestSpecificRetryHandler#isRetriableException

```java
public boolean isRetriableException(Throwable e, boolean sameServer) {
    //针对get请求，okToRetryOnAllErrors为true
    if (okToRetryOnAllErrors) {
        return true;
    } 
    //是否是客户端错误
    else if (e instanceof ClientException) {
        ClientException ce = (ClientException) e;
        if (ce.getErrorType() == ClientException.ErrorType.SERVER_THROTTLED) {
            return !sameServer;
        } else {
            return false;
        }
    } 
    //其他请求  如post delete请求并且 只有是连接错误  才会进行重试
    else  {
        return okToRetryOnConnectErrors && isConnectionException(e);
    }
}
```



超时机制



Hystrix的超时时间=Ribbon的重试次数(包含首次) * (ribbon.ReadTimeout + ribbon.ConnectTimeout)

`ribbon:
  OkToRetryOnAllOperations: true #对所有操作请求都进行重试,默认false
  ReadTimeout: 1000   #负载均衡超时时间，默认值5000
  ConnectTimeout: 3000 #请求连接的超时时间，默认值2000
  MaxAutoRetries: 1    #对当前实例的重试次数，默认0
  MaxAutoRetriesNextServer: 0 #重试切换实例的次数(最多重试几个实例)，默认1，不包括首次所选的server`

而Ribbon的重试次数的计算方式为：

```
Ribbon重试次数(包含首次)= 1 + ribbon.MaxAutoRetries  +  ribbon.MaxAutoRetriesNextServer  +  (ribbon.MaxAutoRetries * ribbon.MaxAutoRetriesNextServer)
```



get请求不尽兴重试，其他请求 连接错误才会进行重试



# feign整合ribbon、hystrix



org.springframework.cloud.openfeign.FeignClientFactoryBean#loadBalance

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
      HardCodedTarget<T> target) {
   //LoadBalancerFeignClient
   Client client = getOptional(context, Client.class);
   if (client != null) {
      builder.client(client);
      Targeter targeter = get(context, Targeter.class);
      //HystrixTargeter
      return targeter.target(this, builder, context, target);
   }

   throw new IllegalStateException(
         "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
}
```



首先获取`Client`的实现类，这个实现类是`LoadBalancerFeignClient`,这个类里融合了Ribbon的相关内容。然后将`Client`包装到Feign.Builder中，接着获取`Targeter`，这里我们存在Hystrix环境，所以`Targeter`的实现类为`HystrixTargeter`



```java
class HystrixTargeter implements Targeter {

   @Override
   public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
                  Target.HardCodedTarget<T> target) {
      if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
         return feign.target(target);
      }
      feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
      SetterFactory setterFactory = getOptional(factory.getName(), context,
         SetterFactory.class);
      if (setterFactory != null) {
         builder.setterFactory(setterFactory);
      }
      Class<?> fallback = factory.getFallback();
      if (fallback != void.class) {
         return targetWithFallback(factory.getName(), context, target, builder, fallback);
      }
      Class<?> fallbackFactory = factory.getFallbackFactory();
      if (fallbackFactory != void.class) {
         return targetWithFallbackFactory(factory.getName(), context, target, builder, fallbackFactory);
      }

      return feign.target(target);
   }
}
```

接着以Feign客户端设置了fallback为例

```java
private <T> T targetWithFallback(String feignClientName, FeignContext context,
                         Target.HardCodedTarget<T> target,
                         HystrixFeign.Builder builder, Class<?> fallback) {
   T fallbackInstance = getFromContext("fallback", feignClientName, context, fallback, target.type());
   return builder.target(target, fallbackInstance);
}
```

接着就是这个代理的创建，现在这个代理中包含了Ribbon和Hystrix。而这个代理类的实现类是`HystrixInvocationHandler`

```java
  public Object invoke(final Object proxy, final Method method, final Object[] args)
      throws Throwable {
    if ("equals".equals(method.getName())) {
      try {
        Object otherHandler =
            args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
        return equals(otherHandler);
      } catch (IllegalArgumentException e) {
        return false;
      }
    } else if ("hashCode".equals(method.getName())) {
      return hashCode();
    } else if ("toString".equals(method.getName())) {
      return toString();
    }

    HystrixCommand<Object> hystrixCommand = new HystrixCommand<Object>(setterMethodMap.get(method)) {
      @Override
      protected Object run() throws Exception {
        try {
          //feign.SynchronousMethodHandler#invoke
          return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
        } catch (Exception e) {
          throw e;
        } catch (Throwable t) {
          throw (Error) t;
        }
      }

      @Override
      protected Object getFallback() {
        if (fallbackFactory == null) {
          return super.getFallback();
        }
        try {
          Object fallback = fallbackFactory.create(getExecutionException());
          Object result = fallbackMethodMap.get(method).invoke(fallback, args);
          if (isReturnsHystrixCommand(method)) {
            return ((HystrixCommand) result).execute();
          } else if (isReturnsObservable(method)) {
            // Create a cold Observable
            return ((Observable) result).toBlocking().first();
          } else if (isReturnsSingle(method)) {
            // Create a cold Observable as a Single
            return ((Single) result).toObservable().toBlocking().first();
          } else if (isReturnsCompletable(method)) {
            ((Completable) result).await();
            return null;
          } else {
            return result;
          }
        } catch (IllegalAccessException e) {
          // shouldn't happen as method is public due to being an interface
          throw new AssertionError(e);
        } catch (InvocationTargetException e) {
          // Exceptions on fallback are tossed by Hystrix
          throw new AssertionError(e.getCause());
        }
      }
    };

    if (isReturnsHystrixCommand(method)) {
      return hystrixCommand;
    } else if (isReturnsObservable(method)) {
      // Create a cold Observable
      return hystrixCommand.toObservable();
    } else if (isReturnsSingle(method)) {
      // Create a cold Observable as a Single
      return hystrixCommand.toObservable().toSingle();
    } else if (isReturnsCompletable(method)) {
      return hystrixCommand.toObservable().toCompletable();
    }
    return hystrixCommand.execute();
  }
```



接着深入`invoke`方法

```java
public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = this.buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();

    while(true) {
        try {
            return this.executeAndDecode(template);
        } catch (RetryableException var5) {
            retryer.continueOrPropagate(var5);
            if (this.logLevel != Level.NONE) {
                this.logger.logRetry(this.metadata.configKey(), this.logLevel);
            }
        }
    }
}
```

这里构建了请求信息和重试策略，具体请求内容在下面：

```java
Object executeAndDecode(RequestTemplate template) throws Throwable {
  //feign的扩展点 对request进行处理
  Request request = targetRequest(template);

  if (logLevel != Logger.Level.NONE) {
    logger.logRequest(metadata.configKey(), logLevel, request);
  }

  Response response;
  long start = System.nanoTime();
  try {
      //org.springframework.cloud.openfeign.ribbon.LoadBalancerFeignClient#execute
    response = client.execute(request, options);
    // ensure the request is set. TODO: remove in Feign 10
    response.toBuilder().request(request).build();
  } catch (IOException e) {
    if (logLevel != Logger.Level.NONE) {
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
    }
    throw errorExecuting(request, e);
  }
  long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

  boolean shouldClose = true;
  try {
    if (logLevel != Logger.Level.NONE) {
      response =
          logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
      // ensure the request is set. TODO: remove in Feign 10
      response.toBuilder().request(request).build();
    }
    if (Response.class == metadata.returnType()) {
      if (response.body() == null) {
        return response;
      }
      if (response.body().length() == null ||
              response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
        shouldClose = false;
        return response;
      }
      // Ensure the response body is disconnected
      byte[] bodyData = Util.toByteArray(response.body().asInputStream());
      return response.toBuilder().body(bodyData).build();
    }
    if (response.status() >= 200 && response.status() < 300) {
      if (void.class == metadata.returnType()) {
        return null;
      } else {
        return decode(response);
      }
    } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
      return decode(response);
    } else {
      throw errorDecoder.decode(metadata.configKey(), response);
    }
  } catch (IOException e) {
    if (logLevel != Logger.Level.NONE) {
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
    }
    throw errorReading(request, response, e);
  } finally {
    if (shouldClose) {
      ensureClosed(response.body());
    }
  }
}
```