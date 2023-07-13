---
title: 「Go」 哈希表
category_bar: true
date: 2023-07-12 14:23:10
tags:
categories: Golang
banner_img:
---

Golang 哈希表相关题目。

<!-- more -->

## 1 概念

哈希表（Hash table，国内也有一些算法书籍翻译为散列表），一般哈希表都是用来快速判断一个元素是否出现集合里。

### 1.1 哈希函数

比如把学生的姓名直接映射为哈希表上的索引，然后就可以通过查询索引下标快速知道这位同学是否在这所学校里了。哈希函数通过 hashCode 把名字转化为数值，一般 hashcode 是通过特定编码方式，可以将其他数据格式转化为不同的数值，这样就把学生名字映射为哈希表上的索引数字了。

![](1.png)

### 1.2 哈希碰撞

如图所示，小李和小王都映射到了索引下标 1 的位置，这一现象叫做哈希碰撞。

![](2.png)

* **拉链法**

刚刚小李和小王在索引1的位置发生了冲突，发生冲突的元素都被存储在链表中。 这样我们就可以通过索引找到小李和小王了。

![](3.png)

数据规模是 dataSize， 哈希表的大小为 tableSize。

* **线性探测法**

使用线性探测法，一定要保证 tableSize 大于 dataSize。 我们需要依靠哈希表中的空位来解决碰撞问题。

例如冲突的位置，放了小李，那么就向下找一个空位放置小王的信息。

## 2 题目

### 2.1 [leetcode 242 题](https://leetcode.cn/problems/valid-anagram/)

给定两个字符串 s 和 t ，编写一个函数来判断 t 是否是 s 的字母异位词。

```go
func isAnagram1(s, t string) bool {
    var c1, c2 [26]int
    for _, ch := range s {
        c1[ch-'a']++
    }
    for _, ch := range t {
        c2[ch-'a']++
    }
    return c1 == c2
}

func isAnagram2(s, t string) bool {
    if len(s) != len(t) {
        return false
    }
    cnt := map[rune]int{}
    for _, ch := range s {
        cnt[ch]++
    }
    for _, ch := range t {
        cnt[ch]--
        if cnt[ch] < 0 {
            return false
        }
    }
    return true
}
```

### 2.2 [leetcode 202 题](https://leetcode.cn/problems/happy-number/)

快乐数。

### 2.3 [leetcode 454 题](https://leetcode.cn/problems/4sum-ii/)

四数之和。