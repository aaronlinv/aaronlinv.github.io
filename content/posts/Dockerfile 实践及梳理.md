---
date: '2021-09-01T09:15:33+08:00'
title: 'Dockerfile 实践及梳理'
categories: ["Docker"]
---

上一节：[Docker 实践及命令梳理](https://www.cnblogs.com/aaronlinv/p/15130730.html)
下一节：[IDEA 配合 Dockerfile 部署 SpringBoot 工程](https://www.cnblogs.com/aaronlinv/p/15228488.html)

---
Dockerfile 是一个文本文件，我们可以通过组合一条条的指令 (Instruction)，来构建满足我们需求的 Docker 镜像
## 文档
[Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

[Reference](https://docs.docker.com/engine/reference/builder/)

[Dockerfile 指令详解](https://yeasy.gitbook.io/docker_practice/image/dockerfile)
## 简单上手
使用 Dockerfile 构建SpringBoot 工程的镜像

1. 新建 SpringBoot 项目，默认的端口是 8080 ，新建 Controller 和 Mapping
```java
@RestController
public class HelloController {
    @GetMapping("hello")
    public String hello() {
        return "hello world!";
    }
}
```
启动项目，访问 http://localhost:8080/hello 测试

2. 打 jar 包
注意，需要在 pom 中添加 spring-boot-maven-plugin 插件，否则运行 jar 包时会提示：没有主清单属性
```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
```

```bash
#打包
mvn package
```
target 目录下就可以找到 .jar 文件，我这里的文件名为：demo-0.0.1-SNAPSHOT.jar
在 Linux 新建 `~/springboot` 文件夹，并将 jar 包上传到这个文件夹下

3. 新建 Dockerfile
在这个文件下新建 Dockerfile 文件
``` dockerfile
# 基于 openjdk:8-jre 这个基础镜像进行构建
FROM openjdk:8-jre

# 这里的 demo-0.0.1- SNAPSHOT.jar 要对应上传的 jar 包名称
# 将 本地 jar包 复制到容器内
COPY demo-0.0.1-SNAPSHOT.jar  app.jar

# 开放 8080 端口
EXPOSE 8080

# 运行命令、参数
ENTRYPOINT ["java","-jar"]
CMD ["app.jar"]
```
保存文件，退出编辑器

4. 编译 Docker 镜像
```bash
# build 是构建 Docker 镜像的命令
# -t 指定镜像的 tag
# 名称：demo 版本：v1.0
# 最后的 . 表示 build context 目录为当前目录，目的是为了找到 所需的 jar 包
docker build -t demo:v1.0 .
```

5. 启动容器
```bash
# 前台启动刚构建的 SpringBoot 容器
# -p 映射容器8080端口 到宿主机的 8080 上
docker run -p 8080:8080 demo:v1.0
```

6. 测试
访问 Linux 的8080 端口，注意替换为自己的 Linux 的地址，并开放 8080 端口

http://192.168.43.161:8080/hello

## build context
Dockerfile 默认会使用它自己所在的目录作为 context，通过 docker 执行构建命令后，Docker daemon 会拷贝 context 目录下的`所有文件`，所以 context 目录不要放置项目无关的文件，或者可以使用 `.dockerignore` 定义忽略文件，也可以指定 context 路径
```bash
# build 命令通过 Dockerfile 构建镜像
# 指定 ~/dockerfile 为 build context
docker build ~/dockerfile
# 不需要添加文件到 context 可以使用 -
docker build -
```
可以通过 stdin 的方式，避免生产 Dockerfile 文件，直接 build 镜像
```bash
docker build -t myimage:latest -<<EOF
FROM busybox
RUN echo "hello world"
EOF
```
除了可以指定 context外，还可以通过-f 指定 Dockerfile 所在的路径
```bash
docker build  -f dockerfiles/Dockerfile .
```
## 最佳实践
非常推荐官方的 Dockerfile最佳实践：[Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

1. 每个容器单一职责，有利于横向拓展和复用
2. 旧版强调减少层数以提高性能，现在只有 RUN, COPY, ADD 这几个命令会创建层，其他命令只会创建中间层。并且只有使用到资源最终会被拷贝到最终镜像
3. 多个参数按字母顺序排列，并使用空格和 `\` 进行分割，提高可读性
4. `--no-cache` 不使用缓存，默认 build 过程中如果检查到有可重用的镜像层则使用。从基础镜像开始，每一条命令逐一检查，如果命令不一样则缓存失效。使用 `ADD` 和 `COPY` 则会校验使用到的文件`校验和`是否相同，除了这两个命令，其他则不会通过文件变化来决定是否匹配缓存，而是仅通过命令本身是否一致来判断是否匹配缓存，比如：`RUN apt-get -y update`会改变容器内的文件，但是也只使用这个命令匹配缓存，而不会通过文件的变动。一旦缓存失效，后续都会产生新的镜像层

## Dockerfile 指令 (instructions)
### FROM
Dockerfile 的第一个命令一般都是 FROM，通过这个指定该镜像的 Base Image，推荐基础镜像：[alpine](https://hub.docker.com/_/alpine/)，因为它完整且轻量，如果不需要 Base Image 可以用 `FROM scratch`，代表该镜像基于一个空镜像进行构建
### RUN
由于上面提到的缓存匹配原则，`RUN apt-get update` 命令可能会导致直接使用了原来缓存的镜像层，而没有执行该命令获取最新的软件列表，可以使用 `RUN apt-get update && apt-get install -y` 来使缓存失效
可以使用 `\` 分割，提高可读性：

```bash
RUN apt-get update && apt-get install -y \
    curl
```

### CMD
指定容器启动时运行的命令，通常默认采用的格式：`CMD ["executable", "param1", "param2"…]`，如：
```dockerfile
CMD ["perl", "-de0"]
```
这样使用 `docker run -it` 命令进入容器时，就会默认进入 shell 界面

### EXPOSE
指定容器需要监听的端口

### ENV
可以使用 ENV 更新 PATH 环境变量，例如
```dockerfile
ENV PATH=/usr/local/nginx/bin:$PATH
```
注意！每一个 `ENV` 指令都会创建一个新的中间层 (intermediate layer)，如果使用 ENV 设置了变量，在未来的层 unset 了变量，那么它在 unset 之前依然是可用的。为了防止这种情况，我们应该用 RUN 进行环境变量的 设置和取消
```dockerfile
ENV ADMIN_USER="mark"
RUN echo $ADMIN_USER > ./mark
RUN unset ADMIN_USER
```
### ADD or COPY
两个命令功能相似，优先使用COPY，它的作用只是将本地文件拷贝到容器内，而 ADD 则有其他特性，比如：自动将本地 tar 文件提取到镜像中、远程URL
如果多个步骤需要使用不同的文件，应该单独 COPY，而不是一次性 COPY，这样部分文件变化不会导致所有的缓存都失效
避免使用 ADD 通过 URL 获取包，可以使用 `curl` 或者 `wget`，这样可以在提取后删除文件，避免镜像多一层，还可以通过管道，就不需要再手动删除中间文件
```dockerfile
RUN mkdir -p /usr/src/things \
    && curl -SL https://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
```
### ENTRYPOINT
使用 ENTRYPOINT 设置主命令，还可以用 CMD 设置默认的可选参数
```dockerfile
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```
运行编译镜像，指定名称为：s3cmd，运行容器
```bash
docker run s3cmd
```
默认会运行 `s3cmd` 并带上 `--help` 参数，即：显示该命令的帮助

运行下面命令：
```bash
docker run s3cmd ls s3://mybucket
```
`ls s3://mybucket` 会覆盖默认可选参数 `--help`

如果需要覆盖 ENTRYPOINT，需要使用 `--entrypoint` 参数
### VOLUME
暴露镜像中可变和用户可修改的数据，比如：存储文件、配置文件，比如：
```dockerfile
VOLUME /data
```
设置的目录会在容器运行时自动挂载为匿名卷，如果没有设置，就会写入容器存储层

### USER
如果不需要使用 `sudo` ，可以通过 USER 切换到非 root 用户，例如：
```dockerfile
RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres
```

### WORKDIR
WORKDIR 指令可以来指定工作目录，不存在会自动创建
Dockerfile 不同于 Shell，下面的命令其实是不同的层，第一条的 `cd` 不会影响第二条命令，最终运行结束会导致在 /app 下找不到 world.txt 文件
```dockerfile
RUN cd /app
RUN echo "hello" > world.txt
```
应该使用：
```dockerfile
WORKDIR /app
RUN echo "hello" > world.txt
```

## 参考资料
[使用 Dockerfile 定制镜像](https://yeasy.gitbook.io/docker_practice/image/build)

[利用构建缓存机制缩短Docker镜像构建时间](https://segmentfault.com/a/1190000018222648)

[Dockerfile: ENTRYPOINT和CMD的区别](https://zhuanlan.zhihu.com/p/30555962)

---

上一节：[Docker 实践及命令梳理](https://www.cnblogs.com/aaronlinv/p/15130730.html)
下一节：[IDEA 配合 Dockerfile 部署 SpringBoot 工程](https://www.cnblogs.com/aaronlinv/p/15228488.html)