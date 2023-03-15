解析->路由->改写->执行->归并
=======




# 执行流程

sql解析->sql路由->sql改写->sql执行->结果归并







# 解析引擎

## SQL种类

在看代码前，首先我们看下SQL的分类，因为ShardingSphere代码中很多地方都会根据这个分类来判断SQL的类型：

- **DML**（Data Manipulation Language），数据操作类语句，包括select、insert、update、delete、selec for update、call
- **DAL**（Data Administration Language，数据管理类语句，包括use、show databases、show tables、show colums、show createtable
- **DDL**（Data Definition Language），数据定义类语句，包括create table、alter table、drop table、truncate table
- **TCL**（Transaction Control Language），事务控制类语句，包括set transaction、set autocimmit、begin、commit、rollback、saveponit
- **DQL**（Data Query Language），数据查询类语句，在ShardingSphere的antlr4文件中select属于DML，但部分类中如ShardingDQLResultMerger，将select又称为DQL。
- **RL**（Replication Language），复制类数据，包括change master to、start slave、stop slave。









## 路由引擎

**路由引擎的职责定位就是计算SQL应该在哪个数据库、哪个表上执行**，前者结果会传给后续执行引擎，然后根据其数据库标识获取对应的数据库连接；后者结果则会传给改写引擎在SQL执行前进行表名的改写，即替换为正确的物理表名









# 执行引擎



1. 是SQLExecutePrepareTemplate，2. 是SQLExecuteTemplate，3. ExecutorEngine。
    SQLExecutePrepareTemplate类负责生成执行分组信息，输入为 Collection<ExecutionUnit> ，输出为Collection<InputGroup<StatementExecuteUnit>>；SQLExecuteTemplate类负责执行具体的SQL操作，输入为Collection<InputGroup<StatementExecuteUnit>>与SQLExecuteCallback，这个类目前并没有自身逻辑，它就是直接调用了ExecutorEngine类完成SQL执行；ExecutorEngine则真正负责完成SQL的串行和并行执行。





归并引擎


















