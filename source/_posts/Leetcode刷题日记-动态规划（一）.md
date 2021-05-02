---
title: Leetcode刷题日记--动态规划（一）
date: 2021-04-06 22:29:09
tags:
	[Leetcode,动态规划]
categories: 算法
---



# 动态规划（一）

## Leetcode 第940题 -- 不同的子序列II

题目描述：

> 给定一个字符串 S，计算 S 的不同非空子序列的个数。
> 	
> 因为结果可能很大，所以返回答案模 10^9 + 7.
>
>
> ​    示例 1：
>
> ​    输入："abc"
>
> ​    输出：7
> ​    解释：7 个不同的子序列分别是 "a", "b", "c", "ab", "ac", "bc", 以及 "abc"。

如果明确了这是一道动态规划的问题，那这道题其实是非常简单的，首先可以知道，运用动态规划的思想，一个长度为N的字符串其实是可以将其简单拆解成计算前面长度为N-1的子串的个数，再将子串个数乘以2就好了，当然这只是简单的分析，因为从实际出发，结果不尽然准确。

<!--more-->

![IMG_BBE9437BDB7C-1](https://tva1.sinaimg.cn/large/008eGmZEgy1gpafyun8jnj32cn0rskb6.jpg)

（这里我们先假设是存在空子串的）对于具体的例子我们显然就可以看到，当在末尾新增加的字符之前从未出现过，则子串个数就很好解释了，等于之前所有的子串再加上之前所有的子串末尾拼上新加的字符（新字符位置为N），即
$$
																					f(N) = f(N - 1) * 2
$$
当新拼接的字符之前出现过时，假设新字符的位置时N，最近一次出现相同字符的位置为M，我们发现，在M位置时，我们其实已经将M-1位置的子串拼接过了，所以最后的结果为：
$$
																		f(N) = f(N-1) * 2 - f(M-1)
$$
那么动态规划过程中，在面对子问题时的决策我们也解决完毕了，接下来就是代码生成的思路了。

```java
public int distinctSubseqII(String S) {
            int MOD = 1_000_000_007;
            int N = S.length();
            
            // 这里默认第零个位置为空字符，数值为1，那么申请的数组内存大小就是N+1
            int[] dp = new int[N + 1];
            dp[0] = 1;

            // 这里其实就是生成一处索引，记录所有字符出现的位置，方便在需要的时候找到M-1的位置
            int[] last = new int[26];
            // 将初值赋值为-1
            Arrays.fill(last, -1);

            // 这里依次遍历目标字符串
            for (int i = 0; i < N; ++i) {
                // 先确定在last里出现的那个位置的字母
                int x = S.charAt(i) - 'a';
                
                // 二话不说先将其前一个位置乘一倍
                dp[i + 1] = dp[i] * 2 % MOD;
                
                // 如果之前这个字符出现过，那么就要减去出现的位置数值
                if (last[x] >= 0)
                    dp[i + 1] -= dp[last[x]];
                dp[i + 1] %= MOD;
                // 每次都要刷新一下位置，因为是要找里N位置最近的出现的相同字符
                last[x] = i;
            }
            // 这里注意一下这里的i其实是对应与N-1哟

            // 多算了一个""字符，减去
            dp[N]--;
            // 如果算出负数加上模就好了
            if (dp[N] < 0) dp[N] += MOD;
            return dp[N];
        }
```

## Leetcode 第1458题 -- 两个子序列的最大点积

题目描述：

> 给你两个数组 nums1 和 nums2 。
>
> 请你返回 nums1 和 nums2 中两个长度相同的 非空 子序列的最大点积。
>
> 数组的非空子序列是通过删除原数组中某些元素（可能一个也不删除）后剩余数字组成的序列，但不能改变数字间相对顺序。比方说，[2,3,5] 是 [1,2,3,4,5] 的一个子序列而 [1,5,3] 不是。
>
>  
>
> 示例 1：
>
> 输入：nums1 = [2,1,-2,5], nums2 = [3,0,-6]
> 输出：18
> 解释：从 nums1 中得到子序列 [2,-2] ，从 nums2 中得到子序列 [3,-6] 。
> 它们的点积为 (2 * 3 + (-2) * (-6)) = 18 。

这道题其实由于是子序列，是规定好了数字的相关顺序的，这样就比较好处理这样的问题了，运用动态规划的思想，其实很很容易就找到思路，建表 dp[m] [n],对应每个位置，第i，j位置上的值就是nums1，nums2两个字数组的最大点积。我们可以画个图来看看具体的思路：

![IMG_9502A9144EAB-1](https://tva1.sinaimg.cn/large/008eGmZEgy1gpbdou46ihj31hc0rrala.jpg)

很显然，红圈里面的数dp[i] [j]的最终数值与它边缘的三个数以及自身nums1[i] * nums2[j]有关，看代码：

```java
public int maxDotProduct(int[] nums1, int[] nums2) {
        int m = nums2.length;
        int n = nums1.length;

        if(m == 0 || n == 0) return 0;

        int[][] dp = new int[m][n];
        dp[0][0] = nums1[0] * nums2[0];

        for(int i = 0; i<m; i++) {
            for(int j = 0; j<n; j++) {

                // 这里先讲第一行、第一列迭代成最大的值
                if(i == 0){
                    if(j != 0) 
                         dp[i][j] = Math.max(dp[i][j-1], nums1[j] * nums2[i]);
                    continue;
                }

                if(j == 0){
                    dp[i][j] = Math.max(dp[i-1][j], nums1[j] * nums2[i]);
                    continue;
                }

                // 先默认dp[i][j]是来自上一行或者上一列最大的值
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);

                // 如果本位置点击小于等于0，则可直接比较dp[i-1][j-1]
                if(nums1[j] * nums2[i] <= 0){
                    dp[i][j] = Math.max(dp[i-1][j-1], dp[i][j]);
                }
                else{
                    // 如果大于0，就得比较三种情况了
                    int temp = Math.max(dp[i-1][j-1]+nums1[j] * nums2[i],nums1[j] * nums2[i]);
                    dp[i][j] = Math.max(dp[i][j], temp);
                }
            }
        }

        return dp[m-1][n-1];
    }
```

## Leetcode 第787题 -- K站中转内最便宜的航班

题目描述：

> 有 n 个城市通过 m 个航班连接。每个航班都从城市 u 开始，以价格 w 抵达 v。
>
> 现在给定所有的城市和航班，以及出发城市 src 和目的地 dst，你的任务是找到从 src 到 dst 最多经过 k 站中转的最便宜的价格。 如果没有这样的路线，则输出 -1。
>
>  
>
> 示例 1：
>
> 输入: 
> n = 3, edges = [[0,1,100],[1,2,100],[0,2,500]]
> src = 0, dst = 2, k = 1
> 输出: 200
> 解释: 
> 城市航班图如下:
>
> ![img](https://tva1.sinaimg.cn/large/008eGmZEgy1gpcm9gidftj30do0a0weg.jpg)
>
>
> 从城市 0 到城市 2 在 1 站中转以内的最便宜价格是 200，如图中红色所示。



看到这个题目我第一眼想到的就是DFS，运用一个数组存放不同城市之间有没有航班，以及花费，最后依次递归到最底部去寻找最少的开销，同时用K做约束条件：

![IMG_1104279CFCA5-1](https://tva1.sinaimg.cn/large/008eGmZEgy1gpcmdlpfsuj32hx0rs4ds.jpg)

这里有两条路线可以从0 到达 2，中转分别是 1， 0

代码：

```java
public int findCheapestPrice(int n, int[][] flights, int src, int dst, int K) {
        int[][] map = new int[n][n];
        for(int i  = 0; i<n; i++){
            Arrays.fill(map[i],-1);
        }

        int len = flights.length;
        for(int i = 0; i<len; i++){
            map[flights[i][0]][flights[i][1]] = flights[i][2];
        }

        int ans = Integer.MAX_VALUE;
        for(int j = 0; j<n; j++){
            if(map[src][j] != -1){
                if(searchPath(n,map,j,dst,map[src][j],K) != -1){
                    ans = Math.min(ans,searchPath(n,map,j,dst,map[src][j],K));
                }
            }
        }
        if(ans == Integer.MAX_VALUE) return -1;
        else return ans;
    }

    public int searchPath(int n, int[][] map, int start, int end, int pay, int k){
        if(start == end && k >= 0) return pay;
        if(k<0) return -1;

        int ans = Integer.MAX_VALUE;

        for(int i = 0; i<n; i++){
            if(map[start][i] > 0){
                int temp = searchPath(n, map, i, end, pay + map[start][i], k-1);
                if(temp != -1){
                    ans = Math.min(temp, ans);
                }
                else continue;
            }
        }

        if(ans == Integer.MAX_VALUE) return -1;

        return ans;
    }
```

但是这里单纯的暴力递归，结果自然可想而知，超时了，很难受。。。

于是思考优化算法：



```java
public class Solution {

        private int[][] graph;
        private boolean[] visited;
        private int res = Integer.MAX_VALUE;

        public int findCheapestPrice(int n, int[][] flights, int src, int dst, int K) {
            K = Math.min(K, n - 2);

            this.graph = new int[n][n];
            for (int[] flight : flights) {
                graph[flight[0]][flight[1]] = flight[2];
            }

            this.visited = new boolean[n];
            // 开始深度优先遍历，注意：这里传 K + 1，这是因为 K 次经停，总共 K + 1 个站
            dfs(src, dst, K + 1, 0);

            if (res == Integer.MAX_VALUE) {
                return -1;
            }
            return res;
        }


        /**
         * 从 src 出发，到 dst 为止，最多经过 k 站（这里 k 包括 src）
         *
         * @param src  起点站
         * @param dst  终点站
         * @param k    经过的站点数限制
         * @param cost 已经花费的价格
         */
        private void dfs(int src, int dst, int k, int cost) {
            if (src == dst) {
                res = cost;
                return;
            }

            if (k == 0) {
                return;
            }

            for (int i = 0; i < graph[src].length; i++) {
                if (graph[src][i] > 0) {
                    if (visited[i]) {
                        continue;
                    }

                    // 剪枝：跳过可能产生较高费用的路径，从而选出最少价格
                    if (cost + graph[src][i] > res) {
                        continue;
                    }

                    visited[i] = true;
                    dfs(i, dst, k - 1, cost + graph[src][i]);
                    visited[i] = false;
                }
            }
        }
    }
}
```

官方题解的这个答案就很灵性，建立全局变量的索引，用一个res去存储已经选出来的最开销，最后可以剪枝优化，还省去了部分搜索，同时用一个Visited数组存放路径经过的节点，防止出现重复计算。

官方题解剩下两个答案我看不懂。。。

## Leetcode 题解第 32 题 -- 最长有效括号

题目描述：

> 给你一个只包含 '(' 和 ')' 的字符串，找出最长有效（格式正确且连续）括号子串的长度。
>
>  
>
> 示例 1：
>
> 输入：s = "(()"
> 输出：2
> 解释：最长有效括号子串是 "()"
> 示例 2：
>
> 输入：s = ")()())"
> 输出：4
> 解释：最长有效括号子串是 "()()"

题目分析：

像这种题目要求你求最长、最大等多过程迭代的问题大概率是可以用动态规划解决的

这道题如果把描述改一下，改成格式正确且不用连续的子串的长度，那这道题就可以采用前面几个题目一样的思路去解决，就是慢慢往后迭代，如果可以找到匹配的就往dp数组里加数字

```java
class Solution {
    public int longestValidParentheses(String s) {
        int len = s.length();
        if(len == 0) return 0;

        int[][] dp = new int[len][2];
        for(int i = 0; i<len; i++)
            Arrays.fill(dp[i],0);
        if(s.charAt(0) == '(') dp[0][1] = 1;

        for(int i = 1; i<len; i++){
            if(dp[i-1][1] >= 1){
                if(s.charAt(i) == ')'){
                    dp[i][1] = dp[i-1][1] - 1;
                    dp[i][0] = dp[i-1][0] + 1;
                }
                else{
                    dp[i][1] = dp[i-1][1] + 1;
                    dp[i][0] = dp[i-1][0];
                }
            }
            else{
                dp[i][0] = dp[i-1][0];
                if(s.charAt(i) == ')'){
                    dp[i][1] = 0;
                    
                }
                else{
                    dp[i][1] = 1;
                    
                }
            }
        }
        return dp[len-1][0] * 2;
    }
}
```

这里我使用了一个二维数组去保存状态，dp[i] [0]表示当前位置有效的括号的数量, dp[i] [1]表示这个位置之前的'('的数量，然后分情况找状态转移方程就可以了，这里分情况就依赖于两个变量，一个是dp[i-1] [1] ,这个表示当前位置的括号怎么与前面的接触，一个就是当前位置的括号，显然正括号和反括号的处理逻辑是不一样的。

当然这道题肯定不是这么写，因为上面的解法是将整个字符串的子串取出来去找最多的正确匹配，子串并不一定在原来的字符串中是连续的，所以我们得换一个动态规划的角度去思考。

新的角度就是，既然是连续的子串，我们动态规划就做切割处理，每一个子问题是有边界的，我们并不是完全依赖于当前的位置，一路迭代一路去保存最大的值，而是在数组里去找到符合规则的子串，再将数值填上，这样当我们跃至子问题时仍然可以用子问题的答案求解切割的串的答案，然后利用一个变量一路去记录最大的值就好了。

```java
public int longestValidParentheses(String s) {
        int len = s.length();
        if (len == 0 || len == 1) return 0;

        int[] dp = new int[len];
        Arrays.fill(dp, 0);

        if (s.charAt(0) == '(' && s.charAt(1) == ')') dp[1] = 2;
        int ans = dp[1];

        for (int i = 2; i < len; i++) {
            if (s.charAt(i) == ')') {
                if (s.charAt(i - 1) == '(') {
                    dp[i] = dp[i - 2] + 2;
                }

                if (s.charAt(i - 1) == ')') {
                    if (i - dp[i - 1] - 1 > 0 && s.charAt(i - dp[i - 1] - 1) == '(') {
                        dp[i] = dp[i - 1] + dp[i - dp[i - 1] - 2] + 2;
                    }
                    if (i - dp[i - 1] - 1 == 0 && s.charAt(i - dp[i - 1] - 1) == '(') {
                        dp[i] = dp[i - 1] + 2;
                    }
                }

                ans = Math.max(ans, dp[i]);
            }
        }
        return ans;
    }
```

这里的dp[i]的意思是当前位置的前临近子串最长的有效括号长度，那这样就很好找他的状态转移方程大致可将情况分为这几种：

- 当s.charAt(i) == '('，这里就很好处理，因为没有任何一个有效的连续子串会以'('结尾，所以默认都做零
- 当s.charAt(i) == ')'，这里就要分情况讨论了：
  - 如果s.charAt(i-1) == '('，那么dp[i] = dp[i-2] + 2
  - 如果s.charAt(i-1) == ')'，这里就会复杂一点，核心逻辑就是先将前一个')'的连续有效的括号的位置省略掉，即去看i - dp[i-1] -1 这个位置的值，这里可以想像把中间已经匹配成功的子串丢掉，我们可以理解就是当前位置与i - dp[i-1] -1 相邻，看该位置是否是'(':
    - 是，则dp[i] = dp[i - 1] + dp[i - dp[i - 1] - 2] + 2;
    - 不是，则这个匹配子串不合法，可以直接置为0

这是核心逻辑，再处理一些边界条件就好了。

## Leetcode 第42题 -- 接雨水

题目描述：

> 给定 n 个非负整数表示每个宽度为 1 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。
>
>  
>
> 示例 1：
>
> 
>
> 输入：height = [0,1,0,2,1,0,1,3,2,1,2,1]
> 输出：6
> 解释：上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水（蓝色部分表示雨水）。 

题目分析：

这道题在我看了答案以后真的就不配叫做一到困难题，当然不看答案我好像也没想到思路

其实思路很简单，我们从左往右遍历，找到左边最高的墙，然后从右往左遍历，找到右边最高的墙，然后每一块位置应该正确接的雨水的数量就是左边与右边的墙的较小值减去本位置的高度就好了

```java
public int trap(int[] height) {
        int len = height.length;
        if(len == 0 || len == 1) return 0;

        int[] left = new int[len];
        int[] right = new int[len];

        left[0] = height[0];
        right[len-1] = height[len-1];

        for(int i = 1; i<len; i++){
            left[i] = Math.max(left[i-1],height[i]);
        }

        for(int j = len - 2; j>=0; j--){
            right[j] = Math.max(right[j+1],height[j]);
        }

        int ans = 0;
        for(int i = 0; i<len; i++){
            ans += Math.min(left[i],right[i]) - height[i];
        }

        return ans;
    }
```

