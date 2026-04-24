太棒了！从“业务痛点”下沉到“源码实现”，这是从高级开发向架构师蜕变的必经之路。

很多开发者觉得 `@TransactionalEventListener` 就像魔法一样，能够精准地在事务提交后触发。但其实，Spring 底层并没有什么黑科技，它依靠的是一套极其严谨的**回调机制**和**线程上下文（ThreadLocal）共享**。

我们现在就扒开这层外衣，看看它是如何“感知”事务的。

---

### 2.2 源码级解密：它是如何感知事务的？

要理解这个机制，我们必须先认识两个核心组件：一个是 Spring 事务体系里的“间谍” `TransactionSynchronization`，另一个是幕后大Boss `TransactionSynchronizationManager`。

#### 🕵️ 探秘 `TransactionSynchronization` 接口

`TransactionSynchronization` 是 Spring 事务模块提供的一个极其重要的回调接口。你可以把它理解为**安插在数据库事务生命周期中的“监听探针”**。

打开 Spring 的源码，你会发现这个接口定义了与事务生命周期完美对应的方法：

```java
public interface TransactionSynchronization extends Flushable {
    // 挂起事务时触发
    default void suspend() {}
    // 恢复事务时触发
    default void resume() {}
    // 数据库 flush 之前触发
    default void flush() {}
    // 【关键】数据库真正执行 Commit 之前触发
    default void beforeCommit(boolean readOnly) {}
    // 事务完成前触发（不论成功失败）
    default void beforeCompletion() {}
    // 【关键】数据库成功执行 Commit 之后触发
    default void afterCommit() {}
    // 【关键】事务彻底结束后触发（带有状态码：STATUS_COMMITTED / STATUS_ROLLED_BACK）
    default void afterCompletion(int status) {}
}
```

**结论：** 所谓的“四大事务阶段”，本质上就是对这个接口中 `beforeCommit`、`afterCommit`、`afterCompletion` 等方法的封装和映射。

---

#### ⚙️ 协同机制：事件广播器与事务管理器的“暗度陈仓”

当我们在代码里调用 `applicationContext.publishEvent(event)` 时，事件到底是怎么和上面的接口搭上线的？它们是通过以下四个步骤完成协同的：

**第一步：拦截与伪装 (`TransactionalApplicationListenerMethodAdapter`)**
当 Spring 容器启动时，它会把你写了 `@TransactionalEventListener` 的方法包装成一个特殊的监听器适配器。
当事件发布时，这个适配器会拦截到事件。但它**不会立刻执行**你的业务逻辑，而是进行下一步。

**第二步：嗅探事务 (`TransactionSynchronizationManager`)**
适配器会去问 `TransactionSynchronizationManager`：“嗨，当前线程有没有正在运行的事务？”
`TransactionSynchronizationManager` 是一个基于 `ThreadLocal` 的管家，它手里握着当前线程所有的数据库连接和事务状态。

**第三步：注册回调（“挂号”机制）**
* **如果有事务**：适配器会把你的事件和业务逻辑，打包封装成一个 `TransactionSynchronization` 的实现类，然后调用 `TransactionSynchronizationManager.registerSynchronization(...)` 把自己**注册（挂号）**到当前线程的事务管家那里。此时，事件发布方法就直接返回了（这就是为什么它不阻塞主业务逻辑的执行，只是把任务存起来了）。
* **如果没有事务**：就去检查我们上一节提到的 `fallbackExecution` 属性，决定是直接执行还是丢弃。

**第四步：事务管理器触发回调（`PlatformTransactionManager` 的最终裁决）**
我们的主干业务继续执行，直到执行完毕，准备提交事务。
此时，`DataSourceTransactionManager`（最常用的事务管理器）接管了控制权。它在执行真实的数据库 `Connection.commit()` 操作的前后，会去查阅 `TransactionSynchronizationManager` 里的“挂号单”。

它的底层伪代码逻辑如下：

```java
// PlatformTransactionManager 执行提交的核心逻辑简写
protected void processCommit(DefaultTransactionStatus status) {
    try {
        // 1. 触发 beforeCommit 回调（你的 BEFORE_COMMIT 监听器在这里执行）
        triggerBeforeCommit(status);
        
        // 2. 真正执行底层数据库的 COMMIT
        doCommit(status);
        
        // 3. 触发 afterCommit 回调（你的 AFTER_COMMIT 监听器在这里执行！）
        triggerAfterCommit(status);
    } catch (Exception ex) {
        // ... 回滚逻辑
    } finally {
        // 4. 触发 afterCompletion 回调
        triggerAfterCompletion(status);
    }
}
```

---

### 💡 核心总结：感知事务的真相

1.  **没有魔法，全是回调**：`@TransactionalEventListener` 根本没有去实时“监听”数据库的状态。它只是借用了 Spring 的事件总线，把一段代码**延迟注册**到了事务管理器的回调列表中。
2.  **ThreadLocal 是桥梁**：事件发布器（Publisher）和事务管理器（TransactionManager）互相并不认识。它们之所以能配合，是因为它们都在往同一个 `ThreadLocal`（当前线程的上下文）里存取数据。这也是为什么在默认情况下，**事件发布和监听必须在同一个线程**才能生效的原因。

理解了这套基于 `ThreadLocal` 的回调机制，你就能瞬间明白为什么在 `AFTER_COMMIT` 阶段直接写数据库会报错，以及为什么引入异步 `@Async` 会导致事务失效。底层原理完全打通了！