---
title: java集合(1)-总的概括
layout: post
date: 2020-12-12
tags: 
- java集合
categories:
- programmer
---
Java集合简略版： 

![Java集合简略版](../assets/images/blog/2020-12-15-Collection-Map/java-collection-framework-hierarchy.jpg)

Java集合详细版：

![Java集合详细版](../assets/images/blog/2020-12-15-Collection-Map/ContainerTaxonomy.png)

不说太多，集合是日常开发基本上是天天都需要用到的东西，仔细地去阅读几个常用的实现还是很有必要的；

Collection接口是集合层次的根接口，定义了一个集合需要具备的能力
<!-- more -->
集合可以分成两个大块
## List
List是继承自Collection接口，它的实现类的特点
1. 有序
2. 可以有重复的值

后面讲实现List接口的ArrayList以及LinkedList的具体实现
## Set 
1. 没有重复值
2. 不保证有序(有的有序、有的无序)

HashSet以及TreeSet会讲到
# Map
Map是一个将键映射到值的对象， 是为了取代Dictionary类的；叫Dictionary还好一些，但是这个名字已经被使用了
1. 不能包含重复的值
2. 不保证有序
3. 是一个独立的接口(不是Collection的派生类)

HashMap(非常重要)、TreeMap、LinedHashMap会去研究研究。

---
**对自己的现状不满意只有付出更多的努力去改变它**

**如果有不对的地方或建议，请指出，谢谢啦**




