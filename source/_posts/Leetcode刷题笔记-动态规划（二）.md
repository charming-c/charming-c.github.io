---
title: Leetcode刷题笔记--动态规划（二）
date: 2021-04-14 00:20:52
tags:
	[leetcode,动态规划]
categories: 算法
---



# 动态规划（二）

## Leetcode 第264题 -- 丑数II

题目描述：

> 给你一个整数 n ，请你找出并返回第 n 个 丑数 。
>
> 丑数 就是只包含质因数 2、3 和/或 5 的正整数。
>
> 
>
> 示例 1：
>
> 输入：n = 10
> 输出：12
> 解释：[1, 2, 3, 4, 5, 6, 8, 9, 10, 12] 是由前 10 个丑数组成的序列。

题目分析：

这道题一看就很好用暴力的方法解出来，因为我们可以暴力从小到大枚举所有的数，这样当这个数是丑数我们就用一个count做记录加1，一直加到count == n，这样第n个整数就找出来了。

<!--more-->

```java
class Solution {
    public int nthUglyNumber(int n) {
        int count = 0;
        int num = 1;
        while(true){
            if(isUgly(num)) {
                count++;
            }
            if(count == n) break;
            num++;
        }
        return num;
    }
    public boolean isUgly(int n) {
        if(n == 1) {
            return true;
        }
        if(n == 0) return false;
        if(n%2 == 0) return isUgly(n/2);
        if(n%3 == 0) return isUgly(n/3);
        if(n%5 == 0) return isUgly(n/5);

        return false;
    }
}
```

这么写毫无疑问就是超时了，毕竟一道中等难度的题目不会这么简单，那有什么更加快捷的算法呢？

其实这道题每个位置的丑数的取值就是一个决策问题，他是依赖于前面的情况所做的一个决策，找到符合条件的，比前面的数大的最小的丑数，那这个问题的核心就是如何去做一个决策。我们从题目里很容易就知道每个丑数的质因数只包含2，3，5，那我们可以从之前已经找到的丑数里去选择最小的2，3，5的倍数就好了，如图：

![IMG_6F23E5EA12D1-1](https://tva1.sinaimg.cn/large/008eGmZEgy1gpikr4llt1j31dv0rszo0.jpg)

利用三个指针，每个指针从第一个位置出发，每次取当前三个指针指向的位置上的数乘以对应的倍数（2，3，5），找到最小的并填上去，然后三个指针当该位置的值被采纳了，指针的位置就往后移动。

```java
class Solution {
    public int nthUglyNumber(int n) {
        int[] dp = new int[n + 1];
        dp[1] = 1;
        int p2 = 1, p3 = 1, p5 = 1;
        for (int i = 2; i <= n; i++) {
            int num2 = dp[p2] * 2, num3 = dp[p3] * 3, num5 = dp[p5] * 5;
            dp[i] = Math.min(Math.min(num2, num3), num5);
            if (dp[i] == num2) {
                p2++;
            }
            if (dp[i] == num3) {
                p3++;
            }
            if (dp[i] == num5) {
                p5++;
            }
        }
        return dp[n];
    }
}
```

