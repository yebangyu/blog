---
layout: post
title: "Introduction To Cuckoo Hashing"
date: 2015-12-19 00:41:37 +0800
comments: true
categories: 算法
keywords: Cuckoo Hashing , Lock Free
description: Cuckoo Hashing
---

## Motivation & Intuition

为什么引入**Cuckoo Hashing**？

常见的**hashing**处理冲突方法一般包括两种：开散列（**Separate Chaining**）和闭散列**（Linear / Quadratic Probing**
）。开散列是将冲突的元素组织成一个链表（其实组织成一个二叉搜索树也是完全没问题的，甚至跳表也行），闭散列将冲突的元素还是放在哈希表**slot**中，使用线性探测等方法进行处理。

那么，这两种方法，都有啥优缺点呢？

<!--more-->

开散列，实现简单，但是对**cache**不友好，**cache miss rate**较高。

闭散列，实现相对复杂一点点，对**cache**很友好，但是对**load factor**要求苛刻：**load factor**稍高性能就急剧下降。

这两种方式下，查找某个元素的最坏时间都是**O(n)**。

你说，**OK，OK**，我知道，这些都是常识。那么，是否可以做到查找最坏是**O(1)**呢？

一种思路是元素只可能被安置到有限的常数(记为**K**)个位置，插入时，如果发生冲突，由于每个元素可以存放的位置有**K**个，因此可以对表部分元素进行重排，产生一个空缺的位置。

**Cuckoo Hashing**就是这样的方式。当**K=2**时，**Cuckoo Hashing**在**load factor**为**50%**左右的情况下表现较佳。如果**K=4**，那么甚至可以在**97%**的**load factor**下良好工作。

## Cuckoo Hashing

一般的**Hashing**只包括一个**Hash Tables**，但是**Cuckoo Hashing**由两张甚至多张表构成。每张表对应一个哈希函数。本文讨论两张哈希表（记为**table1**和**table2**）、两个哈希函数（记为**hf1**和**hf2**）这种常见情形。

### Insert

首先通过**hf1**计算出一个**slot index**，然后查看**table1**中该**slot**是否**vacant**，如果是，则插入；否则通过**hf2**计算出一个**slot index**，通过查看**table2**中该**slot**是否**vacant**，如果是，则插入，否则执行**rearrange**操作。

**rearrange**操作的过程：随机选出一张表，将**slot index**对应的那个元素踢出(**evict**)，把我们待插入的元素插到那个位置。那被踢出来的元素呢？尝试插入到另外一张表对应的**slot**处，这时候可能又踢出一个元素，接下去就是递归的执行这个过程，直到所有元素都安置妥当。

举个例子吧，假如某个时刻，两个哈希表的内容如下：

![table1](http://7xnljs.com1.z0.glb.clouddn.com/table1.jpg)

假设我们待插入的元素为**77**。

**slot index1 = hf1(77) = 1**

**Table1**中的**index**为**1**的**slot**已经被**78**占了。那么看**Table2**：

**slot index2 = hf2(77) = 3**

**Table2**中的**index**为**3**的**slot**已经被**33**占了。因此执行**rearrange**。

执行**rearrange**动作，选择**Table2**，将**slot index = 3**的元素**33**踢出，插入**77**。然后被踢出的元素**33**，计算它在**Table1**中的**index**为**slot index1 = hf2(33) = 2**，因此将**95**踢出，插入**33**。被踢出的元素**95**在**Table2**中的**slot index**为**2**，该**slot**为**vacant**，没人使用，因此将**95**插入。完毕。现在的**Tables**中元素为：

![table2](http://7xnljs.com1.z0.glb.clouddn.com/table2.jpg)

值得注意的是，**rearrange**可能失败（表满了;或者发生“死循环”），此时需要进行**rehash**，因此代码里需要有一定的判断。当**K=2**时，只要**load factor**低于**50%**，需要**rehash**的概率很小很小。

在某些假设下，插入操作的摊还期望复杂度为常数时间。

###Find

要检查的**slot**一共两个，**index**分别为**hf1(key)**和**hf2(key)**，因此只要查看一下**Table1**中的**hf1(key)**以及**Table2**中的**hf2(key)**这两个**slot**即可。时间复杂度为**O(1)**。

###Del

同**Find**，要检查的**slot**也就两个，复杂度为**O(1)**。

## 实现

简单实现了一个，有需要的可以参考[这里](https://github.com/yebangyu/Yedis/blob/master/src/ds/CuckooHashMap.h)

## 其他

1，我们注意到，**Cuckoo Hashing**的精髓是使用两个不同的哈希函数，而不是两张表。两张表的存在，仅仅是为了分析上的方便。

当需要**rehash**而**load factor**又没达到**100%**时，我们其实不需要扩容哈希表，只需要更换哈希函数。

2，为嘛叫做**Cuckoo Hashing**？**Cuckoo**，即杜鹃鸟（布谷鸟），这种鸟有一种尿性：孵卵寄生。把蛋产到别的鸟窝里，让别人帮它孵化。这还不算，还要把人家寄主的一些卵给移走（不然容易引起怀疑嘛，毕竟鸟窝里突然多出几枚蛋。至于移走多少，就得看杜鹃鸟数学合不合格了）！等卵孵化完成，幼雏会将鸟窝里寄主的卵和其他幼雏推出鸟窝。真是牛逼闪闪了。

## 参考文献

http://resources.mpi-inf.mpg.de/departments/d1/teaching/ws14/AlgoDat/materials/cuckoo.pdf

**Cuckoo Hashing**原始论文

https://www.eecs.harvard.edu/~michaelm/postscripts/esa2009.pdf

这篇文章介绍了一些关于**Cuckoo Hashing**的**Open Questions**

http://web.stanford.edu/class/cs166/lectures/13/Slides13.pdf

这个**slides**偏重对**Cuckoo Hashing**理论上的分析，注意其中对于插入操作的处理和本文介绍的不同。

http://excess-project.eu/publications/published/CuckooHashing_ICDCS.pdf

碉堡了，**Lock Free Cuckoo Hashing**。