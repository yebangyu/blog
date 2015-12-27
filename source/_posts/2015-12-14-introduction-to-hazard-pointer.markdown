---
layout: post
title: "Lock Free中的Hazard Pointer(中)"
date: 2015-12-14 22:33:01 +0800
comments: true
categories: 并行编程
keywords: Hazard Pointer,并行编程,对象生命周期,多线程,内存管理
description: 本文介绍了Hazard Pointer在多线程内存管理中的应用
---

看过[上篇](http://www.yebangyu.org/blog/2015/12/10/introduction-to-hazard-pointer/)的朋友，可能会认为：这不就是**Smart Pointer**么？于是可能写出这样的代码：

<!--more-->

```c++
#include<iostream>
#include<thread>
using namespace std;
class SmartPointer
{
public:
  SmartPointer(int *p)
  {
    pointee_ = p;
    ref_counts_ = 0;
  }
  int *pointee_;
  int ref_counts_;
};
SmartPointer sp(new int(1));
void Reader()
{
  ++sp.ref_counts_;
  cout<<*(sp.pointee_)<<endl;
  --sp.ref_counts_;
}
void Writer()
{
  if (sp.ref_counts_ == 0)
    delete sp.pointee_;
}
int main()
{
  thread t1(Reader);
  thread t2(Reader);
  thread t3(Reader);
  thread t4(Writer);
  t1.join();
  t2.join();
  t3.join();
  t4.join();
  return 0;
}
```
然而事实上，这样做是错的。其中的**race condition**请读者自行分析。

那么，**Hazard Pointer**(**HP**)和**Smart Pointer**(**SP**)有什么区别呢？它们的共同点就是管理对应对象的生命周期，然而这两者有本质的区别，**HP**是线程安全的，而**SP**不是。

在**HP**中，每个读线程维护着自己的**HP** **list**，这个**list**，只由该线程写。因此，它是线程安全的。该**list**会（可以）被其他线程读。

每个写线程维护自己的**retire list**，该**retire list**只由该写线程进行读写。由于写线程可以读其他所有读线程的**HP list**，这样，差集（在自己的**retire list**，但是不在所有读线程的**HP list**里的指针），就是可以安全释放的指针。

而**SP**，则被多个线程读写，**18、19**两行也无法做成原子操作，因此，**SP**和**HP**有本质的区别，使用**SP**的程序往往需要搭配使用锁等设施来保证线程安全。