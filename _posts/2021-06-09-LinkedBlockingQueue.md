---
title: JAVA并发(6)-并发队列LinkedBlockingQueue的分析
layout: post
date: 2021-06-09
tags: 
- java多线程
categories:
- programmer
--- 
*若图片有问题，请点击[此处](https://github.com/uk403/uk403.github.io/tree/master/_posts)查看*

本文介绍**LinkedBlockingQueue**，这个队列在线程池中常用到。(~~请结合源码，看本文~~)

# 1. 介绍
> **LinkedBlockingQueue**, **不支持null**，基于**单向链表**的可选**有界**阻塞队列。队列的顺序是**FIFO**。基于**链表的队列**通常比基于**数组的队列**有**更高的吞吐量**, **但在大多数的并发应用中具有更低的可预测性能较差**(这句话，在最后解释一下)
如果不选择队列的容量，默认值是**Integer.MAX_VALUE**，为了**防止队列的过度扩张**.
还实现了**Collection接口**与**Iterator接口**中所有的可选方法。

<!-- more -->

## 1.1 结构
```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable
```

**LinkedBlockingQueue**类图
![LinkedBlockingQueue类图](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210607140858074-167817522.png "LinkedBlockingQueue类图")

**LinkedBlockingQueue**的构造器
```java
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        // head与last指向哨兵节点
        last = head = new Node<E>(null);
    }

    public LinkedBlockingQueue(Collection<? extends E> c)
    ...
```

## 1.2 保证线程安全
**LinkedBlockingQueue**的底层使用**ReetrantLock**保证线程安全，其实就是一个"**消费-生产**"模型，通过本文我们还可以学到**ReetrantLock**的实际使用场景。
```java

    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();

    /**
     * Signals a waiting take. Called only from put/offer (which do not
     * otherwise ordinarily lock takeLock.)
     */
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }

    /**
     * Signals a waiting put. Called only from take/poll.
     */
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }

```
# 2. 源码分析
在讲**LinkedBlockingQueue**前，我们先看看需要它实现的接口**BlockingQueue**
# 2.1 BlockingQueue
实现了**BlockingQueue**的类，必须额外支持在查找元素时，等待队列直到非空为止的操作；在储存元素的时候，要等待队列的空间可用为止。
它的方法有四种形式，处理**不能立即满足但是未来可能满足的操作**的方式各有不同。
1. 直接抛出异常
2. 返回一个特殊值(null 或者 false)
3. 一直等待，直到操作成功
4. 超时设定，超过时间就放弃

![Summary of BlockingQueue methods](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210607150131681-1199577364.png "Summary of BlockingQueue methods")

我们知道了不同方法，不能立即满足的不同的处理方式，这样我们下面就更好理解**LinkedBlockingQueue**的源码了。

下面我们从
* offer(e)
* offer(e, time, unit)
* put(e)
* poll()
去分析一下**LinkedBlockingQueue**

## 2.2 offer(e)
> 添加成功就返回true; 插入值为null,报错；或队列已满，直接返回false，不会等待队列空闲

```java
   public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        // 队列的容量满了，就直接返回了
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return c >= 0;
    }
```
~~代码较简单，就不细讲了。~~

## 2.3 offer(e, time, unit)
> 若队列已满，还没超过设定的时间，就等待，**等待时，会对中断作出反应**；若超过了设定的时间，操作就跟**offer(E e)**一样了
```java
 public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                // 队列已满，超过了特定的时间才会返回false
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }
```

# 2.4 put(e)
> **一直等待直到成功**或者被中断
```java
 public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
           // 防止虚假唤醒
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
```

我没有想到，上面发生**虚假唤醒**的场景(如果知道的同学，请告诉我一下，谢谢了)。
> it is recommended that applications programmers always assume that they can occur and so always wait in a loop. --Condition

**反正使用Condition在循环里等待就对了**

# 2.5 poll()
> 队列为空时，直接返回null，不会await；非阻塞方法

```java
    public E poll() {
        final AtomicInteger count = this.count;
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            if (count.get() > 0) {
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

其余的方法其实都是类似的，直接看上面**BlockingQueue**的四种方式。

最后再讲一个方法**remove(Object o)**, 将会提及一个知识点。

# 2.6 remove
> 删除o， 若成功就返回true，反之.
```java
 public boolean remove(Object o) {
        if (o == null) return false;
        fullyLock();
        try {
            // 算法题，删除某一个链表的结点，可以看一下源码，记录两个结点，一个在前，一个在后。
            for (Node<E> trail = head, p = trail.next;
                 p != null;
                 trail = p, p = p.next) {
                if (o.equals(p.item)) {
                    unlink(p, trail);
                    return true;
                }
            }
            return false;
        } finally {
            fullyUnlock();
        }
    }
```
上面的代码是简单的对链表的操作，我们主要是看**fullyLock()** 与 **fullyUnlock()**的代码

```java

    /**
     * Locks to prevent both puts and takes.
     */
    void fullyLock() {
        putLock.lock();
        takeLock.lock();
    }

    /**
     * Unlocks to allow both puts and takes.
     */
    void fullyUnlock() {
        takeLock.unlock();
        putLock.unlock();
    }
```

**我们都知道解锁顺序应该与获取锁顺序相反，那么是为什么啦**

其实我并不觉得，上面的**fullyUnlock**解锁顺序与获取锁的顺序如果是相同的会出什么问题，也并不会出现**死锁**(如果释放锁与获取锁，中间还存在其他操作就另当别论了)。那它仅仅是为了代码的好看？

假如，我们有下面这段代码(**解锁与获取锁中间有其他操作**)
```java
A.lock();
B.lock();
Foo();
A.unlock();
Bar();
B.unlock();
```
假设**Bar()**是要去重新获取**A锁**的。
时刻一: **线程X**运行到了**Bar()**，**A锁**没有被其他线程获取，此时**线程X**持有**B锁**，要去获取**A锁**
时刻二: **线程Y**运行到代码的最前面，**A.lock()**,获取到了**A锁**，此时**线程Y**持有**A锁**，要去获取**B锁**，此时才会造成死锁。

**解锁顺序与获取锁顺序相反**，为的是
1. 避免上面那种情况造成**死锁**
2. 为了美观

# 3. 总结
**LinkedBlockingQueue**线程安全的队列
* 使用**putLock**、**takeLock**分别对**增加**和**删除**操作保证其**线程安全性**
* 是一个有界(默认值为*Integer.MAX_VALUE*)的底层**基于单向链表**的队列

解释一下，**相对于基于数组的队列，链表队列的可预测性能较差(*less predictable performance in most concurrent applications*)**这句话

我认为是，数组是在初始化会分配"**一块连续的内存**"，而链表是在队列添加元素时动态的分配内存地址的，是不连续的；还有就是它将会处理更多的内存结构，每个元素存在一个链表的节点。
> This means that it has to flush more dirty memory pages between processors when synchronizing.


# 4. 参考
* [Would you explain lock ordering?](https://stackoverflow.com/questions/1951275/would-you-explain-lock-ordering)

* [What does “less predictable performance of LinkedBlockingQueue in concurrent applications” mean?](https://stackoverflow.com/questions/12206840/what-does-less-predictable-performance-of-linkedblockingqueue-in-concurrent-app)
