---
title: 「Go」 字符串
category_bar: true
date: 2023-04-12 15:11:48
tags:
categories: Golang
banner_img:
---

Golang 字符串相关题目。

<!-- more -->

## 1 概念

字符串是一个结构体，包含一个指向底层 byte 数组的指针和 int 类型的长度属性（字节数）。

Go 使用的 UTF-8 变长编码，每个字符所占的字节是不一样的：

```go
func utf_8Test() {
	str := "英文名为Jamison"
	fmt.Println(len(str)) //19
	// 四个汉字每个汉字三个字节，其余英文每个一字节
	// 所以含有一个字节以上的字符不能使用传统 for 循环遍历，应该使用 for range
	for _, s := range str {
		fmt.Printf("%c", s) // result: 英文名为Jamison
	}

	// 错误做法
	for i := 0; i < len(str); i++ {
		fmt.Printf("%c", str[i]) // result: è±æåä¸ºJamison
	}
}
```

* 对字符串使用 `len` 方法得到的是字节数不是字符数

* 对字符串直接使用下标访问，得到的是字节而不是字符

* 字符串被 `range` 遍历时，被解码为 rune 类型的字符

* UTF-8 编码解码算法位于 `runtime/utf8.go`

## 2 题目

### 2.1 [leetcode 344 题](https://leetcode.cn/problems/reverse-string/)

**原地**修改输入数组，反转字符串。

```go
func reverseString(s []byte) {
    left := 0
    right := len(s)-1
    for left < right {
        s[left], s[right] = s[right], s[left]
        left++
        right--
    }
}
```

### 2.2 [剑指 Offer 05](https://leetcode.cn/problems/ti-huan-kong-ge-lcof/)

把字符串 s 中的每个空格替换成"%20"。

如果想把这道题目做到极致，首先扩充数组到每个空格替换成"%20"之后的大小，然后双指针法**从后向前**替换空格。

从前向后填充就是 O(n^2) 的算法了，因为每次添加元素都要将添加元素之后的所有元素向后移动。

### 2.3 [leetcode 151 题](https://leetcode.cn/problems/reverse-words-in-a-string/)

翻转字符串里的单词，不使用辅助空间，空间复杂度要求为 O(1)：

1. 移除多余空格
2. 将整个字符串反转
3. 将每个单词反转

注：在 Go 中，字符串是不可变的，也就是说无法直接通过下标操作来修改字符串的内容。因此，无法直接通过下标操作来实现字符串的反转。要反转字符串，可以将字符串转换为字节数组（[]byte）。