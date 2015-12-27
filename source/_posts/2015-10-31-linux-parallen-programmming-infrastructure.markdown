---
layout: post
title: "Linux环境多线程编程基础设施"
date: 2015-10-31 18:43:00 +0800
comments: true
categories: 并行编程
tags: [并行编程, 多线程, 并发]
keywords: 多线程, volatile, memory barrier, __sync_synchronize
description: linux下多线程编程基础设施
---

本文介绍多线程环境下并行编程的基础设施。主要包括：
> * volatile
> * __thread
> * Memory Barrier
> * __sync_synchronize

## volatile

编译器有时候为了优化性能，会将一些变量的值缓存到寄存器中，因此如果编译器发现该变量的值没有改变的话，将从寄存器里读出该值，这样可以避免内存访问。

但是这种做法有时候会有问题。如果该变量确实（以某种很难检测的方式）被修改呢？那岂不是读到错的值？是的。在多线程情况下，问题更为突出：当某个线程对一个内存单元进行修改后，其他线程如果从寄存器里读取该变量可能读到老值，未更新的值，错误的值，不新鲜的值。

<!--more-->

如何防止这样错误的“优化”？方法就是给变量加上**volatile**修饰。

    volatile int i=10;//用volatile修饰变量i
    ......//something happened 
    int b = i;//强制从内存中读取实时的i的值

**OK**，毕竟**volatile**不是完美的，它也在某种程度上限制了优化。有时候是不是有这样的需求：我要你立即实时读取数据的时候，你就访问内存，别优化；否则，你该优化还是优化你的。能做到吗？

不加**volatile**修饰，那么就做不到前面一点。加了**volatile**，后面这一方面就无从谈起，怎么办？伤脑筋。

其实我们可以这样：

    int i = 2; //变量i还是不用加volatile修饰
    
    #define ACCESS_ONCE(x) (* (volatile typeof(x) *) &(x))

需要实时读取i的值时候，就调用`ACCESS_ONCE(i)`，否则直接使用i即可。

这个技巧，我是从《**Is parallel programming hard？**》上学到的。

听起来都很好？然而险象环生：**volatile**常被误用，很多人往往不知道或者忽略它的两个特点：在**C/C++**语言里，**volatile**不保证原子性；使用**volatile**不应该对它有任何**Memory Barrier**的期待。

第一点比较好理解，对于第二点，我们来看一个很经典的例子：

```c++
volatile int is_ready = 0;
char message[123];
void thread_A
{
  while(is_ready == 0) 
  {
  }
  //use message;
}
void thread_B
{
  strcpy(message,"everything seems ok");
  is_ready = 1;
}

```
线程**B**中，虽然**is_ready**有**volatile**修饰，但是这里的**volatile**不提供任何**Memory Barrier**，因此**12**行和**13**行可能被乱序执行，`is_ready = 1`被执行，而**message**还未被正确设置，导致线程**A**读到错误的值。

这意味着，在多线程中使用**volatile**需要非常谨慎、小心。

## __thread

**__thread**是**gcc**内置的用于多线程编程的基础设施。用**__thread**修饰的变量，每个线程都拥有一份实体，相互独立，互不干扰。举个例子：

```c++
#include<iostream>  
#include<pthread.h>  
#include<unistd.h>  
using namespace std;  
__thread int i = 1;
void* thread1(void* arg);  
void* thread2(void* arg);  
int main()
{  
  pthread_t pthread1;
  pthread_t pthread2;  
  pthread_create(&pthread1, NULL, thread1, NULL);
  pthread_create(&pthread2, NULL, thread2, NULL);
  pthread_join(pthread1, NULL);  
  pthread_join(pthread2, NULL);  
  return 0;  
}  
void* thread1(void* arg)
{  
  cout<<++i<<endl;//输出 2  
  return NULL;
}  
void* thread2(void* arg)
{  
  sleep(1); //等待thread1完成更新
  cout<<++i<<endl;//输出 2，而不是3
  return NULL;
} 
```

需要注意的是：

1，**__thread**可以修饰全局变量、函数的静态变量，但是无法修饰函数的局部变量。

2，被**__thread**修饰的变量只能在编译期初始化，且只能通过常量表达式来初始化。

## Memory Barrier

为了优化，现代编译器和**CPU**可能会乱序执行指令。例如：

```c++
int a = 1;
int b = 2;
a = b + 3;
b = 10;
```

**CPU**乱序执行后，第4行语句和第5行语句的执行顺序可能变为先`b=10`然后再`a=b+3` 

有些人可能会说，那结果不就不对了吗？b为10，a为13？可是正确结果应该是a为5啊。

哦，这里说的是语句的执行，对应的汇编指令不是简单的mov b,10和mov b,a+3。

生成的汇编代码可能是：

```
movl    b(%rip), %eax ; 将b的值暂存入%eax
movl    $10, b(%rip) ; b = 10
addl    $3, %eax ; %eax加3
movl    %eax, a(%rip) ; 将%eax也就是b+3的值写入a,即 a = b + 3
```

这并不奇怪，为了优化性能，有时候确实可以这么做。但是在多线程并行编程中，有时候乱序就会出问题。

一个最典型的例子是用锁保护临界区。如果临界区的代码被拉到加锁前或者释放锁之后执行，那么将导致不明确的结果，往往让人不开心的结果。

还有，比如随意将读数据和写数据乱序，那么本来是先读后写，变成先写后读就导致后面读到了脏的数据。因此，**Memory Barrier**就是用来防止乱序执行的。具体说来，**Memory Barrier**包括三种：

1，**acquire barrier**。**acquire barrier**之后的指令不能也不会被拉到该**acquire barrier**之前执行。

2，**release barrier**。**release barrier**之前的指令不能也不会被拉到该**release barrier**之后执行。

3，**full barrier**。以上两种的合集。

所以，很容易知道，加锁，也就是**lock**对应**acquire barrier**；释放锁，也就是**unlock**对应**release barrier**。哦，那么**full barrier**呢？


## __sync_synchronize

`__sync_synchronize`就是一种**full barrier**。