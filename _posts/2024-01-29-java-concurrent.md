---
title: "Java并发常见知识"
categories:
  - 八股
  - 并发
tags:
  - Java并发常见知识
---

# Java并发常见知识

首先要对Java内存模型有大概的认知。

## volatile

意思是可见性，在 Java 中，`volatile` 关键字可以保证变量的可见性，如果我们将变量声明为 **`volatile`** ，这就指示 JVM，**<u>这个变量是共享且不稳定的，每次使用它都到主存中进行读取</u>**。

`volatile` 关键字能保证数据的可见性，但不能保证数据的原子性。`synchronized` 关键字两者都能保证。

### 如何禁止指令重排序？

**在 Java 中，`volatile` 关键字除了可以保证变量的可见性，还有一个重要的作用就是防止 JVM 的指令重排序。** 如果我们将变量声明为 **`volatile`** ，在对这个变量进行读写操作的时候，会通过插入特定的 **内存屏障** 的方式来禁止指令重排序。

### **双重校验锁实现对象单例（线程安全）**：

```java
public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public  static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

>   #### 双重校验锁
>
>   确保只有一个实例被创建并提供全局访问点
>
>   避免了每次获取实例都进行同步操作的性能损耗。
>
>   双重校验锁的必要性可以从以下几个方面来说明：
>
>   1.  **线程安全性：** 在多线程环境下，如果不使用双重校验锁，而是简单地在方法上加上`synchronized`关键字来实现同步，那么每次获取实例时都会进行同步，这会造成性能上的开销。而使用双重校验锁可以减少同步的开销，只有在实例为null的情况下才进行同步操作，避免了大部分情况下的性能损耗。
>   2.  **延迟加载：** 双重校验锁还可以实现**延迟加载**，即只有在第一次获取实例时才会创建实例，而不是在类加载时就创建实例，这样可以节省系统资源，提高系统的性能。
>   3.  **解决单例模式的线程安全问题：** 单例模式需要保证在多线程环境下只有一个实例被创建，而双重校验锁可以很好地解决这个问题，即使在高并发的情况下也能保证线程安全性。
>   4.  **提高性能：** 双重校验锁通过两次检查实现了对同步块的减少，从而提高了程序的性能。第一次检查是为了避免不必要的同步，第二次检查是为了确保在多个线程同时通过了第一次检查时只有一个线程可以创建实例。

`uniqueInstance` 采用 `volatile` 关键字修饰也是很有必要的， `uniqueInstance = new Singleton();` 这段代码其实是分为三步执行：

1.  为 `uniqueInstance` 分配内存空间
2.  初始化 `uniqueInstance`
3.  将 `uniqueInstance` 指向分配的内存地址

volatile可以保证 1->2->3顺序不被重排导致多线程环境下错误。

## synchronized

在 Java 早期版本中，`synchronized` 属于 **重量级锁**，效率低下。这是因为监视器锁（monitor）是依赖于底层的操作系统的 `Mutex Lock` 来实现的，Java 的线程是映射到操作系统的原生线程之上的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高。

不过，在 Java 6 之后，`synchronized` 引入了大量的优化让 `synchronized` 锁的效率提升了很多。JDK 源码、很多开源框架都大量使用了 `synchronized` 。

### Usage

**1、修饰实例方法** （锁当前对象实例）

进入同步代码前要获得 **当前对象实例的锁** 。

```java
synchronized void method() {
    //业务代码
}
```

**2、修饰静态方法** （锁当前类）

给当前类加锁，会作用于类的所有对象实例 ，进入同步代码前要获得 **当前 class 的锁**。

```java
synchronized static void method() {
    //业务代码
}
```

**3、修饰代码块** （锁指定对象/类）

对括号里指定的对象/类加锁：

-   `synchronized(object)` 表示进入同步代码库前要获得 **给定对象的锁**。
-   `synchronized(类.class)` 表示进入同步代码前要获得 **给定 Class 的锁**

```java
synchronized(this) {
    //业务代码
}
```

### 构造方法可以用 synchronized 修饰么？

先说结论：**构造方法不能使用 synchronized 关键字修饰。**

构造方法本身就属于线程安全的，不存在同步的构造方法一说。

### 底层原理

jvm 编译时处理

`synchronized` **<u>同步语句块</u>**的实现使用的是 `monitorenter` 和 `monitorexit` 指令，其中 `monitorenter` 指令指向同步代码块的开始位置，`monitorexit` 指令则指明同步代码块的结束位置。

`synchronized` 修饰的**<u>方法</u>**并没有 `monitorenter` 指令和 `monitorexit` 指令，取得代之的确实是 `ACC_SYNCHRONIZED` 标识，该标识指明了该方法是一个同步方法。JVM 通过该 `ACC_SYNCHRONIZED` 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

### **为什么添加 synchronized 也能保证变量的可见性呢？**

因为：

1.  线程**解锁前，必须把共享变量的最新值刷新到主内存中。**
2.  线程**加锁前，将清空工作内存中共享变量的值**，从而<u>使用共享变量时需要从主内存中重新读取最新的值。</u>
3.  volatile 的可见性都是通过内存屏障（Memnory Barrier）来实现的。
4.  synchronized 靠操作系统内核互斥锁实现，相当于 JMM 中的 lock、unlock。退出代码块时刷新变量到主内存。

## ReentrantLock

`ReentrantLock` 实现了 `Lock` 接口，是一个可重入且独占式的锁，和 `synchronized` 关键字类似。不过，`ReentrantLock` 更灵活、更强大，增加了轮询、超时、中断、公平锁和非公平锁等高级功能。

`ReentrantLock` 默认使用非公平锁，也可以通过构造器来显式的指定使用公平锁。

```java
// 传入一个 boolean 值，true 时为公平锁，false 时为非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

>   ### 公平锁和非公平锁有什么区别？
>
>   -   **公平锁** : 锁被释放之后，先申请的线程先得到锁。性能较差一些，因为公平锁为了保证时间上的绝对顺序，上下文切换更频繁。
>   -   **非公平锁**：锁被释放之后，后申请的线程可能会先获取到锁，是随机或者按照其他优先级排序的。**性能更好，但可能会导致某些线程永远无法获取到锁。**

### synchronized 和 ReentrantLock

**<u>两者都是可重入锁</u>**，线程可以再次获取自己的内部锁。JDK 提供的所有现成的 `Lock` 实现类，包括 `synchronized` 关键字锁都是可重入的。

#### synchronized 依赖于 JVM 而 ReentrantLock 依赖于 API

`synchronized` 是依赖于 JVM 实现的，`ReentrantLock` 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock() 方法配合 try/finally 语句块来完成）。

#### **选择性通知的实现**

`synchronized`关键字与`wait()`和`notify()`/`notifyAll()`方法相结合可以实现等待/通知机制。`ReentrantLock`类需要借助于`Condition`接口与`newCondition()`方法。

>#### `Condition`接口
>
>**在使用`notify()/notifyAll()`方法进行通知时，被通知的线程是由 JVM 选择的，用`ReentrantLock`类结合`Condition`实例可以实现“选择性通知”** ，这个功能非常重要，而且是 `Condition` 接口默认提供的。
>
>而`synchronized`关键字就相当于整个 `Lock` 对象中只有一个`Condition`实例，所有的线程都注册在它一个身上。
>
>如果执行`notifyAll()`方法的话就会通知所有处于等待状态的线程，这样会造成很大的效率问题。
>
>而`Condition`实例的`signalAll()`方法，只会唤醒注册在该`Condition`实例中的所有等待线程。

## 读写锁

ReentrantReadWriteLock 、 StampedLock

`ReentrantReadWriteLock` 其实是两把锁，一把是 `WriteLock` (写锁)，一把是 `ReadLock`（读锁） 。**<u>读锁是共享锁，写锁是独占锁。</u>**读锁可以被同时读，可以同时被多个线程持有，而写锁最多只能同时被一个线程持有。

需要注意的是`StampedLock`不可重入，不支持条件变量 `Condition`，对中断操作支持也不友好（使用不当容易导致 CPU 飙升）。如果你需要用到 `ReentrantLock` 的一些高级性能，就不太建议使用 `StampedLock` 了。

另外，`StampedLock` 性能虽好，但使用起来相对比较麻烦，一旦使用不当，就会出现生产问题。

## 乐观锁和悲观锁

**悲观锁：共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程**。

 Java 中`synchronized`和`ReentrantLock`等独占锁就是悲观锁思想的实现。

**乐观锁**：认为共享资源每次被访问的时候不会出现问题，线程可以不停地执行，无需加锁也无需等待，只是**<u>在提交修改的时候去验证对应的资源（也就是数据）是否被其它线程修改了</u>**（具体方法可以使用版本号机制或 CAS 算法）。

高并发的场景下，乐观锁相比悲观锁来说，不存在锁竞争造成线程阻塞，也不会有死锁的问题，在性能上往往会更胜一筹。但是，**如果冲突频繁发生（写占比非常多的情况）**，会频繁失败和重试（悲观锁的开销是固定的），这样同样会非常影响性能，导致 CPU 飙升。

### 如何实现乐观锁？

乐观锁一般会使用版本号机制或 CAS 算法实现

#### 版本号机制

一般是在数据表中加上一个数据版本号 `version` 字段，表示数据被修改的次数。当数据被修改时，`version` 值会加一。当线程 A 要更新数据值时，在读取数据的同时也会读取 `version` 值，在提交更新时，若刚才读取到的 version 值为当前数据库中的 `version` 值相等时才更新，否则重试更新操作，直到更新成功。

#### CAS 算法

CAS 的全称是 **Compare And Swap（比较与交换）** ，用于实现乐观锁，被广泛应用于各大框架中。CAS 的思想很简单，就是用一个预期值和要更新的变量值进行比较，两值相等才会进行更新。

>   Java 语言并没有直接实现 CAS，CAS 相关的实现是通过 C++ 内联汇编的形式实现的（JNI 调用）。因此， CAS 的具体实现和操作系统以及 CPU 都有关系。
>
>   `sun.misc`包下的`Unsafe`类提供了`compareAndSwapObject`、`compareAndSwapInt`、`compareAndSwapLong`方法来实现的对`Object`、`int`、`long`类型的 CAS 操作

#### ABA 问题是乐观锁最常见的问题。

ABA 问题的解决思路是在变量前面追加上**版本号或者时间戳**。

#### 循环时间长开销大

CAS 经常会用到自旋操作来进行重试，也就是不成功就一直循环执行直到成功。如果长时间不成功，会给 CPU 带来非常大的执行开销。

#### 只能保证一个共享变量的原子操作

CAS 只对单个共享变量有效，当操作涉及跨多个共享变量时 CAS 无效。但是从 JDK 1.5 开始，提供了`AtomicReference`类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行 CAS 操作.所以我们可以使用锁或者利用`AtomicReference`类把多个共享变量合并成一个共享变量来操作。