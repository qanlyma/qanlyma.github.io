---
title: 「训」 笔记(4)：框架三件套
category_bar: true
date: 2023-03-08 15:42:49
tags:
categories: 字节青训
banner_img:
---

本文介绍 Gorm、Kitex、Hertz 三件套。

<!-- more -->

## Gorm

Gorm 是面向 Golang 语言的一种 **ORM（Object Relational Mapping，对象关系映射）框架**，支持多种数据库的接入，例如 MySQL，PostgreSQL，SQLite，SQL Server，Clickhouse。此框架弱化了开发者对于 SQL 语言的掌握程度，使用提供的 API 进行底层数据库的访问。

快速开始：<https://gorm.cn/zh_CN/docs/index.html>
框架地址：<https://github.com/go-gorm/gorm>

## Kitex

Kitex[kaɪt'eks] 字节跳动内部的 Golang 微服务 **RPC（Remote Procedure Call）框架**，具有高性能、强可扩展的特点，在字节内部已广泛使用。

快速开始：<https://www.cloudwego.io/zh/docs/kitex/getting-started>
框架地址：<https://github.com/cloudwego/kitex>

## Hertz

Hertz[həːts] 是一个 Golang 微服务 **HTTP 框架**，在设计之初参考了其他开源框架 fasthttp、gin、echo 的优势，并结合字节跳动内部的需求，使其具有高易用性、高性能、高扩展性等特点。

快速开始：<https://www.cloudwego.io/zh/docs/hertz/getting-started>
框架地址：<https://github.com/cloudwego/hertz>
