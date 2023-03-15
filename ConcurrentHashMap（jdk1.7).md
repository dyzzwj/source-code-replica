```java
																	ConcurrentHashMap（jdk1.7）
```

说明：

1、Segment数组不能扩容

2、ConcurrentHashMap的扩容仅仅是针对Segment内部的HashEntry数组，某个Segment的HashEntry数组达到阈值后，只会引起当前Segment的HashEntry数组进行扩容操作，对其他的Segment没影响

3、ConcurrentHashMap里大量的Unsafe类的使用，是为了取主内存中数组（包括Segment数组和Segment中HashEntry数组）最新的值，而不是取线程工作内存里的缓存中的值

4、ConcurrentHashMap使用ReentrantLock来保证并发安全（Segment extends ReentrantLock）,初始化Segment数组使用的是Unsafe的cas操作

核心构造方法：

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
  //initialCapacity：初始容量，所有segment下的
  //concurrencyLevel：segment数量，默认16
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {   1
        ++sshift;	
        ssize <<= 1;   
    }
  //sshift和ssize的关系  ==> 2的sshift次方 = ssize
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
  	//只初始化了下标为0的元素 原型模式
  	//之后其他下标初始化的时候，初始化参数就直接使用下标为0的元素的参数（避免再次计算）
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```







put方法

1、先通过hash(key)获取key的hash值

2、(hash >>> segmentShift) & segmentMask 使用hash的高sshift位得到segment数组下标

3、hash & table.length -1 得到segment中hashEntry数组的下标





```java
public V put(K key, V value) {
    Segment<K,V> s;
  //
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
  	//构造函数中定义：this.segmentShift = 32 - sshift;
  	//hash >>> segmentShift 保留hash值的高sshift位 
  	//构造函数中定义：this.segmentMask = ssize - 1;
  	//segmentMask的低sshift位都是1
    int j = (hash >>> segmentShift) & segmentMask;
  	
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```



使用cas对segment数组下标为k的元素赋值。存在两种情况：

1、当前线程对segment数组下标为k的元素赋值

2、获取到其他线程对segment数组下标为k的元素进行赋值之后的元素

```java
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
  	//(Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))：获取ss数组下标为u的元素
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
      //使用ss0作为原型   ss0是在构造方法中创建的并赋给segment数组的0下标
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```



Segment.put()：

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
  	//尝试去加锁，加锁成功继续往下执行；加锁失败则执行scanAndLockForPut（注意是tryLock()，加锁失败不会阻塞）
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
  
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
      	//得到数组索引下标（桶）
        int index = (tab.length - 1) & hash;
      	//得到链表头结点
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);//头插法
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);//扩容
                else
                    setEntryAt(tab, index, node);//头插法
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```



scanAndLockForPut():自旋去加锁的时候，先把可能要创建的HashEntry创建出来，即当前要添加的k,v对应的HashEntry（为了不空转以至于耗费cpu）

线程在scanAndLockForPut()只会对要添加k,v对应的桶做遍历，不会做更改，遍历桶是为了判断是否需要提前创建当前要添加的k,v对应的HashEntry，可能桶中已经存在要添加的k,v

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
  	//获取到要添加k,v对应的桶
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {//进入此分支，是为了知道是不是一定要去创建HashEntery对象，可能要添加的k,v此时在链表中存在
            if (e == null) {// e = null 有两种情况
              //1、first = null
              //2、遍历到了链表的末尾
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
              	//如果要添加的k,v在链表中存在，则跳出（if (retries < 0) ）分支
                retries = 0;
          			//跳出当前（if (retries < 0) ）分支
            else
              	//每遍历链表上的一个节点，都会去tryLock()尝试获取锁
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {
          	//自旋到达一定次数，阻塞
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
          	//偶数次重试的时候判断在遍历链表节点的过程中，是否有其他获取到锁的线程在当前线程要添加的桶里添加过k,v(头插法，first会改变)
            e = first = f; 
          	//如果有线程更改了当前这个桶的节点，从头遍历
            retries = -1;
          	//切换到（if (retries < 0) ）分支
        }
    }
    return node;
}
```



entryForHash()：根据hash值找到要插入的桶

```java
static final <K,V> HashEntry<K,V> entryForHash(Segment<K,V> seg, int h) {
    HashEntry<K,V>[] tab;
    return (seg == null || (tab = seg.table) == null) ? null :
        (HashEntry<K,V>) UNSAFE.getObjectVolatile
        (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
}
```



rehash()：扩容，仅仅是针对Segement内部的扩容

```java
private void rehash(HashEntry<K,V> node) {
  
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
      	//链表头结点
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {//头结点为空代表桶里没有元素
            HashEntry<K,V> next = e.next;
          	//计算新的数组下标，和jdk1.7HashMao不同，不需要重新计算hash(即rehash）
            int idx = e.hash & sizeMask;
            if (next == null)   //链表上只有一个节点
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
              	//找到会在同一个桶中的连续序列（连续节点）
              	//扩容之前在同一个链表上的节点，扩容之后可能不在同一个节点，下面第一个循环就是要找到扩容后可能在同一个桶中的连续节点
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
								
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```



size()：计算ConcurrentHashMap的大小

```java
public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size;
    boolean overflow; // true if size overflows 32 bits
    long sum;         // sum of modCounts
    long last = 0L;   // previous sum
    int retries = -1; // first iteration isn't retry
    try {
        for (;;) {
          	//先不加锁，统计四次
            if (retries++ == RETRIES_BEFORE_LOCK) { //RETRIES_BEFORE_LOCK = 2
              	//如果不加锁的情况下不能正确完成统计（说明并发较高），则进行加锁统计
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation
            }
            sum = 0L;
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) {
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount;
                    int c = seg.count;
                    if (c < 0 || (size += c) < 0)
                        overflow = true;
                }
            }
          	//每次循环统计所有Segment的modCount之和和size之和，如果本次和上一次的总modCount相等，就代表此次统计的size是正确的（两次统计之间，没有其他线程进行增删）
          	//比较本次和上一次的所有Segment的modCount之和（总modCount）
            if (sum == last)
                break;
          	//不相等则把本次统计的总modCount保存
            last = sum;
        }
    } finally {
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock();
        }
    }
    return overflow ? Integer.MAX_VALUE : size;
}
```











