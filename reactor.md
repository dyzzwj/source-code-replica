# 1、webflux

## 1.1函数式接口



Consumer：一个输入 无输出

Function：一个输入 一个输出

Supplier：一个输入 无输出

Predicate：一个输入 一个boolean输出



map和flatmap区别

<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);

<R> Stream<R> map(Function<? super T, ? extends R> mapper);



## 1.2Reactor

**Flux: 代表一个包含0..N个元素的响应式序列**

**Mono:代表一个包含零个或一个（0..1）元素的结果**

订阅前什么都不会发生

Flux.just(1, 2, 3, 4, 5, 6)仅仅声明了这个数据流，此时数据元素并未发出，只有subscribe()方法调用的时候才会触发数据流







2、