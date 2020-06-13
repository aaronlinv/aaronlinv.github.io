---
title: MySQL数据库基础
categories: MySQL笔记
date: 2020-01-16 21:28:31
---

## SQL语句概述
Structured Query Language (结构化查询语言)
SQL是专为数据库而建立的操作命令集，是一种功能齐全的数据库语言

#### SQL语句分类
- DDL DataDefinationLanguage 数据定义语言：用来定义数据库对象：创建库，表，列等
- DML DataManipulationLanguage 数据操作语言：用来操作数据库表中的记录
- DQL DataQueryLanguage 数据查询语言：用来查询数据
- DCL DataControlLanguage 数据控制语言：用来定义访问权限和安全级别

#### SQL数据类型
SQMySQL支持所有标准SQL数值数据类型

常用数据类型
- int：整数
- double：浮点型，例如double(5,2)表示最多5位，其中- 须有2位小数，即最大值为999.99
- char：固定长度字符串类型 char(10)  'abc       ' （abc后面有7个空格）
- varchar：可变长度字符串类型；varchar(10) 'abc'
- text：字符串类型
- blob：二进制类型
- date：日期类型，格式为：yyyy-MM-dd
- time：时间类型，格式为：hh:mm:ss （HH是24小时制，hh是12小时制）
- datetime:日期时间类型 yyyy-MM-dd hh:mm:ss

## DDL (对表结构的操作)
```sql
-- 数据库操作
-- 1.创建数据库 create database 数据库名  character set utf8;
create database my_database character set utf8;

-- 2.修改数据库字符集 alter database 数据库名 charactor set gbk;
alter database my_database character set gbk;


-- 选择数据库 use 数据库名;
use my_database;


-- 表操作
-- 1.新建表 create table 表名 (列名 数据类型,类名 数据类型...);
create table my_table (name varchar(20),age int);

-- 2.查看表的创建细节 show create table 表名;
show create table my_table;

-- 3.查看表的字段信息 desc 表名;
desc my_table;

-- 4.添加一列(字段) alter table 表名 add 列名 数据类型;
alter table my_table add id bigint;

-- 5.修改一个表的字段类型 alter table 表名 modify 字段名 数据类型;
alter table my_table modify id int;

-- 6.修改表的列名 alter table 表名 change 原始列名 新列名 数据类型;
alter table my_table change age id bigint;

-- 7.删除一列 alter table 表名 drop 字段名;
alter table my_table drop id;

-- 8.修改表名 rename table 原始表名 to 要修改的表名;
rename table my_table to my_t;

-- 9.删除表 drop table 表名;
drop table my_t;

-- 10.删除数据库 drop database 数据库名;
drop database my_database;
```
注意点：
- 字符集UTF-8要写作utf8
- modify只能修改数据类型，change可以修改字段名和数据类型，修改的数据类型都在最后指定
- 修改表名比较特殊：rename table 原始表名 to 要修改的表名;

## DML（对表的数据增、删、改）

#### 插入操作
```sql
-- 建表：create table my_table (name varchar(20),age int);
-- insert into 表名（列名1，列名2 ...）value (列值1，列值2...);
insert into my_table (name,age) value ('zs',28);

-- 插入多条
insert into my_table (name,age) values 
    ('ls',30)，
    ('ww',29);

-- 查询是否插入成功 select * from 表名;
select * from my_table;
```
注意事项
- 列名与列值的类型、个数、顺序要一一对应
- 值不要超出列定义的长度
- 插入的日期和字符一样，都使用引号括起来
- 插入多条，多条数据之间用逗号隔开
- 插入一条或者多条时，用value或者values都可

#### 更新操作

```sql
-- updata 表名 set 列名1=列值1,列名2=列值2...  where 列名=值
-- 修改表中所有列的age为10
update my_table set age=10; 

-- 修改name为'zs'的列的age为20
update my_table set age=20 where name='zs'; 
```

#### 删除操作
```sql
-- delete from 表名 where 列名=值;
delete from my_table where name='zs';

-- truncate table 表名;
truncate table my_table;
```
delete与truncate的区别
- delete 删除表中的数据，表结构还在;删除后的数据可以找回
- truncate 删除是把表直接drop掉，然后再创建一个同样的新表，删除的数据不能找回，执行速度比delete快

## DQL（查询表数据）

#### 条件查询和运算符
= != <>(不等于) < <= > >=
BETWEEN...AND...
IN
IS NULL, IS NOT NULL
AND OR NOT 

#### 模糊查询
通配符：
1. _：任意一个字符
2. %：任意0-n个字符
写在WHERE关键词后

```sql
-- 查询姓名中包含“s”字母的学生记录
SELECT * FROM stu WHERE name LIKE '%s%';
```

#### 字段控制查询
1. DISTINCT 去重
2. 查询结果进行运算，必须都是数据型（值可能为NULL的需要转换为0，任何值与NULL相加还是NULL）

```sql
SELECT *,age+IFNULL(score,0) FROM students;
```

3. 对查询结果起别名 AS 可省略AS

```sql
SELECT *, yw+IFNULL(sx,0) AS total FROM score;
```

#### 排序
ORDER BY
默认ASC 升序，DESC降序

#### 聚合行数
COUNT()
MAX()
MIN()
SUM()
AVG()

#### 分组查询
将查询结果按照1个或多个字段进行分组，字段值相同的为一组
当group by单独使用时，只显示出每组的第一条记录
在使用分组时，select后面直接跟的字段一般都出现在group by 后
- group_concat(字段名)：显示分组后每一组的某字段的值的集合
- 聚合函数：显示分组后每一组的聚合函数值
- having：筛选分组之后的结果

#### LIMIT

```sql
-- SELECT * FROM 表名 LIMIT 从哪一行开始查询,需要查询的行数;
SELECT * FROM stu LIMIT 0,3;
```
#### 顺序
书写顺序
![](./mysql-note-3/1.png)

执行顺序
![](./mysql-note-3/2.png)

## 数据完整性

保证用户输入的数据保存到数据库中是正确的
1. 实体完整性：一行（一条记录）数据代表一个实体 （标识每一行数据不重复，行级约束）
   - 主键
   - 唯一
   - 自动增长列
2. 域完整性：限制此单元格的数据正确
   - 数据类型
   - 非空约束 (NOT NULL)
   - 默认值约束 (DEFAULT)
3. 参照完整性：指表与表之间的一种对应关系
通常情况下可以通过设置两表之间的主键、外键关系，或者编写两表的触发器来实现   
对两张表的要求：
   - 数据库的主键和外键类型一定要一致
   - 两个表必须得要是InnoDB类型

```sql
-- 实体完整性
-- 三个完整性约束都可以在建表时，在数据类型后指定
-- CREATE TABLE 表名(字段名 数据类型 PRIMARY KEY | UNIQUE | AUTO_INCREMENT);
CREATE TABLE 表名(字段名1 数据类型 PRIMARY KEY,字段2 数据类型);
CREATE TABLE 表名(字段名1 数据类型, 字段2 数据类型 UNIQUE);
CREATE TABLE 表名(字段名1 数据类型 PRIMARY KEY AUTO_INCREMENT ，字段2 数据类型 UNIQUE);
-- PRIMARY KEY 和 UNIQUE 可以在参数最后指定
CREATE TABLE 表名(字段1 数据类型, 字段2 数据类型,PRIMARY KEY | UNIQUE (要设置的字段));

-- 联合主键：两个字段数据同时相同时，才违反联合主键约束
CREATE TABLE 表名(字段1 数据类型, 字段2 数据类型,PRIMARY KEY(主键1，主键2));
-- 建表后设置主键
ALTER TABLE student  ADD CONSTRAINT  PRIMARY KEY (要设置的字段);


-- 域完整性
-- 非空约束 NOT NULL
CREATE TABLE 表名(字段名1 数据类型 PRIMARY KEY AUTO_INCREMENT ，字段2 数据类型 UNIQUE NOT NULL);
-- 默认值约束DEFAULT
CREATE TABLE 表名(字段名1 数据类型 PRIMARY KEY AUTO_INCREMENT ，字段2 数据类型 NOT NULL DEFAULT '男');

-- 参照完整性
-- 创建student表
CREATE TABLE student(sid int PRIMARY key,name varchar(50) not null,sex varchar(10) default '男');

-- 创建score表，设置外键
CREATE TABLE score(
sid INT,
score DOUBLE,
CONSTRAINT fk_stu_score_sid FOREIGN KEY(sid) REFERENCES student(sid));
-- CONSTRAINT：约束，fk_stu_score_sid是约束名，FOREIGN KEY代表外键，REFERENCES：参考
```
#### 实体完整性
主键约束
- 每个表中要有一个主键（或者联合主键）
- 每个数据唯一，非空

唯一约束
- 指定列的数据不能重复
- 可以为空值

自动增长列
- 指定列的数据自动增长
- 即使数据删除，还是从删除的序号继续往下
- 设置自动增长的列必须时主键

#### 域完整性
三种：数据类型、非空约束、默认值约束

#### 参照完整性
设置参照完整性后 ，外键当中的内值，必须得是主键当中的内容
  
	
## 表之间的关系
一对一
一对多
多对多
- 学生选课:一个学生可以选修多门课程，每门课程可供多个学生选择
- 需要创建中间的关系表
- 拆分表：避免大量冗余数据的出现
![冗余数据](mysql-note-3/3.png)
## 多表查询
#### 合并结果集
合并结果集就是把两个select语句的查询结果合并到一起		
- UNION：合并时去除重复记录
- UNION ALL：合并时不去除重复记录
- 注意事项:被合并的两个结果列数、列类型必须相同

```sql
SELECT * FROM 表1   UNION | UNION ALL   SELECT * FROM 表2；
```
#### 连接查询
也可以叫跨表查询，需要关联多个表进行查询

```sql
-- 同时查询两个表，出现的就是笛卡尔集结果
SELECT * FROM student,score;
	
-- 在查询时要把主键和外键保持一致,逐行判断，相等的留下，不相等的全不要
SELECT * FROM student st,score sc WHERE st.id = sc.sid;			

```
#### 多表连接
内连接
- 等值连接
- 非等值连接
- 自连接

外连接
- 左外连接（左连接）
- 右外连接（右连接）
两种连接方式同理，只是顺序不同

自然连接
- 根据主外键等式，自动去除无用的笛卡尔集
- 两张连接的表中列名称和类型完全一致


```sql
-- 等值连接 (INNER可省略) 与多表联查约束主外键是一样，只是写法改变了
SELECT * FROM student st INNER JOIN score sc ON st.id = sc.sid;

-- 多表连接
-- 99连接法（在WHERE中用AND并列条件）
select * from stu st,score sc,course c where st.id=sc.sid and sc.cid =c.cid;
-- 内连接
select * from stu st 
join score sc on st.id = sc.sid
join course c on c.cid=sc.cid;

-- 非等值连接
-- 查询所有员工的姓名，工资，所在部门的名称以及工资的等级
select e.ename,s.grade from emp e
join dept d on e.deptno = d.deptno
join salgrade s on e.salary BETWEEN s.lowSalary and s.highSalary;

-- 自连接 （给同一个表起不同别名）
-- 求7369员工编号、姓名、经理编号和经理姓名
select e1.ename,e1.empno,e2.ename from 
emp e1 join emp e2 on e1.mgr=e2.empno
where e1.empno = 7369;


-- 左外连接
-- 左边表当中的数据全部查出，右边表当中，只查出满足条件的内容（右表中不存在的会用NULL填充）
-- 查询所有学生的成绩信息（所有学生都会被查询到，没有成绩的同学成绩会显示NULL）
-- 可以省略outer
select * from stu st left outer join score sc on st.id=sc.sid; 

-- 自然连接
select * from stu natural join score;
```


#### 子查询
一个select语句中包含另一个完整的select（子查询语句需要用括号括起来）

子查询出现的位置
- where后，把select查询出的结果当作另一个select的条件值
- from后，把查询出的结果当作一个新表


```sql
-- where后：先查出项羽所在的部门编号
select * from emp 
where deptno=
    (select deptno from emp
    where ename='项羽') ;

-- from后：查询30号部门薪资大于2000
select * from 
(select * from emp where deptno=30) s
where s.salary>2000;
-- 注意要对表起别名
```


## 参考资料
[Java零基础到高级MySQL数据库](https://study.163.com/course/introduction/1005932016.htm)