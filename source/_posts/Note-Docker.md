---
title: 「学」 Docker
category_bar: true
date: 2023-02-27 22:36:34
tags:
categories: 学习笔记
banner_img:
---

Docker 容器将软件以及它运行安装所需的一切文件（代码、运行时、系统工具、系统库）打包到一起，这就保证了不管是在什么样的运行环境，总是能以相同的方式运行。就好像 Java 虚拟机一样，“一次编写，到处运行（Write once, run anywhere）”，而 Docker 是 “一次构建，到处运行（Build once，run anywhere）”。避免了“代码在我这里可以运行啊”的尴尬局面。

<!-- more -->

## Docker 架构

* **镜像（Image）**：相当于是一个 root 文件系统。Docker 镜像是用于创建 Docker 容器的模板，比如 Ubuntu 系统。

* **容器（Container）**：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

* **仓库（Repository）**：仓库可看成一个代码控制中心，用来保存镜像。

* **客户端（Client）**：Docker 客户端通过命令行或者其他工具使用 [Docker SDK](https://docs.docker.com/develop/sdk/) 与 Docker 的守护进程通信。

* **主机（Host）**：一个物理或者虚拟的机器用于执行 Docker 守护进程和容器。

Docker 使用客户端-服务器 (C/S) 架构模式，使用远程 API 来管理和创建 Docker 容器。

![Docker 架构](1.png)

在没有使用 Docker 时，我们开发完毕一个项目，需要打成 war 包或 jar 包。然后，在服务器上进行各种环境的安装、配置以及应用程序维护，比如：JDK、Tomcat、数据库等。而且，上述的配置在开发环境、测试服务器、生产服务器（通常会有很多个），都需要进行一遍同样的操作，工作量相当繁重。

在使用了 Docker 之后，我们可以自己创建一个空的镜像从头构建，也可以使用公共仓库中已经构建好的镜像，直接使用。当需要在不同环境中进行部署时，直接使用构建好的镜像即可。

![Docker 优势](3.png)

## Docker 容器与虚拟机

虚拟机是通过软件模拟的具有完整硬件系统功能的、运行在一个完全隔离环境中的完整计算机系统。创建虚拟机时，会将实体机的部分硬盘和内存容量作为虚拟机的硬盘和内存，每个虚拟机都有独立的硬盘和操作系统，可以像使用实体机一样对虚拟机进行操作。虚拟机会消耗大量系统资源和开销，尤其是当多个虚拟机在同一物理服务器上运行时，每个虚拟机都有自己的子操作系统，大量精力以及资源被用于虚拟化的部署和运行上。

容器类似于虚拟机，只是容器不是完整的操作系统，容器通常只包含必要的操作系统包和应用程序，这就是它们轻量级的原因。

![容器与虚拟机](2.png)

传统的虚拟机是在宿主机之上，又添加了一个新的操作系统，这就导致了虚拟机的臃肿，不适合迁移。而 Docker 是直接寄存在宿主机上，完全就会避免大部分虚拟机带来的困扰。

Docker 是一个黑盒的进程，区别于传统的进程，Docker 可以独立出一个自己的空间，不会使得在 Docker 中的行为以及变量溢出到宿主机上。

## Docker 镜像加速

国内从 DockerHub 拉取镜像有时会遇到困难，此时可以配置镜像加速器。Docker 官方和国内很多云服务商都提供了国内加速器服务：

* 科大镜像：https://docker.mirrors.ustc.edu.cn/
* 网易：https://hub-mirror.c.163.com/
* 阿里云：https://<你的ID>.mirror.aliyuncs.com（[阿里云镜像获取地址](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)）
* 七牛云加速器：https://reg-mirror.qiniu.com

对于使用 systemd 的系统，请在 `/etc/docker/daemon.json` 中写入如下内容（如果文件不存在请新建该文件）：
```linux
{"registry-mirrors":["https://reg-mirror.qiniu.com/"]}
```
之后重新启动服务：
```linux
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```
检查是否生效：
```linux
$ docker info
Registry Mirrors:
    https://reg-mirror.qiniu.com
```

## Docker 常用命令

1. 当前系统 Docker 信息
    ```linux
    $ docker
    $ docker info
    $ docker version
    ```

2. 容器使用

    * **获取镜像**
        如果我们本地没有 ubuntu 镜像，我们可以使用 docker pull 命令来载入 ubuntu 镜像：
        ```linux
        $ docker pull ubuntu
        ```

    * **启动容器**
        以下命令使用 ubuntu 镜像启动一个容器，参数为以命令行模式进入该容器：
        ```linux
        $ docker run -it ubuntu /bin/bash

        参数说明：
        -i: 交互式操作。
        -t: 终端。
        ubuntu: ubuntu 镜像。
        /bin/bash：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是 /bin/bash。

        退出终端：
        # exit
        ```

    * **查看所有容器**
        ```linux
        $ docker ps -a
        ```

        查看当前有哪些容器正在运行
        ```linux
        $ docker ps
        ```

    * **启动、停止、重启容器**
        ```linux
        $ docker start container_id
        $ docker stop container_id
        $ docker restart container_id
        ```

    * **后台运行**
        ```linux
        $ docker run -itd --name ubuntu-test ubuntu /bin/bash
        ```

    * **进入容器**
        ```linux
        $ docker attach container_id
        如果从这个容器退出，会导致容器的停止

        $ docker exec -it container_id /bin/bash
        如果从这个容器退出，容器不会停止，推荐大家使用 docker exec
        ```

    * **删除容器**
        ```linux
        $ docker rm -f container_id
        ```

3. 镜像使用

   * **查看宿主机上的镜像**
        Docker 镜像保存在 / var/lib/docker 目录下
        ```linux
        $ docker images
        ```

   *  **拉取新的镜像**
        ```linux
        $ docker pull image_name:tag
        ```

   * **删除镜像**
        ```linux
        $ docker rmi hello-world
        ```

4. [更多命令](https://www.runoob.com/docker/docker-hello-world.html)

## 个人疑问

**Linux docker 宿主机是否可以运行 Windows 容器？**

* Windows docker 宿主机可以运行 Windows 和 Linux 容器。
* Linux docker 宿主机只能运行 Linux 容器。
* Windows 宿主机可以运行 Linux 容器的原因是： Windows 在后台创建了一个 Linux 子系统，因此 Linux 容器仍在 Linux 上运行。