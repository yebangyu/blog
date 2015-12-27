---
layout: post
title: "Tools of the trade"
date: 2015-10-25 22:09:08 +0800
comments: true
categories: 并行编程
tags: [并行编程, 多进程, 多线程]
keywords: 高并发, 读写锁, 多进程, 多线程
description: 多进程，多线程，原子操作
---

本篇是对《**Is parallel programming hard**》第四章《**Tools of the trade**》的总结，不是单纯的翻译，算是读书笔记，并且略有补充。

本章介绍了并行编程的工具和途径，具体包括

> * Shell Script Languages
> * POSIX 多进程
> * POSIX 多线程
> * 原子操作

## Shell Script Language

<!--more-->

如果不同的执行单元之间没有过多的数据交互，待执行的任务分区性较好，那么我们可以考虑通过**Shell**创建多个进程来完成任务。例如，我们需要计算每连续的**100**个元素的和，需要计算3组，好吧，比如说我们需要计算**1+2+3+...+100**；**101+102+...200**；**201+201+...+300**，那么我们可以编写一个程序，然后通过**Shell**创建**3**个进程，通过命令行传入参数（比如这里的待求和的元素的起点）。

    compute 1 > compute_1.out &
    compute 101 > compute_2.out &
    compute 201 > compute_3.out &
    wait

其中**compute**是可执行程序名，**1**、**101**、**201**是命令行参数，**> x.out**代表将输出结果重定向到文件**x.out**中，**&**表示程序后台运行。**wait**表示等待运行程序结束。

那么多进程的并行设计有什么缺点呢？

1，创建进程的时间略长。在**Intel Core Duo Laptop**上创建一个最小**C**程序需要大概**480ms**。当你的任务执行时间和进程启动时间相比反而不值一提时，这时候创建进程所需的时间就显得很尴尬。多线程**VS**多进程也是**Spark**和**Hadoop**相比的一个不同。

2，进程间不共享内存，不利于通信和数据交互。

3，多进程间的同步相对费事复杂。

## POSIX 多进程

可以通过**fork**、**kill**、**wait**等原语来创建、管理进程。书里简单介绍了这几个原语的使用，小结一下就是：

1，**fork**会有两次返回，一次对**child process**，一次对**parent process**。**fork**的返回值为**0**代表在**child process**的上下文中，负数代表错误，正数代表**parent process**上下文中，并且返回值就是**child process**的**pid**。

2，**parent process**和**child process**并不**share memory**。

## POSIX 多线程

可以通过**pthread_mutex_lock**以及**pthread_mutex_unlock**等原语，以加锁和释放锁的方式，使用多线程来并行设计。

锁有多种，除了互斥锁，读写锁也是常见的一种。读写锁的特点是：

1，同一个时刻，允许多个读线程。当然，此时不能有写线程。

2，同一个时刻，最多只能有一个写线程进行更新操作。

也就是说写写互斥，写读互斥，读读不互斥。换句话说，要么多个读线程，要么一个写线程。

那么读写锁的**scalability**如何呢？作者写了一个程序来分析，程序运行在**64**个**cores**，一共**n**个读线程，每个读线程的流程大概是：

    while(not terminated)
    {
      acquire the lock;
      do something;//t1
      release the lock;
      wait some time;//t2
      ++count of acquisitions;
    }


把其中**t2**的时间设置为**0**，**t1**的控制则是通过更改调整执行循环次数（图上所谓的**100K**、**100M**神马的）。图上的横坐标为线程数目，纵坐标代表 $\frac {C}{nc}$ ，其中**C**是**n**个线程总共的**acquisition**数目，**c**是单个线程**acquisition**数目。

理想情况下，这个值应该是**1.0**。

![expriments for rwl](http://7xnljs.com1.z0.glb.clouddn.com/figure4.10.jpg)

实验表明，当读线程的数目增多，每次**acquire lock**时，花在修改数据结构（锁也是一种数据结构实现，当一个读线程**acquire**或者**release**成功显然需要对数据结构进行修改，加加减减神马的）的时间将显著影响**scalability**。极端情况下，当**n**个读线程同时**acquire**时，第**n**个线程需要等前面的**n-1**个线程都修改完毕，它才能修改。

同时，注意到线程有**n**个，而**CPU** **cores**只有**64**个，因此当**n<=64**时，每个**thread**可以独享一个**core**，当**n>64**后，根据鸽巢原理，至少有一个**core**上有多个**thread**在运行，这也会带来性能下降。

因此，读写锁比较适合临界区比较大的情形（有文件**IO**或者网络访问等）。

如果临界区比较短呢？比如我仅仅是加加一个变量呢？哦，那么原子操作可能是一个很好的选择。

## 原子操作

**gcc**内置提供了一系列的原子操作。很多操作有两个版本，比如说：

`__sync_fetch_and_sub()`与 `__sync_sub_and_fetch()`，如名字所说，一个是先减，然后获得减之后的新值；一个是减，返回的是减之前的**old value**。

此外，非常有名的**CAS**操作：

`__sync_bool_compare_and_swap()`和`__sync_val_compare_and_swap()`，前者返回是否操作成功（待修改变量被替换为新值），而后者返回的是老值。

由于原子操作是让多个步骤看起来是一次的行为，因此往往包含**Memory Barrier**以保证语句的执行顺序。