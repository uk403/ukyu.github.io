---
title: 阅读SpringBoot文档的过程
layout: post
date: 2020-03-20
tags: 
- SpringBoot

categories:
- programmer
---
原文[Spring Boot Reference Documentation](https://docs.spring.io/spring-boot/docs/2.2.4.RELEASE/reference/htmlsingle);听说这是学SpringBoot最好的方法，记录一下阅读它的过程；
<!-- more -->
# 2020-03-20
今天晚上的十点钟，想起这样开始写这么一篇文章。

from：3.8.3. LiveReload

---

* devtools的局限性
  > Restart functionality does not work well with objects that are deserialized by using a standard ObjectInputStream. If you need to deserialize data, you may need to use Spring’s ConfigurableObjectInputStream in combination with Thread.currentThread().getContextClassLoader().Unfortunately, several third-party libraries deserialize without considering the context classloader. If you find such a problem, you need to request a fix with the original authors.

to：4.1.6. Application Events and Listeners[未看]

单词：
* ***JVM has `sufficient[足够]` memory to `accommodate[容纳]` all of the application’s beans***

