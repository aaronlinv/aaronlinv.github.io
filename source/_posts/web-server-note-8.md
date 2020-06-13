---
title: Web服务器笔记-Cookie与Session
categories: Web服务器笔记
date: 2020-02-13 16:15:43
---
## 会话技术
用户开一个浏览器，点击多个超链接，访问服务器多个web资源，然后关闭浏览器，整个过程称之为一个会话

会话技术解决什么问题

保持各个客户端自己的数据
每个用户在使用浏览器与服务器进行会话的过程中，不可避免各自会产生一些数据，程序要想办法为每个用户保存这些数据

使用Request域存在的问题
Request域的生命周期为请求开始时创建，请求结束时销毁没有办法办到同时多个商品

使用ServletContext存在的问题
ServletContext生命周期比较长，服务器启动时创建，服务器关闭时销毁存放在这里，容易导致不同用户的各个浏览器之间的数据混淆浪费服务器存储空间

使用Session会话
Session域：当一个浏览器访问服务器时创建，关闭服务器或过期时，销毁Session域对象

使用Cookie会话
1. 请求时在Servlet当中主动把商品保存到Cookie当中，Cookie是浏览器当中的一个缓存区域
2. 在结算请求时，把浏览器缓存中存放的数据发送给服务器

## Cookie 
#### 服务器怎样把Cookie写给客户端
创建Cookie
```java
Cookie cookie = new Cookie(String cookieName,String cookieValue);
```
cookie会以响应头的形式发送给客户端，Cookie只能存储非中文的字符串

向客户端发送cookie
```java
response.addCookie(cookie名称);
```

访问
第一次访问时， 请求头当中没有cookie，响应当中会看到set-cookie
再一次访问时， 请求头当中就能够看到cookie信息
访问服务器的任何资源，一般情况下都会把cookie带去过

#### Cookie默认存储时间
默认cookie：会话级别
打开浏览器、关闭浏览器为一次会话
如果不设置持久化时间，cookie会存储在浏览器的内存中，浏览器关闭	cookie信息销毁

设置Cookie在客户端的存储时间
```java
cookie.setMaxAge(int seconds);
```
设置的时间为秒
如果设置持久化时间，cookie信息会被持久化到浏览器的磁盘文件里
过期会自动删除

#### 设置Cookie的携带路径
访问某一个资源时，要不要带cookie信息
    如何每一外资源都携带，会影响传输速度 
如果不设置携带路径
默认情况下会在访问创建cookie的web资源相同的路径（相同目录下）都携带cookie信息
```
当前Servlet路径：http://localhost/29-Cookie-Session/CookieServlet
web资源相同的路径：http://localhost/29-Cookie-Session/CookieServlet2
```
在myxq/CookieServlet下创建的cookie，在myxq/下的index.jsp访问时会携带cookie，不是在myxq下，不会携带cookie

设置携带路径
```java
cookie.setPath(String path);

// 只有访问cookieServlet才携带cookie信息
cookie.setPath(“/29-Cookie-Session/cookieServlet”);

// 访问指定的工程时， 都会携带cookie信息
cookie.setPath(“/29-Cookie-Session”);

// 访问服务器下部署的所有工程时都会携带cookie
cookie.setPath(“/”);
```


#### 删除Cookie
如果想删除客户端的已经存储的cookie信息
使用同名同路径的持久化时间为0的cookie进行覆盖即可
```java
Cookie cookie = new Cookie("lk","it666");
cookie.setMaxAge(0);
response.addCookie(cookie);
```

#### 服务器如何获取客户端携带的cookie
通过Request对象的getCookies()方法，获取的是所有的cookie，要进行遍历
```java
// 获取所有cookie对象
Cookie[] cookies = request.getCookies();
if (cookies != null) {
    for (Cookie cookie : cookies) {
        String name = cookie.getName();
        if (name.equals("lk")) {
            response.getWriter().write(cookie.getValue());
        }
    }
}
```
		
#### 记录上次登录时间
1.第一次访问时，获取当前的时间，并把它写到cookie当中，响应给浏览器
2.第一次访问，告诉用户是第一次访问
3.用户下次访问时，获取用户携带的cookie,把日期在浏览器当中显示，记录最新的cookie
```java
// 获取当前日期
Date date = new Date();
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
String formatDate = sdf.format(date);
System.out.println(formatDate);

// 写入cookie
Cookie cookie = new Cookie("lastTime", formatDate);
response.addCookie(cookie);

//
String lastTime = null;
// 取cookie
Cookie[] cookies = request.getCookies();
if (cookies != null) {
    for (Cookie c : cookies) {
        if (c.getName().equals("lastTime")) {
            lastTime = c.getValue();
        }
    }
}


response.setContentType("text/html;charset=utf-8");
if (lastTime != null) {
    response.getWriter().write("上次登录时间为："+lastTime);
}else {
    response.getWriter().write("您是第一次登录");
}
```


## session
Session技术是将数据存储在服务器端的技术
会为每个客户端都创建一块**内存空间**存储客户的数据
客户端需要每次都携带一个标识ID去服务器中寻找属于自己的内存空间
Session需要借助于Cookie存储客户的唯一性标识SESSIONID

![Session](web-server-note-8/1.png)
Session如何办到在一个Servlet当中存数据，在另一个Servlet中取出当初存储的数据
每一个用户访问服务器时，会给该用户分配他自己对应的存储空间
并且创建的存储空间有一个编号我们称为SessionID
第一次访问时， 会把对应的sessionID以Cookie的形式写给浏览器
下次再访问时， 会携带sessionID，到当初创建的那个存储空间中取出数据

#### 获取Session对象
获得专属于当前会话的Session对象
```java
    HttpSession session = request.getSession();
```
如果服务器端没有该会话的Session对象，会创建一个新的Session返回
如果已经有了属于该会话的Session直接将已有的Session返回
本质就是根据SESSIONID判断该客户端是否在服务器上已经存在Session

#### 向Session当中存取数据
已经学习的其它两个域对象：ServletContext域、Request域

Session对象也是一个域对象
```java
session.setAttribute(String name,Object obj);
session.getAttribute(String name);
session.removeAttribute(String name);
```

SessionServlet
```java
HttpSession session = request.getSession();
session.setAttribute("lk", "it666");

```
SessionServlet2
```java
HttpSession session = request.getSession();
System.out.println(session.getAttribute("lk"));// it666
```
#### Session的生命周期
- 创建
第一次执行request.getSession()时创建
- 销毁
1.服务器关闭时
2.Session过期/失效（默认30分钟，默认配置在Servers的web.xml里session-timeout，也可在工程web.xml中定义）是从最后一次操作结束时计时
3.手动销毁对象
```java
session.invadate();
```
浏览器关闭，session就销毁，这句话是不正确的
作用范围：默认在一次会话中，也就是说在一次会话中任何资源公用一个Session对象

#### JsessioID持久化
默认情况下，第一次获取session对象时，会帮你创建一个Session，可以获取该Session的ID，会自动的把id写到Cookie中

存在的问题
第一次访问Sevlet1时存储一些数据，在第二个Servlet当中直接取数据，可以直接取到
再把浏览器关闭，直接到第二个Servlet中取数据，发现取不到数据了

原因
因为访问的时候要求带着JsessionID.由于默认情况下，存储cookie是会话级别的，关闭浏览器，就没有了
所以再次打开浏览器，访问资源时，没有jsessionID.  就会创建一个新的Session，就取不到数据

解决办法
在写数据时，自己手动去把SessionID写到cookie当中
写的时候，设置持久化时间
注意，key值一定是和它自动生成的key值是一样的

SessionServlet
```java
HttpSession session = request.getSession();

Cookie cookie = new Cookie("JSESSIONID", session.getId());
cookie.setPath("/29-Cookie-Session");
cookie.setMaxAge(60*2);// 2分钟
response.addCookie(cookie);

session.setAttribute("lk", "it666");
```
SessionServlet2
```java
HttpSession session = request.getSession();
System.out.println(session.getAttribute("lk"));// it666
```

## 参考资料

[Java零基础到高级JavaWeb与项目](https://study.163.com/course/introduction/1005981003.htm)