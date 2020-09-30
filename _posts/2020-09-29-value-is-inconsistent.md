---
title: 前端接收到的后端传的值不一致？
layout: post
date: 2020-09-29
tags: 
- lombok
- jackson
- bean
categories:
- programmer
---
```
@Data
public class ukyu{
    private String fInfo;
}
```
**有个字段叫作 ```finfo``` ，idea觉得这个单词是错误的，提示修改。我不想要在该字段下有个波浪线，我就把该字段改成了 ```fInfo``` 。耶，问题解决 -------才怪，问题才刚刚开始**
<!-- more -->
# 前提
&emsp;&emsp;使用lombok的@Data为该类的字段提供getter、setter等方法。返回json给前端
# 问题出现
&emsp;&emsp;在前端获取 ```fInfo``` 时，却获取不到。在Idea上使用debug，看见确实返回了这个值的。然后，在浏览器中按F12，查看这个请求，为什么前端接受的值却是 ```finfo```。反复了几次上面的步骤，相信这是个事实了。通过搜索与思考，发现了问题所在。
# 直击问题
&emsp;&emsp;要知道使用@Data，setter与getter是将字段的第一个字母大写。如下：
```
// 使用lombok
 getFInfo()
 setFInfo()

// 若字段第一個字母是小写，得到的getter、setter将首字母大写
// 若字段第二个字母是大写，就直接用get、set拼接字段
// Bean规范得到的getter、setter
 getfInfo()
 setfInfo()

```
数据会经过jackson进行处理成json进行返回给前端。经过搜索，我发现了jackson会将**字段的getter前面的get去掉，将get后面连续的大写字母变成小写字母，所得到的值就是返回的字段**。如下：
```
getFInfo() -> 经过jackson的转化，得到的字段为 finfo
这就是为什么，前端接受到的是 finfo，而不是fInfo了
```
可能你已经明白了问题的所在了，那么我们下面说说如何解决这种问题。

# 解决问题
1. 自己创建有问题字段的setter、getter，但是必须**按照Bean规范来创建**，推荐使用Idea的快捷生成setter、getter方式。**亲测有用**~~那为什么要按照Bean规范来？自己思考~~
2. @JsonProperty
```
@Data
public class ukyu{
    @JsonProperty(value = "fInfo") // 若value的值和字段名相同，可以省略
    private String fInfo;
}
```
 神奇的事情发生了，

 ![传回来了两个值](/assets/images/blog/2020-09-29-value-is-inconsistent/two-values.png)

传回来了两个值。其原因:

 ![jackson对pojo序列化处理](/assets/images/blog/2020-09-29-value-is-inconsistent/jackson-serialized-pojo.png)

所以传给前端就有两个值，```fInfo``` 与 ```finfo```

# 看看源码
> 就是这个方法处理的getter、setter方法，

> com.fasterxml.jackson.databind.util.
BeanUtil#legacyManglePropertyName

```java
protected static String legacyManglePropertyName(String basename, 
    int offset) {  //当是getter或setter时，offset = 3
        int end = basename.length();
        if (end == offset) {
            return null;
        } else {
            char c = basename.charAt(offset);
            char d = Character.toLowerCase(c);
            if (c == d) {
                return basename.substring(offset);
            } else {
                StringBuilder sb = new StringBuilder(end - offset);
                sb.append(d);

                for(int i = offset + 1; i < end; ++i) {
                    c = basename.charAt(i);
                    d = Character.toLowerCase(c);
                    if (c == d) {
                        sb.append(basename, i, end);
                        break;
                    }

                    sb.append(d);
                }

                return sb.toString();
            }
        }
    }
```

# 总结
遇到这个情况的最优解就是自己创建getter、setter；**不到万不得已，创建字段还是要符合Bean规范**