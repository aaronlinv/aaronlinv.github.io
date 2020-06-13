---
title: Web服务器笔记-EL/JSTL
categories: Web服务器笔记
date: 2020-02-16 14:29:34
---
## EL表达式
EL (Express Lanuage)表达式可以嵌入在jsp页面内部
减少jsp脚本的编写
EL出现的目的是要替代jsp页面中脚本的编写

#### EL最主要的作用是获得四大域中的数据
- pageContext
${pageScope.key};
- request
${requestScope.key}
- session
${sessionScope.key}
- application
${applicationScope.key}

```jsp

<%
	pageContext.setAttribute("name", "pageContextValue");
	request.setAttribute("name", "requestValue");
	session.setAttribute("name", "sessionValue");
	application.setAttribute("name", "applicationValue");
%>
${pageScope.name }
<br/>
${requestScope.name }
<br/>
${sessionScope.name }
<br/>
${applicationScope.name }
```

#### 简写
${EL表达式}
EL从四个域中获得某个值${key}
依次从pageContext域，request域，session域，application域中获取属性
在某个域中获取后将不在向后寻找(从小到大)
				
#### EL内置11对象（基本不用，只用pageContext）
- pageScope
获取JSP中pageScope域中的数据
- requestScope
获取JSP中requestScope域中的数据
- sessionScope
获取JSP中sessionScope域中的数据
- applicationScope
获取JSP中applicationScope域中的数据
- param
request.getParameter()
- paramValues
rquest.getParameterValues()
- header
request.getHeader(name)
- headerValues
request.getHeaderValues()
- initParam
this.getServletContext().getInitParameter(name)
- cookie	
request.getCookies()---cookie.getName()---cookie.getValue()
- pageContext
pageContext获得其他八大对象
**获取当前项目的名称：${pageContext.request.contextPath}**

##### EL执行表达式
内部可以进行运算，只要有结果
```jsp
${1+1}
${empty user} 
${user==null?true:false}
```
判读user是否为空 输出true

## JSTL 
(JSP Standard Tag Library) JSP标准标签库
可以嵌入在JSP页面中使用标签的形式完成业务逻辑等功能
JSTL出现的目的同EL一样也是要代替JSP页面中的脚本代码

#### JSTL标准标签库有5个子库
- Core :核心库(其他的库都过时了)
http://java.sun.com/jsp/jstl/core
前缀：c
- I18N：国际化库
http://java.sun.com/jsp/jstl/fmt
前缀：fmt
- SQL
http://java.sun.com/jsp/jstl/sql
前缀：sql
- XML
http://java.sun.com/jsp/jstl/xml
前缀：x
- Functions
http://java.sun.com/jsp/jstl/functions
前缀：fn

#### 使用
1. 把JSTL标签库jar包引入工程当中 jstl-1.2.jar
2. 引入标签库
```jsp
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>
```

#### 标签
if标签
```jsp
<c:if test="${1==1 }">满足条件时，中间的内容才会显示出来</c:if>
```
通过是结合EL表达式一起使用，EL从域中取数据，使用JSTL进行判断或者遍历
没有else所有只能
        
if标签使用场景：
用户登录成功时，进入首页中，显示用户名
步骤
1.登录成功时，把用户写到session域当中，修改LogginServlet
```java
if (u != null) {
    response.getWriter().write("登录成功");
    // 
    HttpSession session = request.getSession();
    session.setAttribute("user", u);
    //
    response.setHeader("refresh", "2;url=/31-Mystore/index.jsp");
} else {
    response.getWriter().write("登录失败");
    response.setHeader("refresh", "2;url=/31-Mystore/login.jsp");
}
```
2.在Header.jsp中进行判断，从session域当中取数据，先引入标签库
3.通过EL结合JSTL进行判断
```html
<div class="header_top_center">
    <div class="h_top_left">欢迎来到码蚁商城</div>
    <div class="h_top_right">
        <!-- 判断有没有用户 session -->
        <c:if test="${empty user}">
            <a href="login.jsp">登录</a>
            <a href="regist.jsp">免费注册</a>
        </c:if>
        
        <c:if test="${!empty user }">
            <!-- 或者直接${ user.username } -->
            欢迎：${user.getUsername() }
            <a href="#">退出</a>
        </c:if>
        
        
        <a href="#">购物车</a> <a href="#">我的订单</a>
    </div>
</div>
```

            
foreach标签
- 普通循环:从域当中取数据，自动把数据存储pagescope
```jsp
<c:forEach begin="0" end="5" var="i">
	${i }
</c:forEach>
<!--等同于 ${pageScope.i} -->
```
- 第二种：增加for循环
遍历字符串集合
```jsp
<%
List<String> strList = new ArrayList<String>();
strList.add("aaa");
strList.add("bbb");
session.setAttribute("strList", strList);
%>
<c:forEach items="${strList}" var="str"> 
	${str }
</c:forEach>
```

遍历对象集合
```jsp
<%
List<User> userList = new ArrayList<User>();
User u1 = new User();
u1.setUsername("zs");
userList.add(u1);

User u2 = new User();
u2.setUsername("ls");
userList.add(u2);

session.setAttribute("userList", userList);
%>
<c:forEach items="${userList}" var="user"> 
	${user.username }
</c:forEach>
```

遍历map
```jsp
<%
Map<String ,String> strMap = new HashMap<String,String>();
strMap.put("name","zs");
strMap.put("age", "30");
session.setAttribute("strMap", strMap);
%>
<c:forEach items="${strMap }" var = "entry">
	${entry.key}:${entry.value}
</c:forEach>
```

商品列表展示
修改goods_list.jsp
```jsp
<c:if test="${empty allGoods }">
    没有商品
</c:if>


<c:forEach items="${allGoods }" var="goods">
    <li>
    <a href="#">
        <img src="images/pimages/${goods.image }" alt="">
        <p>${goods.name }</p>
        <i id="yuan">￥</i> <span id="price">${goods.price }</span>
    </a>
    </li>
</c:forEach>
```
		
## 参考资料

[Java零基础到高级JavaWeb与项目](https://study.163.com/course/introduction/1005981003.htm)