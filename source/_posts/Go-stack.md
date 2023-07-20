---
title: 「Go」 栈与队列
category_bar: true
date: 2023-04-15 11:49:50
tags:
categories: Golang
banner_img:
---

Golang 栈与队列相关题目。

<!-- more -->

## 1 概念

队列是先进先出，栈是先进后出。

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

### 2.2 [leetcode 239 题](https://leetcode.cn/problems/sliding-window-maximum/)

滑动窗口最大值。
