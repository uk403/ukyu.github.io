---
title: Jenkins使用过程中遇到的麻烦
layout: post
date: 2020-04-01
tags: 
- config
- jenkins
categories:
- programmer
---
> Jenkins是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件。
> Jenkins 支持各种运行方式，可通过系统包、Docker 或者通过一个独立的 Java 程序。
[[Jenkins官网](https://jenkins.io/zh/doc/)对其的描述]
<!-- more -->

以下问题 **基于Docker安装的Jenkins**

~~安装这些就不在赘述了，Google一大把~~
## Jenkins更换密码
[更换密码](https://www.kancloud.cn/louis1986/jenkins/481900)

## Jenkins下载超时
下个插件大多数都是fail，下载成功的插件也要花一些时间，才可以下好。

把Update Site改为：清华Jenkins镜像 `https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json`

重启一下，Jenkins 在地址后面+/restart


## Jenkins时间与服务器不一致
[解决方案](https://blog.csdn.net/qq_40168110/article/details/90755684)

## 使用Docker安装Jenkins的注意事项！！！
1. 我用的是docker上的官方Jenkins仓库，下载插件时，它会报“版本太低了”的问题。主要原因是Jenkins的仓库被废弃了 [详情](https://jenkins.io/zh/blog/2018/12/10/the-official-Docker-image/)。只需要改成[正确的仓库](https://hub.docker.com/r/jenkins/jenkins/ )就可以了。
   
2. 出现问题: ***"/usr/share/gradles is not a directory on the Jenkins master (but perhaps it exists on some agents)"***   ![未映射到容器内](/assets/images/blog/2020-04-01-Jenkins-Config/MapFile.png)此时需要你把会用到的宿主文件映射到容器内

3. 问题：***permission deny***![权限拒绝](/assets/images/blog/2020-04-01-Jenkins-Config/permission.png)
   方法：
   1. 启动Jenkins时加  `-u root`
   2. `chmod 777` Jenkins的主文件 (没试过，不知可行性，也不推荐用) 
   3. 创建一个'Dockerfile',将用户指定为root,再运行 `docker build -t jenkins -f jenkins_docker_file .`(-f [你创建文件的名字，若文件名字就是Dockerfile就忽略])
   
---
以上解决方案不一定是最佳方案

总的来说，体验了Jenkins的'一站式服务'。![一站式服务](/assets/images/blog/2020-04-01-Jenkins-Config/UsePurpose.png)

有个问题待解决，我使用的是云服务器的Docker拉取的Jenkins镜像。我想运行，Gradle构建的jar文件，我是通过Jenkins插件 ***Publish Over SSH***，将jar文件传到云服务器(同一个)的某个路径并运行它。速度太慢，而且本来是**同一个云服务器**，应该用不着SSH。**不知有什么好的办法，望告知，谢谢**

