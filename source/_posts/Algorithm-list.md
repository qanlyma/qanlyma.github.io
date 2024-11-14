---
title: 「算」 03. 链表
category_bar: true
date: 2023-03-15 09:48:47
tags:
categories: 算法数构
banner_img:
---

链表相关题目。

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
    Val  int
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
    dmHead := &ListNode{Next: head}
    pre, cur := dmHead, head
    for cur != nil && cur.Next != nil {
        next := cur.Next.Next
        pre.Next = cur.Next
        cur.Next.Next = cur
        cur.Next = next
        pre = cur
        cur = next
    }
    return dmHead.Next
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
func getIntersectionNode1(headA, headB *ListNode) *ListNode {
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

func getIntersectionNode2(headA, headB *ListNode) *ListNode {
    curA := headA
    curB := headB
    lenA, lenB := 0, 0
    // 求 A，B 的长度
    for curA != nil {
        curA = curA.Next
        lenA++
    }
    for curB != nil {
        curB = curB.Next
        lenB++
    }
    var step int
    var fast, slow *ListNode
    // 求长度差，并且让更长的链表先走相差的长度
    if lenA > lenB {
        step = lenA - lenB
        fast, slow = headA, headB
    } else {
        step = lenB - lenA
        fast, slow = headB, headA
    }
    for i := 0; i < step; i++ {
        fast = fast.Next
    }
    // 遍历两个链表遇到相同则跳出遍历
    for fast != slow {
        fast = fast.Next
        slow = slow.Next
    }
    return fast
}
```

### 2.4 [leetcode 142 题](https://leetcode.cn/problems/linked-list-cycle-ii/)

判断链表中是否有环，返回链表开始入环的第一个节点。

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

### 2.5 [leetcode 206 题](https://leetcode.cn/problems/reverse-linked-list/)

反转链表。

注意申明 pre 的时候不可以用 `pre := &ListNode{}`，否则其值是 0 而不是 nil。

```go
func reverseList(head *ListNode) *ListNode {
    var pre *ListNode
    cur := head

    for cur != nil {
        next := cur.Next
        cur.Next = pre
        pre = cur
        cur = next
    }
    return pre
}
```

### 2.5 [leetcode 146 题](https://leetcode.cn/problems/lru-cache/)

LRU 缓存。

哈希表 + 双向链表。

```go
type LRUCache struct {
    cp, size int
    rec map[int]*Node
    head, tail *Node
}

type Node struct {
    Key, Val int
    Pre, Next *Node
}

func Constructor(capacity int) LRUCache {
    r := make(map[int]*Node)
    h, t := &Node{}, &Node{}
    h.Next = t
    t.Pre = h
    newLRU := LRUCache {cp: capacity, size: 0, rec: r, head: h, tail: t}
    return newLRU
}

func (this *LRUCache) Get(key int) int {
    node, exist := this.rec[key]
    if exist {
        this.removeNode(node)
        this.addToHead(node)
        return node.Val
    }
    return -1
}

func (this *LRUCache) Put(key int, value int)  {
    node, exist := this.rec[key]
    if exist {
        this.rec[key].Val = value
        this.removeNode(node)
        this.addToHead(node)
    } else {
        node = &Node{Key: key, Val: value}
        this.rec[key] = node
        if this.size == this.cp {
            delete(this.rec, this.tail.Pre.Key)
            this.removeTail()
        } else {
            this.size++
        }
        this.addToHead(node)
    }
}

func (this *LRUCache) addToHead(node *Node) {
    nx := this.head.Next
    this.head.Next = node
    node.Next = nx
    nx.Pre = node
    node.Pre = this.head
}

func (this *LRUCache) removeNode(node *Node) {
    pr, nx := node.Pre, node.Next
    pr.Next = nx
    nx.Pre = pr
}

func (this *LRUCache) removeTail() {
    this.removeNode(this.tail.Pre)
}
```