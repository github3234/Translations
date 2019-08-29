# 私有数据

## 什么是私有数据？

在某个通道上的一组组织需要对该通道上的其他组织保持数据私密的情况下，它们可以选择创建一个新通道，其中只包含需要访问数据的组织。但是，在每一种情况下创建单独的通道会产生额外的管理开销（维护链码版本、策略、MSP 等），并且不允许在保持部分数据私有的同时，让所有通道参与者都看到交易。

这就是为什么从v1.2开始，Fabric就提供了**创建私有数据集合**的能力，这使得通道上定义的组织子集能够背书、提交或查询私有数据，而不必创建单独的通道。

## 什么是私有数据集？

集合是两个元素的组合：

1. **实际的私有数据**，通过 [Gossip 协议](../gossip.html)点对点地发送给授权可以看到它的组织。私有数据保存在被授权的组织的节点上的私有数据库上（有时候成为“侧”数据库或者“SideDB”），它们可以被授权节点的链码访问。排序节点不能影响这里也不能看到私有数据。注意，由于 gossip 以点对点的方式向授权组织分发私有数据，所以必须设置通道上的锚节点，也就是每个节点上的 CORE_PEER_GOSSIP_EXTERNALENDPOINT 配置，以此来引导跨组织的通信。

2. **该数据的哈希值**，该哈希值被背书、排序之后写入通道上每个节点的账本。哈希值作为交易的证明用于状态验证，还可用于审计。

下图说明了一个被授权拥有私有数据的节点和一个未授权的节点的账本内容。

![private-data.private-data](./PrivateDataConcept-2.png)

如果集合成员陷入争议，或者如果他们想将资产转让给第三方，则可以决定与其他方共享私有数据。然后，第三方可以计算私有数据的哈希，并查看它是否与通道账本上的状态匹配，从而证明在某个时间点上，集合成员之间存在该状态。

### 什么时候在通道内使用集合和单独的通道

* 当所有交易（账本）都必须在通道成员的组织之中保密时，使用**通道**比较合适。

* 当交易（和账本）必须共享给一部分组织，并且只有其中的一部分组织可以在交易中使用这些数据的一部分（或者全部）时，使用**集合**比较合适。此外，由于私有数据是点对点传播的，而不是通过块传播的，所以在交易数据必须对排序服务节点保密时，应该使用私有数据集合。

## 解释集合的用例

设想一个由五个交易农产品的组织组成的通道：

* **农民**向国外售卖他的货物
* **分销商**将货物运往国外
* **托运商**负责参与方之间的货物运输
* **批发商**从分销商采购商品
* **零售商**从托运人和批发商采购商品

**分销商**可能希望与**农民**和**托运商**之间保持私密交易，以对**批发商**和**零售商**保密交易条款（以免暴露他们收取的加价）。

**分销商**还可能希望与**批发商**建立单独的私有数据关系，因为它向批发商收取的价格比**零售商**低。

**批发商**还可能希望与**零售商**和**托运商**建立私有数据关系。

与其为这些关系定义许多小通道，不如定义多个私有数据集合（**PDC**）在以下各方之间共享私有数据：

1. PDC1：**分销商**，**农民**和**托运商**
2. PDC2：**分销商**和**批发商**
3. PDC3：**批发商**、**零售商**和**托运商**

![private-data.private-data](./PrivateDataConcept-1.png)

使用此示例，**分销商**拥有的节点在其账本中拥有多个私有数据库，其中包括来自**分销商**、**农民**和**托运商**关系以及**分销商**和**批发商**关系的私有数据。因为这些数据库与保存通道账本的数据库是分开的，所以私有数据有时被称为“侧数据库（SideDB）”。

![private-data.private-data](./PrivateDataConcept-3.png)

## 私有数据的交易流程

当在链码中引用私有数据集合时，为了保护私有数据的机密性，在提案、背书的和提交交易到账本时，交易流程略有不同。

关于不使用私有数据的交易流程的详细信息，请参阅我们关于[交易流程](../txflow.html)的文档。

1. 客户端应用程序提交一个提案请求，让属于授权集合的背书节点调用链码函数（读取或写入私有数据）。私有数据，或用于在链码中生成私有数据的数据，被发送到提案的 `transient` 字段中。

2. 背书节点模拟交易，并将私有数据存储在临时数据存储（节点的本地临时存储）中。它们根据组织集合的策略将私有数据通过 [gossip](../gossip.html) 分发给授权的节点。

3. 背书节点将提案响应发送回客户端。提案响应中包含经过背书的读写集，该读写集中包含了公共数据和私有数据键和值的哈希。*私有数据不会被发送回客户端*。更多关于带有私有数据的背书的信息，请查看[这里](../private-data-arch.html#endorsement)。

4. 客户端应用程序将交易（包含带有私有数据哈希值的提案响应）提交给排序服务。带有私有数据哈希的交易同样被包含在区块中。带有私有数据哈希的区块被分发给所有节点。这样，通道中的所有节点就可以以同样的方式来验证带有私有数据哈希值的交易，但不需要知道实际的私有数据。

5. 在区块提交的时候，授权节点会根据集合策略来决定它们是否有权访问私有数据。如果是，它们就先检查它们本地临时数据存储，以确定它们在链码背书的时候是否已经接收到了私有数据。如果没有收到数据，它们会尝试从其他授权节点拉取私有数据。然后它们会根据公共区块上的哈希值验证私有数据并将交易提交到账本。当验证和提交结束后，私有数据会被移动到他们私有数据库和私有读写存储的副本中。然后私有数据会从`临时数据存储`中删除。

## 清除私有数据

对于非常敏感的数据，即使共享私有数据的各方也可能希望（或政府法规要求）定期“清除”节点的数据，留下区块链上的数据哈希作为私有数据不可篡改的证据。

在某些情况下，私有数据只需要存在于节点的私有数据库上，直到可以将其复制到区块链外部的数据库中为止。数据也可能只需要存在于节点上，直到使用它完成链码业务流程（交易结算、合约履行等）。

为了支持这些情况，如果已经持续 N 个块都没有修改私有数据了则可以清除它，N 是可配置的。清除的私有数据不能从链码查询，并且其他节点也请求不到。

## 私有数据集合怎么定义

有关集合定义的更多详细信息，以及关于私有数据和集合的其他更底层的信息，请参阅[私有数据主题](../private-data-arch.html)。

<!--- Licensed under Creative Commons Attribution 4.0 International License
https://creativecommons.org/licenses/by/4.0/ -->