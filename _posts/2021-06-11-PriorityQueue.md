---
title: JAVA并发(8)-并发队列PriotityQueue的分析
layout: post
date: 2021-06-11
tags: 
- java多线程
categories:
- programmer
--- 
*若图片有问题，请点击[此处](https://github.com/uk403/uk403.github.io/tree/master/_posts)查看*

本文讲**PriorityBlockingQueue**(优先阻塞队列)

# 1. 介绍
> 一个**无界**的具有**优先级**的阻塞队列，使用跟**PriorityQueue**相同的顺序规则，默认顺序是自然顺序(**从小到大**)。若传入的对象，**不支持比较**将报错( ClassCastException)。**不允许null**。
> 底层使用的是**基于数组的平衡二叉树堆**实现(它的优先级的实现)。
> 公共方法使用单锁**ReetrantLock**保证线程的安全性。

<!-- more -->

# 1.1 类结构
![PriorityBlockingQueue类图](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210610111542376-1050142774.png "PriorityBlockingQueue类图")
* PriorityBlockingQueue类图

**重要的参数**
```java

   // 数组的默认大小，会自动扩容的
   private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    // 为啥是减8，一些虚拟机会在数组中保留一些header words(头字), 应该学到jvm时，就知道了
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    private transient Object[] queue;

    ...

    // 如果是空的话，优先级队列就使用元素的自然顺序(从小到大)
    private transient Comparator<? super E> comparator;
```

**保证线程安全的措施**
```java

    /**
     * Lock used for all public operations
     */
    private final ReentrantLock lock;

    /**
     * Condition for blocking when empty
     */
    // 为啥没有notFull，因为该队列是无界的
    private final Condition notEmpty;

    /**
     * Spinlock for allocation, acquired via CAS.
     */
    // 为啥用CAS，而不是锁，来控制线程安全在扩容时，后面讲
    private transient volatile int allocationSpinLock;
```

# 2. 源码剖析
我们知道**PriorityBlockingQueue**实现了**BlockingQueue**,这篇[博客](https://www.cnblogs.com/ukyu/p/14859956.html)有提到过**BlockingQueue**可以看一下，它定义了四种方式，对不能立即满足条件的不同的方法，有不同的处理方式。

我们一起去看看下面几种类型的方法的具体实现
* 入队
* 出队

## 2.1 入队
```java
    public boolean add(E e) {
        return offer(e);
    }

    public void put(E e) {
        offer(e); // never need to block
    }

    // 忽略时间
    public boolean offer(E e, long timeout, TimeUnit unit) {
        return offer(e); // never need to block
    }
```
上面几个入队方法都是去调用的**offer(e)**,所以主要来看看这个方法的实现吧

```java
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        final ReentrantLock lock = this.lock;
        lock.lock();
        int n, cap;
        Object[] array;

        // 直到扩容成功或溢出为止
        while ((n = size) >= (cap = (array = queue).length))
            tryGrow(array, cap);
        try {
            Comparator<? super E> cmp = comparator;
            if (cmp == null)

                // 二叉堆的插入算法，在后面讲
                siftUpComparable(n, e, array);
            else
                siftUpUsingComparator(n, e, array, cmp);
            size = n + 1;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
        return true;
    }
```
总体步骤很简单，查看是否需要扩容，然后再插入元素到二叉堆里。我们看看扩容的实现

* **扩容**

> 容量小于64，oldCap + (oldCap + 2); 否则oldCap + (oldCap * 0.5)

```java
      private void tryGrow(Object[] array, int oldCap) {
        lock.unlock(); // must release and then re-acquire main lock
        Object[] newArray = null;

        // allocationSpinLock默认是0，表示此时没有线程在扩容
        if (allocationSpinLock == 0 &&
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                     0, 1)) {
            try {
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) : // grow faster if small
                                       (oldCap >> 1));

                // 检查是否溢出
                if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                    int minCap = oldCap + 1;
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();
                    newCap = MAX_ARRAY_SIZE;
                }
                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];
            } finally {
                allocationSpinLock = 0;
            }
        }

        // 此时，另一个线程正在扩容；让出自己的CPU时间片，下次再去抢占CPU时间片
        if (newArray == null) // back off if another thread is allocating
            Thread.yield();

        // 重新获取锁
        lock.lock();

        // newArray已经被初始化了
        // 如果queue != array, queue已经被改变了；有两种可能：
        // 1. 已经有元素被出队了
        // 2. 已经有元素入队了，此时入队的线程肯定扩容成功了(在没有其他元素出队的情况下)
        if (newArray != null && queue == array) {
            queue = newArray;
            System.arraycopy(array, 0, newArray, 0, oldCap);
        }
    }
```

**为什么扩容时，会解锁，并通过CAS去进行新容量的计算？**
> However, allocation during resizing uses **a simple spinlock** (used only while not holding main lock) in order to **allow takes to operate concurrently with allocation**.This avoids repeated postponement of **waiting consumers** and consequent element build-up.

上面的话，大致意思就是，扩容时使用自旋锁而不是**lock**，为了在扩容时，也可以执行出队操作(上面的代码中，扩容比较耗费时间)。避免让阻塞的消费者被反复阻塞(被唤醒后，不满足条件，又被阻塞，反复)。
**Doug Lea** 👍👍

## 2.2 出队
只讲**poll()**的实现；**take()**与**poll(long timeout, TimeUnit unit)**的实现都差不多

```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

我们先看二叉堆的插入方法**siftUpComparable**，再看**dequeue**。

```java
    // k = size x为插入的元素
    private static <T> void siftUpComparable(int k, T x, Object[] array) {
        Comparable<? super T> key = (Comparable<? super T>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1; // (k -1) / 2
            Object e = array[parent];
            if (key.compareTo((T) e) >= 0)
                break;
            array[k] = e;
            k = parent;
        }
        array[k] = key;
    }
```

这个二叉堆是**小根堆**(任何一个结点的左右子节点的值都大于自己)

![堆初始化](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210610222009138-1100834946.png "堆初始化")
* 堆初始化

此时，我们执行**offer(4)**。按照上面的源码，我们最后得到

![堆offer(4)](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210610223557765-65436280.png "堆offer(4)")
* offer(4)

整个堆插入的思路: 欲插入的元素是否比其父结点小，则与父结点互相交换(**小根堆**)

我们再执行**poll()** -> **dequeue()**

> 返回头部元素，然后重新调整堆元素位置
```java

    /**
     * Mechanics for poll().  Call only while holding lock.
     */
    private E dequeue() {
        int n = size - 1;
        if (n < 0)
            return null;
        else {
            Object[] array = queue;

            // 获取第一个值
            E result = (E) array[0];

            // 保存末尾的值，并置空
            E x = (E) array[n];
            array[n] = null;
            Comparator<? super E> cmp = comparator;
            if (cmp == null)
                // 调整堆的位置
                siftDownComparable(0, x, array, n);
            else
                siftDownUsingComparator(0, x, array, n, cmp);
            size = n;
            return result;
        }
    }
```

> 将 x元素插入到k位置，为了维持二叉堆的平衡，一直降级x直到它小于或等于它的子节点
```java
private static <T> void siftDownComparable(int k, T x, Object[] array,
                                               int n) {
        if (n > 0) {
            Comparable<? super T> key = (Comparable<? super T>)x;
            int half = n >>> 1;    // half =  n / 2      // loop while a non-leaf
            while (k < half) {
                int child = (k << 1) + 1; // assume left child is least  child = k * 2 + 1
                Object c = array[child];
                int right = child + 1;
                if (right < n &&
                    ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                    // 左右节点谁小，谁就当父结点
                    c = array[child = right];
                if (key.compareTo((T) c) <= 0)
                    break;
                array[k] = c;
                k = child;
            }
            array[k] = key;
        }
    }
```

![进入siftDownComparable的状态](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210610230505799-267168552.png "进入siftDownComparable的状态")
* 进入siftDownComparable的状态

执行完毕
![退出siftDownComparable的状态.PNG](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210610233259416-1569625766.png "退出siftDownComparable的状态.PNG")

堆获取头结点后的思路: 将最后一个节点保存起来并置空，将它插入到第一个节点，若不满足就执行下面的流程.
* 比较第一个节点的左右节点是否小于该节点，是的话，就交换左右节点的最小的一个值的位置，周而复始。直到满足**最小堆的性质**为止

# 3. 总结
* **PriorityBlockingQueue**入队后的元素的顺序是按照元素的自然顺序(**Comparator为null时**)进行维护的。
* 使用**ReetrantLock**单锁，保证线程的安全性；在扩容时，通过**CAS**来保证只有一个线程可以成功扩容，**同时扩容时，还可以进行出队操作**
* 顺序通过**二叉堆**维护的，默认是**最小堆**

# 4. 参考
* [深入理解Java PriorityQueue](https://www.cnblogs.com/CarpenterLee/p/5488070.html)  -- 对堆的算法讲的很细致
