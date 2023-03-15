# 性能分析工具

## 操作系统工具

CPU : vmstat 

磁盘：iostat

网络：netstat

内存：free -m





## java内置工具

- jinfo

实时查看和修改正在运行的Jvm配置参数

- jstat

查看虚拟机的类装载、堆内存（伊甸园区、幸存者0区、幸存者1区、老年代）和方法区的使用情况、垃圾收集等运行数据（可以每隔一定时间打印一次）

- jps

列出本机所有java进程信息

- jconsole

图形化界面，展示线程使用，类使用、GC信息

- jmap

手动生成堆转储快照dump文件、查看堆内存的详细信息（堆各区域的使用情况、堆中对象的统计信息、类加载信息） 借助安全点机制



HeapDumpBeforeFullGC 实现在Full GC前dump。

HeapDumpAfterFullGC 实现在Full GC后dump。

-XX:+HeapDumpOnOutOfMemoryError   在发生OOM时dump

- jstack

生成java虚拟机指定进程当前时刻的线程快照，显示各个线程调用的堆栈情况

留意几种状态：

死锁 DeadLock

等待资源 waiting on condition

等待获取监视器 waiting on moniter entry

阻塞 Blocked



- jhat

分析堆转储文件





## 第三方软件

MAT

加载堆转储文件

`Retained Heap（深堆）`：代表对象本身和对象关联的对象占用的内存；

`Shallow Heap（浅堆）`：代表对象本身占用的内存。

- **Histogram视图**

1、从Class类的维度展示**每个Class类的实例存在的个数、 占用的Shallow内存和Retained内存大小**。

2、列出某个class的所有实例，**查看某个实例的引用关系链**



- **Dominator Tree**

列出了每个对象（Object Instance）与其引用关系的树状结构，同时包含了占用内存的大小和百分比。通过Dominator Tree视图可以很容易的找出**占用内存最多的几个对象**（根据Retained Heap或Percentage排序）



- **线程视图**

给出了在生成快照那个时刻，JVM中的Java线程对象列表。 选中某个线程对象展开，可以看到线程的调用栈和每个栈的局部变量



## 内存溢出

堆  java.lang.OutOfMemoryError: Java heap space



方法区 java.lang.OutOfMemoryError: PermGen space

元空间 java.lang.OutOfMemoryError: Metaspace

直接内存内存溢出：java.lang.OutOfMemoryError: Direct buffer memory

**栈内存溢出**：java.lang.StackOverflowError

创建本地线程内存溢出：java.lang.OutOfMemoryError: Unable to create new native thread

数组超限内存溢出：java.lang.OutOfMemoryError：Requested array size exceeds VM limit





## 内存泄漏

程序失去对一段已分配内存空间的控制，造成程序无法释放已经不再使用的内存空间

java内存泄漏场景：

1、单例造成的内存泄漏

如果一个对象已经不再需要使用了，而单例对象还持有该对象的引用，就会使得该对象不能被正常回收，从而导致内存泄漏

2、资源未关闭造成的内存泄漏

3、静态集合类引起内存泄漏

4、内部类持有外部类对象

5、改变hash值 一个对象被存储进hash集合后，修改了这个对象参与hash值计算的属性值







# OOM异常分析

## OOM异常出现的区域

堆  java.lang.OutOfMemoryError: Java heap space



方法区 java.lang.OutOfMemoryError: PermGen space

元空间 java.lang.OutOfMemoryError: Metaspace

直接内存内存溢出：java.lang.OutOfMemoryError: Direct buffer memory

**栈内存溢出**：java.lang.StackOverflowError

创建本地线程内存溢出：java.lang.OutOfMemoryError: Unable to create new native thread

数组超限内存溢出：java.lang.OutOfMemoryError：Requested array size exceeds VM limit

`如果线程请求的栈容量超过栈允许的最大容量的话，Java 虚拟机将抛出一个StackOverflow异常；如果Java虚拟机栈可以动态扩展，并且扩展的动作已经尝试过，但是无法申请到足够的内存去完成扩展，或者在新建立线程的时候没有足够的内存去创建对应的虚拟机栈，那么Java虚拟机将抛出一个OutOfMemory 异常`



jstat -gcutil pid interval 用于查看当前GC的状态,它对Java应用程序的资源和性能进行实时的命令行的监控，包括了对Heap size和垃圾回收状况的监控。



JVM参数`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=D:\dump\  `设置出现OOM自动生成堆转储文件或使用jmap生成堆转储快照文件



MAT分析堆转储快照文件





# CPU 100%排查

1 、 使用top命令查看cpu占用资源较高的java进程PID  

2、top -Hp 进程号 查看java进程下的所有线程的CPU占用情况，找到cpu占用较高的线程TID

3、将线程TID转换为十六进制的表示方式

4、通过jstack -l < PID > 输出当前进程的线程信息

5、分析输出的线程信息

发现使用CPU最高的都是GC 线程。

见频繁GC排查





发现使用CPU最高的是业务线程

- io wait
  - 因为磁盘空间不够导致的io阻塞

- 等待内核态锁，如 synchronized
  - `jstack -l pid | grep BLOCKED`  查看阻塞态线程堆栈
  - dump 线程栈，分析线程持锁情况。
  - arthas提供了`thread -b`，可以找出当前阻塞其他线程的线程。针对 synchronized 情况



# 死锁排查

方法一

1、jps查看死锁java进程pid

2、使用 jstack 查看java进程中线程堆栈信息

jstack 进程pid | grep 线程id十六进制

可疑的线程状态

wait on monitor entry： 被阻塞的,肯定有问题

runnable ： 注意IO线程

in Object.wait()： 注意非线程池等待



方法二

jconsole或jvisualvm

图形化界面 直接点击按钮





# 线程池异常

ThreadPoolExecutor#execute





ThreadPoolExecutor#submit





# 接口耗时

1、服务负载情况

2、服务GC情况 STW

3、数据库 索引、深度分页

4、网络  tcp

5、服务器运行情况 cpu 磁盘io 网络 内存

6、代码逻辑 并行调用 加锁







# 频繁GC排查

- 业务

内存泄漏（代码有问题，对象引用没及时释放，导致对象不能及时回收）。

死循环。

大对象。 尤其是大对象，80%以上的情况就是他。 那么大对象从哪里来的呢？

​	- 数据库（包括MySQL和MongoDB等NoSQL数据库），结果集太大。

​	- 第三方接口传输的大对象。

​	- 消息队列，消息太大。



- jvm参数

**jstat -gcutil pid interval** // interval 打印时间间隔 单位:s     按interval定时打印gc情况

minor gc发生的很频繁，考虑jvm堆栈配置不合理,可能年轻代空间不足

当发生full gc时,old区内存基本不被回收,考虑存在内存泄露



















