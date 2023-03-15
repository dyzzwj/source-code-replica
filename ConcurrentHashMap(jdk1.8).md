																ConcurrentHashMap(jdk1.8)

与jdk1.7比较

1、无参构造

2、无Segment

3、都不允许null键、null值

4、jdk1.7使用RennentrantLock加锁；jdk1.8使用synchronized加锁，锁对象是每个数组元素table[i]（table[i]是链表的头结点或红黑树的根节点）

5、链表插入新结点是尾插法

put()：

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

putVal()：

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // tab[i]这一个位置上赋值的并发问题通过cas解决
        }
        else if ((fh = f.hash) == MOVED) //MOVED=-1 表示ConcurrentHasp在进行扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
              	//加锁，注意锁对象
              	//加完锁后再去判断一下 f时候是tab[i]位置的头结点（对于链表）或根节点（对于红黑树）
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {//链表？
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
              	//binCount：1、链表遍历完毕，会到这里，此时binCount记录当前遍历链表的长度
              	//2、如果在链表找到了跟插入的key相同的Node，也回到这里
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

initTable()：初始化数组

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
  	
    while ((tab = table) == null || tab.length == 0) {
      	//sizeCtl < 0表示有其他线程正在初始化，当前线程让出执行权，重新竞争cpu
        if ((sc = sizeCtl) < 0)
          	//让出执行权
            Thread.yield(); 
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
          	//只有一个线程能成功完成数组初始化
          	//使用cas的方式将sizectl从0修改为-1，只有一个线程能修改成功
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);//相当于0.7 * n，设置扩容阈值
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



treeifyBin()：链表转为红黑树

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                  	//hd是双向链表的头结点
                  	//链表先转为双向链表，再由双向链表转为红黑树
          					//双向链表转红黑树的过程是在TreeBin的构造方法中完成的
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

TreeBin：ConcurrentHashMap的红黑树是放在TreeBin对象中，TreeBin的root属性就行红黑树的根节点。原因是加锁的时候锁对象是table[i]，即红黑树的根节点，如果直接对红黑树的根节点加锁，新的结点插入过程中根节点可能发生改变，此时当前线程还未释放锁，但table[i]的锁对象已经发生变化，其他线程就有机会获取table[i]位置的锁。而使用TreeBin,红黑树根节点可能会变，但TreeBin不会变

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;//红黑树根节点
    volatile TreeNode<K,V> first;//双向链表头结点
```

TreeBin构造方法：

```java
TreeBin(TreeNode<K,V> b) {
    super(TREEBIN, null, null, null);
  	//保存双向链表头结点
    this.first = b;
    TreeNode<K,V> r = null;
    for (TreeNode<K,V> x = b, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        if (r == null) {
            x.parent = null;
            x.red = false;
            r = x;
        }
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = r;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);
                    TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    r = balanceInsertion(r, x);
                    break;
                }
            }
        }
    }
    this.root = r;
    assert checkInvariants(root);
}
```





addCount()：ConcurrentHashMap容量 + 1

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```



```java
//当使用cas进行+1操作没有竞争时，使用baseCount计数
private transient volatile long baseCount;

//如果并发导致使用cas对baseCount + 1 的操作失败了，使用 counterCells（一种用于分配计数的填充单元）。
private transient volatile CounterCell[] counterCells;

```

