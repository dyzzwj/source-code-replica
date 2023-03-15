## 执行流程

1、解析全局配置文件

2、解析mapper映射文件

3、解析sql，将#{}替换为?，${}暂不处理（执行时才做替换），将sql的所有信息封装为MappedStatement对象

4、构建SqlSessionFactory

5、使用SqlSessionFactory创建SqlSession，每个SqlSession分配一个Executor

6、使用SqlSession.getMapper(class)获取接口的代理对象

7、调用接口的方法（目标方法），实际被MapperProxy.invoke()拦截到

8、方法参数转换为sql参数，获取MappedStatement，${} 直接替换成参数值，二级缓存，一级缓存，去数据库查，创建StatementHandler（完成StatementHandler、ParameterHandler、ResultSetHandler的创建和插件代理），为？赋值，执行查询

9、处理结果集



## #{}和${}的区别

1、#{}在mybatis启动时，就编译成了 ? ，${}是在执行时，才会做拼接

2、#{}是占位符，预编译处理（PreparedStatmentHanlder），可以防止sql注入；${}是拼接符，字符串替换，没有预编译处理（StatementHandler），不能防止sql注入

3、#{} 的变量替换是在DBMS 中；${} 的变量替换是在 DBMS 外



## 如何防止sql注入

1、sqlz预编译

2、过滤参数中含有的一些数据库关键词

3、避免直接向用户显示数据库错误

4、限制数据库权限



##mybatis一对多





四大对象

- ParameterHandler：处理SQL的参数**对象**
- ResultSetHandler：处理SQL的返回结果集
- StatementHandler：数据库的处理**对象**，用于执行SQL语句
- Executor：**MyBatis**的执行器，用于执行增删改查操作







## **分页插件原理**



##mepper手动传参

方法一：顺序传参法(**不推荐**)

```java
public User selectUser(String name, int deptId);

<select id="selectUser" resultMap="UserResultMap">
    select * from user
    where user_name = #{0} and dept_id = #{1}
</select>
```

\#{}里面的数字代表传入参数的顺序。



方法二：@Param注解传参法

方法三：Map传参法

方法四：javabean传参法



## 模糊查询like语句该怎么写

（1）’%${question}%’ 可能引起SQL注入，不推荐

（2）"%"#{question}"%" 注意：因为#{…}解析成sql语句时候，会在变量外侧自动加单引号’ '，所以这里 % 需要使用双引号" "，不能使用单引号 ’ '，不然会查不到任何结果。

**（3）CONCAT(’%’,#{question},’%’) 使用CONCAT()函数，推荐**

（4）使用bind标签

```xml
<select id="listUserLikeUsername" resultType="com.jourwon.pojo.User">
　　<bind name="pattern" value="'%' + username + '%'" />
　　select id,sex,age,username,password from person where username LIKE #{pattern}
</select>
```

##结果集手动映射

第1种： 通过在查询的SQL语句中定义字段名的别名，让字段名的别名和实体类的属性名一致。

```xml
<select id="getOrder" parameterType="int" resultType="com.jourwon.pojo.Order">
       select order_id id, order_no orderno ,order_price price form orders where order_id=#{id};
</select>
```

第2种： 通过`<resultMap>`来映射字段名和实体类属性名的一一对应的关系。

```xml
<select id="getOrder" parameterType="int" resultMap="orderResultMap">
	select * from orders where order_id=#{id}
</select>

<resultMap type="com.jourwon.pojo.Order" id="orderResultMap">
    <!–用id属性来映射主键字段–>
    <id property="id" column="order_id">

    <!–用result属性来映射非主键字段，property为实体类属性名，column为数据库表中的属性–>
    <result property ="orderno" column ="order_no"/>
    <result property="price" column="order_price" />
</reslutMap>
```



##动态sql

其执行原理为，使用OGNL从sql参数对象中计算表达式的值，根据表达式的值动态拼接sql，以此来完成动态sql的功能。





##一级缓存

一级缓存是SqlSession级别的缓存。在操作数据库时需要构造 sqlSession对象，在对象中有一个(内存区域)数据结构（HashMap）用于存储缓存数据。不同的sqlSession之间的缓存数据区域（HashMap）是互相不影响的。

一级缓存的作用域是同一个SqlSession，在同一个sqlSession中两次执行相同的sql语句，第一次执行完毕会将数据库中查询的数据写到缓存（内存），第二次会从缓存中获取数据将不再从数据库查询，从而提高查询效率。当一个sqlSession结束后该sqlSession中的一级缓存也就不存在了。Mybatis默认开启一级缓存。

**sqlSession去执行commit操作（执行插入、更新、删除），清空SqlSession中的一级缓存**

##二级缓存

二级缓存是mapper级别的缓存，多个SqlSession去操作同一个Mapper的sql语句，多个SqlSession去操作数据库得到数据会存在二级缓存区域，多个SqlSession可以共用二级缓存，二级缓存是跨SqlSession的。

  二级缓存是多个SqlSession共享的，其作用域是mapper的同一个namespace，不同的sqlSession两次执行相同namespace下的sql语句且向sql中传递参数也相同即最终执行相同的sql语句，第一次执行完毕会将数据库中查询的数据写到缓存（内存），第二次会从缓存中获取数据将不再从数据库查询，从而提高查询效率。Mybatis默认没有开启二级缓存需要在setting全局参数中配置开启二级缓存。cacheEnable=true

如果缓存中有数据就不用从数据库中获取，大大提高系统性能。



### 如何获取生成的主键

```xml
<insert id="insert" useGeneratedKeys="true" keyProperty="userId" >
    insert into user( 
    user_name, user_password, create_time) 
    values(#{userName}, #{userPassword} , #{createTime, jdbcType= TIMESTAMP})
</insert>
```



































