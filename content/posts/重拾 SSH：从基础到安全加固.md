---
date: '2025-01-24T08:46:33+08:00'
title: '重拾 SSH：从基础到安全加固'
categories: ["重拾系列"]
---

安全外壳协议（Secure Shell Protocol，简称SSH）是一种加密的网络传输协议，属于应用层协议。[OpenSSH](https://www.openssh.com/) 是最流行的 SSH 实现，它是大量操作系统的默认组件

OpenSSH 套件由以下工具组成：
- 远程操作使用：**ssh**, **scp** 和 sftp 
- 密钥管理：ssh-add, ssh-keysign, ssh-keyscan 和 **ssh-keygen**
- 服务端： **sshd**, sftp-server 和 ssh-agent



## 使用 SSH 连接服务器

### 1. 客户端创建公私钥对


密钥类型选择 `ed25519` 椭圆曲线，它生成的公私钥都要比 `RSA` 更短，具有较高的安全性和性能

```sh
# - a KDF (Key Derivation Function) 的迭代次数 默认：16 ，防止暴力破解
# - t 类型
# Ubuntu 22.04 默认：RSA 3072；Mac OS 默认：ED25519 256

# - C 备注，可以备注上创建年月，定期更换私钥
ssh-keygen -a 256 -t ed25519 -C "Brandon+2025-01@MacBook"
# 可以手动指定路径和密码，也可以一路回车
```

在 `~/.ssh` 下会生成公私钥对

```sh
.
├── [ 411]  id_ed25519
├── [  98]  id_ed25519.pub

# 私钥需要妥善保管，避免暴露
cat id_ed25519

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACCRtC9cJJBFwvVsp4vV058ci8lSHNrf2qcx8W+umtK7OwAAAKArJx9PKycf...
-----END OPENSSH PRIVATE KEY-----

# .pub 结尾为公钥
cat id_ed25519.pub

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJG0L1wkkEXC9Wyni9XTnxyLIt/zHxb66a0rs7 Brandon+2025-01@MacBook
```


### 2. 在服务器上添加公钥

将上面客户端生成的公钥 `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJG0L1wkkEXC9Wyni9XTnxyLIt/zHxb66a0rs7 Brandon+2025-01@MacBook`
加入到服务端 `~/.ssh/authorized_keys`，每个私钥占据一行

也可以使用 `ssh-copy-id
```sh
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.64.6
```


### 3. 客户端 ssh 连接

```sh
# 登录的用户名@目标主机的 IP 地址
ssh root@192.168.16.13

# 首次连接某个服务器会提示
The authenticity of host '192.168.16.13 (192.168.16.13)' can't be established.
ED25519 key fingerprint is SHA256:QawKK4qYtzv/WyymFO64Yby5oxo9bVYZu0TQRvLZsL8.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

这是需要要求你通过 **服务端公钥指纹** 来验证服务端的身份，避免中间人攻击。在确认指纹后输入 `yes` 完成连接，ssh 会将公钥信息写入到 `~/.ssh/known_hosts` 中。后续连接如果服务端指纹变更，就说明可能出现了中间人攻击，ssh 会在连接时提示指纹不一致

#### 获取服务端公钥指纹

在 **服务端** `/etc/ssh/` 下有多对公私钥：

```sh
├── [1.3K]  ssh_host_dsa_key
├── [ 609]  ssh_host_dsa_key.pub
├── [ 513]  ssh_host_ecdsa_key
├── [ 181]  ssh_host_ecdsa_key.pub
├── [ 411]  ssh_host_ed25519_key
├── [ 101]  ssh_host_ed25519_key.pub
├── [2.5K]  ssh_host_rsa_key
├── [ 573]  ssh_host_rsa_key.pub
```

以 `ed25519` 为例，查看公钥指纹：

```sh
# -l：列出密钥的指纹
# -f：后面跟要查看的文件路径
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:WykOLKFPEwaC42OM8B5EgFBS5RlgV4qvXxkCIxPE6h4 root@VM-12-5-ubuntu (ED25519)
```

如果无法登陆服务器，可以直接获取，相对来说还是找管理员索要指纹更加安全

```sh
ssh-keyscan -t ed25519 -p 22 192.168.16.13 2>/dev/null | ssh-keygen -E sha256 -lf -
256 SHA256:WykOLKFPEwaC42OM8B5EgFBS5RlgV4qvXxkCIxPE6h4 root@VM-12-5-ubuntu (ED25519)
```

关于 ssh 还有一个常见用法，就是测试 GitHub 连通性

```sh
# 测试 GitHub 连接
# -T: 禁用伪终端分配

ssh -T git@github.com

# 回应：
# Hi someone! You've successfully authenticated, but GitHub does not provide shell access.
```

## 连接过程

加密连接最重要的就是解决 **密钥传递** 的问题，如果传递密钥时密钥被窃取，那么后续的加密将毫无意义

为了避免传递时泄露密钥，ssh 采用了 **非对称加密算法**，即公私钥的这种方式。服务器和客户端在 **确认对方的公钥是安全的** 之后，就可以通过对方的公钥加密数据，对方通过私钥解密数据，实现安全地传输数据。客户端与服务器连接的具体过程：


1. 建立 TCP 连接
2. 协商 **SSH 版本**：SSH1 或 SSH2
3. 协商使用的 **算法**：加密、密钥交换、消息认证码等
4. 用协商好的密钥交换算法（例如 Elliptic Curve Diffie-Hellman）**生成共享密钥**，通过它进行回话的对称加密5
5. 客户端验证服务器身份: 为了防止中间人攻击，比较服务器的公钥指纹或密钥与客户端已知的主机是否一致, 不一致或者首次连接会提示用户验证服务器的公钥指纹
6. 服务器验证用户身份: 服务器验证客户端的身份
7. 会话建立: 身份验证成功后，客户端和服务器之间建立加密的会话



[Elliptic Curve Diffie-Hellman](https://zh.wikipedia.org/wiki/%E6%A9%A2%E5%9C%93%E6%9B%B2%E7%B7%9A%E8%BF%AA%E8%8F%B2-%E8%B5%AB%E7%88%BE%E6%9B%BC%E9%87%91%E9%91%B0%E4%BA%A4%E6%8F%9B)：
双方分别生成一对临时 **公私钥对**，将自己的 **临时公钥** 发送给对方，双方都可以通过 对方临时公钥 + 自己临时私钥 生成 **共享密钥**，这个共享密钥双方相同。**共享密钥** 作为 KDF 的输入，生成多个密钥，用于加密、完整性校验

交换哈希 (Exchange Hash)：为了确保密钥交换过程没有被篡改，SSH 会计算一个交换哈希值，服务器会对交换哈希进行签名（这里用的是服务器私钥，即 `/etc/ssh/` 下的私钥）。哈希值包含了密钥交换过程中交换的所有数据，包括客户端和服务器版本、算法协商结果、密钥交换参数等

ssh 连接过程的数据包：

![](../重拾SSH：从基础到安全加固/1929786-20250123170050410-1584713012.png)

从流程可以看出，ssh 虽然可以保证会话传输安全，但是无法保证对方身份的真实性，所以在连接时要 **确认公钥指纹**

## 常见问题

### 回话保持

删除 `authorized_keys` 中的 key，该 key 已打开的会话不会被断开

可以通过 pkill 终止特定用户的 SSH 会话：

```sh
pkill -u $username sshd
```

### 权限导致登陆失败

在 `man ssh` 中：
>  ~/.ssh/: the recommended permissions are read/write/execute for the user, and not accessible by others.
>  ~/.ssh/authorized_keys: the recommended permissions are read/write for the user, and not accessible by others.

保证 `~/.ssh` `authorized_keys` 只有 **所有者** 可以访问修改


如果我们修改权限：`chmod 666 authorized_keys`，这样登陆会失败：

```sh
shell failed: ssh failed to authenticate: 'Access denied for 'publickey'. Authentication that can continue: publickey'
```

在 sshd 的日志中也可以看到原因：

```sh
tail -f /var/log/auth.log

2025-01-17T11:36:41.734631+08:00 iptables sshd[1294]: Authentication refused: bad ownership or modes for file /home/ubuntu/.ssh/authorized_keys
```

### .ssh 子目录中的 config

通过 config 简化 ssh 连接

```config
Host aliyun
    HostName 192.168.1.100
    User tom
    IdentityFile ~/.ssh/id_rsa
    Port 22
```

```sh
# ssh $Host 快速连接
ssh aliyun
```


## 安全防护

### 查看连接

```sh
# who 查看当前登录用户的方法。它会显示用户名、终端、登录时间以及远程主机（如果是 SSH 连接）
who

ubuntu   pts/0        2025-01-20 17:43 (192.168.64.1)
```

```sh
# w 命令提供更详细的信息，包括当前登录用户、他们正在运行的进程、系统负载等。
w
 19:44:32 up 14:48,  2 users,  load average: 0.06, 0.04, 0.00

USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU  WHAT
ubuntu            192.168.64.1     17:43    3days  0.00s  0.05s sshd: ubuntu [priv]
ubuntu            192.168.64.1     17:43    3days  0.00s   ?    sshd: ubuntu [priv]
```

```sh
# last 命令显示最近登录的用户列表，包括用户名、终端、登录时间、远程主机和登出时间
last

ubuntu   pts/0        192.168.64.1     Mon Jan 20 17:43   still logged in
ubuntu   pts/0        192.168.64.1     Fri Jan 17 11:33 - 11:47  (00:13)
reboot   system boot  6.8.0-51-generic Fri Jan 17 11:33   still running
reboot   system boot  6.8.0-51-generic Thu Jan  9 16:58 - 17:11  (00:13)
ubuntu   pts/0        192.168.64.1     Thu Jan  9 16:57 - 16:58  (00:00)
```

```sh
# lastlog 命令显示每个用户最后一次登录的时间
lastlog

Username         Port     From                                       Latest
root                                                                **Never logged in**
daemon                                                              **Never logged in**
bin                                                                 **Never logged in**
sys                                                                 **Never logged in**
ubuntu           pts/0    192.168.64.1                              Mon Jan 20 17:43:58 +0800 2025
```


### 安全策略

#### pkill 会话
```sh
ps ajfx

   PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
1 1621867 1621867 1621867 ?             -1 Ss       0   0:49 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 sta
1621867 2226258 2226258 2226258 ?             -1 Ss       0   0:00  \_ sshd: root@pts/5
2226258 2226414 2226414 2226414 pts/5    2227480 Ss       0   0:00      \_ bash --rcfile /dev/fd/63
2226414 2227480 2227480 2226414 pts/5    2227480 R+       0   0:00          \_ ps ajfx
```

通过 SID 可以 kill 掉整个 session，如果 pikill `tmux` 等软件创建的 session，这些 session 的进程还是会继续运行

```sh
# -e: 显示进程将被终止
# -s: 指定一个或多个会话 ID
# pkill -e -s  $SID
pkill -e -s 2226258
```

### 排查用户和密钥

查看非 root 可登录的账号

```bash
awk -F: 'BEGIN {OFS=":"} $3 != 0 && $7 !~ /\/bin\/false|\/nologin/ {print $1}' /etc/passwd
```

查看允许 ssh 登陆的公钥：

**`cat ~/.ssh/authorized_keys`**


### 禁用密码登陆

```sh
grep PasswordAuthentication /etc/ssh/sshd_config
```

注意：这里是 `sshd_config`，而不是少了 `d` 的 `ssh_config`

如果 `/etc/ssh/sshd_config.d` 中的某个文件中定义了某个配置选项，那么它会覆盖 `/etc/ssh/sshd_config` 文件中相同的配置

例如在 `/etc/ssh/sshd_config.d/50-cloud-init.conf` 中配置了 `PasswordAuthentication yes`，那么 `/etc/ssh/sshd_config` 中修改 `PasswordAuthentication` 的值都会被覆盖，即密码登陆一直有效，修改为 `PasswordAuthentication no` 后重新 sshd：`sudo systemctl restart sshd`，就会提示只允许密钥登陆：


```sh
ssh root@192.168.64.6

root@192.168.64.6: Permission denied (publickey).
```

如果没有关闭密码登陆，就可能会被暴力破解，查询不同用户被尝试登陆的次数：

```bash
grep "Failed password" /var/log/auth.log|perl -e 'while($_=<>){ /for(.*?)from/; print "$1\n";}'|sort|uniq -c|sort -nr
```

```bash
   5611  root 
   2013  invalid user hadoop 
   1975  invalid user inspur 
   1973  invalid user cloud 
    285  invalid user roo 
    101  invalid user test 
     84  invalid user user 
     72  invalid user admin 
     68  invalid user oracle 
     61  invalid user postgres 
     61  invalid user git 
     48  invalid user ubuntu 
     31  invalid user test1 
     28  invalid user amandabackup 
     24  invalid user pcpqa 
     22  invalid user ftpuser 
     21  invalid user test2 
     21  invalid user chenly 
```

查询登陆成功的 IP

```bash
grep "Accepted" /var/log/auth.log | awk '{print $11}' | sort | uniq
```

其他安全配置可以参考：

[Linux 应急响应手册 v1.9 - NOP Team](https://github.com/Just-Hack-For-Fun/Linux-INCIDENT-RESPONSE-COOKBOOK)
[如何配置安全的 SSH 服务](https://learnku.com/server/t/36120)
[SSH安全加固指南](https://www.freebuf.com/articles/system/246994.html)

## 登陆告警

在用户登陆时向钉钉发送告警，参考自：[VPS 安全加固之用户登录后向 telegram 发送登录信息](https://blog.k8s.li/linux-login-alarm-telegram.html)


```sh
cd /etc/profile.d/
vim ssh-login-alerm.sh
```

```sh
#!/bin/bash

# 钉钉机器人 Webhook URL
dingtalk_webhook="https://oapi.dingtalk.com/robot/send?access_token=$token" 

# 获取系统信息，并进行转义以避免 JSON 解析错误
message=$(hostname && TZ=UTC-8 date '+%Y-%m-%d %H:%M:%S' && who && w | awk 'BEGIN{OFS="\t"}{print $1,$8}' | sed 's/\\/\\\\/g;s/\n/\\n/g;s/"/\\"/g')

# 使用 curl 发送消息到钉钉机器人
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"ssh: ${message}\"}}" \
  "${dingtalk_webhook}" || echo "Error sending DingTalk message."

#可选：添加日志记录
echo "$(date '+%Y-%m-%d %H:%M:%S') - SSH login detected. Message sent to DingTalk (or error occurred)." >> /var/log/ssh_login_dingtalk.log
```

```sh
# 权限设置为 555 ，这样登陆任何用户都会执行告警
chmod 555 /etc/profile.d/ssh-login-alerm.sh
```

登陆后钉钉会收到一条消息:

```
ssh: localhost
2025-01-23 22:17:16
root     pts/0        2025-01-23 22:17 (192.168.64.5)
22:17:16	load
USER	WHAT
root	bash
```


## 参考资料

[Secure Shell - wiki](https://zh.wikipedia.org/wiki/Secure_Shell)
[OpenSSH - wiki](https://zh.wikipedia.org/wiki/OpenSSH)
[SSH-Keygen的更安全用法](https://luciochen.com/archives/189)
[It’s 2023. You Should Be Using an Ed25519 SSH Key (And Other Current Best Practices)](https://www.brandonchecketts.com/archives/its-2023-you-should-be-using-an-ed25519-ssh-key-and-other-current-best-practices)
[验证远程主机SSH指纹](https://www.cnblogs.com/LiuYanYGZ/p/14803012.html)
[什么是SSH？](https://info.support.huawei.com/info-finder/encyclopedia/zh/SSH.html)
[图解SSH原理](https://www.cnblogs.com/diffx/p/9553587.html)
[SSH协议解析及wireshark抓包分析](https://blog.csdn.net/chen1415886044/article/details/118650286)
[SSH协议握手核心过程](https://www.bilibili.com/video/BV13P4y1o76u)