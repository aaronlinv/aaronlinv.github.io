---
date: '2023-08-24T08:40:33+08:00'
title: 'SpringBoot 测试实践 - 3：@MockBean、@SpyBean 、提升测试运行速度、Testcontainer'
categories: ["Spring"]
---

上一节：[SpringBoot 测试实践 - 2：单元测试与集成测试](https://www.cnblogs.com/aaronlinv/p/17645803.html)

<br>

---

编写测试的时候，我们必须保证外部依赖行为一致，也需要模拟一些边界条件，所以我们需要使用 Mock 来模拟对象的行为。SpringBoot 提供了 `@MockBean` 和 `@SpyBean` 注解，可以方便地将模拟对象与 Spring 测试相结合，简化测试代码的编写

## @MockBean

`@MockBean` 是 Spring Boot Test提供的注解，用于在 Spring Boot 测试中创建一个模拟的 Bean 实例，并注入到测试类中的依赖项中。使用 Mock 可以控制被 Mock 对象的行为：自定义返回值、抛出指定异常等，模拟各种可能的情况，提高测试的覆盖率

```java
@SpringBootTest
@RunWith(SpringRunner.class)
public class MyServiceTest {
    @MockBean
    private ExternalDependency externalDependency;

    @Autowired
    private MyService myService;

    @Test
    public void testSomeMethod() {
        // 定义外部依赖的行为
        Mockito.when(externalDependency.someMethod()).thenReturn("Mocked Result");

        // 调用被测试类的方法
        // 被测方法内部调用了 ExternalDependency 的 someMethod 方法
        String result = myService.someMethod();

        // 验证外部依赖的方法是否被调用
        Mockito.verify(externalDependency).someMethod();

        // 断言结果
        assertEquals("Mocked Result", result);
    }
}
```

需要注意的是：使用了 `@MockBean`，会创建完全模拟的对象，它**完全替代**了被模拟的 Bean，并且所有方法的调用都被模拟。对于未指定行为的方法，返回值如果是基本类型则返回对应基本类型的默认值，如果是引用类型则返回 `null`

## @SpyBean

`@SpyBean` 是 Spring Boot Test 提供的另一个注解，与 `@MockBean` 作用相似，但是它创建的是部分模拟对象，未指定方法行为时，将执行被模拟对象的**真实实现**，返回实际方法的执行结果

常见的情况是：测试依赖外部资源（例如数据库、文件系统等）的方法，我们要在测试中模拟其中一部分方法的行为，同时保留对外部资源的实际访问，那么可以使用 `@SpyBean`

```java
@SpringBootTest
public class MyServiceTest {
    
    @Autowired
    private MyService myService;
    
    @SpyBean
    private MyRepository myRepository;

    @Test
    public void testMyService() {
        // 使用 doReturn 方法模拟调用 myRepository 的方法，并返回指定的值
        Mockito.doReturn(new MyEntity()).when(myRepository).findById(1L);
        
        MyResult result = myService.doSomething(1L);
        Assertions.assertEquals("success", result.getMessage());
    }
}
```

这里有一个很重要的点是：`@SpyBean` 使用 `doReturn` 而不是 `thenReturn`，因为 Spy 对象是基于实例创建的，而 thenReturn 方法会调用实例方法并返回模拟结果，这可能会导致实例状态发生变化，从而影响后续的测试步骤。换而言之如果 Spy 对象使用 `doReturn` 就像这样：`Mockito.when(myRepository.findById(1L)).thenReturn(new MyEntity());`这段代码我们本意是指定这个 Spy 对象的 `findById(1L)` 的行为，但是实际上 when 语句中 `myRepository.findById(1L)` 已经执行了了实际的逻辑，这可能影响整个测试

## 简化 Spring Context 提升测试运行速度

[SpringBoot 测试实践 - 2：单元测试与集成测试](https://www.cnblogs.com/aaronlinv/p/17645803.html) 中提到了单元测试用到的 `@MybatisTest` 以及集成测试用到的 `@SpringBootTest`。`@SpringBootTest` 加载整个 Spring Boot 应用程序的上下文，就像启动了整个 SpringBoot 应用，而 `@MybatisTest` 只配置了用于测试 MyBatis 的组件，速度就非常快。完整的项目有大量的测试用例，如果每个测试都重新加载 Spring Context 这样就非常耗时，所以要尽量减少 `@SpringBootTest`

一般业务代码都会注入外部依赖，如果只在测试方法上使用 `@Test` 注解，这样运行测试就会抛出空指针异常，需要在类上使用 `@ExtendWith(SpringExtension.class)` 将 Spring 的测试支持集成到 JUnit 5 中，这样就可以在测试类中获得 Spring 容器的支持，以便进行依赖注入、加载配置文件、使用 Spring Bean 等。仅加上这个注解是不够的，Spring 容器内依然没有我们需要的依赖，我们还需要使用 `@ContextConfiguration()` 指定要加载的配置文件、配置类或其他资源

如果说 `@SpringBootTest` 是初始化好所有项目中用到的 Bean 的话，那 `@ExtendWith(SpringExtension.class)` 就是按需取用，所以必须保证被测的**类**用到的**所有依赖对象**都装配进 Spring 的 IoC 容器里，否则就会抛出这样的异常：

```log
java.lang.IllegalStateException: Failed to load ApplicationContext

Caused by: org.springframework.beans.factory.UnsatisfiedDependencyException: 
Error creating bean with name 'com.test.ConfigurationRepository#0': 
Unsatisfied dependency expressed through field 'configurationRepository'; 
nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: 
No qualifying bean of type 'com.test.ConfigurationRepository' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: 
{@org.springframework.beans.factory.annotation.Autowired(required=true)}
```

我们可以通过 `@ContextConfiguration()` 来指定配置类或者是类 class。对于被测方法没有使用某些依赖也可以直接用 `@MockBean` 配置一个Mock 对象，保证测试能正常运行

```java
@ExtendWith(SpringExtension.class)
// 指定加载的两个类：RestTemplate.class 和 ExecutorConfig.class
// ExecutorConfig.class 一个自定义的配置类，包含线程池配置
@ContextConfiguration(classes = {RestTemplate.class, ExecutorConfig.class})
class UserServiceTest {
    // 注入被测对象
    @Autowired
    private UserService userService;

    // 使用 Mock 代替容器加载依赖
    @MockBean
    private ConfigurationRepository configurationRepository;

    // 通过 @ContextConfiguration，确保 Spring Context 中会包含 RestTemplate 的相关配置
    @Autowired
    private RestTemplate restTemplate;
}
```

## 避免 ApplicationContext 复用

默认情况下，运行测试 `ApplicationContext` 会被复用，以加快测试的运行速度。但是在某些情况下，比如：多个测试类继承同一个抽象类，这可能会导致测试运行失败。可以在抽象类或每个子类中使用 @DirtiesContext，让 Spring 在测试这些类后重置 `ApplicationContext`

`@DirtiesContext` 默认的 `classMode` 参数为`ClassMode.AFTER_CLASS` 该模式会在 整个测试类运行完毕后重新加载 Spring 测试上下文。如果希望每次测试方法运行后都重新加载 `ApplicationContext` 可以使用 `@DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)`


@DirtiesContext 也可以用于方法级别，在方法运行前或运行后标记为需要重新加载 `ApplicationContext`

```java
@SpringBootTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class CacheIntegrationTest {

    @Autowired
    private CacheService cacheService;

    @Test
    @Order(1)
    @DirtiesContext(methodMode = DirtiesContext.MethodMode.AFTER_METHOD)
    public void testCacheEviction() {
        // 模拟缓存数据，缓存实际为 HashMap
        cacheService.addToCache("key1", "value1");
        cacheService.addToCache("key2", "value2");
    }

    @Test
    @Order(2)
    public void testCacheLookup() {
        // 从缓存中查找数据
        // 因为使用了 @DirtiesContext(methodMode = DirtiesContext.MethodMode.AFTER_METHOD) ApplicationContext 重置，故缓存为空
        String value1 = cacheService.getFromCache("key1");
        String value2 = cacheService.getFromCache("key2");
    }
}
```

## Testcontainer

为了不影响测试环境的数据，涉及数据层修改的测试，我们可以使用 H2 数据库或者是像上一节：[SpringBoot 测试实践 - 2：单元测试与集成测试](https://www.cnblogs.com/aaronlinv/p/17645803.html) 这样的，直接使用专门的测试数据库。还有一种方式是用 Docker 启动一个全新的数据库供测试环境使用。而 [Testcontainer](https://testcontainers.com/getting-started/) 的目标就是简化了整个流程：通过代码的方式指定镜像，测试一启动 `Testcontainer` 将完成初始化工作，自动拉取镜像并创建容器，测试结束后将关闭对应的容器

一些 CI 广泛地使用 `TestContainer` 保证测试环境的一致性。但是如果本地运行，`Testcontainer`依赖本地的 Docker Daemon 或是 [Testcontainers Cloud](https://testcontainers.com/cloud/) 这样的方案。Windows 本地部署 Docker 也会更麻烦一些

`TestContainer` 需要与容器运行时进行交互，第一次要拉取镜像，所以速度上相对慢一些，但是得益于 Docker，`TestContainer` 几乎可以启动任何服务，无论是数据库、缓存或者是 MQ 等等的，可以保证外部环境的一致性

教程可以参考官方的实践：

[Testcontainer Java 官方实践](https://github.com/testcontainers/testcontainers-java/tree/main/examples)

[Testcontainer SpringBootTest 案例](https://github.com/testcontainers/testcontainers-java/tree/main/examples/spring-boot/src/test/java/com/example)

---

<br>

上一节：[SpringBoot 测试实践 - 2：单元测试与集成测试](https://www.cnblogs.com/aaronlinv/p/17645803.html)

<br>

---

## 参考资料
[Context Management and Caching](https://docs.spring.io/spring-framework/reference/testing/integration.html#testing-ctx-management)
[Pitfalls on Testing with Spring Boot](https://www.baeldung.com/spring-boot-testing-pitfalls)
[A Quick Guide to @DirtiesContext](https://www.baeldung.com/spring-dirtiescontext)
[Modern Best Practices for Testing in Java](https://phauer.com/2019/modern-best-practices-testing-java/#mock-remote-service)
[Testing :: Spring Framework](https://docs.spring.io/spring-framework/reference/testing.html)
[Protect REST APIs with Spring Security Reactive and JWT](https://medium.com/zero-equals-false/protect-rest-apis-with-spring-security-reactive-and-jwt-7b209a0510f1)
[Spring - Testing](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)