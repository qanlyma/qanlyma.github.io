---
title: 「训」 笔记(01)：Go 语言
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

### 1.1 简介

在早期 CPU 都是以单核的形式顺序执行机器指令。Go 语言的祖先 C 语言正是这种顺序编程语言的代表。顺序编程语言中的顺序是指：所有的指令都是以串行的方式执行，在相同的时刻有且仅有一个 CPU 在顺序执行程序的指令。

随着处理器技术的发展，单核时代以提升处理器频率来提高运行效率的方式遇到了瓶颈，单核 CPU 发展的停滞，给多核 CPU 的发展带来了机遇。相应地，编程语言也开始逐步向并行化的方向发展。

Go 语言正是在多核和网络化的时代背景下诞生的原生支持并发的编程语言。Go 语言从底层原生支持并发，无须第三方库，开发人员可以很轻松地在编写程序时决定怎么使用 CPU 资源。

Go 语言的并发是基于 goroutine 的，goroutine 类似于线程但并非线程。可以将 goroutine 理解为一种虚拟线程。Go 语言运行时会参与调度 goroutine，并将 goroutine 合理地分配到每个 CPU 中，最大限度地使用 CPU 性能。

多个 goroutine 中，Go 语言使用通道（channel）进行通信，通道是一种内置的数据结构，可以让用户在不同的 goroutine 之间同步发送具有类型的消息。这让编程模型更倾向于在 goroutine 之间发送消息，而不是让多个 goroutine 争夺同一个数据的使用权。程序可以将需要并发的环节设计为生产者模式和消费者的模式，将数据放入通道。通道另外一端的代码将这些数据进行并发计算并返回结果。**Go 语言通过通道可以实现多个 goroutine 之间内存共享**。

**特点**：

* 高性能、高并发
* 语法简单、学习曲线平滑
* 丰富的标准库
* 完善的工具链
* 静态链接
* 快速编译
* 跨平台
* 垃圾回收

### 1.2 容器

#### 1.2.1 array

数组是一段固定长度的**连续内存区域**。在 Go 语言中，数组从声明时就确定，使用时可以修改数组成员，但是数组大小不可变化，因此在 Go 语言中很少直接使用数组。

```go
var 数组变量名 [元素数量]Type
```

也可以是多维数组：

```go
var array_name [size1][size2]...[sizen] array_type
```

#### 1.2.2 slice

切片（slice）是对数组的一个连续片段的引用（左闭右开），所以切片是一个引用类型。Go 语言中切片的内部结构包含地址（指针）、大小和容量。

切片默认指向一段连续内存区域，可以是数组，也可以是切片本身：

```go
slice [开始位置 : 结束位置]
```

* 取出的元素数量为：结束位置 - 开始位置；
* 取出元素不包含结束位置对应的索引，切片最后一个元素使用 slice[len(slice)-1] 获取；
* 当缺省开始位置时，表示从连续区域开头到结束位置；
* 当缺省结束位置时，表示从开始位置到整个连续区域末尾；
* 两者同时缺省时，与切片本身等效；
* 两者同时为 0 时，等效于空切片，一般用于**切片复位**。

切片类型也可以被声明:

```go
var name1 []Type

name2 := make([]Type, size, cap)
```

**使用 make() 函数生成的切片一定发生了内存分配操作，但给定开始与结束位置（包括切片复位）的切片只是将新的切片结构指向已经分配好的内存区域，设定开始与结束位置，不会发生内存分配操作。**

多维切片：

```go
slice := [][]int{{10}, {100, 200}}
```

![](12.png)

#### 1.2.3 map

Go 语言中 map 是一种特殊的数据结构，一种元素对（pair）的**无序**集合，pair 对应一个 key（索引）和一个 value（值），所以这个结构也称为关联数组或字典，这是一种能够快速寻找值的理想结构，给定 key，就可以迅速找到对应的 value。

在 Go 的内部实现中，map 是通过哈希表来实现的。哈希表是一种基于哈希函数的数据结构，它可以将键映射到对应的值上。map 的底层数据结构是一个哈希表的数组，每个元素都是一个桶（bucket）。每个桶中存储了一个键值对，如果多个键映射到同一个桶中，它们会以链表的形式连接起来。

* 当我们插入一个键值对时，首先会根据键的哈希值计算出对应的桶的索引。然后，如果该桶为空，直接将键值对放入其中；如果不为空，则需要遍历链表，查找是否已经存在相同的键。如果存在相同的键，那么会更新对应的值；如果不存在相同的键，会将新的键值对添加到链表的末尾。

* 当我们查询一个键的值时，也是通过计算哈希值找到对应的桶，然后遍历链表查找是否存在相同的键。如果找到了相同的键，就返回对应的值；如果遍历完链表仍然没有找到相同的键，就表示该键不存在。

```go
var mapname map[keytype]valuetype
mapname = make(map[keytype]valuetype)
```

使用 delete() 内建函数从 map 中删除一组键值对：

```go
delete(map, 键)
```

Go 语言中并没有为 map 提供任何清空所有元素的函数、方法，清空 map 的唯一办法就是重新 make 一个新的 map，不用担心垃圾回收的效率，Go 语言中的并行垃圾回收效率比写一个清空函数要高效的多。

#### 1.2.4 list

列表是一种非连续的存储容器，由多个节点组成，节点通过一些变量记录彼此之间的关系，列表有多种实现方法，如单链表、双链表等。

在 Go 语言中，列表使用 `container/list` 包来实现，内部的实现原理是双链表，列表能够高效地进行任意位置的元素插入和删除操作。

```go
list1 := list.New()

var list2 list.List
```

#### 1.2.5 nil

在 Go 语言中，布尔类型的零值（初始值）为 `false`，数值类型的零值为 `0`，字符串类型的零值为空字符串 `""`，而指针、切片、映射、通道、函数和接口的零值则是 `nil`。

**不同类型的 nil 是不能比较的**，不同类型的 nil 值占用的内存大小可能是不一样的。

### 1.3 基础语法

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

### 1.4 关键字和标识符

Go 语言中的关键字一共有 25 个：

![](11.png)

预定义标识符一共有 36 个：

![](13.png)

**make 和 new**：

make 用于初始化内置的数据结构，如数组、切片和 Channel 等；new 用于分配并创建一个指向对应类型的指针。

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

### 2.6 指针

Go 语言为程序员提供了控制数据结构指针的能力，但是并不能进行指针运算。Go 语言允许你控制特定集合的数据结构、分配的数量以及内存访问模式。

#### 2.6.1 基础概念

指针（pointer）在 Go 语言中可以被拆分为两个核心概念：

* 类型指针，允许对这个指针类型的数据进行修改，传递数据可以直接使用指针，而无须拷贝数据，类型指针不能进行偏移和运算。

* 切片，由指向起始元素的原始指针、元素数量和容量组成。

受益于这样的约束和拆分，Go 语言的指针类型变量既拥有指针高效访问的特点，又不会发生指针偏移，从而避免了非法修改关键性数据的问题。同时，垃圾回收也比较容易对不会发生偏移的指针进行检索和回收。

一个指针变量可以指向任何一个值的内存地址，它所指向的值的内存地址在 32 和 64 位机器上分别占用 4 或 8 个字节，占用字节的大小与所指向的值的大小无关。当一个指针被定义后没有分配到任何变量时，它的默认值为 nil。

每个变量在运行时都拥有一个地址，这个地址代表变量在内存中的位置。Go 语言中使用在变量名前面添加 `&` 操作符（前缀）来获取变量的内存地址（取地址操作），格式如下：
```go
ptr := &v    // v 的类型为 T
value := *ptr
```
取地址操作符 `&` 和取值操作符 `*` 是一对互补操作符，`&` 取出地址，`*` 根据地址取出地址指向的值。

#### 2.6.2 使用指针修改值

```go
func swap(a, b *int) {
    // 取 a 指针的值, 赋给临时变量 t
    t := *a
    // 取 b 指针的值, 赋给 a 指针指向的变量
    *a = *b
    // 将 a 指针的值赋给 b 指针指向的变量
    *b = t
}

func main() {
    x, y := 1, 2
    swap(&x, &y)
}
```
`*` 操作符作为**右值**时，意义是取指针的值，作为**左值**时，也就是放在赋值操作符的左边时，表示 a 指针指向的变量。其实归纳起来，`*` 操作符的根本意义就是操作指针指向的变量。

#### 2.6.3 new() 函数

Go 语言还提供了另外一种方法来创建指针变量：`new(类型)`

```go
str := new(string)
*str = "Go语言教程"
fmt.Println(*str)
```

new() 函数可以创建一个对应类型的指针，创建过程会分配内存，被创建的指针指向默认值。

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
