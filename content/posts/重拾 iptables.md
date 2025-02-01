---
date: '2024-12-27T08:31:33+08:00'
title: '重拾 iptables'
categories: ["重拾系列"]
---

iptables 是一个常看常忘的命令，本文试图从应用的角度理解它

iptables 是运行在用户空间的应用软件，通过控制 Linux 内核 netfilter 模块，来管理网络数据包的处理和转发

## 一些常用的场景

### 1. 禁止 ip 访问后端 IP

在 `192.168.64.6` 上增加规则：

```sh
# -A INPUT: 将规则添加到 INPUT 链，表示处理进入的流量
# -s 192.168.64.7: 指定源 IP 地址，即要阻止的 IP
# -d 192.168.64.6: 指定目标 IP 地址，即后端 IP
# -j DROP: 表示丢弃匹配的流量
iptables -A INPUT -s 192.168.64.7 -d 192.168.64.6 -j DROP

# -j REJECT: 丢弃流量的同时向源 IP 返回一个拒绝消息。请求方直接提示：Connection refused
iptables -A INPUT -s 192.168.64.7 -d 192.168.64.6 -j REJECT

# -p 指定协议类型为 TCP
# --dport 指定目标端口
iptables -A INPUT -s 192.168.64.7 -d 192.168.64.6 -p tcp --dport 80 -j REJECT

# 看当前的 iptables 规则
# -L "list"，列出当前的规则
# -n "numeric"，即使用数字 IP 地址和端口号而不是主机名和服务名
# -v "verbose"，显示详细信息
iptables -L -n -v
```

```sh
# 列出带编号的规则
iptables -L --line-numbers
# 删除 INPUT 链中的第 1 条规则
# 注意！删除成功后序号会改变，需要重新查询序号
iptables -D INPUT 1
# 清除 INPUT 链所有规则
iptables -F INPUT  

# 清除当前活跃的表（未指定默认是 filter 表）的所有 iptables 规则
# 等同于 iptables -F -t filter
iptables -F
```

### 2. 端口转发

默认情况下，Linux 系统不会转发目的 IP 地址不是本地网络的 IPv4 数据包。这是出于安全考虑，防止系统意外成为恶意流量的转发表。要启用 IPv4 数据包转发功能，需要修改内核参数 `net.ipv4.ip_forward`

需要注意，上面的命令仅临时启用 IPv4 数据包。需要永久启用转发，需要修改 `/etc/sysctl.conf` 文件。 在该文件中添加或修改 `net.ipv4.ip_forward=1` 一行。 然后运行 `sudo sysctl -p` 应用更改

```sh
cat /proc/sys/net/ipv4/ip_forward
sudo sysctl -w net.ipv4.ip_forward=1
```

#### 将本机的 8080 端口转发到 80 端口

```sh
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port 80

# curl 127.0.0.1:8080
# curl 192.168.64.6:8080
# curl: (7) Failed to connect to 127.0.0.1 port 8080 after 1 ms: Couldn't connect to server

# 非本机访问 ok
# curl 192.168.64.6:8080
```

PREROUTING 链修改的是从外部连接过来时的转发，所以上面的方式本机 `curl 127.0.0.1:8080` 会提示：Couldn't connect to server

如果本机连接到本机的转发，需要修改为 OUTPUT 链：

```sh
# 清除已有 nat 规则
# iptables -F -t nat
iptables -t nat -A OUTPUT -p tcp --dport 8080 -j REDIRECT --to-port 80

# 非本机访问失败：
# curl 192.168.64.6:8080
# curl: (7) Failed to connect to 192.168.64.6 port 8080 after 1 ms: Connection refused

# 本机访问 ok
# curl 192.168.64.6:8080
# curl 127.0.0.1:8080
```

#### 转发内网 IP
在 `192.168.64.6` 上增加规则：

```sh
# 清除已有 nat 规则
# iptables -F -t nat

iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.64.7:80
iptables -t nat -A POSTROUTING -p tcp -d 192.168.64.7 --dport 80 -j SNAT --to-source 192.168.64.6
```

#### 转发公网的 IP 和端口

```sh
# 清除已有 nat 规则
# iptables -F -t nat

iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 106.55.113.110:80
# --to-source 使用公网地址会无法访问
iptables -t nat -A POSTROUTING -p tcp -d 106.55.113.110 --dport 80 -j SNAT --to-source 192.168.64.6
```

需要注意的是 `SNAT` 的 `--to-source` 需要设置为连接公网的网卡对应的内网 IP（通过 ip ad 查询），如果设置为公网 IP，数据包可能被丢弃

### 3. 内部 IP 共享上网

NAT 目的是为了解决 IPv4 公网 IP 不足的问题：
- 当私有网络中的设备发送数据包到公共网络时，NAT 设备会将数据包的 源IP地址（内网 IP）从私有地址转换为 公共IP地址，并维护一个转换表，记录所有的地址转换关系
- NAT 设备接收数据包时，会根据转换表将数据包的目标 IP 地址转换为内部设备的内网 IP，并将其发送到内部网络

包含的操作：
- SNAT (Source Network Address Translation)：修改数据包的源 IP 地址
- DNAT (Destination Network Address Translation)：修改数据包的目的 IP 地址
- MASQUERADE：和 SNAT 类似，但是对每个包都会动态获取指定输出接口（网卡）的 IP，因此如果接口的 IP 地址发送了变化，MASQUERADE 规则不受影响

举个 NAT 的例子：村民张三需要写信给河南的李四，写完后他在信封上写上，寄件人地址：`勤劳村 8 号`，收件人地址：`河南省孟津县陈倪路 20 号`。然后就把这封信投递到村里的邮局。邮递员拿到信件一看，这信要是寄出去，收件人通过 `勤劳村 8 号` 这个回信肯定没办法寄回村里，于是就将信封上寄件人地址修改为：`四川省兴文县勤劳村邮局`，再将信件发出，同时在本子上记录发往河南的这封信对应的是 `勤劳村 8 号`。 李四收到信件就按照信件上的信息编写信封，寄件人地址：`河南省孟津县陈倪路 20 号`，收件人地址：`四川省兴文县勤劳村邮局`，这样这封回信就寄到了村里的邮局，邮递员一看到这封信是来自河南，对着笔记本就知道这封信是送往 `勤劳村 8 号`，于是将收件人地址修改为了 `勤劳村 8 号`，这样邮递员派件的时候就可以把回信送到张三家

详细 NAT 原理可以参考这篇文章：[[译] NAT 穿透是如何工作的：技术原理及企业级实践](https://arthurchiao.art/blog/how-nat-traversal-works-zh/)

#### 实践

我是按着 [使用iptables将ubuntu配置为路由器](https://zu1k.com/posts/linux/ubuntu-iptables-nat/) 进行操作，最后的效果：客户端可以通过连接一台配置了 SNAT 或者 MASQUERADE 的机器访问公网


![](../重拾iptables/1.png)


注意：给网关和客户端指定 `vmnet15` 后还需要手动配置一下 `虚拟网络`：

![](../重拾iptables/2.png)

网关 IP 配置：

```yaml
network:
  version: 2
  ethernets:
    ens33: # WAN 接口
      dhcp4: true
    ens34: # LAN 接口
      dhcp4: no
      addresses: [10.1.2.1/24]
```

客户端配置：

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [10.1.2.2/24]
      gateway4: 10.1.2.1 # 网关
      nameservers:
        addresses: [114.114.114.114]
```

在网关机器上进行 iptables 的配置：

```sh
sudo sysctl -w net.ipv4.ip_forward=1
```

```sh
# 将 10.1.2.0/24 网络中的所有主机伪装成 192.168.184.131 这个公网IP地址，以便它们可以访问外部网络
# -o ens33: 匹配通过 ens33 接口出站的数据包
iptables -t nat -A POSTROUTING -s 10.1.2.0/24 -o ens33 -j SNAT --to-source 192.168.184.131
```

这条命令允许内部网络的设备通过网关访问互联网。内部设备的 IP 地址在数据包离开网关时被替换为网关的公共 IP 地址，从而使外部网络只看到网关的 IP 地址，除了 SNAT 还可以使用 MASQUERADE，二者效果类似

```sh
iptables -t nat -A POSTROUTING -s 10.1.2.0/24 -j MASQUERADE
```

## 理解 iptables

表:

1. filter 表: 这是默认表，用于过滤数据包，决定是否允许数据包通过
2. nat 表: 用于网络地址转换 (Network Address Translation, NAT)。它主要用于修改数据包的源地址或目标地址，例如将私有 IP 地址转换为公有 IP 地址
3. mangle 表: 用于修改数据包的头部信息，例如修改 TTL (Time To Live) 值、TOS (Type of Service) 值等
4. raw 表: 用于在连接跟踪之前处理数据包，主要用于控制连接跟踪是否启用
5. security 表: (较新版本) 用于安全策略的实施，例如 SELinux

链：

1. PREROUTING
2. INPUT
3. FORWARD
4. OUTPUT
5. POSTROUTING

数据包的不同场景：
- 收到的、目的是本机的包：PRETOUTING -> INPUT
- 收到的、目的是其他主机的包：PRETOUTING -> FORWARD -> POSTROUTING
- 本地产生的包：OUTPUT -> POSTROUTING

[](../重拾iptables/3.png)

图片来自：[从零开始认识 iptables](https://morven.life/posts/iptables-wiki/)


表包含若干个链：

1. filter 表包含三个链：INPUT, FORWARD, OUTPUT
2. nat 表包含三个链：PREROUTING, POSTROUTING, OUTPUT
3. mangle 表五个链都包含
4. raw 表包含两个链：PREROUTING, OUTPUT

看到这些排列组合，可能已经凌乱了，可以看下面的这张图，`conntrack` 理解为 `raw` 表，来自：[Netfilter Kernel (Packet) Traversal](http://linux-ip.net/pages/diagrams.html#netfilter-kernel-packet-traversal)

[](../重拾iptables/4.png)

Netfilter 内核数据包遍历就像保卫萝卜（塔防游戏）一样，数据包就像游戏中的怪物，会按照特定的路径移动，链就像在特定位置安置的炮塔，**当数据包经过某个链时，链就会对数据包进行一些操作**，链中包含若干条规则

既然已经有了链，可以在数据包的不同阶段执行特定操作，为什么还需要表呢？原因是不同规则的执行顺序可能会影响结果。比如，有两条规则：
1.	对数据包执行 SNAT
2.	对数据包的源 IP 进行过滤

如果先执行 SNAT，过滤操作会基于 SNAT 修改后的 IP 和端口进行匹配。但如果先执行过滤，数据包可能在 SNAT 应用之前就被过滤掉了。每个表的操作结果都会影响后续表的处理，所以 **表的作用是将规则按照功能进行分类，避免执行顺序导致规则失效**。表的处理顺序：`raw -> mangle -> nat -> filter`

注意：**相同表中相同链中如果多个规则匹配同一个数据包，则只有第一个匹配的规则会被执行**

需要注意的是，在使用 iptables 命令时，如果没有指定表，默认表是 filter（最后处理的那个表）

```sh
# 手动指定 -t 为 nat 表
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port 80
```

## 持久化

iptables 规则存储在内存中，**系统重启后规则会丢失**，可以安装 `iptables-persistent` 来持久化规则，规则保存在 `/etc/iptables/rules.v4`

```sh
sudo apt update
sudo apt install iptables-persistent
```

配置好 iptables 规则后，需要手动运行以下命令持久化规则：

```sh
sudo netfilter-persistent save
```

误改规则，通过该命令将规则恢复到持久化存储的状态：
```sh
sudo netfilter-persistent reload
```

## 参考资料

[iptables - wiki](https://zh.wikipedia.org/wiki/Iptables)
[iptables的四表五链与NAT工作原理](https://tinychen.com/20200414-iptables-principle-introduction/)
[iptables做TCP/UDP端口转发【转】](https://www.cnblogs.com/paul8339/p/14688156.html)
[通过iptables实现端口转发和内网共享上网](https://xstarcd.github.io/wiki/Linux/iptables_forward_internetshare.html)
[iptables error: unknown option --dport](https://serverfault.com/a/563036)
[How iptables tables and chains are traversed](https://unix.stackexchange.com/a/189906)
[[译] NAT - 网络地址转换（2016）](https://arthurchiao.art/blog/nat-zh/)
[[译] 深入理解 iptables 和 netfilter 架构](https://arthurchiao.art/blog/deep-dive-into-iptables-and-netfilter-arch-zh/)
[VMware实现iptables NAT及端口映射](https://cloud.tencent.com/developer/article/1718100)