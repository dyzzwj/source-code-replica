



轻量级锁是为了在**线程交替执行同步块**时提高性能，而偏向锁则是在只有一个线程执行同步块时进一步提高性能。

关于`synchronized`的底层实现，网上有很多文章了。但是很多文章要么作者根本没看代码，仅仅是根据网上其他文章总结、照搬而成，难免有些错误；要么很多点都是一笔带过，对于为什么这样实现没有一个说法，让像我这样的读者意犹未尽。

本系列文章将对HotSpot的`synchronized`锁实现进行全面分析，内容包括偏向锁、轻量级锁、重量级锁的加锁、解锁、锁升级流程的原理及源码分析，希望给在研究`synchronized`路上的同学一些帮助。主要包括以下几篇文章：



大概花费了两周的实现看代码（花费了这么久时间有些忏愧，主要是对C++、JVM底层机制、JVM调试以及汇编代码不太熟），将`synchronized`涉及到的代码基本都看了一遍，其中还包括在JVM中添加日志验证自己的猜想，总的来说目前对`synchronized`这块有了一个比较全面清晰的认识，但水平有限，有些细节难免有些疏漏，还望请大家指正。

本篇文章将对`synchronized`机制做个大致的介绍，包括用以承载锁状态的对象头、锁的几种形式、各种形式锁的加锁和解锁流程、什么时候会发生锁升级。**需要注意的是本文旨在介绍背景和概念，在讲述一些流程的时候，只提到了主要case，对于实现细节、运行时的不同分支都在后面的文章中详细分析**。

本人看的JVM版本是jdk8u，具体版本号以及代码可以在[这里](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/9ce27f0a4683)看到。

## synchronized简介

Java中提供了两种实现同步的基础语义：`synchronized`方法和`synchronized`块， 我们来看个demo：

```java
public class SyncTest {
    public void syncBlock(){
        synchronized (this){
            System.out.println("hello block");
        }
    }
    public synchronized void syncMethod(){
        System.out.println("hello method");
    }
}
```

当SyncTest.java被编译成class文件的时候，`synchronized`关键字和`synchronized`方法的字节码略有不同，我们可以用`javap -v` 命令查看class文件对应的JVM字节码信息，部分信息如下：

```java
{
  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter				 	  // monitorenter指令进入同步块
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String hello block
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit						  // monitorexit指令退出同步块
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit						  // monitorexit指令退出同步块
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
 

  public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED      //添加了ACC_SYNCHRONIZED标记
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String hello method
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
 
}
```

从上面的中文注释处可以看到，对于`synchronized`关键字而言，`javac`在编译时，会生成对应的`monitorenter`和`monitorexit`指令分别对应`synchronized`同步块的进入和退出，有两个`monitorexit`指令的原因是：为了保证抛异常的情况下也能释放锁，所以`javac`为同步代码块添加了一个隐式的try-finally，在finally中会调用`monitorexit`命令释放锁。而对于`synchronized`方法而言，`javac`为其生成了一个`ACC_SYNCHRONIZED`关键字，在JVM进行方法调用时，发现调用的方法被`ACC_SYNCHRONIZED`修饰，则会先尝试获得锁。

在JVM底层，对于这两种`synchronized`语义的实现大致相同，在后文中会选择一种进行详细分析。

因为本文旨在分析`synchronized`的实现原理，因此对于其使用的一些问题就不赘述了，不了解的朋友可以看看[这篇文章](https://blog.csdn.net/luoweifu/article/details/46613015)。

## 锁的几种形式

传统的锁（也就是下文要说的重量级锁）依赖于系统的同步函数，在linux上使用`mutex`互斥锁，最底层实现依赖于`futex`，关于`futex`可以看我之前的[文章](https://github.com/farmerjohngit/myblog/issues/8)，这些同步函数都涉及到用户态和内核态的切换、进程的上下文切换，成本较高。对于加了`synchronized`关键字但**运行时并没有多线程竞争，或两个线程接近于交替执行的情况**，使用传统锁机制无疑效率是会比较低的。

在JDK 1.6之前,`synchronized`只有传统的锁机制，因此给开发者留下了`synchronized`关键字相比于其他同步机制性能不好的印象。

在JDK 1.6引入了两种新型锁机制：偏向锁和轻量级锁，它们的引入是为了解决在没有多线程竞争或基本没有竞争的场景下因使用传统锁机制带来的性能开销问题。

在看这几种锁机制的实现前，我们先来了解下对象头，它是实现多种锁机制的基础。

### 对象头

因为在Java中任意对象都可以用作锁，因此必定要有一个映射关系，存储该对象以及其对应的锁信息（比如当前哪个线程持有锁，哪些线程在等待）。一种很直观的方法是，用一个全局map，来存储这个映射关系，但这样会有一些问题：需要对map做线程安全保障，不同的`synchronized`之间会相互影响，性能差；另外当同步对象较多时，该map可能会占用比较多的内存。

所以最好的办法是将这个映射关系存储在对象头中，因为对象头本身也有一些hashcode、GC相关的数据，所以如果能将锁信息与这些信息**共存**在对象头中就好了。

在JVM中，对象在内存中除了本身的数据外还会有个对象头，对于普通对象而言，其对象头中有两类信息：`mark word`和类型指针。另外对于数组而言还会有一份记录数组长度的数据。

类型指针是指向该对象所属类对象的指针，`mark word`用于存储对象的HashCode、GC分代年龄、锁状态等信息。在32位系统上`mark word`长度为32bit，64位系统上长度为64bit。为了能在有限的空间里存储下更多的数据，其存储格式是不固定的，在32位系统上各状态的格式如下：

![32位系统下的mark word](images/32位系统下的mark word.png)

synchronized 有 4 种锁状态，从低到高分别为：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态，锁可以升级但不能降级。

**可以看到锁信息也是存在于对象的`mark word`中的。**

**当对象状态为偏向锁（biasable）时，`mark word`存储的是偏向的线程ID；当状态为轻量级锁（lightweight locked）时，`mark word`存储的是指向线程栈中`Lock Record`的指针；当状态为重量级锁（inflated）时，为指向堆中的monitor对象的指针。**

### 重量级锁

重量级锁是我们常说的传统意义上的锁，其利用操作系统底层的同步机制去实现Java中的线程同步。

**Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。**

重量级锁的状态下，对象的`mark word`为指向一个堆中monitor对象的指针。

一个monitor对象包括这么几个关键字段：cxq（下图中的ContentionList），EntryList ，WaitSet，owner。

其中cxq ，EntryList ，WaitSet都是由ObjectWaiter的链表结构，owner指向持有锁的线程。

![minitor对象](images/minitor对象.png)

当一个线程尝试获得锁时，如果该锁已经被占用，则会将该线程封装成一个ObjectWaiter对象插入到cxq的队列尾部，然后暂停当前线程。

当持有锁的线程释放锁前，会将cxq中的所有元素移动到EntryList中去，并唤醒EntryList的队首线程。

如果一个线程在同步块中调用了`Object#wait`方法，会将该线程对应的ObjectWaiter从EntryList移除并加入到WaitSet中，然后释放锁。当wait的线程被notify之后，会将对应的ObjectWaiter从WaitSet移动到EntryList中。



当一个线程尝试获得锁时，如果该锁已经被占用，则会将该线程封装成一个`ObjectWaiter`对象插入到cxq的队列的队首，然后调用`park`函数挂起当前线程。

当线程释放锁时，会从cxq或EntryList中挑选一个线程唤醒，被选中的线程叫做`Heir presumptive`即假定继承人（应该是这样翻译），就是图中的`Ready Thread`，假定继承人被唤醒后会尝试获得锁，但`synchronized`是非公平的，所以假定继承人不一定能获得锁（这也是它叫"假定"继承人的原因）。

如果线程获得锁后调用`Object#wait`方法，则会将线程加入到WaitSet中，当被`Object#notify`唤醒后，会将线程从WaitSet移动到cxq或EntryList中去。需要注意的是，当调用一个锁对象的`wait`或`notify`方法时，**如当前锁的状态是偏向锁或轻量级锁则会先膨胀成重量级锁**。

### 轻量级锁

JVM的开发者发现在很多情况下，在Java程序运行时，同步块中的代码都是不存在竞争的，不同的线程交替的执行同步块中的代码。这种情况下，用重量级锁是没必要的。因此JVM引入了轻量级锁的概念。

线程在执行同步块之前，JVM会先在当前的线程的栈帧中创建一个`Lock Record`，其包括一个用于存储对象头中的 `mark word`（官方称之为`Displaced Mark Word`）以及一个指向对象的指针。下图右边的部分就是一个`Lock Record`。

![轻量级锁Lock record](images/轻量级锁Lock record.png)

#### 加锁过程

1.在线程进入同步块时，如果同步对象锁状态为无锁状态（锁标志位为“01”状态，偏向标志位为“0”），虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，并拷贝锁对象头中的Mark Word复制到锁记录（Lock Record）中，官方称之为 Displaced Mark Word。

2.虚拟机将使用CAS操作尝试将对象Mark Word更新为指向当前线程Lock Record的指针，如果cas失败，进入到步骤3。

3、如果cas失败，则有两种情况：

- 虚拟机首先会检查锁对象的Mark Word是否指向当前线程的栈帧，如果是则当前线程已经持有该锁了，代表这是一次锁重入。那么再添加一条 Lock Record 作为重入的计数，并设置`Lock Record`第一部分（`Displaced Mark Word`）为null，起到了一个重入计数器的作用。然后结束。即锁重入会在线程栈中分配一个`Displaced Mark word`为`null`的`Lock Record`。
  ![轻量级锁锁重入](images/轻量级锁锁重入.png)
  
  
  
- 走到这一步说明发生了竞争，有其他线程已经持有了该对象的轻量级锁，此时需要膨胀为重量级锁。

#### 解锁过程

1.用CAS操作将保存在当前线程栈帧中的Lock record中的数据替换到锁对象的Mark Word中，如果成功，遍历线程栈,找到所有`obj`字段等于当前锁对象的`Lock Record`。

2.如果`Lock Record`的`Displaced Mark Word`为null，代表这是一次重入，将`obj`设置为null后continue。

3.如果`Lock Record`的`Displaced Mark Word`不为null，则利用CAS指令将对象头的`mark word`恢复成为`Displaced Mark Word`。如果成功，则continue，否则膨胀为重量级锁。

### 偏向锁

Java是支持多线程的语言，因此在很多二方包、基础库中为了保证代码在多线程的情况下也能正常运行，也就是我们常说的线程安全，都会加入如`synchronized`这样的同步语义。但是在应用在实际运行时，很可能只有一个线程会调用相关同步方法。比如下面这个demo：

```
import java.util.ArrayList;
import java.util.List;

public class SyncDemo1 {

    public static void main(String[] args) {
        SyncDemo1 syncDemo1 = new SyncDemo1();
        for (int i = 0; i < 100; i++) {
            syncDemo1.addString("test:" + i);
        }
    }

    private List<String> list = new ArrayList<>();

    public synchronized void addString(String s) {
        list.add(s);
    }

}
```

在这个demo中为了保证对list操纵时线程安全，对addString方法加了`synchronized`的修饰，但实际使用时却只有一个线程调用到该方法，对于轻量级锁而言，每次调用addString时，加锁解锁都有一个CAS操作；对于重量级锁而言，加锁也会有一个或多个CAS操作（这里的’一个‘、’多个‘数量词只是针对该demo，并不适用于所有场景）。

在JDK1.6中为了**提高一个对象在一段很长的时间内都只被一个线程用做锁对象场景下的性能**，引入了偏向锁，在第一次获得锁时，会有一个CAS操作，之后该线程再获取锁，只会执行几个简单的命令，而不是开销相对较大的CAS命令。我们来看看偏向锁是如何做的。

#### 对象创建

当JVM启用了偏向锁模式（1.6以上默认开启），当新创建一个对象的时候，如果该对象所属的class没有关闭偏向锁模式（什么时候会关闭一个class的偏向模式下文会说，默认所有class的偏向模式都是是开启的），那新创建对象的对象头的`mark word`将是可偏向状态，此时`mark word中`的thread id（参见上文偏向状态下的`mark word`格式）为0，表示未偏向任何线程，也叫做匿名偏向(anonymously biased)。

#### 加锁过程

1. 检测Mark Word是否为可偏向状态，即是否偏向锁为1，锁标识位为01；
2. 若为可偏向状态，则测试线程ID是否为当前线程ID，如果是，则执行步骤（5），否则执行步骤（3）；
3. 如果测试线程ID不为当前线程ID，则通过CAS操作将Mark Word替换为当前线程ID，cas操作成功，则竞争锁成功，执行同步代码块否则执行线程（4）；
4. 通过CAS竞争锁失败，证明当前存在多线程竞争情况，当到达全局安全点，获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码块；
5. 执行同步代码块；

#### 撤销偏向锁（线程安全点）

1. **查看偏向的线程是否存活，如果已经不存活了，则直接撤销偏向锁**。JVM维护了一个集合存放所有存活的线程，通过遍历该集合判断某个线程是否存活。
2. **偏向的线程是否还在同步块中，如果不在了，则撤销偏向锁**。
3. 虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，并拷贝对象头中的Mark Word复制到锁记录（Lock Record）中，然后将对象头指向Lock Record`，这里不需要用CAS指令，因为是在`safepoint`。 **执行完后，就升级成了轻量级锁**。原偏向





另外，偏向锁默认不是立即就启动的，在程序启动后，通常有几秒的延迟，可以通过命令 `-XX:BiasedLockingStartupDelay=0`来关闭延迟。

#### 批量重偏向与撤销

从上文偏向锁的加锁解锁过程中可以看出，当只有一个线程反复进入同步块时，偏向锁带来的性能开销基本可以忽略，但是当有其他线程尝试获得锁时，就需要等到`safe point`时将偏向锁撤销为无锁状态或升级为轻量级/重量级锁。`safe point`这个词我们在GC中经常会提到，其代表了一个状态，在该状态下所有线程都是暂停的（大概这么个意思），详细可以看这篇[文章](https://blog.csdn.net/ITer_ZC/article/details/41892567)。总之，偏向锁的撤销是有一定成本的，如果说运行时的场景本身存在多线程竞争的，那偏向锁的存在不仅不能提高性能，而且会导致性能下降。因此，JVM中增加了一种批量重偏向/撤销的机制。

存在如下两种情况：（见官方[论文](https://www.oracle.com/technetwork/java/biasedlocking-oopsla2006-wp-149958.pdf)第4小节）:

1.一个线程创建了大量对象并执行了初始的同步操作，之后在另一个线程中将这些对象作为锁进行之后的操作。这种case下，会导致大量的偏向锁撤销操作。

2.存在明显多线程竞争的场景下使用偏向锁是不合适的，例如生产者/消费者队列。

批量重偏向（bulk rebias）机制是为了解决第一种场景。批量撤销（bulk revoke）则是为了解决第二种场景。

其做法是：以class为单位，为每个class维护一个偏向锁撤销计数器，每一次该class的对象发生偏向撤销操作时，该计数器+1，当这个值达到重偏向阈值（默认20）时，JVM就认为该class的偏向锁有问题，因此会进行批量重偏向。每个class对象会有一个对应的`epoch`字段，每个处于偏向锁状态对象的`mark word中`也有该字段，其初始值为创建该对象时，class中的`epoch`的值。每次发生批量重偏向时，就将该值+1，同时遍历JVM中所有线程的栈，找到该class所有正处于加锁状态的偏向锁，将其`epoch`字段改为新值。下次获得锁时，发现当前对象的`epoch`值和class的`epoch`不相等，那就算当前已经偏向了其他线程，也不会执行撤销操作，而是直接通过CAS操作将其对象头`mark word`的Thread Id 改成当前线程Id。

当达到重偏向阈值后，假设该class计数器继续增长，当其达到批量撤销的阈值后（默认40），JVM就认为该class的使用场景存在多线程竞争，会标记该class为不可偏向，之后，对于该class的锁，直接走轻量级锁的逻辑。

## End

Java中的`synchronized`有偏向锁、轻量级锁、重量级锁三种形式，分别对应了锁只被一个线程持有、不同线程交替持有锁、多线程竞争锁三种情况。当条件不满足时，锁会按偏向锁->轻量级锁->重量级锁 的顺序升级。JVM种的锁也是能降级的，只不过条件很苛刻，不在我们讨论范围之内。该篇文章主要是对Java的`synchronized`做个基本介绍，后文会有更详细的分析。