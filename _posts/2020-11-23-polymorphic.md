---
title: java基础(1)-多态
layout: post
date: 2020-11-23
tags: 
- java基础
categories:
- programmer
---
*多态是面向对象非常重要的特性之一(其余两个是继承、封装)；多态不是独立存在的一种特性，而是跟其他特性**协同工作的***

# 定义
不同类(继承自同一个基类)的对象对同一个的消息有不同的响应;
<!-- more -->

# 条件
1. 继承
2. 重写
3. 父类指向子类

# 限制
1. 只对普通方法有效，private、final、static方法(**静态方法是与类，而非类的某个对象相关联**)都不行
2. 不能使用导出类的特有方法与属性

# 实现的机制
动态绑定,在运行时根据对象的类型进行绑定.

除了final方法(private方法属于final方法)、static方法，其余的方法都是动态绑定的

---

**仔细阅读下面的代码，你会从代码中获得上面所说的部分证据~**

# 代码示例
```java
/**
 * 关于多态(运行时绑定、动态绑定)
 * 定义： 不同类(继承至同一个基类)的对象对同一个消息的不同的响应；
 * 发送消息给某个对象，做什么事情让该对象断定；
 * 
 * @author ukyu
 * @date 2020/11/21
 **/
public class PolymorphicDemo {
    public static void main(String[] args) {
        Vehicle vehicle = new Car(); // 3. 父类引用指向子类

        // 查看运行时的类
        System.out.println(vehicle.getClass());
        getVehicle(vehicle);

        // 报错，只能显式向下转型才行
        // System.out.println(vehicle.wheelNum); 
    }

    private static void getVehicle(Vehicle v)
    {
        // 无法知道究竟是调用哪一个类的方法；解决的办法就是'动态绑定',
        // 在运行时根据对象的类型进行绑定
        v.transport();
    }
}

class Vehicle {
    void transport() {
        System.out.println("运输~");
    }
}

class Car extends Vehicle{  // 1. 继承
   // 轮胎个数
    int wheelNum = 4;
    
    @Override
    void transport() {  // 2. 重写父类方法
        System.out.println("地上运输");
    }

}

class Plane extends Vehicle{
    @Override
    void transport() {
        System.out.println("天上运输");
    }
}
```
输出的结果
```
class com.ukyu.base.Car
地上运输
```

# 作用
消除类型之间的耦合性；设计模式大多数都用到了'多态'。

将改变的事物与未变的事物分离开。

```java
  Vehicle vehicle = new Plane();
  getVehicle(vehicle);
```

***多态只能发生在方法调用时***

---
**对自己的现状不满意只有付出更多的努力去改变它**

**如果有不对的地方或建议，请指出，谢谢啦**

