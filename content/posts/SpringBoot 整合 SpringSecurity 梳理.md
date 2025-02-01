---
date: '2021-08-24T00:31:33+08:00'
title: 'SpringBoot 整合 SpringSecurity 梳理'
categories: ["Spring"]
---

## 文档
[Spring Security Reference](https://docs.spring.io/spring-security/site/docs/current/reference/html5/)
[SpringBoot+SpringSecurity+jwt整合及初体验](https://www.cnblogs.com/pjjlt/p/10960690.html)
[JSON Web Token 入门教程 - 阮一峰](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)
[JWT 官网](https://jwt.io/)

## SpringSecurity
项目 GitHub 仓库地址：[https://github.com/aaronlinv/springsecurity-jwt-demo](https://github.com/aaronlinv/springsecurity-jwt-demo)
### 依赖
主要用到了: SpringSecurity,Thymeleaf,Web,Lombok
```xml
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
</dependency>
```
### 页面
编写页面和 Controller 进行测试，具体页面可以看 [代码](https://github.com/aaronlinv/springsecurity-jwt-demo/commit/a3653a02ab9e074d0e25387cf387b2b338353d8d)
主要包含了首页(index)，订单(order)，还有 user,role,menu这三个位于 `/system` 下，需要 admin 权限
### 使用内存用户进行表单登录

在 `static` 下新建 `login.html`，用于登录
```html
<form action="/login" method="post">
    <label for="username">账户</label><input type="text" name="username" id="username"><br>
    <label for="password">密码</label><input type="password" name="password" id="password"><br>
    <input type="submit" value="登录">
</form>
```
编写继承 WebSecurityConfigurerAdapter 的 Security 配置类，并开启 @EnableWebSecurity 注解，这个注解包含了 @Configuration
WebSecurityConfigurerAdapter 中有两个方法，它们名称相同，但是入参不同
```java
protected void configure(HttpSecurity http) throws Exception
protected void configure(AuthenticationManagerBuilder auth) throws Exception
```
入参为 HttpSecurity 的 configure 可以配置拦截相关的参数
另一个入参为 AuthenticationManagerBuilder，则是用来配置验证相关的参数

```java
@EnableWebSecurity
// @Configuration 被包括在上面的注解了
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Bean
    // 配置 PasswordEncoder 用于密码的加密和匹配
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 
        http
            // 配置表单登录相关参数
            .formLogin()
                // 登录页面
                .loginPage("/login.html")
                // 表单提交的地址
                .loginProcessingUrl("/login")
                // 登录成功后跳转的地址
                .defaultSuccessUrl("/index")

            // .and() 方法返回的是 HttpSecurity 对象
            .and()
                // 配置权限相关参数
                .authorizeRequests()
                // 匹配路径
                // 需要开放登录的地址，否则访问登录页面时因为没有权限，自动跳转到登录页，进入死循环，导致报错：重定向的次数过多
                .antMatchers("/login.html", "/login")
                // 允许访问
                .permitAll()

                // 匹配路径
                .antMatchers("/order")
                // 必须有指定的任意权限才能访问
                .hasAnyAuthority("ROLE_user", "ROLE_admin")

                // 匹配 /system 下的所有路径
                .antMatchers("/system/**")
                // 拥有指定角色才能访问
                .hasRole("admin")

                // 除了上面的路径，其他都需要认证
                .anyRequest().authenticated()

            // 返回 HttpSecurity 对象
            .and()
                // 关闭 csrf （跨站请求伪造）
                .csrf().disable();

        // 设置 注销地址
        http.logout().logoutUrl("/logout");
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 配置验证
        // 使用内存（非持久化）验证
        auth.inMemoryAuthentication()
                // 配置用户名
                .withUser("user")
                // 配置用 PasswordEncoder 加密后的密码
                .password(passwordEncoder().encode("1234"))
                // 配置角色
                .roles("user")
                .and()

                .withUser("admin")
                .password(passwordEncoder().encode("1234"))
                .roles("admin")

                .and()
                // 配置授权时默认使用的 PasswordEncoder
                .passwordEncoder(passwordEncoder());
        ;
    }
}
```
具体代码参考 [这里](https://github.com/aaronlinv/springsecurity-jwt-demo/commit/fe65f3ec57383be6fc7d8f7425c41bb666b27281)
两个 configure 非常类似，入参对象的方法中包含了具体的配置项，如：`formLogin`,`authorizeRequests`,`csrf`,`logout` 等等，部分配置项还可以通过链式调用，进行该配置项更详细地配置，通过 `.and()` 可以回到 HttpSecurity 对象，再定义其他配置项

使用表单的方式登录需要配置：表单 (formLogin)、授权(authorizeRequests) 、跨站请求伪造(csrf)、注销(logout)，还需要配置验证，先使用最简单的 inMemoryAuthentication，并指定账户密码，再指定密码编码器

然后启动服务，访问登录页面（注意这里的被修改为 8081），输入不同的账号密码，测试不同页面的访问情况，没有权限会提示：403
[http://localhost:8081/login.html](http://localhost:8081/login.html)
### 使用 Json 传递参数，自定义 Handler
修改登录页面，使用 Ajax 向后端传递 账户和密码，需要使用 POST
```html
<head>
    <meta charset="UTF-8">
    <title>登录</title>
    <script src="https://cdn.bootcdn.net/ajax/libs/jquery/3.6.0/jquery.js"></script>
</head>
<body>

<form action="/login" method="post">
    <label for="username">账户</label><input type="text" name="username" id="username"><br>
    <label for="password">密码</label><input type="password" name="password" id="password"><br>
    <input type="submit" onclick="login()" value="登录">
</form>
</body>
<script>
    function login() {
        $.ajax({
            type: "POST",
            url: "/login",
            data: {
                "username": $("#username").val(),
                "password": $("#password").val(),
            },
            success: function (data) {
                if (data.code == 20001) {
                    Location.href = "/index";
                } else {
                    alert(data.msg);
                }
            }
        })
    }
</script>
```
需要编写登录成功和登录失败时调用的 Handler，并配置到SecurityConfig 中
```java
@Component
public class MyAuthenticationFailureHandler implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        response.setContentType("application/json; charset=UTF-8");
        PrintWriter writer = response.getWriter();
        writer.write("{\"code\":\"40001\",\"msg\":\"登录失败\"}");
        writer.flush();
        writer.close();
    }
}
```

```java
@Component
public class MyAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        response.setContentType("application/json; charset=UTF-8");
        PrintWriter writer = response.getWriter();
        writer.write("{\"code\":\"20001\",\"msg\":\"登录成功\"}");
        writer.flush();
        writer.close();
    }
}
```
在 SecurityConfig 中 注入并配置 Handler
```java
    @Autowired
    private AuthenticationSuccessHandler successHandler;

    @Autowired
    private AuthenticationFailureHandler failureHandler;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()
                .loginPage("/login.html")
                .loginProcessingUrl("/login")
                // 指定 Handler
                .successHandler(successHandler)
                .failureHandler(failureHandler)
                // 省略其他代码...
    }
```
具体代码参考 [这里](https://github.com/aaronlinv/springsecurity-jwt-demo/commit/d2badb8ea02fa092c3452a9f06c4105f1df1c15d)
登录页面进行测试：[http://localhost:8081/login.html](http://localhost:8081/login.html)
首页:[http://localhost:8081/](http://localhost:8081/)

### 基于数据库的认证
创建数据库 `jwt_demo` ，导入表数据：[sql 脚本](https://github.com/aaronlinv/springsecurity-jwt-demo/blob/master/jwt_demo.sql)
users 表，包括字段：user_id,user_name,password,status,roles
导入 MySQL 驱动和 JPA 的依赖
```xml
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
```
在 application.properties 中配置数据库信息
```properties
server.port=8081
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.username=root
spring.datasource.password=1234
spring.datasource.url=jdbc:mysql://localhost:3306/jwt_demo?serverTimezone=GMT%2B8&characterEncoding=utf-8
```
UserDetails 接口是 SpringSecurity 用来承载用户信息的载体，SpringSecurity 提供了对这个接口的实现类：org.springframework.security.core.userdetails.User，我们自己定义的用户类通常也叫User，所以导包时候要注意使用 我们自己定义的 User 类

```java
@Entity
@Table(name = "users")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column
    private Long userId;

    @Column(name = "user_name")
    private String userName;

    @Column(name = "password")
    private String password;

    @Column(name = "status")
    private String status;

    @Column(name = "roles")
    private String roles;

    // 对象的权限列表，不需要持久化
    @Transient
    private List<GrantedAuthority> authorities;

    public void setAuthorities(List<GrantedAuthority> authorities) {
        this.authorities = authorities;
    }

    // 必须重写接口的对于 getPassword,getUsername,getAuthorities 等方法
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.authorities;
    }

    @Override
    public String getUsername() {
        return this.userName;
    }

    // 下面 4 个需要方法 return true，否则登录时会被限制
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```
定义 JPA 的 Repository
```java
@Repository
public interface UserDao extends JpaRepository<User, Long> {
}
```
定义 Service 
```java
public interface UserService {
    public User selectUserByUserName(String username);
}
```
定义 Service 对应的实现，通过查询用户名获得用户相关信息
```java
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private UserDao userDao;

    @Override
    public User selectUserByUserName(String username) {
        User user = new User();
        user.setUserName(username);
        List<User> list = userDao.findAll(Example.of(user));
        return list.isEmpty() ? null : list.get(0);
    }
}
```
还需要编写 UserDetailService，供 SpringSecurity 的 DaoAuthenticationProvider 类中的 retrieveUser 方法调用，以此获得对应用户的信息
```java
@Service
public class UserDetailService implements UserDetailsService {
    @Autowired
    private UserService userService;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 调用 Service
        User user = userService.selectUserByUserName(username);
        if (user == null) {
            throw new UsernameNotFoundException("用户" + user.getUsername() + "不存在");
        }
        // 设置权限
        // commaSeparatedStringToAuthorityList 方式将字符串间通过 ',' 进行分割，然后返回 List
        user.setAuthorities(AuthorityUtils.commaSeparatedStringToAuthorityList(user.getRoles()));
        return user;
    }
}
```
```java
// 省略其他...
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    private UserDetailService userDetailService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 将内存授权方式替换为自己实现的 UserDetailService
        auth.userDetailsService(userDetailService)
                .passwordEncoder(passwordEncoder());
    // 省略其他...
}
```

具体代码参考 [这里](https://github.com/aaronlinv/springsecurity-jwt-demo/commit/97f9d10c9f7ae5f193d75dd8ca8d16ee8dedc9c4)

登录页面进行测试：[http://localhost:8081/login.html](http://localhost:8081/login.html)
首页:[http://localhost:8081/](http://localhost:8081)

### 整合 JWT
添加 jjwt 依赖
```xml
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.9.0</version>
        </dependency>
```

在 application.properties 中配置 JWT 参数
```properties
token.header:Authorization
#  令牌秘钥
token.secret:askdhfkjahskjdfhkalsjhdf^112asdfasdf44^%$_@+asdfasdfaskjdhfkjashdfljkahsdklsfjasgdkjfgjahs(IS:)_@@+asdfasdfaskjdhfkjashdfljkahsdklsfja@+asdfasdfaskjdhfkjashdfljkahsdklsfjasgdkjfgjahssgdkjfgjahsdgfjhgsdfsadf+-asdfasdas+as++_sdfsdsasdfasdf
#   令牌有效期（默认30分钟）
token.expireTime:3600000
```

定义统一 API 封装格式
```java
public class RestResult extends HashMap<String, Object> {
    private static final long serialVersionUID = 1L;
    // 状态码
    public static final String CODE_TAG = "code";
    // 返回内容
    public static final String MSG_TAG = "msg";
    // 数据对象
    public static final String DATA_TAG = "data";

    public RestResult() {

    }

    public RestResult(int code, String msg) {
        super.put(CODE_TAG, code);
        super.put(MSG_TAG, msg);
    }

    public RestResult(int code, String msg, Object data) {
        super.put(CODE_TAG, code);
        super.put(MSG_TAG, msg);
        if (data != null) {
            super.put(DATA_TAG, data);
        }
    }

    public static RestResult success() {
        return new RestResult(200, "成功");
    }
}
```
然后准备 JWT 工具类，实现：生成 token、从 token 中获取用户名、检查 token 是否过期、刷新 token、验证 token 等，这里的 KEY 通过双重锁 保证了线程安全
```java
@Data
@Component
@Slf4j
public class JwtTokenUtils {
    @Value("${token.secret}")
    private String secret;

    @Value("${token.expireTime}")
    private Long expiration;

    @Value("${token.header}")
    private String header;

    private static Key KEY = null;

    /**
     * 生成token令牌
     *
     * @param userDetails 用户
     * @return 令token牌
     */
    public String generateToken(UserDetails userDetails) {
        log.info("[JwtTokenUtils] generateToken " + userDetails.toString());
        Map<String, Object> claims = new HashMap<>(2);
        claims.put("sub", userDetails.getUsername());
        claims.put("created", new Date());

        return generateToken(claims);
    }

    /**
     * 从令牌中获取用户名
     *
     * @param token 令牌
     * @return 用户名
     */
    public String getUsernameFromToken(String token) {
        String username = null;
        try {
            Claims claims = getClaimsFromToken(token);
            username = claims.get("sub", String.class);
            log.info("从令牌中获取用户名:" + username);
        } catch (Exception e) {
            username = null;
        }
        return username;
    }

    /**
     * 判断令牌是否过期
     *
     * @param token 令牌
     * @return 是否过期
     */
    public Boolean isTokenExpired(String token) {
        try {
            Claims claims = getClaimsFromToken(token);
            Date expiration = claims.getExpiration();
            return expiration.before(new Date());
        } catch (Exception e) {
            return false;
        }
    }

    /**
     * 刷新令牌
     *
     * @param token 原令牌
     * @return 新令牌
     */
    public String refreshToken(String token) {
        String refreshedToken;
        try {
            Claims claims = getClaimsFromToken(token);
            claims.put("created", new Date());


            refreshedToken = generateToken(claims);
        } catch (Exception e) {
            refreshedToken = null;
        }
        return refreshedToken;
    }

    /**
     * 验证令牌
     *
     * @param token 令牌
     * @param userDetails 用户
     * @return 是否有效
     */
    public Boolean validateToken(String token, UserDetails userDetails) {

        String username = getUsernameFromToken(token);
        return (username.equals(userDetails.getUsername()) &&
                !isTokenExpired(token));
    }

    /**
     * 从claims生成令牌
     *
     * @param claims 数据声明
     * @return 令牌
     */
    private String generateToken(Map<String, Object> claims) {
        Date expirationDate = new Date(System.currentTimeMillis() + expiration);
        return Jwts.builder().setClaims(claims)
                .setExpiration(expirationDate)
                .signWith(SignatureAlgorithm.HS256, getKeyInstance())
                .compact();
    }

    /**
     * 从令牌中获取数据声明
     *
     * @param token 令牌
     * @return 数据声明
     */
    private Claims getClaimsFromToken(String token) {
        Claims claims = null;

        try {
            claims = Jwts.parser().setSigningKey(getKeyInstance()).parseClaimsJws(token).getBody();
        } catch (Exception e) {
            claims = null;
        }
        return claims;
    }

    private Key getKeyInstance() {
        if (KEY == null) {
            synchronized (JwtTokenUtils.class) {
                if (KEY == null) {// 双重锁
                    byte[] apiKeySecretBytes = DatatypeConverter.parseBase64Binary(secret);
                    KEY = new SecretKeySpec(apiKeySecretBytes, SignatureAlgorithm.HS256.getJcaName());
                }
            }
        }
        return KEY;
    }
}
```
然后定义 JwtAuthTokenFilter，用于过滤请求
```java
@Component
public class JwtAuthTokenFilter extends OncePerRequestFilter {
    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private JwtTokenUtils jwtTokenUtils;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
        // 从请求头中获取 Authorization 的值，即 token
        String jwtToken = request.getHeader(jwtTokenUtils.getHeader());

        if (!ObjectUtils.isEmpty(jwtToken)) {
            // 从 token 中获取用户名，用户名存储在负载中，负载一般没有加密，所以负载的内容是可以见，不能在其中存放敏感信息
            // 可以通过 https://jwt.io/ 进行解码
            String username = jwtTokenUtils.getUsernameFromToken(jwtToken);

            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                // 通过 userDetailsService 从数据库中获取对应用户的信息
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                // 这里校验 token 有效性
                if (jwtTokenUtils.validateToken(jwtToken, userDetails)) {
                    // 将 UserDetails 对象 封装为 UsernamePasswordAuthenticationToken 对象
                    // 第一参数是 Object principal，传入的是 UserDetails 对象，在后面的 Service 中会取出 principal
                    UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                    // 交给SpringSecurity管理，在之后的过滤器不会被拦截进行二次授权了
                    SecurityContextHolder.getContext().setAuthentication(authenticationToken);
                }
            }
        }
        // 将请求转发给过滤器链上的下一个对象
        chain.doFilter(request, response);
    }
}
```
编写 JwtAuthService，处理登录的相关逻辑，使用 AuthenticationManager 对传入的账号密码进行认证，成功返回 生成的 token
```java
@Service
public class JwtAuthService {
    @Autowired
    private JwtTokenUtils jwtTokenUtils;

    @Autowired
    private AuthenticationManager authenticationManager;

    public String login(String username, String password) {
        Authentication authentication = null;
        try {
            authentication = authenticationManager.authenticate(new UsernamePasswordAuthenticationToken(username, password));
        } catch (Exception e) {
            throw new RuntimeException("用户名或密码有误");
        } 
        // 这里就是获取的就是在前面 JwtAuthTokenFilter 中传入的 principal
        User loginUser = (User) authentication.getPrincipal();
        return jwtTokenUtils.generateToken(loginUser);
    }
}
```
用于登录的 Controller 
```java
@RestController
public class JwtLoginController {
    @Autowired
    private JwtAuthService jwtAuthService;

    @PostMapping({"/login", "/"})
    public RestResult login(String username, String password) {
        RestResult result = RestResult.success();
        String token = jwtAuthService.login(username, password);
        result.put("token", token);
        return result;
    }
}
```
在 SecurityConfig 中 注入并配置 Handler
```java
    // 省略其他代码...
    @Autowired
    private JwtAuthTokenFilter jwtAuthTokenFilter;

    // 重写 AuthenticationManager，避免报错
    @Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // http.formLogin()
        //         .loginPage("/login.html")
        //         .loginProcessingUrl("/login")
        //         // .defaultSuccessUrl("/index")
        //         // .defaultSuccessUrl("/index")
        //         .successHandler(successHandler)
        //         .failureHandler(failureHandler)
        http.sessionManagement()
                // 不创建和使用 session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()

                .authorizeRequests()
                .antMatchers("/login")
                .anonymous()

                .antMatchers(HttpMethod.GET, "/*.html", "/**/*.html", "/**/*.css", "/**/*.js")
                .permitAll()
        // 省略其他代码...

        // 使用 JWT 过滤器
        http.addFilterBefore(jwtAuthTokenFilter, UsernamePasswordAuthenticationFilter.class);
    }
     // 省略其他代码...
```
可以通过 Postman 先指定参数（注意是用 POST），获取 token：
[http://localhost:8081/login?username=user&password=1234](http://localhost:8081/login?username=user&password=1234)

在 Headers 中添加 `Authorization`，值为获取到的 token
使用 GET 访问：[http://localhost:8081/order](http://localhost:8081/order)
因为 user 没有管理权限，所以访问管理页面会 403：[http://localhost:8081/system/role](http://localhost:8081/system/role)


具体代码参考 [这里](https://github.com/aaronlinv/springsecurity-jwt-demo/commit/a6c52f031e54f1cdc99244fcf9815b9dabdd0163)