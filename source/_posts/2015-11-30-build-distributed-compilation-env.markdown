---
layout: post
title: "利用Distcc和Dmucs构建大规模、分布式C++编译环境(下)"
date: 2015-11-30 22:23:46 +0800
comments: true
categories: 编译链接
keywords: Distcc, Dmucs, 分布式, C++, 编译
description: 本文介绍了利用Distcc和Dmucs构建大规模分布式C++编译环境
---

[上篇](http://www.yebangyu.org/blog/2015/11/23/build-distributed-compilation-ev/)文章，我们介绍了如何利用**Distcc**来搭建分布式编译环境，但是**Distcc**的默认调度策略过于简单，并且并不合理。假如我们的配置是

    export DISTCC_HOSTS="192.168.1.11 192.168.1.22 192.168.1.33"

那么**Distcc**会根据**DISTCC_HOSTS**中机器出现的先后顺序，来安排编译任务，越靠前的机器(比如这里的**192.168.1.11**)获得越多的任务，这显然是不科学的。

因此，我们可以利用**Dmucs**提供的调度策略，来优化我们的方案。它可以根据编译机的负载情况和硬件实力，来合理的调度资源。能者多劳嘛。

为了保证本文的完整性，我们还是不厌其烦地把我们的环境再交待下：

<!--more-->

开发机（**Client**）：这一台机器上有我们的项目、工程文件、代码，平时我们在这台机器上写代码，要编译的对象也在这台机器上。机器**IP**为**192.168.1.99**。

服务器（**Server**）：负责编译的机器，一共有**3**台。**IP**分别为**192.168.1.11**,**192.168.1.22**,**192.168.1.33**。

调度器（**Scheduler**）：调度程序**Dmucs**所在的机器，负责把编译任务合理的派发到编译机器（服务器）的编译程序上。**IP**为**192.168.1.88**。

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

如果看到类似的输出，说明**distccd**成功启动了。

    distccd   3457  0.0  0.0   3260   144 ?        SNs  23:33   0:00 /usr/bin/distccd --pid-file=/var/run/distccd.pid --log-file=/var/log/distccd.log --daemon --allow 127.0.0.1 --listen 0.0.0.0 --nice 10
    distccd   3458  0.0  0.0   3260   144 ?        SN   23:33   0:00 /usr/bin/distccd --pid-file=/var/run/distccd.pid --log-file=/var/log/distccd.log --daemon --allow 127.0.0.1 --listen 0.0.0.0 --nice 10
    distccd   3461  0.0  0.0   3260   144 ?        SN   23:33   0:00 /usr/bin/distccd --pid-file=/var/run/distccd.pid --log-file=/var/log/distccd.log --daemon --allow 127.0.0.1 --listen 0.0.0.0 --nice 10

5，安装**Dmucs**

首先，从[这里](http://sourceforge.net/projects/dmucs/files/dmucs/dmucs%200.6.1/)下载最新版本的**Dmucs**。

安装也就是普通的`configure`、`make`和`make install`，需要注意的是，执行**make**时，需要加上参数，例如：

    make CPPFLAGS=-DSERVER_MACH_NAME=\\\"192.168.1.88\\\"

其中**192.168.1.88**是我们的调度器的机器**IP**，根据实际情况自行修改。服务器上的编译程序，需要把编译负载情况，发送给调度器上的调度程序，作为它调度时的参考信息。

如果您嫌这样麻烦，那么也可以在**make**的时候不指定选项，而是在安装后，打开`/etc/default/dmucs`文件，设置如下字段：

    USE_SERVER = 192.168.1.88

6，启动**loadavg**

**loadavg**是安装**Dmucs**后得到的一个二进制文件，它会定期发送服务器平均负载情况给调度器，供调度程序决策之用。

    sudo loadavg -s 192.168.1.88 &

其中，**192.168.1.88**是调度程序所在机器的**IP**地址。

## 调度器

1，安装**Dmucs**

请参考上面的第**5**步。

2，配置服务器属性

在调度器的`/usr/local/share/dmucs/`目录下，新建**hosts-info**文件，格式和内容如下：

    #Format: machine number-of-cpus power-index
    192.168.1.11       4            10
    192.168.1.22       4            5
    192.168.1.33       4            5

其中，**power-index**（必须大于等于**1**）表示了这台机器的战斗力，值越高代表性能越强，分到的编译任务也越多。能者多劳嘛。

3，启动调度程序

命令为：

    sudo service dmucs start
    
## 开发机

1，安装**Distcc**

当然啦，我们假设您的开发机上已经安装**gcc**了。

2，设置编译资源

这一步是指定哪些机器（也就是上面的**Server**）来负责编译工作。

    export DISTCC_HOSTS="192.168.1.11 192.168.1.22 192.168.1.33"

每台机器的**IP**之间用空格隔开。

如果机器很多，那么这样填写可能不大方便，可以在`/etc/distcc/hosts`里添加。

这两种方法可以任选一种，如果您两种都用，那么**Distcc**只认**DISTCC_HOSTS**值。

3，应用**Distcc**和**Dmucs**

应用**Distcc**和**Dmucs**来编译代码有几种方法：

方法一：修改**makefile**中的**CXX**的值

将**makefile**文件中的这一行

    CXX = g++

改为

    CXX = gethost distcc g++

然后运行`make -j18`即可。这里的**18**，为所有服务器的**CPU Cores**的数量乘以**1.5**。上面有**3**台服务器，每台有**4**核，因此这里设置为**18**。

为了获得最优值，很可能需要反复实验、测试。

方法二：修改**configure**文件

如果您的**makefile**文件由**automake**产生，那么在运行`./configure`时得加上参数，变为：

    ./configure --CXX=gethost distcc g++

那么生成的**makefile**文件将自动使用**Distcc**和**Dmucs**了。

## 附录：调度策略

根据[文档](http://dmucs.sourceforge.net/)，**Dmucs**的调度策略是：

1，**Dmucs**会维护几个列表（原文为**tier**，表示等级、阶梯），每个列表对应一个**power-index**，具有相同**power-index**值的机器会在同一个列表中。因此，上面三台服务器，**192.168.1.22**和**192.168.1.33**会在同一个列表里，**192.168.1.11**在一个列表里。

2，每次获得编译任务请求时，**Dmucs**会优先从高**power-index**对应的列表里随机选一台机器。

3，如果**Dmucs**获知某台机器的平均负载过高（回忆下，该信息是服务器上的**loadavg**进程发过来的），那么会把该机器放到低一级的**power-index**对应的列表里。例如，**192.168.1.11**会被“下放”到**power-index**为**5**对应的列表里。

4，如果某台机器的平均负载过高超过五分钟，那么该机器会被暂时雪藏，不会给它分发编译任务，直到它的负载下降到正常水平。