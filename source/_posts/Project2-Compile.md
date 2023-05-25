---
title: 「毕」 2. 编译 Fabric 源码
category_bar: true
date: 2023-05-09 15:09:12
tags:
categories: 毕设项目
banner_img:
---

本文尝试编译 Fabric 源码来构建网络。

<!-- more -->

## bootstrap.sh

在使用 Tape 测试 Fabric 网络那一篇文章中，我直接执行了 bootstrap.sh 这个脚本一键部署。使用这个方法根本不需要拉取 Fabric 的源代码，它做了三件事情：

1. 从 github 上克隆 hyperledger/fabric-samples 并进入该目录，然后检出适当的版本
2. 在 fabric-samples 目录下安装特定平台的 Hyperledger Fabric 二进制可执行文件和配置文件
3. 下载指定版本的 Hyperledger Fabric 的 docker 镜像

其中第二步是执行 curl 去下载 tar 包并解压，二进制文件会被放在 bin 目录下：

|名称|作用|
|----|----|
|peer|负责启动节点，存储区块链数据，运行维护链码|
|orderer|负责启动排序节点，对交易进行排序，并将排序好的交易打包成模块|
|cryptogen|生成组织结构和身份文件|
|configtxgen|生成配置区块和配置交易|
|configtxlator|解读配置信息|
|fabric-ca-client|fabric-ca 客户端命令|
|discover|fabric 发现服务的客户端命令|
|idemixgen|身份混合机制|

部分镜像说明如下：

|镜像名称|是否可选|镜像说明|
|----|----|----|
|hyperledger/fabric-tools|可选|包含 crytogen、configtxgen、configtxlator 工具的镜像文件|
|hyperledger/fabric-couchdb|可选|CouchDB 的数据库镜像文件、状态数据库选择 CouchDB 的时候才需要|
|hyperledger/fabric-kafka|可选|Kafka 的镜像文件|
|hyperledger/fabric-zookeeper|可选|Zookeeper 的镜像文件|
|hyperledger/fabric-peer|必选|Peer 节点的镜像文件|
|hyperledger/fabric-orderer|必选|排序服务节点的镜像文件|
|hyperledger/fabric-javaenv|可选|Java 链码的基础镜像文件|
|hyperledger/fabric-ccenv|必选|Golang 链码的基础镜像文件|
|hyperledger/fabric-ca|可选|fabric-ca 的镜像文件，用到 fabric-ca 的时候才需要|

## 编译源码

接下来尝试一下从 Fabric 源码入手，来搭建网络。

### 1. 下载 fabric 源码

```Linux
$ git clone https://github.com/hyperledger/fabric.git
$ cd fabric
$ git checkout v2.4.0
```

### 2. 编译 fabric 源码

```Linux
$ make release
```

编译好的二进制可执行文件在 `fabric/release/linux-amd64/bin/` 下：

![二进制文件](1.png)

#### 测试编译

我注意到网上的教程都需要我们把 fabric 的源码放在 `$GOPATH/src/github.com/heperledger/` 目录下，Makefile 文件中的编译路径也是：

```Makefile
PKGNAME = github.com/hyperledger/fabric

pkgmap.configtxgen    := $(PKGNAME)/cmd/configtxgen
pkgmap.configtxlator  := $(PKGNAME)/cmd/configtxlator
pkgmap.cryptogen      := $(PKGNAME)/cmd/cryptogen
pkgmap.discover       := $(PKGNAME)/cmd/discover
pkgmap.ledgerutil     := $(PKGNAME)/cmd/ledgerutil
pkgmap.orderer        := $(PKGNAME)/cmd/orderer
pkgmap.osnadmin       := $(PKGNAME)/cmd/osnadmin
pkgmap.peer           := $(PKGNAME)/cmd/peer
```

为了测试能否在自定义路径中编译源码，我修改了 `fabric/internal/peer/version` :

![version.go](3.png)

然后编译并执行：

```Linux
$ make peer
$ export PATH=$PWD/build/bin
$ peer
```

![输出](4.png)

### 3. 下载 fabric-ca 源码

```Linux
$ git clone https://github.com/hyperledger/fabric-ca.git
$ cd fabric-ca
```

### 4. 编译 fabric-ca 源码

```Linux
$ make fabric-ca-server
$ make fabric-ca-client
```

编译好的二进制可执行文件在 `fabric-ca/bin/` 下：

![二进制文件](2.png)

## 制作镜像

把当前用户加入到 docker 用户组后，分别在 fabric 和 fabric-ca 目录下执行：

```Linux
$ make docker
```

![Docker Images](6.png)

### 镜像代码

`Makefile` 中制作 docker 镜像的代码如下：

```Makefile
.PHONY: docker
docker: $(RELEASE_IMAGES:%=%-docker)

.PHONY: $(RELEASE_IMAGES:%=%-docker)
$(RELEASE_IMAGES:%=%-docker): %-docker: $(BUILD_DIR)/images/%/$(DUMMY)

$(BUILD_DIR)/images/baseos/$(DUMMY):  BUILD_CONTEXT=images/baseos
$(BUILD_DIR)/images/ccenv/$(DUMMY):   BUILD_CONTEXT=images/ccenv
$(BUILD_DIR)/images/peer/$(DUMMY):    BUILD_ARGS=--build-arg GO_TAGS=${GO_TAGS}
$(BUILD_DIR)/images/orderer/$(DUMMY): BUILD_ARGS=--build-arg GO_TAGS=${GO_TAGS}
$(BUILD_DIR)/images/tools/$(DUMMY):   BUILD_ARGS=--build-arg GO_TAGS=${GO_TAGS}

$(BUILD_DIR)/images/%/$(DUMMY):
	@echo "Building Docker image $(DOCKER_NS)/fabric-$*"
	@mkdir -p $(@D)
	$(DBUILD) -f images/$*/Dockerfile \
		--build-arg GO_VER=$(GO_VER) \
		--build-arg ALPINE_VER=$(ALPINE_VER) \
		$(BUILD_ARGS) \
		-t $(DOCKER_NS)/fabric-$* ./$(BUILD_CONTEXT)
	docker tag $(DOCKER_NS)/fabric-$* $(DOCKER_NS)/fabric-$*:$(BASE_VERSION)
	docker tag $(DOCKER_NS)/fabric-$* $(DOCKER_NS)/fabric-$*:$(TWO_DIGIT_VERSION)
	docker tag $(DOCKER_NS)/fabric-$* $(DOCKER_NS)/fabric-$*:$(DOCKER_TAG)
	@touch $@
```

其中 `$(DBUILD) -f` 即 `docker build -f` 指明了 `Dockerfile` 的路径。以 `fabric/images/peer/Dockerfile` 为例：

```Dockerfile
FROM alpine:${ALPINE_VER} as peer-base
RUN apk add --no-cache tzdata
# set up nsswitch.conf for Go's "netgo" implementation
# - https://github.com/golang/go/blob/go1.9.1/src/net/conf.go#L194-L275
# - docker run --rm debian:stretch grep '^hosts:' /etc/nsswitch.conf
RUN echo 'hosts: files dns' > /etc/nsswitch.conf

FROM golang:${GO_VER}-alpine${ALPINE_VER} as golang
RUN apk add --no-cache \
	bash \
	binutils-gold \
	gcc \
	git \
	make \
	musl-dev
ADD . $GOPATH/src/github.com/hyperledger/fabric
WORKDIR $GOPATH/src/github.com/hyperledger/fabric

FROM golang as peer
ARG GO_TAGS
RUN make peer GO_TAGS=${GO_TAGS}

FROM peer-base
ENV FABRIC_CFG_PATH /etc/hyperledger/fabric
VOLUME /etc/hyperledger/fabric
VOLUME /var/hyperledger
COPY --from=peer /go/src/github.com/hyperledger/fabric/build/bin /usr/local/bin
COPY --from=peer /go/src/github.com/hyperledger/fabric/sampleconfig/msp ${FABRIC_CFG_PATH}/msp
COPY --from=peer /go/src/github.com/hyperledger/fabric/sampleconfig/core.yaml ${FABRIC_CFG_PATH}
EXPOSE 7051
CMD ["peer","node","start"]
```

这是一个分阶段构建镜像的过程，每个阶段生成一个镜像，下一个阶段在此基础上生成：

1. **peer-base** 阶段：
    * 基于指定的 Alpine 版本构建基础镜像。
    * 安装 tzdata 包，用于时区设置。
    * 配置 nsswitch.conf 文件，以支持 Go 的 "netgo" 实现。

2. **golang** 阶段：
    * 基于指定的 Go 版本和 Alpine 版本构建 Golang 镜像。
    * 安装构建所需的工具和依赖，如 bash、binutils-gold、gcc、git、make 和 musl-dev。
    * 将**当前目录下**的文件复制到 Golang 镜像的对应路径 `$GOPATH/src/github.com/hyperledger/fabric`。

3. **peer** 阶段：
    * 基于 golang 阶段构建一个临时的 Golang 镜像。
    * 使用 make 命令构建 Peer 节点的二进制文件。
    * 使用 GO_TAGS 参数指定构建所需的 Go 标签。

4. **最终**阶段：
    * 基于 peer-base 阶段构建最终的镜像。
    * 设置环境变量 FABRIC_CFG_PATH 为 /etc/hyperledger/fabric。
    * 创建两个数据卷，分别用于存储 Fabric 配置文件和 Hyperledger 数据。
    * 从 peer 阶段复制生成的二进制文件到 /usr/local/bin 目录下。
    * 从 peer 阶段复制生成的 MSP 配置到 /etc/hyperledger/fabric/msp 目录下。
    * 从 peer 阶段复制生成的核心配置文件 core.yaml 到 /etc/hyperledger/fabric 目录下。
    * 暴露容器的 7051 端口。
    * 使用 CMD 指令在容器启动时运行 peer node start 命令。

**可以看出该镜像并非直接复制主机编译的二进制文件，而是复制当前目录的代码，在容器中自己编译。**

### 测试镜像

保留之前测试编译的更改，制作 peer 镜像并进入容器测试：

```Linux
$ make peer-docker
$ docker run hyperledger/fabric-peer
$ docker exec -it container_id /bin/sh
# peer
```

![输出](5.png)

## 启动网络

```Linux
$ cd fabric-samples
```

将之前编译的二进制文件全部放在 `fabric-samples/bin` 下：

![bin](7.png)

此时还需要生成节点的配置文件，将 `fabric/sampleconfig` 复制到 fabric-samples，并改名 config。

启动：

```Linux
$ cd test-network
$ ./network.sh up
```

成功启动！但是在创建通道的时候组织二的节点无法加入，将 peer 的二进制文件更换为官方提供直接下载的则没有这个问题，初步推测下载的那个版本是在编译时为了适应 fabric-sample 做了某些修改。