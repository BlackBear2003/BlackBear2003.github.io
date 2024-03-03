---
title: "ConcurrentHashMap"
categories:
  - 八股
tags:
  - ConcurrentHashMap
---

# ConcurrentHashMap

## jdk1.7

### 存储结构

![Java 7 ConcurrentHashMap 存储结构](/assets/images/java7_concurrenthashmap.png)

Java 7 中 `ConcurrentHashMap` 的存储结构如上图，`ConcurrnetHashMap` 由很多个 `Segment` 组合，而每一个 `Segment` 是一个类似于 `HashMap` 的结构，所以每一个 `HashMap` 的内部可以进行扩容。`Segment` 的个数一旦**初始化就不能改变**，默认 `Segment` 的个数是 16 个，`ConcurrentHashMap` 默认支持最多 16 个线程并发。

### 初始化

1.  必要参数校验。
2.  校验并发级别 `concurrencyLevel` 大小，如果大于最大值，重置为最大值。无参构造**默认值是 16.**
3.  寻找并发级别 `concurrencyLevel` 之上最近的 **2 的幂次方**值，作为初始化容量大小，**默认是 16**。
4.  记录 `segmentShift` 偏移量，这个值为【容量 = 2 的 N 次方】中的 N，在后面 Put 时计算位置时会用到。**默认是 32 - sshift = 28**.
5.  记录 `segmentMask`，默认是 ssize - 1 = 16 -1 = 15.
6.  <u>**初始化 `segments[0]`**，**默认大小为 2**，**负载因子 0.75**，**扩容阀值是 2\*0.75=1.5**，插入第二个值时才会进行扩容。</u>

### put操作

计算要 put 的 key 的位置，获取指定位置的 `Segment`。

如果指定位置的 `Segment` 为空，则初始化这个 `Segment`.

**初始化 Segment 流程：**

1.  检查计算得到的位置的 `Segment` 是否为 null.

2.  为 null 继续初始化，使用 `Segment[0]` 的容量和负载因子<u>创建一个 `HashEntry` 数组。</u>

3.  再次检查计算得到的指定位置的 `Segment` 是否为 null。因为这时可能有其他线程进行了操作。

4.  使用创建的 `HashEntry` 数组初始化这个 Segment.

    ```java
    // 创建一个 cap 容量的 HashEntry 数组
    HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) { 
    		Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
      	......
    }
    ```

5.  自旋判断计算得到的指定位置的 `Segment` 是否为 null，使用 CAS 在这个位置赋值为 `Segment`.

>   ```java
>   // 自旋检查 u 位置的 Segment 是否为null
>   while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))==null) {
>       // 使用CAS 赋值，只会成功一次
>       if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
>           break;
>   }
>   ```
>
>   1.  `UNSAFE.getObjectVolatile(ss, u)`：这里使用了`Unsafe`类的`getObjectVolatile`方法来获取`ss`对象(一个Segment数组)的第`u`个位置的值。`getObjectVolatile`是一个原子操作，保证了在多线程环境下，读取的是最新的值，并且不会因为缓存导致的问题而出现错误的结果。
>   2.  `(seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null`：将`ss`的第`u`个位置的值赋给`seg`，然后检查`seg`是否为`null`。如果为`null`，则表示该位置的`Segment`还未被初始化。
>   3.  `if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))`：这里使用`compareAndSwapObject`方法，尝试将`ss`的第`u`个位置的值从`null`替换为`seg = s`，即将其初始化为`s`。这是一个原子操作，仅在当前该位置的值为`null`时才会成功替换，并且只会成功一次。
>   4.  `break;`：如果`compareAndSwapObject`成功执行了替换操作，则跳出循环，表示该位置已经被成功初始化。
>
>   #### UNSAFE
>
>   `UNSAFE`类是Java中`sun.misc.Unsafe`类的非公开API。它提供了对底层操作系统和Java虚拟机的低级访问，允许Java程序直接进行内存操作、线程控制、CAS操作等，通常用于实现高效的并发算法、调优以及与本地代码的交互。
>
>   虽然`Unsafe`类提供了丰富的功能，但由于其操作涉及底层系统和内存，如果使用不当，容易导致程序出现严重的安全问题或者不可预测的行为，因此它被标记为“不稳定”的API，不建议直接在生产环境中使用。在Java 9及以后的版本中，对`Unsafe`类的访问受到了更严格的限制。
>
>   `Unsafe`类的常见用途包括：
>
>   1.  **直接内存访问**：通过`allocateMemory`和`putXXX`、`getXXX`等方法，可以直接操作内存，避免了Java对象的开销和垃圾回收的影响，用于实现高性能的数据结构或者与本地代码的交互。
>   2.  **线程控制**：包括创建线程、挂起线程、恢复线程等操作，可以实现更灵活的线程管理策略。
>   3.  **CAS操作**：提供了`compareAndSwapXXX`系列方法，用于实现无锁算法，比如`AtomicXXX`类的底层实现就是基于CAS操作。
>   4.  **类和对象操作**：包括直接操作类和对象的内存布局，例如获取字段的偏移量、修改字段的值等。
>
>   总之，`Unsafe`类提供了Java中非常底层的功能，能够让开发者绕过Java语言的安全性和封装性，直接进行系统级别的操作，但同时也需要开发者对底层系统和内存管理有足够的了解，以避免出现潜在的安全问题和不稳定行为。

6.   **`Segment.put` 插入 key,value 值。**

由于 `Segment` 继承了 `ReentrantLock`，所以 `Segment` 内部可以很方便的获取锁，put 流程就用到了这个功能。

1.  `tryLock()` 获取锁，获取不到使用 **`scanAndLockForPut`** 方法继续获取。

2.  计算 put 的数据要放入的 index 位置，然后获取这个位置上的 `HashEntry` 。

3.  遍历 put 新元素，为什么要遍历？因为这里获取的 `HashEntry` 可能是一个空元素，也可能是链表已存在，所以要区别对待。

    如果这个位置上的 **`HashEntry` 不存在**：

    1.  如果当前容量大于扩容阀值，小于最大容量，**进行扩容**。
    2.  直接头插法插入。

    如果这个位置上的 **`HashEntry` 存在**：

    1.  判断链表当前元素 key 和 hash 值是否和要 put 的 key 和 hash 值一致。一致则替换值
    2.  不一致，获取链表下一个节点，直到发现相同进行值替换，或者链表表里完毕没有相同的。 
        1.  如果当前容量大于扩容阀值，小于最大容量，**进行扩容**。
        2.  直接链表头插法插入。

4.  如果要插入的位置之前已经存在，替换后返回旧值，否则返回 null.

这里面的第一步中的 `scanAndLockForPut` 操作这里没有介绍，这个方法做的操作就是不断的自旋 `tryLock()` 获取锁。当自旋次数大于指定次数时，使用 `lock()` 阻塞获取锁。在自旋时顺表获取下 hash 位置的 `HashEntry`。

### Get操作

1.   计算得到key对应的Segment位置，拿到这个Segment，再拿到这个Segment的HashEntry
2.   遍历指定位置查找相同 key 的 value 值。

>   ```java
>       if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null && (tab = s.table) != null) {
>           for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile(tab, ((long)(((tab.length - 1) & h)) <<TSHIFT) + TBASE);e != null; e = e.next) {
>               // 如果是链表，遍历查找到相同 key 的 value。
>               K k;
>               if ((k = e.key) == key || (e.hash == h && key.equals(k)))
>                   return e.value;
>           }
>       }
>   ```
>
>   -   但是它只是先拿出这个Entry，然后开始遍历，读取key的value，那怎么保证在遍历的过程中没有别的线程修改了这个Entry呢？
>
>   在这段代码中，并没有直接进行修改操作，而是进行了读取操作。在多线程环境下，确实存在一个线程读取 Entry 的同时，另一个线程可能会修改这个 Entry。
>
>   但是，这段代码保证了在读取过程中，不会出现数据不一致的情况，主要是因为：
>
>   1.  对 `segments` 和 `s.table` 的读取是通过 `UNSAFE.getObjectVolatile` 方法进行的，保证了在多线程环境下，线程能够读取到最新的数据。因此，在获取到 Segment 和 HashEntry 数组后，线程得到的是当前最新的数据。
>   2.  对 HashEntry 链表的遍历操作也是在读取完整个 HashEntry 对象后进行的。虽然在遍历的过程中，其他线程可能会修改链表中的某些 Entry，但遍历操作是基于当前获取到的 HashEntry 链表的快照进行的，所以即使其他线程在遍历过程中修改了某个 Entry，也不会影响当前线程已经获取到的 Entry。
>
>   因此，尽管在遍历过程中其他线程可能会修改 Entry，但由于当前线程已经获取了链表的快照，因此不会影响当前线程对 Entry 的读取操作，保证了读写的线性一致性。

### rehash扩容 

`ConcurrentHashMap` 的扩容只会扩容到原来的两倍。老数组里的数据移动到新的数组时，位置要么不变，要么变为 `index+ oldSize`，参数里的 node 会在扩容之后使用链表**头插法**插入到指定位置。

### 总结

在 JDK 1.7 中，`ConcurrentHashMap` 使用了**<u>分段锁（Segmented Locking）</u>**的机制来实现并发控制。每个 Segment 内部包含一个独立的锁，通常是 `ReentrantLock` 或其变种之一。这些锁用于保护对应 Segment 中的数据结构，如 HashEntry 数组。

具体来说，每个 Segment 通常都会包含一个 `ReentrantLock` 或类似的锁对象，该锁对象用于保护该 Segment 内部的数据结构，比如 HashEntry 数组。当一个线程要访问该 Segment 时，它首先需要获取该 Segment 对应的锁。如果当前锁被其他线程持有，那么线程就会被阻塞，直到获取到锁才能继续执行。

当某个线程要对 `ConcurrentHashMap` 进行操作时，首先会根据 key 计算出该 key 所在的 Segment，然后对该 Segment 进行加锁。这样，即使多个线程同时操作 `ConcurrentHashMap`，只有在同一个 Segment 上的操作会相互竞争锁资源，而其他 Segment 上的操作仍然可以并发进行，提高了并发性能。

需要注意的是，分段锁机制可以减少锁的竞争，但也引入了额外的开销，比如每个 Segment 需要维护一个锁对象。此外，如果多个线程在不同的 Segment 上操作不同的 key，那么它们之间是可以并发执行的，但如果多个线程在同一个 Segment 上操作不同的 key，那么它们仍然会竞争锁资源，可能会导致性能下降。

## jdk1.8

![Java8 ConcurrentHashMap 存储结构（图片来自 javadoop）](/assets/images/java8_concurrenthashmap.png)Java8 ConcurrentHashMap 存储结构

可以发现 Java8 的 ConcurrentHashMap 相对于 Java7 来说变化比较大，不再是之前的 **Segment 数组 + HashEntry 数组 + 链表**，而是 **Node 数组 + 链表 / 红黑树**。当冲突链表达到一定长度时，链表会转换成红黑树。

### initTable初始化

```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //　如果 sizeCtl < 0 ,说明另外的线程执行CAS 成功，正在进行初始化。
        if ((sc = sizeCtl) < 0)
            // 让出 CPU 使用权
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

`Thread.yield()` 方法是一个提示线程调度器，表明当前线程愿意放弃 CPU 使用权，以便让其他具有相同优先级的线程有机会执行。调用 `Thread.yield()` 方法的线程会进入就绪状态，让出 CPU 使用权，但不会进入阻塞状态。

在 `initTable()` 方法中，当 `sizeCtl` 小于 0 时，表示另一个线程已经执行了 CAS 操作，正在进行初始化。在这种情况下，当前线程调用 `Thread.yield()` 方法后，会放弃 CPU 使用权，以便让其他线程有机会执行。这样做是为了减少竞争，让其他线程有机会执行初始化操作，从而更快地完成整个初始化过程。

当另外的线程执行 CAS 操作成功并且初始化成功后，如果此时还有线程在执行 `Thread.yield()` 的线程中，该线程会重新获取 CPU 使用权，并继续执行 `initTable()` 方法中的后续逻辑。此时，由于 `(tab = table) != null`，说明初始化已经完成，当前线程会直接退出 `initTable()` 方法，并返回已经初始化完成的 `table` 数组。

>`Thread.yield()` 方法是由 CPU 决定是否唤醒的，而不是由其他线程主动唤醒的。调用 `Thread.yield()` 方法后，当前线程会将 CPU 使用权让出，进入就绪状态，等待 CPU 调度器重新分配 CPU 时间片。CPU 调度器会根据线程的优先级、系统负载等因素决定下一个要执行的线程，因此 `Thread.yield()` 方法只是一种提示，不能保证其他线程会立即执行。

