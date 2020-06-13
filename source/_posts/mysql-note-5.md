---
title: MySQL数据库进阶2
categories: MySQL笔记
date: 2020-01-27 19:29:26
---
## 存储过程
一组可编程的函数，是为了完成特定功能的SQL语句集
经编译创建并保存在数据库字典中，用户可通过指定存储过程的名字并给定参数(需要时)来调用执行

为什么要用存储过程
- 将重复性很高的一些操作，封装到一个存储过程中，简化了对这些SQL的调用
- 批量处理
- 统一接口，确保数据的安全
- 相对于Oracle数据库来说，MySQL的存储过程相对功能较弱，使用较少


```sql
-- 创建存储过程
delimiter $$
CREATE PROCEDURE 存储过程名称()
BEGIN
语句
END $$
delimiter ;

-- 调用存储过程
call  存储过程名称();

-- 查询指定数据库中的存储过程
show procedure status where db='数据库名称';

-- 查询存储过程详情
show create procedure 存储过程名称;


--删除存储过程
drop procedure  存储过程名称;
```

#### DELIMITER
修改标准分隔符，这个命令与存储过程语法无关
作用是告诉MySQL解释器，该段命令是否已经结束了
默认情况下，DELIMITER是分号; ，在命令行客户端中，如果有一行命令以分号结束，那么回车后MySQL将立即执行该语句

在为可能输入较多的语句，且语句中包含有分号,DELIMITER $$ 把标准分隔符; 修改为$$，这样只有当$$出现之后，MySQL解释器才会执行这段语句

```sql
-- 创建存储过程
delimiter $$
create procedure show_emp()
begin
select * from emp;
end$$
delimiter ;

-- 执行存储过程
call show_emp();
```

#### 声明变量

```sql
delimiter $$
create procedure test()
begin
-- 声明变量
-- DECLARE 变量名 数据类型(大小) DEFAULT 默认值;
declare res varchar(20) default '';
-- 声明多个变量
declare x,y int default 0;

-- 通过into 赋值
declare avgRes double default 0;
select avg(salary) into avgRes from emp;

-- 输出变量
select avgRes;

end$$
delimiter ;
```

#### 存储过程参数
- IN 传入值
- OUT 传出值
- INOUT IN和OUT参数的组合

```sql
create produce name(参数类型 参数名称 数据类型(大小) )

-- IN 传入值
delimiter $$
create procedure getName (in name varchar(255))
begin 
select * from emp where ename = name;

end$$
delimiter ;
-- 传入参数
call getName('李白');


-- OUT 传出值
delimiter $$
create procedure getSalary(in n varchar(255), out eSalary int)
begin
select salary into eSalary from emp where ename = n;
end$$
delimiter ;
-- 传入参数
call getSalary('李白',@s);
select @s;

-- INOUT IN和OUT参数的组合
delimiter $$
create procedure getNum(inout num int,in x int )
begin
set num = num + x;

end$$
delimiter ;

-- 设置变量 传入参数
set @num1 = 20;
call getNum(@num1,10);
select @num1;

```

#### 存储过程语句
IF语句
```sql
IF expression THEN 
statements;
END IF;

IF expression THEN
statements;
ELSE
else-statements;
END IF;
```

CASE语句
```sql
CASE  case_expression
WHEN when_expression_1 THEN commands
WHEN when_expression_2 THEN commands
...
ELSE commands
END CASE;

```

循环语句
```sql
-- while循环
WHILE expression DO
statements
END WHILE

-- repeat循环
REPEAT
statements;
UNTIL expression
END REPEAT
```



## 自定义函数
```sql
-- 随机生成字符串

delimiter $$
-- 注意return加s
create function rand_str(n int) returns varchar(255)
begin 

-- 声明一个str 52个字母
declare str varchar(100) default 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';

-- 记录当前是第几个字符
declare i int default 0;

declare res_str varchar(255) default '';

while i < n do
-- 生成1-52 ：floor(1+RAND()*52)  floor()去掉小数
-- substr(str,floor(1+RAND()*52),1); -- 最后一个代表截取一位
set res_str = concat(res_str,substr(str,floor(1+RAND()*52),1));

set i = i + 1;
end while;

return res_str;

end $$
delimiter ;

-- 调用自定义函数
select rand_str(4);
```

#### 创建千万条数据

```sql
-- create table emp (id int,name varchar(50),age int);
-- 插入
delimiter $$
create procedure insert_emp(in startNum int,in max_num int)
begin

declare i int default 0;
-- 默认自动提交sql，这样比较慢，设置为不自动提交sql
set autocommit = 0;

repeat
set i = i + 1;

insert into emp values (startNum+i,rand_str(5),floor(10+rand()*30));

until i = max_num
end repeat;

commit; -- 整体提交所有sql，提高效率 
end$$
delimiter ;

-- 调用存储过程
call insert_emp(100,10000000);
```

## 索引
帮助MySQL高效获取数据的数据结构

优势
- 索引类似大学图书馆建立的书目索引，提高数据检索的效率，降低数据库的IO成本
- 通过索引对数据项进行排序，降低数据排序成本，降低了CPU的消耗

劣势
- 实际上索引也是一张表，该表保存了主键与索引字段，并指向实体表的记录，所以索引列也是要占空间的
- 虽然索引大大提高了查询速度，同时确会降低更新表的速度，如对表进行INSERT、UPDATE、DELETE

索引分类
- 单值索引：即一个索引只包含单个列，一个表可以有多个单列索引
- 唯一索引：索引列的值必须唯一，但允许有空值
- 复合索引：一个索引包含多个列 INDEX MultiIdx(id,name,age)
- 全文索引：只能在MyISAM引擎上才能使用，只能在CHAR,VARCHAR,TEXT类型字段上使用全文索引
- 空间索引：只能在MyISAM引擎上才能使用，对空间数据类型的字段建立索引

索引操作
自动创建索引
- 在表上定义了主键时， 会自动创建一个对应的唯一索引
- 在表上定义了一个外键时，会自动创建一个普通索引

explain
用来查看索引是否正在被使用，并且输出其使用的索引的信息
- key：实际选用的索引
- key_len：显示了MySQL使用索引的长度(也就是使用的索引个数)，当 key 字段的值为 null时，索引的长度就是 null。注意，key_len的值可以告诉你在联合索引中mysql会真正使用了哪些索引。这里就使用了1个索引，所以为1，

```sql
-- 创建索引
-- CREATE INDEX 索引名称 ON table (column[, column]...);
-- 默认是NORMAL类型索引
create INDEX salary_index ON emp(salary);

-- 删除索引
DROP INDEX 索引名称 ON 表名

-- 查看索引
show index from 表名;

```

索引方法：innoDB不支持Hash，只能Btree
Btree索引：B+树
Hash索引：哈希算法


哪些情况需要创建索引
- 主键自动建立唯一索引
- 频繁作为查询条件的字段应该创建索引
- 查询中与其他表关联的字段，外键关系建立索引
- 频繁更新的字段不适合建立索引，因为每次更新不单单是更新了记录还会更新索引
- WHERE条件里用不到的字段不创建索引
- 查询中排序的字段，排序的字段若通过索引去访问将大大提高排序速度
- 查询中统计或者分组字段

哪些情况不需要创建索引
- 表记录太少
- 经常增删改的表
- 如果某个数据列包含许多重复的内容，为它建立索引就没有太大的实际效果

## 参考资料
[Java零基础到高级MySQL数据库](https://study.163.com/course/introduction/1005932016.htm)