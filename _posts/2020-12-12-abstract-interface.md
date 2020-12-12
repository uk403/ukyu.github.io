---
title: java基础(2)-抽象类与接口
layout: post
date: 2020-12-12
tags: 
- java基础
categories:
- programmer
---
**抽象类与接口是两个经常面试会提到的概念，我们去一探这两个概念的究竟**
# 抽象类
**包含抽象方法的类**就是抽象类，此类必须限定为抽象类；~~~但是没有抽象方法，也可以将该类定义为抽象类，但是就失去了抽象类的意义~~~
<!-- more -->

```java
/**
 * @author ukyu
 * @date 2020/12/11
 **/ // 飞行器
abstract class Light {
    int brightness = 0;

    // 得到灯的亮度
    int getBrightness(int brightness) {
        return (this.brightness = brightness);
    }

    // 声明关闭键
    abstract void turnOff();

}

// 手电筒
class FlashLight extends Light {

    @Override
    void turnOff() {
        System.out.println("拨动开关关闭~");
    }

    public static void main(String[] args) {
        Light light = new FlashLight();
        System.out.println(light.getBrightness(100));
        light.turn();
    }

}

```
输出结果：
```
 100
 拨动开关关闭~
```
结合
[多态](https://uk403.github.io/programmer/2020/11/23/polymorphic/)
食用更佳

可以看到派生类使用了基类共享的**getBrightness**的方法，我们实现了抽象方法**turn**(**若不实现的话，其派生类还是一个抽象类**)

再新增一个派生类
```java
// 手机灯
class PhoneLight extends Light {

    @Override
    void turnOff() {
        System.out.println("点击按钮关闭~");
    }

    public static void main(String[] args) {
        Light light = new PhoneLight();
        System.out.println(light.getBrightness(1000));
        light.turn();
    }
}
```

输出结果：
```
 1000
 点击按钮关闭~
```

如果有更多的派生类，我们只需要实现它的'关闭键'就可以了。而，'获取亮度'直接调用父类就可以了，这时，需要想起一个词**代码复用**，这也是使用抽象类的原因之一。
# 接口
接口就是**一个合同**，对类进行约束。实现接口的类必须实现接口定义的行为(行为也可以理解为能力)， 而具体行为的实现不用去管;**完全抽象的类**即接口

```java
/**
*  @author ukyu
*  @date   2020/12/11
**/
interface Turn {

    // 方法的访问修饰符默认是public，也只能是public
    void turnOn();

    void turnOff();

}

public class ImplTurn implements Turn {

    @Override
    public void turnOn() {
        System.out.println("用手开");
    }

    @Override
    public void turnOff() {
        System.out.println("声音关");
    }
}
```
正如实现类中看到，你想怎么实现就怎么实现，但是必须要实现接口声明的方法。就对你进行了**约束**。如果我再编写一个类去实现接口，该类也必须实现接口声明的方法。


# 区别
|  abstract   | interface  |
|  :----:                           | :----:                    |
| 变量修饰符可以是任意的      | 只能是 public final static |
| 可以继承其他Java类，实现多个Java接口       |  只能继承其他的**Java接口** |
| 可以有普通方法(除了抽象方法)       |  只能声明方法，不能有默认的方法体 |
|可以有构造函数|没有构造函数|
|...                      |...|


# 什么时候用？
抽象类的使用场景:

1. 有一些可以让多个类通用(或无聊)的方法；相关对象
2. 你希望访问修饰符不仅仅是public 

接口的使用场景：

1. 面向行为的，不管该对象是什么，只要拥有该行为就行了；不相关对象
2. 对实现接口的类，进行行为的约束，就像插座一样，必须有某种规定的行为才可以插入(三孔插三孔、两孔就插两孔，不管你如何去实现)
3. 需要多继承时
# Java8带来的改变

Java8接口支持default 和 static定义方法，主要的目的就是引入新的特性(eg: lambda函数)后，**向后兼容**；

在接口中使用default和static的情况:
1. 当你声明的接口被实现后，但是你想新增一个行为约束(方法)时。(在Java8之前,~~~每一个实现该接口的类，都必须实现这个新增的约束~~~)


> Default methods enable you to add new functionality to existing interfaces and ensure binary compatibility(兼容性) with code written for older versions of those interfaces

# 结合起来用

```java
// ArrayList头部：
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
          ...
        }

// AbstractList头部:
public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E>{
  ...
}

// List接口头部
public interface List<E> extends Collection<E> {
  ...
}
```
abstract 与 interface都有自身的优势与限制，我们可以像ArrayList一样，将抽象类与接口结合起来使用。

在AbstractList抽象类中实现所有List接口的方法，而在ArrayList类中，只需要自定义自己想实现的方法，可以不全部实现List接口中声明的方法。

# 总结
1. 接口也是类，只是完全抽象的类

2. 抽象程度： 普通类 < 抽象类 < 接口

3. 接口是一个合同，对你的行为进行约束，吃、睡觉是一种行为，怎么吃、怎么睡，就不用关心了；
4. 抽象类是对整个类的抽象，男人、女人都是人(基类)，继承抽象类的派生类，它们具有相似之处;
   
5. 被实现了的老版本的接口，如果你想扩展，可以考虑使用default与static，它们不会破坏实现

---

# 参考
1. [What is the difference between an interface and abstract class?](https://stackoverflow.com/questions/1913098/what-is-the-difference-between-an-interface-and-abstract-class
)
2. [Default Methods](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html
)

---
**对自己的现状不满意只有付出更多的努力去改变它**

**如果有不对的地方或建议，请指出，谢谢啦**



