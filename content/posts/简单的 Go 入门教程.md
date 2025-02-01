---
date: '2021-10-18T09:10:33+08:00'
title: '简单的 Go 入门教程'
categories: ["简单入门"]
---

Go（又称 Golang ）是 Google 开发的一种静态强类型、编译型、并发型，并具有垃圾回收功能的编程语言

Docker 和 Kubernetes 都是使用 Go 进行开发的，这几年 Go 越来越流行，生态也越来越好了

初学 Go 的时候会遇到了一些小问题，在一些教程中没有提及或者因为时效性的缘故，经常需要查阅很多资料才能弄懂，所以想写一篇比较新人视角的文章帮助大家入门

## 安装
Go 的官网就是 [golang.org](https://golang.org/)，点击首页的 [Download Go](https://golang.org/dl/) 就可以跳转到下载页面，然后下载对应操作系统的 Go，如果国内访问缓慢，可以访问镜像站：[golang.google.cn](https://golang.google.cn/)，官方安装教程：[Download and install](https://golang.org/doc/install)

Windows 只要下载对应的 msi 文件，然后打开后按照提示基本上就是下一步下一步... 具体可以参考这篇博客：[Windows Go 开发环境下载、安装并配置](https://www.cnblogs.com/Can-daydayup/p/15177665.html)，安装完成后 Windows 需要 Win键 + R键，然后输入 `cmd`，输入 `go version`，显示版本号就说明安装完成

## GOPROXY
国内下载依赖库会比较缓慢，所有我们需要配置 Go Proxy 加速依赖下载（有点像 Java 中修改 Maven 镜像仓库），这里镜像源使用 [七牛云](https://goproxy.cn)
```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

也可以通过 `go env` 查看所有的 Go 环境变量，其中就包括 GOPROXY，这个变量定义的就是配置 Go 镜像

## Hello World
推荐使用 JetBrains 家的 [GoLand](https://www.jetbrains.com/go/)，使用体验基本和 JetBrains 家的其他软件例如：IDEA、PyCharm 相似，还有一种也比较主流，就是使用 VSCode 配合 Go 插件，可以参考：[VsCode Go插件配置最佳实践指南](https://zhuanlan.zhihu.com/p/320343679)，相对来说需要比较多的配置，而且调试比较麻烦，对于新手不是很友好

Go 圣经中也有更详细的 [Hello, World](https://books.studygolang.com/gopl-zh/ch1/ch1-01.html) 教程

新建文件 ：helloworld.go
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World")
}
```
注意 package 必须指定为 `main` 否则无法运行

静态编译 Go 代码，在代码对应的目录打开命令行
```bash
go build helloworld.go
```
这时候当前目录会生产可执行文件：helloworld

```bash
helloworld.exe
# Linux 或者 Mac 下运行的命令是：./helloworld
```
就可以运行，也可以通过 run 命令，直接编译+运行
```bash
go run helloworld.go
```

## Go Modules
Go modules 是 Go 语言的依赖解决方案，详细可以查看 [官方 Modules Wiki](https://github.com/golang/go/wiki/Modules)，Go 最早使用的依赖解决方案是：GOPATH，然后使用Go Vendor ，这两种方案都并不是特别好用，现在还可以搜索到很多旧教程是教你用这两种管理依赖的，所以让使初学者很困惑

Go 1.11 正式推出 Go Modules，Go 环境变量中添加了：`GO111MODULE`（111指的就是版本11.1），用来控制 Go Modules 是否启用，Go 1.16 开始其默认值设置为 `on`。GO111MODULE 的值为 off 表示禁用 Go Modules，on 表示启用，而 auto 表示当项目在 $GOPATH/src 外且项目根目录有 go.mod 文件时，自动开启 Go Modules。Go 1.14 时 Go modules 已经很稳定了，并且推荐应用在生产上，所以现在使用 Go，其实可以不考虑这些问题，直接使用 Go Modules 即可，当然如果对这个细节感兴趣，可以看这两篇博客：[Go Modules 终极入门](https://segmentfault.com/a/1190000021854441)、[一文搞懂 Go Modules 前世今生及入门使用](https://segmentfault.com/a/1190000022722044)

Go Modules 提供了一些命令，列举几个常用的：
1. `go mod init`  生成 go.mod 文件，（这个文件有点类似 Maven 的 pom）
2. `go mod download` 下载 go.mod 文件中指明的所有依赖
3. `go mod tidy` 整理现有的依赖


演示一下如何更新依赖，新建一个 hello.go
```go
package main

import (
	"fmt"
    // 这里引用了一个依赖
	"rsc.io/quote"
)

func main() {
    // 这里使用了引用依赖的 Hello 方法
	fmt.Println(quote.Hello())
}
```

使用 init 命令创建 go.mod
```bash
# 这里的 example.com/hello 是自定义的 module 名称
go mod init example.com/hello
```

这个时候如果运行 `go build`、`go install`、`go run hello.go` 都会提示依赖不存在
```
hello.go:6:2: no required module provides package rsc.io/quote; to add it:
        go get rsc.io/quote
```
我们可以按照提示使用 `go get rsc.io/quote`，用 `go get` 来获取某个具体的依赖
如果有很多依赖的话，`go get` 就比较麻烦，可以使用 `go mod tidy`，它会自动添加丢失的依赖、删除不需要的依赖

在 `go mod tidy` 后，我们可以运行 `go run hello.go`，这个时候程序就可以正常运行了

## 入门
推荐官方的交互式教程 [A Tour of Go](https://tour.golang.org/welcome/1)，网页就可以敲 Go代码，也有中文版本：[Go 指南](https://tour.go-zh.org/welcome/1)，这个教程可以让你快速上手，想要更细致地学习 Go，推荐 [Go语言圣经（中文版）](https://books.studygolang.com/gopl-zh/)

引用 [Go语言圣经 - 入门](https://books.studygolang.com/gopl-zh/ch1/ch1.html) 中的一句话：
> 学习一门新语言时，会有一种自然的倾向，按照自己熟悉的语言的套路写新语言程序。学习Go语言的过程中，请警惕这种想法，尽量别这么做

我们在解决一个问题的时候很容易思维定势，用已经会的语言的思维思考，推荐视频教程 [神奇代码在哪里Go实战](https://space.bilibili.com/1557732/channel/detail?cid=189786&ctype=0)，可以看看其他人在写 Go 的时候是如何思考的