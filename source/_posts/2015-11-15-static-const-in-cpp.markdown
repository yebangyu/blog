---
layout: post
title: "再谈C++中的static const"
date: 2015-11-15 12:45:16 +0800
comments: true
categories: c++
keywords: static, const, c++, template
description: static const 在C++中的应用
---

废话不多说了，开始这次的讨论吧。

## 提出问题

同事颜廷帅（微博：@颜挺帅）拿了一段代码让我分析。以下代码，能编译链接通过吗？

<!--more-->

```c++
void f(const int &value)
{
}
class Test
{
public:
  static const int a = 1;
};
int main()
{
  f(Test::a);
  return 0;
}
```

我的第一感觉是：应该没问题，吧。在**Visual Studio 2013** 实验了下，顺利通过，一切正常.

然而同事说，**gcc**下是报错的：Undefined reference to 'Test::a'

这是为什么？

## 分析问题

写作本文时所用的环境是**gcc 4.8.2 (Ubuntu 14.04 , X86平台)**。

注意，本文的讨论只针对类的**static const**成员，也就是所谓的**class scope**。**namespace scope**的情况不属于我们的讨论范围内。

把以上代码保存为**test.cpp**，然后用**gcc**编译它：

    g++ -c -o test.o test.cpp
    
这个命令执行之后，我们会在目录下得到**test.o**文件。接着，我们通过**objdump**来查看符号表：

    objdump -x test.o
    
我们可以看到类似如下的输出：

    SYMBOL TABLE:
    00000000 l    df *ABS*	00000000 test.cpp
    00000000 l    d  .text	00000000 .text
    00000000 l    d  .data	00000000 .data
    00000000 l    d  .bss	00000000 .bss
    00000000 l    d  .note.GNU-stack	00000000 .note.GNU-stack
    00000000 l    d  .eh_frame	00000000 .eh_frame
    00000000 l    d  .comment	00000000 .comment
    00000000 g     F .text	00000005 _Z1fRKi
    00000005 g     F .text	00000019 main
    00000000         *UND*	00000000 _ZN4Test1aE

上面的最后一行，**_ZN4Test1aE**就是对应我们程序中声明的**Test::a**。之所以长得这么复杂、奇怪，是因为编译器做了**mangling**处理。

注意到\*UND\*没？根据[文档](https://sourceware.org/binutils/docs/binutils/objdump.html)的解释：

\*UND\* : if the section is referenced in the file being dumped, but not defined there

也就是**_ZN4Test1aE**在本**.o**文件中引用，然而它却木有定义。因此，报Undefined reference to 'Test::a'的错，也就情理之中了。

那么，我们的程序是否真的引用了**_ZN4Test1aE**呢？恩，我们接着往下看。

输入如下命令：

    g++ -S test.cpp
    
我们可以得到类似这样的汇编代码(做了整理，节选)：

    main:
    	pushl	%ebp
    	movl	%esp, %ebp
    	subl	$4, %esp
    	movl	$_ZN4Test1aE, (%esp) ;看到没？_ZN4Test1aE ！
    	call	_Z1fRKi ;调用函数f
    	movl	$0, %eax
    	leave
    	ret

虽然我们已经分析出为什么会报错，然而，我们还有一个疑问，就是，为什么下面的代码，是OK的？

```c++
void f(const int value) //这里没有 &
{
}
class Test
{
public:
  static const int a = 1;
};
int main()
{
  f(Test::a);//没问题
  return 0;
}
```


恩，有了前面的基础，相信读者诸君已经知道怎么分析。我们可以用同样的方法，看看它的汇编代码：

    main:
    	pushl	%ebp
    	movl	%esp, %ebp
    	subl	$4, %esp
    	movl	$1, (%esp)     ;看到没？1，不是_ZN4Test1aE，也不是Test::a
    	call	_Z1fi
    	movl	$0, %eax
    	leave
    	ret

也就是说，在这里，编译器只是把**Test::a**认作一个占位符，实际使用之处用**1**代替了。

## 解决问题

知道原因了，那么怎么解决呢？恩，至少三种方法：

1，我们可以**定义**（而不是声明）**Test::a**。是的，上面的`static const int a = 1;`并不是它的定义式。如果要定义，那么我们应该这么做：

```c++
void f(const int &value)//还是传引用
{
}
class Test
{
public:
  static const int a;
};
const int Test::a = 1;//定义a
int main()
{
  f(Test::a);//现在没问题了
  return 0;
}
```
有兴趣的读者可以看看这个程序对应的符号表，就会发现**Test::a**被放在了程序的**rodata section**，而不是\*UND\*了。

2，如果仅仅声明**a**，那么我们可以按值的方式使用它，这没问题。也就是只使用它的值；而不去获得它的地址(当然，也包括引用。引用本质上也是地址)。

3，使用枚举类型。是的，枚举！像这样：

```c++
void f(const int &value)//还是传引用
{
}
class Test
{
public:
  enum HELLOWORLD {a = 1}; //枚举，而不是 static const
};
int main()
{
  f(Test::a);//没问题
  return 0;
}
```
那么，这种情况下，编译器是如何处理的呢？就留给读者诸君作为练习吧。