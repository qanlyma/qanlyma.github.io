---
title: 「Go」 贪心算法
category_bar: true
date: 2023-07-19 15:38:27
tags:
categories: Golang
banner_img:
---

贪心的本质是选择每一阶段的局部最优，从而达到全局最优。

<!-- more -->

## 1 理论

贪心算法并没有固定的套路。唯一的难点就是如何**通过局部最优，推出整体最优**。

贪心算法一般分为如下四步：

1. 将问题分解为若干个子问题
2. 找出适合的贪心策略
3. 求解每一个子问题的最优解
4. 将局部最优解堆叠成全局最优解

## 2 题目

### 2.1 [leetcode 376 题](https://leetcode.cn/problems/wiggle-subsequence/)

给你一个整数数组 nums，返回 nums 中作为摆动序列的 最长子序列的长度。

### 2.2 [leetcode 55 题](https://leetcode.cn/problems/jump-game/)

跳跃游戏。

```go
func canJump(nums []int) bool {
    cover := 0
    n := len(nums) - 1
    for i := 0; i <= cover; i++ { // 每次与覆盖值比较
        cover = max(i+nums[i], cover) // 每走一步都将 cover 更新为最大值
        if cover >= n {
            return true
        }
    }
    return false
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

### 2.3 [leetcode 134 题](https://leetcode.cn/problems/gas-station/)

加油站。

每个加油站的剩余量 rest[i] 为 gas[i] - cost[i]。如果总油量减去总消耗大于等于零那么一定可以跑完一圈，说明各个站点的加油站剩油量 rest[i] 相加一定是大于等于零的。

i 从 0 开始累加 rest[i]，和记为 curSum，一旦 curSum 小于零，说明 [0, i] 区间都不能作为起始位置，因为这个区间选择任何一个位置作为起点，到i这里都会断油，那么起始位置从 i+1 算起，再从 0 计算 curSum。

```go
func canCompleteCircuit(gas []int, cost []int) int {
	curSum := 0
	totalSum := 0
	start := 0
	for i := 0; i < len(gas); i++ {
		curSum += gas[i] - cost[i]
		totalSum += gas[i] - cost[i]
		if curSum < 0 {
			start = i+1
			curSum = 0
		}
	}
	if totalSum < 0 {
		return -1
	}
	return start
}
```

### 2.4 [leetcode 135 题](https://leetcode.cn/problems/candy/)

分发糖果。

### 2.5 [leetcode 452 题](https://leetcode.cn/problems/minimum-number-of-arrows-to-burst-balloons/)

用最少数量的箭引爆气球。

### 2.6 [leetcode 435 题](https://leetcode.cn/problems/non-overlapping-intervals/)

给定一个区间的集合 intervals ，其中 intervals[i] = [starti, endi]。返回需要移除区间的最小数量，使剩余区间互不重叠。

### 2.7 [leetcode 763 题](https://leetcode.cn/problems/partition-labels/)

划分字母区间。

### 2.8 [leetcode 968 题](https://leetcode.cn/problems/binary-tree-cameras/)

监控二叉树。

```go
func minCameraCover(root *TreeNode) (res int) {
    // 节点状态：0 表示未覆盖，1 表示有监控，2 表示被覆盖
    var DFS func(node *TreeNode) int
    DFS = func(node *TreeNode) int {
        if node == nil {
            return 2
        }

        left := DFS(node.Left)
        right := DFS(node.Right)
        if left == 0 || right == 0 {
            res++
            return 1
        }
        if left == 1 || right == 1 {
            return 2
        }
        return 0
    }

    if DFS(root) == 0 {
        res++
    }
    return
}
```