# 并发编程

## 线程基础

在操作系统层面，并发执行的单元被划分为**进程与线程**
* **进程Process**：[资源分配的最小单位](#进程虚拟地址空间)
  * 每个进程代表了一个正在运行的程序实例，拥有独立的内存空间和系统资源句柄
  * 进程间的资源是隔离的，进程间的通信IPC需要跨越内存边界，开销较大。
* **线程Thread**：**Cpu 调度的最小单位**，[同一进程内的多个线程共享进程的堆内存和方法区](#线程共享)。[每个线程仅维持独立的程序计数器、虚拟机栈和本地方法栈](#线程私有)。
  * 线程创建和切换的开销远小于进程
  * 同进程下的线程通过共享内存通信，效率高但需处理线程安全问题
  * 除守护线程之外的所有线程结束运行时，整个进程终止
  * 进程的销毁将强制结束其所有线程
---


**Java 线程的底层机制**
- Java 线程在主流JVM中通常采用 **1:1 的内核线程模型**。即每一个 Java `Thread` 对象都直接映射到一个操作系统的内核线程，即**Java 线程的创建、销毁以及调度，本质上都依赖于操作系统的系统调用**。

**上下文切换开销**
- **上下文切换**：当一个线程的时间片耗尽或发生阻塞时，Cpu需从当前线程切换至另一线程
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

**线程内存开销**
* **栈内存**：默认情况下，每个线程分配约 1MB 的栈空间（可通过 `-Xss` 参数调整）。若系统创建数千个线程，将消耗数 GB 的内存，极易诱发 `StackOverflowError` 或 `OutOfMemoryError`。
* **元数据开销**：内核还需维护线程控制块（TCB）等数据结构以管理线程状态和调度优先级。


### Java 线程的实现

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

### 线程调度与控制

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


### 异步获取线程执行结果


`Runnable` 接口无法返回结果或抛出异常，自 JDK 1.5 起，`java.util.concurrent` 包引入了 `Callable`、`Future` 及 `FutureTask` 等构建块，完善了异步计算模型
1. **无法返回执行结果**：`run` 方法没有返回值，导致主线程无法直接获取子线程的计算结果。若需获取结果，通常不得不依赖共享变量或复杂的线程间通信机制，增加了代码的耦合度和同步控制的复杂度。
2. **无法抛出受检异常**：`run` 方法未声明抛出任何受检异常（Checked Exception），这意味着任务执行过程中的错误必须在方法内部被捕获处理，无法向上传递给调用者。

#### Callable接口

`Callable` 实例不直接通过 `Thread` 类运行，而是提交到 `ExecutorService` 线程池，由线程池管理
* **泛型支持**：`call()` 方法返回一个泛型类型 `V` 的结果
* **异常传播**：方法签名中声明了 `throws Exception`，允许任务在执行过程中抛出异常，并由任务的调用方进行捕获和处理

```java
public interface Callable<V> {
    V call() throws Exception;
}

```

#### Future

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

####  FutureTask

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

#### CompletableFuture


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
### 线程生命周期：状态模型与流转控制机制

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

### 线程封闭策略

#### 重量级策略：ThreadLocal

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


#### 轻量级策略：栈封闭

[虚拟机栈](#线程私有)是线程私有的，局部变量存在于栈帧中，随着方法的调用而创建，随着方法的结束而销毁，不同线程的栈内存是完全隔离的，因此**栈封闭的变量天然具备线程安全性**
- 基本数据类型直接存储在栈帧的局部变量表中，**绝对线程安全**，无法被其他线程引用。
- 局部变量表中存储的是**对象的引用**，而对象实例本身通常存储在**堆**中，要实现栈封闭，必须确保**该对象的引用永远不会逃逸出当前方法**：**不被返回、不被赋值给静态变量或成员变量**
- JIT利用[逃逸分析](#解释器、即时编译器JIT)技术优化栈封闭场景，打破对象只能分配在堆上的铁律。若对象未逃逸，JIT 不会创建实际的堆对象，而是将该对象的成员变量拆解为若干个被方法使用的**标量**（基本数据类型），直接分配在**栈**或**寄存器**上，达到**零 GC 开销**


---

### 线程组与优先级调度


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


## 线程问题

### 线程安全问题
**线程安全**：在**多线程环境**下，通过**同步确保多线程访问共享变量**的代码符合预期逻辑，**正确的同步需要处理并发编程三大特性**：
- **同步/异步的语义**：
  - 在JavaIO和Future中，同步指是否等待数据就绪（同步非阻塞指在等待就绪的过程中可以完成其他任务），异步指数据就绪后调用回调函数
  - **在并发控制中，同步和异步指线程间是否有协同与互斥**
- **原子性代码块**：**在逻辑上不可分割**，其**中间状态对外部线程不可见**，称为临界区
  - 即使正在执行临界区代码的线程发生上下文切换，其他**受同步机制约束的线程**也无法进入该临界区
  - 若有线程未受同步机制约束则将破坏原子性，可以观测到其他线程修改共享变量操作的开始状态、中间状态、结束状态，造成严重的数据不一致和逻辑错误问题
  - **数据库事务的原子性**确保一个或一组事务要么不可中断地全部执行，要么都不执行，若执行错误就执行回滚操作
  - **并发原子性**通过线程执行的互斥性确保不可观测到中间态
- **可见性共享变量**：当一个线程修改了可见性共享变量的值，其他线程能够立即得知这一修改。
- **有序性代码块**：避免指令重排序后，并发环境下的代码执行顺序不符合逻辑预期


---


#### 线程原子性与上下文切换

程序员希望高级语言中的一行代码是原子性的，但是一条代码往往不是原子性的，**对Cpu而言一行代码往往会编译为多条指令，因为Cpu无法辨别哪些指令属于一个不可分割的整体，所以可以在多条指令执行的间隙执行上下文切换，导致其他线程可以获取共享变量的中间状态，引发数据不一致问题**
- 在两个未同步的线程中，对共享变量i执行i++操作：对Cpu而言i++是Read、Modify、Write三个原子性指令的组合，当上下文切换在原子性指令执行的间隙发生时，其他线程将重复修改共享变量，导致更新丢失

```JAVA
public class Main {
    private static int counter = 0;

    public static void main(String[] args) throws InterruptedException {
        // 创建两个线程，每个线程自增 10,000 次
        Runnable task = () -> {
            for (int i = 0; i < 10000; i++) {
                counter++; // 非原子操作
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        // 最终输出结果小于 20000
        System.out.println("理论预期值: 20000");
        System.out.println("实际运行值: " + counter);
        System.out.println("丢失更新数: " + (20000 - counter));
    }
}

```

---


#### 线程可见性与Cpu多级缓存

现代计算机通过**Cpu多级缓存**解决了Cpu运算速度与主存读写速度不匹配的问题，通过**MESI 缓存一致性协议**解决**缓存一致性**问题，但是需要由软件自行解决**可见性**问题
- **局部性原理**：缓存的有效性基于两个关键假设
  - **时间局部性**：如果一个数据被访问，那么近期它极可能再次被访问
  - **空间局部性**：如果一个数据被访问，那么它相邻的数据也极可能很快被访问，因此多级缓存读写主存**以缓存行为单位**
- **Cpu多级缓存架构**

| 层级 | 真实位置 | 模拟耗时 (假设 CPU 算一下是 1秒) | 容量 | 角色隐喻 |
| --- | --- | --- | --- | --- |
| **寄存器 (Registers)** | **CPU 核心内部** | **1 秒** (即时) | 极小 (几百字节) | **手中的加工件**。CPU 只能直接计算这里的数据。 |
| **L1 Cache** | **CPU 核心内部** | **3 ~ 4 秒** | 小 (32KB - 64KB) | **工位上的工具箱**。伸手就能拿。 |
| **L2 Cache** | **CPU 核心内部** | **10 ~ 12 秒** | 中 (256KB - 512KB) | **身后的背包**。转身就能拿。 |
| **L3 Cache** | **多核共享** | **40 ~ 50 秒** | 大 (MB ~ 几十 MB) | **房间里的共享货架**。所有核心都能看到。 |
| **主存 (DRAM)** | **插在主板上** | **200 ~ 300 秒 (几分钟)** | 巨大 (GB ~ TB) | **楼下的仓库**。去一趟很慢。 |

- **MESI 缓存一致性协议**：确保多个Cpu核心同时缓存主存中的同一行数据时的数据一致性
  - 缓存行共四种状态
    - M-Modified：**缓存行被修改**，与主存不一致，且只存在于当前 Cache 中。
    - E-Exclusive：**缓存行与主存一致**，且只存在于当前 Cache 中。
    - S-Shared：**缓存行与主存一致**，可能存在于多个核心的 Cache 中。
    - I-Invalid：**缓存行已失效**
- **寄存器导致读操作可见性延迟**：
  - 寄存器不在MESI缓存一致性协议生效范围内，Java即时编译器JIT分析代码执行次数，并在寄存器中维护**高频访问的共享变量副本**，不再从Cpu高速缓存中重新读取共享变量
  - 主线程修改主存并同步更新到执行子线程的Cpu的高速缓存中，但是子线程的Cpu在寄存器中维护了变量副本，不会感知其修改，导致可见性延迟


```JAVA
public class Main {
    public static boolean flag = true;
    //空循环并且让主线程睡眠一会儿，让JIT编译器检测热代码将flag放在寄存器里
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            System.out.println("子线程：开始执行，等待 flag 变为 false...");
            while (flag) {}
            //即使一秒后主线程修改flag，子线程也无法退出
            System.out.println("子线程：检测到 flag 变为 false，退出循环！");
        }).start();
        Thread.sleep(1000);
        flag = false;
        System.out.println("主线程：已将 flag 修改为 false");
    }
}
```

- **无效化队列导致读操作可见性延迟**
  - 核心A修改缓存行并发送失效信号，通知其他核心丢弃过期缓存行，拥有该缓存行的核心立即发送确认信号并将更新缓存行的任务压入**无效化队列**，随后该核心准备读取该变量，由于仍没有执行更新，所以读取到旧值，造成短时间的可见性延迟

- **读缓冲导致读操作可见性延迟**
  - 为了避免核心等待内存或L3缓存读取数据，将提前读取任务压入读缓冲，即使该核心还没执行该读指令，其他核心修改数据，该核心还未接收到失效信号或仍未处理失效化队列，导致读缓冲的数据是过期的，造成缓存不一致

- **Store Buffer写缓冲导致写操作可见性延迟**：
  - Cpu核心修改缓存行时必须发送Invalidate 消息给其他核心并等待ACK回复后写入主存，为避免Cpu流水线卡顿先将修改写入缓冲区并继续执行后续指令；其他核心收到Invalidate 消息后使目标缓存行失效，并重新从主存读取目标缓存行，可能因为修改仍在写缓冲区而读取到旧值，导致逻辑上的指令重排序，即实际上指令的执行顺序是先写后读，而逻辑上的指令执行顺序却变成了先读后写，在单线程环境下，由于是一个Cpu核心，所以该逻辑重排序永远不会发生
  - 线程1、2的Cpu核心都仅执行两条代码，如果没有写缓冲，那么一定是先写后读，则最后结果可能是(1,1), (0,1), (1,0)，当先读后写发生时，a和b都读取到旧值0（实际上已经执行了原子指令x和y的赋值），导致逻辑上的错误结果(0,0)



```java
public class Main {
    private static int x = 0, y = 0;
    private static int a = 0, b = 0;
    private static int count01=0;
    private static int count10=0;
    private static int count11=0;

    public static void main(String[] args) throws InterruptedException {
        int count = 0;
        while (true) {
            count++;
            x = 0; y = 0; a = 0; b = 0;

            // 线程 1
            Thread t1 = new Thread(() -> {
                x = 1;  // 写操作
                a = y;  // 读写操作
            });

            // 线程 2
            Thread t2 = new Thread(() -> {
                y = 1;  // 写操作
                b = x;  // 读写操作
            });

            t1.start();
            t2.start();
            t1.join();
            t2.join();

            if (a == 0 && b == 1) {
                count01++;
            }
            if (a == 1 && b == 0) {
                count10++;
            }
            if (a == 1 && b == 1) {
                count11++;
            }

            if (a == 0 && b == 0) {
                System.err.println("第 " + count + " 次实验出现异常：(a=0, b=0)");
                System.out.println("(0,1)出现次数："+count01);
                System.out.println("(1,0)出现次数："+count10);
                System.out.println("(1,1)出现次数："+count11);
                break;
            }
        }
    }
}
```


**伪共享与缓存行填充**
- 无关线程安全，但是影响线程性能；
- Cpu高速缓存和主存之间的读写以行为单位，在64位机中，缓存行是64字节
- **伪共享**：如果不相关的变量A和变量B在同一个缓存行中，线程修改变量A导致缓存行失效，拖慢其他线程对变量B的读写
- **缓存行填充Padding**：通过填充大量long类型空白行，将相邻的变量挤到不同的缓存行里

---


#### 线程有序性与指令重排序

对Cpu而言程序只不过是一段指令，即**指令流水线**，为了不让流水线中断，需要指令重排序
```JAVA
a=b+c
e=e-f
```
- 考虑上述两条代码，翻译为指令都是加载和计算指令，为了不在加载时停顿，通过指令重排序连续加载b、c、e、f，然后再顺序执行两次计算
- 指令重排序保证单线程环境下程序语义不变，但在多线程环境下可能由于乱序而出现程序错误
**指令重排序分为三种**
- **编译器优化重排**，编译器在不改变单线程程序语义的前提下，重新安排语句的执行顺序。
- **指令并行重排**，现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性(即后一个执行的语句无需依赖前面执行的语句的结果)，处理器可以改变语句对应的机器指令的执行顺序。
- **多级缓存导致的逻辑上的指令重排序**


---



### Java内存模型

> **首先区分[Java运行时内存区域](#jvm运行时内存区域)和Java内存模型两个不同又有联系的概念**，Java运行时内存区域是**JVM运行时的具体物理内存划分**，如堆、栈、方法区，而Java内存模型是一套抽象的逻辑协议
- JMM协议**屏蔽了底层复杂的硬件架构**，无需从Cpu、寄存器、多级缓存和汇编指令的角度，而是从线程与内存交互的视角处理线程安全问题
- **JMM协议由四部分组成：逻辑架构、8种原子性操作、Happens-Before 规则、内存栅栏；**

---



#### 逻辑架构

1. **主存**：所有线程共享的内存区域（注意并不是实际的主存DRAM，而是抽象的逻辑概念）
2. **工作内存**：每个线程独占的内存区域，例如读缓冲、写缓冲、失效化队列、寄存器、L1L2高速缓存
3. **交互协议**：
   - 线程对共享变量的所有操作都视为仅在工作内存中进行。
   - 线程首次获取变量时，从主存拷贝到工作内存，修改后视情况刷新回主存
   - 不同线程之间无法直接访问彼此的工作内存，线程间的通信必须通过主存作为中介完成。

---


#### 8种原子性操作

JMM定义了8种不可分割的原子操作，是线程读写主存的最小单位；JMM将底层的Cpu和指令流抽象为线程和8种原子性操作。除非一定要涉及底层硬件，后续文章统一从线程和内存操作的抽象视角出发审视源码
- **Read/Load**：从主存传输到工作内存，把 read 得到的值放入工作内存的变量副本中。

- **Use/Assign**：把工作内存的值传递给Cpu，Cpu处理完后赋值给工作内存变量。

- **Store/Write**：把工作内存的值传送到主存，把 store 得到的值放入主存变量中。

- **Lock/Unlock**：作用于主存，将变量标识为线程独占或释放状态

**约束**
- Read/Load 与 Store/Write 必须成对出现，因为有读可见性延迟、写可见性延迟和指令重排序，所以两个内存操作不一定相邻
- 执行Assign，则必定**在之后某个时间点**执行Store/Write；不执行Assign，则不允许执行Store/Write
- Lock/Unlock是`synchronized`重量级锁的底层抽象
  - **排他性**：同一个变量在同一时刻只允许一个线程对其进行Lock操作，如果该变量已被锁，其他线程必须阻塞等待；
  - **不可分割的操作块**：成功执行Lock操作的线程将获得一个语义上的原子块。在这个块内，无论它执行了多少次 Read/Use/Assign，其他线程都无法介入该变量的交互。同时清空工作内存中该变量的副本，执行Read/Load操作从主存获取最新值
  - **UnLock**：执行多少次Lock操作就必须执行多少次Unlock操作，才能完全释放变量的所有权；同时执行 Unlock操作前，执行Store/Write =操作强制将结果推回主存

**实现同步的最小周期**：
- Read-Load-Use：线程放弃本地缓存从主存获取最新值，避免线程永远在过期的副本上工作
- Assign-Store-Write：线程修改数据并在以后某个时间点写回主存
- Lock/Unlock：定义临界区，临界区内的操作是**无法被其他线程分割的整体**

---


#### 内存栅栏

内存栅栏**屏蔽操作系统底层的差异**，通过**防止不正确的指令重排序**、**强制硬件同步缓存**实现有序性与可见性。内存栅栏是一个确保有序性的**同步点**（分水岭），将指令流分为栅栏前与栅栏后。
- **LoadLoad**： 
  - 确保栅栏前的所有 Load 操作完成读取后，才执行栅栏后的 Load
  - **读可见性**：JMM 保证栅栏后的 Load 会放弃工作内存副本，从主存读取最新值
- **LoadStore**：、
  - 确保栅栏前的所有 Load 操作完成后，才执行栅栏后的 Store 操作。
  - 该同步点**仅负责有序性，不直接保证可见性**，仅防止“读未完、写已到”的逻辑错误
  - 仅存在于 JMM 逻辑定义中，硬件层面常被合并入读栅栏
- **StoreStore**：
  - 确保栅栏前的所有 Store 操作同步到主存，对其他核心可见后，才执行栅栏后的 Store。
  - 该同步点不影响Load操作通过栅栏重排序提前执行
  - **确保写可见性**
- **StoreLoad**：
  - 栅栏前的所有Store操作同步到主存之后，才能执行栅栏后的Store、Load指令，同时JMM保证栅栏后的Load操作会放弃工作内存中的副本，从主存读取最新值
  - 该同步点是**全能栅栏**，**同时确保写可见性和读可见性**

确保一个变量**在所有线程的工作内存和主存中的数据一致性**，需要在该变量所有读写操作的正确位置施加正确的栅栏组合
- **读操作后连续插入**LoadLoad和LoadStore栅栏
  - 读操作 -> [LoadLoad] -> [LoadStore]：读取最新值，并且禁止后面的读写操作提前执行
- **写操作前后分别插入**StoreStore和StoreLoad栅栏
  - [StoreStore] -> 写操作 -> [StoreLoad]：先等前面的写操作完成并刷回主存，执行完本次写操作后立即刷新到主存，并且禁止后面的读写操作提前执行
- 仅对读操作或写操作施加栅栏组合仅能确保有序性，但无法确保跨线程的可见性；


---


#### Happens-Before规则

JMM逻辑架构抽象了硬件底层架构，8种原子性操作抽象了硬件底层操作，内存栅栏封装了Cpu控制指令流执行顺序的硬件底层操作，JMM进一步将复杂的内存栅栏组合封装为一种**可见性声明**，即**Happens-Before规则**：**如果操作A Happens-Before 操作B，那么JMM保证：操作 A 的结果对于操作 B 而言是可见的，且操作 A 的顺序位在 B 之前。不会对这两个操作重排序**


JMM定义8种天然的Happens-Before规则
1. **程序次序规则**：在一个线程内，书写在前面的操作 Happens-Before 后面的操作。
   - 不保证其他线程的观测结果，即由于指令重排序，其他线程可能先看到后面操作的执行结果，
   - 即使发生指令重排序，由于指令重排序技术都确保单线程执行结果的正确性，同一个线程仍能天然地正确观测到前面操作的执行结果，例如先初始化实例，再调用引用更改器不会报错空指针
2. **Lock/Unlock规则**：一个 unlock 操作 Happens-Before 后面对同一个锁的 lock 操作
   - 执行unlock操作前必须将工作内存中的修改刷回主内存，执行lock操作必须先load最新值
3. **Volatile规则**：对一个 volatile 变量的写操作 Happens-Before 后面对这个变量的读操作。
   - volatile 变量写操作后插入StoreLoad 全能屏障，确保后续读操作一定读取到最新变量
4. **线程启动规则**：Thread 对象的 start() 方法 Happens-Before 该线程中的每一个动作。
   - 保证了主线程在启动子线程前修改的共享变量，在子线程开始执行时都是可见的。
5. **线程终止规则**：线程中的所有操作都 Happens-Before 其它线程检测到该线程已经终止
   - 其他线程执行join等操作时，一定能读取到该线程的所有修改
6. **对象终结规则**：一个对象的初始化完成Happens-Before 该对象被GC之前
7. **传递性**：如果 A Happens-Before B，且 B Happens-Before C，那么 A Happens-Before C
   - 最强大的规则，将上述孤立的七条规则联系在一起，例如与程序次序规则结合，将普通变量的读写操作书写在 volatile写操作之前，或者将普通变量的读写书写在临界区中，确保隐式同步，

---





### volatile关键字

在讲解内存栅栏时提到：确保一个变量**在所有线程的工作内存和主存中的数据一致性**，需要在该变量所有读写操作的正确位置施加正确的栅栏组合，而**volatile关键字就是对这一复杂栅栏组合的封装**，JMM保证
- 在每次volatile读操作后面连续插入LoadLoad和LoadStore栅栏，保证volatile读生效，才能执行后面的操作，并且确保后面的读操作都是最新值
- 在volatile写操作前后分别插入StoreStore和StoreLoad栅栏。
  - StoreStore保证前面的写操作不会再volatile写操作后执行，以此确保它们都刷回主存
  - StoreLoad保证后面的读写操作不会在volatile写操作前执行，并确保后续读都是最新值

volatile确保可见性和有序性，但不确保原子性
- 在讲解线程原子性与上下文切换时提到，在指令空隙发生上下文切换导致其他线程获取到共享变量的中间状态；将count++翻译成逻辑上的原子性操作

```text
Read/Load i

[LoadLoad 屏障]
[LoadStore 屏障]

Use/Assign i+1

[StoreStore 屏障]
Store/Write i
[StoreLoad 屏障]

```
- volatile之所以**无法保证原子性**，原因是单次读操作和单次写操作都受到栅栏保护，但是读写操作之间没有受到栅栏保护，即Use/Assign i+1指令没有任何同步措施，在该语句执行前发生上下文切换，其他线程重复操作破坏线程原子性

**可见性的传递性**：得益于Happens-Before规则提到最重要的传递规则
- 普通变量写操作书写在volatile写操作之前时，将一起同步到主存；
- 普通变量读操作书写在volatile读操作之后时，能看到最新值


---

### synchronized

`synchronized`的使用

- **`synchronized`非静态同步方法**：**默认以this对象作为对象锁**
   - **`this`对象锁**被一个线程持有时，其他线程不能调用该对象的**任何**`synchronized` 实例方法
- **`synchronized`静态同步方法**：**默认以`this.Class`类作为类锁**
   - **`this.Class`类锁**被一个线程持有时，其他线程不能调用该`this.Class`类的**任何**`synchronized` `static`方法
- **`synchronized`同步代码块**，需要指定加锁对象（`this`、`this.Class`、其他对象），对给定对象加锁，进入同步代码块前需要获取指定对象锁
- 锁范围：**类锁、对象锁A、对象锁B互相独立**，且它们都不阻塞非同步代码的执行
  - 类锁只管静态同步方法，对象锁A只管A的实例同步方法、对象锁B只管B的实例同步方法

在jdk1.6之前，**`synchronized`是封装Lock/Unlock操作的重量级锁**
- **原子性**：
  - **Lock/Unlock**将一个变量的状态更改为某个线程独占或空闲，该变量即锁，Lock和Unlock操作之间是**临界区**，线程进入临界区需要先获取锁，消除多线程交替执行 Read-Use-Assign-Store 的可能性
- **可传递可见性**
  - Lock操作：强制失效当前线程的工作内存，保证临界区中的所有读操作必须从主存重新获取
  - Unlock操作：执行Unlock之前强制将临界区内所有修改过的变量刷回主存
- **有序性**：
  - 由于只有一个线程能执行`synchronized`临界区，所以`synchronized`临界区内可以安全的指令重排序，但不允许临界区外的操作重排序到临界区内
  - 即使临界区内发生指令重排序，由于串行环境语义不变，对其他线程而言也是顺序执行的
- **可重入锁**：
  - 已持有锁的对象再次申请该锁时立即成功，即Lock多少次就需要Unlock多少次，可重入锁解决单线程环境下的死锁问题

**避免未同步线程破坏`synchronized`临界区的原子性、可见性、有序性**
- `synchronized`所保证的三大特性只防君子不防小人，无论是否加了锁，**`synchronized`临界区内的变量可能被其他线程所获取**，**从而破坏临界区的可见性和有序性**，如果该线程进一步修改该变量，则**破坏了原子性**

---

### 双重检查锁定

> 双重检查锁定完美展示了volatile的可见性、有序性，以及锁的原子性、可见性、有序性，以及设计锁对象和锁范围的哲学
```JAVA
public class Singleton {
    // 1. 必须加 volatile 关键字
    private static volatile Singleton instance;

    // 2. 私有构造方法，防止外部 new
    private Singleton() {
        // 这里的初始化可能比较耗时
    }

    public static Singleton getInstance() {
        // [第一次检查]：也就是 "Double Check" 的第一层
        // 目的：如果对象已经创建好了，就不要去抢锁了，直接返回，提升性能
        if (instance == null) {
            
            // 加锁
            synchronized (Singleton.class) {
                // [第二次检查]：也就是 "Double Check" 的第二层
                // 目的：防止在第一次检查和抢锁之间，别的线程已经把对象创建好了
                if (instance == null) {
                    // 3. 问题核心点：实例化对象
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

双重检查锁定解决的问题：仅写一次其他都是读，写操作是耗时的重量级操作且需要确保在多线程环境下仅执行一次懒加载的写操作，读操作是轻量级操作

**加锁需要考虑锁范围和锁对象**
- **锁范围**：获取资源的代码块不能加锁，所以不能对getInstance加锁，应仅限于重量级的初始化资源代码块中
- **锁对象**：线程进入临界区执行重量级操作前，instance还未创建，同时写操作只能执行一次，因此只能选择类锁

**去掉volatile关键字**
只有尝试执行写操作的线程会被synchronized同步，而正在执行读操作的线程没有任何同步，可能获取到instance在临界区中的中间状态，即没有完全初始化的引用，破坏了synchronized临界区的三大特性
- 即使写操作是最简单的实例化对象，也是以下操作的复合操作，由于构造函数是耗时的，指令重排序会先执行步骤3再执行步骤2，此时其他线程获取instance时不为null，但是没有初始化完成
  1. Allocate：给对象分配内存空间。
  2. Initialize：调用构造函数，初始化对象内部成员。
  3. Assign：将引用 instance 指向刚才分配的内存地址。

**volatile**：
- instance的写操作前后将插入屏障，确保临界区内该写操作之前的所有步骤都执行完毕并刷回主存，instance写操作执行后立即刷回主存，其他线程拿到instance一定是构造好的
- instance的读操作后面插入屏障，确保读取到与主存一致的instance

**双重检查锁定**
- 第一次检查：判定是否需要竞争锁
- 第一次检查instance失效
  - 竞争到锁，instance只能是以下状态：已经构造好、没开始构造，临界区保证同步的线程无法看到彼此的中间状态
  - 第一个竞争到锁的线程执行结束，Unlock确保所有修改刷回主存
  - 第二个竞争到所的线程开始执行，Lock操作确保工作内存同步主存，一定能看到第一个线程构造好的instance，第二次检查失效，不再执行写操作，确保写操作只进行一次
- 第一次检查instance存在：由于instance是volatile变量，所以一定是构造好的对象




---



### synchronized锁升级


synchronized 锁升级是一个复杂且不可降级的过程，不可避免地涉及硬件底层。


JDK1.6之前，`synchronized`锁是**使用操作系统互斥量mutex实现的重量级锁**，线程获取锁失败时的阻塞需要在**用户态和内核态之间的上下文切换**；Jdk1.6引入**无锁、偏向锁、轻量级锁**状态，因为偏向锁的机制太过复杂，Jdk15默认禁用偏向锁状态，直到JDK18直接取消了偏向锁
- `Object`维护一个监视器作为重量级锁的同步工具，监视器包括锁、等待/通知机制（通过调用`Object` 类中的`wait()`, `notify()`, `notifyAll()`方法实现）

任意对象都可以作为锁，在MarkWord中存储锁信息
- **偏向锁状态**：MarkWord存储**线程ID和`Epoch`**
  - 已经计算过 `hashCode()`的对象无法进入偏向锁状态
  - 已经处于偏向锁状态的对象，收到 `hashCode()` 请求，会撤销偏向锁，膨胀为重量级锁，将`HashCode` 存放在 `ObjectMonitor` 类里
- **轻量级/重量级锁**：Mark Word 指向了栈帧中的`Lock Record` 或 堆中的`ObjectMonitor`，原本的 Mark Word 内容（包括 `HashCode`）被保存在了这些外部结构中

| 锁状态 | 偏向锁标识 (1bit) | 锁标志位 (2bit) | 存储内容 (Mark Word 核心区域) | 状态说明 |
| --- | --- | --- | --- | --- |
| **无锁** (Normal) | **0** | **01** | `unused` (25bit) + `HashCode` (31bit) + `分代年龄` (4bit) | 对象刚创建或未加锁。存储了对象的身份哈希码。 |
| **偏向锁** (Biased) | **1** | **01** | `Thread ID` (54bit) + `Epoch` (2bit) + `分代年龄` (4bit) | 锁偏向于某个线程。**注意：HashCode 没了**（如果此时计算 HashCode，偏向锁会被强制撤销）。 |
| **轻量级锁** (Lightweight) | 忽略 | **00** | 指向栈帧中 **Lock Record** (锁记录) 的指针 (62bit) | 发生轻微竞争。Mark Word 的原内容被复制到了线程栈的 Lock Record 中。 |
| **重量级锁** (Heavyweight) | 忽略 | **10** | 指向堆中 **ObjectMonitor** (监视器对象) 的指针 (62bit) | 发生激烈竞争。线程被阻塞，指向重量级锁监视器。 |
| **GC 标记** (Marked for GC) | 忽略 | **11** | 空 / GC 相关的标记信息 | 对象等待被垃圾回收器回收。 |

**无锁**在不同场景下的含义：
- 对象实例化后置于无锁状态；开启偏向锁时，对象创建4秒后处于**匿名偏向状态**，线程ID置0；禁用偏向锁时一直保持无锁状态直到线程尝试获取锁
  - `synchronized`锁处于无锁状态仅是指首次实例化所后一直没有线程获取锁而已
- JIT**锁消除优化**：通过逃逸分析某个对象只能被一个线程访问时，直接消除`synchronized` 关键字
- CAS：依赖CAS指令实现的无锁编程，指没有使用操作系统的悲观锁：重量级互斥量 Mutex

---


#### 偏向锁

为什么需要在对象创建4秒后再置于匿名偏向状态？
- 因为当偏向锁发生竞争时，会暂停持有偏向锁的线程，如果对象创建4秒后还处于无锁状态表示该锁处于**锁不存在多线程竞争，而且总是由同一线程多次获得的场景下**，通过偏向锁消除CAS指令，偏向同一线程，始终由该线程占有锁
- 考虑撤销4秒限制，实例化就处于匿名偏向，在JVM启动执行类加载过程时，由于内部使用了大量的 synchronized 关键字来保证类只被加载一次，系统级的锁面临剧烈的多线程竞争，会出现大量的暂停线程-偏向锁撤销
- 使用4秒限制，无锁状态只能升级为轻量级锁，在竞争失败时只是自旋或自旋失败并膨胀，不需要 Stop The World

偏向锁时间轴
- 前置条件：
  - 对象锁处于**匿名偏向状态**
  - MarkWord 内容：偏向标识 = 1，锁标志 = 01，`ThreadID` = 0 ， `Epoch` = 0。
- 线程A尝试进入临界区，获取到匿名偏向状态的锁，尝试通过CAS将MarkWord的ThreadID置为自己的ID
  - CAS成功：获取到偏向锁，在栈帧中创建Lock Record，但是指向NULL
  - CAS失败：偏向锁被线程B抢先占用，立即请求JVM 撤销 B 的偏向锁
- 在线程A持有锁后
  - 每次进入同步块前不再发起CAS操作，仅比较MarkWord的ThreadID和自己的ID是否一致
  - 退出同步块不会改动MarkWord中的ThreadID，仅清除栈帧中的Lock Record记录，表示退出同步块
  - 如果其他线程发现偏向锁已被占用，则立即请求JVM撤销A的偏向锁

**线程A撤销线程B的偏向锁**
- **只要JVM不干涉用户线程执行，就不会触发Stop the World**，撤销偏向锁和垃圾回收都会Stop the World
- JVM 等待**所有线程**到达全局安全点，检查线程B死没死，是否存在Lock Record记录，存在则表明线程B还在同步块里
  - **仅撤销一个偏向锁就需要Stop the World暂停所有线程，代价太大了，但是不这么做可能会导致数据一致性被破坏，这是绝不能被容忍的**
- 撤销偏向锁，升级轻量级锁，设置偏向标识 = 0，锁标志 = 00
- 如果线程B还在同步块内，JVM还要在线程B的栈帧内构建一个标准的Lock Record轻量级锁替代NULL，并将对象锁的MarkWord中的LockRecord引用指向线程B栈帧中的LOckRecord
- 唤醒线程B继续持有轻量级锁，线程A进入自旋状态等待线程A释放


**为什么不使用偏向锁**
- 在JDK1.6设计偏向锁时，假设的是几乎没有竞争，这种假设下永远不会撤销，也不会STW，皆大欢喜
- 但是一旦有激烈的多线程竞争，偏向锁频繁被撤销，所有线程都被迫高频率频繁卡顿，导致系统吞吐量大幅下降，偏向锁所消除的那点CAS根本得不偿失
- 因此JDK15+默认关闭偏向锁，直接用轻量级锁，也不愿意承担全员卡顿的风险，况且[现代计算机的CAS开销已经很小了](#cas)，现代轻量级锁对比理想状态的偏向锁多出的花销可以忽略不计，且偏向锁机制太过复杂


---


#### 轻量级锁


轻量级锁适用于**不存在锁竞争**的情况：**多个线程在不同时段交替获取同一把锁**
- 通过栈帧中的 `LockRecord` 和 CAS 操作避免使用操作系统层面的互斥量
- 一旦发生锁竞争，立即升级为重量级锁

线程A尝试获取轻量级锁进入同步代码块
- 线程 A 在当前线程的栈帧中创建**LockRecord**，拷贝对象锁的MarkWord到 LockRecord 中
- 尝试CAS将对象头的 MarkWord 替换为**指向自己 LockRecord 的指针**
- CAS 成功：锁标志位`00`，线程 A 获得轻量级锁，直接执行同步代码块。
- CAS 失败：锁对象已被占用，注意：只要MarkWord中存储了指针就失败，因为轻量级锁的加锁逻辑是假设对象锁处于无锁状态
  - 读取对象头当前的 MarkWord，判断**持有锁的线程是不是自己**
  - 如果是自己执行**锁重入**
  - 如果不是自己执行**锁升级**


**锁重入**
- 线程 A 在栈帧中**再压入一个 LockRecord**
- 将该 LockRecord 中的 Displaced Mark Word 设置为 **null**
- `null` 记录仅作为一个可重入锁的计数器，并不会真正释放锁，线程仅丢弃null值的LockRecord

**锁升级**
- 当前竞争线程将对象头的锁标志位修改为 **10**，并将其指向堆内存中新创建的 **ObjectMonitor**
- 竞争线程先在 `ObjectMonitor` 入口处尝试**适应性自旋**，自旋失败才进入阻塞队列，调用操作系统指令挂起线程，等待被唤醒。`_EntryList`
- 适应性自旋：在同一个锁自旋成功时，增加下次自旋的次数，否则减少或直接跳过自旋步骤直接进入阻塞队列；

**解锁**
- 线程A准备退出同步代码块前，执行以下步骤
- 弹出一个 LockRecord
- 检查 LockRecord 中的 `Displaced Mark Word` 是否为 **null**
  - 是 null：直接丢弃该记录，**解锁流程结束**
  - 不是null：需要真正释放锁
- **真正释放锁**：线程 A 尝试使用 CAS 将 LockRecord 中的 `Displaced Mark Word` 写回对象锁的MarkWord，只要不是当前LockRecord的指针就失败
  - CAS成功：没有竞争，对象头恢复为01无锁状态
  - CAS失败：在线程A运行期间，其他线程竞争锁并执行锁升级，此时对象锁的MarkWord指向堆中的 `ObjectMonitor`
    - 线程 A 根据指针找到 ObjectMonitor，释放Monitor所有权
    - 调用 `unpark/notify` 唤醒 `_EntryList` 中被阻塞的线程

---


#### 重量级锁

重量级锁依赖于操作系统的**互斥锁Mutex**实现。由于 Mutex 需要在用户态和内核态之间切换，且操作系统对线程状态的转换（阻塞与唤醒）需要相对较长的时间，因此重量级锁的原始效率较低。为了缓解这种性能损耗，避免线程频繁地挂起和恢复，JVM 在重量级锁中引入了**适应性自旋** 机制

**适应性自旋**
- 在线程真正请求操作系统阻塞之前，线程先在Cpu上适应性自旋一定次数，试图在自旋期间等到锁被释放
- **为什么要自旋？**：许多同步代码块的执行时间非常短，持有锁的时间甚至短于线程挂起和唤醒的开销。自旋可以让线程在不放弃 CPU 的情况下等待锁，从而避免昂贵的上下文切换。
- **适应性**：自旋的次数不固定，而是由**前一次在同一个锁上的自旋时间**和自旋结束后是否获取到锁决定的
  - 如果同一个锁上一次自旋成功了，且当前持有锁的线程正在运行，JVM 会认为这次也很有可能成功，进而允许自旋等待更长的时间
  - 如果对于某个锁，自旋很少成功，JVM 为了避免浪费 CPU 资源，会减少自旋次数，甚至直接省略自旋步骤，让线程直接进入阻塞状态。


**ObjectMonitor**内部结构
* **Contention List (竞争队列)**：所有请求锁的线程（如果自旋失败）将被首先放置到该竞争队列。
* **Entry List (候选队列)**：用于降低对 Contention List 的并发竞争。
* **Wait Set (等待集合)**：获得锁后调用 `wait` 方法被阻塞的线程被放置到 Wait Set
* **OnDeck**：任何时刻最多只能有一个线程正在竞争锁，该线程称为 OnDeck，即**假定继承人**
* **Owner**：当前获得锁的线程称为 Owner。
* **!Owner**：释放锁的线程。


**锁获取与阻塞流程**

当一个线程尝试获得重量级锁时，流程如下：

1. **尝试自旋**：线程首先尝试**适应性自旋**。如果不发生阻塞就能获取到锁，则避免了后续的排队操作。
2. **入队挂起**：如果**自旋失败**，该线程会被封装成一个 `ObjectWaiter` 对象，插入到 **Contention List队列的队首**。
3. **阻塞**：调用系统内核的 `park` 方法挂起当前线程，等待操作系统的调度（此时线程不再消耗 CPU）。

**锁释放与唤醒**
* **假定继承人**：当线程释放锁时，会从 Contention List 或 EntryList 中挑选一个线程唤醒，被选中的线程叫做 **假定继承人**。
  * **OnDeck**：一次只会有一个线程被唤醒作为假定继承人，避免所有线程同时争夺锁
* **非公平性**：假定继承人被唤醒后会尝试获得锁，但 `synchronized` 是**非公平**的。这意味着假定继承人可能需要和新来的线程（刚刚执行自旋的线程）竞争，新来的线程不需要经历阻塞唤醒的开销，往往更容易抢到锁。
  * 如果直接将锁交给继承人，则Cpu需要等待继承人被唤醒
  * 非公平性保证了吞吐量，如果此时有线程在自旋等待则直接插队不浪费Cpu，等继承人被唤醒时，可能已经执行完了。

**Wait/Notify** 机制与锁膨胀
* **进入 WaitSet**：如果线程获得锁后调用 `Object.wait` 方法，则会将线程加入到 **WaitSet** 中，并释放锁。
* **被唤醒**：当被 `Object.notify` 唤醒后，会将线程从 WaitSet 移动到 Contention List 或 EntryList 中去，重新参与竞争。
* **强制膨胀**：需要注意的是，当调用一个锁对象的 `wait` 或 `notify` 方法时，如果当前锁的状态是**偏向锁**或**轻量级锁**，则会先**强制膨胀成重量级锁**，因为只有重量级锁才有 WaitSet

**双队列结构**
- 所有请求锁的线程将被首先放置到Contention List竞争队列，Entry List只会被Owner 线程操作
- 当且仅当Entry List空时，Owner 线程才会读Contention List竞争队列批量弹出一部分线程到Entry List，其他时刻Contention List竞争队列是仅写的
- Contention List竞争队列是高竞争区域，Entry List实现了读写分离，减少对队列锁的竞争，把“多线程并发入队”和“单线程调度唤醒”隔离开，避免 Owner 释放锁时还需要和入队线程抢占队列的控制权。

### 线程性能问题

#### 上下文切换
当线程数多于Cpu核心数时，核心必须轮流执行线程，线程阻塞也会导致上下文切换，上下文切换的开销：
- 切换到内核态保存当前线程的寄存器、程序计数器，并加载下一个线程的状态，随后切换用户态
- 上下文切换往往意味着L1L2缓存、TLB（用于加速虚拟地址到物理地址转换的缓存机制）失效，新线程需要重新加载数据到缓存

减少上下文切换
- 使用最少线程
  - 线程创建和销毁需要切换到内核态，通过系统调用完成，另外线程默认占用1MB的栈空间，通过线程池或虚拟线程都能解决该问题
  - 线程池数量设置$$N_{threads} = N_{CPU} \times U_{CPU} \times (1 + \frac{W}{C})$$
    - $U_{CPU}$：期望的 CPU 利用率
    - $W/C$：等待时间 (Wait) 与计算时间 (Compute) 的比率。
    - 对cpu密集型应用，$W/C$是0，线程数等于核心数
    - 对IO密集型应用，$W/C$越高，所需的线程数越多，往往可以开到核心数的数十倍
  - 使用虚拟线程：将百万级的虚拟线程映射到少量Java线程上，由JVM而不是操作系统调度虚拟线程，因此上下文切换开销非常小
- 减少线程阻塞
  - 减少锁粒度：例如 ConcurrentHashMap 锁分段，不同的线程处理不同段的数据，在多线程竞争的条件下，减少线程持有锁的时间，从而减少线程因阻塞而上下文切换的次数
  - 读写分离：使用`ReadWriteLock` 或 `StampedLock`
  - CAS算法：利用 Atomic包 + CAS 算法来更新数据，采用乐观锁减少**不必要的锁竞争**带来的上下文切换



---

#### 减少串行代码

程序中无法并行的串行代码占比决定了程序执行的速度上限，无论有多少核心都无法超越该上限，而临界区就是串行代码
- 通过拆分锁粒度减少串行代码占比，例如`ConcurrentHashMap`将大锁拆分为小锁，不同的线程处理不同段的数据，这样对每个线程而言减少了串行占比
- 通过减少临界区，将同步代码转化为异步或并行执行
- 通过避免共享数据来减少串行代码


---


#### 减少锁竞争

锁竞争开销：`synchronized` 虽然在 1.6 后有优化，但在高竞争下依然会产生巨大的开销。锁的获取和释放过程本身就包含内存屏障指令，会限制 CPU 的指令重排序和预取策略。
- **读写分离**：使用 `ReadWriteLock` 或 `StampedLock`。
- **乐观锁**：使用 CAS实现无锁算法

---

#### 伪共享



伪共享：Cpu缓存以缓存行为单位，在64位机器上是64字节，如果不相关的变量在同一缓存行，不同核心修改不相关的变量时，触发MESI 协议导致对方核心的缓存失效，产生剧烈的总线流量，性能甚至不如单线程，即缓存行的所有权在多个 CPU 核心之间频繁移交

内存填充：通过在变量前后填充无意义字节实现缓存对齐，确保变量独占一个缓存行，并且前后缓存行全是无意义字节，防止预读导致内存填充失效
- 内存填充缺点：占用更多内存导致缓存命中率下降

使用内存填充的条件：在**多个线程**中执行**高频写**的**至少两个变量在内存中相邻**
- 只读变量或读多写少的变量无需考虑
- 单线程或低并发无需考虑（当线程数达到一定数量，为共享发生将导致总线风暴，让性能断崖式下降）
- 同一个类中的两个字段，物理上是靠近的；数组元素也是紧紧相邻的


Java 8的内部注解`@Contended`可以通过启动参数`-XX:-RestrictContended`开启，开启后业务代码中也能使用，`@Contended`有三种粒度的用法：

- 字段级：确保该字段不与类中的其他变量挤在一个缓存行
- 分组级：将相关的变量放在一起，但与不相关的变量隔离开。相关的变量往往一起修改，进一步利用缓存行的空间局部性

```JAVA
Java
public class SharedData {
    @Contended("group1")
    public long a;
    @Contended("group1")
    public long b; // a 和 b 会挤在一个缓存行，但由于都在 group1，它们会远离 c

    @Contended("group2")
    public long c;
}
```
- 类级：在整个类上加注解，JVM 会在对象的前后都加上填充（Padding），防止不同对象的字段发生伪共享。






### 线程活跃性问题


线程活跃性问题：程序**能否在有限的时间内完成任务**

#### 死锁

死锁：两个或多个线程互相持有对方所需的资源，同时又在等待对方释放资源，导致所有相关线程进入永久阻塞状态。死锁的必要条件：
- **互斥**：资源一次只能被一个线程占用。
- **请求与保持**：线程持有一个资源，同时申请新资源。
- **不可剥夺**：资源在未用完前不能被强行抢占。
- **循环等待**：存在一个线程等待链的闭环。

**排查工具**：
* `jstack`：生成线程转储，自动识别 **"Found one Java-level deadlock"**。
* `JConsole` / `VisualVM`：图形化监控工具。


**预防与解决**：
* **顺序加锁**：确保所有线程按同一顺序获取多个锁。
* **尝试锁 (tryLock)**：使用 `ReentrantLock` 设置获取锁的超时时间。

---

#### 活锁

活锁：线程并没有阻塞，而是在不断地改变状态（通常是重试或让步），但**始终无法向前推进**，Cpu占用率高。

**分布式事务/锁的“礼貌”退避**：常见于需要同时获取多个资源（如锁）的逻辑中。
- 微服务 A 试图同时获取“库存锁”和“订单锁”来处理订单。为了避免死锁，开发者使用了 tryLock()。
  - 线程 1：获取了“库存锁”，尝试获取“订单锁”失败，于是“礼貌地”释放了“库存锁”并准备重试。
  - 线程 2：获取了“订单锁”，尝试获取“库存锁”失败，于是“礼貌地”释放了“订单锁”并准备重试。
  - 结果：如果两个线程的重试节奏完全同步（例如都是失败后立即重试），它们会不断地重复“拿 A 弃 A”、“拿 B 弃 B”的过程。
- 解决建议：在 tryLock 失败后的重试逻辑中引入随机睡眠时间（Jitter），打破步调同步。

**消息队列的“毒丸”重试循环**：在处理 MQ（如 RabbitMQ 或 Kafka）消息时，如果错误处理逻辑写得不够严谨，极易产生局部活锁。
- 消费者从队列拿到一条消息，但在处理时发生了可恢复的异常（例如数据库连接暂时溢出）。
  - 消费者捕获异常，为了保证消息不丢失，调用 nack 并要求消息 requeue（重回到队首）。
  - 由于队列并发量极高或只有这一个消费者，该消息被重新投递给同一个消费者。
  - 消费者再次处理，再次异常，再次重回队首。
  - 结果：消费者线程一直忙于处理这条必失败的消息。虽然它看起来在疯狂工作（CPU 飙升），但实际上该消息序列后面的成千上万条消息都被阻塞了。
- 解决建议：
  - 限制最大重试次数。
  - 使用**死信队列（DLQ）**存放多次失败的消息。
  - 指数退避（Exponential Backoff）重试策略。



---

#### 饥饿与公平锁

一个或多个线程因为始终无法获得所需的资源（如 CPU 时间片、锁）而长时间无法执行。
* **高优先级线程霸占**：低优先级线程永远抢不到 CPU。
* **非公平锁**：`synchronized`锁即使非公平锁，导致排在队列后面的线程被新来的线程不断“插队”。可以考虑使用 `ReentrantLock(true)`等公平锁

---

#### 优先级反转

一个高优先级的线程被一个低优先级的线程阻塞，而由于中等优先级的线程抢占了低优先级的 CPU 资源，间接导致高优先级线程被无限期延迟。
* **优先级继承协议**：当高优先级线程等待锁时，暂时将持有锁的低优先级线程提升到与自己相同的优先级。

---

> 传统的 `synchronized` 无法妥善处理线程活跃性问题，被阻塞线程无法响应中断、不支持超时容易导致死锁、无法变为公平锁解决饥饿问题




## JUC框架核心

### CAS

### Unsafe魔法类



### atomic包

### LockSupport

### AQS：AbstractQueuedSynchronizer





## JUC显式锁

### ReentrantLock

### ReentrantReadWriteLock

### StampedLock




## JUC并发流程控制工具

### CountDownLatch

### CyclicBarrier

### Semaphore

### Exchanger


## 并发集合

## JCTools

## 线程池

## Fork/Join

## 多线程上下文传递

## 虚拟线程
# EOF











