​																									bootstrap容器





```java
public static void main(String[] args) {
     //进入run方法 会发现最终是new 一个SpringApplication实例 然后调用实例的run方法来启动
    SpringApplication.run(SpringcoudApplication.class, args);
  }

```

上面的一行代码实际上做了两件事情

1. **实例化一个SpringApplication对象**
2. **执行SpringApplication的run方法**



先看**实例化一个SpringApplication对象：**

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.sources = new LinkedHashSet();
    this.bannerMode = Mode.CONSOLE;
    this.logStartupInfo = true;
    this.addCommandLineProperties = true;
    this.addConversionService = true;
    this.headless = true;
    this.registerShutdownHook = true;
    this.additionalProfiles = new HashSet();
    this.isCustomEnvironment = false;
    this.lazyInitialization = false;
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //下面两行代码 非常重要 1，使用SPI加载classpath下面所有key为ApplicationContextInitializer 
   	//全限定名的配置。 2，使用SPI加载classpath下面所有key为ApplicationListener的配置
    this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```



上面的注释也提到了 实例化SpringApplication最重要的两行代码就是通过SPI的方式加载 ApplicationContextInitializer和ApplicationListener。加载的listener中包括BootstrapApplicationListener ，这个监听器监听ApplicationEnvironmentPreparedEvent事件

执行SpringApplication的run方法：

```java
public ConfigurableApplicationContext run(String... args) {
   StopWatch stopWatch = new StopWatch();
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
	configureHeadlessProperty();
	SpringApplicationRunListeners listeners = getRunListeners(args);
	listeners.starting();
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
		//创建一个Environment实例  重点是在这里 
		ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
}
```

准备环境

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
    //创建一个ConfigurableEnvironment实例
    ConfigurableEnvironment environment = this.getOrCreateEnvironment();
    //配置ConfigurableEnvironment 
    this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach((Environment)environment);
  //这里会发一个ApplicationEnvironmentPreparedEvent事件，这样是不是和刚才的BootstrapApplicationListener联系上了。
	//BootstrapApplicationListener监听的就是ApplicationEnvironmentPreparedEvent事件啊，所以很显然这里会执行BootstrapApplicationListener的onApplicationEvent方法
    listeners.environmentPrepared((ConfigurableEnvironment)environment);
    this.bindToSpringApplication((ConfigurableEnvironment)environment);
    if (!this.isCustomEnvironment) {
        environment = (new EnvironmentConverter(this.getClassLoader())).convertEnvironmentIfNecessary((ConfigurableEnvironment)environment, this.deduceEnvironmentClass());
    }

    ConfigurationPropertySources.attach((Environment)environment);
    return (ConfigurableEnvironment)environment;
}
```



通过上面代码分析 **spring在初始化环境变量的时候会发送ApplicationEnvironmentPreparedEvent事件，而BootstrapApplicationListener又会监听ApplicationEnvironmentPreparedEvent这个事件**，所以在springboot容器没有创建之前 又到了BootstrapApplicationListener这个监听器这里。
接下来我们就看一下BootstrapApplicationListener.onApplicationEvent方法



```java
@Override
	public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
		ConfigurableEnvironment environment = event.getEnvironment();
		//可以看到还可以通过spring.cloud.bootstrap.enabled=false来关闭bootstrap容器
		//关闭了就不会实例化springcloud容器
		if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class,true)) {
			return;
		}
  //这里相当于是一个判断 容器只初始化一次 因为该监听器会执行多遍。为什么这个监听器会执行多遍？
  //因为第一次调用SpringApplication.run()会执行到创建环境变量的代码 在prepareEnvironment方法中会发送事件
  //但是当BootstrapApplicationListener监听到事件的时候又会通过SpringApplicationBuilder
  //来构建一个SpringApplication对象执行其run方法  所以该监听器 最少执行两次 执行几次根据你引入的包来决定不是绝对的 
  //如果环境变量中已经加载了bootstrap配置 那么久不会继续执行了		
   if(environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
			return;
		}
		ConfigurableApplicationContext context = null;
	//找到对应配置的名字 	spring.cloud.bootstrap.name:bootstrap  如果有配置
	//spring.cloud.bootstrap.name属性 则返回这个属性对应的值 没有返回默认值bootstrap
	//注意 这里从environment中取属性  此时的environment还没来得及加载spring的配置文件 所以
	//这个属性配置在系统属性或者 环境变量 或者servlet初始参数可以生效  在配置文件中不可能生效
		String configName = environment
				.resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
		//这里检查我们有没有在ApplicationContextInitializer中定制化容器 这个接口我们在前面提到 
		//是通过spring.factories加载进SpringApplication对象中去的 我们这里不做进一步的展开 后面再对
		//ApplicationContextInitializer做详细描述
		for (ApplicationContextInitializer<?> initializer :event.getSpringApplication()
				.getInitializers()) {
			if (initializer instanceof ParentContextApplicationContextInitializer) {
				context = findBootstrapContext(
						(ParentContextApplicationContextInitializer) initializer,
						configName);
			}
		}
		if (context == null) {
		//这个方法是核心方法 我们继续跟进该方法
			context = bootstrapServiceContext(environment, event.getSpringApplication(),
					configName);
			event.getSpringApplication()
					.addListeners(new CloseContextOnFailureApplicationListener(context));
		}

		apply(context, event.getSpringApplication(), environment);
	}


```



```java
    private ConfigurableApplicationContext bootstrapServiceContext(ConfigurableEnvironment environment, final SpringApplication application, String configName) {
	  //重新实例化一个Environment 但这个不是StandardServletEnvironment 我们通过之前的
        //Environment机制深入分析 知道environment变量其实是一个StandardServletEnvironment，这也很好理解 bootstrap容器不需要构建在web上面 
        //所以目前bootstrapEnvironment里面只有两个配置 一个是系统变量 还有一个是系统环境变量
        StandardEnvironment bootstrapEnvironment = new StandardEnvironment();
		MutablePropertySources bootstrapProperties = bootstrapEnvironment
				.getPropertySources();
				//这里是将系统变量和系统环境变量移除 因为不需要 最终bootstrapEnvironment是要和
				//最开始初始化的那个Environment合并的 系统变量和系统环境变量在Environment中已经存在
		for (PropertySource<?> source : bootstrapProperties) {
			bootstrapProperties.remove(source.getName());
		}
		//可以通过spring.cloud.bootstrap.location系统变量来设置bootstrap配置文件的位置
		String configLocation = environment
				.resolvePlaceholders("${spring.cloud.bootstrap.location:}");
		Map<String, Object> bootstrapMap = new HashMap<>();
		bootstrapMap.put("spring.config.name", configName);
		bootstrapMap.put("spring.main.web-application-type", "none");
		if (StringUtils.hasText(configLocation)) {
			bootstrapMap.put("spring.config.location", configLocation);
		}
		//向bootstrap环境变量 增加一个bootstrap配置
		bootstrapProperties.addFirst(
				new MapPropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME, bootstrapMap));
		//又将environment中的配置放入 创建的配置中 这里需要排除占位配置		
		for (PropertySource<?> source : environment.getPropertySources()) {
			if (source instanceof StubPropertySource) {
				continue;
			}
			bootstrapProperties.addLast(source);
		}
		//这里我们启动springboot应用差不多 只不过是没有打印banner 环境变量是自定义的环境变量
		//自定义环境变量任然可以参考我们之前关于spring环境变量的文章 非servlet容器 
		SpringApplicationBuilder builder = new SpringApplicationBuilder()
				.profiles(environment.getActiveProfiles()).bannerMode(Mode.OFF)
				.environment(bootstrapEnvironment)
				// Don't use the default properties in this builder
				.registerShutdownHook(false).logStartupInfo(false)
				.web(WebApplicationType.NONE);
		final SpringApplication builderApplication = builder.application();
		if (builderApplication.getMainApplicationClass() == null) {
			builder.main(application.getMainApplicationClass());
		}
		if (environment.getPropertySources().contains("refreshArgs")) {
			builderApplication
			.setListeners(filterListeners(builderApplication.getListeners()));
		}
		//这里可以稍稍注意一下 最终builder的sources 值会设置到SpringApplication的primarySources字段中  这里先不过多介绍我们后面文章会对这个展开
 		builder.sources(BootstrapImportSelectorConfiguration.class);
 		//这里其实就是容器创建刷新的流程 后面讲springboot容器的时候会涉及这里就不做展开
		final ConfigurableApplicationContext context = builder.run();
		context.setId("bootstrap");
		//为springboot容器添加父容器 这里你可能会问 到这里 springboot容器都还没开始创建呢？
		//是的 所以这里只是创建了一个AncestorInitializer加入到SpringApplication 等到springboot容器启动的时候会执行
		//SpringApplication中的所有的ApplicationContextInitializer 执行的时候自动就会把当前的容器设置为 父容器
		//内部实现为	new ParentContextApplicationContextInitializer(this.parent)
		//			.initialize(context);
		addAncestorInitializer(application, context);
		bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
		//这里实际是把两个environment合并 所以到这bootstrap配置文件内容已经成功的放入
		//environment
		mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
		return context;
}
```









