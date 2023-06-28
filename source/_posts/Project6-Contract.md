---
title: 「毕」 6. 合约学习：smallbank
category_bar: true
date: 2023-06-26 17:25:11
tags:
categories: 毕设项目
banner_img:
---

本文学习之后会用于测试的智能合约：smallbank。

<!-- more -->

## smallbank.go

该合约是 caliper-benchmarks 自带的测试合约，模拟银行系统的基本操作，主要用于区块链性能测试和基准测试。

每个账户都有自己的 ID 和 Name，还有两种余额：
```go
type Account struct {
	CustomId        string
	CustomName      string
	SavingsBalance  int
	CheckingBalance int
}
```

下面介绍主要的几个函数（省略了部分错误处理）：

### Init

```go
func (t *SmallbankChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response {
	// nothing to do
	return shim.Success(nil)
}
```
初始化函数，在链码实例化时执行，在这个示例中没有实际的操作。

### Invoke

```go
func (t *SmallbankChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
	function, args := stub.GetFunctionAndParameters()
	switch function {
	case "create_account":
		return t.CreateAccount(stub, args)
	case "transact_savings":
		return t.TransactSavings(stub, args)
	case "deposit_checking":
		return t.DepositChecking(stub, args)
	case "send_payment":
		return t.SendPayment(stub, args)
	case "transferm":
		return t.SendPayment(stub, args)
	case "write_check":
		return t.WriteCheck(stub, args)
	case "amalgamate":
		return t.Amalgamate(stub, args)
	case "query":
		return t.Query(stub, args)
	default:
		return errormsg(ERROR_UNKNOWN_FUNC + ": " + function)
	}
}
```
合约的主要逻辑入口函数，根据传入的函数名调用相应的功能函数。这里我加入了 `transferm` 作为可合并交易的标志，但是由于 Account 结构体的原因目前还不能实现。

### CreateAccount

```go
func (t *SmallbankChaincode) CreateAccount(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 4 { //should be [customer_id, customer_name, initial_checking_balance, initial_savings_balance]
		return errormsg(ERROR_WRONG_ARGS + " create_account")
	}

	key := accountKey(args[0])
	data, err := stub.GetState(key)
	if data != nil {
		return errormsg("Can not create duplicated account")
	}

	checking, errcheck := strconv.Atoi(args[2])

	saving, errsaving := strconv.Atoi(args[3])

	account := &Account{
		CustomId:        args[0],
		CustomName:      args[1],
		SavingsBalance:  saving,
		CheckingBalance: checking}
	err = saveAccount(stub, account)

	return shim.Success(nil)
}
```
创建账户函数，根据传入的参数 `[customer_id, customer_name, initial_checking_balance, initial_savings_balance]` 创建一个新的账户，并将账户信息保存到区块链状态数据库中。

### DepositChecking

```go
func (t *SmallbankChaincode) DepositChecking(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 2 { // should be [amount, customer_id]
		return errormsg(ERROR_WRONG_ARGS + " deposit_checking")
	}
	account, err := loadAccount(stub, args[1])

	amount, _ := strconv.Atoi(args[0])
	account.CheckingBalance += amount
	err = saveAccount(stub, account)

	return shim.Success(nil)
}
```
根据传入的参数 `[amount, customer_id]` 将指定账户的 CheckingBalance 增加指定的金额。

### WriteCheck

```go
func (t *SmallbankChaincode) WriteCheck(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 2 { // should be [amount, customer_id]
		return errormsg(ERROR_WRONG_ARGS + " write_check")
	}
	account, err := loadAccount(stub, args[1])

	amount, _ := strconv.Atoi(args[0])
	account.CheckingBalance -= amount
	err = saveAccount(stub, account)

	return shim.Success(nil)
}
```
根据传入的参数 `[amount, customer_id]` 将指定账户的 CheckingBalance 减少指定的金额。

### TransactSavings

```go
func (t *SmallbankChaincode) TransactSavings(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 2 { // should be [amount, customer_id]
		return errormsg(ERROR_WRONG_ARGS + " transaction_savings")
	}
	account, err := loadAccount(stub, args[1])

	amount, _ := strconv.Atoi(args[0])
	account.SavingsBalance += amount
	err = saveAccount(stub, account)

	return shim.Success(nil)
}
```
根据传入的参数 `[amount, customer_id]` 将指定账户的 SavingsBalance 增加或减少指定的金额。

### SendPayment

```go
func (t *SmallbankChaincode) SendPayment(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 3 { // should be [amount, dest_customer_id, source_customer_id]
		return errormsg(ERROR_WRONG_ARGS + " send_payment")
	}
	destAccount, err1 := loadAccount(stub, args[1])
	sourceAccount, err2 := loadAccount(stub, args[2])
	if err1 != nil || err2 != nil {
		return errormsg(ERR_NOT_FOUND)
	}

	amount, _ := strconv.Atoi(args[0])
	// since the contract is only used for perfomance testing, we ignore this check
	//if sourceAccount.CheckingBalance < amount {
	//	return errormsg("Insufficient funds in source checking account")
	//}
	sourceAccount.CheckingBalance -= amount
	destAccount.CheckingBalance += amount
	err1 = saveAccount(stub, sourceAccount)
	err2 = saveAccount(stub, destAccount)
	if err1 != nil || err2 != nil {
		return errormsg(ERROR_PUT_STATE)
	}

	return shim.Success(nil)
}
```
根据传入的参数 `[amount, dest_customer_id, source_customer_id]` 从源账户向目标账户转账指定金额。

### Amalgamate

```go
func (t *SmallbankChaincode) Amalgamate(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 2 { // should be [dest_customer_id, source_customer_id]
		return errormsg(ERROR_WRONG_ARGS + " amalgamate")
	}
	destAccount, err1 := loadAccount(stub, args[0])
	sourceAccount, err2 := loadAccount(stub, args[1])

	destAccount.CheckingBalance += sourceAccount.SavingsBalance
	sourceAccount.SavingsBalance = 0
	err1 = saveAccount(stub, sourceAccount)
	err2 = saveAccount(stub, destAccount)

	return shim.Success(nil)
}
```
根据传入的参数 `[dest_customer_id, source_customer_id]` 将源账户的 SavingsBalance 转移到目标账户的 CheckingBalance 上，并将源账户的储蓄账户余额设为 0。

### Testonly

```go
func (t *SmallbankChaincode) Testonly(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	if len(args) != 2 { // should be [dest_customer_id, source_customer_id]
		return errormsg(ERROR_WRONG_ARGS + " amalgamate")
	}
	destAccount, err1 := loadAccount(stub, args[0])
	sourceAccount, err2 := loadAccount(stub, args[1])
	if err1 != nil || err2 != nil {
		return errormsg(ERR_NOT_FOUND)
	}

	destAccount.CheckingBalance += sourceAccount.SavingsBalance
	err1 = saveAccount(stub, destAccount)
	if err1 != nil {
		return errormsg(ERROR_PUT_STATE)
	}

	return shim.Success(nil)
}
```
根据传入的参数 `[dest_customer_id, source_customer_id]` 将源账户的 SavingsBalance 转移到目标账户的 CheckingBalance 上。这是我为了测试交易重排添加的函数，可以看到它对 destAccount 做了读写操作，而对 sourceAccount 只做了读操作，所以调用此函数并不会在验证阶段改变 sourceAccount 的版本。

### Query

```go
func (t *SmallbankChaincode) Query(stub shim.ChaincodeStubInterface, args []string) pb.Response {
	key := accountKey(args[0])
	accountBytes, err := stub.GetState(key)

	return shim.Success(accountBytes)
}

```
查询账户信息函数，根据传入的账户 ID 从区块链状态数据库中获取账户信息。