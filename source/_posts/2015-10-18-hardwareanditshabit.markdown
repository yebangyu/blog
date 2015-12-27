---
layout: post
title: "Hardware and its habit"
date: 2015-10-18 12:29:44 +0800
comments: true
categories: 并行编程
tags: [并行编程, CPU, 性能]
keywords: 并行编程, CPU, 性能, cache一致性
description: 并行编程影响cpu性能的几个因素
---

最近在阅读《**Is parallel programming hard**》这本书，本篇就是整理其中第三章《**Hardware and its habit**》，不是单纯的翻译，只是一个总结，略有补充。

这章介绍了影响**CPU**执行效率的几个因素。具体包括：

> * 流水线被打断
> * 内存访问
> * 原子操作
> * Memory Barrier
> * Cache Misses
> * IO 操作

<!--more-->


这其中，前面两个，流水线被打断以及内存访问主要针对串行程序，而后面四个主要针对并行程序，因为在并行程序中显得更为突出。

### 流水线被打断

现代**CPU**执行指令采用流水线设计和指令预取机制，而影响流水的两种重要情况是停机等待和分支判断失败。前者是**CPU**没有足够的信息来判断取哪些指令（例如，涉及到**C++**中的虚函数时）。而分支判断失败，则是取了指令但是没取对。例如

    int a = get();
    if (a == 1)
    {
      //A
    }
    else
    {
      //B
    }

假设**CPU**预取指令**A**。当预测失败时（**a**不等于**1**），流水线中**A**指令需要被冲刷（**flush**），继而载入**B**指令。冲刷流水线和载入**B**指令都是非常昂贵的操作，因此这深深地影响了效率。

因此，在实际编程时，应该将最有可能执行的代码放到最前面。在**gcc**中内置提供了**likely**和**unlikely**宏，来优化程序分支判断。

    #define  likely(x)        __builtin_expect(!!(x), 1) 
    #define  unlikely(x)      __builtin_expect(!!(x), 0) 
    
因此，上面的程序可以改写为：

    int a = get();
    if (unlikely(a == 1)) //根据实际情况选择unlikely或者likely
    {
      //A
    }
    else
    {
      //B
    }

### 内存访问

这个不用说了，内存访问是昂贵操作，相对于寄存器、**cache**而言。

在上世纪的系统中，从内存中读一个值的速度要比**CPU**执行一条指令快速。后来，由于**CPU**的快速发展以及内存容量的增大，这种局面发生了改变。你能想象只有**4KB**内存的机器吗？现在，光是**cache**都不止**4KB**了。

数组通常有比较好的内存访问模式，也就是说访问了**a[0]**，就可以将**a[1]**,**a[2]**,**a[3]**等存进**cache**，等访问到**a[1]**时不需要去访问内存了。但是一般用指针实现的链表的访问模式则比较差。恩，所谓的空间局部性。



### 原子操作

**gcc**内置提供了一系列的原子操作，包括著名的用于**CAS**(**compare** **and** **swap**)的**__sync_bool_compare_and_swap**等。当多个线程对一个内存变量进行原子操作时，往往依赖于硬件支持。在**x86**下，原子操作时，锁住总线，防止其他**cpu** **core**访问该内存单元。

### Memory Barrier

**CPU**对指令可能采取乱序执行，以达到优化的目的。但是，并发访问的锁破坏了这种机制。

    c = 3;
    lock.lock();
    a = 1;
    b = 2;
    lock.unlock();
    d = 4;
	

`d=4`绝对不会在`a=1`之前执行，`c=3`绝对不会在`a=1`之后执行。

**lock**和**unlock**中包含了**memory** **barrier**。由于**memory** **barrier**和乱序执行是对着干的，用来防止乱序执行的；而乱序执行一般是优化的手段和方法，因此**memory** **barrier**往往带来性能下降。

### Cache Misses

先贴一张现代**CPU**和**cache**架构粗略图。

![cmd-markdown-logo](http://7xnljs.com1.z0.glb.clouddn.com/cpuand%20cache.png)

多个**CPU** **core**，一个内存。**cacheline**是**cache**块单位，一般在**32**到**256**字节左右。**cacheline**是这张图中不同模块的数据交互元素。

每两个**cpu** **core**和**Interconnect**组成一个**die**，同一个**die**中的**cpu** **core**通过**Interconnect**来沟通。不同**die**里的**cpu** **core**通过**System** **Interconnect**来沟通。

某个**core**需要对内存变量进行修改时，该变量的**cacheline**如果位于别的**core**的**cache**里，这种情况下的**cache miss**代价很大。

书中举了一个相对简单的例子：**cpu 0**需要对一个变量进行**cas**操作，检查自己的**cache**，发现没有。这时候：

1，**cpu 0**发送请求给**Interconnect**(**cpu 0 & cpu 1**)，后者检查**cpu 1**的**cache**，发现木有。

2，**Interconnect**(**cpu 0 & cpu 1**)将请求发给**System** **Interconnect**，后者检查其他的**3**个**die**，得知**cacheline**位于由**cpu 6**和**cpu 7** 组成的那个**die**里。

3，请求被发给由**cpu 6** 和**cpu 7**组成的那个**die**里的**Interconnect**(**cpu 6** **&** **cpu 7**)，它同时检查**cpu 6**和**cpu 7**的**cache**，得知**cacheline**位于**cpu 7**的**cache**里。

4，**cpu 7** 把**cacheline**发送给**Interconnect**(**cpu 6 & cpu 7**), **and flushes the cacheline from its cache**，以保证**cache**一致性

5，**Interconnect**(**cpu 6 & cpu 7**)将**cacheline**发送给**System** **Interconnect**。

6，**System** **Interconnect**将**cacheline**发送给**Interconnect**(**cpu 0 & cpu 1**)

7，**Interconnect**（**cpu 0 & cpu 1**）将**cacheline**存入**cpu 0**的**cache**里。

是啊，这已经是简单的情况了。想想看，什么情况下更复杂？
