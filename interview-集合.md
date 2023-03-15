ArrayList和LinkedList的区别

1、底层数据结构：ArrayList底层基于数组实现，LinkedList底层基于双向链表

2、大数据量下的遍历：ArrayList是一片连续的内存空间，随机访问效率高，LinkedList需要移动指针从前往后找，随机访问效率低

3、索引查询和修改效率：ArrayList基于索引查询效率高  LinkedList根据索引查询效率低，**插入和删除是否受元素位置的影响**

ArrayList遍历：for循环比迭代器快

LinkedList遍历：迭代器比for循环快

4、内存空间占用：LinkedList 比 ArrayList 更占内存，因为 LinkedList 的节点除了存储数据，还存储了两个引用，一个指向前一个元素，一个指向后一个元素。





ArrayList扩容条件：元素个数 > 数组长度

ArrayList没有缩容







Collection==>List、Set

Map==>HashMap、LinkedHashMap、TreeMap、HashTable(线程安全)、Properties（线程安全）、ConcurrentHashMap、

Set==>HashSet、TreeSet、LinkedHashSet

List==>Arraylist、LinkedList、Vector（线程安全）、Stack

![集合类图](\images\集合类图.jpeg)





List：有序（元素存入集合的顺序和取出的顺序一致）、可重复，可以插入多个null

Set：无序（元素存入和取出顺序有可能不一致），不可重复，只能插入一个null

Map：k,v键值对存储，key唯一，value可重复



## HashMap在jdk1.7和jdk1.8的区别：

1、hash()：根据key计算出hash值

根据key计算hash值的过程，jdk1.8没有jdk1.7那么复杂

**2、jdk1.7 HashMap底层是数组+链表， jdk1.8 HashMap底层是数组+链表+红黑树（链表长度超过8并且数组长度大于64（小于64就扩容）就转化为红黑树）**

3、jdk1.7数组类型是Entry，jdk1.8数组类型是Node，Entry和Node的属性还是一样的

**4、jdk1.7添加元素到链表是头插法、jdk1.8添加元素到链表是尾插法**

5、扩容条件

jdk1.7：if ((size >= threshold) && (null != table[bucketIndex])) 判断hashmap容量是否超过阈值并且要插入的桶中有没有元素

jdk1.8：if (++size > threshold) 只判断hashmap容量有没有超过阈值

6、扩容操作（链表转移：节点从旧数组转移到新数组）

扩容：假设数组长度为length（length是2的幂次方），某元素在数组的下标为index，数组扩容后的长度为2*length，则这个元素在新数组的下标为index或length + index

jdk1.7：链表上的元素是一个一个进行转移 index = e.hash & newCap

jdk1.8：计算链表中结点在新数组中的下标，e.hash * newTab.length - 1， 将链表中的下标(结点在新数组中的下标)为index的结点组成一个链表，将下标为length + index的结点组成一个链表，这样，在数组中下标为n或m+n的链表中的结点就可以一次性完成转移，只需转移每个链表的头结点
7、都允许null键、null值



## Map

HashMap：jdk1.8 底层基于数组+链表+红黑树实现，jdk1.7基于数组+链表实现

LinkedHashMap：继承自HashMap，在HashMap的基础上，增加了一条双向链表，使得可以保持键值对的插入顺序

HashTable：底层基于数组+链表，线程安全(synchronized)，不允许null键null值

TreeMap：底层基于红黑树实现

ConcurrentHashMap：线程安全，锁分段，不支持null键null值





## List

ArrayList

底层是数组，查询快，增删改慢，初始大小是10，扩容会在原有基础上增加50%

LinkedList

底层是双向链表，查询慢，增删改快

1、LinkedList可以作为**FIFO**(先进先出)的队列，作为FIFO的队列时，下面的方法等价：

```java
//队列方法       等效方法
//add(e)        addLast(e)
//offer(e)      offerLast(e)
//remove()      removeFirst()
//poll()        pollFirst()
//element()     getFirst()
//peek()        peekFirst()
```



2、LinkedList可以作为**LIFO**(后进先出)的栈，作为LIFO的栈时，下面的方法等价

```java
//栈方法        等效方法
//push(e)      addFirst(e)
//pop()        removeFirst()
//peek()       peekFirst()
```

Vector

底层是数组，线程安全的（synchronized），扩容会在原有基础上增加1倍



## Set

HashSet

无序、唯一、底层基于HashMap实现，HashSet的值存放在HashMap的key上，只允许一个nul值

LinkedHashSet

有序，唯一，底层基于LinkedHashMap实现

TreeSet

无序（元素存入和取出顺序有可能不一致），唯一，内部有序（内部确保集合元素处于排序状态），支持自然排序(Comparable)和定制排序(Comparator)，元素必须实现两种排序中的一种（或创建TreeSet传入一个Comparator），底层基于TreeMap实现







## Fail-fast机制

快速失败 

##list与array互转

- 数组转 List：使用 Arrays. asList(array) 进行转换。
- List 转数组：使用 List 自带的 toArray() 方法。

