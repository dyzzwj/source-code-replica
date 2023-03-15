#zookeeper

zookeeper保证CP（最终一致性、分区容错性）；与之对应，Eureka保证AP（可用性，分区容错性）；

##zk实例角色：

**Leader**：事务请求的唯一调度和处理者，并保证事务请求的顺序性（事务指能够改变Zookeeper服务状态的操作，一般包括数据节点的创建删除与内容更新、客户端会话创建与失效。每一个事务有全局唯一的ZXID）；集群内部各服务器的调度

**Follower**:处理客户端非事务请求、转发事务请求给Leader、参与事务请求Proposal投票、参与Leader选举投票

**Observer**：只处理非事务请求，转发事务请求给Leader，不参与任何形式的投票，不参与两阶段写提交

##zk结点类型

**EPHEMERAL-临时结点**：临时节点的生命周期与**客户端会话**绑定，一旦客户端会话失效（客户端与zookeeper 连接断开不一定会话失效），那么这个客户端创建的所有临时节点都会被移除。**临时结点下不能创建子节点**

**PERSISTENT-持久结点**：除非手动删除，否则节点一直存在于 Zookeeper 上

**EPHEMERAL_SEQUENTIAL-临时顺序结点**：基本特性同临时结点，只是增加了顺序属性，节点名后边会追加一个由父节点维护的自增整型数字，数字后缀的范围是整型的最大值。

**PERSISTENT_SEQUENTIA-持久顺序结点**：基本特性同持久结点，只是增加了顺序属性，节点名后边会追加一个由父节点维护的自增整型数字，数字后缀的范围是整型的最大值。



##zk选举时机及选举流程

###服务器启动时期的leader选举

若进行Leader选举，则至少需要两台机器，这里选取3台机器组成的服务器集群为例。在集群初始化阶段，当有一台服务器Server1启动时，其单独无法进行和完成Leader选举，当第二台服务器Server2启动时，此时两台机器可以相互通信，每台机器都试图找到Leader，于是进入Leader选举过程。选举过程如下

　　**(1) 每个Server发出一个投票**。由于是初始情况，Server1和Server2都会将自己作为Leader服务器来进行投票，每次投票会包含所推举的服务器的myid和ZXID，使用(myid, ZXID)来表示，此时Server1的投票为(1, 0)，Server2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。

　　**(2) 接受来自各个服务器的投票**。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器。

　　**(3) 处理投票**。针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK，PK规则如下

　　　　**· 优先检查epoch**，epoch比较大的实例优先作为Leader。

​               **· 如果epoch相同，那么比较检查ZXID**。ZXID比较大的服务器优先作为Leader。

　　　　**· 如果ZXID相同，那么就比较myid**。myid较大的服务器作为Leader服务器。

　　对于Server1而言，它的投票是(1, 0)，接收Server2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时Server2的myid最大，于是Server1更新自己的投票为(2, 0)，然后重新投票，对于Server2而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。

　　**(4) 统计投票**。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于Server1、Server2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader。

　　**(5) 改变服务器状态**。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。



###服务器运行时期的leader选举

领导者选举出现的场景：

1、集群启动时

2、leader挂掉

3、超过一半（参与两阶段提交）的节点挂掉了



选举期间集群处于不可用的状态，不能对外提供服务

在Zookeeper运行期间，Leader与非Leader服务器各司其职，即便当有非Leader服务器宕机或新加入，此时也不会影响Leader。但是一旦**Leader服务器挂了或超过一半（参与两阶段提交）的节点挂掉了**，那么整个集群将暂停对外服务，进入新一轮Leader选举，其过程和启动时期的Leader选举过程基本一致。假设正在运行的有Server1、Server2、Server3三台服务器，当前Leader是Server2，若某一时刻Leader挂了，此时便开始Leader选举。选举过程如下

　　(1) **变更状态**。Leader挂后，余下的非Observer服务器都会将自己的服务器状态变更为LOOKING，然后开始进入Leader选举过程。

　　(2) **每个Server会发出一个投票**。在运行期间，每个服务器上的ZXID可能不同，此时假定Server1的ZXID为123，Server3的ZXID为122；在第一轮投票中，Server1和Server3都会投自己，产生投票(1, 123)，(3, 122)，然后各自将投票发送给集群中所有机器。

​	（3）**接受来自各个服务器的投票**

​	（4）处理投票（同上）

​	（5）统计投票（同上）

​	（6）改变服务器状态（同上）

##zk watcher机制

zk允许客户端向服务端注册一个watcher监听，当服务端的一些指定事件触发这个watcher，那么就会向指定客户端发送一个事件通知。

客户端向*Zookeeper*服务器注册*Watcher*事件监听的同时，会将*Watcher*对象存储在 客户端*WatchManager*中。当*Zookeeper*服务器触发*Watcher*事件后，会向客户端发送通知，客户端线程从*WatchManager*中取出对应的*Watcher*对象执行回调逻辑。

监视有两种类型：数据监视点和子节点监视点。创建、删除或者设置znode都会触发这些监视点。exists,getData 可以设置监视点。getChildren 可以设置子节点变化。可能监测的事件类型有: NodeCreated, NodeDataChanged, NodeDeleted, NodeChildrenChanged. 

特点：

1、**一次性触发**
 数据发生改变时，一个watcher event会被发送到client，但是client只会收到一次这样的信息，想再次监听，需重新设置Watcher

2、发送至客户端（异步）

Watch的通知事件是从服务器异步发送给客户端，不同的客户端收到的Watch的时间可能不同。但是ZooKeeper有保证：当一个客户端在收到Watch事件之前是不会看到结点数据的变化的。例如：A=3，此时在上面设置了一次Watch，如果A突然变成4了，那么客户端会先收到Watch事件的通知，然后才会看到A=4。

Zookeeper 客户端和服务端是通过 Socket 进行通信的，由于网络存在故障，所以监听事件很有可能不会成功地到达客户端，监听事件是异步发送至监视者的，Zookeeper 可以保障顺序性(ordering guarantee)：即客户端只有首先收到监听事件后，才会感知到它所监听的 znode 发生了变化.

3、设置watch的数据内容。

Znode改变有很多种方式，例如：节点创建，节点删除，节点改变，子节点改变等等。Zookeeper维护了两个Watch列表，一个节点数据Watch列表，另一个是子节点Watch列表。getData()和exists()设置数据Watch，getChildren()设置子节点Watch。两者选其一，可以让我们根据不同的返回结果选择不同的Watch方式，getData()和exists()返回节点的内容，getChildren()返回子节点列表。因此，setData()触发内容Watch，create()触发当前节点的内容Watch或者是其父节点的子节点Watch。delete()同时触发父节点的子节点Watch和内容Watch，以及子节点的内容Watch。



##zk写数据流程（事务请求）（两阶段提交）

针对每个客户端的事务请求，leader服务器会为其生成对应的事务Proposal，并将其发送给集群中其余所有的机器，然后再分别收集各自的选票，最后进行事务提交

- Leader 接收到消息请求后，将消息赋予一个全局唯一的 64 位自增 id，叫做：zxid，通过 zxid 的大小比较即可实现因果有序这一特性。
- Leader 通过先进先出队列（会给每个follower都创建一个队列，保证发送的顺序性）（通过 TCP 协议来实现，以此实现了全局有序这一特性）将带有 zxid 的消息作为一个提案（proposal）分发给所有 follower。
- 当 follower 接收到 proposal，先将 proposal 写到本地事务日志，写事务成功后再向 leader 回一个 ACK。
- 当 leader 接收到过半的 ACK 后，leader 就向所有 follower 发送 COMMIT 命令（异步），同时会在本地执行该消息。
- 当 follower 收到消息的 COMMIT 命令时，就会处理该消息（修改内存中的Datatree，此时才对客户端可见）



##zk保证数据一致性

1、领导者选举

2、过半机制

3、数据同步

4、两阶段提交



##zk数据同步



zk分布式锁

- 大家都是上来直接创建一个锁节点下的一个接一个的临时顺序节点
- 如果自己不是第一个节点，就对自己上一个节点加监听器
- 只要上一个节点释放锁，自己就排到前面去了，相当于是一个排队机制。





##zk Server工作状态

**Looking**：寻找 Leader 状态。当服务器处于该状态时，它会认为当前集群中没有 Leader，因此需要进入 Leader 选举状态

**Following**：跟随者状态，表明当前服务器角色是follower

**Leading**：领导者状态，表明当前服务器角色是leader

**Observing**：观察者状态，表明当前服务器角色是Observer