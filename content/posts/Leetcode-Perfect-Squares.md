---
title: "Leetcode 题解：完全平方数"
date: 2019-12-08T19:34:46+08:00
draft: true
categories:
- "Leetcode 题解"
tags:
- 数学
- 动态规划
keywords:
- "用动态规划求解完全平方和问题"
---

### 问题

给定正整数 n，找到若干个完全平方数（比如 1, 4, 9, 16, ...）使得它们的和等于 n。你需要让组成和的完全平方数的个数最少。

- 示例 1:
  
  输入: n = 12
  
  输出: 3
  
  解释: 12 = 4 + 4 + 4.

- 示例 2:
  
  输入: n = 13
  
  输出: 2
  
  解释: 13 = 4 + 9.
  
来源：力扣（LeetCode）

链接：[https://leetcode-cn.com/problems/perfect-squares](https://leetcode-cn.com/problems/perfect-squares)

著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

### 思路

用动态规划解决这道题，记f(i)，为将i表达成若干个完全平方数和所需要的完全平方数的最小个数。

从1到n求每个整数i，对应的f(i)。对于整数i，j从1开始，j*j 不超过 i，状态转移方程：

```matlab
f(i) = min(f(i), f(i-j*j)+1)
```

### 代码

```go
func numSquares(n int) int {
  f := make([]int, n+1)

  for i := 1; i <= n; i++ {
      // 最坏情况，等于i个1的和
      f[i] = i
      for j := 1; j*j <= i; j++ {
        f[i] = min(f[i], f[i - j*j]+1)
      }
  }
  return f[n]
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```
