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

`Unsafe`提供了一种**底层**、**不安全**的机制来直接访问和操作内存、线程和对象

Unsafa是**final的**、**构造器私有化**的，`getUnsafe`静态方法返回**全局唯一的静态Unsafe单例对象**，但是`getUnsafe`方法会检查调用者的`classLoader`，只有启动类加载器加载的类才能够调用`getUnsafe`方法，否则抛出`SecurityException`异常。Unsafe的功能可以被划分为8类

**通过反射获取`Unsafe`单例对象**
```JAVA
public static Unsafe getUnsafe() throws IllegalAccessException {
    Field unsafeField = Unsafe.class.getDeclaredField("theUnsafe");
    unsafeField.setAccessible(true);
    Unsafe unsafe =(Unsafe) unsafeField.get(null);
    return unsafe;
}
```
1. **分配堆外内存**

```JAVA
//分配新的本地空间
public native long allocateMemory(long bytes);
//重新调整内存空间的大小
public native long reallocateMemory(long address, long bytes);
//将内存设置为指定值
public native void setMemory(Object o, long offset, long bytes, byte value);
//内存拷贝
public native void copyMemory(Object srcBase, long srcOffset,Object destBase, long destOffset,long bytes);
//清除内存
public native void freeMemory(long address);
```
2. **内存屏障**




3. **对象操作**
   - `objectFieldOffset`获取指定字段的内存相对偏移量
   - 普通读写：提供8对put/get方法写读8种基本数据类型属性值，对Object类型的对象的读写基于对象引用
   - volatile读写：在普通方法后加上Volatile后缀，通过强制与主存交互，保证可见性和有序性
   - 有序性读写：在普通方法后加上Order后缀

```JAVA

long fieldOffset = unsafe.objectFieldOffset(User.class.getDeclaredField("age"));
public native int getInt(Object o, long offset);
 public native void putInt(Object o, long offset, int x);

 //在对象的指定偏移地址获取一个对象引用
 public native Object getObject(Object o, long offset);
 //在对象指定偏移地址写入一个对象引用
 public native void putObject(Object o, long offset, Object x);
```


4. 数组操作
5. CAS操作
6. 线程调度
7. Class操作
8. 系统信息
#### Atomic


#### LongAdder