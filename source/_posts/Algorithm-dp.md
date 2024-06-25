---
title: 「算」 14. 动态规划
category_bar: true
date: 2023-07-23 20:47:39
tags:
categories: 算法数构
banner_img:
---

Dynamic Programming，简称 DP，如果某一问题有很多重叠子问题，使用动态规划是最有效的。

<!-- more -->

## 1 理论

动态规划中每一个状态是由上一个状态推导出来的，贪心没有状态推导，是从局部直接选最优的。

### 1.1 五步曲

1. 确定 dp 数组（dp table）以及下标的含义
2. 确定递推公式
3. dp 数组如何初始化
4. 确定遍历顺序
5. 举例推导 dp 数组

### 1.2 背包问题

![](1.png)

#### 01 背包

有 n 件物品和一个最多能背重量为 w 的背包。第 i 件物品的重量是 weight[i]，得到的价值是 value[i]。每件物品只能用一次，求解将哪些物品装入背包里物品价值总和最大。

对于背包问题，有一种写法是使用二维数组，即 dp[i][j] 表示从下标为 [0-i] 的物品里任意取，放进容量为 j 的背包，价值总和最大是多少。

![](2.png)

那么可以有两个方向推出来 dp[i][j]：
1. 不放物品 i：由 dp[i - 1][j] 推出，即背包容量为 j，里面不放物品 i 的最大价值，此时 dp[i][j] 就是 dp[i - 1][j]。
2. 放物品 i：由 dp[i - 1][j - weight[i]] 推出，dp[i - 1][j - weight[i]] 为背包容量为 j - weight[i] 的时候不放物品 i 的最大价值，那么 dp[i - 1][j - weight[i]] + value[i] 就是背包放物品i得到的最大价值。

递推公式：`dp[i][j] = max(dp[i - 1][j], dp[i - 1][j - weight[i]] + value[i])`

初始化：

![](3.png)

```go
func bag_problem1(weight, value []int, bagweight int) int {
	// 定义 dp 数组
	dp := make([][]int, len(weight))
	for i, _ := range dp {
		dp[i] = make([]int, bagweight+1)
	}
	// 初始化
	for j := bagweight; j >= weight[0]; j-- {
		dp[0][j] = dp[0][j-weight[0]] + value[0]
	}
	// 递推公式
	for i := 1; i < len(weight); i++ {
	    for j := 0; j <= bagweight; j++ {
	        if j < weight[i] {
	            dp[i][j] = dp[i-1][j]
	        } else {
	            dp[i][j] = max(dp[i-1][j], dp[i-1][j-weight[i]]+value[i])
	        }
	    }
	}
	return dp[len(weight)-1][bagweight]
}
```

#### 滚动数组

滚动数组可以把二维 dp 降为一维 dp。

在使用二维数组的时候，递推公式：dp[i][j] = max(dp[i - 1][j], dp[i - 1][j - weight[i]] + value[i])

其实可以发现如果把 dp[i - 1] 那一层拷贝到 dp[i] 上，表达式完全可以是：dp[i][j] = max(dp[i][j], dp[i][j - weight[i]] + value[i])

与其把 dp[i - 1] 这一层拷贝到 dp[i] 上，不如只用一个一维数组了，只用 dp[j]。

在一维dp数组中，dp[j] 表示：**容量为 j 的背包，所背的物品价值可以最大为 dp[j]**。

递推公式：`dp[j] = max(dp[j], dp[j - weight[i]] + value[i])`

初始化：都初始化为 0 

二维 dp 遍历的时候，背包容量是从小到大，而一维 dp 遍历的时候，背包是从大到小。倒序遍历是为了保证物品 i 只被放入一次。

```go
func bag_problem2(weight, value []int, bagWeight int) int {
	dp := make([]int, bagWeight+1)
	for i := 0; i < len(weight); i++ {
		// 这里必须倒序，区别二维，因为二维 dp 保存了 i 的状态
		for j := bagWeight; j >= weight[i]; j-- {
			dp[j] = max(dp[j], dp[j-weight[i]] + value[i])
		}
	}
	return dp[bagWeight]
}
```

#### 完全背包

01 背包和完全背包唯一不同就是体现在遍历顺序上，01 背包内嵌的循环是从大到小遍历，为了保证每个物品仅被添加一次。而完全背包的物品是可以添加多次的，所以要从小到大去遍历。

物品和背包先遍历哪个都可以。

* 如果求组合数就是外层 for 循环遍历物品，内层 for 遍历背包。

* 如果求排列数就是外层 for 遍历背包，内层 for 循环遍历物品。

## 2 题目

### 2.1 [leetcode 509 题](https://leetcode.cn/problems/fibonacci-number/)

斐波那契数。

```go
func fib1(n int) int {
    if n == 0 {
        return 0
    }
    dp := make([]int, n+1)
    dp[0], dp[1] = 0, 1
    for i := 2; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }
    return dp[n]
}

func fib2(n int) int {
    if n < 2 {
        return n
    }
    
    return fib(n-1) + fib(n-2) 
}
```

### 2.2 [leetcode 70 题](https://leetcode.cn/problems/climbing-stairs/)

爬楼梯。

```go
func climbStairs(n int) int {
    dp := make([]int, n+1)
    dp[0], dp[1] = 1, 1
    for i := 2; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }
    return dp[n]
}
```

### 2.2 [leetcode 63 题](https://leetcode.cn/problems/unique-paths-ii/)

一个机器人位于一个 m x n 网格的左上角，机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角。

现在考虑网格中有障碍物。那么从左上角到右下角将会有多少条不同的路径？

```go
func uniquePathsWithObstacles(obstacleGrid [][]int) int {
    dp := make([][]int, len(obstacleGrid))
    way := 1
    for i, _ := range dp {
        dp[i] = make([]int, len(obstacleGrid[0]))
        if obstacleGrid[i][0] == 1 {
            way = 0
        }
        dp[i][0] = way
    }
    way = 1
    for i, _ := range dp[0] {
        if obstacleGrid[0][i] == 1 {
            way = 0
        }
        dp[0][i] = way
    }

    for i := 1; i < len(obstacleGrid); i++ {
        for j := 1; j < len(obstacleGrid[0]); j++ {
            if obstacleGrid[i][j] == 1 {
                dp[i][j] = 0
            } else {
                dp[i][j] = dp[i-1][j] + dp[i][j-1]
            }
        }
    }
    return dp[len(obstacleGrid)-1][len(obstacleGrid[0])-1]
}
```

### 2.3 [leetcode 343 题](https://leetcode.cn/problems/integer-break/)

给定一个正整数 n，将其拆分为 k 个正整数的和（k >= 2），并使这些整数的乘积最大化。

1. dp[i]：分拆数字 i，可以得到的最大乘积为 dp[i]。

2. 从 1 遍历 j，有两种渠道得到 dp[i]：
    * 一个是 j * (i - j) 直接相乘
    * 一个是 j * dp[i - j]，相当于是拆分 (i - j)
    * j 是从 1 开始遍历，拆分 j 的情况，在遍历 j 的过程中其实都计算过了。

```go
func integerBreak(n int) int {
    dp := make([]int, n+1)
    dp[1] = 1
    for i := 2; i <= n; i++ {
        for j := 1; j < i; j++ {
            dp[i] = max(dp[i], max(dp[i-j]*j, (i-j)*j))
        }
    }
    return dp[n]
}
```

### 2.3 [leetcode 96 题](https://leetcode.cn/problems/unique-binary-search-trees/submissions/)

不同的二叉搜索树。

```go
func numTrees(n int) int {
    dp := make([]int, n+1)
    dp[0], dp[1] = 1, 1
    for i := 2; i <= n; i++ {
        for j := 0; j < i; j++ {
            dp[i] += dp[i-j-1] * dp[j]
        }
    }
    return dp[n]
}
```

### 2.4 [leetcode 416 题](https://leetcode.cn/problems/partition-equal-subset-sum/)

给你一个只包含正整数的非空数组 nums。请你判断是否可以将这个数组分割成两个子集，使得两个子集的元素和相等。

```go
func canPartition(nums []int) bool {
    var sum int
    for _, v := range nums {
        sum += v
    }
    if sum%2 != 0 {
        return false
    }
    target := sum / 2
    dp := make([]int, target+1)
    for i := 0; i < len(nums); i++ {
        for j := target; j >= nums[i]; j-- {
            dp[j] = max(dp[j-nums[i]]+nums[i], dp[j])
        }
    }
    return target == dp[target]
}
```

### 2.5 [leetcode 494 题](https://leetcode.cn/problems/target-sum/)

目标和。

left 组合 - right 组合 = target。

left + right = sum，而 sum 是固定的。right = sum - left

left - (sum - left) = target 推导出 left = (target + sum) / 2 。

之前都是求容量为j的背包，最多能装多少。本题则是装满有几种方法。其实这就是一个组合问题了。

dp[j] 表示：填满 j（包括 j）这么大容积的包，有 dp[j] 种方法。

递推公式：`dp[j] += dp[j - nums[i]]`

```go
func findTargetSumWays(nums []int, target int) int {
    var sum int
    for _, v := range nums {
        sum += v
    }
    if target < 0 {
        target = -target
    }
    if (target + sum) % 2 == 1 || target > sum {
        return 0
    }
    bag := (target + sum) / 2 
    dp := make([]int, bag+1)
    dp[0] = 1
    for i := 0; i < len(nums); i++ {
        for j := bag; j >= nums[i]; j-- {
            dp[j] += dp[j-nums[i]]
        }
    }
    return dp[bag]
}
```

### 2.6 [leetcode 518 题](https://leetcode.cn/problems/coin-change-ii/)

给你一个整数数组 coins 表示不同面额的硬币，另给一个整数 amount 表示总金额。请你计算并返回可以凑成总金额的硬币组合数。

完全背包 + 组合。

```go
func change(amount int, coins []int) int {
    dp := make([]int, amount+1)
    dp[0] = 1
    for i := 0; i < len(coins); i++ {
        for j := coins[i]; j <= amount; j++ {
            dp[j] += dp[j-coins[i]]
        }
    }
    return dp[amount]
}
```

### 2.7 [leetcode 377 题](https://leetcode.cn/problems/combination-sum-iv/)

给定一个由正整数组成且不存在重复数字的数组，找出和为给定目标正整数的组合的个数。

完全背包 + 排列。

```go
func combinationSum4(nums []int, target int) int {
    dp := make([]int, target+1)
    dp[0] = 1
    for i := 1; i <= target; i++ {
        for j := 0; j < len(nums); j++ {
            if nums[j] <= i {
                dp[i] += dp[i-nums[j]]
            }
        }
    }
    return dp[target]
}
```

### 2.8 [leetcode 139 题](https://leetcode.cn/problems/word-break/)

单词拆分。

```go
func wordBreak(s string,wordDict []string) bool  {
	wordDictSet := make(map[string]bool)
	for _, w := range wordDict {
		wordDictSet[w] = true
	}
	dp := make([]bool, len(s)+1)
	dp[0] = true
	for i := 1; i <= len(s); i++ {
		for j := 0; j < i; j++ {
			if dp[j] && wordDictSet[s[j:i]] { 
				dp[i] = true
				break
			}
		}
	}
	return dp[len(s)]
}
```

### 2.9 [leetcode 337 题](https://leetcode.cn/problems/house-robber-iii/)

打劫二叉树。

```go
func rob(root *TreeNode) int {
    var dfs func(node *TreeNode) []int 
    dfs = func(node *TreeNode) []int {
        if node == nil {
            return []int{0, 0}
        }
        dp := make([]int, 2)
        left := dfs(node.Left)
        right := dfs(node.Right)
        dp[0] = max(left[0], left[1]) + max(right[0], right[1])
        dp[1] = left[0] + right[0] + node.Val
        return dp
    }
    res := dfs(root)
    return max(res[0], res[1])
} 
```

### 2.10 [leetcode 121 题](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock/)

买卖股票的最佳时机。

```go
// DP
func maxProfit(prices []int) int {
    dp := make([][]int, len(prices))
    for i, _ := range dp {
        dp[i] = make([]int, 2)
    }
    dp[0][0], dp[0][1] = 0, -prices[0]
    for i := 1; i < len(prices); i++ {
        dp[i][0] = max(dp[i-1][0], dp[i-1][1] + prices[i])
        dp[i][1] = max(dp[i-1][1], -prices[i])
    }
    return dp[len(prices)-1][0]
}

// 贪心
func maxProfit(prices []int) int {
    min := prices[0]
    res := 0
    for i := 1; i < len(prices); i++ {
        if prices[i] - min > res {
            res = prices[i]-min
        }
        if min > prices[i] {
            min = prices[i]
        }
    }
    return res
}
```

### 2.11 [leetcode 718 题](https://leetcode.cn/problems/maximum-length-of-repeated-subarray/)

给两个整数数组 nums1 和 nums2，返回两个数组中公共的、长度最长的子数组的长度。

```go
func findLength(nums1 []int, nums2 []int) int {
    var res int
    dp := make([][]int, len(nums1)+1)
    for i, _ := range dp {
        dp[i] = make([]int, len(nums2)+1)
    }

    for i := 1; i <= len(nums1); i++ {
        for j := 1; j <= len(nums2); j++ {
            if nums1[i-1] == nums2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } 
            res = max(res, dp[i][j])
        }
    }
    return res
}
```

### 2.12 [leetcode 1143 题](https://leetcode.cn/problems/longest-common-subsequence/)

给定两个字符串 text1 和 text2，返回这两个字符串的最长公共子序列的长度。

```go
func longestCommonSubsequence(text1 string, text2 string) int {
    dp := make([][]int, len(text1)+1)
    for i, _ := range dp {
        dp[i] = make([]int, len(text2)+1)
    } 

    for i := 1; i <= len(text1); i++ {
        for j := 1; j <= len(text2); j++ {
            if text1[i-1] == text2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[len(text1)][len(text2)]
}
```

### 2.12 [leetcode 1035 题](https://leetcode.cn/problems/uncrossed-lines/)

不相交的线。

```go
func maxUncrossedLines(nums1 []int, nums2 []int) int {
    dp := make([][]int, len(nums1)+1)
    for i, _ := range dp {
        dp[i] = make([]int, len(nums2)+1)
    }

    for i := 1; i <= len(nums1); i++ {
        for j := 1; j <= len(nums2); j++ {
            if nums1[i-1] == nums2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[len(nums1)][len(nums2)]
}
```

### 2.13 [leetcode 53 题](https://leetcode.cn/problems/maximum-subarray/)

最大子数组和。

最大连续子数组的和要用一个变量记录下来比较，不连续的子序列则不需要，直接输出 dp 的最后一个值即可。

```go
func maxSubArray(nums []int) int {
    dp := make([]int, len(nums))
    res := nums[0]
    dp[0] = nums[0]
    for i := 1; i < len(nums); i++ {
        dp[i] = max(dp[i-1]+nums[i], nums[i])
        if dp[i] > res {
            res = dp[i]
        }
    }
    return res
}
```

### 2.14 [leetcode 72 题](https://leetcode.cn/problems/edit-distance/)

编辑距离。

```go
func minDistance(word1 string, word2 string) int {
    // dp[i][j] 表示以下标 i-1 为结尾的字符串 word1，和以下标 j-1 为结尾的字符串 word2，最近编辑距离为 dp[i][j]
    dp := make([][]int, len(word1)+1)
    for i, _ := range dp {
        dp[i] = make([]int, len(word2)+1)
    }
    for i := 0; i <= len(word1); i++ {
        dp[i][0] = i
    } 
    for j := 0; j <= len(word2); j++ {
        dp[0][j] = j
    }

    for i := 1; i <= len(word1); i++ {
        for j := 1; j <= len(word2); j++ {
            if word1[i-1] == word2[j-1] {
                dp[i][j] = dp[i-1][j-1]
            } else {
                dp[i][j] = min(min(dp[i][j-1], dp[i-1][j])+1, dp[i-1][j-1]+1)
            }
        }
    }
    return dp[len(word1)][len(word2)]
}
```

### 2.15 [leetcode 132 题](https://leetcode.cn/problems/palindrome-partitioning-ii/)

分割回文串 II。

深信服笔试题，我当时用的 dfs 忘记通过多少了（反正力扣上是超时的），后来面试的时候又被拿出来手撕，记录一下。

```go
func minCut(s string) int {
    dp := make([]int, len(s)+1)
    dp[0] = -1
    dp[1] = 0
    for i := 1; i <= len(s); i++ {
        dp[i] = i-1
        j := 0
        for j < i {
            if isHW(s[j:i]) {
                dp[i] = min(dp[i], dp[j]+1)
            }
            j++
        }
    }
    return dp[len(s)]
}

func isHW(s string) bool {
    for i, j := 0, len(s)-1; i <= j; i, j = i+1, j-1 {
        if s[i] != s[j] {
            return false
        }
    }
    return true
}
```