---
date: '2021-08-11T23:19:33+08:00'
title: 'Docker 实践及命令梳理'
categories: ["Docker"]
---

下一节：[Dockerfile 实践及梳理](https://www.cnblogs.com/aaronlinv/p/15213211.html)

---
## 文档
[Docker Reference Documentation](https://docs.docker.com/reference/)

[Docker 从入门到实践 【中文】](https://vuepress.mirror.docker-practice.com/)

## 安装
安装 Docker，设置开机启动，然后配置阿里云镜像加速
###  1. 安装 Docker
[Docker 官方安装](https://docs.docker.com/get-docker/)

[CentOS 官方安装教程](https://docs.docker.com/engine/install/centos/)，直接安装速度相对慢，推荐使用 [使用脚本自动安装 Docker](https://vuepress.mirror.docker-practice.com/install/centos/#%E4%BD%BF%E7%94%A8%E8%84%9A%E6%9C%AC%E8%87%AA%E5%8A%A8%E5%AE%89%E8%A3%85)：

```bash
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh --mirror Aliyun
```

```bash
# 开机启动 docker
sudo systemctl enable docker
# 启动 docker
sudo systemctl start docker
```
### 2. 阿里云镜像加速
注意！`registry-mirrors` 需要替换成自己的 [阿里云镜像加速器地址](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)，通过点击地址获取

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["这里替换成自己的阿里云镜像加速器地址"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
## Docker CLI
CLI 是 Command-Line Interface （命令行界面）的缩写
命令详情可以参考官方文档：[Docker Reference Documentation](https://docs.docker.com/engine/reference/commandline/cli/)

这里通过几个场景，把 Docker 先用起来
### 场景0：Tomcat
```bash
# 输入 docker 回车，docker 的命令会被罗列出来，便于查询
docker

# 查询有那些 MySQL 镜像
docker search tomcat
```
相对于直接search，使用搜索 [Docker Hub](https://hub.docker.com/) 更方便

搜索 [Tomcat](https://hub.docker.com/search?q=tomcat&type=image)

可以看到相关的镜像介绍、使用帮助、历史版本：[Tomcat](https://hub.docker.com/_/tomcat)

可以按照文档中的 "How to use this image" 的提示来运行镜像

```bash
# 拉取 tomcat 镜像
docker pull tomcat:8.0-jre8

# 查看镜像列表，可以看到 tomcat 镜像，该命令等特于：docker image ls
docker images

# 运行tomcat 镜像
docker run tomcat:8.0-jre8

# 前台运行，会输出 tomcat 日志
# 按 Ctrl + C 停止


# 添加 -d 后台运行参数
docker run -d tomcat:8.0-jre8
# 查看容器列表，可以查看到容器的id、镜像(IMAGE)、状态(STATUS)、网络端口(PROT)、容器名称(NAME)等信息
docker ps
# 查看 tomcat 日志，这里的 `clever_swanson` 为容器id或名称
docker logs clever_swanson
# 查看容器内部进程信息
docker top clever_swanson

# 进入容器
# i 和 t 参数可以让我们以伪终端的方式进入容器
# bash 是所使用 shell
docker exec -it clever_swanson bash
# 在容器内可以使用 Linux 命令

# 退出容器，回到宿主机（宿主机就是安装 Docker 的这台机器）
exit

# 不需要容器了，可以停止容器
docker stop clever_swanson
# 查看容器列表，tomcat 就隐藏了
docker ps
# -a 参数查看全部所有容器的列表（包括停止的 tomcat）
docker ps -a

# 启动已经停止的容器
docker start clever_swanson

# 删除容器，如果容器还在运行需要加 -f 参数
docker rm clever_swanson
docker rm -f clever_swanson

# 删除了容器，就可以把镜像也删除了，如果有容器还是该镜像需要加 -f 参数
docker rmi tomcat:8.0-jre8
docker rmi -f tomcat:8.0-jre8
```
### 场景1：Tomcat
只是将 Tomcat 容器 run 起来，还是无法满足使用
还需要将容器网络端口映射到宿主机才可以使得外部可以访问容器内部服务
为了方便还需要把 Tomcat 的数据目录和配置目录挂载到宿主机，方便直接进行编辑

可参考官方文档：[Tomcat](https://hub.docker.com/_/tomcat)
```bash
# 部署 Tomcat
# run 容器时，本地不存在对应镜像，会自动 pull
# -p 将容器内的网络端口映射到宿主机 ，8080:8080 前面为宿主机，后面为容器
# --name 指定容器名称
docker run -p 8080:8080 -d --name mytomcat tomcat:8.0-jre8

# 可以通过 docker 的子命令对 容器进行操作，比如：ps,exec,top,stop

# 这个容器已经占用了宿主机的 8080 端口，为了后续的 Tomcat 可以绑定到宿主机的 8080 端口，所以将 这个容器 stop
docker stop mytomcat

# 通过数据卷的方式 将容器内的数据映射到宿主机
# 语法：- v 数据卷名称:容器内目录
# Tomcat 部署的 web应用目录：/usr/local/tomcat/webapps
# 配置文件：/usr/local/tomcat/conf
docker run -p 8080:8080 -v apps:/usr/local/tomcat/webapps -v confs:/usr/local/tomcat/conf -d --name mytomcat2 tomcat:8.0-jre8
```
这个时候 Tomcat 已经启动了，可以通过 `http://ip宿主机:8080` 来访问 Tomcat 的默认主页，例如我的访问地址 [http://192.168.43.166:8080](`http://192.168.43.166:8080`) ，看到汤姆猫的图标就成功了。如果访问失败，可能是对应的 8080 端口没有开放，CentOS7 可以参考：[CentOS7开启端口](https://blog.csdn.net/zx110503/article/details/78787483)
### 场景2：MySQL
可参考官方文档：[MySQL](https://hub.docker.com/_/mysql?tab=description&page=1&ordering=last_updated)

```bash
# 通过 -e 指定参数，指定 MySQL 的 root 账户的密码为：1234
docker run --name mysql -e MYSQL_ROOT_PASSWORD=1234 -p 3306:3306 -d mysql:5.7.32

# 停止容器，防止端口占用
docker stop mysql

# 数据库的数据将会随着容器消失而消失，所以需要将数据库文件持久化到宿主机，
# 配置映射到本地
docker run --name mysql2 -e MYSQL_ROOT_PASSWORD=1234 -p 3306:3306 -d -v mysqldata:/var/lib/mysql -v mysqlconfig:/etc/mysql  mysql:5.7.32
```
MySQL 就成功运行了，可以通过 Navicat 或者其他工具测试数据库，地址为宿主机 ip地址，用户名为 root，密码为：1234，可以尝试存储数据，数据会被存储在数据卷，这里指定的数据卷名称为：mysqldata
```bash
# 查看所有数据卷
docker volume ls
# 查看 MySQL 的数据卷
docker inspect mysqldata
# 返回的 json 对象，其中 Mountpoint 的值就是，文件对应挂载的位置

# 我这里挂载的地址为：/var/lib/docker/volumes/mysqldata/_data
# 进入这个目录，就可以看到存储的文件
```
使用数据卷的好处在于：容器被移除了，重新运行一个新容器，直接挂载原来的数据卷就可了，数据不会丢失
```bash
# 移除容器
docker rm -f mysql2
# 重新运行新的容器，并挂载原来的数据卷
docker run --name mysql2 -e MYSQL_ROOT_PASSWORD=1234 -p 3306:3306 -d -v mysqldata:/var/lib/mysql -v mysqlconfig:/etc/mysql  mysql:5.7.32
```
### Redis
可参考官方文档：[Redis](https://hub.docker.com/_/redis?tab=description&page=1&ordering=last_updated)

需要注意的是：Redis 需要在镜像名称即 `redis:5.0.10` 的后面添加 `redis-server --appendonly yes` ，以此覆盖镜像默认的命令

```bash
docker run --name redis -p 6379:6379 -d redis:5.0.10

# 停止容器，防止端口占用
docker stop redis

# 开启持久化 redis-server --appendonly yes
# 开启后，持久化生成的 aop 文件会被放入容器中的 /data 目录中
docker run --name redis2 -p 6379:6379 -d -v redisdata:/data redis:5.0.10 redis-server --appendonly yes
# 可以使用 Redis Desktop Manager 等工具，通过宿主机 ip 连接，进行测试
```
### 清理容器
```bash
# 查看容器列表可以看到很多容器
docker ps
# -a 可以看到所有的容器，包括已经停止的
docker ps -a
# 如果忘记了参数或者命令可以在命令后面加上 --help，会有提示
docker ps --help
# 可以看到：-q, --quiet   Only display container IDs，即：-q参数仅输出容器id，结合 -a，可以输出所有容器的 id
docker ps -aq
# 结合 rm -f 就可以移除所有的容器了
docker rm -f $(docker ps -aq)

# 清除没有用到的数据卷，有重要数据要谨慎
docker volume prune
```

## docker
我们可以通过下面的命令，来找到 docker 的位置
```bash
whereis docker
# 我这里执行返回的结果是：docker: /usr/bin/docker /etc/docker /usr/libexec/docker /usr/share/man/man1/docker.1.gz
```
可以看到 `docker` 的可执行文件位于 `/usr/bin`，这个路径存在环境变量 PATH 中，所以我们可以在任意路径 使用 `docker` 命令

Docker 是 C/S 架构模式（客户端-服务器），所以上面的 `docker` 实际上是 Docker 的客户端，Docker 的服务器是 Docker Deamon 对应的就是 `dockerd`，也在这个目录下，Deamon 就是 Docker 引擎，Docker 客户端通过 Docker API 与 Deamon 进行通信


`docker` 是一个可执行程序，包含了许多命令，输入
```bash
docker --help
```
会将 Usage（用法）、Option（选项）、Commands（命令）都展示出来，Management Commands 也s是 Commands。每个命令可能会有它自己的子命令、选项
```bash
# run 命令有许多的选项
docker run --help
# volume 命令有许多子命令，例如：ls, rm, inspect
docker volume --help
```
## 命令梳理
使用 `--help` 参数，就可以查询到到对应命令的使用方法，所以我们只要理解 Docker 命令的框架即可，不用记忆命令细节
对镜像进行操作：images, rmi, search
对容器进行操作：run, stop, start, restart, exec, logs, top 
其他：ps, cp, info, pull, version

还有一些 `Management Commands`，例如：image, network, volume
这些命令都是名词，即要操作的对象，而具体的操作通过其子命令指定，语义更加清晰
image 常用子命令：ls, rm, prune
network 常用子命令：create, ls, inspect, rm, prune
volume 常用子命令：同上面 network 的四个


## 参考资料
[Install docker using the convenience script](https://docs.docker.com/engine/install/centos/#install-using-the-convenience-script)
[【编程不良人】Docker容器技术&Docker-Compose实战](https://www.bilibili.com/video/BV1ZT4y1K75K)
[Docker组件](https://www.jianshu.com/p/1ffe8c18dfd5)
[Docker 组件之间的关系](https://www.jianshu.com/p/5dbc12e8b6f9)

---
下一节：[Dockerfile 实践及梳理](https://www.cnblogs.com/aaronlinv/p/15213211.html)