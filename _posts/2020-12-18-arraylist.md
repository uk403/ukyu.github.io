---
title: java集合(2)-ArrayList
layout: post
date: 2020-12-18
tags: 
- java集合
categories:
- programmer
---
ArrayList是集合中用的最多的类之一了。一起探究ArrayList的原理吧。

**开场**

1. 值是否可以重复 -> 可以
2. 值是否可以为null -> 可以
3. 值是否有序 -> 按照插入顺序排列
4. **是否是线程安全** -> 否 

<!-- more -->
ArrayList的名字就说明了底层是由array(数组)实现的

```java
The array buffer into which the elements of the ArrayList are stored.
transient Object[] elementData;

/**
  * Default initial capacity.
  */
private static final int DEFAULT_CAPACITY = 10;

/**
  * The size of the ArrayList (the number of elements it contains).
  *
  * @serial
  */
private int size;
```

三种获取实例的方法
```java
    // 1.初始化容量(推荐)，后面细说原因
    public ArrayList(int initialCapacity) {
      ...
    }

    // 2.默认容量
    public ArrayList() {
      // private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
      this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    // 3.新的ArrayList实例包含c的所有元素
    public ArrayList(Collection<? extends E> c) {
      ...
    }
```

## 扩容机制
ArrayList也称为'可变数组'集合, 容量**自动**发生变化的是在向其中添加数据时。

**扩容的核心逻辑**↓

挺简单的，仔细看就懂了
```java

    // minCapacity = size + 1
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }
    
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    // 扩容政策
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;

        // oldCapacity >> 1 = oldCapacity / 2 
        // eg: 10(二进制: 1010) >>1 -> 5(二进制: 0101)
        // 位运算的速度超过乘除运算
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        // 复制原来的数组，长度为newCapacity.
        // 这个扩容也会耗时间，所以提前初始化容量，是推荐的
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

从底层分析一下下面的代码，以默认容量的方式创建新的ArrayList实例为例；

step1:
```java
//此时，list的elementData只是一个{}(空数组)
    ArrayList<Double> list = new ArrayList<>();
```
此时
```
elementData = {}
```

add方法底层
```java
    public boolean add(E e) {
        // 判断内部能力
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```
step2: 
```java
list.add(1d);
```
在执行add前，都会去执行**ensureCapacityInternal(size + 1)**;

**这个时候，才会将elementData的容量扩展到10(默认容量)**

step3:
```java
list.add(2d);
```
只检查是否满足**该容量最少且能装下所有的元素**,不会扩容

![添加的过程](/assets/images/blog/2020-12-18-arraylist/arraylist-add.png)


## 其中的一些方法
1. System.arrayCopy()
```java 
    public void add(int index, E element) {
        ...
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        ...
    }

    public E remove(int index) {
      ...
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
      ...
    }
```
对elementData的操作很多地方都用到了System.arrayCopy()
```java
    public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```
使用:
```java
  int[] a = {1, 2, 3, 4, 5};
  int[] b = {0, 0, 0, 0, 0, 6, 8, 9, 10};
  System.arraycopy(a, 0, b, 0, 5);
  System.out.println(Arrays.toString(b));
  // output:[1, 2, 3, 4, 5, 6, 8, 9, 10]

  // 将src从src[srcPos] -- src[srcPos + length]的元素 复制给
  // dest从dest[destPos] -- dest[destPos + length]
```
2. subList(int fromIndex, int toIndex)

消除对ArrayList范围操作的需要
```java
  ArrayList<Double> list = new ArrayList<>();
  list.add(1d);
  list.add(2d);
  list.add(3d);
  list.add(4d);
  list.add(5d);
  list.subList(1, 3).clear();
  System.out.println(list);
  //output： [1.0, 4.0, 5.0]
```
3. removeIf(Predicate<? super E> filter) 
```java
 //1.8新增的方法
  ArrayList<Double> list = new ArrayList<>();
  list.add(1d);
  list.add(2d);
  list.add(3d);
  list.add(4d);
  list.add(5d);
  //过滤掉大于4的元素
  list.removeIf(x -> x > 4);
  System.out.println(list);

  //output: [1.0, 2.0, 3.0, 4.0]
```

## 扩展
### 1. modCount
在ArrayList中新增，删除，扩容都会出现对该变量的操作；
>The number of times this list has been <i>structurally modified(结构化改变)</i>. Structural modifications are those that change the size of the list, or otherwise perturb it in such a fashion that iterations in progress may yield incorrect results. -- JDK(AbstractList)

该参数提供**fail-first**行为.

如果modCount意外地被改变了，iterator(迭代器)或者list iterator(列表迭代器)在响应{@code next}, {@code remove}, {@code previous}, {@code set} or {@code add}操作时，将会抛出   ***ConcurrentModificationException***

对modCount的增加在发生结构化改变时，**就是为了得到一个fail-first行为的iterator(和 list iterator)**

### 2. 数组的最大容量
```java
    /**
     * A constant holding the maximum value an {@code int} can
     * have, 2<sup>31</sup>-1.(2^31-1)
     */
    @Native public static final int   MAX_VALUE = 0x7fffffff;

    /**
     * The maximum size of array to allocate.(数组可分配的最大值)
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

```java

// 想测试最大的容量，得到了下面的错.
ArrayList<String> list = new ArrayList<>(Integer.MAX_VALUE - 8);
// output:
// Exception in thread "main" java.lang.OutOfMemoryError: Requested array size exceeds VM limit
```
**我还是不知道数组的最大容量？**

1. 数组的下标是int，所以不能超过Integer.MAX_VALUE;
2. 其容量还跟自己分配给JVM的大小有关

**待我学了 ***JVM***，看能否回答该问题**

## 学到了什么
1. ArrayList的扩容机制，在每次增加时，会检测容量的大小；扩容至1.5倍
2. 推荐使用**初始化容量**的方式获取实例，减少扩容次数
3. 使用无参构造创建对象时，elementData是在第一次add，容量才扩容至10的

---
**对自己的现状不满意只有付出更多的努力去改变它**

**如果有不对的地方或建议，请指出，谢谢啦**

