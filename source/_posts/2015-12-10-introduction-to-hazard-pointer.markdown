---
layout: post
title: "Lock Free中的Hazard Pointer(上)"
date: 2015-12-10 23:00:20 +0800
comments: true
categories: 并行编程
keywords: Hazard Pointer,并行编程,对象生命周期,多线程,内存管理
description: 本文介绍了Hazard Pointer在多线程内存管理中的应用
---

废话不多说了，直接开始讨论。

##险象环生

首先看以下的程序，有问题吗？

<!--more-->

```c++
#include<iostream>
#include<thread> //C++11中的多线程基础设施
using namespace std;
int *p = new int(2);
void reader()
{
  if (nullptr != p) { //nullptr是C++11中引入的
    cout << *p << endl;
  }
}
void writer()
{
  delete p;
  p = nullptr;
}
int main()
{
  thread t1(reader);
  thread t2(writer);
  t1.join();
  t2.join();
  return 0;
}
```
答案当然是有问题。这个程序存在如下的**race** **condition**：如果当线程**reader**判断完**p**发现它不是**nullptr**后，还未执行第**8**行就被调度出去，轮到线程**writer**执行，它执行完**13**、**14**行后，又继续调度线程**reader**，此时执行第**8**行导致程序崩溃。

这里的问题，可以不精确地表述为：多线程程序中，某线程通过一个指针访问一段内存时，如何保证指针所指向的那块内存是有效的？

##Hazard Pointer

这当然有多种方法，加一把互斥锁就万事大吉了(当然，得注意锁本身的生命周期)。不过本文讨论的是**lock** **free**情况下的内存管理，这里要介绍的是**2004**年左右提出的**Hazard** **Pointer**方法。

接着看以下代码：

```c++
#include<thread>
#include<iostream>
#define Strong_Memory_Barrier __sync_synchronize
#define ACCESS_ONCE(x) (* (volatile typeof(x) *) &(x))
using namespace std;
struct HazardPointer
{
  HazardPointer() :p_(nullptr){}
  int *p_; //Hazard Pointer封装了原始指针
};

int *p = new int(2); //原始指针

HazardPointer *hp = new HazardPointer(); //只有一个线程可以写

class Writer
{
public:
  void retire(int *tmp) {
    retire_list = tmp;//设置要释放的指针
  }
  void gc() {
    if (retire_list == hp->p_) {
      //说明有读者正在使用它，不能释放
    } else {
      //可以安全地释放
      delete retire_list;
    }
  }
  void write() {
    int *tmp = ACCESS_ONCE(p);
    p = nullptr;
    retire(tmp);
    gc();
  }
private:
  int *retire_list;//记录待释放的指针
};
class Reader
{
public:
  void acquire(int *tmp) {
    hp->p_ = tmp;
  }
  void release() {
    hp->p_ = nullptr;
  }
  void read() {
    int *tmp = nullptr;
    do
    {
      tmp = ACCESS_ONCE(p);
      Strong_Memory_Barrier();
      acquire(tmp); //封装。这是告诉Writer：我要读，别释放！
    } while (tmp != ACCESS_ONCE(p));//仔细想想，为什么这里还要判断？
    if (nullptr != tmp) { //不妨想想，这里为什么也要判断？
      //it is safe to access *tmp from now on
      cout << *tmp << endl;//没问题~
    }
    //when you finish reading it, just release it .
    release();//其实就是告诉Writer：用完了，可以释放了。
  }
};
int main()
{
  thread t1(&Reader::read, Reader());
  thread t2(&Writer::write, Writer());
  t1.join();
  t2.join();
  return 0;
}
```

总结：

1，对于读线程，每次访问指针前，都通过**acquire**动作，用**Hazard** **Pointer**封装一下待读取指针，此举保护了原始指针，向其他线程宣告，我正在读取它指向的内存，请别释放它。用完就**release**一下。

2，对于写线程，真正释放前，需要检查是否有读线程正在操作它。如何知道？看看自己待释放的指针，有没有存在在读线程的**Hazard** **Pointer**里即可。

至于**52**行，考虑如下情形：读线程刚刚设置了**tmp**指针，还没对它进行保护，就被调度出去；写线程执行**gc**时，发现没有读线程的**Hazard** **Pointer**封装了**tmp**指针，于是将它释放；等读线程重新被调度执行时通过**tmp**进行内存访问，就会导致问题。

至于**53**行，请读者自行分析。

##最后思考

那么，**Hazard** **Pointer**的内存空间，谁来负责管理？

在实际程序里，一般所需**Hazard** **Pointer**的数量不会太多且较为固定，因此基本上只申请，不释放了。

##参考文献

https://www.research.ibm.com/people/m/michael/ieeetpds-2004.pdf

##特别感谢

感谢[韩富晟](http://weibo.com/hanfooo)先生对我的指点，让我加深了对**Hazard** **Pointer**的认识。