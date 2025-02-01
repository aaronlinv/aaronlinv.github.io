---
date: '2021-02-03T17:00:33+08:00'
title: 'Oracle数据库对比MySQL'
categories: ["数据库"]
---

# 基本
Oracle默认端口：1521 默认用户：system
MySQL默认端口：3306 默认用户：root

连接MySQL：
```bash
mysql -u root -p
-- 输入密码

-- 查询所有数据库
show databases;
-- 切换到 "test" 这个数据库
use test;
-- 查询该数据库所有表
show tables;
```

连接Oracle：
```bash
sqlplus
-- 输入用户名
-- 输入密码

-- 查询该用户的表
select TABLE_NAME from user_tables;
```
注意：Oracle 登录需要授予登录用户 session权限，建表需要分配限额

# 常用字段类型
Oracle
    数值 number number(10) number(10,2)
    字符串 varchar2 varchar2(20)
    日期 date

MySQL
    数值 tinyint smallint mediumint int bigint decimal
    字符串 varchar(10)  必须指定
    日期 date time datetime timestamp year

# DML
## Oracle:
```sql
create table t_student(
    sid int primary key ,
    sname varchar2(10) not null ,
    enterdate date,
    gender char(2),
    mail unique,
    age number check (age>19 and age<30)
)
insert into t_student values(stuseq.nextval,'Test',to_date('1990-3-4','YYYY-MM-DD'),'男','1@outlook.com',20);
commit;
```
##　MySQL
```sql
create table t_student(
    sid int primary key auto_increment,
    sname varchar(1) not null ,
    enterdate date,
    gender char(1),
    age int,
    mail varchar(10) UNIQUE
)

insert into t_student values(null,'Test','1990-3-4','男',30,'2@outlook.com')
```
MySQL插入日期使用now() 或 sysdate()，可以插入多条，使用逗号隔开
删表数据：Oracle可以省略from：delete from t_student; (删除所有数据)

外键约束：Oracle是constraints,MySQL是constraint

级联操作：
- Oracle：on delete set null 或者on delete cascade
- MySQL: on delete set null on update CASCADE

# 多表操作
Oracle：92语法：可以内连接，外连接99语法：可以内连接，外连接，全外连接(full join)
```sql
-- SQL92 左外连接（保留左边, 注意(+)要放在右边，记忆：左外，右边会出现空行要+补齐） 
where e.department_id = d.department_id(+)
-- 
```
MySQL：只支持内连接、外连接，并且只能用类似oracle中99语法的格式写，MySQL不完全符合SQL-92规范
# SQL 语句
## MySQL
大小写不敏感(关键字和字段名都不区分) 

阿里巴巴Java开发手册，在MySQL建表规约里有：
【强制】表名、字段名必须使用小写字母或数字 ， 禁止出现数字开头，禁止两个下划线中间只出现数字。数据库字段名的修改代价很大，因为无法进行预发布，所以字段名称需要慎重考虑

**Windows 大小写不敏感，文件名同名大小写不同会覆盖**

MySQL 在 Windows 下不区分大小写，但在 Linux 下默认是区分大小写。因此，数据库名、 表名、字段名，都不允许出现任何大写字母，避免节外生枝
MySQL 的字段 大小写都可以查到
## Oracle
是Oracle大小写不敏感的前提条件是在没有使用双引号 "" 的前提下（表名、字段名）

CREATE TABLE "TableName"("id" number);  // 如果创建表的时候是这样写的，那么就必须严格区分大小写

SELECT * FROM "TableName"; // 不仅要区分大小写而且要加双引号，以便和上面的第三种查询方式区分开
Oracle默认是大写，对字段的具体值是敏感的

# 分页
Oracle：
```sql
-- 利用rownum 
-- rownum从0开始
select * from
(select rownum rr,stu.* from (select * from t_student order by sid desc) stu )
where rr>=1 and rr<=5;
```

MySQL：
```sql
-- 记录从0开始
-- 从第0条开始，取5条数据
select * from test2 order by sid desc  limit 0,5
```

# 时间日期
## Oracle
Java中常用的 "yyyy-MM-dd mm:HH;ss" -> "2021-02-03 16:25:48"
在 Oracle 中的表示方式：'yyyy-mm-dd hh24:mi:ss'
## MySQL
```sql
-- 获取当前时间戳 
select unix_timestamp(); 
-- 1612340981

-- 获取当前日期时间
select now();
2021-02-03 16:30:22

-- 获取当前日期
select date(now());
-- 2021-02-03

-- timestamp -> datetime
select FROM_UNIXTIME(1612340981);
-- 2021-02-03 16:29:41

-- datetime -> varchar  (time与之类似：time_format(time,format))
select  DATE_FORMAT('2008-08-08 22:23:01','%Y %m %d %H %i %s');
-- 2008 08 08 22 23 01

-- varchar -> date   str_to_date(str, format)
select str_to_date('08.09.2008 08:09:30', '%m.%d.%Y %h:%i:%s'); 
-- 2008-08-09 08:09:30
```

# Oracle
Oracle DML 需要手动提交或回滚事务
DML(Data Manipulation Language): 数据操纵语言 针对表数据的增删改查
Oracle select 查询必须有from 所以可以用from dual（这是一张神奇的表）

## 类型转换 
### date <--> varchar2 <--> number
date --> varchar2 : to_char(sysdate,'yyyy-mm-dd')
varchar2 --> date : to_date('2020-02-02','yyyy-mm-dd')

number --> varchar2: to_char(1111111.11,'999,999,999') -- 输出：1,111,111 使用'999,999,999'去匹配数字
varchar2 --> number :to_number('￥001,111,111','L000,000,000') from dual; -- 输出：1111111  

L表示：当地的货币符号 字符串在运算时会自动隐式转换，含有非数字字符会报错：无效数字


# 参考资料
[mysql和oracle的区别](https://zhuanlan.zhihu.com/p/39651803)
[Oracle| Oracle大小写敏感问题](https://blog.csdn.net/u011479200/article/details/89025708)
[SQL连接标准 SQL92\SQL99](https://www.jianshu.com/p/5c7cf8127109)
[MySQL 获得当前日期时间 函数](https://www.cnblogs.com/ggjucheng/p/3352280.html)
