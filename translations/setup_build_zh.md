---
layout: page
title: "[翻译]fabric setup and run"
description: ""
---
{% include JB/setup %}

翻译：[梧桐树](https://wutongtree.com)

---


## 配置开发环境

### 概要
目前的开发环境是利用 Vagrant 运行一个Ubuntu镜像，进而推出Docker容器。从概念上讲，主机启动一个VM，从而推出Docker容器。

**Host -> VM -> Docker**

这种模式允许开发者使用自己喜欢的系系统和编辑器，并且在开发团队中一致且可控的环境中执行该系统。

- 注意：你的Host不应该在VM中运行。如果你试图这样，Host中的VM会提示“VT-x不可用”这样的失败信息。（译者注：即VM中不能再安装运行VM，或者说云主机上不应该使用这种模式）

### 必备

* [Git client](https://git-scm.com/downloads)
* [Go](https://golang.org/) - 1.6 or later
* [Vagrant](https://www.vagrantup.com/) - 1.7.4 or later
* [VirtualBox](https://www.virtualbox.org/) - 5.0 or later
* BIOS 启用虚拟化 - 基于硬件改变

- 注意:BIOS启用虚拟化可能在BIOS中的CPU或安全设置中

### 步骤

#### 设置 GOPATH

确保Host正确的[GOPATH环境变量](https://github.com/golang/go/wiki/GOPATH)。 这用于Host 和 VM 的编译。

#### Windows 用户请注意

如果是 Windows 系统，运行 `git clone` 命令之前，运行下面的命令：

```
git config --get core.autocrlf
```

如果 `core.autocrlf` 是 `true`，必须通过运行以下命令设置为 `false`：

```
git config --global core.autocrlf false
```

如果依然保持 `core.autocrlf` 为 `true`，`vagrant up` 命令就会出 `./setup.sh: /bin/bash^M: bad interpreter: No such file or directory`这样的错误。

#### 克隆 Fabric 项目

因为 Fabric 是一个 `Go` 项目，你需要将项目克隆到 $GOPATH/src 目录。如果 $GOPATH 有多个，使用第一个。 可以使用下面的小命令：

```
cd $GOPATH/src
mkdir -p github.com/hyperledger
cd github.com/hyperledger
```

我们使用 `Gerrit` 管理源码， `Gerrit`有其内在的 git 仓库。 因此需要 [从 Gerrit 克隆](../Gerrit/gerrit.md#Working-with-a-local-clone-of-the-repository)。命令如下：

```
git clone ssh://LFID@gerrit.hyperledger.org:29418/fabric && scp -p -P 29418 LFID@gerrit.hyperledger.org:hooks/commit-msg fabric/.git/hooks/
```
**注意:** 当然你需要替换 `LFID` 为你的 [Linux Foundation ID](../Gerrit/lf-account.md).

#### 用 Vagrant 启动 VM

启动 Vagrant：

```
cd $GOPATH/src/github.com/hyperledger/fabric/devenv
vagrant up
```

喝杯咖啡吧... 这将需要几分钟。完成后，使用 `ssh` 进入刚创建的 Vagrant VM 。

```
vagrant ssh
```

### 构建 fabric

安装好 vagrant 开发环境后，你就可以 [构建和测试](build.md) fabric。在VM里，在`$GOPATH/src/github.com/hyperledger/fabric`下你能发现 peer 项目，它也被安装为 `/hyperledger`。

### 注意事项

**注意:** 任何时候你改变本地 fabric 目录的文件 (在 `$GOPATH/src/github.com/hyperledger/fabric`)，VM fabric目录里对应的文件也会及时变化。（其实是VM的fabric映射到本地fabric）

**注意:** 如果开发环境使用了 HTTP 代理，你需要配置 guest 使配置过程完成，你可以利用 *vagrant-proxyconf* 插件来实现。 用 `vagrant plugin install vagrant-proxyconf` 安装，并在执行`vagrant up`*之前*设置 `VAGRANT_HTTP_PROXY` 和 `VAGRANT_HTTPS_PROXY` 环境变量。[更多细节在此](https://github.com/tmatilai/vagrant-proxyconf/)

**注意:** 第一次运行这个命令会需要很长时间才能完成（可能需要30分钟甚至更长时间）并且这段时间机器看起来像是什么都没做，只要没报错就不要管他直到完成。

**Windows 10 用户注意:** vagrant 在 Windows 10 有个已知的问题（见 [mitchellh/vagrant#6754](https://github.com/mitchellh/vagrant/issues/6754)）。如果 `vagrant up` 命令失败，可能是因为没有安装 Microsoft Visual C++ Redistributable。你需要[下载](http://www.microsoft.com/en-us/download/details.aspx?id=8328)缺失的包并安装。

<br/>
<br/>
<br/>
## 构建 fabric

下面这些说明假设你已经搭建好了[开发环境](devenv.md).

在本地fabric的devenv目录下运行 `vagrant ssh` 进去VM。

```
cd $GOPATH/src/github.com/hyperledger/fabric/devenv
vagrant ssh
```

在VM中，你可以编译、运行、测试fabric。

```
cd $GOPATH/src/github.com/hyperledger/fabric
make peer
```

执行以下命令查看可用命令：

```
peer help
```

你会看到以下输出：

```
    Usage:
      peer [command]

    Available Commands:
      node        node specific commands.
      network     network specific commands.
      chaincode   chaincode specific commands.
      help        Help about any command

    Flags:
      -h, --help[=false]: help for peer
          --logging-level="": Default logging level and overrides, see core.yaml for full syntax


    Use "peer [command] --help" for more information about a command.
```

`peer node start` 命令会创建一个 peer 进程，通过这个命令，可以执行其他命令。例如 `peer node status` 命令会返回正在运行的 peer的状态。全部的命令如下：

```
      node
        start       Starts the node.
        status      Returns status of the node.
        stop        Stops the running node.
      network
        login       Logs in user to CLI.
        list        Lists all network peers.
      chaincode
        deploy      Deploy the specified chaincode to the network.
        invoke      Invoke the specified chaincode.
        query       Query using the specified chaincode.
      help        Help about any command
```

**注意:** 如果 GOPATH 环境变量不止一个，chaincode必须在第一个目录里，否则会部署chaincode失败。

### 单元测试

用一下命令运行所有单元测试：

```
cd $GOPATH/src/github.com/hyperledger/fabric
make unit-test
```

用`-run RE` 标签指定要运行的测试，RE 是一个匹配测试用例名的正则表达式。用 `-v` 标签来指定运行测试的详细输。例如要执行`TestGetFoo` 测试用例，调整到包含 `foo_test.go`的目录并 and 执行：

```
go test -v -run=TestGetFoo
```

### Node.js 单元测试

你必须运行 Node.js 单元测试，以确保 Node.js client SDK 没有被你的改变破坏。按照[这里](https://github.com/hyperledger/fabric/tree/master/sdk/node#unit-tests)的说明来运行Node.js单元测试。

### Behave BDD 测试

[Behave](http://pythonhosted.org/behave/) 测试需要用不同的安全、共识配置以及校验交易正确运行的peer建立networks。为了运行这些测试：

```
cd $GOPATH/src/github.com/hyperledger/fabric
make behave
```

一些 Behave 测试在 Docker 容器里运行。如果测试失败，而你想从Docker 容器中查看log， 那么请以这个选项运行测试：

```
behave -D logs=Y
```

注意，为了直接运行 behave，你必须首先运行 `make images` 来构建比需的 `peer` 和 `member services` docker镜像。当带有一下参数运行 `go test` 时，这些镜像也可以单独构建：

```
go test github.com/hyperledger/fabric/core/container -run=BuildImage_Peer
go test github.com/hyperledger/fabric/core/container -run=BuildImage_Obcca
```

<br/>
<br/>
<br/>
## 在Vagrant之外构建

在Vagrant之外构建项目并运行peer时可能的。一般来说，就是“翻译” vagrant [安装文件](https://github.com/hyperledger/fabric/blob/master/devenv/setup.sh) 里的命令到你选择的平台上。

### 必备

* [Git client](https://git-scm.com/downloads)
* [Go](https://golang.org/) - 1.6以上
* [RocksDB](https://github.com/facebook/rocksdb/blob/master/INSTALL.md)4.1 和它的依赖
* [Docker](https://docs.docker.com/engine/installation/)
* [Pip](https://pip.pypa.io/en/stable/installing/)
* 为操作系统设置打开文件数maximum 为 10000 或者更大

### Docker

确认 Docker 守护进程初始化包括以下选项：

```
-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock
```

通常情况下, docker作为 `service` 运行，配置文件在 `/etc/default/docker`。

请注意，Docker bridge (`CORE_VM_ENDPOINT`) 可能不会在测试环境(`172.17.0.1`)当前假设的IP地址上。用`ifconfig` 或者 `ip addr`来查看 docker bridge。

### RocksDB

```
apt-get install -y libsnappy-dev zlib1g-dev libbz2-dev
cd /tmp
git clone https://github.com/facebook/rocksdb.git
cd rocksdb
git checkout v4.1
PORTABLE=1 make shared_lib
INSTALL_PATH=/usr/local make install-shared
```

### `pip`, `behave` and `docker-compose`
```
pip install --upgrade pip
pip install behave nose docker-compose
pip install -I flask==0.10.1 python-dateutil==2.2 pytz==2014.3 pyyaml==3.10 couchdb==1.0 flask-cors==2.0.1 requests==2.4.3
```

### 在 Z 上构建

为在更简单更快速的在 Z 上构建项目，提供了[脚本](https://github.com/hyperledger/fabric/tree/master/devenv/setupRHELonZ.sh) (这跟vagrant的[安装文件](https://github.com/hyperledger/fabric/blob/master/devenv/setup.sh) 很相似)。该脚本只在 RHEL 7.2 上测试过，并且有些假设(防火墙设置，以root用户开发等)。不过，在个人分配的VM实例中他已经足够开发使用了。

从新安装的OS开始：

```
sudo su
yum install git
mkdir -p $HOME/git/src/github.com/hyperledger
cd $HOME/git/src/github.com/hyperledger
git clone http://gerrit.hyperledger.org/r/fabric
source fabric/devenv/setupRHELonZ.sh
```

从这之后，你可以像上文 Vagrant 开发环境描述的那样继续操作：

```
cd $GOPATH/src/github.com/hyperledger/fabric
make peer unit-test behave
```

### 在 Power 平台构建

在 Power (ppc64le) 系统上开发构建也是在 vagrant 之外操作。为了便于在Ubuntu上安装开发环境，以root权限调用 [脚本](https://github.com/hyperledger/fabric/tree/master/devenv/setupUbuntuOnPPC64le.sh)，该脚本已在Ubuntu 16.04 上验证，并且预设了一些东西(如，开发库，防火墙设置等) ，总的来说可以是临时的.

为启动安装Ubuntu的 Power 服务，首先确认正确的设置 Host的 [GOPATH environment variable](https://github.com/golang/go/wiki/GOPATH)，然后执行下面的命令来编译fabric代码：

```
mkdir -p $GOPATH/src/github.com/hyperledger
cd $GOPATH/src/github.com/hyperledger
git clone http://gerrit.hyperledger.org/r/fabric
sudo ./fabric/devenv/setupUbuntuOnPPC64le.sh
cd $GOPATH/src/github.com/hyperledger/fabric
make dist-clean all
```

### 在OSX系统上构建

首先，[安装Docker](https://docs.docker.com/engine/installation/mac/)。
数据库默认写到 /var/hyperledger。你可以修改此路径在 `core.yaml` 配置文件的`peer.fileSystemPath`。

```
brew install go rocksdb snappy gnu-tar     # For RocksDB version 4.1, you can compile your own, as described earlier

# You will need the following two for every shell you want to use
eval $(docker-machine env)
export PATH="/usr/local/opt/gnu-tar/libexec/gnubin:$PATH"

cd $GOPATH/src/github.com/hyperledger/fabric
make peer
```

<br/>
<br/>
<br/>
## 配置

配置操作使用 [viper](https://github.com/spf13/viper) 和 [cobra](https://github.com/spf13/cobra)库。

**core.yaml** 文件包含peer的配置，很多配置可以通过命令行中与之对应的的环境变量覆盖，这些环境变量的前缀 *'CORE_'*。例如，日志级别通过下面的环境变量操作：

    CORE_PEER_LOGGING_LEVEL=CRITICAL peer


<br/>
<br/>
<br/>
## 日志

日志操作使用 [go-logging](https://github.com/op/go-logging) 库。 

可用的日志级别依序增长如下： *CRITICAL | ERROR | WARNING | NOTICE | INFO | DEBUG*

请查看 [详细日志操作](https://github.com/hyperledger/fabric/blob/master/docs/Setup/logging-control.md)。