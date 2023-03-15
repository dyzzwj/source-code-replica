zookeeper																	

基于zookeeper-3.6.1





# 1、节点类型

## 1.1临时节点

create -e 

- 临时节点的生命周期和客户端会话绑定，如果客户端会话失效，那么这个节点就会自动被清除掉。注意，这里提到的是会话失效，而非连接断开。

- 临时节点下面不能创建子节点

- 当前客户端创建的临时节点对其他客户端是可见的







## 1.2临时顺序节点

create -e -s



## 1.3持久节点

create 



## 1.4持久顺序节点

create -s



## 1.5容器节点

create -c

当节点的最后一个子节点被删除后，容器节点会自动被删除（有点延时）。一个容器节点只有创建过子节点才会被删除



## 1.6ttl节点

create -t 3000 /aaa

zkserver维护一个定时任务 定时删除过期节点(区别于redis的惰性删除)



## 1.7ttl顺序节点





# 2、watch机制

watch监听同客户端会话绑定

当前客户端注册的监听器的生命周期同当前客户端生命周期绑定 当前客户端挂了 监听器就失效了  服务端挂了 当前客户端注册的监听器并不会失效（重连并重新注册）



一次性监听

永久性监听

永久性递归监听





# 3、应用场景



## 3.1分布式配置中心

watch机制

## 3.2分布式锁

临时顺序节点 

watch机制：当前节点监听前一个节点



## 3.3注册中心

临时节点

watch机制







## 3.4集群管理





# 4、zookeeper启动流程 、单机模式请求处理流程

## 4.1单机zkServer启动流程：



启动类 ZooKeeperServerMain

1、解析配置文件 根据配置文件生成ServerConfig

2、初始化FileTxnSnapLog 快照和日志操作工具类

3、初始化jvm监控器 监听jvm是否暂停

4、初始化zkServer

5、初始化adminServer 基于jetty启动一个管理台

6、创建NIOServerCnxnFactory对象 

7、创建ServerSockerChannle，绑定地址和端口 创建accept线程  向ServerSocketChannel注册accept事件

8、初始化workService工作线程池

9、启动selector线程

10、启动acceptor接受连接线程

11、根据快照和日志初始化zkDatabase

12、启动session跟踪器sessionTracker

13、初始化Request processor chain请求处理器链

14、创建requestThrottler

 

## 4.2 Zk server处理请求流程：

PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor

Prep:

1. 以线程+队列的方式处理Request，保证了顺序执行
2. 设置Request的hdr和txn（为后面的持久化做准备）
3. 根据不同的命令操作类型，生成ChangeRecord
4. 验证acl
5. 把Request交给nextPorcessor处理



Sync:

1. 以线程+队列的方式处理Request，保证了顺序执行
2. 把Request中的hsr和txn进行持久化（写请求才会进行持久化）
3. 单独开启一个线程打快照
4. 把Request交给nextPorcessor处理

Final:

1. 根据Request更新zkDatabase
2. 触发watch
3. 发送Response给客户端



## 4.3为什么需要ChangeRecord呢？

ChangeRecord表示修改记录，表示某个节点的修改记录，在处理Request时，需要依赖现有节点上的已有信息，比如cversion（某个节点的孩子节点版本），比如，在处理一个create请求时，需要修改父节点上的cversion（加1），那么这个信息从哪来呢？一开始肯定是从DataTree上来，但是不能每次都从DataTree上来获取父节点的信息，这样性能很慢，比如ZooKeeperServer连续收到两个create请求，当某个create请求在被处理时，都需要先从DataTree获取信息，然后持久化，然后更新DataTree，最后才能处理下一个create请求，是一个串行的过程，那么如果第二个create不合法呢？依照上面的思路，则还需要等待第一个create请求处理完了之后才能对第二个请求进行验证，所以Zookeeper为了解决这个问题，在PrepRequestProcessor中，每验证完一个请求，就把这个请求异步的交给持久化线程来处理，PrepRequestProcessor自己就去处理下一个请求了，打断了串行的链路，但是这时又出现了问题，因为在处理第二个create请求时需要依赖父节点的信息，并且应该处理过第一个create请求后的结果，所以这时就引入了ChangeRecord，PrepRequestProcessor在处理第一个create请求时，先生成一条ChangeRecord记录，然后再异步的去持久化和更新DataTree，然后立即去处理第二个create请求，此时就可以不需要去取DataTree中的信息了（就算取了，可能取到的信息也不对），就直接取ChangeRecord中的信息就可以了。



# 5、zk Client启动、和服务端交互流程

启动 建立连接（连接建立过程） 发送connectRequest  SendThread  EventThread 

重连

和服务端交互

发送请求 （同步请求、异步请求  顺序）   服务端对请求的处理逻辑

接收响应



# 6、watch机制

注册  

服务端注册

客户端注册
触发

服务端触发

客户端触发











# 7、session跟踪机制

Session：watch 临时节点ss

## 7.1session的创建流程

sessionTracker:

k:sessionId k:SessionImpl
protected final ConcurrentHashMap<Long, SessionImpl> sessionsById;

k:SessionImpl   v:过期时间点
 private final ConcurrentHashMap<E, Long> elemMap = new ConcurrentHashMap<E, Long>();

k:过期时间点 v：Set<SessionImpl>  某一个过期时间点对应的SessionImpl集合
 private final ConcurrentHashMap<Long, Set<E>> expiryMap = new ConcurrentHashMap<Long, Set<E>>();



## 7.2session的刷新流程 

服务端只要接收到客户端的请求就刷新（更新过期时间点）

org.apache.zookeeper.server.NIOServerCnxnFactory#touchCnxn



## 7.3session的过期

sessionTracker的run方法会清理过期session 模拟一个closeSession  Request



具体处理逻辑见Prep Final 





清空三个map对应的sessionImpl    org.apache.zookeeper.server.ZooKeeperServer#processTxnForSessionEvents 

处理临时节点  org.apache.zookeeper.server.DataTree#processTxn   搜索closeSession

移除watcher  关闭socket   org.apache.zookeeper.server.NIOServerCnxn#close()





客户端正常关闭或异常关闭 socket连接  watcher会马上被删除 而临时节点不会马上被删除（临时节点受制于session的过期机制，socket断掉了 session并不一定马上失效 ）

客户端进行重连时 会重新注册watcher

临时节点只会在sessionTracker 发现session过期 时才会被删除

## 7.4session的恢复流程 

session信息也会达打到快照中 k:sessionId v:sessionTimeout

服务启动时 加载快照文件中session信息  比对datatree中的session和快照文件中的session 关闭无效session及删除对应临时节点

创建sessiontracker时 也会根据快照文件中的session信息初始化三个map





# 8、zookeeper集群介绍与zab协议解析

集群模式启动类QuorumPeerMain



强一致性 弱一致性 最终一致性

zk保证最终一致性

1、Leader（领导者选举）

2、过半机制(防止脑裂)

3、两阶段提交(2PC) -- 只能保证最终一致性 尽量保证强一致性

4、同步（节点启动时向leader同步数据）



集群模式下  每个zk server节点必须先确定自己的角色 才能接受客户端请求 

领导者选举出现的场景：

1、集群启动时

2、leader挂掉

3、超过一半（参与两阶段提交）的节点挂掉了



新的节点启动时也会进行领导者选举

两种情况：

1、集群已经有leader，当前节点还是会进行投票，其他节点也会把本节点的投票信息（投给leader）发送给当前节点 快速完成投票

2、集群中还没leader



**集群数量 是奇数台   5个节点要保证集群可用性只能挂掉2个  6个节点要保证集群可用性也只能挂掉2个**



两阶段提交

1、leader接收到写请求 生成Request和zxid

2、持久化Request日志

3、prepare预提交（发送Request日志到follower）

4、follower持久化request 持久化完成发送ack

5、leader接收到ack 过半机制

6、leader向follower发送commit提交(异步)  向Observer节点通知当前已经可以提交的提议（异步）    follower收到commit修改DataTree

7、leader修改DataTree

8、leader响应给客户端写成功



集群角色

leader

follower

Observer：不参与选举 不参与两阶段提交（commit时参与）









# 9、zookeeper快速领导者选举算法

9.1启动流程



```java
//为其他的每个节点单独开启一个线程SendWorker来发送数据，key是serverId  SendWorker负责发送serverId相同的queueSendMap的队列中的数据
final ConcurrentHashMap<Long, SendWorker> senderWorkerMap;  
// 当前节点需要发送给其他的数据（选票信息） key为其他节点serverId   value是CircularBlockingQueue（环形队列）
final ConcurrentHashMap<Long, BlockingQueue<ByteBuffer>> queueSendMap;
// put(2, mess)
// put(3, mess)
final ConcurrentHashMap<Long, ByteBuffer> lastMessageSent;

/*
 * Reception queue
 * 本节点接收到的其他节点发送过来的消息（选票）  value是CircularBlockingQueue（环形队列）
 */
public final BlockingQueue<Message> recvQueue;


领导者选举阶段 只有sid大的可以连接sid小的（领导者选举使用单独的端口） 如果是sid小的连接sid大的 sid小的接收到sid大的响应后 会关闭由自己（sid小的）主动建立的socket连接  同时sid大的会主动和sid小的建立socket连接
  只有和某台server选举端口的socket连接建立好了，才会启动该server对应的SendWorker RecvWorker(RecvWorker被包装在SendWorker里了)
    
    
WorkerSender线程从sendqueue中取选票 扔到queueSendMap
WorkerReceiver线程从recvQueue中取选票扔到sendqueue
  
SendWorker从queueSendMap中取选票发给其他节点
RecvWorker获取其他节点发送的选票信息发送到recvqueue 
    



```



![zk领导者选举四个线程](images/zk领导者选举四个线程.png)







集群选举出现的情况（zk1,zk2,zk3三台）  源码

1、zk1,zk2,zk3进行选举 

2、zk1,zk2选举出leader和follower之后 zk3启动





领导者选举出现的场景（源码）：

1、集群启动

2、leader挂掉

3、超过一半的follower挂掉



# 10、两阶段提交

```
Leader
toBeApplied:记录待生效的请求，某个Proposal可以commit了（符合过半机制，但还未向follower发送commit请求），但是暂时还未提交（Proposal添加到队列后，马上就会commit了） 当FinalRequestProcessor处理完该请求后 会移除
outstandingProposals:记录提议的请求 符合过半机制后，添加到toBeApplied生效之前会移除     

CommitProcessor       
queuedRequests:表示接收到的请求（暂时还没有开始两阶段提交，包括读写请求）
queuedWriteRequests:表示接收到的写请求（暂时还没有进行两阶段提交）
committedRequests:表示可以提交的请求 request已经被提交（已经向follower节点发送了commit命令）
requestsToProcess：需要处理的请求 可能是可以直接交给nextProcessor进行处理的，也可能是需要阻塞的 ==> queuedRequests.size
pendingRequests：某个session中需要被执行的请求，按顺序存放
commitIsWaiting：是否存在可以生效的请求（两阶段提交完成） ==> committedRequests.size == 0 



FinalRequestProcessor
committedLog：记录完成了两阶段提交的的写请求信息

```

![zk server请求处理流程](images/zk server请求处理流程.png)

Leader

org.apache.zookeeper.server.quorum.Leader#startZkServer  初始化请求执行器链

LeaderRequestProcessor ->  PrepRequestProcessor ->ProposalRequestProcessor (SyncRequestProcessor -> AckRequestProcessor) -> 

LeaderRequestProcessor  

1. 处理LocalSession 
2. 把请求交给下一个请求处理器执行



PrepRequestProcessor

1、设置Request的hdr和txn

2、changerecord

3、验证acl

4、把请求交给下一个请求处理器执行



ProposalRequestProcessor 

1. 把请求交给nextProcessor（CommitProcessor）处理，异步 添加到队列中， CommitProcessor的run()方法中轮训队列
2. 如果是写请求 就当前request封装为一个提议，发送给follower（异步） 添加到队列中 由LearnerHandler线程轮训去发送
3. 再把请求交给SyncRequestProcessor ，对请求进行持久化，对datatree打快照
4. SyncRequestProcessor 把请求交给AckRequestProcessor，leader向自己发送一个该请求的ack




CommitRrocessor







**LearnerHandler** 是一个线程对象 监听数据同步端口 负责数据同步（领导者选举结束后或有新的节点加入）、ping和ack、其他节点（follower和observer）转发给leader的请求 有几个follower、observer，节点就有几个learnHandler  



leader处理写请求    RequestProcessor

 1、客户端直接发生给leader的

2、客户端发生给follower  由follower进行转发的   LearnHandler中处理



follower参与两阶段提交：Follower#followLeader   处理propsoal请求和commit请求    RequestProcessor





## 两阶段提交与三阶段提交



两阶段提交







# 11、数据同步机制


领导者选举完成后  --> 数据同步 



1、同步epoch

2、等待epoch ack

3、数据同步





zk存储数据的地方：持久化的写请求日志文件、datatree、committedLog、快照  





# 12、Paxos算法、Raft算法

zab协议：领导者选举 过半机制 同步机制

paxos -> zab -> raft 

12.1Raft算法三种角色（跟随者、候选者、领导者）

12.2Raft领导者选举

集群启动 leader挂掉  pk规则



12.3Raft日志复制

12.4网络分区

12.5Paxos算法







raft应用：etcd redis集群   写请求只能发给leader（写请求发给follower，follower不会转发给leader）









