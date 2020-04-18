---
title: Java8新特性
layout: post
date: 2020-04-18
tags: 
- feature
- java
categories:
- programmer
---
# Java8

[学习的视频地址](https://www.bilibili.com/video/BV1xb411w7jM)
## 1.Lambda（糖衣语法）
<!-- more -->
### 1.1目的：

* 优化匿名内部类
  
### 1.2 语法(->)

* 多条语句写‘{}’
* 一个参数，可以省略'()'

### 1.3 四大内置核心函数式接口

Consumer<T>:

​	void accept(T t);g

Supplier<T>:

​	T get();

Function<T,R>:

​	R apply(T t) ;

Predicate<T>:   断言型接口

​	boolean test(T t);


### 1.4 方法引用他与构造器引用

 ![图片](/assets/images/blog//2020-04-18-JAVA-Feature/Java8-reference.png)


## 2. Stream API
### 2.1流
    数据渠道，用于操作数据源（集合、数组）所产生的元素序列

### 2.2 Stream创建步骤
#### 2.2.1 创建Stream
* Collection的stream() & parallelStream();
* Arrays.stream();
* Stream.of();
* Stream.iterate();[seed 种子 开始]

#### 2.2.2 操作Stream(惰性求值 一次性执行全部内容)
1. 筛选与切片
    * filter 
    * limit
    * skip -- 跳过元素，返回一个扔掉前n个元素的流。若不足n个，则返回 **空流**
    * distinct -- 去除重复的元素根据 ***hasCode()*** 与 ***equals()***;

2. 映射
    * map -- 将元素转化成其他形式或提取信息
    * flatMap -- 把流中的所有值都换成另一个流，然后把所有流连接成一个流
  
    **联想list的add() & addAll() 方法**
3. 排序
   * sorted() -- comparable 自然排序
   * sorted(Comparator com) -- 定制排序

4. 查找与匹配
   * allMatch()
   * anyMatch()
   * noneMatch()
   * findFirst()
   * findAny()
   * count()
   * max()
   * min()

5. 归约
   * reduce() 将流中元素反复结合起来，得到一个新值    ----- map-reduce  

6. 收集
   * collect() -- 将流转化成其他形式 (Collectors 提供很多对集合处理的静态方法)
    
#### 2.2.3 终止Stream
* forEach

## 3. 并行流与串行流
   fork-join(工作窃取模式--自己线程没任务勒，就去其他线程去偷)
   
   parallel 并行流(底层 使用fork-join)

## 4. Optional类
   Optional 是一个容器类，代表一个值存在或不存在，可避免空指针异常

## 5. 接口中的默认方法
   > 类优先原则

## 6. 时间与日期
***新的时间与日期是不可变的，每次操作都会产生新的实例***
### 6.1 JAVA8 之前时间存在的问题：
   1. 线程不安全

### 6.2 类
- LocalDate、LocalTime、LocalDateTime

- Instant(以 Unix 元年: 1970年1月1日 00:00:00 到某个时间的毫秒值)

- Duration(时间之间的间隔)、Period(日期之间的间隔)

- TemporalAdjuster (时间校准器、自定义时间)

- DateTimeFormatter(格式化时间/日期) 