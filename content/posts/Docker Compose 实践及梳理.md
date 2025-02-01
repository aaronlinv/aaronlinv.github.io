---
date: '2021-09-15T09:01:33+08:00'
title: 'Docker Compose 实践及梳理'
categories: ["Docker"]
---


上一节：[IDEA 配合 Dockerfile 部署 SpringBoot 工程](https://www.cnblogs.com/aaronlinv/p/15228488.html)

---
Docker Compose 可以实现 Docker 容器集群的编排，可以通过 `docker-compose.yml` 文件，定义我们的服务及其需要的依赖，轻松地运行在测试、生产等环境

## 文档
[Product manuals](https://docs.docker.com/compose/)

[Compose file version 3 reference](https://docs.docker.com/compose/compose-file/compose-file-v3/)

[Docker 从入门到实践 【中文】](https://vuepress.mirror.docker-practice.com/compose/introduction/)

## 安装 Compose
Compose 依赖 Docker Engine，所有要保证环境安装了 Docker，可参考[官方教程](https://docs.docker.com/compose/install/)，主要分为两步：

```bash
# 1. 下载 Compose 只执行文件到 usr/local/bin/ 目录
# 下载失败可以参考下一小结提供地址安装
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 2. 对 Compose 可执行文件添加运行权限
sudo chmod +x /usr/local/bin/docker-compose

# 输入下面命令查看帮助，测试安装是否成功
docker-compose -h
```
Compose 开源在 Docker 官方的 GitHub 仓库：[docker/compose](https://github.com/docker/compose)，所有的 Compose 都会发布在仓库的 [Releases](https://github.com/docker/compose/releases) 里，步骤1就是使用 curl 命令从 Releases 里下载可执行文件，`uname -s`和`uname -m` 可以读取系统的内核名称和硬件架构，用来匹配需要的 Compose 版本， `curl` 的 -L 参数会让 HTTP 请求跟随重定向（默认不跟随），-o (小写o) 会将服务器响应保存成文件，直接下载到：usr/local/bin/ 下，文件名为：docker-compose，因为这个路径已经在环境变量中了，所以完成步骤2，添加可执行权限后，就可以在任意位置使用了

直接从 GitHub 下载比较慢可以通过以下地址下载：
```bash
# https://vuepress.mirror.docker-practice.com/compose/install/
sudo curl -L https://download.fastgit.org/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

## 入门
Compose 的模版指令与 Docker 的 run 命令相关参数很相似，忘记了 docker 命令可以参考之前的一篇博客：[Docker 实践及命令梳理](https://www.cnblogs.com/aaronlinv/p/15130730.html)

Compose 中有两个重要的概念：
- 服务 (service)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例

- 项目 (project)：由一组关联的应用容器组成的一个完整业务单元，在 `docker-compose.yml` 文件中定义

`docker-compose.yml` 格式如下，注意：YAML 文件必须要键值之间的 `:` 后面必须有一个空格，缩进表示层级，要注意缩进
有使用到的 volumes 和 networks 必须声明

```yaml
# 指定版本
version: "3"
# 服务的集合
services:
  # 其中一个服务，服务名为：webapp
  webapp:
    # 指定该服务使用的镜像
    image: examples/web
    # 端口映射
    ports:
      - "80:80"
    # 数据卷
    volumes:
      - "/data"
```

## 简单上手
在一个 Compose 中启动 Tomcat, MySQL, redis，创建 `docker-compose.yml`
```yaml
version: "3.0"

services:
  tomcat:
    container_name: mytomcat # --name
    image: tomcat:8.0-jre8
    ports:
      - "8080:8080"
    volumes:
      - "tomcatwebapps:/usr/local/tomcat/webapps"
    networks:
      - some_network
    # tomcat 服务依赖于 mysql 和 redis
    depends_on:
      - mysql
      - redis
  mysql:
    container_name: mysql
    image: mysql:5.7.32
    ports:
      - "3306:3306"
    volumes:
      - "mysqldata:/var/lib/mysql"
      - "mysqlconf:/etc/mysql"
    environment:
      - MYSQL_ROOT_PASSWORD=1234
    networks:
      some_network:
  redis:
    container_name: redis
    image: redis:5.0.10
    ports:
      - "6379:6379"
    volumes:
      - "redisdata:/data"
    command: "redis-server --appendonly yes"
    networks:
      some_network:

# 使用到的 volumes 和 networks 必须声明
volumes:
  tomcatwebapps: 
  mysqldata:
  mysqlconf:
  redisdata: 

networks:
  # 声明名称为 “some_network” 的网络
  some_network:
```
在 `docker-compose.yml` 所在路径执行 `docker-compose up` 启动 Compose 项目，它会下载使用到的镜像并在前台运行打印日志，可以使用 Ctrl + C 终止

如果需要后台运行执行 `docker-compose up -d`，这时候使用 `docker ps` 可以看到 Compose 已经根据 yaml 创建了相关的容器，使用 `docker-compose  down` 停止 Compse 并移除自动创建的网桥


使用 `docker network ls` 查看网络或者 `docker volume ls` 查看数据卷，Compose 定义的网络或数据卷名称格式为：docker-compose.yml所在文件夹的名称加上下划线再加上 yaml 中定义名称，如果在 "dockerfile" 文件夹下创建 yaml 文件并启动，那么网络名称为：`dockerfile_some_network`


tomcat 服务使用了 `depends_on`，表示它依赖于 redis 和 mysql 服务，Compose 将优先启动它的依赖再启动它

## 命令梳理
Docker Compose 的命令与 Dokcer 类似，可以使用 --help 参数，就可以查询到到对应命令的使用方法
```bash
docker-compose --help
```
默认启动的模版文件名为 docker-compose.yml，可以使用 -f 指定自定义的模版文件
可以通过 config 命令，检查模版文件语法是否正确

docker-compse 也包含很多子命令：
启动停止相关：up, down, restart, stop, pasue, unpause

资源相关：ps, top, kill, run

进入容器：exec

查看日志：logs

很多子命令都可以在后面跟上某个具体的 service 名称，定向地操作，下面不一一举例，
可以使用`docker-compose help` 再跟上子命令名称，查询其用法

```bash
# 后台启动 yaml 定义的所有容器
docker-compose up -d
# 仅启动 mysql 这个service，会启动其依赖的 service
docker-compose up mysql 指定启动的server名称，
# 停止容器并移除自动创建的网桥
docker-compose down 
# 重启所有 service 后面可以指定上某个具体的 service
docker-compose restart

# 暂停 和 恢复
docker-compose pause
docker-compose unpause

# 进入 redis 这个 service 使用 exit 退出
docker-compose exec redis bash

# 列出当前 yaml 中定义的容器的信息
docker-compose ps

# 删除当前 yaml 中定义的容器，需要先 stop，后面可以指定上某个具体的 service
docker-compose rm

# 查看各个 service 容器内运行的进程情况
docker-compose top

# 查看日志默认查看 yaml 所有的，可以跟上具体 service
# -f 可以保持跟踪，新的日志会马上显示在屏幕上
docker-compose logs
```

## 参考资料
[curl 的用法指南](https://www.ruanyifeng.com/blog/2019/09/curl-reference.html)
[【编程不良人】Docker容器技术&Docker-Compose实战](https://www.bilibili.com/video/BV1ZT4y1K75K)

---
上一节：[IDEA 配合 Dockerfile 部署 SpringBoot 工程](https://www.cnblogs.com/aaronlinv/p/15228488.html)