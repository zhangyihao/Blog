---
title: 剑指offer读书笔记-二维数组查找问题
date: 2017-12-27 17:28:42
tags:
  - 剑指offer
  - 算法
categories:
  - 算法
comments: false

---

# 问题描述 #

写一个函数，输入n，求斐波那契（Fibonacci）数列第n项。
斐波那契数列定义为：f(n) = f(n-1) + f(n-2) (n>1) ,其中f(0)=0, f(1)=1。

<!--more-->

# 递归解法 #

斐波那契数列是一个典型的递归解法，代码如下：

```java
public int Fibonacci(int n) {
	if (n == 0) {
		return 0;
	}

	if (n == 1) {
		return 1;
	}
	return Fibonacci(n - 1) + Fibonacci(n - 2);
}
```

以上代码虽然可以计算出第n项，但存在两个问题：

2. 计算f(n)时，f(n-2)、f(n-3)、fn(n-4)....f(2)分别需要两次递归计算，大大浪费了时间和空间。
3. 当n太大时，递归层次太深，导致stackoverflow。

# 算法优化 #


