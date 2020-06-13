---
title: MySQL数据库进阶1
categories: MySQL笔记
date: 2020-01-23 21:28:31
---
## 函数
- 字符串函数
- 数值函数
- 日期和时间函数
- 其他常用函数
## 事务
不可分割的操作，假设该操作有ABCD四个步骤组成
若ABCD四个步骤都成功完成,则认为事务成功，若ABCD中任意一个步骤操作失败，则认为事务失败
每条sql语句都是一个事务，事务只对DML语句有效，**对于DQL无效**
#### 事务的ACID
- 原子性（Atomicity）：原子性是指事务包含的所有操作要么全部成功，要么全部失败回滚
- 一致性（Consistency）：一致性是指事务必须使数据库从一个一致性状态变换到另一个一致性状态，也就是说一个事务执行之前和执行之后都必须处于一致性状态
- 隔离性（Isolation）隔离性是当多个用户并发访问数据库时，比如操作同一张表时，数据库为每一个用户开启的事务，不能被其他事务的操作所干扰，多个并发事务之间要相互隔离
- 持久性（Durability）持久性是指一个事务一旦被提交了，就不能再回滚了，已经把数据保存到数据库当中了

```sql
-- 需要手动开启事务
start transaction
-- 执行update后查询score没有被改变
-- 事务对于DQL无效
update score set score = 10;
-- 执行了commit后，数据才会改变
commit;
-- 如果想撤销之前的sql语句使用回滚
rollback;
```
#### 事务的并发问题
- 脏读:读取到了别的事务回滚前的脏数据
- 不可重复读：一个事务两次相同的查询却返回了不同数据（两次查询中被其他事务修改了数据）
- 幻读：开启事务查询，另一个事务插入或删除数据并提交，再次查询发现与原来数据不同，像是出现了幻觉

不可重复读的重点是**修改**，同样的条件，你读取过的数据，再次读取出来发现值不一样；（主要在于update和delete）
幻读的重点在于**新增或者删除**，同样的条件，第 1 次和第 2 次读出来的记录数不一样。（主要在于insert）

|         事务隔离级别         | 脏读  | 不可重复读 | 幻读  |
| :--------------------------: | :---: | :--------: | :---: |
| 读未提交（read-uncommitted） |  是   |     是     |  是   |
| 不可重复读（read-committed） |       |     是     |  是   |
| 可重复读（repeatable-read）  |       |            |  是   |
|    串行化（serializable）    |       |            |       |

Read uncommitted：读未提交，一个事务可以读取另一个未提交事务的数据
Read committed：读提交，一个事务要等另一个事务提交后才能读取数据
Repeatable read：重复读，在开始读取数据（事务开启）时，不再允许删除或修改操作 (MySQL默认这个级别)
Serializable：事务串行化顺序执行，可以避免脏读、不可重复读与幻读，这种事务隔离级别效率低下，比较耗数据库性能，一般不使用

```sql
-- 查看隔离级别
select @@global.tx_isolation,@@tx_isolation;
-- 修改当前会话隔离级别
set session transaction isolation level 隔离级别;
-- 修改全局会话隔离级别（需要重开会话）
set global transaction isolation level read committed; 

```

## 权限
限制一个用户能够做什么事情，在MySQL中，可以设置全局权限，指定数据库权限，指定表权限，指定字段权限

权限分类
CREATE 创建数据库、表或索引权限
DROP 除数据库或表权限
ALTER 更改表，比如添加字段、索引等
DELETE 删除数据权限
INDEX 索引权限
INSERT 插入权限
SELECT 查询权限
UPDATE 更新权限
CREATE VIEW 创建视图权限
EXECUTE 执行存储过程权限

```sql
-- 创建用户 ，新建的用户只能看到information_schema数据库 ，下面不加单引号也可
create user '用户名'@'localhost';
-- 分配权限 (把mytest数据库里所有表的select权限分配给用户dev)
-- with grant option 表示这个被授予权限的用户可以把权限传递给其他用户
-- flush privileges; 刷新权限
grant select on my_test.* to dev@localhost  with grant option;
flush privileges;

-- 创建对所有数据库的所有权限
grant all privileges on *.* to dev@localhost  with grant option;
flush privileges;

-- 分配一个用户只能对stu表进行CRUD操作的权限
grant insert,update,select,delete on my_test.stu to privuser@localhost ;
flush privileges;

-- 查看权限
show grants
-- 查看指定用户的权限
show grants for root@localhost

-- 删除权限
revoke 权限 on 数据库对象 from 用户名@localhost；
flush privileges;

-- 删除用户
drop user 用户名@localhost;
```

## 视图
视图是一个虚拟表，其内容由查询定义

视图是对若干张基本表的引用，一张虚表，查询语句执行的结果
不存储具体的数据（基本表数据发生了改变，视图也会跟着改变）
可以跟基本表一样，进行增删改查操作(增删改操作有条件限制)
	
优点：安全性 提高查询性能 提高数据独立性

```sql
-- 创建视图
create view emp_salary_view
as(select * from emp 
where salary >2000
);

-- 查询视图
select * from emp_salary_view;

-- 修改视图  
-- create or replace view
create or replace view emp_salary_view
as(select * from emp 
where salary >2999
);

-- 删除视图
drop view emp_salary_view;
```



ALGORITHM参数
- MERGE：替换式，可以通过修改视图数据更新真实表中的数据
- TEMPTABLE：具化式，由于数据存储在临时表中，所以不可以进行更新操作
- UNDEFINED：没有定义ALGORITHM参数，MySQL更倾向于选择MERGE替换方式，因为它更加有效
	
替换式与具化式区别
- MERGE替换式，将视图公式替换后，当成一个整体sql进行处理了
- TEMPTABLE具化式，先处理视图结果，该结果形成一个中间结果暂时存在内存中，后处理外面的查询需求

```sql
-- MERGE替换式（MySQL默认方式）
select * from (select * from emp where salary>2000) t;
-- TEMPTABLE具化式
-- (select * from emp where salary>2000) as temptable; 不可执行用于理解
-- select * from temptable1;
```

WITH CHECK OPTION
- 更新数据时不能插入或更新不符合视图限制条件的记录

LOCAL和CASCADED
- WITH [CASCADED|LOCAL] CHECK OPTION
- 为可选参数，决定了检查测试的范围，MySQL默认值为CASCADED

```sql
-- with check option
-- 通过视图修改salary的值就不能小等于2500
create view emp_s_v
as ( select * from emp where salary>2500 
) with check option;

```


视图不可更新部分：只要视图当中的数据不是来自于基表，就不能够直接修改
- 聚合函数
- DISTINCT 关键字
- GROUP BY子句
- HAVING 子句
- UNION 运算符
- FROM 子句中包含多个表
- SELECT 语句中引用了不可更新视图



## 参考资料
[Java零基础到高级MySQL数据库](https://study.163.com/course/introduction/1005932016.htm)
[数据库事务、事务隔离级别以及锁机制详解](https://www.cnblogs.com/jieerma666/p/10805578.html)