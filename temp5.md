##### TransactionSynchronization 接口

**为了让事务能够感知Spring事件**，Spring设计了回调接口 **`TransactionSynchronization`**
- 此时，**事务不再是简单的commit或rollback，而是拥有四个生命周期的概念**，而回调接口 **`TransactionSynchronization`**提供了**在不同生命周期注入回调函数的钩子**
- 实现 **`TransactionSynchronization`** 回调接口，并挂载到当前事务，事务在不同的生命周期调用回调函数
- **`TransactionSynchronization`** 回调接口的核心方法签名伪代码如下，因此监听器不仅能在成功提交后触发，还能在事务提交前、事务回滚、无论事务是否提交的情况下触发


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


---

**事务挂起与恢复**
- 在执行事务A中的`@Transactional(propagation = Propagation.REQUIRES_NEW)` 方法，调用`suspend()`方法挂起本事务，并开启全新的事务B，等事务B执行结束后，调用`resume()`恢复事务A
- 事务挂起与恢复时需要释放或重新获取资源，比如ThreadLocal变量，但不包括连接，新事务将获取新连接

**beforeCommit(boolean readOnly)**：专属“成功者”的最后狂欢
- 触发条件：只有当事务所有的SQL执行完，并准备正常提交时，它才会被调用
- 如果事务在前面已经抛出了异常，或者被标记为 setRollbackOnly()，这个方法会被直接跳过。
- 核心使命：数据的最终校验与Flush
- 框架级应用：这是为 Hibernate/JPA 等 ORM 框架量身定制的。当业务逻辑执行完，准备提交数据库前，Hibernate 会利用这个钩子，把内存（Session 一级缓存）里的所有脏数据生成真正的 UPDATE/INSERT SQL 语句，并通过网络发送给 MySQL（即 flush 操作）。
- 危险动作：如果你在这个方法里抛出异常，原本即将成功的事务会被强制打断，转为回滚。

**`beforeCompletion`**
- **事务结束前调用**，即无论事务是即将提交还是即将回滚，它都必定被调用
- 正常提交时，**在`beforeCommit` 之后**执行
- 回滚时，在`rollback`之前执行

**`afterCommit()`**
- 事务已提交成功，触发该方法
- 数据已固化，领域状态已变更，调用`afterCommit()`中注册的回调函数：发布领域事件、向消息队列发送异步消息

**`afterCompletion(int status)`**
- 无论事务是成功、失败、报错，该方法一定被执行
- 调用`afterCompletion(int status)`清理线程，将当前线程绑定的各种资源（缓存、ThreadLocal、临时文件）清理干净，防止内存泄漏
- status表示当前事件状态：`STATUS_COMMITTED`、`STATUS_ROLLED_BACK`

---



### 3. 微观机制：它是怎么工作的？

你可能会问：我实现了这个接口，Spring 怎么知道我的存在？

这就不得不提 `TransactionSynchronizationManager` 这个管家了。在业务代码中，你可以通过下面这行代码，把你的“寄生虫”挂载到当前线程的事务上：

```java
// 手动注册寄生虫（实际开发中，@TransactionalEventListener 的底层帮你做了这一步）
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override
    public void afterCommit() {
        System.out.println("数据库已经提交啦！");
    }
});
```
事务管理器在执行 `commit` 或 `rollback` 的过程中，会去 `TransactionSynchronizationManager` 里把所有挂载的“寄生虫”拿出来，依次调用它们对应阶段的方法。

---

