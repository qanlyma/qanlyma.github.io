---
title: 「Go」 09 二叉树
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

1. **满二叉树**

    如果一棵二叉树只有度为 0 的结点和度为 2 的结点，并且度为 0 的结点在同一层上。

    ![](1.png)

2. **完全二叉树**

    除了最底层节点可能没填满外，其余每层节点数都达到最大值，并且最下面一层的节点都集中在该层最左边。

    ![](2.png)

3. **二叉搜索树**

    二叉搜索树（BST）是一个有序树。

   * 若它的左子树不空，则左子树上所有结点的值均小于它的根结点的值
   * 若它的右子树不空，则右子树上所有结点的值均大于它的根结点的值
   * 它的左、右子树也分别为二搜索树

    ![](3.png)

4. **平衡二叉搜索树**

    AVL（Adelson-Velsky and Landis）树，它是一棵空树或它的左右两个子树的高度差的绝对值不超过 1，并且左右两个子树都是一棵平衡二叉树。

    ![](4.png)

5. **红黑树**

    是一个接近平衡的二叉搜索树，每一个节点增加一个存储位表示其颜色。

    ![](8.png)

    **性质**：

    * 每个结点不是红色就是黑色
    * 根节点是黑色的
    * 如果一个节点是红色的，则它的两个孩子结点是黑色的
    * 对于每个结点，从该结点到其所有后代叶子结点的路径上，均包含相同数目的黑色结点
    * 每个叶子结点都是黑色的（此处的叶子结点指的是空结点）

    其最长路径中节点个数不会超过最短路径节点个数的两倍，因为最短路径为全黑，最长路径就是红黑节点交替（因为红色节点不能连续），每条路径的黑色节点相同，则最长路径刚好是最短路径的两倍。

6. **哈夫曼树**

    给定 N 个权值作为 N 个叶子结点，构造一棵二叉树，若该树的带权路径长度达到最小，称这样的二叉树为最优二叉树，也称为哈夫曼树（Huffman Tree），哈夫曼树是带权路径长度最短的树，权值较大的结点离根较近。

    所有**叶结点**的带权路径长度之和：WPL = ∑(叶子节点权值 * 节点深度)

    **构造哈夫曼树**：

    1. 在 n 个权值中选出两个最小的权值，对应的两个结点组成一个新的二叉树，且新二叉树的根结点的权值为左右孩子权值的和；
    2. 在原有的 n 个权值中删除那两个最小的权值，同时将新的权值加入到 n-2 个权值的行列中，以此类推；
    3. 重复 1 和 2 ，直到所以的结点构建成了一棵二叉树为止，这棵树就是哈夫曼树。

    ![](9.png)

    哈夫曼编码算法是基于二叉树构建编码压缩结构的，它是数据压缩中经典的一种算法。算法根据文本字符出现的频率，重新对字符进行编码。为了缩短编码的长度，自然希望频率越高的词编码越短，这样最终才能最大化压缩存储文本数据的空间。

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

根节点的高度就是二叉树的最大深度，所以本题中我们通过后序求的根节点高度来求的二叉树最大深度。

```go
func maxdepth(root *Treenode) int {
    if root == nil {
        return 0
    }
    return max(maxdepth(root.Left), maxdepth(root.Right)) + 1
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

### 3.4 [leetcode 112 题](https://leetcode.cn/problems/path-sum/)

给你二叉树的根节点 root 和一个表示目标和的整数 targetSum。判断该树中是否存在根节点到叶子节点的路径，这条路径上所有节点值相加等于目标和 targetSum。如果存在，返回 true；否则，返回 false。

```go
func hasPathSum1(root *TreeNode, targetSum int) bool {
    if root == nil {
        return false
    }
    var res bool
    var dfs func(node *TreeNode, sum int)
    dfs = func(node *TreeNode, sum int) {
        if node == nil {
            return
        }
        sum += node.Val
        if node.Left == nil && node.Right == nil && sum == targetSum {
            res = true
        }
        dfs(node.Left, sum)
        dfs(node.Right, sum)
    }
    dfs(root, 0)
    return res
}

func hasPathSum2(root *TreeNode, targetSum int) bool {
    if root == nil {
        return false
    }
    targetSum -= root.Val
    if root.Left == nil && root.Right == nil && targetSum == 0 {
        return true
    }
    return hasPathSum(root.Left, targetSum) || hasPathSum(root.Right, targetSum)
}
```

### 3.5 [leetcode 106 题](https://leetcode.cn/problems/construct-binary-tree-from-inorder-and-postorder-traversal/)

从中序与后序遍历序列构造二叉树。

![](7.png)

1. 如果数组大小为零的话，说明是空节点了
2. 如果不为空，那么取后序数组最后一个元素作为节点元素
3. 找到后序数组最后一个元素在中序数组的位置，作为切割点
4. 切割中序数组，切成中序左数组和中序右数组
5. 切割后序数组，切成后序左数组和后序右数组
6. 递归处理左区间和右区间

```go
func buildTree(inorder []int, postorder []int) *TreeNode {
    root := &TreeNode{}
    if len(inorder) == 0 {
        return nil
    }
    v := postorder[len(postorder)-1]
    var cut int
    for i := 0; i < len(inorder); i++ {
        if inorder[i] == v {
            cut = i
            break
        }
    }
    root.Val = v
    root.Left = buildTree(inorder[:cut], postorder[:cut])
    root.Right = buildTree(inorder[cut+1:], postorder[cut:len(postorder)-1])
    return root
}
```

### 3.6 [leetcode 503 题](https://leetcode.cn/problems/minimum-absolute-difference-in-bst/submissions/)

给你一棵所有节点为非负值的二叉搜索树，请你计算树中任意两节点的差的绝对值的最小值。

**二叉搜索树采用中序遍历，其实就是一个有序数组。**最直观的想法，就是把二叉搜索树转换成有序数组，然后遍历一遍数组，就统计出来最小差值了。

其实在二叉搜素树中序遍历的过程中，我们就可以直接计算了。需要用一个 pre 节点记录一下 cur 节点的前一个节点。

```go
func getMinimumDifference(root *TreeNode) int {
    var prev *TreeNode
    min := math.MaxInt64
    var travel func(node *TreeNode)
    travel = func(node *TreeNode) {
        if node == nil {
            return 
        }
        travel(node.Left)
        if prev != nil && node.Val - prev.Val < min {
            min = node.Val - prev.Val
        }
        prev = node
        travel(node.Right)
    }
    travel(root)
    return min
}
```

### 3.7 [leetcode 236 题](https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-tree/)

给定一个二叉树，找到该树中两个指定节点的最近公共祖先。

求最小公共祖先，需要从底向上遍历，那么二叉树只能通过后序遍历（即：回溯）实现从底向上的遍历方式。

```go
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }

    left := lowestCommonAncestor(root.Left, p, q)
    right := lowestCommonAncestor(root.Right, p, q)
    if right != nil && left != nil {
        return root
    } else if right == nil {
        return left
    }
    return right
}
```

在回溯的过程中，必然要遍历整棵二叉树，即使已经找到结果了，依然要把其他节点遍历完，因为要使用递归函数的返回值（也就是代码中的 left 和 right）做逻辑判断。

### 3.8 [leetcode 701 题](https://leetcode.cn/problems/insert-into-a-binary-search-tree/)

给定二叉搜索树（BST）的根节点和要插入树中的值，将值插入二叉搜索树。

```go
func insertIntoBST(root *TreeNode, val int) *TreeNode {
    if root == nil {
        root = &TreeNode{Val: val}
        return root
    }
    if root.Val > val {
        root.Left = insertIntoBST(root.Left, val)
    } else {
        root.Right = insertIntoBST(root.Right, val)
    }
    return root
}
```

### 3.9 [leetcode 450 题](https://leetcode.cn/problems/delete-node-in-a-bst/)

给定一个二叉搜索树的根节点 root 和一个值 key，删除二叉搜索树中的 key 对应的节点，并保证二叉搜索树的性质不变。

单层递归的逻辑有以下五种情况：

1. 没找到删除的节点，遍历到空节点直接返回了
2. 删除节点的左右孩子都为空（叶子节点），直接删除节点， 返回NULL为根节点
3. 删除节点的左孩子为空，右孩子不为空，删除节点，右孩子补位，返回右孩子为根节点
4. 删除节点的右孩子为空，左孩子不为空，删除节点，左孩子补位，返回左孩子为根节点
5. 删除节点的左右孩子节点都不为空，则将删除节点的左子树头结点（左孩子）放到删除节点的右子树的最左面节点的左孩子上，返回删除节点右孩子为新的根节点。

![](1.gif)

```go
func deleteNode(root *TreeNode, key int) *TreeNode {
    if root == nil {
        return root
    }
    if key < root.Val {
        root.Left = deleteNode(root.Left, key)
    } else if key > root.Val {
        root.Right = deleteNode(root.Right, key)
    } else {
        if root.Left == nil && root.Right == nil {
            return nil
        } else if root.Left != nil && root.Right == nil {
            return root.Left
        } else if root.Right != nil && root.Left == nil {
            return root.Right
        } else {
            temp := root.Left 
            cur := root.Right
            for cur.Left != nil {
                cur = cur.Left
            }
            cur.Left = temp
            return root.Right
        }
    }
    return root
}
```

### 3.10 [剑指 Offer 26](https://leetcode.cn/problems/shu-de-zi-jie-gou-lcof/)

输入两棵二叉树 A 和 B，判断 B 是不是 A 的子结构。

```go
func isSubStructure(A *TreeNode, B *TreeNode) bool {
    if B == nil || A == nil {
        return false
    }
    return isSub(A, B) || isSubStructure(A.Left, B) || isSubStructure(A.Right, B)
}

func isSub(A, B *TreeNode) bool {
    if B == nil {
        return true
    }
    if A == nil || A.Val != B.Val {
        return false
    } 
    return isSub(A.Left, B.Left) && isSub(A.Right, B.Right)
}
```