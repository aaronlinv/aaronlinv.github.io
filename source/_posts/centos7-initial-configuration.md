---
title: CentOS7 初次配置
categories: Linux笔记
date: 2020-08-10 15:32:12
---
## 基本配置（使用root账户）

1. 使用静态IP
使用vi编辑器，编辑配置文件，vi编辑器的逻辑和其他编辑器不太一样，第一次使用推荐先学习一下 [Vi 极简入门](https://segmentfault.com/a/1190000008929397)
```
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```
编辑ifcfg-ens33文件
```
# 修改以下内容
BOOTPROTO=static           # 启用静态IP地址
ONBOOT=yes                 # 开启自动启用网络连接

# 添加以下内容(要结合自己实际的ip情况配置)
IPADDR=192.168.43.137      # 设置IP地址
NETMASK=255.255.255.0      # 子网掩码
GATEWAY=192.168.43.1       # 设置网关
```

重启网卡，使配置生效
```
# 不要在Xshell中执行，可能会断开
systemctl restart network
```

2. 更改镜像源

```
# 安装wget（Linux系统中的一个下载文件的工具）
yum install -y wget
       
# 下载Aliyun的repo
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 更新镜像源
yum clean all
yum makecache
```

3. 安装Vim编辑器（Vi的进阶版）

```
yum install -y vim 
```

4. 防火墙
关闭防火墙会简化一些软件的配置过程，但是不安全，生产环境谨慎关闭防火墙，可以通过开放指定端口实现互通
```
# 关闭防火墙
systemctl stop firewalld

# 禁止开机启动
systemctl disable firewalld
```
5. 更新或升级CentOS
这两个命令可能都会升级内核版本，生产环境要注意
```
yum update && yum upgrade
```
6. 添加普通用户到sudo中
解决普通用户使用sudo时提示 “xxx 不在 sudoers 文件中。此事将被报告” (xxx is not in the sudoers file)
```
# 在root用户下
visudo
```
在100行左右添加模仿root账户写法，添加
```
# hadoop为需要指定的账户名
hadoop  ALL=(ALL)       ALL
```

![visudo](centos7-initial-configuration/01.png)
## 参考
[Centos7.7安装及配置教程](https://juejin.im/post/6844904101583519752)
[安装CentOS后的基本配置](https://blog.csdn.net/chengyuqiang/article/details/55044073)
[linux下”is not in the sudoers file“问题的解决办法](https://blog.csdn.net/summy_J/article/details/72846076?utm_source=distribute.pc_relevant.none-task)