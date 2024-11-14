---
title: 「算」 07. 栈与队列
category_bar: true
date: 2023-04-15 11:49:50
tags:
categories: 算法数构
banner_img:
---

栈与队列相关题目。

<!-- more -->

## 1 概念

队列是先进先出（队头删除，队尾插入），栈是先进后出。

![](1.png)

Go 语言中，并没有栈与队列相关的数据结构，但是我们可以借助切片来实现栈与队列的操作。

Linux 系统中，cd 这个进入目录的命令就是栈的应用。

**递归的实现是栈**：每一次递归调用都会把函数的局部变量、参数值和返回地址等压入调用栈中，然后递归返回的时候，从栈顶弹出上一次递归的各项参数，所以这就是递归为什么可以返回上一层位置的原因。

### 1.1 用栈实现队列

```go
type MyQueue struct {
    stackIn  []int // 输入栈
    stackOut []int // 输出栈
}

func Constructor() MyQueue {
    return MyQueue{
        stackIn:  make([]int, 0),
        stackOut: make([]int, 0),
    }
}

// 往输入栈做 push
func (this *MyQueue) Push(x int) {
    this.stackIn = append(this.stackIn, x)
}

// 在输出栈做 pop，pop 时如果输出栈数据为空，需要将输入栈全部数据导入，如果非空，则可直接使用
func (this *MyQueue) Pop() int {
    inLen, outLen := len(this.stackIn), len(this.stackOut)
    if outLen == 0 {
        if inLen == 0 {
            return -1
        }
        for i := inLen - 1; i >= 0; i-- {
            this.stackOut = append(this.stackOut, this.stackIn[i])
        }
        this.stackIn = []int{}      // 导出后清空
        outLen = len(this.stackOut) // 更新长度值
    }
    val := this.stackOut[outLen-1]
    this.stackOut = this.stackOut[:outLen-1]
    return val
}

// 返回队列首部的元素
func (this *MyQueue) Peek() int {
    val := this.Pop()
    if val == -1 {
        return -1
    }
    this.stackOut = append(this.stackOut, val)
    return val
}

func (this *MyQueue) Empty() bool {
    return len(this.stackIn) == 0 && len(this.stackOut) == 0
}
```

### 1.2 用队列实现栈

```go
type MyStack struct {
    queue []int
}


/** Initialize your data structure here. */
func Constructor() MyStack {
    return MyStack {
        queue: make([]int,0),
    }
}


/** Push element x onto stack. */
func (this *MyStack) Push(x int)  {
    this.queue = append(this.queue, x)
}


/** Removes the element on top of the stack and returns that element. */
func (this *MyStack) Pop() int {
    n := len(this.queue) - 1
    for n != 0 { // 除了最后一个，其余的都重新添加到队列里
        val := this.queue[0]
        this.queue = this.queue[1:]
        this.queue = append(this.queue, val)
        n--
    }
    // 弹出元素
    val := this.queue[0]
    this.queue = this.queue[1:]
    return val
}


/** Get the top element. */
func (this *MyStack) Top() int {
    // 利用 Pop 函数，弹出来的元素重新添加
    val := this.Pop()
    this.queue = append(this.queue, val)
    return val
}


/** Returns whether the stack is empty. */
func (this *MyStack) Empty() bool {
    return len(this.queue) == 0
}
```

## 2 题目

### 2.1 [leetcode 20 题](https://leetcode.cn/problems/valid-parentheses/)

有效的括号。

由于栈结构的特殊性，非常适合做对称匹配类的题目。

```go
func isValid(s string) bool {
    hash := map[byte]byte{')':'(', ']':'[', '}':'{'}
    stack := make([]byte, 0)
    if s == "" {
        return true
    }

    for i := 0; i < len(s); i++ {
        if s[i] == '(' || s[i] == '[' || s[i] == '{' {
            stack = append(stack, s[i])
        } else if len(stack) > 0 && stack[len(stack)-1] == hash[s[i]] {
            stack = stack[:len(stack)-1]
        } else {
            return false
        }
    }
    return len(stack) == 0
}
```

### 2.2 [leetcode 150 题](https://leetcode.cn/problems/evaluate-reverse-polish-notation/)

逆波兰表达式求值。

题目本身不难，题解巧妙使用了 err 值得学习。

```go
func evalRPN(tokens []string) int {
	stack := []int{}
	for _, token := range tokens {
		val, err := strconv.Atoi(token)
		if err == nil {
			stack = append(stack, val)
		} else {   // 如果 err 不为 nil 说明不是数字
			num1, num2 := stack[len(stack)-2], stack[(len(stack))-1]
			stack = stack[:len(stack)-2]
			switch token {
			case "+":
				stack = append(stack, num1+num2)
			case "-":
				stack = append(stack, num1-num2)
			case "*":
				stack = append(stack, num1*num2)
			case "/":
				stack = append(stack, num1/num2)
			}
		}
	}
	return stack[0]
}
```

### 2.3 [leetcode 239 题](https://leetcode.cn/problems/sliding-window-maximum/)

滑动窗口最大值。

构建一个递减队列来不断存储更新最大值。

```go
func maxSlidingWindow(nums []int, k int) []int {
    var queue, res []int
    for i := 0; i < k; i++ {
        if len(queue) > 0 {
            if nums[i] > queue[0] {
                queue = queue[0:0]
            } else {
                for nums[i] > queue[len(queue)-1] {
                    queue = queue[:len(queue)-1]
                }
            }
        }
        queue = append(queue, nums[i])
    }
    res = append(res, queue[0])

    for i := k; i < len(nums); i++ {
        if nums[i-k] == queue[0] {
            queue = queue[1:]
        }
        if len(queue) > 0 {
            if nums[i] > queue[0] {
                queue = queue[0:0]
            } else {
                for nums[i] > queue[len(queue)-1] {
                    queue = queue[:len(queue)-1]
                }
            }
        }
        queue = append(queue, nums[i])
        if len(queue) > k {
            queue = queue[:k]
        }
        res = append(res, queue[0])
    }

    return res
}
```

### 2.4 [leetcode 739 题](https://leetcode.cn/problems/daily-temperatures/)

给定一个整数数组 temperatures，表示每天的温度，返回一个数组 answer ，其中 answer[i] 是指对于第 i 天，下一个更高温度出现在几天后。如果气温在这之后都不会升高，请在该位置用 0 来代替。

该题使用暴力解法时间复杂度为 O(n^2)，可以使用**单调栈**，空间换时间，实现 O(n) 的时间复杂度。

本题要使用递增循序（从栈头到栈底的顺序），因为只有递增的时候，栈里要加入一个元素 i 的时候，才知道栈顶元素在数组中右面第一个比栈顶元素大的元素是 i。

**如果求一个元素右边第一个更大元素，单调栈就是递增的，如果求一个元素右边第一个更小元素，单调栈就是递减的**。

```go
func dailyTemperatures(temperatures []int) []int {
    var stack []int
    res := make([]int, len(temperatures))
    for i := 0; i < len(temperatures); i++ {
        if len(stack) == 0 {
            stack = append(stack, i)
        } else if temperatures[i] > temperatures[stack[len(stack)-1]] {
            for len(stack) > 0 && temperatures[i] > temperatures[stack[len(stack)-1]] {
                res[stack[len(stack)-1]] = i - stack[len(stack)-1]
                stack = stack[:len(stack)-1]
            }
        } 
        stack = append(stack, i)
    }
    return res
}
```

### 2.5 [leetcode 42 题](https://leetcode.cn/problems/trapping-rain-water/)

接雨水，单调栈参考[解析](https://programmercarl.com/0042.%E6%8E%A5%E9%9B%A8%E6%B0%B4.html)。

```go
func trap(height []int) int {
    var res int
    stack := []int{0}
    for i := 1; i < len(height); i++ { 
        if height[stack[len(stack)-1]] < height[i] {
            for len(stack) > 0 && height[stack[len(stack)-1]] < height[i] {
                top := stack[len(stack)-1]
                stack = stack[:len(stack)-1]
                if len(stack) > 0 {
                    tmp := (min(height[stack[len(stack)-1]], height[i]) - height[top]) * (i - stack[len(stack)-1] - 1)
                    res += tmp
                }
            }
        } else if height[stack[len(stack)-1]] == height[i] {
            stack[len(stack)-1] = i
            continue
        }
        stack = append(stack, i)
    }
    return res
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```