seata源码



GlobalTransactionScanner  extends AbstractAutoProxyCreator implements InitializingBean



GlobalTransactionScanner实现了InitializingBean接口的afterPropertiesSet()方法



GlobalTransactionScanner覆盖了AbstractAutoProxyCreator的wrapIfNecessary()方法



seata对Connection对象做了包装 持久层获取到的连接实际上是ConnectionProxy
     
在ConnectionProxy的父类AbstractConnectionProxyseata中 
 对PreparedStatement做了包装 持久层获取到的实际上是PreparedStatementProxy





BranchRollbackRequest



insert  只有后置镜像 



delete只有前置镜像



Update 前置镜像 + 后置镜像







每个数据库操作都会对应一个分支事务(包括本地的和远程的)  每个分支事务都会对应undolog表中的一条数据



远程调用xid问题 （以使用feign进行远程调用为例）

SeataRestTemplateInterceptor（服务消费者）：在这里把跟当前线程绑定的xid（放到ThreadLocal）放到request的header中



SeataHandlerInterceptor（服务提供者）：在这里从request的header中取出xid 与当前线程绑定



**分布式事务和分布式事务的隔离性**

分布式事务之间的脏读 其他分布式事务查询到了分布式事务还未提交的但分支事务已提交的数据

seata解决 ：注册分支事务的时候 在分支事务提交之前检查全局锁冲突异常





**分布式事务和本地事务的隔离性**

脏写：本地事务修改了分布式事务还未提交的但分支事务已提交的数据    在本地事务的方法上加@GlobalLock(性能更好)注解或@GlobalTransaction



 脏读：本地事务查询到了分布式事务还未提交的但分支事务已提交的数据  本地查询语句加for update 并且加@GlobalTransaction注解



select for update ==> SelectForUpdateExecutor   判断是否加了@GlobalTransaction

select ==>  PlainExecutor 直接执行sql



