# 工具


## IDEA

## Maven

## Git

## Nginx

## Chocolatey：Windows 命令行软件管理器

## WindTerm终端工具

## 数据库工具

### chiner

### DBeaver

## Apifox

# 轮子

## HTTP调用API
## JUnit 5 + Mockito
## Jackson
## SLF4J + Logback


# Redis

## 数据结构

> Redis不是一个简单的内存缓存，而是一个**数据结构服务器**
### String

**Redis的key 都是 String 类型**，是其他所有复杂数据结构的基础。


#### String 的本质与底层原理 (SDS)

在 Redis 中，String 的值可以是：**纯文本、JSON 字符串、数字（整数或浮点数），甚至是一张图片、音频等二进制数据（序列化后）**。单个 String 的 value 最大可以达到 **512MB**。

> **底层实现——SDS (Simple Dynamic String)**
> 
Redis 并没有直接使用 C 语言传统的字符串（以 `\0` 结尾的字符数组），而是自己构建了一种叫做 **简单动态字符串（SDS）** 的抽象类型。
* **O(1) 获取长度：** C 字符串获取长度需要遍历一遍（O(N)），而 SDS 内部维护了 `len` 变量，直接 O(1) 返回。
* **二进制安全：** C 字符串遇到 `\0` 就认为结束了，所以不能存图片等二进制流。SDS 依靠 `len` 变量来判断内容是否结束，所以你存什么它就吐出什么，绝对安全。
* **杜绝缓冲区溢出：** 空间预分配和惰性空间释放机制，不仅防止了内存溢出，还减少了频繁修改字符串带来的内存重分配消耗。

#### 核心操作命令分类

**基础读写：**
* `SET key value`：存入一个字符串。
* `GET key`：获取一个字符串。
* `MSET k1 v1 k2 v2` / `MGET k1 k2`：批量存取。**（实战建议：批量操作可以有效减少网络往返 RTT，但一次不要 MGET 太多，防止阻塞主线程）**

**生存时间与并发控制（面试高频）：**
* `SETEX key seconds value`：存入并设置过期时间（原子操作，常用于验证码、缓存）。
* `SETNX key value`：**S**et if **N**ot e**X**ists。只有 key 不存在时才设置成功。**（实现“分布式锁”最核心的基础命令！）**

**数值自增自减（原子操作）：**
* `INCR key` / `DECR key`：将对应的值加 1 / 减 1。
* `INCRBY key increment`：增加指定的步长。
* *注意：此时 value 必须是数字格式的字符串，否则会报错。*

#### String 的四大经典实战场景

1. **万物皆可缓存 (Cache)：**
   最典型的场景。将数据库中查询到的商品详情、热点文章，转化为 JSON 字符串后 `SET` 到 Redis 中。下次请求直接命中缓存。
2. **计数器 (Counter)与限流：**
   利用 `INCR` 命令的原子性。可以用来记录短视频的播放量、文章的点赞数。也可以结合过期时间做简单的**高并发限流**（例如：限制某个 IP 每分钟只能请求 10 次 API，`INCR` 每次加 1，超过 10 就拒绝）。
3. **分布式 Session 共享：**
   在微服务架构下，用户登录后的 Session 信息或者 Token，通常会序列化为 String 存入 Redis，并设置 TTL（比如 30 分钟）。这样所有节点都能共享登录状态。
4. **分布式锁的基础（预告）：**
   利用 `SETNX` 的排他性（只能有一个客户端能设置成功），来防止高并发下超卖或者缓存击穿问题。

---

### Hash



在 Redis 中，所有的 Key 本身都是一个 String。而当我们把 Value 设置为 Hash 时，这个 Value 内部其实又是一个键值对的集合。也就是形成了一种 **`Key -> { Field : Value }`** 的映射关系。

大家可以把它想象成关系型数据库里的一行记录：
* **Key** = 表名 + 主键 ID (例如：`user:1001`)
* **Field** = 数据库的列名 (例如：`name`, `age`, `balance`)
* **Value** = 具体的值 (例如：`"张三"`, `25`, `1000`)

>既然 String 能存 JSON，凭什么用 Hash？
- 这绝对是面试中极高频的一道对比题！我们举个例子：你想把用户 `user:1001` 的数据存进 Redis，此时他的 `balance` (余额) 增加了 50 块钱。

* **如果用 String (存 JSON)：**
    你需要：`GET` 取出完整 JSON 字符串 $\rightarrow$ 在代码里反序列化成对象 $\rightarrow$ 修改余额 $\rightarrow$ 重新序列化成 JSON $\rightarrow$ `SET` 塞回 Redis。
    *痛点：* 极度浪费网络带宽（因为传输了一大堆你根本不关心的字段），而且在高并发下极易产生“丢失更新”的并发冲突。
* **如果用 Hash：**
    你只需要一条命令：`HINCRBY user:1001 balance 50`。
    *优势：* 直接在 Redis 服务端完成局部属性的修改！网络传输极小，自带原子性，绝对不会发生并发冲突。

#### 核心命令

**基础读写：**
* `HSET key field value`：设置指定字段的值（Redis 4.0 后支持同时设置多个 field-value）。
* `HGET key field`：获取指定字段的值。
* `HMGET key field1 field2`：批量获取多个字段的值。

**局部计算（极度实用）：**
* `HINCRBY key field increment`：给指定的 field 加上指定的值（整数）。
* `HINCRBYFLOAT key field increment`：加浮点数。

**⚠️ 危险命令区（线上慎用）：**
* `HGETALL key`：获取该 Hash 里的所有 Field 和 Value。
* 如果你的 Hash 里存了成千上万个字段，`HGETALL` 极易导致 Redis 单线程阻塞！如果确实需要遍历，**请务必使用 `HSCAN` 命令**进行渐进式游标遍历。

#### 电商购物车

Hash 简直是为购物车量身定制的。

假设用户的 ID 是 `1001`，我们用一个 Hash 结构就可以完美承载他的购物车：
* **Key：** `cart:1001` (代表这个用户的购物车)
* **Field：** `product:8848` (代表商品 ID)
* **Value：** `2` (代表购买数量)

对应的业务操作：
1.  **添加商品：** `HSET cart:1001 product:8848 1`
2.  **增加数量：** `HINCRBY cart:1001 product:8848 1`
3.  **商品总数：** `HLEN cart:1001`
4.  **删除商品：** `HDEL cart:1001 product:8848`
5.  **清空购物车：** `DEL cart:1001` (直接删掉外层的 Key)

#### 底层原理

Redis 为了极致的节省内存，Hash 在底层的实现并不是一开始就直接用哈希表（Dict）的：

* **数据量小的时候（Listpack 紧凑列表）：** 当 Hash 里的字段较少，且每个字段的值都很短时。Redis 7.0 之后采用了 `Listpack`（替代了老版本的 `Ziplist`）。它是一块连续的内存空间，极其节省内存，虽然查找是 $O(N)$，但在数据量极小的情况下，CPU 缓存命中率极高，速度比真正的哈希表还要快。
* **数据量大的时候（Dict 哈希表）：** 当字段变多，或者某个值变大（触发了阈值），Redis 会自动将底层结构转为真正的哈希表，此时时间复杂度变为 $O(1)$。

---
### List

List 的本质是一个**双端队列（Deque）**

1.  **有序性**：List 会严格按照插入的顺序来保存数据。先进入的元素就在前面，后进入的就在后面。
2.  **元素可重复**：不像后面我们要讲的 Set，List 里面存两个“Java”是完全没问题的。
3.  **两头快、中间慢**：
    * **$O(1)$ 性能**：由于底层是双向链表结构，在**头部或尾部**进行插入（Push）和弹出（Pop）的操作是极快的，无论你的 List 里有一万个还是十万个元素，速度都一样。
    * **$O(N)$ 性能**：如果你想通过索引（Index）去访问中间的某个元素，或者在中间插入元素，Redis 就需要从一头开始数，性能会随着数据量增大而下降。

---

#### 核心命令

Redis 的 List 命令非常形象，通常以 `L`（Left，左/头）或 `R`（Right，右/尾）开头：

**入队操作**：
* `LPUSH key value1 [value2]`：从左侧压入。
* `RPUSH key value1 [value2]`：从右侧压入。

**出队操作**：
* `LPOP key`：从左侧弹出一个值。
* `RPOP key`：从右侧弹出一个值。

**查询与查看**：
* `LLEN key`：获取列表长度。
* `LRANGE key start stop`：获取指定范围内的元素（面试常问：`LRANGE key 0 -1` 代表获取全部）。

**阻塞式弹出（神技）**：
* `BLPOP / BRPOP key timeout`：如果列表里没东西，它不会立刻返回空，而是会**阻塞**住连接，直到有新数据进来或者超时。**这是实现简易消息队列的灵魂命令。**

---

#### 经典实战场景

List 最经典的用法就是**多面手转换**：

1.  **实现 Stack（栈）**：`LPUSH` + `LPOP`（同进同出），先进后出。
2.  **实现 Queue（队列）**：`LPUSH` + `RPOP`（异进异出），先进先出。
3.  **实现消息队列 (Message Queue)**：
    利用 `LPUSH` 生产消息，`BRPOP` 消费消息。阻塞特性保证了消费者不需要不停地去轮询 Redis，节省了 CPU 消耗。
4.  **最新动态 / Timeline**：
    比如社交媒体的关注列表。最新的动态用 `LPUSH` 塞进去，查询时用 `LRANGE` 取前 10 条展示。

---

#### Quicklist

早期的 Redis 列表底层可能是 `Ziplist`（压缩列表）或 `Linkedlist`（双端链表）。
* **Linkedlist** 虽然方便，但每个节点都要存前驱后继指针，太费内存，且内存碎片多。
* **Ziplist** 内存紧凑，但修改起来很麻烦。

现在的 Redis 统一使用了 **`Quicklist`**。你可以理解为：**Quicklist 是一个由多个 Listpack（或旧版的 Ziplist）节点构成的双向链表。** 这样既保证了插入删除的效率，又极大地压缩了内存空间，是空间和时间的极致平衡。

---






### Set
Redis 的 Set 是一个**无序**且**元素不重复**的字符串集合。它的底层实现高度优化了内存和查询效率，主要依赖两种数据编码方式：

1.  **IntSet（整数集合）**：
    * **触发条件**：当集合中的所有元素都是整数，并且元素总数较少时（默认不超过 `set-max-intset-entries` 配置的值，通常是 512）。
    * **优势**：内存极其紧凑，采用连续内存分配。它会根据整数的大小自动选择 16 位、32 位或 64 位的整型来存储，极致节省内存。
    * **查询**：内部元素虽在逻辑上无序，但 IntSet 在物理存储时会按从小到大排序，因此可以使用二分查找，时间复杂度为 O(log N)。
2.  **Hashtable（哈希表/字典）**：
    * **触发条件**：当集合中出现了非整数类型的字符串，或者元素数量超过了设定的阈值，底层编码会升级为 Hashtable。
    * **结构**：与 Java 中的 `HashSet` 类似，Redis 使用一个内部的字典（Dict）来存储，字典的 key 就是 Set 中的元素，而字典所有的 value 都指向 `null`。
    * **查询**：时间复杂度为 O(1)，提供极快的去重和查找能力。

---

#### 核心操作命令分类
Set 的命令除了基础的增删改查，最强大的是其内置的**集合数学运算**能力。

**基础增删查操作**：
* `SADD key member [member ...]`：向集合添加一个或多个元素（自动去重）。
* `SREM key member [member ...]`：移除集合中的指定元素。
* `SMEMBERS key`：获取集合中的所有元素（注意：如果是大 Key，此命令可能阻塞单线程，线上慎用，建议用 `SSCAN` 替代）。
* `SCARD key`：获取集合的元素个数（时间复杂度 O(1)）。
* `SISMEMBER key member`：判断元素是否在集合中。

**随机抽取操作（常用于抽奖）**：
* `SRANDMEMBER key [count]`：随机返回集合中的 count 个元素（**不删除**元素）。
* `SPOP key [count]`：随机返回并**弹出（删除）**集合中的 count 个元素。

**集合数学运算（Set 的杀手锏）**：
* **交集 (Intersection)**：`SINTER key [key ...]`（返回多个集合的共有元素）。
* **并集 (Union)**：`SUNION key [key ...]`（合并多个集合的所有元素并去重）。
* **差集 (Difference)**：`SDIFF key [key ...]`（返回第一个集合中有，而其他集合中没有的元素）。
* *注意：交、并、差集都有对应的 `STORE` 命令（如 `SINTERSTORE`），可以将计算结果直接存入一个新的 Set 中，避免大量数据返回给客户端拉满网络带宽。*

---

#### Set 的经典实战场景
由于其“去重”和“集合运算”的天然特性，Set 在高并发业务中有着不可替代的作用：

1. 社交网络：共同好友 / 关注模型
* **场景**：展示“你可能认识的人”或“我和他的共同好友”。
* **方案**：将用户 A 的好友列表存为一个 Set，用户 B 的好友列表存为另一个 Set。使用 `SINTER user:A:friends user:B:friends` 即可秒级得出共同好友。

2. 高并发抽奖系统
* **场景**：年会抽奖、直播间福袋，需保证同一个用户不能重复中奖。
* **方案**：
    * 用户参与抽奖时，将其 ID 加入集合：`SADD lottery:1001 user_id`。
    * **允许重复参与下一轮**：开奖时使用 `SRANDMEMBER lottery:1001 5` 随机抽取 5 人。
    * **不允许重复中奖**：开奖时使用 `SPOP lottery:1001 5`，抽中后直接从奖池剔除。

3. 用户行为去重：点赞 / 签到 / 投票
* **场景**：一篇文章只能点赞一次，或者一个用户一天只能投票一次。
* **方案**：以文章 ID 为 Key（如 `article:999:likes`），点赞时执行 `SADD` 存入用户 ID。如果返回 1 说明点赞成功，返回 0 说明用户已点赞过。使用 `SCARD` 可以快速获取总点赞数。

4. 用户画像与商品标签匹配 (Tags)
* **场景**：电商系统中筛选出同时带有“数码”、“男士”、“满减”标签的商品。
* **方案**：每个标签建一个 Set，里面存放具有该标签的商品 ID。使用 `SINTER tag:digital tag:male tag:discount` 取交集，即可快速筛选出满足所有条件的商品集合。

5. 全局黑白名单过滤 (IP 防刷)
* **场景**：安全网关需要快速拦截恶意 IP。
* **方案**：将恶意 IP 放入 Set 中，每次请求打过来时，只需使用 `SISMEMBER blacklist:ips {current_ip}`，O(1) 复杂度极速判断并拦截。


### ZSet
ZSet 保留了 Set 元素不能重复的特性，但为每个元素额外增加了一个 `score`（浮点数类型的分数）属性，Redis 就是依靠这个分数来为集合中的成员进行从小到大的排序。

ZSet 的底层实现为了兼顾内存效率与查询性能，采用了两种动态切换的编码方式：

1.  **ZipList（压缩列表） / ListPack（紧凑列表，Redis 7.0+）**：
    * **触发条件**：当 ZSet 存储的元素数量较少（默认小于 128 个），且每个元素的值较短（默认小于 64 字节）时使用。
    * **优势**：内存极其紧凑，采用连续的内存块存储。元素和它的 score 紧挨在一起存放，按照 score 从小到大排序。
2.  **SkipList（跳跃表） + Dict（哈希字典）**：
    * **触发条件**：当元素数量增多或体积变大，超出了上述阈值，底层结构会升级为跳表加字典。
    * **Dict（字典）的作用**：维护“元素”到“分数”的映射关系。使得我们可以用 O(1) 的时间复杂度通过元素查到其对应的 score（例如 `ZSCORE` 命令）。
    * **SkipList（跳表）的作用**：维护“分数”的排序关系。跳表是一种基于多级链表实现的随机化数据结构，它通过在节点中维护多个指向其他节点的指针，实现了类似二分查找的效果。支持平均 O(log N) 复杂度的节点插入、删除和范围查询，性能几乎媲美红黑树，但实现极其简单，且在范围查找上比红黑树更具优势。

---

#### 核心操作命令分类
ZSet 的命令非常丰富，主要围绕**分数（Score）**和**排名（Rank）**展开（注意：Redis 默认按 score 从小到大排序）。

* **基础增删改查**：
    * `ZADD key score member [score member ...]`：添加一个或多个元素及其分数。如果元素已存在，则更新其分数。
    * `ZREM key member [member ...]`：移除指定的元素。
    * `ZSCORE key member`：获取指定元素的分数。
    * `ZINCRBY key increment member`：为指定元素的分数加上 increment（常用于点赞数、积分的累加）。
* **按排名（Rank）操作**：
    * `ZRANK key member`：获取元素的正序排名（从 0 开始计，0 表示分数最低的第一名）。
    * `ZREVRANK key member`：获取元素的逆序排名（分数最高的第一名）。
    * `ZRANGE key start stop [WITHSCORES]`：返回指定正序排名区间的元素（例如 `ZRANGE key 0 9` 获取 Top 10，注意 Redis 6.2 之后 `ZRANGE` 整合了反向、按分数等各种查询能力）。
* **按分数（Score）范围操作**：
    * `ZRANGEBYSCORE key min max`：获取分数在指定范围内的所有元素。
    * `ZCOUNT key min max`：统计分数在指定范围内的元素个数。
    * `ZREMRANGEBYSCORE key min max`：删除分数在指定范围内的元素。
* **集合聚合运算**：
    * 支持 `ZINTERSTORE`（交集）和 `ZUNIONSTORE`（并集）。在合并时，可以指定分数的聚合方式（默认是 SUM 求和，也可以是 MIN 或 MAX），并且可以给不同集合设置权重（WEIGHTS）。

---

#### ZSet 的四大经典实战场景

1. 各类排行榜 (Leaderboard)
这是 ZSet 最核心、最无可替代的场景。
* **场景**：微博热搜榜、游戏战力排行榜、电商销量日榜。
* **方案**：以榜单名为 Key，用户/文章 ID 为 member，积分/阅读量为 score。
    * 增加积分：`ZINCRBY hot_search:20231024 1 "article_id_123"`
    * 获取前 10 名：`ZREVRANGE hot_search:20231024 0 9 WITHSCORES`（逆序获取分最高的 10 个）。

2. 延迟消息队列 (Delay Queue)
* **场景**：订单创建 30 分钟后未支付自动取消。
* **方案**：将订单 ID 作为 member，**订单的超时时间戳**作为 score 存入 ZSet。
    * 消费者开启后台线程，不断使用 `ZRANGEBYSCORE delay_queue 0 {当前时间戳} LIMIT 0 1` 轮询获取已经到期的订单记录。
    * 获取到之后用 `ZREM` 删除该任务（防止多节点重复消费，或者结合 Lua 脚本保证原子性），然后执行取消订单逻辑。

3. 滑动窗口限流 (Rate Limiting)
* **场景**：限制某个 API 接口同一个 IP 每分钟最多访问 100 次，比传统的计数器限流更平滑。
* **方案**：以用户 IP 为 Key，请求的唯一 UUID 为 member，**请求的当前时间戳**为 score 存入 ZSet。
    * 每次请求来时，先清理窗口外的数据：`ZREMRANGEBYSCORE ip:192.168.1.1 0 {当前时间戳 - 60秒}`。
    * 然后使用 `ZCARD ip:192.168.1.1` 判断当前窗口内的请求数是否超过 100。没超则放行并 `ZADD` 记录本次请求。

4. 时间轴 / 按时间排序的 Feed 流
* **场景**：微信朋友圈、推特的时间线，需要按时间倒序展示用户关注的动态。
* **方案**：使用 ZSet 存储关注人的动态，member 为动态 ID，score 为发布时间戳。客户端可以使用类似于“分页”的逻辑，每次传入上一次拉取的最后一条动态的时间戳作为最大 score，使用 `ZREVRANGEBYSCORE` 实现增量拉取（这种方式比使用 List 的 `LRANGE` 按照索引分页更可靠，不会因为中间插入新数据导致分页错乱）。
继续为您补全大纲，接下来是 Redis 中极其精巧且在海量数据下能发挥奇效的“黑科技”数据结构——**Bitmaps（位图）**。

虽然它被单独列出，但理解它的第一步是要知道：在 Redis 中，Bitmap 并不是一种全新的数据结构。

---

### Bitmaps


* **底层结构**：Bitmap 的底层其实就是 **String（字符串）**。因为 Redis 的 String 是二进制安全的（Binary Safe），所以它可以用来存储一连串的 `0` 和 `1`。
* **存储极限**：Redis 的 String 最大可以存储 512 MB 的数据。1 Byte = 8 bit，那么 512 MB 就可以存储 $512 \times 1024 \times 1024 \times 8 \approx 42.9$ 亿个比特位。这意味着一个 Bitmap 键最多可以表示 42.9 亿个状态。
* **核心优势（极致的空间压缩）**：Bitmap 最大的魅力在于**极低的内存占用**。如果我们要记录 1 亿个用户的活跃状态（是/否），如果用常规的 Hash 或 Set 存储用户 ID，可能需要几百 MB 甚至上 GB 的内存；而使用 Bitmap，每个用户只占 1 个 bit，1 亿个 bit 仅仅占用约 **12 MB** 的内存。

---

#### 核心操作命令分类
Bitmap 的命令全部是以 `BIT` 开头的，专门用于操作底层的二进制位：

* **基础读写操作**：
    * `SETBIT key offset value`：设置指定偏移量（offset）上的位的值（value 只能是 0 或 1）。*注意：offset 通常可以直接用整数型的用户 ID 来表示。*
    * `GETBIT key offset`：获取指定偏移量上的位的值。如果 offset 超出范围或 key 不存在，返回 0。
* **统计操作（非常重要）**：
    * `BITCOUNT key [start end]`：统计 Bitmap 中值为 1 的比特位的数量（人口统计）。可以指定字节范围。
    * `BITPOS key bit [start end]`：返回 Bitmap 中第一个值为指定位（0 或 1）的偏移量位置。
* **位运算（Bitmap 的杀手锏）**：
    * `BITOP operation destkey key [key ...]`：对一个或多个 Bitmap 执行位操作，并将结果保存到 `destkey` 中。
    * 支持的 `operation` 有：
        * `AND`（交集/按位与）：所有位都为 1 才为 1。
        * `OR`（并集/按位或）：只要有一个为 1 就为 1。
        * `XOR`（异或）：相同为 0，相异为 1。
        * `NOT`（非）：0 变 1，1 变 0（只能作用于一个 key）。

---

#### Bitmaps 的四大经典实战场景

1. 用户签到系统 (User Check-ins)
* **场景**：记录某个用户在一整年内的每天签到情况。
* **方案**：以用户 ID 为 Key（例如 `sign:user:1001:2023`），以一年中的第几天（1-365）为 offset。
    * 签到：`SETBIT sign:user:1001:2023 297 1`（代表第 297 天签到了）。
    * 查询今天是否签到：`GETBIT sign:user:1001:2023 297`。
    * 统计全年总签到天数：`BITCOUNT sign:user:1001:2023`。
    * 获取首次签到日期：`BITPOS sign:user:1001:2023 1`。

2. 活跃用户统计 (DAU / MAU)
* **场景**：统计亿级体量 APP 的日活（DAU）、月活（MAU）以及留存率。
* **方案**：与签到模型反转，**以日期为 Key，以用户 ID 为 offset**。
    * 用户今天登录：`SETBIT dau:20231024 {user_id} 1`。
    * 统计今日日活（DAU）：`BITCOUNT dau:20231024`。
    * **统计月活（MAU）**：将这 30 天的 Bitmap 进行 `BITOP OR`（按位或）合并到一个新 Key，然后对新 Key 执行 `BITCOUNT`。
    * **统计连续三天登录用户**：对连续三天的 Bitmap 执行 `BITOP AND`（按位与），然后 `BITCOUNT`。

3. 用户在线状态 (Online Status)
* **场景**：即时通讯软件需要快速判断好友是否在线。
* **方案**：使用一个全局的 Bitmap，如 `online_status`，用户 ID 作为 offset。上线时 `SETBIT online_status {user_id} 1`，下线时 `SETBIT online_status {user_id} 0`。不仅修改快，占用内存也极少。

4. 简单的特征开关 / AB 测试状态记录
* **场景**：系统上线了一个新功能弹窗，确保每个用户只弹一次。
* **方案**：建一个 Key `feature:new_popup`，用户 ID 作为 offset。弹窗前先 `GETBIT` 判断，如果是 0 则弹窗，并立即 `SETBIT` 为 1；如果是 1 则不弹。对于 1 亿用户，这个标记墙仅仅占用 12MB。
### HyperLogLog
* **本质**：与 Bitmap 类似，HyperLogLog 在 Redis 中并不是一种独立的数据结构，它的底层实际也是基于 **String** 实现的。
* **核心痛点解决**：如果我们要统计一个拥有千万级访问量的页面的 UV（独立访客），如果用 Set 存用户 ID，会耗费海量内存；如果用 Bitmap，当用户 ID 是随机的 UUID 或者跨度极大的字符串时，Bitmap 的 offset 无法映射，依然会造成巨大的内存浪费。
* **极致的空间复杂度**：HyperLogLog 提供了一种**概率算法**。在 Redis 中，一个 HyperLogLog 键不管里面统计了多少个元素（理论上限是 $2^{64}$ 个），它占用的内存总是固定的，且最大只需要 **12 KB**！
* **底层原理（伯努利试验与抛硬币）**：
    * HLL 的核心思想源于概率学。想象你不断抛硬币，记录第一次抛出正面向上的次数。如果你运气极好，连续抛出了 10 次反面，第 11 次才是正面，这说明你总共抛硬币的基数（总次数）一定非常大。
    * Redis 在内部会将每个存入的元素进行 Hash 运算，转换成 64 位的二进制串。然后记录这个二进制串从低位开始出现连续 `0` 的最大长度。
    * 为了消除单次随机事件的偶然性（误差过大），Redis 将 12 KB 的空间划分为 16384（$2^{14}$）个桶（Register），每个桶占 6 bit。存入元素时，用 Hash 值的前 14 位决定放入哪个桶，用剩下的 50 位计算连续 0 的长度并更新桶中的最大值。最后利用调和平均数公式综合所有桶的结果，得出整体的基数估算值。
* **代价（局限性）**：
    1.  **有误差**：标准误差率大约为 **0.81%**。
    2.  **只进不出**：它只记录计算后的概率特征，**不保存任何原始元素**。也就是说，你能知道大概有多少个不同的人来过，但你无法知道具体是哪几个人（无法像 Set 那样执行 `SMEMBERS`）。

---

#### 核心操作命令分类
HyperLogLog 的命令前缀统一为 `PF`，这是为了向 HyperLogLog 算法的提出者、法国伟大的计算机科学家 Philippe Flajolet 致敬。

* **添加元素**：
    * `PFADD key element [element ...]`：将一个或多个元素添加到指定的 HLL 中。如果内部的估算基数因此发生了变化，返回 1；否则返回 0。
* **统计基数（去重数量）**：
    * `PFCOUNT key [key ...]`：返回给定的一个或多个 HLL 的近似基数。如果传入多个 key，它会在内部自动求并集后返回总数（时间复杂度 O(1)，多 key 为 O(N)）。
* **合并 HLL（极其有用）**：
    * `PFMERGE destkey sourcekey [sourcekey ...]`：将多个源 HLL 合并为一个目标 HLL。合并后的 HLL 将包含所有源 HLL 中元素的并集估算值。

---

#### HyperLogLog 的经典实战场景

1. 亿级网页 UV（独立访客）统计
* **场景**：双十一主会场、热搜头条新闻的实时 UV 统计。业务方只需要知道大概有 5000 万人看过，不需要精确到 50000001 还是 49999999。
* **方案**：以页面 ID 和日期作为 Key（例如 `uv:page:1001:20231024`）。
    * 用户访问时：`PFADD uv:page:1001:20231024 {user_id}`。
    * 后台获取数据时：`PFCOUNT uv:page:1001:20231024`，极速返回，哪怕是亿级规模，内存也牢牢锁死在 12 KB。

2. 海量数据去重分析（大屏日/周/月活聚类）
* **场景**：老板需要在 BI 数据大屏上看到当月有多少个独立设备启动了 APP。
* **方案**：不需要额外写定时任务去跑批处理。每天生成一个当天的 HLL `dau:20231001`、`dau:20231002`...
    * 统计当月月活（MAU）时，直接使用 `PFMERGE mau:202310 dau:20231001 dau:20231002 ...`。
    * 合并后直接 `PFCOUNT mau:202310` 即可秒出结果。

3. 实时垃圾邮件/爬虫 IP 检测网络
* **场景**：风控系统需要判断一个恶意用户在过去 24 小时内使用了多少个完全不同的 IP 地址，或者一个 IP 访问了多少个完全不同的高优接口。
* **方案**：使用 `PFADD risk:user_ips:{user_id} {ip_address}`。当 `PFCOUNT` 超过某个危险阈值时，直接触发风控降级。相比于维护庞大的 IP 黑名单集合，HLL 对风控引擎的内存压力几乎可以忽略不计。

---
**💡 总结与面试常考对比（Set vs Bitmap vs HyperLogLog）：**
* 需要**保存数据**，且需要求交/并/差集绝对精确：用 **Set**（最费内存）。
* 统计**状态量**（只有 0 和 1），需要绝对精确，且 ID 是连续的整数（如用户自增 ID）：用 **Bitmap**（极其省内存）。
* 只需统计**去重后的总数**，不需要原始数据，数据量极大，且允许极小误差：用 **HyperLogLog**（无论数据量多大，只占 12 KB）。

### Geospatial
* **本质（披着 GEO 外衣的 ZSet）**：GEO 在 Redis 中**并不是一种独立的数据结构**。它的底层完全是使用 **ZSet（有序集合）** 来实现的。当你向 GEO 中添加位置信息时，Redis 实际上是把地理位置的名称作为 ZSet 的 `member`，把计算后的坐标值作为 ZSet 的 `score` 存了进去。因此，你可以用任何 ZSet 的命令（如 `ZREM`, `ZRANGE`）来操作 GEO。
* **核心算法（GeoHash）**：地球是一个三维球体，坐标是二维的（经度、纬度），而 ZSet 的 score 只能是一维的浮点数。Redis 是如何把二维坐标塞进一维的 score 里的呢？
    * Redis 使用了业界通用的 **GeoHash 算法**。该算法将地球表面不断地划分成越来越小的网格。
    * 它将二维的经纬度数据通过二分区间法进行编码，最终交叉组合成一个 52 位的整数（即 ZSet 的 score）。
    * **GeoHash 的核心特性**：越是靠近的两个地理位置，它们计算出的 GeoHash 编码（即 score）的公共前缀就越长，在 ZSet 中它们排得就越近。因此，查找“附近的人”本质上就是在一个 ZSet 中进行 score 的范围查找（`ZRANGEBYSCORE` 的变体）。
* **坐标限制**：有效的经度从 -180 度到 180 度。有效的纬度从 -85.05112878 度到 85.05112878 度。超出这个范围会报错。

---

#### 核心操作命令分类
GEO 的核心命令涵盖了位置的录入、距离计算以及范围搜索。

* **位置信息的增查**：
    * `GEOADD key longitude latitude member [longitude latitude member ...]`：添加一个或多个地理空间位置。例如：`GEOADD cities 116.40 39.90 "Beijing"`。
    * `GEOPOS key member [member ...]`：获取一个或多个成员的经纬度坐标。
    * `GEOHASH key member [member ...]`：获取成员的标准的 GeoHash 字符串（常用于前端地图组件直接渲染或缓存）。
* **距离计算**：
    * `GEODIST key member1 member2 [m|km|ft|mi]`：计算两个成员之间的距离。支持米（m）、千米（km）、英尺（ft）和英里（mi）。
* **附近范围查找（GEO 的杀手锏，🚨 注意 Redis 版本差异）**：
    * **Redis 6.2 之前**：
        * `GEORADIUS key longitude latitude radius m|km|ft|mi`：根据指定的经纬度，查找指定半径范围内的成员。
        * `GEORADIUSBYMEMBER key member radius m|km...`：根据集合中已有的某个成员，查找其周围指定半径内的成员。
    * **Redis 6.2 及之后（推荐规范）**：
        * Redis 废弃了上述两个老命令，统一升级为更强大、语意更清晰的 **`GEOSEARCH`** 和 **`GEOSEARCHSTORE`**。
        * `GEOSEARCH key [FROMMEMBER member | FROMLONLAT lon lat] [BYRADIUS radius unit | BYBOX width height unit] [ASC|DESC] [COUNT count]`
        * **优势**：不仅支持“圆形半径查找（BYRADIUS）”，还新增了“矩形框查找（BYBOX）”，极其契合地图拖拽框选的业务需求。

---

#### Geospatial 的经典实战场景

1. "附近的人" / "附近的门店" (LBS 核心场景)
* **场景**：打开美团/饿了么，显示“距离你最近的 10 家麦当劳”；或者打开社交软件查看“附近 1 公里内的用户”。
* **方案**：以业务线为 Key（如 `shops:mcdonalds` 或 `users:locations`）。
    * 门店开业时 / 用户心跳上报位置时：`GEOADD users:locations 116.397128 39.916527 user_1001`
    * 查询附近 2 公里内的最多 10 个人，并按距离从近到远排序：
      `GEOSEARCH users:locations FROMLONLAT 116.39 39.90 BYRADIUS 2 km ASC COUNT 10 WITHDIST`（`WITHDIST` 可以直接连同距离一起返回给前端）。

2. 打车软件 / 骑手派单系统
* **场景**：滴滴打车时，呼叫系统需要快速圈定乘客周围 3 公里内所有的空闲司机，并择优派单。
* **方案**：司机的 APP 每隔几秒钟向后端同步一次经纬度，后端使用 `GEOADD drivers:idle {经度} {纬度} {司机ID}` 更新位置。
    * 乘客发单时，系统可以通过 `GEOSEARCH` 瞬间拉取周围的司机。
    * 配合 `GEOSEARCHSTORE`，系统甚至可以将查找到的候选司机列表直接存入一个新的 ZSet 或 List 中，供后续派单算法后台异步排队处理。

3. 物流运输距离预估
* **场景**：顺丰/京东快递在用户下单时，预估收发件城市之间的直线距离，作为基础运费计算的一个因子。
* **方案**：维护一个全国主要省市级集散中心的坐标 GEO 集合。下单时直接执行 `GEODIST cities "Beijing_Center" "Shanghai_Center" km`，极速返回直线距离（实际业务中通常会结合高德/百度 API 算真实路网距离，但 Redis GEO 可作为一层的兜底/粗筛缓存）。

---
**💡 面试避坑指南：**
既然 GEO 底层是 ZSet，那么它就有 **BigKey 的风险**。
如果在全国范围内使用同一个 Key（比如 `GEOADD users:all`），当上亿用户的数据积压在一个 Key 中时，不仅内存分布极度不均，在执行大范围的 `GEOSEARCH` 时还会导致单线程阻塞卡死。
**正确的生产实践是进行数据切片（Sharding）**：比如按城市划分 Key（`users:beijing`、`users:shanghai`），或者更细粒度地按照 GeoHash 前缀进行分片存储。
## Spring集成
### Spring Data Redis

Spring Data Redis (SDR) 是 Spring Data 家族的一部分，它对 Redis 客户端（如 Lettuce 和 Jedis）进行了高级抽象。

#### 核心组件与架构
* **连接工厂 (RedisConnectionFactory)**：
    * 这是 SDR 的基石，负责管理与 Redis 服务器的连接。
    * **Lettuce (默认)**：基于 Netty，支持异步和响应式编程，性能优异，是目前 Spring Boot 的首选。
    * **Jedis**：传统的阻塞式客户端，简单直观，但在高并发场景下需要配合连接池（如 GenericObjectPoolConfig）。
* **RedisTemplate**：
    * 这是开发者最常用的核心类。它封装了所有 Redis 操作，提供了线程安全的 API。
    * **操作分类接口 (Operations)**：
        * `opsForValue()`: 处理 String。
        * `opsForHash()`: 处理 Hash。
        * `opsForList()`: 处理 List。
        * `opsForSet()`: 处理 Set。
        * `opsForZSet()`: 处理 ZSet。
* **序列化机制 (Serializers)**：
    * Redis 存储的是字节数组，因此 Java 对象存入前必须序列化。
    * `StringRedisSerializer`: 存取人类可读的字符串（常用于 Key）。
    * `Jackson2JsonRedisSerializer / GenericJackson2JsonRedisSerializer`: 将对象转为 JSON 格式。
    * `JdkSerializationRedisSerializer`: **默认配置**。会生成带类信息的二进制数据，不可读且占用空间大，通常建议手动修改配置。

#### Redis Repository (高级抽象)
* 模仿 JPA 的编程模型。
* 通过 `@RedisHash("users")` 注解在实体类上定义 Key 前缀。
* 支持通过 `@Id` 自动生成主键和二级索引。
* 只需定义接口继承 `CrudRepository`，即可像操作数据库一样操作 Redis。

---

### Spring Cache
Spring Cache 是一套基于 AOP（面向切面编程）的缓存抽象。它让你不需要写任何 Redis 代码，通过注解就能实现“透明”的缓存。

#### 1. 核心注解
* **`@EnableCaching`**：开启缓存功能，通常加在启动类或配置类上。
* **`@Cacheable`**：**读操作**。先看缓存，有则直接返回；无则执行方法，并将结果存入缓存。
* **`@CachePut`**：**写/更新操作**。无论如何都会执行方法，并将最新结果更新到缓存。
* **`@CacheEvict`**：**删操作**。执行方法并清除缓存（支持 `allEntries = true` 清空整个分区）。
* **`@Caching`**：组合注解，当一个方法需要复杂的缓存逻辑（如删除多个 Key）时使用。

#### 2. Redis 整合实现
* **RedisCacheManager**：Spring Cache 默认不指定底层实现，集成 Redis 时会使用 `RedisCacheManager`。
* **自定义配置**：可以通过 `RedisCacheConfiguration` 设置全局或针对不同 Cache Name 的**过期时间 (TTL)**、**空值不缓存 (disableCachingNullValues)** 以及序列化方式。

---



💡 核心面试要点
* **Lettuce 与 Jedis 的区别**：Lettuce 是基于 Netty 的，线程安全且支持响应式；Jedis 是直连的，非线程安全（需池化），且只支持同步。
* **缓存击穿的处理**：在 Spring Cache 中，可以通过 `@Cacheable(sync = true)` 开启本地锁，防止多个线程同时穿透缓存去查数据库。
* **序列化器的选择**：为什么要禁用默认的 JDK 序列化？因为它会导致 Key 包含乱码前缀，且无法被其他非 Java 客户端读取。建议 Key 使用 `StringRedisSerializer`，Value 使用 `Jackson2JsonRedisSerializer`。


---

## Redis底层原理

### 线程模型

### 内存管理与淘汰机制

### 过期删除策略

### 持久化机制


## 痛点

### 雪崩、穿透、击穿

### 缓存与数据库双写一致性


### 客户端与连接池规范


## 高可用与分布式集群

### 主从复制架构 
### 哨兵模式
### Redis Cluster分片集群
### 线上风险预防与治理
## 复杂业务解决方案
###  分布式锁 

###  高并发限流方案

###  亿级数据量场景实战

#### 布隆过滤器
#### Bitmap (位图)
#### HyperLogLog
###  消息队列机制


# 消息队列

## RocketMQ

## Kafka

# Spring Framework
# Spring Boot


##  Spring Boot基础

### Spring Boot 启动流程与运行原理

#### `@SpringBootApplication`

`@SpringBootApplication` 是 Spring Boot 项目的灵魂，通常标注在主启动类上。进入源码可以发现，它是一个复合注解，核心由以下三个注解“三体合一”构成：

1. `@SpringBootConfiguration`
* **作用**：标记当前类为 Spring 的配置类。
* **底层**：其实际上就是一个 `@Configuration` 注解的派生注解。这说明 Spring Boot 的主启动类本身也就是一个 Spring IoC 容器的配置类，可以在其中定义 `@Bean`。

2. `@ComponentScan`
* **作用**：组件扫描。默认扫描**当前配置类所在包及其所有子包**下的组件（如 `@Controller`、`@Service`、`@Component`、`@Repository` 等），并将它们注册到 Spring 容器中。
* **注意**：这就是为什么我们通常要求将 Spring Boot 的主启动类放在项目的根包（Root Package）下的原因。

1. `@EnableAutoConfiguration`
* **作用**：开启自动配置机制。它是 Spring Boot 实现“开箱即用”的关键。
* **底层原理**：该注解通过 `@Import(AutoConfigurationImportSelector.class)` 将 `AutoConfigurationImportSelector` 导入到容器中。这个选择器会根据当前项目的 Classpath 环境、配置信息等，动态地决定需要加载哪些自动配置类。

---

#### 自动装配机制

自动装配的核心逻辑是：**收集所有潜在的自动配置类 -> 结合条件注解（Conditionals）进行过滤 -> 将符合条件的 Bean 注入容器**。

1. 自动配置类的加载来源
`AutoConfigurationImportSelector` 会调用 Spring 内部的工厂加载机制来获取自动配置类的全限定名列表。在 Spring Boot 的演进过程中，这里的机制发生过重要改变：

* **Spring Boot 2.7 之前的旧机制 (`spring.factories`)**
    * 通过 `SpringFactoriesLoader` 读取类路径下所有 `META-INF/spring.factories` 文件。
    * 寻找 Key 为 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 的配置项，获取其对应的 Value（逗号分隔的自动配置类全限定名）。
* **Spring Boot 2.7+ 的新机制 (`AutoConfiguration.imports`)**
    * 引入了新的加载机制，直接读取 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件。
    * **改变的原因**：`spring.factories` 文件过于庞大，包含了各种组件（监听器、初始化器等），解析时有性能损耗。独立出 `.imports` 文件，专职负责自动配置，提高了启动加载效率，并且支持以行为单位添加注释，更清晰。

2. 条件注解（`@Conditional` 家族）过滤
加载到的自动配置类并不会全部生效，Spring Boot 大量使用了按需加载的 `@Conditional` 派生注解。只有条件成立，自动配置类及其内部的 `@Bean` 才会被注册：
* `@ConditionalOnClass`：Classpath 中存在指定的类时生效（例如：引入了 Redis 依赖，才配置 RedisTemplate）。
* `@ConditionalOnMissingBean`：容器中不存在指定的 Bean 时生效（保证用户自定义的 Bean 优先于自动配置）。
* `@ConditionalOnProperty`：配置文件（application.yml）中存在指定的属性配置时生效。
* `@ConditionalOnWebApplication`：当前应用是 Web 应用时生效。

---

#### 自定义 Starter 开发

开发自定义 Starter 通常用于封装公司内部的公共中间件（如统一日志、鉴权客户端、自定义 RPC 框架等），实现各微服务间的“引入即用”。

1. 命名规范
* **官方 Starter**：`spring-boot-starter-{name}` (例如：`spring-boot-starter-web`)
* **自定义/第三方 Starter**：`{name}-spring-boot-starter` (例如：`mybatis-spring-boot-starter` 或 `acme-log-spring-boot-starter`)

2. 开发步骤
**步骤一：引入依赖**
创建一个普通的 Maven/Gradle 项目，引入自动配置所需的基础依赖：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional> </dependency>
```

**步骤二：定义属性配置类 (`@ConfigurationProperties`)**
用于接收 `application.yml` 中的配置：
```java
@ConfigurationProperties(prefix = "acme.log")
public class AcmeLogProperties {
    private boolean enabled = true;
    private String level = "INFO";
    // getters and setters...
}
```

**步骤三：编写核心业务组件**
（例如：拦截器、日志切面类等组件逻辑）。

**步骤四：编写自动配置类 (`@AutoConfiguration`)**
结合 `@EnableConfigurationProperties` 和 `@Conditional` 注解完成组装：
```java
@AutoConfiguration // Spring Boot 2.7+ 推荐使用，替代 @Configuration
@EnableConfigurationProperties(AcmeLogProperties.class)
@ConditionalOnProperty(prefix = "acme.log", name = "enabled", havingValue = "true", matchIfMissing = true)
public class AcmeLogAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public LogAspect logAspect(AcmeLogProperties properties) {
        return new LogAspect(properties.getLevel());
    }
}
```

**步骤五：注册自动配置类（暴露给 Spring Boot）**
在 `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件中写入你的自动配置类全限定名：
```text
com.acme.log.starter.AcmeLogAutoConfiguration
```
*(注：如果兼容 Spring Boot 2.7 以前，仍需要写在 `META-INF/spring.factories` 中)*。打包发布后，其他业务线引入该 Starter 即可生效。

---

#### Spring Application 生命周期与事件发布机制

当你调用 `SpringApplication.run(Application.class, args)` 时，Spring Boot 会经历一个复杂的启动生命周期。它通过**观察者模式**（事件发布与监听机制）贯穿始终，允许开发者在启动的不同阶段进行扩展。

1. 核心启动流程阶段

1.  **准备阶段（实例化 SpringApplication）**：
    * 推断当前应用类型（REACTIVE、NONE、SERVLET）。
    * 从 `spring.factories` 加载 `ApplicationContextInitializer`（初始化器）和 `ApplicationListener`（监听器）。
    * 推断 Main 方法所在的类。
2.  **运行阶段（调用 `run()` 方法）**：
    * **获取并启动 RunListeners**：获取 `SpringApplicationRunListeners` 并触发 `starting()` 事件。
    * **准备环境 (Environment)**：解析系统参数、命令行参数、配置文件（application.yml）。触发 `environmentPrepared()` 事件。
    * **打印 Banner**：控制台输出 Spring 图标。
    * **创建上下文 (Context)**：根据应用类型创建对应的 IoC 容器（如 `AnnotationConfigServletWebServerApplicationContext`）。
    * **准备上下文**：应用 Initializers，触发 `contextPrepared()` 事件。将启动类加载到上下文中，触发 `contextLoaded()` 事件。
    * **刷新上下文 (Refresh Context)**：**（绝对核心步骤）** 这一步执行 Spring 框架核心的 `refresh()` 方法。完成包扫描、自动配置类加载、Bean 的实例化、依赖注入、AOP 代理生成、内嵌 Tomcat 启动等。
    * **完成刷新**：触发 `started()` 事件。
    * **执行 Runners**：调用容器中所有的 `ApplicationRunner` 和 `CommandLineRunner` 接口实现类（常用于项目启动后执行初始化数据加载）。
    * **运行完毕**：触发 `ready()` 事件。如果启动失败，会触发 `failed()` 事件。

2. 生命周期事件 (Events) 对应表

如果你需要在启动的特定阶段执行逻辑，可以通过实现 `ApplicationListener<E>` 来监听以下核心事件（按触发顺序排序）：

| 启动事件 | 触发时机 | 典型用途 |
| :--- | :--- | :--- |
| `ApplicationStartingEvent` | Run 方法刚执行，获取到 RunListeners 之后。 | 在任何处理之前做一些极早期的初始化。 |
| `ApplicationEnvironmentPreparedEvent` | Environment 构建完成，但上下文尚未创建。 | 动态修改或注入环境变量/配置项。 |
| `ApplicationContextInitializedEvent` | Context 准备完成，Initializers 执行完毕。 | 在 Bean 定义加载前对上下文做微调。 |
| `ApplicationPreparedEvent` | 配置类/Bean 定义已被加载到容器，但尚未实例化。 | 注册自定义的 BeanDefinition。 |
| `ApplicationStartedEvent` | 容器刷新完成 (`refresh` 结束)，所有 Bean 已被创建。 | 启动后执行一些依赖于完整容器的逻辑。 |
| `ApplicationReadyEvent` | Runners 执行完毕，应用已完全就绪，可以接收请求。 | 暴露健康检查状态，或执行最终的就绪通知。 |
| `ApplicationFailedEvent` | 启动过程中的任何阶段发生异常时触发。 | 记录致命错误日志、发送告警。 |

**补充说明**：由于某些早期事件（如 `ApplicationStartingEvent`）触发时 Spring 容器尚未创建完毕，因此不能通过 `@Component` 的方式注册监听器。必须通过 `SpringApplication.addListeners()` 或在 `META-INF/spring.factories` 中配置才能生效。

### 核心上下文与 Bean 管理
* **Bean 的生命周期**：从实例化、属性赋值、初始化（`@PostConstruct`、`InitializingBean`）到销毁的完整链路。
* **循环依赖**：Spring 如何通过三级缓存解决 Setter 注入的循环依赖？为什么 Spring Boot 2.6+ 默认禁用了循环依赖？
* **条件注解的妙用**：`@ConditionalOnClass`、`@ConditionalOnProperty` 等注解在多环境动态装配中的实战。

---

## Web与网关层

### 统一标准与防御性编程

* **RESTful 进阶**：除了 GET/POST，PUT/PATCH/DELETE 的语义化设计。
* **全局异常处理**：`@RestControllerAdvice` 与 `@ExceptionHandler` 配合，打造统一的 JSON 错误响应体。
* **参数校验（Validation）**：`@Validated` 分组校验、自定义校验注解（如校验手机号格式、敏感词）。
* **Jackson 序列化定制**：全局处理时间格式化、Long 类型精度丢失（转 String 前端接收）、忽略 Null 值。

### 2拦截与切面机制（AOP 实战）
* **过滤器 (Filter) vs 拦截器 (Interceptor) vs 切面 (AOP)**：三者的执行顺序与核心应用场景对比。
* **高频 AOP 场景**：全局操作日志打点、接口耗时统计、基于注解的接口防刷限流。

---

## 数据访问与分布式事务


### MyBatis-Plus
* **基础进阶**：自动代码生成、逻辑删除、自动填充（创建时间/更新时间）。
* **高级特性**：乐观锁插件防超卖、防全表更新/删除插件。
* **性能优化**：MyBatis 一级/二级缓存的坑、批量插入的真实底层优化（`rewriteBatchedStatements`）。

### `@Transactional`
* **事务失效的 8 大经典场景**：内部方法自调用（AOP 代理失效）、异常被 catch 吞没、非 public 方法等。
* **事务传播行为**：`REQUIRES_NEW` 与 `NESTED` 在复杂业务（如记录独立日志、主从表嵌套处理）中的抉择。

### 多数据源与读写分离
* **动态数据源路由**：基于 `AbstractRoutingDataSource` 和 AOP 实现的数据源动态切换。
* **分布式事务初探**：引入 Seata（AT 模式）解决跨库/跨服务的数据一致性。

---

## 中间件集成

### Redis集成
* **缓存一致性治理**：双写一致性策略（延迟双删验证、Canal 监听 Binlog 异步更新缓存）。
* **分布式锁实战**：放弃原生的 `setnx`，全面接入 Redisson（看门狗机制、可重入锁、红锁）。
* **Spring Cache 避坑**：如何自定义缓存管理器，解决默认序列化乱码和无法针对单个 Key 设置 TTL 的痛点。

### 消息队列集成
* **可靠消息投递**：Spring Boot 结合 RocketMQ 事务消息实现分布式最终一致性。
* **消费者幂等与异常处理**：消费失败的重试策略与死信队列（DLQ）的监听处理。

---

## 安全与鉴权
*没有安全防护的系统是在“裸奔”。*

### 认证与授权方案选型
* **Spring Security**：重量级，功能极强但学习曲线陡峭。核心：FilterChain 机制、自定义 UserDetailsService。
* **Sa-Token**：国内极度流行的轻量级权限框架（推荐），零配置实现登录、角色权限、踢人下线。

### Token架构
* **JWT 深度实践**：JWT 的优缺点，Token 续期问题（双 Token 刷新机制：Access Token + Refresh Token）。

---

## 生产运维与性能调优

### 监控与观测 (Observability)
* **Spring Boot Actuator**：暴露健康检查 (`/health`)、环境变量、线程池指标。
* **链路追踪**：集成 SkyWalking 或 Zipkin，解决微服务架构下“错误查不到源头”的痛点。

### 优雅停机 (Graceful Shutdown)
* 如何配置 Spring Boot 和 Tomcat，保证在 K8s 销毁 Pod 时，先拒绝新请求，等老请求处理完、MQ 消息消费完再关闭 JVM？

### JVM 与 Tomcat 调优参数
* Spring Boot 内嵌 Tomcat 的核心参数调优（`max-threads`, `accept-count`, `max-connections`）。
* 针对 Spring Boot 业务特性的 JVM 垃圾回收器选择与内存分配。

---



# Spring Cloud Alibaba

## 服务治理与配置中心：Nacos
> 💡 **入门描述：** > **Nacos = 城市的“114查号台” + “全市广播大喇叭”。**
> 以前服务A想调用服务B，必须在代码里写死B的 IP 地址。如果B的机器换了，A就懵了。现在有了 Nacos（查号台），服务B启动时会告诉 Nacos“我叫B，我的 IP 是多少”。服务A只需要问 Nacos“给我B的地址”，就能精准找到对方。
> 另外，它还是“广播大喇叭”（配置中心），以前改个数据库密码需要重启整个系统，现在你在 Nacos 后台改一下，它会瞬间把新密码广播给所有机器，实现热更新。

* **1. 注册中心机制原理**
    * 服务注册与发现流程（CP 与 AP 模型的切换，Distro 协议与 Raft 协议的底层逻辑）。
    * 服务健康检查机制（临时实例的心跳机制 vs 持久化实例的探活机制）。
* **2. 统一配置管理**
    * 动态刷新的底层原理（长轮询机制，为什么不用长连接或短轮询？）。
    * 配置的隔离级别（Namespace 命名空间隔离环境、Group 分组、DataId 粒度）。
* **3. Nacos 高可用集群部署**
    * 基于 MySQL 存储的持久化方案。
    * Nacos 集群架构设计与线上避坑指南。

---

## 流量防卫兵：Sentinel
> 💡 **入门描述：** > **Sentinel = 城市的“交警队” + “智能保险丝”。**
> 双十一到了，瞬间涌入10万个买家，如果不加干预，服务器绝对会“宕机”（交通瘫痪）。Sentinel 作为交警，可以规定“每秒最多只放行 1000 人，剩下的排队或直接拒绝”（这叫限流）。
> 如果某个被调用的旧系统突然变得极慢（比如堵车了），Sentinel 发现情况不对，会像保险丝一样“啪”地断开（这叫熔断降级），告诉后面的请求“此路不通，请稍后再试”，从而防止整个微服务雪崩。

* **1. 限流（流控）实战**
    * QPS 限流与线程数限流的区别与场景。
    * 流控模式（直接、关联、链路）与流控效果（快速失败、Warm Up 预热、排队等待平滑处理）。
* **2. 熔断与降级机制**
    * 熔断的三大策略：慢调用比例、异常比例、异常数。
    * 熔断器的三种状态转换（Closed, Open, Half-Open 半开探活机制）。
* **3. 热点参数限流与系统自适应保护**
    * 如何防止某个爆款商品打穿缓存？（热点规则）。
    * 基于系统 Load、CPU 使用率的全局最终兜底保护。
* **4. 规则持久化方案**
    * Sentinel 控制台推送规则至 Nacos，微服务监听 Nacos 实现规则持久化（生产环境必考）。

---

## 分布式事务解决方案：Seata
> 💡 **入门描述：** > **Seata = 跨国项目的“铁腕总包工头”。**
> 单体应用时，下订单和扣库存连着同一个数据库，要么一起成功，要么一起回滚。但在微服务中，订单服务和库存服务部署在不同的机器上，连着不同的数据库。如果订单生成了，但网络一抖，库存没扣成功，老板就亏惨了。
> Seata 就是那个总包工头，它协调所有服务：大家先干活，但先别提交。等所有人都确认没问题了，总工头一声令下“一起提交！”；只要有一个人说不行，总工头就下令“所有人，撤销刚才的操作，恢复原样！”。

* **1. 分布式事务的核心痛点与 CAP/BASE 理论基石**
* **2. Seata 架构三大核心角色**
    * TC (Transaction Coordinator)：事务协调者（总线）。
    * TM (Transaction Manager)：事务管理器（发起者）。
    * RM (Resource Manager)：资源管理器（参与者）。
* **3. Seata 的四大模式深度解析（重点掌握前两个）**
    * **AT 模式（绝对主流）**：业务无侵入，基于全局锁与 `undo_log` 回滚日志的机制原理（脏写问题如何避免？）。
    * **TCC 模式**：高性能，业务强侵入。Try、Confirm、Cancel 三个阶段的代码编写实战，以及如何处理“空回滚”与“悬挂”问题。
    * **Saga 模式**：长事务解决方案（状态机）。
    * **XA 模式**：基于数据库原生支持的强一致性方案。

---

## RPC 与负载均衡：Dubbo / OpenFeign
> 💡 **入门描述：** > **OpenFeign = 跨部门打内线电话的“智能分机号”。**
> 微服务之间需要互相调用数据。你当然可以用标准的 HTTP/REST 去请求，但拼写 URL、处理返回结果非常繁琐。使用 OpenFeign，你只需要像调用本地 Java 方法一样写代码（加个注解），它会自动帮你把调用转换成网络请求发出去。如果对方有 5 台机器，它还能自动帮你“掷骰子”（负载均衡），今天打给这台，明天打给那台。

* **1. Spring Cloud OpenFeign**
    * 声明式客户端的动态代理原理。
    * 超时控制与 HTTP 连接池优化。
* **2. LoadBalancer 负载均衡策略**
    * 轮询、随机、加权等算法实战。
* **3. Apache Dubbo (可选拓展)**
    * RPC 与 HTTP 的本质区别（为什么有些大厂偏爱 Dubbo？因为基于 TCP 的自定义协议序列化更小，速度更快）。

---

## 统一流量网关：Spring Cloud Gateway
> 💡 **入门描述：** > **Gateway = 整个城市的“海关总署”兼“保安队长”。**
> 前端小程序和 APP 不可能记住你后台几百个微服务的 IP 地址。所有外部流量必须先经过 Gateway（统一暴露一个入口）。它负责查验你的“通行证”（鉴权），然后根据你访问的路径，把你精确导航到对应的微服务大楼（路由转发）。

* **1. 核心概念与工作机制**
    * Route（路由）、Predicate（断言/匹配规则）、Filter（过滤器）三剑客。
    * 基于 Netty 的响应式非阻塞异步架构优势。
* **2. 高级实战场景**
    * 全局统一鉴权（集成 JWT）。
    * 网关层的全局限流与跨域处理。

---





# MySQL

# MongoDB

# 数据库中间件

# 高并发架构设计

# 领域驱动设计DDD


# Docker

# Kubernetes



