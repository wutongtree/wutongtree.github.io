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
* **可扩展性**。负责特定链码的背书节点和批准节点是正交的，这比所有功能都在同一个节点完成更好扩展。尤其是，当最终不同的链码都指定不同的背书节点时，链码会在不同的背书节点上分散开，它们就可以并行执行了，链码执行这里叫背书（endorsement）。另外，链码执行是非常耗时的，已经从共识服务的关键路径中移除了。
* **机密性**。这个架构使对内容和执行状态更新有*机密性*要求的链码部署更容易了。
* **共识模块性**。这个架构是*模块化的*，运行插件化的共识实现。

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

* *部署交易* 用程序作为它的一个参数创建新的链码。当部署交易成功执行以后，我们就说链码被安装到“链上”了。
* *调用交易* 在前面部署的链码上执行一个操作。一个调用交易指的是链码和它提供的功能。如果成功的话，链码会执行指定的功能，可能会修改相关的状态，然后返回输出。

后面还会介绍，部署交易是调用交易的特殊情况，调用交易创建新的链码就是在系统链码上的调用交易。

**注意**：*这个文档假设一个交易创建新的链码或者调用交易都是在已经部署的链码上操作的。这个文档没有描述如下内容：a) 支持跨链码的交易; b) 查询交易（只读的）的优化*。

### 1.2. 状态

