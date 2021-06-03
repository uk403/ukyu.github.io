---
title: java多线程(5)-ConcurrentLinkedQueue
layout: post
date: 2021-06-03
tags: 
- java多线程
categories:
- programmer
---  
*若图片有问题，请点击[此处](https://github.com/uk403/uk403.github.io/tree/master/_posts)查看*

本文开始介绍并发队列，为后面介绍**线程池**打下基础。并发队列莫非也是**出队**、**入队**操作，还有一个比较重要的点就是如何保证其**线程安全性**，有些并发队列保证**线程安全**是通过**lock**，有些是通过**CAS**。
我们从**ConcurrentLinkedQueue**开始吧。

# 1. 介绍
**ConcurrentLinkedQueue**是**集合框架**的一员，是一个无界限且线程安全，基于**单向链表**的队列。该队列的顺序是**FIFO**。当**多线程访问公共集合**时，使用这个类是一个不错的选择。**不允许null元素**。是一个非阻塞的队列。

<!-- more -->

它的迭代器是**弱一致性**的，不会抛出**java.util.ConcurrentModificationException**，也可能在迭代期间，其他操作也正在进行。**size()**方法，不能保证是正确的，因为在迭代时，其他线程也可以操作该队列。

# 1.1 类图
![ConcurrentLinkedQueue类图](https://img2020.cnblogs.com/blog/2192575/202105/2192575-20210531233315962-1875719382.png "ConcurrentLinkedQueue类图")

(***显示的方法都是公有方法***)

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>
```

继承至**AbstractQueue**，他提供了队列操作的一个框架，有基本的方法，**add**、**remove**，**element**等等，这些方法基于**offer**，**poll**，**peek**(最主要看这几个方法)。

# 2. 源码分析
## 2.1 类的整体结构

队列中的元素**Node**
```java
private static class Node<E> {
        // 保证两个字段的可见性
        volatile E item;
        volatile Node<E> next;

        /**
         * Constructs a new node.  Uses relaxed write because item can
         * only be seen after publication via casNext.
         */
        Node(E item) {
            UNSAFE.putObject(this, itemOffset, item);
        }

        boolean casItem(E cmp, E val) {
            return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

        void lazySetNext(Node<E> val) {
            // putOrderedXXX是putXXXVolatile的延迟版本，设置某个值不会被其他线程立即看到(可见性)
            // putOrderedXXX设置的值的修饰应该是volatile，这样该方法才有用

            // 关于为什么使用这个方法，主要目的肯定是提高效率，但是具体原理，我只能告诉大家跟内存屏障有关(我也不太清楚这一块，待我研究后，再写一篇文章)
            UNSAFE.putOrderedObject(this, nextOffset, val);
        }

        boolean casNext(Node<E> cmp, Node<E> val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        // Unsafe类中的东西，可以去了解一下

        private static final sun.misc.Unsafe UNSAFE;
        private static final long itemOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Node.class;
                itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```
构造器1:
```java
    // private transient volatile Node<E> head;
    // private transient volatile Node<E> tail;
    public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
    }
```

构造器2:
```java
public ConcurrentLinkedQueue(Collection<? extends E> c) {
        Node<E> h = null, t = null;
        for (E e : c) {
            checkNotNull(e);
            Node<E> newNode = new Node<E>(e);
            if (h == null)
                h = t = newNode;
            else {
                t.lazySetNext(newNode);
                t = newNode;
            }
        }
        if (h == null)
            h = t = new Node<E>(null);
        head = h;
        tail = t;
    }
```

下面开始讲方法，从**offer**，**poll**，**peek**从这几个方法入手

## 2.2 offer
> 添加元素到队尾。因为队列是无界的，这个方法永远不会返回false

分为三种情况进行分析(**一定自己跟着代码debug，一步步的走**)

1. 单线程时（**使用IDEA debug一直进入的是 **else if**把我搞迷茫了，我会写一个博客来解释原因**）

```java
        ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();
        queue.offer("A");
        queue.offer("B");
```
以上面的代码，分析每一个步骤。
**执行构造函数后:**

![单线程初始化](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602153919297-1170047519.png "单线程初始化")

此时链表的head与tail指向**哨兵节点**

**插入"A",** **此时没有设置tail**('两跳机制'，这里的原因后面详见)

![单线程插入'A'](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602155433497-964470688.png "单线程插入'A'")

**插入"B",**
![单线程插入'B'](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602161124411-67788549.png "单线程插入'B'")

单线程情况比较简单

2. 多线程offer时
```java
 public boolean offer(E e) {
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);

        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next;
            if (q == null) {
                // p is last node
                // 只有一个线程能够CAS成功，其余的都重试
                if (p.casNext(null, newNode)) {

                    // 延迟设置tail，第一个node入队不会设置tail，第二个node入队才会设置tail
                    //以此类推, '两跳机制'
                    if (p != t) // hop two nodes at a time
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            // 这里是有其他线程正在poll操作才会进入，此时只考虑多线程offer的情况，暂不分析
            else if (p == q)
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                p = (t != (t = tail)) ? t : head;
            else
                // Check for tail updates after two hops.
                // 存在tail被更改前，和更改后的两种情况
                p = (p != t && t != (t = tail)) ? t : q;
        }
    }
```
**结合上面的代码**，看图

* **步骤一**，**线程A**、**线程B**都执行到
```java
   if (p.casNext(null, newNode))
```
![多线程初始化](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602203755768-7253178.png "多线程初始化")


* **步骤二**，只有一个线程执行成功，假设**线程A**成功，**线程B**失败

![多线程Offer2](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602204617735-286206951.png "多线程Offer2")

因为**p(a) == t(a)**, 此时不执行*casTail*，**tail**不变。*q = p.next*, 所以此时**q(b) = Node2** ，那么 *p(b) != q(b)*, **线程B**执行*p = (p != t && t != (t = tail)) ? t : q;*

 **线程B**即将执行
```java
   p = (p != t && t != (t = tail)) ? t : q;
```

* **步骤三** 此时线程C进入。
此时，*p(c) != q(c)*, **线程C**执行
```java
   p = (p != t && t != (t = tail)) ? t : q;
```
执行完后，**q(c)**赋值给**p(c)**. 再次循环，此时，*q(c) == null*, 设置p(c)的next，**线程C**将值入队

![线程C Offer1.PNG](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602211520203-1479630248.png "线程C Offer1.PNG")

* **步骤四** *p(c) != t(c)*, **线程C**执行*casTail(t, newNode)*, **线程C**设置尾结点

![线程C 设置tail后](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602213608358-1719698377.png "线程C 设置tail后")
* 此时线程B执行
```java
   p = (p != t && t != (t = tail)) ? t : q;
```
因为p(b) == t(b)，所以 q(b) 赋值给 p(b)。继续循环，最后得到
![多线程offer，B插入](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602213919565-785118140.png "多线程offer，B插入")


3. **多线程的另一种情况**，回到**步骤三**，此时**线程C**把值入队了，但是还没有设置**tail**

![线程C Offer1.PNG](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602211520203-1479630248.png "线程C Offer1.PNG")

* **线程B**，将值入队成功
在**步骤三**的基础上，**线程B**入队成功后，目前的状况如下:

![线程C 设置tail前](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602221927417-1989854812.png "线程C 设置tail前")

此时，**线程C**执行**casTail(t, newNode)**，但是现在的tail != t(c), CAS失败, 直接返回。

### 2.2.1 小结
上面不管是多线程还是单线程，都是努力的去寻找**next为null**的节点，若为**next节点**为null，再判断是否满足设置**tail**的条件。

多线程**offer**的第一种情况存在**设置tail**滞后的问题，我把它称之为**"两跳机制"**，后面讲使用这种机制的原因。
我们看到上面的情况一直没有进入**else if (p == q)**分支，进入**else if**分支只会发生在有其他线程在**poll**时，我们先讲讲**poll**，再讲讲何时进入**else if**分支。

## 2.3 poll
> 删除并返回头结点的值

简单提一下**单线程**与**多线程**的**poll**，着重分析一下**poll**与**offer**共存的情况

1. 单线程时

![单线程poll](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210602234902514-896561837.png "单线程poll")
单线程比较简单，就不画图了，按照上面的**queue**，进行**一步一步的debug**就行了

2. 多线程，只有**poll**时
```java
 public E poll() {
        restartFromHead:
        for (;;) {
            for (Node<E> h = head, p = h, q;;) {
                E item = p.item;

                // casItem这里只有一个线程能够成功，其余的继续下面的代码
                if (item != null && p.casItem(item, null)) {
                    // Successful CAS is the linearization point
                    // for item to be removed from this queue.
                    if (p != h) // hop two nodes at a time
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                else if ((q = p.next) == null) {
                    updateHead(h, p);
                    return null;
                }
                else if (p == q)
                    continue restartFromHead;
                else
                    p = q;
            }
        }
    }
```

```java
    final void updateHead(Node<E> h, Node<E> p) {
        if (h != p && casHead(h, p))
            // 将之前的头节点，自己指向自己，等待被GC
            h.lazySetNext(h);
    }
```

从上面代码可以看出，修改**item**与**head**都会使用CAS，这些变量都是被**volatile**修饰，所以保证了这些变量的**线程安全性**。不管是单线程还是多线程的**poll**，它们都是去寻找一个**有效的头节点**，删除并返回该值，若不是有效的就继续找，若队列为空了，就返回**null**。

最后分析一下，**offer**与**poll**共存的情况
* **线程A**做**offer**操作，**线程B**做**poll**操作，初始的状态如下：

![多线程offer与poll初始时](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210603154419621-318176292.png "多线程offer与poll初始时")

* **线程A**进入。

![多线程offer与poll，offer1](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210603164825146-463006435.png "多线程offer与poll，offer1")

* **线程A**将要执行
```java
Node<E> q = p.next;
```
**线程B**进入，进行**poll**操作
此时，**线程B**执行了一次内循环，将q(b)赋值给了p(b)；
![多线程offer与poll，poll1](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210603165807529-627261042.png "多线程offer与poll，poll1")

* **线程B**再次执行内循环，此时将**p(b).item**置空，将**p(b)**赋值给**head**，之前的**h(b)**的**next**指向自己，**线程B**退出

![多线程offer与poll，poll2](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210603164104387-1365739830.png "多线程offer与poll，poll2")

* **线程A**执行
```java
  Node<E> q = p.next;
```

![多线程offer与poll，offer2](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210603170915661-1412641137.png "多线程offer与poll，offer2")

此时，**p(a).next** 指向自己(等待被*GC*), 进入**else if (p == q)**分支，**线程A**退出，经过一番执行后，最后得到的状态，如下:

![多线程offer与poll，offer3](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210603172438445-2100669105.png "多线程offer与poll，offer3")

进入**else if (p == q)**分支的情况，只会发生在**poll**与**offer**共存的情况下。

## 2.4 peek
> 获取首个有效的节点，并返回
```java
public E peek() {
        restartFromHead:
        for (;;) {
            for (Node<E> h = head, p = h, q;;) {
                E item = p.item;
                if (item != null || (q = p.next) == null) {
                    updateHead(h, p);
                    return item;
                }
                else if (p == q)
                    continue restartFromHead;
                else
                    p = q;
            }
        }
    }
```
**peek**与**poll**的操作类似，这里就贴一下代码就是了。

# 3. 总结
**ConcurrentLinkedQueue**是使用非阻塞的方式保证线程的安全性，在设置关系到整个**Queue**结构的变量时(这些变量都被**volatile**修饰)，都使用**CAS**的方式对它们进行赋值。

* size方法是**线程不安全的**，返回的结果可能不准确

**关于“两跳机制”(自己取得名字)，**
> Both head and tail are permitted to lag.  In fact, failing to update them every time one could is a significant optimization (fewer CASes). As with LinkedTransferQueue (see the internal documentation for that class), we use a slack threshold of two; that is, we update head/tail when the current pointer appears to be two or more steps away from the first/last node.

> Since head and tail are updated concurrently and independently, it is possible for tail to lag behind head (why not)? -- ConcurrentLinkedQueue

大致意思，**head**与**tail**允许被延迟设置。不是每次更新它们是一个重大的优化，这样做就可以**更少的CAS**(这样在很多线程使用时，积少成多，效率更高)。它的延迟阈值是2，设置head/tail时，当前的结点离first/last有两步或更多的距离。 这就是“*两跳机制*”

我们想不通的地方，可能是这个类或方法的一个优化的地方。向着大佬看齐~

# 4. 引用
[Java多线程 39 - ConcurrentLinkedQueue详解](https://blog.coderap.com/article/249)，讲的非常好，上面的思路是跟着他来的
