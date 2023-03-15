## Callable接口

1、可以有返回值

2、可以抛出Exception异常



与线程池搭配

```
public static void test1(){
		ExecutorService pool = Executors.newCachedThreadPool();
		Future<String> result = pool.submit(new Callable<String>() {
			@Override
			public String call() throws Exception {
				Thread.sleep(2000);
				return "ok";
			}
		});
		try {
			System.out.println(result.get());
		} catch (InterruptedException e) {
			e.printStackTrace();
		} catch (ExecutionException e) {
			e.printStackTrace();
		}
	}
```



非线程池

FutureTask实现了 Runnable接口

```
public static void test2(){
		FutureTask<String> task = new FutureTask<>(new Callable<String>() {
 
			@Override
			public String call() throws Exception {
				return "ok";
			}
		});
		Thread t = new Thread(task);
		t.start();
		try {
			System.out.println(task.get());
		} catch (InterruptedException | ExecutionException e) {
			e.printStackTrace();
		}
	}
```







## 线程状态

初始（NEW）

运行（RUNNABLE）：包括就绪和运行中

阻塞(BLOCKED)：

等待（WAITING）

超时等待（TIMED_WAITING）

终止(TERMINATED)



## 线程安全

- synchronized
- lock
- Cas + volatile
- ThreadLocal
- 阻塞队列（LinkedBlockingQueue）



## CAS的缺点

- ABA问题
- 自旋时间长开销大
- 只能保证一个共享变量的原子操作







## 内存泄漏

程序失去对一段已分配内存空间的控制，造成程序无法释放已经不再使用的内存空间

java内存泄漏场景：

1、单例造成的内存泄漏

如果一个对象已经不再需要使用了，而单例对象还持有该对象的引用，就会使得该对象不能被正常回收，从而导致内存泄漏

2、资源未关闭造成的内存泄漏

3、静态集合类引起内存泄漏

4、内部类持有外部类引用

5、改变hash值，当一个对象被存储进hashset集合时,就不能修改这个对象的那些参与计算hash计算的字段

6、监听器

## 内存溢出









## 线程通信机制

- wait() notify() notofyAll()
- await() signal() signalAll()
- CountDownLatch 倒计数器    构造方法会传入一个整型数N，之后调用CountDownLatch的`countDown`方法会对N减一，直到N减到0的时候，当前调用`await`方法的线程继续执行
- CyclicBarrier 循环栅栏    当多个线程都达到了指定点后，才能继续往下继续执行
- Semaphore 信号量 控制资源能够被并发访问的线程数量，以保证多个线程能够合理的使用特定资源

## 进程通信机制

- 管道
- 信号
- 消息队列
- 共享内存
- 信号量
- 套接字





## JMM内存模型

每个线程创建时 JVM 都会为其创建一个工作内存（有些地方称为栈空间），用于存储线程私有的数据。JMM规定所有变量都存储在主内存，主内存是共享内存区域，所有的线程都可以访问。当线程对这个变量有操作时，必须把这个变量从主内存复制一份到自己的工作空间中进行操作，操作完成后，再把变量写回主内存，不能直接操作主内存的变量。不同的线程无法访问其他线程的工作内存







## 有序性、可见性、原子性



原子性：一个操作或多个操作要么全部执行完成且执行过程不被中断，要么就不执行。



可见性：当多个线程同时访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。（happens-before原则）



有序性：程序执行的顺序按照代码的先后顺序执行（内存屏障）



## 可见性

happens-before规则：向程序员提供跨线程的内存可见性保证

|程序次序规则| 在一个线程内，代码按照书写的控制流顺序执行|
|管程锁定规则| 一个 unlock 操作先行发生于后面对同一个锁的 lock 操作|
|volatile 变量规则| volatile 变量的写操作先行发生于后面对这个变量的读操作|
|线程启动规则| Thread 对象的 start() 方法先行发生于此线程的每一个动作|
|线程终止规则| 线程中所有的操作都先行发生于对此线程的终止检测(通过 Thread.join() 方法结束、 Thread.isAlive() 的返回值检测)|
|线程中断规则| 对线程 interrupt() 方法调用优先发生于被中断线程的代码检测到中断事件的发生 (通过 Thread.interrupted() 方法检测)|
|对象终结规则| 一个对象的初始化完成(构造函数执行结束)先行发生于它的 finalize() 方法的开始|
|传递性| 如果操作 A 先于 操作 B 发生，操作 B 先于 操作 C 发生，那么操作 A 先于 操作 C|





## 重排序

`指令重排序`：在保证程序最终结果一致的前提下，JVM对机器指令进行重新排序。使代码指令更符合cpu执行特性，最大限度的发挥机器性能，提高程序运行效率。

### 重排序分类

- 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。

- 指令级并行的重排序。现代处理器采用了指令级并行技术（ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

- 内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。



### 重排序尊重的规则

- as-if-serial：**不管怎么重排序（编译器和处理器为了提高并行度），（单线程）程序的执行结果不能被改变。** 编译器、runtime和处理器都必须遵守as-if-serial语义。









## 锁的内存语义

### 锁的释放-获取建立的happen-before关系

```java
class MonitorExample{
    int a=0;

    public synchronized void writer(){//1
        a++;						  //2
    }								  //3

    public synchronized void reader(){//4
        int i=a;					  //5
        ...
    }								  //6
}
```



- 假设线程A执行writer()方法，随后线程B执行reader()方法。根据happens-before规则，这个过程包含的happens-before关系可以分为3类
  - 根据程序次序规则，1 happens-before 2，2 happens-before 3；4 happens-before 5，5 happens-before 6
  - 根据监视器规则，3 happens-before 4
  - 根据happens-before的传递性，2 happens-before 5

![锁的释放-获取建立的happen-before关系](images\锁的释放-获取建立的happen-before关系.svg)

- 在线程A释放了锁之后，随后线程B获取同一个锁。因此线程A在释放锁之前所有可见的共享变量，在线程B获取同一个锁之后，将立刻变得对B线程可见



### 锁的释放或获取的内存语义

- 锁释放的内存语义（与volatile写有相同的内存语义）

**当线程释放锁时，JMM会把该线程对应的(所有)本地内存中的共享变量刷新到主内存中**

- 锁获取的内存语义（与volatile读有相同的内存语义）

**当线程获取锁时，JMM会把该线程对应的(所有)本地内存置为无效，从而使得被监视器保护的临界区代码必须从主内存中读取共享变量**





## sleep和wait()的区别：

- sleep是Thread类的方法，wait是Object的方法
- sleep不会释放锁（让出了CPU执行权），wait会释放锁（让出CPU执行权）
- wait必须用在同步代码块中，sleep不需要
- sleep必须指定时间，到时间会自动苏醒，wait可指定可不指定，但必须使用notify或notifyAll唤醒



## 进程与线程的区别：

1、一个进程至少有一个线程，一个进程可以运行多个线程。

2、进程是操作系统资源分配的基本单位，线程是处理器任务调度和执行的基本单位

3、同一进程的线程共享本进程的地址空间和资源，而进程与进程之间的地址空间和资源是相互独立的

4、进程间不会相互影响 ；一个线程挂掉可能会导致整个进程挂掉

5、 进程创建销毁开销大，切换速度慢；线程正相反，开销小，切换速度快

## 死锁

### 死锁四个必要条件：

- 互斥条件：指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的进程用完释放。
- 请求和保持条件：指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放。
- 不可剥夺条件：指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。
- 环路等待条件：指在发生死锁时，必然存在一个进程——资源的环形链，即进程集合{A，B，C，···，Z} 中的A正在等待一个B占用的资源；B正在等待C占用的资源，……，Z正在等待已被A占用的资源。



### 破幻死锁的四个必要条件

- 破坏互斥条件：**使资源同时访问而非互斥使用**，就没有进程会阻塞在资源上，从而不发生死锁（使用共享锁-->读数据）
- 破坏请求和保持条件：采用静态分配的方式，静态分配的方式是指**进程必须在执行之前就申请需要的全部资源，且直至所要的资源全部得到满足后才开始执行**，只要有一个资源得不到分配，也不给这个进程分配其他的资源。
- 破坏不剥夺条件：**即当某进程获得了部分资源，但得不到其它资源，则释放已占有的资源**，但是只适用于内存和处理器资源。
- 破坏循环等待条件：给系统的所有资源编号，规定**进程请求所需资源的顺序必须按照资源的编号**依次进行。









## interrupt、yield、join

Thread.yield：当前线程让出cpu的执行权

thread.join：当前线程等待thread线程执行完 

thread.interrupt：把thread线程的中断标志设置为true

## synchorized

保证原子性、可见性、有序性（但无法禁止指令重排序）

偏向锁

轻量级锁

重量级锁







## synchorized和Lock的对比

- 都支持可重入
- synchorized是java关键字 ->无需手动加锁和释放锁，Lock是juc下的一个类 -> 需要手动加锁和释放锁
- **synchorized是悲观锁，Lock是悲观锁+乐观锁(tryLock)**
- Lock支持可**中断获取锁、超时获取锁、公平锁**
- synchorized在jdk6及之后有优化，偏向锁、轻量级锁、重量级锁
- synchorized通过锁对象wait()、notify()实现线程间的通信机制，Lock通过Condition的await()、singal()实现线程间的通信机制
- synchronized 在发生异常时候会自动释放占有的锁，因此不会出现死锁（JVM保证）；而 lock 发生异常时候，不会主动释放占有的锁，必须在finally块手动 unlock 来释放锁，可能引起死锁的发生（开发人员保证）

## AQS原理

同步队列中的线程状态

```
SIGNAL：表示后继结点在等待当前结点唤醒，当前节点释放锁或被取消时需要唤醒它的后继。后继结点入队时，会将前继结点的状态更新为SIGNAL。

CANCELLED：当前节点已被取消调度。当该线程等待超时或者被中断，需要从同步队列中取消等待，则该线程被置1，即被取消（这里该线程在取消之前是等待状态）。节点进入了取消状态则不再变化；

CONDITION：节点处于等待队列中，节点线程等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁

PROPAGATE：共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。场景：表示下一次的共享状态会被无条件的传播下去；
```







## 线程池ThreadPoolExecutor

```java
//corePoolSize:核心线程数
//maximumPoolSize:最大线程数
//keepAliveTime:该线程池中非核心线程闲置超时时长
//unit:keepAliveTime的单位
//workQueue:该线程池中的任务队列：维护着等待执行的Runnable对象
//threadFactory:新建线程工厂，提供创建新线程的功能
//RejectedExecutionHandler:拒绝策略，当提交任务数超过maxmumPoolSize+workQueue之和时，任务会交给RejectedExecutionHandler来处理
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {}
```

### 线程池执行任务流程：

1.当线程池中的线程数小于corePoolSize时，新提交任务将创建一个新线程执行任务，即使此时线程池中存在空闲线程。
2.当线程池达到corePoolSize时，新提交任务将被放入workQueue中，等待线程池中任务调度执行
3.当workQueue已满，且线程池线程数 < maximumPoolSize时，新提交任务会创建新线程执行任务
4.当线程池中线程数超过maximumPoolSize时，新提交任务由RejectedExecutionHandler处理
5.当线程池中超过corePoolSize线程，空闲时间达到keepAliveTime时，关闭空闲线程
6.当设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize线程空闲时间达到keepAliveTime也将关闭



### workQueue（阻塞队列）

1、`SynchronousQueue` 不存储元素的直接提交队列 

一个内部只能包含一个元素的队列。插入元素到队列的线程被阻塞，直到另一个线程从队列中获取了队列中存储的元素。同样，如果线程尝试获取元素并且当前不存在任何元素，则该线程将被阻塞，直到线程将元素插入队列。

2、`ArrayBlockingQueue` 基于数组结构的有界任务队列

ArrayBlockingQueue 基于数组结构的有界的阻塞队列。此队列按 FIFO（先进先出）原则对元素进行排序。
新元素插入队列的尾部，从头部获得元素。这是一个典型的“有界缓存区”，一旦创建了这样的缓存区，就不能再增加其容量。试图向已满队列中放入元素会导致操作受阻塞；试图从空队列中提取元素将导致类似阻塞

3、`LinkedBlockingQueue **基于链表的FIFO阻塞队列**

LinkedBlockingQueue是单链表的阻塞队列， 尾部插入元素，头部取出元素；其特点是 FIFO（先进先出）。

LinkedBlockingQueue可以手动指定容量也可以不指定容量，不手动指定容量则其容量为Integer.MAX_VALUE

4、`PriorityBlockingQueue` 优先任务队列

PriorityBlockingQueue 是一个按照优先级进行内部元素排序的无限队列。存放在`PriorityBlockingQueue` 中的元素必须实现Comparable 接口，这样才能通过实现`compareTo()`方法进行排序。优先级最高的元素将始终排在队列的头部；

5、`DelayQueue` 延时队列

DelayQueue是一个无界队列，加入其中的元素必须实现Delayed接口。当生产者线程调用put之类的方法加入元素时，会触发Delayed接口中的compareTo方法进行排序，也就是说队列中元素的顺序是按到期时间排序的，而非它们进入队列的顺序。排在队列头部的元素是最早到期的，越往后到期时间赿晚。

6、**LinkedBlockingDeque, （基于链表的FIFO双端阻塞队列）**

LinkedBlockingDeque是一个由链表结构组成的双向阻塞队列，即可以从队列的两端插入和移除元素。双向队列因为多了一个操作队列的入口，在多线程同时入队时，也就减少了一半的竞争。

相比于其他阻塞队列，LinkedBlockingDeque多了addFirst、addLast、peekFirst、peekLast等方法，以first结尾的方法，表示插入、获取获移除双端队列的第一个元素。以last结尾的方法，表示插入、获取获移除双端队列的最后一个元素。

LinkedBlockingDeque是可选容量的，在初始化时可以设置容量防止其过度膨胀，如果不设置，默认容量大小为Integer.MAX_VALUE。







## 非阻塞队列

1、**ArrayDeque, （数组双端队列）**

ArrayDeque （非堵塞队列）是JDK容器中的一个双端队列实现，内部使用数组进行元素存储，不允许存储null值，可以高效的进行元素查找和尾部插入取出，是用作队列、双端队列、栈的绝佳选择，性能比LinkedList还要好。

2、**PriorityQueue, （优先级队列）**

PriorityQueue （非堵塞队列） 一个基于优先级的无界优先级队列。优先级队列的元素按照其自然顺序进行排序，或者根据构造队列时提供的 Comparator 进行排序，具体取决于所使用的构造方法。该队列不允许使用 null 元素也不允许插入不可比较的对象

3、**ConcurrentLinkedQueue, （基于链表的并发队列）**

ConcurrentLinkedQueue （非堵塞队列）: 是一个适用于高并发场景下的队列，通过无锁的方式，实现了高并发状态下的高性能。ConcurrentLinkedQueue的性能要好于BlockingQueue接口，它是一个基于链接节点的无界线程安全队列。该队列的元素遵循先进先出的原则。该队列不允许null元素。





### RejectedExecutionHandler拒绝策略

1、AbortPolicy

直接抛出异常

2、CallerRunsPolicy

用调用者所在的线程来执行任务

3、DiscardOldestPolicy

丢弃阻塞队列中最靠前（最先加入）的任务，并执行当前任务

4、DiscardPolicy

直接丢弃任务



## 阻塞队列原理







## Executors

### 可缓存的线程池



核心线程数为0，最大线程数为Integer.MAX_VALUE，阻塞队列为SynchronousQueue 相当于来一个任务创建一个线程

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```







### 固定线程数量的线程池

```java
 		/**
     * 创建固定大小的线程池 入参为线程池数量
     * 任务队列为无界队列 如果有很多请求积压，阻塞队列越来越长，容易导致OOM（超出内存空间）
     * @param nThreads
     * @return
     */
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```



### 

### 单个线程的线程池

核心线程数为1，最大线程数为1 任务队列为LinkedBlockingQueue 

```java
//任务队列为无界队列 如果有很多请求积压，阻塞队列越来越长，容易导致OOM（超出内存空间）
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```



### 可执行定时任务的线程池

核心线程数为corePoolSize，最大线程数 为Integer.MAX_VALUE  任务队列为DelayedWorkQueue

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```









## 合理估算线程池的大小

最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1）* CPU数目

线程等待时间：非CPU运行时间，比如IO）

线程CPU时间：每个线程CPU运行时间





**线程等待时间所占比例越高（IO密集型），需要越多线程。线程CPU时间所占比例越高（CPU密集型：大量运算，没有阻塞），需要越少线程。**