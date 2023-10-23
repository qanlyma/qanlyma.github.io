---
title: 「毕」 3. 部署自定义网络
category_bar: true
date: 2023-05-23 17:32:58
tags:
categories: 毕设项目
banner_img:
---

手动来部署一个自定义的 Fabric 测试网络。

<!-- more -->

## 1 准备工作

准备好编译/下载的二进制文件和镜像，二进制命令复制到 `/usr/local/bin` 中，以供全局调用：

```Lunux
$ cp * /usr/local/bin
```

也可以将二进制文件加入子进程的临时 PATH 中：

```Lunux
$ export PATH=$PWD/bin:/bin:/usr/bin
```

创建一个目录，用于存放各配置文件:

```Lunux
$ mkdir twoPeerNet
$ cd twoPeerNet
```

## 2 生成配置文件

注意在运行配置文件时报错大概率是因为有 Tab 或者缩进不正确。

### 组织配置文件 crypto-config.yaml

```Linux
$ cryptogen showtemplate > crypto-config.yaml
```

文件内容：
```crypto-config.yaml
OrdererOrgs:
  - Name: Orderer                             # orderer 组织的名称
    Domain: example.com                       # orderer 组织的根域名
    EnableNodeOUs: true                       # 是否使用组织单元
    Specs:
      - Hostname: orderer                     # 可以通过 hostname 设置多个 orderer 节点
                                              # Hostname + Domain 组成该 orderer 节点的完整域名

PeerOrgs:                                     # 一个 PeerOrgs 设置多个 peer 组织
  - Name: Org1                                # peer 组织的名称
    Domain: org1.example.com                  # peer 组织的域名
    EnableNodeOUs: true		
    Template:                                 # 节点的数量（peer0, peer1, .....）
      Count: 1
    Users:                                    # 用户的数量
      Count: 1

  - Name: Org2
    Domain: org2.example.com
    EnableNodeOUs: true
    Template:
      Count: 1
    Users:
      Count: 1
```

### 生成密钥

```Linux
$ cryptogen generate --config=crypto-config.yaml
```

### 网络配置文件 configtx.yaml

```configtx.yaml
Organizations:
    - &OrdererOrg
        Name: OrdererOrg
        ID: OrdererMSP
        MSPDir: ./crypto-config/ordererOrganizations/example.com/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('OrdererMSP.member')"
            Writers:
                Type: Signature
                Rule: "OR('OrdererMSP.member')"
            Admins:
                Type: Signature
                Rule: "OR('OrdererMSP.admin')"
        OrdererEndpoints:
            - orderer.example.com:7050

    - &Org1
        Name: Org1MSP
        ID: Org1MSP
        MSPDir: ./crypto-config/peerOrganizations/org1.example.com/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.peer', 'Org1MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org1MSP.admin')"
            Endorsement:
                Type: Signature
                Rule: "OR('Org1MSP.peer')"

        AnchorPeers:
            - Host: peer0.org1.example.com
              Port: 7051

    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: ./crypto-config/peerOrganizations/org2.example.com/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org2MSP.admin', 'Org2MSP.peer', 'Org2MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org2MSP.admin', 'Org2MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org2MSP.admin')"
            Endorsement:
                Type: Signature
                Rule: "OR('Org2MSP.peer')"

        AnchorPeers:
            - Host: peer0.org2.example.com
              Port: 8051


Capabilities:
    Channel: &ChannelCapabilities
        V2_0: true
    Orderer: &OrdererCapabilities
        V2_0: true
    Application: &ApplicationCapabilities
        V2_0: true

Application: &ApplicationDefaults

    Organizations:

    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        LifecycleEndorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
        Endorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"

    Capabilities:
        <<: *ApplicationCapabilities

Orderer: &OrdererDefaults

    OrdererType: etcdraft

    Addresses:                            # orderer 集群节点
        - orderer.example.com:7050

    EtcdRaft:
        Consenters:
        - Host: orderer.example.com
          Port: 7050
          ClientTLSCert: ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
          ServerTLSCert: ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt

    BatchTimeout: 2s                      # 生成区块超时时间	

    # Batch Size: Controls the number of messages batched into a block
    BatchSize:

        MaxMessageCount: 10               # 区块的消息数量

        AbsoluteMaxBytes: 99 MB           # 区块最大字节数

        PreferredMaxBytes: 512 KB         # 建议消息字节数

    Organizations:

    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        BlockValidation:
            Type: ImplicitMeta
            Rule: "ANY Writers"

Channel: &ChannelDefaults

    Policies:
        # Who may invoke the 'Deliver' API
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        # Who may invoke the 'Broadcast' API
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        # By default, who may modify elements at this config level
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"

    Capabilities:
        <<: *ChannelCapabilities

Profiles:
    TwoOrgsChannel:
        Consortium: SampleConsortium
        <<: *ChannelDefaults
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Org1
                - *Org2
            Capabilities:
                <<: *ApplicationCapabilities
    
    TwoOrgsOrdererGenesis:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2
```

### 创世块文件

channelID 指定的是系统通道名。

```Linux
$ configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block -channelID test-channel
```

### 通道文件

channelID 指定的是应用通道名，不能与上一步同名。

```Linux
$ configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel
```

### org1 和 org2 的锚节点

```Linux
$ configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP

$ configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP
```

## 3 启动节点

### 编写 docker-compose 文件

cli 容器通过环境变量 CORE_PEER_ADDRESS 来指定所代表的 Peer 节点。

```docker-compose.yaml
version: '2'

volumes:
  orderer.example.com:
  peer0.org1.example.com:
  peer0.org2.example.com:
  
networks:
  test:
    name: fabric_test

services:
  orderer.example.com:
    container_name: orderer.example.com
    image: hyperledger/fabric-orderer:latest
    environment:
      - FABRIC_LOGGING_SPEC=INFO
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0		#监听的 IP
      - ORDERER_GENERAL_LISTENPORT=7050			#监听的端口
      - ORDERER_GENERAL_GENESISMETHOD=file        #创世块的来源方式
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block  #创世文件的路径
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP		#MSPID
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp	#容器中 msp 的路径
      - ORDERER_OPERATIONS_LISTENADDRESS=0.0.0.0:17050
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
      - ORDERER_GENERAL_CLUSTER_CLIENTCERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_CLUSTER_CLIENTPRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_CLUSTER_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
      - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
      - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp:/var/hyperledger/orderer/msp
      - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/:/var/hyperledger/orderer/tls
      - orderer.example.com:/var/hyperledger/production/orderer
    ports:
      - 7050:7050
      - 17050:17050
    networks:
      - test
  
  peer0.org1.example.com:
    container_name: peer0.org1.example.com
    image: hyperledger/fabric-peer:latest
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=fabric_test
      - CORE_PEER_ID=peer0.org1.example.com
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051		#服务的 IP 端口
      - CORE_PEER_LISTENADDRESS=0.0.0.0:7051				#本地监听的 IP 端口
      - CORE_PEER_CHAINCODEADDRESS=peer0.org1.example.com:7052	#链码的 IP 端口
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:7052		#链码监听的端口
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051		#向哪个节点发起 gossip 连接
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
      - FABRIC_LOGGING_SPEC=INFO
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
      - CORE_CHAINCODE_EXECUTETIMEOUT=300s
      - CORE_OPERATIONS_LISTENADDRESS=0.0.0.0:17051
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    volumes:
      - /var/run/:/host/var/run/
      - ./crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/etc/hyperledger/fabric/msp
      - ./crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls:/etc/hyperledger/fabric/tls
      - peer0.org1.example.com:/var/hyperledger/production
    ports:
      - 7051:7051
      - 7052:7052
      - 17051:17051
    networks:
      - test
      
  peer0.org2.example.com:
    container_name: peer0.org2.example.com
    image: hyperledger/fabric-peer:latest
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=fabric_test
      - CORE_PEER_ID=peer0.org2.example.com
      - CORE_PEER_ADDRESS=peer0.org2.example.com:8051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:8051
      - CORE_PEER_CHAINCODEADDRESS=peer0.org2.example.com:8052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:8052
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.example.com:8051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.example.com:8051
      - CORE_PEER_LOCALMSPID=Org2MSP
      - FABRIC_LOGGING_SPEC=INFO
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
      - CORE_CHAINCODE_EXECUTETIMEOUT=300s
      - CORE_OPERATIONS_LISTENADDRESS=0.0.0.0:18051
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    volumes:
      - /var/run/:/host/var/run/
      - ./crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp:/etc/hyperledger/fabric/msp
      - ./crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls:/etc/hyperledger/fabric/tls
      - peer0.org2.example.com:/var/hyperledger/production
    ports:
      - 8051:8051
      - 8052:8052
      - 18051:18051
    networks:
      - test
      
  
cli1:
    container_name: cli1
    image: hyperledger/fabric-tools:latest
    tty: true
    stdin_open: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - FABRIC_LOGGING_SPEC=INFO
      - CORE_PEER_ID=cli1
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
      - /var/run/:/host/var/run/
      - ./chaincode/go/:/opt/gopath/src/github.com/hyperledger/multiple-deployment/chaincode/go  #用于映射本地链码的路径
      - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
      - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
    depends_on:
      - peer0.org1.example.com
      - peer0.org2.example.com
      - orderer.example.com
    networks:
      - test
  
cli2:
    container_name: cli2
    image: hyperledger/fabric-tools:latest
    tty: true
    stdin_open: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - FABRIC_LOGGING_SPEC=INFO
      - CORE_PEER_ID=cli2
      - CORE_PEER_ADDRESS=peer0.org2.example.com:8051
      - CORE_PEER_LOCALMSPID=Org2MSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
      - /var/run/:/host/var/run/
      - ./chaincode/go/:/opt/gopath/src/github.com/hyperledger/multiple-deployment/chaincode/go  #映射本地链码路径
      - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
      - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
    depends_on:
      - peer0.org1.example.com
      - peer0.org2.example.com
      - orderer.example.com
    networks:
      - test 
```

### 编排容器

```Linux
$ docker-compose up -d
```

每个 cli 指向一个组织，通过环境变量的设置使 cli 指向特点组织的节点，通过操作 cli 容器就可以操作组织的节点了。

### 创建通道

进入容器 cli1 创建通道：

```Linux
$ docker exec -it cli1 bash

# peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

-o 排序节点
-c 通道名
-f 通道文件的路径

当前目录下生成 mychannel.block。在容器外将 mychannel.block 复制给 cli2：

```Linux
$ docker cp cli1:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block ./

$ docker cp ./mychannel.block cli2:/opt/gopath/src/github.com/hyperledger/fabric/peer
```

### 加入通道

开启两个终端，分别进入 cli1 和 cli2：

```Linux
$ docker exec -it cli1 bash
$ docker exec -it cli2 bash
```

分别加入通道（两个 cli 都需要操作）：

```Linux
# peer channel join -b mychannel.block
```

### 更新锚节点

在 cli1 中：

```Linux
# peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

在 cli2 中：

```Linux
# peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

## 4 链码测试

### 准备工作

利用 caliper 中的简单转账链码进行链码测试：

```Linux
$ mkdir /twoPeerNet/chaincode/go/simple
$ cp caliper-benchmarks/src/fabric/scenario/simple/go/* /twoPeerNet/chaincode/go/simple
```

在 docker-compose 中，`/twoPeerNet/chaincode/go` 与容器内的目录实现了容器卷映射，将链码复制到该目录下，容器中也有了相应的链码。

在 cli1 中设置 goproxy 代理：

```Linux
# go env -w GOPROXY=https://goproxy.cn,direct
```

下载依赖：

```Linux
# cd /opt/gopath/src/github.com/hyperledger/multiple-deployment/chaincode/go/simple
# go mod vendor

# cd /opt/gopath/src/github.com/hyperledger/fabric/peer
```

### 打包链码

在 cli1 中打包链码：

```Linux
# peer lifecycle chaincode package simple.tar.gz --path /opt/gopath/src/github.com/hyperledger/multiple-deployment/chaincode/go/simple --label simple_1.0
```

在容器外将 cli1 中打包的链码复制到 cli2：

```Linux
$ docker cp cli1:/opt/gopath/src/github.com/hyperledger/fabric/peer/simple.tar.gz ./

$ docker cp ./simple.tar.gz cli2:/opt/gopath/src/github.com/hyperledger/fabric/peer
```

### 安装链码

在两个 cli 中分别操作：

```Linux
# peer lifecycle chaincode install simple.tar.gz
```

### 审批链码

查看 package ID：

```Linux
# peer lifecycle chaincode queryinstalled
```

在两个 cli 中分别操作：

```Linux
# peer lifecycle chaincode approveformyorg --channelID mychannel --name simple --version 1.0 --init-required --package-id simple_1.0:539969bf4773133bf852368737a5ba505443ca4d6b5eae92f5f4463fd8bbd171 --sequence 1 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

检测是否成功：

```Linux
# peer lifecycle chaincode checkcommitreadiness --channelID mychannel --name simple --version 1.0 --init-required --sequence 1 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com
```

### 提交链码

只需要在 cli1 执行即可：

```Linux
# peer lifecycle chaincode commit -o orderer.example.com:7050 --channelID mychannel --name simple --version 1.0 --init-required --sequence 1 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:8051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
```

### 调用链码

初始化：

```Linux
# peer chaincode invoke -o orderer.example.com:7050 -C mychannel -n simple --isInit --ordererTLSHostnameOverride orderer.example.com --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:8051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":[]}'
```

调用其他方法：

```Linux
# peer chaincode invoke -o orderer.example.com:7050 -C mychannel -n simple --ordererTLSHostnameOverride orderer.example.com --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:8051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":["open","Qanly","1000"]}'

# peer chaincode query -C mychannel -n simple -c '{"Args":["query","Qanly"]}'
```

## 5 测试一下

使用 Caliper 测试的时候发现报错：找不到 connection-org1.yaml 这个文件，看一下 fabric-sample 网络的启动脚本发现有一个 ccp-generate.sh 脚本是用来生成这个的，把脚本和 .json .yaml 复制到 `twoPeerNet/crypto-config` 中，修改一下再运行就 ok 了。

![目录](1.png)

记得要先修改 `caliper-benchmarks/networks/fabric/test-network.yaml` 中的配置。

```Linux
$ cd caliper-benchmarks

$ npx caliper launch manager \
    --caliper-workspace ./ \
    --caliper-networkconfig networks/fabric/test-network.yaml \
    --caliper-benchconfig benchmarks/scenario/simple/config.yaml \
    --caliper-flow-only-test \
    --caliper-fabric-gateway-enabled
```

测试时我生成了 10 个账户，并用 100 的 tps 进行 500 次交易，可以发现大部分都因为读写冲突被中止了，这也正是我毕设项目希望解决的一个问题。

![结果](2.png)

## 6 查看日志

创建容器后，可以单独开一个窗口查看输出的日志：

```Linux
$ docker logs -f peer0.org1.example.com
$ docker logs -f orderer.example.com
```

## 7 清理操作

删除上述步骤生成的文件与容器。

```Linux
$ docker stop $(docker ps -a -q)
$ docker ps -a | awk '/fabric/ {print $1}' | xargs -r docker stop

$ docker rm $(docker ps -a -q)
$ docker ps -a | awk '/fabric/ {print $1}' | xargs -r docker rm -f

$ docker volume prune // 清除旧的卷
$ docker volume rm $(docker volume ls -qf 'name=fabric-*')

$ docker network rm fabric_test // 清理网络缓存

$ docker image prune -f //清理 none 镜像
$ docker rmi $(docker images | grep dev) // 清理链码镜像
$ docker rmi $(docker images -aq 'hyperledger/fabric-*')
```

## 8 脚本文件

记录一下自己第一次写脚本。

```Bash
#!/bin/bash

## Parse mode
if [[ $# -lt 1 ]] ; then
  echo "errrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrr"
  exit 0
else
  MODE=$1
  AC=$2
fi

function networkUp() {
  echo "===========================================Starting Test Network==========================================="

  echo "Generating files..."

  export PATH=$PWD/bin:/bin:/usr/bin
  cryptogen generate --config=crypto-config.yaml
  configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block -channelID test-channel
  configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel
  configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP
  configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP

  echo "Starting docker containers..."

  docker-compose up -d
  sleep 3

  echo "Creating mychannel..."

  docker exec cli1 peer channel create -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/msp/tlscacerts/tlsca.example.com-cert.pem
  docker cp cli1:/opt/gopath/src/github.com/hyperledger/fabric/peer/mychannel.block ./
  docker cp ./mychannel.block cli2:/opt/gopath/src/github.com/hyperledger/fabric/peer
  docker exec cli1 peer channel join -b mychannel.block
  docker exec cli2 peer channel join -b mychannel.block
  docker exec cli1 peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
  docker exec cli2 peer channel update -o orderer.example.com:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

  echo "Installing test chaincode..."

  docker cp ./simple.tar.gz cli1:/opt/gopath/src/github.com/hyperledger/fabric/peer
  docker cp ./simple.tar.gz cli2:/opt/gopath/src/github.com/hyperledger/fabric/peer

  docker exec cli1 peer lifecycle chaincode install simple.tar.gz
  docker exec cli2 peer lifecycle chaincode install simple.tar.gz

  docker exec cli1 peer lifecycle chaincode approveformyorg --channelID mychannel --name simple --version 1.0 --init-required --package-id simple_1.0:6bcbc248b82a10b08863dcdf3f9da8681ec95b7336f345caf75aaeafbe2d5e64 --sequence 1 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
  docker exec cli2 peer lifecycle chaincode approveformyorg --channelID mychannel --name simple --version 1.0 --init-required --package-id simple_1.0:6bcbc248b82a10b08863dcdf3f9da8681ec95b7336f345caf75aaeafbe2d5e64 --sequence 1 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
  docker exec cli1 peer lifecycle chaincode commit -o orderer.example.com:7050 --channelID mychannel --name simple --version 1.0 --init-required --sequence 1 --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:8051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt
 
  echo "Initializing the chaincode..."

  docker exec cli1 peer chaincode invoke -o orderer.example.com:7050 -C mychannel -n simple --isInit --ordererTLSHostnameOverride orderer.example.com --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem --peerAddresses peer0.org1.example.com:7051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt --peerAddresses peer0.org2.example.com:8051 --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt -c '{"Args":[]}'

  sleep 3
  createAccount

  # cd crypto-config
  # ./ccp-generate.sh
}

COMMAND="docker exec cli1 peer chaincode invoke \
    -o orderer.example.com:7050 \
    -C mychannel \
    -n simple \
    --ordererTLSHostnameOverride orderer.example.com \
    --tls true \
    --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem \
    --peerAddresses peer0.org1.example.com:7051 \
    --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt \
    --peerAddresses peer0.org2.example.com:8051 \
    --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"

function createAccount() {
  echo "Opening new account ..."
  $COMMAND -c '{"Args":["open","A","1000"]}'
  $COMMAND -c '{"Args":["open","B","1000"]}'
  $COMMAND -c '{"Args":["open","C","1000"]}'
  $COMMAND -c '{"Args":["open","D","1000"]}'
}

function sendTransactions() {
  echo "Sending..."
  $COMMAND -c '{"Args":["transfer","A","B","100"]}'
  $COMMAND -c '{"Args":["transfer","A","B","100"]}'
  $COMMAND -c '{"Args":["transfer","A","C","100"]}'
  $COMMAND -c '{"Args":["transfer","D","A","100"]}'
}

function testReorder() {
  echo "Testing..."
  $COMMAND -c '{"Args":["transfer","A","C","100"]}'
  $COMMAND -c '{"Args":["copy","A","B"]}'
  $COMMAND -c '{"Args":["copy","A","C"]}'
  $COMMAND -c '{"Args":["copy","A","D"]}'
}

function queryAccount() {
  if [[ -z $AC ]]; then
    docker exec cli1 peer chaincode query -C mychannel -n simple -c '{"Args":["query","A"]}'
    docker exec cli1 peer chaincode query -C mychannel -n simple -c '{"Args":["query","B"]}'
    docker exec cli1 peer chaincode query -C mychannel -n simple -c '{"Args":["query","C"]}'
    docker exec cli1 peer chaincode query -C mychannel -n simple -c '{"Args":["query","D"]}'
  else 
    docker exec cli1 peer chaincode query -C mychannel -n simple -c '{"Args":["query", "'"$AC"'"]}'
  fi
}

function cli1() {
  docker exec -it cli1 bash
}

function networkDown() {
  echo "===========================================Ending Test Network==========================================="

  echo "Stoping and Pruning fabric dockers..."
  docker ps -a | awk '/fabric/ {print $1}' | xargs -r docker stop
  docker ps -a | awk '/dev/ {print $1}' | xargs -r docker stop
  docker ps -a | awk '/fabric/ {print $1}' | xargs -r docker rm -f
  docker ps -a | awk '/dev/ {print $1}' | xargs -r docker rm -f
  docker volume prune -f
  docker network rm fabric_test
  docker rmi $(docker images | grep dev)
  docker image prune -f

  echo "Removing files..."
  rm -rf ./channel-artifacts
  rm -rf ./crypto-config/ordererOrganizations
  rm -rf ./crypto-config/peerOrganizations
  rm ./mychannel.block
}

if [ "$MODE" == "up" ]; then
  networkUp
elif [ "$MODE" == "down" ]; then
  networkDown
elif [ "$MODE" == "open" ]; then
  createAccount
elif [ "$MODE" == "send" ] || [ "$MODE" == "s" ]; then
  sendTransactions
elif [ "$MODE" == "test" ] || [ "$MODE" == "t" ]; then
  testReorder
elif [ "$MODE" == "query" ] || [ "$MODE" == "q" ]; then
  queryAccount
elif [ "$MODE" == "client" ] || [ "$MODE" == "c" ]; then
  cli1
else
  echo "errrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrr"
  exit 1
fi

echo "====================================================Done===================================================="
```