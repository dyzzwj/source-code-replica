​																			try、catch、finally

#一、try、finally、catch执行顺序：

**1、不管有没有出现异常，finally块中代码都会执行；**

**2、当try和catch中有return时，finally仍然会执行；**

**3、finally是在return语句执行之后，返回之前执行的（此时并没有返回运算后的值，而是先把要返回的值保存起来，不管finally中的代码怎么样，返回的值都不会改变，仍然是之前保存的值），所以函数返回值是在finally执行前就已经确定了；**

**4、finally中如果包含return，那么程序将在这里返回，而不是try或catch中的return返回，返回值就不是try或catch中保存的返回值了。**

注意：

- **finally修改的基本类型是不影响返回结果的。（传值的）**
- **修改list ,map,自定义类等引用类型时，是影响返回结果的。（传址的）对象也是传址的**

**但是date类型经过测试是不影响的。有点奇怪。**



finally不执行的情况（极端）：

①.如果在try 或catch语句中执行了System.exit(0)。

②.在执行finally之前jvm崩溃了。

③.try语句中执行死循环。

④.电源断电。



#二、情况举例

##**情况1：** try{} catch(){}finally{} return;
显然程序按顺序执行。



##**情况2:** try{ return; }catch(){} finally{} return;

1. 先执行try块中return 语句（包括return语句中的表达式运算），但不返回；
2. 执行finally语句中全部代码
3. 最后执行try中return 返回

- finally块之后的语句return不执行，因为程序在try中已经return。

  

```java
 public static int demo5() {
        try {
            return printX();
        }
        finally {
            System.out.println("finally trumps return... sort of");
        }
    }
    public static int printX() {

        System.out.println("X");
        return 0;
    }
```

输出：

```java
X
finally trumps return... sort of
0
```



```java
public static int test1(){

    int i = 0;
    try{
        return i++;
    }catch(Exception e){

    } finally {
        i = 9;
        System.out.println("finally..." + i);
    }
    return 20;

}
```

输出：

```java
finally...9
0
```

##**情况3:** try{ } catch(){return;} finally{} return;

有异常：

- 执行catch中return语句，但不返回
- 执行finally语句中全部代码，
- 最后执行catch块中return返回。 finally块后的return语句不再执行。

无异常：执行完try再finally再return…

##**情况4:** try{ return; }catch(){} finally{return;}

- 执行try块return语句（包括return语句中的表达式运算），但不返回；
- 再执行finally块，
- 执行finally块，有return，从这里返回。

无论有无异常，最后都会在finally中的return中返回；

此时finally块的return值，就是代码执行完后的值

##**情况5:** try{} catch(){return;}finally{return;}

- 程序执行catch块中return语句（包括return语句中的表达式运算），但不返回；
- 再执行finally块，
- 执行finally块，有return，从这里返回。

##**情况6:** try{ return;}catch(){return;} finally{return;}
1、程序执行try块中return语句（包括return语句中的表达式运算），但不返回；

- 有异常：
  - 执行catch块中return语句（包括return语句中的表达式运算），但不返回；
  - 再执行finally块
  - 执行finally块，有return，从这里返回。
- 无异常：
- 再执行finally块
- 执行finally块，有return，从这里返回。。

结论：

- 任何执行try 或者catch中的return语句之后，在返回之前，如果finally存在的话，都会先执行finally语句，
- 如果finally中有return语句，那么程序就return了，所以finally中的return是一定会被return的，
- 编译器把finally中的return实现为一个warning。
- 从finally中返回会导致异常丢失

