---
title: Web服务器笔记-通用Servlet
categories: Web服务器笔记
date: 2020-02-22 19:07:33
---
#### 通用Servlet
定义个请求都要去写一个Servlet，一个商品模块就写了6个Servlet
一个工程当中会有很多的Servlet

#### 解决办法
在一个模块当中只写一个Servlet，在跳转到Servlet时，根据不同的操作，传入一个特定的参数
在Servlet当中接收特定的参数，根据参数的值不同，跳转到不同的Servlet中

实现步骤
1. 假设有一个页面里面有四个请求
```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
	<a href="${pageContext.request.contextPath }/TestServlet?action=add">添加</a>
	<a href="${pageContext.request.contextPath }/TestServlet?action=del">删除</a>
	<a href="${pageContext.request.contextPath }/TestServlet?action=update">更新</a>
</body>
</html>
```
2. 定义一个TestServlet接收请求
3. 定义各自的业务方法
```java
package com.myxq;

@WebServlet("/TestServlet")
public class TestServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;

	protected void service(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("TestServlet");
		String action = request.getParameter("action");
		if ("add".equals(action)) {

			add(request, response);
		} else if ("del".equals(action)) {

			del(request, response);
		} else if ("update".equals(action)) {

			update(request, response);
		}
	}

	protected void add(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		System.out.println("add");
	}

	protected void del(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		System.out.println("del");
	}

	protected void update(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("update");
	}
}
```
```java
if ("add".equals(action)) 
// 这样可以避免action为空，空指针异常，通常用常量调用equals
```

#### 改进
每一个业务中都有一个转发，只是转发的路径不同
把转发拿到service中处理，调用各自方法时，返回跳转的路径
```java
package com.myxq;

@WebServlet("/TestServlet")
public class TestServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;

	protected void service(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("TestServlet");
		String desPath = null;
		
		String action = request.getParameter("action");
		if ("add".equals(action)) {

			desPath = add(request, response);
		} else if ("del".equals(action)) {

			desPath = del(request, response);
		} else if ("update".equals(action)) {

			desPath = update(request, response);
		}
		
		if (desPath != null) {
			request.getRequestDispatcher(desPath).forward(request, response);
		}
	}

	protected String add(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		System.out.println("add");
		return "add.jsp";
	}

	protected String del(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
		System.out.println("del");
		return "del.jsp";
	}

	protected String update(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("update");
		return "update.jsp";
	}
}
```

#### 改进2
service当中if判断比较多，使用反射
接收参数之后，查看当前字节码当中有没有和参数相同的方法
如果有，就动态的来去调用
注意反射直接invoke的方法必须是public的
```java
package com.myxq;

@WebServlet("/TestServlet2")
public class TestServlet2 extends HttpServlet {
	private static final long serialVersionUID = 1L;

	protected void service(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("TestServlet");
		String desPath = null;

		String action = request.getParameter("action");

		// 1.获取字节码
		try {
			Class clazz = this.getClass();
			Method method = clazz.getMethod(action, HttpServletRequest.class, HttpServletResponse.class);
			// 判断有没有传入的方法
			if(method != null) {
				// 使用当前对象调用
				desPath = (String) method.invoke(this, request,response);
			}
			// 转发
			if (desPath != null) {
				request.getRequestDispatcher(desPath).forward(request, response);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}

	}

	public String add(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("add");
		return "add.jsp";
	}

	public String del(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("del");
		return "del.jsp";
	}

	public String update(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("update");
		return "update.jsp";
	}
}
```

#### 抽取基类
把service方法单独存放到一个类当中
以后别人在使用的时候，只需要提供对应的方法

BaseServlet
```java
package com.myxq;

@WebServlet("/BaseServlet")
public class BaseServlet extends HttpServlet {
	private static final long serialVersionUID = 1L;

	protected void service(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("TestServlet");
		String desPath = null;

		String action = request.getParameter("action");

		// 1.获取字节码
		try {
			Class clazz = this.getClass();
			Method method = clazz.getMethod(action, HttpServletRequest.class, HttpServletResponse.class);
			// 判断有没有传入的方法
			if (method != null) {
				// 使用当前对象调用
				desPath = (String) method.invoke(this, request, response);
			}
			// 转发
			if (desPath != null) {
				request.getRequestDispatcher(desPath).forward(request, response);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}

	}
}
```

TestServlet2继承BaseServlet
```java
package com.myxq;

@WebServlet("/TestServlet2")
public class TestServlet2 extends BaseServlet{
	private static final long serialVersionUID = 1L;

	// 进入方法，找service方法，没找到，到父类中查找

	public String add(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("add");
		return "add.jsp";
	}

	public String del(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("del");
		return "del.jsp";
	}

	public String update(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("update");
		return "update.jsp";
	}
}
```

## 通用Servlet改进项目

创建GoodsServlet 继承 BaseServlet
```java
package com.myxq.web;


@WebServlet("/GoodsServlet")
public class GoodsServlet extends BaseServlet {
}

```
修改jsp中的Servlet指向，链接到GoodsServlet，加上指定的action
在GoodsServlet中添加对应的public方法，返回转发的页面String，如果需要处理异常，则在最后return null

GoodsServlet
```java
package com.myxq.web;

@WebServlet("/GoodsServlet")
public class GoodsServlet extends BaseServlet {
	public String getListGoods(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		System.out.println("GoodsListServlet");
		// 调用服务层
		GoodsService goodsService = new GoodsService();
		try {
			List<Goods> allGoods = goodsService.getAllGoods();

			// 反转
			Collections.reverse(allGoods);

			// System.out.println(allGoods);
			// 把数据写到request域
			request.setAttribute("allGoods", allGoods);
			// 转发
			return "admin/main.jsp";
		} catch (SQLException e) {
			e.printStackTrace();
		}
		return null;
	}

	public String delGoods(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		// 1. 接收参数
		String id = request.getParameter("id");
		// System.out.println(id);
		// 2. 调用服务层
		GoodsService goodsService = new GoodsService();

		try {
			goodsService.deleteGoods(id);
			return "GoodsServlet?action=getListGoods";
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;

	}

	public String editGoodsUI(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		String id = request.getParameter("id");
		// System.out.println("GoodsEditUIServlet"+id);
		try {
			// 1.获取当前商品
			GoodsService goodsService = new GoodsService();
			Goods goods = goodsService.getGoodsWithId(id);
			System.out.println(goods);
			// 把商品写入request域中
			request.setAttribute("goods", goods);

			// 2.获取所有分类
			CategoryService categoryService = new CategoryService();
			List<Category> allCategory = categoryService.findCategory();
			request.setAttribute("allCategory", allCategory);
			// 3.转发到JSP页面
			return "/admin/edit.jsp";
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	public String editGoods(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		// 设置字符集 post
		request.setCharacterEncoding("utf-8");
		// System.out.println("GoodsEditServlet");
		// 1.获取所有参数
		Map<String, String[]> parameterMap = request.getParameterMap();
		// 2.封装成Goods对象
		Goods goods = new Goods();
		try {
			BeanUtils.populate(goods, parameterMap);
			System.out.println(goods);
			// 3.根据id更新数据
			// 4.调用服务层 更新数据
			GoodsService goodsService = new GoodsService();
			goods.setImage("goods_10.png");
			goodsService.updateGoods(goods);

			// 5.跳转mian.jsp
			return "GoodsServlet?action=getListGoods";
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return null;
	}

	public String addGoodsUI(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		// 1.取出所有分类
		List<Category> allCategory = null;
		CategoryService categoryService = new CategoryService();
		try {
			allCategory = categoryService.findCategory();
			System.out.println(allCategory);
			// 2.把分类存到域中
			request.setAttribute("allCategory", allCategory);
			// 3.转发edit.jsp
			return "admin/add.jsp";
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	public String addGoods(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		// System.out.println("GoodsAddServlet");
		// 1.获取所有参数
		request.setCharacterEncoding("utf-8");
		Map<String, String[]> map = request.getParameterMap();
		// System.out.println(map);
		Goods goods = new Goods();
		BeanUtils beanUtils = new BeanUtils();
		try {
			BeanUtils.populate(goods, map);
			goods.setImage("goods_11.png");
			System.out.println(request.getParameter("name"));
			System.out.println(goods);
			// 调用服务层
			GoodsService goodsService = new GoodsService();
			goodsService.addGoods(goods);

			// 跳转列表 取最新数据
			return "GoodsServlet?action=getListGoods";
		} catch (Exception e) {
			e.printStackTrace();
		}

		return null;
	}
}
```
出现了一个问题：在子类中添加编码，post表单接收到的还是乱码，可以把设置编码写到BaseServlet中
```java
request.setCharacterEncoding("utf-8");
```
## 参考资料

[Java零基础到高级JavaWeb与项目](https://study.163.com/course/introduction/1005981003.htm)