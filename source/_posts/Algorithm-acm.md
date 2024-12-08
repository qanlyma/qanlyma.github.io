---
title: 「算」 14. 输入输出
category_bar: true
date: 2023-08-07 19:30:22
tags:
categories: 算法数构
banner_img:
---

ACM 模板，可参考：[牛客网 OJ 在线编程常见输入输出练习](https://ac.nowcoder.com/acm/contest/5657)。

<!-- more -->

* Scan 将碰到第一个空格或换行符之前的内容赋值给变量。如果 Scan 中有多个变量，变量值用空格或换行符分割。所以换行和空格是不能存储到变量内的。

* Scanln 和 Scan 基本相同，唯一区别是当读取多个变量当时候，遇到换行符 Scanln 会直接结束，未读到输入值的变量为零值。

* Scanf 按照指定格式对输入进行解析。

## 1 多行输入，每行两个整数

```go
package main

import (
    "fmt"
)

func main() {
    var a, b int
    for {
        n, _ := fmt.Scan(&a, &b) // 返回数据个数 n 和错误
        if n == 0 {
            break
        } else {
            fmt.Printf("%d\n", a + b)
        }
    }
}
```

## 2 多组数据，每组第一行为 n，之后输入 n 行两个整数

```go
package main

import (
    "fmt"
)

func main() {
    var a, b, n int
    fmt.Scan(&n)
    for n > 0 {
        fmt.Scan(&a, &b)
        fmt.Printf("%d\n", a + b)
        n--
    }
}
```

## 3 若干行输入，每行输入两个整数，遇到特定条件终止

```go
package main

import (
    "fmt"
)

func main() {
    var a, b int                    // 声明两个整数类型的变量a和b。
    for {                           // 开始一个无限循环。
        _, err := fmt.Scan(&a, &b)  // 从标准输入读取两个整数值，并将它们存储在a和b中。
        if err != nil {             // 如果读取过程中出现错误，
            break                   // 则跳出循环。
        }
        if a == 0 && b == 0 {       // 如果两个输入都是0，
            break                   // 则同样跳出循环。
        }
        fmt.Println(a + b)          // 否则打印这两个数的和。
    }
}
```

## 4 若干行输入，遇到 0 终止，每行第一个数为 n，表示本行后面有 n 个数

```go
package main

import (
    "fmt"
)

func main() {
    var n int
    for {
        fmt.Scan(&n)
        var sum, a int
        if n == 0 {
            break
        }
        for n > 0 {
            fmt.Scan(&a)
            sum += a
            n--
        }
        fmt.Printf("%d\n", sum)
    }
}
```

## 5 输入的第一行包括一个正整数 t，表示数据组数。接下来 t 行，每行一组数据。每行的第一个整数为整数的个数 n

```go
package main

import (
    "fmt"
)

func main() {
    var n, t int
    fmt.Scan(&t)
    for t > 0 {
        fmt.Scan(&n)
        var sum, a int
        if n == 0 {
            break
        }
        for n > 0 {
            fmt.Scan(&a)
            sum += a
            n--
        }
        fmt.Printf("%d\n", sum)
        t--
    }
}
```

## 6 输入数据有多组，每行表示一组输入数据。每行的第一个整数为整数的个数 n，不知道有多少行

```go
package main

import (
    "fmt"
)

func main() {
    var n int
    for {
        m, _ := fmt.Scan(&n)
        if m == 0 {
            break
        }
        var sum, a int
        for n > 0 {
            fmt.Scan(&a)
            sum += a
            n--
        }
        fmt.Printf("%d\n", sum)
    }
}
```

## 7 输入数据有多组，每行表示一组输入数据。每行不定有 n 个整数，空格隔开

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strconv"
    "strings"
)

func main() {
    inputs := bufio.NewScanner(os.Stdin)
    for inputs.Scan() { // 每次读入一行
        data := strings.Split(inputs.Text(), " ")
        var sum int
        for i := range data {
            val, _ := strconv.Atoi(data[i])
            sum += val
        }
        fmt.Println(sum)
    }
}
```

## 8 输入有两行，第一行 n；第二行是 n 个字符串，字符串之间用空格隔开

输出一行排序后的字符串，空格隔开，无结尾空格

```go
package main

import(
    "fmt"
    "os"
    "bufio"
    "sort"
    "strings"
)
 
func main(){
    in := bufio.NewScanner(os.Stdin)
    in.Scan()
    for in.Scan(){
        str := in.Text()
        s := strings.Split(str, " ")
        sort.Strings(s)
        fmt.Println(strings.Join(s, " "))
    }
}
```

## 9 多个测试用例，每个测试用例一行。每行通过空格隔开

对于每组测试用例，输出一行排序过的字符串，每个字符串通过空格隔开

```go
package main

import(
    "fmt"
    "os"
    "bufio"
    "sort"
    "strings"
)
 
func main(){
    in := bufio.NewScanner(os.Stdin)
    for in.Scan() { 
        str := in.Text()
        s := strings.Split(str, " ")
        sort.Strings(s)
        fmt.Println(strings.Join(s, " "))
    }
}
```

## 10 多个测试用例，每个测试用例一行。通过','隔开

对于每组用例输出一行排序后的字符串，用','隔开，无结尾空格

```go
package main

import(
    "fmt"
    "bufio"
    "os"
    "strings"
    "sort"
)

func main() {
    in := bufio.NewScanner(os.Stdin)
    for in.Scan() {
        strs := strings.Split(in.Text(), ",")
        sort.Strings(strs)
        fmt.Println(strings.Join(strs, ","))
    }
}
```
