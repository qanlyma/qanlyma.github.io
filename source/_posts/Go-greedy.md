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

给你一个整数数组 nums，返回 nums 中作为摆动序列的最长子序列的长度。

```go
func wiggleMaxLength(nums []int) int {
    if len(nums) < 2 {
        return len(nums)
    }
    res := 1
    pre := nums[1] - nums[0]
    if pre != 0 {
        res++
    }
    for i := 2; i < len(nums); i++ {
        cur := nums[i] - nums[i-1]
        if pre >= 0 &&  cur < 0 || pre <= 0 &&  cur > 0 {
            res++
            pre = cur
        }
    }
    return res
}
```

### 2.2 [leetcode 55 题](https://leetcode.cn/problems/jump-game/)

跳跃游戏。

```go
func canJump(nums []int) bool {
    cover := 0
    for i := 0; i <= cover; i++ {
        cover = max(cover, nums[i] + i)
        if cover >= len(nums)-1 {
            return true
        }
    }
    return false
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

```go
func candy(ratings []int) int {
    need := make([]int, len(ratings))
    sum := 0
    for i := 0; i < len(ratings); i++ {
        need[i] = 1
    }
    // 1. 先从左到右，当右边的大于左边的就加 1
    for i := 0; i < len(ratings)-1; i++ {
        if ratings[i] < ratings[i+1] {
            need[i+1] = need[i] + 1
        }
    }
    // 2. 再从右到左，当左边的大于右边的就加 1
    for i := len(ratings)-1; i > 0; i-- {
        if ratings[i-1] > ratings[i] {
            need[i-1] = max(need[i-1], need[i]+1)
        }
    }
    // 3. 计算总共糖果
    for i := 0; i < len(ratings); i++ {
        sum += need[i]
    }
    return sum
}
```

### 2.5 [leetcode 452 题](https://leetcode.cn/problems/minimum-number-of-arrows-to-burst-balloons/)

用最少数量的箭引爆气球。

```go
func findMinArrowShots(points [][]int) int {
    sort.Slice(points, func(a, b int) bool {
        return points[a][0] < points[b][0]
    })
    res := 1
    right := points[0][1]

    for i := 1; i < len(points); i++ {
        if points[i][0] <= right {
            right = min(right, points[i][1])
        } else {
            right = points[i][1]
            res++
        }
    }
    return res
}
```

### 2.6 [leetcode 435 题](https://leetcode.cn/problems/non-overlapping-intervals/)

给定一个区间的集合 intervals ，其中 intervals[i] = [starti, endi]。返回需要移除区间的最小数量，使剩余区间互不重叠。

```go
func eraseOverlapIntervals(intervals [][]int) int {
    sort.Slice(intervals, func(a, b int) bool {
        return intervals[a][0] < intervals[b][0]
    })
    res := 0
    right := intervals[0][1]
    
    for i := 1; i < len(intervals); i++ {
        if intervals[i][0] < right {
            right = min(right, intervals[i][1])
            res++
        } else {
            right = intervals[i][1]
        }
    }
    return res
}
```

### 2.7 [leetcode 763 题](https://leetcode.cn/problems/partition-labels/)

给你一个字符串 s。我们要把这个字符串划分为尽可能多的片段，同一字母最多出现在一个片段中。

```go
func partitionLabels(s string) []int {
    var res []int
    bMap := make(map[byte]int)
    b := []byte(s)
    for i := 0; i < len(b); i++ {
        bMap[b[i]] = i
    }
    var maxPos, lenth int
    for i, v := range b {
        lenth++
        if bMap[v] > maxPos {
            maxPos = bMap[v]
        }
        if i == maxPos {
            res = append(res, lenth)
            lenth = 0
        }
    }
    return res
}
```

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