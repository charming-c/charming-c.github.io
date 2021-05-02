---
title: Leetcode刷题笔记--动态规划（二）
date: 2021-04-14 00:20:52
tags:
	[Leetcode,动态规划]
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

## Leetcode 第 91 题 -- 解码方法

题目描述：

> 一条包含字母 A-Z 的消息通过以下映射进行了 编码 ：
>
> 'A' -> 1
> 'B' -> 2
> ...
> 'Z' -> 26
> 要 解码 已编码的消息，所有数字必须基于上述映射的方法，反向映射回字母（可能有多种方法）。例如，"11106" 可以映射为：
>
> "AAJF" ，将消息分组为 (1 1 10 6)
> "KJF" ，将消息分组为 (11 10 6)
> 注意，消息不能分组为  (1 11 06) ，因为 "06" 不能映射为 "F" ，这是由于 "6" 和 "06" 在映射中并不等价。
>
> 给你一个只含数字的 非空 字符串 s ，请计算并返回 解码 方法的 总数 。
>
> 题目数据保证答案肯定是一个 32 位 的整数。

题目分析：

这也是一个很典型很简单的重叠子问题，这个问题的核心是简化决策思路，将状态转移方程写得尽可能简单，而不是一点一点分情况讨论，比如我就写得很丑很难理解，就是将问题的情况分的太细致了，而忽略了简化思路：

```java
public int numDecodings(String s) {
        int len = s.length();
        int[] dp = new int[len];
        if (s.charAt(0) == '0')
            return 0;
        else dp[0] = 1;

        for (int i = 1; i < len; i++) {
            if (s.charAt(i) == '0') {
                if (s.charAt(i - 1) == '2' || s.charAt(i - 1) == '1') {
                    if (i >= 2)
                        dp[i] = dp[i - 2];
                    else dp[i] = 1;
                } else return 0;
            } else {
                if (s.charAt(i - 1) == '1' || s.charAt(i - 1) == '2') {
                    if ((s.charAt(i) <= '6' || s.charAt(i - 1) == '1') && i - 2 >= 0) dp[i] = dp[i - 1] + dp[i - 2];
                    else {
                        if (s.charAt(i - 1) == '2' && s.charAt(i) <= '6')
                            dp[i] = dp[i - 1] + 1;
                        else if (s.charAt(i - 1) == '1') dp[i] = dp[i - 1] + 1;
                        else dp[i] = dp[i - 1];
                    }
                } else {
                    dp[i] = dp[i - 1];
                }
            }
        }

        return dp[len - 1];
```

而官方题解给出的答案就很简洁，很方便阅读和理解：

```java
public int numDecodings(String s) {
            int n = s.length();
            int[] f = new int[n + 1];
            f[0] = 1;
            for (int i = 1; i <= n; ++i) {
                if (s.charAt(i - 1) != '0') {
                    f[i] += f[i - 1];
                }
                if (i > 1 && s.charAt(i - 2) != '0' && ((s.charAt(i - 2) - '0') * 10 + (s.charAt(i - 1) - '0') <= 26)) {
                    f[i] += f[i - 2];
                }
            }
            return f[n];
```

只要当前的位置的字符不为0，我们就有理由知道，他一定可以继承自前面的状态，至于需不需要决策到再前面一个位置，就要考虑 i - 1 与 i - 2 能否凑成一个小于等于 26 的编码，并且要保持 i - 2 与 i - 1 不可同时为零

## Leetcode 第 120 题 -- 三角形最小路径和

题目描述：

> 给定一个三角形 triangle ，找出自顶向下的最小路径和。
>
> 每一步只能移动到下一行中相邻的结点上。相邻的结点 在这里指的是 下标 与 上一层结点下标 相同或者等于 上一层结点下标 + 1 的两个结点。也就是说，如果正位于当前行的下标 i ，那么下一步可以移动到下一行的下标 i 或 i + 1 。
>
>  
>
> 示例 1：
>
> 输入：triangle = [[2],[3,4],[6,5,7],[4,1,8,3]]
> 输出：11
> 解释：如下面简图所示：
>    2
>   3 4
>  6 5 7
> 4 1 8 3
> 自顶向下的最小路径和为 11（即，2 + 3 + 5 + 1 = 11）。

题目分析：

我们从正向去解析这个题目，其实很容易就能想到递归去解决这个问题，非常非常简单，只需要把路径和交给下一排去解决就好了，所以：

```java
class Solution {
        int maxDepth;
        public int minimumTotal(List<List<Integer>> triangle) {
            maxDepth = triangle.size();
            return minTotal(triangle, triangle.get(0).get(0), 0, 1);
        }

        public int minTotal(List<List<Integer>> triangle, int sum , int location, int depth){
            if(depth >= maxDepth) return sum;
            else
                return Math.min(minTotal(triangle, sum + triangle.get(depth).get(location),location, depth + 1),minTotal(triangle, sum + triangle.get(depth).get(location+1),location + 1, depth+ 1));
        }
    }
```

然后我们惊喜的发现...

那这道题有什么方便一点的方法吗，其实这是一道非常非常简单的动态规划的题目，我们从最底层去回溯这个三角形就好了，可以在最底层往上面做决策，不要每一次都傻傻的无脑递归算清楚每一条路径的值

![IMG_664E89EE30D5-1](https://tva1.sinaimg.cn/large/008i3skNgy1gpsrz6vv50j31qt0rsk19.jpg)

上代码：

```java
public int minimumTotal(List<List<Integer>> triangle) {
        int m = triangle.size();
        int n = triangle.get(m-1).size();
        int[][] dp = new int[m][n];
        for(int i = m-1; i>=0; i--){
            for(int j = 0; j<=i; j++){
                if(i == m-1){
                    dp[i][j] = triangle.get(i).get(j);
                }
                else{
                    dp[i][j] = Math.min(dp[i+1][j],dp[i+1][j+1]) + triangle.get(i).get(j);
                }
            }
        }
        return dp[0][0];
    }
```

当然这里的dp数组的空间大小是可以压缩的。

## Leetcode 第 368 题 -- 最大整除子集

题目描述：

> 给你一个由 无重复 正整数组成的集合 nums ，请你找出并返回其中最大的整除子集 answer ，子集中每一元素对 (answer[i], answer[j]) 都应当满足：
> answer[i] % answer[j] == 0 ，或
> answer[j] % answer[i] == 0
> 如果存在多个有效解子集，返回其中任何一个均可。
>
>  
>
> 示例 1：
>
> 输入：nums = [1,2,3]
> 输出：[1,2]
> 解释：[1,3] 也会被视为正确答案。

题目分析：

这也是一道比较经典的动态规划的题目。我们可以申请一个二维的 dp 数组，第一行用来存储离他最近的前面的因数的位置，方面我们最后回溯，去找到最终的答案，另一个则是记录当前这个位置往前回溯的因数的最大个数，这样方便我们找到最大的答案，这里用图举一个例子：

![IMG_E0FCDD9C3F0F-1](/Users/charming/Downloads/IMG_E0FCDD9C3F0F-1.jpeg)

这里就很明确，状态转移方程：

```java
dp[ i ] [1] = Math.max(dp[ i ] [1], dp[ j ] [1] + 1);
                    if (dp[ i ] [1] == dp[ j ] [1] + 1) {
                        dp[ i ] [0] = j;
                    }
```

而j的寻找就可以从0在前面开始往i遍历，最后记录一下最大的dp[i] [1]的值，这样，在遍历一次就可以找到最长的因数集的末尾位置，再通过dp[i] [0] 找到前面所有的元素，添加到答案里。

代码：

```java
public List<Integer> largestDivisibleSubset(int[] nums) {
        int len = nums.length;
        int[][] dp = new int[len][2];
        Arrays.sort(nums);
        for (int i = 0; i < len; i++)
            Arrays.fill(dp[i], -1);
        dp[0][0] = -1;
        dp[0][1] = 1;
        int max = 1;
        for (int i = 1; i < len; i++) {
            for (int j = 0; j < i; j++) {
                if (nums[i] % nums[j] == 0) {
                    dp[i][1] = Math.max(dp[i][1], dp[j][1] + 1);
                    if (dp[i][1] == dp[j][1] + 1) {
                        dp[i][0] = j;
                    }
                    max = Math.max(max, dp[i][1]);
                }
            }
            if (dp[i][1] == -1) {
                dp[i][0] = -1;
                dp[i][1] = 1;
            }
        }

        int index = 0;
        for (int i = 0; i < len; i++) {
            if (dp[i][1] == max) {
                index = i;
            }
        }
        List<Integer> ans = new ArrayList<>();
        while (dp[index][0] != -1) {
            ans.add(nums[index]);
            index = dp[index][0];
        }
        ans.add(nums[index]);
        return ans;
    }
```

