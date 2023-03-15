mybatis源码



四大对象

- ParameterHandler：处理SQL的参数**对象**
- ResultSetHandler：处理SQL的返回结果集
- StatementHandler：数据库的处理**对象**，用于执行SQL语句
- Executor：**MyBatis**的执行器，用于执行增删改查操作







mybatis



一级缓存不能关闭，但是可以更改其作用域，默认作用域为单个SqlSession级别

二级缓存默认关闭，默认作用域是namespace

每个缓存的单位是namespace(mapper)





二级缓存实现(**装饰者设计模式**)：

SynchorizedCache:同步cache，直接使用synchorized修饰方法

LoggingCache:日志功能 用于记录缓存的命中率 如果开启了debug模式，则会输出命中率日志

SerializedCache:序列化功能，将值序列化后存到缓存中 该功能用于缓存返回一份实例的copy，用于保证线程安全

LruCache:采用了LRU算法的cache实现，移除最近最少使用的key/value  回收策略：LRU（默认）、FIFO、WEAK(软引用)、WEAK(弱引用)

PerpetualCache:作为最基础的缓存类 底层实现比较简单 直接使用了HashMap

二级缓存配置：

```
<cache eviction="设置回收策略" flushInterval="缓存刷新间隔（如果不配置，就不会清空）"></cache>
```













CacheBuilder.build()：构造器模式

```java
public Cache build() {
  /**
   * 添加LruCache
   */
  setDefaultImplementations();
  Cache cache = newBaseCacheInstance(implementation, id);
  setCacheProperties(cache);
  // issue #352, do not apply decorators to custom caches
  if (PerpetualCache.class.equals(cache.getClass())) {
    for (Class<? extends Cache> decorator : decorators) {
      cache = newCacheDecoratorInstance(decorator, cache);
      setCacheProperties(cache);
    }
    cache = setStandardDecorators(cache);
  } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
    cache = new LoggingCache(cache);
  }
  return cache;
}
```



CacheBuilder.setStandardDecorators()    装饰者设计模式

```java
private Cache setStandardDecorators(Cache cache) {
    //cache：默认是LruCache
  try {
    MetaObject metaCache = SystemMetaObject.forObject(cache);
    if (size != null && metaCache.hasSetter("size")) {
      metaCache.setValue("size", size);
    }
    if (clearInterval != null) {
      cache = new ScheduledCache(cache);
      ((ScheduledCache) cache).setClearInterval(clearInterval);
    }
    if (readWrite) {
      cache = new SerializedCache(cache);
    }
    cache = new LoggingCache(cache);
    cache = new SynchronizedCache(cache);
    if (blocking) {
      cache = new BlockingCache(cache);
    }
    return cache;
  } catch (Exception e) {
    throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
  }
}
```







二级缓存对事物的处理

开启事务 -> 添加数据 -> 查询数据（放到缓存） ->提交事务（失败）

如果事务提交失败 那么缓存中的数据就是脏数据‘



解决：

TransactionCacheManager：

TransactionCache：根据数据库事务的执行结果来执行提交或回滚

commit成功的时候 如果事务提交成功 清空真实缓存 把待提交空间保存到真实缓存中

rollback的时候，删除真实缓存区中未命中的缓存









日志框架

切换日志

整合日志 



具体的日志实现

log4j   Logger logger = Logger.getLogger()

jul        Logger logger = Logger.getLogger()



jcl：如果有log4j，就用log4j，如果没有log4j，就是要jul     选择器根据当前项目环境 动态选择日志实现   解决了日志切换

Log log = LogFactory.getLogger()





sl4fj

只引入slf4j的依赖，默认是打印不了日志的

只是一个日志门面，如果要打印日志的话得告诉它具体的日志实现