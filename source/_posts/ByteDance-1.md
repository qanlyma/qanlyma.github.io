---
title: 「训」 笔记（一）：Go 语言
category_bar: true
date: 2023-02-18 15:38:59
tags:
categories: 字节青训
banner_img:
---

关于 Go 语言的基础和进阶教程，示例代码在我的 Github 上：[基础代码](https://github.com/qanlyma/go-by-example)，[进阶代码](https://github.com/qanlyma/go-project-example/tree/V0)

<!-- more -->

## 1 Go 语言基础

### 1.1 Go 语言简介

* 高性能、高并发
* 语法简单、学习曲线平滑
* 丰富的标准库
* 完善的工具链
* 静态链接
* 快速编译
* 跨平台
* 垃圾回收

### 1.2 基础语法

Golang 的安装和基础使用，可以参考[菜鸟教程](https://www.runoob.com/go/go-tutorial.html)。

补充几点：

1. 错误处理

![错误处理](1.png)

2. 字符串操作

![字符串操作](2.png)

3. 字符串格式化

![字符串格式化](3.png)

4. JSON 处理

对于一个已有的结构体，只要保证每个字段的第一个字母是大写（公开字段），就可以用 JSON.Marshal 序列化成一个 JSON 字符串。

![JSON 处理](4.png)

5. 时间处理

![时间处理](5.png)

6. 数字解析

![数字解析](6.png)

7. 进程信息

![进程信息](7.png)

### 1.3 项目实战

1. 猜谜游戏

   - 生成随机数：用时间戳来初始化随机数种子

    ```Go
    func main() {
        maxNum := 100 
        rand.Seed(time.Now().UnixNano()) 
        secretNumber := rand.Intn(maxNum) 
        fmt.Println("The secret number is ", secretNumber)
    }
    ```

   - 读取用户输入

    ```Go
    func main() {
        maxNum := 100
        rand.Seed(time.Now().UnixNano())
        secretNumber := rand.Intn(maxNum)
        fmt.Println("The secret number is ", secretNumber)

        fmt.Println("Please input your guess")
        reader := bufio.NewReader(os.Stdin)

        //读取一行输入
        input, err := reader.ReadString('\n')
        if err != nil {
            fmt.Println("An error occured while reading input. Please try again", err)
            return
        }
        //去掉换行符
        input = strings.Trim(input, "\r\n")
        //转换为数字
        guess, err := strconv.Atoi(input)
        if err != nil {
            fmt.Println("Invalid input. Please enter an integer value")
            return
        }
        fmt.Println("You guess is", guess)
    }
    ```

   - 逻辑判断 + 游戏循环：略

***
2. 命令行词典


***
3. SOCKS5 代理


## 2 Go 语言进阶