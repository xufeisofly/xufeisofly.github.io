---
title: Tendermint 实现逻辑和细节
layout: post-with-content
post-image: "/assets/images/tendermint2-img/head.jpg"
tags:
- tendermint
- consensus
- blockchain
---

本篇文章是阅读了 Tendermint 相关资料，如《区块链架构与实现——Cosmos 详解》和[Tendermint Core 源代码](https://github.com/tendermint/tendermint/blob/main/consensus/state.go)之后，对 Tendermint 理解更新。包括部分协议设计的底层逻辑，并从工程实现的角度（而不仅仅是原理上）对具体模块进行阐述，比如 Proposer 轮换以及 Slashing 策略。

此外，建议读者阅读[这篇论文](https://tendermint.com/static/docs/tendermint.pdf)，它涉及了 Tendermint Core 共识部分的具体实现，而原始的 [Tendermint 论文](https://arxiv.org/abs/1807.04938)更多是理论上的。读者还可以阅读我[上一篇](https://xufeisofly.github.io/blog/tendermint)关于 Tendermint 的文章，两篇文章无论先后，希望能有所帮助。

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

# Lock 和 Unlock 机制的底层逻辑
---

Tendermint 共识协议当中使用了 `LockedValue` 和 `LockedRound` 对提案进行锁定，其实不仅仅是 Tendermint，其他的 BFT 共识都需要 Lock 机制保证共识协议的安全性，比如 HotStuff 中的 `LockedQC`。

我们不放尝试一下没有 Lock 机制会发生什么。下图为 N1, N2, N3, N4 四个节点组成的共识网络，在某一个高度的 Round1 中，N1 作为 Proposer 打包发布提案 B1，四个节点成功接收并返回 Prevote 投票，如 `PV(N1, B1)` 表示 N1 节点针对提案 B1 的 Prevote 投票。然而在 Prevote 阶段收集 +2/3 的 Prevote 投票时，N2 节点发生了网络故障，导致没有收到任何投票，其他节点则成功收集 +2/3 的 Prevote 投票并广播了自己的 Precommit 投票，如 `PC(N1, B1)` 表示 N1 节点针对提案 B1 的 Precommit 投票。不幸的是，进入 Precommit 阶段后，N3, N4 节点也出现了网络问题，因此只有 N1 节点收集到了 3 份 Precommit 投票，从而成功完成了 B1 提案的提交，而 N2, N3, N4 节点由于超时，开启了本高度的 Round2 进行第二次共识尝试。

Round2 中，N2 被选举为 Proposer，由于没有 Lock 机制，N2 打包了一个全新的提案 B2 并广播，此时对于 N1 来说由于已经完成了本高度的共识，不会参与 Round2 的提案共识，最终 N2, N3, N4 成功提交了提案 B2，而由于 B2 与 B1 提案是冲突的，这就导致 N1 的状态与 N2, N3, N4 的状态发生了不一致。

![image.png](/assets/images/tendermint2-img/image_p1.png)

上面例子中仅仅因为网络问题（甚至不涉及拜占庭节点）就使得一个没有 Lock 机制的 Tendermint 丧失了安全性。因此，BFT 共识需要一个 Lock 机制，当一个提案收到 +2/3 的认可后即被 Lock，而新的提案需要保证不与 Locked 提案发生冲突。也就是说 Locked 提案对于节点来说就像一个已经过审的执行方案，是所有节点公认的前进方向，如果在最终执行前有人提出了一个新的提案，必须与 Locked 提案进行比对判断是否冲突。

当然这并不意味着一个提案被 Lock 后就一定被执行，这也是当时我学习 HotStuff 时网上很多资料对 LockedQC 的误解。Locked 提案可以被 Unlock，只要满足一定的条件，具体内容请参考前文对于 LockedValue 和 ValidValue 的解释。之所以需要解锁机制，是因为没有的话会发生死锁导致协议丧失 Liveness。

下图中，N4 是拜占庭节点，Round1 中对于 B1 提案在收集 Prevote 投票时 N2, N3 网络异常没有产生 Precommit 投票，而 N1 和 N4 则由于收到了足够的 Prevote 投票将 B1 设置为 Locked 状态，由于没有节点收集到足够的 Precommit 投票，四个节点超时进入 Round2。

Round2 中，N2 被选为 Proposer，由于 N2 没有 Lock 提案 B1，因此 N2 打包了一个全新的提案 B2 进行广播，此时由于 N1 已经 Lock 了 B1，在没有 Unlock 机制的情况下，N1 直接拒绝 B2 提案，而 N2, N3 由于没有 Lock 提案 B1，通过了 B2 的验证并生成了 Prevote for B2，而 N4 作为拜占庭节点，故意无视自己 Lock 状态，一同对 B2 进行 Prevote 投票，因此每个节点都能收到 +2/3 的 Prevote 投票（来自 N2, N3, N4）。此时 N1 仍然坚持自己已经 Lock 的 B1 提案所以不会生成 Precommit for B2，N2, N3 则会产生 Precommit for B2，而拜占庭节点 N4 呢，此时又选择不再为 B2 投票。最终的结果是，N1 仍然 Lock on B1，而 N2, N3 Lock on B2，且由于没有节点收集到足够的 Precommit 投票，因此没有任何提案被提交，共识协议卡死。

![image.png](/assets/images/tendermint2-img/image_p2.png)

为此 Tendermint 设计了 Unlock on Polka 的策略，所谓 Polka（或称为 POL，Proof of Lock）即在收集到 +2/3 的 Prevote for Block 同意票时生成，类似于 HotStuff 的 LockedQC。当 Validator 发现了一个比 Locked 提案更新的 ValidBlock（Valid 指已经收到 +2/3 Prevote 投票的提案），即 $ POLRound > LockedRound $ 时，就会 Unlock 之前的 LockedBlock 并 Lock 新的 ValidBlock（见上文）。

# Proposer 轮换算法
---

## 算法原理
Tendermint 的 Proposer 轮换算法的目的是：根据权重决定被选为 Proposer 的概率，权重越大概率越大。因此 Proposer 的选择需要满足一些特性：

+ 确定性。相同状态下的不同节点选出来的 Proposer 必须相同。
+ 公平性。一个节点被选为 Proposer 的概率应当与权重成正比。

Tendermint 使用了一个基于加权的轮训算法（[源码地址](https://github.com/tendermint/tendermint/blob/main/types/validator_set.go)），我们可以想象一个队列存放了所有的 Validator，每一个 Round 开始时 Validator 往队列头部移动，权重越大移动的越多，而 Proposer 就是队列头部的 Validator。代码如下：

```go
func (vals *ValidatorSet) incrementProposerPriority() *Validator {
	for _, val := range vals.Validators {
		// 增加 validator 的 ProposerPriority 权重
		newPrio := safeAddClip(val.ProposerPriority, val.VotingPower)
		val.ProposerPriority = newPrio
	}
	// 选出最大权重的 validator 并返回，作为 Proposer
	mostest := vals.getValWithMostPriority()
	// 同时将其权重 - Total 权重，移至队列尾部
	mostest.ProposerPriority = safeSubClip(mostest.ProposerPriority, vals.TotalVotingPower())

	return mostest
}

// 获取权重最大的 validator
func (vals *ValidatorSet) getValWithMostPriority() *Validator {
	var res *Validator
	for _, val := range vals.Validators {
		res = res.CompareProposerPriority(val)
	}
	return res
}

// 比较 validator 的 ProposerPriority，相同的话就根据 address 比较
func (v *Validator) CompareProposerPriority(other *Validator) *Validator {
	if v == nil {
		return other
	}
	switch {
	case v.ProposerPriority > other.ProposerPriority:
		return v
	case v.ProposerPriority < other.ProposerPriority:
		return other
	default:
		result := bytes.Compare(v.Address, other.Address)
		switch {
		case result < 0:
			return v
		case result > 0:
			return other
		default:
			panic("Cannot compare identical validators")
		}
	}
}
```

而调用函数 `incrementProposerPriority` 的时机为每一个新轮次开启时，即 `NewRound`，这时候需要更换 Proposer 重新打包提案。

```go
func (cs *State) enterNewRound(height int64, round int32) {
	...

	validators := cs.Validators
	if cs.Round < round {
		validators = validators.Copy()
        // 可能根据落后程度执行多次 incrementProposerPriority
		validators.IncrementProposerPriority(tmmath.SafeSubInt32(round, cs.Round))
	}
    
	...
}

func (vals *ValidatorSet) IncrementProposerPriority(times int32) {
	...
        
	var proposer *Validator
	for i := int32(0); i < times; i++ {
		proposer = vals.incrementProposerPriority()
	}

	vals.Proposer = proposer
}
        
```

举个例子，如下图。优先级 P1 和 P2 对应的权重（VP, Voting Power）分别为 1 和 3，初始状态下优先级（P, Priority）均为 0，以下是 4 个轮次中优先级的变化和 Proposer 的选择。四个轮次依次选出的 Proposer 为 P2、P1、P2、P2，可见 P1 和 P2 被选择的比例确为 1 : 3。

![image.png](/assets/images/tendermint2-img/image_p3.png)

从上图中我们可以看到一个特点，每轮结束时，系统中的总优先级为 0，这个特点可以保证所有节点的权重永远在 0 值附近，不会有大幅度偏离。但是当验证者集合会动态变化时就有可能破坏这个特性。

## 动态验证者集合
这种基于权重的轮换选择算法在验证者集合为静态时没有问题，但实际项目中，验证者的权重可能会发生改变，或有验证者加入或退出该集合，当涉及到动态验证者集合的 Proposer 选举轮换时，这个算法会产生一些问题，下面我们对每种情况进行逐一分析。

**1.验证者权重增加**

验证者权重突然增加对轮换算法没有任何影响，读者可参考上述静态验证者集合的表格自行分析。

**2.验证者权重减少**

当验证者的权重突然减少时，可能会破坏轮换算法的「公平性」，即验证者被选中的概率与权重不再成正比。比如下面的情况。P1 P2 权重分别为 4 和 3。但在第 1 轮结束时，P1 由于被选中 Proposer 而落到了很后面的地方，此时 P1 的权重变为 1，那么 P1 需要一点一点的追上来，下面的表格中可以看到，P1 与 P2 被选为 Proposer 的概率并非为 1 : 3，因此轮换算法失去了公平性。

![image.png](/assets/images/tendermint2-img/image_p4.png)

**3.验证者退出和加入**

验证者突然的退出和加入会导致系统的总优先级发生偏移，不再等于 0。比如某轮次结束后，三个验证的优先级分别为 1、2、-3，总和为 0，此时 -3 对应的验证者突然退出，那么总优先级就变成了 3。类似的事情重复发生就会导致提案的优先级值发生溢出。

Tendermint 使用 Centering 进行解决，即每一轮开始时对现有的验证者优先级进行「居中对齐」，每一个验证者的优先级值减去所有验证者优先级的平均值，如剩下的两个验证者优先级变为了 

$ A(P1) = 1 - (1+2)/2 = -0.5 $

$ A(P2) = 2 - (1+2)/2 = 0.5 $

还有一个情况，验证者选为 Proposer 后会被移动到队列的最后，一点一点的追上来，但如果此时该验证者退出集合并再次加入集合，就相当于该验证者在队列中向前跳跃了一次。为了解决这个问题，当一个验证者加入时，其优先级的初始值需要设置为 $ A = -1.125P $，其中 $ P $ 为所有验证者的优先级总和。（-1.125 这个系数在此不做解释）。

除此之外，Tendermint Core 的 Proposer 轮换算法还解决了一系列问题，如使用缩放保证提案优先级最大值和最小值之间的差值不会过大等等。此处不过多讨论细节，只需要了解选举轮换算法的整体逻辑即可。

# 举证 Evidence 与惩罚 Slashing
---

## Evidence Pool
只通过 BFT 共识算法本身无法完全避免拜占庭节点的恶意行为，Slashing 惩罚策略是保证共识安全性的重要部分，比如 Nothing At Stake 攻击等。为此，Tendermint Core 实现了 Evidence Pool 用于管理恶意行为的证明 Evidence，用于对接 Cosmos SDK 的 Slashing 模块。

下图展示了一个 Evidence 的生命周期。Evidence 会通过 Block 结构体进行 Propose，用于存放恶意行为的举证，目前有 `DuplicateVoteEvidence` 和 `LightClientAttackEvidence` 实现了这个接口，前者是对双签行为（Double Voting）进行举证，由 Consensus Engine 提供，后者是对轻客户端的一些攻击行为，包括 Lunatic，Equivocation，Amnesia 攻击，由轻客户端提供。

![image.png](/assets/images/tendermint2-img/image_p5.png)

本文不涉及轻客户端相关内容，感兴趣可参阅[这篇文章](https://medium.com/tendermint/different-types-of-evidence-in-tendermint-5de4440fdd54)，重点讨论 Double Voting。

## 双签举证
在讨论以太坊 PoS 的系列文章中我们讨论过 [Nothing At Stake](https://xufeisofly.github.io/blog/ethereum-pos-p1) 攻击，其实就是 Double Voting，指的是验证者为同一个高度的两个不同提案进行签名投票，因为投票并没有成本，这样无论哪个分支最终成为 Canonical 的，验证者都不受影响。

在 Tendermint Core 中，这种恶意行为由验证者在收集投票时发现，并且生成一个 `DuplicateVoteEvidence` 存储在 Evidence Pool 中，用于证明某验证者曾脚踏了两只船。下面代码为验证者接收到投票消息后，检测到投票与之前某个投票冲突而进行上报的过程。

```go
// 接受投票
func (cs *State) tryAddVote(vote *types.Vote, peerID p2p.ID) (bool, error) {
	added, err := cs.addVote(vote, peerID)
	if err != nil {
		// vote 冲突了
		if voteErr, ok := err.(*types.ErrVoteConflictingVotes); ok {
            ...
            // 上报从图投票到 evidence pool
			cs.evpool.ReportConflictingVotes(voteErr.VoteA, voteErr.VoteB)
		}
        ...
	}

	return added, nil
}
```

Tendermint Core 中的`DuplicateVoteEvidence` 结构如下：

```go
type DuplicateVoteEvidence struct {
    VoteA  *Vote         // 第一个投票
    VoteB  *Vote         // 第二个投票
    ...
}
```

我们可以使用 `VoteA` 和 `VoteB` 中获取签名者的公钥，就可以证明该验证者确实签署了两个冲突的区块。由于 Evidence 是 Proposal 的一部分，由 Proposer 进行打包发布，所有的 Validator 会从提案消息中提取 Evidence 进行验证，即进行 Prevote 投票之前，代码如下。

```go
// 接收 Proposal 消息
func (cs *State) defaultDoPrevote(height int64, round int32) {
    ...

	// 验证 proposal
	err := cs.blockExec.ValidateBlock(cs.state, cs.ProposalBlock)
	if err != nil {
		cs.signAddVote(tmproto.PrevoteType, nil, types.PartSetHeader{})
		return
	}

	cs.signAddVote(tmproto.PrevoteType, cs.ProposalBlock.Hash(), cs.ProposalBlockParts.Header())
}

// 验证 Block
func (blockExec *BlockExecutor) ValidateBlock(state State, block *types.Block) error {
	err := validateBlock(state, block)
	if err != nil {
		return err
	}
	return blockExec.evpool.CheckEvidence(block.Evidence.Evidence)
}
```

如果验证 Evidence 失败，说明该举报材料有问题，会影响整个提案的合法性。但如果验证 Evidence 成功，说明确实存在 Double Voting 现象，会将 Evidence 的状态由 `Pending` 改为 `Committed`，供 Cosmos SDK 的 Slashing 模块调用。

在验证双签举证时会重点验证如下内容：

+ 签名者确实是本次 ValidatorSet 的成员。
+ 两个 Vote 对应的高度，轮次，验证者等信息一致，但对应的提案不同。
+ 两个签名都能通过验证。

这里代码逻辑十分清晰。

```go
func VerifyDuplicateVote(e *types.DuplicateVoteEvidence, chainID string, valSet *types.ValidatorSet) error {
    // validator 合法
    _, val := valSet.GetByAddress(e.VoteA.ValidatorAddress)
	if val == nil {
		return fmt.Errorf("address %X was not a validator at height %d", e.VoteA.ValidatorAddress, e.Height())
	}
	pubKey := val.PubKey
	// 两个投票信息一致
	if e.VoteA.Height != e.VoteB.Height ||
		e.VoteA.Round != e.VoteB.Round ||
		e.VoteA.Type != e.VoteB.Type {
		return fmt.Errorf("h/r/s does not match: %d/%d/%v vs %d/%d/%v",
			e.VoteA.Height, e.VoteA.Round, e.VoteA.Type,
			e.VoteB.Height, e.VoteB.Round, e.VoteB.Type)
	}

	// 同一个验证者
	if !bytes.Equal(e.VoteA.ValidatorAddress, e.VoteB.ValidatorAddress) {
		return fmt.Errorf("validator addresses do not match: %X vs %X",
			e.VoteA.ValidatorAddress,
			e.VoteB.ValidatorAddress,
		)
	}

	// 投票对应不同的两个提案
	if e.VoteA.BlockID.Equals(e.VoteB.BlockID) {
		return fmt.Errorf(
			"block IDs are the same (%v) - not a real duplicate vote",
			e.VoteA.BlockID,
		)
	}

	// 公钥和地址对应
	addr := e.VoteA.ValidatorAddress
	if !bytes.Equal(pubKey.Address(), addr) {
		return fmt.Errorf("address (%X) doesn't match pubkey (%v - %X)",
			addr, pubKey, pubKey.Address())
	}

	// 验证者的投票权重和总权重合法
	if val.VotingPower != e.ValidatorPower {
		return fmt.Errorf("validator power from evidence and our validator set does not match (%d != %d)",
			e.ValidatorPower, val.VotingPower)
	}
	if valSet.TotalVotingPower() != e.TotalVotingPower {
		return fmt.Errorf("total voting power from the evidence and our validator set does not match (%d != %d)",
			e.TotalVotingPower, valSet.TotalVotingPower())
	}

	va := e.VoteA.ToProto()
	vb := e.VoteB.ToProto()
	// 两个签名都验证通过
	if !pubKey.VerifySignature(types.VoteSignBytes(chainID, va), e.VoteA.Signature) {
		return fmt.Errorf("verifying VoteA: %w", types.ErrVoteInvalidSignature)
	}
	if !pubKey.VerifySignature(types.VoteSignBytes(chainID, vb), e.VoteB.Signature) {
		return fmt.Errorf("verifying VoteB: %w", types.ErrVoteInvalidSignature)
	}

	return nil
}
```

## Slashing 惩罚策略
Slashing 策略在实现上并不属于 Tendermint Core，而是在 Cosmos SDK，Tendermint Core 只是提供了作恶的证据，Slashing 相关策略推荐参考[这篇资料](https://docs.cosmos.network/main/build/modules/slashing)。Cosmos SKD 涉及的 Slashing 主要场景有两个。

+ Double Signing Slashing（或 Double Voting）。使用 Tendermint Core 层提供的 evidence，一旦发现，Validator 会被永久紧闭，运营方只能重新创建账户参与共识，此时的状态称为 Tombstoned。
+ Liveness Slashing，直接表现为 Validator 长时间不投票，由 Cosmos SDK 进行检测。每一个 Block 都会保留上一个 Committed 区块的 Precommit 投票，Cosmos SDK 使用了一个 bitmap 的结构实现了一个滑动窗口来记录 Blocks 中是否包含了某 Validator 的投票，以此判断 Validator 是否丧失了 Liveness。被发现后 Validator 会被置为 Jailed 状态。一个 Jailed 状态的 Validator 随时可以通过广播 `UnjailMsg` 消息来重新加入 ValidatorSet 参与共识。

以下结构体保存了 Validator 的 Slashing 信息。

```protobuf
message ValidatorSigningInfo {
  ...
  // Validator 处于 Unjailed 状态的初始高度
  int64 start_height = 2;
  // 需要 jail 的时间
  google.protobuf.Timestamp jailed_until = 4
  // 是否被 tombstoned（即逐出 ValidatorSet）
  bool tombstoned = 5;
  // 没有签名的 Block 个数，是个冗余字段，避免从 bitmap 中读取
  int64 missed_blocks_counter = 6;
}
```

需要注意的是，从 Validator 做出了一个 Infraction 行为到这个行为被发现从而被 Slashed，是存在延迟的。对于 Liveness Slashing 来说没有，但对于 Double Signing Slashing 来说，该节点在被发现之前往往还可以进行多次 Double Signing 行为，而 Cosmos 只会惩罚一次，这个代价是巨大的，不仅会没收质押金还会被永远逐出 ValidatorSet。

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
+ [Cosmos 的 Slashing 机制](https://docs.cosmos.network/main/build/modules/slashing)
+ 《区块链架构与实现——Comos 详解》
  	 

