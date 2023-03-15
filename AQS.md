

![aqs加解锁流程](images/aqs加解锁流程.png)

# Node

```java
class Node{
  volatile int waitStatus;
  volatile Node prev;
  volatile Node next;
  volatile Thread thread;
  Node nextWaiter;
}
```

## waitStatus

1、CANCELLED，值为1 。表示当前结点已取消调度

场景：当该线程等待超时或者被中断，需要从同步队列中取消等待，则该线程被置1，即被取消（这里该线程在取消之前是等待状态）。节点进入了取消状态则不再变化；

2、SIGNAL，值为-1。表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为SIGNAL。

场景：后继的节点处于等待状态，当前节点的线程如果释放了同步状态或者被取消（当前节点状态置为-1），将会通知后继节点，使后继节点的线程得以运行；

3、CONDITION，值为-2。场景：节点处于等待队列中，节点线程等待在Condition上，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁

4、PROPAGATE，值为-3。共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点。场景：表示下一次的共享状态会被无条件的传播下去；

5、INITIAL，值为0，新结点入队时的初始状态

**负值表示结点处于有效等待状态，而正值表示结点已被取消。所以源码中很多地方用>0、<0来判断结点的状态是否正常\****

## nextWaiter

Node既可以作为同步队列节点使用，也可以作为Condition的等待队列节点使用(将会在后面讲Condition时讲到)。在作为同步队列节点时，nextWaiter可能有两个值：EXCLUSIVE、SHARED标识当前节点是独占模式还是共享模式；在作为等待队列节点使用时，nextWaiter保存后继节点。 

Node prev：前驱节点，当节点加入同步队列的时候被设置（尾部添加）

Node next：后继节点

同步队列：具有头尾节点的双向链表 head节点是new出来一个新的Node节点，其中head结点不存储Thread，仅保存next结点的引用，而tail是直接指向队列中最后一个节点。



# AQS

```java
class AbstractQueuedSynchronizer {
  /**
  head和tail代表同步队列的头尾节点 
  同步队列：具有头尾节点的双向链表 head节点是new出来一个新的Node节点，其中head结点不存储Thread，仅保存next结点的引用，而tail是直接指向队列中最后一个节点。
  */
  private transient volatile Node tail;	
  private transient volatile Node head;
  private volatile int state;
  
  
}


```

## state

该变量对不同的子类实现具有不同的意义，对ReentrantLock来说，它表示加锁的状态：

·无锁时state=0，有锁时state>0；

·第一次加锁时，将state设置为1；

·由于ReentrantLock是可重入锁，所以持有锁的线程可以多次加锁，经过判断加锁线程就是当前持有锁的线程时（即exclusiveOwnerThread==Thread.currentThread()），即可加锁，每次加锁都会将state的值+1，state等于几，就代表当前持有锁的线程加了几次锁;

·解锁时每解一次锁就会将state减1，state减到0后，锁就被释放掉，这时其它线程可以加锁；

·当持有锁的线程释放锁以后，如果是等待队列获取到了加锁权限，则会在等待队列头部取出第一个线程去获取锁，获取锁的线程会被移出队列





# 独占锁获取

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

tryAcquire()的实现以ReentrantLock为例：尝试获取锁

```java
final boolean nonfairTryAcquire(int acquires) {
//获取当前线程，支持重入锁
    final Thread current = Thread.currentThread();
//获取锁的状态
    int c = getState();
    if (c == 0) {//锁未被获取
	//使用CAS尝试更新锁的状态，返回true代表更新成功，即获取到锁
        if (compareAndSetState(0, acquires)) {
	    //修改获取到锁的线程为当前线程
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {//锁已被获取，判断获取锁的线程是否是当前线程，支持可重入
	//重入次数加1，释放锁时也必须释放同等次数的锁
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
//更新锁的状态
        setState(nextc);
        return true;
    }
    return false;
}
```

tryAcquire()返回false执行addAwaiter()，即获取锁失败后需要把当前线程加入到等待队列中

addWaiter()：加入到同步队列，此时线程并没有被阻塞 返回的是新创建的节点

```java
//此方法的返回值是使用当前线程新创建的节点
private Node addWaiter(Node mode) {
//基于当前线程、节点类型（Node.EXCLUSIVE）创建新的节点
//由于这里是独占模式，因此节点类型就是Node.EXCLUSIVE
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
//将新建节点加入到队尾
//这里为了提高性能，首先执行一次快速入队操作，即直接将新节点插入队尾
    if (pred != null) {//同步队列尾节点不为null，采用尾插入
	//设置尾节点为新建节点的前驱
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {//cas设置新建节点为尾节点，即使多线程并发操作也只有一个线程操作成功并返回true，其余的都要执行后面的入队操作enq()
//设置成功，设置新建节点（此时已经是尾节点）为尾节点（之前的尾节点）的后继
            pred.next = node;
            return node;
        }
    }

// 2.1. 当前同步队列尾节点为null，说明当前线程是第一个加入同步队列进行等待的线程
//执行完整的入队操作
    enq(node);
    return node;
}
```

enq()：

1. 处理当前同步队列尾节点为null时进行入队操作；2. 同步队列尾节点不为空，如果CAS尾插入节点失败，负责自旋进行尝试。

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
	//如果同步队列没有初始化，则进行初始化，即创建一个空的头结点
        if (t == null) { 
	    //cas设置头结点，多线程并发操作只有一个线程能设置成功，其余的都要重复执行循环体（自旋）
            if (compareAndSetHead(new Node()))
		
                tail = head;
        } else {
	    //新创建的节点指向队尾节点，并发情况下这里可能会有多个新创建的新节点指向队尾节点
            node.prev = t;
	     //cas设置尾节点，无论前一步有多少新节点指向队尾节点，这一步只有一个能入队成功，其他的都必须重新执行循环体（自旋）
            if (compareAndSetTail(t, node)) {
                t.next = node;
		//该循环体唯一退出的出口，就是入队成功，否则就要无限重试
                return t;
            }
        }
    }
}
```



acquireQueued()：node入参就是刚入队成功的包含线程的节点   返回值是线程的中断状态

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;//锁资源获取失败标记位
    try {
		//线程被中断标记位
        boolean interrupted = false;
      //这个循环体执行的时机有两个：
      //1、新节点入队
      //2、队列中等待节点被唤醒
        for (;;) {
	    	//获得当前节点的前驱节点
            final Node p = node.predecessor();
	   		//如果当前节点的前驱是头结点，尝试获取锁
            if (p == head && tryAcquire(arg)) {//获取锁成功
				//当前节点获得锁资源后就设置为头结点，等待队列的头节点就表示当前占有锁资源的节点
                setHead(node);
setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了
				//释放当前节点的前驱
                p.next = null; // help GC
				//锁资源成功获取
                failed = false;
                //返回终端标记，表示当前节点是被正常唤醒还是被中断唤醒
        		//此方法唯一的出口
                return interrupted;
            }
	     	//当前节点的前驱不是头结点或当前节点的前驱是头结点但尝试获取锁失败
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
		//分析锁资源获取失败
        if (failed)//如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待
            cancelAcquire(node);
    }
}
```



shouldParkAfterFailedAcquire()：pred入参是当前线程节点的前驱，node入参是当前线程节点，返回true时当前线程将被阻塞

//根据节点状态判断获取锁失败后的节点里的线程是否需要阻塞

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    //获取当前节点前驱的状态
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
   	//SIGNAL表示当前节点在等待前驱唤醒
        return true;
    if (ws > 0) { //CANCELLED
//前驱节点状态是CANCELLED时，从当前节点的前驱节点开始，一直向前找最近的一个没有被取消的节点，同时通过循环将当前节点之前所有取消状态的节点移出队列；
//由于头结点是通过new Node()创建的，它的waitStatus为0，因此不会出现空指针的情况，也就是说最多找到头结点上面的循环就退出了
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
	//根据waitStatus的取值限定，这里waitStatus的值只能是0或者PROPAGATE，设置当前节点前驱waitStatus为SIGNAL然后重新进入shouldParkAfterFailedAcquire()方法进行判断
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

parkAndCheckInterrupt()：将当前线程阻塞

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
//被唤醒之后，返回中断标记，即如果是正常唤醒返回false，由于中断唤醒返回true
    return Thread.interrupted();
}
```

cancellAcquire()：如果等待过程中没有成功获取资源（如timeout，或者可中断的情况下被中断了），那么取消结点在队列中的等待

```java
private void cancelAcquire(Node node) {//node入参是当前获取锁资源失败的节点
    // 如果节点不存在则直接忽略
    if (node == null)
        return;
    //设置该节点不再关联任何线程
    node.thread = null;

    //通过前驱节点跳过取消状态的节点
    //如果当前节点的前驱节点是取消状态，则向前遍历节点，直到遇到第一个不是取消状态的节点，并把该节点设为当前节点的前驱
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

		//前驱节点的后继节点 当前节点的前驱节点是predNext，但predNext的后继却不一定是当前节点
    Node predNext = pred.next;

		//将当前节点设置为取消，这样别的节点在处理时就会跳过该节点
    node.waitStatus = Node.CANCELLED;

		//如果当前节点为尾节点，则将尾节点设置为当前节点的前驱，并将前驱的后继设为null，相当于将当前node节点从队列中移除
		//注：这里不用担心cas失败，因为即使=并发导致失败，该节点也已经被成功删除（其状态为CANCELLED）
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        int ws;
        //1、如果当前节点的前驱不为head
        //2、前驱状态不为SIGNAL则将前驱节点的状态设为SIGNAL
        //3、以上两步都满足时，校验前驱的thread不为null
        //3点均满足将前驱的后继设置为当前节点的后继，将当前节点移除
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
	    //如果node的前驱节点为head或者前驱节点的状态是PROPAGATE，或者设置状态失败，则直接唤醒当前节点的后续节点
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

首先是要获取当前节点的前继节点，如果前继节点的状态是取消状态（即pred.waitStatus > 0），则向前遍历队列，直到遇到第一个waitStatus <= 0的节点，并把当前节点的前继节点设置为该节点，然后设置当前节点的状态为取消状态。

 接下来的工作可以分为3种情况： 

1、当前节点是tail；

2、当前节点不是head的后继节点（即队列的第一个节点，不包括head），也不是tail；

3、当前节点是head的后继节点。

我们依次来分析一下：



1、当前节点是tail

这种情况很简单，因为tail是队列的最后一个节点，如果该节点需要取消，则直接把该节点的前继节点的next指向null，也就是把当前节点移除队列。出队的过程如下：

![aqs1](images/aqs1.png)

注意：经验证，这里并没有设置node的prev为null。

 

2、当前节点不是head的后继节点，也不是tail

![aqs2](images/aqs2.png)

这里将node的前继节点的next指向了node的后继节点，真正执行的代码就是如下一行

***\*compareAndSetNext(pred, predNext, next);\****

3、当前节点是head的后继节点

![aqs3](images/aqs3.png)



这里直接unpark后继节点的线程，然后将next指向了自己。

 

这里可能会有疑问，既然要删除节点，为什么都没有对prev进行操作，而仅仅是修改了next？【以下摘自参考博客中的内容】

要明确的一点是，这里修改指针的操作都是CAS操作，在AQS中所有以compareAndSet开头的方法都是尝试更新，并不保证成功，图中所示的都是执行成功的情况。

 

那么在执行cancelAcquire方法时，当前节点的前继节点有可能已经执行完并移除队列了（参见setHead方法），所以在这里只能用CAS来尝试更新，而就算是尝试更新，也只能更新next，不能更新prev，因为prev是不确定的，否则有可能会导致整个队列的不完整，例如把prev指向一个已经移除队列的node。 

什么时候修改prev呢？其实prev是由其他线程来修改的。回去看下shouldParkAfterFailedAcquire方法，该方法有这样一段代码： 

do {

  node.prev = pred = pred.prev;

} while (pred.waitStatus > 0);

pred.next = node;

该段代码的作用就是通过prev遍历到第一个不是取消状态的node，并修改prev。

这里为什么可以更新prev？因为shouldParkAfterFailedAcquire方法是在获取锁失败的情况下才能执行，因此进入该方法时，说明已经有线程获得锁了，并且在执行该方法时，当前节点之前的节点不会变化（因为只有当下一个节点获得锁的时候才会设置head），所以这里可以更新prev，而且不必用CAS来更新。



# 独占锁释放

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```



```java
protected final boolean tryRelease(int releases) {
   //锁重入次数-1
    int c = getState() - releases;
   //当前线程不是获取锁的线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {//锁释放完毕，此时锁空闲
        free = true;
    //获取锁的线程置为null，表示锁没有被获取
        setExclusiveOwnerThread(null);
    }
    setState(c);//设置锁状态
    return free;
}
```



```java
private void unparkSuccessor(Node node) {
		//node为头结点
    int ws = node.waitStatus;
    if (ws < 0)
		//把标记设置为0，表示唤醒操作已经开始，提高并发环境下的性能
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;//下一个需要唤醒的节点
    if (s == null || s.waitStatus > 0) {//如果当前节点的后继节点为null或已取消
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)//从后向前找，一直找到等待队列中最前面那个不是CANCELLED状态的节点（一直找到离当前节点最近的一个等待唤醒的节点）
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

主要功能就是要唤醒下一个线程，这里s == null || s.waitStatus > 0判断后继节点是否为空或者是否是取消状态，然后从队列尾部向前遍历找到最前面的一个waitStatus小于0的节点，至于为什么从尾部开始向前遍历，回想一下cancelAcquire方法的处理过程，cancelAcquire只是设置了next的变化，没有设置prev的变化，在最后有这样一行代码：node.next = node，如果这时执行了unparkSuccessor方法，并且向后遍历的话，就成了死循环了，所以这时只有prev是稳定的。

 

 

一句话概括：用unpark()唤醒等待队列中最前边的那个不是CANCELLED状态的线程，这里我们也用s来表示吧。此时，再和acquireQueued()联系起来（线程在acquireQueued的parkAndCheckInterrupt()处被阻塞），s被唤醒后，进入if (p == head && tryAcquire(arg))的判断（即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立啦），然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了！





# 共享锁获取

```java
public final void acquireShared(int arg) {
	//tryAcquireShared尝试获取共享锁，返回值小于0表示获取失败
    if (tryAcquireShared(arg) < 0)
	//执行获取锁失败后的代码
        doAcquireShared(arg);
}
```

这里tryAcquireShared()方法是留给用户去实现具体的获取锁逻辑的。关于该方法的实现有两点需要特别说明：

1、该方法必须自己检查当前上下文是否支持获取共享锁，如果支持再进行获取。

2、该方法返回值是个重点

①由上面的源码片段可以看出返回值小于0表示获取锁失败，需要进入等待队列。

②如果返回值等于0表示当前线程获取共享锁成功，但它后续的线程是无法继续获取的，也就是不需要把它后面等待的节点唤醒

③如果返回值大于0，表示当前线程获取共享锁成功且它后续等待的节点也有可能继续获取共享锁成功，也就是说此时需要把后续节点唤醒让它们去尝试获取共享锁。

 

笔者理解：共享锁其实是有多个锁，锁的个数由开发人员自己决定，比如开发人员指定了5个锁，有10个线程去获取锁，线程每拿一个锁，锁的数量减1前五个线程能通过tryAcquireShared()拿到锁（此时此方法返回值大于0），第6到10个线程会进入等待队列，在doAcquireShared()方法中不断自旋尝试获取锁，此时



doAcquireShared()：获取锁失败的线程执行此方法

```java
private void doAcquireShared(int arg) {
	//基于当前线程、节点类型（Node.SHARED）创建新的节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
	    //前驱节点
            final Node p = node.predecessor();
	     
            if (p == head) {
		//尝试获取锁
                int r = tryAcquireShared(arg);
		//r = 0表示无需唤醒，r > 0表示需要唤醒后继节点
                if (r >= 0) {//进入，表示获取到锁
			//获取到锁以后的唤醒操作
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                  	    //如果是因为中断醒来则设置中断标记位
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
	    //挂起逻辑见独占锁
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
	//获取失败的逻辑见独占锁
        if (failed)
            cancelAcquire(node);
    }
}
```



```java
private void setHeadAndPropagate(Node node, int propagate) {//两个入参，一个是当前成功获取共享锁的节点，一个是tryAcquireShared方法的返回值（即剩余锁的个数），注意上面说的，它可能大于0也可能等于0
//记录当前头结点（即旧的头结点，因为即将设置新的头结点）
    Node h = head; // Record old head for check below
//将当前获取到锁的结点设置为头结点
//注：这里是获取到锁之后的操作，所以不需要进行并发控制
    setHead(node);
    //这里有两种情况需要进行唤醒操作
//1、propagate  > 0，表明了调用方指明了后继节点需要被唤醒 
//注意：propagate > 0表示还有空闲的锁 propagate = 0不会进行后续操作
//2、头节点后面的节点需要被唤醒（h.waitStatus < 0），不论是旧的头结点还是新的头结点
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
	//当前节点的后继
        Node s = node.next;
	//如果当前节点无后继或其后继是共享类型，则进行唤醒
	//可以理解为除非明确指定不需要唤醒（后继是独占类型），否则都要唤醒
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

```java
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```



doReleaseShared()：此方法可能会被setHeadAndPropagate或releaseShared两个方法调用

```java
private void doReleaseShared() {
    for (;;) {
	//唤醒操作由头结点开始，注意这里的头结点已经上面新设置的头结点
	//其实就是唤醒上面新获取到共享锁节点的后继节点 
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
	    //表示后继节点需要被唤醒
            if (ws == Node.SIGNAL) {
		//这里需要控制并发，因为doReleaseShared()方法的入口有setHeadAndPropagate和releaseShared两个，避免两次unpark()
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue; 
                //执行唤醒操作
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
	//如果头结点没有发生变化，表示设置完成，退出循环
	//如果头结点发生变化，比如说其他线程获取到了锁，为了使自己的动作可传递，必须进行重试
        if (h == head)                   // loop if head changed
            break;
    }
}
```

# 共享锁释放

```java
public final boolean releaseShared(int arg) {
	//尝试释放共享锁
    if (tryReleaseShared(arg)) {
	//释放成功，进行唤醒操作
        doReleaseShared();
        return true;
    }
    return false;
}
```



这段方法跟独占式锁释放过程有点不同，在共享式锁的释放过程中，对于能够支持多个线程同时访问的并发组件，必须保证多个线程能够安全的释放同步状态，这里采用的CAS保证，当CAS操作失败continue，在下一次循环中进行重试。



**共享锁总结**

跟独占锁相比，共享锁的主要特征在于当一个在等待队列中的共享节点成功获取到锁以后（它获取到的是共享锁），既然是共享，那它必须要依次唤醒后面所有可以跟它一起共享当前锁资源的节点，毫无疑问，这些节点必须也是在等待共享锁（这是大前提，如果等待的是独占锁，那前面已经有一个共享节点获取锁了，它肯定是获取不到的）。当共享锁被释放的时候，可以用读写锁为例进行思考，当一个读锁被释放，此时不论是读锁还是写锁都是可以竞争资源的。









