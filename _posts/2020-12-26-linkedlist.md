---
title: java集合(3)-LinkedList
layout: post
date: 2020-12-26
tags: 
- java集合
categories:
- programmer
---
LinkedList从字面上看就是一个链表，它的底层是使用的**双链表**结构实现的

**开场**

1. 值是否可以重复 -> 可以
2. 值是否可以为null -> 可以
3. 值是否有序 -> 按照插入顺序排列
4. **是否是线程安全** -> 否 

<!-- more -->

# LinkedList的总体结构
**类定义**

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

**重要的参数**
```java
    transient int size = 0;
    
    /**
     * 头节点
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * 尾节点
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
   
    // 通过该对象来定义节点的
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        ...
    }
```

**双链表的结构**
![双链表的结构](/assets/images/blog/2020-12-26-linkedlist/doubly-linked-list.png)

# 方法的剖析
讲一些~~重要的地方~~，都是基于对双链表的操作，看看源代码就可以懂得
* add方法

```java
// add方法有两个重载的方法
// 1. add(E e) 2. add(int index, E element)
// 将第二个add方法，当index == size时，和第一个的效果差不多
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    void linkLast(E e) {
        ...
        final Node<E> newNode = new Node<>(l, e, null);
        ...
    }

    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        ...
        final Node<E> newNode = new Node<>(pred, e, succ);
        ...
    }

// 不难发现，后两个方法，添加元素都会实例一个newNode出来
// 这也正是添加元素比较耗时间的地方
// 若是使用插入某个特定位置的元素的方法，耗得时间还存在遍历到特定位置的时间(去查看上面的Node<E> node(int index)方法)

```
* 获取特定索引的非空节点(node方法)

```java
    /**
     * 这个方法在该类的很多地方运用到。
     * Returns the (non-null) Node at the specified element index.
     */
    Node<E> node(int index) {
        // assert isElementIndex(index);
        
        // 判断index的大小，
        // 是否小于size的二分之一，进行向后或者向前的遍历
        // 以此减少遍历的时间
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
* 获取值

```java
    public E get(int index) {
        // 检查index是否超出范围
        checkElementIndex(index);
        // 这是为什么不能用以下方式遍历该列表的原因
        // for(int i = 0; i < list.size(); i++)
        // {
        //      list.get(i); 
        // }
        // 每个循环一次都要重头开始去node方法里面遍历，获取该索引下的结点的值
        return node(index).item;
    }

```
* 获取头节点与尾节点
     
      LinkedList提供给了多种获取last与first的方法
      具体区别可以查看源码，做个小小的了解.

# 与ArrayList的区别
**其实最关心的也是这两者的区别**
## 写在前面
ArrayList的类定义

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

ArrayList实现了RandomAccess接口

**RandomAccess接口**
> Marker interface used by <tt>List</tt> implementations to indicate that they support fast (generally constant time) random access. (实现了该接口的list，表明他们支持快速的随机访问)，The primary purpose of this interface is to allow generic algorithms to alter their behavior to provide good performance when applied to either random or sequential access lists.(提供更好的性能在随机访问或顺序访问的时候)  

**随机与顺序访问**

![随机与顺序访问](/assets/images/blog/2020-12-26-linkedlist/random-sequential.png)

当我想要9， 第一张图，可以直接访问index = 3的元素，而第二张图，只能通过23->3->17->9或者42->9的方式，这就是随机与顺序访问的区别

## 区别
ArrayList、LinkedList
1. 前者支持随机访问，而后者只支持顺序访问。
2. 前者花费的时间主要是**数组扩容**上，后者花费的时间主要在于根据**索引顺序访问**和新建节点(Node)上
3. 后者的每个节点要保存前后节点的信息，若对象较大的话，占的内存会较多

一般认识都是，前者访问快，后者 **(在last节点、first节点)** 增删较快

若性能需要优化，试试在具体的业务场景中，使用性能测试，记住两者在什么操作上耗时较多即可.

**需注意**

后者使用get(index)循环时，耗时非常大(大量数据), 上文也讲述了原因，可以根据不同类型的列表使用不同的遍历方式

```java
Object o;
if (listObject instanceof RandomAccess)
{
  for (int i=0, n=list.size(); i < n; i++)
  {
      ...
      o = list.get(i); 
      ...
  }

}
else
{
    for (Iterator i=list.iterator(); i.hasNext(); ){
        ...
        i.next();
        ...
    }
}
```

# 学到了什么
1. LinkedList底层主要对双链表进行操作
2. 实现RandomAccess接口的list，**表明该list支持快速随机访问**
3. 实现了RandomAccess的列表，可以使用任何方式遍历，没有实现的就使用迭代的方式进行遍历

---
**对自己的现状不满意只有付出更多的努力去改变它**

**如果有不对的地方或建议，请指出，谢谢啦**









