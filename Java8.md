																						Java8

一、内部类

![内部类总结](images/内部类总结.jpg)



局部内部类和匿名内部类只能访问局部final变量？

**内部类对象的生命周期可能会超过局部变量的生命周期。**



二、lamabda表达式

1、四大函数式接口：Function、Consumer、Producer、Supplier

只包含一个抽象方法的接口，称为**函数式接口**。

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}
```

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```



1、lambda表达式语法

```java
//1、无参、无返回值

Runnable runnable = () -> {
    System.out.println("aaaaa");
};

//2、Lambda需要一个参数，但没有返回值
Consumer<String> consumer = (String x) -> {
    System.out.println(x);
};

//3、数据类型可以省略，因为可由编译期进行类型推断
Consumer<String> consumer1 = x -> {
    System.out.println(x);
};

//4、Lambda 若只需要一个参数时，参数的小括号可以省略
Consumer<String> consumer2 = x -> {
    System.out.println(x);
};

//5、Lambda 需要两个或以上的参数，多条执行语句，并且可以有返回值
Comparator<Integer> comparator = (x,y) -> {
    System.out.println(x);
    System.out.println(y);
    return Integer.compare(x,y);
};

//6、当 Lambda 体只有一条语句时，return 与大括号若有，都可以省略
Comparator<Integer> comparator1 = (x,y) -> Integer.compare(x,y);
```



3、方法引用

方法引用：若lambda体中的内容有方法已经实现了，我们可以使用方法引用。使用**::**操作符将**方法名**和**对象或类**的名字分隔开。

三种形式：

对象::实例方法名

类名::静态方法名

类::实例方法名

注意：

①lambda体中调用方法的参数列表与返回值类型，要与函数式接口中抽象方法的参数列表和返回值类型保持一致

**②当函数式接口方法的第一个参数是需要引用方法的调用者，并且第二个参数是需要引用方法的参数(或无参数)时：ClassName::methodName**

4、构造器引用

调用的构造器参数列表和函数式接口抽象方法的参数列表保持一致

类名::new



三、Stream API

1、Stream关注的数据的运算，面向CPU

集合关注数据的存储。面向内存



2、Stream本身不会存储数据；Stream不会改变源对象，而是会返回一个持有新结果的Stream



3、Stream执行流程：Stream实例化->一系列中间操作->终止操作

注：只有执行了终止操作，才会执行中间操作（延迟执行）



4、创建Stream的4种方式

集合：集合.stream()

数组：Arrays.stream(array)

Stream：Stream.of()

创建无限流：Stream.iterator();Stream.generate()



**多个中间操作可以连接起来形成一个流水线，除非流水线上触发终止操作，否则中间操作不会执行任何的处理！而在终止操作时一次性全部处理，称为惰性求值**

5、中间操作

①筛选与切片

filter(Predict predict)：接收 Lambda ， 从流中排除某些元素

distinct()：通过流所生成元素的 hashCode() 和 equals() 去除重复元素

limit(n)：截断流，使其元素不超过给定数量

skip(n)：跳过元素，返回一个扔掉了前 n 个元素的流。若流中元素不足 n 个，则返回一个空流。与 limit(n) 互补



②映射

map(Function func)：接收一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素。

flatMap(Function func)：接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流



③排序

sorted()：自然排序

sorted(**Comparator**)：定制排序



6、终止操作

①匹配与查找
**allMatch(Predicate p)** ：检查是否匹配所有元素

**anyMatch**(**Predicate p**) ：检查是否至少匹配一个元素

**noneMatch(Predicate p)** ：检查是否没有匹配所有元素

**findFirst()** ：返回第一个元素

**findAny()** ：返回当前流中的任意元素

**count()** ：返回流中元素总数

**max(Comparator c)** ：返回流中最大值

**min(Comparator c)** ：返回流中最小值

**forEach(Consumer c)**：内部迭代(使用 Collection 接口需要用户去做迭代，称为外部迭代。相反，Stream API 使用内部迭代——它帮你把迭代做了)

②规约

**reduce(T iden, BinaryOperator b)** ：可以将流中元素反复结合起来，得到一个值。返回 T

**reduce(BinaryOperator b)** ：可以将流中元素反复结合起来，得到一个值。返回 Optional<T>

③收集

collect(Collector c)：将流转换为其他形式。接收一个 Collector接口的实现，用于给Stream中元素做汇总的方法



