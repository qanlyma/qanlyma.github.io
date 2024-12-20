---
title: 「算」 02. 数组
category_bar: true
date: 2023-03-11 20:50:10
tags:
categories: 算法数构
banner_img:
---

数组相关题目。

<!-- more -->

## 1 二分查找

### 1.1 [二分法](https://leetcode.cn/problems/binary-search/)

这道题目的前提是数组为**有序数组**，同时题目还强调数组中**无重复元素**，因为一旦有重复元素，使用二分查找法返回的元素下标可能不是唯一的，这些都是使用二分法的前提条件。

写二分法，区间的定义一般为两种，左闭右闭即 `[left, right]`，或者左闭右开即 `[left, right)`。

第一种写法，我们定义 target 是在一个在左闭右闭的区间里，也就是 `[left, right]`：

* `for left <= right` 要使用 `<=` ，因为 `left == right` 是有意义的
* `if nums[middle] > target` `right` 要赋值为 `middle - 1`，因为当前这个 `nums[middle]` 一定不是 target 

```go
func search(nums []int, target int) int {
    i, j := 0, len(nums)-1
    for i <= j {
        m := i + (j-i)/2
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

```java
class Solution {
    public int search(int[] nums, int target) {
        int len = nums.length;
        for (int i = 0, j = len-1; i <= j; ) {
            int k = i + (j-i)/2;
            if (nums[k] == target) {
                return k;
            } else if (nums[k] < target) {
                i = k + 1;
            } else {
                j = k - 1;
            }
        }
        return -1;
    }
}
```

* 暴力解法时间复杂度：O(n)
* 二分法时间复杂度：O(logn)

### 1.2 [查找元素](https://leetcode.cn/problems/find-first-and-last-position-of-element-in-sorted-array/)

在排序数组中查找元素的第一个和最后一个位置。

```go
func searchRange(nums []int, target int) []int {
    l, r := getLeft(nums, target), getRight(nums, target)
    if l == -2 || r == -2 {
        return []int{-1, -1}
    }
    if r - l > 1 {
        return []int{l+1, r-1}
    }
    return []int{-1, -1}
}

func getLeft(nums []int, target int) int {
    res := -2
    i, j := 0, len(nums)-1
    for i <= j {
        m := i + (j-i)/2
        if nums[m] < target {
            i = m + 1
        } else {
            j = m - 1
            res = j
        }
    }
    return res
}

func getRight(nums []int, target int) int {
    res := -2
    i, j := 0, len(nums)-1
    for i <= j {
        m := i + (j-i)/2
        if nums[m] > target {
            j = m - 1
        } else {
            i = m + 1
            res = i
        }
    }
    return res
}
```

## 2 快慢指针

### 2.1 [移除元素](https://leetcode.cn/problems/remove-element/)

给你一个数组 nums 和一个值 val，你需要**原地**移除所有数值等于 val 的元素，并返回移除后数组的新长度。

数组的元素在内存地址中是连续的，不能单独删除数组中的某个元素，只能覆盖。这个题目暴力解法就是两层 for 循环，一个 for 循环遍历数组元素，第二个 for 循环更新数组。

使用快慢指针可以在一个 for 循环下完成两个 for 循环的工作：

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

### 3.1 [有序数组的平方](https://leetcode.cn/problems/squares-of-a-sorted-array/)

给你一个按**非递减顺序**排序的整数数组 nums，返回每个数字的平方组成的新数组，要求也按非递减顺序排序。

数组**有序**可以考虑用双指针：

1. 定义一个新数组 result，和 A 数组一样的大小，让 k 指向 result 数组终止位置。

2. 如果 `A[i] * A[i] < A[j] * A[j]` 那么 `result[k] = A[j] * A[j]`，`j--`。

3. 如果 `A[i] * A[i] >= A[j] * A[j]` 那么 `result[k] = A[i] * A[i]`，`i++`。

```go
func sortedSquares(nums []int) []int {
    res := make([]int, len(nums))
    pos := len(nums) - 1
    for i, j := 0, len(nums)-1; i <= j; {
        if nums[i] * nums[i] > nums[j] * nums[j] {
            res[pos] = nums[i] * nums[i]
            i++
        } else {
            res[pos] = nums[j] * nums[j]
            j--
        }
        pos--
    }
    return res
}
```

### 3.2 [比较含退格的字符串](https://leetcode.cn/problems/backspace-string-compare/)

给定 s 和 t 两个字符串，当它们分别被输入到空白的文本编辑器后，如果两者相等，返回 true 。`#` 代表退格字符。

```go
func backspaceCompare(s, t string) bool {
    skipS, skipT := 0, 0
    i, j := len(s)-1, len(t)-1
    for i >= 0 || j >= 0 {
        for i >= 0 {
            if s[i] == '#' {
                skipS++
                i--
            } else if skipS > 0 {
                skipS--
                i--
            } else {
                break
            }
        }
        for j >= 0 {
            if t[j] == '#' {
                skipT++
                j--
            } else if skipT > 0 {
                skipT--
                j--
            } else {
                break
            }
        }
        if i >= 0 && j >= 0 {
            if s[i] != t[j] {
                return false
            }
        } else if i >= 0 || j >= 0 {
            return false
        }
        i--
        j--
    }
    return true
}
```

普通方法是用栈模拟操作，空间复杂度为 O(m + n)。可以使用从后向前的双指针来优化。

## 4 滑动窗口

### 4.1 [长度最小的子数组](https://leetcode.cn/problems/minimum-size-subarray-sum/)

给定一个含有 n 个正整数的数组和一个正整数 target。找出该数组中满足其总和大于等于 target 的长度最小的子数组并返回其长度。如果不存在符合条件的子数组，返回 0。

实现滑动窗口，主要确定如下三点：

1. 窗口内是什么？
2. 如何移动窗口的起始位置？（本题使用 for 循环一次移动多位）
3. 如何移动窗口的结束位置？（本题一次移动一位）

```go
func minSubArrayLen(target int, nums []int) int {
    res := len(nums) + 1
    l, r, sum := 0, 0, 0
    for r < len(nums) {
        sum += nums[r]
        for sum >= target {
            if res > (r - l + 1) {
                res = r - l + 1
            }
            sum -= nums[l]
            l++
        } 
        r++
    }
    if res == len(nums) + 1 {
        return 0
    }
    return res
}
```

* 暴力解法时间复杂度：O(n^2)
* 滑动窗口时间复杂度：O(n)

## 5 模拟过程

### 5.1 [螺旋矩阵 II](https://leetcode.cn/problems/spiral-matrix-ii/)

给你一个正整数 n，生成一个包含 1 到 n^2 所有元素，且元素按顺时针顺序螺旋排列的 n x n 正方形矩阵 matrix。

```go
func generateMatrix(n int) [][]int {
    top, bottom := 0, n-1
    left, right := 0, n-1
    num := 1
    tar := n * n
    matrix := make([][]int, n)
    for i := 0; i < n; i++ {
        matrix[i] = make([]int, n)
    }
    for num <= tar {
        for i := left; i <= right; i++ {
            matrix[top][i] = num
            num++
        }
        top++
        for i := top; i <= bottom; i++ {
            matrix[i][right] = num
            num++
        }
        right--
        for i := right; i >= left; i-- {
            matrix[bottom][i] = num
            num++
        }
        bottom--
        for i := bottom; i >= top; i-- {
            matrix[i][left] = num
            num++
        }
        left++
    }
    return matrix
}
```

## 6 查找算法

### 6.1 [寻找文件副本](https://leetcode.cn/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/)

设备中存有 n 个文件，文件 id 记于数组 documents。若文件 id 相同，则定义为该文件存在副本。请返回任一存在副本的文件 id。

```go
func findRepeatDocument(documents []int) int {
    for i := 0; i < len(documents); {
        if documents[i] == i {
            i++
            continue
        }
        if documents[i] == documents[documents[i]] {
            return documents[i]
        }
        documents[i], documents[documents[i]] = documents[documents[i]], documents[i]
    }
    return -1
}
```

由于此题的特殊限制，可以使用数组下标作为 map。