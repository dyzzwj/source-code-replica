





消息生产

单向消息 sendOneway

同步消息

异步消息

事务消息

延迟消息

顺序消息

消息标记







消息消费



消息消费方式：推 拉

消息消费模式：广播模式 集群模式



并发线程数设置

消息过滤



## 关注点

生产者启动流程
消息发送流程（单向消息、异步消息发送并发线程数，同步消息和异步消息如何实现）
broker处理发送请求

文件存储机制

nameserver启动流程

broker启动流程

消费者启动流程
消费者消费流程

broker处理生产和消费请求
消费者消费失败 如果是集群模式 消费者会将消息重发给broker broker对重发消息的处理流程（重试topic、死信队列）

顺序消费实现  提交消费offset
顺序消费者向broker请求分布式锁

broker HA
SYNC_MASTER和ASYNC_MASTER的实现 

SYNC_FLUSH和ASYNC_FLUSH的实现
延时消息 消息重试通过延时消息实现

事务消息 TransactionalMessageCheckService检查【Half消息】   事务消息存储 事务消息回查

consumequeue构建过程

消息拉取流程
创建mappedfile的两种方式 

主从同步



## neety线程模型





## broker HA

1. Producer：

- 和NameServer建立连接：获取相关配置信息，如Broker信息等
- 和Master，Slave建立连接：Producer向Master发送消息，消息只会向Master进行发送
2. Master：
- 和Producer建立连接：Producer向Master发送消息，消息只会向Broker进行发送
- 和NameServer建立连接：获取相关配置信息，如Slave获取Master信息等
- 和Consumer建立连接：Consumer向Master拉取消息
- 和Slave建立连接：消息同步
- Master之间不建立连接：多个Master之间相互独立
3. Slave：
- 和Master建立连接：消息同步

- 和NameServer建立连接：获取相关配置信息，如Slave获取Master信息等

- 和Consumer建立连接：Consumer向Slave拉取消息

  

- 4.consumer

- 和NameServer建立连接：获取相关配置信息，如Broker信息等

- 和Master、slave建立连接：消费者从master和slave上拉取消息

  





## Q&A

应用程序一 端  ，每个消费组，在同一台应用服务器只需要初始化一个消费者即可。

一个应用程序，一个消费组，只需要一个DefaultMQPushConsumerImpl,，在一个应用中，使用多线程创建多个

消费者，尝试去消费同一个组，没有效果，只会有一个消费者在消费



1、DefaultMQPushConsumerImpl 与PullMessageService关系

一个应用程序（消费端），一个消费组 一个 DefaultMQPushConsumerImpl ，同一个IP:端口，会有一个MQClientInstance ，而每一个MQClientInstance中持有一个PullMessageServive实例，故可以得出如下结论：同一个应用程序中，如果存在多个消费组，那么**多个DefaultMQPushConsumerImpl 的消息拉取，都需要依靠一个PullMessageServive**


2、PullMessageService.LinkedBlockQueue 中的PullRequest对象在什么时候放入的。

RebalanceService#run

就是向 PullMessageService 中放入 PullRequest,才会驱动 PullMessageSerivce run方法的运行，如果 pullRequestQueue 中没有元素，PullMessageService 线程将被阻塞





一个消费者非顺序消费者，内部使用一个线程池来并发消费消息，一个线程一批次最大处理consumeMessageBatchMaxSize条消息。



## 消息消费

### 非顺序消息消费

1、将待消费的消息存入ProcessQueue中存储，并执行消息消费之前钩子函数
2、修改待消费消息的主题（设置为消费组的重试主题）
3、分页消费（每次传给业务消费监听器的最大数量为配置的
sconsumeMessageBatchMaxSize
4、执行消费后钩子函数，并根据业务方返回的消息消费结果（成功，重试）【ACK】确认信息，然后更新消息进度，从ProceeQueue中删除相应的消息





### 顺序消费消费



顺序消费涉及三个点：

1、消息队列负载时，对于移除的队列，如果是集群模式下 并且 是顺序消费， 释放对队列的分布式锁。对于新增的队列，获取messagequeue的分布式锁

2、消息拉取时，只有当前消费者持有该messagequeue的分布式锁 才会拉取该messagequeue

3、消息消费时

Consumer 在严格顺序消费时，通过 三 把锁保证严格顺序消费。（与kafka的区别，支持一个顺序消费者多线程顺序消费）

（1）Broker 消息队列锁（分布式锁） ：
集群模式下，Consumer 从 Broker 获得该锁后，才能进行消息拉取、消费。
广播模式下，Consumer 无需该锁。
（2）Consumer 消息队列锁（本地锁） ：Consumer 获得该锁才能操作消息队列。
（3）Consumer 消息处理队列消费锁（本地锁） ：Consumer 获得该锁才能消费消息队列（粒度更小），使得一个MessageQueue同一个时刻只能被一个消费客户端中一个线程消费（多线程也可以实现顺序消费）

为什么有Consumer消息队列锁还需要有 Consumer 消息队列消费锁呢？

- 消息队列的操作有多种 消费消息队列只是其中一种操作 粒度更小

不同消费组的消费者可以同时锁定同一个消息消费队列，集群模式下同一个消费组内只能被一个消费者锁定





**只有Push模式才有顺序消费  Pull模式不支持顺序消费**



一个消费者中线程池中线程的锁粒度为，MessageQueue,消费队列，也就是说RocketMQ实现顺序消费是针对MessageQueue,也就是RocketMQ无法做到多MessageQueue的全局顺序消费，如果要使用RocketMQ做的主题的全局顺序消费，那该主题只能允许一个队列






## Rebalance


一个消费者可以订阅多个topic
一个消费者只属于一个消费者组


有新的消费者时，会进行负载均衡

每个Consumer都会向所有的broker发送心跳
每个消费者或生产者都对应一个Mqclient


Consumer发送心跳：
1、订阅主题时
2、启动时
3、定时发送心跳


消费者组下的多个实例 自己给自己分配d队列 如何保证分配结果的一致性？？
1、在分配之前，需要对Topic下的多个队列进行排序，对多个消费者实例按照id进行排序
                         
2、每个消费者需要使用相同的分配策略。
                          
尽管每个消费者是各自给自己分配，但是因为使用的相同的分配策略，定位从队列列表中哪个位置开始给自己分配，给自己分配多少个队列，从而保证最终分配结果的一致。
                         







  将一个Topic下的多个队列(或称之为分区)，在同一个消费者组(consumer group)下的多个消费者实例(consumer instance)之间进行重新分配

```java
DefaultMQPushConsumerImpl  - RebalancePushImpl
DefaultMQPullConsumer - RebalancePullImpl
```







   触发rebanlance :
   1、订阅Topic的队列数量变化
       broker宕机
       broker升级等运维操作
       队列扩容/缩容
   2、消费者组信息变化  -  消费者组实例信息  消费者组订阅信息（包括订阅的topic、过滤条件、消费类型（pull push）、消费模式、从什么位置开始消费）
      日常发布过程中的停止与启动
       消费者异常宕机
      网络异常导致消费者与Broker断开连接
      主动进行消费者数量扩容/缩容
      Topic订阅信息发生变化


Broker是通知每个消费者各自Rebalance，即每个消费者自己给自己重新分配队列，而不是Broker将分配好的结果告知Consumer

从这个角度，RocketMQ与Kafka Rebalance机制类似，二者Rebalance分配都是在客户端进行，不同的是：

Kafka：会在消费者组的多个消费者实例中，选出一个作为Group Leader，由这个Group Leader来进行分区分配，分配结果通过Cordinator(特殊角色的broker)同步给其他消费者。相当于Kafka的分区分配只有一个大脑，就是Group Leader。

RocketMQ：每个消费者，自己负责给自己分配队列，相当于每个消费者都是一个大脑。


然而需要注意的是，只要任意一个消费者组需要Rebalance，这台机器上启动的所有其他消费者，也都要进行Rebalance。

RocketMQ按照Topic维度进行Rebalance，会导致一个很严重的结果：如果一个消费者组订阅多个Topic，可能会出现分配不均。
Kafka与RocketMQ不同，其是将组内订阅的所有Topic下的所有队列合并在一起，进行Rebalance。(也要看kafka使用的分区策略range或roundrobin)

由于订阅多个Topic时可能会出现分配不均，这是在RocketMQ中我们为什么不建议同一个消费者组订阅多个Topic的重要原因。另一款消息中间件Kafka会将所有Topic队列合并在一起，然后在消费者中进行分配，因此相对会更加平均









消费者发送心跳时上送的topic信息是否包含组内其他消费者订阅的topic而自己没有订阅  ？？？








1、DefaultMQPushConsumerImpl 与PullMessageService关系
2、PullMessageService中的LinkedBlockQueue 中的PullRequest对象在什么时候放入的。
在这里先解决都一个疑问：
我们知道，一个应用程序（消费端），一个消费组 一个 DefaultMQPushConsumerImpl ，同一个IP:端口，会有一个MQClientInstance ，而每一个MQClientInstance中持有一个PullMessageServive实例，故可以得出如下结论：同一个应用程序中，如果存在多个消费组，那么多个DefaultMQPushConsumerImpl 的消息拉取，都需要依靠一个PullMessageServive。那他们之间又是如何协作的呢





























## mappedFile

创建mappedFile有两种不同的方式
方式一：
mappedFile = ServiceLoader.load(MappedFile.class).iterator().next();
mappedFile.init(req.getFilePath(), req.getFileSize(), messageStore.getTransientStorePool());
这种方式是在broker的配置文件中刷盘方式是异步刷盘并且TransientStorePoolEnable为true的情况下生效，该方式下MappedFile 会将向TransientStorePool 申请的堆外内存（Direct ByteBuffer）空间作为 writeBuffer，写入消息时先将消息写入 writeBuffer，然后将消息提交至 fileChannel 最后再 flush。
方式二：
mappedFile = new MappedFile(req.getFilePath(), req.getFileSize());
这种方式是直接创建 MappedFile 内存映射文件字节缓冲区mappedByteBuffer，将消息写入 mappedByteBuffer 再 flush。
![mappedfile对应的刷盘方式](images/mappedfile对应的刷盘方式.png)





同步刷盘和异步刷盘

同步刷盘

<img src="images/rocketmq同步刷盘.png" alt="rocketmq同步刷盘" style="zoom:80%;" />







异步刷盘

1）transientStorePoolEnable = true

消息在追加时，先放入到 writeBuffer 中，然后定时 commit 到 FileChannel,然后定时flush。

2）transientStorePoolEnable=false（默认）

消息追加时，直接存入 MappedByteBuffer(pageCache) 中，然后定时 flush。


![rocketmq异步刷盘](images/rocketmq异步刷盘.png)







## 消息过滤

消息过滤有两种模式

类过滤classFilterMode,表达式模式(Expression),又分为 ExpressionType.TAG 和 ExpressionType.SQL92。
TAG过滤，在服务端拉取时，会根据 ConsumeQueue 条目中存储的 tag hashcode 与订阅的 tag (hashcode 集合)进行匹配，匹配成功则放入待返回消息结果中，然后在消息消费端（消费者，还会对消息的订阅消息字符串进行再一次过滤。为什么需要进行两次过滤呢？为什么不在服务端直接对消息订阅 tag 进行匹配呢？主要就还是为了提高服务端消费消费队列（文件存储）的性能，如果直接进行字符串匹配，那么 consumequeue 条目就无法设置为定长结构，检索 consuequeue 就不方便。



推拉模式

PUSH 模式可以说基于订阅与发布模式，而PULL模式可以说是基于消息队列模式

特别说明：PULL模式根据主题注册消息监听器，这里的消息监听器，不是用来消息消费的，而是在该主题的队列负载发生变化时，做一下通知。（区别于PUSH模式的消息监听器是用来消费消息的）

> DefaultMQPullConsumerImpl 可以注册多个主题，但多个主题使用同一个消息处理监听器。



**PullMessageService 只为 PUSH 模式服务**，ReblanceService进行路由重新分布时，如果是 RebalancePullImpl, 并不会产 PullRequest（RebalanceImpl#dispatchPullRequest），从而唤醒PullMessageService，PullMessageService 被唤醒后，也是执行 DefaultMQPushConsumerImpl 的 pullMessage 方法。

ReblanceService 线程默认每 20s 进行一次消息队列重新负载，判断消息队列是否需要进行重新分布（如果消费者个数和主题的队列数没有发生改变），则继续保持原样。对于 PULL 模型，如果消费者需要监听某些主题队列发生事件，注册消息队列变更事件方法，则 RebalanceService 会将消息队列负载变化事件通知消费者。(RebalanceImpl#messageQueueChanged)





rocketmq  HA









rocketmq读写分离



消息发送（写） ：master

消息消费（读）：默认为master，只有当master消息拉取出现堆积时才将拉取任务转向slave（默认编号为1的slave，通过客户端命令updateSubGroup配置当主服务器繁忙时，建议从哪个slave读取消息）。







