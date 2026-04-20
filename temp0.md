# Spring Framework

## IoC & DI

**IoC控制反转和 DI依赖注入解决的软件工程核心**：**解耦**。**IoC 是设计思想，DI 是具体实现。**

* **传统开发模式：** 当对象 A 需要使用对象 B 的时候，A 会在自己的代码里显式地 `new` 一个 B 的实例。**控制权在 A 手里**。这种方式导致 A 和 B 强耦合，如果 B 的构造方式变了，A 的代码也得跟着改。
* **IoC (Inversion of Control) 控制反转：** 对象 A 不再自己去 `new` 对象 B，而是把创建对象、管理对象的权力**交给了 Spring 容器**。控制权由程序员的代码“反转”给了框架。
* **DI (Dependency Injection) 依赖注入：** 既然 A 自己不创建 B 了，那 A 运行的时候怎么拿到 B 呢？Spring 容器会在 A 实例化之后，自动把 B 传给 A。这个**由容器把依赖对象传递给当前对象的过程**，就叫依赖注入。

---

### DI 的三种主要注入方式

#### 字段注入

字段注入直接在类的属性上加 `@Autowired`，代码最简洁，写起来最快


```JAVA
@RestController
public class UserController {
    
    @Autowired
    private UserService userService; // 直接在字段上打注解

    public void doSomething() {
        userService.login();
    }
}
```
极度不推荐字段注入的理由：

* **脱离容器后抛出 NullPointerException（致命缺点）：**
    * **原理：** Spring 底层是通过**反射机制**，在 `UserController` 实例化（调用默认无参构造方法）**之后**，强行把 `userService` 塞进去的。
    * **后果：** 如果你在写单元测试，脱离了 Spring 环境，直接 `new UserController()` 时，`userService` 并没有被赋值。一旦调用 `doSomething()` 方法，就会立刻引发空指针异常（NPE）。
* **破坏了类的封装性（隐藏了依赖）：**
    * **原理：** 一个正常的 Java 类，应该通过对外暴露的接口（如构造方法、普通方法）来声明自己需要什么外部资源。
    * **后果：** 字段注入把依赖项“藏”在了类内部。别人调用这个类时，根本不知道它到底依赖了哪些东西。
* **无法使用 `final` 关键字：**
    * **原理：** Java 语法规定，`final` 变量必须在声明时或构造方法中初始化。字段注入是在对象实例化之后才赋值的，所以绝对不能加 `final`。这导致你的 Bean 状态是可变的，存在并发安全隐患。
* **掩盖了“单一职责原则”的破坏：**
    * **原理：** 只要你想，你可以无限地在一个类里写 `@Autowired` 注入几十个依赖，代码依然看起来“很整洁”。但这实际上意味着这个类干了太多杂事，严重违背了面向对象设计原则。

**先实例化对象，再注入依赖的危害**：空指针，**破坏了“快速失败 (Fail-Fast)”原则**
- 假设你的代码由于配置疏忽、依赖的 Bean 名称写错，或者某个特定的 Profile 没有激活，导致某个依赖 Bean 根本没有被注入到 Spring 容器中
- **如果用字段/Setter注入**： Spring 可能会正常启动（某些版本或配置下容忍缺失），或者即使报错你也没注意到。服务看似健康地上线了。直到真实用户点击了某个按钮，触发了那行调用缺失依赖的代码，系统才会当场抛出 NullPointerException (NPE) 并崩溃。
- **如果用构造器注入**： 依赖缺失会导致 Bean 根本无法完成实例化。Spring 容器在启动阶段就会直接崩溃报错（Fail-Fast）。这是一种保护机制：把错误暴露在编译或启动阶段，绝不让带病的服务器接管线上流量。


**无法保障并发环境下的“绝对不可变性”**
* 如果“慢慢准备依赖”，意味着这个对象的属性是可以被修改的（不能加 `final` 关键字）。
* 在 Java 的内存模型（JMM）中，没有被 `final` 修饰的变量，在多线程高并发环境下（Web 服务器天生就是多线程环境），存在**指令重排序**和**可见性**的问题。虽然 Spring 容器对单例 Bean 的生命周期管理能最大程度避免这事，但从面向对象设计的严谨性来说，状态可变的对象天生是不安全的。构造器注入配合 `final`，能在对象发布的第一时间，向所有线程保证其状态的绝对可见和不可变。

**纵容了“循环依赖”这种糟糕的设计**
* 字段注入允许 A 依赖 B，B 依赖 A。Spring 为了帮你填坑，搞出了复杂的“三级缓存”机制。
* 但从架构角度看，**循环依赖本身就是设计缺陷**，说明模块划分不清晰。构造器注入因为要求在实例化时必须拿到完整的依赖，所以**天然不支持循环依赖**。一旦代码出现循环依赖，启动直接报错，这就倒逼开发者去重构代码（比如提取公共的第三个类 C），从而写出更健康的架构。


#### Setter 方法注入

**Setter 方法注入**为属性提供 setter 方法，并在方法上加 `@Autowired`
* **优点（灵活性高）：** 允许在对象实例化之后，甚至在程序运行期间，动态地更改注入的依赖对象。
* **缺点（状态不安全）：** Setter 方法是可以被多次调用的。如果是核心强依赖，一旦在运行时被别人错误地调用 Setter 传入了 `null`，整个系统就会崩溃。
  * 即Setter 方法注入的字段不能是 final
* **适用场景：** 当某个依赖是**非必须的（可选的）**，没有它系统也能按默认逻辑运行时；或者需要在运行期间动态改变依赖时，可以使用 Setter 注入。

```java
@RestController
public class UserController {
    
    private UserService userService;

    @Autowired // 注入点在 Setter 方法上
    public void setUserService(UserService userService) {
        this.userService = userService;
    }
}
```

#### 构造器注入

构造器注入是 Spring 官方强烈推荐的依赖注入方式。通过类的构造方法传入外部依赖，不仅能保证代码的健壮性，还能与面向对象设计原则完美契合。

相比于字段注入，构造器注入具有不可替代的优势：
* **防空指针（保证强依赖强制初始化）：** Java 语法规定，实例化对象必须调用构造方法。这从根源上杜绝了对象创建完毕但依赖尚未准备好的情况，只要 Bean 实例化成功，依赖一定不为 `null`。
* **支持 `final` 关键字（保证不可变性与线程安全）：** 完美契合 `final` 变量必须在构造时赋值的语法要求。依赖一旦注入便不可更改，保障了单例 Bean 在多线程环境下的绝对安全。
* **完美支持单元测试：** 脱离 Spring IoC 容器后，可以通过常规的 `new` 关键字手动传入 Mock 对象（如 `new UserController(mockUserService)`），测试代码更纯粹。
* **主动暴露“代码异味” (Code Smell)：** 如果一个类依赖过多（例如构造方法有 15 个参数），代码会非常臃肿，这就良性地“逼迫”开发者审视该类是否违背了单一职责原则，进而进行拆分重构。

```java
@RestController
public class UserController {
    // 推荐加上 final 关键字
    private final UserService userService;

    // 通过构造方法传入依赖
    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

**最佳实践：结合 Spring 4.3+ 与 Lombok 精简代码**
* **Spring 4.3 隐式注入特性：**
    * **规则：** 如果一个类**只定义了一个构造器**，Spring 会默认使用该构造器进行自动装配，无需显式标注 `@Autowired`。
    * **优势：** 代码更整洁，天然鼓励“单构造器 + 不可变字段”的优秀实践。
* **Lombok `@RequiredArgsConstructor` 注解：**
    * **作用：** 自动生成一个包含类中所有 `final` 字段和带有 `@NonNull` 注解字段的有参构造器。
    * **注意点：** 如果类中既没有未初始化的 `final` 字段，也没有 `@NonNull` 字段，该注解会静默生成一个无参构造器（属于误用，但不报错）。若确实只需无参构造器，应明确使用 `@NoArgsConstructor`。


---

#### `@Autowired` 行为与多构造器解析规则

当类中存在多个构造器时，Spring 会按特定规则寻找合适的构造器来实例化 Bean：

* **`@Autowired` 的 `required` 属性（默认值为 `true`）：**
    * `required = true`：强制依赖。找不到匹配 Bean 时，启动抛出 `NoSuchBeanDefinitionException`，初始化失败。
    * `required = false`：可选依赖。找不到时，Spring 忽略它并尝试正常启动。

* **Spring 多构造器解析步骤与规则：**
    1. **唯一性直接使用：** 若有且仅有一个构造器，直接使用（无需注解）。
    2. **显式首选：** 若有多个构造器，被 `@Autowired` 标记的会成为“首选构造器”。
    3. **冲突报错：** 若一个构造器标为 `@Autowired(required = true)`，此时若有任何其他带有 `@Autowired` 的构造器（无论 true/false），直接启动报错。
    4. **贪婪匹配 (Greedy Matching)：** 允许将所有带 `@Autowired` 构造器的 `required` 设为 `false`。此时 Spring 会检查所有候选者，优先选择**参数最多且能被 Spring 容器完全满足**的构造器。
    5. **单构造器 + `required=false` 的反模式行为：** 如果你**只有一个带参构造器**，却标记为 `@Autowired(required = false)`，当依赖缺失时：
        * 若类中还有无参构造器：退化调用无参构造器生成残缺 Bean（需手动判空）。
        * 若没有无参构造器：直接抛出实例化异常，而不会硬塞 `null`。
        * **设计定性：** 这种做法在设计上是**错误**的。`required = false` 语义是“可选”，而唯一带参构造器语义是“强制”，两者自相矛盾。
        * 
---







#### 注入方案选择

在真实的工程实践中，推荐将依赖严格划分为**强制依赖**和**可选依赖**：

**代码精简最佳实践：结合 Spring 4.3+ 与 Lombok**
* **Spring 4.3 隐式注入特性：** 若类**只定义了一个构造器**，Spring 会默认使用该构造器自动装配，无需显式标 `@Autowired`。
* **Lombok `@RequiredArgsConstructor` 注解：** 自动生成包含所有 `final` 字段和 `@NonNull` 字段的有参构造器。
  * *注意：* 若类中既无未初始化的 `final` 字段也无 `@NonNull` 字段，该注解会静默生成无参构造器（属于误用）。若确需无参，应明确使用 `@NoArgsConstructor`。

1. **强制依赖：** 完全由**构造器注入**完成。
2. **可选依赖：** 完全由 **Setter 方法注入**完成。
3. **CGLIB 代理兼容性保证：**
   * 使用 `@NoArgsConstructor` 生成一个无参构造器，并总是确保**至多只有一个**有参构造器来注入强制依赖。
   * *原因：* Spring AOP 底层依赖 CGLIB 动态代理（通过继承目标类生成子类来实现增强）。CGLIB 要求父类**必须拥有无参构造器**，否则子类无法调用 `super()` 初始化父类状态。
---

### IoC 容器的核心大脑：BeanFactory 与 ApplicationContext

在 Spring 框架中，IoC 容器是负责实例化、配置和装配对象的大管家。Spring 主要提供了两种核心容器接口来实现这些功能。


#### `BeanFactory`



* **定义：** `BeanFactory` 是 Spring IoC 容器的最顶层接口，是所有 Spring 容器的“老祖宗”。
* **本质：** 它是一个 **工厂模式（Factory Pattern）** 的超级实现。它负责生产和管理系统中的所有 Bean。
* **设计哲学：** 极简主义。它只负责定义最基础的 IoC（控制反转）和 DI（依赖注入）规范，不包含任何高级的系统级服务（如 AOP、事件机制等）。


---

`BeanFactory` 的核心工作可以总结为“看图纸，造零件”：

* **看图纸（解析 BeanDefinition）：** Spring 不会直接把 Java 类变成 Bean。它会先通过各种 Reader（如 `XmlBeanDefinitionReader` 或注解解析器）读取配置（XML、注解或 JavaConfig），把类的特征（类名、依赖关系、是否单例、懒加载等）封装成一个中间模型 —— **`BeanDefinition`**（Bean 的图纸）。
* **造零件（实例化与注入）：**
  当调用 `getBean()` 时，`BeanFactory` 会拿着这张 `BeanDefinition` 图纸，利用反射机制实例化对象，并根据图纸上的说明，把需要的依赖项（其他 Bean）装配进去。

---

**BeanFactory 的四大派系（重要子接口）**
- `BeanFactory` 本身的方法很少，Spring 通过一系列子接口扩展了它的能力，这在源码阅读中非常关键：

1. **`ListableBeanFactory`（可列表的工厂）：**
   * **能力：** 突破了顶级接口只能按名字单查的限制，允许按类型、按注解批量获取系统中有哪些 Bean。
   * **常见方法：** `getBeansOfType()`, `getBeanDefinitionNames()`。
2. **`HierarchicalBeanFactory`（分层工厂）：**
   * **能力：** 引入了“父子容器”的概念。允许一个 BeanFactory 拥有一个 Parent。
   * **应用场景：** Spring MVC 中最经典的“父子容器”（Spring 容器做父，装配 Service/Dao；Spring MVC 容器做子，装配 Controller）。子可以访问父的 Bean，父不能访问子的 Bean。
3. **`AutowireCapableBeanFactory`（具备自动装配能力的工厂）：**
   * **能力：** 提供向现有脱管的 Java 实例（非 Spring 创建的对象）强制进行依赖注入的能力。
4. **`DefaultListableBeanFactory`（🔥 终极集大成者）：**
   * **地位：** 这是 Spring 中**最核心的类，没有之一**。它是上述所有接口的默认实现类。
   * **真相：** 你平时用的 `ApplicationContext`，其底层也是偷偷 new 了一个 `DefaultListableBeanFactory` 来干活的（组合模式）。真正造 Bean 的苦力，永远是它。

---

BeanFactory 的核心 API 概览

* `getBean(String name)` / `getBean(Class<T> requiredType)`：最常用的获取 Bean 的方法。
* `containsBean(String name)`：判断容器中是否包含指定名字的 Bean。
* `isSingleton(String name)`：判断某个 Bean 是否是单例模式。
* `isPrototype(String name)`：判断某个 Bean 是否是多例（原型）模式。
* `getType(String name)`：获取某个 Bean 的 Class 类型。
* `getAliases(String name)`：获取某个 Bean 的所有别名。

---

延迟加载的底层表现
- 正如我们之前对比的，`BeanFactory` 最大的特性是**延迟加载**。

* **启动阶段：** 仅仅是读取了配置，将配置转化为 `BeanDefinition` 存入内存的 `ConcurrentHashMap` 中（图纸画好了）。此时没有实例化任何业务 Bean。
* **运行阶段：** 当你第一次执行 `beanFactory.getBean("userService")` 时，底层才会触发类的实例化、属性注入、初始化方法（`@PostConstruct`）等一整套 Bean 的生命周期。


---

为什么现在工程中几乎看不到直接使用 BeanFactory？
- 在早期学习 Spring 或者写极简 Demo 时，你可能会看到这样的代码：
```java
// 传统 BeanFactory 写法（现已极其少见）
Resource resource = new ClassPathResource("beans.xml");
BeanFactory factory = new XmlBeanFactory(resource); // XmlBeanFactory 已被废弃
UserService user = factory.getBean(UserService.class);
```
**被淘汰的原因：**
1. **API 太底层：** 很多功能（如 BeanPostProcessor 的注册）需要手动硬编码调用，极不方便。
2. **缺乏企业级功能：** 不支持 AOP、事务声明、事件发布、环境变量解析。
3. **隐藏配置错误：** 懒加载会导致依赖配置的 Bug 被延迟到运行时才爆炸，这在大型线上系统中是无法容忍的。

**总结结论：** `BeanFactory` 是 Spring 架构设计的地基，理解它能帮助你彻底看懂 Spring IoC 的源码脉络；但在应用开发层面，请永远信任并使用它的进阶版：`ApplicationContext`。



---

#### `ApplicationContext`



`ApplicationContext`核心定位
* **定义：** 它是 Spring 中的高级容器接口，代表了真正的“Spring 应用上下文”，也是日常企业级开发中我们直接打交道的核心对象。
* **底层真相（组合模式）：** 面试常考的一个误区是认为 `ApplicationContext` 覆写了 `BeanFactory` 所有的造 Bean 逻辑。**并不是！**
  `ApplicationContext` 内部其实**持有一个** `DefaultListableBeanFactory` 的实例。遇到创建 Bean、获取 Bean 的脏活累活，它会直接委托给底层的 `BeanFactory` 去做。它自己则腾出手来，专注于提供更高阶的企业级服务。

---

**四大核心扩展能力**
- 为什么它被称为企业级标准？因为它通过继承不同的接口，集成了四大杀手锏功能：

1. **环境与配置管理 (`EnvironmentCapable`)：**
   * **能力：** 统一管理系统的环境变量、JVM 参数、以及我们熟悉的 `application.properties/yml` 配置文件。
   * **实战：** 我们常用的 `@Value("${xxx}")` 属性注入，以及 `@Profile("dev")` 环境隔离，底层都是由它支撑的。
2. **事件发布与监听 (`ApplicationEventPublisher`)：**
   * **能力：** 提供了开箱即用的**观察者模式**实现。允许 Bean 之间通过发布和监听事件进行解耦通信。
   * **实战：** 业务代码中使用 `applicationContext.publishEvent(new OrderCreatedEvent())` 发布事件，另一个类使用 `@EventListener` 异步监听处理（比如发短信），做到主链路与副链路彻底解耦。
3. **统一资源加载 (`ResourcePatternResolver`)：**
   * **能力：** 屏蔽了底层文件系统的差异，提供极其强大的资源读取能力。
   * **实战：** 可以轻松使用 `classpath:mappers/*.xml` 或 `file:/etc/config.json` 这种通配符路径去一次性读取多个配置文件。
4. **国际化支持 (`MessageSource`)：**
   * **能力：** 支持根据不同的国家/语言环境（Locale），返回不同的文本信息（如错误提示语）。

---

预加载与 Fail-Fast（快速失败）原则
- 这是它在架构设计上与 `BeanFactory` 最核心的差异。

* **预加载机制（Eager-load）：** 在容器启动阶段，`ApplicationContext` 会找出所有作用域为 `singleton`（单例）且没有标记为懒加载的 Bean，并**一次性将它们全部实例化、注入依赖并初始化完毕**。
* **架构优势（Fail-Fast）：** * 任何配置错误（类名写错、包扫不到）、依赖缺失（找不到对应的 `@Autowired` Bean）、或者代码循环依赖，都会在**服务器启动的这几秒钟内当场报错，并终止启动**。
  * 这种设计绝不把隐患留到运行时，极大地保障了线上生产环境的安全。

---

**常见的实现类**（容器的实体）
- 在不同时代的 Spring 技术栈中，我们会使用不同的 `ApplicationContext` 实现类来启动容器：

* **`AnnotationConfigApplicationContext`（纯注解时代的主力）：**
  * **场景：** 现代无 XML 的纯 Java 架构。基于 `@Configuration` 配置类和 `@ComponentScan` 包扫描来构建容器。
* **`ClassPathXmlApplicationContext`（古典时代的遗迹）：**
  * **场景：** 老旧的 SSM/SSH 项目。从 ClassPath 路径下读取传统的 `applicationContext.xml` 文件来启动。
* **`AnnotationConfigServletWebServerApplicationContext`（Spring Boot Web 的基石）：**
  * **场景：** 当你写下 `SpringApplication.run()` 启动一个 Spring Boot Web 项目时，底层悄悄实例化的就是这个长名字的怪物。它不仅拥有上述所有能力，还能自动内嵌并启动 Tomcat / Undertow 服务器。


Spring Boot 中，不需要手动指定实现类，而是直接在启动类中`SpringApplication.run(MyApplication.class, args);`启动
- 它会**自动挑选合适的实现类**，如果你写的是 Web 程序，Spring 会自动选一个支持 Servlet 的实现类；如果只是普通的控制台程序，它会选一个轻量级的实现类，无需手动干预选择过程。

```JAVA
public static void main(String[] args) {
    // 这行代码执行完，实现类就已经在后台默默运行了
    SpringApplication.run(MyApplication.class, args);
}

```


---

容器的“大动脉”：`refresh()` 方法
- 如果你准备翻阅 Spring 源码，`ApplicationContext` 接口中最重要的一个方法叫 **`refresh()`**。
* 它是整个 Spring 容器启动的入口和核心流程。
* 这个方法内部有著名的 **13 个步骤**，严格定义了 Spring 容器是如何一步步从解析配置、到调用各种处理器（BeanFactoryPostProcessor）、再到注册监听器、最后完成所有 Bean 实例化的全生命周期。可以说是 Java 世界里最经典的模板方法模式（Template Method Pattern）应用。
* 当 SpringApplication.run() 执行时，它内部调用了 refresh()，然后你的 Bean 就全部出生并准备就绪了

---

**总结结论：** 在实际工程中，除非你正在开发极其底层的框架级组件，或者运行环境的内存只有几兆大小（比如某些老式物联网设备），否则**永远只使用 `ApplicationContext`**。它的预加载机制和丰富的生态扩展，是现代 Java 企业级开发的绝对基石。


---

### 4. Bean 的作用域
在 IoC 容器中，Bean 的生命形态是不同的，最常用的有两大类：

* **Singleton（单例，默认）：** 整个 Spring 容器中，这个 Bean 只有一个实例。不管你请求多少次，拿到的都是同一个对象。适用于**无状态**的类（如普通的 Service、Dao）。
* **Prototype（原型 / 多例）：** 每次请求这个 Bean，Spring 都会为你 `new` 一个全新的实例。适用于**有状态**的类。

> **⚠️ 避坑指南：** 绝对不要在单例 Bean 中声明带有状态的全局变量（比如在单例 Controller 里定义一个 `int count` 记录访问次数），因为所有线程共享这个实例，会引发严重的线程安全（并发）问题。

---

### @Autowired 与 @Resource 的区别
在进行依赖注入时，这两个注解最常被拿来对比：

| 特性 | `@Autowired` | `@Resource` |
| :--- | :--- | :--- |
| **来源** | Spring 框架原生提供 | Java EE 规范提供 (JSR-250) |
| **默认匹配方式** | **按类型 (byType)** 优先 | **按名称 (byName)** 优先 |
| **如果存在多个同类型** | 需配合 `@Qualifier("beanName")` 指定名字 | 如果找不到同名，再退化为按类型寻找 |

这部分内容中，**循环依赖**（例如对象 A 依赖对象 B，对象 B 又依赖对象 A）以及 Spring 是如何用**“三级缓存”**解决单例对象循环依赖的，是进阶面试中最常被深挖的底层原理。需要我为你详细拆解 Spring 的三级缓存机制吗？

### 环境与配置管理

### 资源加载

## AOP
解决非业务代码（如日志记录、权限校验、性能监控）与核心业务代码解耦的问题。

* **底层原理:** 动态代理机制的设计模式（JDK 动态代理 vs CGLIB 动态代理的适用场景与区别）。
* **核心术语:** 切面 (Aspect)、切入点 (Pointcut)、通知/增强 (Advice)、连接点 (Joinpoint)。
* **通知类型:** 前置 (`@Before`)、后置 (`@AfterReturning`)、异常 (`@AfterThrowing`)、最终 (`@After`)、环绕 (`@Around`，最强大也最常用)。
* **实战应用:** 编写自定义注解，结合 AOP 实现方法级别的日志自动记录或接口防刷。

## 数据访问与事务管理
封装传统 JDBC 繁琐的操作，并提供统一且优雅的事务管理能力。

* **Spring JDBC:** `JdbcTemplate` 的基本使用与 CRUD 操作。
* **声明式事务 (Declarative Transaction):**
    * `@Transactional` 注解的基本使用及其底层 AOP 原理。
    * **核心考点一：事务的传播行为 (Propagation)**（例如 `REQUIRED`, `REQUIRES_NEW`, `NESTED` 等 7 种机制的真实业务场景分析）。
    * **核心考点二：事务的隔离级别 (Isolation)** 与数据库底层锁机制的配合。
    * **避坑指南:** 常见 `@Transactional` 事务失效的 6 大场景（如：非 public 方法、同类内部方法互相调用、异常被 try-catch 吞掉等）。

## Spring MVC
经典的 Java Web 表现层框架，它是目前绝大多数后台接口的基石，也是 Spring Boot Web 启动器底层的核心。

* **核心执行流程 (必考):** 用户请求 -> `DispatcherServlet` (核心分发) -> `HandlerMapping` -> `Controller` -> `ViewResolver` (前后端分离后通常省略) -> 响应返回。
* **核心注解的使用:**
    * 请求映射 (`@RequestMapping`, `@GetMapping`, `@PostMapping` 等)。
    * 参数绑定与接收 (`@RequestParam`, `@PathVariable`, `@RequestBody`, `@RequestHeader` 等)。
    * 数据校验 (`@Valid` / `@Validated` 结合 Hibernate Validator)。
* **RESTful 架构:** `@RestController`（`@Controller` + `@ResponseBody`）的原理。
* **高级机制:**
    * **拦截器 (Interceptor):** `HandlerInterceptor` 的配置与执行时机（掌握它与 Servlet Filter 的本质区别）。
    * **全局异常处理:** `@ControllerAdvice` 配合 `@ExceptionHandler` 实现统一错误响应。

### 国际化支持

## 高级特性与底层源码探究
当你掌握了上述使用方法后，拔高阶段需要了解底层原理。

* **事件驱动模型:** Spring Event 机制（`ApplicationEvent` 与 `@EventListener`），实现基于发布/订阅模式的业务解耦。
* **循环依赖问题:** Spring 是如何通过“三级缓存”机制解决属性注入的循环依赖的（面试高频）。
* **容器启动流程:** `AbstractApplicationContext.refresh()` 方法的核心骨架（知道 Bean 是在哪个阶段被扫描、定义和实例化的）。

---

这份大纲涵盖了 Spring Framework 最核心的骨架。你目前是处于零基础准备刚刚入门 Spring 的阶段，还是已经有一定项目经验想要查漏补缺准备面试呢？