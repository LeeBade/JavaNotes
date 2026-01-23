
## 线程安全、锁、通信

### Java内存模型

> **区分[Java运行时内存区域](#jvm运行时内存区域)和Java内存模型两个不同又有联系的概念**
- Java运行时内存区域是**JVM运行时的具体内存划分**
- **Java 内存模型Java Memory Model，JMM**，是一个**抽象模型**，定义了
  1. **线程与主存、工作内存进行交互的规则**
  2. **多线程环境下共享变量的访问规则**


**Cpu多级缓存**与**MESI 缓存一致性协议**
- 引入多级缓存缩小主内存读写速度与Cpu运算速度的差距，同时导致**缓存一致性**问题，因此引入**MESI 缓存一致性协议**，而仅寄存器不在**MESI 缓存一致性协议**生效范围内，导致寄存器是Cpu多级缓存中唯一一个确定属于工作内存的硬件
- 在Cpu多级缓存中，仅寄存器不在**MESI 缓存一致性协议**中，



| 层级 (Hierarchy) | 真实位置 | 模拟耗时 (假设 CPU 算一下是 1秒) | 容量 | 角色隐喻 |
| --- | --- | --- | --- | --- |
| **寄存器 (Registers)** | **CPU 核心内部** | **1 秒** (即时) | 极小 (几百字节) | **手中的加工件**。CPU 只能直接计算这里的数据。 |
| **L1 Cache** | **CPU 核心内部** | **3 ~ 4 秒** | 小 (32KB - 64KB) | **工位上的工具箱**。伸手就能拿。 |
| **L2 Cache** | **CPU 核心内部** | **10 ~ 12 秒** | 中 (256KB - 512KB) | **身后的背包**。转身就能拿。 |
| **L3 Cache** | **多核共享** | **40 ~ 50 秒** | 大 (MB ~ 几十 MB) | **房间里的共享货架**。所有核心都能看到。 |
| **主内存 (DRAM)** | **插在主板上** | **200 ~ 300 秒 (几分钟)** | 巨大 (GB ~ TB) | **楼下的仓库**。去一趟很慢。 |


**线程与主存、工作内存进行交互的规则**
。然而，在多核处理器环境下，各核心拥有独立的缓存，出现，而JMM屏蔽底层定义一种抽象的内存架构来管理线程与内存的交互
  1. **主内存**：所有线程共享的内存区域，存储所有的对象实例、静态变量与数组元素。
  2. **工作内存**：每个线程独占的内存区域（逻辑概念，映射到物理硬件的寄存器、Cpu 缓存及写缓冲区）。
  3. **交互协议**：线程对共享变量的所有操作必须在工作内存中进行。线程首先将变量从主内存拷贝到工作内存，修改后视情况刷新回主内存。不同线程之间无法直接访问彼此的工作内存，线程间的通信必须通过主内存作为中介完成。

**多线程环境下共享变量的访问规则**
- JMM通过解决线程间的通信和同步问题，满足并发编程三大特性：**原子性、有序性、可见性**
- JMM 通过内存屏障实现可见性和禁止重排序，为了用户方便理解该系列规则，设计者提出**Happens-Before概念**


**多线程环境程序是否正确取决于是否满足原子性、有序性、可见性**，即线程安全问题
- **线程安全问题**：多个线程同时访问并修改同一数据且缺乏适当的同步措施时造成线程安全问题
- 线程安全问题需要通过满足原子性、有序性、可见性解决
- **原子性Atomicity**
  - 原子性指一个操作是不可中断的整体，要么全部执行成功，要么完全不执行
  - 现代64位JVM中，**基本数据类型的单次读写是原子性的**
- **可见性Visibility**
  - 可见性指当一个线程修改了共享变量的值，其他线程能够立即感知到这个修改
  - **`volatile`**：强制将修改后的值立即刷新回主内存，并利用缓存一致性协议（如 MESI）使其他线程的缓存行失效。
  - **`synchronized`**：在释放锁之前，必须将工作内存中的变量同步回主内存。
  - **`final`**：被 `final` 修饰的字段在构造器初始化完成后，对其他线程即是可见的
- **有序性Ordering**
  * 有序性指程序执行的顺序按照代码的先后顺序执行

除了线程安全问题，多线程环境程序还需要考虑**活跃性问题**和**性能问题**
- **活跃性问题**
  - **死锁**：两个或多个线程互相持有对方所需的资源，同时等待对方释放资源，导致所有相关线程永久阻塞
  - **活锁**：两个或多个线程没有阻塞并且都在修改各自的状态，而其他线程又依赖这个状态，就导致任何一个线程都无法继续执行，只能重复着自身的动作
  - **饥饿**：
    - 高优先级的线程一直在运行消耗 CPU，所有的低优先级线程一直处于等待
    - 一些线程被永久堵塞在一个等待进入同步块的状态，而其他线程总是能在它之前持续地对该同步块进行访问；
- **性能问题**
  - 多线程有创建线程和线程上下文切换的开销，这种昂贵的开销是需要操作系统完成的，导致多线程并发不一定比单线程串行执行快

---







### JMM与重排序


指令重排序通过减少Cpu停顿时间，不让指令流水线中断，而提高Cpu的效率
- 下面两条指令都需要先加载再计算，为了不在加载时停顿，通过指令重排序连续加载b、c、e、f，然后再连续执行两次计算
- **指令重排序对于提高 Cpu 性能十分必要，该技术保证单线程环境下程序语义不变，但在多线程环境下带来了乱序的问题**
```JAVA
a=b+c
e=e-f
```

**指令重排序分为三种**
- **编译器优化重排**，编译器在不改变单线程程序语义的前提下，重新安排语句的执行顺序。
- **指令并行重排**，现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性(即后一个执行的语句无需依赖前面执行的语句的结果)，处理器可以改变语句对应的机器指令的执行顺序。
- **内存系统重排**，由于处理器使用缓存和读写缓存冲区，这使得加载(load)和存储(store)操作看上去可能是在乱序执行，因为三级缓存的存在，导致内存与缓存的数据同步存在时间差。

**JMM 与顺序一致性模型**
- 当程序未正确同步的时候，就可能存在**数据竞争：在一个线程中写一个变量，在另一个线程读同一个变量，并且写和读没有通过同步来排序**。
- **数据竞争发生时，程序的运行结果是不确定的**，即**同一个变量的读写操作发生的先后顺序是不确定的**
- **JMM承诺**：**正确同步的程序，其执行结果具有顺序一致性**，即该程序在多线程环境下和单线程环境下（顺序一致性模型）的执行结果相同
- **JMM不保证**未正确同步的程序的顺序一致性，因为JMM不仅需要减少用户编程的工作量，还需要尽量兼顾编译期和处理器的大量优化操作
- 但是JMM为未同步程序提供**最小化安全性**：对于未同步的线程，线程读取到的值，要么是之前某个线程写入的值，要么是默认值
  - 实现机制：**实例化对象操作是同步的**，即：内存空间清零，和在该内存空间分配对象两个操作是同步的。
- JMM与顺序性一致模型的差异：
  - JMM 不保证单线程内的操作会按程序的顺序执行
  - JMM 不保证所有线程能看到一致的操作执行顺序
  - JMM 不保证对 64 位的 `long` 型和 `double` 型变量的写操作具有原子性；在32位JVM中，会拆分为两次独立的高位、低位复合操作

**Happens-Before**
- 开发者希望强约束内存模型；编译器和处理器希望弱约束内存模型，以尽可能多的做优化来提高性能。
- JMM向编译器和处理器要求：只要满足正确同步的程序的顺序一致性，编译器和处理器怎么优化都行
- JMM向开发者提供简单易懂的**Happens-Before** 规则，提供了足够强的内存可见性保证
- **Happens-Before**规则**定制了两个同一线程内或不同线程间的操作之间的执行顺序**
  - **操作1 Happens-Before 操作2，则操作1的执行结果对操作2可见（即使它们不在同一个线程内），且操作1的执行顺序排在操作2之前（即使它们不在同一个线程内）。**
  - 操作1 Happens-Before 操作2，不代表JVM必须要按照 Happens-Before 关系指定的顺序来执行，如果重排序之后的执行结果，与按 Happens-Before 关系来执行的结果一致，那么 JMM 也允许这样的重排序。**否则，禁止重排序**


1. **程序次序规则 (Program Order Rule)**：在一个线程内，按照代码顺序，书写在前面的操作 Happens-Before 书写在后面的操作。
2. **监视器锁规则 (Monitor Lock Rule)**：一个 `unlock` 操作 Happens-Before 后续对同一个锁的 `lock` 操作。这是 `synchronized` 可见性的理论基础。
3. **volatile 变量规则**：对一个 `volatile` 变量的写操作 Happens-Before 后续对这个变量的读操作。
4. **传递性 (Transitivity)**：如果 A Happens-Before B，且 B Happens-Before C，则 A Happens-Before C。
5. **线程启动/终止规则**：`Thread.start()` Happens-Before 该线程内的任何操作；线程内的所有操作 Happens-Before 其他线程检测到该线程结束（`Thread.join()` 返回）。




---




---

### volatile关键字

`volatile`在 JMM 层面具有两个核心语义：**保证内存可见性**和**禁止指令重排序**，这是**通过在`volatile`读写操作前后插入屏障**实现的

**对Cpu而言，没有方法，只有指令流**
- **如果没有屏障，Cpu就可以乱序执行无依赖的指令**
- **如果修改不立即同步主内存，对其他线程不可见**
  - 无论是不是volatile变量，**缓存一致性协议**都一直在工作，但是没有立即写回，其他Cpu就收不到普通变量缓存过期的消息，仍使用旧值
- 考虑线程1和线程2对同一个未同步的`ReorderExample`对象调用`writer()`和`reader()`方法，由于指令重排序或内存可见性，可能打印0或1，在顺序性一致模型中只能打印1
  - 元凶1：内存可见性；线程1执行writer后，flag先刷回主内存，在线程2执行完reader后线程1再把a=1刷回主内存
  - 元凶2：指令重排序，因为1、2没有数据依赖，所以可能会出现该事件顺序

| 时间轴 | 线程 A (Writer) | 线程 B (Reader) | 备注 |
| --- | --- | --- | --- |
| T1 | **`flag = true`** (重排序先执行) |  | 此时 `a` 还是 0 |
| T2 |  | `if (flag)` | 读到 true，进入分支 |
| T3 |  | **`int i = a * a`** | **读到 a 为 0**，计算出 i=0 |
| T4 |  | `System.out.println(i)` | **打印 0** (Bug 复现) |
| T5 | `a = 1` (重排序后执行) |  | 一切都晚了 |

```JAVA
class ReorderExample {
  int a = 0;
  boolean flag = false;
  public void writer() {
      a = 1;                   //1
      flag = true;             //2
  }
  public void reader() {
      if (flag) {                //3
          int i = a * a;         //4
          System.out.println(i);
      }
  }
}
```


---

#### volatile保证可见性、有序性
> 内存可见性问题需要从两个方向配合理解：多级缓存架构+Cpu指令流

JMM可**向Cpu指令流插入的屏障有四种**

| 操作类型 | 屏障位置 | 屏障类型 | 作用 |
| --- | --- | --- | --- |
| **Volatile 写** | **前面** | `StoreStore` | **该屏障之前的指令流中所有的读写操作，禁止重排序到本屏障后**<BR>并且**确保写操作在该屏障之前彻底完成（完成指刷回主内存）** |
| **Volatile 写** | **后面** | `StoreLoad` | **该屏障之后的指令流中所有的读写操作，禁止重排序到本屏障前** |
|  |  |  |  |
| **Volatile 读** | **后面** | `LoadLoad` | 该屏障之后的指令流中所有的**普通读**操作，禁止重排序到本屏障前 |
| **Volatile 读** | **后面** | `LoadStore` | 该屏障之后的指令流中所有的**普通写**操作，禁止重排序到本屏障前 |

- **`volatile`写操作**执行完成后，上一个屏障之后到该屏障之前的所有读写指令都已完成，并且所有写操作已刷回主内存，通过**MESI 缓存一致性协议**强迫其他Cpu在下次读取时，去主内存重新拉取最新值
  - **写缓冲**是**导致一切写操作内存可见性问题的罪魁祸首**：Cpu将寄存器中的值放入写缓冲，然后立刻执行别的指令
- **`volatile`读操作**：主要职责是禁止指令重排序，范围是该屏障之后到下一个屏障之前的所有普通读写操作；并且强迫该volatile变量每次都从缓存中取数据，而不是读取寄存器中的值
  - volatile读操作与普通读操作在内存一致性上也存在差别，Cpu在执行普通读时，可能只会从寄存器读取，而不会注意到缓存失效的消息，因为**MESI 缓存一致性协议**的工作范围是L1缓存到主存，不包括寄存器
  - **寄存器看不到MESI 缓存一致性协议是导致一切读操作内存可见性问题的罪魁祸首**



- 考虑下面的代码，**演示了volatile读操作与普通读操作在内存一致性上的差别**
  - 空循环执行一会儿后，JIT发现啊呀，执行几千次了，得优化为本地代码，然后将flag放在寄存器里
  - 执行主线程的Cpu核心修改了flag刷回主内存并且发出缓存失效的消息
  - 但是执行从线程的Cpu核心只关注寄存器里的值，压根不看缓存，听不到缓存失效的消息只管闷头跑
```JAVA
public class Main {
    public static boolean flag = true;
    //空循环并且让主线程睡眠一会儿，让JIT编译器检测热代码将flag放在寄存器里
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            System.out.println("子线程：开始执行，等待 flag 变为 false...");
            while (flag) {}
            System.out.println("子线程：检测到 flag 变为 false，退出循环！");
        }).start();
        Thread.sleep(1000);
        flag = false;
        System.out.println("主线程：已将 flag 修改为 false");
    }
}
```
- 将flag修改为volatile，则每次循环都触发volatile读，每次都会从缓存拉取值，检查缓存是否失效，而不是只读寄存器中的值

#### volatile不保证原子性

> `volatile` **仅保证单次读写操作是原子的，不保证复合操作的原子性**
- 考虑count++所插入的内存屏障：
  - 线程1执行步骤2前发生了线程切换
  - 线程2完整执行了所有步骤并写回内存切换到线程1
  - **由于寄存器不在缓存一致性协议内，即使寄存器的值与内存不一致，也会继续执行**
  - 线程1继续执行步骤2、3并写回内存，线程2执行的数据丢失

```text

1. Load i   <-- volatile 读从内存读 0 到寄存器
   -------------[LoadLoad 屏障][LoadStore 屏障]<-- 保证读完 i 之后才能做后面的事
     

2. Add (寄存器内 0 + 1 = 1)      <-- 纯 CPU 运算 (毫无保护！)

3. -------------[StoreStore 屏障] <-- 保证前面的写完了 (其实这里前面没写操作)
   Store i (把 1 写回内存)       <-- volatile 写
   -------------[StoreLoad 屏障]  <-- 保证写完 i 之后，别的线程能看见

```

- volatile只能**定序**，**确保单次读写操作的原子性**，**并且确保内存可见性**，但是**volatile不能阻止操作系统在屏障之间的时间差内切换线程，自然不能阻止其他线程修改共享的变量**，也就**无法保证复合操作的原子性了**
- **原子性要求：读取、修改、写入这三个动作，必须像一个整体一样执行，中间不允许插入其他线程的操作。**
- **Volatile 的能力**：
  - 它保证了 **单次读 是原子的**（**读的一瞬间拿到的一定是对的**）。
  - 它保证了 **单次写 是原子的**（**写完的一瞬间大家都能看见**）。
  - 但它**无法把“读”和“写”这两个分离的动作“粘”在一起**。

- 比喻：
  - **Synchronized（锁）**：像是把厕所门锁上了。进去后，我想读报纸、抽烟、发呆，由于门锁着，外面的人进不来。哪怕我在里面睡着了（线程切换），外面的人也得等着。
  - **Volatile（屏障）**：像是厕所装了个**强力排风扇**和**透明门**。
    - **透明门（可见性）**：我在里面拉没拉完，外面人看得清清楚楚。
    - **无锁**：我刚进去（Read），还没脱裤子，就被外面闯进来的人（Thread B）挤开了。即使排风扇（屏障）保证了空气流通，也不能阻止此时此刻有两个人在同一个坑位上打架。


#### 双重检查锁定

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


考虑去掉volatile关键字
- `instance = new Singleton();`在 JVM 指令层面并不是原子操作，而是包含了三个步骤：
  1. Allocate：给对象分配内存空间。
  2. Initialize：调用构造函数，初始化对象内部成员。
  3. Assign：将引用 instance 指向刚才分配的内存地址。
- 由于调用构造函数是耗时的，可能会**指令重排序步骤2、3，导致其他线程拿到了没有构造好的对象**

加上volatile关键字
- 由于内存屏障，步骤1、2一定在步骤3之前完成
- 根据Happens-Before的规则，volatile写操作Happens-Before 后续对volatile的读操作

**锁保证原子性**的副作用是：**有条件的可见性**和**锁范围外的有序性**
- **锁释放前，所有修改一定写回主内存**；并且**保证写回的结果是符合顺序一致性模型的**
- **如果其他线程需要获取相同的锁才能进行读写，那么就保证了可见性**
- **如果其他线程不加锁就进行读写，就不保证可见性**；
  - 例如双重检查锁定，如果去掉volatile关键字，其他线程没有就可能在没有任何同步措施的情况下获取到半成品

**从双重检查锁定审视锁对象和锁范围的设计**
- **资源初始化是重量级操作，必须采用懒加载策略，且必须仅执行一次**
- **锁范围**：**限定在初始化资源的代码块中**，**获取资源的代码块必须在锁外**
- **锁对象**：避免多个线程同时执行初始化资源动作，因此加锁对象是`Singleton.class`；而不是对整个`getInstance()`方法加锁，导致读操作无法并发
  - `Singleton.class`是静态全局唯一的，直接从源头掐死其他线程同时初始化资源的机会
  - 等到资源初始化完毕，其他线程获取到锁，执行第二次检查时发现资源已存在，自然无法再初始化
- **获取资源的代码必须支持多线程并发**，即不能加锁，**但是不加锁又可能会导致锁的可见性失效**，所以需要volatile
- **获取资源的代码块通过volatile关键字提供Happens-Before同步保证**：**只要资源不为 null，就一定是初始化完成的完整对象**

---


### synchronized

`synchronized`锁的原子性、可见性、有序性保证参考[双重检查锁定](#双重检查锁定)

`synchronized`的三种应用方式：

1. **`synchronized`实例同步方法**：**默认以this对象作为对象锁**
   - **`this`对象锁**被一个线程持有时，其他线程不能调用该对象的**任何**`synchronized` 实例方法
2. **`synchronized`静态同步方法**：**默认以`this.Class`类作为类锁**
   - **`this.Class`类锁**被一个线程持有时，其他线程不能调用该`this.Class`类的**任何**`synchronized` `static`方法
3. **`synchronized`同步代码块**，需要指定加锁对象（`this`、`this.Class`、其他对象），对给定对象加锁，进入同步代码库前需要获取指定对象锁

锁范围：**类锁、对象锁A、对象锁B互相独立**，且它们都不阻塞非同步代码的执行
- 类锁只管静态同步方法，对象锁A只管A的实例同步方法、对象锁只管B的实例同步方法

`synchronized` 是**可重入锁**，已持有锁的对象再次申请该锁时立即成功，可重入锁解决单线程环境下的死锁问题


---

synchronized 锁升级是一个比较复杂的过程，且不可降级，锁升级内容详细讲解了为什么需要在锁竞争激烈时，对锁对象和锁范围设计保持敬畏


JDK1.6之前，`synchronized`锁是**使用操作系统互斥量mutex实现的重量级锁**，线程获取锁失败时的阻塞需要在**用户态和内核态之间的上下文切换**；Jdk1.6引入**无锁、偏向锁、轻量级锁**状态，因为偏向锁的机制太过复杂，Jdk15默认禁用偏向锁状态
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

---

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

### CAS、MESI、Futex、LockSupport、缓存行填充

**内存屏障、Futex、MESI缓存一致性协议、LockSupport、伪共享与填充作为并发编程的基石**构建了整个并发编程体系，解决了并发编程的原子性、可见性、有序性、线程调度与内存效率问题

**Futex**：Fast Userspace Mutex
- 早期的 Linux使用Mutex构建悲观锁工具，现代Linux底层的Futex改进Mutex，作为`synchronized`、`ReentrantLock` 的底层挂起机制
- Mutex 每次加锁解锁都要陷入内核态，开销极大。但实际上，大部分锁在同一时刻是没有竞争的。
- Futex 维护一个在**用户态**共享的整数。
  - 如果没有竞争，直接在用户态用 **CAS** 原子操作修改这个整数，**完全不进入内核**
  - 只有当 CAS 失败时，才调用系统调用进入内核，挂起线程。

**MESI缓存一致性协议**
- CAS 指令的原子性保证和volatile的可见性依赖于多核Cpu工作在MESI缓存一致性协议下
- MESI缓存一致性协议原理：当一个Cpu核心修改了缓存行中的数据时，必须通过总线通知其他核心对应的缓存行失效
* **M (Modified)**: 修改过，未同步到内存。
* **E (Exclusive)**: 独占，只有我有，且和内存一致。
* **S (Shared)**: 共享，大家都有，只能读。
* **I (Invalid)**: 失效，我的数据过期了，由于别人修改了。


**LockSupport.park/LockSupport..unpark**
- LockSupport是Java 线程调度的最小原语，不同于`Object.wait()`必须配合 `synchronized` 使用，`LockSupport.park()` 可以在任何地方阻塞当前线程
- LockSupport底层使用Futex实现，所有 JUC 工具类中的`ReentrantLock`、`CountDownLatch`、`FutureTask`在阻塞线程时最终调用的都是 `LockSupport.park()`



**伪共享与填充**
- Cpu每次以缓存行为单位加载数据，单个缓存行是64字节；
- **伪共享**：如果变量A和变量B在同一个缓存行中，线程1和线程2分别修改变量A和变量B，会导致互相的缓存行失效，两个不相干的变量最终导致两个线程互相拖慢速度
  - **伪共享**定义：不相关的变量共享了同一个缓存行
- **缓存行填充Padding**：通过填充大量long类型空白行，将相邻的变量挤到不同的缓存行里
- Java 的 `LongAdder`、Disruptor 框架的底层都大量使用了**缓存行填充Padding**来提升并发吞吐量

**CAS：Compare and Swap**


### 无锁

#### CAS

CAS：Compare-and-Swap是无锁的原子操作。乐观锁总是假设对共享资源的访问没有冲突，无需等待或加锁，通过CAS保证多线程冲突时的并发安全性。由于乐观锁是无锁，所以天生免疫死锁。
- 乐观锁：读多写少，避免频繁加锁影响性能；
- 悲观锁：写多读少，避免频繁失败和重试影响性能。

CAS操作
- 三个核心参数
  - 内存位置Value，V：变量在内存中实际的地址值
  - 预期原值Expected，E：线程认为该变量当前应该有的值
  - 新值New，N：线程希望将变量更新为的值
- CAS是系统原语，整个过程由Cpu保证原子性：
  - 判断 V 是否等于 E
  - 如果等于，将 V 的值设置为 N
  - 如果不等，说明已经有其它线程更新了 V，向线程返回V值，什么都不做
- 使用CAS实现`Unsafe`、`Atomic`、`LongAdder`的方式：当多个线程同时使用 CAS 操作一个变量时，只有一个线程成功更新，其余均会失败，失败的线程获取到操作系统返回V值不等于N值，就知道本次CAS操作失败，可以再次尝试CAS操作或放弃操作

操作系统通过指令：`lock cmpxchg`，保证CAS的原子性
- `cmpxchg`：完成比较并交换
- `lock`：`cmpxchg`不是原子的，**lock指令确保目标缓存行被锁定**，**其他Cpu核心的该缓存行被置为失效，且无法执行写入**，直到`cmpxchg`指令执行完，然后从主存拉去该缓存行的新值
- **内存屏障**：**禁止该指令前后的读写指令重排序**，并且确保**写缓冲区中的数据立即刷新到主存**中，保证了可见性和有序性

**CAS的三大问题**
1. **ABA问题**：线程1准备将A改为B。此时线程2插队，将A改为B，又改回A，线程1继续执行检查还是A改为B
   - 对于没有内部结构或引用关系的变量，ABA问题不会导致问题；对于栈或者引用对象可能导致引用错乱
   - 栈：Top-A-B-C
     - 线程1读取栈顶A，并读取下一个节点B作为栈顶，切换线程2
     - 线程2弹出A、B，并压入A，切换线程1，此时栈为Top-A-C
     - 线程1检查栈顶仍为A，将B作为栈顶，这个过程是原子性的，此时栈为Top-B，C及之后的链表内存泄露
   - 根本原因在于CAS执行浅比较，解决方案是使用**AtomicStampedReference**，CAS比较的每个元素是`pair`对，同时包含`reference`和`stamp`两个参数
   - **AtomicStampedReference**维护四个核心参数：
     - `expectedReference`：预期引用
     - `newReference`：新引用，如果预期引用正确，新引用将被设置到该位置
     - `expectedStamp`：预期版本号
     - `newStamp`：新标版本号，如果预期版本号正确，新版本号将被设置到该位置
2. **长时间自旋**
   - CAS 多与自旋结合。如果自旋 CAS 长时间不成功，会占用大量的 CPU 资源，解决思路是让 JVM 支持处理器提供的**pause 指令**
   - pause 指令能让自旋失败时 cpu 睡眠一小段时间再继续自旋，从而使得读操作的频率降低很多，为解决内存顺序冲突而导致的 CPU 流水线重排的代价也会小很多。
3. **多个共享变量的原子操作**
   - CAS无法保证多个共享变量操作的原子性
   - 解决方案一：使用`AtomicReference`类保证对象之间的原子性，把多个变量放到一个对象里面进行 CAS 操作
   - 解决方案二：使用锁

#### Unsafe

`Unsafe` 提供了一组底层、原语级别的操作，允许 Java 代码直接访问内存、操作线程和对象

**获取 Unsafe 实例**
- `Unsafe` 类是单例的，其构造方法是私有的，并且被 `final` 修饰，禁止继承。
- 虽然它提供了一个静态方法 `getUnsafe()`，但普通用户代码直接调用会抛出 `SecurityException`。
  - 这是因为 `Unsafe` 的功能过于底层（如直接内存操作、绕过 JVM 安全检查等），只有由启动类加载器（Bootstrap ClassLoader）加载的系统类（如 `rt.jar` 中的类）才有权限调用。

```java
@CallerSensitive
public static Unsafe getUnsafe() {
    Class var0 = Reflection.getCallerClass();
    // 检查调用者是否由 Bootstrap ClassLoader 加载
    if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
        throw new SecurityException("Unsafe");
    } else {
        return theUnsafe;
    }
}

```
- 通过反射获取

```java
public static Unsafe getUnsafe() throws IllegalAccessException, NoSuchFieldException {
    Field unsafeField = Unsafe.class.getDeclaredField("theUnsafe");
    unsafeField.setAccessible(true);
    Unsafe unsafe = (Unsafe) unsafeField.get(null);
    return unsafe;
}

```

**Native 方法与 JNI**
- `Unsafe` 中的绝大多数方法都是 `native` 方法，通过 `JNI`进行调用。使用 Native 方法的原因主要包括：
1. 需要使用 Java 不具备的、依赖于操作系统的底层特性。
2. 为了获得极高的性能（如直接操作硬件指令）。
3. 复用现有的 C/C++ 库。

---

Unsafe核心功能可以归纳为8类：内存操作、内存屏障、对象操作、数组操作、CAS 操作、线程调度、Class 操作和系统信息。

**内存操作**

- **允许分配堆外内存**
* `allocateMemory(long bytes)`: 分配本地内存。
* `reallocateMemory(long address, long bytes)`: 重新分配内存。
* `setMemory(...)`: 初始化内存块。
* `copyMemory(...)`: 内存拷贝。
* `freeMemory(long address)`: 释放内存。通过 `Unsafe` 分配的内存不受 JVM 垃圾回收（GC）管理，必须手动调用 `freeMemory` 释放，否则会导致内存泄漏。

```java
private void memoryTest() {
    int size = 4;
    // 1. 分配 4 字节的内存
    long addr = unsafe.allocateMemory(size);
    
    try {
        // 2. 写入数据
        unsafe.setMemory(null, addr, size, (byte)1);
        
        // 3. 读取数据 (一次读取 4 个字节作为一个 int)
        // 结果为 16843009 (即 0x01010101)
        System.out.println(unsafe.getInt(addr)); 
        
    } finally {
        // 4. 必须手动释放内存，否则会内存泄漏
        unsafe.freeMemory(addr);
    }
}

```


**内存屏障**：

* `loadFence()`: 禁止读操作重排序。确保屏障前的读操作全部完成，并刷新缓存。
* `storeFence()`: 禁止写操作重排序。
* `fullFence()`: 禁止读写操作重排序。
 - 下列代码参考[volatile关键字](#volatile关键字)
 - 取消内存屏障，JIT优化flag到寄存器，支线程的修改不影响主内存
 - 增加内存屏障，禁止重排序，读屏障后的变量读取需要从主内存拉取，不能假设之前的缓存有效

```JAVA
public class Main {
    public static void main(String[] args) throws InterruptedException, NoSuchFieldException, IllegalAccessException {
        ChangeThread changeThread = new ChangeThread();
        Unsafe unsafe=getUnsafe();
        new Thread(changeThread).start();
        while (true){
            boolean flag =changeThread.isFlag();
            unsafe.loadFence();//加入读内存屏障
            if (flag) {
                System.out.println("detected flag changed");
                break;
            }
        }
        System.out.println("main thread end");
    }
    public static Unsafe getUnsafe() throws IllegalAccessException, NoSuchFieldException {
        Field unsafeField = Unsafe.class.getDeclaredField("theUnsafe");
        unsafeField.setAccessible(true);
        Unsafe unsafe =(Unsafe) unsafeField.get(null);
        return unsafe;
    }
}

@Getter
class ChangeThread implements Runnable{
    boolean flag=false;
    @Override
    public void run() {
        try {
            Thread.sleep(3000);
        }catch(InterruptedException e){
            e.printStackTrace();
        }
        System.out.println("subThread change flag to: "+flag);
        flag= true;
    }
}
```

**对象操作**：读写对象的字段，甚至绕过访问权限（如修改 `private` 字段）。它依赖“偏移量”（Offset）来定位字段在内存中的位置。

* `objectFieldOffset(Field f)`: 获取字段在对象内存布局中的相对偏移量。
* `putInt(Object o, long offset, int x)`: 设置值。
* `getInt(Object o, long offset)`: 获取值。
* `allocateInstance(Class<?> cls)`: **非常特殊的方法**，它可以绕过构造方法直接创建对象实例。


**CAS 操作**
- 并发编程的基础，`java.util.concurrent.atomic` 包下的原子类完全依赖 `Unsafe` 的 CAS 方法。
- CAS 机制包含三个参数：需要读写的内存位置（通过对象+偏移量确定）、预期的原值、拟写入的新值。只有当内存中的值与预期值相等时，才会将内存更新为新值。这是一个原子指令。

* `compareAndSwapInt(Object o, long offset, int expected, int x)`
* `compareAndSwapLong(...)`
* `compareAndSwapObject(...)`

**线程调度**

- `Unsafe` 提供了线程挂起和恢复的原语，这是 `LockSupport` 类和 `AQS`框架的基石。

* `park(boolean isAbsolute, long time)`: 阻塞当前线程。
* `unpark(Object thread)`: 唤醒指定线程。

**Class 操作**

`Unsafe` 提供了动态加载类的方法，可以从字节码数组中定义一个类。

* `defineClass(...)`: 定义匿名类。
* `ensureClassInitialized(...)`: 确保类被初始化。

这是 Lambda 表达式和动态代理技术底层的关键支撑。

**数组操作**

`Unsafe` 提供了计算数组元素内存定位的方法，使得原子地更新数组元素成为可能（如 `AtomicIntegerArray`）。

* `arrayBaseOffset(Class<?> arrayClass)`: 获取数组第一个元素的偏移地址。
* `arrayIndexScale(Class<?> arrayClass)`: 获取数组中元素的增量地址（即每个元素占用的字节数）。

#### Atomic


#### LongAdder




### JUC 显式锁

* *锁分类和JUC*

#### AQS 骨架

* *抽象队列同步器AQS* （CLH 队列设计模式）。
* *线程阻塞唤醒类LockSupport*

#### Locksupport
#### *重入锁ReentrantLock*

* *等待通知条件Condition*

#### *ReentrantReadWriteLock*、StampedLock




### JUC 并发工具集

并发流程控制+数据交换

* *通信工具类*

#### CountDownLatch

#### CyclicBarrier

#### Semaphore.

#### 通信工具

Semaphore、Exchanger、CountDownLatch、CyclicBarrier、Phaser

