##### 组合限流

这是一个要求极度严苛的挑战。既然你要求“堵上你的嘴，让你没有任何提问的机会”，那我就必须把**理论、误区、代码实现、底层暗坑以及最终的架构演进**一次性全部给你打通。

我们直接开始这场没有任何死角的深度剖析。

---

### 第一问：有了 Sentinel，组合限流还用学吗？

**直接给你结论：生产环境中，你绝对会用 Sentinel 来做组合限流；但在学习阶段，你必须手撸一遍 SCG 的组合限流。**

为什么？因为 SCG 的原生 `RequestRateLimiter` 在设计上存在一个“天然的局限性”（一个 Filter 只能接收一个 KeyResolver）。学习如何在 SCG 中实现组合限流，本质上是在学习**如何突破框架自身的限制，如何深度操控 Filter 责任链的执行顺序，以及如何规避 Redis Key 的冲突**。
搞懂了这个，你才算真正吃透了网关的核心底座，以后就算不用 Sentinel，让你自己写一个网关限流插件你也能游刃有余。

---

### 第二问（核心避坑）：你说的“组合”，到底是哪种组合？

这是 90% 的人死在组合限流上的第一关。在动手写代码前，你必须先明确你的业务需求属于下面哪一种，因为它们的实现方案**完全相反**：

#### ➡️ 场景 A：复合特征限流（AND 逻辑，捆绑判定）
* **需求**：同一个 IP（如 192.168.1.1）下的同一个 UserID（如 1001），每秒只能请求 5 次。
* **特点**：如果这个 IP 换了账号，或者这个账号换了 IP，**限流重新计算**。这通常用于防范刷单黑客使用肉鸡 IP 不停切换账号。
* **核心思路**：**把多个特征拼成一个字符串**，作为一个全新的 Key 扔给 Redis。

#### ➡️ 场景 B：多维度独立限流（OR 逻辑，多重关卡）
* **需求**：网关需要同时满足：① 全局 IP 限流（每个 IP 最多 100 QPS） **且** ② 针对核心 API 接口限流（如 `/api/pay` 最多 500 QPS） **且** ③ 针对普通用户限流（每秒 10 次）。
* **特点**：这三个条件是**独立**的。只要触碰了任何一条红线（比如 IP 还没到 100，但用户请求已经到了 11 次），请求立马被干掉。
* **核心思路**：**配置多个独立的限流 Filter 形成责任链**，一层一层剥洋葱。

---

### 第三步：场景 A（复合特征限流）的完整落地

这个最简单，不需要改 YAML，只需要写一个新的 `KeyResolver`。

**完整代码实现：**
```java
@Configuration
public class RateLimitConfig {

    @Bean
    public KeyResolver ipAndUserKeyResolver() {
        return exchange -> {
            // 1. 提取 IP (注意之前说的从 X-Forwarded-For 提取的逻辑，这里简写)
            String ip = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
            
            // 2. 提取 UserID
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            if (userId == null) {
                userId = "anonymous"; // 兜底，防止因为没登录直接报 403
            }
            
            // 3. 核心：将它们拼接成一个复合 Key
            // 生成结果例如："192.168.1.1_1001"
            return Mono.just(ip + "_" + userId);
        };
    }
}
```
**底层执行揭秘：**
Redis 底层会生成类似 `{request_rate_limiter.192.168.1.1_1001}.tokens` 的 Key。只要 IP 和 UserID 不变，扣减的就是同一个桶。没有任何提问空间，这就是纯粹的字符串拼接策略。

---

### 第四步：场景 B（多维度独立限流）的完整落地与暗坑

这是真正的深水区。既然一个 `RequestRateLimiter` 只能配一个 `KeyResolver`，那我就在路由下面配置多个 Filter！

**正确的 YAML 配置姿势：**
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: complex_route
          uri: lb://complex-service
          predicates:
            - Path=/api/complex/**
          filters:
            # 第一道关卡：先防野蛮人（IP 限流）
            - name: RequestRateLimiter
              args:
                key-resolver: "#{@ipKeyResolver}"
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
            
            # 第二道关卡：再防业务刷子（用户限流）
            - name: RequestRateLimiter
              args:
                key-resolver: "#{@userKeyResolver}"
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 10
```

**你脑海中可能冒出的 3 个疑问，我直接全部回答：**

* **疑问 1：这两个 Filter 的执行顺序是怎样的？如果第一个限流了，第二个还会执行吗？**
    * **回答**：从上到下按 YAML 里的顺序严格执行。如果在第一关（IP 限流）发现桶空了，Filter 责任链会**立刻短路（Short-circuit）**，直接向客户端返回 `HTTP 429`。第二关（用户限流）的代码根本连碰都不会碰，更不会去请求 Redis。这极其节省性能。

* **疑问 2：既然如此，我应该把哪个限流放在前面？**
    * **回答**：**把粒度最粗、拦截概率最高、计算代价最小的放在最前面。** 绝对应该先拦截 IP（防 DDOS），再拦截 API（防服务击穿），最后拦截 User（防业务违规）。如果你把用户限流放前面，遇到黑客用同一个账户疯狂发请求，网关还要每次去解析 Header 提取 UserID，白白浪费 CPU。

* **疑问 3（致命暗坑）：如果用户的 ID 恰好和某个 IP 地址的字符串长得一样，Redis 里的 Key 会不会冲突？**
    * **回答**：**会！这就是原生 SCG 的最大设计缺陷！**
    * 原生的 `RedisRateLimiter` 默认使用的 Redis Key 前缀固定是 `request_rate_limiter`。
    * 假设 IP 是 `10.0.0.1`，生成的 Key 是 `{request_rate_limiter.10.0.0.1}.tokens`。
    * 假设有个奇葩用户的 UserID 也是 `10.0.0.1`，生成的 Key 也是 `{request_rate_limiter.10.0.0.1}.tokens`。
    * **结果**：这两个本来毫不相干的限流维度，在 Redis 里共用了一个令牌桶，导致限流完全错乱！

**如何完美填上这个暗坑（终极解决方案）？**
你必须在写 `KeyResolver` 时，手动加上维度前缀强制隔离！
```java
// IP 限流器
@Bean
public KeyResolver ipKeyResolver() {
    return exchange -> Mono.just("IP:" + exchange.getRequest().getRemoteAddress().getAddress().getHostAddress());
}

// 用户限流器
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> Mono.just("USER:" + exchange.getRequest().getHeaders().getFirst("X-User-Id"));
}
```
这样到了 Redis 里，Key 就变成了 `{request_rate_limiter.IP:10.0.0.1}.tokens` 和 `{request_rate_limiter.USER:1001}.tokens`，物理隔离，完美无缺。

---

### 最终章：为什么 Sentinel 能对原生方案进行“降维打击”？

你坚持看完了上面那些弯弯绕绕的 YAML 链条、前缀拼接和执行顺序。现在，我告诉你为什么微服务架构师最终都会选择 Sentinel：

因为你上面辛辛苦苦搞的一大堆类和配置，在 Sentinel 中，仅仅是一段极其优雅、不需要重启、所见即所得的配置。

**在 Sentinel 中实现相同的“组合独立限流”，你只需要在控制台配置三条独立规则：**
1.  **规则 1**：针对 `API=/api/complex`，阈值 500。
2.  **规则 2**：针对 `API=/api/complex` 且提取 `Header[X-User-Id]`，阈值 10。
3.  **规则 3**：针对 `API=/api/complex` 且提取来源 IP，阈值 100。

**Sentinel 底层帮你做了什么？**
* 它**自带了请求参数/Header解析器**，你连 `KeyResolver` 的代码都不用写。
* 它在内存中构建了一棵名为 **Node 树** 的统计结构，不同维度的统计数据天然隔离，绝对不会出现 Redis Key 冲突那种低级错误。
* 如果三种规则同时触发，Sentinel 内部会自动走快速失败（Fast-Fail）逻辑，统一抛出 `BlockException`，由你的降级回调函数统一处理返回结果。

**总结陈词：**
你今天学原生 SCG 组合限流，是为了**知晓网关过滤器链条的运作规律和 Key 隔离的底层逻辑**。当你掌握了这些，你面对 Sentinel 时，看到的就不再是一个黑盒工具，而是一个为了解决你刚刚经历过的那些“痛点”而设计出的完美架构产物。

至此，关于网关限流的理论、实战、避坑与架构选型，已经形成绝对闭环。没有任何遗漏。