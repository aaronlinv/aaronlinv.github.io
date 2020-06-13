---
title: Java笔记-面向对象3
categories: Java笔记
date: 2019-12-29 08:36:40
---
#### 代码块

在类中或方法中使用 { } 括起来的一段代码就称它是一个代码块
代码块当中定义的变量，我们称它是局部变量

1. 局部代码块：直接定义在方法内部的代码块  在调用方法的时候执行。（很少用，if while for（）{} 这个{} 就是局部代码块）
2. 初始化代码块：直接在类当中定义代码块
初始化代码块在运行时，还是要把它放到**构造方法**当中（编译时候直接初始代码块的写到构造方法第一行,带参无参都会）
3. 静态代码块：在初始化代码块前面加上一个static
在加载字节码时就会自动调用在主方法之前执行的。**只执行一次**（先执行静态代码块，然后再执行main）

#### 组合关系和类的加载
组合关系：自己当中的字段是一个“类”类型依赖其它的类（类的成员变量是另一个类）
类加载：
类在什么时候去加载：当第一次使用该类对象的时候，去加载到JVM当中
**只加载一次，下一次直接从内存中使用了**


#### 字段初始化
1. 类的加载：第一次创建该类对象的时候，加载到内存中，加载时会执行static代码块
2. 字段初始化
- 静态字段：在静态代码块中初始化
- 非静态字段：在构造器中初始化
3. 子类构造器默认会调用父类构造器


```java
class SuperClass{
	static {
		System.out.println("SuperClass static代码块");
	}
	public SuperClass() {
		System.out.println("SuperClass 构造器");
	}
}

class SubClass extends SuperClass{
	static {
		System.out.println("SubClass static代码块");
	}
	SubClass() {
		super();
		System.out.println("SubClass 构造器");
	}
}


public class MyXq {
	
	
	//private static MyXq xq=new MyXq(); //静态字段实在静态代码块中初始化
	//编译实际上是：
	 
	private static MyXq xq=null;
	static {
		xq=new MyXq();
		System.out.println("MyXq 类的static静态初始化 ");
	}
	
	
	
	
	//private SubClass sub=new SubClass();
	//编译实际上是:
	private SubClass sub=null;
	MyXq(){
		sub=new SubClass(); //在构造器中初始化
		
		System.out.println("MyXq构造器");
	}
	
	

	
	
	public static void main(String[] args) {
		System.out.println("main");
	}
}

/*
输出结果：
SuperClass static代码块
SubClass static代码块

SuperClass 构造器
SubClass 构造器

MyXq构造器
MyXq 类的static静态初始化
main
*/
```
1. 程序运行：执行main，在main之前需要对类初始化
2. 对MyXq类进行初始化，会执行静态代码块：调用MyXq构造器
3. MyXq构造器：new了一个SubClass，new之前先把SubClass这个类加载到内存，加载子类前会先判断有没有父类，如果有，会先把父类加载成字节码放到内存当中，然后再去把自己加载到内存，所以是先加载SuperClass，执行SuperClass的静态代码块，然后是SubClass的静态代码块
4. 子类构造器第一句为super(),所以先执行SuperClass构造器，再执行SubClass构造器

#### final关键字
继承弊端：破坏了我们的封装，继承可去访问父类当中的实现细节，可以覆盖父类当中的方法

字段：不能再去修改该字段
方法：子类就能再去覆盖该方法
类：该类就不能再去被继承

final关键字：只能用，不能修改

注意点：
- final 修饰字段：必须得要自己手动设置初始值
- final 修饰变量：就代表是一个常量，命令规则：所有的字母都大写MAX_VALUE
- final 可以在局部代码块当中使用
- final 修饰基本数据类型：值不能修改
- final 修饰引用类型：可以修改类里面的成员，但是不能修改地址

#### 单例设计模式
设计模式：之前很多程序员经常无数次的尝试，总结出来一套最佳实践
单例：一个类在内存当中只有一个对象。别人不能再去创建对象
工具类一般都是单例设计模式
工具类：把一些经常使用的功能，写在一个类当中，以后使用该功能时，直接调用


饿汉模式（**这类的单例不能被继承**，因为子类默认有构造器，构造器默认有super()默认访问父类构造器，所以报错，其他形式单例可被继承）
1.必须得要在该类中创建一个对象出来
2.私有化自己的构造器。防止外界通过构造器来创建新的对象
3.给外界提供一个方法，能够获取已经创建好的对象

```java
class ToolUtil{
	private static ToolUtil instance=new ToolUtil();
	private ToolUtil(){}
	
	public static ToolUtil getInstantce() {
		return instance;
	}
}
```
优点
1.控制资源的使用
2.控制实例的产生数量，达到节省资源目的
3.作为通信媒介，数据共享

#### 包装类
对基本数据类型进行包装，把基本数据类型包装一个对象。
把基本数据类型变的更强大，以面向对象的思想来去使用这些类型。
基本数据类型    包装类       （都是首字母大写，只有int和char是英文全称）

| 基本数据类型 | 包装类| 
| :--: | :--: |
| byte      |    Byte |           
| short     |    Short|
| int       |    **Integer**|
| long      |    Long |
| float     |     Float |
| double    |  Double|
| char      |    **Character** |
| boolean   | Boolean|

```java
public static void main(String[] args) {
    int i=20;
    Integer num=new Integer(i);
    
    System.out.println(num);
    System.out.println(num.MAX_VALUE);//本质用类名调用 同下一行
    System.out.println(Integer.MAX_VALUE);
    System.out.println(Integer.MIN_VALUE);
    System.out.println(num.TYPE);
    
    num=Integer.valueOf(30);
}
```

装箱:基本数据类型 -> 包装类 
拆箱:包装类->基本数据类型

自动装箱 自动拆箱（语法糖）
- 自动装箱
Integer i1=10; 
//本质 Integer i1=Integer.valueOf(10);
- 自动拆箱
int i2=i; 
//本质int i2=i.intValue() ;

#### 字符串与其他类型转换
```java
//字符串->包装类型
Integer i =new Integer("10");//字符串中不能有非数字，比如字母

//包装类型->字符串
String s=i.toString();

//基本数据类型->字符串
String str2=2+"";

//字符串->基本数据类型
String str3="2020";
int i3 =Integer.parseInt(str3);

//字符串转boolean
Boolean b=new Boolean("myxq"); //除了字符串true其他都返回false
```
#### 基本数据类型和包装类区别
1. 默认值
int 0 
Integer null
2. 包装类当中提供了很多方法直接给我们使用如：
Integer.toBinaryString(5)
3. 集合框架当中不能存放基本数据类型，只能存对象

什么时候使用基本数据类型什么时候使用包装类
- 在类当中，成员变量一般都使用包装类
- 在方法中，我们一般都使用基本数据类型

方法中，基本数据类型存储在栈当中，包装类型存放在堆当中

#### 包装类valueOf缓存设计
用valueOf获取包装类对象，在缓存范围内直接使用缓存，超过缓存范围，则创建新的对象，返回新的地址
- Boolean：(全部缓存)
- Byte：(全部缓存)
- Character(<= 127缓存)
- Short(-128 — 127缓存)
- Long(-128 — 127缓存)
- Integer(-128 — 127缓存)
- Float(没有缓存)
- Doulbe(没有缓存)

#### 	对基本数据类型包装的好处
1.使用包装对象后，功能变的更加强大
例如：使用Double表示一个人的分数，一个人的分数为0分，可以表示0.0，如果这个人没有来考试可以用null表示

2.包装类当中给我们提供了很多方法
例如：我们要将一个数据转成二进制，使用包装对象后，就可以直接调用方法

---

## 参考资料
[Java零基础到高级JavaEE就业实战](https://study.163.com/course/introduction/1005537028.htm)
[JAVA包装类的缓存范围](https://www.cnblogs.com/javatech/p/3650460.html)
