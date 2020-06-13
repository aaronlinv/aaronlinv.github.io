---
title: Java笔记
categories: Java笔记
date: 2020-03-24 19:51:42
---
## 创建工程%20问题
创建Maven工程，需要获取 resources 下文件夹路径
```java
this.getClass().getResource(“/”).getPath();
```
返回的路径String中含有%20，原因是 resources 完整路径存在空格
所以创建项目应尽量避免使用空格和中文

参考：[【整】getResource().getPath() 路径带 %20 问题展开](https://blog.csdn.net/renminzdb/article/details/88339383)