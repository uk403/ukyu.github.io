---
title: java多线程(4)-ReentrantReadWriteLock的探索
layout: post
date: 2021-05-30
tags: 
- java多线程
categories:
- programmer
---  
# 1. 介绍
本文我们继续探究使用**AQS**的子类**ReentrantReadWriteLock**(读写锁)。老规矩，先贴一下类图
![ReentrantReadWriteLock结构图](https://raw.githubusercontent.com/uk403/diagrams/main/blog/2021-05-30-ReentrantReadWriteLock/ReentrantReadWriteLock%E7%B1%BB%E5%9B%BE.jpg)
**ReentrantReadWriteLock**这个类包含**读锁和写锁**，这两种锁都存在**是否公平**的概念，这个后面会细讲。
> 此类跟[ReentrantLock](https://www.cnblogs.com/ukyu/p/14802757.html "ReentrantLock")类似，有以下几种性质:
> * **可选的公平性政策**
> * **重入**，读锁和写锁同一个线程可以**重复获取**。**写锁可以获取读锁**，反之不能
> * **锁的降级**，重入还可以通过获取写锁，然后获取到读锁，通过释放写锁的方式，从而写锁降级为读锁。 然而，从读锁升级到写锁是不可能的。
> * 获取读写锁期间，支持**不可中断**

<!-- more -->

# 2. 源码剖析
先讲几个必要的知识点，然后我们再对**写锁的获取与释放**，**读锁的获取与释放**进行讲解，中间穿插着讲**公平与非公平**的实现。

**知识点一**:
内部类**Sync**中，将**AQS**中的**state**(*private volatile int state*;长度是32位)逻辑分成了两份，**高16位**代表**读锁**持有的count，**低16位**代表写锁持有的count

**Sync**
```java
    ...
        /*
         * Read vs write count extraction constants and functions.
         * Lock state is logically divided into two unsigned shorts:
         * The lower one representing the exclusive (writer) lock hold count,
         * and the upper the shared (reader) hold count.
         */
        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
   ...
```
> This lock supports a maximum of 65535 **recursive write locks** and 65535 **read locks**. Attempts to exceed these limits result in Error throws from locking methods.（读锁与写锁都最大支持65535个）

**知识点二**：
**HoldCounter**的作用，一个计数器记录每个线程持有的读锁**count**。使用**ThreadLocal**维护。缓存在**cachedHoldCounter**
```java
        static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }
```

使用**ThreadLocal**维护 👇
```java
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
```

维护最后一个使用**HoldCounter**的线程。简言之就是，假如**A线程**持有读锁，**A线程**重入获取读锁，在它之后没有其他线程获取读锁，那么当获取**HoldCounter**时，可以直接将**cachedHoldCounter**赋值给该线程，就不用从**ThreadLocal**中去查询了(***ThreadLocal**内部维持一个 **Map** ，想获取当前线程的值就需要去遍历查询*)，这样做可以节约时间。

```java
        private transient HoldCounter cachedHoldCounter;
```

当前线程持有的重入读锁count，当某个线程持有的count降至0，将被删除。
```java
        private transient ThreadLocalHoldCounter readHolds;
```

初始化在构造函数或**readObject**中
```java
        Sync() {
            readHolds = new ThreadLocalHoldCounter();
            setState(getState()); // ensures visibility of readHolds
        }
```

```java
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            readHolds = new ThreadLocalHoldCounter();
            setState(0); // reset to unlocked state
        }
```

**知识点三**:
是否互斥 :
|  |  读操作 |  写操作 |
| ------------ | ------------ | ------------ |
|  读操作 |  否 |  是 |
| 写操作  |  是 |  是 |

只有读读不互斥，其余都互斥
**获取到了写锁，也有资格获取读锁，反之不行.**

**知识点四**
**ReentranReadWriteLock**，实现了**ReadWriteLock**接口。
```java
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}
```
**ReadWriteLock**维护着**写锁**和**读锁**。**写锁**是排他的，而**读锁**可以同时由多个线程持有。与**互斥锁**相比，**读写锁**的粒度更细

有了上面的知识，等会理解下面的源码就更容易了,故事从下面几个变量开始~

```java
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock readLock = rwl.readLock();
    private final Lock writeLock = rwl.writeLock();
```
**ReentrantReadWriteLock**的构造器，默认是**非公平模式**
```java

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * default (nonfair) ordering properties.
     */
    public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }
```
## 2.1 写锁
写锁，排他锁；一个线程获取了写锁，其他线程只能等待
### 2.1.1 写锁的获取
```java
    writeLock.lock();
```

**java.util.concurrent.locks.ReentrantReadWriteLock.WriteLock#lock**
```java
public void lock() {
            sync.acquire(1);
        }
```
又来到了**AQS**的**acquire**方法

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
我们看**ReentrantReadWriteLock**是如何实现**tryAcquire**的

```java
 protected final boolean tryAcquire(int acquires) {

            /*
            * 对下面的条件做一个总述：
            * 1. 当读锁或者写锁的count不为零时，同时拥有者不是当前线程，返回false
            * 2. 当count达到饱和(超过最大的限制65535)，返回false
            * 3. 如果上面的情况都不是，该线程有资格去获取锁，如果它是可重入获取锁或队列策略允许它。那么就更新state并且设置锁的拥有者
            */
            // 结合总述看下面的代码，很容易就看懂了

            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);

            // 存在读锁或写锁
            if (c != 0) {
                // 情况分析: 1. w = 0, 表示已经获取了读锁(不管是自己还是其他线程)，直接返回false
                // 2. 写锁不为零，且当前锁的拥有者不是当前线程，返回false
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }

            // 不存在读锁或写锁被占用的情况
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

上面代码中的**writerShouldBlock**方法就是**tryAcquire**控制公平与否的关键，我们分别看看公平与非公平是如何实现的

#### 2.1.1.1 写锁的获取(非公平)
默认情况下是**非公平的**
```java
 static final class NonfairSync extends Sync {
        ...
        // 直接返回false
        final boolean writerShouldBlock() {
            return false;
        }
        ...
```

#### 2.1.1.2 写锁的获取(公平)
```java
    static final class FairSync extends Sync {
        ...
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        ...
```
判断是否该线程前面还有其他线程的结点，[上一节](https://www.cnblogs.com/ukyu/p/14802757.html)有讲到过。

这里还贴一下，整个**acquire**的流程图
![acquire的流程图](https://raw.githubusercontent.com/uk403/diagrams/main/blog/2021-05-21-AQS/acquire%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

### 2.1.2 写锁的释放
**下面的这段代码，记得一定放在finally中**
```java
    writeLock.unlock();
```

```java
        public void unlock() {
            sync.release(1);
        }
```

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
又看到了熟悉的面孔，但我们主要看的还是**tryRelease**, 👇
```java
        protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int nextc = getState() - releases;
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
```
很简单，就贴一下**release**的流程图
![release流程图](https://raw.githubusercontent.com/uk403/diagrams/main/blog/2021-05-21-AQS/release%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## 2.2 读锁
读锁与读锁并不互斥，可以存在多个持有读锁的线程📕
### 2.2.1 读锁的获取

```java
        readLock.lock();
```
```java
        public void lock() {
            sync.acquireShared(1);
        }
```

```java
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
~~之前的文章还没有讲解过**tryAcquireShared**在子类如何实现的~~。看看如何实现的 👇

```java
  protected final int tryAcquireShared(int unused) {

            /*
             * 对下面的条件做一个总述:
             * 1. 如果写锁被其他线程持有，失败
             * 2. 否则，如果此线程有资格去获取写锁，首先查询是否需要阻塞(readerShouldBlock).如果不需要，就通过CAS更新state。
             *
             * 3. 如果上面两步都失败了，可能是当前线程没有资格(需要被阻塞)，或CAS失败，或者count数量饱和了，然后执行fullTryAcquireShared进行重试
             */
            Thread current = Thread.currentThread();
            int c = getState();
            // 这里可以看出，若该线程持有写锁，同样也可以去获取读锁
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            int r = sharedCount(c);

            // readerShouldBlock表示此线程是否被阻塞(后面细谈)
            // 多个线程执行compareAndSetState(c, c + SHARED_UNIT), 只有一个线程可以执行成功，其余线程去执行fullTryAcquireShared
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    // firstReader是第一个获取到读锁的线程
                    // firstReaderHoldCount是firstReader持有的count
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    // 重入
                    firstReaderHoldCount++;
                } else {
                    // sync queue还存在其他线程持有读锁，且该线程不是第一个持有读锁的线程
                    // 结合上面的知识点二，理解下面的逻辑
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            // 到达这里的线程，1. 没有资格获取(需要阻塞) 2. CAS失败 3. 读锁count饱和
            return fullTryAcquireShared(current);
        }
```

> Full version of acquire for reads, that handles CAS misses and reentrant reads not dealt with in tryAcquireShared.
```java

        final int fullTryAcquireShared(Thread current) {

             // 主要逻辑跟上面的tryAcquireShared类似。tryAcquireShared的逻辑只有一个线程会成功CAS，
             // 其余的线程都进入fullTryAcquireShared，进行重试(代码不那么复杂)
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) { // 需要被阻塞
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        // 此时, 如果该线程前面已经有线程获取了读锁，且当前线程持有的读锁count为0，从readHolds除去。
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        // 这里表示该线程还是一个new reader, 还没有持有读锁
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // 这里只有一个线程会成功，若CAS失败，一直重试直到成功，或者读锁count饱和，或者需要被阻塞为止
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }

```

我们上面看到了**readerShouldBlock**，这个方法是控制**读锁获取公平与否**，下面我们分别看看在**非公平与公平**模式下的实现 👇

#### 2.2.1.1 读锁的获取(非公平)
```java
    static final class NonfairSync extends Sync {
        ...
        final boolean readerShouldBlock() {
            /* As a heuristic to avoid indefinite writer starvation,
             * block if the thread that momentarily appears to be head
             * of queue, if one exists, is a waiting writer.  This is
             * only a probabilistic effect since a new reader will not
             * block if there is a waiting writer behind other enabled
             * readers that have not yet drained from the queue.
             */
            return apparentlyFirstQueuedIsExclusive();
        }
        ...
    }
```

> 判断sync queue的head的后继结点是否是写锁(独占模式)
```java
    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```

上面的方法是，获取读锁时，避免导致**写锁饥饿(indefinite writer starvation)**的一个措施，下面我们对它进行详细的解释

![写锁饥饿](https://raw.githubusercontent.com/uk403/diagrams/main/blog/2021-05-30-ReentrantReadWriteLock/%E5%86%99%E9%94%81%E9%A5%A5%E9%A5%BF.png)

结合上面的图片，我们设想有一个这样的情况，**写锁没有被获取**，线程A获取到了读锁，此时另一个**线程X**想要获取**写锁**，但是**写锁与读锁互斥**，所以此时将**线程X**代表的**node**添加到**sync queue**中，等待**读锁**被释放，才有资格去获取**写锁**。

上面的情况 + 不存在**判断sync queue的head的后继结点是否是写锁(*apparentlyFirstQueuedIsExclusive*)**的方法时，我们看看会出什么问题

时刻一: **线程B**、**线程C**，**线程D**是新建线程想要去获取读锁(new reader)，此时因为不存在**写锁被获取**，所以**线程B**、**线程C**，**线程D**都会在**fullTryAcquireShared**中不断重试，最终都获得**读锁**

时刻二: **线程A**释放，会执行**unparkSuccessor**，此时**线程X**被唤醒，但是执行到**tryAcquire**，又检测到**读锁被持有(不管是自己还是是其他线程)**，**线程X**又被阻塞。**线程B**释放，还是会出现这种情况，只有等到最后一个**读锁被释放**，**线程X**才能获取到**写锁**。但是想想如果后面一连串的**读锁**，**线程X**不是早就被*'饿死了'*

**apparentlyFirstQueuedIsExclusive**，就可以防止这种**'写锁饥饿'**的情况发生。**线程B**、**线程C**，**线程D**只有被阻塞，等待**线程X**获取到写锁，才有机会获取**读锁**。

#### 2.2.1.2 读锁的获取(公平)
```java

    /**
     * Fair version of Sync
     */
    static final class FairSync extends Sync {
       ...

        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
       ...
    }
```

这里关于读锁的获取(公平与非公平)分析完了，贴一张整个**acquireShared**的流程图
![acquireShared流程图](https://raw.githubusercontent.com/uk403/diagrams/main/blog/2021-05-21-AQS/acquireShared%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

### 2.2.2 读锁的释放
```java
        readLock.unlock();
```

```java
        public void unlock() {
            sync.releaseShared(1);
        }
```

**AQS**
```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            // 唤醒head后驱结点
            doReleaseShared();
            return true;
        }
        return false;
    }
```

```java
 // unused = 1
 protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                // 获取读锁时，会执行 compareAndSetState(c, c + SHARED_UNIT)，这里就是减SHARED_UNIT
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // 释放读锁，对需要读锁的线程没有影响(读读不互斥)
                    // 但是，这里如果返回true,可以唤醒被阻塞的想要持有写锁的线程
                    return nextc == 0;
            }
        }
```

# 3. 总结
1. **ReentrantReadWriteLock**中的防止'写锁饥饿'的操作，值得一看
2. 将**AQS**中的**state**(*private volatile int state;*),逻辑分为**高16位**(代表读锁的state)，**低16位**(代表写锁的state)，是一个值得学习的办法
3. 使用**ThreadLocal**维护每一个现成的读锁的重入数，使用**cachedHoldCounter**维护最后一个使用**HoldCounter**的读锁线程，节省在**ThreadLocal**中的查询

使用**读写锁**的情况，应该取决于与修改相比，读取数据的频率，读取和写入操作持续的时间。
例如:
* 某个集合不经常修改，但是对其元素搜索很频繁，使用**读写锁**就是最佳选择。(简言之，就是**读多写少**)
* 现在也是**读多写少**，但是读操作时间很短，只有一小段代码，而读写锁比互斥锁更加复杂，开销可能大于互斥锁，这种情况使用**读写锁**可能不合适。此时就要通过性能分析，判断使用读写锁在系统中是否合适。

# 4. 参考
* [ReentrantReadWriteLock的readerShouldBlock与apparentlyFirstQueuedIsExclusive 深入理解读锁的非公平实现](https://blog.csdn.net/anlian523/article/details/106964711)
