---
title: 「学」 使用 Tape 测试 Fabric 2.4.0
category_bar: true
date: 2023-04-17 10:53:08
tags:
categories: 学习笔记
banner_img:
---

fabric：<https://github.com/hyperledger/fabric>.
tape：<https://github.com/Hyperledger-TWGC/tape>.
Caliper：<https://hyperledger.github.io/caliper/v0.4.2/installing-caliper/>

<!-- more -->

## 搭建 Fabric 网络

1. 创建项目目录
    ```Linux
    $ mkdir fabric
    $ cd fabric
    ```

2. 拉取 fabric 项目
    ```Linux
    $ git clone https://github.com/hyperledger/fabric.git
    $ cd fabric
    $ git checkout v2.4.0
    ```

3. 拉取 fabric 镜像
    ```Linux
    $ cd fabric/fabric/scripts
    ```

    **此处我们需要修改当前目录下的 bootstrap.sh 脚本**

    此脚本先会拉取 fabric-samples，再拉取环境所用的二进制文件，但国内网络是无法访问的，从而导致后面的操作失败，所以我们选择手动拉取 fabric-samples 再切换到 v2.4.0 分支。此脚本仅仅作为拉取镜像的操作。

    修改：
    ```bootstrap.sh
    DOCKER=true
    SAMPLES=false
    BINARIES=false
    ```

    运行脚本
    ```Linux
    $ ./bootstrap.sh
    ```

4. 拉取 fabric-samples
    ```Linux
    $ cd fabric
    $ git clone https://github.com/hyperledger/fabric-samples.git
    $ git checkout v2.4.0
    ```

5. 下载需要的二进制文件
    ```Linux
    $ cd fabric/fabric-samples
    $ wget https://github.com/hyperledger/fabric/releases/download/v2.4.0/hyperledger-fabric-linux-amd64-2.4.0.tar.gz
    $ tar -xzvf hyperledger-fabric-linux-amd64-2.4.0.tar.gz
    $ wget https://github.com/hyperledger/fabric-ca/releases/download/v1.5.6/hyperledger-fabric-ca-linux-amd64-1.5.6.tar.gz
    $ tar -xzvf hyperledger-fabric-ca-linux-amd64-1.5.6.tar.gz
    ```

6. 启动/关闭网络
    ```Linux
    $ cd fabric/fabric-samples/test-network
    $ ./network.sh up
    ```
    成功启动一个 orderer 节点 和两个 peer 节点。

    关闭网络
    ```Linux
    $ ./network.sh down
    ```
    该命令将停止并删除节点和链码容器，删除组织加密材料，并从 Docker Registry 移除链码镜像，另外还会删除之前运行的通道项目。


## 使用 Tape 测试

1. 启动 Fabric 网络
    ```Linux
    $ cd fabric/fabric-samples/test-network
    $ ./network.sh up createChannel -s couchdb
    ```

2. 安装默认链码
    ```Linux
    $ ./network.sh deployCC -ccn basic -ccp ../asset-transfer-basic/chaincode-go -ccl go
    ```

3. 克隆官方 tape 仓库
   ```Linux
   $ git clone https://github.com/Hyperledger-TWGC/tape
   ```

4. 将 fabric-samples/test-network 的网络生成的证书文件夹（organizations）复制到 tape 文件内

5. 将 tape/config.yaml 文件配置信息替换为如下配置信息：
    ```config.yaml
    # Definition of nodes
    peer1: &peer1
        addr: localhost:7051
        ssl_target_name_override: peer0.org1.example.com
        org: org1
        tls_ca_cert: /config/organizations/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp/tlscacerts/tlsca.org1.example.com-cert.pem
    
    peer2: &peer2
        addr: localhost:9051
        ssl_target_name_override: peer0.org2.example.com
        org: org2
        tls_ca_cert: /config/organizations/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp/tlscacerts/tlsca.org2.example.com-cert.pem
    
    orderer1: &orderer1
        addr: localhost:7050
        ssl_target_name_override: orderer.example.com
        org: org1
        tls_ca_cert: /config/organizations/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem
    
    policyFile: /config/test/andLogic.rego
    
    # Nodes to interact with
    endorsers:
        - *peer1
    # we might support multi-committer in the future for more complex test scenario,
    # i.e. consider tx committed only if it's done on >50% of nodes. But for now,
    # it seems sufficient to support single committer.
    committers: 
        - *peer1
        - *peer2
    
    commitThreshold: 1
    
    orderer: *orderer1
    
    # Invocation configs
    channel: mychannel
    chaincode: basic
    args:
        - GetAllAssets
    mspid: Org1MSP
    private_key: /config/organizations/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp/keystore/priv_sk
    sign_cert: /config/organizations/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp/signcerts/User1@org1.example.com-cert.pem
    num_of_conn: 10
    client_per_conn: 10
    ```

6. 在 tape 文件夹下命令行启动执行
    ```Linux
    $ docker run --network=host -v $PWD:/config guoger/tape tape -c /config/config.yaml -n 50000
    ```

    ![result](1.png)

## 使用 Caliper 测试

Caliper 测试起来感觉比较复杂，不如 Tape 轻量，之前也一直报错，但是最近试了一下又成功了，记录一下。

1. 安装 Caliper
    首先 caliper-benchmarks 需要下载到指定文件路径位置。因为阅读了 caliper-benchmarks 的相关配置文件后，发现它 yaml 文件对 test-network 里的文件是采用相对路径定位的，需要将其下载到 fabric-sample 的上一级目录的位置才能正确执行。当然也可以随便下载到一个位置，然后修改 caliper-benchmarks 的配置内容。
    ![位置](2.png)
    ```Linux
    $ git clone https://github.com/hyperledger/caliper-benchmarks
    $ cd caliper-benchmarks
    $ npm init -y
    $ npm install --only=prod @hyperledger/caliper-cli@0.4.2  # 0.4.2 对应的是 fabric2.x，0.3.2 对应 fabric1.x
    $ npx caliper bind --caliper-bind-sut fabric:2.2  # 绑定，这里用的 2.2，用 2.4 失败了

    $ npx caliper --version  #查看所安装的 caliper 版本
    ```

2. 部署链码
    ```Linux
    $ cd ../fabric-samples/test-network  # 回到 test-network 文件夹
    $ ./network.sh down
    $ ./network.sh up createChannel

    # 部署 caliper 自带的 fabcar 样例，以供展开测试
    $ ./network.sh deployCC -ccn fabcar -ccp ../../caliper-benchmarks/src/fabric/samples/fabcar/go -ccl go
    ```

3. 进行测试
    ```Linux
    $ cd ../../caliper-benchmarks/

    $ npx caliper launch manager \
        --caliper-workspace ./ \
        --caliper-networkconfig networks/fabric/test-network.yaml \
        --caliper-benchconfig benchmarks/samples/fabric/fabcar/config.yaml \
        --caliper-flow-only-test \
        --caliper-fabric-gateway-enabled
    ```

    ![result](3.png)