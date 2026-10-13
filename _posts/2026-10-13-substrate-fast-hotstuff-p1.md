---
title: Substrate 接入 Fast-HotStuff 开发记录
layout: post-with-content
post-image: "https://github.com/xufeisofly/picx-images-hosting/raw/master/substrate-fast-hotstuff-p1/head-updated.7so0628voc.webp"
tags:
- Substrate
- Fast-HotStuff
- consensus
- blockchain
---

## 前言

在此之前我是有过一次开发 HotStuff 共识经验的（Shardora），具体见文章[《HotStuff 工程设计与实现》](https://www.xufeisofly.xyz/blog/shardora-hotstuff)，不过那次开发的完全是一条自研区块链，完成了 600 节点的性能测试，虽然最后数字很漂亮，但现在想来犯了很多错误。本次则是对 Substrate 进行共识的改造，且改造后的链是要真实上线的，对稳定性和安全性的要求很高，因此开发本身时间很短，但耗费了大量时间做测试和审计，同时需要兼容 Substrate 本身的各种功能接口。比如如何新增和删除 Validator，如何快速同步历史区块，节点掉线与恢复，Archive 节点和 Pruned 节点的适配等等。同时，我对节点的扩展性也做了性能实验，并修改了 Substrate 推荐的网络拓扑接口，即使 Validators 数量超过 100 个，也依然支持 8K 以上的吞吐量。

总之，共识开发本身不难，但从开发到最终上线并稳定运行，这个经验对于我来说是十分宝贵。另外忍不住说一句，大部分代码写于 AI 编程还没有普及时期（/悲伤）。

由于链目前还在测试中，因此后续我会持续更新。

## 为什么要改共识

Hyperliquid 的爆火已经有一段时间了，我们发现，通用区块链可以实现 DeFi 必要的功能，但当一个产品追求用户体验的极致时，势必需要一套有针对性优化的基础设施，即使牺牲安全性与去中心化。换句话说，对于一个「好」的 Web3 产品，安全性不必太多，刚好就够了，这听起来让人沮丧，但事实却是如此，毕竟大家仍然把当初 4 个节点的 Hyperliquid 当成一条链来看。

无论如何，牺牲安全性换来的是高性能和低成本。大家发现对于 DeFi 类的场景如 Perp DEX，需要的不是一个如同 Ethereum 或是 Solana 的通用链，而是针对某个特定领域甚至特定应用的专用链。所谓的通用链一般是通过 VM 以及智能合约在线运行业务逻辑，这就导致使用通用链的业务往往受限于成本和性能，通用链的安全性很高，但考虑到功能性与安全性的权衡，安全与去中心化在这些场景下其实是溢出的，因此我们需要一条链，它专注于性能与成本的优化。

而对性能的执著导致大部分开发者选择 Rust 这个编程语言，放眼看去，能够有足够的自定义能力，且使用 Rust 的就是 Substrate 框架了。可惜 Substrate 原生链的性能并不好，其中首要阻塞点就是其默认共识 Aura。

Aura 共识出块时间为 6 s，这显然不符合要求，但它提供了足够的定制化能力，让我们可以替换其共识协议，在当前主流共识中能同时满足一定吞吐量并且具有较低延迟的就是 HotStuff。

![共识协议吞吐量与延迟对比](https://github.com/xufeisofly/picx-images-hosting/raw/master/substrate-fast-hotstuff-p1/consensus-performance.9rk6wdw9f8.webp)

我们也考虑了 Narwhal 系列，但其问题就是虽然吞吐量很高，但交易确认延迟也高，截至方案决策之时，Sui 新研发的 Mysticeti-C 共识解决了 DAG 延迟高的问题，但生产环境是否足够稳定还未可知，而且复杂度比 HotStuff 高出不少。

基于之前 Shardora 链开发经验，一般转账交易的 HotStuff 吞吐量内网下可以达到 20K TPS，而 Substrate 转账交易每笔执行时间约 100-200 μs，因此吞吐量瓶颈上限 10K 是受限于执行层，共识层使用 HotStuff 是合理的，这里有个误区，很多区块链的性能的瓶颈其实并不在共识算法，而在执行层和网络层，Aptos 的 Block-STM 就是为了解决这个问题（而且实话说，一个链的吞吐量做的很高意义不大，大部分号称 TPS 超过 100k 的链上线后 TPS 连 100 都没有，与其关注吞吐量还不如关注交易延迟）。

## HotStuff 如何嫁接到 Substrate

HotStuff 属于 BFT 共识，它的安全性依赖一个重要假设，即恶意节点的数量（或者权重）小于 $f$，$f$ 满足 $N = 3f + 1$，即如果每个节点的权重一样，那么恶意节点数量需要小于 $\frac{1}{3}$，不然整个 HotStuff 协议是不成立的。

相比其他 BFT 共识，HotStuff 有更好的吞吐量和扩展性，这依赖于它是基于单一 Leader 的机制，由一个 Leader 节点发送提案，并收集投票，而不是互相广播，这就使其通讯复杂度由二次方变为了线性，有能力拓展到上千个节点。

Fast-HotStuff（下文简称 FHS） 相比 HotStuff 从三个投票阶段改为了两个，并且解决了分叉攻击等问题，进一步缩小了延迟。因此，纵观使用 HotStuff 的区块链如 Monad，Hyperliquid 等，基本都使用的是二阶段 HotStuff 而非原生的 HotStuff 协议。

稍微麻烦的是，Substrate 虽然支持自定义，但其现有的语义和 BFT 共识并不完全相同，而是更适配中本聪共识，主要表现在几个概念：

+ Best Block：区块链存在多个分叉时，仅有一个分叉是 Canonical 的，其最新的 Block 称之为 Best Block。
+ Fork Choice：当出现分叉时，需要选择一个分叉继续向后出块。

这些概念在 BFT 共识中不存在，这就需要我们把 BFT 共识想象成一个中本聪共识的子集。

**共识语义适配**

Substrate 中关于共识的接口和语义都是面向「中本聪共识」的，而非 BFT 共识，因此它认为的共识协议是这样分层的。

- **Propose Rule**：决定如何产出一个提案，如何出块，比如 PoW，PoS 等，共识协议使用 Propose Rule 生成一个状态转换的建议，其他节点收到后执行。

- **Fork Choice Rule**：Propose Rule 产生的状态机路径并不唯一，也就是说区块链中的分叉，这时候需要 Fork Choice Rule 选择出唯一一条合法的分支，作为 Canonical Chain。比如比特币的 Fork Choice Rule 是 Longest Chain，而以太坊是 GHOST（Heaviest Chain）。

- **Finality Rule**：Fork Choice Rule 的选择并不是即时终局，有可能发生 Reorg，比如本次根据 Fork Choice Rule 选择的 Fork 是 A，而一段时间后可能根据相同的规则就变成 B 了。因此共识协议往往有一个 Finality Rule，它规定了某种情况下 Block 已经最终确定，永远不会因为 Fork 的改变而发生回滚。比如比特币不存在一个显性的 Finality Rule，但存在一个隐性的规则即 6 个区块之前的部分最终确定，以太坊使用 Casper FFG，这是一个 BFT 投票协议，来确定某一个 checkpoint 之前的区块最终确定。Substrate 默认使用 GRANDPA 实现 Finality，它和其共识协议 Aura 是解耦的，换句话说，Aura 本质上是 Propose Rule 和 Fork Choice Rule 的组合。

针对上面的共识接口，Substrate 的共识流程如下：

```text
Proposal -> Best(Imported) Block -> Finalized Block
```

其中 Best Block 即为 Fork Choice Rule 中选择的 Canonical Chain 的最新区块，在 Substrate Client 的语义中，一个 Block 被 Import 可以选择是否是 Best，而一个 Block 被 Commit 后会变为 Finalized。

不同于「中本聪共识」，BFT 共识的特点是**不存在分叉且 Commit 即 Finalized**，因此它的共识语义要简单很多。当一个 Block 有一个 QC 后即为 Best，而两个 QC 触发 Commit 后则变为 Finalized。

![Fast-HotStuff 的 QC 与区块提交关系](https://github.com/xufeisofly/picx-images-hosting/raw/master/substrate-fast-hotstuff-p1/commit-and-qc.232nugwdkx.webp)

## 工程设计和选择

Substrate Fast-HotStuff 协议的代码架构如图。

![Fast-HotStuff 模块架构](https://github.com/xufeisofly/picx-images-hosting/raw/master/substrate-fast-hotstuff-p1/modular-architecture-updated.32ir7oufma.webp)

+ Core：承载共识协议的消息处理的核心逻辑
+ Proposer：负责打包提案和广播，决定了交易打包的策略
+ Pacemaker：负责当 View 超时时维持活性，发送超时消息
+ Block Import：负责导入和提交区块、QC，同时实现 Import Queue，对接 Substrate 的区块同步能力。
+ State：维护当前节点当前的共识状态以及 Leader 选取逻辑
+ Consensus Network：对接 Substrate Gossip 网络，异步收发网络消息
+ Session Info：提供当前 Session 和历史 Session 的信息，包括 ValidatorSet，Voting Power 等，并处理 Session 轮换 callback。
+ Peer Provider：提供 peer 信息，辅助消息收发。
+ Crypto：BLS 和 Ed25519 的签名和验证能力。
+ Chain/Warp Sync：Substrate 默认 Chain Sync 逻辑，以及 FHS 自研 Warp Sync。

## Crypto 模块

我们需要引入 BLS 库用于实现聚合签名，虽然 w3f-bls 库天然兼容 Substrate，但性能较低，这里选择了 bls-signatures，bls-signatures 库底层同时支持使用 bls12-381 和 blstrs，其中 blstrs 是 C 库，性能最好。bls-signatures 中缺少部分聚合签名函数，比如针对 Fast-HotStuff AggQC 的聚合公钥快速签名验证、PoP 相关函数等，都需要自行实现。

## 区块打包

区块打包是 Proposer 模块将 TxPool 中的 Ready Pool 中的交易打包到一个提案中的过程，打包过程一定是基于 Parent Block 已经是 Best Block，否则会一直阻塞。这里涉及 Substrate 的两种 TxPool 模式的选择问题，即 SingleState 和 ForkAware。

所谓 SingleState 是 Substrate 前期一直使用的 TxPool 模型，它的原理是仅维护一个全局的 pool 视图，包含 pool.ready 和 pool.future，这个全局视图是基于 Best Block 的，如果 Best Block 发生了变化，那么 TxPool 视图也会变化。因此，SingleState 无法解决下面的问题。

```text
      A2 - A3
     /
F1
     \
      B2 - B3
```

同一个交易的 nonce 可能在 A3 上已经被消耗，因此是 invalid，但在 B3 上还没有被消耗，是 ready。但如果此时 Best Block 是 B3，但又希望在 A3 之后出块，那么这笔交易就有可能被再次打包。

所以说，SingleState 永远基于 State(Best)，并不会维护多个 Fork 的 States。而 FHS 的 Leader 发起提案时是基于本地 HighQC 最大的分支，恰好，SingleState 会限制「任何想在非本地 HighQC 分支」上提案的 Leader，而诚实 Leader 不会这么做。所以 SingleState 反而是对恶意 Leader 的一个天然抑制。

举个例子，下图中，best2 可能会被 Reorg 到 best3，best3 可能会被 Reorg 到 best4，之所以会被 Reorg，根本原因是网络延迟以及拜占庭节点的恶意行为，总之 Committed Block 之后是有可能出现短暂分叉的。而不同的 Validator 在打包 Proposal 的交易时，基于的都是本地的 HighQC，而 HighQC 对应的恰恰是本地的 Best Block，这就意味着，诚实节点的 Proposal 交易打包一定是正确的，SingleState 没有问题，而恶意节点如果故意在非 Best Block 之后打包，那么它基于的 TxPool 有可能是错乱的，这对它没有好处，反而增加了恶意节点 Proposal 被拒绝的可能，因此 SingleState 模式对于 Fast-HotStuff 来说是合理的，主要就是诚实节点永远只会从 Best Block 之后拓展区块。

![SingleState 交易池与 Best Block 的分叉切换](https://github.com/xufeisofly/picx-images-hosting/raw/master/substrate-fast-hotstuff-p1/single-state-reorg.4922g8o1bz.webp)

总结，对于 FHS 的 TxPool，使用 SingleState 完全满足要求，且相比 ForkAware，使用 SingleState 的吞吐量可增加 10%-20%。

## 网络与节点发现

Substrate 是按照不同的协议进行 P2P 组网的，一个 peer 节点启动后，无论是不是 Validator，都会默认加入部分协议网络，比如 TxPool，sync 协议，这些由 Substrate 节点发现自动执行的。自研共识的组网当然可以复用主协议，但这就会导致非 Validator 也会收到共识消息，因此需要选择自定义协议，将共识的组网仅仅限制在 Validators 节点之内。

另外由于 ValidatorSet 是不断变化的，比如新 Validator 的加入和退出，因此我们需要灵活管理共识网络的成员。使用 ReservedNode 十分方便，并且对于 NonReservedNode，设置为 `NonReservedPeerMode::Deny`，这意味着非 Reserved Node 是无法接收到共识协议的消息的，这样在 ValidatorSet 变化时，同时调整 protocol 的 Reserved Nodes，以实现网络的重组。

一个 network protocol 的格式如下：

`/<genesis_hash>[/fork_id]/<protocol>/<version>`

Substrate 基础节点默认会有如下 protocol

| 类型 | Protocol name | 作用 |
| --- | --- | --- |
| notification | `/<genesis>/block-announces/1` | Block announce，同步入口之一 |
| notification | `/<genesis>/transactions/1` | TxPool/transaction gossip |
| request-response | `/<genesis>/sync/2` | Block request/download |
| request-response | `/<genesis>/state/2` | state request |
| request-response | `/<genesis>/sync/warp` | GRANDPA Warp Sync，仅配置了 warp provider 时注册 |
| request-response | `/<genesis>/light/2` | light client request handler |

FHS 实现了自己的 protocol stream 通道 `/fhs/1`，并通过 Reserved Peers 决定 protocol 上谁可以建立和保持 substream。当 ValidatorSet 变化时会重新 refresh protocol 的 Reserved Peers。

节点发现分为非 Validator 节点发现以及 Validator 节点发现，Substrate 本身具备对普通 node 的节点发现能力，这里主要指的是如何让 Fast-HotStuff 主动发现一个新加入的 Validator，并与之建立网络联系。

*Validator 发现流程*

![新 Validator 的发现与加入流程](https://github.com/xufeisofly/picx-images-hosting/raw/master/substrate-fast-hotstuff-p1/validator-discovery.7p4e8bxodl.webp)

---

未完待续，需要上班
