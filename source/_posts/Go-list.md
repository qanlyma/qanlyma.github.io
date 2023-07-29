---
title: 「Go」 链表
category_bar: true
date: 2023-03-15 09:48:47
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

```go
func swapPairs(head *ListNode) *ListNode {
    dummy := &ListNode{
        Next: head,
    }
    pre := dummy 
    for head != nil && head.Next != nil {
        pre.Next = head.Next
        next := head.Next.Next
        head.Next.Next = head
        head.Next = next
        pre = head  // pre=list[(i+2)-1]
        head = next // head=list[(i+2)]
    }
    return dummy.Next
}
```

### 2.2 [leetcode 19 题](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)

删除倒数第 N 个节点。

```go
func removeNthFromEnd(head *ListNode, n int) *ListNode {
    var count int
    dmHead := &ListNode{Next: head}
    pre, cur := dmHead, head
    for cur != nil {
        if count != n {
            count++
        } else {
            pre = pre.Next
        }
        cur = cur.Next
    }
    pre.Next = pre.Next.Next
    return dmHead.Next
}
```

### 2.3 [leetcode 160 题](https://leetcode.cn/problems/intersection-of-two-linked-lists/)

给你两个单链表的头节点 headA 和 headB，请你找出并返回两个单链表相交的起始节点。

```go
func getIntersectionNode(headA, headB *ListNode) *ListNode {
    vis := map[*ListNode]bool{}
    for tmp := headA; tmp != nil; tmp = tmp.Next {
        vis[tmp] = true
    }
    for tmp := headB; tmp != nil; tmp = tmp.Next {
        if vis[tmp] {
            return tmp
        }
    }
    return nil
}
```

### 2.4 [leetcode 142 题](https://leetcode.cn/problems/linked-list-cycle-ii/)

判断链表中是否有环。

```go
func detectCycle(head *ListNode) *ListNode {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            for slow != head {
                slow = slow.Next
                head = head.Next
            }
            return head
        }
    }
    return nil
}
```