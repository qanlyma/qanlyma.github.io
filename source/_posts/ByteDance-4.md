---
title: 「训」 笔记(4)：框架三件套
category_bar: true
date: 2023-03-10 14:00:49
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

* **基本使用**

    ```go
    package main

    import (
    "gorm.io/gorm"
    "gorm.io/driver/sqlite"
    )

    type Product struct {
        gorm.Model
        Code  string
        Price uint
    }

    func main() {
        // 连接数据库
        db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
        if err != nil {
            panic("failed to connect database")
        }   

        // 迁移 schema
        db.AutoMigrate(&Product{})

        // Create
        db.Create(&Product{Code: "D42", Price: 100})

        // Read
        var product Product
        db.First(&product, 1) // 根据整形主键查找
        db.First(&product, "code = ?", "D42") // 查找 code 字段值为 D42 的记录

        // Update - 将 product 的 price 更新为 200
        db.Model(&product).Update("Price", 200)
        // Update - 更新多个字段
        db.Model(&product).Updates(Product{Price: 200, Code: "F42"}) // 仅更新非零值字段
        db.Model(&product).Updates(map[string]interface{}{"Price": 200, "Code": "F42"})

        // Delete - 删除 product
        db.Delete(&product, 1)
    }
    ```

* **连接数据库**

    GORM 官方支持的数据库类型有： MySQL, PostgreSQL, SQlite, SQL Server。
    ```go
    import (
        "gorm.io/driver/mysql"
        "gorm.io/gorm"
    )

    func main() {
        // 参考 https://github.com/go-sql-driver/mysql#dsn-data-source-name 获取详情
        dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
        db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    }
    ```

## Kitex

Kitex[kaɪt'eks] 字节跳动内部的 Golang 微服务 **RPC（Remote Procedure Call）框架**，具有高性能、强可扩展的特点，在字节内部已广泛使用。

快速开始：<https://www.cloudwego.io/zh/docs/kitex/getting-started>
框架地址：<https://github.com/cloudwego/kitex>

* **定义 IDL**

    如果我们要进行 RPC，就需要知道对方的接口是什么，需要传什么参数，同时也需要知道返回值是什么样的，就好比两个人之间交流，需要保证在说的是同一个语言、同一件事。 这时候，就需要通过 IDL（Interface Definition Language）来约定双方的协议，就像在写代码的时候需要调用某个函数，我们需要知道函数签名一样。
    ```thrift
    namespace go api

    struct Request {
        1: string message
    }

    struct Response {
        1: string message
    }

    service Hello {
        Response echo(1: Request req)
    }
    ```

* **编写 echo 服务逻辑**
  
    服务默认监听 8888 端口。
    ```go
    package main

    import (
        "context"
        "example/kitex_gen/api"
    )

    // EchoImpl implements the last service interface defined in the IDL.
    type EchoImpl struct{}

    // Echo implements the EchoImpl interface.
    func (s *EchoImpl) Echo(ctx context.Context, req *api.Request) (resp *api.Response, err error) {
        // TODO: Your code here...
        return
    }
    ```

* **创建 client**

    ```go
    import "example/kitex_gen/api/echo"
    import "github.com/cloudwego/kitex/client"
    ...
    c, err := echo.NewClient("example", client.WithHostPorts("0.0.0.0:8888"))
    if err != nil {
        log.Fatal(err)
    }
    ```
    上述代码中，echo.NewClient 用于创建 client，其第一个参数为调用的服务名，第二个参数为 options，用于传入参数，此处的 client.WithHostPorts 用于指定服务端的地址。

* **发起调用**

    ```go 
    import "example/kitex_gen/api"
    ...
    req := &api.Request{Message: "my request"}
    resp, err := c.Echo(context.Background(), req, callopt.WithRPCTimeout(3*time.Second))
    if err != nil {
        log.Fatal(err)
    }
    log.Println(resp)
    ```
    上述代码中，我们首先创建了一个请求 req , 然后通过 c.Echo 发起了调用。

## Hertz

Hertz[həːts] 是一个 Golang 微服务 **HTTP 框架**，在设计之初参考了其他开源框架 fasthttp、gin、echo 的优势，并结合字节跳动内部的需求，使其具有高易用性、高性能、高扩展性等特点。

快速开始：<https://www.cloudwego.io/zh/docs/hertz/getting-started>
框架地址：<https://github.com/cloudwego/hertz>

* **基本使用**

    实现服务监听 8080 端口并注册一个 GET 方法的路由函数。
    ```go
    package main

    import (
        "context"

        "github.com/cloudwego/hertz/pkg/app"
        "github.com/cloudwego/hertz/pkg/app/server"
        "github.com/cloudwego/hertz/pkg/common/utils"
        "github.com/cloudwego/hertz/pkg/protocol/consts"
    )

    func main() {
        h := server.Default(server.WithHostPorts("127.0.0.1:8080"))

        h.GET("/ping", func(c context.Context, ctx *app.RequestContext) {
            ctx.JSON(consts.StatusOK, utils.H{"message": "pong"})
        })

        h.Spin()
    }
    ```