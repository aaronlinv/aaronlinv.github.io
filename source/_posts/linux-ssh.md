---
title: Linux-SSH
categories: Linux笔记
date: 2020-02-25 08:34:24
---

昨天在配置Hadoop的时候，使用SSH因为权限配置出现了一点问题
搜索发现，大家的权限配置都不太一致，所以自己开了两个虚拟机测试一下
如果对 SSH原理 不了解可以先看下这个 [图解SSH原理](https://www.cnblogs.com/diffx/p/9553587.html)

两台虚拟机A和B
B的ip：192.168.108.140
用户都为Hadoop

A 通过SSH 免密登录B
1. 在 虚拟机A Hadoop家目录下新建.ssh
2. cd .shh 使用 ssh-keygen -t rsa 生成公私钥 (运行 ssh-keygen -t rsa 后直接敲回车)

```bash
[hadoop@localhost ~]$ mkdir .ssh
[hadoop@localhost ~]$ cd .ssh/
[hadoop@localhost .ssh]$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/hadoop/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/hadoop/.ssh/id_rsa.
Your public key has been saved in /home/hadoop/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:SKDjW+zcm6hoGpDC4muNMZB5W8INPpv7KRZMN3SVR4w hadoop@localhost.localdomain
The key's randomart image is:
+---[RSA 2048]----+
|    .  ..=.      |
|  ..... E o      |
| =o+ ..  .       |
|=oBo=. .         |
|*=.Oo.. S        |
|=o*= .           |
|..*oo .          |
| *=. o o         |
|*o.++ o          |
+----[SHA256]-----+

```
3. 把公钥 id_rsa.pub 通过 scp 拷贝到 B虚拟机 的 /home/hadoop/  如果已存在会覆盖
4. 第一次连接会提示，输入 yes ，然后输入 B虚拟机 的Hadoop账户的密码，默认是用本机的用户名登录，需要登录其他用户的格式是：用户名@ip地址 如：root@192.168.108.140

```sh
[hadoop@localhost .ssh]$ ll
总用量 8
-rw-------. 1 hadoop hadoop 1675 2月  25 08:38 id_rsa
-rw-r--r--. 1 hadoop hadoop  410 2月  25 08:38 id_rsa.pub
[hadoop@localhost .ssh]$ scp id_rsa.pub 192.168.108.140:/home/hadoop/
The authenticity of host '192.168.108.140 (192.168.108.140)' can't be established.
ECDSA key fingerprint is SHA256:tFi+lLLHECXTtozqdB9y5ivkor28HNW6H9j5ooFiaAo.
ECDSA key fingerprint is MD5:3a:c4:5a:fc:fb:e6:e3:b9:f9:63:f3:0c:2e:f1:d9:51.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.108.140' (ECDSA) to the list of known hosts.
hadoop@192.168.108.140's password:
```
5. 切换到 B虚拟机 的Hadoop家目录 新建.ssh
6. 把 A虚拟机 的公钥 追加 .ssh/authorized_keys （不存在这个文件，会自动创建）

```sh
[hadoop@localhost ~]$ ls
id_rsa.pub
[hadoop@localhost ~]$ mkdir .ssh
[hadoop@localhost ~]$ cat id_rsa.pub >> .ssh/authorized_keys
```
这个时候切换到 A虚拟机 SSH登录B虚拟机
```sh
[hadoop@localhost .ssh]$ ssh 192.168.108.140
hadoop@192.168.108.140's password: 
```
发现还是需要密码，原因就是需要修改 虚拟机B 的权限
7. 修改 虚拟机B authorized_keys 的权限为644
```sh
[hadoop@localhost ~]$ cd .ssh/
[hadoop@localhost .ssh]$ ll
总用量 4
-rw-rw-r--. 1 hadoop hadoop 410 2月  25 08:42 authorized_keys
[hadoop@localhost .ssh]$ chmod 644 authorized_keys 
[hadoop@localhost .ssh]$ 
[hadoop@localhost .ssh]$ ll -al
总用量 4
drwxrwxr-x. 2 hadoop hadoop  29 2月  25 08:42 .
drwx------. 3 hadoop hadoop 113 2月  25 08:42 ..
-rw-r--r--. 1 hadoop hadoop 410 2月  25 08:42 authorized_keys
```
切换回 虚拟机A 使用SSH登录 B虚拟机 发现还是需要输入密码，因为 .ssh 权限也需要更改
8. 修改 虚拟机B .ssh 的权限为700

```sh
[hadoop@localhost ~]$ chmod 700 .ssh/
[hadoop@localhost ~]$ 
[hadoop@localhost ~]$ ll -al
总用量 20
drwx------. 3 hadoop hadoop  113 2月  25 08:42 .
drwxr-xr-x. 3 root   root     20 2月  25 00:12 ..
drwx------. 2 hadoop hadoop   29 2月  25 08:42 .ssh
```

切换回 虚拟机A 使用SSH登录 B虚拟机，成功
```sh
[hadoop@localhost .ssh]$ ssh 192.168.108.140
hadoop@192.168.108.140's password: 

[hadoop@localhost .ssh]$ ssh 192.168.108.140
Last login: Tue Feb 25 08:30:57 2020 from 192.168.108.1
[hadoop@localhost ~]$ exit
登出
Connection to 192.168.108.140 closed.
```

权限为什么要这么配
>因为sshd为了安全，对属主的目录和文件权限有所要求。如果权限不对，则ssh的免密码登陆不生效。
用户目录权限为 755 或者 700，就是不能是77x、777，需要保障other用户不能有w权限
.ssh目录权限一般为755或者700。
rsa_id.pub 及authorized_keys权限一般为644
rsa_id权限必须为600
--[ssh目录权限说明](https://blog.csdn.net/levy_cui/article/details/59524158)

因为一般情况 用户目录,rsa_id.pub,rsa_id 都符合权限要求，所以只要更改 .ssh 和authorized_keys即可

## 总结
1. 想免密登录谁，就把公钥给谁
2. 一般情况下，修改 authorized_keys 为 644，.ssh 为 700即可
3. 可以通过配置单机回环来验证SSH免密码登录是否成功
A 需要免密登录 B
可以先在B上生成公私钥，最佳生成的公钥到 authorized_keys ，配置权限
ssh localhost （自己登自己）不需要输入密码说明配置成功，记得通过 exit 退出当前的远程
然后就可以通过把 A 的公钥加入到 authorized_keys 中，使得 A 可以免密登录 B



## 参考资料
[ssh目录权限说明](https://blog.csdn.net/levy_cui/article/details/59524158)
[ssh密钥文件的权限](https://blog.51cto.com/merrycheng/1340280)
[图解SSH原理](https://www.cnblogs.com/diffx/p/9553587.html)
