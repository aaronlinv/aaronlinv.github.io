---
title: MyBatis笔记
categories: MyBatis笔记
date: 2020-03-22 17:23:19
---
MyBatis执行流程：
1. 创建 SqlSessionFactoryBuilder
2. 通过 SqlSessionFactoryBuilder 读取 SqlMappingConfig.xml 核心配置文件（还需要加载映射文件），创建 SqlSessionFactory
3. 获取 Session
4. 执行 Sql语句
5. 关闭 Session

注意点：
1. Session线程不安全，不能定义为字段
2. 映射文件中 resultType 或 parameterType 为domian 值需要写domain的全限定类名(或用 typeAliases 定义别名)
3. 涉及修改数据库的操作都需要commit
4. 不同操作在映射文件中写不同标签 select updata delete
5. 返回添加过后自增的主键的写法

#{} 和 ${} 区别
1. 都可以${}可以接收简单类型值或pojo属性值
2. #{}：内部用?作为占位符，执行的时候会给参数两边加单引号，可以防止sql注入，可以实现preparedStatement向占位符中设置值，自动进行Java类型和JDBC类型转换
3. ${}：表示拼接sql串，将parameterType 传入的内容拼接在sql中且**不进行jdbc类型转换**
4. parameterType传输单个简单类型值，${}括号中只能为value即：${value}，而#{}括号中**任意**
<br/>


传统方式写DAO
1. 定义接口
2. 写接口实现类，在实现类中调用 SqlSession 执行对于语句

使用Mapper动态代理，避免写实现类，需要满足要求
1. namespace 必须为 Mapper接口全限定类名
2. id 必须和 Mapper接口方法名一致
3. parameterType 与接口方法参数类型一致
4. resultType 与接口方法返回值类型一致
<br/>

传递多个参数
1. 在映射文件的sql语句中使用#{arg0} #{arg1} 或 #{param1} #{param2} 
2. 在接口方法参数加注解@Param("")  在#{}中使用定义好的名称**或 #{param1} #{param2}**
3. 传入 Map<String,Object> put键值对 在#{}中使用定义好的名称
4. POJO类 在#{}中使用POJO类字段名

输出类型
1. 简单类型
2. Map<String,Object> 键值对 resultType="map"
3. Map<Integer,Customer> 需要在接口方法指定Map的键 @MapKey("") resultType为bean
4. resultMap：表字段名与POJO类属性名不一致（resultMap中的id标签指定为主键）


MyBatis核心配置文件
- properties
- settings: mapUnderscoreToCamelCase sql打印
- typeAliases: 可批量定义，与子包类名冲突，在类上使用注解@Alias("别名")
- typeHandlers
- plugins
- environments
- databaseIDProvider
- mappers:指定package时mapper接口名称和mapper映射文件名称需相同，且放在同一个目录


多表查询
- 表设置外键约束
- bean属性需要引用另一个bean
- 连接表后需要用 resultMap 封装数据：直接把连接后的所有列写在resultMap里或**写在association标签里**

分步查询：在association标签的select属性写上查询的接口方法




