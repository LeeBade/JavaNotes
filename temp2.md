**`@TransactionalEventListener`感知事务的本质是实现了`TransactionSynchronization` 接口**


**监听器没有实现 `TransactionSynchronization` 接口，是怎么被事务管理器调用的？**

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void doSomething(MyEvent event) { ... }
```


`ApplicationListenerMethodTransactionalAdapter`

- 在 Spring 启动阶段，当它扫描到带有 `@TransactionalEventListener` 的方法时，它会偷偷创建一个**代理对象**，**也是一个适配器**，全称叫 `ApplicationListenerMethodTransactionalAdapter`。
- 这个适配器是整个魔术的核心。当业务代码执行 `publisher.publishEvent(event)` 时，**事件其实是交给了这个适配器，而不是直接交给你的业务方法**！



- 当适配器拦截到事件后，它的执行链路是这样的（高度提炼的伪代码）：

```java
// 适配器内部的事件处理逻辑
@Override
public void onApplicationEvent(ApplicationEvent event) {
    
    // 1. 探针：当前线程有活跃的事务吗？
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        
        // 2. 偷梁换柱：将“你的方法”和“当前事件”打包，伪装成一个 TransactionSynchronization 寄生虫！
        TransactionSynchronization synchronization = new TransactionalEventSynchronization(event, this.listenerMethod);
        
        // 3. 挂号登记：把这个寄生虫塞进当前线程的 ThreadLocal 口袋里
        TransactionSynchronizationManager.registerSynchronization(synchronization);
        
        // 4. 立即溜走：不执行业务方法，直接 return！
        return; 
    }
    // ... 无事务的降级逻辑（稍后小节讲）...
}
```

**关键角色揭秘：`TransactionSynchronizationManager`**
这是一个非常霸气的类，但它的本质极其简单——**它就是一个封装了 `ThreadLocal` 的管家**。
它在当前线程里维护了一个 `List<TransactionSynchronization>`。

**“偷梁换柱”的完整拼图：**
1. 业务主线程调用 `publishEvent()`。
2. 适配器接到事件，发现当前在事务中。
3. 适配器不执行你的监听器代码，而是生成一个回调对象（寄生虫），塞进 `ThreadLocal` 的列表里（这叫**挂号**）。
4. `publishEvent()` 瞬间执行完毕，主线程继续往下走！
5. 等到主线程走到 AOP 层面准备 `commit` 的时候，事务管理器（TransactionManager）再去翻看 `ThreadLocal`，把刚才挂号的寄生虫拿出来，依次触发它们的 `afterCommit()` 方法。

---

