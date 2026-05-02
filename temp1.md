#### 限流
##### 限流层

**微服务系统架构中，限流有三道防线**

**Nginx接入层限流**：**整个系统的最外层，直面公网流量**

* **核心目标**：阻挡恶意攻击（如 DDoS、恶意爬虫），进行最粗粒度的流量清洗
* **限流维度**：
    * **单 IP 并发连接数**：比如同一个 IP 瞬间发起了 1000 个连接，直接拉黑
    * **请求速率**：Nginx 的 `limit_req` 模块

**网关层限流**Spring Cloud Gateway：内网门户
- **核心目标**：基于**业务属性**进行精细化的流量分发和管控。网关能够解析 HTTP 协议，看得懂 Header、Token 和请求路径。
* **限流维度**：通过`KeyResolver` 实现
    * **接口级**：解析URL路由，限制指定接口的QPS为500或300，能精细控制每个接口的限流策略
    * **用户级**：解析 JWT 拿到 UserID，限制某个特定用户每分钟只能发 10 条评论，防止他刷屏。
    * **应用级**：根据 Header 里的特征，限制 App 端的流量，优先保证 Web 端可用。

**微服务层限流Sentinel：微服务内部**
* **核心目标**：**服务的绝对自我保护**。为什么要这一层？因为有时哪怕网关放行的是正常流量，但微服务内部的数据库突然卡了，或者内部系统 A 疯狂调用系统 B，如果没有自我保护，微服务就会被拖死，进而引发全链路雪崩。
* **限流维度**：
    * **具体 Java 方法/代码块级**：网关只能限制 URI，但微服务内部可以限制某个具体的 `createOrder()` 方法。
    * **热点参数限流**：比如双 11 抢购，大家都在买同一个 iPhone（ID=999），微服务层可以针对 `productId=999` 这个特定参数进行限流，而不影响买其他手机的人。

---
##### 令牌桶算法

**网关配置的核心参数**
* **补充速率 replenishRate：** 系统每秒钟往桶里放入多少个令牌。这代表了期望的**稳态平均并发量**。
* **突发容量burstCapacity：** 这个桶最大能装多少个令牌。这代表了系统能允许的**瞬间最大突发流量**。
* **请求消耗requestedTokens：** 默认情况下每个请求消耗 1 个令牌，通常不需要改动。


SCG 默认采用令牌桶算法，是目前**应对突发流量**最优雅的限流算法
- 假设桶的容量是 100，当前是满的。即使瞬间涌入 100 个请求，只要它们能瞬间拿走这 100 个令牌，系统就会立刻放行这批请求，而不会让它们排队
- 可以采取多维度的限流策略
  - 比如同时采取**按用户User ID限流**，**按 API 接口（提取请求的 URI，比如 /api/order/create）限流**
  - 它们的桶**在不同的命名空间**，互不干扰。
  - **每个命名空间有多个独立桶**，比如按用户User ID限流，处理过多少个UserID请求，该命名空间下就有多少个令牌桶
- 每个桶有一个**固定容量`burstCapacity`**，代表突发容量，代表最多可以存放多少个令牌Token
- 每个请求到来时
  - 如果UserID或接口资源没有桶，就为它定制一个桶初始化为满令牌
  - 下次这个UserID再访问时，读取当前的令牌数，通过数学公式懒加载计算，距离上次访问应该增加多少个令牌，然后消耗令牌然后更新令牌
- **怎么计算两次访问的时间差应该增加多少桶？**
  - **匀速放入令牌`replenishRate`补充速率**：系统以恒定的速率，例如每秒 10 个，向桶中放入令牌
  - 时间差$\times$`replenishRate`就是要增加的桶
  - 如果放入令牌时桶已经满了，**多余的令牌会被直接丢弃。**
- **请求消耗令牌requestedTokens**：可以通过配置文件声明，这种类型的限流策略，每个请求要拿走多少个令牌，才能继续被处理
- **如果没有充足的令牌，当前请求被拒绝**，网关返回429状态码，message是"Too Many Requests"

---

SCG通过网络和Redis通信，将`Read-Modify-Write`操作写在Lua脚本中发给Redis，Redis返回allowed（1代表放行，0代表拒绝）和new_tokens（当前令牌数）
- **为什么要用Lua脚本？**
  - 单机Redis是单线程模型，Redisa把Redis视为大一点的单一命令
  - 在任何情况下，**没有执行完一段Lua脚本，绝不会继续执行下一个命令**
  - 天然**保证Lua脚本的原子性**，不会出现并发更新覆盖
- **Redis不会为逻辑错误背锅**
  - * **Lua包含耗时操作或死循环）**
    - Redis的`lua-time-limit`配置**默认是 5 秒**，**一旦脚本执行时间超过5秒**，为了保持脚本执行不中断，且告知管理员这段脚本肯定有问题，会**处理其他请求并直接回复BUSY 错误**，仍然不会执行实际的Lua脚本
    - 所以Lua脚本写死循环、进行超大规模的数据遍历是错误的，这会**将整个Redis 节点锁死**，**所有限流请求都会超时，进而导致所有路由不可用，引发生产事故**
    - 一旦Lua脚本无法正常结束，超出5秒后续请求一直报错，只能通过`SCRIPT KILL` 来强行杀掉执行超时的脚本
  - **Redis Cluster 集群模式**
    - Redis Cluster中数据会被打散分配在不同节点上的，Redis 严格要求：**同一个 Lua 脚本中操作的所有 Key，必须存在于同一个集群节点上。**
    - 如果在脚本里同时去 **`GET` 不同机器上的的限流数据，脚本直接报错**
    - 解决方案：**使用 Hash Tag（例如把 key 命名为 `{rate_limit:user}:A`），强制把相关的 Key 路由到同一个槽位**
  - **主从架构中，如果 Lua 脚本每次执行的结果不可预测，会导致主从数据不一致**
    - 错误使用：在 Lua 脚本里调用 `math.random()` 生成随机数来做某些判断，或者调用 `TIME` 命令获取当前系统时间进行逻辑计算
    - 主节点执行时的时间/随机数，和从**节点重放这段脚本**时的时间/随机数是不一样的，会导致两边扣减的令牌数对不上
  - **返回极大的数据集**
    - Lua 脚本执行完后把几万条数据一次性吐给网关。这不仅会造成网关内存飙升（OOM），也会占用大量的网络带宽，拉长Redis单线程的阻塞时间



**为了防止 Redis 里堆积大量不再使用的限流 Key，必须给 Key 设置过期时间**
- 比如某个用户只访问了一次就再也不来了，会造成内存泄漏
- 把空桶填满需要的时间是 `capacity / rate`
- SCG 通常会将 TTL 设置为这个填满时间`capacity / rate`的 **2倍**


Redis 主从架构
- **Redis 5.0 以前**，从节点通过重放脚本来实现主从复制
- **Redis 5.0 起**，默认变成了**效果复制**
  - Redis 不再把整段 Lua 发给从节点执行
  - 而是直接把 Lua 脚本执行完后产生的最终写指令打包发给从节点
  - 从底层化解了非确定性脚本带来的不一致风险



```LUA
-- 接收 Java 传过来的参数
local tokens_key = KEYS[1]       -- 记录剩余令牌的key
local timestamp_key = KEYS[2]    -- 记录时间戳的key
local rate = tonumber(ARGV[1])   -- replenishRate (补充速率)
local capacity = tonumber(ARGV[2])-- burstCapacity (最大容量)
local now = tonumber(ARGV[3])    -- 当前时间戳
local requested = tonumber(ARGV[4]) -- 消耗令牌数 (通常是1)

-- 1. 获取桶容量，如果桶不存在，默认填满
local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
  last_tokens = capacity
end

-- 2. 获取上次更新时间，如果不存在，默认为0
local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
  last_refreshed = 0
end

-- 3. 核心：计算时间差，补充令牌
local delta = math.max(0, now - last_refreshed)
local filled_tokens = math.min(capacity, last_tokens + (delta * rate))

-- 4. 判断令牌够不够扣
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens

if allowed then
  -- 如果够扣，执行扣减
  new_tokens = filled_tokens - requested
end

-- 5. 将最新的令牌数和时间戳写回 Redis
redis.call("setex", tokens_key, ttl, new_tokens)
redis.call("setex", timestamp_key, ttl, now)

-- 6. 返回结果给微服务 (1表示放行，0表示拒绝)
return { allowed, new_tokens }
```



---






##### Spring Cloud Gateway的令牌桶算法实现



**`RequestRateLimiterGatewayFilterFactory`**
- 在启动或者配置热更新时，`RequestRateLimiterGatewayFilterFactory`动态创建`RequestRateLimiter`过滤器
- 配置中标注了该过滤器对应的`KeyResolver`的Bean的名字，和该过滤器需要的配置参数
- 读取参数、寻找KeyResolver组件，并组装成带有请求上下文的Filter实例

**`RequestRateLimiter`作为Filter插入过滤器责任链**，它将拦截请求执行以下操作：
- `RequestRateLimiter`调用`KeyResolver`，提取一个`Mono<String>`，这个String作为key用于限流，一个值对应一个独立的桶
  - 如果`KeyResolver`什么都没提取到，SCG设置了`denyEmptyKey=true`，即提取不到key时默认拒绝请求，报 `HTTP 403 Forbidden`
- **将Lua脚本、参数、key提交给Redis**：`ReactiveRedisTemplate.execute(script, keys, args)`
   * Redis 返回 `allowed = true`：Filter 放行
   * Redis 返回 `allowed = false`：Filter 直接中断责任链，向客户端返回 `HTTP 429 Too Many Requests`，并在 Header 中塞入当前的限流指标（如剩余令牌数）。
- **Java如何调用 Lua 脚本？**
  - 在 `RedisRateLimiter` 的源码中，当它拿到 `key-resolver` 提取出的值（比如 `1001`）后
  - 它会在前面加上前缀（默认是 `request_rate_limiter`），拼装成两个真正的 Redis Key：
    * `{request_rate_limiter.1001}.tokens`
    * `{request_rate_limiter.1001}.timestamp`
  - 注意这里的 `{}`，它代表 Hash Tag，保证这两个 Key 落在同一个 Redis Cluster 节点上


**无论成功还是被拒绝，网关都会在 HTTP 响应头中返回类似这样的字段**
- `X-RateLimit-Remaining: 2`（你的桶里还剩 2 个令牌）
- `X-RateLimit-Burst-Capacity: 20 `（你的桶总容量是 20）
- `X-RateLimit-Replenish-Rate: 10` （每秒恢复 10 个）
- **原因**
  - 让前端业务人员主动控制发送请求的频率，减少网关压力
  - **指导重试机制**
    - 开发者可以手动增加`Retry-After: 5`
    - 明确告诉调用方5秒后重试，指导调用方不要无限死循环重试
  - **指导第三方调用者优化代码，主动控制并发量**
---

**`KeyResolver`**
- **限流的业务语义**完全是由 `KeyResolver` 赋予的。传给 Redis 什么 Key，它就在什么维度上限流。
- 需要把不同的 `KeyResolver` 注册为 `@Bean`，单维度限流场景落地如下

**1. IP 限流 (`IpKeyResolver`)**
  - **场景**：防止恶意爬虫单 IP 狂刷接口。
    ```java
    @Bean
    public KeyResolver ipKeyResolver() {
        return exchange -> {
            // 生产级避坑：如果网关前面还有 Nginx，直接用 getRemoteAddress() 拿不到真实客户端 IP，拿到的是 Nginx 的 IP。
            // 必须从 X-Forwarded-For 头部获取。
            String ip = exchange.getRequest().getHeaders().getFirst("X-Forwarded-For");
            if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
                ip = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
            }
            return Mono.just(ip);
        };
    }
    ```

**2. 用户/租户限流 (`UserKeyResolver`)**
   - **场景**：每个账号每秒只能发 5 条弹幕。
    ```java
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            // 假设前置的鉴权 Filter 已经验证过 JWT，并将 userId 放在了请求 Header 中
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            // 也可以从请求参数中拿：exchange.getRequest().getQueryParams().getFirst("userId")
            return Mono.justOrEmpty(userId); // 如果 userId 为 null，返回 Empty
        };
    }
    ```

**3. 接口/路由限流 (`PathKeyResolver`)**
   - **场景**：保护核心高危接口（如提现接口）。
    ```java
    @Bean
    public KeyResolver pathKeyResolver() {
        return exchange -> {
            // 直接获取请求的 URI 路径，例如 "/api/order/create"
            String path = exchange.getRequest().getURI().getPath();
            return Mono.just(path);
        };
    }
    ```

---

**`RedisRateLimiter` 绑定配置参数和`KeyResolver`**
- 在 application.yml 中配置

    ```yaml
    spring:
    cloud:
        gateway:
        routes:
            - id: order_route
            uri: lb://order-service
            predicates:
                - Path=/api/order/**
            filters:
                - name: RequestRateLimiter
                args:
                    # 1. 令牌填充速度 (QPS)
                    redis-rate-limiter.replenishRate: 10
                    # 2. 桶的总容量 (允许的突发流量)
                    redis-rate-limiter.burstCapacity: 20
                    # 3. 每次请求消耗的令牌数 (不写默认就是 1)
                    redis-rate-limiter.requestedTokens: 1
                    # 4. 重点！！告诉网关使用哪个 Bean 来提取特征。
                    # 注意这里的 SpEL 表达式语法：#{@beanName}
                    key-resolver: "#{@userKeyResolver}" 
    ```



---


**为什么要结合配置中心实现热更新？**
- 限流参数（`replenishRate`）绝不能写死在代码或本地 yml 里。双十一来了，需要把订单接口的 QPS 从 100 调到 5000，如果你还得重启网关集群，系统早崩溃了。

1. 把上面那段 YAML 配置，从本地的 `application.yml` 挪到 Nacos 或 Apollo 的控制台中（通常命名为 `gateway-routes.yaml`）。
2. 在网关工程中引入 Nacos Config 依赖。
3. **底层核心逻辑**：当你在 Nacos 控制台修改了 `replenishRate` 为 5000 并点击发布。Nacos 客户端会监听到配置变更。SCG 内部监听了 Spring 的 `EnvironmentChangeEvent` 机制，会触发一个叫做 `RefreshRoutesEvent` 的事件。
4. **无缝热更新**：网关在不重启的情况下，会自动销毁旧的路由对象，重新根据 Nacos 里的新配置构建路由和 `RequestRateLimiter`，下一秒进入网关的请求，就会按照 5000 的速率执行新的限流策略。

---




##### 组合限流



这是你提到最复杂的部分。原生 SCG 针对同一个路由通常只能配置一个 `KeyResolver`。当我们需要“全局 IP 限流 + 特定 API 接口限流 + VIP 用户不限流”时，该怎么做？

* **方案一：原生扩展（Filter 链式叠加）**
    * **实现方式**：在路由配置中，配置多个 `RequestRateLimiter`，每个分配不同的 `KeyResolver` 和 Redis key 前缀。
    * **学习重点**：理解 Filter 的执行顺序（Order），处理多重限流的短路逻辑。
* **方案二：深度定制（自定义 RateLimiter）**
    * **实现方式**：实现自定义的 `RateLimiter` 接口，替代默认的 `RedisRateLimiter`。
    * **学习重点**：在自定义逻辑中，同时校验多个维度，并重写 Lua 脚本以支持多维度的原子扣减。
* **方案三：降维打击（引入专业流控组件 Sentinel - 强烈推荐）**
    * **实现方式**：放弃 SCG 自带的 Redis 限流，整合 **Alibaba Sentinel**。
    * **学习重点**：学习 `GatewayFlowRule`。Sentinel 天生支持按 API 分组、按参数（Header/URL 参数）等极其复杂的组合限流策略，并且自带可视化的控制台，是目前企业级微服务网关限流的主流选择。

