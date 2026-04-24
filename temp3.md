

#### `@TransactionalEventListener`（待重构）

##### `@TransactionalEventListener`的四大事务阶段

`EventListener`注解，当主线程调用监听器方法时，立即执行，主线程等待监听器执行完后再继续执行

`@TransactionalEventListener` 注解是**事务观察者**，在**同步阻塞模型下的执行流程**如下：
- 主线程准备发布事件`publishEvent()`
- `@TransactionalEventListener` 的适配器**拦截到事件**，通过适配器方法将事件和监听器代码打包，塞进ThreadLocal中，将该ThreadLocal变量注册到`TransactionSynchronizationManager`，而不是立即执行监听器方法，**即向事务管理器注册回调函数**
  - **如果发布者压根没有标注`@Transactional`**，`@TransactionalEventListener`根本无法拦截事件，直接罢工不执行
- 完成后`publishEvent()`方法立即结束，执行后续业务代码，直到方法结束提交事务
- **监听器和事务的同步阻塞点**：
  - 复习知识：事务管理器PlatformTransactionManager作为无状态Bean负责向不同的数据源发送命令
  - 事务管理器在执行提交的过程中，检查线程的`TransactionSynchronizationManager`有没有东西，没有就正常执行，否则在四个同步点执行`TransactionSynchronizationManager`里的回调函数

```JAVA
//事务管理器执行本同步阻塞点的伪代码
protected final void processCommit(DefaultTransactionStatus status) {
    try {
        // 【同步点 1：BEFORE_COMMIT】
        // 去 ThreadLocal 遍历你的监听器，执行 beforeCommit 方法
        triggerBeforeCommit(status);
        
        // --- 核心动作 ---
        // 调用底层数据库驱动（如 JDBC）真正执行提交
        doCommit(status); 
        
        // 【同步点 2：AFTER_COMMIT】
        // 数据库已经 commit 成功！去 ThreadLocal 遍历你的监听器，执行 afterCommit 方法
        triggerAfterCommit(status);
        
    } catch (UnexpectedRollbackException ex) {
        // 【同步点 3：AFTER_ROLLBACK】(如果 commit 失败报错了)
        triggerAfterRollback(status);
    } finally {
        // 【同步点 4：AFTER_COMPLETION】
        // 无论如何，清空 ThreadLocal 里的东西，做最后的收尾
        triggerAfterCompletion(status);
    }
}
```

---


这四大特定阶段，由 `TransactionPhase` 枚举来定义，通过`@TransactionalEventListener`的配置是 `phase = TransactionPhase.AFTER_COMMIT`来使用

1. `BEFORE_COMMIT`（提交前校验与补录）
   * **触发时机**：主业务逻辑已经执行完毕，Hibernate/MyBatis 甚至可能已经执行了 `flush` 操作，但**数据库尚未真正执行 `COMMIT` 命令之前**。
   * **核心特征**：此时监听器依然**处于主事务的边界内**。
   * **典型应用场景**：
     * **同库同事务的数据补录**：比如订单创建完毕后，发布事件，监听器在同一个事务中插入一条“订单流转日志”。
     * **最终校验拦截**：在真正提交前做最后一次数据一致性检查，如果监听器抛出异常，整个主事务将被**拦截并回滚**。
   * **⚠️ 危险区**：它会拖慢主事务的提交速度。如果这里发生耗时操作，会导致数据库行锁迟迟不释放，引发并发性能问题。

2. `AFTER_COMMIT`（提交后放飞——最核心、默认选项）
   * **触发时机**：数据库已经成功执行了 `COMMIT` 命令，主业务的数据**已经永久固化到磁盘**。
   * **核心特征**：此时主事务**已经结束（Closed）**。这完美契合了我们在第一模块探讨的“DDD 事件契约”——既定事实已发生，不会再空欢喜。
   * **典型应用场景**：
     * **执行不可逆的副作用**：发送短信、发送邮件。
     * **跨限界上下文的最终一致性**：向 MQ（RabbitMQ/Kafka）投递消息，通知库存中心扣减库存。
   * **⚠️ 危险区（高频踩坑点）**：由于主事务已经提交，你**绝对不能**在 `AFTER_COMMIT` 监听器中直接执行数据库的 `UPDATE/INSERT` 操作（除非你显式开启一个新事务 `REQUIRES_NEW`，这点我们在后面的避坑指南中会细讲）。如果你在这里写库，它要么不生效，要么报连接获取异常。

3. `AFTER_ROLLBACK`（回滚后的灾后重建）
   * **触发时机**：主事务因为异常（或者主动要求）触发了 `ROLLBACK`，数据变更被撤销之后。
   * **核心特征**：此时状态已经恢复到事务发生之前，主业务的数据“全当没发生过”。
   * **典型应用场景**：
     * **外部资源的清理**：比如在主事务进行中，你提前上传了一张图片到阿里云 OSS。后来事务报错回滚了，数据库里没留存这条记录，你就需要在这个阶段去 OSS 把那张“孤儿图片”删掉，防止占用空间。
     * **缓存清理**：主业务在操作时提前删除了 Redis 缓存，事务回滚后，需要在这里进行补偿动作。

4. `AFTER_COMPLETION`（尘埃落定的最终收尾）
   * **触发时机**：无论主事务是成功提交（Commit）还是失败回滚（Rollback），只要事务生命周期彻底结束，它就会执行。
   * **核心特征**：类似于 Java 异常处理中的 `finally` 代码块。
   * **典型应用场景**：
     * **状态无关的清理动作**：比如清理当前线程的 `ThreadLocal` 上下文变量。
     * **监控与打点**：记录本次事务从开启到结束的总耗时，发送给 Prometheus 或系统日志，用于性能监控。

---

**降级配置：`fallbackExecution`**
- 在实际开发中，你可能会遇到一种尴尬的情况：**发布事件的方法上，根本就没有标注 `@Transactional`。**
- 默认情况下，如果 `@TransactionalEventListener` 发现当前线程**没有活跃的事务**，它会选择**直接罢工（不执行）**！因为它认为自己是绑定在事务上的，没有事务就没有它生存的土壤。
- 但有时候，我们的代码既会被事务方法调用，也会被非事务方法调用。为了防止事件无声无息地丢失，我们可以使用 `fallbackExecution` 属性：

```java
@TransactionalEventListener(
    phase = TransactionPhase.AFTER_COMMIT, 
    fallbackExecution = true // 关键兜底配置
)
public void handleOrderCreated(OrderCreatedEvent event) {
}
```
- **开启 `fallbackExecution = true` 后，它在无事务环境下就会退化（降级）为一个普通的同步监听器**，这在编写具有强兼容性的底层基础组件时非常有用。
- Spring底层做了兜底，伪代码如下，因此业务逻辑只需要开启降级，Spring会兜底是打包挂号，还是退化为 @EventListener立即同步执行

```JAVA
public void onApplicationEvent(ApplicationEvent event) {
    // 1. 检查当前线程是否存在活跃的事务
    if (TransactionSynchronizationManager.isActualTransactionActive()) {
        
        // 2. 如果有事务，走“挂号”流程
        // 包装成一个 Synchronization，塞进 ThreadLocal 列表
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionalEventSynchronization(event, this.handlerMethod, this.phase)
        );
        // 此时方法直接返回，监听器代码并没有执行
        
    } else {
        // 3. 如果没有事务，检查 fallbackExecution 配置
        if (this.fallbackExecution) {
            
            // 4. 允许兜底：退化成普通的 @EventListener，立刻【同步执行】
            this.processEvent(event); 
            
        } else {
            // 5. 不允许兜底：直接无视，事件丢失
            log.warn("No transaction, skipping event: " + event);
        }
    }
}
```


---

##### 事务的感知能力

在 Spring 中，事务不仅仅是一个简单的 `Connection.commit()`，**事务被抽象成了一个具有完整生命周期的概念**。**`TransactionSynchronization`回调接口能够让其他组件依附在事务的生命周期上**，比如缓存组件、消息队列、事件监听器

`@TransactionalEventListener`感知事务的能力是通过以下三个组件协作实现的：
- **事件广播器（EventMulticaster）**
- **事务同步管理器（TransactionSynchronizationManager）** 
-  **事务管理器（PlatformTransactionManager）** 


---


**`TransactionSynchronization`回调接口**伪代码
```java
public interface TransactionSynchronization extends Flushable {

    // 事务挂起时触发（比如遇到了 REQUIRES_NEW 的新事务）
    default void suspend() {}

    // 事务恢复时触发
    default void resume() {}

    // 【对应 BEFORE_COMMIT】
    default void beforeCommit(boolean readOnly) {}

    // 【对应 AFTER_COMPLETION 的前置准备】
    default void beforeCompletion() {}

    // 【对应 AFTER_COMMIT】
    default void afterCommit() {}

    // 【对应 AFTER_COMPLETION 和 AFTER_ROLLBACK】
    // status 参数会告诉你最终是 STATUS_COMMITTED 还是 STATUS_ROLLED_BACK
    default void afterCompletion(int status) {}
}
```

**核心结论：** `@TransactionalEventListener` 之所以能感知到四大阶段，就是因为它在底层**把自己伪装（包装）成了一个 `TransactionSynchronization` 实现类**，并挂载到了当前线程的事务上下文中。

---

**“偷梁换柱”：适配器模式与事件拦截**

- 当你写下 `@TransactionalEventListener` 注解时，Spring 在启动阶段并不会把它当成一个普通的 `@EventListener` 来处理。

- Spring 内部有一个专门的工厂类 `TransactionalEventListenerFactory`，它会为你的方法生成一个**适配器监听器**：`ApplicationListenerMethodTransactionalAdapter`。

- 当主业务调用 `ApplicationEventPublisher.publishEvent()` 时，**事件广播器（Multicaster）**会把事件推送给这个**适配器**，而不是直接执行你的业务代码
- 适配器的核心源码逻辑如下：

```java
// 适配器的事件处理逻辑（伪代码提炼）
@Override
public void onApplicationEvent(ApplicationEvent event) {
    // 1. 判断当前线程是否有事务？
    if (TransactionSynchronizationManager.isSynchronizationActive() &&
            TransactionSynchronizationManager.isActualTransactionActive()) {
        
        // 2. 有事务！创建一个 TransactionSynchronization 的实现类
        TransactionSynchronization transactionSynchronization = 
            createTransactionSynchronization(event);
            
        // 3. 核心动作：向 ThreadLocal 中的事务同步管理器“挂号登记”
        TransactionSynchronizationManager.registerSynchronization(transactionSynchronization);
        
    } else {
        // 4. 没有事务？走 fallbackExecution 降级逻辑（你前面已经学过的部分）
        if (this.fallbackExecution) {
            processEvent(event); // 直接执行
        }
    }
}
```

**关键点：** 广播器推送事件时，适配器只是把事件和你的方法引用**暂存**到了 `TransactionSynchronizationManager`（一个基于 `ThreadLocal` 的工具类）里。此时，主线程的 `publishEvent()` 直接返回，继续往下执行！

---


**PlatformTransactionManager 的协同调用**
- 主业务逻辑执行完毕，终于来到了方法的尽头，Spring AOP 切面准备结束事务。此时，舞台交给了事务管理器 `PlatformTransactionManager`（最常用的实现是 `DataSourceTransactionManager`）。

- 在它执行提交或回滚时，会调用基类 `AbstractPlatformTransactionManager` 的一系列流转方法。这时候，之前“挂号登记”的监听器才真正迎来执行时机：

```java
// AbstractPlatformTransactionManager 的事务提交流程（提炼与整合）
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        boolean beforeCompletionInvoked = false;
        try {
            // 【1. 触发 beforeCommit】
            // 从 ThreadLocal 获取所有 Synchronization，遍历执行 beforeCommit()
            triggerBeforeCommit(status);
            
            // 触发 beforeCompletion
            triggerBeforeCompletion(status);
            beforeCompletionInvoked = true;
            
            // --- 真正的数据库 commit ---
            doCommit(status); 
            // ------------------------

        } catch (UnexpectedRollbackException ex) {
            // 数据库 commit 失败，触发回滚
            triggerAfterRollback(status);
            throw ex;
        }

        // 【2. 触发 afterCommit】
        try {
            // 数据库成功提交后，遍历 ThreadLocal，执行 afterCommit()
            triggerAfterCommit(status);
        } finally {
            // 【3. 触发 afterCompletion】
            // 无论成功还是失败，最后清理现场
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    } finally {
        // 清空当前线程的 ThreadLocal 数据
        cleanupAfterCompletion(status);
    }
}
```

- 当你配置了 `phase = TransactionPhase.AFTER_COMMIT` 时，在上面的 `triggerAfterCommit(status)` 方法内部，Spring 会从 `ThreadLocal` 里取出之前那个适配器放进去的 `TransactionSynchronization` 对象，调用它的 `afterCommit()` 方法。而这个 `afterCommit()` 方法的内部实现，就是**通过反射，真正去执行你写在 `@TransactionalEventListener` 注解下的那个业务方法**。

---


回顾整个流程，这其实是一个教科书级别的**解耦设计**：

1. **发布者 (Publisher)**：只负责发消息，根本不知道监听器是同步的还是绑在事务上的，它发完就走。
2. **适配器 (Adapter)**：充当“拦截者”，它把事件拦下来，发现有事务，就不执行，而是转换成回调对象塞进 `ThreadLocal`。
3. **事务管理器 (TransactionManager)**：并不直接依赖事件机制。它只负责管理数据库连接，并在特定节点（如 commit 前后）无脑地去调用 `ThreadLocal` 里的 `TransactionSynchronization` 回调函数。

通过 `TransactionSynchronizationManager` 这个基于 `ThreadLocal` 的中介，事件机制与事务机制在**不相互强依赖（低耦合）**的前提下，完成了完美的跨模块协同。

##### 不可预期的空指针与连接泄露


**为什么在 `AFTER_COMMIT` 阶段更新数据库会失败？**
- 这通常被称为 **“静默失败（Silent Failure）”**，特别是当你使用 Hibernate/Spring Data JPA 时尤为明显。


- 结合上一阶段的源码知识，在 `AFTER_COMMIT` 触发时，底层状态是这样的：
  * **数据库层面：** `doCommit(status)` 已经执行完毕，数据库底层的事务已经结账走人了。
  * **Spring 层面：** 此时还在 `processCommit` 方法的 `finally` 块执行前，也就是说，**当前线程的 ThreadLocal 中，依然保留着原来那个事务的上下文（TransactionInfo）**。

---

 失败的根本原因
当你在这个监听器中调用 `XxxRepository.save()` 或 `XxxMapper.insert()` 时：
1.  ORM 框架（如 JPA）去检查当前环境，发现：“哦，ThreadLocal 里有事务上下文，那我直接加入这个已有的事务吧！”
2.  于是，你的 `save()` 操作产生的数据变更，被放进了一级缓存（Persistence Context / Session）。
3.  **但是（致命点）**：因为主事务的 `doCommit()` 已经执行过了，Spring **绝对不会**再去调用第二次 `commit` 或 `flush`。
4.  最终，你的方法执行结束，连接被归还，缓存被清空，**你的数据就像一阵风一样，从没真正落盘，且没有任何异常抛出**。

---

不可预期的空指针与连接泄露
* **空指针（NPE）**：在某些配置或 MyBatis 版本中，主事务提交后，底层的 `Connection` 可能已经被标记为归还或关闭。如果监听器代码强行去获取数据库连接执行写操作，可能会拿到一个处于“半关闭”状态的代理对象，从而抛出 NPE。
* **连接泄露隐患**：如果你的监听器执行了非常耗时的外部网络调用（比如调用第三方 API），会导致主事务的最后一步 `cleanupAfterCompletion()` 迟迟无法执行，原本应该归还给 HikariCP 的数据库连接被长时间霸占。

---

**破局之法：开启新事务 (`REQUIRES_NEW`)**

既然旧的事务已经“名存实亡”（提交完了但还没清理），我们就必须明确告诉 Spring：**不要用旧的，给我开个全新的！**

这就是 `@Transactional(propagation = Propagation.REQUIRES_NEW)` 出场的时候了。

 黄金代码范例
```java
@Component
public class OrderEventListener {

    // 1. 绑定事务阶段：主事务成功提交后才执行
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    // 2. 声明事务传播行为：挂起当前（已完成的）事务，开启一个全新的物理事务
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void handleOrderCreatedEvent(OrderCreatedEvent event) {
        // 在这里执行 INSERT 或 UPDATE，必定能成功落盘
        auditLogRepository.save(new AuditLog("Order Created", event.getOrderId()));
    }
}
```
---

 底层运行轨迹
当 AOP 代理拦截到 `REQUIRES_NEW` 时：
1.  它发现当前线程有事务上下文（虽然已经 commit 了）。
2.  触发 `TransactionSynchronizationManager.suspend()`，**把旧事务挂起**（暂时挪到一边）。
3.  向数据库连接池（如 HikariCP）**申请一根全新的数据库连接（Connection B）**。
4.  开启新事务，执行监听器里的业务代码。
5.  执行新事务的 `doCommit()`，数据真实落盘。
6.  关闭新事务，释放 Connection B。
7.  触发 `resume()`，恢复旧事务，让主线程继续去完成最后的清理收尾工作。

---

高危警告：连接池死锁（Pool Exhaustion Deadlock）
虽然 `REQUIRES_NEW` 完美解决了数据无法保存的问题，但它引入了一个致命的并发隐患：**一个请求，同时占用了两根数据库连接**。
* **主线程**持有 `Connection A`（等待监听器执行完才释放）。
* **监听器线程**（默认是同一个线程，同步阻塞）申请 `Connection B`。
* **场景推演**：假设你的连接池最大数量是 10。如果瞬间并发来了 10 个请求，它们全都拿到了 `Connection A`，然后准备执行监听器，同时去连接池申请 `Connection B`。此时连接池空了，10 个请求都在死等 `Connection B`，而 `Connection A` 又死死不放。**砰！连接池彻底死锁，系统宕机。**

---

既然同步执行 `@Transactional(propagation = Propagation.REQUIRES_NEW)` 会让一条线程同时霸占两个连接，甚至引发死锁，那么你觉得，**如果结合 `@Async` 注解将监听器彻底异步化，是不是就能完美避开这个雷区了呢？我们需要探讨一下 `@Async` 和 `@TransactionalEventListener` 联手时的化学反应吗？**




