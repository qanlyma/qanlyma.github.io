---
title: 「Go」 排序算法
category_bar: true
date: 2023-07-17 14:21:34
tags:
categories: Golang
banner_img:
---

Golang 实现各种排序算法。

<!-- more -->

## 1 冒泡排序

两两比较相邻记录的关键字，如果反序则交换，直到没有反序的记录为止。

```go
func BubbleSort(arr []int) {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        for j := 0; j < n-i-1; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
}
```
时间复杂度：最好情况下为 O(n)，最坏和平均情况下为 O(n^2)。

## 2 快速排序

通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，分别对这两部分记录继续进行排序，以达到整个序列有序。

```go
func QuickSort(arr []int) {
    if len(arr) < 2 {
        return
    }
    pivot := arr[0]
    var left, right []int
    for _, num := range arr[1:] {
        if num <= pivot {
            left = append(left, num)
        } else {
            right = append(right, num)
        }
    }
    QuickSort(left)
    QuickSort(right)
    copy(arr, append(append(left, pivot), right...))
}
```
时间复杂度：最好和平均情况下为 O(nlogn)，最坏情况下为 O(n^2)。

## 3 插入排序

对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。

```go
func InsertionSort(arr []int) {
    n := len(arr)
    for i := 1; i < n; i++ {
        key := arr[i]
        j := i - 1
        for j >= 0 && arr[j] > key {
            arr[j+1] = arr[j]
            j--
        }
        arr[j+1] = key
    }
}
```
时间复杂度：最好情况下为 O(n)，最坏和平均情况下为 O(n^2)。

## 4 希尔排序

将待排序的数组分成多个子序列进行排序，逐步减小子序列的间隔，最终将整个数组变为有序。希尔排序的核心思想是利用插入排序在部分有序数组上的高效性能。

```go
func ShellSort(arr []int) {
    n := len(arr)
    gap := n / 2
    for gap > 0 {
        for i := gap; i < n; i++ {
            temp := arr[i]
            j := i
            for j >= gap && arr[j-gap] > temp {
                arr[j] = arr[j-gap]
                j -= gap
            }
            arr[j] = temp
        }
        gap /= 2
    }
}
```
时间复杂度：最好情况下为 O(nlog^2n)，最坏情况下为 O(n^2)，平均情况下为 O(nlogn)。

## 5 选择排序

对数据操作 n-1 轮，每轮找出一个最小（大）值。以此类推，直到所有元素均排序完毕。

```go
func SelectionSort(arr []int) {
    n := len(arr)
    for i := 0; i < n-1; i++ {
        minIdx := i
        for j := i+1; j < n; j++ {
            if arr[j] < arr[minIdx] {
                minIdx = j
            }
        }
        arr[i], arr[minIdx] = arr[minIdx], arr[i]
    }
}
```
时间复杂度：最好情况下为 O(n^2)，最坏和平均情况下为 O(n^2)。

## 6 堆排序

堆排序使用二叉堆数据结构构建最大堆，并逐步将最大值移动到数组的末尾。

```go
func HeapSort(arr []int) {
    n := len(arr)

    // 构建最大堆
    for i := n/2 - 1; i >= 0; i-- {
        heapify(arr, n, i)
    }

    // 逐个将最大值移到数组末尾
    for i := n - 1; i > 0; i-- {
        arr[0], arr[i] = arr[i], arr[0]
        heapify(arr, i, 0)
    }
}

func heapify(arr []int, n, i int) {
    largest := i
    left := 2*i + 1
    right := 2*i + 2

    if left < n && arr[left] > arr[largest] {
        largest = left
    }

    if right < n && arr[right] > arr[largest] {
        largest = right
    }

    if largest != i {
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
    }
}
```
时间复杂度：最好、最坏和平均情况下均为 O(nlogn)。

## 7 归并排序

归并排序的原理其实就是分治法。它首先将数组不断地二分，直到最后每个部分只包含一个数据。然后再对每个部分分别进行排序，最后将排序好的相邻的两部分合并在一起，这样整个数组就有序了。

![](1.png)

```go
func MergeSort(arr []int) []int {
    if len(arr) < 2 {
        return arr
    }
    mid := len(arr) / 2
    left := MergeSort(arr[:mid])
    right := MergeSort(arr[mid:])
    return merge(left, right)
}

func merge(left, right []int) []int {
    result := make([]int, 0, len(left)+len(right))
    i, j := 0, 0
    for i < len(left) && j < len(right) {
        if left[i] <= right[j] {
            result = append(result, left[i])
            i++
        } else {
            result = append(result, right[j])
            j++
        }
    }
    result = append(result, left[i:]...)
    result = append(result, right[j:]...)
    return result
}
```
时间复杂度：最好、最坏和平均情况下均为 O(nlogn)。
