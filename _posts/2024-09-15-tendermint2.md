---
title: 从 HotStuff 回看 Tendermint (二)
layout: post-with-content
post-image: "/assets/images/tendermint2-img/head.jpg"
tags:
- tendermint
- consensus
- blockchain
---

本篇文章是阅读了 Tendermint Core 源代码之后，对 Tendermint 理解更新。包括共识协议的流程设计和实现细节。

此外，建议读者阅读[这篇论文](https://tendermint.com/static/docs/tendermint.pdf)，它涉及了 Tendermint Core 共识部分的具体实现，而原始的 [Tendermint 论文](https://arxiv.org/abs/1807.04938)更多是理论上的。读者还可以阅读我[上一篇](https://xufeisofly.github.io/blog/tendermint)关于 Tendermint 的文章，两篇文章无论先后，希望能有所帮助。

本系列共有三篇文章，此为第二篇。

+ [从 HotStuff 回看 Tendermint (一)](https://xufeisofly.github.io/blog/tendermint)
+ [从 HotStuff 回看 Tendermint (三)](https://xufeisofly.github.io/blog/tendermint3)

在进入正题之前先谈谈一个最近的小思考。

# 如何判断一个共识协议的水平
---

在研究了 Tendermint、HotStuff、以太坊 Gasper 之后，我认为评估共识协议水平的标准为以下四个方面：

+ 信任假设的强度。
+ 安全性 Safety。
+ 活性 Liveness。
+ 吞吐量 Throughput

每个共识协议无论是中本聪共识还是传统的 BFT 共识，都存在信任假设，不谈其基于什么信任假设而只谈共识有多安全多快速完全是在耍流氓。信任假设越强，说明共识协议对环境的要求越高，这可不是好事。

比如某人说自己的 BFT 共识多么安全，但你要看到它的恶意节点数量必须小于 1/3；有人夸比特币 PoW 多么安全，但你要看到它要求网络必须是同步的，必须使用很长的出块时间提供保证。不同的信任假设会造成不同特性的牺牲，BFT 的拜占庭假设牺牲了特定条件的活性和安全性，比特币的同步模型假设牺牲了大量性能。我在[上一篇文章](https://xufeisofly.github.io/blog/tendermint)中提到过同步模型和部分同步模型的区别，它们都需要设置网络通讯上限 Timeout，如果部分同步模型的 Timeout 过短，只是会短暂的失去活性，但如果同步模型的 Timeout 过短，就会彻底丧失安全性。所以说 PoW 的信任假设也是很强的。

安全性 Safety 是共识的基础，我个人将它分为以下两点：

+ Consistency：一致性，它表示整个系统状态完成收敛的能力，首先必须能够收敛，其次要评估收敛速度，长时间发生分叉即使最后能达成共识也是安全性低的表现。
+ Finality：最终确定性，它体现了一个已经完成共识的区块是否安全，不会被恶意回滚。

活性 Liveness 是指一个共识协议是否能持续不断的产生共识，不会因为某个原因被长时间的卡住。安全性和活性都是一个共识协议必须要满足的，只是不同场景下的强弱可能有所不同，这个场景我们需要结合实际来分析，比如 BFT 共识中恶意节点小于 1/3 是否符合实际。如果在一个现实场景中丧失了其中一个特性，共识协议就无法使用。

最后就是吞吐量，在共识协议在需求场景中可以使用的前提下，自然是吞吐量越高越好。

下面进入正题。

# Tendermint 状态机：文档和源码的差异
---

Tendermint 的状态流转过程基本上都是基于下面这张官方图，具体在[上一篇文章](https://xufeisofly.github.io/blog/tendermint)中有详细介绍，这里就不再重复了。

![image.png](/assets/images/tendermint2-img/image1.png)

通过源代码总结出来了当前版本 Tendermint 状态机示意图如下，我认为描述更为全面。

![image.png](/assets/images/tendermint2-img/image2.png)

上图中，绿色和红色箭头分别代表 Happy Path（正常完成共识）和 Unhappy Path（共识失败导致视图切换），+2/3 Block、+2/3 Nil 代表对应的阶段的同意票和反对票，+2/3 Any 表示任意投票。图中呈现出了以下几种阶段流转:

+ `NewHeight` -> `NewRound`：开启一个新的 `Height` 意味着上一次共识成功，要对一个全新的 `Block` 进行共识，而一个 `Block` 的共识可能需要多个 `Round`，因此在进入 `NewHeight` 阶段一定时间之后（`CommitTimeout`）会自动开始一个 `NewRound`。这段时间可以帮助节点收集更多的 `Precommit` 投票，保证 `NewRound` 是从上一个 `Height` 的 `CommitTime` 结束后的 `CommitTimeout` 时间开始的。其实，`NewHeight` 就是 Tendermint 的 ViewChange 阶段，通过收集足够的 `Precommit` 投票完成视图切换。
+ `NewRound` -> `Propose`：从 `NewRound` 状态到 `Propose` 状态有两种方式，一种是当交易池中的 `Txs` 准备好时进入 `Propose` 阶段。另一种是开启 `NewRoundTimeout` 计时，倒计时结束后 Proposer 会尝试打包一个新提案，若 `Txs` 还没有准备好则提案就会是一个空的区块。
+ `Propose` -> `Prevote`：进入 `Propose` 阶段后，Proposer 会将打包好的提案广播给其他 Validators，同时开始计时 `ProposeTimeout`，Validator 会在收到 `Proposal` 之后，或者是 `ProposeTimeout` 结束时刻进入 `Prevote` 阶段。
+ `Prevote` -> `Precommit`：进入 `Prevote` 阶段后 Validator 会对 `Proposal` 进行验证并投票`Prevote for *`，投票有可能是 Yes 投票（`Prevote for Block`）或者是 No 投票（`Prevote for Nil`）。Validator 在投票后开始收集其他投票，并在收集到 +2/3 任意投票后开启 `PrevoteTimeout` 计时。接下来有三种情况会使得节点进入 `Precommit` 阶段：
    a. 收集到 +2/3 的 Block 投票：会导致后续 `Precommit` 投同意票。
    b. 收集到 +2/3 的 nil 投票：会导致后续 `Precommit` 投反对票。
    c. `PrevoteTimeout` 到时：会导致后续 `Precommit` 投反对票。
+ `Precommit` -> `NewHeight`：进入 `Precommit` 阶段后 Validator 会根据 `Prevote` 收到的投票情况发起 `Precommit` 投票，并收集其他节点发来的 `Precommit` 投票。当收集到 +2/3 的任意投票后开启 `PrecommitTimeout` 计时，之后继续收集投票，并分为三种情况：
    - 收集到 +2/3 的同意票：`Commit` 本提案，并进入 `NewHeight` 阶段，本次提案共识成功。
    - `PrecommitTimeout` 到时：本 `Round` 未共识成功，开启一个 `NewRound`，`Height` 不变。
    - 收集到了所有 Validator 的投票：开启一个 `NewRound`，`Height` 不变。

代码流程图和官方提供的流程图相比，展现内容上有一些区别：

1. 代码中，`NewHeight` 之后必会进入 `NewRound`，原图中 `NewHeight` 直接跳过了 `NewRound`。
2. Tendermint 从 `NewHeight` 到 `NewRound` 期间要视图收集所有 Validator 的 `Precommit` 投票，原图中并没有体现这一点，而这是视图切换的核心逻辑。
3. 原图中没有体现出各个阶段的 `Timeout`，但这是部分同步网络下 Tendermint 能够实现的重要基础。这里我们去掉代码流程图中的多余逻辑只关注不同阶段流转时的超时，如下图所示。

![image.png](/assets/images/tendermint2-img/image3.png)

如果一个 `Round` 共识失败，那么下一个 `Round` 中发生超时的 `Timeout` 会比之前的 `Round` 增加一个固定的时间，来保证 Tendermint 共识协议在部分同步网络中的活性，避免卡死。

# 视图切换：尽量收集所有 Precommit 投票
---

在 `Precommit` 阶段中如果成功收集到 +2/3 `Precommit for Block` 投票（称为 Happy Path）并完成 `Commit`，会立刻进入 `NewHeight` 阶段，可以说 `NewHeight` 阶段就是 Tendermint 的视图切换阶段。`NewHeight` 阶段需要等待 `CommitTimeout` 时间用于更多的收集 `Precommit` 投票才可以进入 `NewRound` 阶段，开启新一轮共识。代码中具体表现为：

```go
// 执行 Commit
func (cs *State) finalizeCommit(height int64) {
	// ...
	// 此时已进入 NewHeight 阶段
	// 计算 NewHeight Start 时间：StartTime = timeoutCommit + LastBlockTime
	cs.StartTime = cs.config.Commit(cs.CommitTime)
	// 需要等待 timeoutCommit 时间才进入 
	sleepDuration := rs.StartTime.Sub(tmtime.Now())
	cs.scheduleTimeout(sleepDuration, rs.Height, 0, cstypes.RoundStepNewHeight)
	// ...
}

// 接收 Timeout 消息
func (cs *State) handleTimeout(ti timeoutInfo, rs cstypes.RoundState) {
	// ...
	switch ti.Step {
	case cstypes.RoundStepNewHeight:
		// 进入 NewRound 阶段
		cs.enterNewRound(ti.Height, 0)
	// ...
    }
	// ...
}
```

上述代码能看到 `NewHeight` 阶段与 `NewRound` 阶段之间存在的 `CommitTimeout` 等待，这是为了更多的收集剩余的 `Precommit` 投票。如果没有收集到 +2/3 `Precommit for Block`，即 Unhappy Path 中，节点也会等待 `PrecommitTimeout` 时间才会进入 `NewRound` 阶段，同样是为了收集更多的 `Precommit` 投票。

实际上，这部分「多余的」等待并没有在 Tendermint 论文中提现，当人们按照论文实现了 Tendermint 之后发现会出现死锁的情况。那么为什么要尽量收集剩余的 `Precommit` 投票？这与 Tendermint 的视图切换过程相关。下图为 Tendermint 的视图切换过程：

![image.png](/assets/images/tendermint2-img/image4.png)

这个例子也在 [Part1 文章](https://xufeisofly.github.io/blog/tendermint)中介绍过。图中所描述的过程如下：系统存在 1，2，3，4 四个 Validator，其中 1 为 Leader 或 Proposer，4 是一个拜占庭节点，可能存在恶意行为。在一轮正常的共识的 Prevote 阶段中，节点 2，3 没有收到 +2/3 的 `Prevote for Block` 投票，因此没有产生最新的 `LockedQC`（`LockedQC` 是 HotStuff 的概念，在 Tendermint 中即为 `LockedBlock`，在 Casper FFG 中即为 Justification，为了方便，后面统一使用 LockedQC），节点 1，4 成功产生 `LockedQC`。很显然，此时由于 `LockedQC` 不够 +2/3 共识过程陷入僵局，需要进行视图切换更换 Leader 重新发起共识。

与 PBFT 和 HotStuff 不同，为了实现 $ O(n) $ 通讯复杂度的视图切换，Tendermint 的需要收集到所有的 Validator 的 `Precommit` 投票，这样才能保证新 Leader 获得最新的 `LockedQC`（从节点 1 中获得），从而使下一个 Round 打包一个不与 `LockedQC` 冲突的 `Block` 新提案。

接下来我们看下如果没有收集到所有的 Precommit 投票会怎么样？

不妨假设新 Leader 即节点 2 没有收到节点 1 的包含 Locked 信息的 `Precommit` 投票，于是新 Leader 就会认为上一个 `Round` 中 `Proposal` 提案并没有生成 `LockedQC`，我应该重新发起一个全新的 `Block` 作为提案。但由于节点 1 已经 Lock 了之前的提案，因此新一个 `Round` 的全新提案不会被它验证通过，节点 1 会对新的提案投反对票。如下图。

![image.png](/assets/images/tendermint2-img/image5.png)

验证提案的代码如下，节点 1 会因 `LockedBlock` 冲突而拒绝新提案，验证逻辑我们在下文中会详细介绍。

```go
func (cs *State) defaultDoPrevote(height int64, round int32) {
	logger := cs.Logger.With("height", height, "round", round)

	// 已经有 Locked Block，则会直接为 Locked Block 投票，若此时新提案与 Locked Block 不同，
	// 则相当于对新提案投了反对票。
	if cs.LockedBlock != nil {
		// 为 LockedBlock 签名投票
		cs.signAddVote(tmproto.PrevoteType, cs.LockedBlock.Hash(), cs.LockedBlockParts.Header())
		return
	}

	// ...
}
```

可以看到，如果 Validator 此时已经有 `LockedBlock`，则会直接为 `LockedBlock` 投票，若此时新提案与 `LockedBlock` 不同，相当于对新提案投了反对票。

节点 1 反对新提案，节点 2，3 同意新提案，节点 4 是恶意节点可能会进行冲突的投票。后续过程可能如下：节点 1 认为系统状态是 B，节点 2，3 认为系统状态是 A，节点 4 是恶意节点可能会发送不同的系统状态。那么由于节点 4 的存在，经过 `Prevote` 阶段后仅有节点 2 和 4 成功产生了 `Lock for A`，考虑到 4 是恶意节点，可以认为只有节点 2 产生了 `Lock for A`。如下图。

![image.png](/assets/images/tendermint2-img/image6.png)

> 多说一句，如果只有一个阶段此时共识已经分叉了，这也就是为什么 BFT 协议最少有两个阶段的原因。
>

接下来所有节点进入 `Precommit` 阶段，所有会广播自己的 `Precommit` 投票，节点 2 广播 `vote for A`，其他节点广播 `vote for nil`。并最终由于无法达成共识而进行视图切换。发现了吗，这就又回到了一开始的情况，即仅有一个节点拥有 `LockedQC`，视图切换尝试收集到它的 `LockedQC`（即 `Precommit` 投票）。

这说明，**只要在收集投票时恰好没有收集到唯一拥有 `LockedQC` 那一个节点的 `Precommit` 投票，恶意节点就可以通过拜占庭行为导致共识丧失活性，所以要尽量收集所有的 `Precommit` 投票。**这就是为什么 Tendermint 需要设计一个 `CommitTimeout` 和 `PrecommitTimeout`，保证收集到那个拥有 `LockedQC` 的节点的 `Precommit` 投票，因为如果 `Timeout` 时间过短，就会降低系统活性，甚至有可能陷入无限循环，彻底丧失活性。因此 `Timeout` 时间需要合理选择，代码中是会动态进行修正的。

有趣的是，在 Happy Path 情况下，Tendermint 代码额外提供了一个路径进入 `NewRound` 阶段，即收集到所有的 `Precommit` 投票。代码中提供了一个 `SkipTimeoutCommit` 开关，当 `SkipTimeoutCommit == true` 时，只要节点已经收集到所有的 `Precommit` 投票，就可以无需等待 `CommitTimeout` 时间结束，直接开启一个 `NewRound`，完成视图切换，代码如下：

```go
// 接受 precommit 投票
func (cs *State) addVote(vote *types.Vote, peerID p2p.ID) (added bool, err error) {
	// ...
	// 如果收到了 +2/3 的 Precommit 投票
	blockID, ok := precommits.TwoThirdsMajority()
	if ok {
		// ...
		if len(blockID.Hash) != 0 { 
			// 如果 +2/3 是同意票，则进入 Commit 阶段，提交提案
			cs.enterCommit(height, vote.Round)
			// 如果收集到所有的 precommits，直接开启 NewHeight
			if cs.config.SkipTimeoutCommit && precommits.HasAll() {
				cs.enterNewRound(cs.Height, 0)
			}
		} else {
			// 如果收集到了 +2/3 是 nil 票，则进入 precommitTimeout，尽量收集更多的 Precommit 投票
			// 会在 precommitTimeout 结束时发起 NewRound 的
			cs.enterPrecommitWait(height, vote.Round)
		}
	}

	// ...
}
```

这么做的原因不难理解，一旦收集到所有的 `Precommit` 投票，就一定可以收集到拥有最新 Locked 状态节点的投票，那么新的提案就能避免和这个节点发生冲突，导致系统丧失活性。我们常说 Tendermint 由于要等待固定的 `Timeout` 而丧失了 Responsiveness，这应该也是它增加 Responsiveness 的一个手段。

# Round-Based 设计与 ValidBlock 概念
---

这部分源于我对 Tendermint 协议中 `ValidBlock` 和 `LockedBlock` 的疑惑，为此查看了[论文](https://arxiv.org/abs/1807.04938)和[源码](https://github.com/tendermint/tendermint/blob/main/consensus/state.go)实现。

与 PBFT 和 HotStuff 不同，Tendermint 的论文中除了存在 Locked 概念还有 Valid 概念，具体来说是 `LockedBlock` 和 `LockedRound`，以及 `ValidBlock` 和 `ValidRound`。查看源码会发现，这两个变量基本都是在一个 `Block` 收集到 +2/3 的 `Prevote for Block` 投票后设置的，让人疑惑它们之间的区别。在 HotStuff 中当一个提案产生了 `LockedQC`，新的提案不能与之发生冲突，根据这个经验，不难想象 Tendermint 中的 `LockedBlock` 是跟分叉冲突检测有关，实际也确实如此：当一个 Validator 产生了一个 `LockedBlock`，则该 Validator 只会处理该 `LockedBlock` 的相关投票和验证，不会理会其他的提案，除非 `LockedBlock` 应该被更新的提案更换，并提供证明。

但 `ValidBlock` 和 `ValidRound` 呢？接下来介绍 Locked 概念和 Valid 概念在 Tendermint 中的作用，以及为什么多需要了一个 Valid 概念。

## Round-Based 设计和 Valid 概念
和其他 BFT 共识协议不同的是，Tendermint 不仅仅有 `Height`，同一个 `Height` 还有多个 `Round`。`Height` 表示这个位置需要共识出一个提案，而 `Round` 表示对该 `Height` 进行共识时，由于存在网络问题，可能需要多个 `Round` 才能成功。如下图，Height N 的共识从 `Round0` 开始，一直到 `Round3` 才提交成功。

![image.png](/assets/images/tendermint2-img/image7.png)

当一个提案收到 +2/3 的 `Prevote for Block` 投票，说明这个提案被系统认可，有可能被成功提交，这些提案在论文中被称为 Possible Decision，会被设置到 `ValidBlock` 变量当中，而对应的 `Round` 会被设置为 `ValidRound`，因此 `ValidBlock` 是一个 Possible Decision，最终被提交的提案一定会从中选出。

一个 `Round` 中，如果之前没有产生过任何 `ValidBlock`，那么 Proposer 打包的提案是全新的，但如果已经有了 `ValidBlock`（可能是之前的 `Round` 得到了认可但没有成功提交的提案），就会将该 `ValidBlock` 作为本 `Round` 的提案，发起共识。原因也很好理解，为了尽量成功，肯定优先选择打包一个最可能完成提交的提案，而 `ValidBlock` 之前已经被证明至少可以获得 +2/3 `Prevote for Block `投票认可了，成功概率较大。另一个原因是我们需要打包一个可以通过 Validator 验证的提案，而这个提案一定是从 `ValidBlock` 当中选择，且最新的那个 `ValidBlock` 最有可能。因为对一个提案的验证需要满足两个条件：

+ 提案本身内容合法。
+ 不能与 `LockedBlock` 冲突。

所以说，同一个提案之前 `Round` 可以通过验证不代表这一次也可以通过验证，因为 `LockedBlock` 可能变了。

如果 Proposer 打包了 `ValidBlock` 作为提案，它同时会将对应的 `ValidRound` 也打包进来，作为一个证明，告诉其他 Validator 当前这个 `ValidBlock` 之前在哪个 `Round` 中收到过 +2/3 的 `Prevote` 投票。因此一个 `Proposal` 可以简化成 `<ValidBlock, ValidRound> `的 Pair，由一个已经得到过认可的 `Block` 及其对应的 `Round` 组成。当然，如果之前没有 `ValidBlock`，如 `Round0`，则 `Proposal` 为 `<NewBlock, -1>`。实际上，`ValidRound` 在 `Proposal` 中的字段叫做 `POLRound`（POL 表示 Proof of Lock），顾名思义这是一个证明，证明当前提案是可以被 Lock 的。因此 `POLRound` 的用处就是试图令当前的 `Block` 被 Validator 接受并锁定，替换掉其之前的 `LockedBlock`。

我们看一下 `Proposal` 的结构和打包代码	。

```go
type Proposal struct {
	Type      tmproto.SignedMsgType
	Height    int64     `json:"height"`
	Round     int32     `json:"round"`    
	POLRound  int32     `json:"pol_round"` // -1 if null.
	BlockID   BlockID   `json:"block_id"`
	Timestamp time.Time `json:"timestamp"`
	Signature []byte    `json:"signature"`
}
```

```go
proposal := types.NewProposal(height, round, cs.ValidRound, propBlockID)

func NewProposal(height int64, round int32, polRound int32, blockID BlockID) *Proposal {
	return &Proposal{
		Type:      tmproto.ProposalType,
		Height:    height,
		Round:     round,
		BlockID:   blockID,
		POLRound:  polRound,
		Timestamp: tmtime.Now(),
	}
}
```

一个 `Proposal` 收到 +2/3 的 `Prevote` 投票后，就会产生一个 POL，类似于 HotStuff 的 `LockedQC`，然后会记录下来此时的 `Block` 和 `Round`，即 `ValidBlock` 和 `ValidRound`，而只要它们满足条件，就会被 Validator 所 Lock，否则 Validator 不会认可这个提案。

## Locked 概念
一个 Validator 进入 `Precommit` 阶段发送 `Precommit for Block` 时，就把该提案设置为 `LockedBlock`，当前 `Round` 设置为 `LockedRound`，用于验证后续的提案。因此 `LockedBlock` 一定是一个 `ValidBlock`，而且是最新的那个 `ValidBlock`。

最终能够 Commit 的提案一定是 `LockedBlock`，一旦一个提案被一个 Validator Lock，就意味着该 Validator 从历史上众多 `ValidBlock` （Possible Decision）中选择了一个它最支持的提案，从此轻易不会给其他其他投票，除非有足够的证据证明它应该改变 `LockedBlock`。

一个新 `Round` 的新提案被 Validator 接受（改变 `LockedBlock`）只有两种情况：

1. Validator 发现了一个比 `LockedBlock` 更靠后的 Possible Decision（`ValidBlock`），即 $ POLRound > LockedRound $，并收到了证明，即 $ Prevote.votes(POLRound).HasTwoThirdsMajority() $ 。这意味着 `LockedBlock` 已经过时，`LockedBlock` 会被当前的 `Proposal.Block`（即 `ValidBlock`）替换掉，因此不会有冲突。
2. 当前提案的 `Block` 就是 Validator 的 `LockedBlock`。对于 Validator 来说「正合我意」，因此直接通过验证。

这就是为什么 `Proposal` 中如果要打包之前的 `ValidBlock` 就要携带 `ValidRound` 的原因，它可以证明这一轮的提案比 Validator 的 `LockedBlock` 更加靠后，从而通过 Validator 验证。

## 为什么需要 ValidBlock 和 ValidRound
HotStuff 中的 Locked 概念可以理解为 Tendermint 中 `ValidBlock` 和 `LockedBlock` 概念的合并，这是因为 Tendermint 是一个 Round-Based BFT 共识。当一个提案成为 `ValidBlock` 后，只要是其 `ValidRound` 是最新的，这个 `ValidBlock` 就会称为新的 `LockedBlock`。但问题是，在异步环境下我们无法保证 `ValidBlock` 是最新的。

假设有一个 Validator 处于 `Round100`，此时突然收到了一个 `Round1` 的 `Proposal`，且之前已经有了 +2/3 的投票，那么也要更换 `LockedBlock` 通过验证吗？显然不能，这就说明在同一个 `Height` 中，多个 `Round` 的提案是有顺序的，我们直接收新的提案，这需要 `ValidRound` 来保证。再设想一下，如果把 `ValidBlock` 和 `ValidRound` 概念去掉，每个 `Round` 的提案一旦收到 +2/3 的 `Prevote` 投票就 Lock，那么由于网络延迟，有可能不同 Validator 的 `LockedBlock` 都不一致。**为了保证各个 Validator 的 `LockedBlock` 的一致性，我们引入了 `ValidBlock` 和 `ValidRound`。**

举个例子，如果我们把一个 Height 当中多个 Round 的共识过程展示出来，应该是下图这个样子。

![image.png](/assets/images/tendermint2-img/image8.png)

图中红色箭头表示 Proposer 打包了箭头指向的 `ValidBlock` 作为新提案。上图中，r0 因为没有收集到 +2/3 的 `Prevote for Block`投票，没有称为 `ValidBlock`，共识失败。r1 打包了一个全新的提案，并获得了 POL，但没有成功 Commit。r2 因此打包了 r1 的提案作为本轮提案，并且设置 `POLRound = r2`（红色箭头），并成功获得 POL，但没有成功 Commit。同理，r3 打包了 r2 作为提案，获得 POL 并在本轮成功完成提交。Height N 共识完成。

这是一个大致过程，表示不同 Round 的提案依赖关系，但其实 Validator 还会判断新提案是否与本地的 Locked 提案冲突。

再看下图的两个例子，均是针对 Height N 进行共识，其中 r0 已经获得 POL 但提交失败，r1 获取 POL 失败，于是 r2 基于 r0 继续共识且获取了 POL 并被 Locked，但依然没有 Commit，此时进入 r3。图中讨论了两个情况，上面例子中，r3 看到了 r2，并基于 r2（`POLRound = r2`）继续共识，此时由于 $ POLRound\geq LockedRound $，因此 Validator 认为 r3 合法。而下面的例子中，r3 无视 r2 做过的努力，基于 r0 （`POLRound = r0`）继续共识，此时由于 $ POLRound < LockedRound $，因此 Validator 认为 r3 非法，会投 `Prevote for Nil` 票。

![image.png](/assets/images/tendermint2-img/image9.png)

总的来说，`ValidBlock` 和 `ValidRound` 是使用 Round-Based BFT 共识的代价。**在同一个 `Height` 中，由于多个 `Round` 的提案都有可能收获足够的投票，如果贸然设置成 `LockedBlock`，我们无法保证不同 Validator 之间 `LockedBlock` 的一致性。而通过设计 `ValidBlock` 和 `ValidRound` 的包含机制可以保证 `LockedBlock` 永远是最新的 `ValidBlock`。**


# 代码中的一些细节
---

## 额外的同步机制
在部分同步网络中，消息可能无法及时收到，导致 Validator Set 中的节点状态出现短暂的不一致，这就会造成频繁触发视图切换和 `NewRound`，使得共识吞吐量的下降。

Tendermint 在上述共识流程的执行过程中，执行了额外的同步操作，并且使用事件的方式进行解耦。具体来说，代码中在共识模块初始化之时注册了三个事件，当事件触发后会广播对应的消息给其他节点：

+ `EventNewRoundStep`：每次改变共识阶段时，即 `RoundState` 发生改变时，会发布 `EventNewRoundStep` 事件，广播 `RoundState` 给其他节点。
+ `EventValidBlock`: 当发现自己缺少 `Proposal` 时，发布 `EventValidBlock` 事件，广播 `RoundState` 给其他节点。
+ `EventVote`：每次 Validator 收到一个 `Vote` 时，会发布 `EventVote` 事件，广播 `Vote` 给其他节点。

以 `EventVote` 事件为例，其他节点收到源节点发送的 `Vote` 消息后，会将其保存在本地内存的一个 Bitmap 中，因此每个 Validator 都记录了其他 Validator 在某个 `<Height, Round, Step>` 中对提案的投票情况。

同理对于 `EventValidBlock` 和 `EventNewRoundStep` 来说，Validator 也保存了其他节点最新的 `RoundState` 状态。当发现某个节点的状态滞后时（比如缺少 `Proposal`），就可以主动同步自己的状态给它，协助其快速「追上进度」，保证所有 Validator 当下状态的一致性。比如一个新节点刚刚加入，由于历史状态滞后，就需要这个同步逻辑协助它「追上进度」。

举例，以下代码中，当一个节点发现其他节点的 `Height` 滞后时会同步自己的 `Precommit` 投票给该节点，帮助其快速进入下一个 `Height`：

```go
func (conR *Reactor) gossipVotesRoutine(peer p2p.Peer, ps *PeerState) {
	// ...
	for {
		// ...
		// 如果节点落后当前节点 1 个 Height，则随机发送本地保存的其他节点的一份 Precommit 投票
		// 帮助其尽快 Commit。该投票是 EventVote 同步而来的。
		if prs.Height != 0 && rs.Height == prs.Height+1 {
			ps.PickSendVote(rs.LastCommit)
		}
	}
}
```

## WAL 的使用
状态机流转过程中会不断将一些中间信息写入 WAL，包括：

+ `TimeoutInfo`：`Timeout` 如 `PrevoteTimeout` 到期后触发的消息。
+ `MsgInfo`：通用消息，包括 Vote、Proposal Block 和 Proposal Block Parts。
+ `EndHeightMessage`：一个 `Height` 结束时（即 Commit 后）的相关信息。
+ `RoundState`：每次阶段变更时节点当前的状态信息。

如下面代码，一个 `Round` 中每个阶段变更时会调用 `newStep`函数：

```go
// 变更 RoundState
func (cs *State) newStep() {
	rs := cs.RoundStateEvent()
	if err := cs.wal.Write(rs); err != nil {
		cs.Logger.Error("failed writing to WAL", "err", err)
	}
	// ...
}
```

这些信息可在 Validator 发生 Crash 并重启后快速恢复节点 Crash 前的状态，使其可以继续参与后续共识。在之前的 Chained HotStuff 设计中我们是使用邻居节点同步的方式恢复 Crash 节点状态的，而 Tendermint 由于每个 Validator 收集的投票可能是不同的，因此从 WAL 中恢复更加可靠方便。当然上文中也讲到了 Tendermint 的同步机制，邻居节点提供 Proof 证明当前的状态也可以进行恢复。这里仅作为 HotStuff 后续设计的一个额外参考。

## PartSet 和 Merkle Proof
Tendermint 考虑了 `Proposal` 内容过大的情况，提供了将 `Block` 分成多个 Part，并分次发送给其他 Validator 的能力。这是一个基础能力，不仅仅用于共识 `Proposal`。下面为基本数据结构：

```go
type PartSet struct {
	total uint32
	hash  []byte

	mtx           tmsync.Mutex
	parts         []*Part
	partsBitArray *bits.BitArray
	count         uint32
	// 总大小，用于保证不超过上限
	byteSize int64
}

type Part struct {
	Index uint32           `json:"index"`
	Bytes tmbytes.HexBytes `json:"bytes"`
	Proof merkle.Proof     `json:"proof"`
}

type Proof struct {
	Total    int64    `json:"total"`     // item 的总数
	Index    int64    `json:"index"`     // item 的 index
	LeafHash []byte   `json:"leaf_hash"` // item 的 Hash 值
	Aunts    [][]byte `json:"aunts"`     // 计算 root hash 需要的链路节点哈希值
}
```

Proposer 会将一个 `Block` 内容拆分成多个 `Part`，并发送 `Proposal`（包含 `BlockHash`）以及所有的 `Part`，`Part` 中提供 Merkle Proof 用以证明是 `Proposal` 的一部分。生成并发送提案的代码如下：

```go
// 生成 Block，并拆分成多个 Block Part
func (state State) MakeBlock(...) (*types.Block, *types.PartSet) {
	block := types.MakeBlock(height, txs, commit, evidence)
	return block, block.MakePartSet(types.BlockPartSizeBytes)
}
```

```go
// 发起一个 Proposal，广播给所有的 Validator
func (cs *State) defaultDecideProposal(height int64, round int32) {
	// ...
	// 新建 Proposal
	proposal := types.NewProposal(height, round, cs.ValidRound, propBlockID)

	// ...
	// 广播 Proposal 和 Block Parts
	cs.sendInternalMessage(msgInfo{&ProposalMessage{proposal}, ""})

	for i := 0; i < int(blockParts.Total()); i++ {
		part := blockParts.GetPart(i)
		cs.sendInternalMessage(msgInfo{&BlockPartMessage{cs.Height, cs.Round, part}, ""})
	}
}
```

Validator 在收到 `Proposal` 和 Block Parts 时会分别进行处理，针对 Block Part 会使用其携带的 Merkle Proof 逐一验证，保证其真实属于该 `Proposal`。简化后的代码如下：

```go
// 接收消息
func (cs *State) handleMsg(mi msgInfo) {
    // ...
    switch msg := msg.(type) {
	case *ProposalMessage: // Proposal 类型消息
		// 记录下 Proposal，主要是其中的 Block Hash
		err = cs.setProposal(msg.Proposal) 
	case *BlockPartMessage: // Block Part 消息
		// 使用 Merkle Proof 验证后加入 PartSet
		added, err = cs.AddPart(msg.Part)
		// 当 Proposal Block Parts 完整后，再处理 Proposal
		if added && cs.ProposalBlockParts.IsComplete() {
			cs.handleCompleteProposal(msg.Height)
    	}
    }
    // ...
}

func (ps *PartSet) AddPart(part *Part) (bool, error) {
	// 使用 Merkle Proof 验证 Part
	if part.Proof.Verify(ps.Hash(), part.Bytes) != nil {
		return false, ErrPartSetInvalidProof
	}

	// 存入 PartSet
	ps.parts[part.Index] = part
	ps.partsBitArray.SetIndex(int(part.Index), true)
	ps.count++
	ps.byteSize += int64(len(part.Bytes))
	return true, nil
}
```

# 推荐阅读
---

+ [Tendermint 共识协议部分代码]( https://github.com/tendermint/tendermint/blob/main/consensus/state.go)
+ [Tendermint Essay: The latest gossip on BFT consensus ](https://arxiv.org/abs/1807.04938)
+ [Tendermint: Consensus without Mining](https://tendermint.com/static/docs/tendermint.pdf)
  	 

