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

在一个二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。
请完成一个函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

<!--more-->

# 解题思路 #

## 暴力破解 ##

暴力破解方式也就是对二维数组中的每个元素逐个检查，结束条件为找到和目标值相等的元素或者查询到最后一个元素。

对于一个行列数都为n的二维数组而言，最坏的情况是将整个数组遍历完，这种情况下时间复杂度为O(n^2)，并不是一个好的解决方法。

## 优化解法 ##



# 代码 #

```java
public boolean find(int target, int[][] array) {
	if(array == null || array.length == 0) {
		return false;
	}
	int rowLength = array.length;
	int colLength = array[0].length;
	
	boolean find = false;
	for(int col=colLength-1, row=0; col>=0 && row<rowLength;) {
		int num = array[row][col];
		if(num == target) {
			find = true;
			break;
		}else if (num<target) {
			row++;
		} else {
			col--;
		}
	}
	return find;
}
```