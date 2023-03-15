从源码的角度分析一个服务方法调用经历怎么样的磨难以后到达服务端



前一篇文章讲到的是引用服务的过程，引用服务无非就是创建出一个代理。供消费者调用服务的相关方法。本节将从调用方法开始讲解内部的整个调用链。我们就拿dubbo内部的例子讲。

```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring/dubbo-consumer.xml");
context.start();
DemoService demoService = context.getBean("demoService", DemoService.class);
String hello = demoService.sayHello("world");
System.out.println("result: " + hello);
```

这是dubbo-demo-xml-consumer内的实例代码。接下来我们就开始来看调用demoService.sayHello方法的时候，dubbo执行了哪些操作。

# 执行过程

## （一）InvokerInvocationHandler的invoke

```java
public class InvokerInvocationHandler implements InvocationHandler {
    private static final Logger logger = LoggerFactory.getLogger(InvokerInvocationHandler.class);
    private final Invoker<?> invoker;

    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 获得方法名
        String methodName = method.getName();
        // 获得参数类型
        Class<?>[] parameterTypes = method.getParameterTypes();
        // 如果方法参数类型是object类型，则直接反射调用
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        // 基础方法，不使用 RPC 调用
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }

        // 这里的recreate方法很重要，他会调用AppResponse的recreate方法，
        // 如果AppResponse对象中存在exception信息，则此方法中会throw这个异常
        /**
         * 如果调用某个方法 服务端出现了异常 如果是RuntimeException 服务端会将异常信息以字符串的方式返回
         * 客户端接受到请求后 判断服务端抛出了异常 将返回的字符串异常信息转化为Exception对象
         */
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
}
```

可以看到上面的源码，首先对Object的方法进行了处理，如果调用的方法不是这些方法，则先会 创建RpcInvocation，然后再调用invoke。

RpcInvocation的构造方法

```java
public RpcInvocation(Method method, Object[] arguments) {
    this(method.getName(), method.getParameterTypes(), arguments, null, null);
}
```



```java
public RpcInvocation(String methodName, Class<?>[] parameterTypes, Object[] arguments, Map<String, String> attachments, Invoker<?> invoker) {
    // 设置方法名
    this.methodName = methodName;
    // 设置参数类型
    this.parameterTypes = parameterTypes == null ? new Class<?>[0] : parameterTypes;
    // 设置参数
    this.arguments = arguments == null ? new Object[0] : arguments;
    // 设置附加值
    this.attachments = attachments == null ? new HashMap<String, String>() : attachments;
    // 设置invoker实体
    this.invoker = invoker;
}
```



创建完RpcInvocation后，就是调用invoke。先进入的是ListenerInvokerWrapper的invoke。

## （二）MockClusterInvoker的invoke

可以参考《dubbo源码解析（四十一）集群——Mock》的（二）MockClusterInvoker，降级后的返回策略的实现，根据配置的不同来决定不用降级还是强制服务降级还是失败后再服务降级。



## （三）AbstractClusterInvoker的invoke

可以参考《dubbo源码解析（三十五）集群——cluster》的（一）AbstractClusterInvoker，该类是一个抽象类，其中封装了一些公用的方法，AbstractClusterInvoker的invoke也只是做了一些公用操作。主要的逻辑在doInvoke中

## （四）FailoverClusterInvoker的doInvoke

可以参考《dubbo源码解析（三十五）集群——cluster》的（十二）FailoverClusterInvoker，该类实现了失败重试的容错策略。



## （五）InvokerWrapper的invoke

可以参考《dubbo源码解析（二十二）远程调用——Protocol》的（五）InvokerWrapper。该类用了装饰模式，不过并没有实现实际的功能增强。



## （六）ProtocolFilterWrapper的内部类CallbackRegistrationInvoker的invoke



```java
static class CallbackRegistrationInvoker<T> implements Invoker<T> {

    private final Invoker<T> filterInvoker;
    private final List<Filter> filters;

    public CallbackRegistrationInvoker(Invoker<T> filterInvoker, List<Filter> filters) {
        this.filterInvoker = filterInvoker;
        this.filters = filters;
    }

    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        // 执行过滤器链
        Result asyncResult = filterInvoker.invoke(invocation);

        // 过滤器都执行完了之后，回调每个ListenableFilter过滤器的onResponse或onError方法
        asyncResult = asyncResult.whenCompleteWithContext((r, t) -> {
            // 循环各个过滤器
            for (int i = filters.size() - 1; i >= 0; i--) {
                Filter filter = filters.get(i);
                // onResponse callback
                // 如果该过滤器是ListenableFilter类型的
                if (filter instanceof ListenableFilter) {
                    // 强制类型转化
                    Filter.Listener listener = ((ListenableFilter) filter).listener();
                    if (listener != null) {
                        if (t == null) {
                            // 如果内部类listener不为空，则调用回调方法onResponse
                            listener.onResponse(r, filterInvoker, invocation);
                        } else {
                            // 否则，直接调用filter的onResponse，做兼容。
                            listener.onError(t, filterInvoker, invocation);
                        }
                    }
                } else {
                    filter.onResponse(r, filterInvoker, invocation);
                }
            }
        });
        return asyncResult;
    }
}
```





这里看到先是调用拦截器链的invoke方法。下面的逻辑是把异步返回的结果放到上下文中，具体的ListenableFilter以及内部类的设计，还有thenApplyWithContext等方法我会在异步的实现中讲到。



## （七）ProtocolFilterWrapper的buildInvokerChain方法中的invoker实例的invoke方法。

```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {

    //构造链表
    Invoker<T> last = invoker;
    // 获得过滤器的所有符合条件的扩展实现类实例集合
    // 根据url获取filter，根据url中的parameters取key为key的value所对应的filter，但是还会匹配group
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

    if (!filters.isEmpty()) {
        // 从最后一个过滤器开始循环，创建一个带有过滤器链的invoker对象
        for (int i = filters.size() - 1; i >= 0; i--) {
            final Filter filter = filters.get(i);
            // 记录last的invoker
            final Invoker<T> next = last;
            // 新建last
            last = new Invoker<T>() {

                @Override
                public Class<T> getInterface() {
                    return invoker.getInterface();
                }

                @Override
                public URL getUrl() {
                    return invoker.getUrl();
                }

                @Override
                public boolean isAvailable() {
                    return invoker.isAvailable();
                }
                /**
                 * 关键在这里，调用下一个filter代表的invoker，把每一个过滤器串起来
                 * @param invocation
                 * @return
                 * @throws RpcException
                 */
                @Override
                public Result invoke(Invocation invocation) throws RpcException {
                    Result asyncResult;
                    try {
                        // 依次调用各个过滤器，获得最终的返回结果
                        asyncResult = filter.invoke(next, invocation);
                    } catch (Exception e) {
                        // onError callback
                        // 捕获异常，如果该过滤器是ListenableFilter类型的
                        if (filter instanceof ListenableFilter) {
                            // 获得内部类Listener
                            Filter.Listener listener = ((ListenableFilter) filter).listener();
                            if (listener != null) {
                                //调用onError，回调错误信息
                                listener.onError(e, invoker, invocation);
                            }
                        }
                        throw e;
                    }
                    return asyncResult;
                }

                @Override
                public void destroy() {
                    invoker.destroy();
                }

                @Override
                public String toString() {
                    return invoker.toString();
                }
            };
        }
    }

    return new CallbackRegistrationInvoker<>(last, filters);
}
```



## （八）ConsumerContextFilter的invoke

```java

@Activate(group = CONSUMER, order = -10000)
public class ConsumerContextFilter extends ListenableFilter {

    public ConsumerContextFilter() {
        super.listener = new ConsumerContextListener();
    }


    /**
     * RpcContext记录了一次调用状态信息，然后先调用后面的调用链，再回来把附加值设置到RpcContext中。然后返回RpcContext，再清空，这样是因为后面的调用链中的附加值对前面的调用链是不可见的。
     * @param invoker
     * @param invocation
     * @return
     * @throws RpcException
     */
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // 获得上下文，设置invoker，会话域，本地地址和原创地址
        RpcContext.getContext()
                .setInvoker(invoker)
                .setInvocation(invocation)
                .setLocalAddress(NetUtils.getLocalHost(), 0)
                .setRemoteAddress(invoker.getUrl().getHost(), invoker.getUrl().getPort())
                .setRemoteApplicationName(invoker.getUrl().getParameter(REMOTE_APPLICATION_KEY))
                .setAttachment(REMOTE_APPLICATION_KEY, invoker.getUrl().getParameter(APPLICATION_KEY));
        // 如果该会话域是rpc会话域
        if (invocation instanceof RpcInvocation) {
            // 设置实体域
            ((RpcInvocation) invocation).setInvoker(invoker);
        }
        try {
            // 移除服务端的上下文
            RpcContext.removeServerContext();
            // 调用下个调用链
            return invoker.invoke(invocation);
        } finally {
            RpcContext.removeContext();
        }
    }

    static class ConsumerContextListener implements Listener {
        @Override
        public void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
            // 设置附加值
            RpcContext.getServerContext().setAttachments(appResponse.getAttachments());
        }

        @Override
        public void onError(Throwable t, Invoker<?> invoker, Invocation invocation) {

        }
    }
}
```

可以参考《dubbo源码解析（二十）远程调用——Filter》，不过上面的源码是最新的，而链接内的源码是2.6.x的，虽然做了一些变化，比如内部类的的设计，后续的过滤器也有同样的实现，但是ConsumerContextFilter作用没有变化，它依旧是在当前的RpcContext中记录本地调用的一次状态信息。该过滤器执行完成后，会回到ProtocolFilterWrapper的invoke中的

```java
Result result = filter.invoke(next, invocation);
```

然后继续调用下一个过滤器FutureFilter。

## （九）FutureFilter的invoke

```java
public Result invoke(final Invoker<?> invoker, final Invocation invocation) throws RpcException {
    // 该方法是真正的调用方法的执行
    fireInvokeCallback(invoker, invocation);
    // need to configure if there's return value before the invocation in order to help invoker to judge if it's
    // necessary to return future.
    return invoker.invoke(invocation);
}
```



```java
class FutureListener implements Listener {
    @Override
    public void onResponse(Result result, Invoker<?> invoker, Invocation invocation) {
        if (result.hasException()) {
            // 如果有异常，则调用异常处理方法
            fireThrowCallback(invoker, invocation, result.getException());
        } else {
            // 如果正常的返回结果，则调用正常的处理方法
            fireReturnCallback(invoker, invocation, result.getValue());
        }
    }

    @Override
    public void onError(Throwable t, Invoker<?> invoker, Invocation invocation) {

    }
}
```



可以参考《dubbo源码解析（二十四）远程调用——dubbo协议》中的（十四）FutureFilter，其中会有部分结构不一样，跟ConsumerContextFilter一样，因为后续版本对Filter接口进行了新的设计，增加了onResponse方法，把返回的执行逻辑放到onResponse中去了。其他逻辑没有很大变化。等该过滤器执行完成后，还是回到ProtocolFilterWrapper的invoke中的，继续调用下一个过滤器MonitorFilter



## （十）MonitorFilter的invoke

```java
public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // 如果开启监控
        if (invoker.getUrl().hasParameter(MONITOR_KEY)) {
            // 设置监控开始时间
            invocation.setAttachment(MONITOR_FILTER_START_TIME, String.valueOf(System.currentTimeMillis()));
            // 方法的执行次数+1
            getConcurrent(invoker, invocation).incrementAndGet(); // count up
        }
        return invoker.invoke(invocation); // proceed invocation chain
    }
```

可以看到该过滤器实际用来做监控，监控服务的调用数量等。其中监控的逻辑不是本文重点，所以不细讲。接下来调用的是ListenerInvokerWrapper的invoke。

## （十一）ListenerInvokerWrapper的invoke

```java
public Result invoke(Invocation invocation) throws RpcException {
    // AsyncToSyncInvoker
    return invoker.invoke(invocation);
}
```

可以参考《dubbo源码解析（二十一）远程调用——Listener》，这里用到了装饰者模式，直接调用了invoker。该类里面做了服务启动的监听器。我们直接关注下一个invoke。

## （十二）AsyncToSyncInvoker的invoke

```java

public Result invoke(Invocation invocation) throws RpcException {
    // 异步转同步
    Result asyncResult = invoker.invoke(invocation);  // AsyncRpcResult--->CompletableFuture--->DefaultFuure

    try {
        // 如果invocation指定是同步的，则阻塞等待结果
        if (InvokeMode.SYNC == ((RpcInvocation) invocation).getInvokeMode()) {
            //阻塞等待
            asyncResult.get(Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
        }
    } catch (InterruptedException e) {
        throw new RpcException("Interrupted unexpectedly while waiting for remoting result to return!  method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (ExecutionException e) {
        Throwable t = e.getCause();
        if (t instanceof TimeoutException) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } else if (t instanceof RemotingException) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    } catch (Throwable e) {
        throw new RpcException(e.getMessage(), e);
    }
    return asyncResult;
}
```

AsyncToSyncInvoker类从名字上就很好理解，它的作用是把异步结果转化为同步结果。新的改动中每个调用只要不是oneway方式调用都会先以异步调用开始，然后根据配置的情况如果是同步调用，则会在这个类中进行异步结果转同步的处理。当然，这里先是执行了invoke，然后就进入下一个AbstractInvoker的invoke了。

## （十三）AbstractInvoker的invoke

```java
public Result invoke(Invocation inv) throws RpcException {
    // if invoker is destroyed due to address refresh from registry, let's allow the current invoke to proceed
    // 如果服务引用销毁，则打印告警日志，但是通过
    if (destroyed.get()) {
        logger.warn("Invoker for service " + this + " on consumer " + NetUtils.getLocalHost() + " is destroyed, "
                + ", dubbo version is " + Version.getVersion() + ", this invoker should not be used any longer");
    }
    RpcInvocation invocation = (RpcInvocation) inv;
    // 会话域中加入该调用链
    invocation.setInvoker(this);
    if (CollectionUtils.isNotEmptyMap(attachment)) {
        // 把附加值放入会话域
        invocation.addAttachmentsIfAbsent(attachment);
    }
    Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
    // 把上下文的附加值放入会话域
    if (CollectionUtils.isNotEmptyMap(contextAttachments)) {
       
        invocation.addAttachments(contextAttachments);
    }
    // 如果开启的是异步调用，则把该设置也放入附加值
    invocation.setInvokeMode(RpcUtils.getInvokeMode(url, invocation));
    // 加入编号
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

    try {
        // 执行调用链
        return doInvoke(invocation);
    } catch (InvocationTargetException e) { // biz exception
        // 获得异常
        Throwable te = e.getTargetException();
        if (te == null) {
            // 创建默认的异常异步结果
            return AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
        } else {
            if (te instanceof RpcException) {
                // 设置异常码
                ((RpcException) te).setCode(RpcException.BIZ_EXCEPTION);
            }
            // 创建默认的异常异步结果
            return AsyncRpcResult.newDefaultAsyncResult(null, te, invocation);
        }
    } catch (RpcException e) {
        if (e.isBiz()) {
            return AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
        } else {
            throw e;
        }
    } catch (Throwable e) {
        return AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
    }
}
```

可以参考《dubbo源码解析（二十二）远程调用——Protocol》的（三）AbstractInvoker。该方法做了一些公共的操作，比如服务引用销毁的检测，加入附加值，加入调用链实体域到会话域中等。然后执行了doInvoke抽象方法。各协议自己去实现。然后就是执行到doInvoke方法了。使用的协议不一样，doInvoke的逻辑也有所不同，我这里举的例子是使用dubbo协议，所以我就介绍DubboInvoker的doInvoke，其他自行查看具体的实现。此次的异步改造加入了InvokeMode，我会在后续中介绍这个

## （十四）DubboInvoker的doInvoke

```java
protected Result doInvoke(final Invocation invocation) throws Throwable {
    // rpc会话域
    RpcInvocation inv = (RpcInvocation) invocation;
    // 获得方法名
    final String methodName = RpcUtils.getMethodName(invocation);
    // 把path放入到附加值中
    inv.setAttachment(PATH_KEY, getUrl().getPath());
    // 把版本号放入到附加值
    inv.setAttachment(VERSION_KEY, version);

    // 一个DubboInvoker对象可能并发的同时去调用某个服务
    // 那么单独的一次调用都需要一个单独的client去发送请求
    // 所以这里会去选择使用本次调用该使用哪个client
    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        // 轮询使用clients
        currentClient = clients[index.getAndIncrement() % clients.length];
    }

    try {
        //是否是单向发送 isOneway为true，表示请求不需要拿结果
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        // 拿当前方法的所配置的超时时间，默认为1000,1秒
        int timeout = getUrl().getMethodPositiveParameter(methodName, TIMEOUT_KEY, DEFAULT_TIMEOUT);
        // 如果是单项发送
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            // 单向发送只负责发送消息，不等待服务端应答，所以没有返回值
            currentClient.send(inv, isSent);
            // 生成一个默认的值的结果，value=null
            return AsyncRpcResult.newDefaultAsyncResult(invocation);
        } else {
            // 异步调用
            AsyncRpcResult asyncRpcResult = new AsyncRpcResult(inv);
            // 异步去请求，得到一个CompletableFuture
            CompletableFuture<Object> responseFuture = currentClient.request(inv, timeout);

            // responseFuture会完成后会调用asyncRpcResult中的方法，这里并不会阻塞，如果要达到阻塞的效果在外层使用asyncRpcResult去控制
            asyncRpcResult.subscribeTo(responseFuture);
            // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
            FutureContext.getContext().setCompatibleFuture(responseFuture);
            return asyncRpcResult;
        }
    } catch (TimeoutException e) {
        throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (RemotingException e) {
        throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```



可以参考《dubbo源码解析（二十四）远程调用——dubbo协议》的（一）DubboInvoker，不过链接内的文章的源码是2.6.x版本的，而上述的源码是最新版本的，其中就有对于异步的改动，比如加入了异步返回结果、除了单向调用，一律都先处理成AsyncRpcResult等。具体的AsyncRpcResult以及其中用到的CompletableFuture我会在下文介绍。

上述源码中执行currentClient.request或者currentClient.send，代表把请求放入channel中，交给channel来处理请求。最后来看一个currentClient.request，因为这其中涉及到了Future的构建。



### （十五）ReferenceCountExchangeClient的request

```java
public CompletableFuture<Object> request(Object request) throws RemotingException {
    return client.request(request);
}
```



ReferenceCountExchangeClient是一个记录请求数的类，用了适配器模式，对ExchangeClient做了功能增强。

可以参考《dubbo源码解析（二十四）远程调用——dubbo协议》的（八）ReferenceCountExchangeClient



## （十六）HeaderExchangeClient的request

```java
public class HeaderExchangeClient implements ExchangeClient {
    public HeaderExchangeClient(Client client, boolean startTimer) {
        Assert.notNull(client, "Client can't be null");
        this.client = client;
        // 创建信息交换通道
        this.channel = new HeaderExchangeChannel(client);

        if (startTimer) {
            URL url = client.getUrl();
            /**
             * 定时重连定时任务
             */
            startReconnectTask(url);
            /**
             * 消费者定时发送心跳定时任务
             */
            startHeartBeatTask(url);
        }
    }
    
   public CompletableFuture<Object> request(Object request) throws RemotingException {
        //HeaderExchangeChannel
        return channel.request(request);
    }
    
}

```

该类也是用了适配器模式，该类主要的作用就是增加了心跳功能，可以参考《dubbo源码解析（十）远程通信——Exchange层》的（四）HeaderExchangeClient。然后进入HeaderExchangeChannel的request

## （十七）HeaderExchangeChannel的request

可以参考《dubbo源码解析（十）远程通信——Exchange层》的（二）HeaderExchangeChannel，在这个request方法中就可以看到

```java
/**
 *  为  检查请求超时 生成一个定时任务
 *  如果请求没有超时  在HeaderExchangeHandler#handleResponse()会取消这个定时任务
 */
DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout);
```

生成了需要的future。异步请求结果就是从这个future中获取。关于DefaultFuture也可以参考《dubbo源码解析（十）远程通信——Exchange层》的（七）DefaultFuture。

后面channel.send方法就是跟远程通信有关了，例如使用netty作为通信实现，则会使用netty实现的客户端进行通信。



## （十八）AbstractPeer的send

可以参考《dubbo源码解析（九）远程通信——Transport层》的（一）AbstractPeer，其中send方法比较简单，根据sent配置项去做消息发送。接下来看AbstractClient的send





## （十九）AbstractClient的send

可以参考《dubbo源码解析（九）远程通信——Transport层》的（四）AbstractClient

```java
public void send(Object message, boolean sent) throws RemotingException {
    // 如果需要重连或者没有链接，则连接
    if (needReconnect && !isConnected()) {
        connect();
    }
    //获得通道
    Channel channel = getChannel();
    //TODO Can the value returned by getChannel() be null? need improvement.
    if (channel == null || !channel.isConnected()) {
        throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
    }
    //sent值为true，等待消息发出，消息发送失败将抛出异常。
    //sent值为false，不等待消息发出，将消息放入 IO 队列，即刻返回。默认false
    // NettyChannel.send
    channel.send(message, sent);
}
```

该方法中做了重连的逻辑，然后就是通过通道发送消息，dubbo有几种通信的实现，我这里就按照默认的netty4实现来讲解，所以下一步走到了NettyChannel的send。



## （二十）NettyChannel的send

可以参考《dubbo源码解析（十七）远程通信——Netty4》的（一）NettyChannel。这里其中先执行了下面父类AbstractChannel的send，检查了一下通道是否关闭，然后再走下面的逻辑。当执行writeAndFlush方法后，消息就被发送。

dubbo数据包可以查看《dubbo源码解析（十）远程通信——Exchange层》的（二十五）ExchangeCodec，后续关于netty发送消息，以及netty出站数据在发出之前还需要进行编码操作我就先不做介绍，主要是跟netty知识点强相关，只是dubbo做了一些自己的编码，以及集成了各类序列化方式。



















