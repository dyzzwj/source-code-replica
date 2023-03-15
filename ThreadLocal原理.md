																				ThreadLocal原理

set()：

```java
public void set(T value) {
  	//得到当前线程对象
    Thread t = Thread.currentThread();
  	//得到当前线程对象的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
  	//如果map存在，则将当前线程对象作为key，要存储的对象作为value存到map里面去
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```



ThreadLocalMap:

ThreadLocalMap是ThreadLocal的内部类，用Entry类进行存储，我们的**值都是存储到这个Map上的，key是当前ThreadLocal对象**

```java
static class ThreadLocalMap {
		//实现了弱引用：发现即回收 弱引用关联的对象只能生存到下一次垃圾回收之前
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

  //Entry虽然是弱引用，但它是 ThreadLocal 类型的弱引用（也即上文所述它是对 键 的弱引用），而非具体实例(value属性)的的弱引用，所以无法避免具体实例相关的内存泄漏。
```

如果该map不存在，则初始化一个

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

如果该map存在，则从Thread中获取

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

Thread中维护了threadLocals变量

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

从上面可以看出，ThreadLocalMap是在ThreadLocal使用内部类编写的，但对象的引用在Thread中。

综上：Thread为每个线程维护了ThreadLocalMap这么一个map，而ThreadLocalMap的key是ThreadLocal本身，value则是要存储的对象





**ThreadLocal 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，也即变量在线程间隔离而在方法或类间共享的场景**

应用场景：

- 每个线程需要有自己单独的实例
- 实例需要在多个方法中共享，但不希望被多线程共享



总结：

- ThreadLocal 并不解决线程间共享数据的问题
- ThreadLocal 通过隐式的在不同线程内创建独立实例副本避免了实例线程安全的问题
- 每个线程持有一个 Map 并维护了 ThreadLocal 对象与具体实例的映射，该 Map 由于只被持有它的线程访问，故不存在线程安全以及锁的问题
- ThreadLocalMap 的 Entry 对 ThreadLocal 的引用为弱引用，避免了 ThreadLocal 对象无法被回收的问题
- ThreadLocalMap 的 set 方法通过调用 replaceStaleEntry 方法回收键为 null 的 Entry 对象的值（即为具体实例）以及 Entry 对象本身从而防止内存泄漏
- ThreadLocal 适用于变量在线程间隔离且在方法间共享的场景





# ThreadLocal内存泄漏

每个thread中都存在一个map, map的类型是ThreadLocal.ThreadLocalMap. **Map中的key为一个threadlocal实例**. 这个Map的key属于弱引用,不过弱引用只是针对key. 每个key都弱引用指向threadlocal. 当把threadlocal实例置为null以后,没有任何强引用指向threadlocal实例,所以threadlocal将会被gc回收. 但是,我们的value却不能回收,因为存在一条从current thread连接过来的强引用. 只有当前thread结束以后, current thread就不会存在栈中,强引用断开, Current Thread, Map, value将全部被GC回收。所以得出一个结论就是只要这个线程对象被gc回收，就不会出现内存泄露，但在threadLocal设为null和线程结束这段时间不会被回收的，就发生了我们认为的内存泄露。其实这是一个对概念理解的不一致，也没什么好争论的。最要命的是线程对象不被回收的情况，这就发生了真正意义上的内存泄露。**比如使用线程池的时候，线程结束是不会销毁的，会再次使用的就可能出现内存泄露 。（在web应用中，每次http请求都是一个线程，tomcat容器配置使用线程池时会出现内存泄漏问题）**



ThreadLocalMap中的元素   == super(k);  ==  map中的key是弱引用

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```