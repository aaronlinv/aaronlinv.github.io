---
date: '2023-08-22T08:48:33+08:00'
title: 'SpringBoot 测试实践 - 2：单元测试与集成测试'
categories: ["Spring"]
---



上一节：[SpringBoot 测试实践 - 1：常用的工具](https://www.cnblogs.com/aaronlinv/p/17645009.html)
下一节：[SpringBoot 测试实践 - 3：@MockBean、@SpyBean 、提升测试运行速度、Testcontainer](https://www.cnblogs.com/aaronlinv/p/17652720.html)

<br>

---

## 单元测试 vs. 集成测试

只编写单测，无法测试方法之间的集成情况，而且某些需求可能会修改多个方法，这可能会影响方法对应的单测，涉及到大量的相关单测的修改，这样的维护成本很高

可以把重心放在完善集成测试上，专注从外部判断程序是否符合预期。对于一些非常重要的方法，增加单元测试可以减轻集成测试排查错误的难度

先导知识可以参考上一节：[SpringBoot 测试实践 - 1：常用的工具](https://www.cnblogs.com/aaronlinv/p/17645009.html)

## SpringBootTest 和 MockMvc 进行集成测试

从 [Spring Boot 2.1](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.1-Release-Notes#junit-5) 开始 `@ExtendWith({SpringExtension.class})` 作为元注解包含在 Spring Boot 测试注解中，例如 @DataJpaTest、@WebMvcTest 和 @SpringBootTest，所以我们不用重复添加 `@ExtendWith({SpringExtension.class})` 注解

### HelloWorld 测试

使用 SpringBoot 一个简单的 HelloWorld 案例，通过 `@SpringBootTest` 可以在测试环境中加载整个 Spring 应用程序上下文，`@SpringBootTest` 注解会扫描应用程序的主配置类，并加载所有的 Bean（包括依赖的 Bean）到测试上下文中。这样，测试中就可以使用完整的 Spring 功能，包括依赖注入、AOP、事务管理等


使用 `@AutoConfigureMockMvc` 自动配置 `MockMvc`，通过 `MockMvc` 可以模拟 HTTP 请求，并对响应的结果进行断言和验证

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

@AutoConfigureMockMvc
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class MySpringBootTest {

    @Autowired
    private MockMvc mockMvc; // 注入 MockMvc

    @Test
    public void testHelloWorld() throws Exception {
        // 发送 GET 请求
        mockMvc.perform(MockMvcRequestBuilders.get("/hello")
                // 设置请求头
                .accept(MediaType.APPLICATION_JSON))
                // 验证响应状态码
                .andExpect(MockMvcResultMatchers.status().isOk())
                // 验证响应内容
                .andExpect(MockMvcResultMatchers.content().string("Hello, World!"));
    }
}
```

### 涉及数据层的测试：H2

部分操作涉及到数据库，一般都会引入数据层的依赖，在对应的 HTTP 请求后，对响应体和数据库数据进行断言和验证，就像下面这样：

```java
@AutoConfigureMockMvc
@Transactional
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private UserRepository userRepository;

    @Test
    void createUser() throws Exception {
        User user = new User("张三", 20);
        mockMvc.perform(post("/users")
                .andDo(print())
                .contentType(MediaType.APPLICATION_JSON)
                .content(new ObjectMapper().writeValueAsString(user)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").isNotEmpty())
                .andExpect(jsonPath("$.name").value(user.getName()))
                .andExpect(jsonPath("$.age").value(user.getAge()));

        assertThat(userRepository.count()).isEqualTo(1);
    }

    @Test
    void getUser() throws Exception {
        User user = userRepository.save(new User("张三", 20));

        mockMvc.perform(get("/users/{id}", user.getId()))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(user.getId()))
                .andExpect(jsonPath("$.name").value(user.getName()))
                .andExpect(jsonPath("$.age").value(user.getAge()));
    }

    @Test
    void deleteUser() throws Exception {
        User user = userRepository.save(new User("张三", 20));

        mockMvc.perform(delete("/users/{id}", user.getId()))
                .andExpect(status().isOk());

        assertThat(userRepository.findById(user.getId())).isEmpty();
    }
}
```

涉及到数据库的集成测试一般都会对数据库进行增删改，可能会影响到测试环境的数据库，所以在测试中有一种方式是用 H2 内存数据库代替测试环境数据库进行测试，但是 H2 数据库和我们实际使用的 MySQL 或 PostgreSQL 不同，所以它有些问题：

1. SQL 语法不同，初始化测试环境需要使用 SQL 脚本，要将实际数据库的脚本转化为符合 H2 语法的 SQL 脚本
3. 与实际数据库差异在断言的时候也可能导致一些奇奇怪怪的问题，浮点数精度不一致等等的
2. 需要额外进行一些配置，方言 (Dialect) 等等的

故不推荐这种方式，对这种方式感兴趣可以参考这篇文章：
[How to Write Integration Tests with H2 In-Memory Database and Springboot](https://1kevinson.com/how-to-write-integration-tests-with-h2-in-memory-database-and-springboot/)


代码可以可参考这个仓库，提供了集成测试以及不同层的测试：[spring-boot-fullstack-professional](https://github.com/amigoscode/spring-boot-fullstack-professional/blob/13-testing/src/test/java/com/example/demo/integration/StudentIT.java)

### 使用实际数据库完成单元测试

直接本地部署与线上相同类型的数据库，并创建好对应的库，指定所有测试都使用这个库，这样就可以解决 H2 兼容性的问题

这里以 [若依 v3.8.6](https://github.com/yangzongzhuan/RuoYi-Vue-fast/tree/v3.8.6) 为例：

1. 本地部署 MySQL 并且新建一个测试专用的库：`ry_vue_test`，在代码中增加 `application-test.yml`：

```yaml
spring:
  datasource:
    # UTC+0
    url: jdbc:mysql://localhost:3306/ry_vue_test?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
    username: root
    password: ^PsL)H~
    driver-class-name: com.mysql.cj.jdbc.Driver
```

2. 增加 `MyBatis` 测试依赖

```xml
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter-test</artifactId>
            <version>2.2.0</version>
            <scope>test</scope>
        </dependency>
```

3. 先写一个针对 [com.ruoyi.project.system.mapper.SysUserMapper](https://github.com/yangzongzhuan/RuoYi-Vue-fast/blob/v3.8.6/src/main/java/com/ruoyi/project/system/mapper/SysUserMapper.java) 的数据层测试：

**注意！不熟悉流程的情况下建议先注释测试数据库的地址，避免测试代码影响到测试库数据**

```java
package com.ruoyi.project.system.mapper;

import com.ruoyi.framework.config.ApplicationConfig;
import com.ruoyi.project.system.domain.SysUser;
import org.junit.jupiter.api.Test;
import org.mybatis.spring.boot.test.autoconfigure.MybatisTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.jdbc.Sql;

import static org.assertj.core.api.AssertionsForInterfaceTypes.assertThat;
import static org.junit.jupiter.api.Assertions.*;
import static org.springframework.test.context.jdbc.Sql.ExecutionPhase.BEFORE_TEST_METHOD;

// 自动配置 MyBatis 相关的组件，并创建一个测试专用的数据库会话。这样可以在测试中使用 MyBatis 的功能
@MybatisTest
// 指定了测试上下文的配置类，这样可以加载应用程序的配置，并将其用于测试中的依赖注入
@ContextConfiguration(classes = {ApplicationConfig.class})
// 指定了要激活的配置文件，对应上面创建的 `application-test.yml`
@ActiveProfiles("test")
// 禁用了自动配置替换测试数据库的功能，测试将使用真实的数据库进行操作，而不是使用内存数据库或其他替代数据库
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
// 在每个测试方法执行之前都运行该脚本：初始化每个测试用例的数据库初始数据
@Sql(value = "classpath:sql/sys_user.sql", executionPhase = BEFORE_TEST_METHOD)
class SysUserMapperTest {
    @Autowired
    private SysUserMapper userMapper;

    @Test
    void selectUserById() {
        SysUser sysUser = userMapper.selectUserById(1L);
        assertThat(sysUser).isNotNull();
        assertThat(sysUser.getUserName()).isEqualTo("admin");
        // 添加更多的断言
    }
}
```

这里必须配置 `@ContextConfiguration(classes = {ApplicationConfig.class})`，因为 [ApplicationConfig.class](https://github.com/yangzongzhuan/RuoYi-Vue-fast/blob/v3.8.6/src/main/java/com/ruoyi/framework/config/ApplicationConfig.java) 包含了 Mapper 的扫描配置：`@MapperScan("com.ruoyi.project.**.mapper")`

复盘一下整个测试运行的流程：

1. 自动配置 MyBatis 相关的组件，创建一个测试专用的数据库会话
2. 加载 ApplicationConfig 类作为测试上下文的配置
3. 激活名为 "test" 的配置文件，将其用于测试中的依赖注入，这里包含 MySQL 的相关配置
4. 禁用自动配置替换测试数据库的功能，使用真实的数据库进行操作，即上面 "test" 的 MySQL
5. 在每个测试方法执行之前，运行位于类路径下的 "sql/sys_user.sql" SQL 脚本，完成用户数据的初始化
6. 自动注入 SysUserMapper 对象，该对象是需要进行测试的 MyBatis 映射器
7. 执行 `selectUserById(1L)`，查询数据并执行断言


这个流程不仅可以测试数据层，还可以直接用于测试服务层。因为 `@MybatisTest` 不需要启动完整的应用程序上下文，使得整个测试非常快。具体可以参考：[MyBatis 测试文档](https://mybatis.org/spring-boot-starter/mybatis-spring-boot-test-autoconfigure/zh/index.html)

### 使用实际数据库完成集成测试

与上面 `@MybatisTest` 不同的是：集成测试使用 `@SpringBootTest`，这个注解会加载整个 Spring Boot 应用程序的上下文，并配置所有的组件，它会启动整个应用程序，并模拟实际的运行环境。所以需要保证 SpringBoot 的初始化配置，否则会应用会启动失败。待测接口为：[com.ruoyi.project.system.controller.SysUserController](https://github.com/yangzongzhuan/RuoYi-Vue-fast/blob/v3.8.6/src/main/java/com/ruoyi/project/system/controller/SysUserController.java)


1. 增加用于集成测试的 Spring 配置 `application-it.yml`，需要指定好测试的数据库，最好把可能影响测试环境的配置（Redis 等的）都修为集成测试专用的环境

```yaml
# 数据源配置
spring:
    datasource:
        type: com.alibaba.druid.pool.DruidDataSource
        driverClassName: com.mysql.cj.jdbc.Driver
        druid:
            # 主库数据源
            master:
                # UTC+0
                url: jdbc:mysql://localhost:3306/ry_vue_test?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
                username: root
                password: ^PsL)H~
            # 从库数据源
            slave:
                # 从数据源开关/默认关闭
                enabled: false
                url: 
                username: 
                password: 
            # 初始连接数
            initialSize: 5
            # 最小连接池数量
            minIdle: 10
            # 最大连接池数量
            maxActive: 20
            # 配置获取连接等待超时的时间
            maxWait: 60000
            # 配置连接超时时间
            connectTimeout: 30000
            # 配置网络超时时间
            socketTimeout: 60000
            # 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒
            timeBetweenEvictionRunsMillis: 60000
            # 配置一个连接在池中最小生存的时间，单位是毫秒
            minEvictableIdleTimeMillis: 300000
            # 配置一个连接在池中最大生存的时间，单位是毫秒
            maxEvictableIdleTimeMillis: 900000
            # 配置检测连接是否有效
            validationQuery: SELECT 1 FROM DUAL
            testWhileIdle: true
            testOnBorrow: false
            testOnReturn: false
            webStatFilter: 
                enabled: true
            statViewServlet:
                enabled: true
                # 设置白名单，不填则允许所有访问
                allow:
                url-pattern: /druid/*
                # 控制台管理用户名和密码
                login-username: ruoyi
                login-password: 123456
            filter:
                stat:
                    enabled: true
                    # 慢SQL记录
                    log-slow-sql: true
                    slow-sql-millis: 1000
                    merge-sql: true
                wall:
                    config:
                        multi-statement-allow: true

    # redis 配置
    redis:
        # 地址
        host: localhost
        # 端口，默认为6379
        port: 6379
        # 数据库索引
        database: 1
        # 密码
        password:
        # 连接超时时间
        timeout: 10s
        lettuce:
            pool:
                # 连接池中的最小空闲连接
                min-idle: 0
                # 连接池中的最大空闲连接
                max-idle: 8
                # 连接池的最大数据库连接数
                max-active: 8
                # #连接池最大阻塞等待时间（使用负值表示没有限制）
                max-wait: -1ms
```

2. 创建应用初始化必须的库，否则启动容器会启动失败，提示表不存在：`; bad SQL grammar []; nested exception is java.sql.SQLSyntaxErrorException: Table 'ry_vue_test.sys_config' doesn't exist`，手动创建以下必须的表，确保应用正常启动

```sql
drop table if exists sys_config;
create table sys_config (
                            config_id         int(5)          not null auto_increment    comment '参数主键',
                            config_name       varchar(100)    default ''                 comment '参数名称',
                            config_key        varchar(100)    default ''                 comment '参数键名',
                            config_value      varchar(500)    default ''                 comment '参数键值',
                            config_type       char(1)         default 'N'                comment '系统内置（Y是 N否）',
                            create_by         varchar(64)     default ''                 comment '创建者',
                            create_time       datetime                                   comment '创建时间',
                            update_by         varchar(64)     default ''                 comment '更新者',
                            update_time       datetime                                   comment '更新时间',
                            remark            varchar(500)    default null               comment '备注',
                            primary key (config_id)
) engine=innodb auto_increment=100 comment = '参数配置表';

drop table if exists sys_job;
create table sys_job (
                         job_id              bigint(20)    not null auto_increment    comment '任务ID',
                         job_name            varchar(64)   default ''                 comment '任务名称',
                         job_group           varchar(64)   default 'DEFAULT'          comment '任务组名',
                         invoke_target       varchar(500)  not null                   comment '调用目标字符串',
                         cron_expression     varchar(255)  default ''                 comment 'cron执行表达式',
                         misfire_policy      varchar(20)   default '3'                comment '计划执行错误策略（1立即执行 2执行一次 3放弃执行）',
                         concurrent          char(1)       default '1'                comment '是否并发执行（0允许 1禁止）',
                         status              char(1)       default '0'                comment '状态（0正常 1暂停）',
                         create_by           varchar(64)   default ''                 comment '创建者',
                         create_time         datetime                                 comment '创建时间',
                         update_by           varchar(64)   default ''                 comment '更新者',
                         update_time         datetime                                 comment '更新时间',
                         remark              varchar(500)  default ''                 comment '备注信息',
                         primary key (job_id, job_name, job_group)
) engine=innodb auto_increment=100 comment = '定时任务调度表';

drop table if exists sys_dict_data;
create table sys_dict_data
(
    dict_code        bigint(20)      not null auto_increment    comment '字典编码',
    dict_sort        int(4)          default 0                  comment '字典排序',
    dict_label       varchar(100)    default ''                 comment '字典标签',
    dict_value       varchar(100)    default ''                 comment '字典键值',
    dict_type        varchar(100)    default ''                 comment '字典类型',
    css_class        varchar(100)    default null               comment '样式属性（其他样式扩展）',
    list_class       varchar(100)    default null               comment '表格回显样式',
    is_default       char(1)         default 'N'                comment '是否默认（Y是 N否）',
    status           char(1)         default '0'                comment '状态（0正常 1停用）',
    create_by        varchar(64)     default ''                 comment '创建者',
    create_time      datetime                                   comment '创建时间',
    update_by        varchar(64)     default ''                 comment '更新者',
    update_time      datetime                                   comment '更新时间',
    remark           varchar(500)    default null               comment '备注',
    primary key (dict_code)
) engine=innodb auto_increment=100 comment = '字典数据表';
```

3. 为了保证集成环境数据的一致性，我们可以把数据的初始化都放在 `sys_user_it.sql` 中，每次运行测试前都先执行

```sql
drop table if exists sys_config;
create table sys_config (
                            config_id         int(5)          not null auto_increment    comment '参数主键',
                            config_name       varchar(100)    default ''                 comment '参数名称',
                            config_key        varchar(100)    default ''                 comment '参数键名',
                            config_value      varchar(500)    default ''                 comment '参数键值',
                            config_type       char(1)         default 'N'                comment '系统内置（Y是 N否）',
                            create_by         varchar(64)     default ''                 comment '创建者',
                            create_time       datetime                                   comment '创建时间',
                            update_by         varchar(64)     default ''                 comment '更新者',
                            update_time       datetime                                   comment '更新时间',
                            remark            varchar(500)    default null               comment '备注',
                            primary key (config_id)
) engine=innodb auto_increment=100 comment = '参数配置表';

insert into sys_config values(1, '主框架页-默认皮肤样式名称',     'sys.index.skinName',            'skin-blue',     'Y', 'admin', sysdate(), '', null, '蓝色 skin-blue、绿色 skin-green、紫色 skin-purple、红色 skin-red、黄色 skin-yellow' );
insert into sys_config values(2, '用户管理-账号初始密码',         'sys.user.initPassword',         '123456',        'Y', 'admin', sysdate(), '', null, '初始化密码 123456' );
insert into sys_config values(3, '主框架页-侧边栏主题',           'sys.index.sideTheme',           'theme-dark',    'Y', 'admin', sysdate(), '', null, '深色主题theme-dark，浅色主题theme-light' );
insert into sys_config values(4, '账号自助-验证码开关',           'sys.account.captchaEnabled',    'true',          'Y', 'admin', sysdate(), '', null, '是否开启验证码功能（true开启，false关闭）');
insert into sys_config values(5, '账号自助-是否开启用户注册功能', 'sys.account.registerUser',      'false',         'Y', 'admin', sysdate(), '', null, '是否开启注册用户功能（true开启，false关闭）');
insert into sys_config values(6, '用户登录-黑名单列表',           'sys.login.blackIPList',         '',              'Y', 'admin', sysdate(), '', null, '设置登录IP黑名单限制，多个匹配项以;分隔，支持匹配（*通配、网段）');

drop table if exists sys_job;
create table sys_job (
                         job_id              bigint(20)    not null auto_increment    comment '任务ID',
                         job_name            varchar(64)   default ''                 comment '任务名称',
                         job_group           varchar(64)   default 'DEFAULT'          comment '任务组名',
                         invoke_target       varchar(500)  not null                   comment '调用目标字符串',
                         cron_expression     varchar(255)  default ''                 comment 'cron执行表达式',
                         misfire_policy      varchar(20)   default '3'                comment '计划执行错误策略（1立即执行 2执行一次 3放弃执行）',
                         concurrent          char(1)       default '1'                comment '是否并发执行（0允许 1禁止）',
                         status              char(1)       default '0'                comment '状态（0正常 1暂停）',
                         create_by           varchar(64)   default ''                 comment '创建者',
                         create_time         datetime                                 comment '创建时间',
                         update_by           varchar(64)   default ''                 comment '更新者',
                         update_time         datetime                                 comment '更新时间',
                         remark              varchar(500)  default ''                 comment '备注信息',
                         primary key (job_id, job_name, job_group)
) engine=innodb auto_increment=100 comment = '定时任务调度表';

insert into sys_job values(1, '系统默认（无参）', 'DEFAULT', 'ryTask.ryNoParams',        '0/10 * * * * ?', '3', '1', '1', 'admin', sysdate(), '', null, '');
insert into sys_job values(2, '系统默认（有参）', 'DEFAULT', 'ryTask.ryParams(\'ry\')',  '0/15 * * * * ?', '3', '1', '1', 'admin', sysdate(), '', null, '');
insert into sys_job values(3, '系统默认（多参）', 'DEFAULT', 'ryTask.ryMultipleParams(\'ry\', true, 2000L, 316.50D, 100)',  '0/20 * * * * ?', '3', '1', '1', 'admin', sysdate(), '', null, '');

drop table if exists sys_dict_data;
create table sys_dict_data
(
    dict_code        bigint(20)      not null auto_increment    comment '字典编码',
    dict_sort        int(4)          default 0                  comment '字典排序',
    dict_label       varchar(100)    default ''                 comment '字典标签',
    dict_value       varchar(100)    default ''                 comment '字典键值',
    dict_type        varchar(100)    default ''                 comment '字典类型',
    css_class        varchar(100)    default null               comment '样式属性（其他样式扩展）',
    list_class       varchar(100)    default null               comment '表格回显样式',
    is_default       char(1)         default 'N'                comment '是否默认（Y是 N否）',
    status           char(1)         default '0'                comment '状态（0正常 1停用）',
    create_by        varchar(64)     default ''                 comment '创建者',
    create_time      datetime                                   comment '创建时间',
    update_by        varchar(64)     default ''                 comment '更新者',
    update_time      datetime                                   comment '更新时间',
    remark           varchar(500)    default null               comment '备注',
    primary key (dict_code)
) engine=innodb auto_increment=100 comment = '字典数据表';

insert into sys_dict_data values(1,  1,  '男',       '0',       'sys_user_sex',        '',   '',        'Y', '0', 'admin', sysdate(), '', null, '性别男');
insert into sys_dict_data values(2,  2,  '女',       '1',       'sys_user_sex',        '',   '',        'N', '0', 'admin', sysdate(), '', null, '性别女');
insert into sys_dict_data values(3,  3,  '未知',     '2',       'sys_user_sex',        '',   '',        'N', '0', 'admin', sysdate(), '', null, '性别未知');
insert into sys_dict_data values(4,  1,  '显示',     '0',       'sys_show_hide',       '',   'primary', 'Y', '0', 'admin', sysdate(), '', null, '显示菜单');
insert into sys_dict_data values(5,  2,  '隐藏',     '1',       'sys_show_hide',       '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '隐藏菜单');
insert into sys_dict_data values(6,  1,  '正常',     '0',       'sys_normal_disable',  '',   'primary', 'Y', '0', 'admin', sysdate(), '', null, '正常状态');
insert into sys_dict_data values(7,  2,  '停用',     '1',       'sys_normal_disable',  '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '停用状态');
insert into sys_dict_data values(8,  1,  '正常',     '0',       'sys_job_status',      '',   'primary', 'Y', '0', 'admin', sysdate(), '', null, '正常状态');
insert into sys_dict_data values(9,  2,  '暂停',     '1',       'sys_job_status',      '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '停用状态');
insert into sys_dict_data values(10, 1,  '默认',     'DEFAULT', 'sys_job_group',       '',   '',        'Y', '0', 'admin', sysdate(), '', null, '默认分组');
insert into sys_dict_data values(11, 2,  '系统',     'SYSTEM',  'sys_job_group',       '',   '',        'N', '0', 'admin', sysdate(), '', null, '系统分组');
insert into sys_dict_data values(12, 1,  '是',       'Y',       'sys_yes_no',          '',   'primary', 'Y', '0', 'admin', sysdate(), '', null, '系统默认是');
insert into sys_dict_data values(13, 2,  '否',       'N',       'sys_yes_no',          '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '系统默认否');
insert into sys_dict_data values(14, 1,  '通知',     '1',       'sys_notice_type',     '',   'warning', 'Y', '0', 'admin', sysdate(), '', null, '通知');
insert into sys_dict_data values(15, 2,  '公告',     '2',       'sys_notice_type',     '',   'success', 'N', '0', 'admin', sysdate(), '', null, '公告');
insert into sys_dict_data values(16, 1,  '正常',     '0',       'sys_notice_status',   '',   'primary', 'Y', '0', 'admin', sysdate(), '', null, '正常状态');
insert into sys_dict_data values(17, 2,  '关闭',     '1',       'sys_notice_status',   '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '关闭状态');
insert into sys_dict_data values(18, 99, '其他',     '0',       'sys_oper_type',       '',   'info',    'N', '0', 'admin', sysdate(), '', null, '其他操作');
insert into sys_dict_data values(19, 1,  '新增',     '1',       'sys_oper_type',       '',   'info',    'N', '0', 'admin', sysdate(), '', null, '新增操作');
insert into sys_dict_data values(20, 2,  '修改',     '2',       'sys_oper_type',       '',   'info',    'N', '0', 'admin', sysdate(), '', null, '修改操作');
insert into sys_dict_data values(21, 3,  '删除',     '3',       'sys_oper_type',       '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '删除操作');
insert into sys_dict_data values(22, 4,  '授权',     '4',       'sys_oper_type',       '',   'primary', 'N', '0', 'admin', sysdate(), '', null, '授权操作');
insert into sys_dict_data values(23, 5,  '导出',     '5',       'sys_oper_type',       '',   'warning', 'N', '0', 'admin', sysdate(), '', null, '导出操作');
insert into sys_dict_data values(24, 6,  '导入',     '6',       'sys_oper_type',       '',   'warning', 'N', '0', 'admin', sysdate(), '', null, '导入操作');
insert into sys_dict_data values(25, 7,  '强退',     '7',       'sys_oper_type',       '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '强退操作');
insert into sys_dict_data values(26, 8,  '生成代码', '8',       'sys_oper_type',       '',   'warning', 'N', '0', 'admin', sysdate(), '', null, '生成操作');
insert into sys_dict_data values(27, 9,  '清空数据', '9',       'sys_oper_type',       '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '清空操作');
insert into sys_dict_data values(28, 1,  '成功',     '0',       'sys_common_status',   '',   'primary', 'N', '0', 'admin', sysdate(), '', null, '正常状态');
insert into sys_dict_data values(29, 2,  '失败',     '1',       'sys_common_status',   '',   'danger',  'N', '0', 'admin', sysdate(), '', null, '停用状态');

drop table if exists sys_post;
create table sys_post
(
    post_id       bigint(20)      not null auto_increment    comment '岗位ID',
    post_code     varchar(64)     not null                   comment '岗位编码',
    post_name     varchar(50)     not null                   comment '岗位名称',
    post_sort     int(4)          not null                   comment '显示顺序',
    status        char(1)         not null                   comment '状态（0正常 1停用）',
    create_by     varchar(64)     default ''                 comment '创建者',
    create_time   datetime                                   comment '创建时间',
    update_by     varchar(64)     default ''			       comment '更新者',
    update_time   datetime                                   comment '更新时间',
    remark        varchar(500)    default null               comment '备注',
    primary key (post_id)
) engine=innodb comment = '岗位信息表';

drop table if exists sys_user_post;
create table sys_user_post
(
    user_id   bigint(20) not null comment '用户ID',
    post_id   bigint(20) not null comment '岗位ID',
    primary key (user_id, post_id)
) engine=innodb comment = '用户与岗位关联表';
```

添加 `sys_user.sql`：

```sql
drop table if exists sys_user;
create table sys_user
(
    user_id     bigint(20) not null auto_increment comment '用户ID',
    dept_id     bigint(20) default null comment '部门ID',
    user_name   varchar(30) not null comment '用户账号',
    nick_name   varchar(30) not null comment '用户昵称',
    user_type   varchar(2)   default '00' comment '用户类型（00系统用户）',
    email       varchar(50)  default '' comment '用户邮箱',
    phonenumber varchar(11)  default '' comment '手机号码',
    sex         char(1)      default '0' comment '用户性别（0男 1女 2未知）',
    avatar      varchar(100) default '' comment '头像地址',
    password    varchar(100) default '' comment '密码',
    status      char(1)      default '0' comment '帐号状态（0正常 1停用）',
    del_flag    char(1)      default '0' comment '删除标志（0代表存在 2代表删除）',
    login_ip    varchar(128) default '' comment '最后登录IP',
    login_date  datetime comment '最后登录时间',
    create_by   varchar(64)  default '' comment '创建者',
    create_time datetime comment '创建时间',
    update_by   varchar(64)  default '' comment '更新者',
    update_time datetime comment '更新时间',
    remark      varchar(500) default null comment '备注',
    primary key (user_id)
) engine=innodb auto_increment=100 comment = '用户信息表';

insert into sys_user
values (1, 103, 'admin', '若依', '00', 'ry@163.com', '15888888888', '1', '',
        '$2a$10$7JB720yubVSZvUI0rEqK/.VqGOZTH.ulu33dHOiBE8ByOhJIrdAu2', '0', '0', '127.0.0.1', sysdate(), 'admin',
        sysdate(), '', null, '管理员');
insert into sys_user
values (2, 105, 'ry', '若依', '00', 'ry@qq.com', '15666666666', '1', '',
        '$2a$10$7JB720yubVSZvUI0rEqK/.VqGOZTH.ulu33dHOiBE8ByOhJIrdAu2', '0', '0', '127.0.0.1', sysdate(), 'admin',
        sysdate(), '', null, '测试员');
```

4. 增加测试代码：

```java
package com.ruoyi.project.system.controller;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.test.context.jdbc.SqlGroup;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;

import static org.springframework.test.context.jdbc.Sql.ExecutionPhase.BEFORE_TEST_METHOD;

@AutoConfigureMockMvc
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
// 指定了要激活的配置文件，对应上面创建的 `application-it.yml`
@ActiveProfiles("it")
@SqlGroup({
        @Sql(value = "classpath:sql/sys_user_it.sql", executionPhase = BEFORE_TEST_METHOD),
        @Sql(value = "classpath:sql/sys_user.sql", executionPhase = BEFORE_TEST_METHOD)
})
class SysUserControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @Test
    void getInfo() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.get("/system/user/1")
                        // 设置请求头
                        .accept(MediaType.APPLICATION_JSON))
                // 验证响应状态码
                .andExpect(MockMvcResultMatchers.status().isOk())
                // 验证响应内容
                .andExpect(MockMvcResultMatchers.content().string(""));
    }
}
```

运行测试，测试将失败，后端返回：`{"msg":"请求访问：/system/user/1，认证失败，无法访问系统资源","code":401}`，权限校验还需要处理


5. 引入 SpdringSecurityTest 依赖

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <version>6.1.2</version>
    <scope>test</scope>
</dependency>
```

6. 在测试代码中 Mock 用户，因为若依使用的自定义的 `UserDetails`，所以必须手动设置 `UserDetails`：`.with(user(new LoginUser(new SysUser(1L), new HashSet<>(Arrays.asList("system:user:query")))))`，不能直接用注解 Mock，更多写法可以参考：[Running a Test as a User in Spring MVC Test](https://docs.spring.io/spring-security/reference/servlet/test/mockmvc/authentication.html)


```java
package com.ruoyi.project.system.controller;

import com.ruoyi.framework.security.LoginUser;
import com.ruoyi.project.system.domain.SysUser;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.test.context.jdbc.SqlGroup;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;
import org.springframework.web.context.WebApplicationContext;

import java.util.Arrays;
import java.util.HashSet;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.user;
import static org.springframework.test.context.jdbc.Sql.ExecutionPhase.BEFORE_TEST_METHOD;

@AutoConfigureMockMvc
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
// 指定了要激活的配置文件，对应上面创建的 `application-it.yml`
@ActiveProfiles("it")
@SqlGroup({
        @Sql(value = "classpath:sql/sys_user_it.sql", executionPhase = BEFORE_TEST_METHOD),
        @Sql(value = "classpath:sql/sys_user.sql", executionPhase = BEFORE_TEST_METHOD)
})
class SysUserControllerTest {
    @Autowired
    private WebApplicationContext context;

    @Autowired
    private MockMvc mockMvc;

    @Test
    void getInfo() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.get("/system/user/1")
                        .with(user(new LoginUser(new SysUser(1L), new HashSet<>(Arrays.asList("system:user:query")))))
                        // 设置请求头
                        .accept(MediaType.APPLICATION_JSON))

                // 验证响应状态码
                .andExpect(MockMvcResultMatchers.status().isOk())
                // 验证响应内容
                .andExpect(MockMvcResultMatchers.jsonPath("$.msg").value("操作成功"))
                .andExpect(MockMvcResultMatchers.jsonPath("$.code").value(200))
                .andExpect(MockMvcResultMatchers.jsonPath("$.data.userName").value("admin"))
                .andExpect(MockMvcResultMatchers.jsonPath("$.data.roles[0].roleName").value("超级管理员"));
                // 添加更多的断言
    }
}
```


---

<br>

上一节：[SpringBoot 测试实践 - 1：常用的工具](https://www.cnblogs.com/aaronlinv/p/17645009.html)
下一节：[SpringBoot 测试实践 - 3：@MockBean、@SpyBean 、提升测试运行速度、Testcontainer](https://www.cnblogs.com/aaronlinv/p/17652720.html)

<br>

---


## 参考资料

[MyBatis 测试](https://mybatis.org/spring-boot-starter/mybatis-spring-boot-test-autoconfigure/zh/index.html)
[Spring boot Mybatis-Plus数据库单测实战（三种方式）](https://blog.csdn.net/u012397189/article/details/109288747)
[如何高效地协作开发：一些 Google 的实践](https://1byte.io/google-large-scale-dev/)
[Java Web 项目单元测试/集成测试](https://www.bilibili.com/video/BV1YU4y1a73N/)
[@SpringBootTest not working (execution throwing NullPointerException)](https://stackoverflow.com/questions/68840412/springboottest-not-working-execution-throwing-nullpointerexception/68840522#68840522)
[直播回放 | 7月21日「JetBrains码上道」| 主题：Java中的测试与重构](https://www.bilibili.com/video/BV11W4y1y72D)
[代码很烂，但它能用，你敢重构吗？](https://www.bilibili.com/video/BV18g411g72M/)
[单元测试为什么在互联网滑铁卢](https://www.bilibili.com/video/BV1734y1v7UE/)
[单元测试有必要吗？](https://www.v2ex.com/t/821608)
[Testing the Web Layer](https://spring.io/guides/gs/testing-web/)
[Software Testing Tutorial - Learn Unit Testing and Integration Testing](https://www.youtube.com/watch?v=Geq60OVyBPg)
[Test-Driven Security](https://github.com/eleftherias/springone-2021)