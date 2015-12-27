---
layout: post
title: "空指针(NULL)不能用吗？"
date: 2015-11-06 22:38:30 +0800
comments: true
categories: c++
keywords: C++, 空指针, NULL, 对象内存布局
description: 介绍了空指针使用的注意事项；介绍了NULL、C++对象模型、内存布局和程序段
---

我们常常被告知，使用指针前需要判断是否为**NULL**；如果是**NULL**而你去使用它就会出问题。真相果真是这样吗？

同事颜廷帅（微博：@颜挺帅）给我看以下一个程序，问我，这段程序执行后，有问题吗？

<!--more-->

```c++
#include<iostream>
#include<cstdlib>
using namespace std;
class Test1
{
private:
  int a;
public:
  void f()
  {
    cout<<"Test1: Core or not ? "<<endl;
  }
};
int main()
{
  Test1 *p = NULL;
  p->f();//会core吗？会出大事吗？
  return 0;
}
```

这里，**p**是一个空指针，通过这个空指针，我们访问了函数**f**。没**core**，没问题，成功输出了 **Test1: Core or not ?**

发生了什么事？空指针也能用？

如果我们把**f**稍作修改，程序其他地方不做任何变动：

    void f()
    {
      cout<<"Test1: Core or not ? "<<a<<endl;//access a
    }

那么程序运行后分分钟**core**掉了。

有没有感觉了？嗯，相信有了。我们继续往下看：

```c++
#include<iostream>
#include<cstdlib>
using namespace std;
static int global = 1;
class Test2
{
private:
  int a;
public:
  void f()
  {
    cout<<"Test1: Core or not ? "<<global<<endl;//access global
  }
};
int main()
{
  Test2 *p = NULL;
  p->f();//会core吗？会出大事吗？
  return 0;
}
```

也没问题。

嗯，你可能已经知道了真相。通过空指针访问东西，只要那个东西是确实存在的，就不会有问题。

怎么理解“确实存在”？它是一个实体，看得见，摸得着。

这得说到**C++**程序中对象的内存布局。

在**C++**中，成员函数、静态变量是独立于对象存放的；而普通的数据成员是和对象相关的。

    Test1 obj1;
    Test1 obj2;

**obj1**和**obj2**是共用函数**f**的，函数**f**对**obj1**和**obj2**是相同的，内存中只有一份实体；而**obj1**和**obj2**有自己的实体**a**。

然而，注意到`Test1 *p = NULL;`仅仅是声明式，而非定义式。这时候，没有定义任何的对象出来，通过**p**如何访问**a**呢？哪来的**a**呢？**a**在内存里并不存在。因此，访问**a**必定**core**。

而函数**f**呢？它是独立于对象存放的，自然没问题。一般说来，**f**位于程序的代码段，而全局变量一般位于**BSS**段或者**DATA**段（这个比较复杂，和该全局变量是否初始化以及初始化为**0**还是非**0**有关）。而当我们定义对象时，才为该对象分配内存，才有数据成员**a**的存在：

```c++
#include<iostream>
#include<cstdlib>
using namespace std;
static int global1 = 1; //DATA段
static int global2; //BSS段
class Test3
{
private:
  int a;
public:
  void f()
  {
    cout<<"Test1: Core or not ? "<<endl;
  }
};
Test3 obj3; //DATA段 (调用默认构造函数进行初始化)
int main()
{
  Test3 obj1; //stack段。定义obj1，这时候自然为a分配内存了
  Test3 *obj2 = new Test3();//heap段，也为a分配内存了。
  Test3 *p;//只是声明，没有定义。没有对象，也没有a。
  return 0;
}
```
受王杰兄（微博：@skyline09_）启发和提示，其实我们可以换一种形式来理解。

我们知道，**C++**中，类的非静态成员函数会被编译器改写：

`void Test::f()`被改写为类似于`void Test__f(Test *const this)`

而

    Test *p = NULL;
    
    p->f();

将被编译器改写为

    Test *p = NULL;
    Test__f(p);

因此`Test::f`中带有一个值为**NULL**的**this**指针（**p**），如果通过这个空指针**p**读写数据，就会崩溃。否则，安然无事。

那么，**this**指针可以操纵哪些东西呢？哦，类的非静态数据成员。而类的静态数据成员，全局变量等，是不会通过**this**指针访问的，因此，上例中，访问**a**崩溃，访问**global**则安全。

最后，我们看一个问题。这个问题，最早我是从杜克伟兄（微博：@小伙伴-小伙伴儿）那里听到的。
```c++
class Test4
{
public:
  void f()
  {
    cout<<&(*this)<<endl; // 有问题吗？
  }
};
int main()
{
  Test4 *p = NULL;
  p->f();
  return 0;
}
```

没问题。**&(*this)**就是**this**，值和**p**相等。因此上面会输出**0**。

综上，使用空指针并不一定会发生问题，关键是怎么用。遇到问题得理性分析，不要想当然。

纸上学来终觉浅，绝知此事要躬行。