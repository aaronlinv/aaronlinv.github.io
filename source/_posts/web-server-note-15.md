---
title: Web服务器笔记-分页
categories: Web服务器笔记
date: 2020-02-23 13:48:47
---
封装PageBean，减少写入域中的数据
```java
package com.myxq.domain;

@Getter@Setter
public class PageBean {
    // 当前是哪一页
    private Integer currentPage;
    // 共多少页
    private Integer totalPage;
    // 多少条记录
    private Integer totalCount;
    // 当前页商品
    private List<Goods> goodsList = new ArrayList<>();
}

```
修改admin_index.jsp中main的指向
```jsp
        <frame src="${pageContext.request.contextPath }/GoodsServlet?action=getPageDate&currentPage=1" name="mainFrame" >
```

```java
public String getPageDate(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		try {
			// 1.获取当前页码
			String currentPage = request.getParameter("currentPage");
			// 2.把页码给业务层 返回一个pageBean
			GoodsService goodsService = new GoodsService();
			PageBean pageBean = goodsService.getPageBean(Integer.parseInt(currentPage));

			// 3.把pageBean写到域中
			request.setAttribute("pageBean",pageBean);
			// 4.转发
			return "admin/main.jsp";
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}
```


```java
	public PageBean getPageBean(Integer currentPage) throws SQLException {
		PageBean pageBean = new PageBean();
		// 设置当前页
		pageBean.setCurrentPage(currentPage);
		// 获取有多少条记录
		// 从数据库中查询
		Long count = goodsDao.getCount();
		pageBean.setTotalCount(count.intValue());
		// 一页展示多少条数据
		Integer pageCount = 5;

		// 总页数 // 16/5=3  1 所以要向上取整
		// 注意两个Integer相除只会整数
		double totalPage = Math.ceil(1.0 * pageBean.getTotalCount() / pageCount);
		pageBean.setTotalPage((int)totalPage);

		// 当前页商品的角标
		Integer index = (pageBean.getCurrentPage()-1)*pageCount;
		List<Goods> list  = goodsDao.getPageDate(index,pageCount);

		pageBean.setGoodsList(list);
		return pageBean;
	}
```

```java
	public Long getCount() throws SQLException {
		String sql = "select count(*) from goods";
		Long count = (Long) qr.query(sql,new ScalarHandler());
		return count;

	}

	public List<Goods> getPageDate(Integer index, Integer pageCount) throws SQLException {
		String sql = "select * from goods limit ?,?";
		List<Goods> pageGoods = qr.query(sql, new BeanListHandler<Goods>(Goods.class), index, pageCount);
		return pageGoods;
	}
```

查询结果就一个使用ScalarHandler结果集
查询结果是一个Long类型，不能强转为Integer，因为**包装类型不能用括号强转**，要调用用方法才能转换

修改getListGoods方法



修改分页
```jsp
		<c:forEach items="${pageBean.goodsList }" var="goods" varStatus="status">


//分页<script>
		//分页
		$("#page").paging({
			pageNo : ${pageBean.currentPage},
			totalPage : ${pageBean.totalPage},
			totalSize : ${pageBean.totalCount},
			callback : function(num) {
				// alert(num);
				// 重新发送请求
				$(window).attr('location', '${ctx}/GoodsServlet?action=getPageDate&currentPage='+num);
				// ${pageContext.request.contextPath }/GoodsServlet?action=getPageDate&currentPage=1
			}
		});

	</script>
```

## 参考资料

[Java零基础到高级JavaWeb与项目](https://study.163.com/course/introduction/1005981003.htm)