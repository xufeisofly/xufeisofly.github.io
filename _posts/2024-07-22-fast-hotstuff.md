---
title: Fast HotStuff 理解和思考
layout: post-with-content
post-image: "https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/head.1e8rdphh1l.jpg"
tags:
- hotstuff
- consensus
- blockchain
---

# 原生 HotStuff 的局限
---

相比其他 BFT 类共识算法，HotStuff（下文简称 HS） 通过增加一个投票阶段的方式实现了正常和异常情况下 O(n) 的通讯复杂度，并且没有牺牲响应性（Responsiveness）。具体请参考上篇文章[HotStuff 工程设计与实现](https://xufeisofly.github.io/blog/shardora-hotstuff)。

然而，HS 有以下两个局限：

- **交易确认延迟高**

为了在超时视图切换时达到线性时间复杂度，同时避免像 Tendermint 之类共识一样牺牲系统响应性，HS 在 Lock 之前增加了一个 Key 阶段（即 Prepare 阶段），成为了一个三阶段投票的共识算法。多数工程实现都采用 Chained HotStuff，因此多一个阶段对吞吐量并没有太大影响。然而交易打包到最终提交期间的延迟却增大了一个视图的时间。

- **分叉攻击影响吞吐量**

HS 对于新交易的验证逻辑保证了共识协议的安全性（Safety）和活性（Liveness）。在 Chained HotStuff 中，每个提案通过 QC 指针与前一个提案相连，当一个 Leader 节点打包新的提案时，并没有要求这个提案必须从最新块之后打包，只需要在最新的 Locked 提案之后即可。这就使得恶意节点可以进行分叉攻击。如下图。

![Untitled](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/Untitled.5fkqs3lkem.png)

新提案 View4 本应该从 View3 之后出块，但恶意节点可以选择从 View1 （最新 Locked 视图）之后出块而且能顺利通过验证。这种攻击行为不会对系统安全性造成影响，却令吞吐量大幅下降，极端情况如下图。

![Untitled-1](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/Untitled-1.3k85zh94sr.png)

Fast HotStuff 论文中给出了在持续的分叉攻击下，随着拜占庭节点数量 f 增加，系统吞吐量的改变情况。可以看到 f 增加时，系统吞吐量快速下降（Average Case & Worst Case）。

![Untitled-2](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/Untitled-2.5tr6iytv9n.png)

# Fast HotStuff 逻辑
---

Fast HotStuff(下文简称 FHS) 的核心逻辑，就是**确保新提案永远从最长的链后继续出块**，**即 Leader 确保提案中的 QC 是最新的 QC（即 HighQC），并为之提供证明**。为此 FHS 分别讨论了正常路径（Happy path） 和超时路径（Unhappy path）两种情况，如下图：

![Untitled-3](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/Untitled-3.73u3pabuks.png)

- Happy path: 由于提案的 View 是 HighQC 决定的，因此在 Happy path 下，Replica 只需要要求新提案的 View 必须大于自己的 CurView 即可保证 QC 为 HighQC。
- Unhappy path: 超时触发视图切换后，Leader 收集 2/3+1 节点各自的 HighQC 及其签名，生成聚合签名 AggQC，作为 HighQC 的证明。新提案必须根据此 HighQC 生成。

FHS 论文中证明了，只要满足「新提案永远打包 HighQC」这个事实，即可保证**提案在收到两个 QC 后就可以安全提交**，且不需要像 Tendermint, Casper 和 两阶段 HotStuff 那样等待一个网络延迟。

> Indeed, the presence of the proof for the *highQC* guarantees that a replica can safely commit a block after two-chain without waiting for the maximum network delay as done in two-chain HotStuff [7], Tendermint [5], and Casper. — 《Fast HotStuff》
> 


然而，由于 Unhappy path 视图切换时需要收集 2/3+1 节点对各自 HighQC 的签名，FHS 需要使用能够对不同消息进行签名的聚合签名来替换原生 HS 的阈值签名，这使得 Unhappy path 视图切换的通讯复杂度升级为 O(n^2)。

> Unfortunately, we are not able to avoid the transfer of quadratic view change messages over the wire during view change due to primary failure (also called unhappy path). — 《Fast HotStuff》
> 


好在 FHS 优化了验证逻辑，只需要验证两个 QC，在节点数量可控的情况下，实测计算耗时没有太大影响。下文对聚合签名进行介绍。

# Unhappy Path 下为什么需要 AggQC
---

上文所述，AggQC 是 2/3+ 节点各自 HighQC 的聚合，从中我们可以获取到系统目前公认的最高的 HighQC（至少是 2/3+ 节点公认的）。当某视图共识异常而导致视图切换时，所有节点的状态可能参差不齐，新 Proposer 的状态也可能不是最新的，这就需要提供一个全网认可的、最新的状态，并基于此状态继续进行共识。AggQC 在系统因视图切换而发生状态错乱时，能够从中获取之前最高的 HighQC 及其证明，从而保证所有的 Proposal 都是基于最高的区块向后延伸的。
	
那么是否可以不使用 AggQC，毕竟签名与验签十分耗费性能，会拉低吞吐量，或者说不使用 AggQC 是否可以同样保证 Proposer 发布的提案打包的是最高的 HighQC？结论是否定的，且这会导致共识丧失活性。
	
举个例子。下图中，初始状态下 4 个节点的 HighQC 均为 QC3，在 View4 的共识中节点 4 成功生成 QC4，然而广播 QC4 时只有节点 3、4 接收到因而触发超时视图切换。此时节点 1、2 的 HighQC = QC3，节点 3、4 的 HighQC = QC4。
	
![image20](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/image20.70ahrkiruy.png)

在视图切换时，新 Proposer 节点 1 收集到节点 1、2、4 的 HighQC 均为 QC3（其中节点 4 为拜占庭节点）。此时我们不生成 AggQC，让节点 1 打包自己认为的 HighQC 即 QC3 并广播提案（视图 View 4'），会发生什么？

结果便是：节点 1、2 投同意票，节点 3 由于发现 QC3 小于自己的 QC4 而拒绝该提案，节点 4 作为拜占庭节点投同意票，新 Proposer 为 View 4' 生成 QC4'。不幸的是，下一轮的 Proposer 恰好是拜占庭节点 4，那么此时节点 4 同时拥有 QC4 与 QC4'，它可以选择基于两者任何一个分支发布新的提案并且都会获得通过（如果它打包 QC4 会获得所有节点的同意，如果它打包 QC4' 会获得除了节点 3 外其余节点的同意）。

![image21](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/image21.67xm9u264q.png)

此外它还可以选择将 QC4' 仅广播给节点 1、2，自己则为节点 3 站台 QC4，这样系统便无法进行视图切换而丧失活性。如下图。

![image22](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/image22.3uuzsmocy3.png)

出现这种情况的直接原因是，不携带 AggQC 导致同时出现 QC4 与 QC4'，并且两者都是合法的，最终可能造成状态分叉。而使用 AggQC(HighQC=QC3) 的意义便在于告知所有节点 QC4 是非法的，从而确定 QC4' 这唯一一个合法分支。

# FHS 使用聚合签名
---

论文中，FHS 和原生 HS 有个很大的区别，就是使用聚合签名而非阈值签名，其实不仅仅是 FHS，如 LibraBFT 等工程实现中，也都将阈值签名换成了聚合签名，这是因为聚合签名相比阈值签名有很多工程优势。

**阈值签名（Threshold Signature）**
    
    基于沙米尔秘密分享原理（多项式），每个节点初始时仅拥有部分私钥和联合公钥。利用部分私钥对同一个消息体进行签名，当 Leader 收到任意 m 个部分签名后可以从中恢复出完成的签名并使用联合公钥进行验证。
    
    阈值签名最大的问题是一开始需要进行 DKG（分布式密钥生成）来分发部分私钥并生成联合公钥，并且分发的过程需要进行复杂的验证。然而一旦 DKG 完成，在签名和验签阶段效率较高。
    
**聚合签名（Aggregation Signature）**
    
    能够将 m 个签名「压缩」成一个聚合后的签名，仅对该聚合后的签名进行验证就能证明这 m 个签名的合法性，不需要单独一个个验证，但验证时仍然需要使用 m 个签名对应的公钥，因此聚合签名中是包含「谁签了名」这个信息的。
    
    聚合签名最大的特点是节省了存储 m 个签名所用的空间，但计算过程时间复杂度比阈值签名要高，其验证过程看似一步完成，本质上仍然需要一个 m 次循环逐一进行配对计算。
    
    此外聚合签名有一个重要特点，就是允许对不同的消息体做签名，这和阈值签名是很不一样的。阈值签名要求所有的部分签名都是针对同一个消息的，否则无法恢复出完成签名。这也是 FHS 使用聚合签名的主要原因。
    

下面代码是 bls12 的聚合签名验证逻辑，可见验证过程拥有线性时间复杂度：

```go
func (bls *bls12Base) coreAggregateVerify(publicKeys []*PublicKey, messages [][]byte, signature *bls12.PointG2) bool {
	n := len(publicKeys)
	// ...
	engine := bls12.NewEngine()

	// O(n) 复杂度
	for i := 0; i < n; i++ {
		q, err := engine.G2.HashToCurve(messages[i], domain)
		if err != nil {
			return false
		}
		engine.AddPair(publicKeys[i].p, q)
	}

	engine.AddPairInv(&bls12.G1One, signature)
	return engine.Result().IsOne()
}
```

无论是阈值签名和聚合签名都可以实现 HS 所需要的 m-n 签名。

对比阈值签名，聚合签名最大优点是节省了复杂的 DKG 过程，并且可以记录签名者信息，这在 HS 实现时可以避免恶意不投票行为。然而，聚合签名的验证过程时间复杂度高，如果有 N 个签名进行了聚合，那么多这个聚合签名验证时仍需要 N 次配对计算，随着节点数量的增多，聚合签名的验证时间会线性增高，同时由于 AggQC 中携带了 `2/3+1` 节点的 HighQC，Unhappy path 下新提案的带宽也会增加。

下表为阈值签名和聚合签名特性对比。

![Untitled-4](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/Untitled-4.41y7o2aido.png)

FHS 在 Unhappy path 时对 HighQC 进行聚合签名生成 AggQC，打包了 AggQC 的提案相当于告诉所有的 Replicas 自己是从最长链之后出块的，因此可以避免分叉攻击，Replica 需要对 AggQC 进行两个方面的验证：

- **验证 AggQC 是否合法**。如果合法，说明该 AggQC 确实饱含了 `2/3+1` 个节点对于自己 HighQC 的签名。
- **验证 AggQC 中最高 HighQC 是否合法并且是否是最新的**。如果仅验证 AggQC 合法性，恶意 Leader 完全可以使用一个旧的 AggQC 作为证明，因此 Replica 还要对 AggQC 其中的 HighQC 进行验证，确保确实比本地的 HighQC 高。


一旦 Replica 发现 AggQC 不满足上述两个条件，就会做出惩罚，这就增加了恶意 Leader 进行分叉攻击的成本。

## 对于工程实现的一个疑问

和 TC 方案不同，TC 方案中所有的节点是对 CurView 做阈值签名的，这就要求 `2/3+1` 节点的 CurView 是一致的，而 FHS 并不需要保证这一点，那么当 Leader 收到 `2/3+1` 个超时消息时，是如何能保证这 m 个消息是针对统一视图产生，里面没有旧的消息呢？

即，在异步情况下，新 Leader 是如何得知这 2/3+1 对 HighQC 的签名是针对统一视图的？🤔

# 不同签名对 HotStuff 激励机制的影响
---

阈值签名和聚合签名还有一个区别，即阈值签名无法得知具体参与签名的节点是哪些，而聚合签名可以。因此阈值签名方案便存在一个问题：**无法甄别恶意 Replica 的故意不投票行为**。

对于一个恶意的 Replica，在收到诚实 Leader 的提案后可以不进行投票，而在收到与之串谋的恶意 Leader 提案时再投票，由于使用了阈值签名，我们无法知道参与了投票的节点有哪些，导致这种行为并不会收到任何惩罚。而当一个恶意行为零成本时，就会导致恶意节点越来越多，最终打破拜占庭假设，造成共识系统彻底崩溃。

而使用了聚合签名便可解决该问题，如 LibraBFT。聚合签名可以得知具体参与投票的节点身份，因此可以：

- **给参与投票的节点一定激励**
- **提升参与投票的节点下次称为 Leader 的概率**


不过需要说明的是，聚合签名由于是 Leader 生成的，该 Leader 完全可以对该投票节点的列表进行恶意操作，比如去掉自己不喜欢的节点，而放入串谋节点的签名。因此我们虽然可以保证聚合签名的正确性，但却无法完全保证聚合签名的真实性，这个问题需要额外进行解决。下图中示意了恶意 Leader 在收集投票签名时，故意不使用 Sig3 而使用 Sig4 的过程。

![Untitled-5](https://xufeisofly.github.io/picx-images-hosting/fast-hotstuff/Untitled-5.45u7dzhql.png)

# 总结
---

FHS 解决了 HS 工程实现中的重大问题，即分叉攻击导致吞吐量降低，并且减少了一个阶段使得交易确认延迟降低。

为此，其使用聚合签名代替阈值签名，除了可以保证 Unhappy Path 下的依然从最长链之后出块以外，还带来了诸多工程上的便利。如消除 DKG，解决避免恶意不投票行为等。

但对于超大量节点的共识场景（比如超过 5000 节点），由于聚合签名验证过程提升了复杂度，FHS 的可行性需要额外验证。同时，超时的工程实现中可能仍然需要生成 TC 已保证视图的一致性。

# 参考资料
---

- [HotStuff: BFT Consensus in the Lens of Blockchain](https://arxiv.org/abs/1803.05069)
- [Fast-HotStuff: A Fast and Robust BFT Protocol for Blockchains](https://arxiv.org/pdf/2010.11454)
- [State Machine Replication in the Libra Blockchain](https://diem-developers-components.netlify.app/papers/diem-consensus-state-machine-replication-in-the-diem-blockchain/2019-06-28.pdf)
- [https://github.com/relab/hotstuff](https://github.com/relab/hotstuff)
