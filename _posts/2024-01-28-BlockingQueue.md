---
title: "阻塞队列"
categories:
  - 八股
tags:
  - ArrayBlockingQueue
---

# 阻塞队列

## ArrayBlockingQueue

`ArrayBlockingQueue` 是有界队列，即添加的元素达到上限之后，再次添加就会被阻塞或者抛出异常。

阻塞队列就是典型的**生产者-消费者模型**，它可以做到以下几点:

1.  **当阻塞队列数据为空时，所有的消费者线程都会被阻塞**，等待队列非空。
2.  当生产者往队列里填充数据后，**队列就会通知消费者队列非空**，消费者此时就可以进来消费。
3.  当阻塞队列因为消费者消费过慢或者生产者存放元素过快导致队列填满时**无法容纳新元素时，生产者就会被阻塞，等待队列非满时继续存放元素**。当消费者从队列中消费一个元素之后，队列就会通知生产者队列非满，生产者可以继续填充数据了。

阻塞队列在多线程开发中有着广泛的运用，最常见的例子无非是我们的线程池,从源码中我们就能看出<u>当核心线程无法及时处理任务时，这些任务都会扔到 `workQueue` 中。</u>

```java
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            ThreadFactory threadFactory,
                            RejectedExecutionHandler handler) {// ...}
```

就像 Go 中有大小的 Channel

>   阻塞队列（Blocking Queue）和 Go 中的通道（Channel）在某种程度上实现了相似的功能，但它们在实现方式和语言特性上存在一些不同之处。
>
>   1.  **语言特性**:
>       -   **Java 中的阻塞队列**是基于类和接口的概念，它们是 Java 语言的一部分，提供了一种在多线程环境下安全地传输数据的方式。
>       -   **Go 中的通道**是一种原生的语言特性，被设计成 Goroutine 之间通信的主要机制，是 Go 语言的核心部分。
>   2.  **用法**:
>       -   **阻塞队列**通常在生产者-消费者模型中使用，其中生产者向队列中添加数据，消费者从队列中取出数据。
>       -   **通道**在 Go 中被广泛用于 Goroutine 之间的通信和同步。
>   3.  **内存管理**:
>       -   **通道**在 Go 中的实现由运行时系统负责，包括内存管理和调度。
>       -   **阻塞队列**在 Java 中通常是基于数组或链表实现的，开发者需要管理队列的大小和内存。
>   4.  **错误处理**:
>       -   **Go 中的通道**可以通过 `select` 语句结合超时或默认分支进行非阻塞的数据交换，并且可以通过关闭通道来通知接收者不再有数据。
>       -   **Java 中的阻塞队列**一般需要额外的机制来处理队列已满或为空的情况，比如使用 `offer()` 方法来避免阻塞，或者通过 `poll()` 方法轮询检查队列是否为空。
>
>   在某种程度上，可以将一般的阻塞队列类比为有大小的通道，而无大小的通道则类似于 `SynchronousQueue`。然而，这种类比只是在一些概念上的相似之处，而实际上这些概念在不同语言和框架中的实现可能有所不同。
>
>   1.  **有大小的通道和阻塞队列**:
>       -   有大小的通道和阻塞队列都有一个容量限制，当数据元素达到容量上限时，继续往里面发送数据会被阻塞，直到有空间可用。
>       -   在 Java 中，阻塞队列（如 `ArrayBlockingQueue`）可以有限大小，超出容量时会阻塞生产者或者丢弃数据（具体取决于实现方式和设置）。
>       -   在 Go 中，有缓冲的通道（Buffered Channels）也具有类似的行为，当通道满时，发送操作会被阻塞。
>   2.  **无大小的通道和 `SynchronousQueue`**:
>       -   无大小的通道和 `SynchronousQueue` 都是一种特殊的队列，在发送数据时不需要等待接收方，也不会保留任何数据。
>       -   在 Go 中，无缓冲的通道是阻塞的，直到发送方和接收方都准备好进行通信。发送和接收操作是同步的。
>       -   在 Java 中，`SynchronousQueue` 也是一种特殊的队列，用于在生产者和消费者之间进行直接传递。它不会存储任何元素，每个插入操作必须等待另一个线程调用相应的删除操作，否则插入操作就会一直阻塞。

`put`、`take` 阻塞的存和取方法，非阻塞的入队和出队方法 `offer` 和 `poll`。

### 源码实现

#### put

```java
public void put(E e) throws InterruptedException {
    //确保插入的元素不为null
    checkNotNull(e);
    //加锁
    final ReentrantLock lock = this.lock;
    //这里使用lockInterruptibly()方法而不是lock()方法是为了能够响应中断操作，如果在等待获取锁的过程中被打断则该方法会抛出InterruptedException异常。
    lock.lockInterruptibly();
    try {
            //如果count等数组长度则说明队列已满，当前线程将被挂起放到AQS队列中，等待队列非满时插入（非满条件）。
       //在等待期间，锁会被释放，其他线程可以继续对队列进行操作。
        while (count == items.length)
            notFull.await();
           //如果队列可以存放元素，则调用enqueue将元素入队
        enqueue(e);
    } finally {
        //释放锁
        lock.unlock();
    }
}
private void enqueue(E x) {
   //获取队列底层的数组
    final Object[] items = this.items;
    //将putindex位置的值设置为我们传入的x
    items[putIndex] = x;
    //更新putindex，如果putindex等于数组长度，则更新为0
    if (++putIndex == items.length)
        putIndex = 0;
    //队列长度+1
    count++;
    //通知队列非空，那些因为获取元素而阻塞的线程可以继续工作了
    notEmpty.signal();
}
```

从源码中可以看到入队操作的逻辑就是在数组中追加一个新元素，整体执行步骤为:

1.  获取 `ArrayBlockingQueue` 底层的数组 `items`。
2.  将元素存到 `putIndex` 位置。
3.  更新 `putIndex` 到下一个位置，如果 `putIndex` 等于队列长度，则说明 `putIndex` 已经到达数组末尾了，下一次插入则需要 0 开始。(`ArrayBlockingQueue` 用到了循环队列的思想，即从头到尾循环复用一个数组)
4.  更新 `count` 的值，表示当前队列长度+1。
5.  调用 `notEmpty.signal()` 通知队列非空，消费者可以从队列中获取值了。

#### take

```java
public E take() throws InterruptedException {
       //获取锁
     final ReentrantLock lock = this.lock;
     lock.lockInterruptibly();
     try {
             //如果队列中元素个数为0，则将当前线程打断并存入AQS队列中，等待队列非空时获取并移除元素（非空条件）
         while (count == 0)
             notEmpty.await();
            //如果队列不为空则调用dequeue获取元素
         return dequeue();
     } finally {
          //释放锁
         lock.unlock();
     }
}
private E dequeue() {
  //获取阻塞队列底层的数组
  final Object[] items = this.items;
  @SuppressWarnings("unchecked")
  //从队列中获取takeIndex位置的元素
  E x = (E) items[takeIndex];
  //将takeIndex置空
  items[takeIndex] = null;
  //takeIndex向后挪动，如果等于数组长度则更新为0
  if (++takeIndex == items.length)
      takeIndex = 0;
  //队列长度减1
  count--;
  if (itrs != null)
      itrs.elementDequeued();
  //通知那些被打断的线程当前队列状态非满，可以继续存放元素
  notFull.signal();
  return x;
}

```

-   `notFull`：队列不满的条件。当队列中还有空间可以插入元素时，线程可以等待这个条件。
-   `notEmpty`：队列不空的条件。当队列中有元素可被消费时，线程可以等待这个条件。

### 非阻塞式获取和新增元素

`ArrayBlockingQueue` 非阻塞式获取和新增元素的方法为：

-   `offer(E e)`：将元素插入队列尾部。如果队列已满，则该方法会直接返回 false，不会等待并阻塞线程。
-   `poll()`：获取并移除队列头部的元素，如果队列为空，则该方法会直接返回 null，不会等待并阻塞线程。
-   `add(E e)`：将元素插入队列尾部。如果队列已满则会抛出 `IllegalStateException` 异常，底层基于 `offer(E e)` 方法。
-   `remove()`：移除队列头部的元素，如果队列为空则会抛出 `NoSuchElementException` 异常，底层基于 `poll()`。
-   `peek()`：获取但不移除队列头部的元素，如果队列为空，则该方法会直接返回 null，不会等待并阻塞线程。

```java
public boolean offer(E e) {
        //确保插入的元素不为null
        checkNotNull(e);
        //获取锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
             //队列已满直接返回false
            if (count == items.length)
                return false;
            else {
                //反之将元素入队并直接返回true
                enqueue(e);
                return true;
            }
        } finally {
            //释放锁
            lock.unlock();
        }
    }
public E poll() {
        final ReentrantLock lock = this.lock;
        //上锁
        lock.lock();
        try {
            //如果队列为空直接返回null，反之出队返回元素值
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }

```

如果已经有一个线程在 `put()` 方法中被阻塞，那么其他线程在调用 `offer()` 方法时会被阻塞，直到队列有空间或者另一个线程释放了锁。

这种实现确保了对队列的修改操作是互斥的，即同一时间只能有一个线程对队列进行修改，这是为了防止并发访问导致的数据不一致性。

## LinkedBlockingQueue

`ArrayBlockingQueue` 和 `LinkedBlockingQueue` 是 Java 并发包中常用的两种阻塞队列实现，它们都是线程安全的。不过，不过它们之间也存在下面这些区别：

-   底层实现：`ArrayBlockingQueue` 基于数组实现，而 `LinkedBlockingQueue` 基于链表实现。
-   是否有界：`ArrayBlockingQueue` 是有界队列，必须在创建时指定容量大小。`LinkedBlockingQueue` 创建时可以不指定容量大小，默认是`Integer.MAX_VALUE`，也就是无界的。但也可以指定队列大小，从而成为有界的。
-   **锁是否分离**： `ArrayBlockingQueue`中的锁是没有分离的，即生产和消费用的是同一个锁；`LinkedBlockingQueue`中的锁是分离的，即生产用的是`putLock`，消费是`takeLock`，这样可以防止生产者和消费者线程之间的锁争夺。
-   内存占用：`ArrayBlockingQueue` 需要提前分配数组内存，而 `LinkedBlockingQueue` 则是动态分配链表节点内存。这意味着，`ArrayBlockingQueue` 在创建时就会占用一定的内存空间，且往往申请的内存比实际所用的内存更大，而`LinkedBlockingQueue` 则是根据元素的增加而逐渐占用内存空间。

## SynchronousQueue

`SynchronousQueue` 是 Java 并发包中提供的一种特殊的队列实现，它的特点如下：

1.  **容量为 0**：`SynchronousQueue` 是一个容量为 0 的队列，意味着它不存储任何元素。
2.  **直接传输**：与其他队列不同，`SynchronousQueue` 中的元素不是被插入到队列中进行存储，而是直接传输给等待的线程。
3.  **一对一通信**：每个插入操作（`put()` 方法）都必须等待另一个线程的对应的移除操作（`take()` 方法），反之亦然。因此，`SynchronousQueue` 提供了一种线程之间进行直接交换数据的机制，实现了一对一的线程通信。
4.  **公平性选择**：`SynchronousQueue` 提供了两种模式：公平模式和非公平模式。在公平模式下，等待时间较长的线程会优先获得执行；而在非公平模式下，线程获取执行权的顺序是不确定的。

`SynchronousQueue` 在一些场景中非常有用，比如实现生产者-消费者模式的特定情况，或者在多个线程之间传递任务。

#### `SynchronousQueue` 与 Go 语言中的 Channel 在某些方面有相似之处，但也存在一些不同：

相似之处：

1.  **用途**：`SynchronousQueue` 和 Go 中的 Channel 都用于在多个线程（Go 中的 goroutine）之间进行通信和同步。
2.  **同步机制**：两者都提供了一种同步机制，能够在数据发送和接收之间进行同步，确保发送和接收操作是成对的。

不同之处：

1.  **内存模型**：Go 中的 Channel 是基于 CSP（Communicating Sequential Processes）模型设计的，它是一个 First-class 的数据类型，而且 Channel 的缓冲大小是可选的。而 `SynchronousQueue` 是 Java 并发包提供的一种数据结构，它是一个容量为 0 的队列，只有当发送者和接收者准备好时才能进行数据传输。
2.  **用法和语法**：Go 中的 Channel 是一种内置类型，使用起来非常简洁，支持直接的发送和接收操作，以及 `select` 语句来进行多路复用。而在 Java 中，`SynchronousQueue` 是一个特定的类，使用起来需要显式地创建和操作，需要使用 `put()` 和 `take()` 方法来发送和接收数据。
3.  **错误处理**：Go 中的 Channel 支持对关闭的 Channel 进行操作，可以通过 `range` 关键字或者对返回值进行判断来判断 Channel 是否已经关闭。**而在 Java 中，`SynchronousQueue` 不支持关闭操作，一旦创建就不能关闭。**

## DelayQueue

`DelayQueue` 是 JUC 包(`java.util.concurrent)`为我们提供的延迟队列，用于实现延时任务比如订单下单 15 分钟未支付直接取消。它是 `BlockingQueue` 的一种，**底层是一个基于 `PriorityQueue` 实现的一个无界队列**，是**线程安全**的。

-   `DelayQueue` 是基于数组实现的，所以可以动态扩容，另外它插入元素的顺序并不影响最终的输出顺序。

`DelayQueue` 中存放的元素必须实现 `Delayed` 接口，并且需要重写 `getDelay()`方法（计算是否到期）。

```java
public interface Delayed extends Comparable<Delayed> {
    long getDelay(TimeUnit unit);
}
```

默认情况下, `DelayQueue` 会按照到期时间升序编排任务。只有当元素过期时（`getDelay()`方法返回值小于等于 0），才能从队列中取出。

`DelayQueue` 的 4 个核心成员变量如下：

```java
//可重入锁，实现线程安全的关键
private final transient ReentrantLock lock = new ReentrantLock();
//延迟队列底层存储数据的集合,确保元素按照到期时间升序排列
private final PriorityQueue<E> q = new PriorityQueue<E>();

//指向准备执行优先级最高的线程
private Thread leader = null;
//实现多线程之间等待唤醒的交互
private final Condition available = lock.newCondition();
```

-   `lock` : 我们都知道 `DelayQueue` 存取是线程安全的，所以为了保证存取元素时线程安全，我们就需要在存取时上锁，而 `DelayQueue` 就是基于 `ReentrantLock` 独占锁确保存取操作的线程安全。
-   `q` : 延迟队列要求元素按照到期时间进行升序排列，所以元素添加时势必需要进行优先级排序,所以 `DelayQueue` 底层元素的存取都是通过这个优先队列 `PriorityQueue` 的成员变量 `q` 来管理的。
-   `leader` : 延迟队列的任务只有到期之后才会执行,对于没有到期的任务只有等待,为了确保优先级最高的任务到期后可以即刻被执行,设计者就用 `leader` 来管理延迟任务，只有 `leader` 所指向的线程才具备定时等待任务到期执行的权限，而其他那些优先级低的任务只能无限期等待，直到 `leader` 线程执行完手头的延迟任务后唤醒它。
-   `available` : 上文讲述 `leader` 线程时提到的等待唤醒操作的交互就是通过 `available` 实现的，假如线程 1 尝试在空的 `DelayQueue` 获取任务时，`available` 就会将其放入等待队列中。直到有一个线程添加一个延迟任务后通过 `available` 的 `signal` 方法将其唤醒。













