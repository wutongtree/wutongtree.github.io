---
layout: page
title: "[翻译]Next Consensus Architecture Proposal"
description: ""
---
{% include JB/setup %}

原文：<https://github.com/hyperledger/fabric/blob/master/proposals/r1/Next-Consensus-Architecture-Proposal.md>

作者: Elli Androulaki, Christian Cachin, Angelo De Caro, Konstantinos Christidis, Chet Murthy, Binh Nguyen, Alessandro Sorniotti, and Marko Vukolić

翻译：[梧桐树](https://wutongtree.com)

---

这篇文档记录的是区块链基础架构，区块链节点根据角色分为*peers*（维护状态/总账的节点）和*consenters*（批准区块链状态中交易顺序的节点）。在通常的区块链架构中（包括2016年7月的Hyperledger fabric），这些角色都是集成的（参看Hyperledger fabic的*validating peer*）。这个架构还引入了背书节点（endorsers）的概念，它其实是一种特殊类型的节点，用来模拟执行和背书交易（大概和Hyperldger fabric 0.5-developer-preview的执行/验证交易是类似的）。

这种架构和peers/consenters/endorsers集成的设计相比有如下的优势：

* **链码的可信灵活性（chaincode trust flexibility）**。从架构上，区分开链码（区块链应用程序）是*可信*的*假设*和共识是不可信的假设。就是说，参与到共识服务中的节点可能是批准节点（consenters），也能容忍一些节点失败或者捣乱。每个链码的背书节点也可能是不一样的。
* **可扩展性（Scalability）**。负责特定链码的背书节点和批准节点是正交的，这比所有功能都在同一个节点完成更好扩展。尤其是，当最终不同的链码都指定不同的背书节点时，链码会在不同的背书节点上分散开，它们就可以并行执行了，链码执行这里叫背书（endorsement）。另外，链码执行是非常耗时的，已经从共识服务的关键路径中移除了。
* **机密性（Confidentiality）**。这个架构使对内容和执行状态更新有*机密性*要求的链码部署更容易了。
* **共识模块性（Consensus modularity）**。这个架构是*模块化的*，运行插件化的共识实现。

## 目录
1. 系统架构
1. 基本的交易背书工作流
1. 背书策略
1. 区块链数据结构
1. 状态转换和检查点
1. 机密性

---

## 1. 系统架构
区块链是由很多相互通信的节点组成的分布式系统。区块链运行程序（叫链码），保存状态和总账数据，执行交易。链码是最核心的元素，交易是在链码上执行的操作，并且只有链码才能更改状态。交易必须要有背书，只有有背书的交易才能被提交，才能影响状态。有一些具有管理功能和参数的特殊链码，统称*系统链码*。

### 1.1. 交易

交易有两种类型：

* *部署交易（Deploy transactions）* 用程序作为它的一个参数创建新的链码。当部署交易成功执行以后，我们就说链码被安装到“链上”了。
* *调用交易（Invoke transactions）* 在前面部署的链码上执行一个操作。一个调用交易指的是链码和它提供的功能。如果成功的话，链码会执行指定的功能，可能会修改相关的状态，然后返回输出。

后面还会介绍，部署交易是调用交易的特殊情况，调用交易创建新的链码就是在系统链码上的调用交易。

**注意**：*这个文档假设一个交易创建新的链码或者调用交易都是在已经部署的链码上操作的。这个文档没有描述如下内容：a) 支持跨链码的交易; b) 查询交易（只读的）的优化*。

### 1.2. 状态（State）

**区块链状态（Blockchain state）**。区块链的状态（世界状态：world state）有一个简单的结构，建模成了一个带版本控制的KV存储（KVS），其中键是名称，值是任意的二进制大对象。这些数据由运行在区块链上的链码通过`put`和`get`的KVS进行操作。这些状态是被永久存储的，更新状态也会写日志。注意带版本控制的KVS只是状态的模型，实现方式可以实际的KVS系统，也可以是关系型数据库系统或者其他的解决方案。

形式化的表示就是，区块链状态`s`是`K -> (V X N)`映射的一个元素，其中：

* `K`是键的集合
* `V`是值的集合
* `N`是无数个有序的版本号集合。单射函数`next: N -> N`有输入`N`的一个元素，返回下一个版本号。

`V`和`N`都有一个特殊的元素`\bot`，代表`N`最小的元素。初始化的时候，所有的键都被映射到`(\bot,\bot)`。`s(k)=(v,ver)`这个表达式中，`v`用`s(k).value`来表示，`ver`用`s(k).version`来表示。

KVS操作是这样建模的：

* `put(k,v)`操作。对`k\in K`和`v\in V`键值对，区块链状态`s`的新状态`s'`计算方法是：`s'(k)=(v,next(s(k).version))`。并且对所有的`k'!=k`，表达式`s'(k')=s(k')`都成立。
* `get(k) `操作。返回`s(k)`。

**状态分区（State partitioning）**。KVS中的键可以通过名称就能识别出它们属于哪个链码，所以只有特定链码的交易才能修改属于这个链码的键。原则上，任意的链码都能读取属于其他链码的键（当然机密链码的状态是不能读的，参考第6部分）。*修改2个或者多个链码状态的跨链交易，以后会支持。*

**总账（Ledger）**。区块链状态的变化过程（历史）是保存在*总账*中的。总账是交易区块的散列链（hashchain），总账中的交易是全序的。

区块链状态和总账会在第4部分详细描述。

### 1.3. 节点（Nodes）

节点是区块链的通信实体。节点是一个逻辑的概念，不同类型的节点是可以运行在同一个物理服务器上的。重要的是节点是怎么被分组成“信任域（trust domains）”，怎么和控制它们的逻辑实体关联的。

有3种类型的节点：

1. **客户端（Client）**或者**提交客户端（submitting-client）**：提交实际交易请求的客户端。

2. **伙伴（Peer）**：一个提交交易，维护状态和总账副本的节点。伙伴有两种特殊的角色：
 
    a. **提交伙伴（submitting peer）**或者**提交者（submitter）**

    b. **背书伙伴（endorsing peer）**或者**背书者（endorser）**

3. **共识服务节点（Consensus-service-node）**或者**批准者（consenter）**：一个运行了有送达保证（delivery guarantee，比如原子广播）通信服务的节点，送达保证典型的实现方法是运行共识服务。

注意，批准者和客户端是不维护总账和区块链状态的，只有伙伴才会。

节点的类型下面会详细解释。

#### 1.3.1. 客户端（Client）

客户端表示的是代表终端用户的实体，它必须连接到伙伴才能和区块链通信。客户端可以根据自己的选择连接到任意一个伙伴上，然后创建再调用交易。

#### 1.3.2. 伙伴（Peer）

伙伴通过共识服务通信维护区块链状态和总账。它们从共识服务接收有序的更新状态，然后更新本地维护的状态。

伙伴可以选择下面描述的两种角色：

* **提交伙伴（Submitting peer）**。*提交伙伴*是一种特殊的角色，它给客户端提供接口，这样客户端就可以连接到提交伙伴调用交易和获取结果。这个伙伴代表一个或者多个客户端和其他节点通信来执行交易。

* **背书伙伴（Endorsing peer）**。*背书伙伴*的特殊功能是对特定的链码，在其提交交易前对它进行*背书*。每个链码都可以指定一个*背书策略*，可能会涉及背书伙伴的集合。策略会定义一个有效的交易背书（典型情况是背书着的签名集合）的充要条件，这会在第2和3部分描述。一个特殊情况是，安装新链码的部署交易中，（部署）背书策略是系统链码的背书策略指定的。

强调一个伙伴同时有提交伙伴和背书伙伴角色的时候，就叫它*交付伙伴（committing peer）*。

#### 1.3.3. 共识服务节点（Consensus service node (Consenters)）

*批准者*组成了*共识服务*，比如，一个提供交付保证的通信组织。共识服务可以有多种实现方式：从中心化的服务（比如：部署和测试）到目标是不同网络和节点容错模型的分布式协议。

伙伴是共识服务的客户端，在于共识服务给它提供了一个有广播交易信息的共享*通信通道*。伙伴连接到通道上，可以发送或者接收消息。通道支持所有消息的*原子*交付，就是，消息通信是全序交付的和（跟实现相关）可靠的。换句话说，通道输出给所有连接的节点相同的消息，而且输出的逻辑顺序是相同的。原子通信保证又叫*全序广播（total-order broadcast）*，*原子广播（atomic broadcast）*，或者分布式系统环境下的*共识（consensus）*。通信过的消息就是会保存在区块链状态中的候选交易了。

**共识通道分区（Partitioning (consensus channels)）**。共识服务可能支持多*通道*，类似发布/订阅（pub/sub）消息系统的*主题*。客户端链接到一个指定的通道，就可以发送或者获取到达的消息。通道可能会有分区 - 客户端连接到一个通道是不知道其他通道的存在的，但是客户端可以连接到多个通道。为简单起见，本文档后面的部分，除非明确的提到了的其他情况，我们都假设共识服务是有单个通道/主题组成的。

**共识服务API（Consensus service API）**。伙伴连接到共识服务提供的通道，是通过共识服务的API。共识服务API有两种基本的操作（更通用的叫*异步事件*）：

* `broadcast(blob)`：提交伙伴调用它在通道上广播任意的消息`blob`。这在BFT中，给服务发送一个请求，又叫`request(blob)`。

* `deliver(seqno, prevhash, blob)`：共识服务调用它给伙伴发送带非负序列号`seqno`和最近一次发送发送消息的hash`prevhash`的消息`blob`。换句话说，它是共识服务的输出事件。`deliver()`在发布/订阅系统中叫`notify()`，BFT系统中叫`commit()`。

注意共识服务客户端（比如伙伴）只通过`broadcast()`和`deliver()`事件和服务进行交互。

**共识内容（Consensus properties）**。共识服务（或者原子广播通道）有如下的保证，回答了这些问题：`广播消息发生了什么`，`交付消息间有什么关系`?

1. **安全性 - 一致性保证（Safety (consistency guarantees)）**：只要伙伴连接到通道有足够的时间（它们可以断开或者宕掉，重启或者重新连接就可以），它们就能看到`相同`顺序的交付消息`(seqno, prevhash, blob)`。这意味着，给所有伙伴的输出（`deliver()`事件）都是`相同顺序`的，相同的序号都是`相同的内容`（`blob`和`prevhash`）。需要注意的是，这只是一个`逻辑顺序`，并且，给一个伙伴的`deliver(seqno, prevhash, blob)`并不需要和给另外一个伙伴输出的`deliver(seqno, prevhash, blob)`有时间上的关联。换句话说，给定一个特定的`seqno`，`没有`两个正常的伙伴会发送`不同`的`prevhash`和`blob`。而且是，除非有共识客户端（伙伴）真正的调用了`broadcast(blob) `，是不会发送`blob`消息的，最好呢，只发送`一次`每个广播的`blob`。

还有，`deliver()`事件包含了上一个`deliver()`事件的加密哈希`prevhash`。当共识服务执行一个原子广播保证时，`prevhash`是序号为`seqno-1`的`deliver()`事件的加密哈希。这就在不同的`deliver()`的事件之间建立了一个哈希链，能够用来帮助验证共识输出的完整性，后面的第4和5部分会讨论这个。特殊情况是，第一个`deliver()`事件的`prevhash`有一个默认值。

2. **活跃度 - 交付保证（Liveness (delivery guarantee)）**：共识服务的活跃度保证是共识服务的实现指定的。精确的保证要依赖网络和节点容错模型。

原则上，如果提交没有失败，共识服务应该保证每个连接到共识服务的伙伴最终都能交付每个提交的交易。

总结一下，共识服务保证了下面的内容：

* *协议（Agreement）*。正常伙伴的任意两个事件，`deliver(seqno, prevhash0, blob0)`和`deliver(seqno, prevhash1, blob1)`，如果有相同的`seqno`，则有`prevhash0==prevhash1`，`blob0==blob1`成立；

* *哈希链完整性（Hashchain integrity）*。正常伙伴的任意两个事件，`deliver(seqno-1, prevhash0, blob0)`和`deliver(seqno, prevhash, blob)`，有`prevhash = HASH(seqno-1||prevhash0||blob0)`；

* *不遗漏（No skipping）*。如果一个共识服务给一个正常节点`p`输出`deliver(seqno, prevhash, blob)`，如果`seqno>0`，则`p`一定已经交付了`deliver(seqno-1, prevhash0, blob0)`事件；

* *不创建（No creation）*。一个正常节点的任意`deliver(seqno, prevhash, blob)`事件，前面一定有一个伙伴发送了`broadcast(blob)`事件；

* *不重复（No duplication，可选的）*。对任意的两个事件`broadcast(blob)`和`broadcast(blob')`，当正常伙伴交付了两个事件，`deliver(seqno0, prevhash0, blob)`和`deliver(seqno1, prevhash1, blob')`，如果`blob`==`blob'`，则有`seqno0==seqno1`和`prevhash0==prevhash1`成立；

* *活跃度（Liveness）*。如果一个正常伙伴产生了`broadcast(blob)`事件，则每个正常伙伴“最终”都会发出一个`deliver(*, *, blob)`事件，其中`*`代表任意值。

## 2. 交易背书的基本流程

下面我们概要性的介绍一个交易的高层请求流程。

**备注**：*注意后面的协议并不假定所有的交易都是确定性的，允许不确定性的交易*。

### 2.1. 客户端创建一个交易并自己选择发送给提交伙伴

要调用一个交易，客户端发送如下的消息给提交伙伴`spID`。

`<SUBMIT,tx,retryFlag>`，其中：

- `tx=<clientID,chaincodeID,txPayload,clientSig>`，其中：
    - `clientID`是提交客户端的ID，
    - `chaincodeID`指的是交易所属的链码，
    - `txPayload`是提交的交易本身的有效载荷，
    - `clientSig`是客户端`tx`消息其他项的签名。
- `retryFlag`是一个布尔值，告诉提交伙伴万一交易失败了要不要重传，

调用交易和部署交易的`txPayload`是不一样的（比如，调用交易会引用一个部署相关的系统链码）。如果是**调用交易**，`txPayload`只有一个项：

- `invocation = <operation, metadata>`，其中：
    - `operation`代表链码的操作（函数）和参数，
    - `metadata`代表调用相关的属性。

如果是**部署交易**，会有两个项：

- `chainCode = <source, metadata>`，其中：
    - `source`代表链码的源码路径，
    - `metadata`代表链码和应用相关的属性。
- `policies`包含了所有伙伴都能访问的链码策略，比如背书策略。

TODO：确定是否要在客户端显示的包含本地/逻辑时间（时间戳）。

### 2.2. 提交伙伴准备一个交易并发送给背书者获取背书
