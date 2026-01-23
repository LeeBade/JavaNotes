#### Atomic

`java.util.concurrent.atomic` 基于CAS和Unsafe实现的一系列原子操作类，无锁实现高效的线程安全操作，避免加锁带来的上下文切换和阻塞等待开销
- Atomic底层使用Unsafe提供的native方法直接操作堆外内存完成CAS，以 `AtomicInteger` 的 `getAndIncrement()` 方法为例

```java
public final int getAndIncrement() {
    // 调用 Unsafe 类的 getAndAddInt 方法
    // this: 当前 AtomicInteger 对象
    // valueOffset: 实际 int 值在内存中的偏移量
    // 1: 增加的数值
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

```

`getAndAddInt` 内部利用了 CAS 操作。CAS 是一种无锁算法，它包含三个操作数：

* **内存位置 (V)**
* **预期原值 (A)**
* **新值 (B)**

只有当内存位置 V 的值等于预期原值 A 时，处理器才会将该位置更新为新值 B；否则，处理器不做任何操作（或重试）。这种机制允许在多线程环境中原子地更新值，而无需暂停线程。

## 三、 Java 原子操作类分类详解

Java `atomic` 包中的类主要可以分为四类：基本类型、数组类型、引用类型和字段更新类型。

### 3.1 原子更新基本类型

主要包括：

* `AtomicBoolean`：原子更新布尔类型。
* `AtomicInteger`：原子更新整型。
* `AtomicLong`：原子更新长整型。

**常用方法（以 AtomicInteger 为例）：**

* `addAndGet(int delta)`：增加指定值，返回新值。
* `incrementAndGet()`：自增 1，返回新值。
* `getAndSet(int newValue)`：设置新值，返回旧值。
* `getAndIncrement()`：获取当前值，并自增 1。

**AtomicBoolean 的实现细节：**
`AtomicBoolean` 内部也会转化为整型进行处理（true 为 1，false 为 0），最终调用 `unsafe.compareAndSwapInt` 实现原子更新。

```java
public final boolean compareAndSet(boolean expect, boolean update) {
    int e = expect ? 1 : 0;
    int u = update ? 1 : 0;
    return unsafe.compareAndSwapInt(this, valueOffset, e, u);
}

```

### 3.2 原子更新数组

如果需要原子地更新数组中的某个元素，可以使用以下类：

* `AtomicIntegerArray`：原子更新整型数组里的元素。
* `AtomicLongArray`：原子更新长整型数组里的元素。
* `AtomicReferenceArray`：原子更新引用类型数组里的元素。

**使用示例：**
这些类的方法通常多一个参数 `i`，用于指定数组索引。

```java
int[] value = new int[]{1, 2, 3};
AtomicIntegerArray integerArray = new AtomicIntegerArray(value);
// 对数组索引为 1 的元素原子加 5
int result = integerArray.getAndAdd(1, 5); 
// 结果：integerArray 变为 [1, 7, 3]，返回值为旧值 2

```

### 3.3 原子更新引用类型

用于原子地更新对象引用：

* `AtomicReference`：原子更新引用类型。
* `AtomicReferenceFieldUpdater`：原子更新引用类型里的字段。
* `AtomicMarkableReference`：带有标记位的引用类型。

**使用示例（AtomicReference）：**

```java
public class AtomicDemo {
    private static AtomicReference<User> reference = new AtomicReference<>();

    public static void main(String[] args) {
        User user1 = new User("a", 1);
        reference.set(user1);
        
        User user2 = new User("b", 2);
        // 原子地将 user1 替换为 user2
        User oldUser = reference.getAndSet(user2);
    }
}

```

### 3.4 原子更新字段类型

如果需要更新某个类中现有的 `volatile` 字段，可以使用字段更新器：

* `AtomicIntegerFieldUpdater`：原子更新整型字段。
* `AtomicLongFieldUpdater`：原子更新长整型字段。
* `AtomicStampedReference`：带有版本号的引用类型，**用于解决 CAS 的 ABA 问题**。

**使用条件：**

1. 必须使用静态方法 `newUpdater()` 创建更新器，并指定类和字段名。
2. 被更新的字段必须使用 `public volatile` 修饰。

**使用示例（AtomicIntegerFieldUpdater）：**

```java
public class AtomicDemo {
    // 创建更新器，指定更新 User 类的 age 字段
    private static AtomicIntegerFieldUpdater<User> updater = 
            AtomicIntegerFieldUpdater.newUpdater(User.class, "age");

    static class User {
        private String userName;
        public volatile int age; // 必须是 public volatile

        public User(String userName, int age) {
            this.userName = userName;
            this.age = age;
        }
    }

    public static void main(String[] args) {
        User user = new User("a", 1);
        // 原子地将 age 字段加 5
        updater.getAndAdd(user, 5); 
        // user.age 变为 6
    }
}

```

## 四、 总结

Java 的原子操作类通过硬件级别的 CAS 指令和 Unsafe 类，提供了比 `synchronized` 更轻量级的线程安全机制。它们不仅包括基本数据类型的原子更新，还覆盖了数组、引用以及对象字段的更新。

* **性能**：在并发冲突不剧烈的场景下，原子类通常比锁具有更好的性能。
* **ABA 问题**：在使用 CAS 时需注意 ABA 问题（即值被修改为 B 又改回 A，CAS 无法感知变化），可以使用 `AtomicStampedReference` 通过版本号来解决。

掌握这些原子类的使用及其背后的 CAS 原理，是构建高性能并发应用的基础。