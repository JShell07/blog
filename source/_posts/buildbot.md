---
title: buildbot
date: 2019-07-02 10:25:56
tags:
    - buildbot
    - openwrt
    - CI
categories:
    - CI
---

在openwrt 官网上看到buildbot， 他以此来完成在pull 代码之后，自动编译的工作并产出报告或者邮件通知等。
[buildbot官网](http://buildbot.net/)

持续集成（Continuous Integration，CI）的主要优势是，能够通过软件的自动化构建以及测试和软件度量标准（可选）精简品质保证周期。每次更改源代码并为项目生命期提供即时反馈和报告。

<!--more-->

### 1. buildbot 简介  
BuildBot是一个开源的基于python的持续集成系统，BuildBot用python写的，该python程序只依赖python环境和Twisted（一个python网络框架），可以在很多平台运行。 自动化构建一般包括自动下载源码，编译，测试，打包。他的工作原理可以参见下图:  
![](http://buildbot.net/img/overview.png)

Buildbot的原理是git，SVN等源码服务器上代码发生变化后，buildmaster（服务端）通知连接到它上的buildslave（客户端）从git或SVN服务器上自动下载源码，编译，测试，打包。最后把各个buildslave的自动化构建的结果搜集起来在web上展现，或通过email,IRC等方式通知相应的项目开发人员。

BuildBot的常用架构是一个Master和一堆Slave，Master负责对接VCS，然后管理调度各个Slave各司其职，收集Slave传回来的数据并且整理成报告。Slave负责按照Master发过来的命令跑各种任务，并将环境信息，结果，log文件等收集起来报告给Master。
- master  master就是Buildbot的核心，我们使用Buildbot所需要做的各种工作也是在Master上进行。Buildbot的使用方式就是在Master上编辑master.cfg文件，这其实是一个Python文件。使用者在里面定义对接的VCS，Schedule和Build的各种条件以及具体的Build任务，结果的收集报告方式等。
- slave 当slave连接到master后就会不断跟master进行通信。当有任务时，master会将命令逐个发送给slave执行。

### 2. Openwrt buildbot
我们可以访问如下链接， 可以看到openwrt 的buildbot 的情况。
- Phase 1: [target/subtargets](https://phase1.builds.lede-project.org/builders)  
- Phase 2: [packages](https://phase2.builds.lede-project.org/builders)  

![](https://raw.githubusercontent.com/JShell07/images/master/buildbot/waterfall.png)

![](https://raw.githubusercontent.com/JShell07/images/master/buildbot/grid.png)

![](https://raw.githubusercontent.com/JShell07/images/master/buildbot/console%20view.png)

### 3. 其他常见持续集成工具  
常见的持续集成工具有：  
- [Jenkins](https://jenkins.io/zh/) （java 方向）  
- [Hudson](http://hudson-ci.org/) (java 方向，jenkins 的前身)  
- [strider](http://stridercd.com/)  
- [travis](https://travis-ci.com/) （对开源项目免费）  
- [codeship](https://codeship.com/)（对开源项目免费）  

部署工具(将这个版本的所有文件打包（ tar filename.tar * ）存档，发到生产服务器)  
- [ansible](https://www.ansible.com/)  
- [chef](https://www.chef.io/products/chef-infra/)  
- [puppet](https://puppet.com/)  
  
### 参看资料   
[Buildbot Tutorial](https://docs.buildbot.net/current/tutorial/)  
[Buildbot初探](https://www.cnblogs.com/lkiversonlk/p/4878129.html)   
[使用 Buildot 实现持续集成](https://www.ibm.com/developerworks/cn/linux/l-buildbot/index.html)   
[buildbot自动化测试工具安装及快速入门](https://blog.csdn.net/LSMEGR/article/details/53618045)  
[持续集成的魅力：工具推荐](https://www.cnblogs.com/xing901022/p/4414263.html)  
[六款不容错过的开源持续集成工具](http://cloud.51cto.com/art/201508/487605.htm)  
[持续集成是什么？](http://www.ruanyifeng.com/blog/2015/09/continuous-integration.html)  