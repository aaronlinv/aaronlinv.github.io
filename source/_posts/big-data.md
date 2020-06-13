---
title: Hadoop大数据平台搭建
date: 2019-11-08 12:16:40
categories: 大数据笔记
---
## 背景
在网上看到的一张图片，很真实的描述了学习大数据前后对大数据的看法 


![大数据](bigData/1.jpg)

 
## 一些概念
- 大数据

维基的解释是：在传统数据处理应用软件不足以处理的大或复杂的数据集

大数据有四个特点(4个" V ")：数据量大(Volume)、数据类型繁多(Variety)、处理速度快(Velocity)、价值密度低(Value)

我自己的理解就是在以前对很多数据没有办法进行筛选、存储、分析等操作，但随着技术的发展，现在有很多技术可以完成这些工作，从而使得我们可以得到很多蕴含在这些数据之间的信息。例如网购的订单，最开始电商平台可能只能存储这些订单信息，还没有进行分析，随着技术的发展，就可以分析这些订单之间的联系，从而实现针对性的商品推荐等功能
- 云计算   

通过网络提供可伸缩的、廉价的分布式计算能力。云计算使得计算资源像自来水一样，可以方便取用。有这几种服务类型

基础设施即服务 IaaS (Infrastructure as a Service) 提供了基础设施，CPU、内存、网络、存储等，客户需要自己配置环境，然后运行自己的服务
平台即服务 PasS (Platform as a Service) 提供了服务器平台或者开发环境，客户可以在这个平台上部署自己的服务
软件即服务 SaaS (Software as a Service) 提供的就是软件成品，客户可以按需购买，就像购买一些邮件服务，即开即用，不需要关心它运行在什么平台或者是运行在什么硬件上
-  物联网

利用通信技术把传感器、机器、控制器、人员、物连接在一起，实现彼此间的通信、收集和传输数据。最常见的应用就是一些智能可穿戴设备、智能家居、智能交通
- 相互的联系 

大数据、云计算、物联网之间既有区别也有联系
云计算为大数据提供了技术基础
物联网是大数据的重要来源

## Hadoop开源分布计算式计算平台
Hadoop是Apache软件基金会旗下的一个开源分布计算式计算平台，Hadoop的核心是HDFS(Hadoop Distributed File System)和MapReduce

HDFS是面向普通硬件环境的分布式文件系统，MapReduce是开发并行应用的程序

## Hadoop安装
我是参考厦门大学林子雨老师的[大数据处理架构Hadoop安装教程](http://dblab.xmu.edu.cn/blog/285/)，这个教程提供了多种不同安装方式的详细指导

我选择的是VMware虚拟机中安装Ubuntu，伪分布式安装Hadoop2.7.1，建议使用这个教程提供的系统和各种软件，遇到需要输入或者起名的步骤尽量和教程保持一致，这样可以规避一些奇奇怪怪的问题。在学习一个技术的初期，如果因为软件配置、兼容性等问题而浪费大量时间会非常闹心

解压Hadoop后，如果是是单机模式，无需配置，直接开始使用就好了。如果选择伪分布式，需要配置core-site.xml和hdfs-site.xml即可，需要特别注意的是：配置 hadoop.tmp.dir 参数，则默认使用的临时目录为 /tmp/hadoo-hadoop，而这个目录在重启时有可能被系统清理掉，导致必须重新执行 format 才行。同时也指定 dfs.namenode.name.dir 和dfs.datanode.data.dir 避免出错。

配置好后就要格式化NameNode(在Hadoop目录下)
``` bash
./bin/hdfs namenode -format #格式化NameNode
```

![Namenode format](bigData/2.png)

看到 successfully formatted 和 Exitting with status 0 就表示格式化成功，然后可以启动所有进程
``` bash
./sbin/start-all.sh #启动所有进程
```
![star-all.sh](bigData/3.png)

进程启动后可以用jps来查看所有的Java进程，
``` bash
jps
```
如果没有这些进程，可以尝试先停止所有进程，然后再启动
``` bash
./sbin/start-all.sh #先执行关闭所有进程

./sbin/stop-all.sh #所有进程都关闭后，再启动所有进程
```

## HDFS Shell命令
HDFS是一个分布式文件系统，所以我们可以对HDFS中的文件进行上传、下载、复制等文件操作。可以用shell命名完成这些操作

对于文件操作这个功能，HDFS有三种命令标记，如果用来操作HDFS这三中貌似用起来是一模一样的
- hadoop fs  适用于本地系统和HDFS系统
- hadoop dfs 只能操作HDFS文件系统，已经Deprecated(不推荐使用)，推荐使用第三个hdfs dfs
- hdfs dfs 只能操作HDFS文件系统

实际上很多命令和Linux的shell差不多，不过要注意格式，用ls(显示目录下的文件信息)命令举例
``` bash
hdfs dfs -ls / #这里用hdfs dfs 用其他两个效果是一样的
```
> 命令前面一定要加- 如：-ls

> 命令之后一定要指定目录 如：/     (左斜杆就是代表HDFS目录的根)

例举一些常用的命令
``` bash
hdfs dfs -ls / #ls 查看更目录下的文件信息

hdfs dfs –mkdir –p /user/hadoop #mkdir 新建文件夹 -p 递归操作

hdfs dfs –rm –r /input #rm 删除目录 -r 递归删除

hdfs dfs -cp /myLocalFile.txt /user/hadoop/ #cp 在HDFS复制文件 复制根目录下的myLocalFile.txt到/user/hadoop/

hdfs dfs –cat /myLocalFile.txt # cat 查看文件内容 

hdfs dfs -put /home/hadoop/myLocalFile.txt  / #put 上传文件到HDFS 将本地的myLocalFile.txt文件上传到HDFS的根目录下
hdfs dfs -put /home/hadoop/myLocalFile.txt  /test.txt #将本地的myLocalFile.txt文件上传到HDFS的根目录下,并且重命名为test.txt

hdfs dfs -get /myLocalFile.txt  /home/hadoop/ # get 下载文件到本地 下载HDFS根目录下的myLocalFile.txt文件到本地的/home/hadoop/
```

## 参考资料
 [IaaS，PaaS，SaaS 的区别 作者： 阮一峰](http://www.ruanyifeng.com/blog/2017/07/iaas-paas-saas.html)

 [什么是物联网？其发展前景如何？](https://www.zhihu.com/question/19751763)

 [Hadoop安装教程_伪分布式配置_CentOS6.4/Hadoop2.6.0](http://dblab.xmu.edu.cn/blog/install-hadoop-in-centos/)

 [Hadoop：hadoop fs、hadoop dfs与hdfs dfs命令的区别](https://blog.csdn.net/pipisorry/article/details/51340838)