
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

### 2.7 数组操作

`Unsafe` 提供了计算数组元素内存定位的方法，使得原子地更新数组元素成为可能（如 `AtomicIntegerArray`）。

* `arrayBaseOffset(Class<?> arrayClass)`: 获取数组第一个元素的偏移地址。
* `arrayIndexScale(Class<?> arrayClass)`: 获取数组中元素的增量地址（即每个元素占用的字节数）。

**定位公式**：


## 三、 总结

`sun.misc.Unsafe` 是 Java 世界的一把双刃剑：

1. **极高的性能与控制力**：它赋予了 Java 操作底层系统资源的能力，是高性能网络框架（如 Netty）、并发库（JUC）和序列化工具的基石。
2. **风险**：它破坏了 Java 的类型安全和内存安全机制。使用不当会导致 JVM 崩溃（Crash）或难以排查的内存泄漏。

随着 Java 版本的迭代，为了安全性，官方正在逐步限制 `Unsafe` 的直接访问，并引入更安全的替代品（如 `VarHandle`）。但在理解 Java 底层原理时，`Unsafe` 始终是绕不开的关键知识点。