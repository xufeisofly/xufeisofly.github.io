---
title: 以太坊的 PoS - Part2 LMD GHOST
layout: post-with-content
post-image: "/assets/images/ethereum-pos-p2-img/head.jpg"
tags:
- ethereum
- consensus
- blockchain
---

# 共识协议是个组合

---

学习以太坊 PoS 时，可以发现所谓的共识协议不是一个单一的协议，而是以下三个部分的组合。

- **Propose Rule**: 区块发布规则，负责打包区块并发布到链上，比如以太坊的 Propose Rule 是 Proof of Stake，比特币是 Proof of Work，BFT 共识则是通过多轮投票机制对一个提案进行确认并提交。
- **Fork Choice Rule**: 分叉选择规则，负责在多个分叉中选择出权威链（canonical chain），使系统状态的一致性得到保证，同时也为系统提供了活性，。以太坊使用 LMD GHOST，比特币使用 Longest Chain，而 BFT 共识不会出现分叉，因此选择规则也无从谈起了。
- **Finality Rule**: 最终确定性规则，负责最终敲定一个区块，使其永远无法被回滚，为该区块提供了绝对的安全性。以太坊使用 Casper FFG，比特币则不存在 Finality Rule，只是随着子块越来越多而无限趋近 Finality，而 BFT 共识每一个区块的提交即确定，Finality 的实现是即时的。

可以发现，BFT 共识（例如前面介绍过的 HotStuff 或者 Tendermint）在这三个方面较为特殊，我们很难从中看出这三种规则，因为它们是浑然一体的。乍一看 BFT 共识协议简单清晰，不会分叉具有很高的安全性，同时每个块都具有最终确定性，但其最大缺点是需要满足恶意节点的权重 $<{1\above{1pt}3}$ 的拜占庭假设，否则就会丧失活性。相比安全性，活性是以太坊优先需要保证的（否则无法正常出块还谈什么安全性），为此以太坊使用了 LMD GHOST 作为 Fork Choice Rule。

# 一些前置概念

---

下面介绍一些以太坊中的基本概念便于理解后文。

- **Canonical Chain**: 权威链，被 Fork Choice Rule 选中的分叉即为权威链。
- **Validator Set**: 参与质押的节点即可成为一个 Validator，多个 Validator 组成一个 Validator Set 负责一个 Slot 的出块。每次出块时会从 Validator 中选出一个 Proposer 负责打包区块，这是根据质押金作为权重随机选取的，即 PoS 原理。
- **Slot**: Slot 是区块之间的最小间隔单位，相当于以太坊中的时间概念，共识是一个接一个 Slot 递进的。Slot 之间间隔为 12s，每个 Slot 出现时网络会尝试出块填充一个 Slot，因此每个 Slot 会对应一个 Validator Set 负责共识。实际上，以太坊会将整个网络的 Validators 分成 32 个 Validator Set，轮流对 32 个 Slot 出块，32 个 Slot 组成一个 Epoch，因此一个 Validator 在一个 Epoch 中只会参与一次出块。注意，Slot 和我们通常说的 Block height 不是一个概念，Slot 中可以有 Block，也可能没有 Block。
- **Root Block**: Root Block 是应用 Fork Choice Rule 时的开始区块，比如最近一次已经具有 Finality 的区块（在以太坊中即为 Checkpoint 区块）可以作为下一次的 Root Block，Genesis Block 是最初始的 Root Block。
- **Weight**: 当一个 Proposer 被选中发布了一个新块之后，Validator Set 中的其他 Validator 需要对该块进行验证并投票，投票者的质押金表示了该投票的权重大小，投票权重的累加值即为该区块的权重大小。权重越大表示该区块所在的链得到了越多的认可。

# GHOST，为提高比特币吞吐量而生

---

学习 GHOST 的论文是必不可少的，即使它又臭又长。

以太坊的 LMD GHOST 只是 GHOST 的一个变种，GHOST 全称为 The Greedy Heaviest-Observed Sub-Tree，其原本目的是解决比特币扩容后频繁分叉（块之间发生冲突）造成的安全性下降问题，以取代比特币的 Longest Chain 规则。其实在 GHOST 论文发出之时，以太坊已经采用了一个 GHOST 协议变体辅助 PoW 进行分叉选择了，这使得以太坊可以用较低的成本处理更多的交易，将比特币 10min 的出块时间压缩至 15s。

比特币的吞吐量是出了名的低，其设置的挖矿难度让出块时间维持在 10 分钟左右。这样做一是因为 PoW 基于一个同步模型，需要足够的间隔时间保证消息能被所有节点接收，二是为了避免产生大量分叉，导致算力分散降低共识安全性。如果要提高比特币的吞吐量，增大区块大小和降低出块时间是最自然的两个方法，不过这样就会令网络中出现大量冲突区块和分叉。导致分叉的原因有两个：

- 网络延迟。可能有两个节点同时基于相同的父区块发布了一个合法区块。
- 主动攻击。攻击者希望回滚交易，因此尝试主动分叉，一旦攻击链说服了其他节点，那么就可将权威链和上面的交易回滚，这也是 double spending attack 的常见方式。我们在上一篇文章中提到了 long range attack 其实就是分叉攻击的具体形式之一。

下图中，因提高吞吐量出现了大量区块冲突，产生分叉。对于比特币使用的 Longest Chain Rule 来说，攻击者 A 很容易发动攻击，恶意回滚交易。

![image.png](/assets/images/ethereum-pos-p2-img/image1.png)

攻击者 A 私自准备了一条攻击链，此时若使用 Longest Chain 作为权威链（图中蓝色区块所在链），攻击链 A 很容易就能成功替换掉它。这是因为分叉会导致算力的分散，不在最长链上的区块没有任何价值（high wastage and low security），因此攻击链算力不需要达到 50% 就可以完成攻击。

GHOST 论文中提出了新的 Fork Choice Rule 协议来解决上述问题，The Greedy Heaviest-Observed Sub-Tree，顾名思义，权威链的标准不再是 Longest Chain 而是 Heaviest Chain。当决定哪个分支时，GHOST 认为每个子块都是对其所有祖先的一次认可，因此分支的所有子块都会被纳入权重。如果每个子块的权重为 1，那么可以得到下图中的 GHOST chain 作为权威链（图中绿色区块所在链）。

![image.png](/assets/images/ethereum-pos-p2-img/image2.png)

具体证明过程请见论文，但其实可以这样想：如果一个块后面出现了两个子块（分叉），说明这两个子块在彼此竞争，但其实还存在另一个隐含信息，**即这两个子块都为其父块投了认可票。**这是 longest chain 规则没有考虑到的。

一言蔽之，GHOST 分叉选择规则没有选择「最长的链」作为权威链，而是选择了「最重的子树」。这是因为任何一个父块的子块不仅仅是对于该父块的一个直接投票（认可），也是对于它所有祖先的间接投票（认可），因此整个子树都应该纳入权重。

# GHOST 这么好为什么不用

---

GHOST 协议在原生的 PoS 或 PoW 中作为 Fork Choice Rule 并没有问题，但以太坊的 PoS 中增加了实现 Finality 的 Casper FFG 协议，这就不一样了。

先简单介绍 Casper FFG（下篇文章会详细介绍），在每共识若干个区块之后，系统会尝试产生一个 Checkpoint 块，这个块以及它所在分支之前的所有区块都被确定，无法被其他分叉回滚。Checkpoint 产生的条件是区块受到 ${2\above{1pt}3}$ 以上 Validator 的认可，即 > 66.6% Stake 权重的投票。而我们所说的 Fork Choice Rule 的使用范围一般是 Checkpoint 之后未确定的部分。因此，有可能 Validator Set 对于系统状态的最新认知和 GHOST 协议选择的分支是相互冲突的，如下图。

![image.png](/assets/images/ethereum-pos-p2-img/image3.png)

图中，绿色的分叉权重为 111，黄色为 96，根据 GHOST 规则绿色分叉会赢得胜利，然而黄色链的最后一个区块权重为 65，更接近成为 Checkpoint。为了解决这个问题，Vlad 和 Vitalik 分别提出了 LMD GHOST 协议和 IMD GHOST 协议，IMD GHOST 协议不是基于权重的加和而是基于权重的最大值进行分叉选择，这里对 IMD GHOST 不进行过多介绍，感兴趣可以看[这篇](https://ethresear.ch/t/immediate-message-driven-ghost-as-ffg-fork-choice-rule/2561?u=benjaminion) Vitalik 的文章。

# 以太坊的选择 —— LMD GHOST

---

这部分参考了 Vitalik 的[文章](https://vitalik.eth.limo/general/2018/12/05/cbc_casper.html)以及[这个](https://eth2book.info/capella/part2/consensus/lmd_ghost/#incentives-in-lmd-ghost)和[这个](https://hackmd.io/@Gua00va/B1x9vMQnwn)。

以太坊的 Fork Choice Rule 使用的是 LMD GHOST 协议，所谓 LMD（Latest Message Driven）就是指一个 Validator 最近一次针对某一个块的投票，我们称之为 attestation。下面是 attestation 的结构，`beacon_block_root` 字段为这个投票具体支持的区块。每个 Validator 都保存了所有验证者各自最新的那一次投票：

```rust
pub struct AttestationData {
    pub slot: Slot,
    pub index: u64,

    // LMD GHOST vote
    pub beacon_block_root: Hash256,

    // FFG Vote
    pub source: Checkpoint,
    pub target: Checkpoint,
}
```

当然，以太坊的 Propose Rule 还是 Proof of Stake，即根据 Stake 权重随机选取 Proposer。LMD GHOST 只是负责在出现多个分叉时根据一定规则告诉你要选择哪个分支，如果我们将 Fork Choice Rule 使用一个函数表示，那将是：

```go
function GetHead(Store) → HeadBloc
```

Store 是当前的状态数据，需要说明的是，Store 状态和时间有关，这么做的目的是实现区块的 Finality。一旦 Validator 确定了一个区块，即使之后发现了一个与之冲突的区块，也无法修改它之前的选择（个人猜测这使得以太坊最终选择了 LMD GHOST 而非 IMD GHOST，具体可看[这篇](https://ethresear.ch/t/immediate-message-driven-ghost-as-ffg-fork-choice-rule/2561?u=benjaminion)文章）。时间参量使得 Validator 做过的事情无法后悔，如果删掉 Store 仅提供创始块是无法恢复完整区块链数据的（见 Part1 中 Vlad 提供的建议）。

LMD 表示在选取 HeadBlock 考虑分叉权重时，只需要考虑每个 Validator 最近的一次投票，而不像 GHOST 那样考虑所有 Validator 历史上的全部投票，换句话说，LMD 仅考虑 Validator 最近一次投票的区块的权重并进行求和，而不会考虑整个子树上所有区块的权重。
当然即使是这样，当恶意节点超过一定阈值仍然可能会让分叉攻击成功，不过由于 LMD GHOST 还具有 sticky 性质，会使「突然叛变」行为容易被系统检测，从而能够对叛变者进行惩罚，这将在后文进行介绍。

## 分叉选择逻辑

下面举个例子介绍 LMD GHOST 协议选择分叉的基本逻辑。
假设网路中仅包含 A, B, C, D, E 这 5 个节点，为了简化过程，我们令它们的质押金相同（权重相同），且将一个 Epoch 分为 10 个 Slot，这样每个 Slot 的 Validator Set 只有一个节点。每个 Slot 出块后，只需要 Proposer 自己给自己投票就行了。下图中，5 个节点会轮流负责签名出块，形成一条分叉的 Block Tree：

![image.png](/assets/images/ethereum-pos-p2-img/image4.png)

图中，绿色的区块由于肯定包含了 Proposer 自己的 attestation，因此可以作为这 5 个 Validator 对应的 Latest Message。由于所有 Validator 的权重一致，因此每个区块算作对其所在分支的一份同意投票，每一个区块拥有的投票数量即为其直接子区块的投票份额累积，如下图。

![image.png](/assets/images/ethereum-pos-p2-img/image5.png)

当使用 LMD GHOST 决定权威链时，要从 root block 开始选择权重最大的路径，即 5→5→4→4→2→1。（这个例子中恰好是 Longest Chain，但这只是巧合）

![image.png](/assets/images/ethereum-pos-p2-img/image6.png)

为了更直观的理解这个过程，这里推荐[这个LMD GHOST源码项目](https://github.com/protolambda/lmd-ghost/tree/master)。下面摘取并注释了 `GetHead` 部分的代码：

```go
// 选择并获取权威链的叶子节点
func (gh *SpecLMDGhost) HeadFn() *dag.DagNode {
	head := gh.dag.Justified // 从 root block 开始查找
	for {
		if len(head.Children) == 0 { // 直到叶子节点
			return head
		}
		bestItem := head.Children[0]
		var bestScore int64 = 0
		for _, child := range head.Children {
			childVotes := gh.getVoteCount(child) // 选择投票权重最大的子块所在路径
			if childVotes > bestScore {
				bestScore = childVotes
				bestItem = child
			}
		}
		head = bestItem
	}
}

// 获取 block 权重
func (gh *SpecLMDGhost) getVoteCount(block *dag.DagNode) int64 {
	totalWeight := int64(0)
	for target, weight := range gh.latestScores { // 仅考虑各个 Validator 最新的一次投票
		if anc := gh.getAncestor(target, block.Slot); anc != nil && anc == target { // 找到投票的目标区块，并进行权重求和
			totalWeight += weight
		}
	}
	return totalWeight
}

/// 从 attestation 所在的区块往回找，找到其投票的目标区块
func (gh *SpecLMDGhost) getAncestor(block *dag.DagNode, slot uint64) *dag.DagNode {
	if block.Slot == slot {
		return block
	} else if block.Slot < slot {
		return nil
	} else {
		return gh.getAncestor(block.Parent, slot)
	}
}
```

## 权重计算

上面例子为了便于理解，令每个 Validator Set 中仅有一个节点，所以 Proposer 不需要额外收集投票。实际上每一个 Block 发布后需要收集 Validator Set 中其他验证者投票，并以投票权重之和作为该 Block 的最终权重。一个 Block 的权重等于该 Block 收到的 attestation 投票的权重总和，加上其子块提供的权重，并且在计算权重时仅考虑 Validator 最后一次 attestation，如下图中 Validator Set 为 A,B,C,D,E 5 个节点。

![image.png](/assets/images/ethereum-pos-p2-img/image7.png)

## Sticky 性质

LMD GHOST 具有 sticky 的性质：**节点一旦给某一个分支投票就倾向于继续为这个分支投票，除非一定比例的节点发生突然叛变（会收到惩罚）。**比如下图中 A,C 同时叛变到 B 的攻击链上时，这条攻击链才有可能成功。
下面是 Vitalik 举的例子。

![image.png](/assets/images/ethereum-pos-p2-img/image8.png)

还是 A,B,C,D,E 5 个节点分到了 10 个 Slot 中，且 Stake 相同，因此区块本身即可表示一个权重为 1 的投票。图中的 B 作为恶意节点尝试独自构建一条攻击链，正常情况下，剩余 4 个诚实节点并不会买账，会根据 LMD GHOST 继续选择下面的一条链作为权威链（canonical chain）。
图中，每个节点都分别进行了两次出块投票，我们只关注第二次出块时的分叉选择，并查看该节点在选择分叉时的视图情况如下图。

![image.png](/assets/images/ethereum-pos-p2-img/image9.png)

图中，绿色表示当前 Proposer 要打包的区块，蓝色表示所有 Validator 最新产生的区块（投票），所以蓝色的块决定了绿色块即将选择的分支。比如在 A 的视图中，当 A 尝试进行第二次投票时看到的是 B 投给了上面的链，A, C, D, E 投给了下面，因此 `Bottom : Top = 4 : 1`。

此处说明一下，D 和 E 的视图中之所以没有从 C 第二次投票之后继续出块可能是因为 D 和 E 并没有看到 C 的第二个块（网络延迟等原因），就算看到了也不会影响最后的分叉选择。这里也说明了即使是诚实的 Validator 也有可能没有把票投给最终的合法链（因此不应当受到惩罚）。

从这个过程就可以发现，诚实节点更倾向于沿着之前的权威链继续出块，而没有动力去频繁更换分支的选择，除非此时有两个节点发生「恶意叛变」，投票给上面的链，但这很容易被发现并收到惩罚。

![image.png](/assets/images/ethereum-pos-p2-img/image10.png)

> we can very generally say that any validator's new message will have the same opinion as their previous messages, unless two other validators have already switched sides first.

## 安全性分析

安全性由 Finality 决定，以太坊使用了 Casper FFG 作为 Finality Rule，这是一个 BFT 共识机制，像 Tendermint 一样，我们使用两轮投票实现一个区块的 Finality。然而在没有 Casper 的情况下，LMD GHOST 的是否可以提供部分安全性，又是如何量化的？
我们定义一个参数$q$，称为安全值，表示一条分支链得到 Validator 投票的权重与总 Validator 权重的比值。例如，5 个 Validator 中有 4 个 选择该分支，那么$q = 0.8$。我们设$q_b$为$b$这个块的安全值，那么从$b$块发布开始，随着其子树上面的投票权重越来越大，$q_b$也会越来越高。一般认为当安全值大于某个阈值，即$q_b > q_{min}$时，块$b$成为了一个真正「安全的」区块，无法被攻击回滚。如下图。

[![pAAY5rD.png](https://s21.ax1x.com/2024/08/29/pAAY5rD.png)](https://imgse.com/i/pAAY5rD)

对于 LMD GHOST 来说，阈值$q_{min}$被定义为 $β + {1\above{1pt}2}$，其中 $β$ $β$表示恶意节点控制的 Stake 权重值，一般来说认为小于 ${1\above{1pt}3}$。也就是说当 Validator 全部是诚实节点时，只要块 $b$ 收获一半 Validator 的认可，该块就会被「最终确认」。

将 Finality Rule 和 Fork Choice Rule 解耦最大的好处就是以太坊在保证活性的前提下，即使区块的 Finality Rule 执行失败（如恶意节点数量 $>{1\above{1pt}3}$），也可以由客户端自己判断当前安全性。如果客户端觉得当前区块的 $q$ 值过小，可以选择继续等待，等待 Casper FFG 执行成功或者 $q$ 值达到了自己的要求。客户端使用自己规则来定义 Finality 标准，而不用在协议中设置一个阈值常量。

# LMD GHOST 的激励和惩罚

---

以下内容目前片面地参考了这篇[文章](https://eth2book.info/capella/part2/consensus/lmd_ghost/#incentives-in-lmd-ghost)，笔者之后会查阅源码和其他资料以保证准确性。

## 激励

共识协议不仅仅是一个计算机科学问题，更是一个经济学问题。我们通过经济学手段鼓励诚实行为，惩罚恶意行为，最终使共识结果安全可信。这就像是一个社会的运行，除了使用科技和法律的手段进行治理，也要运用宗教、道德进行进一步约束。
区块的 Proposer（发布区块） 和 Validator（验证区块并投票） 如果诚实地选择权威链后应当进行激励，否则进行惩罚。其中 Proposer 诚实选择分叉的动力显而易见：如果选择了错误的分叉，它发布的区块最终将大概率被遗弃，导致无法获得出块奖励，因此对 Proposer 分叉选择的激励是间接的。而 Validator 在收到区块后，如果验证区块合法将会对该区块签名投票，这个签名将会出现在下一个 Slot 的区块中作为诚实的证据，该 Validator 会因此直接收到一小部分奖励，而收录该 Validator 签名的 Proposer 也会按比例获得其中一部分激励。

> validators are directly rewarded for voting accurately. When a validator makes an accurate head vote, and its attestation is included in a block in the very next slot, it receives [a micro reward](https://eth2book.info/capella/part2/incentives/rewards/#introduction). A perfectly performing validator will gain about 22% of its total protocol rewards from making accurate head votes. Proposers, in turn, are incentivised to include such attestations in blocks as they receive a proportionate micro, micro reward for each one they manage to get in.

_个人认为这里可以用来解决 HotStuff 中 Leader 恶意控制投票节点列表的问题：由于在 HotStuff 中哪些节点投了票完全是 Leader 一个人说的算，因此 Leader 在凑足 ${2\above{1pt}3}+1$ 的投票后可以故意的排除一些自己不喜欢节点，包含一些自己喜欢的节点的投票。而令 Leader 一部分激励来收录节点的投票或许可以解决这个问题。_

说明一下，如果 Validator 投票的区块最终不是那条合法链，Validator 也不会收到惩罚，因为由于网络延迟等原因 Validator 是有可能没有看到足够信息就进行投票的，贸然惩罚是不公平的。

## 惩罚

LMD GHOST 会对 nothing at stake 行为进行惩罚。在 [Part1](https://xufeisofly.github.io/blog/ethereum-pos-p1) 文章中提到，nothing at stake 是指 PoS 共识过程中，由于 Validator 对某分支投票没有任何成本（只需要签名，不像 PoW 需要消耗大量算力），因此可以对所有的分支都进行投票来规避投错票的风险。这种行为是易于检测的，一旦被发现，系统会罚没该 Validator 的部分 Stake，并强制该节点退出 Validator Set。

Proposer 同样存在 nothing at stake 行为，一个 Proposer 可能选择同时选择多条分支打包区块而不是只选择权威链，以避免日后自己选择的分支被回滚的风险。与 Validator 的 nothing at stake 惩罚不同，Proposer 的此类行为是通过第三方检测的，第三方发现后会提交一个证明，由之后的区块发布时中会打包，完成系统对 Proposer 的罚没。

# 总结

---

共识协议由 Propose Rule、Fork Choice Rule 以及 Finality Rule 组成。以太坊的 Fork Choice Rule 使用了 LMD GHOST 协议，放弃了简单的 Longest Chain 作为权威链选择的规则，这在吞吐量提高导致出现大量分叉时确实能提供一定的安全性。另外 Fork Choice Rule 协议本身为系统提供了活性，这与 BFT 共识不同，即使恶意节点数量超过某个阈值，安全性大幅降低，也不会影响系统正常出块，这本质上是因为 Chain-Based 共识协议不像 BFT-Based 共识协议那样有一个需要凑足 ${2\above{1pt}3}$ 权重的强阈值要求。

然而网络延迟或恶意攻击导致频繁分叉不可避免，造成系统的安全性不是一成不变的，会不断发生波动（如 attestation 最分散时攻击者作恶的难度最低）。攻击者可以早早准备好一个恶意区块，然后等待一个系统安全性最弱的时机发布出去，这就是 Long Range Attack。为了解决这个问题以太坊引入了 Casper FFG 协议为共识提供了 Finality，我将在下一篇文章详细讲述。

# 推荐阅读

---

- [Run your own Ethereum Validator (Part 1)- PBFT, Casper FFG, Casper CBC and LMD GHOST](https://medium.com/coinmonks/run-your-own-ethereum-validator-part-1-pbft-casper-ffg-casper-cbc-and-lmd-ghost-e68e8461acf8)
- [Upgrading Ethereum](https://eth2book.info/capella/part2/consensus/lmd_ghost/#finding-the-head-block)
- [An in Depth Look at the GHOST Protocol: Part 2- How GHOST Works](https://medium.com/coinmonks/an-in-depth-look-at-the-ghost-protocol-part-2-how-ghost-works-50e49edf7a52)
- [A CBC Casper Tutorial](https://vitalik.eth.limo/general/2018/12/05/cbc_casper.html)
- [Ethereum 2.0 Phase 0 -- Beacon Chain Fork Choice](https://github.com/ethereum/annotated-spec/blob/master/phase0/fork-choice.md)
- [Github:protolambda/lmd-ghost](https://github.com/protolambda/lmd-ghost/tree/master)
