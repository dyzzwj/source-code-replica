varchar和char区别

- 可变长度 & 固定长度

varchar 仅使用必要的空间(根据实际字符串的长度改变存储空间)。

char值会根据需要采用空格进行剩余空间填充

- 存储方式

VARCHAR需要使用1（小于等于255）或2个额外字节记录字符串的长度

CHAR适合存储很短或长度近似的字符串

- 存储容量

  对于char类型来说，最多只能存放的字符个数为255，和编码无关

  单列字段`情况下，varchar一般最多能存放(65535 - 3)个`字节



## innodb和myisam区别

- 存储格式

innodb两个文件：.frm(表定义)、.ibd

myisam三个文件：.frm（表定义）、.MYD（数据文件）、.MYI(索引文件)

- 事物、外键支持

innodb支持事务、行级锁、外键，myisam都不支持

- 聚簇索引和非聚簇索引

innodb：聚簇索引（将数据行和索引集中存储），按主键大小有序插入，查询结果默认按主键排序

myisam：非聚簇索引（将数据行和索引分开存储，索引结构的叶子节点指向了数据的对应行），按记录插入顺序保存，查询顺序和插入顺序一致

- 表的行数

count(*)：myisam内部维护了一个计数器，可以直接调取

- 全文索引的支持

  innodb默认不支持fulltext数据类型
  
  
  
- 日志文件 

  redo undo



## 取topN

```sql
create table SC(
sid varchar(10) comment "学生ID",
cid varchar(10) comment "课程ID",
score decimal(18,1) comment "课程成绩");

insert into SC values('01' , '01' , 80);
insert into SC values('01' , '02' , 90);
insert into SC values('01' , '03' , 99);
insert into SC values('02' , '01' , 70);
insert into SC values('02' , '02' , 60);
insert into SC values('02' , '03' , 80);
insert into SC values('03' , '01' , 80);
insert into SC values('03' , '02' , 80);
insert into SC values('03' , '03' , 80);
insert into SC values('04' , '01' , 50);
insert into SC values('04' , '02' , 30);
insert into SC values('04' , '03' , 20);
insert into SC values('05' , '01' , 76);
insert into SC values('05' , '02' , 87);
insert into SC values('06' , '01' , 31);
insert into SC values('06' , '03' , 34);
insert into SC values('07' , '02' , 89);
insert into SC values('07' , '03' , 98);
```





求每门课成绩前两名

```sql
SELECT a.CId, a.SId, a.score
FROM SC a
	LEFT JOIN SC b
	ON a.CId = b.CId
		AND a.score < b.score
GROUP BY a.CId, a.SId, a.score
HAVING COUNT(a.score) < 2
ORDER BY a.CId, a.score DESC
```

如何理解：a.score < b.score和 HAVING COUNT(a.score) < 2

前两名：第一名和第二名

对于第一名，比第一名分数高(a.score < b.score)的人数是0 -> COUNT(a.score) = 0

对于第二名。比第二名分数高(a.score < b.score)的人数是1 -> COUNT(a.score) = 1



子查询

```sql
SELECT a.CId, a.SId, a.score
FROM SC a
WHERE (
	SELECT COUNT(score)
	FROM SC
	WHERE CId = a.CId
		AND score < a.score
) < 3
ORDER BY a.CId, a.score DESC;
```







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

4）不能有效区分数据（离散度）的列不适合做索引列

5）尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。

6）定义有外键的数据列一定要建立索引。

7）查询中order by、group by的字段

8）对于定义为text、image和bit的数据类型的列不要建立索引。





## 索引失效情况：

没有遵循最左匹配原则

在索引列上做操作

左模糊

is null、 is not null、!= mysql会根据查询优化器计算出的成本 决定是否使用索引

字符串没加单引号

范围之后的索引失效  between and

使用or连接

order by 、group by后字段排序顺序与索引创建顺序不一致 不能走索引



## 前缀索引

对于一些字段比较长的列，可以使用前缀索引，截取这个列的前部分作为索引

语法：`index(field(10))`，使用字段值的前10个字符建立索引，默认是使用字段的全部内容建立索引。

前提：前缀的标识度高。比如密码就适合建立前缀索引，因为密码几乎各不相同。

实操的难度：在于前缀截取的长度。

我们可以利用`select count(*)/count(distinct left(password,prefixLen));`，通过从调整`prefixLen`的值（从1自增）查看不同前缀长度的一个平均匹配度，接近1时就可以了（表示一个密码的前`prefixLen`个字符几乎能确定唯一一条记录）



## 聚簇索引和非聚簇索引

聚簇索引：将数据行和索引集中存储

非聚簇索引：将数据行和索引分开存储，索引结构的叶子节点指向了数据的对应行



## 事务

### 事务特性原理（ACID）

- 原子性

 undolog，实现原子性的关键，是当事务回滚时能够撤销所有已经成功执行的sql语句。InnoDB实现回滚，靠的是undo log：当事务对数据库进行修改时，InnoDB会生成对应的undo log；如果事务执行失败或调用了rollback，导致事务需要回滚，便可以利用undo log中的信息将数据回滚到修改之前的样子。



- 持久性 

redo log，事务开始之后就产生redo log，redo log的落盘并不是随着事务的提交才写入的，在内存数据页准备修改前将修改信息写入内存中的redo buffer中，然后再对内存数据页中的数据执行修改操作；而且保证在发出事务提交指令时，先向内存中的redo buffer写入日志，写入完成后才执行提交动作；聚集索引、二级索引、undo页面的修改，均需要记录Redo日志

- 隔离性 

​	   (一个事务)写操作对(另一个事务)写操作的影响：锁机制保证隔离性

​	   (一个事务)写操作对(另一个事务)读操作的影响：MVCC保证隔离性 （mvcc通过 undo log 形成一个事务执行过程中的版本链）

- 一致性

一致性指数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态

​    保证原子性、持久性和隔离性，如果这些特性无法保证，事务的一致性也无法保证

​    数据库本身提供保障，例如不允许向整形列插入字符串值、字符串长度不能超过列的限制等

​    应用层面进行保障，例如如果转账操作只扣除转账者的余额，而没有增加接收者的余额，无论数据库实现的多么完美，也无法保证状态的一致



### 脏读  不可重复读 幻读

脏读：读未提交

不可重复读：前后多次读取，数据内容不一致（内容被修改或被删除） -> update/delete

幻读：前后多次读取，数据总量不一样（新增）-> insert 

**幻读在当前读下(mysql在RR级别下基于MVCC解决了快照读的幻读问题)才会出现，幻读仅指新插入的行**



### 事务隔离级别

Read Uncommitted：读未提交 ==>会出现脏读



Read Committed：读已提交==>解决脏读，会出现不可重复读



Repetable Read：（使用mvcc机制） 可重复读==>解决脏读、不可重复读    mysql默认隔离级别   RR在“快照读"的情况下是可以解决“幻读”的问题的。基于MVCC  

Serializable：串行化==>解决脏读、不可重复读、幻读





### MVCC(多版本并发控制)

#### 快照读与当前读

**快照读（SnapShot Read）** 是一种**一致性不加锁的读**

> 这里的**一致性**是指，事务读取到的数据，要么是**事务开始前就已经存在的数据**，要么是**事务自身插入或者修改过的数据**。

**当前读**就是**读取最新数据**，而不是历史版本的数据，并且对读取的记录加锁（也会可能因加锁失败而阻塞）。加锁的 SELECT 、update、delete、insert就属于当前读

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

- **DELETED_BIT** 1byte, 记录被更新或删除，并不代表真的删除，而是删除flag变了



#### undo日志

undo log 存储的是**逻辑格式**的日志，记录变化的过程，根据每行记录进行记录，保存了事务发生之前的上一个版本的数据，可以用于回滚。当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着 undo 链找到满足其可见性的记录。由于查询操作（SELECT）并不会修改任何用户记录，所以在查询操作执行时，并不需要记录相应的undo log。undo log主要分为3种：

- **Insert undo log** ：在insert 操作中产生的undo log，因为insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。
- **Update undo log**：update操作产生的undolog，包含被修改行的主键(以便知道修改了哪些行)、修改了哪些列、这些列在修改前后的值等信息，回滚时便可以使用这些信息将数据还原到update之前的状态。
- Delete undo log
  - 删除操作只是设置一下老记录的DELETED_BIT，并不真正将过时的记录删除。
  - 为了节省磁盘空间，InnoDB有专门的purge线程来清理DELETED_BIT为true的记录。为了不影响MVCC的正常工作，purge线程自己也维护了一个read view（这个read view相当于系统中最老活跃事务的read view）;如果某个记录的DELETED_BIT为true，并且DB_TRX_ID相对于purge线程的read view可见，那么这条记录一定是可以被安全清除的。



对MVCC有帮助的实质是**update undo log** ，undo log实际上就是存在rollback segment中旧记录链，它的执行流程如下：

1. **比如一个事务向person表插入了一条新记录，记录如下，name为Jerry, age为24岁，隐式主键是1，事务ID和回滚指针，我们假设为NULL**

![db-mysql-mvcc-1](images\db-mysql-mvcc-1.png)

2、**现在来了一个事务1对该记录的name做出了修改，改为Tom**

1. 在事务1修改该行(记录)数据时，数据库会先对该行加排他锁
2. 然后把该行数据拷贝到undo log中，作为旧记录，即在undo log中有当前行的拷贝副本
3. 拷贝完毕后，修改该行name为Tom，并且修改隐藏字段的事务ID为当前事务1的ID, 我们默认从1开始，之后递增，回滚指针指向拷贝到undo log的副本记录，既表示我的上一个版本就是它
4. 事务提交后，释放锁

![db-mysql-mvcc-2](/images\db-mysql-mvcc-2.png)



3、**又来了个事务2修改person表的同一个记录，将age修改为30岁**

1. 在事务2修改该行数据时，数据库也先为该行加锁
2. 然后把该行数据拷贝到undo log中，作为旧记录，发现该行记录已经有undo log了，那么最新的旧数据作为链表的表头，插在该行记录的undo log最前面
3. 修改该行age为30岁，并且修改隐藏字段的事务ID为当前事务2的ID, 那就是2，回滚指针指向刚刚拷贝到undo log的副本记录
4. 事务提交，释放锁

![db-mysql-mvcc-3](/images\db-mysql-mvcc-3.png)



从上面，我们就可以看出，不同事务或者相同事务的对同一记录的修改，会导致该记录的undo log成为一条记录版本线性表，既链表，undo log的链首就是最新的旧记录，链尾就是最早的旧记录（当然就像之前说的该undo log的节点可能是会purge线程清除掉，向图中的第一条insert undo log，其实在事务提交之后可能就被删除丢失了，不过这里为了演示，所以还放在这里）



#### Read View

Read View就是事务进行**快照读**操作的时候生产的读视图(Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID(当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以最新的事务，ID值越大)

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



在RR级别下的某个事务对某条记录的第一次快照读会创建一个快照及Read View, 将当前系统活跃的其他事务记录起来，此后在调用快照读的时候，还是使用的是同一个Read View，所以只要当前事务在其他事务提交更新之前使用过快照读，那么之后的快照读使用的都是同一个Read View，所以对之后的修改不可见；

即RR级别下，快照读生成Read View时，Read View会记录此时所有其他活动事务的快照，这些事务的修改对于当前事务都是不可见的。而早于Read View创建的事务所做的修改均是可见

而在RC级别下的，事务中，**每次**快照读都会新生成一个快照和Read View, 这就是我们在RC级别下的事务中可以看到别的事务提交的更新的原因











## 数据库优化

1、建索引、优化sql

2、选择合适的存储引擎

3、分库分表、读写分离

4、表设计

5、mysql服务器的参数调优

### sql优化

1、开启慢查询并设置阈值进行捕获

2、explain + 慢sql分析

3、show profile查询sql的执行细节和生命周期

4、Join小表驱动大表 in后跟小表 exists后跟大表



### order by 使用索引排序

1、order by 使用索引最左前缀列

2、使用where子句与order by 子句条件列组合满足索引最左前缀(where使用索引的最左前缀定义为常量)



### order by优化

1、order by时只select需要的字段

2、尝试提供sort_buffer_size

3、尝试提高max_length_for_sort_buffer





### 大表数据查询，怎么优化

1. 优化shema、sql语句+索引；
2. 加缓存，memcached, redis；
3. 主从复制，读写分离；
4. 垂直拆分，根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统；
5. 水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key, 为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改，sql中尽量带sharding key，将数据定位到限定的表上去查，而不是扫描全部的表；

### 超大分页



limit实质：`limit 10000,10`的语法实际上是mysql查找到前10010条数据,之后丢弃前面的10000行,这个步骤其实是浪费掉的.

#### 用id优化

select * from user limit 1000000,10

优化思路：先找到上次分页的最大ID,然后利用id上的索引来查询,类似于`select * from user where id > 1000000 limit 100`.

这样的效率非常快,因为主键上是有索引的,但是有两个前提条件，**就是ID必须是连续的,并且查询不能有where语句**,因为where语句会造成过滤数据.  

另外两种写法：

子查询的方式 

SELECT * FROM product WHERE ID > =(select id from product limit 866613, 1) limit 20

join的方式（延迟关联）

SELECT * FROM product a JOIN (select id from product limit 866613, 20) b ON a.ID = b.id

```
select * from product limit 866613, 20   37.44秒

# 只查询Id
select id from product limit 866613, 20 0.2秒

# 查询所有字段
SELECT * FROM product WHERE ID > =(select id from product limit 866613, 1) limit 20

# 查询所有字段的另一种写法
SELECT * FROM product a JOIN (select id from product limit 866613, 20) b ON a.ID = b.id


如果对于有where 条件，又想走索引用limit的，必须设计一个索引，将where 后的字段放第一位，limit用到的主键放第2位，而且只能select 主键！
```



### join优化

小结果集驱动大结果集

被驱动表上join字段建立索引





### 慢查询优化

- 首先分析语句，看看是否load了额外的数据，可能是查询了多余的行并且抛弃掉了，可能是加载了许多结果中并不需要的列，对语句进行分析以及重写。
- 分析语句的执行计划，然后获得其使用索引的情况，之后修改语句或者修改索引，使得语句可以尽可能的命中索引。
- 如果对语句的优化已经无法进行，可以考虑表中的数据量是否太大，如果是的话可以进行横向或者纵向的分表。

### count(*) count(1) count(列)

#### 执行效果

- count(*)包括了所有的列，相当于行数，在统计结果的时候，不会忽略列值为NULL
- count(1)包括了忽略所有列，用1代表代码行，在统计结果的时候，不会忽略列值为NULL
- count(列名)只包括列名那一列，在统计结果的时候，会忽略列值为空（这里的空不是只空字符串或者0，而是表示null）的计数，即某个字段值为NULL时，不统计。

#### 执行效率

列名为主键，count(列名)会比count(1)快 
列名不为主键，count(1)会比count(列名)快 
如果表多个列并且没有主键，则 count（1） 的执行效率优于 count（*）
如果有主键，则 select count（主键）的执行效率是最优的 
如果表只有一个字段，则 select count（*）最优。



### exists和in对比

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







