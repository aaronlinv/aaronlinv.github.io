---
title: SpringMVC笔记
categories: SpringMVC笔记
date: 2020-03-01 16:40:23
---
## 上手实践
1. WEB-INF/lib/ 添加Spring相关JAR包 
2. src或resources下配置springmvc.xml (注意：1.要开启对包的注解扫描 2.可配置在返回地址上添加前缀和后缀(视图解析器InternalResourceViewResolver) 3.配置静态资源访问）
3. web.xml 配置DispatcherServlet前端控制器(注意：1.配置springmvc.xml位置2.load-on-startup标签 Tomcat启动时加载DispatcherServlet 3.SpringMVC的拦截路径 4.配置编码过滤器解决post提交表单乱码)
4. 创建类 @Controller @RequestMapping

## 注意点
1. 需要传递参数返回ModelAndView 不传递参数可以返回String
2. 重定向：返回字符串前加redirect:
3. @RequestMapping匹配请求.action可省略；浏览器、重定向：redirect 发送请求不可省略
4. 配置 mvc:view-controller 可设置跳转的页面 (需要添加mvc:annotation-driven 否则不会加载RequestMappingHandlerAdapter 会导致@RequestMapping失效)


web.xml中SpringMVC的url-patten取值
- / :拦截所有,不包括jsp，**包含.js .png .css** (推荐) 
- /* :拦截所有:jsp .js .png .css  
- *.action *.do: 拦截以do action 结尾的请求
	
## 数据绑定
1. 通过传入 HttpServletRequest request 可获取网页传递数据 getParameter
2. 请求参数名称与处理器形参名称一致时会将请求参数与形参进行绑定（表单也可以）
3. 使用@RequestParam(value = "") 指定请求参数，其他参数：required,defaultValue 
4. 传递Bean要求表单和属性一致
5. 表单name属性相同封装数组
6. 包装类绑定数据需要使用：类名.属性名，包装类List使用：类名[n].属性名 (n>=0)

自定义Date参数绑定
1. 网页传输的参数都是String类型，由转换器转换为对应类型，转换失败404(Date默认格式yyyy/MM/dd)
2. 实现 Converter<String,Date> 类，实现convert方法
3. springmvc.xml 注入转换器，添加注解驱动

RequestMapping参数
1. value 设置多个路径共同访问对应方法
2. method 限制请求方式
3. params 限制请求参数和请求值
4. headers 限制请求头

@RequestHeader
@CookieValue


Ant URL风格
1. 一个?匹配一个字符
2. *匹配任意字符
3. **匹配多重路径

REST 资源定位及资源操作风格
1. URL定位资源，HTTP描述操作(POST,GET,PUT,DELETE 分别对应 CRUD)
2. RequestMapping方法参数使用 @PathVariable 接收路径参数 
3. 默认情况下Form表单是不支持PUT请求和DELETE请求的（POST请求转换为PUT或DELETE请求：1.配置过滤器 2.表单添加_method字段）
4. Tomcat 8 开始 JSPs 只允许GET,POST,HEAD 需要重定向，否则404


mvc:annotation-driven
会自动注册三个Bean
- RequestMappingHandlerMapping
- RequestMappingHandlerAdapter
- ExceptionHandlerExceptionResolver


SpringMVC form
- input
- radiobutton
- checkboxes
- select
- error (配合Hibernate Validator)

## JSON
前端使用Ajax发送请求时，服务器以JSON的数据格式响应给浏览器
1. 方法加 @ResponseBody 注解返回JSON对象
2. 可以返回对象、Map、List
3. 表单序列化 方法参数使用@RequestBody 注解接收浏览器发送的JSON
4. 默认情况发送接收都是简单Key-Value形式（无法发送复杂类型如Array），要指定发送JSON
5. 需要解决 JSON.stringify() 不会自动解析，识别单个值为字符串，而不是数组的问题

记：如果没有在接收JSON的RequestMapping方法前没有加 @ResponseBody 注解
服务器可以接收到JSON，方法返回字符串，对应的xhr响应 404
```java
    @RequestMapping("/testJson")
    @ResponseBody
    public String testJson(@RequestBody User user){
        System.out.println(user);
        return "success";
    }
```
服务器可以接收到JSON，如果方法返回对象，对应的xhr响应 500
```java
    @RequestMapping("/testJson2")
    @ResponseBody
    public User testJson2(@RequestBody User user) {
        System.out.println(user);
        return user;
    }
```

## 值传递
Request域：一次请求过程中传递处理的数据
1. RequestMapping方法参数传入 ModeAndView,Model,Map,Bean 数据都是写入Request域
2. 键值对、domian(也是放在键值对里，键：类名小写，值：对象)、HashMap、List（元素包装类型小写为key）
3. mergeAttributes：Map键相同不会覆盖 
4. containsAttribute：是否包含属性

HttpSession：多个请求之间可以共享这个属性
类上添加@SessionAttributes 指定需要存放到Session域中的key值或者类型
方法参数也可用用这个注解@SessionAttributes

RequestMapping方法参数内使用@ModelAttribute 可以给修改存入Request域的键值
@ModelAttribute注解注释的方法会在此controller每个方法执行前被执行

方法参数内使用@ModelAttribute修饰的Bean 会覆盖掉已存在的属性值（表单没有传递这个属性，那么还继续用ModelAttribute中定义的属性值）

ModelAttribute写的内容会真覆盖Session内容


## 其他
下载
文件名中文编码问题
上传

异常处理
1. 先按继承关系最近的 带@ExceptionHandler注解的方法
2. 没找到再找带 @ControllerAdvice 的类

国际化

拦截器

如果发生异常
    处理器方法执行之前调用 preHandle
    / by zero
    请求处理完成之后 afterCompletion

多个拦截器执行顺序
- preHandle 按照 springmvc.xml 配置的顺序执行
- postHandle,afterCompletion 都是倒叙执行
## SpringMVC执行过程
1. 用户发送请求至 前端控制器DispatcherServlet
2. DispatcherServlet 收到请求调用 HandlerMapping处理器映射器
3. 处理器映射器 根据请求url找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet
4. DispatcherServlet 通过 HandlerAdapter处理器适配器 调用处理器
5. 执行处理器(Controller，也叫后端控制器)
6. Controller 执行完成返回 ModelAndView
7. HandlerAdapter 将 Controller执行结果 ModelAndView返回给 DispatcherServlet
8. DispatcherServlet 将 ModelAndView 传给 ViewReslover视图解析器
9. ViewReslover 解析后返回具体View
10. DispatcherServlet 对View进行渲染视图（即将模型数据填充至视图中）
11. DispatcherServlet 响应用户

自定义部分：前端控制器DispatcherServlet、处理器Handler、视图View

HandlerMapping：映射@RequestMapping注解，映射成功返回HandlerMethod
HandlerAdapter：对@RequestMapping方法进行适配，解析对应方法
ViewReslover：在springmvc.xml中配置视图解析器，给返回地址添加前缀和后缀


## 参考资料
[SpringMvc接受json参数报404问题解决](https://blog.csdn.net/little_pig_lxl/article/details/88249255)