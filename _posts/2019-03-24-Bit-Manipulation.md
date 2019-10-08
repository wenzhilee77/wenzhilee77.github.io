---
layout: post
title:  一些有趣有用的位操作
categories: Algorithms Bit-Manipulation
tags:  Algorithms Bit-Manipulation
author: wenzhilee77
---

# 判断两个整数是否异号
```java
int x = -1, y = 2;
bool f = ((x ^ y) < 0); // true

int x = 3, y = 2;
bool f = ((x ^ y) < 0); // false
```

PS：这个技巧还是很实用的，利用的是补码编码的符号位。如果不用位运算来判断是否异号，需要使用 if else 分支，还挺麻烦的。读者可能想利用乘积或者商来判断两个数是否异号，但是这种处理方式可能造成溢出，从而出现错误。关于补码编码和溢出，可参见前文 x * x >= 0 一定成立吗？。

# 交换两个数
```java
int a = 1, b = 2;
a ^= b;
b ^= a;
a ^= b;
// 现在 a = 2, b = 1
```

# 算法常用操作 n&(n-1)

这个操作是算法中常见的，作用是消除数字 n 的二进制表示中的最后一个 1。

1. 计算汉明权重（Hamming Weight）
```java
int hammingWeight(uint32_t n) {
    int res = 0;
    while (n != 0) {
        n = n & (n - 1);
        res++;
    }
    return res;
}
```

2. 判断一个数是不是 2 的指数

一个数如果是 2 的指数，那么它的二进制表示一定只含有一个 1：
如果使用位运算技巧就很简单了（注意运算符优先级，括号不可以省略）：
```java
bool isPowerOfTwo(int n) {
    if (n <= 0) return false;
    return (n & (n - 1)) == 0;
}
```


# 参考

https://mp.weixin.qq.com/s/UJoDkBi9VRXW9R3RelR0fQ