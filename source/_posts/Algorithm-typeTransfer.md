---
title: 「算」 01. 类型转换
category_bar: true
date: 2022-11-13 22:29:45
tags:
categories: 算法数构
banner_img:
---

在做 leetcode 和自己写 go 程序的时候，总是会遇到一些类型转换的问题，在这里总结一下。

<!-- more -->

## int 

* int → int64: `i := int64(int)`

* int → uint64: `ui := uint64(int)`

* int → float: `f := float64(int)`

* int → string: `str := strconv.Itoa(int)`

* int64 → string: `str := strconv.FormatInt(int64, 10)`

## uint

* uint64 → string: `str := strconv.FormatUint(uint64, 10)`

## float

* float → int: `i := int(float)`

* float → string: `str := strconv.FormatFloat(float64, 'E', -1, 64)`

## string

* string → int: `i, err := strconv.Atoi(string)`

* string →float: `f, err := strconv.ParseFloat(string, 64)`

* string → bool: `b, err := strconv.ParseBool("true")`

* string → []byte: `b := []byte(string)`

## byte

[]byte → string: `str := string([]byte)`

## bool

* bool → string: `string := strconv.FormatBool(true)`

## interface

* interface→int: `interface.(int64)`

* interface→string: `interface.(string)`

* interface→float: `interface.(float64)`

* interface→bool: `interface.(bool)`

## 查看类型

```go
import(
	"fmt"
	"reflect"
)
 
func main() {
 	a := 1
 	fmt.Println("a type by reflect: ", reflect.TypeOf(a))
}
```