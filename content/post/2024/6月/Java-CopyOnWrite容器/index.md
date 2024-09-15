---

title: "Java CopyOnWrite容器"
slug: "Java CopyOnWrite容器"
description:
date: "2024-06-13T20:12:48+08:00"
lastmod: "2024-06-13T20:12:48+08:00"
image: cover.png
math:
license:
hidden: false
draft: false
categories: ["Java基础"]
tags: ["CopyOnWrite","Java容器"]

---

> 封面为国漫《诛仙》中的女主 陆雪琪


**CopyOnWrite（缩写COW）** 机制是一种并发编程的常用策略，用于处理数据集合在读写操作同时发生时的一致性问题。它的基本思想是**每次修改操作（如添加或删除元素）都会创建集合的一个新副本**，从而确保读操作始终在不变的集合视图上执行。

<a name="rtAHp"></a>
## COW工作机制

- **读操作**：
   - 所有读操作（如 **get()**、**size()**）在当前不变的数组副本上进行，不需要加锁，因此性能非常高。
- **写操作**：
   - 写操作（如 **add()**、**remove()**）会复制当前数组，进行修改，并将修改后的新数组设置为底层数组。
   - 写操作需要加锁以确保线程安全，但由于写操作是在新数组副本上进行的，因此不会阻塞读操作。

了解了COW机制后，就明白读操作是快而轻，写操作就很重。所以就会有以下的特点

- **内存开销**：由于每次写操作都会创建数组的副本，因此在写操作频繁的场景中会产生较高的内存开销和垃圾回收负担。
- **一致性**：在 **CopyOnWrite** 集合中，迭代器返回的元素可能是过时的，因为迭代器遍历的是快照，不会反映最新的修改。

COW通常适用于读多写少的场景

- **适用场景**：**CopyOnWrite** 机制适用于读操作频繁而写操作较少的场景，例如缓存、白名单和黑名单等配置。


<a name="h0UHS"></a>
## Java中的COW容器
Java中CopyOnWrite机制具体容器有两个

- CopyOnWriteArrayList
- CopyOnWriteArraySet：基于CopyOnWriteArrayList来实现的

所以主要看`CopyOnWriteArrayList`的实现。

属性只有两个，底层就是一个数组，使用ReentrantLock来进行写数组的保护。
```java
/** The lock protecting all mutators */
final transient ReentrantLock lock = new ReentrantLock();

/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;
```


下面是add方法的逻辑，实现比较简单。
```java
/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 写操作加锁保护
    lock.lock();
    try {
        // 获取数据快照
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```


再来看看get的逻辑。更简单了，和平时访问一个数据元素一样，不需要锁的保护。这里需要注意的就是先通过getArray获取了数据快照。
```java

/**
 * {@inheritDoc}
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E get(int index) {
    return get(getArray(), index);
}
final Object[] getArray() {
    return array;
}
```


<a name="Ugd2o"></a>
## 对比Java8和21的实现变化
Java8和Java21中的实现变化不大，主要还是在使用锁的变化上。

- Java8：使用ReentrantLock
- Java21：使用synchronized关键字

这里贴一下Java21中的add代码
```java
/**
 * The lock protecting all mutators.  (We have a mild preference
 * for builtin monitors over ReentrantLock when either will do.)
 */
final transient Object lock = new Object();

/**
 * Appends the specified element to the end of this list.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    synchronized (lock) {
        Object[] es = getArray();
        int len = es.length;
        es = Arrays.copyOf(es, len + 1);
        es[len] = e;
        setArray(es);
        return true;
    }
}
```

可以看到lock属性发生了变化。属性上边的注释
> We have a mild preference for builtin monitors over ReentrantLock when either will do.

也就是作者更倾向于使用synchronized。但是在Java21中，**使用synchronized会造成运行Virtual Thread的Carrier Thread被pinned住的问题**。不知道后面会不会改回来使用ReentrantLock。


## 附录

### 参考
