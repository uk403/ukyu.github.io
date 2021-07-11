---
title: JAVA并发(9)-ThreadPoolExecutor的讲解
layout: post
date: 2021-07-09
tags: 
- 多线程基础
categories:
- programmer
---
*若图片有问题，请点击[此处](https://github.com/uk403/uk403.github.io/tree/master/_posts)查看*


很久前(~~2020-10-23~~)，就有想法学习线程池并输出博客，但是写着写着感觉看不懂了，就不了了之了。现在重拾起，重新写一下(学习一下)。

线程池的优点也是老生常谈的东西了
1. 减少线程创建的开销(**任务数大于线程数时**)
2. 统一管理一系列的线程(资源)

<!-- more -->

---

在讲**ThreadPoolExecutor**前，我们先看看它的父类都有些啥。

![Executor的继承关系](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210628234056891-99588073.png "Executor的继承关系")

**Executor**，执行提交的**Runnable**任务的**对象**，将任务提交与何时执行分离开。
**execute**方法是**Executor**接口的唯一方法。

```java
    // 任务会在未来某时执行，可能执行在一个新线程中、线程池或调用该任务的线程中。
    void execute(Runnable command);
```

![ExecutorService的继承关系](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210628234032788-1764340529.png "ExecutorService的继承关系")

**ExecutorService**是一个**Executor**，提供了管理终止的方法和返回**Future**来跟踪异步任务的方法(**sumbit**)。
终止的两个方法

* shutdown(), 正在执行的任务继续执行，不接受新任务
* shutdownNow(), 正在执行的任务也要被终止

![AbstractExecutorService的继承关系](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210628233938136-468918050.png "AbstractExecutorService的继承关系")

**AbstractExecutorService**，实现了**ExecutorService**的**sumbit**、**invokeAny**,**invokeAll**

![ThreadPoolExecutor的继承关系](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210628233551592-54533128.png "ThreadPoolExecutor的继承关系")


# 介绍

🐱‍🏍**线程池主要元素**

![线程池主要元素](https://img2020.cnblogs.com/blog/2192575/202107/2192575-20210707232050622-306691488.png "线程池主要元素图一")

![线程池主要元素](https://img2020.cnblogs.com/blog/2192575/202107/2192575-20210707232159448-537576060.png "线程池主要元素图二")
## 底层变量
### ctl
我们讲讲先ctl(The main pool control state), 其包含两个信息

1. 线程池的状态(最高三位)
2. 线程池的workerCount，有效的线程数

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```

🐱‍🏍线程池的状态转化图

![线程池的状态转化图](https://img2020.cnblogs.com/blog/2192575/202107/2192575-20210708223307715-1805524998.png "线程池的状态转化图")

看一下每个状态的含义

1. RUNNING, 接受新的任务并且处理阻塞队列的任务
2. SHUTDOWN, 拒绝新任务，但是**处理阻塞队列的任务**
3. STOP, 拒绝新任务，并且抛弃阻塞队列中的任务，还要**中断正在运行的任务**
4. TIDYING,**所有任务执行完**(包括阻塞队列中的任务)后, 当前线程池活动线程为0, 将要调用terminated方法
5. TERMINATED, 终止状态。**调用terminated方法后的状态**

### workers
工作线程都添加到这个集合中。可以想象成一个集中管理的平台，可以通过**workers**获取活跃的线程数，中断所有线程等操作。

```java
    private final HashSet<Worker> workers = new HashSet<Worker>();
```

## 可修改变量
### 构造器中的参数
```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime, // 最大等待任务的时间，超过则终止超过corePoolSize的线程
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue, // 阻塞队列
                              ThreadFactory threadFactory, // executor使用threadFactory创建一个线程
                              RejectedExecutionHandler handler) // 拒绝策略
```

corePoolSize、maximumPoolSize，workQueue三者的关系:

* 当线程数小于**corePoolSize**，任务进入，即使有其他线程空闲，也会创建一个新的线程
* 大于**corePoolSize**且小于**maximumPoolSize**，**workQueue**未满，将任务加入到**workQueue**中;只有**workQueue**满了，才会新建一个线程
* 若**workQueue**已满，且任务大于**maximumPoolSize**，将会采取**拒绝策略**(handler)

拒绝策略:
1. **AbortPolicy**, 直接抛出RejectedExecutionException
2. **CallerRunsPolicy**, 使用调用者所在线程来执行任务
3. **DiscardPolicy**, 默默丢弃
4. **DiscardOldestPolicy**, 丢弃头部的一个任务,重试

### allowCoreThreadTimeOut
控制空闲时，core threads是否被清除。

# 探索源码
最重要的方法就是**execute**
> 提交的任务将在未来某个时候执行

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */

        // 获取workCount与runState
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

根据上面的注释，我们将execute分为三个部分来讲解

* 当正在运行的线程数小于corePoolSize
* 当大于corePoolSize时，需要入队
* 队列已满

## 当正在运行的线程数小于corePoolSize

```java
        // execute第一部分代码
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
```

**addWorker**, 创建工作线程。
当然它不会直接就添加一个新的工作线程，会检测**runState**与**workCount**，来避免不必要的新增。检查没问题的话，新建线程，将其加入到**wokers**，并将线程启动。

```java
   // firstTask，当线程启动时，第一个任务
   // core，为true就是corePoolSize作为边界，反之就是maximumPoolSize
   private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

上面的代码很长，我们将它分为两部分

```java
// addWorker()第一部分代码
// 这部分主要是通过CAS增加workerCount
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 下面的条件我们将它转化成👇
            // if (rs >= SHUTDOWN &&
            //      (rs != SHUTDOWN ||
            //        firstTask != null ||
            //         workQueue.isEmpty()))
            // 结合线程池状态分析！

            // 情况1. 当前的线程池状态为STOP、TIDYING，TERMINATED
            // 情况2. 当前线程池的状态为SHUTDOWN且firstTask不为空，只有RUNNING状态才可以接受新任务
            // 情况3. 当前线程池的状态为SHUTDOWN且firstTask为空且队列为空。
            // 这几种情况，没有必要新建worker(线程)。
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // CAS增加workerCount成功，继续第二部分操作
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // 这里的线程池状态被改变了，继续外部循环，再次检查线程池状态
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
```

经过上面的代码，我们成功通过CAS使**workerCount + 1**，下面我们就会新建**worker**并添加到**workers**中，并启动通过**threadFactory**创建的线程。
```java
// addWorker()第二部分代码

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    // 第一种情况，rs为RUNNING
                    // 第二种情况是rs为SHUTDOWN，firstTask为null， 但是workQueue(阻塞队列)不为null，创建线程进行处理
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 这里是该线程已经被启动了，我觉得的原因是threadFactory创建了两个相同的thread，不知道还有其他原因没。
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // 上面的线程创建可能失败，或者线程工厂返回null
            // 或者线程启动时，抛出OutOfMemoryError
            if (! workerStarted)
                // 回滚状态
                addWorkerFailed(w);
        }
        return workerStarted;
```

看完了addWorker的步骤，代码中有个Worker类，**看似是线程但又不完全是线程**，我们去看看它的结构。

**Worker**
这个类的主要作用是，维护**线程运行任务的中断控制状态**和**记录每个线程完成的任务数**。

整体结构
```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable{

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** 每个线程的任务完成数 */
        volatile long completedTasks;

        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            // 在创建线程时，将任务传入到threadFactory中
            this.thread = getThreadFactory().newThread(this);
        }

        public void run() {
            // 将运行委托给外部方法runWorker，下面会详见。

            // 这里是运行任务的核心代码✨
            runWorker(this);
        }

        // 实现AQS的独占模式的方法，该锁不能重入。
        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }
        // 线程运行之后才可以被中断
        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {}
            }
        }
}
```

我们来看看**runWorker**的实现。这个类主要的工作就是，**不停地**从**阻塞队列**中获取任务并执行，若**firstTask**不为空，就直接执行它。
```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // getTask控制阻塞等待任务或者是否超时就清除空闲的线程
            // getTask非常之重要✨，后面会讲到
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                // 第二种情况，重新检测线程池状态，因为此时可能其他线程会调用shutdownNow
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 执行前
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行firstTask的run方法
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                    // 执行后
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 处理worker退出
            processWorkerExit(w, completedAbruptly);
        }
    }

```
🐱‍🏍 启动一个线程，大致执行的方法流程
![具体执行的方法](https://img2020.cnblogs.com/blog/2192575/202107/2192575-20210708224838162-132814909.png "具体执行的方法")

**getTask**，我们来看看它是怎样**阻塞或定时等待任务的**。

Performs blocking or timed wait for a task, depending on current configuration settings, or returns null if this worker must exit because of any of:
1. There are **more than maximumPoolSize workers** (due to a call to setMaximumPoolSize).
2. The pool is **stopped**.
3. The pool is **shutdown** and the **queue is empty.**
4. This worker timed out waiting for a task, and timed-out workers are subject to termination (that is, allowCoreThreadTimeOut || workerCount > corePoolSize) both before and after the timed wait, and if the queue is non-empty, this worker is not the last thread in the pool. 🐱‍🏍(**超时等待任务的worker，在定时等待前后都会被终止（情况有，allowCoreThreadTimeOut || wc > corePoolSize**)

Returns:
task, or null if the worker must exit, in which case workerCount is decremented(worker退出时，workerCount会减一)

```java
  private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 检测是否有必要返回新任务，注意每个状态的含义就明白了
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // 检测worker是否需要被淘汰
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            // 下面的代码结合上面的timed变量，超时后，当大于corePoolSize时，返回null
            // 或者当allowCoreThreadTimeOut = true时，超时后，返回null
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                // 这里r为null的话，只能是timed = true的情况；take()，一直会阻塞直到有任务返回
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

我们来看看当**getTask**返回**null**时，线程池是如何处理**worker**退出的

根据**runWorker**的代码，getTask为null，循环体正常退出，此时**completedAbruptly = false;**

**processWorkerExit**
```java
  private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 1. 有异常退出的话， workerCount将会减一
        // 2. 正常退出的话，因为在getTask中已经减一，所以这里不用理会
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        // 将worker完成的任务数加到completedTaskCount
        // 从workers中移除当前worker
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        // 检测线程池是否有资格设置状态为TERMINATED
        tryTerminate();

        int c = ctl.get();
        // 此时的状态是RUNNING或SHUTDOWN
        if (runStateLessThan(c, STOP)) {
            // 1. 非正常退出的，addWorker()
            // 2. 正常退出的, workerCount小于最小的线程数，就addWorker()
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```

**getTask**是保证存在的线程不被销毁的核心，**getTask**则利用**阻塞队列**的**take**方法，一直阻塞直到获取到任务为止。


## 当大于corePoolSize时，需要入队
```java
// execute第二部分代码
        // 线程池状态是RUNNING(只有RUNNING才可以接受新任务)
        // 此时，workerCount >= corePoolSize, 将任务入队
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 此时线程池可能被shutdown了。
            // 需要清除刚添加的任务，若任务还没有被执行，就可以让它不被执行
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 若此时没有worker，新建一个worker去处理队列中的任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
```

## 队列已满
```java
       // execute第三部分代码
       // addWorker第二个参数false表明，以maximumPoolSize为界限
        else if (!addWorker(command, false))
            // workerCount > maximumPoolSize 就对任务执行拒绝策略
            reject(command);
```

我们就讲完了执行方法**execute()**，有兴趣的同学可以去看看关闭方法**shutdown()**以及**shutdownNow()**，看看他们的区别。当然也可以去研究一下其他方法的源码。

# 探究一些小问题
1. runWorker为啥这样抛错

```java
    try {
        task.run();
    } catch (RuntimeException x) {
        thrown = x; throw x;
    } catch (Error x) {
        thrown = x; throw x;
    } catch (Throwable x) {
        thrown = x; throw new Error(x);
    } finally {
    ...
    }
```
>We separately handle RuntimeException, Error (both of which the specs guarantee that we trap) and arbitrary Throwables. Because we cannot rethrow Throwables within Runnable.run, we wrap them within Errors on the way out (to the thread's UncaughtExceptionHandler). Any thrown exception also conservatively causes thread to die.

大致意思就是，分别处理RuntimeException、Error和任何的Throwable。因为不能在 Runnable.run 中重新抛出 Throwables，所以将它们包装在 Errors中（到线程的 **UncaughtExceptionHandler**).
在**Runnable.run**不能抛出Throwables的原因是，Runnable中的run并没有定义抛出任何异常，继承它的子类，抛错的范围不能超过父类
**UncaughtExceptionHandler**可以处理“逃逸的异常”，可以去了解一下。

2. 创建线程池最好手动创建，参数根据系统自定义
![自定义线程数的依据](https://img2020.cnblogs.com/blog/2192575/202107/2192575-20210707101334815-1590751209.png "自定义线程数的依据")
图中的设置线程数的策略只是初步设置，下一篇我们去研究具体的**线程数调优**

3. 为什么创建线程开销大
启动一个线程时，将涉及大量的工作
> * 必须为线程堆栈分配和初始化一大块内存。
> * 需要创建/注册native thread在host OS中
> * 需要创建、初始化描述符并将其添加到 JVM 内部数据结构中。

虽然启动一个线程的时间不长，耗费的资源也不大，但有个东西叫"积少成多"。就像
**Doug Lea**写的源码一样，有些地方的细节优化，看似没必要，但是请求一多起来，那些细节就是"点睛之笔"了。
当我们有大量需要线程时且每个任务都是独立的，尽量考虑使用**线程池**

# 总结
线程池的总体流程图
![线程池的总体流程](https://img2020.cnblogs.com/blog/2192575/202107/2192575-20210709144924263-38446742.png "线程池的总体流程")

线程池新建线程，如何保证可以不断地获取任务，就是通过阻塞队列(BlockingQueue)的take方法，阻塞自己直到有任务才返回。

本篇博客也到这里就结束了，学习线程池以及输出博客，中间也拖了很久，最后送给大家以及自己最近看到的一句话
> 往往最难的事和自己最应该做的事是同一件事

# 参考
* [Why is creating a Thread said to be expensive?](https://stackoverflow.com/questions/5483047/why-is-creating-a-thread-said-to-be-expensive "Why is creating a Thread said to be expensive?") 创建线程为何开销较大
* [Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html "Java线程池实现原理及其在美团业务中的实践") 讲了线程池的原理以及在美团的一些实际运用
* [10问10答：你真的了解线程池吗？](https://mp.weixin.qq.com/s/axWymUaYaARtvsYqvfyTtw "10问10答：你真的了解线程池吗？") 一些使用线程池的建议
