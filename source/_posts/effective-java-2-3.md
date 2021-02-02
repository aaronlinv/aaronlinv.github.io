---
title: 《Effective Java》笔记 2~3
categories: 《Effective Java》
date: 2021-02-02 10:44:56
---
# 2. 创建和销毁对象
## 1. 静态工厂方法替代构造器
优点：
1. 名称清晰
2. 每次调用不必new对象
3. 可以返回原返回类型任意子类型对象
4. 返回的对象可以随着调用而发生改变
5. 返回的对象所属的类，在编写该静态工厂方法的类时可以不存在

缺点：
1. private 构造器导致，就不能有子类，子类构造器会默认访问父类构造器
## 2. 多个构造器参数时可以使用构建器（建造者模式 Builder）
印象比较深刻的是：之前写安卓用到了OkHttp，使用的OkHttp是用Kotlin写的，其中实例化对象用的就是建造者模式，当时以为是Kotlin链式调用的某种语法特性，后来才知道是设计模式

主要用于多参数时，避免重叠构造器和避免无参构造器创建对象依次set参数过程中JavaBean可能处于的不一致状态
## 3. 私有化构造器或者枚举类型强化Singleton属性
Singleton常见实现方法：
1. final修饰的公有静态成员
2. 静态工厂
3. 单元素Enum

通过反射调用私有构造器，可以修改构造器，创建第二个实例时抛出异常
序列化时除了实现 Serializable接口，还需要提供readResolve，防止反序列化创建新的实例
## 4. 私有构造器强化不可实例化
## 5. 优先考虑依赖注入来引用资源
SOLID 原则中的 D 依赖反转原则 (Dependency Inversion Principle)，依赖注入是该原则的一种实现方式
创建一个新的实例时，就将该资源传到构造器中
## 6. 避免创建不必要的对象
1. 不使用 new String() 方式创建String实例，使用 String s ="hello"; 方式，同一台虚拟机的代码，字符串字面常量相同，该对象就会被重用
2. 复用创建成本较高的实例：正则Pattern实例
3. 优先使用基本类型，自动装箱一定程度降低性能
注意！在提倡保护性拷贝时，重用对象付出代价远大于创建重复对象
## 7. 清除过期的对象引用
1. 栈pop时应将pop的对象设置为null
2. 避免缓存内存泄漏的一种方式：WeakHashMap，除了WeakHashMap的键之外，如果没有存在对某个键的引用，会被自动删除
3. 监听器和其他回调造成的内存泄漏：只保留他们的弱引用，例如保存成WeakHashMap的键
## 8.避免使用终结方法和清除方法
终结方法 (finalizer) 和 清除方法(cleaner JDK9) 都不可预测且会造成性能损失
注重时间的任务不应该使用这两种方法来完成
不应该依赖这两种方法来更新重要的持久状态（比如：释放共享资源，可能还没开始释放资源，系统就因为资源不足垮掉了）
TODO p25 终结方法攻击(finalizer attack)
TODO 合理用途：
1. 安全网，忘记close
2. 回收对象的本地对等体(native peer)
## 9. try-with-resources 优先于try-finally
实现了AutoCloseable 接口
1. 优雅
2. 避免底层物理设备异常导致第一个异常被第二个异常抹除，增加排错成本

# 3. 对于所有对象都通用的方法
## 10. 覆盖Equals时请注意遵守通用约定
不用覆盖的情况（满足其一即可）
1. 类的每个实例本质都唯一
2. 类无需提供逻辑相等功能
3. 父类的equals方法足够满足使用
4. 类是私有或者默认权限或确定不会调用到equals

覆盖equals的通用规范
1. 自反性 非null值，x.equals(x) 为true 子类和父类不同的equals方法，相互equals会违反自反性
2. 对称性 非null值，x.equals(y) 等于 y.equals(x) 
3. 传递性 非null值，x.equals(y) 为ture 且 y.equals(z) 为true -> x.equals(z)为true 子类相较于父类增加了一些用于equals的属性，但是子类还使用父类equals，违反传递性
4. 一致性 非null值，值未修改，多次调用equals 结果应一致
5. x非null值，x.equals(null) 返回 false

子类与父类 自反性和传递性的对立：无法在拓展可实例化的类的同时，既增加新的值组件，同时又保留equals约定

IDEA 默认子类equals写法就是：使用getClass() 比较对象，然后调用父类equals最后对比子类拓展的属性

Stream 初始化Set：
```java
    private static final Set<Point> unitCircle = Stream.of(
            new Point(1, 0),
            new Point(0, 1),
            new Point(-1, 0),
            new Point(0, -1)
    ).collect(Collectors.toCollection(HashSet::new));
```

辨析：instanceof    getClass()==
- instanceof 这个对象是否为这个类**或其子类**的实例
- getClass()== 运行时期对象的类

使用复合优于继承：提供私有Point域以及共有视图（view）方法

JDK反例：public class Timestamp extends java.util.Date，在同一个集合中使用或者其他方式混合使用，可能有不正确的行为

instanceof 第一个操作符为null 那么返回的一定为false，使用 instanceof 可以省略 null 判断

一致性，不要使equals方法依赖于不可靠的资源，JDK反例：URL equals

高质量equals诀窍
1. == 检查是否为这个对象的引用
2. instanceof 检查是否为正确类型（同时也可以排除掉null）
3. 转换为正确的类型
4. 检查每个关键域
   - 基本类型： ==
   - 浮点数：Float.compare(float,float) Double.compare(double,double) ，使用Float.equals或Double.equals 自动装箱减低性能
   - 数组域：Arrays.equals
   - 合法null：Objects.equals(Object,Object) 避免抛出空指针异常
   - 顺序上按照：最有可能不一致或开销最小的域

注意点：
1. 覆盖equals时总要覆盖hashCode
2. 不要过度寻找等价关系，比如考虑别名形式
3. 不要把equals参数定义为非Object 这样是重载而非重写

## 11. 覆盖equals时总要覆盖hashCode
1. 同个对象多次调用hashCode返回同一个值
2. equals(Object) 相等 hashCode返回整数也相等
3. equals(Object) 不相等 hashCode 有可能相等

Object的hashCode方法为native方法：public native int hashCode();
hashCode注释提到：hashCode返回的是由对象存储地址转化得到的值
```
    As much as is reasonably practical, the hashCode method defined by
    class {@code Object} does return distinct integers for distinct
    objects. (This is typically implemented by converting the internal
    address of the object into an integer, but this implementation
    technique is not required by the
    Java&trade; programming language.)
```
如果没有覆盖hashCode导致两个相同实例具有不同散列码，HashMap有一项优化，可以将每个项相关联的散列码缓存起来，如果散列码不匹配，不会校验对象相等性
好的散列函数倾向于“为不相等的对象产生不相等的散列码”，每个对象都被映射到同一个散列桶中，会实其退化为链表

简单解决方法：
1. 定义 int result ，初试化为对象第一个关键域散列码
2. 对每个关键域f完成这些步骤，得到散列码c
    - 计算f散列值：基本类型 包装类.hashCode(f)；对象引用递归调用hashCode，或者为域计算一个范式，范式调用hashCode；null返回0；数组中没有重要元素用常数代替，都很重要用Arrays.hashCode(f)
    - 累加：result = 31* result + c;
3. 返回 result

使用31原因：
1. 31为奇素数，避免乘以偶数导致的乘法移除信息丢失
2. 乘以31可以用移位和减法代替 31*i == (i<<5) -1

```java
计算机在进行数值运算的时候，是通过补码表示每个数值的
正数原反补相同；负数反码符号位不变，其它位都取反；负数的补码在反码的基础上加1

Java 三种位运算（补码）
<< 左移：丢弃最高位，0补最低位
>> 右移：符号位不变，左边填充符号位
>>> 无符号右移：忽略了符号位，0补最高位
```
Objects类：public static int hash(Object... values) 便捷，但是相对速度慢一些：可变参数引发数组创建，基本类型需要拆箱装箱
不可变类用使用private 变量 缓存hash值， 延迟初始化(lazily initialize)

构造器为：PhoneNumber(short areaCode, short prefix, short lineNum) ，必须强转 (short)1
直接传入整数，否者报错，没有int类型构造器

注意：
1. 不要通过排除关键域来提高性能，反而可能导致实例被映射在极少数散列码上
2. 不要对hashCode返回值做具体规定，可能影响其在未来的改进

## 12. 始终要覆盖toString 
Object实现：类名称@散列码无符号十六进制表示
toString 返回对象中包含的**所有**值得关注的信息

可以在文档中指定返回的格式，并配套静态工厂或者构造器，便于相互转换，JDK例子：BigInteger、BigDecimal、包装类
静态工具类和大多数枚举类编写toString意义不大
## 13. 谨慎地覆盖clone
记得实现Cloneable接口（空的interface），否者抛出异常：java.lang.CloneNotSupportedException
Object中的clone方法：protected native Object clone() throws CloneNotSupportedException;

TODO p46
实现Cloneable接口的类是为了提供一个功能适当复杂的公有clone方法，它无需调用构造器就可以创建对象
不变类永远都不应该提供clone方法

Clone方法就是另一个构造器；必须保证它不会伤害到原始对象，并确保正确地创建被克隆对象中的约束条件
1. 递归调用clone拷贝内部信息
2. 如果域含有对象数组，要注意递归或迭代深拷贝

如果域是final修饰，clone是禁止给final域赋值，Cloneable架构于引用可变对象的final域的正常用法是不相兼容的

线程安全：Object类 clone 没有同步

实现了Cloneable接口的类
1. 都应该覆盖clone方法，且是公有方法，**返回类型为本身**
2. 调用super.clone()
3. 修正域（深拷贝）

拷贝对象更好的方法是提供拷贝构造器和拷贝工厂

最佳实践：用clone复制数组

## 14. 考虑实现Comparable接口 
Comparable接口：public int compareTo(T o);
将这个对象与指定对象比较，大于、等于、小于指定对象返回负整数、零和正整数，类型不匹配抛出RuntimeException：ClassCastException

通用约定
1. sgn(x.compareTo(y)) == -sgn(y.compareTo(x))
2. 可传递：(x.compareTo(y) > 0) && (y.compareTo(z) > 0) -> (x.compareTo(z) > 0)
3. (x.compareTo(y) == 0) 所有的z满足：sgn(x.compareTo(z) == 0) == sgn(y.compareTo(z))

建议：(x.compareTo(y) == 0) -> x.equals(y)

依赖比较关系的类有：TreeSet TreeMap Collections Arrays

与equals相同：无法在用新的值组件拓展课实例化的类时，同时保持compareTo约定，除非放弃面向对象抽象优势；可以通过组合方式实现Comparable接口的类增加值组件（提供“视图” view方法）

```java
        BigDecimal d1 = new BigDecimal("1.0");
        BigDecimal d2 = new BigDecimal("1.00");
        System.out.println(d1.equals(d2)); // false
        System.out.println(d1.compareTo(d2)); // 0
        Set<BigDecimal> bigDecimals = new HashSet<>();
        // equals 比较
        bigDecimals.add(d1);
        bigDecimals.add(d2);
        System.out.println(bigDecimals); // [1.0, 1.00] 

        Set<BigDecimal> treeSets = new TreeSet<>();
        // compareTo 比较
        treeSets.add(d1);
        treeSets.add(d2);
        System.out.println(treeSets); // [1.0]
```
注意Double和Float 使用compare比较而非 < > 
Java7 提供了包装类的静态compare方法，建议在compareTo中使用

从关键域开始逐步比较所有域，某个域产生非零结果立即返回

Java 8 提供了Comparator接口，简洁，但是要付出性能成本
```java
    private static final Comparator<PhoneNumber> COMPARATOR =
            comparingInt((PhoneNumber pn) -> pn.areaCode)
                    .thenComparingInt((PhoneNumber pn) -> pn.prefix)
                    .thenComparingInt((PhoneNumber pn) -> pn.lineNum);

    @Override
    public int compareTo(PhoneNumber pn) {
        return COMPARATOR.compare(this, pn);
    }
```



# 参考资料
[GitHub effective-java-3e-source-code](https://github.com/jbloch/effective-java-3e-source-code)
[Effective Java - 豆瓣](https://book.douban.com/subject/30412517/)
[Guide to WeakHashMap in Java](https://www.baeldung.com/java-weakhashmap)
[java中instanceof和getClass()的作用](https://www.cnblogs.com/aoguren/p/4822380.html)
[Initializing HashSet at the Time of Construction](https://www.baeldung.com/java-initialize-hashset)
[Java Object.hashCode()源码分析](https://blog.csdn.net/changrj6/article/details/100043822)
[通俗易懂的 Java 位操作运算讲解](https://blog.csdn.net/briblue/article/details/70296326)
[Java 位运算(移位、位与、或、异或、非）](https://blog.csdn.net/xiaochunyong/article/details/7748713)