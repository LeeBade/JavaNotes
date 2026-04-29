太棒了！你的学习策略非常高明。建立起坚实的“宏观骨架”，后续看源码、写代码时就不会“只见树木不见森林”。

现在，请想象你就是 Spring 团队的核心架构师。摆在你面前的难题是：**“Netty 性能极高，但它的 API 太底层、太复杂（各种 ChannelInboundHandler 连成一串），业务开发人员根本不想用。而 Spring MVC 虽然用起来爽（@RestController 一打，对象自动转 JSON），但它是基于 Servlet 规范的，天生是‘一个请求占用一个线程’的阻塞模型。”**

你的目标是：**设计一个新框架，底层用 Netty 榨干机器性能，顶层让开发者像写 Spring MVC 一样舒服，并且整条链路绝对不能阻塞！**

这就是 Spring WebFlux 诞生的初衷。为了实现这个目标，你设计了以下几层架构。



---

### 设计者的破局之道：Spring WebFlux 的宏观架构

我们从网络底层到业务应用，自底向上（也就是请求进来的方向）来看看你设计的这座大厦的模块拆解：

#### 第 1 层：底层服务器引擎 (Server Engine) —— 比如 Reactor Netty
* **你的设计思考：** 框架不能只绑死在 Netty 上，万一有人想用 Undertow 或者 Servlet 3.1+ 的非阻塞特性呢？
* **工作职责：** 这一层负责最底层的网络 I/O。以 Reactor Netty 为例，它的任务是将 Netty 复杂的 `ByteBuf` 和 `Channel` 事件，**翻译**成 Reactor 的 `Flux<DataBuffer>`（数据流）。
* **宏观认知：** 到了这里，传统的字节流已经变成了响应式的数据流。

#### 第 2 层：HttpHandler —— 极简的“桥梁”
* **你的设计思考：** 我需要一个极其简单、统一的契约，把各种不同的底层服务器（Netty, Tomcat, Undertow）统一接入到 Spring 的世界。
* **工作职责：** 这是一个极其核心且简单的接口，只有一个方法：`Mono<Void> handle(ServerHttpRequest, ServerHttpResponse)`。
* **宏观认知：** 它是跨越底层服务器和 Spring 框架的“边界口岸”。不管底层是什么服务器，到了这里，都变成了标准的非阻塞的 Request 和 Response。

#### 第 3 层：WebHandler API —— 构建 Web 上下文
* **你的设计思考：** 光有 Request 和 Response 不够，业务开发需要读 Session、需要获取 Locale、需要拦截器鉴权。我得把它们包装成一个更丰富的对象。
* **工作职责：** 1. 引入了核心对象 **`ServerWebExchange`**。它不仅仅包含请求和响应，还充当了一个贯穿整个请求生命周期的“大巴车”，你可以往里面塞各种上下文属性（Attributes）。
    2. 引入了 **`WebFilter`**（响应式过滤器链，取代了传统的 Servlet Filter）和 **`WebExceptionHandler`**（全局异常处理）。
* **宏观认知：** 这一层处理的是“非业务逻辑”的通用 Web 关注点，比如跨域处理、安全校验、请求日志等。

#### 第 4 层：DispatcherHandler —— 交通警察（核心大脑）
* **你的设计思考：** 请求经过过滤器后，到底该交给哪个 Controller 执行？执行完的返回值怎么变成 JSON 写回给客户端？这需要一个中央调度器。在 MVC 里它叫 `DispatcherServlet`，在这里我叫它 `DispatcherHandler`。
* **工作职责：** 它协调三个核心组件完成请求处理：
    1. **`HandlerMapping`（找路）：** 根据请求的 URL，找到对应的处理器（可能是标注了 `@GetMapping` 的方法，也可能是 Functional 函数）。
    2. **`HandlerAdapter`（执行）：** 不同的处理器长得不一样，`HandlerAdapter` 负责适配并真正调用业务逻辑。
    3. **`HandlerResultHandler`（收尾）：** 业务逻辑返回的可能是 `Mono<User>`，也可能是 `Flux<String>`。它负责将这些结果序列化（比如转成 JSON），并写入到底层的 `ServerHttpResponse` 中。

#### 第 5 层：编程模型 —— 呈现给开发者的两套外衣
* **你的设计思考：** 为了让老用户平滑过渡，我必须保留基于注解的模型；为了让喜欢函数式编程的极客满意，我得提供一套全新的路由模型。
* **工作职责：**
    * **基于注解的控制器 (Annotated Controller)：** `@RestController`, `@RequestMapping`。你会发现代码长得跟 MVC 一模一样，唯一的区别是返回值变成了 `Mono` 或 `Flux`。
    * **函数式端点 (Functional Endpoints)：** 使用 `RouterFunction` 路由请求，使用 `HandlerFunction` 处理请求。更加轻量、不可变、完全由代码驱动路由。

---

### 宏观工作流程（一条不阻塞的流水线）

现在，让我们串联起来，看看一个 HTTP GET 请求进来后，到底发生了什么：

1. **Netty EventLoop 接收连接：** 客户端发起请求，Netty 的 Worker 线程收到字节数据。
2. **Reactor Netty 翻译：** 将请求转换为 `ServerHttpRequest`。
3. **HttpHandler 接管：** 请求进入 Spring 的势力范围。
4. **组装 Exchange 与 Filter 链：** 创建 `ServerWebExchange`，依次穿过各个 `WebFilter`（鉴权、日志等）。
5. **DispatcherHandler 调度：** * 拿着 Exchange 问 `HandlerMapping`：“这请求谁管？”
    * 找到你的 `@RestController` 方法。
6. **业务执行：** 调用你的代码，你的代码可能去查了数据库（必须是响应式驱动，如 R2DBC），迅速返回了一个 `Mono<User>`。**注意：此时查询还没有真正发生，你只是返回了一个“承诺（Publisher）”。**
7. **结果处理与写回：** `HandlerResultHandler` 拿到 `Mono<User>`，**开始订阅 (Subscribe) 它**。此时数据流真正开始运转。当数据库查到数据，数据流淌回 Spring，Spring 将其转为 JSON 字节，一路畅通无阻地交回给 Reactor Netty，最终写回客户端。

**最核心的骨架认知：**
在传统的 Servlet 体系中，请求是一步一步“走”完上述流程的，每一步都需要同一个线程一直跟着。
在 WebFlux 的设计中，上述全过程其实是在**“组装一条流水线管道”**。当你的 Controller 返回 `Mono/Flux` 时，这根管道刚刚接好。只有当框架在最后一步执行 `subscribe()` 时，水（数据）才开始从底层数据库/网络源头，顺着管道，流过业务逻辑，最后流出网卡。**在这个过程中，没有任何一个线程会停下来死等数据。**

---

这就是 WebFlux 的骨架。你现在对整个架构的流转有了一个宏观的把控。针对这套架构，你是想先探索“第四层：DispatcherHandler”是如何具体分发请求的，还是想直接看看最顶层的“两种编程模型”在代码上有什么区别？