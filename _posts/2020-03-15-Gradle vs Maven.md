---
title: Gradle vs Maven
layout: post
date: 2020-03-18
tags: 
- gradle
- maven
- 翻译技术文档
categories:
- programmer
---

翻译[Gradle vs Maven Comparison](https://gradle.org/maven-vs-gradle/#close-notification)
通过灵活性、性能、用户体验和依赖管理进行比较。

# 灵活性
Google选用Gradle作为安卓的官方构建工具；
不是因为构建脚本是代码，
***but because Gradle is modeled in a way that is extensible in the most fundamental ways*** (是因为高度可定制)。
Gradle的模型也可以用于C/C++的原生开发并且可以扩展到覆盖任何的生态。例如，Gradle
设计时考虑使用Tooling API(通过这个，可以将Gradle嵌入自己的软件，IntelliJ IDEA使用它导入你的Gradle项目和运行各种任务)进行嵌入。
<!-- more -->
Gradle和Maven都提供了配置约定。可是，Maven提供了一个非常死板的模板，导致自定义乏味，有时一些自定义不可能实现。尽管这样它会让你更容易理解任何一个Maven构建，但是只要你没有任何特殊要求，它也不适合许多自动化问题。另一方面，***Gradle, on the other hand, is built with an empowered and responsible user in mind.***(Gradle在构建时考虑的是授权和负责人的用户)

# 性能
缩短构建时间是加快交付时间最直接方法之一。Gradle和Maven都采用某种形式的并行项目构建和并行依赖项解析。最大的不同是Gradle采用避免工作和增加工作量的机制。使Gradle比Maven快的前三的功能：
* 增量性 —— Gradle通过跟踪输入、输出的任务并只运行必要的内容，仅处理可能的情况下发生变化的文件来避免工作
* 构建缓存 —— ***Reuses the build outputs of any other Gradle build with the same inputs, including between machines.***(重用具有相同输入的任何其他Gradle构建的构建输出，包括在机器之间)
* Gradle守护程序 —— 一个长期存在的过程，将构建信息 ***“hot”*** 在内存中

# 用户体验
Maven出现时间更长，这意味着它对许多用户来说更支持IDE。可是，对Gradle的IDE支持，在持续快速地提升。例如，Gradle现在有个 ***Kotlin-based DSL*** ，它提供一个更好的IDE体验。Gradle的团队和IDE的开发商合作，以使编辑支持更加完善 ———— 时刻关注更新

虽然IDE很重要，但是大量的用户偏向于使用命令行来执行构建操作。Gradle提供一个现代的CLI(command-line interface),它具有发现功能(例如gradle tasks)，以及改进的日志记录和命令行的自动完成。
Gradle还提供了一个基于Web的交互式UI，用于调试和优化构建的工具：[bulid scans](https://gradle.com/build-scans?_ga=2.252124545.903271646.1584519569-1327398428.1583646788)

# 依赖管理
两个构建系统都提供了内置功能解决可配置仓库中的依赖项，两者都可以本地缓存依赖项和并行下载它们。

作为库的消费者，Maven允许重写一个依赖，但是只能通过版本号。Gradle提供可定制的 ***dependency selection*** (依赖选择)和 ***substitution rules*** (替代规则),这个可以一次声明并处理整个项目中不需要的依赖。替代机制会启用Gradle来一起构建多个源项目以创建“复合构建”

Maven有很少的内置 ***dependency scopes***(依赖范围)，在常见的情况下(比如 using test fixtures或者代码生成)，这会迫使笨拙的模块体系结构。例如，单元测试和集成测试之间没有分隔。Gradle允许自定义依赖范围，它提供了更好的建模和更快的构建。

Maven依赖冲突解决使用“最短路径”，它受声明顺序的影响。Gradle解决了所有冲突，选择最高版本的依赖在“图”中。另外，使用Gradle你可以严格地声明版本，这使它们拥有高于过渡版本的优先级，从而降低依赖项。

作为库的生产者，Gradle允许生产者声明 'api' and 'implementation' 'dependencies'来防止不需要的库渗透进消费者的类路径。Maven允许发布者通过可选的依赖项提供元数据，但仅作为文档提供。Gradle完全支持 ***feature variants and optional dependencies.***

# 总结
这个是自己的一些感悟，非原文。使用过Maven与Gradle的人最直观感受的就是Maven太过繁琐依赖项比Gradle，Gradle就是一个小清新。

---

记录一下，第一次翻译

ps：里面有些专业名词不太懂