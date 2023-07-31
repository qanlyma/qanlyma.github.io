---
title: 「学」 Markdown 使用教程
category_bar: true
date: 2022-11-08 22:15:52
tags:
categories: 学习笔记
---

Markdown 是一种轻量级标记语言，在 2004 年由 John Gruber 创建。它允许人们使用易读易写的纯文本格式编写文档。
本文介绍使用 Markdown 编写博客时常用的标记符号供后续使用时参考。

<!-- more -->

## Markdown 标题

使用 # 号标记，一级标题对应一个 # 号，二级标题对应两个 # 号，以此类推。

## Markdown 格式

可以实现的格式： *斜体文本* ， **粗体文本** ， ***粗斜体文本*** ， ~~删除线~~ ， <u>带下划线文本</u>

分割线
***

## Markdown 列表

* 第一项
* 第二项
* 第三项

1. 第一项：
    - 第一项嵌套的第一个元素
    - 第一项嵌套的第二个元素
2. 第二项：
    - 第二项嵌套的第一个元素
    - 第二项嵌套的第二个元素

## Markdown 区块

> 最外层
> > 第一层嵌套
> > > 第二层嵌套
> > > 第二层嵌套

## Markdown 代码

`Println()` 函数输出 “Hello, world!” :

```Go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

## Markdown 链接

欢迎访问我的 [Github 仓库](https://github.com/qanlyma): <https://github.com/qanlyma>

## Markdown 图片

![我最喜欢的彩虹六号](love.jpg)

## Markdown 表格

| 左  对  齐 | 右  对  齐 | 居  中  对  齐 |
| :---------| --------: | :--------: |
| 单元格 | 单元格 | 单元格 |
| 单元格 | 单元格 | 单元格 |

## Markdown 高级

### 支持部分 HTML 元素

使用 <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Del</kbd> 重启电脑

### 转义

**文本加粗** 
\*\* 正常显示星号 \*\*
