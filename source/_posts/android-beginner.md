---
title: 安卓初学指南
categories: 编程杂记
date: 2020-06-01 14:34:26
---
项目第二阶段即将结束，这段时间遇到挺多坑的，所以整理了一下经验，如果你想尝试安卓开发，希望能帮到你。在此之前可以先看我之前写的[初见安卓开发](https://aaronlinv.github.io/2020/03/30/android-note/)，了解一下安卓开发目前的情况

## 预先准备
1. 需要熟悉Java基础语法
2. 安装开发环境（JDK、AndroidStudio(简称AS)、虚拟机等），安装环境比较繁琐，而且需要下载很多东西（默认都是安装到C盘，总共可能会占用20G左右的空间）。安装可能会劝退一部分朋友，这里推荐两个教程跟着做就没问题了

安装AS之前一般会先安装Java开发工具包(JDK)，可以参考这篇博客[【Android Studio安装部署系列】JDK开发环境搭建](https://www.cnblogs.com/whycxb/p/9032559.html)，比较值得一说的是，现在下载JDK要到Oracle官网，而且还需要注册账号，有点麻烦。然后开始安装AS，参考这个视频教程[1#Android Studio开发环境 (Attect) Android开发教程](https://www.bilibili.com/video/BV18b411H7Fr)

3. [Android Studio 设置代码提示和代码自动补全快捷键](https://blog.csdn.net/wyf2017/article/details/81355414) 
4. [Android Studio 真机测试/开发者模式](https://www.cnblogs.com/zlc364624/p/10704980.html)

初学比较常用的快捷键：
- 智能建议：Alt 回车
- 代码整理（格式化代码）： Ctrl Alt L
- 注释：Ctrl /
- 块注释：Ctrl Shift /

## 开始学习
如果没有任何开发经验，比较推荐看视频[Android开发教程（2019最新版,使用JetPack）](https://www.bilibili.com/video/av50954019)，这个教程使用的是JetPack库，前40集使用Java，从41集开始换为了Kotlin。个人觉得这作者讲的深入浅出，而且教程中也传递了很多规范化的思想，很适合初学者。要注意作者早期视频使用的是内测版本的ViewModel库，而现在默认自带稳定版，所以不需要手动添加ViewModel依赖，视频中使用的ViewModel构造方法已过时，应该使用下面这个：
``` java
MyViewModel = myViewModel = new ViewModelProvider(this).get(MyViewModel.class);
```

初学最好按着教程一步一步来，变量名也最好跟着教程来，这样出错了跟着视频，排错起来也比较容易。一定要跟着敲代码，边敲边理解整个逻辑，刚开始可能比较懵，但是到后面，对整个体系有了一定了解，就会豁然开朗，这个时候可以看看[官方文档](https://developer.android.com/guide)，这样会加深对安卓开发或是JetPack的理解

<br>觉得学的差不多了，就可以开始在GitHub上找一些感兴趣的安卓项目（或者是找一些最佳实践），克隆下来，看看别人是怎么写的，模仿这写一写，这个过程会遇到很多问题，解决这些问题，就会收获很大的提升

## 遇到的问题
1. 如果开始使用数据库或者网络相关操作就会遇到不能在主线程（UI线程）上运行这些耗时操作的问题，这个时候一般解决方案就是new Thread，但是这样的话在new出来的线程里操作UI控件会报错：
```
Only the original thread that created a view hierarchy can touch its views.
```
这个时候就需要学习一下[Android多线程和异步任务](https://www.bilibili.com/video/BV1m4411r73w)，Kotlin的话可以用协程
2. 涉及网络请求推荐看这个系列，从Java原生API到OkHttp再到Retrofit：[Android开发基础-网络编程](https://www.bilibili.com/video/BV1TJ411v75g)
3. 如果界面比较复杂，使用系统自带的控件可能无法直接实现我们的需求，这个时候需要写一些自定义控件，推荐这个系列[Android开发自定义控件基础课程](https://www.bilibili.com/video/BV1CZ4y1p7n5)
4. 页面跳转会使用到Jetpack中的Navigation组件：[Jetpack 之 Navigation](https://www.duanyitao.com/2019/03/31/Jetpack-%E4%B9%8B-Navigation/)

## 组件库
安卓有很多成熟的第三方组件库，合理使用可以简化开发，加速开发速度
在使用这些组件库时需要考虑安全性、稳定性等问题
推荐这个项目，可以帮助你找到适合的组件库[AndroidLibs](https://github.com/XXApple/AndroidLibs)