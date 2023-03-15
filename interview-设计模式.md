





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



