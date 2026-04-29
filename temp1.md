#### 上下文透传与全链路灰度发布

##### 网关层：流量染色与打标

一切的起点在于网关。外部用户的请求（携带 Token 或 UserID）到达 Spring Cloud Gateway。
* **业务逻辑：** 网关会查询规则中心（如 Nacos 配置、Redis 或 Apollo），判断当前用户是否命中灰度规则（如：尾号为 8 的用户）。
* **底层动作：** 如果命中，网关会在 HTTP Request 中强行塞入一个特定的 Header。
  * 例如：`X-Gray-Version: v2`。
* **结果：** 此时，原本普通的流量就被“染色”了，变成了一股带有 `v2` 标记的灰度流量，发往微服务 A。


##### 微服务 A 接收端：拦截提取与 ThreadLocal 存储
当带有 `X-Gray-Version: v2` 的请求到达微服务 A（如订单服务）的 Tomcat 时，业务层需要把这个标签存起来，供后续调用微服务 B（如库存服务）时使用。
* **痛点：** Spring MVC 的 `HttpServletRequest` 生命周期极短，且后续发起 Feign 调用时往往在不同的方法深处，总不能把 Header 作为参数一层层往下传（侵入性太强）。
* **企业级解法：** 使用 **Spring MVC 拦截器（`HandlerInterceptor`） + `ThreadLocal`**。
  1. 拦截器在请求到达 Controller 之前，提取 Header 中的 `X-Gray-Version`。
  2. 将其放入当前线程的 `ThreadLocal<String>` 中。
* **⚠️ 极度致命的生产踩坑：** 绝对不能使用 JDK 原生的 `ThreadLocal`！因为微服务中常有异步操作（`@Async`）或熔断线程池。必须使用阿里的 **`TransmittableThreadLocal` (TTL)**，它能保证在父子线程、线程池切换时，灰度标签绝对不丢。


##### 微服务 A 发起端：OpenFeign 拦截器透传
微服务 A 的业务逻辑执行完毕，准备通过 OpenFeign 调用微服务 B。此时发出的却是一个**全新**的 HTTP 请求，原本的 Header 已经没了。
* **企业级解法：** 实现 OpenFeign 的 `RequestInterceptor`（请求拦截器）。
* **底层动作：** 1. Feign 在构建真实 HTTP 请求前，会执行该拦截器。
  2. 拦截器从 TTL（`ThreadLocal`）中拿出刚才存进去的 `X-Gray-Version: v2`。
  3. 将其重新塞入到 Feign 即将发出的 HTTP Header 中。
* **结果：** 灰度标签成功跨越了进程边界，向下一个微服务传递。


##### Spring Cloud LoadBalancer：动态路由与元数据匹配
现在，Feign 准备好了一个带有 `v2` 标签的请求，接下来就要交给 LoadBalancer（SCL）去 Nacos 里挑机器了。
* **Nacos 侧的准备：** 在微服务 B 的 v2 版本节点启动时，在其 Nacos 配置中打上元数据标签：`spring.cloud.nacos.discovery.metadata.version=v2`。
* **SCL 侧的定制路由逻辑：**
  1. 开发者需要实现 SCL 的 `ReactorServiceInstanceLoadBalancer` 接口，替换掉默认的轮询算法。
  2. 在 `choose()` 方法中，通过 SCL 提供的 `RequestDataContext` 提取出本次请求的 Header（也就是 `X-Gray-Version: v2`）。
  3. SCL 向 Nacos 索要微服务 B 的所有实例列表（比如一共 10 台机器，2 台 v2，8 台 v1）。
  4. **核心匹配过滤：** 遍历这 10 台机器的 `metadata`。发现 Header 是 v2，就强行把那 2 台 `metadata` 里带有 `version=v2` 的机器挑出来。
  5. 在这两台 v2 机器中进行随机或轮询调用。

兜底机制（Fallback）设计建议
在企业级路由算法中，必须写兜底逻辑：**如果根据 v2 标签去找机器，发现 v2 的机器全部宕机了怎么办？**
标准做法是：记录一条 Warning 日志，然后**降级将流量打回 v1 的常态节点**，宁可执行老代码，也不能让业务报错资损。

---





