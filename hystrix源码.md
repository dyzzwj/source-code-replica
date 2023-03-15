                                                                       hystrix源码

降级 熔断 限流

熔断状态  开启 关闭 半开

# 自动配置原理

@EnableHystrix

```java
@EnableCircuitBreaker
public @interface EnableHystrix {

}
```



```java
@Import(EnableCircuitBreakerImportSelector.class)
public @interface EnableCircuitBreaker {

}
```





```java
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableCircuitBreakerImportSelector
      extends SpringFactoryImportSelector<EnableCircuitBreaker> {
```



```java
public abstract class SpringFactoryImportSelector<T>
      implements DeferredImportSelector, BeanClassLoaderAware, EnvironmentAware {

   @SuppressWarnings("unchecked")
   protected SpringFactoryImportSelector() {
       //拿到的是@EnableCircuitBreaker
      this.annotationClass = (Class<T>) GenericTypeResolver
            .resolveTypeArgument(this.getClass(), SpringFactoryImportSelector.class);
   }

   @Override
   public String[] selectImports(AnnotationMetadata metadata) {
      if (!isEnabled()) {
         return new String[0];
      }
      AnnotationAttributes attributes = AnnotationAttributes.fromMap(
            metadata.getAnnotationAttributes(this.annotationClass.getName(), true));

      Assert.notNull(attributes, "No " + getSimpleName() + " attributes found. Is "
            + metadata.getClassName() + " annotated with @" + getSimpleName() + "?");

      //去META-INF/spring.factories文件中找key为EnableCircuitBreaker的配置项 == HystrixCircuitBreakerConfiguration
      List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader
            .loadFactoryNames(this.annotationClass, this.beanClassLoader)));

      if (factories.isEmpty() && !hasDefaultFactory()) {
         throw new IllegalStateException("Annotation @" + getSimpleName()
               + " found, but there are no implementations. Did you forget to include a starter?");
      }

      if (factories.size() > 1) {
         this.log.warn("More than one implementation " + "of @" + getSimpleName()
               + " (now relying on @Conditionals to pick one): " + factories);
      }

      return factories.toArray(new String[factories.size()]);
   }
}
```





HystrixCircuitBreakerConfiguration

```
@Configuration
public class HystrixCircuitBreakerConfiguration {
    //注入切面
	 @Bean
    public HystrixCommandAspect hystrixCommandAspect() {
        return new HystrixCommandAspect();
    }
}
```



## HystrixCommandAspect

切面类 拦截 @HystrixCommond

```java
@Aspect
public class HystrixCommandAspect {
    private static final Map<HystrixCommandAspect.HystrixPointcutType, HystrixCommandAspect.MetaHolderFactory> META_HOLDER_FACTORY_MAP;

    public HystrixCommandAspect() {
    }
	
	//拦截@HystrixCommand注解j
    @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand)")
    public void hystrixCommandAnnotationPointcut() {
    }
	//做请求合并的
    @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCollapser)")
    public void hystrixCollapserAnnotationPointcut() {
    }
    
    //环绕通知
    @Around("hystrixCommandAnnotationPointcut() || hystrixCollapserAnnotationPointcut()")
    public Object methodsAnnotatedWithHystrixCommand(ProceedingJoinPoint joinPoint) throws Throwable {
        //拿到拦截的方法对象
        Method method = AopUtils.getMethodFromTarget(joinPoint);
        //如果既有HystrixCommand注解，又有HystrixCollapser注解
        if (method.isAnnotationPresent(HystrixCommand.class) && method.isAnnotationPresent(HystrixCollapser.class)) {
            throw new IllegalStateException("method cannot be annotated with HystrixCommand and HystrixCollapser annotations at the same time");
        } else {
            //根据HystrixCommand或HystrixCollapser拿到对应的MetaHolderFactory
            HystrixCommandAspect.MetaHolderFactory metaHolderFactory = (HystrixCommandAspect.MetaHolderFactory)META_HOLDER_FACTORY_MAP.get(HystrixCommandAspect.HystrixPointcutType.of(method));
            //封装方法、参数、切入点、hystrixcommond的属性等信息
            MetaHolder metaHolder = metaHolderFactory.create(joinPoint);
            //封装成Commond
            HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
            ExecutionType executionType = metaHolder.isCollapserAnnotationPresent() ? metaHolder.getCollapserExecutionType() : metaHolder.getExecutionType();

            try {
                Object result;
                if (!metaHolder.isObservable()) {
                    //执行Commond
                    result = CommandExecutor.execute(invokable, executionType, metaHolder);
                } else {
                    result = this.executeObservable(invokable, executionType, metaHolder);
                }

                return result;
            } catch (HystrixBadRequestException var9) {
                throw var9.getCause();
            } catch (HystrixRuntimeException var10) {
                throw this.hystrixRuntimeExceptionToThrowable(metaHolder, var10);
            }
        }
    }
}
    
     private Observable executeObservable(HystrixInvokable invokable, ExecutionType executionType, final MetaHolder metaHolder) {
        return ((Observable)CommandExecutor.execute(invokable, executionType, metaHolder)).onErrorResumeNext(new Func1<Throwable, Observable>() {
            public Observable call(Throwable throwable) {
                if (throwable instanceof HystrixBadRequestException) {
                    return Observable.error(throwable.getCause());
                } else if (throwable instanceof HystrixRuntimeException) {
                    HystrixRuntimeException hystrixRuntimeException = (HystrixRuntimeException)throwable;
                    return Observable.error(HystrixCommandAspect.this.hystrixRuntimeExceptionToThrowable(metaHolder, hystrixRuntimeException));
                } else {
                    return Observable.error(throwable);
                }
            }
        });
    }   
    
```



###  metaHolderFactory.create(joinPoint);

com.netflix.hystrix.contrib.javanica.aop.aspectj.HystrixCommandAspect.MetaHolderFactory#create

```java
public MetaHolder create(ProceedingJoinPoint joinPoint) {
    Method method = AopUtils.getMethodFromTarget(joinPoint);
    Object obj = joinPoint.getTarget();
    Object[] args = joinPoint.getArgs();
    Object proxy = joinPoint.getThis();
    return this.create(proxy, method, obj, args, joinPoint);
}
```

com.netflix.hystrix.contrib.javanica.aop.aspectj.HystrixCommandAspect.CommandMetaHolderFactory#create

```java
public MetaHolder create(Object proxy, Method method, Object obj, Object[] args, ProceedingJoinPoint joinPoint) {
            HystrixCommand hystrixCommand = (HystrixCommand)method.getAnnotation(HystrixCommand.class);
            ExecutionType executionType = ExecutionType.getExecutionType(method.getReturnType());
            Builder builder = this.metaHolderBuilder(proxy, method, obj, args, joinPoint);
            if (EnvUtils.isCompileWeaving()) {
                builder.ajcMethod(HystrixCommandAspect.getAjcMethodFromTarget(joinPoint));
            }

            return builder.defaultCommandKey(method.getName()).hystrixCommand(hystrixCommand).observableExecutionMode(hystrixCommand.observableExecutionMode()).executionType(executionType).observable(ExecutionType.OBSERVABLE == executionType).build();
        }
```

###  HystrixCommandFactory.getInstance().create(metaHolder)

com.netflix.hystrix.contrib.javanica.command.HystrixCommandFactory#create

```java
public HystrixInvokable create(MetaHolder metaHolder) {
    Object executable;
    if (metaHolder.isCollapserAnnotationPresent()) {
        executable = new CommandCollapser(metaHolder);
    } else if (metaHolder.isObservable()) {
        executable = new GenericObservableCommand(HystrixCommandBuilderFactory.getInstance().create(metaHolder));
    } else {
        //走这里
        executable = new GenericCommand(HystrixCommandBuilderFactory.getInstance().create(metaHolder));
    }

    return (HystrixInvokable)executable;
}
```



```java
public class GenericCommand extends AbstractHystrixCommand<Object> {
    public GenericCommand(HystrixCommandBuilder builder) {
        super(builder);
    }
}
```

AbstractCommand的属性都是可以配置的

```java
protected AbstractCommand(HystrixCommandGroupKey group, HystrixCommandKey key, HystrixThreadPoolKey threadPoolKey, HystrixCircuitBreaker circuitBreaker, HystrixThreadPool threadPool, Setter commandPropertiesDefaults, com.netflix.hystrix.HystrixThreadPoolProperties.Setter threadPoolPropertiesDefaults, HystrixCommandMetrics metrics, AbstractCommand.TryableSemaphore fallbackSemaphore, AbstractCommand.TryableSemaphore executionSemaphore, HystrixPropertiesStrategy propertiesStrategy, HystrixCommandExecutionHook executionHook) {
    this.commandState = new AtomicReference(AbstractCommand.CommandState.NOT_STARTED);
    this.threadState = new AtomicReference(AbstractCommand.ThreadState.NOT_USING_THREAD);
    this.executionResult = ExecutionResult.EMPTY;
    this.isResponseFromCache = false;
    this.commandStartTimestamp = -1L;
    this.isCommandTimedOut = new AtomicReference(AbstractCommand.TimedOutStatus.NOT_EXECUTED);
    this.commandGroup = initGroupKey(group);
    this.commandKey = initCommandKey(key, this.getClass());
    this.properties = initCommandProperties(this.commandKey, propertiesStrategy, commandPropertiesDefaults);
    this.threadPoolKey = initThreadPoolKey(threadPoolKey, this.commandGroup, (String)this.properties.executionIsolationThreadPoolKeyOverride().get());
    this.metrics = initMetrics(metrics, this.commandGroup, this.threadPoolKey, this.commandKey, this.properties);
    //熔断器
    this.circuitBreaker = initCircuitBreaker((Boolean)this.properties.circuitBreakerEnabled().get(), circuitBreaker, this.commandGroup, this.commandKey, this.properties, this.metrics);
    //线程池
    this.threadPool = initThreadPool(threadPool, this.threadPoolKey, threadPoolPropertiesDefaults);
    this.eventNotifier = HystrixPlugins.getInstance().getEventNotifier();
    this.concurrencyStrategy = HystrixPlugins.getInstance().getConcurrencyStrategy();
    HystrixMetricsPublisherFactory.createOrRetrievePublisherForCommand(this.commandKey, this.commandGroup, this.metrics, this.circuitBreaker, this.properties);
    this.executionHook = initExecutionHook(executionHook);
    this.requestCache = HystrixRequestCache.getInstance(this.commandKey, this.concurrencyStrategy);
    this.currentRequestLog = initRequestLog((Boolean)this.properties.requestLogEnabled().get(), this.concurrencyStrategy);
    this.fallbackSemaphoreOverride = fallbackSemaphore;
    this.executionSemaphoreOverride = executionSemaphore;
}
```

com.netflix.hystrix.AbstractCommand#initCircuitBreaker

```java
private static HystrixCircuitBreaker initCircuitBreaker(boolean enabled, HystrixCircuitBreaker fromConstructor,
                                                        HystrixCommandGroupKey groupKey, HystrixCommandKey commandKey,
                                                        HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
    if (enabled) {//是否开启熔断器
        if (fromConstructor == null) {
            // get the default implementation of HystrixCircuitBreaker
            return HystrixCircuitBreaker.Factory.getInstance(commandKey, groupKey, properties, metrics);
        } else {
            return fromConstructor;
        }
    } else {
        //若关闭熔断器 返回一个没有任何操作的熔断器
        return new NoOpCircuitBreaker();
    }
}
```



com.netflix.hystrix.HystrixCircuitBreaker.Factory#getInstance()

```java
public static HystrixCircuitBreaker getInstance(HystrixCommandKey key, HystrixCommandGroupKey group, HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
    // this should find it for all but the first time
    //从缓存中拿熔断器
    HystrixCircuitBreaker previouslyCached = circuitBreakersByCommand.get(key.name());
    if (previouslyCached != null) {
        return previouslyCached;
    }
    HystrixCircuitBreaker cbForCommand = circuitBreakersByCommand.putIfAbsent(key.name(), new HystrixCircuitBreakerImpl(key, group, properties, metrics));
    if (cbForCommand == null) {
        // this means the putIfAbsent step just created a new one so let's retrieve and return it
        return circuitBreakersByCommand.get(key.name());
    } else {
      
        return cbForCommand;
    }
}
```

### HystrixCircuitBreakerImpl（熔断器实现）

熔断器的具体实现

```java
static class HystrixCircuitBreakerImpl implements HystrixCircuitBreaker {
    private final HystrixCommandProperties properties;
    private final HystrixCommandMetrics metrics;

    /* track whether this circuit is open/closed at any given point in time (default to false==closed) */  
    //描述熔断器开关的状态  true-开 false-关
    private AtomicBoolean circuitOpen = new AtomicBoolean(false);

    /* when the circuit was marked open or was last allowed to try a 'singleTest' */
    //上一次半开的时间 描述半开这种状态
    private AtomicLong circuitOpenedOrLastTestedTime = new AtomicLong();

    protected HystrixCircuitBreakerImpl(HystrixCommandKey key, HystrixCommandGroupKey commandGroup, HystrixCommandProperties properties, HystrixCommandMetrics metrics) {
        this.properties = properties;
        this.metrics = metrics;
    }

    public void markSuccess() {
        if (circuitOpen.get()) {
            if (circuitOpen.compareAndSet(true, false)) {
                //win the thread race to reset metrics
                //Unsubscribe from the current stream to reset the health counts stream.  This only affects the health counts view,
                //and all other metric consumers are unaffected by the reset
                metrics.resetStream();
            }
        }
    }
	
     //true：断路器关闭
    //false：断路器打开
    @Override
    public boolean allowRequest() {
        //判断是否开启强制打开断路器
        if (properties.circuitBreakerForceOpen().get()) {
            return false;
        }
        //判断是否强制关闭断路
        if (properties.circuitBreakerForceClosed().get()) {
            isOpen();
        
            return true;
        }
        //isOpen()：true-断路器打开 false-断路器关闭
        
        return !isOpen() || allowSingleTest();
        //如果断路器打开 就去判断是否半开
        //如果断路器关闭 就直接返回  不用判断半开状态
    }


    //true：断路器打开
    //false：断路器关闭
    @Override
    public boolean isOpen() {
        //如果打开 直接返回true
        if (circuitOpen.get()) {
            // if we're open we immediately return true and don't bother attempting to 'close' ourself as that is left to allowSingleTest and a subsequent successful test to close
            return true;
        }

        // we're closed, so let's see if errors have made us so we should trip the circuit open
        HealthCounts health = metrics.getHealthCounts();

        //判断调用次数是否小于配置的阈值 返回false 熔断器关闭 样本不够
        //注意 调用失败次数是否大于配置的阈值  不代表熔断器打开
        if (health.getTotalRequests() < properties.circuitBreakerRequestVolumeThreshold().get()) {
            // we are not past the minimum volume threshold for the statisticalWindow so we'll return false immediately and not calculate anything
            return false;
        }
				//失败的百分比小于配置的百分比阈值
        if (health.getErrorPercentage() < properties.circuitBreakerErrorThresholdPercentage().get()) {
            return false;
        } else {
            // our failure rate is too high, trip the circuit
            //使用cas打开开关
            if (circuitOpen.compareAndSet(false, true)) {
                // if the previousValue was false then we want to set the currentTime
                circuitOpenedOrLastTestedTime.set(System.currentTimeMillis());
                return true;
            } else {
                // How could previousValue be true? If another thread was going through this code at the same time a race-condition could have
                // caused another thread to set it to true already even though we were in the process of doing the same
                // In this case, we know the circuit is open, so let the other thread set the currentTime and report back that the circuit is open
                return true;
            }
        }
    }
    
    
    //true：断路器关闭
    //false：断路器打开
    public boolean allowSingleTest() {
        long timeCircuitOpenedOrWasLastTested = circuitOpenedOrLastTestedTime.get();
        //如果断路器是打开状态 并且从我们打开断路器已经过了circuitBreakerSleepWindowInMilliseconds时间
        //相当于每隔circuitBreakerSleepWindowInMilliseconds时间 就有一次半开
        //timeCircuitOpenedOrWasLastTested：记录上一次半开的时间
        if (circuitOpen.get() && System.currentTimeMillis() > timeCircuitOpenedOrWasLastTested + properties.circuitBreakerSleepWindowInMilliseconds().get()) {
            // We push the 'circuitOpenedTime' ahead by 'sleepWindow' since we have allowed one request to try.
            // If it succeeds the circuit will be closed, otherwise another singleTest will be allowed at the end of the 'sleepWindow'.
            if (circuitOpenedOrLastTestedTime.compareAndSet(timeCircuitOpenedOrWasLastTested, System.currentTimeMillis())) {
                // if this returns true that means we set the time so we'll return true to allow the singleTest
                // if it returned false it means another thread raced us and allowed the singleTest before we did
                return true;
            }
        }
        return false;
    }

}
```



### CommandExecutor.execute(invokable, executionType, metaHolder)

```java
public static Object execute(HystrixInvokable invokable, ExecutionType executionType, MetaHolder metaHolder) throws RuntimeException {
    Validate.notNull(invokable);
    Validate.notNull(metaHolder);

    switch (executionType) {
        //同步
        case SYNCHRONOUS: {
            return castToExecutable(invokable, executionType).execute();
        }
        //异步
        case ASYNCHRONOUS: {
            HystrixExecutable executable = castToExecutable(invokable, executionType);
            if (metaHolder.hasFallbackMethodCommand()
                    && ExecutionType.ASYNCHRONOUS == metaHolder.getFallbackExecutionType()) {
                return new FutureDecorator(executable.queue());
            }
            return executable.queue();
        }
        //发布订阅
        case OBSERVABLE: {
            HystrixObservable observable = castToObservable(invokable);
            return ObservableExecutionMode.EAGER == metaHolder.getObservableExecutionMode() ? observable.observe() : observable.toObservable();
        }
        default:
            throw new RuntimeException("unsupported execution type: " + executionType);
    }
}
```

com.netflix.hystrix.HystrixCommand#execute

```java
public R execute() {
        try {
            return queue().get();
        } catch (Exception e) {
            throw Exceptions.sneakyThrow(decomposeException(e));
        }
    }
```

com.netflix.hystrix.HystrixCommand#queue

```java
public Future<R> queue() {
     /*
      * The Future returned by Observable.toBlocking().toFuture() does not implement the
      * interruption of the execution thread when the "mayInterrupt" flag of Future.cancel(boolean) is set to true;
      * thus, to comply with the contract of Future, we must wrap around it.
      */
    //toObservable()：定义观察者（添加监听器） 
    //toFuture()：发布事件
     final Future<R> delegate = toObservable().toBlocking().toFuture();
   
    //....省略一些代码
```



```java
public Observable<R> toObservable() {
    final AbstractCommand<R> _cmd = this;

    //doOnCompleted handler already did all of the SUCCESS work
    //doOnError handler already did all of the FAILURE/TIMEOUT/REJECTION/BAD_REQUEST work
    final Action0 terminateCommandCleanup = new Action0() {

        @Override
        public void call() {
            if (_cmd.commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.TERMINAL)) {
                handleCommandEnd(false); //user code never ran
            } else if (_cmd.commandState.compareAndSet(CommandState.USER_CODE_EXECUTED, CommandState.TERMINAL)) {
                handleCommandEnd(true); //user code did run
            }
        }
    };

    //mark the command as CANCELLED and store the latency (in addition to standard cleanup)
    final Action0 unsubscribeCommandCleanup = new Action0() {
        @Override
        public void call() {
            if (_cmd.commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.UNSUBSCRIBED)) {
                if (!_cmd.executionResult.containsTerminalEvent()) {
                    _cmd.eventNotifier.markEvent(HystrixEventType.CANCELLED, _cmd.commandKey);
                    try {
                        executionHook.onUnsubscribe(_cmd);
                    } catch (Throwable hookEx) {
                        logger.warn("Error calling HystrixCommandExecutionHook.onUnsubscribe", hookEx);
                    }
                    _cmd.executionResultAtTimeOfCancellation = _cmd.executionResult
                            .addEvent((int) (System.currentTimeMillis() - _cmd.commandStartTimestamp), HystrixEventType.CANCELLED);
                }
                handleCommandEnd(false); //user code never ran
            } else if (_cmd.commandState.compareAndSet(CommandState.USER_CODE_EXECUTED, CommandState.UNSUBSCRIBED)) {
                if (!_cmd.executionResult.containsTerminalEvent()) {
                    _cmd.eventNotifier.markEvent(HystrixEventType.CANCELLED, _cmd.commandKey);
                    try {
                        executionHook.onUnsubscribe(_cmd);
                    } catch (Throwable hookEx) {
                        logger.warn("Error calling HystrixCommandExecutionHook.onUnsubscribe", hookEx);
                    }
                    _cmd.executionResultAtTimeOfCancellation = _cmd.executionResult
                            .addEvent((int) (System.currentTimeMillis() - _cmd.commandStartTimestamp), HystrixEventType.CANCELLED);
                }
                handleCommandEnd(true); //user code did run
            }
        }
    };
	//应用hystrix语义
   	
    final Func0<Observable<R>> applyHystrixSemantics = new Func0<Observable<R>>() {
        @Override
       
        public Observable<R> call() {
            if (commandState.get().equals(CommandState.UNSUBSCRIBED)) {
                return Observable.never();
            }
             //最后会调用到这里
            return applyHystrixSemantics(_cmd);
        }
    };

    final Func1<R, R> wrapWithAllOnNextHooks = new Func1<R, R>() {
        @Override
        public R call(R r) {
            R afterFirstApplication = r;

            try {
                afterFirstApplication = executionHook.onComplete(_cmd, r);
            } catch (Throwable hookEx) {
                logger.warn("Error calling HystrixCommandExecutionHook.onComplete", hookEx);
            }

            try {
                return executionHook.onEmit(_cmd, afterFirstApplication);
            } catch (Throwable hookEx) {
                logger.warn("Error calling HystrixCommandExecutionHook.onEmit", hookEx);
                return afterFirstApplication;
            }
        }
    };

    final Action0 fireOnCompletedHook = new Action0() {
        @Override
        public void call() {
            try {
                executionHook.onSuccess(_cmd);
            } catch (Throwable hookEx) {
                logger.warn("Error calling HystrixCommandExecutionHook.onSuccess", hookEx);
            }
        }
    };

    return Observable.defer(new Func0<Observable<R>>() {
        @Override
        public Observable<R> call() {
             /* this is a stateful object so can only be used once */
            if (!commandState.compareAndSet(CommandState.NOT_STARTED, CommandState.OBSERVABLE_CHAIN_CREATED)) {
                IllegalStateException ex = new IllegalStateException("This instance can only be executed once. Please instantiate a new instance.");
                //TODO make a new error type for this
                throw new HystrixRuntimeException(FailureType.BAD_REQUEST_EXCEPTION, _cmd.getClass(), getLogMessagePrefix() + " command executed multiple times - this is not permitted.", ex, null);
            }

            commandStartTimestamp = System.currentTimeMillis();

            if (properties.requestLogEnabled().get()) {
                // log this command execution regardless of what happened
                if (currentRequestLog != null) {
                    currentRequestLog.addExecutedCommand(_cmd);
                }
            }

            final boolean requestCacheEnabled = isRequestCachingEnabled();
            final String cacheKey = getCacheKey();

            /* try from cache first */
            if (requestCacheEnabled) {
                HystrixCommandResponseFromCache<R> fromCache = (HystrixCommandResponseFromCache<R>) requestCache.get(cacheKey);
                if (fromCache != null) {
                    isResponseFromCache = true;
                    return handleRequestCacheHitAndEmitValues(fromCache, _cmd);
                }
            }

            Observable<R> hystrixObservable =
                    Observable.defer(applyHystrixSemantics)
                            .map(wrapWithAllOnNextHooks);

            Observable<R> afterCache;

            // put in cache
            if (requestCacheEnabled && cacheKey != null) {
                // wrap it for caching
                HystrixCachedObservable<R> toCache = HystrixCachedObservable.from(hystrixObservable, _cmd);
                HystrixCommandResponseFromCache<R> fromCache = (HystrixCommandResponseFromCache<R>) requestCache.putIfAbsent(cacheKey, toCache);
                if (fromCache != null) {
                    // another thread beat us so we'll use the cached value instead
                    toCache.unsubscribe();
                    isResponseFromCache = true;
                    return handleRequestCacheHitAndEmitValues(fromCache, _cmd);
                } else {
                    // we just created an ObservableCommand so we cast and return it
                    afterCache = toCache.toObservable();
                }
            } else {
                afterCache = hystrixObservable;
            }

            return afterCache
                    .doOnTerminate(terminateCommandCleanup)     // perform cleanup once (either on normal terminal state (this line), or unsubscribe (next line))
                    .doOnUnsubscribe(unsubscribeCommandCleanup) // perform cleanup once
                    .doOnCompleted(fireOnCompletedHook);
        }
    });
}
```

com.netflix.hystrix.AbstractCommand#applyHystrixSemantics

```java
private Observable<R> applyHystrixSemantics(final AbstractCommand<R> _cmd) {
    // mark that we're starting execution on the ExecutionHook
    // if this hook throws an exception, then a fast-fail occurs with no fallback.  No state is left inconsistent
    //扩展点
    executionHook.onStart(_cmd);

    /* determine if we're allowed to execute */
    //判断断路器状态
    //true：断路器关闭
    //false：断路器打开
    if (circuitBreaker.allowRequest()) {
      	//判断当前策略是否为信号量
        final TryableSemaphore executionSemaphore = getExecutionSemaphore();
        final AtomicBoolean semaphoreHasBeenReleased = new AtomicBoolean(false);
        final Action0 singleSemaphoreRelease = new Action0() {
            @Override
            public void call() {
                if (semaphoreHasBeenReleased.compareAndSet(false, true)) {
                    executionSemaphore.release();
                }
            }
        };

        final Action1<Throwable> markExceptionThrown = new Action1<Throwable>() {
            @Override
            public void call(Throwable t) {
                eventNotifier.markEvent(HystrixEventType.EXCEPTION_THROWN, commandKey);
            }
        };
				//是否开启信号量资源隔离，未配置走 com.netflix.hystrix.AbstractCommand.TryableSemaphoreNoOp#tryAcquire 默认返回通过
        if (executionSemaphore.tryAcquire()) {
            try {
                /* used to track userThreadExecutionTime */
                executionResult = executionResult.setInvocationStartTime(System.currentTimeMillis());
              	//执行commond
                return executeCommandAndObserve(_cmd)
                        .doOnError(markExceptionThrown)
                        .doOnTerminate(singleSemaphoreRelease)
                        .doOnUnsubscribe(singleSemaphoreRelease);
            } catch (RuntimeException e) {
                return Observable.error(e);
            }
        } else {
            return handleSemaphoreRejectionViaFallback();
        }
    } else {
        //执行降级
        return handleShortCircuitViaFallback();
    }
}
```

判断当前策略是否为信号量

```java
protected AbstractCommand.TryableSemaphore getExecutionSemaphore() {
  	//如果是信号量隔离策略
    if (this.properties.executionIsolationStrategy().get() == ExecutionIsolationStrategy.SEMAPHORE) {
        if (this.executionSemaphoreOverride == null) {
            AbstractCommand.TryableSemaphore _s = (AbstractCommand.TryableSemaphore)executionSemaphorePerCircuit.get(this.commandKey.name());
            if (_s == null) {
                executionSemaphorePerCircuit.putIfAbsent(this.commandKey.name(), new AbstractCommand.TryableSemaphoreActual(this.properties.executionIsolationSemaphoreMaxConcurrentRequests()));
                return (AbstractCommand.TryableSemaphore)executionSemaphorePerCircuit.get(this.commandKey.name());
            } else {
                return _s;
            }
        } else {
            return this.executionSemaphoreOverride;
        }
    } else {
        return AbstractCommand.TryableSemaphoreNoOp.DEFAULT;
    }
}
```











com.netflix.hystrix.AbstractCommand#executeCommandAndObserve

执行命令 执行方法

```java
private Observable<R> executeCommandAndObserve(final AbstractCommand<R> _cmd) {
  
  　// Action和Func都是定义的一个动作，Action是无返回值，Func是有返回值
    final HystrixRequestContext currentRequestContext = HystrixRequestContext.getContextForCurrentThread();

  	// doOnNext中的回调。即命令执行之前执行的操作
    final Action1<R> markEmits = new Action1<R>() {
        @Override
        public void call(R r) {
            if (shouldOutputOnNextEvents()) {
                executionResult = executionResult.addEvent(HystrixEventType.EMIT);
                eventNotifier.markEvent(HystrixEventType.EMIT, commandKey);
            }
            if (commandIsScalar()) {
                long latency = System.currentTimeMillis() - executionResult.getStartTimestamp();
                eventNotifier.markCommandExecution(getCommandKey(), properties.executionIsolationStrategy().get(), (int) latency, executionResult.getOrderedList());
                eventNotifier.markEvent(HystrixEventType.SUCCESS, commandKey);
                executionResult = executionResult.addEvent((int) latency, HystrixEventType.SUCCESS);
                circuitBreaker.markSuccess();
            }
        }
    };
		// doOnCompleted中的回调。命令执行完毕后执行的操作
    final Action0 markOnCompleted = new Action0() {
        @Override
        public void call() {
            if (!commandIsScalar()) {
                long latency = System.currentTimeMillis() - executionResult.getStartTimestamp();
                eventNotifier.markCommandExecution(getCommandKey(), properties.executionIsolationStrategy().get(), (int) latency, executionResult.getOrderedList());
                eventNotifier.markEvent(HystrixEventType.SUCCESS, commandKey);
                executionResult = executionResult.addEvent((int) latency, HystrixEventType.SUCCESS);
                circuitBreaker.markSuccess();
            }
        }
    };

   // onErrorResumeNext中的回调。命令执行失败后的回退逻辑
    final Func1<Throwable, Observable<R>> handleFallback = new Func1<Throwable, Observable<R>>() {
        @Override
        public Observable<R> call(Throwable t) {
            Exception e = getExceptionFromThrowable(t);
            executionResult = executionResult.setExecutionException(e);
            if (e instanceof RejectedExecutionException) {
                return handleThreadPoolRejectionViaFallback(e);
            } else if (t instanceof HystrixTimeoutException) {
                return handleTimeoutViaFallback();
            } else if (t instanceof HystrixBadRequestException) {
                return handleBadRequestByEmittingError(e);
            } else {
                /*
                 * Treat HystrixBadRequestException from ExecutionHook like a plain HystrixBadRequestException.
                 */
                if (e instanceof HystrixBadRequestException) {
                    eventNotifier.markEvent(HystrixEventType.BAD_REQUEST, commandKey);
                    return Observable.error(e);
                }

                return handleFailureViaFallback(e);
            }
        }
    };
		// doOnEach中的回调。`Observable`每发射一个数据都会执行这个回调，设置请求上下文
    final Action1<Notification<? super R>> setRequestContext = new Action1<Notification<? super R>>() {
        @Override
        public void call(Notification<? super R> rNotification) {
            setRequestContextIfNeeded(currentRequestContext);
        }
    };

    Observable<R> execution;
 		 // 是否开启超时降级
    if (properties.executionTimeoutEnabled().get()) {
        //选择具体的策略（线程池隔离、信号量隔离）执行commond
        execution = executeCommandWithSpecifiedIsolation(_cmd)
                .lift(new HystrixObservableTimeoutOperator<R>(_cmd));
    } else {
        execution = executeCommandWithSpecifiedIsolation(_cmd);
    }
		// 发射
    return execution.doOnNext(markEmits)
            .doOnCompleted(markOnCompleted)
            .onErrorResumeNext(handleFallback)
            .doOnEach(setRequestContext);
}
```

#### 资源隔离

线程池隔离 执行命令使用的hystrix的线程

信号量隔离 执行命令使用tomcat的线程 hytrix控制执行当前commond的tomcat线程数



| 隔离方式   | 是否支持超时                                                 | 是否支持熔断                                                 | 隔离原理             | 是否是异步调用                         | 资源消耗                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------- | -------------------------------------- | -------------------------------------------- |
| 线程池隔离 | 支持，可直接返回                                             | 支持，当线程池到达maxSize后，再请求会触发fallback接口进行熔断 | 每个服务单独用线程池 | 可以是异步，也可以是同步。看调用的方法 | 大，大量线程的上下文切换，容易造成机器负载高 |
| 信号量隔离 | 不支持，如果阻塞，只能通过调用协议（如：socket超时才能返回） | 支持，当信号量达到maxConcurrentRequests后。再请求会触发fallback | 通过信号量的计数器   | 同步调用，不支持异步                   | 小，只是个计数器                             |



com.netflix.hystrix.AbstractCommand#executeCommandWithSpecifiedIsolation

```java
private Observable<R> executeCommandWithSpecifiedIsolation(final AbstractCommand<R> _cmd) {
 		 // 是否开启 THREAD 资源隔离降级
    if (properties.executionIsolationStrategy().get() == ExecutionIsolationStrategy.THREAD) {
        //线程池隔离
        return Observable.defer(new Func0<Observable<R>>() {
            @Override
            public Observable<R> call() {
                executionResult = executionResult.setExecutionOccurred();
                if (!commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.USER_CODE_EXECUTED)) {
                    return Observable.error(new IllegalStateException("execution attempted while in state : " + commandState.get().name()));
                }

                metrics.markCommandStart(commandKey, threadPoolKey, ExecutionIsolationStrategy.THREAD);

                if (isCommandTimedOut.get() == TimedOutStatus.TIMED_OUT) {
                    // the command timed out in the wrapping thread so we will return immediately
                    // and not increment any of the counters below or other such logic
                    return Observable.error(new RuntimeException("timed out before executing run()"));
                }
                if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.STARTED)) {
                    //we have not been unsubscribed, so should proceed
                    HystrixCounters.incrementGlobalConcurrentThreads();
                    threadPool.markThreadExecution();
                    // store the command that is being run
                    endCurrentThreadExecutingCommand = Hystrix.startCurrentThreadExecutingCommand(getCommandKey());
                    executionResult = executionResult.setExecutedInThread();
                    /**
                     * If any of these hooks throw an exception, then it appears as if the actual execution threw an error
                     */
                    try {
                        executionHook.onThreadStart(_cmd);
                        executionHook.onRunStart(_cmd);
                        executionHook.onExecutionStart(_cmd);
                        //执行commond
                        return getUserExecutionObservable(_cmd);
                    } catch (Throwable ex) {
                        return Observable.error(ex);
                    }
                } else {
                    //command has already been unsubscribed, so return immediately
                    return Observable.error(new RuntimeException("unsubscribed before executing run()"));
                }
            }
        }).doOnTerminate(new Action0() {
            @Override
            public void call() {
                if (threadState.compareAndSet(ThreadState.STARTED, ThreadState.TERMINAL)) {
                    handleThreadEnd(_cmd);
                }
                if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.TERMINAL)) {
                    //if it was never started and received terminal, then no need to clean up (I don't think this is possible)
                }
                //if it was unsubscribed, then other cleanup handled it
            }
        }).doOnUnsubscribe(new Action0() {
            @Override
            public void call() {
                if (threadState.compareAndSet(ThreadState.STARTED, ThreadState.UNSUBSCRIBED)) {
                    handleThreadEnd(_cmd);
                }
                if (threadState.compareAndSet(ThreadState.NOT_USING_THREAD, ThreadState.UNSUBSCRIBED)) {
                    //if it was never started and was cancelled, then no need to clean up
                }
                //if it was terminal, then other cleanup handled it
            }
        }).subscribeOn(threadPool.getScheduler(new Func0<Boolean>() {
            @Override
            public Boolean call() {
                return properties.executionIsolationThreadInterruptOnTimeout().get() && _cmd.isCommandTimedOut.get() == TimedOutStatus.TIMED_OUT;
            }
        }));
    } else {
        //信号量隔离
        return Observable.defer(new Func0<Observable<R>>() {
            @Override
            public Observable<R> call() {
                executionResult = executionResult.setExecutionOccurred();
                if (!commandState.compareAndSet(CommandState.OBSERVABLE_CHAIN_CREATED, CommandState.USER_CODE_EXECUTED)) {
                    return Observable.error(new IllegalStateException("execution attempted while in state : " + commandState.get().name()));
                }

                metrics.markCommandStart(commandKey, threadPoolKey, ExecutionIsolationStrategy.SEMAPHORE);
                
                endCurrentThreadExecutingCommand = Hystrix.startCurrentThreadExecutingCommand(getCommandKey());
                try {
                    executionHook.onRunStart(_cmd);
                    executionHook.onExecutionStart(_cmd);
                  	//真正执行
                    return getUserExecutionObservable(_cmd);  //the getUserExecutionObservable method already wraps sync exceptions, so this shouldn't throw
                } catch (Throwable ex) {
                    //If the above hooks throw, then use that as the result of the run method
                    return Observable.error(ex);
                }
            }
        });
    }
}
```





```java
private Observable<R> getUserExecutionObservable(AbstractCommand<R> _cmd) {
    Observable userObservable;
    try {
      	//执行
        userObservable = this.getExecutionObservable();
    } catch (Throwable var4) {
        userObservable = Observable.error(var4);
    }

    return userObservable.lift(new AbstractCommand.ExecutionHookApplication(_cmd)).lift(new AbstractCommand.DeprecatedOnRunHookApplication(_cmd));
}
```







```java
protected final Observable<R> getExecutionObservable() {
    return Observable.defer(new Func0<Observable<R>>() {
        public Observable<R> call() {
            try {
              	//执行
                return Observable.just(HystrixCommand.this.run());
            } catch (Throwable var2) {
                return Observable.error(var2);
            }
        }
    }).doOnSubscribe(new Action0() {
        public void call() {
            HystrixCommand.this.executionThread.set(Thread.currentThread());
        }
    });
}
```