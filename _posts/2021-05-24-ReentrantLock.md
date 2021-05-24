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

# 3. 总结
本文介绍了利用AQS实现的锁**ReetrantLock**
* 讲解了**tryRelease**与**tryAcquire**的实现原理
* 说了说**锁的公平与否**的实现，是否在意**当前线程是否有其他线程排队**
