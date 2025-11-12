---
date: '2025-11-12T21:58:06+08:00'
title: '一次由 DNS 反解析引发的 SpringBoot 启动卡顿问题'
categories: ["编程杂记"]
---

## TL;DR

1. 使用 114 DNS 时，反解析内网 IP 无响应，导致 SpringBoot 启动时 Liquibase 初始化阻塞约 30 秒
2. `InetAddress.getLocalHost()` 获取到的可能是`127.0.0.1`，而非实际的内网 IP（如 192.168.x.x）


## 现象

SpringBoot启动时（HikariPool 初始化后）卡顿 30s

```log
2025-11-09 15:14:33 INFO  [main] com.zaxxer.hikari.HikariDataSource - HikariPool-1 - Starting...
2025-11-09 15:14:33 INFO  [main] com.zaxxer.hikari.pool.HikariPool - HikariPool-1 - Added connection com.p6spy.engine.wrapper.ConnectionWrapper@7f6137fb
2025-11-09 15:14:33 INFO  [main] com.zaxxer.hikari.HikariDataSource - HikariPool-1 - Start completed.
```


## 排查

通过 jstack 分析线程栈，定位到 `liquibase.util.NetUtil.getLocalHostName()` 阻塞

```sh
# jps
jstack 66713
```

> 下方线程栈显示阻塞点位于 `Inet6AddressImpl.getHostByAddr()`

```log
"main" #1 prio=5 os_prio=31 cpu=3549.34ms elapsed=14.86s tid=0x000000010a808200 nid=0xd03 runnable  [0x000000016fa00000]
   java.lang.Thread.State: RUNNABLE
	at java.net.Inet6AddressImpl.getHostByAddr(java.base@17.0.14/Native Method)
	at java.net.InetAddress$PlatformNameService.getHostByAddr(java.base@17.0.14/InetAddress.java:940)
	at java.net.InetAddress.getHostFromNameService(java.base@17.0.14/InetAddress.java:662)
	at java.net.InetAddress.getHostName(java.base@17.0.14/InetAddress.java:605)
	at java.net.InetAddress.getHostName(java.base@17.0.14/InetAddress.java:577)
	at liquibase.util.NetUtil.getLocalHostName(NetUtil.java:79)
	at liquibase.sqlgenerator.core.LockDatabaseChangeLogGenerator.<clinit>(LockDatabaseChangeLogGenerator.java:30)
```

对应源码：

```java
public class LockDatabaseChangeLogGenerator extends AbstractSqlGenerator<LockDatabaseChangeLogStatement> {

    @Override
    public ValidationErrors validate(LockDatabaseChangeLogStatement statement, Database database, SqlGeneratorChain sqlGeneratorChain) {
        return new ValidationErrors();
    }

    protected static final String hostname;
    protected static final String hostaddress;
    protected static final String hostDescription = (System.getProperty("liquibase.hostDescription") == null) ? "" :
        ("#" + System.getProperty("liquibase.hostDescription"));

    static {
        try {
            // NetUtil.getLocalHostName() 导致阻塞
            hostname = NetUtil.getLocalHostName();
            hostaddress = NetUtil.getLocalHostAddress();
        } catch (Exception e) {
            throw new UnexpectedLiquibaseException(e);
        }
    }
    
    // ...
}
```


## 分析

`NetUtil.getLocalHostName()` 获取机器 IP：遍历网卡，调用 InetAddress.getHostName() 对内网 IP （192.168.10.2）做反解析，114 DNS（`114.114.114.114`）无响应，则导致阻塞 30s


```java
package liquibase.util;

// ...

public class NetUtil {
    // ...

    /**
     * @return Machine's host name. This method can be better to call than getting it off {@link #getLocalHost()} because sometimes the external address returned by that function does not have a useful hostname attached to it.
     * This function will make sure a good value is returned.
     */
    public static String getLocalHostName() {
        if (hostName == null ) {
            try {
                // 遍历所有网络接口，找出 已启用 的 非点对点 网络接口，然后打印这些接口上每个非本地地址对应的主机名（hostname）
                InetAddress localHost = getLocalHost();
                if(localHost != null) {
                    // 使用指定的 DNS 反解析获取的 IP
                    hostName = localHost.getHostName();
                    if (hostName.equals(localHost.getHostAddress())) {
                        //sometimes the external IP interface doesn't have a hostname associated with it but localhost always does
                        InetAddress lHost = InetAddress.getLocalHost();
                        if (lHost != null) {
                            hostName = lHost.getHostName();
                        }
                    }
                }
                else {
                    hostName = UNKNOWN_HOST_NAME;
                }
            } catch (Exception e) {
                Scope.getCurrentScope().getLog(NetUtil.class).fine("Error getting hostname", e);
                if (hostName == null) {
                    hostName = UNKNOWN_HOST_NAME;
                }
            }
        }
        return hostName;
    }
}
```


> 大多数公共 DNS（如阿里、Google）在无法解析内网地址时返回 NXDOMAIN，而 114 DNS 无响应，导致 Java 原生反解析方法阻塞

```sh
# 耗时：0.228s
nslookup 192.168.10.2 223.5.5.5
Server:		223.5.5.5
Address:	223.5.5.5#53

** server can't find 2.10.168.192.in-addr.arpa: NXDOMAIN
```

```sh
# 耗时：15.137s
nslookup 192.168.10.2 114.114.114.114
;; connection timed out; no servers could be reached
```


## 遍历网卡的原因

获取机器 IP 常用的方法：`InetAddress.getLocalHost()` 获取到的可能是`127.0.0.1`（与 JDK 实现有关），是一个本地回环地址（loopback address）。而非对外通信用的实际的网络 IP（例如 192.168.x.x 或 10.x.x.x）

为了获取 **实际的网络 IP**，一般使用类似上面 `NetUtil.getLocalHostName()` 的方式，遍历网卡获取实际 IP

参考：[JDK-4665037 : InetAddress.getLocalHost() ambiguous on Linux systems](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=4665037)

## 解决方案

最有效的解决方法是更换合适的 DNS。官方 issue 中有反馈类似的问题

参考：[Slow NetUtil.getLocalHostName in Windows WSL2 #2440](https://github.com/liquibase/liquibase/issues/2440)



## 参考资料
[oepnjdk Inet6AddressImpl.c](https://github.com/openjdk/jdk/blob/master/src/java.base/unix/native/libnet/Inet6AddressImpl.c#L693)

[liquibase NetUtil.java](https://github.com/liquibase/liquibase/blob/master/liquibase-standard/src/main/java/liquibase/util/NetUtil.java#L37)

