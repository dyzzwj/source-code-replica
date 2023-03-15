

# mq

## 作用

异步、解耦、削峰

## 缺点

降低系统可用性、提高系统复杂度、降低数据一致性

## 顺序消费



## 重复消费



## 消息积压





## 消息的可靠性





# rabbitmq

Message（消息体）

Producer（消息生产者）

Consumer（消息消费者）

Virtual Host（虚拟节点）

Exchange（交换机）

- direct路由

  direct模式下Exchange将使用AMQP消息中所携带的Routing-Key和Queue中的Routing Key进行比较。如果两者**完全匹配**，就会将这条消息发送到这个Queue中



- fanout路由

Fanout路由模式不需要Routing Key。当设置为Fanout模式的Exchange收到AMQP消息后，将会将这个AMQP消息**复制多份**，分别发送到和自己绑定的各个Queue中



- topic路由

Topic模式是Routing Key的匹配模式。Exchange将支持使用‘#’和‘ * ’通配符进行Routing Key的匹配查找（**‘#’表示0个或若干个关键词，‘ \* ’表示一个关键词，注意是关键词不是字母**



**如果Exchange交换机没有找到任何匹配Routing Key的Queue，那么这条AMQP消息会被丢弃。**（只有Queue有保存消息的功能，但是Exchange并不负责保存消息）





Queue（队列）等





# rocketmq

## 消息类型

单向消息

同步消息

异步消息

顺序消息

延迟消息

事物消息

批量消息

`消息过滤`

`重试消息`



## 事物消息

![rocketmq事物消息](\images\rocketmq事物消息.webp)



## 顺序消息

1、生产者发送消息的有序性（MessageQueueSelector）

2、broker存储消息的有序性（顺序写）

3、consumer消费消息的有序性





单个messagequeue的消息严格有序。

## 延迟消息与消息重试





## 刷盘机制





## 消息存储

commitLog：存储消息内容,单个broker上所有topic在CommitLog中顺序写

consumequeue：存储消息索引，根据`topic`和`queueId`来组织文件，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值

IndexFile（索引文件）：只是为消息查询提供了一种通过key或时间区间来查询消息的方法









# kafka

生产者

拦截器->序列化器->分区器





当ISR中的个副本的LEO不一致时，如果此时leader挂掉，选举新的leader时并不是按照LEO的高低进行选举，而是按照ISR中的顺序选举。



## kafka的ack机制

1：只要集群的leader分区副本接收到了消息，就会向生产者发送一个成功响应的ack.

0：producer无需等待来自broker的确认而继续发送下一批消息

-1：producer需要等待ISR中的leader和所有follower都确认接收到数据后，生产者才会接收到来自服务器的响应，此时如果ISR同步副本的个数小于`min.insync.replicas`（设定ISR中的最小副本数是多少）的值，消息不会被写入



## kafka生产者分区策略

Partition接口  

1. 判断消息中的`partition`字段是否有值，有值的话即指定了分区，直接将该消息发送到指定的分区就行。
2. 如果没有指定分区，则使用分区器进行分区路由，首先判断消息中是否指定了`key`。
3. 如果指定了`key`，则使用该`key`进行hash操作，并转为正数，然后对`topic`对应的分区数量进行取模操作并返回一个分区。
4. 如果没有指定`key`，则通过先产生随机数，之后在该数上自增的方式产生一个数，并转为正数之后进行取余操作。



## partition

 leader处理partition的所有读写请求，follower会被动定期地去复制leader上的数据

2.4版本之前不支持从副本进行数据读取，2.4版本的新特性支持消费者从副本中读取数据。

为什么不支持？

- 数据一致性问题

- 延时问题，kafka follower副本同步数据需要经历网络→主节点内存→主节点磁盘→网络→从节 点内存→从节点磁盘





topic的分区数只能增加不能减少



## kafka选举

- broker选举

在Kafka集群中会有一个或多个broker，其中有一个broker会被选举为控制器,主要负责Partition管理和副本状态管理，也会执行类似于重分配partition之类的管理任务

Kafka Controller的选举是依赖Zookeeper来实现的，在Kafka集群中哪个broker能够成功创建/controller这个临时（EPHEMERAL）节点他就可以成为Kafka Controller。

- partition选举

分区leader副本的选举由Kafka Controller 负责具体实施，当**创建分区**（创建主题或增加分区都有创建分区的动作）或**分区上线**（比如分区中原先的leader副本下线，此时分区需要选举一个新的leader上线来对外提供服务）的时候都需要执行leader的选举动作。**按照AR集合中副本的顺序查找第一个存活的副本，并且这个副本在ISR集合中。**

一个分区的AR集合在分配的时候就被指定，并且只要不发生重分配的情况，集合内部副本的顺序是保持不变的，而分区的ISR集合中副本的顺序可能会改变

- 消费者组选举

组协调器GroupCoordinator需要为消费组内的消费者选举出一个消费组的leader，相当于随机



ISR





## 顺序消息

1、partition内部有序

2、broker顺序写

3、消费者单线程消费







## 事物消息



**kafak事物消息保证多个消息之间的事务约束，即多条消息要么都发送成功，要么都发送失败**





![kafka-transaction](\images\kafka-transaction.svg)





上图中的 Transaction Coordinator 运行在 Kafka 服务端，下面简称 TC 服务。

__transaction_state 是 TC 服务持久化事务信息的 topic 名称，下面简称事务 topic。

Producer 向 TC 服务发送的 commit 消息，下面简称事务提交消息。

TC 服务向分区发送的消息，下面简称事务结果消息。



**-     寻找 TC 服务地址     -** 

Producer 会首先从 Kafka 集群中选择任意一台机器，然后向其发送请求，获取 TC 服务的地址。Kafka 有个特殊的事务 topic，名称为__transaction_state ，负责持久化事务消息。这个 topic 有多个分区，默认有50个，每个分区负责一部分事务。事务划分是根据 transaction id， 计算出该事务属于哪个分区。这个分区的 leader 所在的机器，负责这个事务的TC 服务地址。



**-     事务初始化     -** 

Producer 在使用事务功能，必须先自定义一个唯一的 transaction id。有了 transaction id，即使客户端挂掉了，它重启后也能继续处理未完成的事务。

Kafka 实现事务需要依靠幂等性，而幂等性需要指定 producer id 。所以Producer在启动事务之前，需要向 TC 服务申请 producer id。TC 服务在分配 producer id 后，会将它持久化到事务 topic。



**-     发送消息     -** 

Producer 在接收到 producer id 后，就可以正常的发送消息了。不过发送消息之前，需要先将这些消息的分区地址，上传到 TC 服务。TC 服务会将这些分区地址持久化到事务 topic。然后 Producer 才会真正的发送消息，这些消息与普通消息不同，它们会有一个字段，表示自身是事务消息。

这里需要注意下一种特殊的请求，提交消费位置请求，用于原子性的从某个 topic 读取消息，并且发送消息到另外一个 topic。我们知道一般是消费者使用消费组订阅 topic，才会发送提交消费位置的请求，而这里是由 Producer 发送的。

Producer 首先会发送一条请求，里面会包含这个消费组对应的分区（每个消费组的消费位置都保存在 __consumer_offset topic 的一个分区里），TC 服务会将分区持久化之后，发送响应。Producer 收到响应后，就会直接发送消费位置请求给 GroupCoordinator。

**-     发送提交请求     -** 

Producer 发送完消息后，如果认为该事务可以提交了，就会发送提交请求到 TC 服务。Producer 的工作至此就完成了，接下来它只需要等待响应。这里需要强调下，Producer 会在发送事务提交请求之前，会等待之前所有的请求都已经发送并且响应成功。

**-     提交请求持久化     -** 

TC 服务收到事务提交请求后，会先将提交信息先持久化到事务 topic 。持久化成功后，服务端就立即发送成功响应给 Producer。然后找到该事务涉及到的所有分区，为每 个分区生成提交请求，存到队列里等待发送。

读者可能有所疑问，在一般的二阶段提交中，协调者需要收到所有参与者的响应后，才能判断此事务是否成功，最后才将结果返回给客户。那如果 TC 服务在发送响应给 Producer 后，还没来及向分区发送请求就挂掉了，那么 Kafka 是如何保证事务完成。因为每次事务的信息都会持久化，所以 TC 服务挂掉重新启动后，会先从 事务 topic 加载事务信息，如果发现只有事务提交信息，却没有后来的事务完成信息，说明存在事务结果信息没有提交到分区。

**-     发送事务结果信息给分区     -** 

后台线程会不停的从队列里，拉取请求并且发送到分区。当一个分区收到事务结果消息后，会将结果保存到分区里，并且返回成功响应到 TC服务。当 TC 服务收到所有分区的成功响应后，会持久化一条事务完成的消息到事务 topic。至此，一个完整的事务流程就完成了。





消息投递的三种语义







## kafka的消费者分区

### 消费者分区时机

- 组成员个数发生变化。例如有新的 consumer 实例加入该消费组或者离开组。
- 订阅的 Topic 个数发生变化。
- 订阅 Topic 的分区数发生变化。







- offset提交方式

自动提交
手动提交：   同步提交
            异步提交
            异步+手动提交



### range策略

**range策略针对于每个topic，各个topic之间分配时没有任何关联**，分配步骤如下：

1. topic下的所有有效分区平铺，例如P0, P1, P2, P3… …

2. 消费者按照字典排序，例如C0, C1, C2

3. 分区数除以消费者数，得到n

4. 分区数对消费者数取余，得到m

5. 消费者集合中，前m个消费者能够分配到n+1个分区，而剩余的消费者只能分配到n个分区



### roundrobin策略

**roundrobin策略针对于全局所有的topic和消费者**，分配步骤如下：

1. 消费者按照字典排序，例如C0, C1, C2… …，并构造环形迭代器。

2. topic名称按照字典排序，并得到每个topic的所有分区，从而得到所有分区集合。

3. 遍历第2步所有分区集合，同时轮询消费者。

4. 如果轮询到的消费者订阅的topic不包括当前遍历的分区所属topic，则跳过；否则分配给当前消费者，并继续第3步。





## kafka offset管理

Current Offset是针对Consumer的poll过程的，它可以保证每次poll都返回不重复的消息；而Committed Offset是用于Consumer Rebalance过程的，它能够保证新的Consumer能够从正确的位置开始消费一个partition，从而避免重复。

Current Offset成功提交后就是Committed Offset



在Kafka 0.9前，Committed Offset信息保存在zookeeper的[consumers/{group}/offsets/{topic}/{partition}]目录中（zookeeper其实并不适合进行大批量的读写操作，尤其是写操作）。

在0.9之后，所有的offset信息都保存在了Broker上的一个名为__consumer_offsets的topic中。

## offset提交

在 Consumer 正常消费中，可以利用 commitAsync 的自动重试来规避那些瞬时错误

在 Consumer 要关闭前，可以利用 commitSync() 方法执行同步阻塞式的 Offset 提交。commitSync() 方法会一直重试，直到提交成功或者发生无法恢复的错误。

```java
Properties props = new Properties();
props.setProperty("bootstrap.servers", "localhost:9092");
props.setProperty("group.id", "sync-async-commit-offset-example");
props.setProperty("enable.auto.commit", "false");
props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("behavior"));
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(200));
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("offset = %d, key = %s, value = %s\n", record.offset(), record.key(), record.value());
        }
        // 异步提交
        consumer.commitAsync(new OffsetCommitCallback() {
            @Override
            public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception e) {
                if (e != null) {
                    LOG.error("Commit failed for offsets {}", offsets, e);
                }
            }
        });
    }
} catch (Exception e) {
    LOG.error("consumer error", e);
} finally {
    try {
        // 同步提交
        consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```





## isr

Kafka 0.9.0.0版本及以后只支持：replica.lag.time.max.ms延迟时间

Kafka 0.9.0.0之前：replica.lag.time.max.ms延迟时间         replica.lag.max.messages延迟条数





## kafka文件存储

topic == partition是逻辑上的关系

partition==segment物理上的关系   每个partition(目录)被平均分配到多个大小相等的segment(段)数据文件中

segment == 由.log数据文件和.index索引文件组成  partition全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值







## LEO和HW

LEO：表示每个partition的log最后一条Message的位置。

HW：是指consumer能够看到的此partition的位置







## rocketmq与kafka比较



- 事物消息

  

- 延迟消息



- 消息过滤



- 消息重试

- 死信队列

  



- 服务发现

rocket mq nameserver存储topic-broker的关系数据

kafka broker通过zk实现服务发现 

- 高可用架构

rocketmq broker分为master和slave

kafka的broker基本是平等的



- 存储

kafka一个topic分为多个partition，一个partition分为多个固定大小的segment，每个segment对应.log和.index文件

rocketmq单个broker上所有topic在CommitLog中顺序写，基于 commitlog 文件构建消息消费队列文件(Consumequeue)，消息消费队列的组织结构按照 /topic/{queue} 进行组织



- 消费offset管理和offset提交

kafka在0.9版本之前把offset存储在zk上，0.9版本之后存储在Broker上的一个名为__consumer_offsets的topic中

rocketmq 广播模式消费进度存储在本地， 集群模式消费进度存储在broker  rocketmq offset无须手动提交



kafka可以手动提交，异步提交+同步提交

rocketmq只能自动提交



- 同步刷盘、异步刷盘  同步master和异步Master





- 消费请求处理

rocketmq的salve可以处理消费者的读请求 

kafka的follower只负责数据同步



- 消费者分配分区

kafka在消费者中选举出一个Leader，由leader来分配分区

rocketmq由各个消费者按照相同的分配算法分配自己的分区

