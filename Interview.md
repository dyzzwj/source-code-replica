

# java基础

//Java源代码---->编译器---->jvm可执行的Java字节码(即虚拟指令)---->jvm---->jvm中解释器----->机器可执行的二进制机器码---->程序运行。 					



## 8种基本数据类型

![基本数据类型](images/基本数据类型.png)

##java基本数据类型转换

![基本数据类型转换](images/基本数据类型转换.png)



**boolean类型与其他基本类型不能进行类型的转换（既不能进行自动类型的提升，也不能强制类型转换）， 否则，将编译出错**。

**char型其本身是unsigned型，同时具有两个字节，其数值范围是0 ~ 2^16-1，因为，这直接导致byte型不能自动类型提升到char，char和short直接也不会发生自动类型提升（因为负数的问题），同时，byte当然可以直接提升到short型。**

在Java中，整数类型（byte/short/int/long）中，对于未声明数据类型的整形，其默认类型为int型。在浮点类型（float/double）中，对于未声明数据类型的浮点型，默认为double型

```java
byte b = 3;//编译通过，注意3为int型直接量，编译期间可以直接进行判定

int i = 4;
byte c = i;//编译不通过，i为int型变量，需要到运行期间才能确定

```

## Math.round()

Math.round(11.5) 等于多少？Math.round(-11.5)等于多少

Math.round(11.5)的返回值是 12，Math.round(-11.5)的返回值是-11。**四舍五入的原理是在参数上加 0.5 然后进行下取整。**

## switch支持的类型：

- 基本数据类型：byte, short, char, int
- 包装数据类型：Byte, Short, Character, Integer
- 枚举类型：Enum
- 字符串类型：String（Jdk 7+ 开始支持）







## **String**

### String、StringBuilder、StringBuffer

StringBuilder：线程不安全

StringBuffer：线程安全

如果要操作少量的数据用 = String

单线程操作字符串缓冲区 下操作大量数据 = StringBuilder

多线程操作字符串缓冲区 下操作大量数据 = StringBuffer

### 字符串常量池

### String.intern()

### 底层数据结构

- jdk8及之前是char[]，char占两个字节
- jdk9及之后是byte[]





## 基本数据类型赋值

### float f=3.4;是否正确

不正确。3.4 是双精度数，将双精度型（double）赋值给浮点型（float）属于下转型（down-casting，也称为窄化）会造成精度损失，因此需要强制类型转换float f =(float)3.4; 或者写成 float f =3.4F;。

### short s1 = 1; s1 = s1 + 1;有错吗?short s1 = 1; s1 += 1;有错吗

对于 short s1 = 1; s1 = s1 + 1;由于 1 是 int 类型，因此 s1+1 运算结果也是 int型，需要强制转换类型才能赋值给 short 型。

而 short s1 = 1; s1 += 1;可以正确编译，因为 s1+= 1;相当于 s1 = (short(s1 + 1);其中有隐含的强制类型转换

## 访问修饰符

private : 在同一类内可见。使用对象：变量、方法。 注意：不能修饰类（外部类）
default (即缺省，什么也不写，不使用任何关键字）: 在同一包内可见，不使用任何修饰符。使用对象：类、接口、变量、方法。
protected : 对同一包内的类和所有子类可见。使用对象：变量、方法。 注意：不能修饰类（外部类）。
public : 对所有类可见。使用对象：类、接口、变量、方法

![访问修饰符](images/访问修饰符.png)

Protected > default   protected不仅允许本包的类（包括本包子类）访问，还允许外包的子类访问 

default只允许本包的类（包括本包的子类）访问，但不允许外包的子类访问



## 基本数据类型和包装类型的区别

1、包装类型可以为null，基本数据类型不能为null -> po类 的属性必须用包装类型。

2、包装类型可以用于泛型，而基本数据类型不行

3、基本数据类型比包装数据类型更高效 -> 基本数据类型在栈中存储的具体的值，而包装类型则存储的堆中的引用。

4、基本数据类型在自动装箱时，如果数字在 -128 至 127 之间时，会直接使用缓存中的对象，而不是重新创建一个对象

5、基本数据类型比较值相等用== 包装类型比较值相等用equals





## 对象实例化的过程：

**父类的类构造器<clinit>(静态变量和静态代码块) -> 子类的类构造器<clinit>(静态变量和静态代码块) -> 父类的成员变量和实例代码块 -> 父类的构造函数 -> 子类的成员变量和实例代码块 -> 子类的构造函数**





## final 有什么用？

用于修饰类、属性和方法；

- 被final修饰的类不可以被继承
- 被final修饰的方法不可以被重写
- 被final修饰的变量不可以被改变，被final修饰不可变的是变量的引用，而不是引用指向的内容，引用指向的内容是可以改变的



## final finally finalize区别

- final可以修饰类、变量、方法，修饰类表示该类不能被继承、修饰方法表示该方法不能被重写、修饰变量表
  示该变量是一个常量不能被重新赋值。
- finally一般作用在try-catch代码块中，在处理异常的时候，通常我们将一定要执行的代码方法finally代码块
  中，表示不管是否出现异常，该代码块都会执行，一般用来存放一些关闭资源的代码。
- finalize是一个方法，属于Object类的一个方法，而Object类是所有类的父类，finalize()方法在对象被回收之前由jvm自动调用



## this与super

- super()和this()均需放在构造方法内第一行。
- 尽管可以用this调用一个构造器，但却不能调用两个
- this和super不能同时出现在一个构造函数里面，因为this必然会调用其它的构造函数

## Static

在类初次被加载的时候，会按照static块的顺序来执行每个static块，并且只会执行一次。

静态成员只能访问静态成员



## 实现Serializable接口为什么要声明serialVersionUID？

serialVersionUID 是 Java 为每个序列化类产生的版本标识，可用来保证在反序列时，发送方发送的和接受方接收的是可兼容的对象。如果接收方接收的类的 serialVersionUID 与发送方发送的 serialVersionUID 不一致，进行反序列时会抛出 InvalidClassException。**建议显式声明 serialVersionUID 的值。**









## 抽象类与接口

1、抽象类可以有普通方法，接口都是抽象方法

2、抽象方法有可以有构造器 -> 可以实例化，接口中没有构造器 -> 不能实例化

3、抽象类的方法可以使用任意访问修饰符，接口的方法修饰符只能是public

4、类是单继承，接口是多继承

5、接口的属性只能是static final的



## object类的方法

- finalize、wait、notify、toString、hashcode、equlas、clone、getClass





## 面向对象五大基本原则

单一职责

开放封闭

里式替换

依赖倒置

接口分离





## 反射

反射创建对象：Class.newInstance()、Constructor.newInstance()

获得Class对象：Class.forName("")、类.class.getClass()、this.getClass()



## 多态

方法重载是编译时多态（前绑定），方法重写是运行时多态（后绑定）

### 重载与重写

重载：

在一个类里面，**方法名相同，参数列表不同**（个数、顺序、类型），对 方法返回值、是否抛出异常、访问权限无限制

重写：

发生在子父类之间

**参数列表与父类被重写方法必须完全相同**

**访问权限不能比父类被重写方法低（里式替换）**

**不能抛出比被重写方法声明的更大的强制性异常（非运行时），可以抛出非强制性异常（运行时异常）**





## 内部类

### 静态内部类

静态内部类可以访问外部类所有的静态变量（private修饰的也可以访问），而不可访问外部类的非静态变量；

静态内部类的创建方式，`new 外部类.静态内部类()`

Outer.StaticInner inner = new Outer.StaticInner();

### 成员内部类

成员内部类可以访问外部类所有的变量和方法，包括静态和非静态，私有和公有。

**成员内部类依赖于外部类的实例**，它的创建方式`外部类实例.new 内部类()`;

Outer outer = new Outer();

Outer.Inner inner = outer.new Inner();

### 局部内部类

局部内部类只在当前方法有效,

定义在实例方法中的局部类可以访问外部类的**所有变量(final修饰)**和方法，定义在静态方法中的局部类只能访问外部类的静态变量和方法。

局部内部类的创建方式，在对应方法内，`new 内部类()`，如下：

### 匿名内部类

匿名内部类就是没有名字的内部类，日常开发中使用的比较多。

- 匿名内部类必须继承一个抽象类或者实现一个接口。
- 匿名内部类不能定义任何静态成员和静态方法。
- 当所在的方法的形参需要被匿名内部类使用时，必须声明为 final。
- 匿名内部类不能是抽象的，它必须要实现继承的类或者实现的接口的所有抽象方法



## equals()与hashcode()

覆盖equals时一定要覆盖hashCode

**如果两个对象equals相等，那么它们的hashCode必然相等，**
**但是hashCode相等，equals不一定相等。**

## 值传递和引用传递

**java中只有值传递**

- 一个方法不能修改一个基本数据类型的参数
- 一个方法可以改变一个对象参数的状态（属性）。
- 一个方法不能让对象参数引用一个新的对象。



## 反射：

### 获取class对象

Class.forName("");

对象.getClass;

类.class

### 反射创建对象

1、Class.newInstance();

2、Constructor.newInstance();



### Class.forName和ClassLoader.loadClass的区别

Class.forName会加载静态代码块，ClassLoader.loadClass不会加载静态代码块



### LocalDateTime





## 包装类型的常量池

Byte,Short,**Integer**,Long,Character，这5种整型的包装类的常量池范围在**-128~127**之间；

Boolean也实现了对象池技术





## bio、nio、io多路复用、aio

- bio：同步阻塞

当调用系统调用read时，用户线程会一直阻塞到内核空间有数据到来为止，否则就一直阻塞。

- nio：同步非阻塞

一个线程不断的轮询内核缓冲区的状态（数据到达没有）

- io多路复用

先调用select这个系统调用去查询是否有数据就绪的socket，然后有数据就绪，才调用read这个系统调用来读

- aio：异步非阻塞

异步非阻塞无需一个线程去轮询所有IO操作的状态改变，在相应的状态改变后（数据准备好了），系统会通知对应的线程来处理





# 集合

Collection==>List、Set

Map==>HashMap、LinkedHashMap、TreeMap、HashTable(线程安全)、Properties（线程安全）、ConcurrentHashMap、

Set==>HashSet、TreeSet、LinkedHashSet

List==>Arraylist、LinkedList、Vector（线程安全）、Stack

![集合类图](images/集合类图.jpeg)





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





# 网络编程





## 七层网络模型（物链网输会示用）



应用层

表示层

会话层

传输层

网络

数据链路层

物理层







## 四层网络模型

应用层：http、ftp、ssh、smtp

传输层：tcp、udp

网络层：ip

数据链路层：



## tcp、udp

1. 传输控制协议-TCP：提供面向连接的，可靠的数据传输服务。
2. 用户数据协议-UDP：提供无连接的，尽最大努力的数据传输服务（不保证数据传输的可靠性）







## 三次握手：

- 客户端给服务端发送 SYN=1 seq=k 进入SYN_SENT状态

- 服务端收到后，发送SYN=1 ACK=1 acknowledge number=k+1 seq = j 进入SYN_RECV状态

- 客户端收到后，发送ACK= 1 acknowledge number= j+1 进入ESTABLISHED状态，服务器检查ACK为1和acknowledge number为序列号+1之后，也进入ESTABLISHED状态；



第三次握手中，如果客户端的ACK未送达服务器，会怎样？

`Server端：由于Server没有收到ACK确认，因此会每隔 3秒 重发之前的SYN+ACK（默认重发五次，之后自动关闭连接进入CLOSED状态），Client收到后会重新传ACK给Server`



三次握手过程中可以携带数据吗？

**第三次握手的时候，是可以携带数据的**。但是，**第一次、第二次握手绝对不可以携带数据**





三次握手的本质是确认通信双方收发数据的能力

首先，我让信使运输一份信件给对方，**对方收到了，那么他就知道了我的发件能力和他的收件能力是可以的**。

于是他给我回信，**我若收到了，我便知我的发件能力和他的收件能力是可以的，并且他的发件能力和我的收件能力是可以**。

此时他还不知道他的发件能力和我的收件能力到底可不可以，于是我最后回馈一次，**他若收到了，他便清楚了他的发件能力和我的收件能力是可以的**。

四次挥手的目的是关闭一个连接



## 四次挥手

第一次挥手：Client给Server发送FIN = 1,发送seq = k给server,进入`FIN_WAIT_1`状态；

第二次挥手：Server收到FIN之后，发送ACK=1，acknowledge number=k+1；进入`CLOSE_WAIT`状态。此时客户端已经没有要发送的数据了，但仍可以接受服务器发来的数据。客户端收到服务端的确认后，进入`FIN_WAIT2`（终止等待 2）状态，等待服务端发出的连接释放报文段

第三次挥手：Server发送FIN=1，seq = j给Client；进入`LAST_ACK`状态；

第四次挥手：Client收到服务器的FIN后，进入TIME_WAIT状态；发送ACK=1，acknowledge number= j + 1给服务器；服务器收到后，确认acknowledge number后，变为`CLOSED`状态，不再向客户端发送数据。客户端等待2*MSL（**一个报文的来回时间**）时间后，也进入`CLOSED`状态。完成四次挥手。







## 从输入 URL 到页面展示到底发生了什么

1、浏览器查找域名的ip地址；

浏览器DNS缓存 -> 操作系统DNS缓存 -> hosts文件 -> 本地DNS服务器 ->根域DNS服务器 

2、浏览器获得域名对应的IP地址以后，浏览器会以一个随机端口（1024<端口<65535）向服务器的WEB程序80端口发起TCP的连接请求，tcp连接的三次握手；

3、TCP/IP链接建立起来后，浏览器向服务器发送HTTP请求；

4、服务器接收到这个请求，根据http请求协议的格式解析请求报文（请求首行、请求头、空行、请求体），并根据路径参数映射到特定的请求处理器进行处理，并将处理结果及相应的视图返回给浏览器；

5、浏览器根据http响应协议的格式解析并渲染视图，若遇到对js文件、css文件及图片等静态资源的引用，则重复上述步骤并向服务器请求这些资源；

6、HTML解析成DOM树，解析css，执行js代码

7、keep-alive **HTTP1.1** 默认开启keep-alive机制（即在请求头）

8、断开连接 四次挥手



## http

无状态。默认端口号80 https端口443

### http请求协议格式

请求首行（请求类型、请求资源、协议版本）、请求头、空行、请求体

```
POST / HTTP/1.1
 Host: www.example.com
 User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.7.6)
 Gecko/20050225 Firefox/1.0.1
 Content-Type: application/x-www-form-urlencoded
 Content-Length: 40
 Connection: Keep-Alive

 sex=man&name=Professional  
```



### http响应协议格式

响应首行（协议版本、状态码、状态消息）、响应头、空行、响应体

```
HTTP/1.1 200 OK

Server:Apache Tomcat/5.0.12
Date:Mon,6Oct2003 13:23:42 GMT
Content-Length:112

<html>
```









## get与post的区别

①点击刷新按钮，post会被重复提交

②get编码类型是application/x-www-form-urlencoded，post编码类型是 application/x-www-form-urlencoded、multipart/form-data、application/json、text/html

③get数据长度有限制，post数据长度无限制

④get安全性较差

⑤get只允许 ASCII字符，post没有限制









## Keep-alive

http默认（在非keep-alive模式时）是无状态的，每个请求/应答客户和服务器都要新建一个连接，完成之后立即断开连接（HTTP 协议为无连接的协议）

当使用keep-alive模式，Keep-Alive 功能使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，**Keep-Alive 机制能保持当前的TCP连接，避免了建立或者重新建立连接**

在 HTTP 1.0 中默认是关闭的，如果浏览器要开启 Keep-Alive，它必须在请求头中添加：`Connection: Keep-Alive`

如果要关闭 Keep-Alive，需要在 HTTP 请求的包头里添加：`Connection:close`

HTTP 长连接不可能一直保持，例如 `Keep-Alive: timeout=5, max=100`，表示这个TCP通道可以保持5秒，max=100，表示这个长连接最多接收100次请求就断开







## 对称加密和非对称加密

- 对称加密的加密和解密使用的同一个密钥；非对称加密，一般使用公钥进行加密，使用私钥进行解密。
- 对称加密速度快，效率高，非对称加密速度慢，效率低



## https

 HTTPS = HTTP + SSL / TLS。

对称加密+非对称加密



![https原理](images/https原理.png)





1.客户端请求 HTTPS 网址，然后连接到 server 的 443 端口 (HTTPS 默认端口，类似于 HTTP 的80端口)。

2.采用 HTTPS 协议的服务器必须要有一套数字 CA (Certification Authority)证书，证书是需要申请的，并由专门的数字证书认证机构(CA)通过非常严格的审核之后颁发的电子证书 (当然了是要钱的，安全级别越高价格越贵)。颁发证书的同时会产生一个私钥和公钥。私钥由服务端自己保存，不可泄漏。公钥则是附带在证书的信息中，可以公开的。证书本身也附带一个证书电子签名，这个签名用来验证证书的完整性和真实性，可以防止证书被篡改。

3.服务器响应客户端请求，将证书传递给客户端，证书包含公钥和大量其他信息，比如证书颁发机构信息，公司信息和证书有效期等。Chrome 浏览器点击地址栏的锁标志再点击证书就可以看到证书详细信息。

![CA证书](\images\CA证书.png)

4.客户端解析证书并对其进行验证。如果证书不是可信机构颁布，或者证书中的域名与实际域名不一致，或者证书已经过期，就会向访问者显示一个警告，由其选择是否还要继续通信。就像下面这样：

![不合法的CA证书](\images\不合法的CA证书.png)





如果证书没有问题，客户端就会从服务器证书中取出服务器的公钥A。然后客户端还会生成一个随机码 KEY，并使用公钥A将其加密。

5.客户端把加密后的随机码 KEY 发送给服务器，作为后面对称加密的密钥。

6.服务器在收到随机码 KEY 之后会使用私钥B将其解密。经过以上这些步骤，客户端和服务器终于建立了安全连接，完美解决了对称加密的密钥泄露问题，接下来就可以用对称加密愉快地进行通信了。

7.服务器使用密钥 (随机码 KEY)对数据进行对称加密并发送给客户端，客户端使用相同的密钥 (随机码 KEY)解密数据。

8.双方使用对称加密愉快地传输所有数据。

# 并发编程

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

4、非静态内部类创建静态实例造成的内存泄漏

5、ThreadLocal

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

每个线程创建时 JVM 都会为其创建一个工作内存（有些地方称为栈空间），用于存储线程私有的数据。JMM规定所有变量都存储在主内存，主内存是共享内存区域，所有的线程都可以访问。当线程有对这个变量有操作时，必须把这个变量从主内存复制一份到自己的工作空间中进行操作，操作完成后，再把变量写回主内存，不能直接操作主内存的变量。不同的线程无法访问其他线程的工作内存







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

![锁的释放-获取建立的happen-before关系](\images\锁的释放-获取建立的happen-before关系.svg)

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

4、进程间不会相互影响 ；线程一个线程挂掉可能会导致整个进程挂掉

5、 进程创建销毁开销大，切换速度慢；线程正相反，开销小，切换速度快

## 死锁

### 死锁四个必要条件：

- 互斥条件：指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的进程用完释放。
- 请求和保持条件：指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放。
- 不剥夺条件：指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。
- 环路等待条件：指在发生死锁时，必然存在一个进程——资源的环形链，即进程集合{A，B，C，···，Z} 中的A正在等待一个B占用的资源；B正在等待C占用的资源，……，Z正在等待已被A占用的资源。



### 破幻死锁的四个必要条件

- 破坏互斥条件：使资源同时访问而非互斥使用，就没有进程会阻塞在资源上，从而不发生死锁（使用共享锁-->读数据）
- 破坏请求和保持条件：采用静态分配的方式，静态分配的方式是指进程必须在执行之前就申请需要的全部资源，且直至所要的资源全部得到满足后才开始执行，只要有一个资源得不到分配，也不给这个进程分配其他的资源。
- 破坏不剥夺条件：即当某进程获得了部分资源，但得不到其它资源，则释放已占有的资源，但是只适用于内存和处理器资源。
- 破坏循环等待条件：给系统的所有资源编号，规定进程请求所需资源的顺序必须按照资源的编号依次进行。



## interrupt、yield、join





## synchorized

保证原子性、可见性、有序性（但无法禁止指令重排序）

偏向锁

轻量级锁

重量级锁







## synchorized和Lock的对比

- 都支持可重入，

- synchorized是java关键字 ->无需手动加锁和释放锁，Lock是juc下的一个类 -> 需要手动加锁和释放锁
- synchronized 在发生异常时候会自动释放占有的锁，因此不会出现死锁（JVM保证）；而 lock 发生异常时候，不会主动释放占有的锁，必须手动 unlock 来释放锁，可能引起死锁的发生（开发人员保证）

- synchorized是悲观锁，Lock是悲观锁+乐观锁(tryLock)

- Lock支持可中断获取锁、超时获取锁、公平锁
- synchorized在jdk6及之后有优化，偏向锁、轻量级锁、重量级锁

- synchorized通过锁对象wait()、notify()实现线程间的通信机制，Lock通过Condition的await()、singal()实现线程间的通信机制



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



### 固定线程数量的线程池











### 单个线程的线程池



### 可执行定时任务的线程池











## 合理估算线程池的大小

最佳线程数目 = （线程等待时间与线程CPU时间之比 + 1）* CPU数目

线程等待时间：非CPU运行时间，比如IO）

线程CPU时间：每个线程CPU运行时间





**线程等待时间所占比例越高（IO密集型），需要越多线程。线程CPU时间所占比例越高（CPU密集型：大量运算，没有阻塞），需要越少线程。**





# JVM



## jvm运行时数据区划分

堆

方法区

虚拟机栈

本地方法栈

程序计数器





## 标记算法

引用计数算法

可达性分析算法

`GCRoots`：虚拟机栈的对象、本地方法栈的对象、静态变量、锁对象



## 垃圾收集算法

标记-清除

复制算法

标记-整理（压缩）

## 垃圾回收器

年轻代：Serial GC、ParNew GC、Parallel GC

老年代：Serial Old GC、Parallel Old GC、CMS

整堆：G1



## jvm性能优化









## 类加载器

启动类加载器

扩展类加载器

应用类加载器



## 双亲委派机制

如果一个类加载器收到了类加载的请求，它首先不会自己去加载这个类，而是把这个请求委派给父类加载器去完成，每一层的类加载器都是如此，这样所有的加载请求都会被传送到顶层的启动类加载器中，只有当父加载无法完成加载请求（它的搜索范围中没找到所需的类）时，子加载器才会尝试去加载类。



### tomcat打破双亲委派机制

#### 原理

自定义类加载器，重写ClassLoder方法



#### 为什么要重写

- 对于各个 `webapp`中的 `class`和 `lib`，需要相互隔离，不能出现一个应用中加载的类库会影响另一个应用的情况，而对于许多应用，需要有共享的lib以便不浪费资源。
- 与 `jvm`一样的安全性问题。使用单独的 `classloader`去装载 `tomcat`自身的类库，以免其他恶意或无意的破坏；
- 热加载。相信大家一定为 `tomcat`修改文件不用重启就自动重新装载类库而惊叹吧。











# 设计模式

## 单例模式

### 1、饿汉式

```java
/**
 * 饿汉式
 *  优点：
 *      1、单例对象的创建是线程安全的；
 *      2、获取单例对象时不需要加锁。
 *   缺点：单例对象的创建，不是延时加载。
 *
 */
public class Hungry {
    private static final Hungry instance = new Hungry();
    public static Hungry getInstance(){
        return instance;
    }

}
```



### 2、懒汉式

```java
/**
 * 懒汉式
 *
 *  优点：
       1、对象的创建是线程安全的。
       2、支持延时加载。
 * 缺点：
 *    1、获取对象的操作被加上了锁，影响了并发度。
 *    2、如果单例对象需要频繁使用，那这个缺点就是无法接受的。
 *    3、如果单例对象不需要频繁使用，那这个缺点也无伤大雅。
 *
 */
public class Lazy {

    private static Lazy instance = null;
    private Lazy(){}

    public static Lazy getInstance(){

        synchronized (Lazy.class){
            if(instance == null){
                instance = new Lazy();
            }
            return instance;
        }
    }
```

### 3、双重检测懒汉式

```
/**
 * 双重检测懒汉式
 *
 *  双重检测单例优点：
 *
 *    1、对象的创建是线程安全的。
 *    2、支持延时加载。
 *    3、获取对象时不需要加锁。
 */
public class DoubleCheckLazy {

    private static volatile DoubleCheckLazy singleton = null;

    private DoubleCheckLazy(){}

    public static DoubleCheckLazy getInstance(){
        if(singleton == null){
            synchronized (DoubleCheckLazy.class){
                if(singleton == null){
                    singleton = new DoubleCheckLazy();
                }
            }
        }
        return singleton;
    }
}
```



### 4、静态内部类

```java
/**
 * 静态内部类
 *  优点：
 *  1、对象的创建是线程安全的。
 *  2、支持延时加载。
 *  3、获取对象时不需要加锁。
 *
 */
public class StaticInnerClass {


    private StaticInnerClass(){}

    public static StaticInnerClass getInstance(){
        return InnerClass.instance;
    }

    static class InnerClass{
        private static StaticInnerClass instance = new StaticInnerClass();

    }
}
```



### 5、枚举

```java
/**
 * 枚举类
 *  1、多线程安全
 *  2、懒加载
 *  3、序列化/反序列化安全
 *  4、写法简单
 */
public class EnumClass {


    public static void main(String[] args) {
        EnumClass instance = Enum.INSTACNE.getInstance();
    }
    enum Enum{
        INSTACNE;
        private EnumClass enumClass = null;
        private Enum(){
            enumClass = new EnumClass();
        }
        public EnumClass getInstance(){
            return enumClass;
        }
    }
}
```





# 性能分析工具

## 操作系统工具

CPU : vmstat 

磁盘：iostat

网络：netstat

内存：free -m











## java内置工具

- jinfo

查看正在运行的应用程序的扩展参数、修改一部分正在运行的jvm参数

- jstat

查看虚拟机的**GC、堆内存**、新生代内存及垃圾回收、老年代内存及垃圾回收



- jps

列出本机所有java进程的pid

- jconsole

图形化界面，展示线程使用，类使用、GC信息

- jmap

生成堆转储快照dump文件、查看堆中对象的统计信息、查看堆的详细信息



- jstack

生成java虚拟机当前时刻的线程快照

- jhat

分析堆转储文件













# Spring

## springmvc执行顺序：

1、将request和response绑定到当前线程

2、检查文件上传

3、根据request获取由HandlerMethod和拦截器链组成的处理器执行器链HandlerExecutionChain

4、根据HandlerMethod获取HandlerAdaptor

5、执行HandlerInterceptor.preHandle()

6、使用HandlerAdaptor执行HandlerMethod，返回ModelAndView

- 参数解析器
- 方法返回值处理器

7、执行HandlerInterceptor.postHandler()方法，此时response body已经写出去了

8、根据ModelAndView处理分发结果，视图渲染

9、执行HandlerInterceptor.afterCompletion()方法



## 核心概念

IOC（控制反转、依赖注入）和AOP（面向切面）

**IOC：创建和查找依赖对象的控制权交给了容器，由容器进行注入组合对象**



**AOP：把与核心业务逻辑无关的代码全部抽取出来，放置到某个地方集中管理，然后在具体运行时，再由容器动态织入这些共有代码**



##spring中的设计模式

- 工厂模式：BeanFactory就是简单工厂模式的体现，用来创建对象的实例；
- 单例模式：Bean默认为单例模式。
- 代理模式：Spring的AOP功能用到了JDK的动态代理和CGLIB字节码生成技术；
- 模板方法：用来解决代码重复的问题。比如. RestTemplate, JmsTemplate, JpaTemplate。
- 观察者模式：定义对象键一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知被制动更新，如Spring中listener的实现–ApplicationListener。
- 策略模式





## BeanFactory和FactoryBean 







## BeanFactory和ApplicationContext

BeanFactory：是Spring里面最底层的接口，包含了各种Bean的定义，读取bean配置文档，管理bean的加载、实例化，控制bean的生命周期，维护bean之间的依赖关系。

ApplicationContext接口作为BeanFactory的派生，除了提供BeanFactory所具有的功能外，还提供了更完整的框架功能：

- 继承MessageSource，因此支持国际化。
- 统一的资源文件访问方式。
- 提供在监听器中注册bean的事件。
- 同时加载多个配置文件。
- 载入多个（有继承关系）上下文 ，使得每一个上下文都专注于一个特定的层次，比如应用的web层



## 依赖注入

构造器注入

setter方法注入

属性注入



## 循环依赖

- 只支持单例，不支持多例

- 只支持属性和setter循环依赖，不支持构造器循环依赖



A中注入B，B中注入A

假如先进行A的生命周期，把A添加到正在被创建的bean集合中，创建完A对象后，为A初始化一个具有提前完成对象生命周期的工厂（此时工厂并没有执行）并把工厂添加到三级缓存中，在对A进行属性赋值时，需要注入B，发现容器中还没有B，就去进行B的生命周期，创建完B对象后，对B对象进行属性赋值，需要注入A，容器中此时还没有A，（此时对象A还没有完成生命周期），就去进行A的生命周期，先根据A的beanName尝试从一级缓存（单例池）中获取，发现单例池中没有并且A在正在被创建的bean集合中，就根据A的beanName去二级缓存中获取，二级缓存中也没有并且允许提前引用，就根据A的beanName去三级缓存里获取单例工厂，A的单例工厂是存在的，就调用单例工厂提前完成A的生命周期(aop)，

```
AbstractAutoProxyCreator#getEarlyBeanReference() : 提前完成bean的aop，把beanName添加到earlyProxyReferences集合中
  AbstractAutoProxyCreator#postProcessAfterInitialization()中判断beanName不在earlyProxyReferences集合中，才会去进行代理
```

然后把完全体的A放到二级缓存，把A的对象工厂从三级缓存中移除。

B的生命周期完成之后，继续A的属性赋值，然后执行A的初始化方法，最后调用AbstractAutoProxyCreator#postProcessAfterInitialization判断是否需要aop（此时不会进行aop了）



#### Spring循环依赖三级缓存是否可以去掉第三级缓存？







## 给容器中注册组件

- @ComponentScan + @Component

- @Bean + @Configuration

- @Import快速导入

  ​	1、@Import(要导入到容器中的组件)：容器自动注册这个组件，组件id默认是全类名

  ​	2、@Import(ImportSelector)：ImportSelector 返回需要导入的组件的全类名数组

  ​	3、@Import(ImportBeanDefinitionRegistrar)：ImportBeanDefinitionRegistrar 手动注册bean到容器中

- FactoryBean





## BeanPostProcessor的执行顺序

第一次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean()

```java
//为aop做准备
//实际执行 InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);

```

第二次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean()

```java
//推断构造方法
//实际执行SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors()
instanceWrapper = createBeanInstance(beanName, mbd, args);
```

第三次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean()

```java
//先找到需要注入的属性信息(@Autowired、@Value、@Resource)并缓存 以便后面注入时使用
//MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()
applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
```

第四次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean()

```java
//getEarlyBeanReference():判断是否需要aop 为循环依赖提前暴露工厂 
//实际执行 SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

第五次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean()

```java
//是否进行属性注入 postProcessAfterInstantiation()返回false == 不进行属性注入
InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
```

第六次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean()

```java
//属性注入
InstantiationAwareBeanPostProcessor.postProcessPropertyValues()
```

第七次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean()

```java
//BeanPostProcessor.postProcessBeforeInitialization  处理jsr规范提供的声明周期注解
//CommonAnnotationBeanPostProcessor在此处执行,处理@PostConstruct @PreDestory注解
wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
//spring的声明周期方法在之后执行，但是是直接写在代码里了，没有通过BeanPostProcessor进行调用
```

```java
/**
 * 调用spring的生命周期方法
 * 1、实现InitializingBean
 * 2、@Bean(init = "",destory = "")
 */
```



第八次：org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean()

```java
//BeanPostProcessor.postProcessAfterInitialization
//aop实现：AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator
wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
```







推断构造方法

为循环依赖做准备

属性注入

aop









## bean的生命周期

创建（包括属性赋值）：

- ​	单实例 容器启动的时候创建对象
- ​	多实例 每次获取的时候创建对象

 

BeanPostProcessor.postProcessBeforeInitialization

初始化：对象创建完成，并赋好值，调用初始化方法

BeanPostProcessor.postProcessAfterInitialization

 

销毁：

- ​	单实例 容器关闭的时候

- ​	多实例 容器不会管理多实例bean，不会调用销毁方法

  

①@Bean(initMethod,destroyMethod);

②通过让bean实现InitializingBean定义初始化逻辑，实现DisposableBean定义销毁逻辑（代码有侵入）

​	注：Spring不推荐使用InitializationBean 来调用其初始化方法，因为它不必要地将代码耦合到Spring

③使用JSR250规范提供的注解：spring中提供了CommonAnnotationBeanPostProcessor负责支持@PostConstruct和@PreDestroy

​	@PostConstruct：在bean创建并完成属性赋值后，执行初始化逻辑

​	@PreDestroy：在容器销毁bean之前执行销毁逻辑

④BeanPostProcessor：bean的后置处理器，在bean初始化前后进行一些处理工作

​	postProcessBeforeInitialization：在bean初始化之前调用

​	postProcessAfterInitialization：在bean初始化之后调用





初始化顺序：

@PostConstruct

InitializingBean

@Bean(initMethod)





## Spring自动装配方式

- no：默认的方式是不进行自动装配的，通过手工设置ref属性来进行装配bean。

- byName：通过bean的名称进行自动装配，如果一个bean的 property 与另一bean 的name 相同，就进行自动装配。

- byType：通过参数的数据类型进行自动装配。

- constructor：利用构造函数进行装配，并且构造函数的参数通过byType进行装配。

  

## @Autowired

①默认优先按照类型去容器中找对应的组件；

**②如果根据类型找到多个类型相同的组件，再将属性的名称作为组件的id去容器中查找，如果根据属性名称找不到，则报错；如果根据类型找不到组件，则报错**

③@Qualifier 使用@Qualifier指定需要装配的组件id，而不是属性名

④@Autowired自动装配默认一定要完成组件注入（容器中可能没有这个组件），没有就会报错，可以使用@Autowired(required = false)

⑤@Primary，让spring进行自动装配的时候，默认使用首选的Bean，也可以继续使用@Qualifier知道需要装配的bean的名字 

优先级： @Qualifier > @Primary





## spring aop



`AbstractAutoProxyCreator`



- jdk动态代理
- cglib代理

将增强逻辑封装成拦截器链







### spring切面5种通知类型

前置通知

后置通知

环绕通知

返回通知

异常通知





## spring事务

`TransactionInterceptor`



### spring事务传播行为

**支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRED：** 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
- **TransactionDefinition.PROPAGATION_SUPPORTS：** 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
- **TransactionDefinition.PROPAGATION_MANDATORY：** 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）

**不支持当前事务的情况：**

- **TransactionDefinition.PROPAGATION_REQUIRES_NEW：** 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NOT_SUPPORTED：** 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
- **TransactionDefinition.PROPAGATION_NEVER：** 以非事务方式运行，如果当前存在事务，则抛出异常。

**其他情况：**

- **TransactionDefinition.PROPAGATION_NESTED：** 如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。





## spring监听器

- 实现ApplicationListener接口并添加到容器中 由ApplicationListenerDetector后置处理器处理，添加到监听器列表
- 方法上加@EventListener注解   由EventListenerMethodProcessor处理（bean工厂的后置处理器）



事件多播器派发事件时，会根据事件类型找监听该事件的监听器（事件泛型）









# redis

## 数据类型







## 数据结构

- 简单动态字符串

```java
struct sdshdr {
    // 记录buf数组中已使用字节的数量，即SDS所保存字符串的长度
    unsigned int len;
    // 记录buf数据中未使用的字节数量
    unsigned int free;
    // 字节数组，用于保存字符串
    char buf[];
};
```



- 跳跃表

有序集合（Sorted Set）

1. **多层**的结构组成，每层是一个**有序的链表**
2. 最底层（level 1）的链表包含所有的元素
3. 跳跃表的查找次数近似于层数，时间复杂度为O(logn)，插入、删除也为 O(logn)
4. 跳跃表是一种随机化的数据结构(通过抛硬币来决定层数)



- 压缩列表







## redis高性能

- redis基于纯内存操作，内存读写速度快
- redis单线程，减少线程上下文切换和避免使用锁（保证原子性）
- redis io对路复用模型
- redis数据结构
- 自定义通信协议，编解码简单





## redis持久化

### aof

过程

- 命令追加(append)：将Redis的写命令追加到缓冲区aof_buf；
- 文件写入(write)和文件同步(sync)：根据不同的同步策略将aof_buf中的内容同步到硬盘；
- 文件重写(rewrite)：定期重写AOF文件，达到压缩的目的。

缓存区的文件同步策略：

​	always：命令写入aof_buf后立即调用系统fsync操作同步到AOF文件，fsync完成后线程返回

​	no：命令写入aof_buf后调用系统write操作，不对AOF文件做fsync同步；同步由操作系统负责，通常同步周期为30秒

​	everysec（推荐）：命令写入aof_buf后调用系统write操作，write完成后线程返回；fsync同步文件操作由专门的线程每秒调用一次

优点：

1、AOF可以更好的保护数据不丢失，一般AOF会以每隔1秒，通过后台的一个线程去执行一次fsync操作，如果redis进程挂掉，最多丢失1秒的数据。适合做**热备**

 2、AOF以appen-only的模式写入，所以没有任何磁盘寻址的开销，写入性能非常高。

 3、AOF日志文件的命令通过非常可读的方式进行记录，这个非常适合做灾难性的误删除紧急恢复，

### rdb（内存快照）

触发方式：

手动触发

​	save（废弃）：save命令会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在Redis服务器阻塞期间，服务器不能处理任何命令请求)

​	bgsave：bgsave命令会创建一个子进程，由子进程来负责创建RDB文件，父进程(即Redis主进程)则继续处理请求

自动触发

​	save m n：指定当m秒内发生n次变化时，会触发bgsave。

优点：

1、RDB会生成多个数据文件，每个数据文件都代表了某一个时刻中redis的数据，这种多个数据文件的方式，非常适合做**冷备**。
2、RDB对redis对外提供读写服务的时候，影像非常小，因为redis  主进程只需要fork一个子进程出来，让子进程对磁盘io来进行rdb持久化
3、RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

缺点：

1、如果redis要故障时要尽可能少的丢失数据，RDB没有AOF好，例如1:00进行的快照，在1:10又要进行快照的时候宕机了，这个时候就会丢失10分钟的数据。
 2、RDB每次fork出子进程来执行RDB快照生成文件时，如果文件特别大，可能会导致客户端提供服务暂停数毫秒或者几秒

## redis键的过期策略

惰性过期

定期过期

定时过期

**Redis中同时使用了惰性过期和定期过期两种过期策略。**

## redis的内存淘汰机制

LRU：最近最少使用

- 新数据直接插入到列表头部
- 缓存数据被命中，将数据移动到列表头部
- 缓存已满的时候，移除列表尾部数据。



**allkeys-lru**：从所有key中使用LRU算法进行淘汰

noeviction（默认）：对于写请求不再提供服务，直接返回错误

volatile-lru：从设置了过期时间的key中使用LRU算法进行淘汰

allkeys-random：从所有key中随机淘汰数据

volatile-random：从设置了过期时间的key中随机淘汰

volatile-ttl：在设置了过期时间的key中，根据key的过期时间进行淘汰，越早过期的越优先被淘汰







## 主从复制

1、连接建立

2、数据同步

- 全量同步
- 部分同步

3、命令传播

## sentinel哨兵原理



- 定时任务

1、通过向master发送info命令获取最新的主从结构

2、通过发布订阅功能获取其他哨兵节点的信息

3、通过向其他节点发送ping命令进行心跳检测，判断是否下线

- 主观下线

在心跳检测的定时任务中，如果其他节点超过一定时间没有回复，哨兵节点就会将其进行主观下线

- 客观下线（只有master节点才需要进行客观下线）

哨兵节点在对主节点进行主观下线后，会通过sentinel is-master-down-by-addr命令询问其他哨兵节点该主节点的状态；如果判断主节点下线的哨兵数量达到一定数值，则对该主节点进行客观下线

- 选举领导者哨兵节点

当主节点被判断客观下线以后，各个哨兵节点会进行协商，选举出一个领导者哨兵节点，并由该领导者节点对其进行故障转移操作。

监视该主节点的所有哨兵都有可能被选为领导者，选举使用的算法是Raft算法；Raft算法的基本思路是先到先得：即在一轮选举中，哨兵A向B发送成为领导者的申请，如果B没有同意过其他哨兵，则会同意A成为领导者。选举的具体过程这里不做详细描述，一般来说，哨兵选择的过程很快，**谁先完成客观下线，一般就能成为领导者**。

- 故障转移

1、在从节点中选择新的主节点：选择的原则是，首先过滤掉不健康的从节点；然后选择优先级最高的从节点(由slave-priority指定)；如果优先级无法区分，则选择复制偏移量最大的从节点；如果仍无法区分，则选择runid最小的从节点。

2、更新主从状态：通过slaveof no one命令，让选出来的从节点成为主节点；并通过slaveof命令让其他节点成为其从节点。

3、将已经下线的主节点(即6379)设置为新的主节点的从节点，当6379重新上线后，它会成为新的主节点的从节点。







## 缓存穿透

查询数据库和redis都不存在的数据，导致一直查询数据库

- 布隆过滤器
- 缓存空值



## 缓存雪崩

redis服务宕机   或 大量的数据在同一个时间点全部过期（热点key同时失效）

- redis高可用
- 过期时间随机
- 熔断限流







## 缓存击穿

一个热点key,在不停的扛着大量的并发，当key在失效的瞬间，持续的大并发就会穿破缓存，直接请求到数据库。对数据库造成瞬间压力过大。

- 热点key永不过期
- 互斥锁









## 一致性hash算法



- hash取余分区

计算key的hash值，然后对节点数量进行取余，从而决定数据映射到哪个节点上。该方案最大的问题是，当新增或删减节点时，节点数量发生变化，系统中所有的数据都需要重新计算映射关系，引发大规模数据迁移。

- 一致性hash分区

一致性哈希算法将整个哈希值空间组织成一个虚拟的圆环，如下图所示，范围为0-2^32-1；对于每个数据，根据key计算hash值，确定数据在环上的位置，然后从此位置沿环顺时针行走，找到的第一台服务器就是其应该映射到的服务器。

![一致性hash分区](D:\repo\source-code\images\一致性hash分区.png)





与哈希取余分区相比，一致性哈希分区将增减节点的影响限制在相邻节点。以上图为例，如果在node1和node2之间增加node5，则只有node2中的一部分数据会迁移到node5；如果去掉node2，则原node2中的数据只会迁移到node4中，只有node4会受影响。

一致性哈希分区的主要问题在于，当节点数量较少时，增加或删减节点，对单个节点的影响可能很大，造成数据的严重不平衡。还是以上图为例，如果去掉node2，node4中的数据由总数据的1/4左右变为1/2左右，与其他节点相比负载过高。

- 带虚拟节点的一致性hash分区

该方案在一致性哈希分区的基础上，引入了虚拟节点的概念。**Redis****集群使用的便是该方案，其中的虚拟节点称为槽（slot****）。**槽是介于数据和实际节点之间的虚拟概念；每个实际节点包含一定数量的槽，每个槽包含哈希值在一定范围内的数据。引入槽以后，数据的映射关系由数据hash->实际节点，变成了数据hash->槽->实际节点。

**在使用了槽的一致性哈希分区中，槽是数据管理和迁移的基本单位。槽解耦了数据和实际节点之间的关系，增加或删除节点对系统的影响很小**



##redis cluster原理



redis6.0支持多线程



![redis6多线程](D:\repo\source-code\images\redis6多线程.png)







**主要流程**：

1. 主线程负责接收建立连接请求，获取 `socket` 放入全局等待读处理队列；
2. 主线程通过轮询将可读 `socket` 分配给 IO 线程；
3. 主线程阻塞等待 IO 线程读取 `socket` 完成；
4. 主线程执行 IO 线程读取和解析出来的 Redis 请求命令；
5. 主线程阻塞等待 IO 线程将指令执行结果回写回 `socket`完毕；
6. 主线程清空全局队列，等待客户端后续的请求。



思路：**将主线程 IO 读写任务拆分出来给一组独立的线程处理，使得多个 socket 读写可以并行化，但是 Redis 命令还是主线程串行执行**



我们所说的redis单线程：**Redis 在处理客户端的请求时，包括获取 (socket 读)、解析、执行、内容返回 (socket 写) 等都由一个顺序串行的主线程处理，这就是所谓的「单线程」**







# rocketmq

## 消息类型

单向消息

同步消息

异步消息

顺序消息

延迟消息

事物消息

批量消息

`消息过滤`

`重试消息`



## 事物消息





## 顺序消息





## 延迟消息与消息重试





## 刷盘机制





## 消息存储

commitLog：存储消息内容

consumequeue：存储消息索引，根据`topic`和`queueId`来组织文件，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值

IndexFile（索引文件）：只是为消息查询提供了一种通过key或时间区间来查询消息的方法









# kafka



## rocketmq与kafka比较

架构

事物消息

延迟消息

消息过滤

消息重试

服务发现

消息存储

同步刷盘

异步刷盘









# mysql





## innodb和myisam区别

- 存储格式

innodb两个文件：.frm(表定义)、.ibd

myisam三个文件：.frm（标定仪）、.MYD（数据文件）、.MYI(索引文件)

- 事物支持

innodb支持事务、行级锁、外键，myisam都不支持

- 聚簇索引和非聚簇索引

innodb：聚簇索引（将数据行和索引集中存储），按主键大小有序插入

myisam：非聚簇索引（将数据行和索引分开存储，索引结构的叶子节点指向了数据的对应行），按记录插入顺序保存

- 表的具体行数

count(*)：myisam内部维护了一个计数器，可以直接调取

- 全文索引的支持

  innodb默认不支持fulltext数据类型



## 取topN







## sql顺序

### sql语句的语法顺序：

(1)SELECT[DISTINCT]

(2)FROM

(3)JOIN

(4)ON

(5)WHERE

(6)GROUP BY

(7)HAVING

(8)UNION

(9)ORDER BY

(10)LIMIT



### sql语句的执行顺序：

(1)from 

(3) join 

(2) on 

(4) where 

(5)group by(开始使用select中的别名，后面的语句中都可以使用)

(6) avg,sum.... 

(7)having 

(8) select 

(9) distinct 

(10)union

(11) order by

(12) limit 







## 创建索引的原则

1） 最左前缀匹配原则，**组合索引**非常重要的原则，mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。

2）较频繁作为查询条件的字段

3）更新频繁字段不适合创建索引

4）不能有效区分数据（列的离散度）的列不适合做索引列

5）尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。

6）定义有外键的数据列一定要建立索引。

7）查询中order by、group by的字段

8）对于定义为text、image和bit的数据类型的列不要建立索引。





## 索引失效情况：

没有遵循最左前缀原则

在索引列上做操作

使用> < !=

左模糊

is null或 is not null

字符串没加单引号

范围之后的索引失效  between and

使用or连接



## 前缀索引

对于一些字段比较长的列，可以使用前缀索引，截取这个列的前部分作为索引

语法：`index(field(10))`，使用字段值的前10个字符建立索引，默认是使用字段的全部内容建立索引。

前提：前缀的标识度高。比如密码就适合建立前缀索引，因为密码几乎各不相同。

实操的难度：在于前缀截取的长度。

我们可以利用`select count(*)/count(distinct left(password,prefixLen));`，通过从调整`prefixLen`的值（从1自增）查看不同前缀长度的一个平均匹配度，接近1时就可以了（表示一个密码的前`prefixLen`个字符几乎能确定唯一一条记录）



##聚簇索引和非聚簇索引





## 事务

### 事务特性原理（ACID）

- 原子性

 undolog，实现原子性的关键，是当事务回滚时能够撤销所有已经成功执行的sql语句。InnoDB实现回滚，靠的是undo log：当事务对数据库进行修改时，InnoDB会生成对应的undo log；如果事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的样子。



- 持久性 

redo log，事务开始之后就产生redo log，redo log的落盘并不是随着事务的提交才写入的，在内存数据页准备修改前将修改信息写入内存中的redo buffer中，然后再对内存数据页中的数据执行修改操作；而且保证在发出事务提交指令时，先向内存中的redo buffer写入日志，写入完成后才执行提交动作；聚集索引、二级索引、undo页面的修改，均需要记录Redo日志

- 隔离性 

​	   (一个事务)写操作对(另一个事务)写操作的影响：锁机制保证隔离性

​	   (一个事务)写操作对(另一个事务)读操作的影响：MVCC保证隔离性

- 一致性

一致性指数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态

​    保证原子性、持久性和隔离性，如果这些特性无法保证，事务的一致性也无法保证

​    数据库本身提供保障，例如不允许向整形列插入字符串值、字符串长度不能超过列的限制等

​    应用层面进行保障，例如如果转账操作只扣除转账者的余额，而没有增加接收者的余额，无论数据库实现的多么完美，也无法保证状态的一致



### 脏读  不可重复读 幻读

脏读：读未提交

不可重复读：前后多次读取，数据内容不一致（内容被修改或被删除） -> update/delete

幻读：前后多次读取，数据总量不一样（新增）-> insert 

**幻读在当前读下才会出现，幻读仅指新插入的行**





### MVCC(多版本并发控制)

#### 快照读与当前读

**快照读（SnapShot Read）** 是一种**一致性不加锁的读**

> 这里的**一致性**是指，事务读取到的数据，要么是**事务开始前就已经存在的数据**，要么是**事务自身插入或者修改过的数据**。

**当前读**就是读取最新数据，而不是历史版本的数据。加锁的 SELECT 就属于当前读

```sql
SELECT * FROM t WHERE id=1 LOCK IN SHARE MODE;

SELECT * FROM t WHERE id=1 FOR UPDATE;
```

#### 事务版本号

每开启一个事务，我们都会从数据库中获得一个事务 ID（也就是事务版本号），这个事务 ID 是自增长的，通过 ID 大小，我们就可以判断事务的时间顺序



#### 行记录的隐藏列

InnoDB 的叶子段存储了数据页，数据页中保存了行记录，而在行记录中有一些重要的隐藏字段：

- **DB_ROW_ID** 6byte, 隐含的自增ID（隐藏主键），如果数据表没有主键，InnoDB会自动以DB_ROW_ID产生一个聚簇索引

- **DB_TRX_ID** 6byte, 最近修改(修改/插入)事务ID：记录创建这条记录/最后一次修改该记录的事务ID

- **DB_ROLL_PTR** 7byte, 回滚指针，指向这条记录的上一个版本（存储于rollback segment里）

- **DELETED_BIT** 1byte, 记录被更新或删除并不代表真的删除，而是删除flag变了



#### undo日志

undo log 存储的是**逻辑格式**的日志，记录变化的过程，根据每行记录进行记录，保存了事务发生之前的上一个版本的数据，可以用于回滚。当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着 undo 链找到满足其可见性的记录。由于查询操作（SELECT）并不会修改任何用户记录，所以在查询操作执行时，并不需要记录相应的undo log。undo log主要分为3种：

- **Insert undo log** ：在insert 操作中产生的undo log，因为insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。
- **Update undo log**：update操作产生的undolog，包含被修改行的主键(以便知道修改了哪些行)、修改了哪些列、这些列在修改前后的值等信息，回滚时便可以使用这些信息将数据还原到update之前的状态。
- Delete undo log
  - 删除操作只是设置一下老记录的DELETED_BIT，并不真正将过时的记录删除。
  - 为了节省磁盘空间，InnoDB有专门的purge线程来清理DELETED_BIT为true的记录。为了不影响MVCC的正常工作，purge线程自己也维护了一个read view（这个read view相当于系统中最老活跃事务的read view）;如果某个记录的DELETED_BIT为true，并且DB_TRX_ID相对于purge线程的read view可见，那么这条记录一定是可以被安全清除的。



对MVCC有帮助的实质是**update undo log** ，undo log实际上就是存在rollback segment中旧记录链，它的执行流程如下：

1. **比如一个事务向插入persion表插入了一条新记录，记录如下，name为Jerry, age为24岁，隐式主键是1，事务ID和回滚指针，我们假设为NULL**

![db-mysql-mvcc-1](images\db-mysql-mvcc-1.png)

2、**现在来了一个事务1对该记录的name做出了修改，改为Tom**

1. 在事务1修改该行(记录)数据时，数据库会先对该行加排他锁
2. 然后把该行数据拷贝到undo log中，作为旧记录，即在undo log中有当前行的拷贝副本
3. 拷贝完毕后，修改该行name为Tom，并且修改隐藏字段的事务ID为当前事务1的ID, 我们默认从1开始，之后递增，回滚指针指向拷贝到undo log的副本记录，既表示我的上一个版本就是它
4. 事务提交后，释放锁

![db-mysql-mvcc-2](\images\db-mysql-mvcc-2.png)



3、**又来了个事务2修改person表的同一个记录，将age修改为30岁**

1. 在事务2修改该行数据时，数据库也先为该行加锁
2. 然后把该行数据拷贝到undo log中，作为旧记录，发现该行记录已经有undo log了，那么最新的旧数据作为链表的表头，插在该行记录的undo log最前面
3. 修改该行age为30岁，并且修改隐藏字段的事务ID为当前事务2的ID, 那就是2，回滚指针指向刚刚拷贝到undo log的副本记录
4. 事务提交，释放锁

![db-mysql-mvcc-3](\images\db-mysql-mvcc-3.png)



从上面，我们就可以看出，不同事务或者相同事务的对同一记录的修改，会导致该记录的undo log成为一条记录版本线性表，既链表，undo log的链首就是最新的旧记录，链尾就是最早的旧记录（当然就像之前说的该undo log的节点可能是会purge线程清除掉，向图中的第一条insert undo log，其实在事务提交之后可能就被删除丢失了，不过这里为了演示，所以还放在这里）



#### Read View

Read View就是事务进行快照读操作的时候生产的读视图(Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID(当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以最新的事务，ID值越大)

ReadView 主要包含以下几个部分：
m_ids：生成 ReadView 时系统中活跃的事务 id 集合
min_trx_id：生成 ReadView 时系统中活跃的最小事务 id，也就是 m_ids 中的最小值
max_trx_id：生成 ReadView 时系统中应该分配给下一个事务的 id 值，比 m_ids 的最大值要大
creator_trx_id：生成该 ReadView 的事务 id

由于 READ UNCOMMITTED 级别下每次读取最新记录，SERIALIZABLE 级别下通过加锁访问数据，
**所以 ReadView 仅对 READ COMMITTED 和 REPEATABLE READ 有效。**



怎么通过这个 ReadView 来判别当前事务能看到的版本呢？过程如下：

1. 如果被访问版本的 trx_id 与 ReadView 中的 creator_trx_id 值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。

2. 如果被访问版本的 trx_id 小于 ReadView 中的 min_trx_id 值，表明生成该版本的事务在当前事务生成 ReadView 前已经提交，所以该版本可以被当前事务访问。

3. 如果被访问版本的 trx_id 大于 ReadView 中的 max_trx_id 值，表明生成该版本的事务在当前事务生成 ReadView 后才开启，所以该版本不可以被当前事务访问。

4. 如果被访问版本的 trx_id 属性值在 ReadView 的 min_trx_id 和 max_trx_id 之间，那就需要判断一下 trx_id 是不是在 m_ids 列表中，
   如果在，说明生成 ReadView 时生成该版本的事务还是活跃的，该版本不可以被访问；
   如果不在，说明生成 ReadView 时生成该版本的事务已经被提交，该版本可以被访问；

5. 如果某个版本的数据对当前事务不可见的话，那就顺着版本链找到下一个版本的数据，继续按照上边的步骤判断可见性，依此类推，直到版本链中的最后一个版本。
   如果最后一个版本也不可见的话，那么就意味着该条记录对该事务完全不可见，查询结果就不包含该记录。

   

ReadView 生成时机：

- **READ COMMITTED —— 每次读取数据前都生成一个ReadView（不能保证可重复读）**
- **REPEATABLE READ —— 在第一次读取数据时生成一个ReadView（能够保证可重复读）**



在RR级别下的某个事务的对某条记录的第一次快照读会创建一个快照及Read View, 将当前系统活跃的其他事务记录起来，此后在调用快照读的时候，还是使用的是同一个Read View，所以只要当前事务在其他事务提交更新之前使用过快照读，那么之后的快照读使用的都是同一个Read View，所以对之后的修改不可见；

即RR级别下，快照读生成Read View时，Read View会记录此时所有其他活动事务的快照，这些事务的修改对于当前事务都是不可见的。而早于Read View创建的事务所做的修改均是可见

而在RC级别下的，事务中，**每次**快照读都会新生成一个快照和Read View, 这就是我们在RC级别下的事务中可以看到别的事务提交的更新的原因





### 事务隔离级别

Read Uncommitted：读未提交 ==>会出现脏读



Read Committed：读已提交==>解决脏读，会出现不可重复读



Repetable Read：（使用mvcc机制） 可重复读==>解决脏读、不可重复读 mysql默认隔离级别

Serializable：串行化==>解决脏读、不可重复读、幻读



###sql优化

1、开启慢查询并设置阈值进行捕获

2、explain + 慢sql分析

3、show profile查询sql的执行细节和生命周期

4、sql数据服务器的参数调优



数据库优化

1、建索引、优化sql

2、选择合适的存储引擎

3、分库分表、读写分离

4、表设计





###order by 使用索引排序

1、order by 使用索引最左前缀列

2、使用where子句与order by 子句条件列组合满足索引最左前缀(where使用索引的最左前缀定义为常量)



order by优化

1、order by时只select需要的字段

2、尝试提供sort_buffer_size

3、尝试提高max_length_for_sort_buffer





####大表数据查询，怎么优化

1. 优化shema、sql语句+索引；
2. 加缓存，memcached, redis；
3. 主从复制，读写分离；
4. 垂直拆分，根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统；
5. 水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key, 为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改，sql中尽量带sharding key，将数据定位到限定的表上去查，而不是扫描全部的表；

####超大分页怎么处理？

超大的分页一般从两个方向上来解决.

```sql
select * from product limit 866613, 20   37.44秒

# 只查询Id
select id from product limit 866613, 20 0.2秒

# 查询所有字段
SELECT * FROM product WHERE ID > =(select id from product limit 866613, 1) limit 20

# 查询所有字段的另一种写法
SELECT * FROM product a JOIN (select id from product limit 866613, 20) b ON a.ID = b.id


如果对于有where 条件，又想走索引用limit的，必须设计一个索引，将where 后的字段放第一位，limit用到的主键放第2位，而且只能select 主键！
```



###慢查询优化

- 首先分析语句，看看是否load了额外的数据，可能是查询了多余的行并且抛弃掉了，可能是加载了许多结果中并不需要的列，对语句进行分析以及重写。
- 分析语句的执行计划，然后获得其使用索引的情况，之后修改语句或者修改索引，使得语句可以尽可能的命中索引。
- 如果对语句的优化已经无法进行，可以考虑表中的数据量是否太大，如果是的话可以进行横向或者纵向的分表。

##count(*) count(1) count(列)

###执行效果

- count(*)包括了所有的列，相当于行数，在统计结果的时候，不会忽略列值为NULL
- count(1)包括了忽略所有列，用1代表代码行，在统计结果的时候，不会忽略列值为NULL
- count(列名)只包括列名那一列，在统计结果的时候，会忽略列值为空（这里的空不是只空字符串或者0，而是表示null）的计数，即某个字段值为NULL时，不统计。

###执行效率

列名为主键，count(列名)会比count(1)快 
列名不为主键，count(1)会比count(列名)快 
如果表多个列并且没有主键，则 count（1） 的执行效率优于 count（*）
如果有主键，则 select count（主键）的执行效率是最优的 
如果表只有一个字段，则 select count（*）最优。



exists和in对比

exists对外表用loop逐条查询，每次查询都会查看exists的条件语句，当 exists里的条件语句能够返回记录行时(无论记录行是的多少，只要能返回)，条件就为真，返回当前loop到的这条记录，反之如果exists里的条 件语句不能返回记录行，则当前loop到的这条记录被丢弃，exists的条件就像一个bool条件，当能返回结果集则为true，不能返回结果集则为 false

```
select * from user where exists (select 1);
```

对user表的记录逐条取出，由于子条件中的select 1永远能返回记录行，那么user表的所有记录都将被加入结果集，所以与 select * from user;是一样的

又如下

> select * from user where exists (select * from user where userId = 0);

可以知道对user表进行loop时，检查条件语句(select * from user where userId = 0),由于userId永远不为0，所以条件语句永远返回空集，条件永远为false，那么user表的所有记录都将被丢弃

**not exists与exists相反，也就是当exists条件有结果集返回时，loop到的记录将被丢弃，否则将loop到的记录加入结果集**

总的来说，**如果A表有n条记录，那么exists查询就是将这n条记录逐条取出，然后判断n遍exists条件** 

not exists与exists相反，也就是当exists条件有结果集返回时，loop到的记录将被丢弃，否则将loop到的记录加入结果集

in查询相当于多个or条件的叠加

```
select * from user where userId in (1, 2, 3);
```

等效于

```
select * from user where userId = 1 or userId = 2 or userId = 3;
```



in查询就是先将子查询条件的记录全都查出来，假设in子查询的结果集为B，共有m条记录，然后在将子查询条件的结果集分解成m个，再进行m次查询



**in查询的子条件返回结果必须只有一个字段，exists没有这个限制**

例如

```
select * from user where userId in (select id from B);
```

而不能是

```
select * from user where userId in (select id, age from B);
```



考虑如下sql

```
1: select * from A where exists (select * from B where B.id = A.id);

2: select * from A where A.id in (select id from B);
```



查询1.可以转化以下伪代码，便于理解

```java
for ($i = 0; $i < count(A); $i++) {

　　$a = get_record(A, $i); #从A表逐条获取记录

　　if (B.id = $a[id]) #如果子条件成立

　　　　$result[] = $a;

}

return $result;。
```

大概就是这么个意思，其实可以看到,**查询1主要是用到了B表的索引**，A表如何对查询的效率影响应该不大



假设B表的所有id为1,2,3,查询2可以转换为

```
select * from A where A.id = 1 or A.id = 2 or A.id = 3;
```

这个好理解了，**这里主要是用到了A的索引，B表如何对查询影响不大**





下面再看not exists 和 not in

```
1. select * from A where not exists (select * from B where B.id = A.id);

2. select * from A where A.id not in (select id from B);
```

看查询1，还是和上面一样，用了B的索引

而对于查询2，可以转化成如下语句

```
select * from A where A.id != 1 and A.id != 2 and A.id != 3;
```

可以知道not in是个范围查询，这种**!=的范围查询无法使用任何索引**,等于说A表的每条记录，都要在B表里遍历一次，查看B表里是否存在这条记录

故not exists比not in效率高



**如果查询的两个表大小相当，那么用in和exists差别不大**。

**如果两个表中一个较小，一个是大表，则子查询表大的用exists，子查询表小的用in**



not in 和not exists如果查询语句使用了not in 那么内外表都进行全表扫描，没有用到索引；而**not extsts 的子查询依然能用到表上的索引**。**所以无论那个表大，用not exists都比not in要快**。





##mysql主从复制原理





##mysql读写分离解决方案

###方案一

使用mysql-proxy代理

优点：直接实现读写分离和负载均衡，不用修改代码，master和slave用一样的帐号，mysql官方不建议实际生产中使用

缺点：降低性能， 不支持事务

###方案二

使用AbstractRoutingDataSource+aop+annotation在dao层决定数据源。
如果采用了mybatis， 可以将读写分离放在ORM层，比如mybatis可以通过mybatis plugin拦截sql语句，所有的insert/update/delete都访问master库，所有的select 都访问salve库，这样对于dao层都是透明。 plugin实现时可以通过注解或者分析语句是读写方法来选定主从库。不过这样依然有一个问题， 也就是不支持事务， 所以我们还需要重写一下DataSourceTransactionManager， 将read-only的事务扔进读库， 其余的有读有写的扔进写库。

###方案三

使用AbstractRoutingDataSource+aop+annotation在service层决定数据源，可以支持事务.

缺点：类内部方法通过this.xx()方式相互调用时，aop不会进行拦截，需进行特殊处理。













#linux

##目录说明

- **/bin**： 存放二进制可执行文件(ls,cat,mkdir等)，常用命令一般都在这里；
- **/etc**： 存放系统管理和配置文件；
- **/home**： 存放所有用户文件的根目录，是用户主目录的基点，比如用户user的主目录就是/home/user，可以用~user表示；
- **/usr **： 用于存放系统应用程序；
- **/opt**： 额外安装的可选应用程序包所放置的位置。一般情况下，我们可以把tomcat等都安装到这里；
- **/proc**： 虚拟文件系统目录，是系统内存的映射。可直接访问这个目录来获取系统信息；
- **/root**： 超级用户（系统管理员）的主目录（特权阶级o）；
- **/sbin:** 存放二进制可执行文件，只有root才能访问。这里存放的是系统管理员使用的系统级别的管理命令和程序。如ifconfig等；
- **/dev**： 用于存放设备文件；
- **/mnt**： 系统管理员安装临时文件系统的安装点，系统提供这个目录是让用户临时挂载其他的文件系统；
- **/boot**： 存放用于系统引导时使用的各种文件；
- **/lib **： 存放着和系统运行相关的库文件 ；
- **/tmp**： 用于存放各种临时文件，是公用的临时文件存储点；
- **/var**： 用于存放运行时需要改变数据的文件，也是某些大文件的溢出区，比方说各种服务的日志文件（系统启动日志等。）等；
- **/lost+found**： 这个目录平时是空的，系统非正常关机而留下“无家可归”的文件（windows下叫什么.chk）就在这里。



文件管理命令

###cat

cat 命令用于连接文件并打印到标准输出设备上。

cat 主要有三大功能：

1.一次显示整个文件:

```sh
cat filename
```

2.从键盘创建一个文件:

```shell
cat > filename
```

只能创建新文件，不能编辑已有文件。

3.将几个文件合并为一个文件:

```shell
cat file1 file2 > file
```

###chmod

用于改变 linux 系统文件或目录的访问权限

每一文件或目录的访问权限都有三组，每组用三位表示，分别为文件属主的读、写和执行权限；与属主同组的用户的读、写和执行权限；系统中其他用户的读、写和执行权限。可使用 ls -l test.txt 查找

```shell
chmod 751 t.log 
```



###chown

将指定文件的拥有者改为指定的用户或组



###head

用来显示档案的开头至标准输出中，默认 head 命令打印其相应文件的开头 10 行

###less

使用 less 可以随意浏览文件，而 more 仅能向前移动，却不能向后移动，而且 less 在查看之前不会加载整个文件

```shell
b  向后翻一页
d  向后翻半页
```

###more

功能类似于 cat, more 会以一页一页的显示方便使用者逐页阅读，而最基本的指令就是按空白键（space）就往下一页显示，按 b 键就会往回（back）一页显示



### grep

```shell
#查找指定进程
ps -ef | grep svn
#查找指定进程个数
ps -ef | grep svn -c
```

###df

显示磁盘空间使用情况

```shell
#以易读方式列出所有文件系统及其类型
df -haT
```

### du 命令

du 命令也是查看使用空间的，但是与 df 命令不同的是 Linux du 命令是对文件和目录磁盘使用的空间的查看

### ps 

ps(process status)，用来查看当前运行的进程状态，一次性查看，如果需要动态连续结果使用 top

```sh
#与grep联用查找某进程
ps -aux | grep apache
```



###netstat

用于显示网络状态。

### top 

显示当前系统正在执行的进程的相关信息，包括进程 ID、内存占用率、CPU 占用率等



#tomcat

##目录说明

bin：存放tomcat的启动、停止等批处理脚本

conf：tomcat相关的配置文件

- ​	catalina.policy：Tomcat 运行的安全策略配置
- ​	catalina.properties：Tomcat 的环境变量配置 
- ​	**context.xml**：用于定义所有web应用均需加载的Context配置，如果web应用指定了自己的context.xml，该文件将被覆盖 
- ​	logging.properties：Tomcat 的日志配置文件， 可以通过该文件修改Tomcat 的日志级别及日志路径等 
- ​	**server.xml**：Tomcat 服务器的核心配置文件 
- ​	tomcat-users.xml：定义Tomcat默认的用户及角色映射信息配置 
- ​	**web.xml**：Tomcat 中所有应用默认的部署描述文件， 主要定义了基础Servlet和MIME映射。 

lib：Tomcat 服务器的依赖包 

webapps：Tomcat 默认的Web应用部署目录 

work：Web 应用JSP代码生成和编译的临时目录 



##整体架构

1、Http服务器：**处理socket连接**，负责网络字节流Request和Response对象的转化

2、Servlet容器：**加载和管理Servlet**，以及具体处理request请求

流程：当客户请求某个资源时，HTTP服务器会用一个ServletRequest对象把客户的请求信息封 装起来，然后调用Servlet容器的service方法，Servlet容器拿到请求后，根据请求的URL 和Servlet的映射关系，找到相应的Servlet，如果Servlet还没有被加载，就用反射机制创建这个Servlet，并调用Servlet的init方法来完成初始化，接着调用Servlet的service方法 来处理请求，把ServletResponse对象返回给HTTP服务器，HTTP服务器会把响应发送给 客户端。

##Coyote连接器

coyote是tomcat的Connector框架的名字，简单说就是coyote来处理底层的socket，并将http请求、响应等字节流层面的东西，包装成Request和Response两个类（这两个类是tomcat定义的，而非servlet中的ServletRequest和ServletResponse），供容器使用；同时，为了能让我们编写的servlet能够得到ServletRequest，tomcat使用了facade模式，将比较底层、低级的Request包装成为ServletRequest（这一过程通常发生在Wrapper容器一级

###io模型协议

bio

nio

nio2.0

arp

###应用层协议

http/1.1

http/2

ajp

###连接器组件

![Coyote组件](images/Coyote组件.png)

####EndPoint

1、Coyote 通信端点，即通信监听的接口，是具体Socket接收和发送处理器，是对传输层的抽象，因此EndPoint用来实现TCP/IP协议的2、Tomcat 并没有EndPoint 接口，而是提供了一个抽象类AbstractEndpoint ， 里面定 义了两个内部类：Acceptor和SocketProcessor。Acceptor用于监听Socket连接请求。 SocketProcessor用于处理接收到的Socket请求，它实现Runnable接口，在Run方法里 调用协议处理组件Processor进行处理。为了提高处理能力，SocketProcessor被提交到 线程池来执行。而这个线程池叫作执行器（Executor)



####Processor

Coyote 协议处理接口 ，如果说EndPoint是用来实现TCP/IP协议的，那么Processor用来实现HTTP协议，Processor接收来自EndPoint的Socket，读取字节流解析成Tomcat Request和Response对象，并通过Adapter将其提交到容器处理，Processor是对应用层协议的抽象。

####ProtocalHandler

Coyote 协议接口， 通过Endpoint 和 Processor ， 实现针对具体协议的处理能力。Tomcat 按照协议和I/O 提供了6个实现类 ： AjpNioProtocol ， AjpAprProtocol， AjpNio2Protocol ， Http11NioProtocol ，Http11Nio2Protocol ， Http11AprProtocol。我们在配置tomcat/conf/server.xml 时 ， 至少要指定具体的 ProtocolHandler , 当然也可以指定协议名称 ， 如 ： HTTP/1.1 ，如果安装了APR，那么 将使用Http11AprProtocol ， 否则使用 Http11NioProtocol 。

#### Adaptor 适配器

由于协议不同，客户端发过来的请求信息也不尽相同，Tomcat定义了自己的Request类 来“存放”这些请求信息。ProtocolHandler接口负责解析请求并生成Tomcat Request类。 但是这个Request对象不是标准的ServletRequest，也就意味着，不能用Tomcat Request作为参数来调用容器。Tomcat设计者的解决方案是引入CoyoteAdapter，这是适配器模式的经典运用，连接器调用CoyoteAdapter的Sevice方法，传入的是Tomcat Request对象，CoyoteAdapter负责将Tomcat Request转成ServletRequest，再调用容器的Service方法。 



## Catalina容器

Catalina 是Servlet 容器实现，包含了之前讲到的所有的容器组件，以及后续章节涉及到的安全、会话、集群、管理等Servlet 容器架构的各个方面。它通过松耦合的方式集成 Coyote，以完成按照请求协议进行数据读写。同时，它还包括我们的启动入口、Shell程序等。

![tomcat模块分层](images/tomcat模块分层.png)



![Catalina结构](images/Catalina结构.png)

Catalina负责管理Server，而Server表示着整个服务器。Server下面有多个 服务Service，**每个Service都包含着一个或多个连接器组件Connector（Coyote 实现）和一个容器组件Container（一个Engine）**。在Tomcat 启动的时候， 会初始化一个Catalina的实例。

组件说明：

Catalina：负责解析Tomcat的配置文件 , 以此来创建服务器Server组件，并根据命令来对其进行管理 

Server：服务器表示整个Catalina Servlet容器以及其它组件，负责组装并启动Servlet引擎,Tomcat连接器。Server通过实现Lifecycle接口，提供了一种优雅的启动和关闭整个系统的方式 

Service：一个Server包含多个Service。**每个Service都包含着一个或多个连接器组件Connector（Coyote 实现）和一个容器组件Container（一个Engine）**

Connector：连接器，处理与客户端的通信，它负责接收客户请求，然后转给相关的容器处理，最后向客户返回响应结果 

Container（狭义上的catalina）：容器，负责处理用户的servlet请求，并返回对象给web用户的模块 





####Container结构

![tomcat结构](images/tomcat结构.png)

Engine：表示整个Catalina的Servlet引擎，用来管理多个虚拟站点，**一个Service最多只能有一个Engine**，但是一个引擎可包含多个Host 

Host：代表一个虚拟主机，每个虚拟主机和某个网络域名相匹配。每个虚拟主机下都可以部署一个或者多个WebApp，每个WebApp对应于一个Context，有一个ContextPath。当Host获得一个请求时，将把该请求匹配到某个Context上，然后把该请求交给该Context来处理。匹配的方法是“最长匹配”，所以一个path==“”的Context将成为该Host的默认Context，所有无法和其他Context的路径名匹配的请求都将最终和该默认Context匹配。

Context：一个Context对应于一个Web Application（Web应用），一个Web应用有一个或多个Wrapper组成，Context在创建的时候将根据配置文件$CATALINA_HOME/conf/web.xml和$WEBAPP_HOME/WEB-INF/web.xml载入Servlet类。如果找到，则执行该类，获得请求的回应，并返回。

Wrapper：表示一个Servlet，Wrapper 作为容器中的最底层，不能包含子容器



##tomcat启动流程



##请求处理流程

![tomcat请求处理流程](images/tomcat请求处理流程.png)



1）Connector组件Endpoint中的Acceptor监听客户端套接字连接并接收Socket。 

2) 将连接交给线程池Executor处理，开始执行请求响应任务。 

3) Processor组件读取消息报文，解析请求行、请求体、请求头，封装成Request对象。 

4) Mapper组件根据请求行的URL值和请求头的Host值匹配由哪个Host容器、Context容器、Wrapper容器处理请求。

5) CoyoteAdaptor组件负责将Connector组件和Engine容器关联起来，把生成的Request对象和响应对象Response传递到Engine容器中，调用 Pipeline。 

6) Engine容器的管道开始处理，管道中包含若干个Valve、每个Valve负责部分处理逻辑。执行完Valve后会执行基础的 Valve--StandardEngineValve，负责调用Host容器的 Pipeline。 

7) Host容器的管道开始处理，流程类似，最后执行 Context容器的Pipeline。 

8) Context容器的管道开始处理，流程类似，最后执行 Wrapper容器的Pipeline。 

9) Wrapper容器的管道开始处理，流程类似，最后执行 Wrapper容器对应的Servlet对象的处理方法。





#zookeeper

zookeeper保证CP（强一致性、分区容错性）；与之对应，Eureka保证AP（可用性，分区容错性）；

##zk实例角色：

**Leader**：事务请求的唯一调度和处理者，并保证事务请求的顺序性（事务指能够改变Zookeeper服务状态的操作，一般包括数据节点的创建删除与内容更新、客户端会话创建与失效。每一个事务有全局唯一的ZXID）；集群内部各服务器的调度

**Follower**:处理客户端非事务请求、转发事务请求给Leader、参与事务请求Proposal投票、参与Leader选举投票

**Observer**：只处理非事务服务，转发事务请求给Leader，不参与任何形式的投票，不参与两阶段写提交

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

　　对于Server1而言，它的投票是(1, 0)，接收Server2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时Server2的myid最大，于是更新自己的投票为(2, 0)，然后重新投票，对于Server2而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。

　　**(4) 统计投票**。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于Server1、Server2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader。

　　**(5) 改变服务器状态**。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。



###服务器运行时期的leader选举

在Zookeeper运行期间，Leader与非Leader服务器各司其职，即便当有非Leader服务器宕机或新加入，此时也不会影响Leader，但是一旦Leader服务器挂了，那么整个集群将暂停对外服务，进入新一轮Leader选举，其过程和启动时期的Leader选举过程基本一致。假设正在运行的有Server1、Server2、Server3三台服务器，当前Leader是Server2，若某一时刻Leader挂了，此时便开始Leader选举。选举过程如下

　　(1) **变更状态**。Leader挂后，余下的非Observer服务器都会讲自己的服务器状态变更为LOOKING，然后开始进入Leader选举过程。

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
- 当 leader 接收到过半的 ACKs 后，leader 就向所有 follower 发送 COMMIT 命令（异步），同时会在本地执行该消息。
- 当 follower 收到消息的 COMMIT 命令时，就会执行该消息（内存中的Datatree）



##zk保证数据一致性





##zk数据同步







##zk Server工作状态

**Looking**：寻找 Leader 状态。当服务器处于该状态时，它会认为当前集群中没有 Leader，因此需要进入 Leader 选举状态

**Following**：跟随者状态，表明当前服务器角色是follower

**Leading**：领导者状态，表明当前服务器角色是leader

**Observing**：观察者状态，表明当前服务器角色是Observer







# Springboot

## springboot配置文件加载顺序

如果工程中同时存在application.properties文件和 application.yml文件，**properties文件会先加载，而后加载的yml文件会覆盖properties文件**



–file:./config/                 当前项目目录下的一个/config子目录

–file:./                              当前项目目录 

–classpath:/config/         项目的resources即一个classpath下的/config包

–classpath:/                      项目的resources即classpath根路径（root）



## springboot自动配置原理















# Mybatis



## 执行流程

1、解析全局配置文件

2、解析mapper映射文件

3、解析sql，将#{}替换为?，${}暂不处理（执行时才做替换），将sql的所有信息封装为MappedStatement对象

4、构建SqlSessionFactory

5、使用SqlSessionFactory创建SqlSession，每个SqlSession分配一个Executor

6、使用SqlSession.getMapper(class)获取接口的代理对象

7、调用接口的方法（目标方法），实际被MapperProxy.invoke()拦截到

8、方法参数转换为sql参数，获取MappedStatement，${} 直接替换成参数值，二级缓存，一级缓存，去数据库查，创建StatementHandler（完成StatementHandler、ParameterHandler、ResultSetHandler的创建和插件代理），为？赋值，执行查询

9、处理结果集







## #{}和${}的区别

1、#{}在mybatis启动时，就编译成了 ? ，${}是在执行时，才会做拼接

2、#{}是占位符，预编译处理（PreparedStatmentHanlder），可以防止sql注入；${}是拼接符，字符串替换，没有预编译处理（StatementHandler），不能防止sql注入

3、#{} 的变量替换是在DBMS 中；${} 的变量替换是在 DBMS 外





##mybatis一对多









##mepper手动传参

方法一：顺序传参法(**不推荐**)

```java
public User selectUser(String name, int deptId);

<select id="selectUser" resultMap="UserResultMap">
    select * from user
    where user_name = #{0} and dept_id = #{1}
</select>
```

\#{}里面的数字代表传入参数的顺序。



方法二：@Param注解传参法

方法三：Map传参法

方法四：javabean传参法



## 模糊查询like语句该怎么写

（1）’%${question}%’ 可能引起SQL注入，不推荐

（2）"%"#{question}"%" 注意：因为#{…}解析成sql语句时候，会在变量外侧自动加单引号’ '，所以这里 % 需要使用双引号" "，不能使用单引号 ’ '，不然会查不到任何结果。

**（3）CONCAT(’%’,#{question},’%’) 使用CONCAT()函数，推荐**

（4）使用bind标签

```xml

<select id="listUserLikeUsername" resultType="com.jourwon.pojo.User">
　　<bind name="pattern" value="'%' + username + '%'" />
　　select id,sex,age,username,password from person where username LIKE #{pattern}
</select>
```

##结果集手动映射

第1种： 通过在查询的SQL语句中定义字段名的别名，让字段名的别名和实体类的属性名一致。

```xml
<select id="getOrder" parameterType="int" resultType="com.jourwon.pojo.Order">
       select order_id id, order_no orderno ,order_price price form orders where order_id=#{id};
</select>
```

第2种： 通过`<resultMap>`来映射字段名和实体类属性名的一一对应的关系。

```xml
<select id="getOrder" parameterType="int" resultMap="orderResultMap">
	select * from orders where order_id=#{id}
</select>

<resultMap type="com.jourwon.pojo.Order" id="orderResultMap">
    <!–用id属性来映射主键字段–>
    <id property="id" column="order_id">

    <!–用result属性来映射非主键字段，property为实体类属性名，column为数据库表中的属性–>
    <result property ="orderno" column ="order_no"/>
    <result property="price" column="order_price" />
</reslutMap>
```



##动态sql

其执行原理为，使用OGNL从sql参数对象中计算表达式的值，根据表达式的值动态拼接sql，以此来完成动态sql的功能。





##一级缓存

一级缓存是SqlSession级别的缓存。在操作数据库时需要构造 sqlSession对象，在对象中有一个(内存区域)数据结构（HashMap）用于存储缓存数据。不同的sqlSession之间的缓存数据区域（HashMap）是互相不影响的。

一级缓存的作用域是同一个SqlSession，在同一个sqlSession中两次执行相同的sql语句，第一次执行完毕会将数据库中查询的数据写到缓存（内存），第二次会从缓存中获取数据将不再从数据库查询，从而提高查询效率。当一个sqlSession结束后该sqlSession中的一级缓存也就不存在了。Mybatis默认开启一级缓存。

**sqlSession去执行commit操作（执行插入、更新、删除），清空SqlSession中的一级缓存**

##二级缓存

二级缓存是mapper级别的缓存，多个SqlSession去操作同一个Mapper的sql语句，多个SqlSession去操作数据库得到数据会存在二级缓存区域，多个SqlSession可以共用二级缓存，二级缓存是跨SqlSession的。

  二级缓存是多个SqlSession共享的，其作用域是mapper的同一个namespace，不同的sqlSession两次执行相同namespace下的sql语句且向sql中传递参数也相同即最终执行相同的sql语句，第一次执行完毕会将数据库中查询的数据写到缓存（内存），第二次会从缓存中获取数据将不再从数据库查询，从而提高查询效率。Mybatis默认没有开启二级缓存需要在setting全局参数中配置开启二级缓存。cacheEnable=true

如果缓存中有数据就不用从数据库中获取，大大提高系统性能。







# netty







# springcloud









# seata







# nacos









# sentinel







# dubbo

















