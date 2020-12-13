---
title: java基础(3)-Comparable&Comparator
layout: post
date: 2020-12-12
tags: 
- java基础
categories:
- programmer
---
Java提供两种比较的方式，一种是**类直接相互比较**(自己实现Comparable接口);一种是自定义一个**比较器**(Comparator接口)将两个欲比较的对象放进去比较。

看看这两者的区别与使用场景吧。

# Comparable
Comparable只是一个接口，具体的逻辑放在CompareTo方法(自然比较方法)中

> 如果List或Array中的对象实现了Comparable接口，就可以调用Collections.sort或Arrays.sort自动排序；当对象实现了Comparable接口时，可以用作SortedMap的key值，SortedSet的元素，不需要指明一个比较器(Comparator)   -- JDK

CompareTo方法，返回结果三种情况：
1. 大于返回正数
2. 等于返回0
3. 小于返回负数

<!-- more -->

```java
/**
 * 按照Student的name进行自然排序
 * @author ukyu
 * @date 2020/12/12
 **/
public class Student implements Comparable<Student>{

    private final String name;
    private Integer id;

    Student(String name, Integer id){
        this.name = name;
        this.id = id;
    }

    public String getName(){
        return name;
    }

    public Integer getId(){
        return id;
    }

    @Override
    public int compareTo(Student o) {
        // 简便了判断
        return Integer.compare(this.name.compareTo(o.name), 0);
    }

    public static void main(String[] args) {
        Student demo1 = new Student("a", 1);
        Student demo2 = new Student("b", 2);
        Student demo3 = new Student("b", 2);
        Student demo4 = new Student("c", 3);

        System.out.println(demo1.compareTo(demo2));
        System.out.println(demo2.compareTo(demo3));
        System.out.println(demo4.compareTo(demo2));

    }
}

// output：
// -1
// 0
// 1
```
**Comparable只能按照一种参数进行排序；Comparable可以理解为'内比较器'**

上面的排序是按照name来排序的，如果我想通过id来排序，此时要么就放弃原有的name排序，但是别人可能就需要name排序, 这种情况的话，就引出Comparator(比较器)接口来解决该问题.

# Comparator
Comparator就是比较器的意思，它与要比较的类是分开的。可以理解为 **'外比较器'**
```java
/**
 * 按照Student的id进行排序
 * @author ukyu
 * @date 2020/12/12
 **/
public class IdComparator implements Comparator<Student> {

    @Override
    public int compare(Student o1, Student o2) {
        return Integer.compare(o1.getId().compareTo(o2.getId()), 0);
    }

    public static void main(String[] args) {
        StudentComparator d = new StudentComparator();
        Student d1 = new Student("a", 1);
        Student d2 = new Student("b", 2);
        Student d3 = new Student("b", 2);
        Student d4 = new Student("c", 3);

        System.out.println(d.compare(d1, d2));
        System.out.println(d.compare(d2, d3));
        System.out.println(d.compare(d4, d3));
    }

}

// output：
// -1
// 0
// 1
```
若想比较其他的Student属性，就再增加其他的Comparator，按照自己想要的方式去实现compare方法就行了。

# 总结
1. 当你需要对象排序基于自然数排序时，使用Comparable接口
2. 当你需要不满意已有的Comparable接口的实现方式，或想使用其他属性进行排序，请使用Comparator接口
3. Comparble接口比较的是'this'引用与传入的对象，Comparator接口是比较两个不同(不同于Comparator)的对象。

根据具体情况，选用不同的比较接口

---
**对自己的现状不满意只有付出更多的努力去改变它**

**如果有不对的地方或建议，请指出，谢谢啦**

     