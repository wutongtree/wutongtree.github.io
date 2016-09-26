---
layout: page
title: "[翻译]如何编写chaincode"
description: ""
---
{% include JB/setup %}

原文：<https://github.com/IBM-Blockchain/learn-chaincode/blob/master/README.md>

翻译：[梧桐树](https://wutongtree.com)

---



# 如何编写 chaincode

本教程演示基本构建block和构建一个基本的[Hyperledger fabric](https://github.com/hyperledger/fabric) chaincode 应用必要的功能。据此，你就可以逐步构建一个可运行的能够创建通用资产的chaincode。然后你就可以用网络API与chaincode交互。完成该教程后你应该能明确的回答以下问题：

- 什么是 chaincode？
- 如何实现 chaincode？
- 实现 chaincode 需要什么依赖？
- 有哪些主要功能？
- 如何编译 chaincode？
- 如何为参数传递不同的值？
- 如何安全地在自己的blockchain网络上登记用户？
- 如何使用 REST API与chaincode交互？


## 什么是 chaincode?

Chaincode 是一段部署在 [Hyperledger fabric](https://github.com/hyperledger/fabric) peer 网络上的一段代码，这些peer在网络中交互共享账本。

***

# 实现第一个 Chaincode

#### 设置开发环境

目前，Hyperledger Fabric 支持用Go语言编写chaincode，因此需要 [Go 1.6](https://blog.golang.org/go1.6)。如果已经安装了Go，请跳到步骤3。如果你已经安装好了 Hyperledger fabric [d开发环境](https://github.com/hyperledger/fabric/blob/master/docs/dev-setup/devenv.md)，请跳过该部分。

1. 下载并安装 Golang 。如果是第一次安装 Go， 请按照说明[正确安装](https://golang.org/doc/install#testing) 。
2. 添加 Hyperledger shim 源码到 Go 工作目录 (就是你设置的 $GOPATH) 。打开终端输入以下命令：

	```
	cd $GOPATH
	go get github.com/hyperledger/fabric/core/chaincode/shim
	```
	
3. [IBM Bluemix](https://console.ng.bluemix.net/) 的 IBM Blockchain service 目前要求chain代码在 [GitHub](https://Github.com/) 仓库中。因此，如果还没有Github账号就[注册一个](http://github.com)。
4. 如果没有安装Git，就先 [安装](https://help.github.com/articles/set-up-git/)。

## 部署 chaincode 到 IBM Bluemix 上

1. Fork 仓库到自己的github账号 (滚动到顶部，点击 **Fork**)。
2. 克隆代码到 $GOPATH。

	```
	cd $GOPATH
	mkdir -p src/github.com/<yourgithubid>/
	cd src/github.com/<yourgithubid>/
	git clone https://github.com/<yourgithubid>/learn-chaincode.git
	```
	
3. 注意，在本教程中我们提供了两个不同的chaincode版本：[初始版](https://github.com/IBM-Blockchain/learn-chaincode/blob/master/start/chaincode_start.go) - chaincode的框架，你可据此继续开发，和 [完成版](https://github.com/IBM-Blockchain/learn-chaincode/blob/master/finished/chaincode_finished.go) - 完成的chaincode。
4. 确保以下在本地环境执行：
	- 打开终端
	
	```
	cd $GOPATH/src/github.com/<yourgithubid>/learn-chaincode/start
	go build ./
	```
	- 如果已上命令执行出错，请确保在所有步骤之前[Go安装正确](https://golang.org/doc/install#testing)。


### 实现 chaincode 接口

首先，你需要在你的golang代码里实现 chaincode shim 的接口，即三个方法 **Init**、 **Invoke**、 和 **Query**。
这三个方法有相同之处：都接收方法名和string数组参数。不同的是各自的调用时机。我们将构建一个chaincode来创建通用资产。

### 依赖

chaincode构建成功需要导入很多包：

- `fmt` - 包含输出调试和日志信息的 `Println`。
- `errors` - 标准化错误格式。
- `github.com/hyperledger/fabric/core/chaincode/shim` - 有chaincode代码用的peer接口。

### Init()

Init 在第一次deploy chaincode时调用。就行其名字暗示的，这个方法用来初始化chaincode需要初始化的东西。在我们的例子中，用此方法设置账本中一个变量的初始状态（world state）。

在`chaincode_start.go` 文件里，修改 `Init` 方法来存储 `args` 参数的第一个元素为key="hello_world"的值。

```go
func (t *SimpleChaincode) Init(stub *shim.ChaincodeStub, function string, args []string) ([]byte, error) {
	if len(args) != 1 {
		return nil, errors.New("Incorrect number of arguments. Expecting 1")
	}

	err := stub.PutState("hello_world", []byte(args[0]))
	if err != nil {
		return nil, err
	}

	return nil, nil
}
```

着用到了 shim 中的 `stub.PutState`方法，此方法的第一个参数是string类型的key，第二个参数是byte数组类型的值，返回错误或者是否存在。

### Invoke()

`Invoke` 方法在用chaincode处理真正的业务时调用，调用交易将被抓获为一个chain上的block。`Invoke`方法结构很简单，它接收一个`function`参数，并依赖此参数调用chaincode里的具体的Go方法。

在`chaincode_start.go` 文件里，修改 `Invoke` 方法来调用的`write`方法。

```go
func (t *SimpleChaincode) Invoke(stub *shim.ChaincodeStub, function string, args []string) ([]byte, error) {
	fmt.Println("invoke is running " + function)

	// Handle different functions
	if function == "init" {
		return t.Init(stub, "init", args)
	} else if function == "write" {
		return t.write(stub, args)
	}
	fmt.Println("invoke did not find func: " + function)

	return nil, errors.New("Received unknown function invocation")
}
```

现在来看一下 `write` ，在 `chaincode_start.go` 里写下该方法。

```go
func (t *SimpleChaincode) write(stub *shim.ChaincodeStub, args []string) ([]byte, error) {
	var key, value string
	var err error
	fmt.Println("running write()")

	if len(args) != 2 {
		return nil, errors.New("Incorrect number of arguments. Expecting 2. name of the key and value to set")
	}

	key = args[0]                            //rename for fun
	value = args[1]
	err = stub.PutState(key, []byte(value))  //write the variable into the chaincode state
	if err != nil {
		return nil, err
	}
	return nil, nil
}
```

这个`write` 方法看起来跟刚刚修改的 `Init` 方法很想，主要的不同是`write` 方法中用`PutState`既可以设置key又可以设置valu。这个方法允许存储任意的key/value对到区块链账本中。 

### Query()

`Query` 在查询chaincode state时调用，查询不会在chain中添加block，只是来读取chaincode state的key/value对。

在 `chaincode_start.go` 文件里，修改 `Query` 方法来调用 `read` 方法。

```go
func (t *SimpleChaincode) Query(stub *shim.ChaincodeStub, function string, args []string) ([]byte, error) {
	fmt.Println("query is running " + function)

	// Handle different functions
	if function == "read" {                            //read a variable
		return t.read(stub, args)
	}
	fmt.Println("query did not find func: " + function)

	return nil, errors.New("Received unknown function query")
}
```

现在看一下 `read` 方法，在 `chaincode_start.go` 里写下该方法。

```go
func (t *SimpleChaincode) read(stub *shim.ChaincodeStub, args []string) ([]byte, error) {
	var key, jsonResp string
	var err error

	if len(args) != 1 {
		return nil, errors.New("Incorrect number of arguments. Expecting name of the key to query")
	}

	key = args[0]
	valAsbytes, err := stub.GetState(key)
	if err != nil {
		jsonResp = "{\"Error\":\"Failed to get state for " + key + "\"}"
		return nil, errors.New(jsonResp)
	}

	return valAsbytes, nil
}
```

`read` 方法调用`GetState`来获取之前 `PutState`保存的值。`GetState`方法是需要一个string类型的参数，该参数是要获取值得key，方法返回byte数组值并响应给REST请求。

### Main()

最后，需要创建一个简短的 `main` 方法，该方法在每一个peer部署该chaincode实例时执行。此方法启动chaincode并将之注册到peer上，chaincode_start.go 和 chaincode_finished.go两个文件已经写好了该方法，不需要再修改其代码：

```go
func main() {
	err := shim.Start(new(SimpleChaincode))
	if err != nil {
		fmt.Printf("Error starting Simple chaincode: %s", err)
	}
}
```

### 需要帮助？

如有任何不明之处，请查看 chaincode_finished.go 文件，用该文件验证你写的 chaincode_start.go 是否正确。

# 与第一个 Chaincode 互动

测试chaincode最快的方式是使用peer上的rest接口。Swagger UI不需要写任何其它代码就可以测试部署chaincode。

### Swagger API

第一步就是查看 swagger api。

1. 登陆 [IBM Bluemix](https://console.ng.bluemix.net/login)
1. 登陆进 Dashboard，如果没有则点击 "Dashboard" 标签。
1. 还要确保在相同的 Bluemix "空间" ，该空间包含 IBM Blockchain 服务，其导航在左边。
1. 在Bluemix dashboard底部有个 Services 面板，可以查看服务。
1. 至此可以看到页面上有 "Welcome to the IBM Blockchain..."字样，点击右侧的 "LAUNCH" 按钮。
1. 在显示页面上有两个表，下边一个表可能是空的。
	- 选项卡上的信息应该注意：
		- **Peer Logs** 在上边的表，找到peer1那一行，然后点击像文件的图标。
			- 这样会打开新窗口，这就是peer 日志。
			- 除了这种静态查看日志的方式以外，还可以通过页面上边的 **View Logs** 标签实时查看日志。
		- **ChainCode Logs** 在下边的表，每个chaincode都有一行记录，以deploy时返回的的hash作为标记。找到对应的chaincodeID选择peer并点击像文件一样的图标。
			- 这样会打开新窗口，这就是peer的chaincode日志。
	- **Swagger Tab** 用来查看API文档。
		-点击便进入了 Swagger API 页面。

### 安全注册

调用 `/chaincode` restAPI时需要 secure context ID，这就意味着你必须从服务证书列表中通过注册enrollID，以使多数REST调用被接受。

- 点击链接 **+ Network's Enroll IDs** 来展开 enrollID及对应 secrets。
- 打开记事本并复制一组证书，你之后会需要用到.
- 点击展开 "Registrar" API
- 点击展开 `POST /registrar` 
- 设置请求body，应该是包含 enrollID 和 secret 的Json数据，例如：

![Register Example](https://raw.githubusercontent.com/IBM-Blockchain/learn-chaincode/master/imgs/registrar.PNG)

![Register Example](https://raw.githubusercontent.com/IBM-Blockchain/learn-chaincode/master/imgs/register_response.PNG)

如果没有收到 "Login successful" 的相应信息，返回查看是否正确复制enrollID 和 secret。如此你的enrollID 便设置成功，只有就可以用这个ID对chaincode 来deplo、invoke、quer了。

### Deploy

为了通了rest接口deploy chaincode，你需要将chaincode源码放到github仓库。当你向peer发送一个deploy请求时，将chaincode仓库的url作为必要的参数包含到请求里。

**deploy 之前**，确保chaincode源码能本地编译通过！

- 打开终端命令行工具
- 进入包含 `chaincode_start.go` 的文件夹并且输入：

	```
	go build ./
	```
- 返回无错信息
- 点击展开 "Chaincode" API
- 点击展开 `POST /chaincode`
- 设置 `DeploySpec` 字段(其他字段设空)。复制下边的示例并将path替换为自己的chaincode path，也将secureContext设为上一步 `/registrar` 中的enrollID。
- chaincode path格式应该如`"https://github.com/johndoe/learn-chaincode/finished"`，即fork的仓库里包含 chaincode_finished.go 文件的那个路径。

	```js
	{
		"jsonrpc": "2.0",
		"method": "deploy",
		"params": {
			"type": 1,
			"chaincodeID": {
				"path": "https://github.com/johndoe/learn-chaincode/finished"
			},
			"ctorMsg": {
				"function": "init",
				"args": [
					"hi there"
				]
			},
			"secureContext": "user_type1_191b8c2993"
		},
		"id": 1
	}
	```

请求响应应该如下：

![Deploy Example](https://raw.githubusercontent.com/IBM-Blockchain/learn-chaincode/master/imgs/deploy_response.PNG)

deploy的响应信息中包含一个与chaincode相关的ID（message字段），这个ID是128位的hash字符串，向之前的enrollID那样复制这个ID保存下来，用于对该chaincode的invoke或query。

### Query

接下来查询之前 `Init` 方法设置的`hello_world` 的值。

- 点击展开 "Chaincode" API
- 点击展开 `POST /chaincode`
- 设置 `QuerySpec` 字段 (其他字段设空)。复制下面示例并以deploy时返回的hash ID替换chaincode name，也是用之前 `/registrar` 的enrollID。

	```js
	{
		"jsonrpc": "2.0",
		"method": "query",
		"params": {
			"type": 1,
			"chaincodeID": {
				"name": "CHAINCODE_HASH_HERE"
			},
			"ctorMsg": {
				"function": "read",
				"args": [
					"hello_world"
				]
			},
			"secureContext": "user_type1_xxxxxxxxx"
		},
		"id": 2
	}
	```

![Query Example](https://raw.githubusercontent.com/IBM-Blockchain/learn-chaincode/master/imgs/query_response.PNG)

希望你看到 `hello_world` 是 "hi there"，这是在deploy时在请求body中设置的。

### Invoke

接下来调用 `invoke` 方法，将`hello_world` 的值改为 "go away"。

- 点击展开 "Chaincode" API 
- 点击展开 `POST /chaincode`
- 设置 `InvokeSpec` 字段(其他字段设空)。复制下面示例并以hash ID替换chaincode name ，也使用 `/registrar` 时的enrollID。

	```js
	{
		"jsonrpc": "2.0",
		"method": "invoke",
		"params": {
			"type": 1,
			"chaincodeID": {
				"name": "CHAINCODE_HASH_HERE"
			},
			"ctorMsg": {
				"function": "write",
				"args": [
					"hello_world",
					"go away"
				]
			},
			"secureContext": "user_type1_xxxxxxxxx"
		},
		"id": 3
	}
	```

![Invoke Example](https://raw.githubusercontent.com/IBM-Blockchain/learn-chaincode/master/imgs/invoke_response.PNG)

现在来测试下invoke修改值是否成功，重新执行query即可。

![Query2 Example](https://raw.githubusercontent.com/IBM-Blockchain/learn-chaincode/master/imgs/query2_response.PNG)


以上即是编写基础chaincode的全部。
