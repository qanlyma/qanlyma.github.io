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

多个 goroutine 中，Go 语言使用通道（channel）进行通信，通道是一种内置的数据结构，可以让用户在不同的 goroutine 之间同步发送具有类型的消息。这让编程模型更倾向于在 goroutine 之间发送消息，而不是让多个 goroutine 争夺同一个数据的使用权。程序可以将需要并发的环节设计为生产者模式和消费者的模式，将数据放入通道，通道另外一端的代码将这些数据进行并发计算并返回结果。

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

```go
// runtime/slice.go
type slice struct {
	array unsafe.Pointer 	// 元素指针
	len   int 				// 长度 
	cap   int 				// 容量
}
```

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
var name1 []Type // nil 切片

name2 := make([]Type, size, cap) // 空切片
```

**使用 make() 函数生成的切片一定发生了内存分配操作，但给定开始与结束位置（包括切片复位）的切片只是将新的切片结构指向已经分配好的内存区域，设定开始与结束位置，不会发生内存分配操作。**

**nil 切片与空切片：**

* nil 切片：指针并不指向底层的数组，而是指向一个没有实际意义的地址；len = 0 且 cap = 0；`var s []int`
* 空切片：指针指向底层数组的地址；len = 0，容量由指向的底层数组决定；`s1 := []int{}` 或 `s2 := make([]int, 0)`
* nil 切片和空切片的区别主要在于指向的地址不同。

多维切片：

```go
slice := [][]int{{10}, {100, 200}}
```

![](12.png)

注意，底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。

```go
func main() {
	slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
	s1 := slice[2:5]
	s2 := s1[2:6:7]

	s2 = append(s2, 100)
	s2 = append(s2, 200)

	s1[2] = 20

	fmt.Println(s1)			// [2 3 20]
	fmt.Println(s2)			// [4 5 6 7 100 200]
	fmt.Println(slice)			// [0 1 2 3 20 5 6 7 100 9]
}
```

大致流程如下：

![](14.png)
![](15.png)
![](16.png)

s2 扩容之后更换了底层数组，所以不再受 s1 影响了。

![](17.png)

**扩容规则（1.18 版本之后）：**

* 当原 slice 容量小于 256 的时候，新 slice 容量为原来的 2 倍
* 原 slice 容量超过 256，新 slice 容量 newcap = oldcap + (oldcap+3*256)/4
* 由于内存对齐，新 slice 的容量要大于等于按照前半部分生成的 newcap

当 slice 作为函数参数时，就是一个普通的结构体。若直接传 slice，在调用者看来，实参 slice 并不会被函数中的操作改变；若传的是 slice 的指针，在调用者看来，是会被改变原 slice 的。

* 通过类似 `s[i] = 10` 这种操作可以改变 slice 底层数组元素值
* 在函数中使用 `s = append(s, 100)` 是无法改变外层 slice 的

#### 1.2.3 map

Go 语言中 map 是一种特殊的数据结构，一种元素对（pair）的**无序**集合，pair 对应一个 key（索引）和一个 value（值），所以这个结构也称为关联数组或字典，这是一种能够快速寻找值的理想结构。

在 Go 的内部实现中，map 是通过哈希表来实现的。哈希表用一个哈希函数将 key 分配到不同的桶（bucket，也就是数组的不同 index）。这样，开销主要在哈希函数的计算以及数组的常数访问时间。哈希查找表一般会存在“碰撞”的问题，就是说不同的 key 被哈希到了同一个 bucket。一般有两种应对方法：**链表法**和**开放地址法**。链表法将一个 bucket 实现成一个链表，落在同一个 bucket 中的 key 都会插入这个链表。开放地址法则是碰撞发生后，通过一定的规律，在数组的后面挑选“空位”，用来放置新的 key。Go 使用前者解决哈希碰撞问题。

* 当我们插入一个键值对时，首先会根据键的哈希值计算出对应的桶的索引。然后，如果该桶为空，直接将键值对放入其中；如果不为空，则需要遍历链表，查找是否已经存在相同的键。如果存在相同的键，那么会更新对应的值；如果不存在相同的键，会将新的键值对添加到链表的末尾。

* 当我们查询一个键的值时，也是通过计算哈希值找到对应的桶，然后遍历链表查找是否存在相同的键。如果找到了相同的键，就返回对应的值；如果遍历完链表仍然没有找到相同的键，就表示该键不存在。

```go
var mapname map[keytype]valuetype		// 若只有此行，添加元素会 panic
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
* **协程**：goroutine 是一种比线程更加轻量级的存在。一个进程可以有多个线程，一个线程可以有多个协程。

**区别：**

* **内存占用**

	创建一个 goroutine 的栈内存消耗为 2 KB，实际运行过程中，如果栈空间不够用，会自动进行扩容。创建一个 thread 则需要消耗 1 MB 栈内存，而且还需要一个被称为 “a guard page” 的区域用于和其他 thread 的栈空间进行隔离。

* **创建和销毀**

	Thread 创建和销毀都会有巨大的消耗，因为要和操作系统打交道，是内核级的，通常解决的办法就是线程池。而 goroutine 因为是由 Go runtime 负责管理的，创建和销毁的消耗非常小，是用户级。

* **切换**

	当 threads 切换时，需要保存各种寄存器，以便将来恢复；而 goroutines 切换只需保存三个寄存器：Program Counter，Stack Pointer，BP。goroutines 切换成本比 threads 要小得多。

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

### 2.2 CSP

Communicating Sequential Processes 提倡**通过通信共享内存**而不是通过共享内存而实现通信。

![CSP](8.png)

### 2.3 Channel

* 无缓冲通道：`make(chan int)`
* 有缓冲通道：`make(chan int, 2)`

应用：停止信号、任务定时、解耦生产方和消费方、控制并发数

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

| 操作 | nil channel | closed channel | not nil, not closed channel |
| - | - | - | - |
| close | panic | panic | 正常关闭 |
| 读 <- ch | 阻塞 | 读到对应类型的零值 | 阻塞或正常读取数据 |
| 写 ch <- | 阻塞 | panic | 阻塞或正常写入数据 |

Channel 可能会引发 goroutine 泄漏：

泄漏的原因是 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处于等待队列中，不见天日。

### 2.4 Lock 并发安全

Go 语言的锁：

* 互斥锁 `Mutex`
* 读写锁 `RWMutex`

```go
var (
	x    int64
	lock sync.Mutex // 互斥锁
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

## 3 面向对象编程

Go 语言的面向对象编程有三个重要的思想：

* 封装：Go 语言通过 struct 结构体的方式来实现封装，结构体可以包含各种类型的变量和方法，可以将一组相关的变量和方法封装在一起。使用首字母大小写控制变量和方法的访问权限，实现了信息隐藏和访问控制。

* 继承：Go 语言中没有传统的继承机制，但是可以使用嵌入类型来实现类似继承的效果，将一个类型嵌入到另一个类型中，从而继承嵌入类型的方法和属性。嵌入式类型的特点是可以直接访问嵌入类型的属性和方法，不需要通过接口或者其他方式进行转换。在 Go 语言中，可以通过 struct 嵌套和 interface 嵌套来实现继承的效果。

* 多态：Go 语言通过接口来实现多态，一个类型只需要实现了接口的所有方法，就可以被赋值给该接口类型的变量。这样可以实现类似于面向对象语言中的多态性。多态性使得程序可以根据上下文环境自动选择合适的方法实现，提高了代码的灵活性和可复用性。

## 4 标准库

### 4.1 context

context 主要用来在 goroutine 之间传递上下文信息，包括：取消信号、超时时间、截止时间、k-v 等。context.Context 类型的值可以协调多个 groutine 中的代码执行“取消”操作，并且可以存储键值对。最重要的是它是并发安全的。

Go 常用来写后台服务，通常只需要几行代码，就可以搭建一个 http server。在 Go 的 server 里，通常每来一个请求都会启动若干个 goroutine 同时工作：有些去数据库拿数据，有些调用下游接口获取相关数据......

![](18.png)

这些 goroutine 需要共享这个请求的基本数据，例如登陆的 token，处理请求的最大超时时间等等。当请求被取消或是处理时间太长，这有可能是使用者关闭了浏览器或是已经超过了请求方规定的超时时间，请求方直接放弃了这次请求结果。这时，所有正在为这个请求工作的 goroutine 需要快速退出，因为它们不再被需要了。在相关联的 goroutine 都退出后，系统就可以回收相关的资源。

Go 语言中的 server 实际上是一个**协程模型**，也就是说一个协程处理一个请求。例如在业务的高峰期，某个下游服务的响应变慢，而当前系统的请求又没有超时控制，或者超时时间设置地过大，那么等待下游服务返回数据的协程就会越来越多。而我们知道，协程是要消耗系统资源的，后果就是协程数激增，内存占用飙涨，甚至导致服务不可用。更严重的会导致雪崩效应，整个服务对外表现为不可用，这肯定是 P0 级别的事故。context 包就是为了解决上面所说问题而开发的：在一组 goroutine 之间传递共享的值、取消信号、deadline 等等。

![](19.png)

```go
type Context interface {
	Done() <-chan struct{}
	Err() error
	Deadline() (deadline time.Time, ok bool)
	Value(key interface{}) interface{}
}
```

* Done() 返回一个 channel，可以表示 context 被取消的信号：当这个 channel 被关闭时，说明 context 被取消了。注意，这是一个只读的 channel。读一个关闭的 channel 会读出相应类型的零值，并且源码里没有地方会向这个 channel 里面塞入值。换句话说，这是一个 receive-only 的 channel。因此在子协程里读这个 channel，除非被关闭，否则读不出来任何东西。也正是利用了这一点，子协程从 channel 里读出了值（零值）后，就可以做一些收尾工作，尽快退出。

* Err() 返回一个错误，表示 channel 被关闭的原因。例如是被取消，还是超时。

* Deadline() 返回 context 的截止时间，通过此时间，函数就可以决定是否进行接下来的操作，如果时间太短，就可以不往下做了，否则浪费系统资源。当然，也可以用这个 deadline 来设置一个 I/O 操作的超时时间。

* Value() 获取之前设置的 key 对应的 value。

对于 Web 服务端开发，往往希望将一个请求处理的整个过程串起来，这就非常依赖于 Thread Local（对于 Go 可理解为单个协程所独有）的变量，而在 Go 语言中并没有这个概念，因此需要在函数调用的时候传递 context。

现实场景中可能是从一个 HTTP 请求中获取到的 Request-ID：

```go
const requestIDKey int = 0

func WithRequestID(next http.Handler) http.Handler {
	return http.HandlerFunc(
		func(rw http.ResponseWriter, req *http.Request) {
			// 从 header 中提取 request-id
			reqID := req.Header.Get("X-Request-ID")
			// 创建 valueCtx。使用自定义的类型，不容易冲突
			ctx := context.WithValue(
				req.Context(), requestIDKey, reqID)
			
			// 创建新的请求
			req = req.WithContext(ctx)
			
			// 调用 HTTP 处理函数
			next.ServeHTTP(rw, req)
		}
	)
}

// 获取 request-id
func GetRequestID(ctx context.Context) string {
	ctx.Value(requestIDKey).(string)
}

func Handle(rw http.ResponseWriter, req *http.Request) {
	// 拿到 reqId，后面可以记录日志等等
	reqID := GetRequestID(req.Context())
	...
}

func main() {
	handler := WithRequestID(http.HandlerFunc(Handle))
	http.ListenAndServe("/", handler)
}
```

取消 goroutine：我们先来设想一个场景：打开外卖的订单页，地图上显示外卖小哥的位置，而且是每秒更新 1 次。app 端向后台发起 websocket 连接（现实中可能是轮询）请求后，后台启动一个协程，每隔 1 秒计算 1 次小哥的位置，并发送给端。如果用户退出此页面，则后台需要取消此过程，退出 goroutine，系统回收资源。

```go
func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
	go Perform(ctx)

	// ……
	// app 端返回页面，调用 cancel 函数
	cancel()
}

func Perform(ctx context.Context) {
    for {
        calculatePos()
        sendResult()

        select {
        case <-ctx.Done():
            // 被取消，直接返回
            return
        case <-time.After(time.Second):
            // block 1 秒钟 
        }
    }
}
```

WithTimeOut 函数返回的 context 和 cancelFun 是分开的。context 本身并没有取消函数，这样做的原因是取消函数只能由外层函数调用，防止子节点 context 调用取消函数，从而严格控制信息的流向：由父节点 context 流向子节点 context。

### 4.2 reflect

Go 语言提供了一种机制在运行时更新变量和检查它们的值、调用它们的方法，但是在编译时并不知道这些变量的具体类型，这称为反射机制。

使用反射的常见场景有以下两种：

* 不能明确接口调用哪个函数，需要根据传入的参数在运行时决定。
* 不能明确传入函数的参数类型，需要在运行时处理任意对象。

反射的缺点：

* 与反射相关的代码，经常是难以阅读的。
* Go 语言作为一门静态语言，编码过程中编译器能提前发现一些类型错误，但是对于反射代码是无能为力的。
* 反射对性能影响比较大，比正常代码运行速度慢一到两个数量级。

### 4.3 unsafe

相比于 C 语言中指针的灵活，Go 的指针多了一些限制：

* 限制一：Go 的指针不能进行数学运算。
* 限制二：不同类型的指针不能相互转换。
* 限制三：不同类型的指针不能使用 == 或 != 比较。
* 限制四：不同类型的指针变量不能相互赋值。

unsafe 包提供了 2 点重要的能力：

1. 任何类型的指针和 unsafe.Pointer 可以相互转换。
2. uintptr 类型和 unsafe.Pointer 可以相互转换。

pointer 不能直接进行数学运算，但可以把它转换成 uintptr，对 uintptr 类型进行数学运算，再转换成 pointer 类型。uintptr 并没有指针的语义，意思就是 uintptr 所指向的对象会被 gc 无情地回收。而 unsafe.Pointer 有指针语义，可以保护它所指向的对象在“有用”的时候不会被垃圾回收。

**如何实现字符串和 byte 切片的零拷贝转换：原理上是利用指针的强转。**

完成这个任务，我们需要了解 slice 和 string 的底层数据结构：

```go
type StringHeader struct {
	Data uintptr
	Len  int
}

type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

上面是反射包下的结构体，路径：src/reflect/value.go。只需要共享底层 Data 和 Len 就可以实现 zero-copy：

```go
func string2bytes(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(&s))
}
func bytes2string(b []byte) string{
	return *(*string)(unsafe.Pointer(&b))
}
```

## 5 依赖管理

* **GOPATH**：设置环境变量，项目代码依赖 src 下的代码。无法实现 package 的多版本控制。
* **Go Vendor**：项目目录下增加 vendor 文件存放依赖包副本。同一项目无法依赖一个 package 的两个不同版本。
* **Go Module**：通过 go.mod 文件管理依赖包版本

依赖管理三要素：

1. 配置文件，描述依赖：go.mod
2. 中心仓库管理依赖库：Proxy
3. 本地工具：go get/mod

![Proxy](9.png)

## 6 测试

### 6.1 单元测试

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

### 6.2 Mock

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

### 6.3 Benchmark

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
