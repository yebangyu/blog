---
layout: post
title: "利用Distcc和Dmucs构建大规模、分布式C++编译环境(上)"
date: 2015-11-23 22:48:32 +0800
comments: true
categories: 编译链接
keywords: Distcc, Dmucs, 分布式, C++, 编译
description: 本文介绍了利用Distcc和Dmucs构建大规模分布式C++编译环境
---

如果您的**C++**项目非常庞大，含有**1000**个**.h**文件，**2000**个**.cpp**文件，那么我敢打赌，每次编译所花的时间，都足够您喝**3000**杯咖啡了。如何加快编译速度？

**Distcc**是开源的用于搭建分布式编译环境的利器，它通过利用多台机器的资源，并行编译，来解决这个棘手的问题。然而，它的调度算法过于简单，不大合理，因此我们利用**DMUCS**提供的调度功能，来搭建一个相对完美的分布式编译平台。本文，我们首先介绍如何（单独）使用**Distcc**来加速编译，下一篇介绍如何组合使用**Distcc**+**DMUCS**来做进一步的完善和优化。

话不多说，我们的编译平台包括：

<!--more-->

开发机（**Client**）：这一台机器上有我们的项目、工程文件、代码，平时我们在这台机器上写代码，要编译的对象也在这台机器上。机器**IP**为**192.168.1.99**

服务器（**Server**）：负责编译的机器，一共有**3**台。**IP**分别为**192.168.1.11**,**192.168.1.22**,**192.168.1.33**.

调度器（**Scheduler**）：调度程序所在的机器，负责把编译任务合理的派发到编译机器（服务器）的编译程序上。**IP**为**192.168.1.88**。这台机器的部署和配置，我们留到下一篇博文介绍。

以上所有机器都是安装的**Ubuntu 14.04**操作系统。

## 服务器

1，安装**gcc**

    sudo apt-get install gcc

2，安装**Distcc**

    sudo apt-get install distcc

安装后可以得到两个二进制文件，**distccd**和**distcc**。前者主要负责网络数据处理，后者可以认为是**g++**的前端，调用**g++**进行编译。

3，配置**Distcc**

打开`/etc/default/distcc`，设置如下配置项：

    STARTDISTCC="true"  
    ALLOWEDNETS="127.0.0.1 192.168.1.0/24"
    LISTENER="0.0.0.0"

第一行设置开机就启动**distccd**

第二行设置允许利用本机进行编译的开发机

第三行设置监听的网络

4，启动**distccd**

    sudo service distcc start
    

经过以上配置，每次机器开机，都会自动运行**distccd**。

运行如下命令确认下：

    ps -aux | grep distccd

如果看到类似的输出，说明一切OK了。

    distccd   3457  0.0  0.0   3260   144 ?        SNs  23:33   0:00 /usr/bin/distccd --pid-file=/var/run/distccd.pid --log-file=/var/log/distccd.log --daemon --allow 127.0.0.1 --listen 0.0.0.0 --nice 10
    distccd   3458  0.0  0.0   3260   144 ?        SN   23:33   0:00 /usr/bin/distccd --pid-file=/var/run/distccd.pid --log-file=/var/log/distccd.log --daemon --allow 127.0.0.1 --listen 0.0.0.0 --nice 10
    distccd   3461  0.0  0.0   3260   144 ?        SN   23:33   0:00 /usr/bin/distccd --pid-file=/var/run/distccd.pid --log-file=/var/log/distccd.log --daemon --allow 127.0.0.1 --listen 0.0.0.0 --nice 10

## 开发机

1，安装**Distcc**

当然啦，我们假设您的开发机上已经安装**gcc**了。

2，设置编译资源

这一步是指定哪些机器（也就是上面的**Server**）来负责编译工作。

    export DISTCC_HOSTS="192.168.1.11 192.168.1.22 192.168.1.33"

每台机器的**IP**之间用空格隔开。

如果机器很多，那么这样填写可能不大方便，可以在`/etc/distcc/hosts`里添加。

这两种方法可以任选一种，如果您两种都用，那么**Distcc**只认**DISTCC_HOSTS**值。

3，应用**Distcc**

应用**Distcc**来编译代码有几种方法：

方法一：修改**makefile**中的`CXX`的值

将**makefile**文件中的这一行

    CXX = g++ 

改为

    CXX = distcc g++
    
然后运行`make -j18`即可。这里的**18**，为所有服务器的**CPU Cores**的数量乘以**1.5**。上面有3台服务器，每台有**4**核，因此这里设置为**18**。

为了获得最优值，很可能需要反复实验、测试。

方法二：修改**configure**文件

如果您的**makefile**文件由**automake**产生，那么在运行`./configure`时得加上参数，变为：

    ./configure --CXX=distcc g++

那么生成的**makefile**文件将自动使用**Distcc**了。

如果不出意外，编译速度已经大大提升了。然而，还有提高的空间，欲知详情，请看[下篇](http://www.yebangyu.org/blog/2015/11/30/build-distributed-compilation-env/)分解。
