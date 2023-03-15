# 一、Join查询原理

查询原理：MySQL内部采用了一种叫做 nested loop join（嵌套循环连接）的算法。Nested Loop Join 实际上就是通过驱动表的结果集作为循环基础数据，然后一条一条的通过该结果集中的数据作为过滤条件到下一个表中查询数据，然后合并结果。如果还有第三个参与 Join，则再通过前两个表的 Join 结果集作为循环基础数据，再一次通过循环查询条件到第三个表中查询数据，如此往复，基本上MySQL采用的是最容易理解的算法来实现join。所以驱动表的选择非常重要，驱动表的数据小可以显著降低扫描的行数。

一般情况下参与联合查询的两张表都会一大一小，如果是join，在没有其他过滤条件的情况下MySQL会自动选择**小表作为驱动表**。简单来说，驱动表就是主表，left join 中的左表就是驱动表，right join 中的右表是驱动表。

- left join，保留左表所有数据，左边没有数据设置为 null。
- right join，保留右表所有数据，游标没有数据设置为 null。
- inner join，取左右表数据的交集。





# 二.Nested-Loop Join

在Mysql中，使用Nested-Loop Join的算法思想去优化join，Nested-Loop Join翻译成中文则是“嵌套循环连接”。

举个例子：`select * from t1 inner join t2 on t1.id=t2.tid`
（1）t1称为外层表，也可称为驱动表。
（2）t2称为内层表，也可称为被驱动表。

//伪代码表示：

```java
List<Row> result = new ArrayList<>();
for(Row r1 in List<Row> t1){
	for(Row r2 in List<Row> t2){
		if(r1.id = r2.tid){
			result.add(r1.join(r2));
		}
	}
}
```


mysql只支持一种join算法：Nested-Loop Join（嵌套循环连接），但Nested-Loop Join有三种变种：

Simple Nested-Loop Join：SNLJ，简单嵌套循环连接
Index Nested-Loop Join：INLJ，索引嵌套循环连接
Block Nested-Loop Join：BNLJ，缓存块嵌套循环连接
在选择Join算法时，会有优先级，理论上会优先判断能否使用INLJ、BNLJ：

Index Nested-LoopJoin > Block Nested-Loop Join > Simple Nested-Loop Join


# 三. Simple Nested-Loop Join

如下图，r为驱动表，s为匹配表，可以看到从r中分别取出r1、r2、…、rn去匹配s表的左右列，然后再合并数据，对s表进行了rn次访问，对数据库开销大。如果table1有1万条数据，table2有1万条数据，那么数据比较的次数=1万 * 1万 =1亿次，这种查询效率会非常慢。

![Simple Nested-Loop Join](\images\Simple Nested-Loop Join.png)

所以Mysql继续优化，然后衍生出Index Nested-LoopJoin、Block Nested-Loop Join两种NLJ算法。在执行join查询时mysql会根据情况选择两种之一进行join查询。



# 四.Index Nested-LoopJoin（减少内层表数据的匹配次数）

索引嵌套循环连接是基于索引进行连接的算法，索引是基于内层表的，通过外层表匹配条件直接与**内层表索引**进行匹配，避免和内层表的每条记录进行比较， 从而利用索引的查询减少了对内层表的匹配次数，优势极大的提升了 join的性能：

原来的匹配次数 = 外层表行数 * 内层表行数
优化后的匹配次数= 外层表的行数 * 内层表索引的高度

使用场景：只有内层表join的列有索引时，才能用到Index Nested-LoopJoin进行连接。

由于用到索引，如果索引是辅助索引而且返回的数据还包括内层表的其他数据，则会回内层表查询数据，多了一些IO操作。

这个要求非驱动表（匹配表s）上有索引，可以通过索引来减少比较，加速查询。

在查询时，驱动表（r）会根据关联字段的索引进行查找，当在索引上找到符合的值，再回表进行查询，也就是只有当匹配到索引以后才会进行回表查询。

如果非驱动表（s）的关联健是主键的话，性能会非常高，如果不是主键，要进行多次回表查询，先关联索引，然后根据二级索引的主键ID进行回表操作，性能上比索引是主键要慢

<img src="images/Index Nested-LoopJoin.png" alt="Index Nested-LoopJoin" style="zoom:60%;" />









# 五.Block Nested-Loop Join（减少内层表数据的循环次数）

缓存块嵌套循环连接通过一次性缓存多条数据，把参与查询的列缓存到Join Buffer 里，然后拿join buffer里的数据批量与内层表的数据进行匹配，从而减少了内层循环的次数（遍历一次内层表就可以批量匹配一次Join Buffer里面的外层表数据）。

当不使用Index Nested-Loop Join的时候(内层表查询不适用索引)，默认使用Block Nested-Loop Join。

什么是Join Buffer？
（1）Join Buffer会缓存所有参与查询的列而不是只有Join的列。
（2）可以通过调整join_buffer_size缓存大小
（3）join_buffer_size的默认值是256K，join_buffer_size的最大值在MySQL 5.1.22版本前是4G-1，而之后的版本才能在64位操作系统下申请大于4G的Join Buffer空间。
（4）使用Block Nested-Loop Join算法需要开启优化器管理配置的optimizer_switch的设置block_nested_loop为on，默认为开启。

<img src="images/Block Nested-Loop Join.png" alt="Block Nested-Loop Join" style="zoom:67%;" />









# 六.如何优化Join速度

用小结果集驱动大结果集，减少外层循环的数据量：
如果小结果集和大结果集连接的列都是索引列，mysql在内连接时也会选择用小结果集驱动大结果集，因为索引查询的成本是比较固定的，这时候外层的循环越少，join的速度便越快。
为匹配的条件增加索引：争取使用INLJ，减少内层表的循环次数
增大join buffer size的大小：当使用BNLJ时，一次缓存的数据越多，那么外层表循环的次数就越少
减少不必要的字段查询：
（1）当用到BNLJ时，字段越少，join buffer 所缓存的数据就越多，外层表的循环次数就越少；
（2）当用到INLJ时，如果可以不回表查询，即利用到覆盖索引，则可能可以提示速度。



# 七、On VS Where

前提说明：数据库在通过连接两张或多张表来返回记录时，都会生成一张中间的临时表，然后再将这张临时表返回给用户。

- on 条件是在生成临时表时使用的条件，它不管 on 中的条件是否为真，都会返回左边表中的记录。
- where 条件是在临时表生成好后，再对临时表进行过滤的条件。这时已经没有 left join 的含义（必须返回左边表的记录）了，条件不为真的就全部过滤掉。

即

- 如果条件中同时有on和where 条件：
  - SQL的执行实际是两步
    - 第一步：根据on条件得到一个临时表
    - 第二步：根据where 条件对上一步的临时表进行过滤，得到最终返回结果。
- 如果条件中只有on：
  - 那么得到的临时表就是最终返回结果









