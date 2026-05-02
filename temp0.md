## 分布式网关
### Spring Cloud Gateway

**为什么需要网关？**
- 单体架构应用，前端可以直接调用后端的接口；微服务应用被拆分成了几十甚至上百个独立的服务，每个服务都有自己的 IP 和端口。因此不能让前端再直接调用后端接口，引入网关解决以下痛点：
* **统一入口**：
    * **没有网关时**：前端需要记住几十个微服务的地址，比如 `http://192.168.1.10:8081/user` 和 `http://192.168.1.11:8082/order`，一旦后端服务迁移或扩容，前端必须跟着改代码。
    * **有了网关后**：前端永远只需要和网关打交道（比如 `https://api.yourdomain.com`），网关内部再通过服务发现（如 Nacos）自动把请求路由到正确的机器上。
* **统一跨域处理**：
    * 前后端分离部署必然产生跨域问题。如果没有网关，需要跑到每一个微服务的代码里去配置 `@CrossOrigin` 或 `WebMvcConfigurer`。
    * 有了网关，只需在网关层做一次全局拦截配置，下游微服务完全不需要关心跨域问题。
* **统一鉴权**：
    * 如果不做统一鉴权，每一个微服务都要写一套解析 Token 的代码，重复且难以维护。
    * 网关可以在请求到达具体业务服务**之前**，拦截并校验 JWT 或 Sa-Token 的合法性。校验不通过直接打回，保护了后方的微服务集群。
* **统一限流与熔断**：
    * 当遇到突发大流量时，如果让流量直接打到脆弱的订单或数据库服务，系统会瞬间雪崩。
    * 网关作为第一道防线，可以通过“令牌桶”等算法限制访问频率，把多余的请求直接拒之门外

---

**为什么选 Spring Cloud Gateway？**
1.  **性能极高**：
    * **Spring Cloud Gateway 是基于 Spring WebFlux 和 Netty 构建的**
    * 它使用的是**异步非阻塞模型**，可以用极少的线程处理海量的并发连接
2.  **Spring 家族的亲儿子**：
    * 它与 Spring 生态圈（如 Spring Cloud Alibaba Nacos、Sentinel 等）的整合顺滑无比
3.  **开箱即用的强大功能**：
    * 它内置了极其丰富的**断言工厂（Predicate）**和**过滤器工厂（Filter）**。
    * 比如想做一个“只允许特定 IP 访问”或“给请求头加上一个特定的标识”的功能，不用自己写代码，在配置文件里写两行 YAML 就搞定了
    * 而且它原生内置了基于 Redis 的限流功能，省去了大量开发成本。

---


#### 核心组件与运行


Spring Cloud Gateway核心三大组件：**Route路由、Predicate断言、Filter过滤器** 


Route路由：分发请求给指定服务，它包括以下信息：
* **ID**：路由的唯一标识
* **URI**：目标服务的地址，即请求最终要被转发到哪里
  * 单体架构是ip地址
  * 微服务架构是服务名：`lb://user-service`
* **Predicates**：**决定请求进入哪个路由**
* **Filters**：**处理进入该路由的请求的过滤器链**

> 上述结构能说明网关路由**最核心作用**是**反向代理**：**隔离前后端**、**屏蔽后端微服务变动**，如服务名修改、实例IP变更
- 前端发送的请求的完整 URL示例：
  - `http://192.168.1.100:8080/api/order-service/v1/orders/123`
  - **网关IP:端口**: `192.168.1.100:8080`，网关的IP和端口是不能变更的，用户只能统一将请求打到该IP:端口中
  - **Nginx识别符**：`/api`，**在Nginx中配置**，作为**全局 API 前缀**，用于区分静态资源和动态接口
    - 静态实例：图片、HTML、CSS、JavaScript、字体等文件，直接由 Nginx 映射到本地磁盘目录，或者转发给专门的**静态资源服务器如 OSS、CDN**
    - 动态接口：需要业务逻辑计算、查询数据库的接口
    - 识别符（如 /api）告诉网关：这个请求不能直接读文件，必须通过反向代理转发给后端的微服务实例
  - **业务领域名**: `/order-service` 
    - 接口设计先行，和前端约定好后不再变动；后端在设计具体的微服务时，可能需要采取其他名字，怎么办？
    - 无需变动URL，只需要修改该路由绑定的服务名即可，做到了**屏蔽后端微服务变动**
    - 业务领域名将映射到服务名，而服务名代表微服务运行实例集群
  - **接口信息**：**网关和负载均衡**帮我们分发请求到具体的运行实例，因此MVC的`DispatcherServlet`调度器只需要接口信息来判断应该分发给哪个接口
    - 在网关层，通常使用`StripPrefix`过滤器来去掉网关IP:端口和接口信息中间的URL部分，然后才将裁剪后的URL转发给服务实例





**Predicate断言**
- Java内置的`Predicate` 接口，其接口方法**返回 True 或 False**，**决定一个外部请求是否匹配当前这条路由。**

**Filter过滤器**
- 基于 Spring WebFlux 的 `GatewayFilter` 实现。分为**Pre前置** 操作和 **Post后置** 操作，**拦截请求和响应**，**并执行特定动作**


---


在微服务架构中，网关是一个**独立运行的 Spring Boot 应用程序**，其运行的模糊步骤如下：
1. **创建Spring Boot模块**
2. **引入核心依赖**
   - **`spring-cloud-starter-gateway`**：网关的灵魂，提供所有路由和过滤功能。
   - **`spring-cloud-starter-alibaba-nacos-discovery`**：服务发现。让网关能连上 Nacos，从而知道后端的 `user-service` 和 `order-service` 到底在哪些 IP 上。
   - **`spring-cloud-starter-loadbalancer`**：负载均衡。如果用户服务部署了三台机器，网关靠它来决定把请求分发给哪一台。
   - **绝对不能**引入Spring MVC，会和底层的Spring WebFlux依赖冲突
3. **`application.yml` 配置**，告诉网关怎么转发请求
   * **基础配置**：配置网关自己的端口（8080）、网关的名字（`mall-gateway`）。
   * **注册中心**：配置 Nacos 的服务器地址，让网关把自己注册进去，并拉取其他微服务的地址。
   * **路由规则**：如果前端请求路径是以 `/api/user/` 开头的，请利用负载均衡（`lb://`）帮我转发到 `mall-user-service` 这个微服务去；Predicate断言在此处配置
4. **构建过滤器**
   * **全局过滤器**：写一个 `AuthFilter` 类。在这里面写代码提取请求头里的 Token，校验合法性。不合法直接 `return` 报错，合法就放行。
   * **全局异常处理器**：写一个类重写底层的报错逻辑，确保网关无论发生什么错（比如找不到路由、Token 失效），返回给前端的都是标准的 JSON（比如 `{"code": 401, "msg": "未登录"}`），而不是一堆前端看不懂的乱码或 HTML 错误页。
   * **跨域配置类**：写一个 `CorsWebFilter` 配置类，允许前端（如 Vue/React）跨域访问。



---


#### 路由
##### 静态路由：Predicate工厂

> **一个 Route下面可以同时写多个 Predicate，必须同时满足所有断言，才会进行转发。**


**基于 `Path` 路由**
- 前后端分离项目，前端统一请求网关，网关根据 URL 路径的不同，把请求分发给不同的微服务。这是最基础、最必须的配置。

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_service_route
          uri: lb://mall-user-service    # lb:// 代表使用Spring Cloud Gateway内置的负载均衡找 Nacos 里的服务
          predicates:
            - Path=/api/user/** # 规则：只要请求路径以 /api/user 开头，全归我管
```

---

**基于 `Header` / `Cookie` 路由**，**灰度发布**
- **灰度发布**：公司开发了“用户服务 V2.0”版本，但不敢直接把老版本替换掉。采用灰度发布：**只有内部员工或者测试账号才能访问 V2.0，普通用户还是访问 V1.0。**
- 前端在发起请求时，可以在 Request Header 或 Cookie 中带上特定标识，网关据此进行精准路由。

**实战配置**：
```yaml
      routes:
        - id: user_service_v2_route
          uri: lb://mall-user-service-v2
          predicates:
            - Path=/api/user/**
            - Header=X-Vision, v2        # 规则：请求头必须包含 X-Vision 且值为 v2
            # 或者 - Cookie=tester, true # 规则：Cookie 中必须有 tester=true
```


---

**基于 `Query` 参数 / `Method` 路由**，**精细化控制**
- **精细化控制**：
  * **Method**：实现严格的读写分离。比如所有的 `GET` 请求路由到从库服务节点，所有的 `POST/PUT` 请求路由到主库服务节点。
  * **Query**：强制要求某个接口必须携带特定的 URL 参数（比如 `?source=app` 才能调用某服务）。

**实战配置**：
```yaml
      routes:
        - id: order_read_route
          uri: lb://mall-order-read-service
          predicates:
            - Path=/api/order/**
            - Method=GET                 # 规则：只接收 GET 请求
            - Query=source, app          # 规则：URL 后面必须带 ?source=app
```

---

**基于 `Weight` 权重路由**，**金丝雀发布**
- **金丝雀发布**：上线新版本时，为了安全起见，决定采用**金丝雀发布，即按流量比例打入**。
- 让 90% 的流量依然走老版本，抽出 10% 的真实用户流量去新版本试水。如果没报错，再慢慢把比例调成 50% -> 100%。
```yaml
      routes:
        - id: payment_v1_route
          uri: lb://mall-payment-service-v1
          predicates:
            - Path=/api/payment/**
            - Weight=payment_group, 9    # 规则：在 payment_group 这个分组里，占据 90% 的权重

        - id: payment_v2_route
          uri: lb://mall-payment-service-v2
          predicates:
            - Path=/api/payment/**
            - Weight=payment_group, 1    # 规则：在 payment_group 这个分组里，占据 10% 的权重
```


---

> **静态路由的缺点：只要服务有任何变动，都需要重新启动 Gateway 网关服务，因为application.yml的配置参数只会在初始化时读取**

##### 动态路由



**动态路由**是指网关在**不重启**的情况下，能够实时地更新、新增或删除路由规则。
- Spring Cloud Gateway 内部维护了一个 `RouteDefinitionLocator` 接口，它负责加载路由定义。
1.  **静态路由**：默认使用 `PropertiesRouteDefinitionLocator`，它从配置文件读取。
2.  **动态路由**：我们需要自定义或使用非默认的实现，将路由信息存放在 **数据库（MySQL）**、**Redis** 或 **配置中心（Nacos/Consul）** 中。

实现动态路由的三个关键动作：
1.  **存储（Persistence）**：将路由信息（JSON 格式）存入 Nacos 或数据库。
2.  **监听（Listening）**：网关监听配置中心的变化，或者提供一个 HTTP 接口接收更新指令。
3.  **刷新（Refreshing）**：当监听到变化后，调用 `RouteDefinitionWriter` 写入新规则，并发布一个 `RefreshRoutesEvent` 告诉网关更新路由规则




**Nacos + Gateway**
1.  **配置下发**：在 Nacos 管理后台直接修改一段 JSON 配置。
2.  **代码监听**：在网关代码中写一个监听器（Listener），一旦 Nacos 里的 JSON 变了，代码自动获取最新内容。
3.  **刷新路由**：代码内部调用网关的 API 将 JSON 转换为 `RouteDefinition` 对象，并刷新网关内部缓存。
- ps：
  - Nacos使用 `@RefreshScope`可以实现配置的热更新
  - 但对于 Gateway 的**路由列表**来说，仅仅刷新配置是不够的
  - 还需要手动或自动地触发一个 `RefreshRoutesEvent` 事件，网关才会重新构建路由表。



---
#### 过滤器

在 Spring Cloud Gateway 中，过滤器分为两种：
1. **GatewayFilter局部过滤器**：只对配置了它的**单个或部分路由**生效。通常通过 `application.yml` 配置。
2. **GlobalFilter全局过滤器**：对**所有路由**都生效。通常通过编写 Java 代码实现。


##### 局部过滤器


网关内置了三十多种**局部过滤器**工厂，**经常使用的有三种**
- 局部过滤器专门负责对 HTTP 报文的细节（路径、Header、参数）进行简单的增删改。它们高度依赖配置文件，非常死板，但胜在极其方便

---
**`StripPrefix`去除前缀**
- 前面讲到，前端发送的请求的URL由多部分拼接而成，发送给微服务运行实例的只需要接口信息，否则会有设计冗余，即nacos、网关、负载均衡器已经将请求打到负责它的运行实例上，只需要SpringBoot项目告诉`DispatcherServlet`分发给哪个接口就行；
- 因此需要`StripPrefix`去除接口信息前的前缀：nignx修饰符、业务领域名，因为SpringMVC的接口路径根本咩有这些前缀
- `StripPrefix=1` ：以/作为分隔符，砍掉第1部分，如果配置2则砍掉前两部分


```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user_service_route
          uri: lb://mall-user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1   # 规则：转发前，砍掉路径里的第 1 个部分（即 /api）
            # 如果配置 StripPrefix=2，就会砍掉 /api/user，只把后面的部分发给微服务
```

---

**`AddRequestHeader`添加请求头**
- 有些后端微服务需要根据特定的请求头来执行特定逻辑。比如，订单服务需要知道这个请求是不是从网关过来的（防止别人绕过网关直接拿内网 IP 访问订单服务）。
- 我们可以在网关层统一给转发出去的请求加上一个特殊的 Header 作为“暗号”。

**实战配置**：
```yaml
      routes:
        - id: order_service_route
          uri: lb://mall-order-service
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, Mall-Gateway  # 规则：增加 Header，键为 X-Gateway-Source，值为 Mall-Gateway
```

---

**`RewritePath`重写路径**
- `StripPrefix` 只能从头开始“砍”路径，**如果路径改造需求很复杂怎么办？**
- 比如
  - 为了兼容老旧的系统，前端请求的是 `/api/v1/user/info`
  - 但后端重构后，新接口的路径变成了 `/user/v1/api/info`

- 它接收两个参数：一个是**正则表达式**（用来匹配原始路径），另一个是**替换目标**。
```yaml
      routes:
        - id: legacy_user_route
          uri: lb://mall-user-service
          predicates:
            - Path=/api/v1/user/**
          filters:
            # 规则：将 /api/v1/(后面的所有内容) 替换为 /user/v1/api/(后面的所有内容)
            # $\1 代表正则表达式中第一个括号 (...) 匹配到的内容
            - RewritePath=/api/v1/(?<segment>.*), /user/v1/api/$\{segment}
```


---
##### 过滤器高频使用的WebFlux API

###### 控制方法


假设有过滤器链：1-2-3-4-5；
- 前置处理阶段执行顺序：1-2-3-4-5-末端过滤器
  - 工作线程执行前置处理逻辑
  - 调用`chain.filter(exchange)`方法将`Mono<void>`流水线、请求上下文ServerWebExchange、过滤器链交给下一个过滤器，下一个过滤器拼接流水线时，等价于本过滤器订阅了下一个过滤器
  - 末端过滤器用 HTTP 客户端把请求发给真正的微服务
- 后置处理阶段执行顺序：末端过滤器-5-4-3-2-1
  - 下游微服务处理完请求，把响应头和响应体完整地返还给网关，并由网关的 HTTP 客户端处理完毕时，末端过滤器发送`onComplete`信号
  - 因为上游过滤器订阅了下游过滤器，因此每个`chain.filter()`将收到`onComplete`信号
  - 如果`chain.filter()`后紧跟着 `.then()`，则它会先执行自己的后置逻辑
  - 执行完后置逻辑后， `chain.filter()`会自动向上游过滤器发送它自己的 `onComplete` 信号。
```JAVA
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // 【Pre 阶段】请求到达目标服务之前的逻辑
    System.out.println("进入全局过滤器...");

    return chain.filter(exchange).then(Mono.fromRunnable(() -> {
        // 【Post 阶段】目标服务返回响应后的逻辑
        System.out.println("目标服务执行完毕，返回响应...");
    }));
}
```
---

**前置处理阶段执行的控制方法**
- 这个阶段发生在网关刚刚接收到请求，**还没有**交给下游微服务之前
- 我们在这里做的事情通常是：查验身份、修改请求头、或者直接拒之门外。

**`chain.filter(exchange)`**
* 这是前置处理的**最后一步**。代表当前过滤器的前置逻辑已全部执行完毕，正式将组装好的请求交给过滤器链中的下一个节点。
* 通常作为 `return` 语句的主体返回。
    ```java
    // 前置逻辑：修改请求头
    ServerHttpRequest mutatedRequest = exchange.getRequest().mutate().header("X-Flag", "true").build();
    ServerWebExchange mutatedExchange = exchange.mutate().request(mutatedRequest).build();
    
    // 放行，交给下一个过滤器
    return chain.filter(mutatedExchange); 
    ```

**拦截：`exchange.getResponse().setComplete()`**
- 当你发现请求非法（IP在黑名单、Token造假）时，你需要立刻终止这个请求，绝不能让它流向下游。
- 调用这个方法相当于向上游直接发送 `onComplete` 完成信号，**提前让上游开始执行后置逻辑**，彻底切断后续的 `chain.filter()` 链路。
* 必须配合 `setStatusCode()` 使用，否则客户端会一脸懵逼地收到一个空的 200 OK。
    ```java
    // 前置逻辑：发现非法请求
    exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
    
    // 终止请求，不再执行 chain.filter
    return exchange.getResponse().setComplete(); 
    ```

**异步前置处理：`flatMap()`**
* 如果需要在前置阶段去请求 Redis 或数据库（这些操作在 WebFlux 中都会返回 Mono）
* 不能直接 `if-else`。必须用 `flatMap` 把这个异步结果“铺平”，等待结果回来后，再决定是调用 `chain.filter()` 放行，还是调用 `setComplete()` 拦截。
    ```java
    return redisTemplate.hasKeyReactive("token_blacklist:" + token) // 返回 Mono<Boolean>
        .flatMap(isBlacklisted -> {
            if (isBlacklisted) {
                // 黑名单中，拦截
                exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
                return exchange.getResponse().setComplete();
            } else {
                // 正常，放行
                return chain.filter(exchange);
            }
        });
    ```

---

**后置处理阶段执行的控制方法**
- 这个阶段发生在下游微服务已经处理完业务，并将 HTTP 响应交还给网关之后。此时前置的单向传递已经结束，正在原路返回
- 我们在这里做的事情通常是：记录响应时间、修改响应头、或者记录状态码。

**等待与衔接：`.then()`**
* 连接前置与后置的**桥梁**。
* 它永远挂在 `chain.filter(exchange)` 的屁股后面
* 它的语义是：“**死死盯住**下游服务，**只要下游服务返回了完整的响应（发出 onComplete 信号）**，**就立刻触发我里面包裹的代码”**。它是**启动后置处理阶段的唯一扳机**。
* **`then()` 里面必须塞入另一个 `Mono`**，它不关心上游的数据流，只关心上游是不是“结束”了。
- 在前置处理阶段，工作线程会执行then方法，将入参Mono对象存入自己的内存变量，等到后置阶段才会执行它

**同步收尾逻辑：`Mono.fromRunnable()`**
* 后置处理的**工作台**。
* 大部分后置逻辑（比如打印日志、计算耗时）都是普通的同步 Java 代码，不需要再返回什么复杂的数据流了。`fromRunnable()` 完美地将这些普通代码包装成 Reactor 能看懂的 `Mono<Void>`，塞进 `.then()` 里面执行。
    ```java
    return chain.filter(exchange) // 1. 前置结束，向下游发车
        .then(                    // 2. 蹲守下游服务处理完毕的信号
            Mono.fromRunnable(() -> { // 3. 信号到了，执行后置逻辑
                HttpStatus statusCode = exchange.getResponse().getStatusCode();
                System.out.println("响应状态码是：" + statusCode);
            })
        );
    ```

**为什么需要`Mono.defer()`？**
- 后置阶段的异步逻辑必须使用defer
  - 原因：**异步必须使用Mono或Flux来构建流水线，然后其组装必须要延迟到后置处理阶段，不能在前置处理阶段被组装**
- 考虑不使用`Mono.defer()`的代码
  - 示例方法**确实返回了`Mono<void>`**，但是由于Java饿汉式参数求值的特性，这个`Mono<void>`会在组装期，也就是过滤器的前置处理期被执行，拿到了一个错误的、还没更新的 HTTP 状态码（比如 null 或默认的 200）
  - 在 Java 中，当你调用一个方法，并将另一个方法的返回值作为参数传入时，**内部的方法会先执行。**
  - 因此Java会立即执行`auditService.saveErrorLog(...)`方法拿它的返回值
  - 最后，then方法确实拿到了一个Mono，**在前置处理阶段这个Mono 确实不会被执行，但是已经在前置处理阶段被错误地组装好了**
    ```JAVA
    return chain.filter(exchange).then(
        auditService.saveErrorLog(exchange.getResponse().getStatusCode())
    );
    ```
- 所以**必须使用`Mono.defer()`将真正的异步代码逻辑用lambda包裹作为defer的入参**
  - 示例方法中，`Mono.defer(lambda代码块)`，Java先看then的入参是Mono.defer方法，会执行它，然后看defer的入参是lambda代码块，因为**lambda的参数是延迟计算惰性求值**，所以组装期，也就是在前置处理阶段不会组装Mono
  - **直到后置处理阶段，才会执行defer的入参**，完成任务：**根据特定值，返回不同的Mono**
  - **前置处理阶段，执行defer方法时，会打包lambda代码块，将它交给then方法，等到then方法执行时才打开包装，看看生成的到底是什么Mono，然后执行该Mono**
    ```JAVA
    return chain.filter(exchange).then(
        Mono.defer(() -> {
            // 这是一段被封印的代码
            return auditService.saveErrorLog(exchange.getResponse().getStatusCode());
        })
    );

    return chain.filter(exchange).then(
    // 使用 Mono.defer 包装
    Mono.defer(() -> {
            // 【关键点】这里面的代码，只有在下游服务处理完毕、.then() 收到完成信号后，才会被真正执行！
            
            // 1. 此时拿到的绝对是下游微服务返回的真实状态码
            HttpStatus statusCode = exchange.getResponse().getStatusCode();
            
            System.out.println("拿到真实状态码：" + statusCode);

            // 2. 根据状态码，动态决定返回什么样的 Mono 流水线
            if (statusCode != null && statusCode.is5xxServerError()) {
                System.out.println("检测到 5xx 异常，启动异步审计日志记录...");
                // 返回一个异步的网络请求 Mono
                return auditService.saveErrorLog(statusCode); 
            } else {
                // 如果正常，返回一个空的 Mono，直接结束
                return Mono.empty(); 
            }
        })
    );
    ```


---


###### 请求/响应的篡改方法

WebFlux中的请求上下文
- `ServerWebExchange`以及它包裹的`ServerHttpRequest`是**不可变的**
- `ServerHttpResponse` 的 `Header` 和 `StatusCode` **在响应被提交之前，是允许直接修改的**

---

**前置阶段篡改请求**
- 不能修改`ServerWebExchange`、`ServerHttpRequest`，则只能克隆并修改，因此WebFlux提供了**Builder 建造者模式**，**通过`.mutate()` 方法触发**
- 核心的任务：**把网关层计算出的数据（比如鉴权后解析出的 UserID），悄悄塞进请求里，带给下游微服务。**

- **因为 Request 是包在 Exchange 里面的**，**而且两者都是不可变的**，所以必须进行**两次脱壳与重新包装**。
    ```java
    // 【核心步骤拆解】

    // 第一步：对原有的 Request 进行 mutate（克隆），在克隆的过程中“夹带私货”
    ServerHttpRequest newRequest = exchange.getRequest().mutate()
        .header("X-User-Id", "10086")       // 新增或覆盖 Header
        .header("X-User-Role", "ADMIN")     // 可以链式调用，添加多个
        .path("/new-path/api")              // 甚至可以篡改请求路径 (URI 转发)
        .build();                           // build() 生成了全新的 Request 对象

    // 第二步：对原有的 Exchange 进行 mutate（克隆），把刚才造好的新 Request 塞进去
    ServerWebExchange newExchange = exchange.mutate()
        .request(newRequest)                // 替换掉旧的 Request
        .build();                           // build() 生成了全新的 Exchange 对象

    // 第三步：将崭新的 Exchange 交给下一环
    return chain.filter(newExchange);
    ```

- **篡改/追加 Query 参数**
  - 除了 Header，有时我们需要给 URL 后面追加参数（例如 `?source=gateway`）
  - 因为**原生的 `mutate()` 没有直接提供追加参数的方法**，我们需要对 URI 进行重构。

    ```java

    // 1. 获取原 URI
    URI uri = exchange.getRequest().getURI();

    // 2. 利用 UriComponentsBuilder 追加参数并生成新 URI
    URI newUri = UriComponentsBuilder.fromUri(uri)
        .queryParam("source", "gateway_filter")
        .build(true)
        .toUri();

    // 3. 将新 URI 塞入新 Request，再塞入新 Exchange
    ServerHttpRequest newRequest = exchange.getRequest().mutate().uri(newUri).build();
    return chain.filter(exchange.mutate().request(newRequest).build());
    ```

---

**后置阶段篡改响应**
-  `ServerHttpResponse` 的 Header 和 StatusCode **在响应被提交之前，是允许直接修改的！**
- **直接修改：状态码与 Header**
  - **场景 A：拦截时直接篡改（前置阶段直接结束）**
    ```java
    // 发现非法请求，直接设置状态码并添加提示 Header
    exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
    exchange.getResponse().getHeaders().add("X-Error-Reason", "Token Expired");

    return exchange.getResponse().setComplete(); // 终止请求
    ```
  - **场景 B：正常返回时追加 Header（后置阶段）**
    - 给所有经过网关成功返回的响应，都加上一个全局追踪 ID（Trace ID）。
    ```java
    return chain.filter(exchange).then(Mono.defer(() -> {
        // 下游服务已经处理完毕，准备返回给前端前，强行塞入一个 Header
        exchange.getResponse().getHeaders().add("X-Trace-Id", UUID.randomUUID().toString());
        
        return Mono.empty(); // 或者继续其他后置逻辑
    }));
    ```

---

**篡改Body请求体/响应体**
- 网关**应该只负责路由、鉴权、限流这些轻量级的控制面工作**，而具体的业务逻辑（生成或解析 Body 数据）应该交给具体微服务
- 在网关层篡改Body请求体/响应体非常危险
  - **极易引发内存泄漏**：
    - WebFlux 底层使用的是 Netty 的 DataBuffer 来管理内存（堆外内存）
    - 如果你在网关里读了 Body 流，稍有不慎忘记手动 release()（释放内存），网关跑几天就会内存溢出 OOM，直接宕机。
  - **极度损耗性能**：
    - 网关的优势是高并发、非阻塞
    - 如果你把大体积的 JSON 流拦截下来，转成字符串，做正则替换或解密，再转回流，这会剧烈消耗网关 CPU，把网关从“高速收费站”变成了“货物开箱检查站”，并发量直接暴跌。

- 需要篡改Body请求体/响应体的奇葩场景
  - **填前人的坑**：
    - 正常做法：在每个微服务里写一个 `@RestControllerAdvice`全局响应拦截器进行统一包装
    - 由于旧系统代码年久失修、没人敢动，或者由于是第三方外包写的系统根本拿不到源码。这时候，只能在网关层拦截所有响应流，把乱七八糟的 Body 读出来，套上 code 和 message 的外壳，再重写回响应流里发给前端。
  - **全局接口加解密**
    - 金融、政务或对安全性要求极高的 App，前端发往后端的请求体通常不会是明文 JSON，而是一整串 AES 或 RSA 加密过的密文
  - **全局数据脱敏**
    - 下游微服务数量太多。用户服务返回了包含真实身份证号和手机号的 JSON。
    - 为了满足合规要求，网关层通过后置拦截响应体，利用正则匹配，把所有 13812345678 替换为 138****5678，再返回给外部网络
- **如果真的需要改`Body` 必须使用`ModifyRequestBodyGatewayFilterFactory` 和 `ModifyResponseBodyGatewayFilterFactory`**


- **为什么篡改Body请求体/响应体极其复杂？**
  - 因为在 WebFlux 中，Body 不是一个简单的 `String` 或 `byte[]`，它是一个 **`Flux<DataBuffer>`（数据流）**。
  - 流的致命特性是：**只能被读取/消费一次！**
  - 如果你在网关过滤器里通过 `exchange.getRequest().getBody()` 把流读了出来，下游微服务收到的 Body 就会是**空的**，直接报反序列化异常。

- **解决方案：装饰器模式**：使用**装饰器**来拦截和重写流。它的**核心骨架**如下：
  1. **写一个内部类继承 `ServerHttpRequestDecorator`（请求装饰器）或 `ServerHttpResponseDecorator`（响应装饰器）。**
  2. **重写里面的 `getBody()` / `writeWith()` 方法。** 在这里面，你把原始的 `DataBuffer` 截获，转成 `String` 修改掉，然后再重新封装成 `DataBuffer` 丢出去。
  3. **用这个装饰器替换掉原生的 Request/Response。**

    ```java
    // 1. 创建响应装饰器
    ServerHttpResponseDecorator decoratedResponse = new ServerHttpResponseDecorator(exchange.getResponse()) {
        @Override
        public Mono<Void> writeWith(Publisher<? extends DataBuffer> body) {
            // 在这里面大做文章：
            // 拦截 body -> 转成 String -> 修改 JSON 内容 -> 重新打包成新的 DataBuffer 流发给前端
            return super.writeWith(modifiedBodyStream); 
        }
    };

    // 2. 将装上了装饰器的 Response 放行（注意这里修改了 exchange）
    return chain.filter(exchange.mutate().response(decoratedResponse).build());
    ```

---



##### 全局过滤器



**全局过滤器GlobalFilter**对**网关中所有的路由都默认生效**，不需要任何额外的路由绑定配置。
- 全局过滤器通常用于处理那些**所有请求都必须经过的通用逻辑**。
- 全局过滤器只需要 **让自定义的类实现 `GlobalFilter` 和 `Ordered` 这两个接口，并将其注册为 Spring 的 `@Component`**
- `Ordered` 接口用于指定过滤器的执行顺序，**数字越小，优先级越高**。
    ```java
    @Component
    public class StandardGlobalFilter implements GlobalFilter, Ordered {

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            // 【Pre 阶段】请求到达目标服务之前的逻辑
            System.out.println("进入全局过滤器...");

            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                // 【Post 阶段】目标服务返回响应后的逻辑
                System.out.println("目标服务执行完毕，返回响应...");
            }));
        }

        @Override
        public int getOrder() {
            return 0; // 设置优先级
        }
    }
    ```

接下来讲的三种应用场景代码都很基础
- 日志肯定是高度定制化的，必须手写，通常在过滤器里把收集到的日志数据直接组装成 JSON，异步推送到 Kafka 或者 ELK（Elasticsearch + Logstash + Kibana）日志中心
- 黑名单/白名单过滤：后台微服务把黑名单写进数据库并同步到 Redis；网关过滤器全程只和 Redis（或本地内存）打交道，利用 ReactiveRedisTemplate 的 flatMap 进行非阻塞拦截。
- 鉴权后面再细讲
---

**业务场景 1：全局请求日志记录**
- 网关是微服务的统一入口，记录全局日志是基础需求。我们可以记录请求的 IP、路径、耗时等。

- **注意：** 在 WebFlux 环境下，由于请求体（Request Body）和响应体（Response Body）是以数据流（Flux/Mono）的形式存在的，只能被消费一次。如果直接在过滤器中读取，后端服务就读不到了。完整的请求体记录通常需要借助 `ServerHttpRequestDecorator` 对请求进行包装重写，这里我们先实现**请求耗时、IP 和基础信息的记录**。

    ```java
    @Component
    public class RequestLogFilter implements GlobalFilter, Ordered {

        private static final String START_TIME_KEY = "request_start_time";

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            // 1. 获取客户端 IP 和请求路径
            String clientIp = exchange.getRequest().getRemoteAddress().getHostString();
            String path = exchange.getRequest().getURI().getPath();
            
            // 2. 记录请求开始时间，存入 exchange 的 attributes 中传递
            exchange.getAttributes().put(START_TIME_KEY, System.currentTimeMillis());
            
            System.out.println(String.format(">> [LogFilter] 请求进入 | IP: %s | Path: %s", clientIp, path));

            // 3. 放行请求，并在响应后计算耗时
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                Long startTime = exchange.getAttribute(START_TIME_KEY);
                if (startTime != null) {
                    long executeTime = System.currentTimeMillis() - startTime;
                    int statusCode = exchange.getResponse().getStatusCode().value();
                    System.out.println(String.format("<< [LogFilter] 请求完成 | 状态码: %d | 耗时: %d ms", statusCode, executeTime));
                }
            }));
        }

        @Override
        public int getOrder() {
            // 日志记录通常优先级较高（或者设为最低也可以，看具体统计需求），这里设为 -1
            return -1; 
        }
    }
    ```

---

**业务场景 2：全局统一鉴权拦截**

- 这是网关最重要的功能之一。在这里拦截所有请求，校验 Header 中的 Token。如果有效则放行，并可将解析出的用户信息透传给下游微服务；如果无效，直接在网关层驳回，保护后端服务。

- 这也是你接下来整合 **JWT / Sa-Token** 的核心入口。

    ```java
    @Component
    public class AuthGlobalFilter implements GlobalFilter, Ordered {

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            String path = exchange.getRequest().getURI().getPath();

            // 1. 白名单放行（如登录、注册接口不需要鉴权）
            if (path.contains("/login") || path.contains("/register")) {
                return chain.filter(exchange);
            }

            // 2. 从请求头中获取 Token
            String token = exchange.getRequest().getHeaders().getFirst("Authorization");

            // 3. 校验 Token (这里写伪代码，后续由 JWT 或 Sa-Token 替换)
            if (token == null || token.isEmpty() || !"valid-token".equals(token)) {
                System.out.println("[AuthFilter] 无效的 Token，拒绝访问！");
                
                // 设置响应状态码为 401 (未授权)
                exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.UNAUTHORIZED);
                // 结束请求，不向下游继续转发
                return exchange.getResponse().setComplete();
            }

            // 4. Token 校验通过，甚至可以将解析后的 userId 写入 Header 透传给下游
            // ServerHttpRequest mutatedRequest = exchange.getRequest().mutate()
            //         .header("X-User-Id", "1001").build();
            // ServerWebExchange mutatedExchange = exchange.mutate().request(mutatedRequest).build();
            // return chain.filter(mutatedExchange);

            System.out.println("[AuthFilter] 鉴权通过，放行！");
            return chain.filter(exchange);
        }

        @Override
        public int getOrder() {
            // 鉴权过滤器的优先级通常要很高，确保非法请求尽早被拦截
            return 0; 
        }
    }
    ```

---

**业务场景 3：IP 黑白名单过滤**
- 为了防止恶意刷接口或进行内部系统访问限制，我们可以在最外层拦截特定 IP。

    ```java
    @Component
    public class IpBlackListFilter implements GlobalFilter, Ordered {

        // 模拟从配置中心或数据库加载的黑名单
        private static final List<String> BLACK_LIST = Arrays.asList("192.168.1.100", "10.0.0.5");

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            // 1. 获取真实访问 IP
            String clientIp = exchange.getRequest().getRemoteAddress().getHostString();

            // 2. 检查是否在黑名单中
            if (BLACK_LIST.contains(clientIp)) {
                System.out.println("[IpBlackListFilter] 拦截到黑名单 IP: " + clientIp);
                
                // 返回 403 Forbidden 状态码
                exchange.getResponse().setStatusCode(org.springframework.http.HttpStatus.FORBIDDEN);
                return exchange.getResponse().setComplete();
            }

            // 3. 不在黑名单，正常放行
            return chain.filter(exchange);
        }

        @Override
        public int getOrder() {
            // IP 黑名单拦截应该是最先执行的防线之一，优先级极高
            return -100; 
        }
    }
    ```

---



##### 过滤器执行顺序

**标准的护城河排序规范**：

1. **绝对防线层 (`order = -200` 到 `-100`)**：
   * **跨域处理 (CORS)**：最早处理预检请求。
   * **全局黑白名单 (BlackList)**：最快拦截掉恶意 IP，不浪费哪怕一点点后续的计算资源。
2. **安全防线层 (`order = -10` 到 `0`)**：
   * **全局统一鉴权 (Auth / JWT)**：验证身份是否合法。
   * **接口防刷/限流 (RateLimit)**：基于用户身份或 IP 进行流控。
3. **业务增强层 (`order = 1` 到 `50`)**：
   * **请求头修改 (Mutate Header)**：为通过了鉴权的请求加上 UserId 传给下游。
   * **参数校验**：一些全局的格式校验。
4. **日志监控层 (极高或极低均可，但常用 `HighestPrecedence` 或 `-1`)**：
   * **全局请求日志**：如前所述，为了包裹整个请求生命周期，通常放在极高的优先级，最先进入，最后离开。

在 Spring Cloud Gateway 中，无论是局部过滤器还是全局过滤器，它们最终都会被框架合并成一条完整的**过滤器链**，并严格按照顺序执行。

`int order`：数字越小，在过滤器链中的位置越靠前
- 前置阶段：`order` 值越小，越**先**执行
- 后置阶段：`order` 值越小，越**后**执行。

日志记录
- 如果想记录最准确的**总耗时**，日志过滤器优先级必须设得**极高**
> 这样它在 Pre 阶段第一个记录开始时间，在 Post 阶段最后一个计算结束时间，完美包裹住了中间所有的鉴权、路由和微服务调用耗时。

---

控制顺序的两种等效方式
1. **实现 `Ordered` 接口**（最推荐的硬核写法）
    ```java
    @Component
    public class AuthFilter implements GlobalFilter, Ordered {

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            // ... 逻辑 ...
            return chain.filter(exchange);
        }

        // 重点在这里：重写 getOrder 方法
        @Override
        public int getOrder() {
            return 0; // 返回一个整数，数字越小优先级越高
        }
    }
    ```
   * **优点**：你可以将逻辑写在代码里，甚至可以通过读取配置中心的值来**动态返回**这个顺序（虽然很少这么干），更加灵活。
2. **使用 `@Order` 注解**
   - 如果你不想多实现一个接口，可以直接在类上打一个 `@Order` 注解。

    ```java
    @Component
    @Order(-100) // 重点在这里：直接通过注解指定，数字越小优先级越高
    public class BlackListFilter implements GlobalFilter {

        @Override
        public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
            // ... 逻辑 ...
            return chain.filter(exchange);
        }
    }
    ```
   * **优点**：代码侵入性小，一眼就能看到优先级。
   * **缺点**：**值是静态写死的，在编译期就已经确定**。

- 两者对比与冲突处理

| 特性 | `Ordered` 接口 | `@Order` 注解 |
| :--- | :--- | :--- |
| **代码量** | 稍微多一点，需重写方法 | 极少，一行注解搞定 |
| **灵活性** | 高（可动态计算并 return） | 低（必须是常量） |
| **优先级冲突** | **同等生效**，若一个类同时用了两者，**接口的优先级高于注解** |



---
#### CORS统一跨域处理

跨域CORS的本质：浏览器的自我保护
- 跨域并不是后端的限制，而是**浏览器的同源策略**在作祟。
- 当你的前端应用去请求不同域名、不同端口、或者不同协议的后端接口时，浏览器会产生警觉。
- 对于复杂的请求，浏览器会先偷偷发一个 **`OPTIONS` 预检请求**。
- 浏览器在问服务器：“大哥，那个 `localhost:8080` 的前端想跨域调你，你同意吗？”
- 如果后端没有配置跨域允许头，浏览器就会直接拦截真实的业务请求，并在控制台无情地抛出经典的 `CORS error` 红字。

**前端**部署需要购买域名，比如`https://www.niubi.com`，请求的Origin（跨域来源头）都是这个域名。后端设置allowedOrigins属性也是这个域名

如果后端设置`allowedOrigins: "*"`，意味着其他所有域名的请求都能通过网关，能直接调用服务，这是很危险的


---

**为什么要在网关做统一跨域？**
- 在没有网关的时代，我们必须在**每一个微服务**（用户服务、订单服务、支付服务）里去写 `@CrossOrigin` 注解或者配置 `CorsFilter`。
- 有了网关之后，微服务架构的统一入口优势就体现出来了：
- **让下游微服务彻底“躺平”，专心做业务。所有的跨域请求校验、OPTIONS 预检拦截、允许跨域 Header 的添加，全部在网关这一层一次性搞定！**

---


> ⚠️ **死规则**：既然决定在网关层做跨域，那就**必须彻底清理掉**下游**所有微服务里的跨域配置**（删掉微服务里的 `@CrossOrigin` 和自定义的 `CorsFilter`）！

**为什么？**
- 如果网关配置了跨域（会在响应头里加上 `Access-Control-Allow-Origin: *`），而你的下游微服务也配置了跨域（又加了一次 `Access-Control-Allow-Origin: *`）。
- 当响应回到浏览器时，浏览器一看：好家伙，有两个同名 Header（`Access-Control-Allow-Origin: *, *`）！浏览器会认为服务器配置混乱，直接抛出 CORS 跨域异常。

---

**最优雅的 YAML 零代码方案**
- 在 Spring Cloud Gateway 中，跨域配置已经标准化到了极点，你**完全不需要写任何 Java 代码**，直接在 `application.yml` 中配置即可。Gateway 底层会自动将其转化为我们之前聊过的 `CorsWebFilter`，执行优先级极高。

```yaml
spring:
  cloud:
    gateway:
      globalcors: # 全局跨域配置
        add-to-simple-url-handler-mapping: true # 解决 HTTP OPTIONS 请求被拦截的问题
        cors-configurations:
          '[/**]': # 拦截网关所有的请求路径
            allowedOrigins: 
              - "http://localhost:8080" # 允许的前端地址。在开发环境可以写 "*" 放行所有，生产环境必须写死具体的域名
              - "https://www.yourdomain.com"
            allowedMethods: # 允许的前端请求方法
              - "GET"
              - "POST"
              - "PUT"
              - "DELETE"
              - "OPTIONS" # 务必放行 OPTIONS，这是浏览器的预检请求
            allowedHeaders: "*" # 允许前端携带的请求头（比如允许前端携带 Authorization Token）
            allowCredentials: true # 是否允许前端携带 Cookie/凭证。注意：如果设为 true，allowedOrigins 绝对不能写 "*"
            maxAge: 3600 # 预检请求的缓存时间（秒）。在 1 小时内，浏览器不需要再为同一个接口发送 OPTIONS 预检请求了，提升性能。
```


1. **`add-to-simple-url-handler-mapping: true`**
   * 这是 WebFlux 架构下的一个特定配置。如果不加这一行，有些浏览器的 `OPTIONS` 预检请求可能会直接报 404，导致跨域失败。
2. **`'[/**]'`**
   * 这是一个路径匹配表达式，代表该 CORS 规则对网关下代理的所有微服务路由都生效。
3. **`allowCredentials` 与 `allowedOrigins` 的相爱相杀**
   * 出于极高的安全性考虑，W3C 规范严格规定：如果你允许前端跨域携带 Cookie 或身份凭证（`allowCredentials: true`），那么你的 `allowedOrigins` 就**绝对不能使用通配符 `*`**，必须老老实实写明具体的前端域名。否则配置会直接失效。

---
#### 全局异常处理机制


网关需要把所有的意外，都转化成前端能够轻松解析的标准 JSON 格式（比如 `{"code": 500, "message": "网关服务开小差了"}`）。

**为什么不能用 `@RestControllerAdvice`？**
- `@RestControllerAdvice`是Spring MVC的注解，在Spring WebFlux中无效

WebFlux使用`ErrorWebExceptionHandler`作为全局异常兜底处理器
- **实现这个接口，拦截异常，手动拼装 JSON，然后写回给前端。**
- **异常处理器不是过滤器**，异常处理器链位于整条过滤器链之外；Order越小，在异常处理器链中的位置越靠前，会优先处理异常
- Spring Boot WebFlux 底层自带了一个 `DefaultErrorWebExceptionHandler`，专门用来生成那个丑陋的英文报错页面；它的默认 Order 值通常是 `-1`。所以必须**设置自定义异常处理器的Order小于-1**，并且**确保流在自定义处理器阶段就结束**，不会将异常抛给默认异常处理器

**响应已提交**
- **什么时候会触发响应提交？**
  - **开始写入 Body 时**： 当调用 `response.writeWith(Publisher)` 并且 `Publisher` 发出第一个数据块（`DataBuffer`）时。因为**HTTP 协议要求必须先发头部再发数据体**，所以**框架必须在此刻锁定并发送头部**。
  - **显式结束响应时**： 当调用 `response.setComplete()` 声明不需要写入任何 Body，直接结束请求时。
- **响应已提交的限制**：
  - **无法修改状态码**： 不能再将 200 OK 更改为 500 Internal Server Error 或 302 Found。
  - **无法修改响应头**： 不能再添加、删除或修改 Headers（例如，无法再添加新的 Cookie，或者修改 Content-Type）。
  - **只能写入响应体**Body： 此时，HTTP 报文的头部已经发走，**网络流处于只写数据体的阶段**。在 WebFlux 中，这通常意味着你返回的 Flux 或 Mono 正在将数据流（DataBuffers）源源不断地推向客户端。

**为什么异常处理器要检查响应是否已提交？`if (response.isCommitted())`**
- 因为状态码 200 OK 已经发给了客户端，不可能再把它改成 500 Internal Server Error
- 也不可能在原本应该是二进制文件流的末尾硬塞进一段 { "error": "Gateway Timeout" } 的 JSON，那样会导致客户端解析文件损坏。
- 所以此时正确的做法就是代码里写的：`return Mono.error(ex)`;
- 异常将被抛给下一个异常处理器，每个异常处理器都要求执行`if (response.isCommitted())`，直到被抛给Netty
- Netty收到异常后，直接发送一个 RST 包，**强制掐断 TCP 连接**。客户端会收到一个“网络连接异常中断”的错误，从而知道下载失败了

**自定义异常处理器的返回值**
- 要么返回`response.writeWith(...)`将数据写入Body，触发响应提交，并且写入响应数据
- 要么返回`Mono.empty()`触发onComplete 信号，终止异常处理责任链，跳过后面的处理器
- 除非是已提交状态，不能抛出 **`return Mono.error(ex)`，否则会触发下一个处理器的执行**

**`@Order(-2)` 是保命符**
- Spring Boot WebFlux 底层自带了一个 `DefaultErrorWebExceptionHandler`，专门用来生成那个丑陋的英文报错页面；它的默认 Order 值通常是 `-1`。
- **自定义的处理器优先级必须小于-1**




```java

@Component
@Order(-2) // 极其重要：优先级必须设得足够高（小于 -1），为了覆盖 Spring Boot 默认的异常处理器
public class GlobalExceptionHandler implements ErrorWebExceptionHandler {

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        ServerHttpResponse response = exchange.getResponse();

        // 1. 如果响应已经处于提交状态（说明响应头已经发给前端了），我们无能为力，直接抛给框架兜底
        if (response.isCommitted()) {
            return Mono.error(ex);
        }

        // 2. 统一设置响应头，告诉前端这是一串 JSON，编码是 UTF-8
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

        // 3. 根据异常类型，精细化定制状态码和提示信息
        int statusCode = 500;
        String message = "网关内部系统异常，请稍后再试";

        if (ex instanceof ResponseStatusException) {
            // 比如 404 Not Found
            statusCode = ((ResponseStatusException) ex).getStatusCode().value();
            message = ex.getMessage();
        } else if (ex instanceof IllegalArgumentException) {
            statusCode = 400;
            message = "非法的请求参数";
        }
        // 这里可以继续 if-else 拦截你自定义的业务异常 (如 TokenExpiredException)

        // 设置真正的 HTTP 状态码
        response.setStatusCode(HttpStatus.valueOf(statusCode));

        // 4. 构建标准化的 JSON 字符串 (企业中通常会用 ObjectMapper 序列化一个 Result 对象，这里为了直观用字符串拼接)
        String jsonResult = String.format("{\"code\": %d, \"message\": \"%s\", \"data\": null}", statusCode, message);

        // 5. 将字符串包装成 DataBuffer (响应式数据流)
        DataBuffer buffer = response.bufferFactory().wrap(jsonResult.getBytes(StandardCharsets.UTF_8));

        // 6. 将数据流写回给前端，完成异常的优雅处理
        return response.writeWith(Mono.just(buffer));
    }
}
```

---
#### Gateway与Spring Cloud Alibaba Sentinel



**Alibaba Sentinel** 是目前微服务生态中最主流的面向分布式服务架构的**流量控制组件**。
相比于网关原生限流，拥有几个杀手级特性：
* **功能极度丰富**：不仅支持按 QPS 限流，还支持按并发线程数限流、热点参数限流、系统自适应限流。
* **自带熔断降级**：支持基于响应时间、异常比例、异常数的自动熔断。
* **可视化控制台**：自带一个强大的 Web 控制台，可以直接在界面上动态修改规则，实时生效，并且能看到秒级的流量监控曲线，而不需要像原生 SCG 那样去修改配置文件或写代码。


**Sentinel 与 Spring Cloud Gateway 的底层关系**

* **插件式寄生**：Sentinel 官方专门提供了一个适配器依赖包 `sentinel-spring-cloud-gateway-adapter`
* 在网关中引入这个包时，Sentinel 会把自己包装成一个 **Gateway Filter网关过滤器**，插入到 SCG 原生的过滤器链条中。
* **分工明确，全链路布防**：
    * **网关层的 Sentinel**：镇守城门。处理**南北向流量**（外网调用内网），主要针对外部 IP、API 路径、App 来源进行整体的流量清洗。
    * **微服务层的 Sentinel**：镇守内院。处理**东西向流量**（内网微服务之间的互相调用），主要针对特定的 Java 方法、核心数据库操作进行细粒度的保护和熔断。



**既然最终要用 Sentinel，为什么还要先学 SCG 原生实现？**

1. Sentinel 替换的仅仅是计算令牌的引擎，把 Redis Lua 换成了 Sentinel 内部的算法。但是，**流量拦截的骨架依然是 SCG 提供的**。
   - 用Sentinel精准配置网关级流控规则的前提是，知道SCG`KeyResolver` 如何从请求中提取特征
2. **兜底：微服务阵亡时的遮羞布**
   - 微服务暴毙时，部署在微服务内部的 Sentinel跟着暴毙，它根本没有任何机会去执行微服务层面的降级策略
   - 网关层的全局熔断与降级机制，是独立于微服务之外的最后一道防线。即使微服务死透了，网关依然能拦截并返回友好的 JSON


---
#### 限流



#### 熔断
    * 当后端微服务宕机或响应过慢时，网关如何快速返回友好提示（兜底数据），避免雪崩。
    * 整合 Sentinel 或 Resilience4j 实现网关层容错。

#### 服务降级

