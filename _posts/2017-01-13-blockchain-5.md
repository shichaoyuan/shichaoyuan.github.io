---
layout: post
title: 区块链技术 5 - fabirc Next-Consensus-Architecture-Proposal [翻译]
date: 2017-01-13 15:00:00 +08:00
tags: blockchain
---

# Hyperledger fabirc 下一代共识架构提案

[原文链接](https://github.com/hyperledger/fabric/blob/master/proposals/r1/Next-Consensus-Architecture-Proposal.md)

作者： Elli Androulaki, Christian Cachin, Konstantinos Christidis, Chet Murthy, Binh Nguyen, and Marko Vukolić

本文记载了区块链基础设施的架构，区块链中节点的角色被划分为 *peers* 角色（维护状态和账本）和 *orderers* 角色（对账本中交易的顺序达成共识）。在通常的区块链架构（包括 Hyperledger fabric v0.6 以及更早的版本）中这些角色是统一的（例如 Hyperledger fabric v0.6 中的 *validating peer* ）。该架构同时引入了 *endorsing peers* (endorsers) ，这种类型的 peers 负责模拟交易执行和签署交易（大致对应 HL fabric 0.6 中的执行交易）。

该架构相对于 peers/orderers/endorsers 统一的设计（比如 HL Fabirc v0.6）具有以下优势：

* **链码信任的灵活性(Chaincode trust flexibility).** 该架构将链码（区块链应用）的信任假设和共识顺序的信任假设分离开来。换句话说，ordering service 可以由一组节点(orderers)来提供，容忍部分节点故障或者有恶意行为，而每个链码对应的 endorsers 可以不同。

* **可扩展性(Scalability).** 由于负责特定链码的 endorser 节点与 orderers 节点是独立的，所以系统具有更好的*扩展性*。尤其是当不同的链码指定了不同的 endorsers 集合的时候，这样就使得链码在 endorsers 之间分离，所以链码就可以并行地执行(endorsement)。另外还将耗时的链码执行过程从 ordering service 过程中分离出来。

* **保密性(Confidentiality).** 该架构促进了具有*保密性*需求的链码的开发，保密性需求涉及交易的内容和状态更新。

* **共识模块化(Consensus modularity).** 该架构是*模块化*的，并且允许可插拨的共识（也就是 ordering service）实现。

该架构推动 Hyperledger Fabirc v0.6之后版本的开发，正如下面所提到的，某些特性包含在 Hyperledger Fabric v1 中，另外的延期到 v1 之后的版本。

## 目录

**Part I: Hyperledger Fabric v1 相关的架构元素**

1. 系统架构
1. 交易签署的基本工作流程
1. 签署策略

	**Part II: v1 版本之后的架构元素**

1. 账本检查点（剪枝）


---

## 1. 系统架构

区块链是一个由许多相互通信的节点组成的分布式系统。区块链运行着称为链码的程序，维护状态和账本数据，执行交易。由于交易就是链码中调用的操作，所以链码是架构的关键元素。交易必须被签署，只有签署的交易才能被提交，影响链上的状态。可能会存在一个或多个特定的链码来管理方法和参数，这些链码统称为*系统链码*。


### 1.1. 交易

交易有两种类型：

* *部署交易* 创建新的链码，以参数的形式接收程序。当部署交易执行成功后，链码就被安装到了区块链上。

* *调用交易* 在之前部署的链码上下文中执行操作。某个调用交易指向某个链码以及其提供的某个方法。当交易成功的时候，链码执行指定的方法——可能涉及修改相应的状态，返回一个输出。

就像后面描述的那样，部署交易是调用交易的特例，部署交易创建新的链码对应于系统链码上的调用交易。

**注：** *该文档假设某个交易要么创建新的链码，要么调用部署的链码提供的某个操作。该文档没有描述：a) 查询交易（只读）的优化（包含在v1版本中）， b) 跨链码交易的支持（v1之后版本提供）*

### 1.2. 区块链数据结构

#### 1.2.1. 状态

区块链最新的状态（简单来说，*状态*）被模型化为带版本的键值存储（KVS），其中键是名字，值是任意数据。这些条目由链码通过`put`和`get`方法操作。状态被持久地存储，对状态的更新被记录在日志中。注意，带版本的KVS作为状态模型，但是具体的实现可是实际的KVS，也可以是RDBMS或者其它解决方案。

更正式化地讲，状态 `s` 被模型化为映射 `K -> (V X N)` 的某个元素，其中：

* `K` 是键的集合
* `V` 是值得集合
* `N` 是版本号的无限有序集合。单射函数 `next: N -> N`，输入 `N` 的一个元素，返回下一个版本号。

`V` 和 `N` 都包含一个特殊元素 `\bot`，表示 `N` 的最小的元素。初始状体，所有的键都映射到 `(\bot,\bot)`。比如 `s(k)=(v,ver)`，我们用 `v` 表示 `s(k).value`, 用 `ver` 表示 `s(k).version`。

KVS 操作被模型化为：

* `put(k,v)`，对于 `k \in K` 和 `v \in V`，取出状态 `s` 将其改为 `s'`，使得 `s'(k)=(v,next(s(k).version))` 并且对于所有 `k'!=k`，`s'(k')=s(k')`。   
* `get(k)` 返回 `s(k)`.

状态由 peers 维护，而不是 orderers 和 clients。

**状态分割.** 根据KVS中键的名称就能识别出它们属于哪个链码，从某种意义上说，只有属于某个特定链码的交易才能改变属于它自己的键。原则上，某个链码可以读取其他链码的键值。 *对于跨链码交易的支持，修改两个或更多链码状态是 v1 版本之后的特性。*

#### 1.2.2 账本

账本提供一个可验证的历史记录，包括系统执行过程中所有成功的状态更改（也就是*有效的*交易）和不成功的状态更改（也就是*无效的*交易）。

账本由 ordering service 构建，形式为交易的*区块*组成的全序哈希链。哈希链确保账本中区块的全序排列，每个区块包含一组全序排列的交易。这就确保了所有交易的全序排列。

账本保存在所有的 peers 中，可选地保存在部分 orderers 中。在 orderer 上下文中，我们将账本称为 `OrdererLedger`，在 peer 上下文中，我们将账本称为 `PeerLedger`。`PeerLedger` 不同于 `OrdererLedger` 之处在于 peers 本地维护着一个标记交易是否有效的位图。

Peers 可以修剪 `PeerLedger` （v1 之后版本的特性）。Orderers 维护 `OrdererLedger` 用于容错和可用性，可以在任意时间决定修剪账本，前提是 ordering service 是可靠的。

账本允许 peers 重新执行所有交易的历史记录来构建状态。因此1.2.1节描述的状态是可选的数据结构。

### 1.3. 节点

节点是区块链的通信实体。“节点”仅仅是一个逻辑概念，在这种意义上，不同类型的多个节点可以运行在同一物理服务器上。关键是节点如何组织可信域，与控制它们的逻辑实体相关联。

有三类节点：

1. **Client** 或者 **submitting-client**：向 endorsers 提交实际的交易调用，广播交易提案到 ordering service。

1. **Peer**：提交交易、维护状态和账本副本。另外 peers 可以具有特别的 **endorser** 角色。

1. **Ordering-service-node** 或者 **orderer**：运行通信服务，该服务实现投递保证，比如原子或者全序广播。

节点的类型将在下文中进一步解释。

#### 1.3.1. Client

client 相当于代表终端用户操作的实体。它必须连接某个 peer 才可以与区块链通信。client 可以连接任意可选的 peer。clients 创建和调用交易。

第2节将会详细介绍，clients 既能与 peers 通信，也能与 ordering service 通信。

#### 1.3.2. Peer

peer 从 ordering service 接收*区块*形式的有序的状态更新，维护状态和账本。

另外 peers 可以分配特别的 **endorsing peer** 或 **endorser** 角色。*endorsing peer* 的功能与特定的链码相关联，在交易提交前*签署*交易。每个链码可以指定 *endorsement policy*。策略定义了有效交易签署（通常是一组 endorsers 的签名）的必要和充分条件。对于部署交易这种特殊情况，endorsement policy 针对系统链码。

#### 1.3.3. Ordering service 节点  (Orderers)

*orderers* 节点组成 *ordering service*，也就是提供投递保障的通信结构。可以以多种方式实现 ordering service：从中心化服务（比如开发测试时使用）到分布式协议，针对不同的网络和节点故障模型。

ordering service 为 clients 和 peers 提供一个共享的*通信信道*，为包含交易的消息提供广播服务。clients 连接到信道广播消息，消息将会投递到所有 peers。信道支持所有消息的*原子性*投递，也就是具有全序投递和可靠性（特定于实现）的消息通信。换句话说，信道为所有连接的 peers 输出相同的消息，并且消息具有相同的逻辑顺序。这种原子性通信保障也被称为*全序广播*、*原子广播*或者分布式系统中的*共识*。通信的消息是区块链状态的候选交易。

**分区（ordering service 信道）.** ordering service 可以支持多个*信道*，类似于发布/订阅（pub/sub）消息系统中的*主题*。clients 可以连接到某个信道，然后发送消息，获取到达的消息。信道可以被看作分区——连接到某个信道的 clients 不知道其他信道的存在，但是 clients 可以连接多个信道。即使 Hyperledger Fabric v1 中实现的某些 ordering service 将会支持多信道，但是为了简化描述，下文中假设 ordering service 仅有单个信道。

**Ordering service API.** peers 通过 ordering service 提供的接口连接到其提供的信道。ordering service API 包含两个基本操作（更一般地说 *异步事件*）：

**待办事项** 加入通过序列号获取特定区块的 API。

* `broadcast(blob)`：clients 调用该操作通过信道广播任意消息 `blob`。当向服务发送请求时，该操作在 BFT 算法中也被称为 `request(blob)`。

* `deliver(seqno, prevhash, blob)`：ordering service 调用 peer 上的这个操作来投递消息 `blob`，附带特定的非负整数序列号（`seqno`）和上一次投递的消息的哈希（`prevhash`）。换句话说，这是 ordering service 的输出事件。`deliver()` 也被称作 pub-sub 系统中的 `notify()` 和 BFT 系统中的 `commit()`。

**Ledger and block formation.** 账本包含 ordering service 输出的所有数据。简而言之，账本就是一系列 `deliver(seqno, prevhash, blob)` 事件，通过 `prevhash` 连成一条哈希链。

大多数时候，为了系统的效率 ordering service 将会打包（批处理）消息，在单个 `deliver` 事件中输出*区块*，而不是单个交易。在这种情况下，ordering service 必须保证区块内消息的确定性的顺序关系。区块中消息的数量可以由 ordering service 动态决定。

在下文中为了简化表述，我们假设每个 `deliver` 事件投递一个交易。这很容易扩展到区块，基于上文提到的区块中消息的确定性顺序关系，将区块的 `deliver` 事件设想为一系列单独消息的 `deliver` 事件。

**Ordering service 属性**

ordering service（原子广播信道）的保证规定了广播信息的行为，以及投递的消息之间的关系。这些保证包括：

1. **Safety**：只要 peers 连接到信道足够长的时间周期（可能会断掉连接或者崩溃，但是将会重启或重连），peers 将会获得*一致的* `(seqno, prevhash, blob)` 消息序列。这意味着，输出（`deliver()` 事件）在所有的节点上以*相同的顺序*发生，基于序列号携带*一致的内容*（`blob` 和 `prevhash`）。注意这仅仅是*逻辑顺序*，某个 peer 上的 `deliver(seqno, prevhash, blob)` 事件与其它 peer 上的同一事件没有实时的关系。换句话说，给定某个 `seqno`，*不存在*两个正常的 peer 给出*不同*的 `prevhash` 或 `blob`。此外，除非某个 client 调用了 `broadcast(blob)`，否则不会有 `blob` 被投递，最好每次广播的消息只被投递*一次*。

	而且 `deliver()` 事件包含前一事件的密码学哈希（`prevhash`）。当 ordering service 实现原子性广播保证时，`prevhash` 是序列号为 `seqno-1` 的 `deliver()` 事件的密码学哈希。这就为 `deliver()` 事件序列建立了一条哈希链，可以用来验证 ordering service 输出的完整性。对于第一个 `deliver()` 事件，`prevhash` 为默认值。

1. **Liveness**：ordering service 的 Liveness 保证由特定的实现指定。具体的保证依赖网络和节点故障模型。

	原则上，如果 client 没有故障，那么 ordering service 应该保证每个正常的 peer 最终会收到所有提交的交易。

总的来说 ordering service 保证下述属性：

* *一致性（Agreement）.* 对于正常的 peers 中的任意两个事件 `deliver(seqno, prevhash0, blob0)` 和 `deliver(seqno, prevhash1, blob1)`，如果具有相同的 `seqno`，那么 `prevhash0==prevhash1` and `blob0==blob1`；
* *哈希链完整性（Hashchain integrity）.*  对于正常的 peers 中的任意两个事件 `deliver(seqno-1, prevhash0, blob0)` 和 `deliver(seqno, prevhash, blob)`，那么 `prevhash = HASH(seqno-1||prevhash0||blob0)`.
* *没有跳过（No skipping）*. 如果 ordering service 对某个正常的 peer *p* 投递了 `deliver(seqno, prevhash, blob)`，并且 `seqno>0`，那么 *p* 已经收到了事件 `deliver(seqno-1, prevhash0, blob0)`。
* *没有创造（No creation）*. 某个正常 peer 上任意 `deliver(seqno, prevhash, blob)` 事件一定是由某个节点（可能是不同的节点）上的 `broadcast(blob)` 事件产生的；
* *没有重复（No duplication）（可选的，但是需要）*. 对于任意两个事件 `broadcast(blob)` 和 `broadcast(blob')`，当两个事件 `deliver(seqno0, prevhash0, blob)` 和 `deliver(seqno1, prevhash1, blob')` 发生在正常的 peer 上的时候，如果 `blob == blob'`，那么 `seqno0==seqno1` 并且 `prevhash0==prevhash1`。
* *可用性（Liveness）*. 如果某个正常的 client 调用事件 `broadcast(blob)` 那么所有正常的 peer 最终将会收到 `deliver(*, *, blob)` 事件，其中 `*` 代表任意值。

## 2. 交易签署的基本工作流程

下面我们将介绍交易请求的大致流程。

**注：** *注意下述协议并没有假设所有交易是确定性的，也就是此协议允许非确定性交易。*

### 2.1. client 创建交易，然后发送给其选择的 endorsing peers

为了调用交易，client 发送 `PROPOSE` 消息给其选择的一组 endorsing peers（可能并不是同时发送）。client 可以通过 peer 使用给定 `chaincodeID` 的 endorsing peers 集合，反过来也可以通过 endorsement policy 获取 endorsing peers 集合。例如，交易可以发送给某个 `chaincodeID` 的*所有* endorsers。即便如此，某些 endorsers 可能离线，其它的可能反对，所以选择不签署该交易。submitting client 尝试满足 endorsers 指定的策略。

下面我们将详细介绍 `PROPOSE` 消息的格式，然后讨论 submitting client 与 endorsers 之间的交互模式。

### 2.1.1. `PROPOSE` 消息格式

`PROPOSE` 的消息格式为 `<PROPOSE,tx,[anchor]>`，其中 `tx` 是必选参数，`anchor` 是可选参数。

- `tx=<clientID,chaincodeID,txPayload,timestamp,clientSig>` 其中
	- `clientID` 是 submitting client 的 ID，
	- `chaincodeID` 指定交易相关的链码，
	- `txPayload` 提交的交易自身的内容，
	- `timestamp` 是 client 维护的单调递增（对于每个新的交易）整数，
	- `clientSig` 是 client 对于 `tx` 字段的签名。

	调用交易和部署交易的 `txPayload` 的字段不同。对于**调用交易**，`txPayload` 包含两个字段

	- `txPayload = <operation, metadata>` 其中
		- `operation` 表示链码的操作（方法）和参数，
		- `metadata` 表示调用相关的属性。

	对于**部署交易**，`txPayload` 包含三个字段

	- `txPayload = <source, metadata, policies>` 其中
		- `source` 表示链码的源代码，
		- `metadata` 表示链码和应用相关的属性，
		- `policies` 表示链码相关的策略，比如签署策略。注意签署策略本身并不是由部署交易中的 `txPayload` 提供，但是部署交易包含签署策略的 ID 和参数。

- `anchor` 包含*读版本依赖*，更具体地说就是主键版本对（即 `anchor` 是 `KxN`的子集），用来指定 `PROPOSE` 请求中键的版本。如果 client 指定了 `anchor` 参数，那么 endorser 仅仅依赖本地 KVS 中的读版本签署交易。

`tx` 的密码学哈希 `tid`（即 `tid=HASH(tx)`）用于所有的节点做为唯一的交易标识符。client 将 `tid` 存储在内存中，等待 endorsing peers 返回答复。

#### 2.1.2. 消息模式

client 决定与 endorsers 交互的序列。例如，client 通常会发送 `<PROPOSE, tx>`（没有 `anchor` 参数）给单个 endorser，endorser 将会返回 client 稍后将会使用的版本依赖（`anchor`）。另外一种情况，client 将会直接发送 `<PROPOSE, tx>`（没有 `anchor`）给所有其选择的 endorsers。不同的交互模式都是可以的，client 具有选择的自由。

### 2.2. endorsing peer 模拟交易执行，然后返回签署签名

收到 client 发来的 `<PROPOSE,tx,[anchor]>` 消息之后，endorsing peer `epID` 首先验证 client 的签名 `clientSig`，然后模拟交易执行。如果 client 指定了 `anchor` 那么 endorsing peer 仅仅依赖读版本号（即下面解释的 `readset`）模拟交易执行。

模拟交易就是 endorsing peer 试验性地*执行*交易（`txPayload`），调用 `chaincodeID` 指定的链码，复制 endorsing peer 本地维护的状态。

执行的结果就是，endorsing peer 计算*读版本依赖*（`readset`）和*状态更新*（`writeset`），这个过程在DB的语境下也被称为 *MVCC+postimage info*。

回忆一下包含键值（k/v）对的状态。所有k/v实体是带版本的，也就是每个实体包含有序的版本信息，每次值更新的时候，版本号也递增。模拟交易的 peer 记录链码访问的所有 k/v，但是 peer 不更新状态。 更具体的说：

* 给定 endorsing peer 执行交易之前的状态 `s`，对于交易读取的每个 `k`，(k,s(k).version)将被添加到 `readset` 中。
* 另外，对于交易修改的每个 `k`，`(k,v')` 将被添加到 `writeset` 中，其中 `v'`是更新后的新值。另外 `v'`也可以是相对于之前值(`s(k).value`)的差值。

如果 client 在 `PROPOSE` 消息中指定了 `anchor` 参数，那么当模拟交易时 client 指定的 `anchor` 必须与 `readset` 中的相同。

然后 peer 内部转发 `tran-proposal`（可能是 `tx`）到其签署交易的逻辑，也就是**endorsing logic**。默认情况下，endorsing logic 接收 `tran-proposal`，然后简单地对其签名。然而 endorsing logic 可以执行任意的功能，比如与遗留系统进行交互进而决定是否签署交易。

如果endorsing logic决定签署某个交易，那么发送 `<TRANSACTION-ENDORSED, tid, tran-proposal,epSig>` 消息给 submitting client(`tx.clientID`)，其中：

- `tran-proposal := (epID,tid,chaincodeID,txContentBlob,readset,writeset)`,

	其中 `txContentBlob` 是链码/交易指定的消息。 用 `txContentBlob` 表示 `tx` （比如 `txContentBlob=tx.txPayload`）。

-  `epSig` 是 endorsing peer 对 `tran-proposal` 的签名

另外对于 endorsing logic 拒绝签署交易的情况，endorser *可能* 发送 `(TRANSACTION-INVALID, tid, REJECTED)` 消息给 submitting client。

注意 endorser 在这一步并没有修改状态，交易模拟产生的更新在签署的过程中并没有影响状态！

### 2.3. submitting client 收集交易的签署，然后通过 ordering service 广播

submitting client 等待接收到“足够”多的消息，`(TRANSACTION-ENDORSED, tid, *, *)` 的签名表示该交易提案被签署了。如在2.1.2节讨论的，这可能涉及一到多次 endorsers 间的交互。

“足够”的具体数量依赖链码的endorsement policy。如果满足endorsement policy，那么交易就被*签署*了；注意此时并没有被提交。endorsing peers 返回的签过名的 `TRANSACTION-ENDORSED` 消息集合被称为 *endorsement*。

如果 submitting client 没有为交易提案收集到 endorsement，那么它将抛弃该交易，或者稍后重试。

对于签署有效的交易，我们现在开始使用 ordering service。submitting client 发送 `broadcast(blob)` 消息调用 ordering service，其中 `blob=endorsement`。如果 client 没有直接调用 ordering service 的能力，它可以选择某个 peer 代理发送广播消息。这个 peer 必须是可信的，不会从 `endorsement` 移除消息，否则交易将被认定无效。注意，然而代理 peer 不能伪造有效的 `endorsement`。

### 2.4. ordering service 投递交易到 peers

当 `deliver(seqno, prevhash, blob)` 事件发生的时候，并且 peer 已经执行了版本号小于 `seqno` 的所有状态更新，那么 peer 将会执行下述操作：

* 基于链码（`blob.tran-proposal.chaincodeID`）的策略检查 `blob.endorsement` 是否有效。

* 通常情况下同时也会验证没有违反依赖（`blob.endorsement.tran-proposal.readset`）。对于更复杂的用例，`tran-proposal` 的字段可能根据不同的 endorsement policy 而不同。

根据状态更新所选的一致性属性或者“隔离保证”，依赖验证的实现可以有不同的方式。除非链码 endorsement policy 指定不同的隔离保证，否则**串行化（Serializability）**是默认的隔离保证。串行化可以这样实现，要求 `readset` 中*每个*键相关的版本与状态中的版本一致，拒绝不满足要求的交易。

* 如果所有这些检查都通过了，那么交易就被认为是*有效的*或者*committed*。在这种情况下，peer 将 `PeerLedger` 位图中对应该交易的位置标记为1，将 `blob.endorsement.tran-proposal.writeset` 应用到区块链的状态（如果 `tran-proposals` 是相同的，否则由 endorsement policy 的逻辑定义处理 `blob.endorsement` 的方法）。

* 如果 endorsement policy 验证 `blob.endorsement` 失败，那么交易无效，peer 将 `PeerLedger` 位图中对应该交易的位置标记为0。特别需要注意的是，无效交易不会改变状态。

注意上述流程已经足够让所有正常的 peers 维护相同的状态。也就是通过 ordering service 的保障，所有正常的 peers 将会收到相同的 `deliver(seqno, prevhash, blob)` 序列。因为 endorsement policy 评估和 `readset` 的版本依赖评估是确定性的，所以所有正常的 peers 将会对交易是否有效得出相同的结论。因此，所有 peers 提交和应用相同的交易序列，并且以同样的方式更新状态。

![Illustration of the transaction flow (common-case path).](http://ofenxpygt.bkt.clouddn.com/flow-4.png)

Figure 1. 某种可能的交易流程展示

---

## 3. Endorsement policies

### 3.1. Endorsement policy 规范

**endorsement policy** 就是*签署*交易的条件。区块链节点上有些预设的 endorsement policies，部署链码的时候在部署交易中指定。endorsement policies 可以通过参数配置，这些参数可以在部署交易中指定。

为了保障区块链和安全属性，endorsement policies **必须是已经被证实的策略**，也就是具有有限的功能来确保有限的执行时间（终止），具有确定性，具有性能和安全保障。

endorsement policies 的动态组合对上述要求非常敏感，所以目前不允许动态组合 endorsement policies，但是以后可能会支持。

### 3.2. 基于 endorsement policy 的交易评估

只有交易被签署了之后才被认定是有效的。某个调用交易首先必须获取满足链码策略的 *endorsement*，否则该调用交易不能提交。正如第2节所阐述的，这一步通过 submitting client 和 endorsing peeers 之间的交互完成。

形式上来说 endorsement policy 是关于签署的断言，以及下一步评估为 TRUE 或 FALSE 的状态。对于部署交易，endorsement 基于系统范围的策略（比如系统链码）获取。

endorsement policy 断言指定某些变量。可以指定：

1. 与链码相关的键或标识（从链码的 metadata 获取），比如 endorsers 的集合；
2. 更多链码的 metadata 信息；
3. `endorsement` 和 `endorsement.tran-proposal` 的元素；
4. 更多。

上述列表按照表达能力和复杂性增加排序，也就是说仅仅与节点的键和标识相关的策略比较容易支持。

**对于 endorsement policy 断言的评估必须是确定性的** 每个 peer 应该在本地评估 endorsement，peer 之间不需要交互，但是所有正常的 peer 以相同的方式评估 endorsement。

### 3.3. endorsement policies 示例

断言包含评估为 TRUE 或 FALSE 的逻辑表达。通常来说条件将会使用 endorsing peers 对交易的数字签名。

假如链码指定 endorser 集合为 `E = {Alice, Bob, Charlie, Dave, Eve, Frank, George}`。 某些策略可能是：

- 一个来自集合 E 所有成员的对于同一 `tran-proposal` 的有效签名。

- 一个来自集合 E 任一成员的有效签名。

- 来自根据条件
`(Alice OR Bob) AND (any two of: Charlie, Dave, Eve, Frank, George)` 组成的 endorsing peers 的有效签名。

- 来自7个 endorsers 中任意5个 endorsers 的有效签名。（更一般地说，对于拥有 `n > 3f` 个 endorsers 的链码，来自 `n` 个 endorsers 中任意 `2f+1` 个 endorsers 的有效签名，或者多于 `(n+f)/2` 个 endorsers 的有效签名。）

- 假如 endorsers 都被赋予了“股份”或“权重”，
  比如 `{Alice=49, Bob=15, Charlie=15, Dave=10, Eve=7, Frank=3, George=1}`,
  其中总股份为100：此策略要求超过一半股份的有效签名（也就是 endorsers 组合的股份大于50），比如 `{Alice, X}` 其中的 `X` 不同于 George 的，或者 `{everyone together except Alice}` 等等。

- 前一例子中的股份分配可以是静态的（硬编码在链码的 metadata 中），也可以是动态的（比如依赖链码的状态，在执行过程中更新）。

- 来自 (Alice OR Bob) 关于 `tran-proposal1` 的有效签名以及来自 `(any two of: Charlie, Dave, Eve, Frank, George)` 关于 `tran-proposal2` 的有效签名，其中 `tran-proposal1` 和 `tran-proposal2` 仅仅在 endorsing peers 和状态更新上不同。

如何使用这些策略依赖应用的场景，依赖解决方案对于 endorsers 发生故障和异常时所期望的弹性。

## 4 (v1 之后版本). 有效账本和 `PeerLedger` 检查单 （剪枝）

### 4.1. 有效账本 (VLedger)

为了维护仅包含有效交易的账本，peers 除了维护状态和账本，还得维护 *有效账本（VLedger）*。这是过滤掉账本中无效交易后的哈希链。

VLedger 区块（vBlocks）的构建如下所述。因为 `PeerLedger` 区块可能包含无效交易，所以首先过滤掉区块中的无效交易。每个 peer 单独过滤（比如使用 `PeerLedger` 关联的位图）。vBlock 就是过滤掉无效交易的区块。这些 vBlocks 的大小是动态的，也可能是空的。下图给出构建 vBlock 的示例。
     ![Illustration of the transaction flow (common-case path).](http://ofenxpygt.bkt.clouddn.com/blocks-3.png)

Figure 2. 从账本（`PeerLedger`）区块构建有效账本区块

vBlocks 由每个 peer 连接成一个哈希链。更具体来说，有效账本的区块包含：

* 前一个 vBlock 的哈希。

* vBlock 号。

* 最近的 vBlock 中所有有效交易的有序列表（也就是对应区块中有效交易的列表）。

* `PeerLedger` 中对应区块的哈希。

所有这些信息由 peer 连接并且哈希，计算有效账本中 vBlock 的哈希。

###4.2. `PeerLedger` 检查点

账本包含无效交易，所以没有必要永远保存。但是一旦构建了对应的 vBlocks，peers 却不能简单地抛弃 `PeerLedger` 区块，进而修剪 `PeerLedger`。在剪枝的情况下，如果某个新的 peer 加入网络，那么其它 peers 既不能传输抛弃的区块到新加入的 peer 了，也不能使该新 peer 信服其 vBlocks 的有效性。

为了促进 `PeerLedger` 的剪枝，该文档描述了一种*检查点*的机制。该机制通过 peer 网络建立 vBlocks 的有效性，使得检查点内的 vBlocks 取代抛弃的 `PeerLedger` 区块。反过来这也压缩了存储空间，因为没有必要存储无效交易了。这也减少了新加入网络的 peer 构建状态的工作量（因为它们不需要通过重新执行 `PeerLedger` 验证单个交易的有效性，而是简单地执行有效账本中的状态更新即可）。

####4.2.1. 检查点协议

peers 周期性的对 *CHK* 个区块执行检查点协议，其中 *CHK* 是一个可配置的参数。为了发起一个检查点， peers 广播（比如 gossip）有效账本对应的消息 `<CHECKPOINT,blocknohash,blockno,stateHash,peerSig>` 到其他 peers，其中 `blockno` 是当前的区块号，`blocknohash` 是对应的区块哈希，`stateHash` 是最新状态的哈希（比如通过 Merkle hash），`peerSig` 是该 peer 对 `(CHECKPOINT,blocknohash,blockno,stateHash)` 的签名。

某个 peer 收到足够多签名正确的 `CHECKPOINT` 消息，并且 `blockno`、 `blocknohash` 和 `stateHash` 都匹配，那么就可以创建一个*有效检查点*。

对于区块号 `blockno` 和 `blocknohash` 一旦创建了有效的检查点，那么一个 peer：

* 如果 `blockno>latestValidCheckpoint.blockno`，那么该 peer 赋值 `latestValidCheckpoint=(blocknohash,blockno)`，
* 将构建有效检查点的签名存储到 `latestValidCheckpointProof`，
* 将 `stateHash` 对应的状态存储到 `latestValidCheckpointedState`，
* （可选的）修剪 `PeerLedger` 到区块号 `blockno`（包含）。

####4.2.2. 有效检查点

很明显检查点协议引发了下述问题：*什么时候 peer 可以修剪 `PeerLedger`? 多少个 `CHECKPOINT` 消息才是“足够多”？*。这由*检查点有效策略*定义，至少有两个可能的方法，也可以结合起来：

* *本地（peer指定的）检查点有效策略（LCVP）.* 某个 peer *p* 的本地策略可以指定一组其信任的 peers，它们的 `CHECKPOINT` 消息足够创建有效的检查点。例如 peer *Alice* 的 LCVP 可以定义 *Alice* 需要接收来自 Bob 的 `CHECKPOINT` 消息，或者来自 *Charlie* 和 *Dave* 的。

* *全局检查点有效策略（GCVP）.* 检查点有效策略也可以指定为全局的。这与本地策略类似，除了该策略是系统粒度的规定，而不是 peer 粒度。例如，GCVP 可以指定：
	* 每个 peer 可以信任被*11*个不同的 peers 确认的检查点。
	* 对于在每个 orderer 同时是 peer 的部署场景，其中最多有 *f* 个 orderers 产生拜占庭故障，那么 peer 可以信任被 *f+1* 个不同 peers 确认的检查点。
