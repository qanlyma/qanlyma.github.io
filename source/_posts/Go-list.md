---
title: 「Go」 链表
category_bar: true
date: 2023-07-12 09:48:47
tags:
categories: Golang
banner_img:
---

Golang 链表相关题目。

<!-- more -->

## 1 链表

### 1.1 类型

* **单链表**

![](1.png)

* **双链表**

![](2.png)

* **循环链表**

![](3.png)

### 1.2 存储方式

链表中的节点在内存中不是连续分布的 ，而是散乱分布在内存中的某地址上，分配机制取决于操作系统的内存管理。

### 1.3 操作

* **定义**

```go
type ListNode struct {
    Val int
    Next *ListNode
}
```

* **删除**

![](4.png)

建议使用虚拟头节点：

```go
func removeElements(head *ListNode, val int) *ListNode {
    dmHead := &ListNode{}
    dmHead.Next = head
    cur, pre := head, dmHead
    for cur != nil {
        if cur.Val == val {
            pre.Next = cur.Next
        } else {
            pre = pre.Next
        }
        cur = cur.Next
    }
    return dmHead.Next
}
```

* **添加**

![](5.png)

### 1.4 性能分析

![](6.png)

## 2 题目

### 2.1 [leetcode 24 题](https://leetcode.cn/problems/swap-nodes-in-pairs/)

两两交换链表中的节点。

### 2.2 [leetcode 19 题](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)

删除倒数第 N 个节点。

### 2.3 [leetcode 160 题](https://leetcode.cn/problems/intersection-of-two-linked-lists/)

给你两个单链表的头节点 headA 和 headB，请你找出并返回两个单链表相交的起始节点。

### 2.4 [leetcode 142 题](https://leetcode.cn/problems/linked-list-cycle-ii/)

判断链表中是否有环。