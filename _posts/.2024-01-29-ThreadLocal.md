---
title: "ThreadLocal"
categories:
  - 八股
  - 并发
tags:
  - ThreadLocal
---

# ThreadLocal

ThreadLocal 是一个线程的本地变量，也就意味着这个变量是线程独有的，是不能与其他线程共享的，这样就可以避免资源竞争带来的多线程的问题。

![img](/assets/images/ThreadLocal-1.png)

ThreadLocal有一个静态内部类ThreadLocalMap，ThreadLocalMap又包含了一个Entry数组，Entry本身是一个弱引用，他的key是指向ThreadLocal的弱引用，Entry具备了保存key value键值对的能力。

每个线程都有自己独立的 `ThreadLocalMap`

![img](/assets/images/ThreadLocal-2.png)

ThreadLocalMap 和HashMap的功能类似，但是实现上却有很大的不同：

1.  HashMap 的数据结构是数组+链表
2.  ThreadLocalMap的数据结构**仅仅是数组**
3.  HashMap 是通过链地址法解决hash 冲突的问题
4.  ThreadLocalMap 是通过**<u>开放地址法来解决hash 冲突</u>**的问题
5.  HashMap 里面的Entry 内部类的引用都是强引用
6.  ThreadLocalMap里面的Entry 内部类中的**<u>key 是弱引用，value 是强引用</u>**

### ThreadLocalMap 采用开放地址法原因

1.  ThreadLocal 中看到一个属性 HASH_INCREMENT = 0x61c88647 ，0x61c88647 是一个神奇的数字，让哈希码能均匀的分布在2的N次方的数组里
2.  ThreadLocal 往往存放的数据量不会特别大（而且key 是弱引用又会被垃圾回收，及时让数据量更小），这个时候开放地址法简单的结构会显得更省空间，同时数组的查询效率也是非常高，加上第一点的保障，冲突概率也低

### 弱引用

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

弱引用的目的是为了防止内存泄露，如果是强引用那么ThreadLocal对象除非线程结束否则始终无法被回收，弱引用则会在下一次GC的时候被回收。

但是这样<u>**还是会存在内存泄露的问题**</u>，假如key和ThreadLocal对象被回收之后，entry中就<u>**存在key为null，但是value有值的entry对象**</u>，但是永远没办法被访问到，同样除非线程结束运行。

### ThreadLocal 内存溢出问题

根据源码，我们可以发现 get 和set 方法都可能触发清理方法expungeStaleEntry()， 所以正常情况下是不会有内存溢出的 但是如果我们没有调用get 和set 的时候就会可能面临着内存溢出

有一种危险是，如果线程是线程池的， 在线程执行完代码的时候并没有结束，只是归还给线程池，这个时候ThreadLocalMap 和里面的元素是不会回收掉的。