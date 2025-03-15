---
title: 从 HotStuff 回看 Tendermint (三)
layout: post-with-content
post-image: "https://xufeisofly.github.io/picx-images-hosting/tendermint3/head.ic9yrrt6r.jpg"
tags:
- tendermint
- consensus
- blockchain
---

本篇文章是阅读了《区块链架构与实现——Cosmos 详解》之后，对 Tendermint 理解更新。包括部分协议设计的底层逻辑，从工程实现的角度（而不仅仅是原理上）对具体模块进行阐述，比如 Proposer 轮换以及 Slashing 策略。

本系列共有三篇文章，此为第三篇。

+ [从 HotStuff 回看 Tendermint (一)](https://xufeisofly.github.io/blog/tendermint)
+ [从 HotStuff 回看 Tendermint (二)](https://xufeisofly.github.io/blog/tendermint2)

# Lock 和 Unlock 机制的底层逻辑
---

Tendermint 共识协议当中使用了 `LockedValue` 和 `LockedRound` 对提案进行锁定，其实不仅仅是 Tendermint，其他的 BFT 共识都需要 Lock 机制保证共识协议的安全性，比如 HotStuff 中的 `LockedQC`。

我们不放尝试一下没有 Lock 机制会发生什么。下图为 N1, N2, N3, N4 四个节点组成的共识网络，在某一个高度的 Round1 中，N1 作为 Proposer 打包发布提案 B1，四个节点成功接收并返回 Prevote 投票，如 `PV(N1, B1)` 表示 N1 节点针对提案 B1 的 Prevote 投票。然而在 Prevote 阶段收集 +2/3 的 Prevote 投票时，N2 节点发生了网络故障，导致没有收到任何投票，其他节点则成功收集 +2/3 的 Prevote 投票并广播了自己的 Precommit 投票，如 `PC(N1, B1)` 表示 N1 节点针对提案 B1 的 Precommit 投票。不幸的是，进入 Precommit 阶段后，N3, N4 节点也出现了网络问题，因此只有 N1 节点收集到了 3 份 Precommit 投票，从而成功完成了 B1 提案的提交，而 N2, N3, N4 节点由于超时，开启了本高度的 Round2 进行第二次共识尝试。

Round2 中，N2 被选举为 Proposer，由于没有 Lock 机制，N2 打包了一个全新的提案 B2 并广播，此时对于 N1 来说由于已经完成了本高度的共识，不会参与 Round2 的提案共识，最终 N2, N3, N4 成功提交了提案 B2，而由于 B2 与 B1 提案是冲突的，这就导致 N1 的状态与 N2, N3, N4 的状态发生了不一致。

![image_p1](https://xufeisofly.github.io/picx-images-hosting/tendermint3/image_p1.7snd9tje6c.png)

上面例子中仅仅因为网络问题（甚至不涉及拜占庭节点）就使得一个没有 Lock 机制的 Tendermint 丧失了安全性。因此，BFT 共识需要一个 Lock 机制，当一个提案收到 +2/3 的认可后即被 Lock，而新的提案需要保证不与 Locked 提案发生冲突。也就是说 Locked 提案对于节点来说就像一个已经过审的执行方案，是所有节点公认的前进方向，如果在最终执行前有人提出了一个新的提案，必须与 Locked 提案进行比对判断是否冲突。

当然这并不意味着一个提案被 Lock 后就一定被执行，这也是当时我学习 HotStuff 时网上很多资料对 LockedQC 的误解。Locked 提案可以被 Unlock，只要满足一定的条件，具体内容请参考前文对于 LockedValue 和 ValidValue 的解释。之所以需要解锁机制，是因为没有的话会发生死锁导致协议丧失 Liveness。

下图中，N4 是拜占庭节点，Round1 中对于 B1 提案在收集 Prevote 投票时 N2, N3 网络异常没有产生 Precommit 投票，而 N1 和 N4 则由于收到了足够的 Prevote 投票将 B1 设置为 Locked 状态，由于没有节点收集到足够的 Precommit 投票，四个节点超时进入 Round2。

Round2 中，N2 被选为 Proposer，由于 N2 没有 Lock 提案 B1，因此 N2 打包了一个全新的提案 B2 进行广播，此时由于 N1 已经 Lock 了 B1，在没有 Unlock 机制的情况下，N1 直接拒绝 B2 提案，而 N2, N3 由于没有 Lock 提案 B1，通过了 B2 的验证并生成了 Prevote for B2，而 N4 作为拜占庭节点，故意无视自己 Lock 状态，一同对 B2 进行 Prevote 投票，因此每个节点都能收到 +2/3 的 Prevote 投票（来自 N2, N3, N4）。此时 N1 仍然坚持自己已经 Lock 的 B1 提案所以不会生成 Precommit for B2，N2, N3 则会产生 Precommit for B2，而拜占庭节点 N4 呢，此时又选择不再为 B2 投票。最终的结果是，N1 仍然 Lock on B1，而 N2, N3 Lock on B2，且由于没有节点收集到足够的 Precommit 投票，因此没有任何提案被提交，共识协议卡死。

![image_p2](https://xufeisofly.github.io/picx-images-hosting/tendermint3/image_p2.m8a6qflw.png)

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

![image_p3](https://xufeisofly.github.io/picx-images-hosting/tendermint3/image_p3.5mnyo1rqf9.png)

从上图中我们可以看到一个特点，每轮结束时，系统中的总优先级为 0，这个特点可以保证所有节点的权重永远在 0 值附近，不会有大幅度偏离。但是当验证者集合会动态变化时就有可能破坏这个特性。

## 动态验证者集合
这种基于权重的轮换选择算法在验证者集合为静态时没有问题，但实际项目中，验证者的权重可能会发生改变，或有验证者加入或退出该集合，当涉及到动态验证者集合的 Proposer 选举轮换时，这个算法会产生一些问题，下面我们对每种情况进行逐一分析。

**1.验证者权重增加**

验证者权重突然增加对轮换算法没有任何影响，读者可参考上述静态验证者集合的表格自行分析。

**2.验证者权重减少**

当验证者的权重突然减少时，可能会破坏轮换算法的「公平性」，即验证者被选中的概率与权重不再成正比。比如下面的情况。P1 P2 权重分别为 4 和 3。但在第 1 轮结束时，P1 由于被选中 Proposer 而落到了很后面的地方，此时 P1 的权重变为 1，那么 P1 需要一点一点的追上来，下面的表格中可以看到，P1 与 P2 被选为 Proposer 的概率并非为 1 : 3，因此轮换算法失去了公平性。

![image_p4](https://xufeisofly.github.io/picx-images-hosting/tendermint3/image_p4.1ap5gi8ex1.png)

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

![image_p5](https://xufeisofly.github.io/picx-images-hosting/tendermint3/image_p5.2h8gp3xbie.png)

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


# 推荐阅读
---

+ [Tendermint 共识协议部分代码]( https://github.com/tendermint/tendermint/blob/main/consensus/state.go)
+ [Cosmos 的 Slashing 机制](https://docs.cosmos.network/main/build/modules/slashing)
+ 《区块链架构与实现——Comos 详解》
  	 

