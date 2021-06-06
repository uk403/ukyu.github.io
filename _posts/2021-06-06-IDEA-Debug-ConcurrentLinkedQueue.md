---
title: IDEA debug ConcurrentLinkedQueue时抽风
layout: post
date: 2021-06-06
tags: 
- 工具
categories:
- programmer
---  
*若图片有问题，请点击[此处](https://github.com/uk403/uk403.github.io/tree/master/_posts)查看*

# 1. 介绍
如标题所见，我在使用**IDEA** **debug** *ConcurrentLinkedQueue*的**Offer**方法时，发生了下面的情况。

代码如下:
```java
        ConcurrentLinkedQueue<string> queue = new ConcurrentLinkedQueue<>();
        queue.offer("A");
        queue.offer("B");
```

<!-- more -->

第一种打断点的地方：

![未取消前，一个断点debug](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210604114447314-880316956.png "未取消前，一个断点debug")

第二种打断点的地方：

![未取消前，两个断点debug](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210604152901697-127705636.png "未取消前，两个断点debug")

如你所见，相同的地方，打断点的地方不同，导致代码执行的路径都不同，当时我也是迷茫的。

# 2. 解释原因
![IDEA debug 默认设置](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210604153708703-1520596210.png "IDEA debug 默认设置")
IDEA 默认会开启以上两个配置
>*  **Enable alternative view for Collection classes**； Select this option to display contents of classes that implement Collection, Map, or List **in a more convenient format** (for example, to display each map entry as a key-value pair).属于集合的类以更加方便的格式展示其内容(例如，展示map的entry就以键值对展示)

> * **Enable toString() object view**；Allows you to configure which classes use the result of toString() as their display value.展示值以它们toString()的结果展示。

**debug**时，代码之外，额外执行的只有**toString**，第一个配置也会调用**toString**，所以我们定位到了罪魁祸首是**toString**。我们看看**ConcurrentLinkedQueue**的**toString**

# 2.1 toString
我们去看看哪个类中实现了**toString**
![ConcurrentLinkedQueue继承关系](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210604161424381-1096109595.png "ConcurrentLinkedQueue继承关系")

👇

![AbstractQueue继承关系](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210604161603658-1664803217.png "AbstractQueue继承关系")

👇

![AbstractCollection继承关系](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210604161719710-1500760771.png "AbstractCollection继承关系")

最后找到了**AbstractCollection**中实现了**toString**

![AbstractCollection的toString](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210604162209160-1089814230.png "AbstractCollection的toString")

**iterator()**, 在**ConcurrentLinkedQueue**有实现。
在**iterator()**的实现中，跟这篇文章有关的方法是**advance()**

> 移动到下一个**有效节点**并返回 item 以返回 next()，如果没有，则返回 nul
```java
private E advance() {
            lastRet = nextNode;
            E x = nextItem;

            Node<e> pred, p;
            // 第一次进来，nextNode还没被赋值，此时默认值为null
            if (nextNode == null) {
                // 这里就是关键了
                p = first();
                pred = null;
            } else {
                pred = nextNode;
                p = succ(nextNode);
            }

            for (;;) {
                if (p == null) {
                    nextNode = null;
                    nextItem = null;
                    return x;
                }
                E item = p.item;
                if (item != null) {
                    nextNode = p;
                    nextItem = item;
                    return x;
                } else {
                    // skip over nulls
                    Node<e> next = succ(p);
                    if (pred != null && next != null)
                        pred.casNext(p, next);
                    p = next;
                }
            }
        }
```

> 返回链表第一个活跃的结点(非空，指向的item不为空)，如果链表为空就返回null
```java
    Node<e> first() {
        restartFromHead:
        for (;;) {
            for (Node<e> h = head, p = h, q;;) {
                boolean hasItem = (p.item != null);
                if (hasItem || (q = p.next) == null) {
                    // 更新头结点
                    updateHead(h, p);
                    return hasItem ? p : null;
                }
                else if (p == q)
                    continue restartFromHead;
                else
                    p = q;
            }
        }
    }
```

这篇文章也提到过，updateHead()方法。[JAVA并发(4)-并发队列ConcurrentLinkedQueue](https://www.cnblogs.com/ukyu/p/14832585.html)
> 更新头结点，将之前的头结点的next指向自己。
```java
    final void updateHead(Node<e> h, Node<e> p) {
        if (h != p && casHead(h, p))
            h.lazySetNext(h);
    }
```

# 2.2 debug
我们按照**最上面的代码**且按照**第一种打断点的方式**重新**debug**，下面会以图片形式展示整个过程。
* 初始状态，此时**A**已进入队成功

![重新debug，初始状态](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210606104157951-2141176813.png "重新debug，初始状态")

* 我们知道了，使用IDEA debug时，会调用类的**toString()方法**，此时调用**toString()方法**后，状态如下

![重新debug，调用toString后](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210606104851531-1561480637.png "重新debug，调用toString后 ")

此时，**Node1**的**next**被修改成指向自身。
这里也是网上很多博客会认为，第一次入队后，会把第一个节点的**next**指向自身的原因，其实并不会的。

* 当我们debug到**queue.offer("B")**时，此时执行到**offer()方法**中的**else if (p == q)**时，就为**true**了

![重新debug，执行p == q](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210606112443519-1092533863.png "重新debug，执行p == q")

# 3. 总结
经过了上面的分析，大家应该知道为什么会出现文章开头的问题了吧。也许你会迷迷糊糊的，因为涉及到了**ConcurrentLinkedQueue**的源码分析。
那我就用一句话，告诉你原因吧，当使用**IDEA** **debug**时，默认那两个配置是启用的，两个配置会调用**toString**，我们应该清楚**toString**是否被**重写**；是否影响**debug**某个类时，**代码的执行路径**。

可能你会觉得是**IDEA**的**bug**(我当时也这样认为)，但是我们先看看下面取消两个配置前后的**debug**情况

* 取消配置前
![取消配置前](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210606114417227-1150952674.png "取消配置前")

* 取消配置
![取消配置](https://img2020.cnblogs.com/blog/2192575/202106/2192575-20210606114716865-1819670719.png "取消配置")

一眼就可以看出，**取消配置前**在**debug**时，更加直观。
可能你会认为是**Doug Lea**的**bug**(反正我不敢这么想。当然这句话只是开玩笑啦)。

我只是让大家记住**IDEA**在**debug**时会存在这样的问题，大家也可以在评论区告诉其他同学，除了**ConcurrentLinkedQueue**外，还有**哪些类**，在**哪种情况**下会存在这样的问题

可能大家会有疑问，在**debug**时，调用了**toString**，那是否影响后续的执行。不会的，因为**tail**节点会被修改的在后续的执行中。可以结合上面那篇博客，就很清楚了。

# 4. 参考
* [IDEA debug ConcurrentLinkedQueue 的时候踩的坑](https://blog.csdn.net/AUBREY_CR7/article/details/106331490)   --- 给我提供了问题所在
* [Customize views](https://www.jetbrains.com/help/idea/customizing-views.html#configure-additional-options)
