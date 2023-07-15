---
title: 「Go」 数组
category_bar: true
date: 2023-06-11 20:50:10
tags:
categories: Golang
banner_img:
---

Golang 数组相关题目。

<!-- more -->

## 1 二分查找

### [leetcode 704 题](https://leetcode.cn/problems/binary-search/)

这道题目的前提是数组为**有序数组**，同时题目还强调数组中**无重复元素**，因为一旦有重复元素，使用二分查找法返回的元素下标可能不是唯一的，这些都是使用二分法的前提条件。

写二分法，区间的定义一般为两种，左闭右闭即 `[left, right]`，或者左闭右开即 `[left, right)`。

第一种写法，我们定义 target 是在一个在左闭右闭的区间里，也就是 `[left, right]`：

* `for left <= right` 要使用 `<=` ，因为 `left == right` 是有意义的
* `if nums[middle] > target` `right` 要赋值为 `middle - 1`，因为当前这个 `nums[middle]` 一定不是 target 

```go
func search(nums []int, target int) int {
    i, j := 0, len(nums)-1
    for i <= j {
        m := (i+j) / 2
        if nums[m] < target {
            i = m + 1
        } else if nums[m] > target {
            j = m - 1
        } else {
            return m
        }
    }
    return -1
}
```

* 暴力解法时间复杂度：O(n)
* 二分法时间复杂度：O(logn)

## 2 快慢指针

### [leetcode 27 题](https://leetcode.cn/problems/remove-element/)

给你一个数组 nums 和一个值 val，你需要**原地**移除所有数值等于 val 的元素，并返回移除后数组的新长度。

使用 O(1) 额外空间并原地修改输入数组。数组的元素在内存地址中是连续的，不能单独删除数组中的某个元素，只能覆盖。

这个题目暴力解法就是两层 for 循环，一个 for 循环遍历数组元素，第二个 for 循环更新数组。

使用快慢指针在一个 for 循环下完成两个 for 循环的工作，时间复杂度 O(n) ：

* 快指针：寻找新数组的元素 ，新数组就是不含有目标元素的数组
* 慢指针：指向更新新数组下标的位置

```go
func removeElement(nums []int, val int) int {
    slow, fast := 0, 0 
    for fast < len(nums) {
        if nums[fast] != val {
            nums[slow] = nums[fast]
            slow++
        } 
        fast++
    }
    return slow
}
```

* 暴力解法时间复杂度：O(n^2)
* 双指针时间复杂度：O(n)

## 3 双指针法

### [leetcode 977 题](https://leetcode.cn/problems/squares-of-a-sorted-array/)

给你一个按**非递减顺序**排序的整数数组 nums，返回每个数字的平方组成的新数组，要求也按非递减顺序排序。

数组**有序**可以考虑用双指针：

1. 定义一个新数组 result，和 A 数组一样的大小，让 k 指向 result 数组终止位置。

2. 如果 `A[i] * A[i] < A[j] * A[j]` 那么 `result[k] = A[j] * A[j]`，`j--`。

3. 如果 `A[i] * A[i] >= A[j] * A[j]` 那么 `result[k] = A[i] * A[i]`，`i++`。

## 4 滑动窗口

### [leetcode 209 题](https://leetcode.cn/problems/minimum-size-subarray-sum/)

实现滑动窗口，主要确定如下三点：

* 窗口内是什么？
* 如何移动窗口的起始位置？（本题使用 for 循环一次移动多位）
* 如何移动窗口的结束位置？（本题一次移动一位）

* 暴力解法时间复杂度：O(n^2)
* 滑动窗口时间复杂度：O(n)