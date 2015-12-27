---
layout: post
title: "编写高质量代码(上)"
date: 2015-11-06 00:24:56 +0800
comments: true
categories: c++
keywords: C++, 代码规范, 重构, 设计, 正确性, 算法
description: 本文介绍如何编写高质量的C++代码。从代码规范、软件工程、算法设计等角度综合讲解
---

我们知道，**int**和**double**能表示的数值的范围不同。其中，**64**位有符号整数的范围是[**-9223372036854775808**,**9223372036854775807**]，而**64**位无符号整数的范围是[**0**,**18446744073709551615**]。这两个区间有一定的**overlap**，而**double**可以表示的范围更大。

现在，需要编写两个函数:给定一个**double**型的**value**，判断这个**value**是否是一个合法的**int64_t**或者**uint64_t**。本文说的“合法”，是指数值上落在了范围内。

    bool is_valid_uint64(const Double &value);

    bool is_valid_int64(const Double &value);

这里我们用**Double**而不是**double**，原因是我们的**double**不是基础数据类型，而是通过一定方法实现的**ADT**，这个**ADT**的成员函数有：

<!--more-->

```c++
class Double
{
  public:
    int get_next_digit(bool &is_decimal);
    bool is_zero();
    bool is_neg();
};
```
通过调用`get_next_digit`，可以返回一个数字，不断调用它，可以得到所有**digits**。举个例子，对于值为**45.67**的一个**Double**对象，调用它的`get_next_digit`成员函数将依次得到

4 is_decimal = false //表示整数部分

5 is_decimal = false //表示整数部分

6 is_decimal = true //表示小数部分

7 is_decimal = true //表示小数部分

当`get_next_digit`返回**-1**时，表示读取完毕。

如何利用**Double**类里的成员函数，来实现`is_valid_uint64`和`is_valid_int64`这两个函数呢？

一些新手可能会写这样的代码：

```c++
bool is_valid_uint64(const Double &value)
{
  bool is_valid = true;
  int digits[2000];
  int counts = 0;
  if (value.is_zero()) { 
    is_valid = true;
  } else if(value.is_neg()) { 
    is_valid = false;
  } else { 
    bool is_decimal = false;
    int digit = 0;
    while((digit=value.get_next_digit(is_decimal)) != -1) {
      if (is_decimal) { 
        is_valid = false;
        break;
      } else {
        digits[counts++] = digit;
      }
    }
    uint64_t tmp = 0;
    uint64_t base = 1;
    for (int i = counts - 1; i >= 0; i++) {
      tmp += digits[i] * base;
      if (tmp > UINT64_MAX) {
        is_valid = false;
        break;
      }
      base *= 10;
    }
  }
  return is_valid;
}
bool is_valid_int64(const Double &value)
{
  bool is_valid = true;
  int digits[2000];
  int counts = 0;
  if (value.is_zero()) { 
    is_valid = true;
  } else if(value.is_neg()) { 
    bool is_decimal = false;
    int digit = 0;
    while((digit=value.get_next_digit(is_decimal)) != -1) {
      if (is_decimal) { 
        is_valid = false;
        break;
      } else {
        digits[counts++] = digit;
      }
    }
    uint64_t tmp = 0;
    uint64_t base = 1;
    for (int i = counts - 1; i >= 0; i++) {
      tmp += digits[i] * base;
      tmp *= -1;
      if (tmp < INT64_MIN) {
        is_valid = false;
        break;
      }
      base *= 10;
    }
  } else { 
    bool is_decimal = false;
    int digit = 0;
    while((digit=value.get_next_digit(is_decimal)) != -1) {
      if (is_decimal) { 
        is_valid = false;
        break;
      } else {
        digits[counts++] = digit;
      }
    }
    uint64_t tmp = 0;
    uint64_t base = 1;
    for (int i = counts - 1; i >= 0; i++) {
      tmp += digits[i] * base;
      if (tmp > INT64_MAX) {
        is_valid = false;
        break;
      }
      base *= 10;
    }
  }
  return is_valid;
}
```
这样的代码，存在诸多问题。

##设计问题

不难发现，两个函数存在很多相似甚至相同的代码；而同一个函数内部，也有不少代码重复。重复的东西往往不是好的。重构？

##性能问题

先获得所有digits，然后从最低位开始向最高位构造值，效率较低。难道没有可以从最高位开始，边获得边计算，不需要临时数组存储所有digits的方法吗？

##正确性问题

随便举几个例子：

第**24**行，`tmp += digits[i] * base`;有没有考虑到可能的溢出呢？

第**68**行，难道有小数部分就一定不是合法的**int64**吗？那么，**123.000**？嗯？

##规范问题

帅哥，这么多代码，一行注释都没有，这样真的好吗？

因此，毫无疑问，这是烂代码，不合格的代码，需要重写的代码。

------

以下是我个人认为比较好的设计和实现，仅供参考。

```c++

bool is_valid_uint64(const Double &value)

{

  bool ret = false;

  check_range(&ret, NULL, value);

  return ret;

}



bool is_valid_int64(const Double &value)

{

  bool ret = false;

  check_range(NULL, &ret, value);

  return ret;

}



void check_range(const Double &value,

                 bool *is_valid_uint64,

                 bool *is_valid_int64) const

{

  /*

   * 对于一个负数的value，它不可能是一个合法的uint64.

   * 因此，只剩下三种可能：

   * I 输入的value是负数，判断是否是合法的int64

   * II 输入的value是正数，判断是否是合法的uint64

   * III 输入的value是正数，判断是否是合法的int64

   * 对于第II、III这两种情况：只要判断value的值是否超过uint64、int64的上界即可

   * 对于第I种情况，我们利用-A > -B 等价于 A < B （其中A、B是正数）

   * 因此，在第I种情况里，可以判断value的绝对值，是否超过int64的最小值的绝对值即可。

   * （int64的最小值的绝对值？那不就是int64的最大值？哦，不！）

   * 因此，不管哪种情况，判断绝对值是否超过某个上界即可。

   * 这三种情况，上界不一样。把三个上界存到了一个二维数组THRESHOLD里

  */



  bool *is_valid = NULL;

  static const int FLAG_INT64 = 0;

  static const int FLAG_UINT64 = 1;

  static const int SIGN_NEG = 0;

  static const int SIGN_POS = 1;

  int flag = FLAG_INT64;

  if (NULL != is_valid_uint64) {

    is_valid = is_valid_uint64;

    flag = FLAG_UINT64;

  } else {

    is_valid = is_valid_int64;

  }

  *is_valid = true;

  if (value.is_zero()) {

    //do nothing。0是合法的uint64，也是合法的int64

  } else {

    int sign = value.is_neg() ? SIGN_NEG : SIGN_POS;

    if ((SIGN_NEG == sign) && (FLAG_UINT64 == flag)) {

      *is_valid = false;//负数不可能是合法的uint64

    } else {

      uint64_t valueUint = 0;

      static uint64_t ABS_INT64_MIN = 9223372036854775808ULL;

                                         //int64        uint64

      static uint64_t THRESHOLD[2][2] = { {ABS_INT64_MIN, 0}, //neg

                                         {INT64_MAX,     UINT64_MAX} }; //pos

      int digit = 0;

      bool is_decimal = false;

      while ((digit = value.get_next_digit(is_decimal)) != -1) {

        if (!is_decimal) {

          //为了防止溢出，我们不能这么写:

          //"value * 10 + digit > THRESHOLD[index]"

          if (valueUint > (THRESHOLD[sign][flag] - digit) / 10) {

            *is_valid = false;

            break;

          } else {

            valueUint = valueUint * 10 + digit;//霍纳法则（也叫秦九韶算法）

          }

        } else {

          if (!digit) {

            *is_valid = false; //小数部分必须是0；否则不可能是合法的uint64、int64

            break;

          }

        }

      }

    }

  }

}
```

