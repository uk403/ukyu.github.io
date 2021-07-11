---
title: JAVA并发(7)-并发队列ArrayBlockingQueue
layout: post
date: 2021-06-10
tags: 
- java多线程
categories:
- programmer
--- 
*若图片有问题，请点击[此处](https://github.com/uk403/uk403.github.io/tree/master/_posts)查看*


本文讲**ArrayBlockingQueue**
# 1. 介绍
一个基于数组的**有界阻塞队列**，FIFO顺序。支持等待消费者和生产者线程的**可选公平策略**(默认是非公平的)。公平的话通常会降低吞吐量，但是可以减少可变性并避免之前被阻塞的线程饥饿。

<!-- more -->

# 1.1 类结构
![ArrayBlockingQueue继承关系](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210609103418260-1750643907.png "ArrayBlockingQueue继承关系")
* *ArrayBlockingQueue继承关系*

![ArrayBlockingQueue类图](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210609103633831-628166857.png "ArrayBlockingQueue类图")
* *ArrayBlockingQueue类图*

**构造器**
```java
    // 默认是非公平的
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    public ArrayBlockingQueue(int capacity, boolean fair) {
       ...
    }


    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        ...
}
```


**比较重要的几个参数**
```java

    // 储存元素的数组
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    // 与putIndex相互配合可以将数组变成一个可循环利用的数组，不需要扩容，后面会讲到
    // 每次出队的索引
    int takeIndex;

    /** items index for next put, offer, or add */
    // 每次入队的索引
    int putIndex;

    /** Number of elements in the queue */
    int count;

    /**
     * Shared state for currently active iterators, or null if there
     * are known not to be any.  Allows queue operations to update
     * iterator state.
     */
    // 迭代的时候会用到，在后面详讲
    transient Itrs itrs = null;
```

**保证线程安全的措施**
```java

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
```

我们可以看到**ArrayBlockingQueue**使用的是**单锁**控制线程安全，而**[LinkedBlockingQueue](https://www.cnblogs.com/ukyu/p/14859956.html)**是**双锁**控制的, 后者的细粒度更小。

# 2. 源码剖析
**ArrayBlockingQueue**也是继承至**BlockingQueue**(可以去看看上面提到的那篇博客有提到BlockingQueue)，它对于不同的方法不能立即满足要求的，作出的回应是不一样的。

我们分别介绍下面的方法的具体实现
* offer(E e)
* offer(E e, long timeout, TimeUnit unit)
* put(E e)
* poll()
* remove(Object o)

## 2.1 offer(E e) & poll()
> 插入成功就返回true；若队列满了就直接返回false，**不会阻塞自己**

```java
    public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                return false;
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
```

上面的代码比较简单，我们来看看入队的具体操作
```java
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;

        // 为什么putIndex+1 等于数组长度时会变成0
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        notEmpty.signal();
    }
```

为了解答上面注释中的问题，我们先看看**poll()**的实现

```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
```

```java
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;

        // takeIndex + 1等于了数组的长度也会将值置为0
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
```

结合上面的**入队、出队源码**，我们来分析一下：

* 单线程下，首先执行
```java
        ArrayBlockingQueue<String> array = new ArrayBlockingQueue<>(3);
        array.offer("A");
        array.offer("B");
        array.offer("C");
```

此时队列的状态

![offer'A''B''C'](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210609144239862-2115158077.png "offer'A''B''C'")

* 再执行
```java
        array.poll();
        array.offer("D");
```

最后队列的状态

![offer'D'](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210609144856135-1578000223.png "offer'D'")

大家可能会有点疑问，上面的队列不是输出是**"D B C"**, 咋回事？ 肯定不是啦，我们看看类重写的**toString**就明白了。

```java
 public String toString() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int k = count;
            if (k == 0)
                return "[]";

            final Object[] items = this.items;
            StringBuilder sb = new StringBuilder();
            sb.append('[');

            // 主要代码
            for (int i = takeIndex; ; ) {
                Object e = items[i];
                sb.append(e == this ? "(this Collection)" : e);
                if (--k == 0)
                    return sb.append(']').toString();
                sb.append(',').append(' ');
                if (++i == items.length)
                    i = 0;
            }
        } finally {
            lock.unlock();
        }
    }
```
思考一下，就会明白了。
通过上面的分析，我们看出了**数组就像一个循环数组一样，每个地址都被重复使用**。我们也知道了**基于数组的队列如何实现的**。

**offer(E e, long timeout, TimeUnit unit)** 与 **put(E e)**~~实现都比较简单~~，大家看看源码即可。

# 2.2 remove(Object o)
> 若o存在则移除，返回true;反之。这个操作会改变队列的结构，~~但是该方法一般很少使用~~
```java
 public boolean remove(Object o) {
        if (o == null) return false;
        final Object[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count > 0) {
                final int putIndex = this.putIndex;
                int i = takeIndex;
                do {
                    if (o.equals(items[i])) {
                        // 主要删除逻辑
                        removeAt(i);
                        return true;
                    }
                    if (++i == items.length)
                        i = 0;
                } while (i != putIndex);
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
```

```java
void removeAt(final int removeIndex) {
        // assert lock.getHoldCount() == 1;
        // assert items[removeIndex] != null;
        // assert removeIndex >= 0 && removeIndex < items.length;
        final Object[] items = this.items;
        if (removeIndex == takeIndex) {
            // removing front item; just advance
            items[takeIndex] = null;
            if (++takeIndex == items.length)
                takeIndex = 0;
            count--;
            if (itrs != null)
                itrs.elementDequeued();
        } else {
            // an "interior" remove

            // slide over all others up through putIndex.
            // 此时removeIndex != takeIndex
            // 为啥要执行下面的代码，大家可以按照上面图片的最后状态，
            // 按照下面代码走一下，就明白了.主要是设置putIndex
            final int putIndex = this.putIndex;
            for (int i = removeIndex;;) {
                int next = i + 1;
                if (next == items.length)
                    next = 0;
                if (next != putIndex) {
                    items[i] = items[next];
                    i = next;
                } else {
                    items[i] = null;
                    this.putIndex = i;
                    break;
                }
            }
            count--;
            if (itrs != null)
                itrs.removedAt(removeIndex);
        }
        notFull.signal();
    }
```

# 2.3 解释解释Itrs

```java
    // 当前活动迭代器的共享状态; 允许队列操作更新迭代器的状态；
    transient Itrs itrs = null;
```

这个变量可以理解成，在一个线程使用迭代器时，其他的线程可以对队列进行更新操作的一个保障。
源码注释中对Itrs的描述，迭代器和它们的队列之间共享数据，**允许在删除元素时修改队列以更新迭代器。** 我们可以看到对队列进行了删除操作时，队列都会执行下面的语句

```java
   if (itrs != null)
      itrs.removedAt(removeIndex);
```

初始化该值是在使用迭代器时

```java
    public Iterator<E> iterator() {
        return new Itr();
    }

    ...

  Itr() {
            // assert lock.getHoldCount() == 0;
            lastRet = NONE;
            final ReentrantLock lock = ArrayBlockingQueue.this.lock;
            lock.lock();
            try {
                    ...
                        itrs = new Itrs(this);
                    ...
                }
            } finally {
                lock.unlock();
            }
        }
```

# 3. 总结
**ArrayBlockingQueue**的实现整体不难，使用**ReetrantLock**保证了线程安全，**putIndex**与**takeIndex**分别维护入队与出队的位置，一起构成一个循环数组