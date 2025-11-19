---
title: Substrate x Hotstuff 开发记录 - 区块同步
layout: post-with-content
post-image: "https://xufeisofly.github.io/picx-images-hosting/substrate-hotstuff-sync/head.4n856jrc20.jpg"
tags:
- substrate
- hotstuff
- blockchain
---

Substrate 是 Polkadot SDK 的区块链节点开发框架，我在尝试使用 Fast Hotstuff 替换其默认的共识和最终性协议（Aura + Grandpa），将期间遇到的知识、问题和思考在此记录。

由于接入了 Hotstuff 后区块链的吞吐量增大很多，使得新节点加入更难追上共识产出新区块的速度，Substrate 中虽然提供了 Warp Sync，但由于其与 Grandpa 耦合，在接入时也遇到了一些问题，我参考了 Tendermint 的 State sync 设计，简化默认的快照同步逻辑，同时对源码的一些修改也在这里记录，供读者参考。

下面首先介绍 Subtrate 的区块同步协议，以此分析它要如何实现一个 BFT 共识的快照同步。

# Substrate 同步协议
---

## 代码架构概述
与 Substrate 很多模块类似，sc-network-sync 使用 service + worker 的 Actor 架构，由`SyncService`提供外部调用接口，由`SyncEngine`负责底层的逻辑和计算，两者直接通过 oneshot channel 通讯。结构大致如下图，这里不进行详细介绍。

![image_1](https://xufeisofly.github.io/picx-images-hosting/substrate-hotstuff-sync/image_1.szdnl9e4q.png)

## 三种同步模式
一个区块链需要有高效的同步策略以应对不同的同步场景，比如 validator 节点可能突然掉线，需要同步区块回归共识进程，比如非 validator 节点往往是通过从其他节点那里实时同步区块而保持状态，再比如新节点加入网络后，需要尽快同步所有数据。

Substrate 提供了三种同步模式，

+ Full Mode：同步完整的缺失区块，包含 header 和 body，并执行 body 中的交易，相当于 Replay，性能最差，适合已运行节点某区块缺失时的偶发同步。
+ LightState Mode：仅同步缺失区块的 header，不执行交易，最后同步最新区块对应的 State 快照，用于节点区块相差很多时快速追赶。
+ Warp Mode：直接同步最新 finalized 的区块 header 以及对应的 State 快照，让节点快速获取最新状态，之后在后台逐渐同步历史区块，比较适用于新节点加入网络。

在实现上，Substrate 设计了三种同步 Strategy，对应了代码中三个模块：

+ ChainSync Strategy：负责区块同步。
+ StateSync Strategy：负责状态同步。
+ WarpSync Strategy：负责 warp proof 和 target 区块同步。

不同的同步模式是由上述的三种同步策略组合实现的，如下图。

![image_2](https://xufeisofly.github.io/picx-images-hosting/substrate-hotstuff-sync/image_2.3rbnr3hnm3.png)

## ChainSync - 区块的同步
ChainSync 主要为了同步区块和 Finality 证明(Justification)。

ChainSync 中 `block_requests()` 函数负责构建区块的请求，并放入 StrategyActions，其中包括了

+ 哪些区块需要被同步。
+ 要从哪些 peers 同步。
+ 需要同步区块的具体属性。

### PeerSync 的维护
每个 node 时刻维护了网络中其它节点的同步状态，这个状态作为 ChainSync 的基础，内容包括：

```markdown
PeerSync:
  - peer_id
  - common_block # 本节点认为的该 peer 与本地最新 common block
  - best_block   # 本节点认为的该 peer 的最新 best block
  - peer_sync_state # 当前对该 peer 同步的状态，如 Available, AncestorSearch, DownloadingNew 等
```

**可见，PeerSync 的 common_block 和 best_block，与本节点链状态的 diff，就决定了从该 peer 处同步区块的逻辑。**

PeerSyncState 是一个枚举类型：

```markdown
enum PeerSyncState:
  - Available # 空闲
  - AncestorSearch # 寻找 common_block
  - DownloadingNew # 下载新区块
  - DownloadingStale # 下载旧区块（低于我们的 best block）
  - DownloadingJustification # 单独下载 Justifications
  - DownloadingState # 下载 state，ChainSync 中也包含了 StateSync 对象
  - DownloadingGap # 在 warp sync 之后下载 block history
```

从这个 enum 总结出 ChainSync 有以下职责：

+ 寻找 common_block，维护 PeerSync。
+ 下载区块，包括 Header + Justifications + Body。
+ 下载 State，为了方便，ChainSync 中也包含了 StateSync 对象。
+ 下载 Gap，仅用于 Warp Mode，正常来说 Full Mode 和 LightState Mode 都是按顺序依次同步的，但 Warp Mode 是先同步最新区块，之后在慢慢同步历史区块。

只要收到了某 peer 的 block 数据，就会尝试更新其 PeerSync 的 common_block 和 best_block，而只要本节点对该 peer 做出了相应的动作或者收到其消息，就会更新其 PeerSyncState。比如：

+ 收到该 peer 的 block announcement
+ 或者向该 peer 发出 block request 然后收到 response

### 同步优先级
ChainSync 同步的策略存在以下优先级：

1. 当本地的 best_block 领先 peer.common 太多时，尝试更新 peer.common。
2. 更新 Canonical chain，即 peer.common -> peer.best，这一段的具体策略见 `needed_blocks`。
3. 更新 Fork chain。
4. 如果 peer.common 之前存在 gap，则更新 gap，gap sync 负责更新 peer.common 之前缺失的部分，优先级最低，且为了节省带宽，GapSync 的 max_parallel 被设置为了 1。

四种优先级的逻辑展示如下图，其中红色代表需要同步的部分：

![image_3](https://xufeisofly.github.io/picx-images-hosting/substrate-hotstuff-sync/image_3.8ok4kxvff1.png)

### 计算需要同步的 Blocks
`needed_blocks`函数决定了从 peer.common 到 peer.best 这段区间中应该下载的区块，它在选择时考虑了如下方面：

+ 即使同一个 range 正在下载，也应该达到要求的并行度 `max_parallel`
+ 如果这段范围中存在 gap，则下载这个 gap

所以 peer.common 和 peer.best 的确定就尤为关键。ChainSync 同步可以认为分为两种子模式：

+ `ChainSyncMode::Full`
+ `ChainSyncMode::LightState`

不同子模式影响同步区块所需要的 attributes，前者为 `Header + Justifications + Body`，后者为 `Header + Justifications`。

## StateSync - 状态的同步
StateSync 用来在本地区块头已拿到(有目标 header / body / justifications)，但没有对应完整状态时，通过按键范围分段下载和(可选) Trie range proof 验证，构建该目标区块的状态数据库，然后一次性导入。proof 验证是对部分 MPT 哈希值的匹配验证，逻辑不赘述。

一个 State 请求如下：

```
StateRequest {
  block: target_hash,
  start: last_key (可能是多段：主 trie 或子 trie 路径),
  no_proof: skip_proof
}
```

当下载完成 State 并完成 proof 验证之后，构建导入请求。

```rust
let origin = BlockOrigin::NetworkInitialSync;
let block = IncomingBlock {
    hash,
    header: Some(header),
    body,
    indexed_body: None,
    justifications,
    origin: None,
    allow_missing_state: true,
    import_existing: true, // 允许重复导入
    skip_execution: true,
    state: Some(state), // 附带 State
};
self.actions.push(SyncingAction::ImportBlocks { origin, blocks: vec![block] });
```

## WarpSync - 快照同步
### BFT 快照同步的基本思路
一个新节点的加入，不可能使用 Full Mode 或者 LightState Mode 从创世块逐一开始同步，即使只同步 Block Header 也是一个线性增长的复杂度，因此我们需要快照级别的同步，即直接同步最新的 Finalized 区块及其状态，并从该区块往后继续同步。我们可以称这个最新的 Finalized 区块为 Target Block。

那么核心就是要证明 Target Block 确实可以 Finalize，因此需要提供 Finalize Proof，由于目前主流区块链的 Finality 协议基本都是依赖 BFT 投票，Finality Proof 自然就是那个 2/3+1 权重的签名。比如，如果我们使用 Fast Hotstuff 作为共识（包含了 Finality 协议），那么它的 Finality Proof 就应该是下面的样子。

![image_4](https://xufeisofly.github.io/picx-images-hosting/substrate-hotstuff-sync/image_4.8adou2n4jx.png)

图中包含 Target Block 已经之后的两个 QC（Fast Hotstuff 需要连续两个 QC 才能提交一个区块），以及验证 QC 所需要的 Validators，只要完成两个 QC 的签名验证即可得到证明，然而问题是：**如何知道 QC 对应的 ValidatorSet，或者说，如果某节点提供了 ValidatorSet，如何证明它是真实的？**

### Substrate 做法
Substrate 中的 WarpSync 目前是针对 Grandpa 开发的，Grandpa 与 Hotstuff 类似都是一个 BFT 类的协议，因此都会产生一个 2/3+1 节点的证明。因此理论上它只需要同步 Finalized 的区块（Target Block）及其状态，帮助新节点快速跟上进度。而历史区块可以之后在后台慢慢同步。

WarpSync 可以分解成三个步骤

+ Warp Proof 下载与验证。
+ Target Block 的 Header + Justifications 同步。
+ Target Block 对应 State 同步。

如下图。

![image_5](https://xufeisofly.github.io/picx-images-hosting/substrate-hotstuff-sync/image_5.64ea8avgsu.png)

然而实际情况是，warp proof 不得不回溯到 Genesis block，问题就发生在 proof 的验证上，warp proof 的目的就是证明 target block 的合法性，我们可以看一下 grandpa WarpProof 的大致结构，warp proof 由多个区块的 finality proof 组成，这些区块有个特点，它们都是记录 authority set 变化的区块，

```markdown
WarpProof:
  - proofs: [(block_hash, finality_proof)]
```

此处可能会产生疑问，为什么需要多个区块的 finality proof？只有 target block 的 finality proof 为什么不够？那是因为 finality proof 的验证需要知道当时签名的 authority set，而 grandpa 并不是每个区块中都记录了对应的 authority set，而是只有 authority set 改变的时候记录在对应的区块中。因此若想知道 target block 的 authority set，就需要之前在此之前的那一个负责变更 authority set 的区块，而那个区块也需要进行 finality proof 的验证，验证它又需要它的 authority set，而它的 authority set 又记录在它之前的那个变更区块中……如此递归，我们发现最终会回溯到 genesis block，因为 genesis block 记录了最开始的 authority set，所以 warp proof 是一条由 authority set 变更的区块及其 finality proof 组成的证据链，可以成为 authority change 链，有了这条完整的证据链才能证明 target block 的合法性。上述原理如下图。

![image_6](https://xufeisofly.github.io/picx-images-hosting/substrate-hotstuff-sync/image_6.96a69iwszz.png)

总的来说，为了同步一个区块的快照，我们不需要同步之前的所有区块，因为只有部分区块记录了 authority set 的变更，但仍然需要回溯到创世块。如果总区块数量很多且 authority set 变更频繁，仍然需要很长时间完成同步。有没有更快的方法呢，我们不妨看一下 Cosmos 是怎么做的，它使用的 Tendermint 是一个典型的 BFT 共识协议，并没有单独的 Finality 协议，与 Hotstuff 十分类似。

### Cosmos 做法
与 grandpa 类似的是，Tendermint 要验证 commit proof（即 finality proof） 同样需要知道其 validator set，但不同的是，为了让每个区块都有自己的 validator set，tendermint 每个区块中都记录了当前区块的 validator set 和下一个区块的 validator set，因此从上一个区块中就可以获取到当下区块需要的 validator set 逐一进行区块的验证，验证时仍然需要回溯到 genesis block，不过 substrate 只需要回溯 checkpoint 区块，而 tendermint 不存在 checkpoint 需要每个区块进行验证，反而更多了。

Cosmos 为了加快回溯的速度，做了一个很巧妙的事情，只适用于 BFT 共识，它提供了一个 trusted block 概念，并实现了一个跳跃式验证机制。

1. Trusted block 是已被信任的区块，genesis block 当然是一个 trusted block，除此之外我们还可以从区块链浏览器上人为获取一些已经被 finalized 的 block 作为 trusted block。
2. 使用跳跃验证扩散 trusted block。如果某个区块与 trusted block 的 validator set 重叠度超过 1/3，则该区块直接可信，验证复杂度降为 O(logN)。如下图。

![image_7](https://xufeisofly.github.io/picx-images-hosting/substrate-hotstuff-sync/image_7.4qrr49kes0.png)

因为如果一个区块的 validator set 与 trusted block 的 validator set 重叠度超过 1/3，则说明一定有一个诚实节点同时参与这两个区块的签名投票，那么该区块自然就会变为 trusted。这样，trusted block 会不断向前推进，如果无法通过 1/3 重叠的逻辑进行跳跃式扩散，那么就需要一个一个区块进行推进，直到将 Target block 也变为 trusted block，即可完成同步。甚至，如果一开始 Target block 的 validator set 就与 trusted block 的 validator set 有超过 1/3 的验证者重叠，那么 Target block 就会直接完成验证。

# Substrate x Hotstuff 秒级快照同步
---

Substrate 之所以敢于逐个进行验证，是因为存在 session 概念，只有在 session 轮换之时才能变更 authority set，因此 AuthorityChange 链本来就不长。但 Cosmos 每个区块都可能有自己的 validator set ，validator set  的变化并不需要等待 session 轮换，因此如果逐个回溯工作量巨大，因此才有了基于 1/3 validator set  重叠就可以让信任扩散的跳跃式验证机制。

Hotstuff 共识更贴近 Cosmos 的验证思路，这里有一个很重要的问题：在 Cosmos 中，是否一定能找到一个 trusted block 使其 Validators 与 target block 的 Validators 重叠度超过 1/3？不能，因为 Cosmos 中每个区块的 Validators 都有可能改变。但是 Substrate 是可以的，因为只有 session 轮换时 Validators 才发生变化，而一个 session 中会有很多区块。

因此 Substrate x Hotstuff 可以借鉴 Cosmos 的信任扩散思路，人工提供 trusted block，但我们会尽量找到一个距离 target block 很近的 trusted block，力求直接将信任一次性扩散到 target block 上。大致过程如下：

+ 人工从区块链浏览器中寻找一个 trusted block，尽量靠近 target block
+ 节点会根据 trusted block 自动获取其 validators
+ Proof 验证时会计算 trusted_block.validators 与 target_block.validators 是否超过 1/3 重叠，超过则完成验证
+ 如果不超过，不会进行二分查找，而是直接报错返回，新节点启动失败，重新寻找一个更近的 trusted block 重试即可。

由于 Substrate 中 target block 之前一定存在一个与其 validators 相同的 trusted block，只需要找到该 trusted block 并重试就一定可以通过验证，这样就避免了一个随意选择的 trusted block 而触发频繁二分查找造成的时间浪费，实现秒级的快照同步。



# Substrate 源码其它调整
---

**1.放开区块导入限制**

Substrate 的区块导入和 finalize 依赖从 genesis 开始的完整的区块链，在 WarpSync 时，也是先按顺序导入 Proof 当中之前所有的 checkpoint 区块，再导入 Target Block。对于纯 BFT 共识来说，由于不会分叉，区块的验证不需要能够回溯到 genesis block，因此需要允许一个确实了父区块的区块导入和 finalize。

**2.屏蔽 WarpSync 后的 GapSync**

warp 同步了某个 finalized block header 及其 state 之后，会回到 ChainSync 的同步策略，同时创建一个 GapSync(from: genesis_block, to: warp_block)。由于 GapSync 在 ChainSync 中优先级最低，因此只有在没有最新区块可以同步时才会同步（所以官方声称这些历史区块的同步是 background）。然而对于 BFT 链来说，历史区块实际上不是必要的，我们完全可以去掉这个 GapSync 以减缓 CPU 和带宽压力。或者即使有 GapSync，也并不需要使用 ChainSyncMode::Full 去同步（会同步 Body 数据并执行交易），而是可以改成 ChainSyncMode::LightState（仅同步 Header + Justifications），这会大大提升同步效率。

# 推荐阅读
---

+ [What kinds of sync mechanisms does Substrate implement?](https://substrate.stackexchange.com/questions/334/what-kinds-of-sync-mechanisms-does-substrate-implement/)
+ [Tendermint-core source code](https://github.com/tendermint/tendermint/tree/main/statesync)
+ [Substrate source code](https://github.com/paritytech/polkadot-sdk/tree/master/substrate/client/network/sync)

