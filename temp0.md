基于目标博客内容`https://javabetter.cn/thread/jmm.html`，剔除无关技术的内容，检查其技术正确性。书写脱胎于该博客的技术文章，确保技术文章的内容完整性和逻辑连贯性，文章风格应是严谨的、学术的、详略得当的。特别注意：删减有关技术的内容可能会造成重大事故


# 并发编程

## 底层原理

### Java内存模型与硬件基础

CPU 缓存架构、缓存一致性协议、伪共享

* *Java的内存模型(JMM)*：主内存与工作内存、三大特性、Happens-Before 原则
* *volatile关键字*：volatile 的内存屏障实现、final 的语义

多个线程同时访问并修改同一数据且缺乏适当的同步措施时造成线程安全问题，线程安全问题应从三个方面考虑：**原子性、可见性、有序性**
- **原子性**：原子性的操作是不可被中断的一个或一系列操作，单行代码可能不是原子性的，在汇编层面是多个原子指令组成的。
- **可见性**：现代多核架构中的每个Cpu拥有独立的私有高速缓存，线程修改变量值可能仅写入其所在核心的缓存而未同步写入主内存，修改对其他线程不可见
- **有序性**：为提高程序性能，Cpu会对指令重排序

**死锁**：两个或多个线程互相持有对方所需的资源，同时等待对方释放资源，导致所有相关线程永久阻塞


### 线程基础✅



在操作系统层面，并发执行的单元被划分为**进程与线程**
* **进程Process**：**资源分配的最小单位**
  * 每个进程代表了一个正在运行的程序实例，拥有独立的内存空间和系统资源句柄
  * 进程间的资源是隔离的，进程间的通信IPC需要跨越内存边界，开销较大。
* **线程Thread**：**Cpu 调度的最小单位**，同一进程内的多个线程共享进程的堆内存和方法区。每个线程仅维持独立的**程序计数器**、**虚拟机栈**和**本地方法栈**。
  * 线程创建和切换的开销远小于进程
  * 同进程下的线程通过共享内存通信，效率高但需处理线程安全问题
  * 除守护线程之外的所有线程结束运行时，整个进程终止
  * 进程的销毁将强制结束其所有线程


**Java 线程的实现本质**
- Java 线程在主流JVM中通常采用 **1:1 的内核线程模型**。即每一个 Java `Thread` 对象都直接映射到一个操作系统的内核线程，即**Java 线程的创建、销毁以及调度，本质上都依赖于操作系统的系统调用**。

**上下文切换**
- 上下文切换：当一个线程的时间片耗尽或发生阻塞时，Cpu需从当前线程切换至另一线程
- 多线程数量与Cpu利用率不是正相关，核心制约因素在于**上下文切换**的开销。
  - 对于**计算密集型**任务，**线程数超过 Cpu 核心数反而会导致性能下降**
  - 对于**I/O 密集型**任务，**多线程通过重叠 I/O 等待时间来显著提升吞吐量。**
- 上下文切换的流程
  1. **保存现场**：将当前线程的寄存器状态、程序计数器等保存到线程控制块TCB或栈中。
  2. **恢复现场**：从目标线程的存储位置加载其执行状态。
  3. **缓存失效**：**切换可能导致 Cpu 高速缓存 L1/L2 Cache和TLB页表缓冲失效**，从而降低后续指令的执行效率。
- 上下文切换的开销
  - **直接开销**：
    - 操作系统核心需保存当前线程的执行现场，包括程序计数器、寄存器状态及栈指针，并加载新线程的对应状态
    - 上述过程在用户态与内核态之间频繁转换，消耗宝贵的 Cpu 指令周期。
  - **间接开销**：
    - 上下文切换导致 Cpu 缓存（L1/L2/L3 Cache）与转换后备缓冲器TLB失效
    - 新调度的线程往往无法利用旧线程遗留的热缓存，导致缓存未命中率飙升，此时 Cpu 必须等待慢速内存的数据加载，造成显著的流水线停顿。

**线程消耗的内存资源**
* **栈内存**：默认情况下，每个线程分配约 1MB 的栈空间（可通过 `-Xss` 参数调整）。若系统创建数千个线程，将消耗数 GB 的内存，极易诱发 `StackOverflowError` 或 `OutOfMemoryError`。
* **元数据开销**：内核还需维护线程控制块（TCB）等数据结构以管理线程状态和调度优先级。

---


#### Java 线程的实现

Java 提供了三种机制定义和执行线程任务

1. **继承 `Thread` 类**
   - 通过创建一个继承自 `Thread` 的子类并重写其 `run()` 方法。
   - 编写简单，但是单继承限制类的扩展性

```java
public class MyThread extends Thread {
    @Override
    public void run() {
        // 业务逻辑
    }
}
// 启动: new MyThread().start();

```

1. **实现 `Runnable` 接口**
   - 创建一个实现 `Runnable` 接口的类，并将其实例传递给 `Thread` 对象。
   - 避免单继承限制，**解耦任务逻辑与执行机制**

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        // 业务逻辑
    }
}
// 启动: new Thread(new MyRunnable()).start();

```


1. **实现 `Callable` 接口**
   - **`Runnable` 的 `run()` 方法没有返回值且无法抛出受检异常**
   - `Callable` 接口和异步计算接口适用于需要获取异步执行结果或处理异常的场景

```java
public class CallerTask implements Callable<String> {
    public String call() throws Exception {
        return "Result";
    }
}
```

**`start()` 与 `run()` 的本质区别**
* **`run()`**：仅是一个普通的类方法调用。如果在主线程中调用 `run()`，代码将在主线程上下文中同步执行，不会创建新线程。
* **`start()`**：是启动线程的唯一正确方式。该方法会请求 JVM 分配新的栈空间并初始化线程上下文，随后由 JVM 自动调用该线程的 `run()` 方法，从而实现并发执行。

---

#### 线程调度与控制

**线程休眠`sleep`**
- `Thread.sleep(long millis)` **使当前正在执行的线程暂停指定的时间，进入TIMED_WAITING 状态**。
* **特性**：**`sleep` 不会释放当前持有的锁，仅让出 CPU 时间片**。
* **异常处理**：**必须捕获 `InterruptedException`，这是线程响应中断的标准机制**。

**线程合并`join`**：
- **调用 `t.join()` 会使当前线程（调用者）进入阻塞状态，直到目标线程 `t` 执行完毕**。
* **应用场景**：**用于处理线程间的依赖关系，例如主线程需要等待子线程完成数据初始化后才能继续执行。**

**守护线程 `Daemon`**
- 通过 `setDaemon(true)` 可将线程标记为守护线程。
* **定义**：**守护线程是为用户线程提供服务的后台线程（如垃圾回收线程）。**
* **生命周期**：JVM 的生命周期取决于非守护线程，即用户线程。当所有非守护线程结束时，JVM 进程将退出，此时所有的守护线程会被强制终止，无论其是否执行完毕。

线程让步`yield`
- `Thread.yield()` 是一种静态方法，用于向线程调度器发出建议。
* 语义：暗示当前线程愿意放弃剩余的时间片，重新回到 RUNNABLE 状态。
* **不确定性**：这仅是一个提示，操作系统调度器可能会忽略此请求，或者在让步后立即再次调度该线程。因此，不能依赖 `yield` 来保证程序的逻辑执行顺序。业务环境不要使用该函数

---


#### 异步获取线程执行结果


`Runnable` 接口无法返回结果或抛出异常，自 JDK 1.5 起，`java.util.concurrent` 包引入了 `Callable`、`Future` 及 `FutureTask` 等构建块，完善了异步计算模型
1. **无法返回执行结果**：`run` 方法没有返回值，导致主线程无法直接获取子线程的计算结果。若需获取结果，通常不得不依赖共享变量或复杂的线程间通信机制，增加了代码的耦合度和同步控制的复杂度。
2. **无法抛出受检异常**：`run` 方法未声明抛出任何受检异常（Checked Exception），这意味着任务执行过程中的错误必须在方法内部被捕获处理，无法向上传递给调用者。

##### Callable接口

`Callable` 实例不直接通过 `Thread` 类运行，而是提交到 `ExecutorService` 线程池，由线程池管理
* **泛型支持**：`call()` 方法返回一个泛型类型 `V` 的结果
* **异常传播**：方法签名中声明了 `throws Exception`，允许任务在执行过程中抛出异常，并由任务的调用方进行捕获和处理

```java
public interface Callable<V> {
    V call() throws Exception;
}

```

##### Future

`Callable` 任务通常在另一个线程中异步执行，**调用者无法立即获得`Callable` 任务的执行结果**。而`Future`是**对异步计算的结果的抽象**，**充当了异步计算结果的句柄或占位符**

`Future` 接口定义了 5 个核心方法，用于查询任务状态和获取结果
1. **任务取消**：`boolean cancel(boolean mayInterruptIfRunning)`
   * 尝试取消任务的执行。如果任务已完成或无法取消，则返回 `false`。参数 `mayInterruptIfRunning` 决定是否允许中断正在运行的任务线程。
2. **取消状态查询**：`boolean isCancelled()`
   * 如果任务在正常完成前被取消，则返回 `true`。
3. **完成状态查询**：`boolean isDone()`
   * 如果任务已完成（包括正常结束、异常结束或被取消），则返回 `true`。
4. **获取结果（阻塞）**：`V get()`
   * **阻塞当前线程，直到任务执行完毕并返回结果**
   * 如果任务被取消，抛出 `CancellationException`
   * 如果任务执行异常，抛出 `ExecutionException`。
5. **获取结果（超时）**：`V get(long timeout, TimeUnit unit)`
   * 在指定的时间范围内阻塞等待结果，若超时仍未完成，则抛出 `TimeoutException`。

#####  FutureTask

`FutureTask` 类同时实现 `Future`和`Runnable` 接口，提供两个构造函数支持分别支持传入 `Callable` 实例，或 `Runnable` 实例配合一个预设的结果值。
- `FutureTask`作为`Runnable`提交给 `Thread`执行
- `FutureTask`作为`Callable` 提交给 `ExecutorService` 执行
- `FutureTask`作为异步计算结果的句柄，供调用者获取最终计算结果、查询状态、管理生命周期

```java
public class FutureTask<V> implements RunnableFuture<V>{}

public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

##### CompletableFuture


`CompletableFuture` 同时承载了**异步计算结果句柄**与**异步计算阶段**的双重身份
* **`Future<T>`**：代表一个**异步计算的结果**。它关注的是计算的“终点”，即结果是否产生、如何获取结果以及如何取消任务。
* **`CompletionStage<T>`**：代表一个**异步计算的阶段**。它关注的是计算的过程与流转，定义了当一个阶段完成时，如何触发下一个阶段


```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {}
```

**`Future` 接口并未真正实现异步执行**
1. **阻塞获取**：`get()` 方法是阻塞的，若不使用轮询（`isDone()`），主线程必须挂起等待，违背了异步编程释放资源的初衷。
2. **缺乏被动完成机制**：`Future` 的结果只能由底层线程执行结束来设定，外部线程无法手动设置结果。
3. **异常处理割裂**：缺乏优雅的异常回调机制，异常通常需在 `get()` 时捕获 `ExecutionException` 处理。

**CompletableFuture对 Future 接口的增强**：

* **主动完成**：提供了 `complete(T value)` 和 `completeExceptionally(Throwable ex)` 方法，允许外部线程在任意时刻显式地终结任务并传递结果或异常。
* **非阻塞链式调用**：不再依赖 `get()` 等待，而是通过注册回调函数，当数据就绪时自动触发后续逻辑。

**`CompletionStage`是 `CompletableFuture` 能够进行异步编排的灵魂**。它定义了约 38 个接口方法，可以严格解耦为三个维度：**依赖关系**、**执行线程**、**功能语义**

一、**依赖关系**：**根据当前阶段执行所需的上游依赖数量**，又可细分为三类
  1. **串行关系**
     * **语义**：当前阶段依赖于上一个阶段的完成。
     * **API 示例**：`thenApply`, `thenAccept`, `thenRun`, `thenCompose`.
     * **逻辑表示**：$$Stage A \rightarrow Stage B$$
  2. **汇聚关系**
     * **语义**：当前阶段需要等待两个上游阶段**全部**完成。
     * **API 示例**：`thenCombine` (组合结果), `thenAcceptBoth` (消费结果), `runAfterBoth` (仅执行)。
     * **逻辑表示**：$$Stage A \land Stage B \rightarrow Stage C$$
  3. **竞争关系**
     * **语义**：当前阶段仅需等待两个上游阶段中的**任意一个**完成。
     * **API 示例**：`applyToEither`, `acceptEither`, `runAfterEither`。
     * **逻辑表示**：$$Stage A \lor Stage B \rightarrow Stage C$$

二、**功能语义（参数类型）**：根据传入的lambda表达式参数的不同，决定了数据流的形态

| 核心方法 | 接收参数 (Functional Interface) | 返回类型 | 语义描述 |
| --- | --- | --- | --- |
| **`thenApply`** | `Function<T, U>` | `CompletableFuture<U>` | **转换**：接收上一步结果，计算并返回新结果。 |
| **`thenAccept`** | `Consumer<T>` | `CompletableFuture<Void>` | **消费**：接收上一步结果进行处理，无返回值。 |
| **`thenRun`** | `Runnable` | `CompletableFuture<Void>` | **行动**：不关心上一步结果，仅执行操作。 |
| **`thenCompose`** | `Function<T, CompletionStage<U>>` | `CompletableFuture<U>` | **扁平化**（FlatMap）：连接两个异步源，避免嵌套。 |

`thenCompose`避免嵌套
```JAVA
CompletableFuture<User> getUserInfo(id)

CompletableFuture<Double> getAccountBalance(user)

CompletableFuture<CompletableFuture<Double>> result = getUserInfo(id).thenApply(user -> getAccountBalance(user));

CompletableFuture<Double> result = getUserInfo(id).thenCompose(user -> getAccountBalance(user));
```


三、**执行线程（Async 后缀）**：`CompletionStage` 的每个方法通常有两个变体，以 `thenApply` 为例：

1. **`thenApply(fn)`**：默认执行方式。由执行上一个阶段的线程继续执行当前阶段
2. **`thenApplyAsync(fn)`**：异步执行。将任务提交给 `ForkJoinPool.commonPool()` 执行。
3. **`thenApplyAsync(fn, executor)`**：指定线程池异步执行。将任务提交给自定义的 `Executor`

四、**`CompletableFuture`对`CompletionStage`语义的扩展**：除了实现 `CompletionStage` 的接口外，`CompletableFuture` 类本身还提供了一些静态方法和控制方法，用于处理更复杂的工程场景。
1. **静态工厂方法**
   - **为了更简洁地启动异步任务**，**`CompletableFuture` 提供了以下静态入口，无需手动 `new Thread`：**
     * `supplyAsync(Supplier<U> supplier)`: 启动一个有返回值的异步任务。
     * `runAsync(Runnable runnable)`: 启动一个无返回值的异步任务。
2. **多任务聚合（AllOf 与 AnyOf）**：
   - **在微服务并发调用中，常需要等待多个接口同时返回，或取最快返回的一个。**
   * **`allOf(CompletableFuture<?>... cfs)`**：返回一个新的 Future，当所有传入的 Future 都完成时，该 Future 完成
   * **`anyOf(CompletableFuture<?>... cfs)`**：返回一个新的 Future，当任意一个传入的 Future 完成时，该 Future 即完成。
3. **异常处理与恢复**：
   * `CompletableFuture` 提供了比 `try-catch` 更优雅的异常流处理：
   * **`exceptionally(fn)`**：仅当上游发生异常时触发，用于返回兜底值（Fallback）。
   * **`handle(biFn)`**：无论正常或异常均会触发，类似于 `finally` 块，但在回调中可根据异常参数动态决定返回值。
4. **join() vs get()**：
   * `CompletableFuture` 引入了 `join()` 方法，其行为与 `Future.get()` 类似，均为阻塞等待结果。

   * **关键区别**：`join()` 抛出的是未经检查的 `CompletionException`（运行时异常），这使得它在 Lambda 表达式或 Stream 流操作中比抛出受检异常 `ExecutionException` 的 `get()` 更加方便。





---
#### 线程生命周期：状态模型与流转控制机制

**线程状态模型**
- 传统操作系统理论中，进程/线程通常被描述为五种状态：
  1. **新建（New）**：进程被创建，但未初始化。
  2. **就绪（Ready）**：已准备好执行，等待 CPU 调度。
  3. **运行（Running）**：正在 CPU 上执行。
  4. **等待/阻塞（Waiting/Blocked）**：等待 IO 或其他事件。
  5. **终止（Terminated）**：执行完毕。

- JVM对线程状态进行了更细粒度的抽象。根据 `java.lang.Thread.State` 枚举，Java 线程包含以下六种状态：
  1. **NEW（新建）**：线程对象已创建，但尚未调用 `start()` 方法。
  2. **RUNNABLE（可运行）**：对应操作系统层面的 **Ready** 和 **Running**。处于该状态的线程可能正在 JVM 中执行，也可能正在等待操作系统的资源（如 CPU 时间片）。
  3. **BLOCKED（阻塞）**：线程正在等待监视器锁（Monitor Lock）以进入 `synchronized` 代码块或方法。
  4. **WAITING（等待）**：线程无限期等待另一个线程执行特定操作（如 `notify`）。
  5. **TIMED_WAITING（超时等待）**：线程在指定时间内等待另一个线程执行特定操作。
  6. **TERMINATED（终止）**：线程已执行完毕。

**状态流转机制**
- **NEW 到 RUNNABLE**：**通过调用 `Thread.start()` 方法触发。**
  - `start()` 方法内部调用了本地方法 `start0()`，由 JVM 请求操作系统创建新的线程
  - 若重复调用 `start()`，会抛出 `IllegalThreadStateException`。

- **RUNNABLE 的内部流转**
  - 在 RUNNABLE 状态下，线程由操作系统调度器管理。操作系统层面的Ready和Running对JVM是透明的，统称为 RUNNABLE。
  - 当线程获得 CPU 时间片时，处于“运行中”；
  - 当时间片用完或发生线程切换时，处于“就绪”状态。

- **RUNNABLE 与 阻塞/等待状态的转换**

| 状态 | 触发条件 | 唤醒/恢复条件 | 关键特征 |
| --- | --- | --- | --- |
| **BLOCKED** | 等待获取 `synchronized` 监视器锁。 | 获得锁。 | **被动等待**。通常发生在进入同步块或 `wait()` 后重获锁时。 |
| **WAITING** | `Object.wait()`使当前线程处于等待状态直到另一个线程唤醒它<br>`Thread.join()`等待线程执行完毕，底层调用的是 Object 的 wait 方法 <br> `LockSupport.park()`除非获得调用许可，否则禁用当前线程进行线程调度 | `Object.notify()`、`notifyAll()` 或 `LockSupport.unpark()`。 | **主动等待**。线程释放 CPU，且需被显式唤醒。 |
| **TIMED_WAITING** |  `Thread.sleep(t)`使当前线程睡眠指定时间<br>`wait(t)`线程休眠指定时间，等待期间可以通过notify()/notifyAll()唤醒；<br>`join(t)` 等待当前线程最多执行 millis 毫秒，如果 millis 为 0，则会一直执行；<br>`LockSupport.parkNanos(long nanos)`除非获得调用许可，否则禁用当前线程进行线程调度指定时间<br> `LockSupport.parkUntil(long deadline)`同上| 时间到期或被显式唤醒。 | **自动唤醒**。即使无外界干扰，时间到后也会自动返回。 |

**BLOCKED vs WAITING/TIMED_WAITING状态分辨**
- BLOCKED是想执行但是没有锁执行不了
- WAITING/TIMED_WAITING是自己不想干了，等代码逻辑（信号/时间）满足了才继续干

| 维度 | BLOCKED (阻塞) | WAITING / TIMED_WAITING (等待) |
| --- | --- | --- |
| **触发原因** | **争夺锁失败** (被动) | **显式放弃 CPU** (主动) |
| **底层语义** | 试图进入 `synchronized` 临界区但未获得监视器锁（Monitor）。 | 调用了 `wait()`, `sleep()`, `join()`, `park()` 等方法，等待特定条件或时间。 |
| **关键动作** | **"卡在门口"**：无法执行代码，被 JVM 挂起，等待锁释放。 | **"休息中"**：线程暂停执行，等待被唤醒（notify）或时间结束。 |
| **锁的持有** | 根本还没拿到锁。 | 可能持有锁（如 `sleep`），也可能释放了锁（如 `wait`）。 |

```JAVA
public class Main {
    public static void main(String[] args) throws InterruptedException {
        Main main = new Main();
        main.blockedTest();
    }

    public void blockedTest() throws InterruptedException {
        Thread a = new Thread(new Runnable() {
            @Override
            public void run() {
                testMethod();
            }
        }, "a");

        Thread b = new Thread(new Runnable() {
            @Override
            public void run() {
                testMethod();
            }
        }, "b");

        //main对象作为锁由ab竞争
        a.start();//a开始休眠
        Thread.sleep(1000L); // 需要注意这里main线程休眠了1000毫秒，而testMethod()里休眠了2000毫秒
        b.start();//b开始休眠
        System.out.println(a.getName() + ":" + a.getState()); // a获取到锁在休眠，打印TIMED_WATING
        System.out.println(b.getName() + ":" + b.getState()); // b未获取到锁在阻塞
    }

    // 同步方法争夺锁
    private synchronized void testMethod() {
        try {
            Thread.sleep(2000L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

**核心控制方法**

**`wait()` vs `sleep()`**
* **`Object.wait()`**:
  * **归属**：`java.lang.Object` 类。
  * **锁机制**：**必须释放当前持有的监视器锁（Monitor）**，允许其他线程进入同步块。
  * **前置条件**：调用前必须拥有该对象的锁（即在 `synchronized` 块内），否则抛出 `IllegalMonitorStateException`。
  * **用途**：**线程间通信（等待/通知机制）**。


* **`Thread.sleep(long millis)`**:
  * **归属**：`java.lang.Thread` 静态方法。
  * **锁机制**：**不会释放任何锁**。若线程持有锁进入睡眠，其他线程将一直等待该锁。
  * **用途**：暂停执行，让出 CPU 时间片给其他线程。

**`join()`**：方法用于线程同步，即“等待另一个线程结束”。
* **实现原理**：`join()` 内部实际上是通过调用 `wait()` 来实现的（在目标线程对象上等待）。当目标线程终止（TERMINATED）时，JVM 会自动调用 `notifyAll()` 唤醒等待在该线程对象上的所有线程。

**`interrupt()`**：Java 的中断机制是一种协作机制，而非强制停止。

* **`interrupt()`**: 设置中断标志位为 true。
* **响应中断**：
  * 若线程处于 BLOCKED、WAITING 或 TIMED_WAITING 状态，会抛出 `InterruptedException` 并清除中断标志。
  * 若线程处于 RUNNABLE 状态，仅设置标志位，线程需通过 `isInterrupted()` 主动轮询来决定是否响应。

`yield()`：一种提示性的静态方法。
* **语义**：暗示调度器当前线程愿意放弃当前的时间片，从 Running 转为 Ready。
* **局限性**：调度器可以忽略此提示。且让出 CPU 后，该线程可能立即再次被调度执行。它不会导致线程进入阻塞状态，仅保持在 RUNNABLE。

---

#### 线程封闭策略

##### 重量级策略：ThreadLocal

> ThreadLocal在设计之初**先天假设条目极少，因此才采用线性探测法**，而线性探测法在条目较少时比链表法更快。因为Entry数组在物理内存上是紧凑的，利好Cpu预读取到缓存行，因此发生哈希冲突时真正数据所在的槽位大概率已经加载到了该缓存行中。具体的业务场景需要考虑实际的数据量，应控制对象在几十个以内，且该对象应是重量级的上下文Context，例如**Session 对象、登录信息**

ps：数据库连接绝对不能放进ThreadLocal，线程独占数据库连接在高并发场景下会导致连接池快速耗尽，后续线程全部阻塞，系统假死

> 优先使用轻量级的栈封闭，而不是重量级的`ThreadLocal`。如果必须使用`ThreadLocal`，**务必在方法出口处调用 `remove()` 以释放底层数组槽位**，**避免 `ThreadLocalMap` 因堆积过久而触发频繁的扩容和线性探测开销。**

**ThreadLocal实现**
- 线程持有类型为 `ThreadLocal.ThreadLocalMap`的变量`threadLocals`，底层是`Entry[] table`数组
- `Entry`**包装业务对象`ThreadLocal<T>`的引用作为VALUE**
- `Entry`继承`WeakReference<ThreadLocal<?>>`，**实例化`Entry`对象作为KEY**
  - `WeakReference<ThreadLocal<?>>`继承`Reference<T>`，在`Reference<T>`的字段`referent`中存储KEY：`ThreadLocal<?>`
- **`ThreadLocal<T>`对象是与ThreadLocal交互的凭证和入口**
  - 使用实例化时分配到的`threadLocalHashCode`计算索引
  - **利用线性探测法解决哈希冲突**
  - **通过`ThreadLocal<T>`对象调用set、get、remove方法**
- `Entry[] table`**数组容量**：$2^n$，初始容量为16，利用位运算 `(len - 1) & hash` 快速计算索引

```JAVA
public class ThreadLocal<T> {
    final int threadLocalHashCode = nextHashCode();
    public void set(T value) {
        //1. 获取当前线程实例对象
        Thread t = Thread.currentThread();
        //2. 通过当前线程实例获取到ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        if (map != null)
        //3. 如果Map不为null,则以当前ThreadLocal实例为key,值为value进行存入
        map.set(this, value);
        else
        //4.map为null,则新建ThreadLocalMap并存入value
        createMap(t, value);
    }

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    public void remove() {
        ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null) {
            m.remove(this);
        }
    }
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        private static final int INITIAL_CAPACITY = 16;
        private Entry[] table;
        private int size = 0;
    }
}
```


**`remove`方法**
- 核心是通过`ThreadLocalMap`对象调用其`remove()`方法
- **Entry对象本身是弱引用，该弱引用指向`ThreadLocal<T>`对象，Entry对象内部维护一个强引用，该强引用指向实际的业务对象**
- `ThreadLocalMap`的`remove()`方法执行下列动作
  - `Entry`继承`WeakReference<T>`继承`Reference<T>`，将`Reference<T>`的字段`referent = null`
  - `table[i].value = null`
  - `table[i] = null`
  - **修复哈希探测链**
    - 继续向后扫描，直到遇到下一个 null 槽位为止
    - 对于扫描到的每一个非空 Entry：重新计算其哈希位置 $h$
    - 如果 $h$ 不等于当前位置（说明它处于被冲突挤占的位置），且 $h$ 到当前位置之间出现了空位（刚刚被我们删除的坑），则将该 Entry 搬迁到离 $h$ 更近的空闲位置。
- 实例化`ThreadLocal<T>`对象后必须通过`remove`方法删除，**防止内存泄漏，保护ThreadLocal的查找性能**
- 如果使用线程池，确保线程回到池中时，其ThreadLocal已清空

**内存泄露**：
- **长期存在的线程，未调用`remove`方法之前就丢失`ThreadLocal<T>`对象的所有强引用，而`threadLocals-table[]-table[i]`链条导致`Entry`被线程强引用，VALUE又被`Entry`强引用，且 Entry 被线程强引用，导致业务对象无法被常规 GC 回收**
- 丢失`ThreadLocal<T>`对象的所有强引用后，**堆上的 `ThreadLocal<T>` 实例只有一条来自 `Entry` 的弱引用**，**GC回收该 `ThreadLocal<T>` 实例**，`Entry` 中的 `WeakReference<T>` 字段内部的 `referent`字段被置为 `null`
- **陈旧条目**定义：KEY为 `null`，VALUE 不为 `null`

**顺带清理机制**
- **原因**：ThreadLocalMap 无法收到 GC 的通知，只能采用被动式的清理策略
- **时机**：执行定位槽位任务发生哈希冲突向后探测时
- **行为**：一旦遇到一个陈旧条目，它不仅会清理当前这个，还会继续向后扫描，直到遇到 null 槽位为止
- 虽然 ThreadLocalMap 存在顺带清理机制，但该机制依赖于后续的哈希冲突或特定操作，具有极大的滞后性和不确定性。**若线程不再操作 ThreadLocal 或探测未覆盖该槽位，该对象将长期驻留内存，构成事实上的内存泄露**。

> 如果`ThreadLocal<T>`对象没有调用`remove`方法就丢失引用，直接判定为内存泄露


`ThreadLocal<T>`对象提供的`set`方法遵循引用赋值，如果两个线程的`ThreadLocal<T>`对象**set了相同对象的引用**，将**造成虚假的线程隔离**

---


##### 轻量级策略：栈封闭

[虚拟机栈](#线程私有)是线程私有的，局部变量存在于栈帧中，随着方法的调用而创建，随着方法的结束而销毁，不同线程的栈内存是完全隔离的，因此**栈封闭的变量天然具备线程安全性**
- 基本数据类型直接存储在栈帧的局部变量表中，**绝对线程安全**，无法被其他线程引用。
- 局部变量表中存储的是**对象的引用**，而对象实例本身通常存储在**堆**中，要实现栈封闭，必须确保**该对象的引用永远不会逃逸出当前方法**：**不被返回、不被赋值给静态变量或成员变量**
- JIT利用[逃逸分析](#解释器、即时编译器JIT)技术优化栈封闭场景，打破对象只能分配在堆上的铁律。若对象未逃逸，JIT 不会创建实际的堆对象，而是将该对象的成员变量拆解为若干个被方法使用的**标量**（基本数据类型），直接分配在**栈**或**寄存器**上，达到**零 GC 开销**


---

#### 线程组与优先级调度


**在 Java 并发编程的早期模型中**，`ThreadGroup`（线程组）和线程优先级（Thread Priority）提供了对线程生命周期和 CPU 资源分配的基础管理能力。尽管现代并发工具（如 `ExecutorService`）已成为主流，但理解底层的线程组层级结构及优先级的操作系统映射机制，对于深入掌握 JVM 线程调度模型依然具有重要意义

- **现代编程中ThreadPoolExecutor 完全接管了生命周期管理：线程的创建、保活、销毁由线程池的逻辑控制。异常处理由 Future 或 afterExecute 钩子处理**。ThreadGroup 在这里仅仅是为了满足 JVM 的非空检查，或者用于一些简单的统计和安全检查（SecurityManager），不再承担业务逻辑。
- Java 语言规范强制要求：每一个 Thread 对象在创建时，必须属于一个 ThreadGroup。 如果不指定，它会默认加入到创建它的那个线程（父线程）所在的组。因此，现代线程池中的每一个 Worker 线程，底层依然隶属于某个 ThreadGroup。**ThreadGroup 内部依然持有数组，但在池化管理下，其副作用被屏蔽了。**
- **优先级目前唯一保留的一点用处是设置 Thread.MIN_PRIORITY 给那些做后台清理工作的线程，作为一个“微弱的暗示”给 OS：如果 CPU 很忙，可以先挂起我。**

---

`java.lang.ThreadGroup` 是 Java 用于批量管理线程的机制，采用树状（Tree）数据结构，将线程和子线程组，组织成层级体系。
* **树状层级结构**：每个 `Thread` 必然属于一个 `ThreadGroup`。如果在创建线程时未显式指定组，新线程将默认加入创建它的父线程所在的组。
  * **根节点**：JVM 启动时会创建名为 `system` 的根线程组。
  * **Main 线程组**：`system` 组下挂载 `main` 线程组，运行 `public static void main` 的主线程即属于此组。
  * **层级引用**：`ThreadGroup` 内部持有 `Thread[] threads` 数组和 `ThreadGroup[] groups` 数组，形成了向下引用的强引用链。
    * `Thread[] threads`：存放归属于该组的所有活跃线程。
    * `ThreadGroup[] groups`：存放归属于该组的所有子线程组。


* 核心管理功能
1. **状态检查**：通过 `activeCount()` 获取活跃线程数，或使用 `list()` 打印组内所有线程的调试信息。
2. **批量控制**：
   * **中断**：调用 `group.interrupt()` 可触发组内所有线程的中断信号。
   * **异常处理**：`ThreadGroup` 实现了 `Thread.UncaughtExceptionHandler` 接口。当组内线程抛出未捕获异常时，JVM 会回调 `uncaughtException` 方法，允许开发者在组级别统一处理异常。
* 设计缺陷与废弃现状：尽管设计初衷良好，但 `ThreadGroup` 在现代开发中已被视为**过时（Obsolete）**的设计，主要原因如下：
  * **线程安全不足**：部分方法（如 `destroy`）并非线程安全。
  * **功能重叠**：`java.util.concurrent` 包下的线程池提供了更强大、安全的任务分组与管理能力。
  * **生命周期管理困难**：**复杂的层级结构容易导致内存泄漏（无法回收被引用的线程对象）**。
    * 只要父级线程组活着（通常由 JVM 维护，很难死掉），它引用的所有子组和子线程都无法被 GC 回收，即使已经不再需要它们，或者引用它们的逻辑变量已经置空
    * ThreadGroup 必须时刻监控线程的生命周期，增加了锁的竞争，而且一旦线程因为异常没有正确通知组，或者组没有被显式销毁，这些引用就会一直悬挂在内存中。
    * 无法自动清理：现代的池化技术在线程空闲时可以自动回收线程资源。但 ThreadGroup 这种强引用链结构，默认是倾向于“持有”而非“释放”。如果开发者忘记手动调用 destroy()（且该组内必须无活跃线程才能销毁），这个子树就会永久驻留内存。

> **Best Practice**: 在新系统中，应优先使用线程池（ThreadPoolExecutor）而非手动创建 `ThreadGroup`。

---


**线程优先级**：Java 采用**抢占式（Preemptive）**调度策略，优先级机制旨在为调度器提供“建议”，以决定哪个线程更应获得 CPU 时间片。
- Java 定义了 1 到 10 的优先级范围，通过 `Thread` 类中的常量表示：
  * `Thread.MIN_PRIORITY` (1)
  * `Thread.NORM_PRIORITY` (5) - **默认值**
  * `Thread.MAX_PRIORITY` (10)
- 操作系统映射的不确定性：Java 优先级的核心问题在于**平台无关性与底层 OS 调度的冲突**。
  - JVM 定义了 10 个级别，但底层操作系统可能仅支持更少的级别。例如，某些 Linux 发行版可能忽略 Java 优先级，或将其映射到 OS 的 3 个级别（低、中、高）。
  * **优先级塌陷**：不同的 Java 优先级可能在 OS 层面被映射为相同的优先级。
  * **不可靠性**：**不能依赖优先级来控制业务逻辑的执行顺序**（如依赖高优先级线程先完成任务）。

- **优先级倒置与饥饿**
  * **优先级倒置**：低优先级线程持有锁，导致高优先级线程被阻塞。
  * **线程饥饿**：在某些高负载系统下，低优先级线程可能永远无法获得 CPU 时间片。


**`ThreadGroup` 对组内线程的优先级具有强制约束力。**
* **最大优先级（Max Priority）**：`ThreadGroup` 维护一个 `maxPriority` 属性。
* **约束规则**：当调用 `thread.setPriority(newPriority)` 时，如果 `newPriority` 超过了该线程所在组的 `maxPriority`，则线程的实际优先级将被截断为组的最大优先级。

---

### 线程安全与锁

#### 无锁编程

##### *乐观锁CAS* （CAS 原理）

##### *魔法类 Unsafe* （Unsafe 类）

##### *原子操作类Atomic* （Atomic 包）

##### LongAdder。

#### synchronized内置锁

* *synchronized关键字*
内置锁：*synchronized的四种锁状态*（锁升级过程：偏向->轻量->重量）。
* *深入浅出偏向锁*

#### JUC 显式锁

* *锁分类和JUC*

##### AQS 骨架

* *抽象队列同步器AQS* （CLH 队列设计模式）。
* *线程阻塞唤醒类LockSupport*

##### *重入锁ReentrantLock* （ReentrantLock 公平/非公平）。

* *等待通知条件Condition*

##### *ReentrantReadWriteLock* （与 StampedLock）。

## JUC 并发工具集

### 并发流程控制

* *通信工具类*

#### CountDownLatch

#### CyclicBarrier

#### Semaphore.

### 数据交换Exchanger

## 并发集合

* *Java的并发容器*

### ConcurrentHashMap

* *ConcurrentHashMap*
JDK 1.7 Segment 分段锁 vs JDK 1.8 CAS+Synchronized 的演进。

### BlockingQueue

* *BlockingQueue*
* *ConcurrentLinkedQueue*

#### ArrayBQ vs LinkedBQ

#### DelayQueue

#### SynchronousQueue

### CopyOnWrite

* *CopyOnWriteArrayList*

## 执行框架

### ThreadPoolExecutor

* *线程池*
* *ScheduledThreadPoolExecutor*


### Fork/Join

* *Fork/Join*

### 多线程上下文传递

InheritableThreadLocal

TransmittableThreadLocal

### 虚拟线程

## 并发实战与避坑

### *生产者-消费者模式*

### 死锁排查与避免

### ThreadLocal 内存泄漏

### 线程池参数配置最佳实践