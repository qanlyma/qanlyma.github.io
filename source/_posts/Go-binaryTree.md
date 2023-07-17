---
title: 「Go」 二叉树
category_bar: true
date: 2023-07-15 16:28:45
tags:
categories: Golang
banner_img:
---

Golang 二叉树相关题目。

<!-- more -->

## 1 理论

### 1.1 二叉树的种类

1. 满二叉树

如果一棵二叉树只有度为 0 的结点和度为 2 的结点，并且度为 0 的结点在同一层上，则这棵二叉树为满二叉树。

![](1.png)

2. 完全二叉树

除了最底层节点可能没填满外，其余每层节点数都达到最大值，并且最下面一层的节点都集中在该层最左边的若干位置。

![](2.png)

3. 二叉搜索树

二叉搜索树是一个有序树。

* 若它的左子树不空，则左子树上所有结点的值均小于它的根结点的值
* 若它的右子树不空，则右子树上所有结点的值均大于它的根结点的值
* 它的左、右子树也分别为二搜索树

![](3.png)

4. 平衡二叉搜索树

AVL（Adelson-Velsky and Landis）树，它是一棵空树或它的左右两个子树的高度差的绝对值不超过 1，并且左右两个子树都是一棵平衡二叉树。

![](4.png)

### 1.2 二叉树的存储方式

二叉树可以链式存储，也可以顺序存储。链式存储方式就用指针， 顺序存储的方式就是用数组。

* 链式存储

![](5.png)

* 顺序存储

![](6.png)

如果父节点的数组下标是 `i`，那么它的左孩子就是 `i * 2 + 1`，右孩子就是 `i * 2 + 2`。

### 1.3 二叉树的定义

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}
```

## 2 遍历

* 深度优先遍历：先往深走，遇到叶子节点再往回走。
  * 前序遍历
  * 中序遍历
  * 后序遍历

* 广度优先遍历：一层一层的去遍历。
  * 层次遍历

做二叉树相关题目，经常会使用递归的方式来实现深度优先遍历，也就是实现前中后序遍历，使用递归是比较方便的。栈其实就是递归的一种实现结构，也就说前中后序遍历的逻辑其实都是可以借助栈使用非递归的方式来实现的。

而广度优先遍历的实现一般使用队列来实现，这也是队列先进先出的特点所决定的，因为需要先进先出的结构，才能一层一层的来遍历二叉树。

### 2.1 **前序遍历**：[leetcode 144 题](https://leetcode.cn/problems/binary-tree-preorder-traversal/)

1. 确定递归函数的参数和返回值
2. 确定终止条件
3. 确定单层递归的逻辑

```go
func preorderTraversal(root *TreeNode) []int {
    var res []int
    var dfs func(node *TreeNode) 
    dfs = func(node *TreeNode) {
        if node == nil {
            return
        }
        res = append(res, node.Val)
        dfs(node.Left)
        dfs(node.Right)
    }

    dfs(root)
    return res
}
```

迭代

```go
func preorderTraversal(root *TreeNode) []int {
    var res []int
    var stack []*TreeNode
    if root == nil {
        return res
    }
    stack = append(stack, root)
    for len(stack) > 0 {
        node := stack[len(stack)-1]
        res = append(res, node.Val)
        stack = stack[:len(stack)-1]
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
    }
    return res
}
```

### 2.2 **层序遍历**：[leetcode 102 题](https://leetcode.cn/problems/binary-tree-level-order-traversal/)

```go
// 递归
func levelOrder(root *TreeNode) [][]int {
	arr := [][]int{}
	depth := 0
	var order func(root *TreeNode, depth int)
	order = func(root *TreeNode, depth int) {
		if root == nil {
			return
		}
		if len(arr) == depth {
			arr = append(arr, []int{})
		}
		arr[depth] = append(arr[depth], root.Val)

		order(root.Left, depth+1)
		order(root.Right, depth+1)
	}

	order(root, depth)
	return arr
}

// 切片模拟队列
func levelOrder(root *TreeNode) [][]int {
    var res [][]int
    if root == nil {
        return res
    }
    var queue []*TreeNode
    queue = append(queue, root)
    for len(queue) > 0 {
        l := len(queue)
        var temp []int
        for i := 0; i < l; i++ {
            node := queue[i]
            temp = append(temp, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        queue = queue[l:]
        res = append(res, temp)
    }
    return res
}
```

## 3 题目

### 3.1 [leetcode 101 题](https://leetcode.cn/problems/symmetric-tree/)

给你一个二叉树的根节点 root ， 检查它是否轴对称。

```go
func isSymmetric(root *TreeNode) bool {
    var dfs func(node1, node2 *TreeNode) bool
    dfs = func(node1, node2 *TreeNode) bool {
        if node1 == nil && node2 == nil {
            return true
        } else if node1 == nil && node2 != nil {
            return false
        } else if node1 != nil && node2 == nil {
            return false
        } else if node1.Val != node2.Val {
            return false
        }

        return dfs(node1.Left, node2.Right) && dfs(node1.Right, node2.Left)
    }

    return dfs(root.Left, root.Right)
}
```

### 3.2 [leetcode 104 题](https://leetcode.cn/problems/maximum-depth-of-binary-tree/submissions/)

给定一个二叉树，找出其最大深度。

而根节点的高度就是二叉树的最大深度，所以本题中我们通过后序求的根节点高度来求的二叉树最大深度。

```go
func maxdepth(root *treenode) int {
    if root == nil {
        return 0;
    }
    return max(maxdepth(root.left), maxdepth(root.right)) + 1;
}
```

### 3.3 [leetcode 404 题](https://leetcode.cn/problems/sum-of-left-leaves/)

给定二叉树的根节点 root ，返回所有左叶子之和。

```go
func sumOfLeftLeaves(root *TreeNode) int {
    var sum int
    var dfs func(node *TreeNode, isLeft bool) 
    dfs = func(node *TreeNode, isLeft bool) {
        if node == nil {
            return
        }
        if node.Left == nil && node.Right == nil && isLeft {
            sum += node.Val
        }
        dfs(node.Left, true)
        dfs(node.Right, false)
    }

    dfs(root, false)
    return sum
}
```