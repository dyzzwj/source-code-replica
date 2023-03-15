数据表介绍

--1.学生表
Student(SId,Sname,Sage,Ssex)
--SId 学生编号,Sname 学生姓名,Sage 出生年月,Ssex 学生性别

--2.课程表
Course(CId,Cname,TId)
--CId 课程编号,Cname 课程名称,TId 教师编号

--3.教师表
Teacher(TId,Tname)
--TId 教师编号,Tname 教师姓名

--4.成绩表

SC(SId,CId,score)
--SId 学生编号,CId 课程编号,score 分数



**学生表 Student**

```sql
create table Student(SId varchar(10),Sname varchar(10),Sage datetime,Ssex varchar(10));
insert into Student values('01' , '赵雷' , '1990-01-01' , '男');
insert into Student values('02' , '钱电' , '1990-12-21' , '男');
insert into Student values('03' , '孙风' , '1990-12-20' , '男');
insert into Student values('04' , '李云' , '1990-12-06' , '男');
insert into Student values('05' , '周梅' , '1991-12-01' , '女');
insert into Student values('06' , '吴兰' , '1992-01-01' , '女');
insert into Student values('07' , '郑竹' , '1989-01-01' , '女');
insert into Student values('09' , '张三' , '2017-12-20' , '女');
insert into Student values('10' , '李四' , '2017-12-25' , '女');
insert into Student values('11' , '李四' , '2012-06-06' , '女');
insert into Student values('12' , '赵六' , '2013-06-13' , '女');
insert into Student values('13' , '孙七' , '2014-06-01' , '女');

```

**科目表 Course**

```sql
create table Course(CId varchar(10),Cname nvarchar(10),TId varchar(10));
insert into Course values('01' , '语文' , '02');
insert into Course values('02' , '数学' , '01');
insert into Course values('03' , '英语' , '03');

```

**教师表Teacher**

```java
create table Teacher(TId varchar(10),Tname varchar(10));
insert into Teacher values('01' , '张三');
insert into Teacher values('02' , '李四');
insert into Teacher values('03' , '王五');

```

**成绩表 SC**

```sql
create table SC(SId varchar(10),CId varchar(10),score decimal(18,1));
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



# 练习题目

1. 查询" 01 "课程比" 02 "课程成绩高的学生的信息及课程分数


```sql
SELECT a.SId, a.score, b.score
FROM SC a
	LEFT JOIN SC b ON a.SId = b.SId
WHERE a.score > b.score
	AND a.CId = '01'
	AND b.CID = '02'
```

1.1 查询同时存在" 01 "课程和" 02 "课程的情况

```sql
SELECT SId
FROM SC
WHERE CId IN ('01', '02')
GROUP BY SId
HAVING COUNT(CId) = 2
```



1.2 查询存在" 01 "课程但可能不存在" 02 "课程的情况(不存在时显示为 null )

```sql
SELECT a.*, b.*
FROM (
	SELECT *
	FROM SC
	WHERE CId = '01'
) a
	LEFT JOIN (
		SELECT *
		FROM SC
		WHERE CId = '02'
	) b
	ON a.SId = b.SId
```

1.3 查询不存在" 01 "课程但存在" 02 "课程的情况

```sql
SELECT *
FROM SC
WHERE SId NOT IN (
		SELECT SId
		FROM SC
		WHERE CId = '01'
	)
	AND CId = '02'
```



2.查询平均成绩大于等于 60 分的同学的学生编号和学生姓名和平均成绩

```sql
SELECT s.SId, s.Sname, AVG(sc.score)
FROM SC sc
	LEFT JOIN student s ON sc.SId = s.SId
GROUP BY sc.SId
HAVING AVG(sc.score) > 60
```

3.查询在 SC 表存在成绩的学生信息

```sql
SELECT s.*
FROM student s
WHERE SId IN (
	SELECT DISTINCT SId
	FROM SC
)
```

4.查询所有同学的学生编号、学生姓名、选课总数、所有课程的总成绩(没成绩的显示为 null )

```sql
SELECT s.SId, s.Sname, COUNT(sc.CId)
	, SUM(sc.score)
FROM student s
	LEFT JOIN SC sc ON s.SId = sc.SId
GROUP BY sc.SId
```

4.1 查有成绩的学生信息

```sql
SELECT s.SId, s.Sname, COUNT(sc.CId)
	, SUM(sc.score)
FROM student s
	INNER JOIN SC sc ON s.SId = sc.SId
GROUP BY sc.SId
```

5.查询「李」姓老师的数量

```sql
SELECT COUNT(Tname)
FROM teacher
WHERE Tname LIKE "李%"
```

6.查询学过「张三」老师授课的同学的信息

```sql
SELECT s.*
FROM student s
	LEFT JOIN SC sc ON s.SId = sc.SId
	LEFT JOIN course c ON sc.CId = c.CId
	LEFT JOIN teacher t ON c.TId = t.TId
WHERE t.Tname = '张三'
```

7.查询没有学全所有课程的同学的信息

```sql
SELECT SId
FROM SC
GROUP BY SId
HAVING COUNT(CId) < (
	SELECT COUNT(CId)
	FROM course
)
```

8.查询至少有一门课与学号为" 01 "的同学所学相同的同学的信息

```sql
SELECT *
FROM SC
WHERE SId NOT IN ('01')
	AND CId IN (
		SELECT CId
		FROM SC
		WHERE SId = '01'
	)
LIMIT 1
```

9.查询和" 01 "号的同学学习的课程 完全相同的其他同学的信息

```sql
SELECT SId
FROM SC
WHERE SId IN (
		SELECT CId
		FROM SC
		WHERE SId = '01'
	)
	AND SId NOT IN ('01')
GROUP BY SId
HAVING COUNT(CId) = (
	SELECT COUNT(CId)
	FROM SC
	WHERE SId = '01'
)
```



10、查询没学过"张三"老师讲授的任一门课程的学生姓名

```sql
SELECT Sid
FROM student
WHERE SId NOT IN (
	SELECT sc.SId
	FROM SC sc
		LEFT JOIN course c ON sc.CId = c.CId
		LEFT JOIN teacher t ON c.TId = t.TId
	WHERE t.Tname = '张三'
)
```



11.查询两门及其以上不及格课程的同学的学号，姓名及其平均成绩

```sql
SELECT s.*, AVG(sc.score)
FROM student s
	LEFT JOIN SC sc ON s.SId = sc.SId
WHERE score < 60
GROUP BY SId
HAVING COUNT(score) > 1
```

12.检索" 01 "课程分数小于 60，按分数降序排列的学生信息

```sql
SELECT *
FROM SC
WHERE CId = '01'
	AND score < 60
ORDER BY score DESC
```

13.按平均成绩从高到低显示所有学生的所有课程的成绩以及平均成绩

```sql
SELECT a.*, b.avg_score
FROM SC a
	LEFT JOIN (
		SELECT SId, AVG(score) AS avg_score
		FROM SC
		GROUP BY SId
	) b
	ON a.SId = b.SId
ORDER BY b.avg_score DESC
```





14.按各科成绩进行排序，并显示排名， Score 重复时合并名次



15.查询各科成绩前三名的记录

```sql
SELECT a.CId, a.score, a.SId
FROM sc a
	LEFT JOIN sc b
	ON a.CId = b.CId
		AND a.score < b.score
GROUP BY a.CId, a.SId, a.score
HAVING COUNT(a.score) < 3
ORDER BY a.CId, a.score DESC
```



```sql
SELECT *
FROM sc a
WHERE (
	SELECT COUNT(SId)
	FROM sc
	WHERE CId = a.CId
		AND score > a.score
) < 2
ORDER BY CId, score DESC
```









16.查询每门课程被选修的学生数

```
SELECT CId, COUNT(SId)
FROM SC
GROUP BY CId
```



17.查询出只选修两门课程的学生学号和姓名

```sql
SELECT s.SId, s.Sname
FROM sc c
	LEFT JOIN student s ON c.SId = s.SId
GROUP BY c.SId
HAVING COUNT(c.CId) = 2
```



18.













