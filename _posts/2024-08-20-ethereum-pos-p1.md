---
title: 以太坊的 PoS - Part1 共识协议总览
layout: post
post-image: "https://xufeisofly.github.io/picx-images-hosting/ethereum1/head.77dpnis7t5.jpg"
tags:
- ethereum
- consensus
- blockchain
---

我打算写一个系列记录干掉以太坊的过程，系列中包括对以太坊的技术学习也包括自我思考和设计，逐步将好的机制应用到 Shardora 之中。由于之前几篇文章都是针对区块链共识算法的，因此打算先从以太坊的 PoS 入手。

在 Shardora 共识协议开发之前我产生过两个疑惑？一个是 BFT 共识居然要求少于 1/3 的恶意节点，这听起来是个很强的信任假设，这真的可行吗？另一个是 HotStuff 共识这么好用，为什么主流的这几条公链不用，只是因为历史原因吗？在了解了以太坊 PoS 设计过程之后，我基本有了答案。接下来我会用三篇文章介绍以太坊 PoS 的设计思路和具体原理。本篇内容为前置知识和总览，第二篇和第三篇将分别介绍 LMD GHOST 协议和 Casper FFG 协议，这两个协议是以太坊 PoS 的核心。

# 补充两个共识协议中的概念

---

## 最终确定性

其实很多共识协议中，当区块链中增加一个区块时，这个区块是有可能会被回滚的，也就是还没有最终确定下来。最终确定性是共识协议安全性的一个表现，它表示区块链中有些区块是无法再被回滚的。

我们之前讨论的 HotStuff 或者 Tendermint 等 BFT 类共识协议中，每个提案需要收集 2/3 节点的投票，并且经过 2-3 轮的签名收集，因此每个区块在提交时都具备最终确定性。这也是 BFT 类共识不分叉的原因。但 PoW 和其他很多中本聪共识实际上并没有最终确定性，当一个区块打包发送给验证者并被接受后，由于存在分叉，这个区块所在的分支可能会被另一个分支取代，从而造成回滚。对于 PoW 来说，理论上只要你算力足够强大（比如达到 51%），你可以回滚除创始块之外的任意一个块，而我们通常所说的比特币 6 个区块后确认，只是因为我们认为以恶意节点的算力赶超 6 个区块几乎是不可能的。因此随着某个区块之后的区块个数不断增加，我们可以认为这个区块在**不断逼近「被确认」的状态**，但永远不会 100% 达到。

## 分叉选择规则

我们口口声声所说的区块链实际情况下可能是一个「区块树」，即存在多个分支，如下图。

![image](https://xufeisofly.github.io/picx-images-hosting/ethereum1/image.8adeyeo1os.png)

这是因为网络环境是复杂且不可靠的，比如由于消息延迟等原因节点同时收到两个合法区块，区块链就会出现分叉。分叉选择规则就是用于在有多条分叉出现的情况下，告诉节点选择哪一条链作为合法链，从而从该链之后继续出块。

- 比特币和 PoW 版本的以太坊使用「最长合法链」作为分叉选择规则，因为最长的链代表了这一条路径下累计了最多的工作量。
- 当前以太坊的 Casper FFG 协议通过被最终确定的检查点高度来选择分支，即选择高度最高检查点所在的分支，而其使用的 GHOST 协议使用权重最大的子树作为合法链。
- 传统的 BFT 类共识协议已提交的区块不存在分叉，无需分叉选择规则。而未提交的区块往往选择拥有 HighQC 的分支并继续出块。

# 以太坊 PoS 设计所面临的问题

---

以太坊从 2014 年开始研究从 PoW 转向 PoS 的方案，这经历了漫长的过程，直到 2022 年信标链与主网合并，以太坊的共识协议才正式从 PoW 切换到 PoS。对于这个艰难程度我一直心存疑惑，如果你稍微搜索一下 PoS 就会发现，这就是一个以质押金多少作为概率随机被选为 Leader 的逻辑，并没有比 PoW 复杂太多。

我找到了以太坊 PoS 主要设计者之一 Vlad Zamfir 的[文章](https://medium.com/@Vlad_Zamfir/the-history-of-casper-chapter-4-3855638b5f0e)。在文章中 Vlad 详细介绍了自己从加入以太坊团队开始负责设计 PoS 共识协议到最后选择使用 Casper 的整个过程，虽然当前以太坊的 PoS 并没有使用 Vlad 提出的方案，但要设计的基本思路和面对的解决问题是一样的，最终「Casper + GHOST」的架构也是相同的。

总的来说，以太坊希望更换共识协议为 PoS，但并不是原生的 PoS，而是希望能将其修改成一个更加现代的 PoS 协议，为的是解决下面两个重要问题。

## Nothing At Stake 问题 -- 小孩子才做选择！

在 PoW 和 PoS 共识协议中，区块链会产生分叉，即从某个节点之后，可能有多个交易执行的历史记录版本，这些版本不同且相互冲突，只能有一个分支被最终接受，例如比特币中最长的链即为合法链。矿工在打包出块时，一般只选择一个分支继续向后出块，具体就是将该分支最后一个块的哈希值包含在自己的块中并进行计算（俗称挖矿），这是为了避免算力分散。

但 PoS 却不同，由于 PoS 的矿工不用耗费大量算力计算一个哈希值，所以负责出块的节点可以选择同时在多个分支之后出块，实际上这也是利益最大化的方法，无论日后哪个分支胜出，自己都可以获得奖励。

如下图，B3 块之后会出现两条分支，B4 ← B5 和 B4’ ← B5’，此时新的矿工打包块 B6 时，PoW 选择其中一个分支作为父块并沿着它继续挖矿避免分散算力，而 PoS 选择两个更能保证收益。

![image-1](https://xufeisofly.github.io/picx-images-hosting/ethereum1/image-1.41y7okxswa.png)

这是因为在 PoS 系统中，矿工是通过锁定他们的代币参与共识的，而不是通过消耗计算资源。这意味着即使矿工在所有分叉上都进行签名，也不会多消耗什么资源。因此，如果多个分叉出现，矿工可能会在所有分叉上签名，以确保他们利益最大。但这就会导致矿工会沿着所有的分支出块，从而使交易的执行无法收敛到一个版本中。这就是原生 PoS 面临的一个重要问题，称为 Nothing at Stake。

无论是 PoW 还是 PoS，在出现分叉时能够根据某个规则选择其中一个分支继续出块是十分重要的，这就是上文中的「分叉选择规则」。

对于 Nothing at Stake，Vitalik 在 2014 年 1 月份曾提出过用「惩罚」作为解决方案的核心思想，矿工需要押注某一个分支，如果该分支最终胜出则会获得奖励，如果失败则会丧失押金，Vitalik 称之为「[赌注共识](https://blog.ethereum.org/2015/12/28/understanding-serenity-part-2-casper)」。这与 PoW 很像，PoW 每一次链的选择其实也是赌注，不同长度的链拥有不同的赔率，当一条链越来越长时赔率也会变得越来越大，最终所有人都会押注同一条链，分叉的最终确定就完成了。实际上 Vitalik 是希望这样使用「赌注共识」来实现分支的最终确定性，但发现了一些风险之后转移到了 BFT 方案上。不过这种质押一定金额并在出现恶意行为时进行处罚的方式被最终采纳，用以解决 PoS 的 Nothing at Stake 问题。

## Long Range Attack 攻击 -- 你的私钥卖不卖？

Long Range Attack 是一种主要针对 PoS 的攻击方法。在这种攻击中，攻击者试图通过从区块链某个早期区块开始分叉，逐步构建一条伪造的、替代的链，最终说服其他节点以取代主链。对于 PoS  来说，构建这种攻击链需要攻击者拿到之前矿工的私钥，而这其实很容易，比如：

- 之前的矿工已经不玩了，退出了网络并卖掉了自己的 ETH。
- 攻击者可以贿赂之前的出块矿工，让他们事先转移自己的 ETH，然后拿这个空账户的私钥进行攻击，这称为贿赂攻击（Bribing Attack）。

贿赂攻击对于没有质押金的 PoS 十分致命，但其实即使有质押金，攻击者也可以在该矿工已经取出质押金之后拿到他的私钥，并构建一条攻击链，如下图。

![image-2](https://xufeisofly.github.io/picx-images-hosting/ethereum1/image-2.8z6oifbkp6.png)

图中，攻击者从很早的块开始分叉，逐步构建出一条比当前主链还长的链，然后就可以试图说服其他验证者采纳这条新链了（具体取决于分叉选择规则），这样主链上的交易都会被回滚掉。也就是说，只要你能拿到这些出块节点的私钥，就可以随意创造区块链的历史。

虽然 Long Range Attack 主要针对 PoS 进行攻击，但在 PoW 中也是存在的，只是因为计算哈希值需要耗费大量的资源，单单攻击者的算力并不足以产生一条最长合法链，所以被 Long Range Attack 回滚交易的可能性很小。换句话说，PoW 的大量算力成本和使用最长链的分叉选择规则使得 Long Range Attack 基本无法实现。而 PoS 就不同了，只要得到了出块矿工的私钥，就可以很容易的生成一个合法的区块。一个原因是由于伪造旧区块的成本很小，另一个原因是因为这些区块并没有被最终确定，随时可以被回滚，也就是缺少上文我们所说的「最终确定性」。

Vlad 曾提出过一个简单的解决方案用于解决 Long Range Attack，就是认为节点提交了质押金之后它的签名才是有效的，这其实有一定道理，「你已经把质押金取走了，我凭什么还相信你？」。这就避免了节点取出质押金之后 Long Range Attack 无法进行惩罚的问题。然而这也意味着，一个区块的合法性不是通过历史区块就能验证的，还需要当前的数据，即有质押的节点列表。这就使得一个区块的合法性和时间产生了关系，一个现在合法的区块过几天就有可能变得不合法。这个设想的提出让 PoS 与 PoW 从根本上产生了区别：**区块的验证依赖了节点当下的状态。**Vitalik 一开始不是很喜欢这个方案，他还是希望仅凭历史区块就能完成对一个区块的验证，但最后还是接受了，并提出了 weak subjectivity scoring rule，我争取在后面文章进行介绍。

虽然已经解决了 Nothing at Stake 和 Long Range Attack 问题，Vlad 发现以太坊 PoS 仍然有很多问题没有解决，比如他发现目前的分叉选择规则是依赖于一个强同步的网络的。

> *我们不知道区块将如何被创建。我们不清楚激励机制的具体细节，只知道「双重签名」会导致你失去押金。我们没有明确我们的协议与网络同步假设之间的关系（我们需要可预测的网络延迟）。我们依赖于客户端最终会选择同一个链条，因为它们遵循相同的分叉选择规则，但没有完全意识到这种机制依赖于强同步性假设。 —— Vlad*
> 

这句话可能不好理解，我们的网络模型分为同步、异步和部分同步三种模型（具体可以参考 Tendermint 那篇文章）。而以太坊的分叉选择规则是基于同步模型的，这意味着它必须依赖如下假设：一条消息在一个固定的、可预测的时间内一定可以收到。由于现实生活中的网络情况很难满足这个假设，因此共识协议的可靠性会被降低。

举个例子，如果一个恶意矿工在两个分支上同时出块并签名，正常情况下会验证者被发现并没收它的质押金，但如果网络环境不是强同步的，有些节点没有收到两个块，或者比如下一个块产出之前上一个块还没有收到，那么该恶意行为就可能不会被发现并惩罚。换个角度说，假如 PoS 规定 15s 出一个块，如果我们把这个时间大幅缩小，就很难保证 PoS 的安全性了，这是因为我们必须有足够的时间保证系统节点收到消息。显然，将协议的可靠性依赖于一个强同步网络假设有较大风险。因此 Vlad 转向了研究传统的 BFT 共识，具体来说就是 Tendermint。

# 恶意节点要少于 1/3，凭什么

---

相比中本聪共识，BFT 类共识是一类传统的共识协议，主要用于解决分布式系统下共识的安全性问题。Tendermint 是其中一个，它建立在一个部分同步的网络假设上。在研究了 Tendermint 之后，Vlad 发现 BFT 类共识通过收集 2/3 投票可以即时的确定一个块的合法性，由于区块当场就被「最终确定」，这样就不存在 Long Range Attack 问题。另外 Tendermint 本身不会分叉，因此也不存在 Nothing at Stake 问题。Tendermint 本身还提供了 PoS 方案，可以利用 Stake 作为权重进行投票。

一切听起来十分美好，然而 BFT 共识必须满足一个前提，即拜占庭假设，它要求全部节点中恶意节点的数量不能超过 1/3，正是这个假设让 Vlad 和 Vitalik 等人无法接受。

## Cartel Censorship Attack

Cartel 这个词代表一个恶意的小团体，Censorship Attack 被翻译成了「审查攻击」（有点不知所云），有人说 Censorship 是打马赛克的意思，总之区块链中的 Cartel Censorship Attack 可以理解为一个或多个参与区块链网络的节点联合起来，通过共谋控制网络的共识过程，从而干扰特定交易或区块。

一个共识算法能容忍多少节点联合起来进行攻击，就代表了它对 Censorship Attack 的抵御程度。比如 PoW 和 PoS，一个 51% 的 Cartel 可以对剩余节点进行攻击，而 BFT 中只需要 34% 的 Cartel 就可以阻止共识的达成，或者进行分叉攻击造成冲突区块的提交，比如诚实节点中有一半支持 A 区块，一半支持 B 区块，两个区块冲突，此时 Cartel 可以给两边发送不同的投票，导致 A 和 B 同时得到 +2/3 的投票被提交。因此 Vlad 认为，**这种对于恶意节点数量有很强假设的共识协议不适合现有的公链**，因为现有的公链中大部分权力（无论是 Work 还是 Stake）都被小部分 Cartel 所控制。

这意味着当 1/3 节点联合成为 Cartel 并尝试干扰共识时，整个共识协议将会卡死，丧失活性。多说一句，我在 HotStuff 那片[文章](https://xufeisofly.github.io/blog/shardora-hotstuff)中提到的 BFT 共识相比中本聪共识会降低安全性是不正确的，实际上安全性是提高的但是活性会降低，在此更正。

除此之外，BFT 共识也牺牲了去中心化程度（毕竟安全性和可扩展性都很高，根据不可能三角只能是牺牲去中心化了）。Matthew Wampler-Doty 在 2015 年给出了一个关于去中心化的定义：**当一个协议哪怕只剩下一个节点也能恢复过来时，这个协议就是去中心化的。**显然 BFT 由于必须满足恶意节点个数小于 1/3 而不能满足这个条件。

> A protocol is decentralized only if it can fully recover from the permanent removal of all but one of its nodes.

最后以太坊团队提出了 Gasper 协议，其实是将 Casper FFG 协议套用在改进后的 GHOST 协议上（LMD GHOST），前者负责在特定检查点最终确定一个区块（最终确定性），后者负责进行分叉选择（分叉选择规则）。

# 以太坊 PoS 的舍与得

---

## CAP 理论

分布式系统中有一些神奇的理论，最有名的可能是 FLP 和 CAP 理论了（FLP 理论是一个不可能理论，指在一个异步网络环境下，如果节点发生故障，则系统永远不可能达成共识）。

CAP 理论介绍了一个分布式系统存在三个性质，即

- Consistency（一致性）
- Availability（可用性）
- Partition Tolerance（分区容错性）

CAP 理论指出，当网络异常导致节点发生分区（Partition Tolerance）时，不可能同时满足 Consistency 和 Availability，只能选择一个。这不难理解，借用[这篇文章](https://eth2book.info/capella/part2/consensus/preliminaries/)中的一个例子，如下图。

![image-3](https://xufeisofly.github.io/picx-images-hosting/ethereum1/image-3.92qag54nez.png)

图中网络发生了故障，节点分成了 A 和 B 两个部分，单个部分之内的节点可以正常通讯，但 A 和 B 之间发送的消息无法彼此触达。此时，如果客户端同时给 A 和 B 发送一条请求消息，只会有两种情况。

1. A 和 B 都接受并处理了这条消息，但由于分区，所以 A 和 B 处理之后的结果可能不同。（丧失 Consistency）
2. A 和 B 都拒绝处理这条消息。（丧失 Availability）

区块链系统本质上就是一个分布式系统，所以也满足 CAP 理论，具体体现在共识协议的安全性和活性的取舍上。

## 共识协议的 Safety 和 Liveness

Safety（安全性）和 Liveness（活性）是共识协议经常关注的两个指标，抽象的说，Safety 代表「绝对不会发生不正确的事情」，Liveness 代表「正确的事情一定会发生」。在区块链系统中，Safety 的一个表现就是 CAP 理论中的 Consistency，而 Liveness 的表现就是 Availability，因此无论是什么共识算法一定会在这两个性质上面有所取舍。

传统的 BFT 共识算法如 PBFT，Tendermint 或者 HotStuff，相比中本聪共识的最大特点就是**每一个视图都会被确认，且这个确认即使生效，不可回滚。**这是因为 BFT 中每一个块都需要 2/3 的节点投票认可，这就使得每个块都立刻具备了最终确定性。换句话说，这类协议十分的安全（拥有 Safety），然而问题是如果超过 1/3 节点宕机，整个系统会变得不可用（牺牲 Liveness），或者超过 1/3 节点是恶意节点，也会影响已提交区块的安全性（分叉攻击）。因此 BFT 共识的 Safety 和 Liveness 都是基于满足拜占庭假设这个前提的。

原生的 PoW 和 PoS 之类的中本聪共识无需满足拜占庭假设，无论恶意节点有多少都可以持续出块（拥有 Liveness），但却没有最终确定性导致区块有可能被回滚（如遭遇 Long Range Attack 或者 51% Attack），因此并不安全（牺牲 Safety）。

以太坊选择的方案是将两者结合，当满足拜占庭假设时可以即时确定一个区块保证 Safety，但当 1/3 节点宕机时也能正常出块保证系统可用。此处引用 Vitalik 的叙述。

> *我们希望尽可能多地产生共识：如果诚实节点超过 2/3，我们就能获得常规共识，但如果少于 2/3，那么区块链仍然可以继续增长。虽然新块的安全性会暂时降低，但也强过停滞不前、什么都不提供。如果某个应用程序对这种较低安全的区块很不满意，它可以自由地忽略这些块，直到它们被最终确定为止。*
> 

# 总结

---

为了解决 Nothing at Stake 和 Long Range Attack 问题，同时避免 BFT 类共识容易丧失系统活性的问题，以太坊使用改进的 GHOST 协议作为分叉选择规则，同时引入 BFT 共识协议 Casper FFG 用于最终确定区块，在有限保证活性的情况下尽量具备安全性。我将在下一篇文章对 LMD GHOST 协议进行介绍。

# 参考资料

---

- [https://learnblockchain.cn/article/4778](https://learnblockchain.cn/article/4778)
- [https://arxiv.org/pdf/2003.03052](https://arxiv.org/pdf/2003.03052)
- [https://www.ethereum.cn/casper-ffg-explainer](https://www.ethereum.cn/casper-ffg-explainer)
- [https://medium.com/taipei-ethereum-meetup/history-and-state-of-ethereums-casper-research-85e8fba26002](https://medium.com/taipei-ethereum-meetup/history-and-state-of-ethereums-casper-research-85e8fba26002)
- [https://medium.com/taipei-ethereum-meetup/intro-to-casper-ffg-and-eth-2-0-95705e9304d6](https://medium.com/taipei-ethereum-meetup/intro-to-casper-ffg-and-eth-2-0-95705e9304d6)
- [https://medium.com/unitychain/intro-to-casper-ffg-9ed944d98b2d](https://medium.com/unitychain/intro-to-casper-ffg-9ed944d98b2d)
- [https://notes.ethereum.org/@vbuterin/serenity_design_rationale?type=view](https://notes.ethereum.org/@vbuterin/serenity_design_rationale?type=view)
- [https://blog.ethereum.org/2015/12/28/understanding-serenity-part-2-casper](https://blog.ethereum.org/2015/12/28/understanding-serenity-part-2-casper)
- [https://blog.ethereum.org/2015/12/28/understanding-serenity-part-2-casper](https://medium.com/@Vlad_Zamfir/the-history-of-casper-part-1-59233819c9a9)
- [https://medium.com/@Vlad_Zamfir/the-history-of-casper-chapter-5-8652959cef58](https://medium.com/@Vlad_Zamfir/the-history-of-casper-chapter-5-8652959cef58)
- [https://blog.ethereum.org/2014/11/25/proof-stake-learned-love-weak-subjectivity](https://blog.ethereum.org/2014/11/25/proof-stake-learned-love-weak-subjectivity)
- [https://eth2book.info/capella/part2/consensus/preliminaries/](https://eth2book.info/capella/part2/consensus/preliminaries/)
- [https://blog.ethereum.org/2016/05/09/on-settlement-finality](https://blog.ethereum.org/2016/05/09/on-settlement-finality)
