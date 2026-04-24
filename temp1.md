#### @TransactionalEventListener

我们已经清晰认识到**当数据库成功提交后才能执行监听器**

难点在于事务管理器`PlatformTransactionManager`是非常纯粹的，它底层只和 JDBC 的 `Connection` 打交道，要么commit要么rollback。**事务管理器根本不认识Spring事件**



---

##### TransactionSynchronization 接口

认识 Spring 事务生命周期的“寄生虫”。

事务挂起、恢复与状态回调的基本定义。
##### 底层协作：“偷梁换柱”的适配器模式

ApplicationListenerMethodTransactionalAdapter 如何拦截事件。

事件向 TransactionSynchronizationManager (ThreadLocal) 的“挂号登记”机制。

##### 四大事务阶段

##### 降级机制：fallbackExecution


##### AFTER_COMMIT 阶段的静默失败