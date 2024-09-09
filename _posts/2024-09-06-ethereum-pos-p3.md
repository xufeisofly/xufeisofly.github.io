---
title: 以太坊的 PoS - Part3 Casper FFG
layout: post-with-content
post-image: "/assets/images/ethereum-pos-p3-img/head.jpg"
tags:
- ethereum
- consensus
- blockchain
---

本文是介绍以太坊 PoS 共识的第三部分。

# 最终确定性 Finality
---

最终确定性（Finality）是指区块保证不会被回滚，会永远成为链的一部分。[上篇文章](https://xufeisofly.github.io/blog/ethereum-pos-p2)介绍了作为以太坊 Fork Choice Rule 的 LMD GHOST，它使得共识协议拥有了不断出块的活性 Liveness，但仍然无法完全保证一个区块不会被恶意节点回滚掉，比如遭遇 Long Range Attack（见 [Part1](https://xufeisofly.github.io/blog/ethereum-pos-p1)）。

在比特币的 PoW 中并没有实现 Finality，而是默认了 51% 攻击不会存在，所以当超过 51% 的算力真的被恶意节点所拥有时任何区块都是不安全的，而且如果出现多个分叉这个阈值还要更小。这方面 BFT 共识算法就要好很多，每一个区块提交时就会被 Finalized，不过需要满足拜占庭假设，即恶意节点数量 $ <{1\above{1pt}3} $，否则不光是 Finality，BFT 共识协议连续出块的活性都会无法保证。

以太坊开发团队经历了长时间的方案讨论和取舍，提出了 Capser 协议，Vitalik 和 Vlad 分别主导了 Casper FFG 和 Casper CBC 两种协议的方案设计。最终以太坊选择了 Casper FFG，这是一套基于 BFT 共识改进的上层共识协议，作为 Finality Rule 用于实现区块的最终确定性，其底层还需要依赖其他共识协议完成出块和分叉选择。相比 PoW，以太坊的 Finality Rule 直接给出了 yes 或 no，而不是一个区块被 Finalized 的概率，同时还基于 PoS 为其加入了惩罚机制（Slashing），使得 Casper FFG 在不满足拜占庭假设的情况下依然具有很强的安全性。

那么 Casper FFG 是什么协议呢？

# Finality Rule：Casper FFG 协议
---

[Part2](https://xufeisofly.github.io/blog/ethereum-pos-p2) 中根据我的理解，将共识协议分为三个部分。

+ Proposal Rule
+ Fork Choice Rule
+ Finality Rule

这里不再解释。业界会把 Proposal Rule + Fork Choice Rule 合称为 Proposal Mechanism（其实我更喜欢分开处理，能将 Fork Choice Rule 单独看待，没人规定 PoW 就一定要使用 Longest Chain Rule 去做分叉选择），它或许可以是 PoW + Longest Chain Rule，或许可以是 PoS + GHOST，用于不断的产出区块，并完成对区块内交易的共识。Casper FFG 是一个在 Proposal Mechanism 上层的补充协议，用于实现区块的 Finality，是以太坊的 Finality Rule。Casper FFG 不能脱离 Proposal Mechanism 单独使用。

Casper FFG 论文中，将 PoS 划分为两类：

+ Chain-Based PoS 协议
+ BFT-Based PoS 协议

Casper FFG 就属于 BFT-Based PoS 协议，这是因为在一些传统的 BFT 共识中我们需要收集足够的投票以改变区块状态，比如 PBFT 和 HotStuff，而 Casper FFG 中的投票权重是通过质押的 Stake 进行计算的。拥有相同机制的还有对接了 PoS 的 Tendermint 和 Fast HotStuff，它们都是 BFT-Based PoS 协议。其实我个人认为这个名字更应该叫做 PoS-Based BFT 协议，因为只需要将传统的 BFT 共识投票改为 Stake 权重就行了，这是一个很小的改动（前提是不能使用阈值签名，具体见 [Fast HotStuff 那篇文章](https://xufeisofly.github.io/blog/fast-hotstuff)）。

## 两阶段 BFT 共识
Casper FFG 是一个两阶段的 BFT 共识协议，同时加入了 Stake 作为投票的统计权重。它的两个阶段在论文中称为 Prepare 和 Commit，与 PBFT 相同，当然只要你愿意也可以改成 Prevote 和 Precommit 什么的。名字并不重要，我们重点关注两个阶段起到的作用，因此下面这两个名字更能表达两个阶段的真正含义（据说是取自 Casper CBC）。

+ **Justification**：相当于 BFT 共识中的 Lock 阶段，在 BFT 共识中 Locked 区块可用于检测提案是否冲突，并用于 Unhappy Path 下的视图切换。当一个区块收到 $ >{2\above{1pt}3} $ 的 Prepare 投票时则被认为 Justified。
+ **Finalization**：相当于 BFT 共识中的 Commit 阶段，完成一个区块的提交。当一个 Justified 的区块的再次收到 $ >{2\above{1pt}3} $ 的 Commit 投票时则被认为 Finalized。

下文中会统一使用 Justification 和 Finalization 表示 Casper FFG 中的两个阶段，使用 `Prepare` 和 `Commit` 表示两个阶段发送的投票类型。

那么整个过程就简单了：一个区块首先被 Proposer 打包进提案并广播给其他 Validators，Validator 验证该提案通过则广播 `Prepare` 投票，同时收集其他 Validator 的投票，当收集到 $ >{2\above{1pt}3} $ 的投票（基于 Stake）时，该区块状态改为 Justified。当发现自己投票的区块被 Justified 之后，Validator 会再次广播 `Commit` 投票，同样当 Validator 的收集到 $ >{2\above{1pt}3} $ 的 `Commit` 投票后，该区块就会被 Finalized。这就是两个阶段的大致过程。

以太坊的 Casper FFG 并不会对每一个交易区块都进行 Finality 的共识，那样会消耗大量带宽和计算资源，尤其是随着 Validator 数量不断变多，会严重影响系统吞吐量，这点我们从那些纯粹使用 BFT 共识的公链当中就能看到，这些公链如 Zilliqa、Diem 包括 Shardora 都规定了 Validator Set 中节点的数量上限。

还有一个原因，以太坊在套用 Casper FFG 之前，已经有了底层的 Proposal Mechanism 在不断产生区块了，Liveness 和一部分 Safety 已经得到了保证，此时套上一个完整覆盖所有区块的 BFT 共识会导致系统整体 Liveness 下降和 Safety 冗余。

## Checkpoint 和 Link
以太坊使用了 Epoch 这个概念，规定一个 Epoch 分为 32 个 Slot，并将所有的 Validator（参与质押的节点）也分为 32 个 Validator Set，分别负责 32 个 Slot 的出块。Casper FFG 的粒度是 Epoch 而非 Slot，每一个 Epoch 中最多有 32 个区块，而第一个区块称为 Checkpoint。因此 Checkpoint 就是一个普通区块，只不过我们会对 Checkpoint 尝试进行 Justify  和 Finalize，而被 Finalized 的 Checkpoint 的所有祖先区块都会被 Finalized，如下图。


![image.png](/assets/images/ethereum-pos-p3-img/image1.png)


整个共识过程就是一个普通的 $ O(n^2) $ 通讯复杂度的 BFT 共识，只不过共识的内容不再是交易，而是 Checkpoint 区块的 Finality 信息，交易的共识已经在 Proposal Mechanism 中被完成了。换句话说，Proposal Mechanism 是对交易内容进行共识，而 Finality Rule 是对区块是否 Finalized 进行共识。

Checkpoint 的结构大致如下：

```go
struct Checkpoint:
    Height     uint64 // Epoch
    BlockHash  Hash   // Checkpoint 中包含的 Block Hash 指针
```

Vote 投票包括如下字段：`<v, s, t, h(s), h(t)>`，其中：

+ v 代表消息类型，在非 Pipeline 的消息中，包括 `Prepare` 消息和 `Commit` 消息。
+ s 代表 Source Checkpoint，即来源 Checkpoint。
+ t 代表 Target Checkpoint，即目标 Checkpoint。
+ h(Checkpoint) 表示该 Checkpoint 的高度，这个高度并非 Block Height 而是 Checkpoint Height，对于以太坊来说就是所在的 Epoch。

由于 Height 已经是 Checkpoint 当中包含的信息，所以一个投票就可以简化为对 $ s\to t $ 的投票。一个诚实节点的 $ s $ 是它能看到的高度最高的 Justified Checkpoint，我们将在后文 Fork Choice Rule 中进行介绍，因此为 $ s\to t $ 投票也包含了自己对该 $ s $ 已经 Justified 这个事实的认可，Validator 在验证提案时也需要验证 $ s $ 是否已经是 Justified。

如果一个 Validator 对 $ s\to t $ 进行了投票，我们就认为在它本地视图中 $ s $ 之间 $ t $ 构成了一个 Link。一旦 $ s\to t $ 收集到超过 $ 2\above{1pt}3 $ 的投票， $ t $ 就会完成 Justification，此时我们说 $ s $ 与 $ t $ 之间构建了一个 **Supermajority Link**。如下图。本文所有的图片中使用蓝色箭头表示 Link，红色箭头表示 Supermajority Link。

![image.png](/assets/images/ethereum-pos-p3-img/image2.png)

接下来我们简化一下区块链结构用于理解 Casper FFG 协议。在 Finality Rule 中不用考虑常规 Block 只考虑 Checkpoint，所以我们去掉非 Checkpoint 的区块，仅保留每个 Epoch 中的 Checkpoint，就可以构成一个 Checkpoint tree，Checkpoint 之间用虚线箭头连接，表示中间省略了常规区块，如下图。

![image.png](/assets/images/ethereum-pos-p3-img/image3.png)

每个 Checkpoint 都会去尝试收集 Validator 的两轮投票（`Prepare` 和 `Commit`），一旦收集到超过 $ >{2\above{1pt}3} $ 权重的投票，就会在 Source Checkpoint 和 Target Checkpoint 之间建立起一个 Supermajority Link（红色箭头表示），并认为该 Target Checkpoint 已经被 Justified。如果对已 Justified 的 Checkpoint 再次收集到超过 $ >{2\above{1pt}3} $ 权重的投票，则认为该 Checkpoint 被 Finalized。如下图。

![image.png](/assets/images/ethereum-pos-p3-img/image4.png)

上图中，每个 Checkpoint 都会尝试成为 Target Checkpoint，却不一定收集到足够的投票完成 Justification（白色区块），原因基本为：

+ 网络问题，导致本次投票消息没有收到。
+ 诚实节点状态不一致（如网络延迟导致），因此投给了不同的分支（是诚实行为）。
+ 存在拜占庭节点，故意不投票或者投票给了别的分支。

## Pipeline 流水线设计
共识过程虽然有两个阶段，但 Casper FFG 使用了类似 [HotStuff 的 Pipeline 设计](https://xufeisofly.github.io/blog/shardora-hotstuff#31-%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86)，每一个 Epoch 只进行一轮投票，这轮投票内容不仅包含了本次 Epoch 的新提案，也包含了上一个 Epoch 的提案状态，因此如果上一个提案已经完成 Justification，再收集 $ 2\above{1pt}3 $ 投票上一个 Epoch 提案就会被 Finalized，而本 Epoch 提案被 Justified，这样一个流水线结构会减少一次 Epoch 中的投票通讯过程。这增加了一个 Checkpoint 的最终确认需要的时间，为两个 Epochs，即 12.8min，不过系统依然是 6.4min Finalize 一个 Checkpoint，吞吐量并没有改变。

在 Pipeline 设计下，判断 Checkpoint 是否 Finalized 只需要数连续 Justified Checkpoint 的个数就可以了。下图中，黄色代表 Justified Checkpoint，绿色代表 Finalized Checkpoint。如果一个 Justified Checkpoint 的直接子 Checkpoint 也被 Justified，那么就可以将本 Checkpoint 变为 Finalized 状态。如果两个 Justified Checkpoint 不是父子关系，那么就无法完成父 Checkpoint 的 Finalization。

![image.png](/assets/images/ethereum-pos-p3-img/image5.png)

正如 BFT 的 Locked 状态可用于验证新提案是否冲突，完成了 Justification 的 Checkpoints 也可以用于检测新的提案，检测的规则称为 Casper Commandments，这来源于 Vitalik 研究 BFT 共识后总结出来的两个 Slashing Conditions（惩罚条件）。

# 两个惩罚条件
---

如果两个 Checkpoint 在不同的分支上，就说明两个 Checkpoint 是冲突的，很显然，一个提案中 Source Checkpoint 和 Target Checkpoint 不能是冲突的，这是提案的基本验证规则，如下图中的 $ B\to C $。

![image.png](/assets/images/ethereum-pos-p3-img/image6.png)

在设计 Casper FFG 协议时，我们明显不希望有两个互相冲突的 Checkpoint 被 Finalized，因为这样就意味着一个分支被另一个分支回滚了，也就意味着有了发生分叉攻击的可能性。

为此 Casper FFG 协议约定了两个 Slashing Conditions，一旦 Validator 触犯了任何一个，都会收到惩罚，即，某个 Validator 不能发出两个投票 $ s_1\to t_1 $ 及 $ s_2 \to t_2 $ 使得以下任意条件成立：

+ 条件一： $ h(t_1) = h(t_2) $，不让此条件成立称为 no double vote，意思是一个 Validator 只能给相同高度的 Target Checkpoint（可能相同或不同的 Checkpoint）投一票。
+ 条件二： $ h(s_1) < h(s_2) < h(t_2) < h(t_1) $，不让此条件成立称为 no surround vote，意思是 Validator 投票的 Link 之间不能发生嵌套。

画图表示这两个条件成立的情况如下：

![image.png](/assets/images/ethereum-pos-p3-img/image7.png)


注意，如果 $ s $ 和 $ t $ 中间存在其他 Checkpoint，那么为 $ s\to t $ 投票也相当于为它们中间的这些 Checkpoint 投了票，这部分投票也不能有 double vote 存在。举个例子，下图中， $ a' $ 已经收集到了超过 $ 2\above{1pt}3 $ 的投票，这就意味着 $ a $ 没有收集到足够的投票，而如果此时 $ c $ 又被 Justified，就说明至少有超过 $ 1\above{1pt}3 $ 的 Validator 同时对 $ a $ 和 $ a' $ 投了票，它们的质押金将会被罚没。

![image.png](/assets/images/ethereum-pos-p3-img/image8.png)

由此我们可以得到一个结论：**一个高度** $ h $ **最多只能拥有一个 Justified Checkpoint。**

由两个 Slashing Conditions 我们就可以得出并证明 Casper FFG 的两个重要特性。**Accountable Safety 和 Plausible Liveness。**

# Casper FFG 的安全性与活性
---

## Accountable Safety
**Accountable Safety: **当满足拜占庭假设的前提下，不可能存在两个冲突的 Checkpoint 被 Finalized。即使没有满足拜占庭假设，也不能在不违反 Slashing Conditions 的情况下 Finalize 两个冲突的 Checkpoint。

> _与论文中不同，我在这里特别注明了要满足拜占庭假设。因为 Casper FFG 的安全性是分成满足和不满足拜占庭假设两种情况的。_
>

反证法：

如果存在两个冲突的 Checkpoint， $ a_m $ 和 $ b_n $ 被 Finalized，那就说明它们的直接子 Checkpoint，即 $ a_{m+1} $ 和 $ b_{n+1} $ 是被 Justified 的，换句话说存在 $ a_m\to a_{m+1} $ 和 $ b_n\to b_{n+1} $ 两个 Supermajority Link。那么这两组 Checkpoint 的高度一定满足 $ n > m+1 $ 或者是 $ m>n+1 $，我们假设 $ n > m+1 $，如下图。

![image.png](/assets/images/ethereum-pos-p3-img/image9.png)

而由于 $ b_n $ 是一个 Finalized Checkpoint，它也一定是某一个 Supermajority Link 的 Target， $ b_n $ 与 $ a_{m+1} $ 之间一定存在一个 $ b_i $ 使得 $ b_i $ 是 $ a_{m+1} $ 之后第一个 Target Checkpoint。如下图。

![image.png](/assets/images/ethereum-pos-p3-img/image10.png)


而 $ b_i $ 一定也是一个 Supermajority Link 的 Target，此时就可以推导出这个 Link 的 Source Checkpoint 一定在 $ a_{m} $ 之前了，这就与条件二不能发生 Link 嵌套冲突。如下图。

![image.png](/assets/images/ethereum-pos-p3-img/image11.png)

Accountable Safety 证明了在满足拜占庭假设的前提下共识协议的安全性无法被攻击，而不满足拜占庭假设时如果想要攻击 Checkpoint，就一定会违反 Slashing Conditions。也就是说，如果出现两个冲突的 Checkpoint 被 Finalized，那么一定存在超过 $ {1\above{1pt}3} $ 的 Validator 违反了 Slashing Conditions。

换句话说，**Casper FFG 作为 BFT 共识，恶意节点权重** $ <{1\above{1pt}3} $ **时一定具备 Finality。而 Slashing Conditions 使得恶意节点超过** $ {1\above{1pt}3} $ **时，也能具备经济学意义上的 Finality，称为 Economic Finality。**这是 Casper FFG 十分重要的特性，下文中会进一步介绍。

## Plausible Liveness
根据 Accountable Safety 我们知道，攻击安全性必须要违反 Slashing Conditions，而丧失活性则不需要，同传统 BFT 一样，Casper FFG 共识在恶意节点超过 $ {1\above{1pt}3} $ 时就会丧失活性。而在满足拜占庭假设的条件下，Casper FFG 实现了 Plausible Liveness。

**Plausible Livess: 在满足拜占庭假设的前提下，**永远可以添加 Supermajority Link 来 Justify 和 Finalize 一个 Checkpoint，但不是每轮一定有 Checkpoint 被 Finalized。

证明：

设 $ a $ 是最高的 Justified Checkpoint， $ b $ 是最高的拥有投票但 Unjustified 的 Checkpoint，两个 Checkpoints 可能不在同一个分支。此时永远可以存在一个 Checkpoint $ c $，高度满足 $ h(c)=h(b)+1 $，且 $ a\to c $ 是合法的（不和任何一个 Slashing Condition 冲突）。如下图所示。

![image.png](/assets/images/ethereum-pos-p3-img/image12.png)

由于 $ a $ 是最高的那一个 Justified Checkpoint，因此 $ c $ 不会是其他 Suparmajority Link 的 Target，没有违反 Slashing Condition 1。而 $ a\to c $ 之间仅存在一个 $ b $ 且 $ b $ 身上没有 Suparmajority Link，因此也没有违反 Slashing Condition 2。Plausible Liveness 得到证明。

Plausible Livenss 和真正的 Liveness 不同，首先 Capser FFG 的 Liveness 只是针对 Finalization 的活性，并非出块的活性。其次，Plausible Liveness 只是表示理论上我们永远有能力去 Finalize 一个 Checkpoint，但不代表我们这个 Epoch 一定有 Checkpoint 被 Finalized，系统可能很长时间都不会 Finalize 一个 Checkpoint。试想每次选择 Suparmajority Link 时如果胡乱选择，是可能不违反 Slashing Conditions 同时又没有 Checkpoint 被 Finalized 的。

为了解决这个问题，Casper FFG 需要一个对应的 Proposal Mechanism（具体来说是它的 Fork Choice Rule） 辅助，以保证 Finalization 的活性。在以太坊中也就是 LMD GHOST，即使恶意节点超过 $ {1\above{1pt}3} $ Liveness 也不会受到影响。

# Economic Finality，解决 BFT 的弱点
---

Accountable Safety 实现了 Economic Finality。

传统的 BFT 共识协议只有在满足拜占庭假设时才能保证 Safety，如果恶意节点超过 $ {1\above{1pt}3} $ 就会丧失这个特性，比如恶意节点可以组成 Censorship 向一半节点投票 A 分支，向另一半节点投票 B 分支，从而造成冲突的 Checkpoints 以回滚一些已经 Finalized 区块，这就是 [Part1](https://xufeisofly.github.io/blog/ethereum-pos-p1) 中提到的 Cartel Censorship Attack。现有的公链中大部分权力（无论是工作量资源还是权益资源）都被小部分 Cartel 所控制。因此拜占庭假设就显得过于生硬且不现实了，我们需要考虑如何在恶意节点超过 $ {1\above{1pt}3} $ 时依然能保证系统 Safety 和 Liveness。

在以太坊共识协议的设计中，这部分 Liveness 由 Fork Choice Rule 提供，而 Casper FFG 负责提供了额外的 Safety。这个额外的 Safety 并不是通过计算机科学保证的，而是基于经济学，具体来说就是引入惩罚机制。根据 Accountable Safety 我们得知，无论恶意节点有多大占比，只要其尝试进行分叉攻击以试图挑战某个已经被 Finalized 的  Checkpoint，就会与 Slashing Conditions 冲突。这是很容易检测的，那么我们就可以对其进行的巨额惩罚，并勒令其退出 Validator Set。

这种因高昂的攻击成本迫使节点变得诚实而实现的 Finality 称为 Economic Finality。**这个特性使得 Casper FFG 不允许「后悔」，一旦 Validator 投票给了一个分支，那么即使此时恶意节点权重突破** $ {1\above{1pt}3} $ **，也不允许改变之前的选择。**

这是 Casper FFG 一个重要的设计目标和特点，它使得在 Casper FFG 中进行 51% 攻击的成本大约是 7kw 美元，这是 PoW 不具备的（PoW 只是损失算力成本，没有这么大），只有 PoS 才能做到。

> _注意，这只是针对于已经 Finalized 的 Checkpoint 来说的，对于还没有 Finalized 的 Checkpoint，只能依靠 LMD GHOST 累积安全性。Part2 中提到了 LMD GHOST 安全性分析，即只要恶意节点数量没有超过 1/2，LMD GHOST 就有可能达到_ $ q_{min} $ _的安全阈值实现一定的 Finality。_
>

总结一下，Casper FFG 在恶意节点少于 $ {1\above{1pt}3} $ 时提供了基于计算机科学的安全性，超过 $ {1\above{1pt}3} $ 时提供了经济学的安全性，即使 Casper FFG 丧失了活性而无法进行新 Checkpoint 的最终确认，我们也可以利用 Fork Choice Rule 实现安全性的逐步累积，这个安全性的判断标准就会交由客户端制定。

> _容易误解的是，出现两个冲突的 Checkpoints 并不是针对 Casper FFG 的特有攻击，其他 BFT 共识比如 HotStuff 也会出现。只不过我们在使用传统 BFT 共识时一直默认满足拜占庭假设，因此不会分叉，也就不需要考虑两个完成提交的区块发生冲突时怎么办，应该如何选择，这就是为什么 BFT 不涉及 Fork Choice。但这是因为我们给了它一个很强的信任假设，而这个信任假设很可能是不切实际的。_
>

# Dynamic Validator Set，加入与退出
---

参与 Casper FFG 投票的 Validator Set 并不是一成不变的，Validator 应当随时可以加入或选择退出。但是动态的 Validator Set 会降低 Accountable Safety，因为可能被罚没的 Stake 会少于 $ 1\above{1pt}3 $。如果我们设 Economic Finality 为最多不会被罚没的 Stake 比重，那么对于一个纯静态的 Validator Set 为 $ 2\above{1pt}3 $，而动态的 Validator Set 为 $ {2\above{1pt}3}-ε $。

Casper FFG 论文中提到了动态的 Validator Set 的设计，这里提取了两点信息：

+ Validator 如果在 Epoch $ d $ 申请加入或退出，会在 Epoch $ d+2 $ 生效。
+ 一个 Validator 一生只能加入或退出一次，避免系统维护多个可投票时间段。

然而，以太坊 2.0 没有按照这个设计实现，而是写死参数限制了一个 Epoch 中 Validator Set 加入和退出的比例为 0.0015%。

# Casper FFG 对 Fork Choice Rule 的要求
---

Casper FFG 对其底层的 Fork Choice Rule 的要求为：

**Fork Choice Rule 必须选择拥有最高 Justified Checkpoint 的那个分叉。**

被 Justified 的 Checkpoint 就如同传统 BFT 共识中当中被 Locked 的提案一样，是一个节点对于该提案在本地视图下的认可，也是用于检测新提案是否与本地视图冲突的依据。如果新提案与本地 locked 提案冲突，则无法通过改节点的认证。因此不难想到，Fork Choice Rule 必须选择拥有最高 Justified Checkpoint 的那个分叉，以避免 Validator 验证时的冲突。

**因此，Fork Choice Rule 需要针对 Casper FFG 做适配**。

比特币的 Longest Chain Rule 就不能用于 Capser FFG，否则会因违反 Slashing Conditions 造成系统卡死。下图中，Checkpoint $ a $ 和 $ b $ 都没有完成 Justification，而 $ a' $ 被 Justified。如果此时使用 Longest Chain Rule，应该选择 $ c $ 作为下一个 Checkpoint，但由于 $ a' $ 已经收到了 $ 1\above{1pt}2 $ 的 Commit 投票，这就意味着上面的分叉不可能出现 Finalized 的 Checkpoint 了，除非有 $ 1\above{1pt}6 $ 的 Stake 被罚没，因此只能选择 $ b' $ 作为新的 Checkpoint。

![image.png](/assets/images/ethereum-pos-p3-img/image13.png)

这个例子再次印证了 Fork Choice Rule 必须从最高 Justified Checkpoint 之后选择 `HeadBlock` 的要求。对于以太坊来说，LMD GHOST 需要使用高度最高的 Justified Checkpoint 作为每次的 `RootBlock`。每次进行分叉选择时，LMD GHOST 都会从最高 Justified Checkpoint 开始逐步根据 GHOST 原则选择分叉，直到获取 `HeadBlock`。比特币的 PoW 如果想使用 Casper FFG 也应该进行类似的适配。

受 Casper FFG 影响，最终的 Fork Choice Rule 的执行过程如下：

1. 从 `HeadBlock`开始，初始为创始块。
2. 找到 `HeadBlock` 后代中已经 Justified 并拥有最多 Commit 投票的 Checkpoint。
3. 将这个 Checkpoint 设置为新的`HeadBlock`，然后回到第 2 步。
4. 当第 2 步中无法找到这个 Checkpoint 时，使用 Fork Choice Rule 进行 `HeadBlock`选择（例如 Longest Chain 或者 GHOST）。

# 思考：Casper FFG 对 BFT 共识的启发
---

网络上有很多 Casper FFG 和 PBFT 的对比，毕竟这是 Casper 的灵感源头，它们有很多相似也有很多不同之处，然而有些在我看来只是 PBFT 的缺陷而已，在其他的 BFT 共识中都有改进，比如 PBFT 不是 Pipeline 设计（也可以改造成 Pipeline 啊）等，这些并不是核心差别。我个人觉得最大的区别在于 Casper FFG **作为 Proposal mechanisim 上层的定位**，以及**惩罚机制的引入**，这从根本上解决了 BFT 类共识在恶意节点超过 $ 1\above{1pt}3 $ 时的 Liveness 和 Safety 缺失的问题。

## 需要一个 Proposal mechanisim
之所以需要一个 Proposal Mechanism 作为底层共识机制，是因为 Casper FFG 并不负责交易的共识，而是负责对区块 Finality 的共识。以太坊将我们常说的共识内容分为两个部分：**「交易共识」**和**「Finality 共识」**，「交易共识」负责令所有节点对某个交易（比如 Alice 转给 Bob 10 块钱）的执行结果（Alice - 10, Bob + 10）达成一致，而「Finality 共识」负责所有节点对某一个区块是否拥有 Finality（不可以回滚）达成一致。不难想到，「交易共识」对 Liveness 更加看重，因为需要优先保证能够不断出块，「Finality」共识对 Safety 更加看重，因为 Finality 本身就是 Safety 的重要性质。

如果去看一些基于 BFT 共识协议的区块链，会发现 BFT 共识协议将「交易共识」和「Finality 共识」的实现杂糅在了一起，很难区分，但这并不是 BFT 共识协议的问题，而是我们一直将其用作「交易共识」使用而恰好它又能提供 Finality 造成的。

BFT 类共识协议拥有巨大的作为「Finality 共识」协议的优势，但在「交易共识」最需要的 Liveness 特性上却有一个令人不爽的信任假设作为前提条件，这点前两篇文章反复提过，这里不再赘述。Vitalik 等人发现了这一点，因此主动将以太坊的共识机制分为两层，底层的 Proposal Mechanism 使用 PoS + LMD GHOST 负责交易的共识，保证「交易共识」的 Liveness，上层包裹一层 Finality Rule 使用 Casper FFG 间歇性地对已经完成「交易共识」的区块进行「Finality 共识」。

## 惩罚机制的引入
BFT 共识的安全性看似很高，但都是基于拜占庭假设这个大前提的，然而谁能保证现实中的恶意节点数量一定小于 $ 1\above{1pt}3 $ ？惩罚机制聪明地利用的共识中的 Stake 质押解决了这个问题，并重点解决了 Long Range Attack 和 Nothing at Stake 两种攻击场景。

其实我认为，即使满足了拜占庭假设，BFT 共识同样也有 Nothing at Stake 问题，由于 BFT 已经提交的块不分叉，这个问题主要体现在对未 Locked 提案的投票上。下图中我们以 HotStuff 举例（HotStuff 相关内容请见[文章](http://xufeisofly.github.io/blog/shardora-hotstuff)），假设某一时刻存在两个 Leader 各自打包了基于提案 3 的提案 4 和 4'，造成分叉，此时「聪明」的 Validator 就可以两个提案都投票（虽然按照 HotStuff 现有的逻辑不会给重复的同一个 ViewNumber 提案投票，但 Validator 可以自行去掉这段限制代码），这样无论哪个分叉最终被采纳，Validator 都能获得收益。这样 4 和 4' 都产生了 PrepareQC，区块链会继续分叉，并在试图同时拥有 LockedQC 时被检测出冲突而导致其中一个分支被抛弃，而 Validator 则能保证从这种 Nothing at Stake 行为中获得收益。

> _有人可能会说 HotStuff 对当前视图的 Leader 有校验，但是 Validator 完全可以主动去掉这个校验，而且还可能会有恶意 Leader 广播两个不同的提案主动引起分叉，这写都会造成 Nothing at Stake 问题。_
>

这种 Nothing at Stake 行为和 Chain-Based PoS 中的没有什么不同，是典型的 double vote，在 Casper FFG 中会被 Slashing Condition 检测和惩罚。目前我也在思考是否可以在 HotStuff 引入相同的惩罚机制来避免这种攻击行为。

![image.png](/assets/images/ethereum-pos-p3-img/image14.png)

下面图中简单绘制了传统 BFT 共识、中本聪共识以及 Casper FFG 的 Safety 和 Liveness 对比，并区分了是否满足拜占庭假设的情况。在满足拜占庭假设时，传统 BFT 共识拥有较高的 Liveness 和很强的 Safety，相比而言，中本聪共识的优势则是 Liveness，其 Safety 由于没有实现 Finality 而偏低一些，而 Casper FFG 兼具两者优势。但在恶意节点超过 $ 1\above{1pt}3 $ 后，BFT 共识便瞬间丧失 Liveness 和 Safety，而 Casper FFG 由于 Liveness 依赖 Proposal Mechanism 不受影响，而 Safety 由于惩罚机制实现了 Economic Finality 也能保证较强的安全性。

![image.png](/assets/images/ethereum-pos-p3-img/image15.png)

# 总结
---

Casper FFG 是一个基于 BFT 共识改进的二阶段共识协议，除了拥有传统 BFT 共识的特性之外，它还利用 Validator 的质押保证了不满足拜占庭假设时的安全性（Economic Finality）。

以太坊共识协议由 PoS + LMD GHOST（Proposal Mechanism） + Casper FFG（Finality Rule） 组成，分别实现了「交易共识」以及「Finality 共识」，并利用惩罚机制以及 Fork Choice Rule 分别解决了恶意节点超过 $ 1\above{1pt}3 $ 时 Safety 和 Liveness 丧失的问题 。

对比 PoW，以太坊的 PoS 更安全（拥有 Finality 和更高的攻击成本）、更快速（LMD GHOST 提升吞吐量）；对比 BFT 共识，以太坊的 PoS 活性更强（LMD GHOST 持续出块），整体安全性也更高（Casper FFG 的 Economic Finality）。

# 推荐阅读
---

+ [Casper FFG 历史，Vitalik 推文记录 ](https://hackmd.io/@liangcc/BJZDR1mIX?type=view)
+ [Essay: Casper the Friendly Finality Gadget](https://arxiv.org/abs/1710.09437)
+ [Book: Upgrading Ethereum](https://eth2book.info/capella/part2/consensus/casper_ffg/#fn-15)
+ [Casper FFG: Consensus Protocol for the Realization of Proof-of-Stake](https://medium.com/unitychain/intro-to-casper-ffg-9ed944d98b2d)
+ [Vlad 系列: The History of Casper](59233819c9a9)
+ [Vitalik: Minimal Slashing Conditions](20f0b500fc6c)
+ [Video: Casper CBC](https://www.youtube.com/watch?v=GNGbd_RbrzE)

