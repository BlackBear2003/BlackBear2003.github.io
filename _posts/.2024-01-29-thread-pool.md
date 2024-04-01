---
title: "线程池"
categories:
  - 八股
  - 并发
tags:
  - 线程池
---

# 线程池

## Executor框架

Java的`Executor`是Java并发编程中用于执行异步任务的框架之一。它提供了一种将任务提交与执行解耦的机制，从而允许开发者专注于任务的业务逻辑而无需关心执行的细节。

### **任务(`Runnable` /`Callable`)**

在Java的`ExecutorService`中，`submit()`和`execute()`方法都用于向线程池提交任务（`Runnable`或`Callable`），但它们之间存在一些差别：

1.  **返回值类型**：
    -   `submit()`方法具有返回值，返回一个`Future`对象，可以通过该对象获取任务的执行结果，无论任务是`Runnable`还是`Callable`。
    -   `execute()`方法没有返回值，因此无法获取任务的执行结果。
2.  **异常处理**：
    -   通过`submit()`方法提交的任务，如果任务执行过程中抛出了异常，可以通过`Future`对象的`get()`方法获取到异常，以及执行过程中可能出现的其他异常。
    -   通过`execute()`方法提交的任务，如果任务执行过程中抛出了异常，线程池会捕获并处理该异常，通常会打印堆栈信息，但开发者无法直接捕获到。
3.  **泛型**：
    -   `submit()`方法是泛型方法，可以接受任意类型的`Runnable`或`Callable`任务。
    -   `execute()`方法不是泛型方法，只能接受`Runnable`任务。
4.  **异常处理策略**：
    -   通过`submit()`方法提交的任务，可以使用`Future`对象的`get()`方法指定超时时间，并且可以在超时后取消任务。
    -   通过`execute()`方法提交的任务，无法直接控制任务的执行时间和取消任务。

一般来说，如果需要获取任务的执行结果或者需要更灵活的异常处理，可以使用`submit()`方法；如果只是简单地提交任务并且不需要关心任务的执行结果，可以使用`execute()`方法。

`Executor`框架的核心接口是`Executor`，它定义了一个单一方法`execute`，用于执行传入的`Runnable`任务。`Executor`接口的实现有多种，其中包括：

### **ThreadPoolExecutor**：

>   《阿里巴巴 Java 开发手册》中强制线程池不允许使用 `Executors` 去创建，而是通过 `ThreadPoolExecutor` 构造函数的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险

这是`Executor`接口的一个强大的实现，它管理一个线程池，可以灵活地控制线程的创建、销毁以及任务的执行。可以通过配置线程池的大小、队列策略等来优化线程的使用。

**`ThreadPoolExecutor` 3 个最重要的参数：**

-   **`corePoolSize` :** 任务队列未达到队列容量时，最大可以同时运行的线程数量。
-   **`maximumPoolSize` :** 任务队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数。
-   **`workQueue`:** 新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

`ThreadPoolExecutor`其他常见参数 :

-   **`keepAliveTime`**:线程池中的线程数量大于 `corePoolSize` 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 `keepAliveTime`才会被回收销毁。

-   **`unit`** : `keepAliveTime` 参数的时间单位。

-   **`threadFactory`** :executor 创建新线程的时候会用到。

-   **`handler`** :饱和策略。关于饱和策略下面单独介绍一下。

    >   **`ThreadPoolExecutor` 饱和策略定义:**
    >
    >   如果当前同时运行的线程数量达到最大线程数量并且队列也已经被放满了任务时，`ThreadPoolExecutor` 定义一些策略:
    >
    >   -   `ThreadPoolExecutor.AbortPolicy`：抛出 `RejectedExecutionException`来拒绝新任务的处理。
    >   -   `ThreadPoolExecutor.CallerRunsPolicy`：调用执行自己的线程运行任务，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。
    >   -   `ThreadPoolExecutor.DiscardPolicy`：不处理新任务，直接丢弃掉。
    >   -   `ThreadPoolExecutor.DiscardOldestPolicy`：此策略将丢弃最早的未处理的任务请求。

![线程池各个参数的关系](/assets/images/线程池各个参数之间的关系-JlZBQPFq.png)

### `execute()`执行流程

1.   如果当前运行的线程数小于核心线程数，那么就会新建一个线程来执行任务。

2.   如果当前运行的线程数等于或大于核心线程数，但是小于最大线程数，那么就把该任务放入到任务队列里等待执行。

3.   如果向任务队列投放任务失败（任务队列已经满了），但是当前运行的线程数是小于最大线程数的，就新建一个线程来执行任务。

4.   如果当前运行的线程数已经等同于最大线程数了，新建线程将会使当前运行的线程超出最大线程数，那么当前任务会被拒绝，饱和策略会调用`RejectedExecutionHandler.rejectedExecution()`方法。

![图解线程池实现原理](/assets/images/thread-pool-principle.png)

在 `execute` 方法中，多次调用 `addWorker` 方法。`addWorker` 这个方法主要用来创建新的工作线程。

## 内置线程池的弊端

`Executors` 返回线程池对象的弊端如下(后文会详细介绍到)：

-   **`FixedThreadPool` 和 `SingleThreadExecutor`**：使用的是无界的 `LinkedBlockingQueue`，任务队列最大长度为 `Integer.MAX_VALUE`,可能堆积大量的请求，从而导致 OOM。
-   **`CachedThreadPool`**：使用的是同步队列 `SynchronousQueue`, 允许创建的线程数量为 `Integer.MAX_VALUE` ，可能会创建大量线程，从而导致 OOM。
-   **`ScheduledThreadPool` 和 `SingleThreadScheduledExecutor`** : 使用的无界的延迟阻塞队列`DelayedWorkQueue`，任务队列最大长度为 `Integer.MAX_VALUE`,可能堆积大量的请求，从而导致 OOM。

说白了就是：**使用有界队列，控制线程创建数量。**

### 监测线程池状态

可以通过一些手段来检测线程池的运行状态比如 SpringBoot 中的 Actuator 组件。

除此之外，我们还可以利用 `ThreadPoolExecutor` 的相关 API 做一个简陋的监控。从下图可以看出， `ThreadPoolExecutor`提供了获取线程池当前的线程数和活跃线程数、已经执行完成的任务数、正在排队中的任务数等等。

## 如何配置线程池参数

有一个简单并且适用面比较广的公式：

-   **CPU 密集型任务(N+1)：** 这种任务消耗的主要是 CPU 资源，可以将线程数设置为 N（CPU 核心数）+1。比 CPU 核心数多出来的一个线程是为了**<u>防止线程偶发的缺页中断</u>**，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。
-   **I/O 密集型任务(2N)：** 这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。

## 关闭方法

线程池提供了两个关闭方法：

-   **`shutdown（）`** :关闭线程池，线程池的状态变为 `SHUTDOWN`。线程池不再接受新任务了，但是队列里的任务得执行完毕。
-   **`shutdownNow（）`** :关闭线程池，线程池的状态变为 `STOP`。线程池会终止当前正在运行的任务，停止处理排队的任务并返回正在等待执行的 List。



















