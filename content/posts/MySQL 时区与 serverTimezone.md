---
date: '2024-12-19T08:40:33+08:00'
title: 'MySQL 时区与 serverTimezone'
categories: ["数据库"]
---

## TL;DR

1. 手动为 MySQL 指定非偏移量的时区，以避免 `TIMESTAMP` 类型夏令时问题和时区转化性能瓶颈
2. TIMESTAMP 范围：'1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07'
3. 连接 MySQL 数据库时，serverTimezone 参数用于指定数据库服务器的时区，需要设置为与 MySQL 服务端相同的时区

## MySQL 时区设置影响 TIMESTAMP 类型数据和部分时间函数

MySQL 会话时区设置会影响 `TIMESTAMP` 和 时间函数（NOW()、CURDATE()、CURTIME()、CURRENT_TIMESTAMP()）

存储 `TIMESTAMP` 类型数据时，MySQL 会根据当前会话的时区将时间转换为 UTC 时间，MySQL 实际存储的是 UTC 时间。检索时 MySQL 根据会话的时区将存储的 UTC 时间转换为会话对应时区的时间。而 DATETIME 类型的字段存储的时间值是原始值，不受时区影响

MySQL 默认使用 SYSTEM 时区（即操作系统的时区），每个需要时区计算的 MySQL 函数调用都会调用系统库来确定当前系统时区。此调用可能受到全局互斥体的保护，从而导致争用，建议显式设置时区


### 查询当前时区

```sql
# time_zone：MySQL 使用 SYSTEM 的时区
# system_time_zone：SYSTEM 为 CST 时区
show variables like "%time_zone%";
+------------------+--------+
| Variable_name    | Value  |
+------------------+--------+
| system_time_zone | CST    |
| time_zone        | SYSTEM |
+------------------+--------+
```

### 不同会话时区对 时间函数 的影响

```sql
# 当前时区 
# 查看当前的全球和会话时区值
SELECT @@GLOBAL.time_zone, @@SESSION.time_zone;

SELECT NOW(), CURDATE(), CURTIME(), CURRENT_TIMESTAMP();

set time_zone = 'America/New_York';

SELECT NOW(), CURDATE(), CURTIME(), CURRENT_TIMESTAMP();
```

### 不同会话时区对 TIMESTAMP 类型的影响

```sql
# UTC +8
set time_zone = 'Asia/Shanghai';

CREATE TABLE events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    event_name VARCHAR(255) NOT NULL,
    event_timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    event_datetime DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO events (event_name, event_timestamp, event_datetime) VALUES ('10.24 15:45:00', '2022-10-24 15:45:00', '2022-10-24 15:45:00');

INSERT INTO events (event_name, event_timestamp, event_datetime) VALUES ('12.24 15:45:00', '2022-12-24 15:45:00', '2022-12-24 15:45:00');
```

```sql
SELECT * FROM events;

+----+----------------+---------------------+---------------------+
| id | event_name     | event_timestamp     | event_datetime      |
+----+----------------+---------------------+---------------------+
|  1 | 10.24 15:45:00 | 2022-10-24 15:45:00 | 2022-10-24 15:45:00 |
|  2 | 12.24 15:45:00 | 2022-12-24 15:45:00 | 2022-12-24 15:45:00 |
+----+----------------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

```sql
# 仅修改当前会话的时区
set time_zone = 'America/New_York';
```

```sql
SELECT * FROM events;

+----+----------------+---------------------+---------------------+
| id | event_name     | event_timestamp     | event_datetime      |
+----+----------------+---------------------+---------------------+
|  1 | 10.24 15:45:00 | 2022-10-24 03:45:00 | 2022-10-24 15:45:00 |    <- 夏令时，相差 12 小时
|  2 | 12.24 15:45:00 | 2022-12-24 02:45:00 | 2022-12-24 15:45:00 |    <- 平时相差 13 小时
+----+----------------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

纽约 UTC 时差通常为 UTC-5（EST），夏令时为 UTC-4（EDT），所以将原本的会话从上海(UTC+8) 转到纽约时，`TIMESTAMP` 相差了 13 或 12(夏令时) 小时，所以为了自动转换夏令时，指定时区最好使用时区名词 `Asia/Shanghai`，避免使用偏移量：`'+08:00'`

## JDBC 连接 MySQL 时 serverTimezone 对于 TIMESTAMP 类型的影响

连接 MySQL 时我们使用 URL：`jdbc:mysql://192.168.1.2:3306/mydb?useSSL=false&serverTimezone=Asia/Shanghai`

这里的 `serverTimezone` 参数用于指定连接到 MySQL 数据库时所使用的时区，不显示指定使用 JVM 默认时区

MySQL 服务端处理 `TIMESTAMP` ：写入时根据会话时区转为 UTC 时间戳存储，读取时将 UTC 还原为会话时区的时间，保证了写入和读取数据的一致。数据库会话时区与 JVM 时区相同时，JVM 读写的 `TIMESTAMP` 一致，如果不一致就会出现问题，`serverTimezone` 就是为了告诉 JDBC 从 MySQL 服务端获取到的 `TIMESTAMP` 是什么时区，知道了它所使用的时区，JDBC 就可以进行预处理

MyBatis 在处理 `TIMESTAMP` 类型的数据时会有一些差异，实体映射为 `Timestamp` 或 `Date` 在读写时会进行上面提到的预处理，而 `LocalDateTime` 则不会

### JDBC 读取 TIMESTAMP 类型数据时

JDBC 执行命令时，调用不同的 ResultSet 方法会有不同结果：
- ResultSet 的 getString 方法：直接读取时间，即直接返回 **数据库根据会话时区转化后的时间**
- ResultSet 的 getTimestamp 方法：将 **数据库根据会话时区转化后的时间** 根据 serverTimezone 设置的时区进行转化，得到 **数据库根据会话时区转化后的时间** 对应的 UTC 时间毫秒戳，然后将这个 UTC 毫秒时间戳转换为 `Timestamp` 类型（它本身不包含时区信息），打印时会根据 JVM 的时区转化为对应的时区时间

`getTimestamp` 转化 `Timestamp` 的源码在：`com.mysql.cj.result.SqlTimestampValueFactory`


![](../MySQL时区与serverTimezone/1.png)


这里的 `this.connectionTimeZone` 就是连接 url 中指定的 `serverTimezone`

假设 MySQL 默认设置的会话时区为 `Asia/Shanghai`，通过默认会话读取该 TIMESTAMP 的值为：2022-10-24 15:45:00。而 MySQL 实际存储的 TIMESTAMP 为 UTC 时间：2022-10-24 07:45:00。MySQL JDBC 驱动通过默认会话获取该值时，MySQL 会自动根据默认时区提供转化好时间：2022-10-24 15:45:00，驱动则会根据 `serverTimezone` 配置的时区，将 MySQL 的时间转化为 `Calendar` 对象，通过 `c.getTimeInMillis()` 获取对应的 UTC 时间戳，用于创建 `Timestamp` 对象


### JDBC 写入 TIMESTAMP 类型：

- now()写入，数据库 server 端会获取数据库当前时区
- 按照字符串写入：MySQL 服务端根据会话时区转成对应的 UTC 毫秒数存储
- 通过变量绑定写入：传入 Timestamp 对象，JDBC 将其编码为 serverTimezone 所代表的时间字符串，类似：`2022-06-22 03:29:29`，然后发送给 MySQL 服务端

### 验证

```java
import org.junit.jupiter.api.Test;

import java.sql.*;
import java.util.TimeZone;

public class JDBCTest {
    private static String url = "jdbc:mysql://host:3306/mydb?useSSL=false&serverTimezone=UTC";
    private static String username = "root";
    private static String password = "";

    @Test
    void testInsertTimestamp() {
        TimeZone.setDefault(TimeZone.getTimeZone("Asia/Shanghai"));

        try (Connection connection = DriverManager.getConnection(url, username, password); PreparedStatement ps = connection
                .prepareStatement("insert into events(id,event_name,event_timestamp,event_datetime) values (1,'now()',now(),now())");) {
            ps.execute();
        }catch (Exception e){
            e.printStackTrace();
        }
        
        try (Connection connection = DriverManager.getConnection(url, username, password); PreparedStatement ps = connection
                .prepareStatement("insert into events(id,event_name,event_timestamp,event_datetime) values (2,'2022-06-22 03:29:29','2022-06-22 03:29:29', '2022-06-22 03:29:29')");) {
            ps.execute();
        }catch (Exception e){
            e.printStackTrace();
        }

        try (Connection connection = DriverManager.getConnection(url, username, password); PreparedStatement ps = connection
                .prepareStatement("insert into events(id,event_name,event_timestamp,event_datetime) values (3,'1733539800000L',?,?)")) {

            // Sat Dec 07 2024 02:50:00 GMT+0000
            // Sat Dec 07 2024 10:50:00 GMT+0800 (中国标准时间)
            long timestamp = 1733539800000L;
            Timestamp ts1 = new Timestamp(timestamp);
            Timestamp ts2 = new Timestamp(timestamp);

            ps.setTimestamp(1, ts1);
            ps.setTimestamp(2, ts2);
            ps.execute();
            // 根据 serverTimezone 将 Timestamp 预处理为 UTC 时间：2024-12-07 02:50:00
            // 相当于执行下列 SQL
            // insert into events(id,event_name,event_timestamp,event_datetime) values (3,'1733539800000L','2024-12-07 02:50:00','2024-12-07 02:50:00')
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    @Test
    void testGetTimestamp() {
        TimeZone.setDefault(TimeZone.getTimeZone("Asia/Shanghai"));

        try (Connection connection = DriverManager.getConnection(url, username, password); PreparedStatement ps = connection
                .prepareStatement("select * from events where id=3"); ResultSet rs = ps.executeQuery();) {
            while (rs.next()) {
                // getTimestamp is 2024-12-07 10:50:00.0
                // 根据 serverTimezone，认定数据库时区为 UTC，转化为 本地 Asia/Shanghai 需要 +8，则预处理为：2024-12-07 10:50:00.0
                System.out.println("getTimestamp is " + rs.getTimestamp("event_timestamp"));
                // getString is 2024-12-07 02:50:00
                System.out.println("getString is " + rs.getString("event_datetime"));
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

实验环境：
MySQL 8.0.40
mysql-connector-j 9.1.0
mybatis-spring-boot-starter 3.0.4

## 参考资料

[7.1.15 MySQL Server Time Zone Support](https://dev.mysql.com/doc/refman/8.4/en/time-zone-support.html)
[13.2.2 The DATE, DATETIME, and TIMESTAMP Types](https://dev.mysql.com/doc/refman/8.4/en/datetime.html)
[一文讲透MySQL driver读取时间时的时区处理](https://kaimingwan.com/2022/06/20/ns4y3a/)
[修改mysql时区的三种方法](https://www.cnblogs.com/jie-fang/p/10279439.html)
[MySQL 中存储时间的最佳实践](https://www.upyun.com/tech/article/652/MySQL%20%E4%B8%AD%E5%AD%98%E5%82%A8%E6%97%B6%E9%97%B4%E7%9A%84%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5.html)