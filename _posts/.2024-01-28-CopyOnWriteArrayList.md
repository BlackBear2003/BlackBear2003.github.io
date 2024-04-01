---
title: "CopyOnWriteArrayList"
categories:
  - 八股
tags:
  - CopyOnWriteArrayList
---

# CopyOnWriteArrayList

为了将读操作性能发挥到极致，`CopyOnWriteArrayList` 中的**读取操作是完全无需加锁的。**更加厉害的是，**写入操作也不会阻塞读取操作**，只有写写才会互斥。这样一来，读操作的性能就可以大幅度提升。

写时复制机制非常适合读多写少的并发场景，能够极大地提高系统的并发性能。

```java
// 插入元素到 CopyOnWriteArrayList 的尾部
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取原来的数组
        Object[] elements = getArray();
        // 原来数组的长度
        int len = elements.length;
        // 创建一个长度+1的新数组，并将原来数组的元素复制给新数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 元素放在新数组末尾
        newElements[len] = e;
        // array指向新数组
        setArray(newElements);
        return true;
    } finally {
        // 解锁
        lock.unlock();
    }
}

```

每次写操作都需要通过 `Arrays.copyOf` 复制底层数组，时间复杂度是 O(n) 的，且会占用额外的内存空间。因此，`CopyOnWriteArrayList` 适用于读多写少的场景，在写操作不频繁且内存资源充足的情况下，可以提升系统的性能表现。

`add`方法内部用到了 `ReentrantLock` 加锁，保证了同步，避免了多线程写的时候会复制出多个副本出来。锁被修饰保证了锁的内存地址肯定不会被修改，并且，释放锁的逻辑放在 `finally` 中，可以保证锁能被释放。



#### `get`方法是 弱一致性 的，在某些情况下可能读到旧的元素值。

