---
title: Java笔记-基础
date: 2019-12-23 10:52:41
categories: Java笔记
---
## Java 基础

#### 常量（6种）
- 整数
  * 二进制   0b 0B （Java7的新特性）（这个是数字零，而不是字母O） 
  * 八进制   0
  * 十六进制 0x 0X 
- 浮点数
- 字符
- 字符串
- 布尔型
- null
  
#### 数据类型（Java是强类型语言）
- 基本数据类型（4类8种）
  - 整数
    - byte 1字节 
    - short 2字节
    - int 4字节
    - long 8字节 
  - 浮点数
    - float 4字节
    - double 8字节
  - 字符 
    - char 2字节
  - 布尔
    - boolean 1字节
  - 注意点
    - 整数默认int，浮点数默认double 
    - long后缀用L标记，float用F标记
      - 浮点数默认是double，所以把浮点数赋值给float会报错，应该在浮点数后面加F
    - 科学计数法：3.14e2， 3.14E2 返回double
    - float和double都不能精确表示小数
  - 自动转化
    - 一个算术表达式中包含多个基本数据类型(boolean除外),不同类型的数据先转化为同一类型，然后进行运算。
    - byte,short,char --> int --> long --> float  --> double
      ``` Java
      short s=8;
      byte s2=5;
      short s=s1+s2;报错
      int s=s1+s2;  //正确 
      //自动转化为int类型就像：
      int s = (int)s1 +(int)s2; 
      //或者强转 
      short s=(short) (s1+s2);
      ```
- 引用数据类型
  - 类
  - 接口
  - 数组


#### 运算符
  - 算术运算
    ``` Java 
    System.out.println('a' + 1); //98
    System.out.println("a" + 2); //a2 字符串拼接

    //System.out.println(100 /0); 报错

    System.out.println(100.0 / 0);//Infinity 
    System.out.println(-100.0 / 0);//-Infinity 
    //0自动转化为double 0.0 一个数除以很无限接近0的数，就无限大 

    System.out.println(0.0 / 0.0);//NaN  not a number

    System.out.println(-10 % -3);//   -3 模除符号只取决于第一个数

    short s = 8;
    //s = s + 1; 报错 结果为int不能赋值给short
    s += 5; //自动强转为short类型
    ```
  - 逻辑运算符的短路


#### 流程语句
  - switch
    ```java
      switch (表达式)
      {
        case A值:
          语句1;
          break;
        case B值:
          语句2;
          break;
        default:
          语句3;
      }
    ```
    1. 表达式支持 byte , short , char , int (没有long)，从Java7开始支持String类型
    2. 一旦符合找到匹配的case就开始往下执行（不管后面的case是否匹配）--穿透，除非遇到break或return
    3. 找不到匹配的case，执行default，一般放在最后，放在其他位置，也会穿透
  - for
    ```java
    outter:for(){
      for(){
        if() break outter; //直接退出外层循环
      }
    }
    ```
  - return 
    - 结束循环所在的方法
  
#### 方法与数组
  - 注意点
    - 一个方法前面有static 调用的方法也应该有static
    - 遵循标识符的规范，多个单词用驼峰表示法：myName
  - 方法签名
    - 方法签名:方法名称  +  方法参数列表;
    - 在同一个类中,方法签名是唯一的,否则编译报
  - 增强for （foreach） 语法糖
```java
int [] arrays=new int [] {1,2,3};

for (int array:arrays) {
  System.out.println(array);
}
```
方法的可变参数 语法糖
    - 可变参数就是,方法的数组参数的一种简写 ( 会自动把... 转成数组 )
    - 可变参数必须作为方法的最后一个参数


## 参考资料
[Java零基础到高级JavaEE就业实战](https://study.163.com/course/introduction/1005537028.htm)