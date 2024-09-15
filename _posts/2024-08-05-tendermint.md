---
title: 从 HotStuff 回看 Tendermint — 理论
layout: post
post-image: "/assets/images/tendermint-img/head.jpg"
tags:
- tendermint
- hotstuff
- consensus
- blockchain
---

在生产环境中，Tendermint 是一个比 HotStuff 出现更早、更加成熟的方案，最有名的是 Cosmos 中的应用。我们在 Shardora 实现了 HotStuff 作为共识层之后，学习并参考了 Tendermint，特别是参考其对接 PoS 的部分，以进一步完善 Shardora 共识的效率和安全性。此外，Tendermint 中涉及很多 BFT 的基本概念，是 BFT 类共识发展历史的一个重要节点，了解 Tendermint 能让我们更加熟悉 BFT 类共识的底层原理，更能了解 BFT 类共识普遍面对的问题，以及不同共识算法对应的解决方案。

[Tendermint 项目地址](https://github.com/tendermint/tendermint/)

[Shardora HotStuff 代码地址](https://github.com/tenondvpn/shardora/tree/main/src/consensus/hotstuff)

现在的 Tendermint 不仅仅指代共识算法，而是个十分成熟的开源项目，不仅包括共识部分，还包括网络层和 ABCI（Application BlockChain Interface）用于不同的语言快速接入。项目分层如下：

- Application Layer (Cosmos SDK)
- Consensus Layer (Tendermint Core)
- Network Layer (Tendermint Core)

本文主要是对 Tendermint 共识层的学习和思考，不涉及其他部分。而学习 Tendermint 共识的前提，是了解**部分同步模型**这个概念。

# 同步、异步和部分同步模型
---

根据网络中消息从发送到被接受的时间，我们将网络分为**同步、异步和部分同步**三种模型。

- **同步模型**：消息从一个节点发送到另一个节点所需要的时间存在一个**固定的、可知的**上限。
- **异步模型**：不存在那个固定的时间上限，时间可以无限长。异步系统中无法准确的给出一个时间参数，使得节点的消息会在该时间内被接收，消息时延完全依赖于网络情况，一个诚实节点发出的消息可能经过 1 万年才能被收到，虽然很夸张，但异步系统需要处理这些情况。
- **部分同步模型**：假设存在一个全剧稳定时间，称为 GST，在 GST 时刻之后，我们认为整个系统达到如下状态：**所有诚实节点发送的消息会在一个时间 ∆ 以内必定被接受**。在 GST 之前，系统被认为是异步的，共识速度与系统参数无关，只和真实的网络传输速度有关。而在 GST 之后，系统被认为是同步的，有一个固定时间参数保证消息的到达。总的来说，可以认为部分同步系统存在一个总的消息传输时间上限，这个时间上限不是固定的，也无法准确得知，可以很短也可以很长。在设计部分同步系统时要考虑这个不确定的时间上限造成共识失败时如何恢复系统活性的问题。

目前已经证明，在异步模型下，如果存在节点宕机的可能，系统将永远无法达成共识，而同步模型由于需要一个漫长固定时间的等待，吞吐量往往较低。其实相比同步和异步，我们的网络更接近部分同步模型，Tendermint 和 HotStuff 都是基于部分同步模型设计的共识算法，即它们假设网络在某些时间段内可能是异步的，但在某个时刻之后变得同步。

我相信很多人一开始跟我的疑惑是一样的，就是异步模型可以理解，但同步模型和部分同步模型的区别究竟是什么。我们说同步模型的消息通讯存在一个固定可知的上限，又说部分同步模型也存在一个时间上限但不可知，这是什么意思？我愿意从另一个角度进行解释：既然同步模型和部分同步模型都存在一个通讯时间上限，那么如果超过这个上限后还没有收到消息，这两种模型会发生什么？

比特币的 PoW 就是一个典型的同步模型，它认为这个时间上限是 10 分钟，即默认 10 分钟一定可以令所有的节点收到矿工的消息。我们假设 10 分钟的出块时间缩短至 10s，那么系统就会频繁出现分叉，当然比特币存在 Longest Chain Rule 来选择最终的分叉，但如果我们继续缩短出块时间，这个分叉会越来越多，极限状态下，每个节点都可能有自己的分叉，于是共识出现不一致性，共识的 Safety 丧失。如下图：

![Untitled](/assets/images/tendermint-img/image1.png)

而 BFT 类共识基本都是部分同步模型，如 HotStuff 和 Tendermint 也同样有 Timeout 时间，但由于部分同步模型的时间上限是不可知的，我们需要也允许调整 Timeout。当在 Timeout 中没有收集到足够多的投票，BFT 共识会进入视图切换，重新发起一轮共识，然后我们会适当的延长 Timeout 时间，争取下一次完成共识。也就是说在部分同步模型中如果消息的传播超过了 Timeout，并不会造成不一致性，只是短时间的丧失了 Liveness 而已。而同步模型一旦超过了 Timeout，就会丧失 Safety。


# Tendermint 共识算法
---

与 HotStuff 和其他 BFT 类共识一样，Tendermint 的前提是拜占庭假设，即满足恶意节点的数量 f 小于 1/3 总节点数。不过 Tendermint 接入了 PoS，使得每个节点的投票权重依赖于 Stake，而不是每个节点一票，Stake 大小决定了投票权重（Voting Power），所以我们所说的 2/3+1 指代的不再是节点数量，而是 Voting Power 的累积值。这点在使用阈值签名的原生 HotStuff 中无法实现，但在使用了聚合签名的 Fast HotStuff 中就可以做到。

此外，Voting Power 还会对 Leader 即打包提案的 Proposer 的选举产生影响，Tendermint 使用 weighted round-robin 的方式，根据 Voting Power 作为权重选举 Leader，权重越大，被选为 Leader 的概率也就越大。同时通过动态调整 Voting Power 尽量避免连续视图选出重复的 Leader。具体细节本文不再涉及。

在拜占庭假设和部分同步系统假设的前提下，Tendermint 的核心逻辑就展现出来了。它包括三个阶段，两次投票。下面是 Tendermint 广为流传的一张示意图。

![Untitled](/assets/images/tendermint-img/Untitled.png)

图中，

- 系统首先选出一个 Leader，Leader 打包并 Propose 一个提案，Replica 节点收到提案后进行验证。如果验证通过，该 Replica 便广播 Prevote 同意票(prevote for block)，失败则广播反对票(prevote for nil)，投票之后该节点进入 Prevote 阶段。
- 在发送了 Prevote 投票后，Replica 等待并收集其他节点的 Prevote 投票，如果收到 2/3+1 的 prevote for block 投票，则广播 precommit for block 投票，否则广播 precommit for nil 投票，之后进入 Precommit 阶段。
- 发送了 Precommit 投票后，Replica 等待并收集其他节点的 Precommit 投票，如果收到了 2/3+1 的 precommit for block 投票，则触发对该提案的 Commit 操作，并增加 Height，切换视图，发起一个全新提案的共识，如果没有收集到足够的 Precommit 同意票，则会对相同提案重新发起 Propose。

在 Tendermint 中，即使提案或某阶段验证失败，Replicas 也会投反对票（而不是什么都不做），即使 Replica 在特定时间内没有收集到足够的同意票而触发超时，也不会像 HotStuff 那样当场进行视图切换，反对意见最终会累积到 Precommit 阶段后触发视图切换。而在 HotStuff 中，如果新提案验证失败，Replicas 不会投票，而是在 Leader 长时间收不到 2/3+1 的同意票后触发超时，当场进行视图切换并发起重试。能做到这点是因为 HotStuff 拥有更快的快速视图切换作为优势。

Tendermint 分为三个阶段，分别是：

- **Proposal 阶段**: 由 Leader 发起提案，可能是新的，也可能是上一轮未共识成功的提案。
- **Prevote 阶段**: 所有节点对提案进行验证，根据结果广播相应的 Prevote 投票，之后收集 2f+1 其他节点的 Prevote 投票，尝试产生有效性证明（相当于 LockedQC）。此证明将用于系统共识的安全性，避免提案之间相互冲突。
- **Precommit 阶段**: 将 Prevote 阶段产生的证明广播给其他节点（Precommit 投票），并尝试收集 2f+1 的 Precommit 同意票，成功则 Commit 并发起新的 Proposal，失败则重试原提案的共识。

我尝试绘制了下图，从节点消息收发的角度描述了 Tendermint 共识进行的过程。其中红色箭头表示同意票，蓝色箭头表示反对票。为了更好的表现出超时时间的作用，图中消息到达节点的时间是不同的。每个阶段中当收到 2f+1 任意投票后会开始一个 timeout 计时。

![Untitled](/assets/images/tendermint-img/Untitled%201.png)

这张图中有三段 timeout，这也是 tendermint 中体现部分同步模型的地方：

- **proposal timeout**：从进入 Proposal 阶段开始计时，如果在 proposal timeout 时间内节点没有收到新的 Proposal，则广播 prevote for nil 票。
- **prevote timeout**：从收到 2/3+1 的 prevote 投票开始计时，如果在 prevote timeout 时间内没有收到 2/3+1 的 prevote for block 投票，则广播 prevote for nil 投票。
- **precommit timeout**：从收到 2/3+1 的 precommit 投票开始计时，如果在 precommit timeout 时间内没有收到 2/3+1 的 precommit for block 投票，则对该提案重新发起一轮共识尝试。

其实，这些 timeout 的本质就是部分同步模型中能够保证所有诚实节点消息到达的时间上限，保证在 timeout 结束时 Replica 能收到所有诚实节点的消息。此处解释一个我曾产生的疑问，即 timeout 并不是上文模型中的 ∆ ，∆ 是从 GST 时刻开始计算的，但由于我们无法明确 GST，所以在工程中是设计和讨论 ∆ 的长度是没有意义的，只需要关心 ∆ 的结束时刻就可以了。 

我们需要尽量保证 timeout 的结束时刻在 ∆ 结束时刻之后，这便涉及到 timeout 的选取。timeout 是动态选取的，当 timeout 结束时刻 < ∆ 结束时刻时，由于同意投票不足，我们无法完成针对提案的共识，系统便会丧失活性（liveness），但不会影响共识的安全性（safety），此时只需要延长 timeout 就行了。理论上当 timeout 结束时刻 ≥ ∆ 结束时刻，系统可以恢复活性。

接下来对三个阶段进行详细描述。

**Proposal 阶段**

系统选出 Leader（Leader 的共识不在本次讨论范围之内，可以参考 HotStuff，或者使用 Tendermint 论文中的 weighted round robin），由 Leader 打包一个提案，这个提案可能是新的，也可能是一个上一轮没有成功达成共识但已经 Lock 的提案。Leader 将该提案广播给所有的 Replica 节点。Replica 在收到 Proposal 消息之后，对提案进行验证，通过后广播一个 Prevote 同意票（prevote for block），若验证失败或超时则广播一个 Prevote 反对票（prevote for nil），并进入 Prevote 阶段，如下图，为了表达清晰，图中仅表示了节点 2 收到 Proposal 并广播投票的过程，其他节点进行了省略。

![Untitled](/assets/images/tendermint-img/Untitled%202.png)

无论是验证失败，还是在 timeout 之前没有收到 Proposal，都会导致节点 2 广播反对票。

**Prevote 阶段**

Replica 在广播了自己的 Prevote 投票后，开始收集其他 Replicas 的 Prevote 投票，目标是能收集到所有诚实节点的 Prevote 投票，为了做到这一点，Replica 在收集到第 2f+1 个 Prevote 投票（无论是同意还是反对）后开启 prevote timeout 计时，并等待 PrevoteTimeout 时间。在部分同步系统的假设中，所有诚实节点的消息会在一个不确定的 ∆ 时间内到达，因此该 prevote timeout 即可作为 Prevote 阶段的最后期限。

如果在 prevote timeout 结束前收集到了 2f+1 个同意票，此 Replica 就会根据此 2f+1 的同意票生成一个证明，证明该提案已经被大部分节点所认可，并将该提案 Lock。对应到 HotStuff 中，这就是 LockedQC 的生成时机。Replica 生成 LockedQC 后（虽然 Tendermint 中并没有 QC 概念，但本质是一样的，后文使用这个概念方便统一理解）会广播 Precommit 同意票给其他节点，否则在超时结束时广播一个 Precommit 反对票，并进入 Precommit 阶段。如下图。

![Untitled](/assets/images/tendermint-img/Untitled%203.png)

**Precommit 阶段**

进入 Precommit 阶段后，Replica 开始等待并收集其他节点的 Precommit 投票，与 Prevote 相同，当收到 2f+1 的同意票后，即可产生一个证明，这个证明表明了该提案已经可以被 Commit，所以在 BFT 类共识中我们称之为 Proof of Commit，HotStuff 中的 CommitQC 就是一个 Proof of Commit。

谁拥有了 Proof of Commit，就可以当场提交本视图的提案，不会影响共识的安全性。为了收集 2f+1 的 Precommit 同意票，Replica 在收到 2f+1 Precommit 任意投票（同意或反对）后开启 precommit timeout 计时，

期间只要收集到了 2f+1 的同意票，就会触发 Replica 对该块的提交，将提案从 Locked 状态变为 Committed 状态，然后开始视图切换：选出下一任 Leader 并安全地发起新提案的共识。而如果在超时结束之前没有收集到 2f+1 的 Precommit 同意票，应当放弃本次收集，更换视图，重新发起针对本提案的新一轮共识（height 不变，round+1）。如下图。此处说明，height 表示提案区块的高度，height 变化对应共识内容的变化，round 代表对同一个 height 的共识轮次，重试 Proposal 意味着 round 增加一次。

![Untitled](/assets/images/tendermint-img/Untitled%204.png)

仔细思考能发现这里有三种情况：

- 第一，新 Leader 完成提交并发起了新提案的共识，但此时其他 Replicas 还没有完成上一个提案的 Commit。那么新 Proposal 中可以携带上一视图的 Proof of Commit，促使其他节点在 Proposal 阶段完成上一个提案的提交。
- 第二，新 Leader 已经 Lock 了该提案，但 precommit timeout 超时时还没有完成提交，然而其他 Replicas 有的已经完成了提交。那么新 Leader 会对提案（已经 Locked）进行「重试」（round+1），其他节点会重新投票，再走一遍 Prevote 和 Precommit 流程，直到下一任 Leader 能够成功 Commit。
- 第三，在 precommit timeout 结束时，新 Leader 没有完成提交，并且上一阶段甚至没有 Lock 该提案，此时如果其他 Replicas 有的已经完成提交，那么新 Leader 发起的新提案可能与其他节点冲突，导致系统丧失活性。
 
前两种情况都是是 ok 的，而第三种情况是由于 timeout 时间太短造成系统活性丧失。因此我们可以再次发现，就像比特币的 10min 出块时间一样，timeout 值的选取十分重要，如果太短，会导致节点永远无法收集到 2f+1 的同意票，从而系统不断地对同一个提案发起共识「重试」操作，使得系统丧失活性，但并不会影响共识的安全性。这也是部分同步系统中「同步」部分带来的问题。HotStuff 也有这个问题，如果我们将 HotStuff 中每个 View 的 timeout 设置的很小，那么 HotStuff 将会无法产生 QC，不断进行视图切换并重试投票，导致共识无法进行。

下面是 Tendermint 论文中的协议伪代码，为了便于理解，我去掉了其中的 `validValue` 和 `validRound` 变量，因为我认为 `lockedValue` 和 `lockedRound` 可以表达新视图中需要打包旧提案的含义。同时还进行了一些细节改动，学习 tendermint 建议仔细阅读并保证理解。

```go
(PROPOSAL, h_p, round_p, v, lockedRound) // Proposal 消息，
(PREVOTE, h_p, round_p, nil/id(v))       // Prevote 反对/同意票
(PRECOMMIT, h_p, round_p, nil/id(v))     // Precommit 反对/同意票

Initialization:
  p                               /* 当前节点的 index */
  h_p := 0                        /* 当前高度，或正在执行的共识实例 */
  round_p := 0                    /* 当前轮次 */
  step_p ∈ {propose, prevote, precommit}  /* 当前步骤，可以是 propose, prevote, precommit 之一 */
  lockedValue_p := nil            /* 锁定的Block */
  lockedRound_p := -1             /* 锁定的轮次 */

upon start do StartRound(0, h_genesis)     /* 启动时从第 0 轮开始 */

Function StartRound(round, h):     /* 开始一个新的轮次 */
  round_p = round
  step_p = propose
  h_p = h
  if proposer(h_p, round_p) == p then   /* 如果当前节点是提议者 */
    if lockedValue_p ≠ nil then         /* 如果有锁定的区块 */
      v ← lockedValue_p          /* 使用有效的值作为提案 */
    else
      v ← getBlock()            /* 否则生成一个新的提案值 */
    broadcast (PROPOSAL, h_p, round_p, v, lockedRound_p)  /* 广播提案 */
  else
    schedule OnTimeoutPropose(h_p, round_p) to be executed after timeoutPropose(round_p)  /* 否则设置提案超时 */

// lockedRound 为 -1，说明 Leader 的上一个提案已经成功提交，这是一个全新的提案
upon (PROPOSAL, h_p, round_p, v, -1) AND step_p = propose do  /* 当收到提案并且当前步骤为 propose 时 */
  if valid(v) ∧ (lockedRound_p = −1 ∨ lockedBlock_p = v) then  /* 如果提案有效且符合锁定条件 */
    broadcast (PREVOTE, h_p, round_p, id(v))  /* 广播预投票 */
  else
    broadcast (PREVOTE, h_p, round_p, nil)  /* 否则投空票 */
  step_p ← prevote  /* 进入预投票阶段 */

// lockedRound 存在，说明 Leader 的上一个提案没有成功提交，发起了共识重试
// 此时如果本地节点已经存有上一个提案的 LockedQC，那么直接投同意票，否则投空票
upon (PROPOSAL, h_p, round_p, v, lockedRound) AND 2f + 1 (PREVOTE, h_p, lockedRound, id(v)) while step_p = propose ∧ (vr ≥ 0 ∧ vr < round_p) do  /* 当收到提案和足够的预投票并且当前步骤为 propose 时 */
  if valid(v) ∧ (lockedRound_p ≤ lockedRound ∨ lockedValue_p = v) then  /* 如果提案有效且符合锁定条件 */
    broadcast (PREVOTE, h_p, round_p, id(v))  /* 广播预投票 */
  else
    broadcast (PREVOTE, h_p, round_p, nil)  /* 否则投空票 */
  step_p ← prevote  /* 进入预投票阶段 */

// 当收到 2f+1 PREVOTE 的同意票时，产生 LockedQC
upon 2f + 1 (PREVOTE, h_p, round_p, id(v)) do
  if step_p = prevote then
    lockedValue_p ← v  /* 锁定提案值 */
    lockedRound_p ← round_p  /* 锁定轮次 */
    broadcast (PRECOMMIT, h_p, round_p, id(v))  /* 广播预提交 */
		step_p ← precommit  /* 进入预提交阶段 */

// 当收到 2f+1 的 PREVOTE 空票时，投 PRECOMMIT 空票
upon 2f + 1 (PREVOTE, h_p, round_p, nil) do
  if step_p = prevote then
	  broadcast (PRECOMMIT, h_p, round_p, nil)  /* 广播预提交空值 */
	  step_p ← precommit  /* 进入预提交阶段 */

// 从收到 2f+1 的投票开始计算超时
upon 2f + 1 (PREVOTE, h_p, round_p, *) do
	if step_p = prevote then
	  schedule OnTimeoutPrevote(h_p, round_p) to be executed after timeoutPrevote(round_p)  /* 设置预投票超时 */

// 当收到 2f+1 的 PRECOMMIT 同意票时，提交提案，并开启新的视图
upon 2f + 1 (PRECOMMIT, h_p, r, id(v)) do
   Commit(h_p, v)
   lockedValue_p ← nil  /* 重置锁定的值 */
   lockedRound_p ← -1  /* 重置锁定的轮次 */
   StartRound(0, h_p + 1)  /* 开始新的轮次，高度 + 1 */    
    
// 从收到 2f+1 的投票开始计算超时
upon 2f + 1 (PRECOMMIT, h_p, round_p, *) do  /* 当收到足够的预提交并且当前步骤为 precommit 时 */
	if step_p = precommit then
	  schedule OnTimeoutPrecommit(h_p, round_p) to be executed after timeoutPrecommit(round_p)  /* 设置预提交超时 */
    
upon f + 1 (*, h_p, round, *, *) while round > round_p do  /* 当收到足够的预提交并且当前步骤为 precommit 时 */
	StartRound(round, h_p) /* 更新 round 重新发起共识 */

Function OnTimeoutPropose(height, round):  /* 提案超时函数 */
  if height = h_p ∧ round = round_p ∧ step_p = propose then  /* 如果当前高度和轮次匹配并且步骤为 propose */
    broadcast (PREVOTE, h_p, round_p, nil)  /* 广播预投票空值 */
    step_p ← prevote  /* 进入预投票阶段 */

Function OnTimeoutPrevote(height, round):  /* 预投票超时函数 */
  if height = h_p ∧ round = round_p ∧ step_p = prevote then  /* 如果当前高度和轮次匹配并且步骤为 prevote */
    broadcast (PRECOMMIT, h_p, round_p, nil)  /* 广播预提交空值 */
    step_p ← precommit  /* 进入预提交阶段 */

// Unhappy Path 下的视图切换
Function OnTimeoutPrecommit(height, round):  /* 预提交超时函数 */
  StartRound(round_p + 1, h_p)  /* 开始新的轮次 */

```

# 视图切换
---

在之前的文章中我们了解了 HotStuff 快速的视图切换方案，现在来看一下 Tendermint 是如何做的，以及为什么说它牺牲了响应性。

Tendermint 并不存在 Happy Path 和 Unhappy Path，而是把这两种情况合二为一，复用同一套逻辑，这是 Tendermint 的一大优点。而 HotStuff 则是在发生超时时认为进入了 Unhappy Path，使用独立的超时逻辑解决，即发送 Timeout 消息，收集 highQC 以进行视图切换。而 Tendermint 无论是否成功提交，都会在 Precommit 阶段末尾使用同一套逻辑发起视图切换，如下图。

![Untitled](/assets/images/tendermint-img/Untitled%205.png)

为了简化理解，我们省略了其他节点接受的 Precommit 投票，仅针对下一任 Leader(节点2)收集投票过程进行分析。由于 Prevote 阶段中仅节点 1，4 收到了 2f+1 个 PREVOTE 投票，导致仅两个节点 Lock 该提案，而节点4又是拜占庭节点。因此在节点2收集 Precommit 投票时，在收到 2f+1 投票时有可能收集的都是提案处于 unlocked 状态，那么新 Leader 就会打包一个与节点1冲突的提案，因此需要多等待一个 ∆ 时间，即 precommit timeout，由于是部分同步系统，可以保证 Timeout 结束前能够收到所有诚实节点的消息，即节点 1 的 locked 消息，从而保证视图切换的安全。

实际项目中 timeout 时间是动态的，如果 timeout 当前时间短于实际需要的时间（即收到所有诚实节点消息的时间），那么就会导致频繁的共识失败，此时需要增加 timeout 时间，用于保证共识的活性。以下配置为 Tendermint Core 中不同超时时间的配置。

```go
# config/config.toml

[consensus]
...

timeout_propose = "3s"
timeout_propose_delta = "500ms"
timeout_prevote = "1s"
timeout_prevote_delta = "500ms"
timeout_precommit = "1s"
timeout_precommit_delta = "500ms"
timeout_commit = "1s"
/*
timeout_propose = how long we wait for a proposal block before prevoting nil
timeout_propose_delta = how much timeout_propose increases with each round
timeout_prevote = how long we wait after receiving +2/3 prevotes for “anything” (ie. not a single block or nil)
timeout_prevote_delta = how much the timeout_prevote increases with each round
timeout_precommit = how long we wait after receiving +2/3 precommits for “anything” (ie. not a single block or nil)
timeout_precommit_delta = how much the timeout_precommit increases with each round
timeout_commit = how long we wait after committing a block, before starting on the new height (this gives us a chance to receive some more precommits, even though we already have +2/3)
*/
```

其中 `timeout_commit` 在论文中并没有涉及，其实 precommit timeout 已经能保证下一任 Leader 收到所有诚实节点的 precommit 投票，实现正确的视图切换，因此 `timeout_commit` 自认为不是必要的，tendermint 源代码中也是可以通过配置跳过的。以下是源代码关于 `timeout_commit` 的部分注释。

```go
type ConsensusConfig struct {
  // ...
  
	// How long we wait after committing a block, before starting on the new
	// height (this gives us a chance to receive some more precommits, even
	// though we already have +2/3).
	// NOTE: when modifying, make sure to update time_iota_ms genesis parameter
	TimeoutCommit time.Duration `mapstructure:"timeout_commit"`

	// Make progress as soon as we have all the precommits (as if TimeoutCommit = 0)
	SkipTimeoutCommit bool `mapstructure:"skip_timeout_commit"`
	
}
```

HotStuff 视图之间的切换是通过对 highQC 的收集进行的，QC 作为 2/3+1 节点对当前状态的共识，可以作为一个 Status Cert 提供给下一个视图的 Replica 节点，说服它们接受当前新的提案。不光是视图的切换，从上一个阶段进入下一个阶段时，HotStuff 的 Leader 也需要等待 2/3+1 的有效投票，从而达成共识。

Tendermint 并不会像 HotStuff 那样轻易进行视图切换，而是在广播自己的投票进入下一阶段后等待一段时间 ∆ 来收集 2/3+1 节点的同意投票，这就分为 Happy path 和 Unhappy path 两种情况。

- Happy path 意味着超时时间内节点收集到足够同意票并完成提交，该提案达成共识，可顺利进行视图切换。
- Unhappy path 意味着节点在超时时间内并没有收集足够的同意票，需要等到超时结束。


Tendermint 基于部分同步模型，认为 ∆ 时间内可以收集到所有诚实节点的投票，因此对于 Unhappy path 来说系统永远需要等待一个固定时间才能进行视图切换，这就是为什么说 Tendermint 牺牲了响应性，更确切的说是在 Unhappy path 下丧失了响应性（Responsiveness），具体可以参考 [The latest view on Tendermint's responsiveness](https://informal.systems/blog/tendermint-responsiveness)。

# Q & A
---

**1. GST 究竟是什么？∆ 是什么？如果不等待 ∆ 会发生什么？**

GST 是基于假设的，没有明确的标志，我们认为当 Replica 发送一个消息的时刻是 在 GST 之后的，这样就能保证消息可以最终被接收到。进入 GST 就意味着系统<变为同步模型，此时再等待 ∆ 固定时间，即可认为 Leader 可以收到诚实节点的 highest locked 消息。

- GST：全局稳定时间，在 GST 之前，系统处于异步状态，消息不一定能够到达，在 GST 之后，系统处于同步状态，消息在 ∆ 内必定到达。GST 的值基于假设，只是一个逻辑概念，我们并不用关心具体数值，实际情况中，是需要调整 ∆ 来进行整体时间的选取的。
- ∆ ：GST 之后同步模式下的固定等待时间，是一个人为设置的系统参数，如同比特币的 10min。∆ 是通过不断调试系统活性得出的，这就如同问比特币 10min 出块时间是怎么确定的一样。通过实践和调试证明，比特币 10min 的出块时间或者以太坊 15s 的出块时间可以保证系统安全性和活性，但无论怎样，由于部分同步模型中存在异步的部分，即使有一个 ∆ 还是比纯同步模型要快的多。

**2. HotStuff 没有 ∆ 为什么也是部分同步模型？**

Tendermint 由于引入了 ∆ 可以尽量避免 ViewChange 失败，保证在 ∆ 之后新 Leader 一定能够获取到真正的 Highest Lock。HotStuff 无法保证这一点，但它存在超时时间，加入 ViewChange 阶段超时，HotStuff 就会更换 Leader 再试一次。Tendermint 就好比领导和你说，「我给你 3 天时间把事儿做完，参考历史情况绰绰有余！」，而 HotStuff 只给你 2 天，做不完就不断换人。

HotStuff 的超时时间本质上也是一个 ∆ ，它会告诉你，「兄弟，你大概率会在超时时间之内收到消息，但也有可能收不到，如果没有收到，你就弹劾那个 Leader！」而 Tendermint 告诉你，「兄弟，你老实等 ∆ 时间，以我的经验，肯定能收到所有消息，实在不行你再换 Leader 再来一遍。」

HotStuff 的超时时间，效果与等待 ∆ 时间是一样的。它们都依赖设定超时时间，设想这个超时时间无限小，你还能保证 HotStuff 和 Tendermint 的活性吗，因此它们都不是异步模型。

不同的是，视图切换时 HotStuff 不需要等待更长的时间来保证收到所有诚实节点的状态，因此能做到更快（大不了重试），所以可以用更低的成本更换 Leader 来解决异步场景下带来的活性问题。而 Tendermint 往往需要等待更长的时间，保证收到所有诚实节点的消息来规避掉了这个问题，代价就是降低了吞吐。

# 参考资料
---

- [Dive Into Tendermint Consensus Protocol](https://coinexsmartchain.medium.com/dive-into-tendermint-consensus-protocol-i-fbc9b56f75e8)
- [The latest view on Tendermint's responsiveness](https://informal.systems/blog/tendermint-responsiveness)
- [Compared with traditional PBFT, what advantage...](https://www.reddit.com/r/cosmosnetwork/comments/8i42qa/compared_with_traditional_pbft_what_advantage/)
- [On the Design and Accountability of Byzantine Fault-Tolerant Protocols](https://www.youtube.com/watch?v=MJ8NxwmBFhU)
- [Tendermint Explained — Bringing BFT-based PoS to the Public Blockchain Domain](https://blog.cosmos.network/tendermint-explained-bringing-bft-based-pos-to-the-public-blockchain-domain-f22e274a0fdb)
- [Tendermint-2-共识算法:Tendermint-BFT详解](https://blog.csdn.net/weixin_43988498/article/details/119031945)
