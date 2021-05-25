---
title: java多线程(3)-ReentrantLock
layout: post
date: 2021-05-24
tags: 
- java多线程
categories:
- programmer
---  
上节，我们讲了[AQS](https://www.cnblogs.com/ukyu/p/14775600.html)的阻塞与释放实现原理，线程间通信(**Condition**)的原理。这次，我们就讲讲基于**AQS**实现的**ReentrantLock**(重入锁)。

<!-- more -->

# 1. 介绍
![ReentrantLock类图](https://raw.githubusercontent.com/uk403/diagrams/main/blog/2021-05-24-ReentrantLock/ReentrantLock%E7%B1%BB%E5%9B%BE.png)
结合上面的**ReentrantLock**类图，**ReentrantLock**实现了**Lock**接口，它的内部类**Sync**继承自**AQS**，绝大部分使用**AQS的子类**需要**自定义的方法**存在**Sync**中。而**ReentrantLock**有公平与非公平的区别，即'是否先阻塞就先获取资源'，它的主要实现就是**FairSync**与**NonfairSync**，后面会从源码角度看看它们的区别。

# 2. 源码剖析
**Sync**是**ReentrantLock**控制同步的基础。它的子类分为了公平与非公平。使用**AQS**的**state**代表获取锁的数量
```java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        ...
    }
```
我们可以看出内部类**Sync**是一个**抽象类**，继承它的子类(**FairSync**与**NonfairSync**)需要实现抽象方法**lock**。

下面我们先从**非公平锁**的角度来看看获取资源与释放资源的原理
故事就从就两个变量开始:
```java
    // 获取一个非公平的独占锁
    /**
    * public ReentrantLock() {
    *    sync = new ReentrantLock.NonfairSync();
    * }
    */
    private Lock lock = new ReentrantLock();
    // 获取条件变量
    private Condition condition = lock.newCondition();
```



## 2.1 上锁(获取资源)
```java
   lock.lock()
```

```java
    public void lock() {
        sync.lock();
    }
```

```java
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        // 获取资源
        final void lock() {
            // 若此时没有线程获取到资源，直接设置当前线程独占访问资源。
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                // AQS的方法
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            // 实现在父类Sync中
            return nonfairTryAcquire(acquires);
        }
    }
```

**AQS**的**acquire**
```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

```
[上一节](https://www.cnblogs.com/ukyu/p/14775600.html)已经剖析过**AQS**的**acquire**的整个流程了，就差子类如何去实现**tryAcquire**了。

```java
  // Sync实现的非公平的tryAcquire
  final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // 此时若没有线程获取到资源，当前线程就直接占用该资源
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 若当前线程已经占用了该资源，可以再次获取该资源  ->这个行为就是可重入锁的支撑
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
尝试获取资源的过程是非常简单的，这里再贴一下**acquire**的流程图
![acquire流程图](https://raw.githubusercontent.com/uk403/diagrams/main/blog/2021-05-21-AQS/acquire%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## 2.2 释放资源
```java
    lock.unlock();
```

```java
    public void unlock() {
        // AQS的方法
        sync.release(1);
    }
```

**AQS**的**release**
```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

**release**的流程已经剖析过了，接下来看看**tryRelease**的实现
```java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            // 这里可以看出若没有持有锁，就释放资源，就会报错
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

**tryRelease**的实现也很简单，这里再贴一下**release**的流程图
![release流程图](https://raw.githubusercontent.com/uk403/diagrams/main/blog/2021-05-21-AQS/release%E6%B5%81%E7%A8%8B%E5%9B%BE.png)
## 2.3 公平锁与非公平锁的区别
公平锁与非公平锁，即'是否先阻塞就先获取资源', **ReentrantLock**中**公平与否**的控制就在**tryAcquire**中。下面我们看看，公平锁的**tryAcquire**
```java
static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // (2.3.1)
                // sync queue中是否存在前驱结点
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```
区别在**代码(2.3.1)**
**hasQueuedPredecessors**
> 判断当前线程的前面有无其他线程排队；若当前线程在队列头部或者队列为空返回false
```java
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

结合下面的入队代码(**enq**), 我们分析**hasQueuedPredecessors**为**true**的情况:
1. ***h != t*** ,表示此时**queue**不为空; ***(s = h.next) == null***, 表示另一个结点已经运行了**下面的步骤(2)**，还没来得及运行**步骤(3)**。简言之，就是B线程想要获取锁的同时，A线程获取锁失败刚好在入队(*B入队的同时，之前占有的资源的线程，刚好释放资源*)
2. ***h != t*** 且 ***(s = h.next) ！= null***，表示此时至少有一个结点在**sync queue**中；***s.thread != Thread.currentThread()***，这个情况比较复杂，设想一下有这三个结点 **A -> B C**， **A**此时获取到资源，而**B**此时因为获取资源失败正在**sync queue**阻塞，**C**还没有获取资源(还没有执行**tryAcquire**)。

   2.1 时刻一：**A**释放资源成功后(执行**tryRelease**成功)，**B**此时还没有成功获取资源(**C**执行***s = h.next***时，**B**还在**sync queue**中且是**老二**)
 
   2.2 时刻二: **C**此时执行**hasQueuedPredecessors**，***s.thread != Thread.currentThread()***成立，此时**s.thread**表示的是 **B**

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node())) // (1) 第一次初始化
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) { // (2) 设置queue的tail
                    t.next = node; // (3)
                    return t;
                }
            }
        }
    }
```

> Note that 1. because cancellations due to interrupts and timeouts may occur at any time, a true return does not guarantee that some other thread will acquire before the current thread（虚假true）. 2. Likewise, it is possible for another thread to win a race to enqueue after this method has returned false, due to the queue being empty（虚假false）.

这位[大佬](https://blog.csdn.net/anlian523/article/details/106173860)对**hasQueuedPredecessors**进行详细的分析，他文中解释了**虚假true**以及**虚假false**。我这里简单解释一下:
1. **虚假true**, 当两个线程都执行**tryAcquire**，都执行到**hasQueuedPredecessors**，都返回**true**，但是只有一个线程执行 **compareAndSetState(0, acquires)** 成功
2. **虚假false**，当一个**线程A**执行**doAcquireInterruptibly**，发生了中断，还没有清除掉被该结点时；此时，**线程B**执行**hasQueuedPredecessors**时，返回true

# 3. 总结
本文介绍了利用AQS实现的锁**ReetrantLock**
* 讲解了**tryRelease**与**tryAcquire**的实现原理
* 说了说**锁的公平与否**的实现，是否在意**当前线程是否有其他线程排队**
* 分析了一下**hasQueuedPredecessors**的几种情况

# 4. 参考
* [AQS深入理解 hasQueuedPredecessors源码分析 JDK8](https://blog.csdn.net/anlian523/article/details/106173860)