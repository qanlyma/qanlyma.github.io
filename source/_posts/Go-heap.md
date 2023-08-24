---
title: 「Go」 10 堆结构
category_bar: true
date: 2023-07-16 15:49:50
tags:
categories: Golang
banner_img:
---

本文以 [leetcode 347 题](https://leetcode.cn/problems/top-k-frequent-elements/)为例，介绍一下堆结构，参考 [Carl 的教程](https://programmercarl.com/0347.%E5%89%8DK%E4%B8%AA%E9%AB%98%E9%A2%91%E5%85%83%E7%B4%A0.html)。

<!-- more -->

## 题目介绍

`给你一个整数数组 nums 和一个整数 k ，请你返回其中出现频率前 k 高的元素。你可以按任意顺序返回答案。你所设计算法的时间复杂度必须优于 O(n*logn)，其中 n 是数组大小。`

## 思路

这道题目主要涉及到如下三块内容：

1. 要统计元素出现频率
2. 对频率排序
3. 找出前K个高频元素

首先统计元素出现的频率，这一类的问题可以使用 map 来进行统计。

然后是对频率进行排序，这里我们可以使用**优先级队列**。

**堆**就是优先级队列的一种实现。

## 堆（heap）

堆是一棵**完全二叉树**，树中每个结点的值都不小于（或不大于）其左右孩子的值。 如果父亲结点是大于等于左右孩子就是**大顶堆**（根节点最大），小于等于左右孩子就是**小顶堆**（根节点最小）。

![heap](3.png)

堆本身就是一个数组，因此创建一个数组来创建堆。

![存储](4.png)

假设当前元素的索引位置为 i，可以得到规律：

```heap
parent(i) = i/2（取整）
left child(i) = 2*i
right child(i) = 2*i + 1
```

### 插入元素

![插入](1.png)

在完全二叉树中，插入的节点与它的父节点相比，如果比父节点小，就交换它们的位置，再往上和父节点相比，如果比父节点小，再交换位置，直到比父节点大为止。

在数组中，插入的节点与 n/2 位置的节点相比，如果比 n/2 位置的节点小，就交换它们的位置，再往前与 n/4 位置的节点相比，如果比 n/4 位置的节点小，再交换位置，直到比 n/(2^x) 位置的节点大为止。

这就是插入元素时进行的堆化，也叫**自下而上的堆化**。

插入元素的时间复杂度为 O(log n)。

### 删除元素

![删除](2.png)

在完全二叉树中，移除根节点首先把最后一个节点放到堆顶，然后与左右子节点中**小**的交换位置（因为是小顶堆），依次往下，直到其比左右子节点都小为止。

在数组中，把最后一个元素移到下标为 1 的位置，然后与下标为 2 和 3 的位置对比，发现 8 比 2 大，且 2 是 2 和 3 中间最小的，所以与 2 交换位置；然后再下标为 4 和 5 的位置对比，发现 8 比 5 大，且 5 是 5 和 7 中最小的，所以与 5 交换位置，没有左右子节点了，堆化结束。即 `parent = child; child = parent*2;`。

这就是删除元素时进行的堆化，也叫**自上而下的堆化**。

删除元素的时间复杂度为 O(log n)。

## 题解

此题使用小顶堆，因为要统计最大前 k 个元素，只有小顶堆每次将最小的元素弹出，最后小顶堆里积累的才是前 k 个最大元素。

![过程](5.png)

## 代码实现

使用 go 标准库给我们提供的 heap，需要实现这些接口定义的方法：

* Len() int
* Less(i, j int) bool
* Swap(i, j int)
* Push(x interface{})
* Pop() interface{}

```go
// 方法一：小顶堆
func topKFrequent(nums []int, k int) []int {
    map_num := map[int]int{}
    // 记录每个元素出现的次数
    for _, item := range nums{
        map_num[item]++
    }

    // 初始化 heap
    h := &IHeap{}
    heap.Init(h)
    // 所有元素入堆，堆的长度为 k
    for key, value := range map_num {
        heap.Push(h, [2]int{key, value})
        if h.Len() > k {
            // heap.Pop 并非下方定义的 h.Pop
            heap.Pop(h)
        }
    }

    res := make([]int, k)
    // 按顺序返回堆中的元素
    for i := 0; i < k; i++ {
        res[k-i-1] = heap.Pop(h).([2]int)[0]
    }
    return res
}

// 构建小顶堆
type IHeap [][2]int

func (h IHeap) Len()int {
    return len(h)
}

func (h IHeap) Less(i, j int) bool {
    // 如果 h[i][1] < h[j][1] 生成的就是小根堆，h[i][1] > h[j][1] 生成的就是大根堆
    return h[i][1] < h[j][1]
}

func (h IHeap) Swap(i, j int) {
    h[i], h[j] = h[j], h[i]
}

func (h *IHeap) Push(x interface{}) {
    *h = append(*h, x.([2]int))
}

func (h *IHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0:n-1]
    return x
}
```

另一个解法是先使用 map 存储元素与出现次数，然后将元素放入切片，再按照频率对切片排序。

```go
// 方法二：利用 O(nlogn) 排序
func topKFrequent(nums []int, k int) []int {
    ans := []int{}
    map_num := map[int]int{}
    for _, item := range nums {
        map_num[item]++
    }
    for key, _ := range map_num {
        ans = append(ans, key)
    }

    // 切片排序
    sort.Slice(ans, func (a, b int) bool {
        // 第一个参数是：待排序数据
        // 第二个参数是：排序判断方法
        // 返回值：代表 a, b 是否交换，true：交换，false：不交换
        return map_num[ans[a]] > map_num[ans[b]]
        // < 是升序；> 是降序
    })
    return ans[:k]
}
```

当匿名函数返回值为 true 时，表示 a 索引处的元素应该在 b 索引处的元素之前。相反，当返回值为 false 时，表示 a 索引处的元素应该在 b 索引处的元素之后。