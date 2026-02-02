## 线程安全、锁、通信

### 线程安全问题

> **核心逻辑**：先讲清楚“为什么会有线程安全问题”，即硬件架构与 JVM 的抽象。

1. **硬件背景**
* CPU 多级缓存架构（L1/L2/L3）与主内存。
* **MESI 缓存一致性协议**（解决缓存一致性，但引入可见性延迟）。
* 指令重排序（编译器与处理器的优化手段）。

**硬件伪共享**

#### Java内存模型
* **抽象架构**：主内存 vs 工作内存（屏蔽硬件差异）。
* **核心矛盾**：CPU 高速缓存 vs 内存速度差异。
* **三大特性**：原子性、可见性、有序性。
#### Happens-Before
* JMM 的承诺：在不改变结果的前提下允许优化。
* 8 大规则速查（重点：程序次序、volatile 规则、锁规则、传递性）。






### volatile关键字
> **核心逻辑**：Java 语言层面提供了什么工具来解决上述三大问题？

1. **volatile 关键字**
* **核心语义**：保证可见性、禁止指令重排序（**不保证原子性**）。
* **实现原理**：
* **内存屏障** (LoadLoad, StoreStore 等) 的插入策略。
* **底层指令**：`lock` 前缀指令与缓存行失效。
#### 内存屏障

#### volatile保证可见性、有序性

#### volatile不保证原子性

### synchronized

#### 双重检查锁定


### synchronized锁升级


### 无锁底层实现

#### CAS

`cmpxchg` 指令 + `lock` 信号

* 三大问题：ABA 问题（`AtomicStampedReference`）、循环时间长、只能保证一个变量原子性。

#### Unsafe魔法类
* 功能：内存操作（堆外内存）、对象字段偏移量、线程挂起/恢复 (`park`/`unpark`)。
* *地位：它是 JUC 所有工具的后门。*

### JUC框架核心

Doug Lea 大神如何利用 CAS + volatile + Unsafe 构建出一套通用的锁框架


#### atomic包

#### LockSupport

#### AQS：AbstractQueuedSynchronizer





### JUC显式锁

#### ReentrantLock

#### ReentrantReadWriteLock

#### StampedLock

### 线程活跃性问题


### JUC并发流程控制工具

#### CountDownLatch

#### CyclicBarrier

#### Semaphore

#### Exchanger

### 线程性能问题
