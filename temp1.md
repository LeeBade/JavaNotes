### Bean


#### Bean的定义与身份

在 Spring 中，Bean 并不是凭空产生的，容器在实例化 Bean 之前，首先会将我们的配置（XML、注解等）解析成一种内部的元数据结构——**`BeanDefinition`**。


**Bean 的元数据**：Spring 是怎么描述一个对象的？

- `BeanDefinition` 就好比是建造 Bean 的“图纸”。即使你写了一个普通的 Java 类，Spring 也需要将其包装成 `BeanDefinition` 才能进行管理。它包含了以下关键信息：

  * **Bean 的全限定类名**（包名+类名，用于反射创建对象）。
  * **作用域（Scope）**：单例、多例等。
  * **行为配置**：是否懒加载（Lazy）、是否是首选（Primary）。
  * **生命周期回调**：初始化方法名（`init-method`）、销毁方法名（`destroy-method`）。
  * **依赖信息**：构造函数参数、属性值（用于依赖注入）。

---


**配置方式对比**

| 配置方式 | 核心表现 | 适用场景 | 优缺点 |
| :--- | :--- | :--- | :--- |
| **XML 配置** | `<bean id=".." class="..">` | 维护遗留老系统。 | **优**：统一管理，代码零侵入。<br>**缺**：配置繁琐，容易导致 XML 地狱，无法做到类型安全。 |
| **注解驱动** | `@Component`及其衍生 (`@Service`, `@Controller`) | 我们自己编写的业务代码。 | **优**：开发极快，和代码高内聚。<br>**缺**：相对分散，找起来需要全局搜索；对第三方包里的类无能为力（因为你不能去改别人的源码加注解）。 |
| **JavaConfig** | `@Configuration` 类内部的 `@Bean` 方法 | 引入第三方库的组件（如 `DataSource`、`RestTemplate`）。 | **优**：类型安全，集中配置，非常灵活（可以在方法里写复杂的实例化逻辑）。<br>**缺**：仍需要编写额外的配置类。 |

-----

JavaConfig 的核心在于两个注解：`@Configuration`（标注配置类）和 `@Bean`（标注产生 Bean 的方法）。

**代码示例：**

```java
@Configuration // 告诉 Spring：这是一个配置类，相当于以前的 XML 文件
public class AppConfig {

    // 示例 1：定义一个普通的业务 Bean
    @Bean
    public UserService userService() {
        // 这里可以手动控制对象的创建过程
        return new UserServiceImpl(userRepository()); 
    }

    // 示例 2：定义另一个 Bean（依赖注入）
    @Bean
    public UserRepository userRepository() {
        return new UserRepositoryImpl();
    }
}
```

**为什么引入第三方组件还需要配置？**
- 这是一个非常经典的问题。既然有了 `@Component` 自动扫描，为什么还要写配置？
* **无法修改源码**：当你引入 MyBatis 的 `SqlSessionFactory` 或阿里巴巴的 `DruidDataSource` 时，这些类都在 `.jar` 包里。你不能打开人家的源码，在类名上面加一个 `@Component`。
* **实例化过程复杂**：很多第三方组件的创建不是简单的 `new` 一下。比如 `DataSource` 需要设置数据库 URL、用户名、密码；`RestTemplate` 可能需要配置连接超时时间。
* **控制权归属**：Spring 必须知道“如何”去创建这个对象。通过 JavaConfig，你可以精确控制第三方类的初始化逻辑（比如调用 setter 方法或有参构造器）。

**这个配置在哪里？**
- 在 Spring Boot 或现代 Spring 项目中，配置通常分布在以下地方：

  1.  **项目内部的配置类**：通常在项目的 `config` 包下。比如 `RedisConfig.java`、`SecurityConfig.java`。
  2.  **自动配置（Auto-Configuration）**：这是 Spring Boot 的核心。Spring Boot 内部预写了成百上千个 JavaConfig 类（藏在 `spring-boot-autoconfigure.jar` 里）。当你引入一个 starter 依赖时，它会自动生效，除非你手动定义一个同名的 Bean 来覆盖它。
  3.  **配置文件**：虽然 JavaConfig 负责“怎么创建”，但“具体的参数”（如 IP 地址、密码）通常放在 `application.properties` 或 `application.yml` 中，然后通过 `@Value` 或 `@ConfigurationProperties` 注入到 JavaConfig 中。

---

#### 给Bean取名


**给 Bean 取名**
- 默认情况下，Spring 会把类名的首字母小写作为 Bean 的名字（比如 `UserService` 变成 `userService`）。但如果你想自己做主，有以下几种常见方式：
  1. **在模式注解上直接指定**
       - 通过 `@Component`、`@Service`、`@Controller` 等注解的 `value` 属性直接命名：
        ```java
        // 我偏不叫 aliPayService，我就要叫 aliPay
        @Service("aliPay") 
        public class AliPayService implements PayService { ... }
        ```

  2. **在 `@Bean` 方法上指定**
        - 在使用 `@Configuration` 配置类手动创建 Bean 时，可以通过 `name` 属性指定（甚至可以起多个名字作为别名）：
        ```java
        @Configuration
        public class MyConfig {
            @Bean(name = {"myRedisTemplate", "redisTool"})
            public RedisTemplate redisTemplate() { ... }
        }
        ```

---

**为什么要给 Bean 取名？**
- 在简单的增删改查项目里，默认名字确实够用了。但一旦业务变复杂，自定义 Bean 名字就成了刚需。以下是三个最核心的实战场景：
- **场景 1：解决“多胞胎”冲突（消除歧义）**
  - 假设你有一个接口 `PayService`，它有两个实现类：`AliPayServiceImpl` 和 `WechatPayServiceImpl`。
如果你在别的类里直接这样写：
    ```java
    @Autowired
    private PayService payService; // 报错警告！
    ```
  - **Spring 会当场崩溃**（报 `NoUniqueBeanDefinitionException`）。因为容器里有两个 `PayService` 类型的 Bean，Spring 就像个迷茫的老父亲：“这俩多胞胎，你到底要我抱哪个给你？”

  - **破局思路：利用名字精确打击**，先给实现类取好名字，再配合 `@Qualifier` 或者 `@Resource` 按名字注入：
    ```java
    @Service("wechatPay")
    public class WechatPayServiceImpl implements PayService { ... }

    @Service("aliPay")
    public class AliPayServiceImpl implements PayService { ... }

    // 使用方：
    @Autowired
    @Qualifier("aliPay") // 明确告诉 Spring：我要名字叫 aliPay 的那个！
    private PayService payService;
    ```

- **场景 2：结合 Map 实现优雅的“策略模式”（封神级用法）**
  -  这是高级开发最爱用的一招，用来消灭代码里成百上千行的 `if-else`。

  - 如果你在 Service 里注入一个 `Map<String, 接口>`，Spring 会施展一个魔法：**它会自动把容器里所有实现该接口的 Bean 全找出来，把它们的【Bean 名字】作为 Key，【Bean 实例】作为 Value，塞进这个 Map 里！**

    ```java
    @Service
    public class OrderService {
        
        // Spring 的魔法注入：Key 就是 Bean 的名字（"aliPay" 或 "wechatPay"）
        @Autowired
        private Map<String, PayService> payStrategyMap; 

        public void checkout(String payType) {
            // 前端传过来 payType = "aliPay"，直接去 Map 里拿对应的 Bean，连 if-else 都不用写！
            PayService targetService = payStrategyMap.get(payType);
            targetService.pay();
        }
    }
    ```
  - 在这个场景里，**精确控制 Bean 的名字，就是控制策略路由的 Key。**

- **场景 3：AOP 面向切面的“精准拦截”**
  -  在写 AOP 切面时，我们通常按包名或注解来拦截方法。但有时候，你只想拦截特定的几个 Bean。
  -  在 AspectJ 的切点表达式里，可以直接使用 `bean(你的Bean名字)`：
    ```java
    // 只拦截名字叫 wechatPay 或者以 Service 结尾的 Bean
    @Pointcut("bean(wechatPay) || bean(*Service)") 
    public void pointcut() {}
    ```

---




---




#### Bean 的作用域Scope
##### Singleton单例作用域

在 Spring 中，如果你不显式指定，所有的 Bean 默认都是单例的。

* **核心机制**：对于同一个 Bean 定义，Spring IoC 容器中**只存在一个共享的实例**。无论你通过 `@Autowired` 注入多少次，或者通过 `context.getBean()` 获取多少次，拿到的永远是内存中的同一个对象。
* **创建时机**：默认情况下，单例 Bean 在 **Spring 容器启动时**就会被实例化并初始化好（除非你加了 `@Lazy` 延迟加载）。

---

**线程安全问题**
- 由于单例 Bean 在整个应用中共享，如果多个线程同时访问并修改它的成员变量，就会引发线程安全问题。

* **无状态 Bean（安全）**：比如我们常用的 `xxxController`、`xxxService`、`xxxDao`。它们通常只有方法（执行逻辑），没有用来存储数据的成员变量（或者只有被 `final` 修饰的依赖组件）。这种 Bean 是线程安全的。
* **有状态 Bean（危险）**：如果你的单例 Bean 里定义了一个普通的成员变量（比如 `private int count = 0;`），多个请求并发调用修改 `count` 时，数据必然错乱。
* **破局方案**：
    1.  尽量避免在单例 Bean 中定义可变的成员变量。
    2.  如果必须存储状态，使用 `ThreadLocal` 将变量绑定到当前线程。
    3.  使用同步锁（`synchronized` 或 `ReentrantLock`），但会严重影响性能，极不推荐。

---
##### Prototype多例作用域

当你使用 `@Scope("prototype")` 标注一个 Bean 时，它就变成了多例。

* **核心机制**：每次向 Spring 容器请求这个 Bean（通过注入或 `getBean`）时，容器都会 **重新 `new` 一个全新的实例** 给你。
* **创建时机**：容器启动时**不会**创建多例 Bean，只有在被请求时才创建。
* **生命周期管理差异**：这是一个容易被忽略的点。对于单例，Spring 管杀也管埋（负责销毁）；但对于多例，**Spring 把对象创建好交给你之后，就不再管理它了**。多例 Bean 的 `@PreDestroy` 销毁方法永远不会被 Spring 调用，回收工作完全交由 JVM 垃圾回收器负责。

---

##### 当单例依赖多例

这是 Spring 中最经典、最容易让人抓狂的坑，请务必留意。

- **场景还原**：假设你有一个单例的 `OrderService`，它内部通过 `@Autowired` 注入了一个多例的 `OrderIdGenerator`（用于生成唯一的订单号）。

- **现象：**
你期望每次调用 `OrderService.generate()` 时，都能用到一个全新的 `OrderIdGenerator` 实例。但实际运行你会发现，无论调用多少次，`OrderIdGenerator` **始终是同一个**，多例失效了！

- **原理解析：**
Spring 注入依赖的动作**只发生一次**（在 `OrderService` 这个单例 Bean 初始化的时候）。那时候 Spring 确实去拿了一个新的 `OrderIdGenerator` 塞给了 `OrderService`。但因为 `OrderService` 是单例的，它永远活在内存里，它肚子里的那个 `OrderIdGenerator` 也就被“定格”了，再也不会刷新。

- 错误代码演示
```JAVA

// 1. 定义一个多例的组件
@Component
@Scope("prototype") // 标记为多例：每次获取都应该创建新对象
public class OrderIdGenerator {
    public OrderIdGenerator() {
        System.out.println("====== OrderIdGenerator 被 new 出来了 ======");
    }
    
    public String generateId() {
        return "ID-" + System.currentTimeMillis();
    }
}

// 2. 定义一个单例的 Service
@Service
public class OrderService {
    
    // 【致命错误点】：直接使用 @Autowired 注入多例对象
    @Autowired
    private OrderIdGenerator orderIdGenerator; 
    
    public void generate() {
        // 无论你调用这个方法多少次，用的都是同一个 orderIdGenerator 实例
        System.out.println("使用的生成器实例: " + orderIdGenerator);
        orderIdGenerator.generateId();
    }
}
```

---

**优雅的解决方案：**
- 不要再用 `@Autowired` 直接注入多例 Bean。**使用`@Lookup` 注解**
- 在单例 Bean 中写一个返回多例 Bean 的方法，并打上 `@Lookup` 注解。Spring 会使用 CGLIB 字节码技术动态重写这个方法，每次调用都去容器里拿新的实例。
    ```java
    @Service
    public class OrderService {
        // 每次调用这个方法，Spring 都会拦截并返回一个新的多例对象
        @Lookup
        protected OrderIdGenerator getOrderIdGenerator() {
            return null; // 方法体随意，因为 Spring 运行时会覆盖它，压根不会运行方法体内的代码
        }
        
        public void generate() {
            OrderIdGenerator generator = getOrderIdGenerator(); // 拿到的是新实例！
        }
    }
    ```

- 当 Spring 启动并扫描到你的 OrderService 里有 @Lookup 注解时，它不会把原始的 OrderService 放到容器里。相反，它会在内存里动态地写一个 OrderService 的子类，并把这个子类放进容器里供大家使用。
- 我们可以想象 Spring 在底层悄悄帮你生成了类似下面这段代码（伪代码）：
```JAVA
// 这是 Spring 动态生成的子类（代理类），它继承了你写的 OrderService
public class OrderService$$EnhancerBySpringCGLIB extends OrderService {
    
    // Spring 强行重写（覆盖）了你的方法！
    @Override
    protected OrderIdGenerator getOrderIdGenerator() {
        // 它根本不会去执行你写的 return null;
        // 而是直接向 Spring 容器要一个全新的对象！
        return applicationContext.getBean(OrderIdGenerator.class); 
    }
}
```

---

##### Web 专属作用域

在引入 `spring-webmvc` 后，Spring 会在原有的 Singleton（单例）和 Prototype（原型）基础上，增加与 HTTP 生命周期绑定的特殊作用域。

---

Session作用域 & Application作用域
* **Session 作用域**：同一个 HTTP Session 共享同一个 Bean 实例。传统项目中常用于存储购物车、登录态，但在如今前后端分离、JWT 无状态认证的架构下，**已基本淘汰，极少使用**。
* **Application 作用域**：生命周期与 Web 容器的 `ServletContext` 一致。在绝大多数情况下，它的表现和普通的 Singleton（单例）没有区别，**通常直接使用默认的单例即可**。

---

**Request 作用域**
- **每次发起一个完整的 HTTP 请求，Spring 都会为该请求创建一个全新的 Bean 实例；当请求处理完毕并返回响应后，该 Bean 自动销毁**。
- **核心场景**：非常适合作为**当前请求上下文**来使用。例如：存储拦截器解析出来的当前用户信息（User ID）、链路追踪的 TraceId 等。
* **好处**：**避免了在 Controller -> Service -> DAO 的整条调用链路上，把 `userId` 作为方法参数痛苦地传来传去。**

---

**为什么 Request 作用域必须依赖 `proxyMode`？**

**1. 致命的“寿命冲突”问题**
通常，我们的 Service 是默认的**单例（寿命极长，随项目启动而生）**。如果我们想在 Service 中注入一个 Request 作用域的 Bean（**寿命极短，请求来了才生，请求结束就死**），会发生什么？
* 项目启动阶段，Spring 尝试实例化 Service，并试图把 Request Bean 注入进去。
* **但此时根本没有任何 HTTP 请求！** Spring 找不到 Request 对象，直接导致启动报错；或者发生严重的“数据串门”（所有请求复用了同一个假请求对象）。

**2. 破局之道：`proxyMode` (代理模式)**
为了解决长生命周期依赖短生命周期的问题，我们在声明 Request 作用域时，必须加上 `proxyMode`（生成代理对象）。
* **原理**：Spring 启动时，并不会把真正的 Request Bean 注入给 Service，而是给 Service 塞入一个**空壳代理对象（Proxy）**。
* **运行**：当真实的 HTTP 请求进来，调用到 Service 执行相关代码时，这个代理对象会动态去当前线程的 HTTP 请求中，抓取**属于当前请求的、真正的** Request Bean 来执行逻辑。

---

**代码模板**

以下是一个使用 Request 作用域存储当前用户上下文的完整闭环代码：

**1. 定义 Request 作用域的上下文对象（必须加 proxyMode）**
```java
import org.springframework.context.annotation.Scope;
import org.springframework.context.annotation.ScopedProxyMode;
import org.springframework.stereotype.Component;
import org.springframework.web.context.WebApplicationContext;

@Component
// 声明为 Request 作用域，并告诉 Spring 使用 CGLIB 生成目标类的代理对象
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class CurrentUserContext {
    
    private String userId;
    private String userName;

    // 省略 getter 和 setter
    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }
}
```

**2. 在拦截器（Interceptor）或 Filter 中初始化数据**
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class AuthInterceptor implements HandlerInterceptor {

    @Autowired
    private CurrentUserContext currentUserContext; // 注入的是代理对象

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // 模拟从请求头中的 JWT Token 解析出 userId
        String token = request.getHeader("Authorization");
        String userId = parseTokenToGetUserId(token); 
        
        // 将 userId 塞入当前请求专属的 Context 中
        currentUserContext.setUserId(userId); 
        return true;
    }
}
```

**3. 在任意深度的单例 Service 中优雅使用**
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class OrderService { // 默认单例

    @Autowired
    private CurrentUserContext currentUserContext; // 注入的是代理对象

    public void createOrder() {
        // 直接获取当前用户的 ID，不需要 Controller 通过参数传进来！
        // 代理对象会自动去当前 HTTP 请求的专属 Context 里拿数据，绝对不会拿错成别人的
        String currentUserId = currentUserContext.getUserId();
        
        System.out.println("正在为用户 " + currentUserId + " 创建订单...");
        // 执行业务逻辑...
    }
}
```


#### Bean 的生命周期

Bean 的生命周期宏观上只有四个大阶段：**实例化 $\rightarrow$ 属性赋值 $\rightarrow$ 初始化 $\rightarrow$ 销毁**。


请务必先死死记住这四个词，并且**千万不要把“实例化”和“初始化”搞混**：

1.  **实例化 (Instantiation)**：在堆内存中开辟空间（相当于 `new` 了一个空壳对象）。
2.  **属性赋值 (Populate Properties)**：也就是**依赖注入**，把这个 Bean 需要用到的其他 Bean 塞进去（比如处理 `@Autowired`）。
3.  **初始化 (Initialization)**：执行一些自定义的准备工作，让这个 Bean 达到真正能对外提供服务的状态。
4.  **销毁 (Destruction)**：容器关闭时，清理资源。

---

在上面这四个骨架中，Spring 提供了一大堆“回调接口”（钩子），让我们可以随时插手 Bean 的创建过程。

#####  实例化阶段、属性赋值阶段、循环依赖

**实例化阶段**
- 实例化阶段通过**执行构造器注入**完成Bean的实例化


**属性赋值阶段**
- 属性赋值阶段完成**字段注入和Setter注入**

---

**循环依赖**
- **字段注入/Setter注入可以解决循环依赖，而构造器注入直接报错死锁**
- 构造器注入阶段，Bean还没有创建出来，互相依赖的Bean找不到彼此，直接报错
- 在实例化阶段刚结束、对象刚刚被 new 出来时，Spring 就会立马将这个半成品暴露到**三级缓存**中。因此，当进入属性赋值阶段时，即便发生循环依赖，字段/Setter 注入也能从缓存中拿到彼此的引用，从而化解死锁。
- 在最严苛的架构规范中，根本不允许出现强循环依赖，出现了就打回去重构代码。这也是为什么 Spring Boot 2.6.0 之后，**默认直接禁用了所有的循环依赖**（哪怕是字段注入也会报错），就是为了逼迫开发者去写出更优雅的架构。


**强循环依赖本身就是设计缺陷**
-  **面向对象最佳实践告诉我们**：强依赖必须用构造器注入，且应当声明为 `final` 以保证不可变性。
-  **Spring 框架底层限制告诉我们**：构造器注入无法解决循环依赖，直接报错。
- 如果 A 和 B 互相作为绝对必需的强依赖，这意味着 A 离开 B 活不了，B 离开 A 也活不了。在软件工程里，这通常意味着**它们违反了单一职责原则（SRP），它们本质上应该是一个整体，或者纠缠了不该纠缠的业务。**
- 从设计上根本解决强循环依赖
  1. **提取第三者（引入中介者模式）**：
     假设 `OrderService`（订单）和 `PayService`（支付）互相强依赖。
     * **破局**：把互相调用的那部分公共逻辑抽离出来，新建一个 `OrderPayFacade`（门面层）或者 `TransactionService`（C）。让 A 和 B 都去强依赖 C，或者让 C 强依赖 A 和 B。循环依赖瞬间瓦解。
  2. **事件驱动（解耦利器）**：
     如果 `UserService` 注册完成后必须调用 `EmailService` 发邮件，而 `EmailService` 又需要强依赖 `UserService` 查数据。
     * **破局**：使用 Spring 的 `ApplicationEventPublisher`。`UserService` 注册完只管发一个“用户已注册事件”，`EmailService` 监听这个事件去发邮件。彻底切断双向强依赖。
- 撤销构造器注入，在两个类里老老实实写上 `@Autowired` 字段注入，让三级缓存解决循环依赖，放弃 `final` 关键字和不可变性，存在被意外修改的风险，且脱离 Spring 容器无法进行单元测试
- 引入代理魔法：
  - 如果老项目无法大重构，但又必须坚持构造器注入和 final 关键字，可以在构造参数上使用 `@Lazy` 注解。
  - Spring 会在实例化阶段注入一个“空壳代理对象（Proxy）”来欺骗当前 Bean，从而打破第一阶段的死锁。直到真正在代码里调用该依赖的方法时，代理对象才会去容器里获取真实的 Bean。

---

在实例化阶段和属性赋值阶段中，业务开发场景下**几乎绝对不会去实现和调用任何自定义的钩子函数**，只有需要开发底层中间件（比如自研的 RPC 框架、自研的分布式配置中心、动态数据源切换组件）时，才会用到这两个阶段的钩子。


---





#####  初始化阶段

###### Aware接口回调

在 Spring 的世界里，我们自己写的 `OrderService`、`UserService` 默认都是普通的 Java 对象（POJO）。它们就像一个个被蒙着眼睛干活的工人，**它们根本不知道自己是生活在 Spring 容器里的**。

这种“无知”在绝大多数情况下是好事（解耦），但有时候，某个 Bean 突然需要用到 Spring 框架底层的功能了，比如：
* 想要获取当前的运行环境配置（`Environment`）
* 想要动态去拉取其他的 Bean（`ApplicationContext`）
* 想要知道自己在 Spring 里的 Bean 名字是什么（`BeanName`）

这时候，Spring 怎么把这些底层资源交给你？**通过 `Aware` 接口。**
- `Aware` 翻译过来是“察觉的、意识到的”。只要你的类实现了一个特定的 `Aware` 接口，Spring 就会在这个阶段主动调用你，把底层资源当做参数塞给你，让你“觉醒”。
- 它发生的时机极其精准：**紧紧跟在“属性赋值阶段（DI 完成）”之后，在其他任何复杂的初始化逻辑开始之前。**
- 也就是：此时这个 Bean 已经有了完整的业务依赖（你写的 `@Autowired` 已经塞满了），但尚未执行任何你自定义的启动逻辑（`@PostConstruct` 还没执行）。Spring 赶紧趁这个空档，把“框架级”的依赖发给你。
- Spring 源码里有几十个 `Aware` 接口，但在业务开发中最常用Aware 接口有三个

---

`ApplicationContextAware`
- 这是日常开发中出场率最高的 Aware 接口。
* **作用**：让 Bean 获取到当前的 Spring 整体容器对象（`ApplicationContext`）。
* **高频场景**：我们在上一轮讨论中提到的“解决策略模式动态获取 Bean”的问题。很多项目里都有一个叫 `SpringContextUtil` 的工具类，就是靠它实现的。

**实战代码：**
```java
@Component
public class SpringContextHolder implements ApplicationContextAware {
    // 静态变量保存上下文
    private static ApplicationContext applicationContext;

    // Spring 在初始化第一步，会主动调用这个方法，把容器塞进来
    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        SpringContextHolder.applicationContext = context;
    }

    // 业务侧全局随便调
    public static <T> T getBean(Class<T> clazz) {
        return applicationContext.getBean(clazz);
    }
}
```

---

配置小能手：`EnvironmentAware`
* **作用**：让 Bean 获取到 `Environment` 对象，里面包含了 `application.yml` 里的所有配置、系统环境变量、JVM 参数等。
* **高频场景**：当你不想用 `@Value` 写死在字段上，而是想在代码逻辑里**动态**读取某个配置项，或者判断当前处于 `dev` 还是 `prod` 环境时。

**实战代码：**
```java
@Component
public class EnvChecker implements EnvironmentAware {
    
    private Environment env;

    @Override
    public void setEnvironment(Environment environment) {
        this.env = environment;
    }

    public void printCurrentEnv() {
        // 动态获取环境标识和自定义配置
        String[] activeProfiles = env.getActiveProfiles();
        String myUrl = env.getProperty("custom.api.url");
    }
}
```

---

偶尔一用：`BeanNameAware`
* **作用**：告诉这个 Bean，你在 Spring 容器里叫什么名字（默认是类名首字母小写）。
* **场景**：比较冷门。一般用于在记录复杂日志、或者注册某些底层组件时，需要用到当前 Bean 的唯一标识。

---



###### BeanPostProcessor 的前置处理

如果说前面的阶段是 Spring 在给你搭架子，那么从这一步开始，Spring 展现出了它无与伦比的**扩展性**

在讲“前置处理”之前，必须先隆重介绍 `BeanPostProcessor`。
它不同于你写的普通业务 Bean，它是 Spring 容器里的**质检员或安检机**。

* **普通 Bean 的视角**：“我”只关心我自己的生死和逻辑。
* **BPP 的视角**：它是一个拦截器。Spring 容器里**所有**的 Bean 在初始化的前后，都必须从 BPP 这个安检机里过一遍。

BPP 接口只有两个方法，简直对称得完美：
1.  **`postProcessBeforeInitialization` (前置处理 —— 我们现在要讲的)**
2.  `postProcessAfterInitialization` (后置处理 —— 第四步要讲的 AOP 诞生地)

---

**前置处理 `postProcessBeforeInitialization` 到底干了啥？**

- 它发生在 Aware 接口回调结束之后，在你自定义的 `@PostConstruct` 初始化方法执行**之前**。
-  `Object postProcessBeforeInitialization(Object bean, String beanName)`
- 它的超能力：拦截与改造，请注意它的返回值是 `Object`！这意味着：
     * **你可以原封不动地返回原来的 Bean**（安检通过，放行）。
     * **你可以对 Bean 的属性进行疯狂修改**（安检员发现你没穿工服，强行给你套上一件）。
     * **你甚至可以返回一个完全不同的新对象**（狸猫换太子，或者包一层代理）。

---


**`@PostConstruct` 是怎么执行的？**
- 你可能会好奇，为什么我们加个 `@PostConstruct` 注解，方法就会自动执行？其实它根本不是 Java 原生的魔法，而是**正是由这个“前置处理钩子”实现的！**

- Spring 内部有一个自带的质检员，叫 `InitDestroyAnnotationBeanPostProcessor`。
- 在这个前置处理阶段，这个质检员会拿着放大镜看你这个 Bean：
> *“哎？你这个 Bean 里面有个方法头上戴着 `@PostConstruct` 的帽子？好，我立刻用反射帮你执行这个方法！”*

- **破案了！** 初始化阶段的第二步（BPP 前置处理），恰恰是驱动第三步（自定义初始化 `@PostConstruct`）的幕后推手！

---

**业务开发与中间件实战：我们能用它干嘛？**

- 还是那句话：普通的纯业务 CRUD 开发，**几乎用不到**去自定义 BPP。
但如果你在公司里负责写**公共组件、基础架构、或者是造轮子**，这个钩子就是你的神兵利器。

**场景：强制代码规范检查（大厂常有）**
- 假设你们架构组出了一个铁律：“所有以 `Service` 结尾的类，里面必须至少有一个方法，否则不允许启动！”（防止有人建空类当占位符）。

- 你可以自己写一个“质检员”：

```java
@Component
public class ServiceRuleCheckPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // 1. 只拦截名字以 Service 结尾的 Bean
        if (beanName.endsWith("Service")) {
            // 2. 利用反射获取类里所有的方法
            Method[] methods = bean.getClass().getDeclaredMethods();
            
            // 3. 执行质检逻辑
            if (methods.length == 0) {
                // 如果发现是个空类，当场抛出异常，让 Spring 启动失败！
                throw new RuntimeException("架构组严厉警告：类 " + beanName + " 是个空类，不允许上线！");
            }
        }
        // 质检通过，原样放行
        return bean;
    }
}
```
- **这就是“非侵入式”设计的巅峰。** 业务线的程序员完全不知道这个质检员的存在，他照常写他的代码，但只要违规，项目启动第一阶段就会被拦截报错。

---

**总结**
- 初始化阶段的第二步（`BeanPostProcessor` 的前置处理），是 Spring 给所有 Bean 设置的**统一安检站**。它不仅是实现各种自定义注解（如 `@PostConstruct`）的底层引擎，更是架构师用来全局拦截、校验和修改 Bean 的绝佳位置。

---
###### 自定义初始化方法

自定义初始化方法和构造器
- 职责分离：在设计上，构造器应该专注于实例化而不是初始化，任何初始化逻辑都不该放在构造器中
- 如果构造器使用了还没有注入的依赖，例如字段注入的依赖，直接报错

Spring 给我们提供了三种方式来写自定义初始化逻辑。如果一个类同时用了这三种方式，Spring 会按照极其严格的顺序来执行它们：

**第一名：`@PostConstruct` 注解**
* **出场率**：99%。日常业务开发中，只要你需要初始化，用它就对了。
* **特点**：极其简单，在普通方法头上加个注解就行。
* **底层秘密**：我们在上一步学过，它其实是被 `InitDestroyAnnotationBeanPostProcessor` 这个“安检员”在前置处理阶段用反射抠出来执行的。
* **实战代码**：
    ```java
    @Service
    public class DictCache {
        @Autowired
        private DictMapper dictMapper;

        @PostConstruct // 依赖注入完成后，自动执行此方法
        public void initData() {
            // 此时 dictMapper 已经有值了，安全！
            List<Dict> list = dictMapper.selectAll();
            System.out.println("本地缓存预热完毕！");
        }
    }
    ```

第二名（老派做法）：`InitializingBean` 接口
* **出场率**：业务代码中很少用，但在 **Spring 框架自己的底层源码**中满地都是。
* **特点**：需要让你的类实现 `InitializingBean` 接口，并重写 `afterPropertiesSet()` 方法。名字翻译过来非常直白：**“在属性设置（属性赋值）之后”**。
* **为什么业务开发不爱用了？** 因为它是一种**强侵入式**的设计。你的业务类一旦实现了这个接口，就被死死地绑在 Spring 框架的战车上了，脱离了 Spring 就无法运行。
* **实战代码**：
    ```java
    @Service
    public class MyService implements InitializingBean {
        @Override
        public void afterPropertiesSet() throws Exception {
            System.out.println("执行 InitializingBean 的初始化逻辑...");
        }
    }
    ```

第三名（借鸡生蛋）：`init-method` 属性
* **出场率**：经常用于**整合第三方组件**。
* **特点**：不修改 Bean 类本身的任何代码，而是在外部配置（XML 或 `@Bean` 注解）中指定哪个方法是初始化方法。
* **实战场景**：假设你引入了一个第三方的 Redis 客户端包，里面的类是只读的，你没法往它源码里加 `@PostConstruct`，也没法让它实现接口。这时候只能在外部指定：
    ```java
    @Configuration
    public class RedisConfig {
        // 告诉 Spring：把第三方类里的 "startConnection" 方法当做初始化方法来执行
        @Bean(initMethod = "startConnection")
        public ThirdPartyRedisClient redisClient() {
            return new ThirdPartyRedisClient();
        }
    }
    ```

---


如果一个无聊的程序员，在一个类里把这三种方式全写了，它们的执行顺序是什么？

**标准答案：**
1.  先执行 `@PostConstruct` 注解修饰的方法。
2.  再执行 `InitializingBean` 接口的 `afterPropertiesSet()` 方法。
3.  最后执行通过 `@Bean(initMethod = "...")` 指定的方法。

**为什么是这个顺序？**
稍微回想一下我们的生命周期流水线就能理解：`@PostConstruct` 是在第二步（BPP 的前置处理）里被顺手执行掉的；而后面两个，是 Spring 在彻底度过前置处理阶段后，才去挨个检查并调用的。

---

可以在一个类里给多个方法加上了 @PostConstruct，项目不仅能正常启动，而且这几个方法也都会被执行。能这么做，但不建议这么做
1. **执行顺序完全不可控**
   - Spring 底层是通过 Java 反射（class.getDeclaredMethods()）来获取类中所有方法的，然后挨个检查谁头上戴了 @PostConstruct 帽子。
   - 但是，Java 官方的 JDK 文档明确指出：反射返回的方法数组，其顺序是没有任何保证的！ 不同的 JDK 版本、不同的操作系统，甚至同一台机器上不同的启动批次，获取到的方法顺序都可能不一样。
2. **违反了 JSR-250 官方规范**
   - **在给定的一个类中，只能有一个方法被@PostConstruct注解修饰**
   - 虽然 Spring 底层（InitDestroyAnnotationBeanPostProcessor）做得比较宽松，没有严格按照规范去拦截报错，但这并不代表我们应该去钻这个空子


**唯一合理的“多 @PostConstruct”场景：父子类继承**
- 在整个软件工程中，只有一种情况允许一个 Bean 经历多次 @PostConstruct，那就是继承。
- 如果你有一个父类和一个子类，它们各自都有自己的 @PostConstruct 方法，这是完全合法且被 Spring 完美支持的
- 在类的继承体系中，Spring 会严格保证先调用父类的 @PostConstruct，再调用子类的 @PostConstruct。这在封装底层通用组件时非常有用。
```JAVA
public class BaseService {
    @PostConstruct
    public void baseInit() {
        System.out.println("1. 先执行父类的通用初始化...");
    }
}

@Service
public class UserServiceImpl extends BaseService {
    @PostConstruct
    public void childInit() {
        System.out.println("2. 再执行子类的专属初始化...");
    }
}
```

---

**优雅地编排多个初始化逻辑**
- 如果你真的有很多杂七杂八的逻辑需要初始化，千万别打多个 `@PostConstruct`，正确的做法是：化繁为简，统一入口。
- 只保留一个 `@PostConstruct` 方法作为总导演，在里面显式地、按确定顺序调用其他普通的私有方法：

```Java
@Service
public class OrderService {

    // 唯一暴露给 Spring 的生命周期入口
    @PostConstruct
    public void init() {
        // 顺序由你自己用代码写死，绝对安全！
        step1_loadCache();
        step2_checkConfig();
        step3_startTask();
    }

    // 下面都是普通的私有方法，供 init() 调度
    private void step1_loadCache() { ... }
    private void step2_checkConfig() { ... }
    private void step3_startTask() { ... }
}
```

###### BeanPostProcessor 的后置处理




`postProcessAfterInitialization`是 `BeanPostProcessor` 接口里的第二个方法。它的执行时机是：**Bean 的所有初始化工作彻底结束之后。**

- 在这个方法里，诞生了 Spring 框架的两大核心支柱之一：**AOP（面向切面编程）的动态代理。**

**AOP的核心工作流（狸猫换太子）：**
1. **审查**：专门处理 AOP 的质检员（比如 `AnnotationAwareAspectJAutoProxyCreator`）拿着放大镜来看这个刚刚完全初始化好的 Bean。
2. **匹配**：它去翻看你写的切面规则（Aspect），或者看看这个 Bean 的方法上有没有挂着 `@Transactional`（事务）、`@Async`（异步）、`@Cacheable`（缓存）等注解。
3. **包装（核心操作）**：
   * 如果没有任何匹配，安检员挥挥手：“没你事了，过去吧。”原样返回真实的 Bean。
   * **如果匹配上了！** 安检员会当场利用 JDK 动态代理 或 CGLIB 技术，动态生成一个**空壳代理对象（Proxy）**。把原来的真实 Bean 像套娃一样藏在代理对象肚子里。
4. **入池**：最终，这个方法将**代理对象**返回。Spring 拿着这个返回的代理对象，喜滋滋地扔进了单例池（IoC 容器）中。

---

**单例池里的“冒牌货”**
- 当你为一个类配置了事务（`@Transactional`）或者切面拦截后，**在 Spring 的大池子里（IoC 容器里），根本就不存在你原来写的那个类的真正实例！**。池子里躺着的，是一个长得跟你的类一模一样、但满脑子都是拦截逻辑的**代理替身**。
- 当 Controller 层 `@Autowired` 注入这个 Service 时，Spring 塞给 Controller 的也是这个替身。
当 Controller 调用 `service.doSomething()` 时，实际上是替身先接到了请求，替身去开启数据库事务，然后再把请求转交给肚子里的真实 Service 执行，真实代码执行完，替身再负责提交或回滚事务。

---

**同类方法调用的“AOP 失效”惨案**

- 理解了后置处理产生的“代理替身”，你就能瞬间秒杀 Spring 开发中最臭名昭著的一个大坑：**同类方法内部调用，事务/切面为什么会失效？**

**事故现场：**
```java
@Service
public class OrderService {

    // 方法 A 没有事务
    public void createOrder() {
        System.out.println("订单创建准备...");
        // 🚨 致命报错预警：这里直接调用同类内部的方法 B
        this.pay(); 
    }

    // 方法 B 有事务
    @Transactional
    public void pay() {
        System.out.println("扣减余额，此操作受事务保护...");
        // 数据库操作...
    }
}
```
- 如果你在外部调用 `orderService.createOrder()`，当代码执行到 `this.pay()` 时，哪怕 `pay` 方法上加了 `@Transactional`，**事务也绝对不会生效！发生异常也不会回滚！**

**破案分析：**
1. 外部 Controller 调用 `createOrder()` 时，其实是调用的**代理替身**。替身看了一眼 `createOrder`，发现没事务注解，于是直接把任务扔给了肚子里的**真实对象**。
2. 真实对象开始执行 `createOrder` 的代码逻辑。
3. 当走到 `this.pay()` 时，关键来了！这里的 `this` 是谁？**是真实对象自己！**
4. 真实对象直接调用了自己的 `pay()` 方法，**完美绕过了外面的代理替身！** 既然没有经过替身，替身怎么可能帮你开启事务呢？

**如何优雅地解决？（大厂规范）**
- 既然问题出在绕过了代理，那我们**强行把代理对象抓过来调用**就行了。最优雅的做法是，自己注入自己（Spring 支持这种操作，三级缓存能搞定它）：

```java
@Service
public class OrderService {

    // ✅ 自己注入自己的代理对象
    @Autowired
    @Lazy // 最好加个 @Lazy 防止一些极端的循环依赖报错
    private OrderService selfProxy;

    public void createOrder() {
        // ✅ 使用代理对象去调用 pay()，事务完美生效！
        selfProxy.pay(); 
    }

    @Transactional
    public void pay() { ... }
}
```

---





#####  使用阶段、销毁阶段


严格来说，**使用阶段没有任何 Spring 生命周期“钩子”**。因为钩子是用来干预创建和销毁的，而在使用阶段，主角是你写的业务代码。但在这个阶段，有一个**全网最高频、引发无数生产事故的绝对红线**：
- 线程安全问题：绝对不要在单例 Bean里定义带有状态的成员变量，如果非要全局共享数据，必须使用**并发安全的集合**（如 `ConcurrentHashMap`）或者**原子类**（如 `AtomicInteger`）。
- 用户的状态数据，要么扔进 `ThreadLocal` 里，要么通过**方法参数**老老实实往下传（局部变量是线程安全的）。

---


当服务器准备关机重启，或者应用下线时，Spring 容器会发出销毁指令。**`@PreDestroy` 注解**清理那些 **Spring 管不到的、或者你自己手动开辟的底层资源**

**高频业务实战**：
  * 你自己在 Service 里 `new` 了一个业务专属的线程池，关机前必须把它优雅关闭，否则可能导致正在跑的任务数据丢失。
  * 你自己开了一个对外的 WebSocket 连接、或者某个底层硬件的 Socket 通讯，关机前需要发个“再见”的报文给对面。
  * 你在内存里积攒了一批日志或埋点数据，准备每 10 秒批量刷入数据库，突然要关机了，你必须在销毁前把最后一点尾巴数据刷进去。

**实战代码：**
```java
@Service
public class LogAsyncService {

    // 自己开的线程池，Spring 不会自动帮你关！
    private ExecutorService myThreadPool = Executors.newFixedThreadPool(5);

    public void asyncLog() {
        myThreadPool.submit(() -> System.out.println("异步记录日志..."));
    }

    // 离职交接钩子
    @PreDestroy
    public void cleanUp() {
        System.out.println("收到关机指令，正在优雅关闭日志线程池...");
        myThreadPool.shutdown(); // 拒绝新任务，等老任务跑完
        try {
            if (!myThreadPool.awaitTermination(5, TimeUnit.SECONDS)) {
                myThreadPool.shutdownNow(); // 5秒还没跑完，强杀！
            }
        } catch (InterruptedException e) {
            myThreadPool.shutdownNow();
        }
        System.out.println("线程池关闭完毕，安心离职。");
    }
}
```

 **整合第三方组件的收尾：`@Bean(destroyMethod = "...")`**
- 这个用法和初始化阶段的 `init-method` 是一对。当你用 `@Bean` 把第三方的类塞进 Spring 时，可以用它指定第三方类里自带的关闭方法。

**实战代码：**
```java
@Configuration
public class RedisConfig {
    
    // 假设引入了一个第三方的缓存客户端组件
    // 告诉 Spring：销毁这个 Bean 的时候，去调它里面的 "closeConnection" 方法
    @Bean(destroyMethod = "closeConnection")
    public ThirdPartyCacheClient cacheClient() {
        return new ThirdPartyCacheClient();
    }
}
```

---

##### 销毁阶段的坑
**销毁阶段的最大坑：“杀鸡取卵”的 `kill -9`**
- 你要特别注意：`@PreDestroy` **并不是绝对可靠的**！

* 如果运维人员在服务器上使用 `kill -15`（优雅终止）关闭进程，或者正常点击停止按钮，Spring 会捕获到关闭信号，`@PreDestroy` **完美执行**。
* 但如果遇到服务器突然断电，或者运维不耐烦了直接使用 **`kill -9`（强制物理击杀）**，整个 JVM 瞬间灰飞烟灭，Spring 根本来不及反应，**`@PreDestroy` 绝对不会被执行！**

**业务建议：** **永远不要把绝对不能丢失的、涉及到钱的核心业务数据持久化动作放在 `@PreDestroy` 里。它只能用来做“资源释放”和“尽力而为”的收尾。**

---


**`Prototype`原型 / 多例作用域**
- 在类上加 `@Scope("prototype")` 或者 `@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)`
- 每次依赖注入时，或者通过 `applicationContext.getBean()` 获取它时，Spring 都会**立刻走一遍前面学过的实例化、属性赋值、初始化流程，给你 `new` 一个全新的对象**。
- 对于 Prototype 作用域的 Bean，**Spring 是管生不管杀的！**
- Spring执行完 `@PostConstruct` 交给你之后，不会把这个对象放进单例池，而是**彻底撒手不管了**
- 当你的业务代码用完这个对象，并且没有其他引用指向它时，它会被 **JVM 的垃圾回收器（GC）** 默默回收掉。**它绝对不会执行 `@PreDestroy` 销毁方法！**
- 给 Prototype 作用域的 Bean 加上 @PreDestroy 方法，不报错，也不执行，写了白写
- **业务场景**：当你需要一个包含状态（比如存了特定用户的数据），且绝对不能被其他线程共享的工具类对象时，会声明为 Prototype。但现在很多人更倾向于直接自己 `new`，所以出场率在逐渐下降。


---

**Web 专属作用域**
-  `Request` 作用域寿命大概只有几百毫秒。当客户端（浏览器）发起一次 HTTP 请求，Tomcat 接收到请求的那一刻，Spring 悄悄实例化这个 Bean。等这几个微服务的链路走完，HTTP 响应（Response）打回给前端时，**这个 Bean 立刻被 Spring 销毁**。
-  Bean被 Spring 销毁时会 执行销毁钩子，Spring 会正常调用它的 `@PreDestroy`。




---




#### Bean 的高级特性
延迟初始化 (Lazy Init)： 优缺点及使用场景。

自动装配规则： ByName, ByType 的深度解析。

条件装配 (@Conditional)： 如何根据环境决定是否创建某个 Bean。

第五阶段：核心注解对比与冲突处理
@Autowired vs @Resource： 来源、注入规则（按类型还是按名称）、对 Null 的处理。

@Primary 与 @Qualifier： 当容器中有多个同类型 Bean 时，如何优雅地“选妃”。

