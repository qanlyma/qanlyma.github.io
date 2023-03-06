---
title: 「训」 笔记（2）：性能调优
category_bar: true
date: 2023-03-05 12:07:42
tags:
categories: 字节青训
banner_img:
---

请参考：[性能优化建议代码](https://github.com/RaymondCode/go-practice)，[pprof 示例代码](https://github.com/wolfogre/go-pprof-practice)

<!-- more -->

## 1 性能优化建议

### 1.1 Slice

**预分配内存**：使用 `make()` 初始化切片时提供容量信息。

* 切片本质是一个数组片段的描述，包括：
    * 数组指针
    * 片段的长度
    * 片段的容量
* 切片操作并不复制切片指向的元素
* 创建一个新的切片会复用原来切片的底层数组
* 切片有三个属性，指针(ptr)、长度(len) 和容量(cap)。append 时有两种场景：
    * 当 append 之后的长度小于等于 cap，将会直接利用原底层数组剩余的空间
    * 当 append 后的长度大于 cap 时，则会分配一块更大的区域来容纳新的底层数组

![append](2.png)

因此，为了避免内存发生拷贝，如果能够知道最终的切片的大小，预先设置 cap 的值能够获得最好的性能。

```go
func NoPreAlloc(size int) {
	data := make([]int, 0)
	for k := 0; k < size; k++ {
		data = append(data, k)
	}
}

func PreAlloc(size int) {
	data := make([]int, 0, size)
	for k := 0; k < size; k++ {
		data = append(data, k)
	}
}
```

![result](1.png)

**另一个陷阱**：大内存得不到释放。

在已有切片的基础上进行切片，不会创建新的底层数组。因为原来的底层数组没有发生变化，内存会一直占用，直到没有变量引用该数组因此很可能出现这么一种情况，原切片由大量的元素构成，但是我们在原切片的基础上切片，虽然只使用了很小一段，但底层数组在内存中仍然占据了大量空间，得不到释放。

推荐的做法：**使用 copy 替代 re-slice**

```go
// slice.go
func GetLastBySlice(origin []int) []int {
	return origin[len(origin)-2:]
}

func GetLastByCopy(origin []int) []int {
	result := make([]int, 2)
	copy(result, origin[len(origin)-2:])
	return result
}

// slice_test.go
func testGetLast(t *testing.T, f func([]int) []int) {
	result := make([][]int, 0)
	for k := 0; k < 100; k++ {
		origin := generateWithCap(128 * 1024) // 1M
		result = append(result, f(origin))
	}
	printMem(t)
	_ = result
}

func TestLastBySlice(t *testing.T) {
	testGetLast(t, GetLastBySlice)
}

func TestLastByCopy(t *testing.T) {
	testGetLast(t, GetLastByCopy)
}
```

![result](3.png)

### 1.2 Map

**预分配内存**
* 不断向 map 中添加元素的操作会触发 map 的扩容
* 根据实际需求提前预估好需要的空间
* 提前分配好空间可以减少内存拷贝和 Rehash 的消耗

### 1.3 字符串处理

使用 strings.Builder 代替 "+"

```go
func Plus(n int, str string) string {
	s := ""
	for i := 0; i < n; i++ {
		s += str
	}
	return s
}

func StrBuilder(n int, str string) string {
	var builder strings.Builder
	for i := 0; i < n; i++ {
		builder.WriteString(str)
	}
	return builder.String()
}

func ByteBuffer(n int, str string) string {
	buf := new(bytes.Buffer)
	for i := 0; i < n; i++ {
		buf.WriteString(str)
	}
	return buf.String()
}
```

![分别传入(1000, "string")](4.png)

* 字符串在 Go 语言中是不可变类型，占用内存大小是固定的，当使用 "+" 拼接 2 个字符串时，生成一个新的字符串，那么就需要开辟一段新的空间，新空间的大小是原来两个字符串的大小之和
* strings.Builder，bytes.Buffer 的内存是以倍数申请的
* strings.Builder 和 bytes.Buffer 底层都是 []byte 数组，bytes.Buffer 转化为字符串时重新申请了一块空间，存放生成的字符串变量，而 strings.Builder 直接将底层的 []byte 转换成了字符串类型返回

### 1.4 空结构体

**使用空结构体节省内存**
* 空结构体不占据内存空间，可作为占位符使用。比如实现简单的 Set：Go 语言标准库没有提供 Set 的实现，通常使用 map 来代替。对于集合场景，只需要用到 map 的键而不需要值。

## 2 性能调优实战

### 2.1 分析工具

![pprof](5.png)

### 2.2 搭建项目

[参考教程](https://blog.wolfogre.com/posts/go-ppof-practice/)
项目提前埋入了一些炸弹代码，产生可观测的性能问题。

```go
import (
	"log"
	"net/http"
	_ "net/http/pprof"  // 自动注册 pprof 的 handler 到 http server
    // ...
)

func main() {
	log.SetFlags(log.Lshortfile | log.LstdFlags)
	log.SetOutput(os.Stdout)

	runtime.GOMAXPROCS(1)                   // 限制 CPU 使用数
	runtime.SetMutexProfileFraction(1)      // 开启锁调用跟踪
	runtime.SetBlockProfileRate(1)          // 开启阻塞调用跟踪

	go func() {
        // 启动 http server
		if err := http.ListenAndServe(":6060", nil); err != nil {
			log.Fatal(err)
		}
		os.Exit(0)
	}()

	for {
		for _, v := range animal.AllAnimals {
			v.Live()
		}
		time.Sleep(time.Second)
	}
}
```

### 2.3 浏览器查看指标

![debug](6.png)

* CPU

![CPU](7.png)

* `go tool pprof "http://127.0.0.1:6060/debug/pprof/profile?seconds=10"`
* `topN` 查看占用资源最多的函数
* `list` 根据指定的正则表达式查找代码行
* `web` 调用关系可视化

![排查过程](8.png)

* heap-堆内存
* goroutine-协程
* mutex-锁
* block-阻塞
