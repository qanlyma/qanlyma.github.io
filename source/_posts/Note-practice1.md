---
title: 「学」 Practice
category_bar: true
date: 2024-11-12 23:24:48
tags:
categories: 学习笔记
banner_img:
---

生成平台后端项目。

<!-- more -->

## 1 整体架构

系统保持前后端分离 + Spring Boot 单体技术架构，包括前端控制台，业务后台，算法服务之间采用 HTTP 协议通信，JSON 数据格式进行交互。

* 前端采用 Vue.js 2.x、Bootstrap 和 ElementUI 开发，并使用 Nginx 作为反向代理服务器。
* 后端基于 Spring Boot 实现，包括公共模块和业务模块，并使用 RocketMQ 进行异步任务处理。
* 文件存储使用 MinIO 服务器，专门存储图片和视频数据。
* 数据库使用 MySQL 存储关系型数据，Redis 用于缓存和非关系型数据存储。
* 业务后台与算法服务通过 HTTP 协议交互，可根据性能需求增加相应服务器。

![](1.png)

## 2 项目目录

![](2.png)

### 2.1 aspect

这个包用于存放切面类，这些类主要用于实现 AOP（面向切面编程）的功能，如事务管理、日志记录等。在此项目中用于记录每一次从前端收到的请求和返回值并输出到 log。

### 2.2 config

配置相关的类或文件通常放在这个包中，比如 Spring 框架的配置类、数据库连接配置、安全配置等：

* 配置和初始化 MinIO 客户端，使得应用程序能够与 MinIO 服务器进行交互。
* Mybatis 的分页功能，MyBatis是一个流行的持久层框架，它简化了 Java 应用程序与关系数据库之间的交互。MyBatis 的主要功能是将 Java 对象与 SQL 数据库中的记录进行映射，从而简化数据库操作。
* WebSocket 的配置，WebSocket是一种全双工通信协议，允许在客户端和服务器之间建立持久连接，从而实现实时数据传输。
* Swagger 的配置，使其能够自动生成 API 文档。

### 2.3 controller

该包下的类通常是控制器，负责接收前端请求并调用服务层处理业务逻辑，然后返回响应给前端。这是 MVC 架构中的 C（Controller）部分。

```Java
@RestController
@RequestMapping("/generateApi")
@Api(tags = "生成任务服务")
public class GenerateTaskController {
    @Autowired
    private GenerateTaskService generateTaskService;
    @PostMapping("/taskGenerate")
    @ApiOperation("任务生成")
    public ResponseData taskGenerate(@Valid @RequestBody GenerateParams params) {
        return generateTaskService.taskGenerate(params);
    }
}
```

* `@RestController` 注解，说明它是一个 RESTful 风格的控制器，所有的方法返回值都会直接写入 HTTP 响应体中，而不是返回视图名。
* `@RequestParam` 可以将请求参数绑定到你的控制器方法参数上。当客户端向服务器发送请求时，如果 URL 中包含请求参数，那么 Spring MVC 会尝试匹配并将这些请求参数绑定到使用 `@RequestParam` 注解的方法参数上。
* 这个控制器类中定义了一个 POST 请求的处理方法 `taskGenerate`，该方法接收一个 `GenerateParams` 类型的请求体参数，并调用 `GenerateTaskService` 的 `taskGenerate` 方法处理这个请求，最后返回处理结果
* POST 请求的 URL 路径是 `/generateApi/taskGenerate`，这是由类级别的 `@RequestMapping("/generateApi")` 和方法级别的 `@PostMapping("/taskGenerate")` 两个注解共同决定的。
* 方法参数前的 `@Valid` 注解表示启用对这个参数的数据校验，`@RequestBody` 注解表示这个参数的值来自于请求体。

### 2.4 entity

实体类通常存放于此包中，每一个类代表了数据库中的一个表结构，用于 ORM（对象关系映射）操作。

* `@Data` 是 Lombok 的注解，会自动为类的所有属性生成 getter 和 setter 方法，以及 equals、canEqual、hashCode、toString 方法。
* `@TableName`是 MyBatis-Plus 的注解，表示这个类对应的数据库表名。
* 类中的每个属性都对应着数据表中的一个字段，用 `@TableField` 注解标注。`value` 属性表示对应的数据库字段名。

### 2.5 enums

枚举类型放在这个包里，用于定义一些固定值的集合，例如状态码、操作类型等。

### 2.6 handle

用于处理特定任务的类，例如异常处理、数据转换等。

### 2.7 listener

监听器类位于此包内，它们可以监听应用程序中的某些事件，如上下文初始化/销毁、请求开始/结束等，并执行相应的操作。此包中定义了一个名为 TaskListener 的类，它是一个 RocketMQ 的消息监听器，当有新的消息到达时，RocketMQ 会自动调用这个类的 `onMessage` 方法。

* `@Component`：这个类被标记为 Spring 的组件，所以 Spring 会自动创建这个类的实例，并且可以通过自动装配将这个实例注入到其他的 Bean 中。
* `@RocketMQMessageListener`：这是 RocketMQ 提供的注解，用于标记一个类为 RocketMQ 的消息监听器。`consumerGroup` 属性表示消费者组名，`topic` 属性表示要监听的主题，`consumeMode` 属性表示消费模式，这里设置为 `ConsumeMode.ORDERLY` 表示顺序消费。
* 在 `onMessage` 方法中，首先从 Redis 中获取一个锁，然后将接收到的消息转换为 `TaskCenterListener` 对象。然后根据任务的状态进行不同的处理。如果任务的状态是 0（排队中），则将状态改为 1（进行中）。如果任务的状态是 3 或 4（已结束），则不做任何处理并直接返回。
* 根据任务的算法类型调用不同的服务进行处理。如果在处理过程中发生了异常，那么将任务的状态设置为 4（异常结束），并记录异常信息。
* 在处理完任务后，会将任务的状态更新到 Redis 中，并更新任务的数据库记录。最后，释放 Redis 中的锁，表示任务处理结束。

### 2.8 mapper

MyBatis 的 Mapper 接口是用于定义数据库操作的方法的接口。每个方法对应数据库中的一种操作，如查询、插入、更新和删除等。这些方法的名称和参数需要与 Mapper XML 文件中的 SQL 语句对应。

### 2.9 model

* **DTO**：数据传输对象，用于在不同层之间传递数据，特别是在跨越网络边界时。通常用于将数据库查询结果或业务逻辑处理结果封装成一个对象，以便在不同层之间传输。

* **Params**：参数对象，用于封装控制器层接收到的请求参数。Params 对象通常用于将多个请求参数组合成一个对象，便于方法调用和参数校验。

* **VO**：值对象，用于封装业务逻辑层处理后的数据，通常用于展示层（如视图层）的数据展示。VO 通常包含业务逻辑处理后的最终结果，可以直接用于渲染页面或返回给前端。

### 2.10 service

服务层，包含了业务逻辑的实现。通常，控制器会调用服务层的方法来完成具体的业务需求。定义了不同的接口，并且在 `\impl` 中进行了实现。

`@Service` 是 Spring 框架中的一个注解，用于标记类作为服务层的组件。它是 `@Component` 注解的一个特化形式，主要用于标识那些包含业务逻辑的服务类。通过使用 `@Service` 注解，Spring 容器可以自动检测并管理这些服务类，从而实现依赖注入和其他 Spring 功能。

### 2.11 system

包含系统级别的组件，比如系统配置、工具方法等，用于检测使用的GPU使用率和内存等信息。

### 2.12 util

工具类一般放在这个包中，提供一些通用的功能，如字符串处理、日期格式化、加密解密等。

### 2.13 websocket

主要作用是管理 WebSocket 连接，允许服务器与客户端进行实时通信。通过使用 Redis，可以在分布式环境中管理 WebSocket 连接的状态。

### 2.14 Application.java 

这是一个 Spring Boot 应用程序的主类。

```java
@SpringBootApplication
@MapperScan("org.example.mapper")
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean
    public RequestContextListener requestContextListener() {
        return new RequestContextListener();
    }
}
```

* `@SpringBootApplication`：这个注解表示这是一个 Spring Boot 应用程序。
* `@MapperScan("org.example.mapper")`：这个注解用于指定 MyBatis 的 Mapper 接口所在的包，Spring Boot 会自动扫描这个包下的接口，并创建对应的代理对象。这些接口用于操作数据库。
* `public static void main(String[] args)`：这是 Java 程序的入口点。在这个方法中，调用了 `SpringApplication.run(Application.class, args)` 来启动 Spring Boot 应用程序。
* `@Bean public RequestContextListener requestContextListener()`：这个方法定义了一个 Bean，Bean 的类型是 `RequestContextListener`.`RequestContextListener` 是一个 Servlet 监听器，作用是在每个 HTTP 请求开始和结束时，将请求的信息（如请求参数、请求头、Session 等）绑定和解绑到当前线程，这样在处理请求的过程中，可以在任何地方获取到当前请求的信息。

## 3 调用流程

![](3.png)