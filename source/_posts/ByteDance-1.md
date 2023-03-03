---
title: 「训」 笔记（一）：Go 语言
category_bar: true
date: 2023-02-18 15:38:59
tags:
categories: 字节青训
banner_img:
---

关于 Go 语言的基础和进阶教程，示例代码在我的 Github 上：
[基础代码](https://github.com/qanlyma/go-by-example)，[进阶代码](https://github.com/qanlyma/go-project-example/tree/V0)

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

### 2.1 Goroutine

* **线程**：程序执行的最小单元，是由寄存器集合和堆栈组成，线程是进程中的一个实体，可共享同一进程中所拥有的全部资源。
* **协程**：Goroutine 是一种比线程更加轻量级的存在。一个进程可以有多个线程，一个线程可以有多个协程。

```go
func hello(i int) {
	println("hello goroutine : " + fmt.Sprint(i))
}

func HelloGoRoutine() {
	for i := 0; i < 5; i++ {
		go func(j int) {
			hello(j)
		}(i)
	}
	time.Sleep(time.Second)
}
```

### 2.2 CSP（Communicating Sequential Processes）

提倡**通过通信共享内存**而不是通过共享内存而实现通信。

![CSP](8.png)

### 2.3 Channel

* 无缓冲通道：`make(chan int)`
* 有缓冲通道：`make(chan int, 2)`

```go
func CalSquare() {
	src := make(chan int)
	dest := make(chan int, 3)
	go func() {
		defer close(src)
		for i := 0; i < 10; i++ {
			src <- i
		}
	}()
	go func() {
		defer close(dest)
		for i := range src {
			dest <- i * i
		}
	}()
	for i := range dest {
		//复杂操作
		println(i)
	}
}
```

### 2.4 Lock 并发安全

```go
var (
	x    int64
	lock sync.Mutex
)

func addWithLock() {
	for i := 0; i < 2000; i++ {
		lock.Lock()
		x += 1
		lock.Unlock()
	}
}
func addWithoutLock() {
	for i := 0; i < 2000; i++ {
		x += 1
	}
}

func Add() {
	x = 0
	for i := 0; i < 5; i++ {
		go addWithoutLock()
	}
	time.Sleep(time.Second)
	println("WithoutLock:", x)
	x = 0
	for i := 0; i < 5; i++ {
		go addWithLock()
	}
	time.Sleep(time.Second)
	println("WithLock:", x)
}
```

### 2.5 WaitGroup

计数器：开启协程 +1；执行结束 -1；主协程阻塞直到计数器为 0。

```go
func ManyGoWait() {
	var wg sync.WaitGroup
	wg.Add(5)                   // 计数器 +5
	for i := 0; i < 5; i++ {
		go func(j int) {
			defer wg.Done()     // 计数器 -1
			hello(j)
		}(i)
	}
	wg.Wait()                   // 阻塞直到计数器为 0
}
```

## 3 依赖管理

* **GOPATH**：设置环境变量，项目代码依赖 src 下的代码。无法实现 package 的多版本控制。
* **Go Vendor**：项目目录下增加 vendor 文件存放依赖包副本。同一项目无法依赖一个 package 的两个不同版本。
* **Go Module**：通过 go.mod 文件管理依赖包版本

依赖管理三要素：

1. 配置文件，描述依赖：go.mod
2. 中心仓库管理依赖库：Proxy
3. 本地工具：go get/mod

![Proxy](9.png)

## 4 测试

### 4.1 单元测试

* 测试文件以 `_test.go` 结尾
* 函数 `func TestXxx(t *testing.T)`
* 初始化逻辑放入 `TestMain(m *testing.M)`

print.go：
```go
func HelloTom() string {
	return "Tom"
}
```

print_test.go：
```go
import (
	"github.com/stretchr/testify/assert"
	"testing"
)

func TestHelloTom(t *testing.T) {
	output := HelloTom()
	expectOutput := "Tom"
	assert.Equal(t, expectOutput, output)
}
```

覆盖率：`go test Xxx_test.go Xxx.go --cover`

### 4.2 Mock

为一个函数/方法打桩，不再依赖本地文件。

```go
func ReadFirstLine() string {
	open, err := os.Open("log")
	defer open.Close()
	if err != nil {
		return ""
	}
	scanner := bufio.NewScanner(open)
	for scanner.Scan() {
		return scanner.Text()
	}
	return ""
}

func ProcessFirstLine() string {
	line := ReadFirstLine()
	destLine := strings.ReplaceAll(line, "11", "00")
	return destLine
}
```
以上函数读取 log 文件的第一行并将 “11” 替换为 “00”，依赖于本地的文件，而使用 Mock 可以避免依赖。
```go
func TestProcessFirstLine(t *testing.T) {
	firstLine := ProcessFirstLine()
	assert.Equal(t, "line00", firstLine)
}

func TestProcessFirstLineWithMock(t *testing.T) {
	monkey.Patch(ReadFirstLine, func() string {
		return "line110"
	})
	defer monkey.Unpatch(ReadFirstLine)
	line := ProcessFirstLine()
	assert.Equal(t, "line000", line)
}
```

### 4.3 Benchmark

基准测试 `go test -bench .`

```go
func InitServerIndex() {
	for i := 0; i < 10; i++ {
		ServerIndex[i] = i+100
	}
}

func Select() int {
	return ServerIndex[rand.Intn(10)]
}
```

```go
func BenchmarkSelect(b *testing.B) {
	InitServerIndex()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		Select()
	}
}
func BenchmarkSelectParallel(b *testing.B) {
	InitServerIndex()
	b.ResetTimer()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			Select()
		}
	})
}
```
![result](10.png)
