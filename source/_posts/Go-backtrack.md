---
title: 「Go」 回溯算法
category_bar: true
date: 2023-07-19 12:41:03
tags:
categories: Golang
banner_img:
---

回溯法也可以叫做回溯搜索法，它是一种搜索的方式。回溯是递归的副产品，只要有递归就会有回溯。

<!-- more -->

## 1 理论

回溯的本质是穷举，穷举所有可能，然后选出我们想要的答案。

### 1.1 解决的问题

* 组合问题：N 个数里面按一定规则找出 k 个数的集合
* 切割问题：一个字符串按一定规则有几种切割方式
* 子集问题：一个 N 个数的集合里有多少符合条件的子集
* 排列问题：N 个数按一定规则全排列，有几种排列方式
* 棋盘问题：N 皇后，解数独等等

### 1.2 回溯法模板

1. 回溯函数模板返回值以及参数
2. 回溯函数终止条件
3. 回溯搜索的遍历过程

![](4.png)

for 循环可以理解是横向遍历，backtracking（递归）就是纵向遍历，这样就把这棵树全遍历完了。

```go
func backtracking(参数) {
    if (终止条件) {
        存放结果
        return
    }

    for (选择：本层集合中元素（树中节点孩子的数量就是集合的大小）) {
        处理节点
        backtracking(路径，选择列表) // 递归
        回溯，撤销处理结果
    }
}
```

## 2 题目

### 2.1 [leetcode 77 题](https://leetcode.cn/problems/combinations/)

给定两个整数 n 和 k，返回范围 [1, n] 中所有可能的 k 个数的组合。你可以按任何顺序返回答案。

最直接的解法是使用 k 个 for 循环，但是当 k 过大时这个方法显然不适用。所以回溯搜索法来了，虽然回溯法也是暴力，但至少能写出来，不像 for 循环嵌套 k 层让人绝望。

用树形结构来理解回溯：

![n = 4, k = 2](1.png)

每次从集合中选取元素，可选择的范围随着选择的进行而收缩，调整可选择的范围。

图中可以发现n相当于树的宽度，k相当于树的深度。每次搜索到了叶子节点，我们就找到了一个结果。

1. **递归函数的返回值以及参数**

在这里要定义两个全局变量，一个用来存放符合条件单一结果，一个用来存放符合条件结果的集合。

```go
var path []int
var res [][]int
```

函数里一定有两个参数，既然是集合 n 里面取 k 个数，那么 n 和 k 是两个 int 型的参数。

然后还需要一个参数，为 int 型变量 start，这个参数用来记录本层递归的中，集合从哪里开始遍历（集合就是[1,...,n]）。start 用于防止出现重复的组合。

```go
var backTrace func(n, k, start int) 
```

2. **回溯函数终止条件**

path 这个数组的大小如果达到 k，说明我们找到了一个子集大小为 k 的组合了，在图中 path 存的就是根节点到叶子节点的路径。

![path](2.png)

此时用 res 二维数组，把 path 保存起来，并终止本层递归。

```go
if len(path) == k { 
    tmp := make([]int, k)
    copy(tmp, path)
    res = append(res, tmp)
    return
}
```

**注意不能直接 `append(res, path)`，后续修改 path 时会影响到 res。**

3. **单层搜索的过程**

回溯法的搜索过程就是一个树型结构的遍历过程，在如下图中，可以看出 for 循环用来横向遍历，递归的过程是纵向遍历。

![遍历](3.png)

for 循环每次从 start 开始遍历，然后用 path 保存取到的节点 i。

```go
for start <= n {                // 控制树的横向遍历
    path = append(path, start)  // 处理节点
    backTrace(n, k, start + 1)  // 递归：控制树的纵向遍历，注意下一层搜索要从 i+1 开始
    path = path[:len(path)-1]   // 回溯：撤销处理的节点
    start++
}
```

代码实现

```go
func combine(n int, k int) [][]int {
    var path []int
    var res [][]int

    var backTrace func(a, b, start int) 
    backTrace = func(a, b, start int) {       
        if len(path) == b { 
            tmp := make([]int, b)
            copy(tmp, path)
            res = append(res, tmp)
            return
        }

        // 从start开始，不往回走，避免出现重复组合
        for start <= a {
            if a - start + 1 < b - len(path) { // 剪枝
                break
            }
            path = append(path, start)
            backTrace(a, b, start + 1)
            path = path[:len(path)-1]
            start++
        }
    }

    backTrace(n, k, 1)
    return res
}
```