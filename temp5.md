### Nacos

#### 领域模型

**Namespace**
- 最高级别的**物理隔离**：不同 Namespace 之间的数据、服务完全不可见
- 企业需要分配有限的Nacos集群算力
  - 根据环境分配：`dev`、`test`、`prod`，不同环境下的运行实例绝对无法调用彼此
  - 根据租户分配：B端将一套系统租给多个客户，需要物理隔离它们的服务
- **必须先在Nacos的控制台新建namespace**，然后Nacos 会为每个 Namespace 生成一个**全局唯一的 UUID**
- 在SpringBoot项目配置中，**必须填UUID，不能填名称**
    ```yaml
    spring:
      cloud:
        nacos:
          discovery:
            server-addr: 127.0.0.1:8848
            # 将当前服务死死锁定在 dev 环境的隔离墙内
            namespace: 7b2a2b4a-5c6d-7e8f-9a0b-1c2d3e4f5a6b 
    ```
---


**Group**
- **同一个Namespace逻辑划分为多个Group**
- 如果不配置，Nacos 默认把所有服务塞进 `DEFAULT_GROUP`
- **多业务线 / 灰度压测**
    * **场景A（多业务线）**：在一个大环境（比如 prod）下，公司有“电商事业部”和“游戏事业部”，两边的服务可能重名。电商配置 `group: E_COMMERCE`，游戏配置 `group: GAME`，互不干扰。
    * **场景B（全链路压测）**：双十一前要在生产环境做真实流量压测。为了不影响真实用户，我们会启动一批特殊的“压测实例”，并将它们的组名设为 `group: SHADOW_GROUP`（影子组）。压测流量只会在影子组内部流转。
    ```yaml
    spring:
      cloud:
        nacos:
          discovery:
            group: SHADOW_GROUP # 更改分组名称
    ```

---

**Service**
- Nacos根据Service的名字，也就是**微服务的名字**来识别微服务
- 为每个微服务生成一个**DataId**，即该微服务的**配置文件的全名**，微服务启动时获取DataId来查找自己的配置文件
  - `DataId =${spring.application.name}-${spring.profiles.active}.${file-extension}`（file-extension文件扩展名）
- 同一个 Namespace + 同一个 Group 下，Service 名称**必须唯一**。
    ```yaml
    spring:
      application:
        name: order-service # 这就是 Service 的名称
    ```
---

**Cluster**
- Cluster集群是**运行同一个Service的实例集合**，按照**物理位置划分**
- **异地多活与同城双活**
  - **就近路由**：采用负载均衡策略，nacos分发请求给就近集群
  - 容灾：集群失效后，nacos才将请求跨区分发给其他地方的集群
    ```yaml
    spring:
      cloud:
        nacos:
          discovery:
            cluster-name: GZ_ZONE # 标记当前实例所属的物理集群
    ```

---

#### 实例类型


CAP定理：在分布式系统中，一致性、可用性和分区容错性三者最多只能同时满足两项，无法兼顾三者。
- **AP架构**：满足**可用性、容错性**的架构，
- **CP架构**：满足**一致性、容错性**的架构

**临时实例，AP 架构，默认配置**

* **底层机制（主动上报）**：临时实例启动后，必须**主动**每隔 5 秒向 Nacos 发送一个心跳包（Ping），告诉 Nacos “我还活着”。
* **生命周期（无情淘汰）**：
    * 如果 Nacos **15秒** 没收到心跳，会把该实例标记为**不健康**。
    * 如果 Nacos **30秒** 没收到心跳，Nacos 会认为这台机器已经彻底死机，直接将其从注册表中**剔除（删除）**。
* **CAP 模型**：采用 **AP（高可用性）** 模型，底层通过 Nacos 自研的 Distro 协议同步数据。
* **企业级应用场景**：
    * **绝大多数的 Spring Boot 微服务**
    * 适应云原生时代（K8s、Docker）频繁的弹性扩缩容。机器随时可能被销毁或新建，死了就赶紧踢掉，不要影响流量。

---

**持久化实例，CP 架构**

* **底层机制（服务端探测）**：持久化实例**不需要**自己发心跳。相反，是 Nacos 服务端**主动**发起网络请求（TCP/HTTP）去探测这个实例的端口通不通。
* **生命周期（死不注销）**：
    * 如果 Nacos 探测失败，只会把该实例标记为**不健康**。
    * **绝对不会自动剔除！** 哪怕这台机器断电一年，它的名字依然会挂在 Nacos 的注册表里，除非运维人员通过 API 或控制台**手动删除**。
* **CAP 模型**：采用 **CP（强一致性）** 模型，底层通过 Raft 选举协议保证集群数据绝对一致，数据会被持久化到磁盘上。
* **企业级应用场景**：
    * **基础设施与中间件**：比如把 MySQL 数据库、Redis 集群作为服务注册到 Nacos 供别人发现。数据库宕机只是暂时的，重启后 IP 不变，不需要也不应该被踢出通讯录。

---

| 维度 | 临时实例 (Ephemeral = true) | 持久化实例 (Ephemeral = false) |
| :--- | :--- | :--- |
| **健康检查方式** | 客户端主动发送心跳 | 服务端主动探测客户端端口 |
| **异常处理** | 30秒后自动剔除 | 仅标记为不健康，永久保留 |
| **数据存储** | 内存中，不落盘 | 磁盘持久化（Raft 协议同步） |
| **适用对象** | 普通微服务进程（Java/Go 等） | 数据库、第三方服务、非标老系统 |

---

**代码配置方式：**

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        # true 为临时实例（默认），false 为持久化实例
        ephemeral: false 
```

---




#### 服务交互流程


**服务与角色**
- **服务**：一个SpringBoot项目打包成Jar包，运行后的一个Java进程
- 当服务需要调用其他服务时，Nacos Server**只负责提供其他服务的联络方式**，**不负责转发请求**，该服务拿到目标服务的位置后，自己发起HTTP请求
- `Provider`/`Consumer` ：服务的**逻辑角色**：它们是发生一次具体的网络调用时，双方扮演的角色
  - `Provider`：会被调用的微服务启动时，需要把自己的IP、端口、服务名等元数据交给 Nacos，即服务注册
  - `Consumer`：需要调用其他微服务时，从Nacos拉取服务列表
  - `Nacos Server`：独立的服务，记录服务的元数据，并在**服务列表、配置变动**时**主动通知注册的服务**、


##### **服务注册**

Provider获取**服务名、IP 地址、端口号、集群名称、权重等元数据**，向Nacos发起**gRPC 长连接**，并提交自己的元数据，在nacos中完成服务注册，注册完成后，保持**gRPC 长连接**

**第一步**：该SpringBoot服务的**项目依赖**中，必须引入**服务发现依赖**，即
  ```XML
  <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-loadbalancer</artifactId>
  </dependency>
  ```

**第二步：`application.yml`项目配置**：Nacos需要的元数据
```YML
server:
  port: 8081 # Provider 的端口

spring:
  application:
    name: order-provider # 微服务名称（Service 级别名片）
  cloud:
    nacos:
      discovery:
        # ==================== 1. 核心连接与安全配置 ====================
        # 测试环境写单机端口
        server-addr: 127.0.0.1:8848
        # Nacos 集群地址。企业级绝对不会写单机 IP，通常是内网域名或 Nginx 代理的 VIP（虚拟IP）
        server-addr: nacos-cluster.yourdomain.internal:8848 
        # Auth 鉴权机制，Nacos默认打开，防止别人拿 IP 直接连入控制台或调用 OpenAPI
        username: nacos
        password: Jt5aEAEfICll
        
        # ==================== 2. 领域模型与隔离配置====================
        # 命名空间 ID（环境隔离）。生产绝对不用默认的 public，这里填 Nacos 控制台生成的 UUID
        namespace: 3b1a2c3d-4e5f-6g7h-8i9j-0k1l2m3n4o5p 
        # 服务分组（业务隔离）。默认 DEFAULT_GROUP，企业级按业务大线划分
        group: E_COMMERCE_GROUP 
        # 集群名称（机房容灾）。实现就近路由（如同城双活下的广州机房 A 区）
        cluster-name: GZ_ZONE_A 

        # ==================== 3. 实例元数据与服务治理 ====================
        # true 表示临时实例（AP 架构），心跳断开即剔除；微服务绝大多数场景为 true
        ephemeral: true 
        # 实例权重（0.0 ~ 100.0）。用于平滑上线、机器性能不均时的流量调配，或灰度发布
        weight: 1.0 
        # 是否注册到nacos。上一节讨论的核心点，如果是纯 Consumer 可改为 false
        register-enabled: true 
        # 【核心】自定义元数据集合。运维大盘监控和自定义路由策略全靠这个
        metadata:
          version: v2.1.0        # 用于基于版本的灰度发布（如流量只打给 v2 版本）
          env: staging           # 标记为预发环境
          owner: zhangsan        # 负责人标记，出问题直接告警给对应人

        # ==================== 4. 容器化与复杂网络适配 (Docker / K8s) ====================
        # 微服务部署在 Docker 容器或多网卡物理机时，极易将内网 172.x 或 127.0.0.1 注册上去，导致其他机器调不通。
        # 强制指定网卡名称（指定用哪张网卡的 IP 去注册）
        # network-interface: eth0 
        # 强制指定向 Nacos 注册的具体 IP（通常在跨网络隔离或宿主机映射时手动指定）
        # ip: 192.168.100.15 
        # 强制指定注册的端口（通常在使用 Docker 端口映射 -p 8080:80 时配置）
        # port: 8080
```

**第三步：在启动类上添加`@EnableDiscoveryClient`注解**
- **Spring Cloud提供`@EnableDiscoveryClient`注解**：开启服务注册、服务发现能力，并在应用启动后自动将自己登记到注册中心，以Nacos为例

- **激活自动注册**
  - 读取`application.yml`中的`spring.application.name`和`spring.application.cloud`配置
  - 通过`ServiceRegistry` 接口向Nacos 发送gRPC 请求完成服务注册
  - 如果不想注册该服务，可以通过`spring.cloud.nacos.discovery.register-enabled: false`配置关闭

- **注入发现客户端**
  - 注入发现客户端`DiscoveryClient`后，服务拥有发现Nacos的能力，连带拥有从Nacos拉取服务列表的能力
  - 注入发现客户端是服务注册和开启生命周期管理的前提

- **开启生命周期管理**，**实现微服务优雅上下线**
  * **启动时**：等所有 Bean 初始化完成、Web 环境准备好后再注册，确保别人调你时你已经准备好了。
  * **关闭时**：当 JVM 停止前，它会主动向 Nacos 发送一个“下线请求”，告诉 Nacos：“我要收摊了，别再让别人调我了”，而不是等 30 秒心跳超时。



##### 心跳维持、TCP、gRPC

临时实例的心跳，本质上是 **Nacos 客户端中运行的一个定时任务线程**。
- 当你开启了 `@EnableDiscoveryClient` 并在 Nacos 注册成功后，Nacos 客户端 SDK 会在后台启动一个名为 `BeatReactor` 的线程池。
* 它会严格按照设定好的间隔时间（默认 5000 毫秒），向 Nacos Server 发送一个包含自身运行状态的 JSON 数据包。
* Nacos Server 收到后，会更新内存中这个实例的 `lastBeatTime`（最后一次心跳时间）。
* 同时，Nacos Server 后台也有一个定时任务（每 5 秒执行一次），它会去遍历所有实例的 `lastBeatTime`。
    * 如果发现 `当前时间 - lastBeatTime > 15秒`，把该实例的 `healthy` 字段标为 `false`。
    * 如果发现 `当前时间 - lastBeatTime > 30秒`，直接调用注销接口，把该实例从内存注册表中 `remove` 掉。


**如何自定义这三个时间？**
- 默认的 5/15/30 秒在绝大多数场景下是合理的。但在某些**对网络抖动极度敏感**（希望 1 秒内感知掉线），或者**弱网环境**（希望 60 秒不发心跳也不被踢）的场景下，你需要修改它。
- 在 Spring Cloud Nacos 中，修改心跳参数并不是直接写在最外层，而是通过 **实例元数据（metadata）** 传递给 Nacos Server 的：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        metadata:
          # 自定义心跳间隔，这里改为 3 秒 (单位: 毫秒)
          preserved.heart.beat.interval: 3000 
          # 自定义不健康阈值，改为 10 秒
          preserved.heart.beat.timeout: 10000 
          # 自定义剔除阈值，改为 20 秒
          preserved.ip.delete.timeout: 20000  
```

---


**持久化实例的探活**

- Nacos Server 支持两种主流的健康检查（探活）机制：
  * **TCP 探测**：Nacos Server 会尝试和持久化实例的 IP 和端口建立一次 TCP 握手。如果握手成功（能 `ping` 通端口），说明存活；如果连接被拒绝（Connection Refused），说明服务宕机。
  * **HTTP 探测**：更高级一点，Nacos Server 会向你配置的一个特定 URL 发送 HTTP GET 请求。如果返回状态码是 200，就算健康；如果是 500 或者请求超时，就算不健康。

- **为什么需要这种机制？**
  - 在真实的微服务集群中，不是所有服务都是 Java 写的，也不是所有服务都能引入 Nacos 客户端依赖！
  - 比如：**MySQL 数据库、Redis 节点、或者是古老的 C++ 遗留系统**。
  - 你可以通过 Nacos 开放平台 API，强行把这些第三方服务作为“持久化实例”注册到 Nacos 里。
  - 它们不会主动发心跳，Nacos 只能通过 TCP 探测它们对应的 3306 端口或 6379 端口，以此来判断它们是否存活，并供其他微服务发现。

---

**Nacos 1.x 与 2.x 的心跳协议演进**

* **在 Nacos 1.x 时代（HTTP 时代）**：
  - 心跳是一个真实的 **HTTP POST 请求**。
  - 如果你的集群有 1000 个微服务，每 5 秒发一次 HTTP 请求，Nacos Server 每秒要承受几百个 HTTP 并发，网络开销和解析开销极其巨大，容易引发性能瓶颈。
* **在 Nacos 2.x 时代（gRPC 时代）**：
  - Nacos 引入了 **gRPC 长连接**。客户端启动后，直接和 Nacos Server 建立一条 TCP 长连接通道。
  - 心跳不再是笨重的 HTTP 请求，而是复用了 gRPC 底层极度轻量级的 **KeepAlive 机制**（发送极其微小的控制帧 ping/pong）。这使得 Nacos 2.x 的性能比 1.x 提升了十倍以上。

---

**为什么有了gRPC 长连接，还需要心跳检测？**
- **进程暴毙** vs **内核暴毙**
  - **进程暴毙**：操作系统介入回收该进程的所有资源，包括gRPC连接（建立在TCP连接之上），主动向对Nacos Server发送 RST 或 FIN 数据包，通知Nacos进程已死
  - **内核暴毙**：断电、断网时，操作系统暴死，但是Nacos Server仍然维护该TCP连接的状态描述符，即**TCP 半打开状态**
- **心跳检测的作用**：检测网络连接是否中断，检测应用能否跑业务
  - **防范物理机断电**、断网导致 TCP 假死
  - **防范植物人进程**：操作系统正常，gRPC 连接稳固，但微服务已死亡，比如数据库连接池耗尽导致工作线程全部死锁，植物人无法处理业务，自然也无法发送心跳包
  - **链路保活**：
    - 在真实的机房架构中，Consumer 和 Nacos 之间往往隔着防火墙、NAT 网关或负载均衡器。
    - 这些中间设备为了节省资源，通常会有一个强制策略：**如果一条 TCP 连接长时间没有数据传输，防火墙会直接将其单方面掐断，且不会通知通信双方**。
    - 心跳的另一个作用就是**充当 Keep-Alive 探针**，定时在长连接通道里发声，告诉中间设备“这条通道还在使用”，从而防止长连接被意外阻断。

---

**TCP 半打开状态**
- TCP是一个极其严谨的**状态机**，从三次握手到四次挥手，都需要**明确的控制报文来驱动**
  - 正常情况下进程结束，操作系统会发出带 FIN 标志位的报文，告诉对方：“我要关了”。
  - 遇到异常，发出带 RST（Reset）标志位的报文：“出错了，强制重置”。
  - 当发生断电、断网线、系统内核崩溃时，机器瞬间暴毙。它连发遗言（FIN/RST）的机会都没有。既然没收到合法的状态切换指令，B 机器的 TCP 协议栈就必须、也只能死死守住 ESTABLISHED 状态
- TCP 诞生时，网络线路极其不稳定，经常发生拥塞或短暂断连。因此，TCP 的设计哲学是“**没有消息，就是最好的消息**”
  - 静默是金： 如果双方都没有数据要发送，TCP 通道上是绝对安静的，一个字节都不会传。
  - 网络总是会好的： 如果一条连接长时间没有数据交互，TCP 协议栈默认认为“对方只是现在没话跟我说”，或者是“中间的路由器太堵了”，而不是认为“对方死机了”。它会极其耐心地等下去。
- TCP Keepalive：
  - TCP Keepalive是操作系统级别的保活，但是没啥用，甚至很多系统的 TCP Keepalive 默认是关闭的
  - 就算开启了，Linux 系统默认的 Keepalive 探测时间是 7200 秒
- **TCP 半打开状态**唯一的解决办法：**应用层主动制造数据交互，即心跳包**

---

**gRPC协议**
- Google 开源的一套 RPC框架，Remote Procedure Call/远程过程调用
- 传统的 RESTful API 大多基于 HTTP/1.1，它有一个致命弱点：单行道、一来一回。发一个请求，必须等响应回来了，这个连接才能发下一个请求
- 基于 **HTTP/2 协议**：**运行在TCP协议上的协议**
  - **多路复用**（Multiplexing）： 在一条物理 TCP 连接上，可以同时并行发送成百上千个请求和响应，互不干扰。再也不用排队了。

  - **双向流**（Bi-directional Streaming）： 这是 Nacos 2.x 抛弃 UDP 的核心原因。客户端不仅可以向服务端发请求，服务端也能顺着这条已经建好的连接，主动把数据推给客户端。

  - **二进制分帧**： HTTP/2 会把请求头和请求体切分成极小的二进制帧来传输，解析速度远超传统的纯文本 HTTP。
- **Protobuf**：极致压缩的**二进制数据**，替代JSon，

---


**对外 RESTful，对内 gRPC**

**`@FeignClient`即OpenFeign，基于HTTP/1.1和JSON**
- **对外的网关接口与前端交互**必须使用`@FeignClient`，不可能让前端去解析gRPC协议
- 注重“人类可读性”与快速调试的业务：gRPC抓包结果全是二进制
- **并发量适中、非核心链路的常规 CRUD**，`@FeignClient`+线程池完全能胜任

**gRPC业务场景**
- **内部核心高频调用链路**：可以使用Dubbo而不是`@FeignClient`
- **大规模数据流传输**：音视频流的信令控制、海量日志的实时收集、大文件的分块上传
- **跨语言微服务团队**：通过.proto 契约文件生成不同语言的调用代码，强制所有人遵守规范，消灭沟通成本。


---
##### 服务发现与动态维护

Consumer通过**拉取 - 缓存 - 订阅推送**获取 Provider的真实地址
- **通过本地缓存服务列表，同时做到了低延迟和高可用**

**首次调用与主动拉取**
- 当 Consumer 启动并准备向目标服务发起第一次远程调用时，它只知道目标服务的服务名，却不知道具体的网络地址。
* 此时，Consumer 会向 Nacos Server 发起一次主动查询请求。
* Nacos 接收到请求后，会在注册表中查找该服务名下所有**健康可用**的实例节点，并将其以“IP + 端口”列表的形式返回给 Consumer。

**本地缓存与容灾兜底**
- Consumer 拿到实例列表后，**绝对不会**在每次发起调用时都去请求 Nacos（否则 Nacos 会被瞬间打挂），而是会将这份列表**缓存在本地进程的内存中**。
* **容灾保命机制：** 这个本地缓存是微服务高可用的核心保障。即使此时 Nacos 集群遭遇史诗级故障全部宕机，Consumer 依然可以凭借本地缓存的通信录，继续与现存的 Provider 实例进行稳定的点对点调用。**“注册中心瘫痪，不影响现有微服务之间的正常通信”**，这就是本地缓存的兜底价值。


**订阅监听与动态推送**
- 既然通讯录被缓存在了本地，那么如果 Provider 突然扩容了新机器，或者某台机器宕机了，Consumer 怎么才能知道呢？
* **建立订阅关系：** 在 Consumer 第一次拉取列表时，底层会同时向 Nacos 注册一个**订阅者（Subscriber）**身份，建立起监听机制。
* **服务端主动推送：** 一旦 Nacos Server 察觉到该服务的实例列表发生变化（例如由于心跳超时剔除了宕机节点，或者有新节点注册进来），Nacos 会**立即通过gRPC将最新的服务列表主动推送（Push）给所有订阅了该服务的 Consumer**。
* **动态更新：** Consumer 监听到推送事件后，会迅速刷新本地的内存缓存。这样一来，后续的请求就会自动避开宕机的节点，或者平滑地分流到新扩容的节点上，实现近乎实时的动态路由。

---




##### 负载均衡与Nacos权重

**`@FeignClient`注解基于负载均衡器 `LoadBalancer`和`DiscoveryClient`实现**
- 服务需要调用其他服务时，使用`@FeignClient`接口类说明要调用的目标服务、目标方法
- `@FeignClient`注解采用传统的 `HTTP/1.1` + `JSON`调用其他服务，而不是gRPC协议
  ```JAVA
  @FeignClient(value = "seckill-stock") // 重点：这里的 value 必须和库存服务在 Nacos 里的名字一模一样！
  public interface StockClient {

      /**
       * 这里的方法签名，必须和你要调用的库存服务的接口一模一样
        * 就像是把库存服务的 Controller 里的方法复制过来一样
        */
      @GetMapping("/stock/test")
      Result<String> test();
  }
  ```

**`@FeignClient`方法调用过程**
1. Feign将test请求交给负载均衡器LoadBalancer
2. 负载均衡器需要从集群选择具体的运行实例，调用DiscoveryClient
3. DiscoveryClient从Nacos拉取服务的实例集合，并**返回目标运行实例集合**；如果本地已缓存了服务实例则不再拉取
4. 负载均衡器根据策略（**轮询、随机、权重等**）挑选一个运行实例地址交给Feign
5. Feign拼凑最终的URL发起HTTP调用，访问目标Provider

如果Consumer本地存储了目标Service的Provider，则无需访问Nacos


---

**Spring Cloud LoadBalancer**
- Spring官方推出的负载均衡器，取代已停止维护的Ribbon

**Nacos 权重**
- 默认策略是轮询，平均分配流量给每个实例所寄生的机器
- 同一个服务的不同实例，权重可以设置 0.0 ~ 100.0的合法值，
  - 为三个实例分别设置：80、10、10的权重
  - 则100个请求有80个精准交给第一个机器

---

**负载均衡器**
- 负载均衡器在选中目标实例的过程中，采用二级筛选机制：**同集群优先**
  - 先筛选Cluster集群，**优先筛选相同 Cluster中的实例**，被筛选出来的实例是**同一个Service下的运行实例**
  - 如果同集群内没有任何可用实例，跨集群去调用其他机房的实例
  - **如果同集群内有实例，权重在这些实例构成的集合中生效，严格按照配置的权重比例去瓜分流量**
- 可以在项目中配置权重，**但是一个Service下的运行实例都会配置为相同的权重**
    ```YML
  spring:
    application:
      name: order-provider
    cloud:
      nacos:
        discovery:
          server-addr: nacos-cluster.yourdomain.internal:8848 
          username: nacos
          password: Jt5aEAEfICll
          weight: 1.0 
  ```
- **在Nacos控制台中手动调整实例的权重**
  - 如果项目中配置了权重，则服务重启时会覆盖掉控制台修改的权重
- 绝大多数情况下，**不要在项目的配置文件里去配 weight 属性。让它保持默认的 1**
- **在控制台将权重设置为0，让服务优雅下线**


**Nacos 控制台修改了权重，但Spring Cloud LoadBalancer默认不会生效**
- **必须将Spring Cloud LoadBalancer的轮询引擎替换成 Nacos 提供的权重引擎**

---

**全局替换引擎**：**所有服务**都将Spring Cloud LoadBalancer的轮询引擎替换成 Nacos 提供的权重引擎
- `@LoadBalancerClients` ：开启全局配置，注意比局部替换使用的`@LoadBalancerClient`**多了一个s**

```JAVA
@SpringBootApplication
@EnableFeignClients
// 重点在这里：加了 s，并且使用 defaultConfiguration
@LoadBalancerClients(defaultConfiguration = NacosWeightLoadBalancerConfig.class)
public class ConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}
```
---

**局部替换引擎**：**指定的服务**将Spring Cloud LoadBalancer的轮询引擎替换成 Nacos 提供的权重引擎
- **该类不能加 @Configuration 或 @Component，防止被全局扫描到**
  ```java
  public class NacosWeightLoadBalancerConfig {

      @Bean
      public ReactorLoadBalancer<ServiceInstance> reactorServiceInstanceLoadBalancer(
              Environment environment,
              LoadBalancerClientFactory loadBalancerClientFactory) {
          // 获取你要调用的微服务名称
          String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
          // 返回 Nacos 官方提供的支持权重的负载均衡器！取代默认的轮询！
          return new NacosLoadBalancer(loadBalancerClientFactory
                  .getLazyProvider(name, ServiceInstanceListSupplier.class), name);
      }
  }
  ```

- **在 Consumer 端告诉 Feign 使用这个新引擎**
  ```java
  // name = 你要调用的服务名
  // configuration = 刚才写的那个权重配置类
  @FeignClient(name = "order-provider")
  @LoadBalancerClient(name = "order-provider", configuration = NacosWeightLoadBalancerConfig.class)
  public interface OrderFeignClient {
      // ... 你的接口方法
  }
  ```



---

##### 优雅上下线

**优雅上线**
- 准备好服务后再接收流量，避免新实例刚启动还没连接好数据库就被大流量打挂
- 优雅上线：**给新启动的实例一点缓冲时间，让它加载缓存、建立连接池，然后再承受大流量**
- 部署新实例时：以较小的权重，比如0.1，启动新实例，不要以默认权重启动
  - 有很多种方法，这里只列出Java启动命令`java -jar seckill-order-1.0.0.jar --spring.cloud.nacos.discovery.weight=0.1`
- 接受少量流量，完成预热，比如JIT 编译器完成编译，各种连接池建立完毕后，再通过 Nacos 控制台动态将权重调回 `1`



**优雅下线**
- 直接 `kill -9` 杀掉进程时，流量可能命中已下线的实例，正在处理的请求也直接失败
- 优雅下线：**先切断外部流量入口，等正在处理的内部请求执行完毕后，再真正关闭进程。**
1. **从 Nacos 摘除流量**
   - 通过 Nacos 控制台将实例状态改为下线，或者将其权重调整为 `0`
   - Nacos 服务端将这个变更推送给所有调用方
2. **等待客户端刷新缓存**
   - 调用方本地是有服务列表缓存的，通常几秒钟刷新一次
   - 在这个时间差内，仍然会有残余流量打过来
   - 摘除流量后，通常需要**等待一段时间**，确保所有调用方的缓存都已更新，不再向该实例发送新请求。
3. **应用层优雅停机**
   - Spring Boot 中，可以开启内置的优雅停机功能
      ```yaml
      server:
        shutdown: graceful
      spring:
        lifecycle:
          timeout-per-shutdown-phase: 30s # 设置最大等待时间
      ```
   - 向应用发送 `SIGTERM` 信号，Spring Boot 的 Web 容器会拒绝接收新请求，并等待现有的 HTTP 请求处理完毕，然后再关闭数据库连接池等资源，最后退出进程。

##### 保护阈值


**级联雪崩现象**
- 假设订单服务有 10 个实例，每个实例的 CPU 和线程池负载都在 80% 左右
- 有 2 个实例可能因为物理机的网络稍微抖动了一下，或者碰到了 JVM 的 Full GC 停顿了几秒，导致它们没能及时回应 Nacos 的心跳检测。
- Nacos 介入，判定这 2 个实例“不健康”，将它们从可用列表中剔除
- 原本每个实例负载是 80%，现在流量增加了 25%，剩下 8 个实例的负载瞬间飙升到了 100%
- 因为这 8 个实例严重超载，它们的 CPU 打满，处理请求变慢，线程池全部阻塞。紧接着，这 8 个实例中，又有 3 个因为响应太慢，被 Nacos 判定为超时、不健康而剔除。
- 只剩下 5 个实例，却要扛平时的全量流量，这 5 个实例会瞬间被压垮、宕机。最终，10 个实例一个不剩，整个订单服务彻底瘫痪。

**保护阈值**
- 保护阈值，取值范围是 `0` 到 `1`，健康实例数 / 总实例数 = 健康比例
- 健康比例 <= 保护阈值时，Nacos 会触发保护机制。
- 它会停止剔除不健康实例，强制将该服务下的所有实例，包括已经挂掉的实例全部返回给调用方
- **弃车保帅**：虽然请求会报错，但是**变相地限制了打到健康实例上的流量**
- **宁可牺牲部分请求的成功率**，也**坚决不能让整个服务集群被彻底压垮**
- 在Nacos 控制台每个服务的详情页设置，可以设置为0.5

---


#### 分布式配置



#### 集群部署



#### 底层原理



