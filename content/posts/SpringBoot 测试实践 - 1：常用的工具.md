---
date: '2023-08-21T09:00:33+08:00'
title: 'SpringBoot 测试实践 - 1：常用的工具'
categories: ["Spring"]
---

下一节：[SpringBoot 测试实践 - 2：单元测试与集成测试](https://www.cnblogs.com/aaronlinv/p/17645803.html)

<br>

---

我自己接触到的一些商业或是开源的基于 SpringBoot 项目，它们大部分是没有测试代码的，`test` 文件夹只有脚手架初始化生成的那个测试类，跟不同的开发聊到这个话题，发现他们中的大部分没有写测试的习惯，或者是觉得写测试代码麻烦，主要还是依赖测试工程师做黑盒的测试。只做黑盒测试的话有一定的的局限性，一些边界的条件可能就覆盖不到，而且相对来说人也比较容易出错、遗漏。而测试代码能解决其中很大一部分的问题，利用好单元测试和集成测试在某些情况下相对于直接通过 UI 进行测试是要更方便、节省时间的，所以想通过几篇博客来分享一下自己的测试实践

## 为什么要写测试（优点）

1. 覆盖更多的边界条件，且随时都可以运行测试代码（一劳永逸）
2. 缩小测试范围：测试某个方法只需要运行对应的测试代码，而不需要运行整个项目通过请求接口进行测试
3. 对重构更友好，可以随时重构有集成测试的代码，不用担心打破原有的代码
4. 其他人也可以通过测试快速地理清楚对应被测代码的主线逻辑（类似文档的作用，特别是复杂代码，通过测试能快速理解上手）
5. 写测试的过程，给自己一个新的视角去审视代码结构的设计，有助于改善代码设计


当然代码方式的测试也并非完美无缺：测试代码增加编写和维护的成本，同时一些外部依赖也需要通过 Mock 的方式实现，这些都提高了整个测试编写的门槛。也倒逼我们思考更好地组织代码，减少依赖

另一个方面：测试对于重构也是至关重要的，随着对业务的理解越来越深刻，可以重构代码，抽象出了一些共性的逻辑，优化代码结构，但是如果没有相关测试，面对着旧代码就只能望而却步了

## 测试工具：JUnit 5, AssertJ，Mockito

`spring-boot-starter-test` 自带常用的测试工具：`JUnit5`、`Assertj`、`Mockito`，可以直接使用

### JUnit5

Junit 5 包含：

- JUnit Platform：Test Engine
- Jupiter：编程模型和拓展模型
- Vintage：兼容老版本

JUnit 4 和 5 使用的包有所不同

```java
// JUnit 4
import org.junit.Test;
import static org.junit.Assert.assertEquals;

// JUnit 5
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;
```

如果不考虑兼容 JUnit 4 的测试，我们可以直接在依赖中直接排除 JUnit 4 的依赖，这样也可以避免在使用的时候错误地引入 JUnit 4 的包

```xml
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <version>{spring-boot-starter-test-version}</version>
        <scope>test</scope>
        <exclusions>
            <exclusion>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
            </exclusion>
            <exclusion>
                <groupId>org.junit.vintage</groupId>
                <artifactId>junit-vintage-engine</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

</dependencies>
```

还有一点值得注意的是：[JUnit 5 中 @RunWith 被 @ExtendWith 代替](https://www.baeldung.com/junit-5-runwith)

### AssertJ

AssertJ 是一种支持链式编程的断言库，相对于 JUnit 自带的断言，它提供了更多的方法，也提供了更好的断言不匹配时的信息展示

```java
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

public class MyTest {
    // 变量断言
    @Test
    public void test() {
        String name = "Alice";
        int age = 30;
        
        assertThat(name).isEqualTo("Alice");
        assertThat(age).isGreaterThan(18).isLessThan(60);
    }

    // List 断言
    @Test
    public void testList() {
        List<String> list = Arrays.asList("foo", "bar", "baz")
        assertThat(list).containsExactly("foo", "bar", "baz").hasSize(3);
    }

    // Map 断言
    @Test
    public void testMap() {
        Map<String, Integer> map = new HashMap<>();
        map.put("apple", 1);
        map.put("banana", 2);
        map.put("orange", 3);

        assertThat(map).containsEntry("banana", 2);

        assertThat(map).containsKey("banana");

        assertThat(map).containsValue(2);
    }

    // hasNoNullFieldsOrProperties 来断言测试对象的每个属性都不为 null
    @Test
    public void testHasNoNullFieldsOrProperties() {
        Person person = new Person("Alice", 30);
        assertThat(person).hasNoNullFieldsOrProperties();
    }

    // 异常断言
    @Test
    public void testDivideByZeroThrowsException() {
        assertThatThrownBy(() -> {
            int result = 1 / 0;
        }).isInstanceOf(ArithmeticException.class)
          .hasMessageContaining("/ by zero");
    }
}
```

### Mockito

Mockito 是一个 Java Mock 框架，用于创建各种类型的 Mock 对象，并设置 Mock 对象的行为

```java
import static org.mockito.Mockito.*;

import org.junit.jupiter.api.Test;

public class MyTest {
    // 创建 mock 对象
    @Test
    public void testCreateMock() {
        List<String> list = mock(List.class);
    }
    
    // 设置 mock 对象的行为
    @Test
    public void testMockBehavior() {
        List<String> list = mock(List.class);
        when(list.get(0)).thenReturn("foo");
        when(list.size()).thenReturn(1);
    }
    
    // 验证 mock 对象的方法调用
    @Test
    public void testMockVerification() {
        List<String> list = mock(List.class);
        list.add("foo");
        list.add("bar");
        verify(list).add("foo");
        verify(list).add("bar");
        // 验证调用方法的次数
        verify(list, times(2)).add(anyString());
    }

    // 模拟方法抛出异常
    @Test
    public void testMockException() {
        List<String> list = mock(List.class);
        doThrow(new RuntimeException()).when(list).clear();
        assertThrows(RuntimeException.class, () -> list.clear());
    }
}
```

也可以用注解来声明 Mock 对象，这样更清晰

```java
import static org.mockito.Mockito.*;

import java.util.List;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

// 启用 Mockito 扩展
@ExtendWith(MockitoExtension.class)
public class MyMockitoTest {
    
    // 创建一个 List 类型的 Mock 对象
    @Mock
    List<String> mockList;
    
    // 使用 @InjectMocks，将会自动注入被测试类中所声明的 Mock 对象到这个对象中 
    @InjectMocks
    MyService myService;
    
    @Test
    public void testMock() {
        // 模拟 Mock 对象的行为
        when(mockList.get(0)).thenReturn("foo");
        when(mockList.size()).thenReturn(1);
        
        String result = myService.doSomething();
        
        assertEquals("foo", result);

        verify(mockList).get(0);
    }
}

class MyService {
    
    private List<String> list;
    
    public MyService(List<String> list) {
        this.list = list;
    }
    
    public String doSomething() {
        return list.get(0);
    }
}
```

<br>

下一节：[SpringBoot 测试实践 - 2：单元测试与集成测试](https://www.cnblogs.com/aaronlinv/p/17645803.html)

<br>

---


## 参考资料
[《重构 改善既有代码的设计》](https://book.douban.com/subject/4262627/)
[有哪个开源项目，单元测试用例覆盖的比较全的？](https://www.v2ex.com/t/681939)
[业务代码写单元测试的最佳姿势是什么？](https://v2ex.com/t/777305)
[单元测试有落地效果好的团队吗？](https://www.v2ex.com/t/881655)
[Modern Best Practices for Testing in Java](https://phauer.com/2019/modern-best-practices-testing-java/)
[Thoughts on efficient enterprise testing (1/6) - Sebastian Daschner](https://blog.sebastian-daschner.com/entries/thoughts-on-efficient-testing)