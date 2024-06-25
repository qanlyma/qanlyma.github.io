---
title: 「毕」 6. 测试合约及脚本
category_bar: true
date: 2023-10-9 15:56:01
tags:
categories: 毕设项目
banner_img:
---

本文记录用于测试的智能合约以及脚本，对于源码的优化修改请看我的 [Github 仓库](https://github.com/qanlyma/fabricMan)。

<!-- more -->

首先说明一下我整个实验的文件构成：

![](1.png)

* `caliper-benchmarks` 是用于性能测试的框架，js 测试文件就在其中。

* `fabricMan` 是我对 fabric 源码的修改，使用该源码编译生成 docker 镜像。

* `twoPeerNet` 是在本地使用 docker 搭建的一个区块链网络，包含了各种配置文件和 shell 脚本。

## 1 simpletest.go

该合约功能较为简单，主要有以下几个函数：

* Init 函数：这是链码的初始化函数，它在链码部署时被调用，但在后续的交易中不会再次调用。

* Invoke 函数：这是链码的调用函数，它根据传入的函数名称执行不同的操作。根据传入的函数名，它会调用 Open、Delete、Query、Transfer 或 Copy 函数，或者返回格式错误的错误消息。

* Open 函数：接受两个参数，账户名和初始金额，检查账户是否已存在，如果不存在则创建账户并存储初始金额。

* Delete 函数：接受一个参数，账户名，然后删除该账户。

* Query 函数：接受一个参数，账户名，然后返回账户的当前余额。

* Transfer 函数：接受三个参数，源账户名、目标账户名和要转移的金额。它会检查源账户的余额是否足够，如果足够则执行转账操作。

* R2w2 函数：接受四个参数，对前两个账户进行读操作，对后两个进行写操作，是用于重排序测试而添加的函数。

* main 函数：程序的入口点，用于启动链码。它使用 shim.Start 函数启动链码，如果启动失败，则输出错误信息。

```go
package main

import (
	"strconv"

	"github.com/hyperledger/fabric-chaincode-go/shim"
	pb "github.com/hyperledger/fabric-protos-go/peer"
)

const ERROR_SYSTEM = "{\"code\":300, \"reason\": \"system error: %s\"}"
const ERROR_WRONG_FORMAT = "{\"code\":301, \"reason\": \"command format is wrong\"}"
const ERROR_ACCOUNT_EXISTING = "{\"code\":302, \"reason\": \"account already exists\"}"
const ERROR_ACCOUNT_ABNORMAL = "{\"code\":303, \"reason\": \"abnormal account\"}"
const ERROR_MONEY_NOT_ENOUGH = "{\"code\":304, \"reason\": \"account's money is not enough\"}"

type SimpleChaincode struct {
}

func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response {
	// nothing to do
	return shim.Success(nil)
}

func (t *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
	function, args := stub.GetFunctionAndParameters()

	if function == "open" {
		return t.Open(stub, args)
	}
	if function == "delete" {
		return t.Delete(stub, args)
	}
	if function == "query" {
		return t.Query(stub, args)
	}
	if function == "transfer" {
		return t.Transfer(stub, args)
	}
	if function == "r2w2" {
		return t.R2w2(stub, args)
	}

	return shim.Error(ERROR_WRONG_FORMAT)
}

// open an account, should be [open account money]
func (t *SimpleChaincode) Open(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 2 {
		return shim.Error(ERROR_WRONG_FORMAT)
	}

	account := args[0]
	money, _ := stub.GetState(account)
	if money != nil {
		return shim.Error(ERROR_ACCOUNT_EXISTING)
	}

	stub.PutState(account, []byte(args[1]))

	return shim.Success(nil)
}

// delete an account, should be [delete account]
func (t *SimpleChaincode) Delete(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 1 {
		return shim.Error(ERROR_WRONG_FORMAT)
	}

	stub.DelState(args[0])

	return shim.Success(nil)
}

// query current money of the account,should be [query accout]
func (t *SimpleChaincode) Query(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 1 {
		return shim.Error(ERROR_WRONG_FORMAT)
	}

	money, _ := stub.GetState(args[0])

	if money == nil {
		return shim.Error(ERROR_ACCOUNT_ABNORMAL)
	}

	return shim.Success(money)
}

// transfer money from account1 to account2, should be [transfer account1 account2 money]
func (t *SimpleChaincode) Transfer(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 3 {
		return shim.Error(ERROR_WRONG_FORMAT)
	}
	money, _ := strconv.Atoi(args[2])

	moneyBytes1, _ := stub.GetState(args[0])
	moneyBytes2, _ := stub.GetState(args[1])

	if moneyBytes1 == nil || moneyBytes2 == nil {
		return shim.Error(ERROR_ACCOUNT_ABNORMAL)
	}

	money1, _ := strconv.Atoi(string(moneyBytes1))
	money2, _ := strconv.Atoi(string(moneyBytes2))
	if money1 < money {
		return shim.Error(ERROR_MONEY_NOT_ENOUGH)
	}

	money1 -= money
	money2 += money

	stub.PutState(args[0], []byte(strconv.Itoa(money1)))
	stub.PutState(args[1], []byte(strconv.Itoa(money2)))

	return shim.Success(nil)
}

// read 2 account and write 2 account, should be [r2w2 account1 account2 account3 account4]
func (t *SimpleChaincode) R2w2(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 4 {
		return shim.Error(ERROR_WRONG_FORMAT)
	}

	moneyBytes1, _ := stub.GetState(args[0])
	moneyBytes2, _ := stub.GetState(args[1])

	if moneyBytes1 == nil || moneyBytes2 == nil {
		return shim.Error(ERROR_ACCOUNT_ABNORMAL)
	}

	stub.PutState(args[2], moneyBytes1)
	stub.PutState(args[3], moneyBytes2)

	return shim.Success(nil)
}

func main() {
	shim.Start(new(SimpleChaincode))
}
```

## 2 js 模块

### 2.1 operation-base.js

OperationBase 类是一个继承自 WorkloadModuleBase 的基类，它包含了一些方法和属性，用于执行特定操作的性能测试。这些方法包括初始化、断言连接器类型、断言设置、创建连接器请求等。

这个文件中的 OperationBase 类是 Hyperledger Caliper 性能测试框架的一部分，它提供了一些通用的功能和规范，以便在具体的区块链性能测试模块中执行不同的操作。子类可以继承 OperationBase 并根据需要实现抽象方法，以执行特定操作的性能测试。这有助于在一个统一的框架下执行不同区块链平台的性能测试。

### 2.2 simple-state.js

这个文件中的 SimpleState 类是一个用于管理账户状态的工具，它允许生成新的账户、查询账户和执行账户之间的资金转移。这在性能测试等场景中可能非常有用，因为它可以模拟不同账户之间的交互。

* getOpenAccountArguments 方法：用于获取创建新账户所需的参数。每次调用这个方法都会生成一个新的账户，并返回账户名称和初始资金。

* getQueryArguments 方法：用于获取查询账户所需的参数。它会随机选择一个账户进行查询操作。

* getTransferArguments 方法：用于获取转账操作所需的参数。它会随机选择两个账户和转账金额。

* getCopyArguments 方法：用于获取我新加的函数 Copy 的参数。

### 2.3 open.js

这个文件定义了一个用于在系统下测试（SUT）中初始化各种账户的工作负载模块 Open。该模块继承了一个通用的操作基类 OperationBase，并使用 SimpleState 类来管理账户状态。通过实现 createSimpleState 和 submitTransaction 方法，这个模块可以模拟打开新账户的操作，并通过 sutAdapter 向 SUT 发送相应的请求。最后，通过导出 createWorkloadModule 函数，其他代码可以轻松地创建并配置该工作负载模块。

```js
'use strict';

const OperationBase = require('./utils/operation-base');
const SimpleState = require('./utils/simple-state');

class Open extends OperationBase {
    constructor() {
        super();
    }

    createSimpleState() {
        return new SimpleState(this.workerIndex, this.initialMoney, this.moneyToTransfer);
    }

    async submitTransaction() {
        let createArgs = this.simpleState.getOpenAccountArguments();
        await this.sutAdapter.sendRequests(this.createConnectorRequest('open', createArgs));
    }
}

function createWorkloadModule() {
    return new Open();
}

module.exports.createWorkloadModule = createWorkloadModule;
```

* Open 类是一个工作负载模块，它继承了 OperationBase 类，这意味着它继承了 OperationBase 类中定义的方法和属性。这个类的主要功能是执行账户的初始化操作。

* constructor 方法：这个构造函数调用了 super()，以初始化基类 OperationBase。通常，在派生类的构造函数中，需要首先调用基类的构造函数。

* createSimpleState 方法：这个方法是在基类 OperationBase 中定义的抽象方法，子类必须实现它。在这个特定的子类中，它被实现为创建一个新的 SimpleState 对象，以便进行账户状态的初始化。

* submitTransaction 方法：这个方法是特定于 Open 类的方法，用于生成并提交交易以打开新账户。它首先调用 simpleState.getOpenAccountArguments() 来获取创建新账户所需的参数，然后使用 sutAdapter（SUT 适配器）将请求发送到系统下测试中。

### 2.4 query.js

这个文件定义了一个用于查询各种账户的工作负载模块 Query。该模块继承了一个通用的操作基类 OperationBase，并使用 SimpleState 类来管理账户状态。通过实现 createSimpleState 和 submitTransaction 方法，这个模块可以模拟查询账户的操作，并通过 sutAdapter 向 SUT 发送相应的请求。

```js
'use strict';

const OperationBase = require('./utils/operation-base');
const SimpleState = require('./utils/simple-state');

class Query extends OperationBase {
    constructor() {
        super();
    }

    createSimpleState() {
        const accountsPerWorker = this.numberOfAccounts / this.totalWorkers;
        return new SimpleState(this.workerIndex, this.initialMoney, this.moneyToTransfer, accountsPerWorker);
    }

    async submitTransaction() {
        const queryArgs = this.simpleState.getQueryArguments();
        await this.sutAdapter.sendRequests(this.createConnectorRequest('query', queryArgs));
    }
}

function createWorkloadModule() {
    return new Query();
}

module.exports.createWorkloadModule = createWorkloadModule;
```

* Query 类是一个工作负载模块，它继承了 OperationBase 类，这意味着它继承了 OperationBase 类中定义的方法和属性。这个类的主要功能是执行账户查询操作。

* constructor 方法：这个构造函数调用了 super()，以初始化基类 OperationBase。通常，在派生类的构造函数中，需要首先调用基类的构造函数。

* createSimpleState 方法：这个方法是在基类 OperationBase 中定义的抽象方法，子类必须实现它。在这个特定的子类中，它被实现为创建一个新的 SimpleState 对象，以便进行账户状态的查询。它还会计算每个工作者所需的账户数量，以确保查询分布均匀。

* submitTransaction 方法：这个方法是特定于 Query 类的方法，用于生成并提交查询交易以查询账户。它首先调用 simpleState.getQueryArguments() 来获取执行查询所需的参数，然后使用 sutAdapter（SUT 适配器）将请求发送到系统下测试中。

### 2.5 transfer.js

这个文件定义了一个用于在不同账户之间执行资金转移操作的工作负载模块 Transfer。该模块继承了一个通用的操作基类 OperationBase，并使用 SimpleState 类来管理账户状态。通过实现 createSimpleState 和 submitTransaction 方法，这个模块可以模拟账户之间的资金转移操作，并通过 sutAdapter 向 SUT 发送相应的请求。

```js
'use strict';

const OperationBase = require('./utils/operation-base');
const SimpleState = require('./utils/simple-state');

class Transfer extends OperationBase {
    constructor() {
        super();
    }

    createSimpleState() {
        const accountsPerWorker = this.numberOfAccounts / this.totalWorkers;
        return new SimpleState(this.workerIndex, this.initialMoney, this.moneyToTransfer, accountsPerWorker);
    }

    async submitTransaction() {
        const transferArgs = this.simpleState.getTransferArguments();
        await this.sutAdapter.sendRequests(this.createConnectorRequest('transfer', transferArgs));
    }
}

function createWorkloadModule() {
    return new Transfer();
}

module.exports.createWorkloadModule = createWorkloadModule;
```

* Transfer 类是一个工作负载模块，它继承了 OperationBase 类，这意味着它继承了 OperationBase 类中定义的方法和属性。这个类的主要功能是执行资金转移操作。

* constructor 方法：这个构造函数调用了 super()，以初始化基类 OperationBase。通常，在派生类的构造函数中，需要首先调用基类的构造函数。

* createSimpleState 方法：这个方法是在基类 OperationBase 中定义的抽象方法，子类必须实现它。在这个特定的子类中，它被实现为创建一个新的 SimpleState 对象，以便进行账户状态的转账。它还会计算每个工作者所需的账户数量，以确保转账分布均匀。

* submitTransaction 方法：这个方法是特定于 Transfer 类的方法，用于生成并提交资金转移交易。它首先调用 simpleState.getTransferArguments() 来获取执行资金转移所需的参数，然后使用 sutAdapter（SUT 适配器）将请求发送到系统下测试中。

## 3 config.yaml

测试的配置文件：

```yaml
simpleArgs: &simple-args
  initialMoney: 10000
  moneyToTransfer: 100
  numberOfAccounts: &number-of-accounts 1000

test:
  name: simple
  description: >-
    This is an example benchmark for Caliper, to test the backend DLT's
    performance with simple account opening & querying transactions.
  workers:
    number: 1
  rounds:
    - label: open
      description: >-
        Test description for the opening of an account through the deployed
        contract.
      txNumber: *number-of-accounts
      rateControl:
        type: fixed-rate
        opts:
          tps: 50
      workload:
        module: benchmarks/scenario/simple/open.js
        arguments: *simple-args
    - label: query
      description: Test description for the query performance of the deployed contract.
      txNumber: *number-of-accounts
      rateControl:
        type: fixed-rate
        opts:
          tps: 100
      workload:
        module: benchmarks/scenario/simple/query.js
        arguments: *simple-args
    - label: transfer
      description: Test description for transfering money between accounts.
      txNumber: 50
      rateControl:
        type: fixed-rate
        opts:
          tps: 5
      workload:
        module: benchmarks/scenario/simple/transfer.js
        arguments:
          << : *simple-args
          money: 100
```

* `simpleArgs: &simple-args`：定义了一个命名的数据结构 simple-args，它包含了一些初始参数值。

* workers：指定了测试中工作人员（或并发用户）的数量，这里是 1。

* rounds：这是测试的轮次配置，包括多个轮次的描述。每个轮次测试不同的操作。

	* label：轮次的名称或标签。
	* description：轮次的描述。
	* txNumber：执行事务的数量。
	* rateControl：控制事务执行的速率。
	* workload：指定了执行测试操作的工作负载模块和参数。

		* module：指定了要使用的工作负载模块的路径。
		* arguments：指定了传递给工作负载模块的参数。在这里，使用了 YAML 锚点来引用之前定义的参数。

## 4 test-network.yaml

这个配置文件的主要目的是为 Caliper 基准测试工具提供有关要执行的性能测试的信息，包括测试的通道、智能合约、参与组织和连接配置等。它使您能够定义测试环境和参数，以便执行性能测试并评估区块链系统的性能。

```yaml
name: Caliper Benchmarks
version: "2.0.0"

caliper:
  blockchain: fabric

channels:
  # channelName of mychannel matches the name of the channel created by test network
  - channelName: mychannel
    # the chaincodeIDs of all the fabric chaincodes in caliper-benchmarks
    contracts:
    - id: fabcar
    - id: fixed-asset
    - id: marbles
    - id: simple
    - id: smallbank

organizations:
  - mspid: Org1MSP
    # Identities come from cryptogen created material for test-network
    identities:
      certificates:
      - name: 'User1'
        clientPrivateKey:
          path: '../twoPeerNet/crypto-config/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp/keystore/priv_sk'
        clientSignedCert:
          path: '../twoPeerNet/crypto-config/peerOrganizations/org1.example.com/users/User1@org1.example.com/msp/signcerts/User1@org1.example.com-cert.pem'
    connectionProfile:
      path: '../twoPeerNet/crypto-config/peerOrganizations/org1.example.com/connection-org1.yaml'
      discover: true
```