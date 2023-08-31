---
title: 「学」 Docker
category_bar: true
date: 2023-02-27 22:36:34
tags:
categories: 学习笔记
banner_img:
---

Docker 容器将软件以及它运行安装所需的一切文件（代码、运行时库、系统工具、系统库）打包到一起，这就保证了不管是在什么样的运行环境，总是能以相同的方式运行。就好像 Java 虚拟机一样，“一次编写，到处运行（Write once, run anywhere）”，而 Docker 是 “一次构建，到处运行（Build once，run anywhere）”。

<!-- more -->

## 1 Docker 基本概念

在没有使用 Docker 时，我们开发完毕一个项目，需要打成 war 包或 jar 包。然后，在服务器上进行各种环境的安装、配置以及应用程序维护，比如：JDK、Tomcat、数据库等。而且，上述的配置在开发环境、测试服务器、生产服务器（通常会有很多个），都需要进行一遍同样的操作，工作量相当繁重。

在使用了 Docker 之后，我们可以自己创建一个空的镜像从头构建，也可以使用公共仓库中已经构建好的镜像，直接使用。当需要在不同环境中进行部署时，直接使用构建好的镜像即可。

![Docker 优势](3.png)

### 1.1 基本组成

![Docker 架构](1.png)

* **镜像（Image）**：相当于是一个 root 文件系统。Docker 镜像是用于创建 Docker 容器的模板，比如 Ubuntu 系统。

* **容器（Container）**：镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的类和实例一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

* **仓库（Repository）**：仓库可看成一个代码控制中心，用来保存镜像。

* **客户端（Client）**：Docker 客户端通过命令行或者其他工具使用 [Docker SDK](https://docs.docker.com/develop/sdk/) 与 Docker 的守护进程通信。

* **主机（Host）**：一个物理或者虚拟的机器用于执行 Docker 守护进程和容器。

Docker 使用客户端-服务器 (C/S) 架构模式，使用远程 API 来管理和创建 Docker 容器。

### 1.2 Docker-Compose

通过 Docker-Compose ，不需要使用 shell 脚本来启动容器，而使用 YAML 文件来配置应用程序需要的所有服务，然后使用一个命令根据 YAML 的文件配置创建并启动所有服务。

### 1.3 网络

当你安装 Docker 时，它会自动创建三个网络：

* bridge：创建容器默认连接到此网络，此模式会为每一个容器分配、设置 IP 等，并将容器连接到一个 docker0 虚拟网桥，通过 docker0 网桥以及 Iptables nat 表配置与宿主机通信。
* none：该模式关闭了容器的网络功能。
* host：容器和宿主机共享 Network namespace。容器将不会虚拟出自己的网卡，配置自己的 IP 等，而是使用宿主机的 IP 和端口。

建议使用自定义的网桥来控制哪些容器可以相互通信，还可以自动 DNS 解析容器名称到 IP 地址。

## 2 Docker 容器与虚拟机

创建虚拟机时，会将实体机的部分硬盘和内存容量作为虚拟机的硬盘和内存。而 Docker 有着比虚拟机更少的抽象层，不需要实现硬件资源虚拟化，运行在 Docker 容器上的程序直接使用实际物理机的硬件资源。Docker 利用的是宿主机的内核，而不需要 Guest OS。

![容器与虚拟机](2.png)

传统的虚拟机是在宿主机之上，又添加了一个新的操作系统，这就导致了虚拟机的臃肿，不适合迁移。而 Docker 是直接寄存在宿主机上，完全就会避免大部分虚拟机带来的困扰。

Docker 是一个黑盒的进程，区别于传统的进程，Docker 可以独立出一个自己的空间，不会使得在 Docker 中的行为以及变量溢出到宿主机上。

## 3 Docker 镜像

镜像是一种轻量级、可执行的独立软件包，用来打包软件运行环境和基于运行环境开发的软件，包含运行某个软件所需的所有内容，包括**代码、运行时库、环境变量和配置文件**。

### 3.1 构建镜像

所有的 Docker 镜像都起始于一个基础镜像层，当进行修改或培加新的内容时，就会在当前镜像层之上，创建新的镜像层。

举一个简单的例子，假如基于 Ubuntu Linux 16.04 创建一个新的镜像，这就是新镜像的第一层；如果在该镜像中添加 Python 包，就会在基础镜像层之上创建第二个镜像层；如果继续添加一个安全补丁，就会创健第三个镜像层如下图所示：

![](5.png)

Docker 镜像都是只读的，当容器启动时，一个新的可写层加载到镜像的顶部。这一层就是我们通常说的容器层，容器之下的都叫镜像层。

自己构建镜像时首先需要写一个 dockerfile 文件：

1. 编写一个 dockerfile 文件
2. docker build 构建称为一个镜像
3. docker run 运行镜像

```dockerfile
FROM                # 基础镜像，一切从这里开始构建
MAINTAINER          # 镜像是谁写的， 姓名+邮箱
RUN                 # 镜像构建的时候需要运行的命令
ADD                 # 添加内容
WORKDIR             # 镜像的工作目录
VOLUME              # 挂载的目录
EXPOSE              # 保留端口配置
CMD                 # 指定这个容器启动的时候要运行的命令，只有最后一个会生效，可被替代。
ENTRYPOINT          # 指定这个容器启动的时候要运行的命令，可以追加命令
ONBUILD             # 当构建一个被继承 dockerfile 这个时候就会运行 ONBUILD 的指令，触发指令。
COPY                # 类似 ADD，将我们文件拷贝到镜像中
ENV                 # 构建的时候设置环境变量
```

### 3.2 镜像加速

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

## 4 Docker 常用命令

![](4.png)

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
        ```
        参数说明：
        -i: 交互式操作。
        -t: 终端。
        ubuntu: ubuntu 镜像。
        /bin/bash：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是 /bin/bash。

        退出终端：
        ```Linux
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
        $ docker kill container_id
        ```

    * **后台运行**
        ```linux
        $ docker run -itd --name ubuntu-test ubuntu /bin/bash
        ```

    * **进入容器**
        ```linux
        $ docker attach container_id
        # 进入容器正在执行的终端，如果从这个容器退出，会导致容器的停止

        $ docker exec -it container_id /bin/bash
        # 进入当前容器后开启一个新的终端，如果从这个容器退出，容器不会停止
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

## 5 个人疑问

**Linux docker 宿主机是否可以运行 Windows 容器？**

* Windows docker 宿主机可以运行 Windows 和 Linux 容器。
* Linux docker 宿主机只能运行 Linux 容器。
* Windows 宿主机可以运行 Linux 容器的原因是： Windows 在后台创建了一个 Linux 子系统，因此 Linux 容器仍在 Linux 上运行。