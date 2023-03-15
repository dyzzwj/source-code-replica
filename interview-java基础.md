# java基础

//Java源代码---->编译器---->jvm可执行的Java字节码(即虚拟指令)---->jvm---->jvm中解释器----->机器可执行的二进制机器码---->程序运行。 					



## 8种基本数据类型

![基本数据类型](\images\基本数据类型.png)

##java基本数据类型转换

![基本数据类型转换](\images\基本数据类型转换.png)



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



String:  jdk8及之前 final char value[]    jdk9及之后 final bytevalue[]

StringBuilder：char[] value;

StringBuffer：char[] value;



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

![访问修饰符](\images\访问修饰符.png)

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

在类初次被加载的时候，会按照**static块的顺序**来执行每个static块，并且只会执行一次。

静态成员只能访问静态成员，不能访问非静态成员



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

