## bio、nio、io多路复用、aio

- bio：同步阻塞

当调用系统调用read时，用户线程会一直阻塞到内核空间有数据到来为止，否则就一直阻塞。

- nio：同步非阻塞

一个线程不断的轮询内核缓冲区的状态（数据到达没有），直至内核空间数据准备就绪，再把数据从内核空间复制到用户空间

- io多路复用

一个或几个线程调用select这个系统调用去查询是否有数据就绪的socket，如果有数据就绪，才调用read这个系统调用来读

- aio：异步非阻塞

当用户线程发起IO调用后，会立即返回，不会阻塞。而内核会在数据准备就绪后，将数据从内核空间复制到用户空间，并且会向用户线程发送一个信号或者执行用户线程注册的回调接口





本质：网络应用框架

实现：异步、事件驱动

##bio、nio、aio

bio：accept会阻塞（等待连接），read会阻塞（等待客户端发送数据）。在不考虑多线程的情况下，bio无法处理并发请求





# 阻塞、非阻塞

阻塞：没有数据传过来时，读操作会阻塞直到有数据，缓冲区满时，写操作也会阻塞

非阻塞遇到这些情况时，都是直接返回

# 同步异步

数据就绪后，数据操作（读取、写入）谁来完成

同步：数据就绪后（网卡到内核空间），需要自己去操作（内核空间到用户空间）是同步

异步：数据就绪后，系统直接操作好再回调给程序



# Reactor

reactor模式：注册感兴趣的事件-->多路复用器扫描是否有感兴趣的事件发生，事件发生后作出相应的处理



Reactor模式

Reactor单线程模式

```java
EventLoopGroup eventGroup = new NioEventLoopGroup(1);
ServerBootstrap b = new ServerBootstrap();
b.group(eventGroup);
```

非主从Reactor多线程模式

```java
EventLoopGroup eventGroup = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(eventGroup);
```

主从Reactor多线程模式

```java
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
```





## Tcp粘包、半包

粘包：

发送方每次写入数据  <  套接字缓冲区大小

接收方读取套接字缓冲区不够及时

半包：

发送方每次写入数据  >  套接字缓冲区大小

发生的数据大于协议的最大传输单元，必须拆包



**根本原因：tcp是流式协议，消息无边界**

解决

固定长度：FixedLengthFrameDecoder

分隔符：DelimiterBasedFrameDecoder

**固定长度字段存储内容的长度信息：LengthFieldBasedFrameDecoder（推荐）**



##codec

处理粘包、半包的codec

处理二次编解码的codec



处理业务的本质

数据在pipeline中所有的Handler的channelRead()执行过程

NioSocketChannel.read()是读数据，NioServerSocketChannel.read()是创建连接



## 组件

Bootstrap、ServerBootstrap：引导类

ChannelFuture：

Channel：

EventLoop：每个EventLoop都和唯一的一个Thread和Selector绑定，包含一个taskQueue，一个delayedTaskQueue，单个EventLoop可能会被指派用于服务多个Channel，一个 Channel 在它的生命周期内只能注册于一个 EventLoop，当一个连接到达时，Netty 就会注册一个 Channel，然后从 EventLoopGroup 中分配一个 EventLoop 绑定到这个Channel上，在该Channel的整个生命周期中都是有这个绑定的 EventLoop 来服务的，EventLoop 的职责是处理所有注册到本线程多路复用器 Selector 上的 Channel

ChannelHandler：

ChannelInboudHandler（入栈）：

ChannelOutboundHandler（出栈）：

ChannelPipeline：每个channel都有一个ChannelPipeline与之对应，ChannelPipeline中维护了一个由ChannleHandlerContext组成的双向链表，并且每个ChannleHandlerContext又关联着一个ChannelHandler



每个boss NioEventLoop循环执行的步骤：

1、轮训accept事件

2、处理accept事件，与client建立连接，生成NioSocketChannel，并将其注册到某个workor NioEventLoop上的selector

3、处理任务队列的任务，runAllTask()



每个work NioEventLoop循环执行的步骤有

1、轮训read、write事件

2、处理I/O事件，即read、write事件，在对应NioSocketChannel处理

3、处理任务队列的任务，runAllTask()

每个work NioEventLoop处理业务时，会使用pipeline，pipeline中包含了ChannelHandler









NioEventLoop 

executor：用于创建线程

taskQueue：任务队列

tailTasks：任务队列



Selector

io.netty.channel.nio.NioEventLoop#openSelector 替换SelectorImpl的两个属性值



NioEventLoopGroup

EventExecutor[] children：NioEventLoop









pipeline

ChannelHandlerContext







bind():















netty性能：

1、替换SelectorImpl中的属性   HashSet == > SelectedSelectionKeySet

2、解决java nio空轮训的bug

3、串行无锁化 inEventLoop() 当前事件循环是否由当前事件循环绑定的线程执行的

4、mask

5、堆外内存



NioServerSocketChannel的创建

<img src="images/NioServerSocketChannel继承关系.png" alt="NioServerSocketChannel继承关系" style="zoom:50%;" />









  

NioSocketChannel创建

<img src="images/NioSocketChannel继承关系.png" alt="NioSocketChannel继承关系" style="zoom:50%;" />



pipeline头尾节点没有ChannelHandler 







netty解码器

FixedLengthFrameDecoder:固定长度的解码器

LengthFieldBasedFrameDecoder：基于不定长的解码器

DelimiterBasedFrameDecoder：基于分隔符的解码器

LineBasedFrameDecoder：基于换行符的解码器





netty响应流程



ChannelHandlerContext.writeAndFlush("");从pipeline中的当前节点开始出栈     AbstractChannelHandlerContext#writeAndFlush()
ChannelHandlerContext.channel().writeAndFlush(""); 从pipeline的tail节点开始出栈    AbstractChannel#writeAndFlush()



netty写出去的数据一定是堆外内存













任务

springboot 整合netty



