# 快速入门

本文设定了一个简单的Fabric网络场景，包括2个organization，每个有2个peer，并使用“solo” ordering服务。网络实体所需的加密材料（x509证书）已预先生成并放到相应目录和配置文件里了，你无需修改这些配置。`examples/e2e_cli`文件夹里包含了docker-compose文件和要用来创建和测试网络的脚本文件。

本文还演示了使用配置生成工具`configtxgen`生成网络配置。

## 前提

完成以下安装Fabric源码和编译`configtxgen`工具：

* 完成[环境安装](http://hyperledger-fabric.readthedocs.io/en/latest/dev-setup/devenv.html)步骤，并设置正确的`$GOPATH`环境变量。此过程会将各种必须的软件依赖安装到你的机器上。
* 拉取Fabric源码

		git clone https://github.com/hyperledger/fabric.git

* 编译`configtxgen`工具

	如果运行在Linux，打开终端并在Fabric目录下执行以下命令：
		
		cd $GOPATH/src/github.com/hyperledger/fabric
		make configtxgen
		# 如果出错：'ltdl.h' file not found
		sudo apt install libtool libltdl-dev
		# 然后再运行make
		make configtxgen
	
	如果运行在OSX，首先安装Xcode 8.0及以上版本，然后打开终端并在Fabric目录下执行以下命令：
	
		# install Homebrew
		/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
		# add gnu-tar
		brew install gnu-tar --with-default-names
		# add libtool
		brew install libtool
		# make the configtxgen
		make configtxgen
	
	编译成功后会有以下输出：
	
		build/bin/configtxgen
		CGO_CFLAGS=" " GOBIN=/Users/johndoe/work/src/github.com/hyperledger/fabric/build/bin go install -ldflags "-X github.com/hyperledger/fabric/common/metadata.Version=1.0.0-snapshot-8d3275f -X github.com/hyperledger/fabric/common /metadata.BaseVersion=0.3.0 -X github.com/hyperledger/fabric/common/metadata.BaseDockerLabel=org.hyperledger.fabric"       github.com/hyperledger/fabric/common/configtx/tool/configtxgen
		Binary available as build/bin/configtxgen``
		
	其可执行文件放在Fabric目录的`build/bin/configtxgen`
	
## 执行完整脚本

为了加快部署过程，我们提供了一个脚本来执行所有任务。执行该脚本会生成配置结果、本地网络、Chaincode测试。

进入`examples/e2e_cli`目录，首先从Docker Hub拉取镜像：

	# make the script an executable
	chmod +x download-dockerimages.sh
	# run the script
	./download-dockerimages.sh	

这个过程会需要几分钟。脚本执行完成后，在终端有以下输出：

	===> List out hyperledger docker images
	hyperledger/fabric-ca          latest               35311d8617b4        7 days ago          240 MB
	hyperledger/fabric-ca          x86_64-1.0.0-alpha   35311d8617b4        7 days ago          240 MB
	hyperledger/fabric-couchdb     latest               f3ce31e25872        7 days ago          1.51 GB
	hyperledger/fabric-couchdb     x86_64-1.0.0-alpha   f3ce31e25872        7 days ago          1.51 GB
	hyperledger/fabric-kafka       latest               589dad0b93fc        7 days ago          1.3 GB
	hyperledger/fabric-kafka       x86_64-1.0.0-alpha   589dad0b93fc        7 days ago          1.3 GB
	hyperledger/fabric-zookeeper   latest               9a51f5be29c1        7 days ago          1.31 GB
	hyperledger/fabric-zookeeper   x86_64-1.0.0-alpha   9a51f5be29c1        7 days ago          1.31 GB
	hyperledger/fabric-orderer     latest               5685fd77ab7c        7 days ago          182 MB
	hyperledger/fabric-orderer     x86_64-1.0.0-alpha   5685fd77ab7c        7 days ago          182 MB
	hyperledger/fabric-peer        latest               784c5d41ac1d        7 days ago          184 MB
	hyperledger/fabric-peer        x86_64-1.0.0-alpha   784c5d41ac1d        7 days ago          184 MB
	hyperledger/fabric-javaenv     latest               a08f85d8f0a9        7 days ago          1.42 GB
	hyperledger/fabric-javaenv     x86_64-1.0.0-alpha   a08f85d8f0a9        7 days ago          1.42 GB
	hyperledger/fabric-ccenv       latest               91792014b61f        7 days ago          1.29 GB
	hyperledger/fabric-ccenv       x86_64-1.0.0-alpha   91792014b61f        7 days ago          1.29 GB

现在运行完整脚本：

	./network_setup.sh up <channel-ID>

如果没有设置`channel-ID`参数，channel名默认是`mychannel`。脚本执行成功后，终端显示如下信息：

	===================== Query on PEER3 on channel 'mychannel' is successful =====================

	===================== All GOOD, End-2-End execution completed =====================

此时，网络启动运行并测试成功。

## 清理

关闭网络：

	# make sure you're in the e2e_cli directory
	docker rm -f $(docker ps -aq)

然后执行`docker images`命令查看Chaincode镜像，会有类似下面的内容：

	REPOSITORY                     TAG                  IMAGE ID            CREATED             SIZE
	dev-peer3-mycc-1.0             latest               13f6c8b042c6        5 minutes ago       176 MB
	dev-peer0-mycc-1.0             latest               e27456b2bd92        5 minutes ago       176 MB
	dev-peer2-mycc-1.0             latest               111098a7c98c        5 minutes ago       176 MB
	
删除这些镜像：

	docker rmi <IMAGE ID> <IMAGE ID> <IMAGE ID>
	
例如：

	docker rmi -f 13f e27 111

最后删除配置结果，进入`crypto/orderer`目录，删除`orderer.block`和`channel.tx`。

## configtxgen

configtxgen工具用来生成两个内容：	Orderer的**bootstrap block**和Fabric的**channel configuration transaction**。

这orderer block是ordering服务的创世区块，这channel transaction文件在create channel时会被广播给orderer。

`configtx.yaml`包含网络的定义，并给出了网络组件的脱皮结构-2个成员（Org0和Org1）分别管理维护2个peer。还指出每个网络实体的加密材料的存储位置。`crypto`目录包含每个实体的admin证书、ca证书、签名证书和私钥。

为了方便使用，我们提供了一个脚本`generateCfgTrx.sh`，该脚本整合了`configtxgen`的执行过程，执行后会生成两个配置结果：`orderer.block`和`channel.tx`。如果你运行过上边的`network_setup.sh`则这两个配置结果就生成过，进入`crypto/orderer`目录确认是否被删除。

## 执行`generateCfgTrx.sh`脚本

在`e2e_cli`目录下执行：

	cd $GOPATH/src/github.com/hyperledger/fabric/examples/e2e_cli

`generateCfgTrx.sh`脚本有个可选参数`channel-ID`，如果不设此参数，则默认为`mychannel`。

	# as mentioned above, the <channel-ID> parm is optional
	./generateCfgTrx.sh <channel-ID>

该脚本执行成功后，输出一下内容：

	2017/02/28 17:01:52 Generating new channel configtx
	2017/02/28 17:01:52 Creating no-op MSP instance
	2017/02/28 17:01:52 Obtaining default signing identity
	2017/02/28 17:01:52 Creating no-op signing identity instance
	2017/02/28 17:01:52 Serializing identity
	2017/02/28 17:01:52 signing message
	2017/02/28 17:01:52 signing message
	2017/02/28 17:01:52 Writing new channel tx

生成的`orderer.block`和`channel.tx`两个文件存放在`crypto/orderer`目录。

`orderer.block`是ordering服务的创世区块，`channel.tx`包含新channel的配置信息。如前所述，这俩文件都来自`configtx.yaml`及其包含的向加密材料和网络信息的数据。

***注意：***也可以手动执行脚本`generateCfgTrx.sh`里的命令，打开脚本并分析命令语法，如果使用这种方式，则必须用`e2e_cli`目录里的`configtx.yaml`文件替换Fabric common/configtx/tool目录下默认的`configtx.yaml`，然后返回fabric目录执行这些命令，前提是确保之前`generateCfgTrx.sh`生成的两个文件删除。

## 启动网络

我们将使用docker-compose来启动网络，如果没有拉取Fabric镜像，则返回上边去拉取镜像。

脚本`script.sh`嵌入到docker-compose文件里，该脚本将peer加入到channel并向peer发送read/write请求，如此便可不需手动执行命令而查看交易流程。如果不想使用这个脚本自动执行交易，可以跳到下面“手动执行交易”一节。

在`e2e_cli`目录下使用docker-compose生成网络实体并执行嵌入的脚本：

	CHANNEL_NAME=<channel-id> docker-compose up -d

如果之前创建了一个channel名，就必须将其作为参数，否则使用默认的`mychannel`。例如：

	CHANNEL_NAME=mychannel docker-compose up -d
	
等待30s，因为背后有交易会发送到peer。执行`docker ps`查看运行状态的container，可以看到如下内容：

	vagrant@hyperledger-devenv:v0.3.0-4eec836:/opt/gopath/src/github.com/hyperledger/fabric/examples/e2e_cli$ docker ps
	CONTAINER ID        IMAGE                        COMMAND                  CREATED              STATUS              PORTS                                              NAMES
	45e3e114f7a2        dev-peer3-mycc-1.0           "chaincode -peer.a..."   4 seconds ago        Up 4 seconds                                                           dev-peer3-mycc-1.0
	5970f740ad2b        dev-peer0-mycc-1.0           "chaincode -peer.a..."   24 seconds ago       Up 23 seconds                                                          dev-peer0-mycc-1.0
	b84808d66e99        dev-peer2-mycc-1.0           "chaincode -peer.a..."   48 seconds ago       Up 47 seconds                                                          dev-peer2-mycc-1.0
	16d7d94c8773        hyperledger/fabric-peer      "peer node start -..."   About a minute ago   Up About a minute   0.0.0.0:10051->7051/tcp, 0.0.0.0:10053->7053/tcp   peer3
	3561a99e35e6        hyperledger/fabric-peer      "peer node start -..."   About a minute ago   Up About a minute   0.0.0.0:9051->7051/tcp, 0.0.0.0:9053->7053/tcp     peer2
	0baad3047d92        hyperledger/fabric-peer      "peer node start -..."   About a minute ago   Up About a minute   0.0.0.0:8051->7051/tcp, 0.0.0.0:8053->7053/tcp     peer1
	1216896b7b4f        hyperledger/fabric-peer      "peer node start -..."   About a minute ago   Up About a minute   0.0.0.0:7051->7051/tcp, 0.0.0.0:7053->7053/tcp     peer0
	155ff8747b4d        hyperledger/fabric-orderer   "orderer"                About a minute ago   Up About a minute   0.0.0.0:7050->7050/tcp                             orderer

## 背后发生了什么呢？

* 在CLI容器中执行了脚本`script.sh`。该脚本用默认的`mychannel`执行`createChannel`命令，这个命令用到了之前`configtxgen `工具生成的`channel.tx`。
* `createChannel `执行后会生成一个创世区块`mychannel.block`并保存到当前目录。
* 对4个peer分别执行`joinChannel `命令，通过初始区块`mychannel.block`加入channel。
* 至此，有一个channel包含4个peer和2个organization。
* `PEER0 `和`PEER1`属于Org0，`PEER2 `和`PEER3`属于Org1。
* 这些关系的定义都在`configtx.yaml`中
* Chaincode ` chaincode_example02 `被install到`PEER0`和`PEER2`
* 然后Chaincode在`PEER2`上instantiate。实例化是指启动容器和初始化与Chaincode相关的键值对，本例中的初始值是` [“a”,”100” “b”,”200”]`。实例化的结果是一个名为`dev-peer2-mycc-1.0`的容器启动，注意，这个容器仅是针对`PEER2`。***（译注：尤其注意这里仅仅是启动了一个container）***
* 实例化时还会带有背书策略参数，本例中背书策略为`-P "OR ('Org0MSP.member','Org1MSP.member')"，意味着任何交易必须由绑定到Org0或者Org1的peer背书。
* 对于“a”的query请求发送到`PEER0 `。在之前Chaincode被install到`PEER0 `了，所以就启动了一个名为`dev-peer0-mycc-1.0 `的新容器，然后返回查询结果。由于没有write操作发生，所以“a”的值依然是“100”。
* 从“a“转移”10“给”b”的invoke请求发送到`PEER0 `
* Chaincode install到`PEER3 `
* 对“a”的query请求发送到`PEER3 `。这启动了第三个名为`dev-peer3-mycc-1.0`的容器，并返回查询结果90，正确的反映了之前的交易。

## 这表明了什么呢？

Chaincode必须被install到一个peer上才能成功的对这个peer的ledger执行read/write操作。此外，只有当一个peer上针对chaincode执行read/write操作时，这个peer上才会启动该chaincode 容器。（比如，查询“a”的值）***交易致使容器启动***。channel中的所有peer都会维护一个准确的ledger，ledger包含存储了不可变的、有序的交易记录的block，还有维护current state的statedb。这包括那些没有install chaincode的peer（就像上例中的`PEER3 `），然后在这种peer上install chaincode之后就可以直接访问了（就像上例中的`PEER3 `），因为已经instantiate过了。

## 如何查看这些交易？

查看CLI容器的log：
	
	docker logs -f cli
	
会有以下内容：

	2017-02-28 04:31:20.841 UTC [logging] InitFromViper -> DEBU 001 Setting default logging level to DEBUG for command 'chaincode'
	2017-02-28 04:31:20.842 UTC [msp] GetLocalMSP -> DEBU 002 Returning existing local MSP
	2017-02-28 04:31:20.842 UTC [msp] GetDefaultSigningIdentity -> DEBU 003 Obtaining default signing identity
	2017-02-28 04:31:20.843 UTC [msp] Sign -> DEBU 004 Sign: plaintext: 0A8F050A59080322096D796368616E6E...6D7963631A0A0A0571756572790A0161
	2017-02-28 04:31:20.843 UTC [msp] Sign -> DEBU 005 Sign: digest: 52F1A41B7B0B08CF3FC94D9D7E916AC4C01C54399E71BC81D551B97F5619AB54
	Query Result: 90
	2017-02-28 04:31:30.425 UTC [main] main -> INFO 006 Exiting.....
	===================== Query on chaincode on PEER3 on channel 'mychannel' is successful =====================

	===================== All GOOD, End-2-End execution completed =====================

你也可以实时查看日志，需要打开两个终端。

首先，停止运行着的docker容器：

	docker rm -f $(docker ps -aq)

在第一个终端启动docker-compose脚本：

	# add the appropriate CHANNEL_NAME parm
	CHANNEL_NAME=<channel-id> docker-compose up -d

在第二个终端查看log：
	
	docker logs -f cli

这将实时输出通过`script.sh`执行的交易信息。

## 如何查看chaincode log？

对每个chaincode容器单独查看log，输出一下内容：

	$ docker logs dev-peer2-mycc-1.0
	04:30:45.947 [BCCSP_FACTORY] DEBU : Initialize BCCSP [SW]
	ex02 Init
	Aval = 100, Bval = 200

	$ docker logs dev-peer0-mycc-1.0
	04:31:10.569 [BCCSP_FACTORY] DEBU : Initialize BCCSP [SW]
	ex02 Invoke
	Query Response:{"Name":"a","Amount":"100"}
	ex02 Invoke
	Aval = 90, Bval = 210

	$ docker logs dev-peer3-mycc-1.0
	04:31:30.420 [BCCSP_FACTORY] DEBU : Initialize BCCSP [SW]
	ex02 Invoke
	Query Response:{"Name":"a","Amount":"90"}

## 手动执行交易

在当前工作目录停止所有容器：

	docker rm -f $(docker ps -aq)

然后，执行`docker images`命令查看chaincode镜像，会有类似以下内容：

	REPOSITORY                     TAG                  IMAGE ID            CREATED             SIZE
	dev-peer3-mycc-1.0             latest               13f6c8b042c6        5 minutes ago       176 MB
	dev-peer0-mycc-1.0             latest               e27456b2bd92        5 minutes ago       176 MB
	dev-peer2-mycc-1.0             latest               111098a7c98c        5 minutes ago       176 MB

	Remove these images:

	.. code:: bash

    	docker rmi <IMAGE ID> <IMAGE ID> <IMAGE ID>

	For example:

	.. code:: bash

    	docker rmi -f 13f e27 111

确保之前生成的配置内容还在，如果删除了就再执行脚本：

	./generateCfgTrx.sh <channel-ID>

或者使用脚本中的命令手动生成。

## 修改docker-compose文件

打开docker-compose文件注释掉执行`script.sh`脚本的命令，如下：

	working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
	# command: /bin/bash -c './scripts/script.sh ${CHANNEL_NAME}'

保存文件，重启网络：

	# make sure you are in the e2e_cli directory where you docker-compose script resides
	# add the appropriate CHANNEL_NAME parm
	CHANNEL_NAME=<channel-id> docker-compose up -d

## 命令语法

参照`scripts`目录下的`script.sh`脚本中的create和join命令。下面的命令只是针对`PEER0`的，当对orderer和peer执行命令时，需要修改下面给出的四个环境变量的值。

	# Environment variables for PEER0
	CORE_PEER_MSPCONFIGPATH=$GOPATH/src/github.com/hyperledger/fabric/peer/crypto/peer/peer0/localMspConfig
	CORE_PEER_ADDRESS=peer0:7051
	CORE_PEER_LOCALMSPID="Org0MSP"
	CORE_PEER_TLS_ROOTCERT_FILE=$GOPATH/src/github.com/hyperledger/fabric/peer/crypto/peer/	peer0/localMspConfig/cacerts/peerOrg0.pem

这些每个peer的环境变量的值都在docker-compose文件中。

## Create channel

在cli容器中执行：

	docker exec -it cli bash

如果执行成功，会有以下信息：

	root@0d78bb69300d:/opt/gopath/src/github.com/hyperledger/fabric/peer#

用`-c`指定channel name，`-f`指定channel configuration transaction（此例中是`channel.tx`），当然也可以使用不同的名称安装 configuration transaction。

	# the channel.tx and orderer.block are mounted in the crypto/orderer directory within your cli container
	# as a result, we pass the full path for the file
 	peer channel create -o orderer0:7050 -c mychannel -f crypto/orderer/channel.tx --tls $CORE_PEER_TLS_ENABLED --cafile $GOPATH/src/github.com/hyperledger/fabric/peer/crypto/	orderer/localMspConfig/cacerts/ordererOrg0.pem

由于此例的`channel create`命令是针对orderer的，所以需要修改之前的环境变量，因此上边的命令应该是：

	CORE_PEER_MSPCONFIGPATH=$GOPATH/src/github.com/hyperledger/fabric/peer/crypto/orderer/localMspConfig CORE_PEER_LOCALMSPID="OrdererMSP" peer channel create -o orderer0:7050 -c mychannel -f crypto/orderer/channel.tx --tls $CORE_PEER_TLS_ENABLED --cafile $GOPATH/src/github.com/hyperledger/fabric/peer/crypto/orderer/localMspConfig/cacerts/ordererOrg0.pem
	
**注意：**剩下的其他命令依然在CLI容器中执行，而且要记住之前命令里每个peer对应的环境变量。

## Join channel

将指定的peer加入到channel：

	# By default, this joins PEER0 only
	# the mychannel.block is also mounted in the crypto/orderer directory
 	peer channel join -b mychannel.block

完整的命令应该是：

	CORE_PEER_MSPCONFIGPATH=$GOPATH/src/github.com/hyperledger/fabric/peer/crypto/peer/peer0/localMspConfig CORE_PEER_ADDRESS=peer0:7051 CORE_PEER_LOCALMSPID="Org0MSP" CORE_PEER_TLS_ROOTCERT_FILE=$GOPATH/src/github.com/hyperledger/fabric/peer/crypto/peer/peer0/localMspConfig/cacerts/peerOrg0.pem peer channel join -b mychannel.block
	
修改四个环境变量也可以将其他的peer加入到channel中。

## 在peer上install chaincode

将示例Go代码安装到四个对等节点中的一个：

	# remember to preface this command with the global environment variables for the appropriate peer
	peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02

## Instantiate chaincode并定义背书策略

在一个peer上实例化chaincode，这将对该peer启动一个chaincode容器，并为该chaincode设置背书策略。此例中定义的策略是有`Org0`或`Org1`中的一个peer背书即可。命令如下：

	# remember to preface this command with the global environment variables for the appropriate peer
	# remember to pass in the correct string for the -C argument.  The default is mychannel
	peer chaincode instantiate -o orderer0:7050 --tls $CORE_PEER_TLS_ENABLED --cafile $GOPATH/src/github.com/hyperledger/fabric/peer/crypto/orderer/localMspConfig/cacerts/ordererOrg0.pem -C mychannel -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02 -c '{"Args":["init","a", "100", "b","200"]}' -P "OR ('Org0MSP.member','Org1MSP.member')"

## Invoke chaincode

	# remember to preface this command with the global environment variables for the appropriate peer
	peer chaincode invoke -o orderer0:7050  --tls $CORE_PEER_TLS_ENABLED --cafile $GOPATH/src/github.com/hyperledger/fabric/peer/crypto/orderer/localMspConfig/cacerts/ordererOrg0.pem  -C mychannel -n mycc -c '{"Args":["invoke","a","b","10"]}'
	
## Query chaincode

	# remember to preface this command with the global environment variables for the appropriate peer
	peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
	
以上命令的执行结果：

	Query Result: 90

## 手动生成镜像

构建peer和orderer镜像：

	# make sure you are in the fabric directory
	# if you are unable to generate the images natively, you may need to be in a vagrant environment
	make peer-docker orderer-docker

执行`docker images`命令，如果镜像编译成功，会有以下输出：

	vagrant@hyperledger-devenv:v0.3.0-4eec836:/opt/gopath/src/github.com/hyperledger/fabric$ docker images
	REPOSITORY                     TAG                             IMAGE ID            CREATED             SIZE
	hyperledger/fabric-orderer     latest                          264e45897bfb        10 minutes ago      180 MB
	hyperledger/fabric-orderer     x86_64-0.7.0-snapshot-a0d032b   264e45897bfb        10 minutes ago      180 MB
	hyperledger/fabric-peer        latest                          b3d44cff07c6        10 minutes ago      184 MB
	hyperledger/fabric-peer        x86_64-0.7.0-snapshot-a0d032b   b3d44cff07c6        10 minutes ago      184 MB
	hyperledger/fabric-javaenv     latest                          6e2a2adb998a        10 minutes ago      1.42 GB
	hyperledger/fabric-javaenv     x86_64-0.7.0-snapshot-a0d032b   6e2a2adb998a        10 minutes ago      1.42 GB
	hyperledger/fabric-ccenv       latest                          0ce0e7dc043f        12 minutes ago      1.29 GB
	hyperledger/fabric-ccenv       x86_64-0.7.0-snapshot-a0d032b   0ce0e7dc043f        12 minutes ago      1.29 GB
	hyperledger/fabric-baseimage   x86_64-0.3.0                    f4751a503f02        4 weeks ago         1.27 GB
	hyperledger/fabric-baseos      x86_64-0.3.0                    c3a4cf3b3350        4 weeks ago         161 MB
	