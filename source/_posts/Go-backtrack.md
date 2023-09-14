---
title: 「Go」 10 回溯算法
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
var res  [][]int
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

        // 从 start 开始，不往回走，避免出现重复组合
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

### 2.2 [leetcode 40 题](https://leetcode.cn/problems/combination-sum-ii/)

给定一个数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。

candidates 中的每个数字在每个组合中只能使用一次。解集不能包含重复的组合。

这道题的难点在于去重。要去重的是同一树层上的“使用过”，同一树枝上的都是一个组合里的元素，不用去重。

![](5.png)

* `used[i - 1] == true`，说明同一树枝 candidates[i - 1] 使用过，是进入下一层递归
* `used[i - 1] == false`，说明同一树层 candidates[i - 1] 使用过，是回溯而来的

```go
func combinationSum2(candidates []int, target int) [][]int {
    sort.Ints(candidates)
    var res [][]int
    var path []int
    used := make([]bool, len(candidates))
    var sum int
    var backtracking func(start int)
    backtracking = func(start int) {
        if sum > target {
            return
        } else if sum == target {
            tmp := make([]int, len(path))
            copy(tmp, path)
            res = append(res, tmp)
        }

        for i := start; i < len(candidates); i++ {
            if i > 0 && candidates[i] == candidates[i-1] && !used[i-1] {
                continue
            }
            sum += candidates[i]
            path = append(path, candidates[i])
            used[i] = true
            backtracking(i+1)
            sum -= candidates[i]
            path = path[:len(path)-1]
            used[i] = false
        }
    }
    backtracking(0)
    return res
}
```

### 2.3 [leetcode 131 题](https://leetcode.cn/problems/palindrome-partitioning/)

给定一个字符串 s，将 s 分割成一些子串，使每个子串都是回文串。

切割问题本质上和组合问题是一样的。

```go
func partition(s string) [][]string {
    var res [][]string
    var path []string
    var backtracking func(sub string) 
    backtracking = func(sub string) {
        if len(sub) == 0 {
            tmp := make([]string, len(path))
            copy(tmp, path)
            res = append(res, tmp)
        }

        for i := 1; i <= len(sub); i++ {
            if ifStr(sub[0:i]) {
                path = append(path, sub[0:i])
                backtracking(sub[i:])
                path = path[:len(path)-1]
            }
        }
    }
    backtracking(s)
    return res
}

func ifStr(s string) bool {
    b := []byte(s)
    for i, j := 0, len(b)-1; i < j; i, j = i+1, j-1 {
        if b[i] != b[j] {
            return false
        }
    }
    return true
}
```

### 2.4 [leetcode 90 题](https://leetcode.cn/problems/subsets-ii/)

给你一个整数数组 nums，其中可能包含重复元素，请你返回该数组所有可能的子集（幂集）。

```go
func subsetsWithDup(nums []int) [][]int {
    sort.Ints(nums)
    var res [][]int
    var path []int
    used := make([]bool, len(nums))
    var backtracking func(start int) 
    backtracking = func(start int) {
        tmp := make([]int, len(path))
        copy(tmp, path)
        res = append(res, tmp)

        for i := start; i < len(nums); i++ {
            if i > 0 && nums[i] == nums[i-1] && !used[i-1] {
                continue
            }
            path = append(path, nums[i])
            used[i] = true
            backtracking(i+1)
            path = path[:len(path)-1]
            used[i] = false
        }
    }
    backtracking(0)
    return res
}
```

### 2.5 [leetcode 491 题](https://leetcode.cn/problems/non-decreasing-subsequences/submissions/)

给你一个整数数组 nums，找出并返回所有该数组中不同的递增子序列，数组中可能含有重复元素。

不能对数组进行排序，在每一层新建一个 map 进行去重。

```go
func findSubsequences(nums []int) [][]int {
    var res [][]int
    var path []int
    var backtracking func(start int) 
    backtracking = func(start int) {
        if len(path) >= 2 {
            tmp := make([]int, len(path))
            copy(tmp, path)
            res = append(res, tmp)
        }

        usedmap := make(map[int]bool)
        for i := start; i < len(nums); i++ {
            if (len(path) != 0 && nums[i] < path[len(path)-1]) || usedmap[nums[i]] {
                continue
            }
            path = append(path, nums[i])
            usedmap[nums[i]] = true
            backtracking(i+1)
            path = path[:len(path)-1]
        }
    }
    backtracking(0)
    return res
}
```

### 2.6 [leetcode 47 题](https://leetcode.cn/problems/permutations-ii/submissions/)

给定一个可包含重复数字的序列 nums，按任意顺序返回所有不重复的全排列。

```go
func permuteUnique(nums []int) [][]int {
    sort.Ints(nums)
    var res [][]int 
    var path []int
    var backtracking func() 
    used := make([]bool, len(nums))
    backtracking = func() {
        if len(path) == len(nums) {
            tmp := make([]int, len(path))
            copy(tmp, path)
            res = append(res, tmp)
            return
        }
        for i := 0; i < len(nums); i++ {
            // used[i] 表示在上一层用过，!used[i-1] 表示在同一层用过
            if used[i] || (i > 0 && nums[i] == nums[i-1] && !used[i-1]) {
                continue
            }
            path = append(path, nums[i])
            used[i] = true
            backtracking()
            path = path[:len(path)-1]
            used[i] = false
        }
    }
    backtracking()
    return res
}
```

### 2.7 [leetcode 698 题](https://leetcode.cn/problems/partition-to-k-equal-sum-subsets/)

给定一个整数数组 nums 和一个正整数 k，找出是否有可能把这个数组分成 k 个非空子集，其总和都相等。

```go
func canPartitionKSubsets(nums []int, k int) bool {
    var sum int
    for _, v := range nums {
        sum += v
    }
    if sum % k != 0 {
        return false
    }
    target := sum / k
    sort.Ints(nums)
    used := make([]bool, len(nums))
    var dfs func(start, sum, count int, used []bool) bool
    dfs = func(start, sum, count int, used []bool) bool {
        if count == k {
            return true
        } else if sum == target {
            return dfs(len(nums)-1, 0, count+1, used)
        }

        for i := start; i >= 0; i-- {
            if used[i] || sum + nums[i] > target {
                continue
            } 
            used[i] = true
            if dfs(i-1, sum+nums[i], count, used) {
                return true
            }
            used[i] = false
            if sum == 0 {
                return false
            }
        }
        return false
    }
    return dfs(len(nums)-1, 0, 0, used)
}
```

### 2.8 [leetcode 51 题](https://leetcode.cn/problems/n-queens/)

N 皇后。

### 2.9 [leetcode 37 题](https://leetcode.cn/problems/sudoku-solver/)

解数独。

## 3 递归

在这里整理几道不是二叉树的递归题目。

### 3.1 [剑指 Offer 25](https://leetcode.cn/problems/he-bing-liang-ge-pai-xu-de-lian-biao-lcof/)

合并两个排序的链表。

```go
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    if l1 == nil {
        return l2
    } else if l2 == nil {
        return l1
    }

    var l *ListNode
    if l1.Val <= l2.Val {
        l = l1
        l1 = l1.Next
    } else {
        l = l2
        l2 = l2.Next
    }
    l.Next = mergeTwoLists(l1, l2)
    return l
}
```

### 3.2 字节面试题

计算字符串表达式，字符串包含数字、括号、加法、乘法，例如："2+3*(4+5)"。

递归 + 栈：

```go
func calculateExpression(expr string) int {
	stack := []int{}
	op := '.'

	for i := 0; i < len(expr); i++ {
		if expr[i] != '+' && expr[i] != '*' {
			num := 0
			if expr[i] >= '0' && expr[i] <= '9' {
				for i < len(expr) && expr[i] >= '0' && expr[i] <= '9' {
					num = num*10 + int(expr[i]-'0')
					i++
				}
				i--
			} else if expr[i] == '(' {
				cnt := 1
				j := i + 1
				for cnt > 0 {
					if expr[j] == '(' {
						cnt++
					} else if expr[j] == ')' {
						cnt--
					}
					j++
				}
				num = calculateExpression(expr[i+1 : j-1])
				i = j - 1
			}

			if op == '*' {
				num *= stack[len(stack)-1]
				stack[len(stack)-1] = num
			} else {
				stack = append(stack, num)
			}
		}

		if expr[i] == '*' || expr[i] == '+' {
			op = rune(expr[i])
		}
	}

	result := 0
	for _, val := range stack {
		result += val
	}
	return result
}
```

## 4 图论

直接看题吧，参考[卡哥教程](https://programmercarl.com/%E5%9B%BE%E8%AE%BA%E6%B7%B1%E6%90%9C%E7%90%86%E8%AE%BA%E5%9F%BA%E7%A1%80.html)

### 4.1 [leetcode 797 题](https://leetcode.cn/problems/all-paths-from-source-to-target/)

所有可能的路径。

使用深度优先搜索遍历图即可，由于题目限制不用考虑环的问题。

```go
func allPathsSourceTarget(graph [][]int) [][]int {
    var res [][]int
    var path []int
    dst := len(graph) - 1
    var dfs func(node int)
    dfs = func(node int) {
        if node == dst {
            path = append(path, dst)
            tmp := make([]int, len(path))
            copy(tmp, path)
            res = append(res, tmp)
            return
        }
        path = append(path, node)
        for i := 0; i < len(graph[node]); i++ {
            nxt := graph[node][i]
            dfs(nxt)
            path = path[:len(path)-1]
        }
    }
    dfs(0)
    return res
}
```

### 4.2 [leetcode 200 题](https://leetcode.cn/problems/number-of-islands/)

岛屿数量。

```go
// 遍历 grid 遇到 '1' 就开始递归染色，相连的 '1' 都染成 '0'
// 属于深度优先 dfs
func numIslands(grid [][]byte) int {
    m, n := len(grid), len(grid[0])
    var res int
    var mark func(i, j int) 
    mark = func(i, j int) {
        if i < 0 || i >= m || j < 0 || j >= n || grid[i][j] == '0' {
            return
        }
        grid[i][j] = '0'
        mark(i-1, j)
        mark(i+1, j)
        mark(i, j-1)
        mark(i, j+1)
    }

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' {
                res++
                mark(i, j)
            }
        }
    }
    return res
}
```

### 4.3 [leetcode 207 题](https://leetcode.cn/problems/course-schedule/)

课程表。

一道拓扑排序的题，由两种解法：

* BFS，维护一个入度表以及边的信息，每次将入度为 0 的边进行排序，并根据边的信息消除它们对其他顶点入度的影响。
* DFS，维护一个 visited 数组用于标记节点状态：0：未访问；1：搜索中；2：已搜索，若 dfs 中再次搜索到 1 的节点，则判断成环。

```go
// BFS
func canFinish(numCourses int, prerequisites [][]int) bool {
    tomap := make(map[int][]int)
    indeg := make([]int, numCourses)
    for _, v := range prerequisites {
        tomap[v[1]] = append(tomap[v[1]], v[0])
        indeg[v[0]]++
    }
    
    for numCourses > 0 {
        var queue []int
        for i := 0; i < len(indeg); i++ {
            if indeg[i] == 0 {
                // 将入度为 0 的顶点加入队列排序
                queue = append(queue, i)
                indeg[i]--
            }
        }
        l := len(queue)
        if l == 0 {
            return false
        }
        for i := 0; i < l; i++ {
            for j := 0; j < len(tomap[queue[i]]); j++ {
                // 消除入度为 0 的顶点的影响
                indeg[tomap[queue[i]][j]]--
            }
        }
        numCourses -= l
    }
    return true
}

// DFS
func canFinish(numCourses int, prerequisites [][]int) bool {
    res := true
    visited := make([]int, numCourses)
    var dfs func(n int) 
    dfs = func(n int) {
        if visited[n] == 2 || !res {
            return 
        } else if visited[n] == 1 {
            res = false
        }
        visited[n] = 1
        for _, v := range prerequisites {
            if v[1] == n {
                dfs(v[0])
            }
        }
        visited[n] = 2
    }

    for i := 0; i < numCourses; i++ {
        if visited[i] == 0 && res {
            dfs(i)
        }
    }
    return res
}
```

## 5 并查集

并查集常用来解决连通性问题。当我们需要判断两个元素是否在同一个集合里的时候，就要想到用并查集。

并查集主要有两个功能：

* 将两个元素添加到一个集合中
* 判断两个元素在不在同一个集合

### 5.1 [leetcode 200 题](https://leetcode.cn/problems/find-if-path-exists-in-graph/)

寻找图中是否存在路径。

```go
func validPath(n int, edges [][]int, source int, destination int) bool {
    father := make([]int, n)

    // 寻根
    var find func(n int) int 
    find = func(n int) int {
        if father[n] == n { return n }
        father[n] = find(father[n]) // 路径压缩
        return father[n]
    }

    // 判断 u 和 v 是否找到同一个根
    var isSame func(u, v int) bool 
    isSame = func(u, v int) bool {
        return find(u) == find(v)
    }

    // 加入并查集
    var join func(u, v int) 
    join = func(u, v int) {
        u = find(u)
        v = find(v)
        // 一定要先通过 find 函数寻根再进行关联，不可以写 if find(u) == find(v) { return }
        if u == v { return }
        father[v] = u
    }

    // 并查集初始化
    for i, _ := range father {
        father[i] = i
    }

    for _, v := range edges {
        join(v[0], v[1])
    }
    return isSame(source, destination)
}
```