---
title: 关于区块链这几天我想到的
layout: post
post-image: "https://xufeisofly.github.io/picx-images-hosting/about-blockchain/head.1lbz9na4f6.jpg"
tags:
- blockchain
---

# 实在不是一个好名字
---

区块链（BlockChain）这个词在中本聪的论文中本来是分开使用的（Block 和 Chain），不知怎么后来被合起来传播。这个名字实在是不好，太 technical 了，无论是「区块」还是「链表」都是软件领域的专业词汇，90% 的人乍一听都不知所云。看看 AI，「人工智能」这个名字简单形象，加上无数科幻小说和电影早已聊透，闭上眼睛就能想象出来是什么东西。再如「操作系统」，虽然偏技术感但也是容易理解的，至少人们知道这是个系统，可以在上面进行一些操作。而「区块链」在我脑袋里永远是一堆被绳子连起来的方块，除此之外没有别的信息，所以说起名字是软件最大的难题不是玩笑，我觉得不如把区块链改成「人造信任系统」（Artificial Trust System）什么的。

另外还有 Web3 这个名字，让人觉得它是 Web2 的一次进化，其实我认为所谓的 Web3 相对于 Web2 来说不是一次进化，只是一种选择，你可以选择用 Web2 的中心化的方案搭建应用，也可以使用 Web3 的去中心化的方案，是各有利弊的。

# 区块链技术十分有趣
---

简单的描述一下区块链做的事情。

当你打开微博并发出一条状态，或是你在某宝下单并完成支付，你其实是向这些 App 背后的公司发送了请求，公司的服务器集中存放并修改你的数据。再比如，当你通过银行卡转账 10 块钱给 Bob，你其实是向银行发送了一条转账请求，银行会从你的账户上扣钱，给 Bob 的账户加钱，这样一次转账就完成了。这是目前大部分应用程序的逻辑，都是中心化的方案，因为它们都依赖于一个中心机构对用户数据的操作和存储。而我们使用这些方案的前提是对这个机构有足够的信任。

信任不是白给的，一些中心机构在利益的驱使下会变得不值得信任，它们有可能擅自修改我的数据，或者多扣我的钱，或者直接跑路。当然如果发生这种情况我会投诉，寻求另一个更高级的、值得信任的机构的支持。但是生活中就是有些场景下不存在这种更高级的机构（比如跨地域或跨国业务），或是这个机构不能帮我们解决这个问题，或者解决了但仍然会给我们造成损失，因此我们希望寻找一种机制可以天然避免中心化机构的恶意行为，在没有信任的前提下仍然能够享受一些服务，这就是区块链可以做到的。当然，这些应用场景并不是区块链设计的初衷，区块链是被比特币带火的，它一开始更专注于金融领域，不过以太坊发布以后区块链变成了一台世界计算机，上面可以自由编写程序，让区块链有了将一切事物去中心化的能力。

与中心化方案不同，区块链不是仅有少量的被某个机构拥有的服务器用于管理数据，而是成千上万台服务器。这些服务器（称为节点）连同其所有权一起分散在世界各地，属于不同的人或组织，这些人互不相识，但他们的节点都保存了一份相同的数据副本。理想情况下，你可以在自己的电脑或手机上运行一个节点，保存这些用户数据的一个副本。还是用转账举例，当你给 Bob 转账的时候，不是像银行一样单单修改自己的数据库就可以的，而是需要全世界**所有的节点同时**修改自己的那份数据，当所有节点的数据都成功被修改时，才认为这个交易真正被完成。（早期区块链也被称为分布式账本，现在看来账本这个词太局限于金融领域了。）

![Untitled](https://xufeisofly.github.io/picx-images-hosting/about-blockchain/Untitled.4jo9d5idwk.png)

区块链上每一份数据的修改请求都需要所有节点达成共识，实现它的技术称为共识算法，这是区块链中的核心部分。区块链共识算法与一些互联网大厂中的数据容灾或异地多活所用的一致性方案是不同的。首先区块链的节点数量是巨大的（目前比特币节点数超过 7 万个，以太坊约7千个），其次节点可能是有恶意的，比如某个恶意节点可以大声告诉其他节点说「这比钱不是转给 Bob 的，而是转给我的」。因此如何能在恶意节点干扰下让如此多的节点针对某一笔交易达成共识，就是共识算法要解决的问题。

*听说谷歌的区块链团队中有的工程师并不相信区块链的价值但仍然愿意参与开发，只是因为觉得这里涉及的技术足够有趣。我不会介绍具体技术，而是希望能尽量简单地进行描述，在研究之后我发现这简直就是人类社会运转机制的缩影，很有意思。*

如何让所有节点共同执行一条命令？一种方案是比特币所使用的，我个人称之为「独裁者」方案。它的基本逻辑是要从所有节点中选出来一个 King，这个 King 拥有极大的权利，可以命令其他节点无条件执行它发布的任何命令（前提是合法命令）。当然，所有节点都想当 King，因此大家约定，一起算一道很难的数学题，谁先算出这道题的答案谁就可以成为 King。King 可以打包想要执行的提案发送给其他节点，并附带自己算出的答案证明自己是这次的 King，其他节点收到 King 发来的提案就会统一无条件地执行。这种方法称为中本聪共识，包括比特币的 PoW，以太坊的 PoS 等虽然不一定都会花费时间计算，但都需要一个「独裁者」的出现，本质都是这种方案。

第二种方案我称为「民主」方案，它是基于投票的，和「独裁者」方案不同的是，所有节点都可以轮流发起一个提案，但其他节点收到提案之后需要判断自己是否认可这个提案，然后投票同意或者反对，当大多数节点都认可该提案时，这个提案才会被所有节点一同执行。这种方法称为拜占庭容错共识。

是否很像人类社会的基本运转模式？更有意思的是，「民主」投票的方案是有一些前提的，首先就是恶意节点的数量不能超过某个阈值（不超过所有节点的 1/3，这称为拜占庭假设），否则系统会被恶意节点控制而永远达不成共识，或持续通过一些坏的提案。其次「民主」方案中诚实节点也需要更加的「聪明」，来承担验证提案真伪的职责，确保自己不会被发起提案的节点所欺骗。最后，参与投票的节点的数量也有要求，太少的话会不安全（容易被恶意节点控制），太多的话效率低下，因此往往是会选出一个数量合适委员会 Commitee 来承担投票的职责，为保证委员会不会变坏，还需要定期轮换，仔细想想就十分奇妙。

# 拿着锤子找钉子
---

在我接触区块链之前更多的是好奇，为什么支持者和反对者的观点有那么大的差异。

区块链从真正进入大众视野到现在也十几年了，与其说给世界带来了什么变化，更多的还是质疑和恐惧，质疑这个技术实际价值配不上热度，恐惧从中衍生出来的那些金融骗局。在很多人心里「炒币」这个词基本是和「区块链」划等号的，结果就是大部分人敬而远之，偶尔会因比特币价格暴涨或暴跌跟着关注并感慨一下。近些年随着 AI 的崛起区块链逐渐脱离大众和资本的视野，陷入低谷。

这怪不得别人，区块链声称要干的事情宏大而难理解，它并不像工业革命的几次技术突破，包括如今的 AI，带来的更多是人类工作效率上的飞跃。而区块链大部分场景下是会降低效率的，最有名的就是比特币每秒钟只能处理 7 笔交易这件事，效率还低意味着成本变高，所以不要拿比特币买咖啡喝，确实对不起那笔手续费。

两年前的我还没有参与区块链项目，更多的都是自学产生的简单认知，对区块链是嗤之以鼻的态度，原因有两个。一是我不知道它能用来干什么，很多去中心化应用就是把现有的 App 重做了一遍，再冠以去中心化的噱头，也不看看人们是否真的关心。也有一些项目在探索新的领域，但那苦苦尝试的尽头，让我一度怀疑这是否是拿着锤子找钉子。现在我确信了，这就是拿着锤子找钉子，不过计算机刚发明出来也只是为了军事用途后来才逐渐进入家庭，复印机刚出现时也被认为根本卖不出去，因为没有需求。技术进步和需求创造想来都是交替前进的，有了锤子再发明钉子好像也能说的过去，更何况区块链已经有了一个很成功的应用，就是加密货币，只不过与广大民众的生活关系没那么大而已。

二是我始终认为去中心化是反人性的。人类几千年建立的中心化社会结构，运转高效，坚固可靠，更别说牵扯众多利益，不是说去掉就能去掉的。现在看来，区块链过早的被吹上神坛赋予了「颠覆世界」的使命，导致业外人士不明所以却又对它有很高的期待，所以当发现这么多年后关于区块链依然是炒币，挖矿之类的叙事时人们失望透顶。至于业内人士，有句话说「人有了信仰就如同获得了解放」，只有相信了你在做的工作才有意义，为了这份情绪价值大家都在忙着解放自己。我解放的并不彻底，我相信区块链一定能改变世界，但至少是个漫长的过程。

# 没有那么多场景需要去中心化
---

2018 年 Joe Rogan 对 Elon Musk 采访时谈到了人们隐私数据的保护问题，Musk 当场反问道，「人们真的需要保护隐私吗？他们明明把所有东西都放到网上。」这句话虽然有点绝对，但人们关心更多的还是当下的体验，比如页面加载快不快、内容是否优质、这个功能是否好用等等，至于隐私安全，至少我更倾向于掩耳盗铃，没发生什么大事就好。公司收集用户隐私数据，大部分情况也会反映到体验的提升上，当然我不否认数据售卖这种龌龊的事情，但真的有很多人关心这些吗？总体来看这是一个有利有弊，不能一棒打死的事情，这在区块链领域也是一样的。

虽然区块链能让你的数据不被篡改，能赋予用户更多的权利，但不得不承认目前这些方案使用的也没有任何问题，并且因为背后有一个机构在运营使得效率容易做得更高，不仅仅是程序运行的效率，也包括社区治理和开发迭代的效率，这些终归会反映到用户体验之上。此外，中心化模式为公司带来的数据控制权也为其带来了额外的利润，激励公司去做更多的优化以提升用户的体验，而毫无疑问大部分用户心目中体验的权重是很高的。

这并不是说去中心化应用就无法给用户带来相同的体验，毕竟区块链基建技术日新月异。只是从理论上看同样的体验提升所付出的成本无疑是区块链更高一些。举个例子，Warpcast 是一个去中心化的类似 Twitter 的社交应用，我用了一段时间，实话说功能上没有太大差别。然而如果你想要发贴，Warpcast 是需要收费的，这是因为数据在区块链上，需要给矿工付钱。当然随着区块链可扩展性相关技术的快速进步，比如 Layer2，这个成本会大幅下降，但理论上也是不可能低于中心化应用的，毕竟有几千台机器都在费电保存你的数据，帮你验证每一条命令。此外，在体验和功能上，Warpcast 能做到的，Twitter 也都能做的更好更快，那么普通用户为什么要付出更多的成本迁移过去呢。当然由于部署在区块链上，它的好处就是不会限制你的言论，没有一个权利中心可以随意的对用户禁言或者删帖，然而也不难想到这是一把双刃剑，而且现实是单凭这个优势只能吸引一小部分极其看中这个特性的用户。

# 不是去中心化，而是要解决无中心的问题
---

大部分生活类场景实在没有必要去做去中心化的尝试，区块链从业者不要天天想着颠覆世界，把那些已经很好的应用都换一遍是个只有噱头没有价值的事情。与其想尽办法去中心化，不妨想想如何去解决一些目前没有中心的问题。

很多场景没有办法拥有一个中心机构，比如跨国贸易，比如如何让科学数据跨国界的流通。这个时候区块链本身就可以作为一个权威机构去进行管理，为不同国家地区之间提供信任机制。因为在这些领域传统的方式本身没有成本优势，这也是为什么金融领域目前还是区块链最主要的应用场景。此外我也听过一些别的想法，比如一些创作者想上架自己的内容到境外的平台上来赚钱，只能找一些中间商操盘，但中间商可能会恶意隐瞒数据，现在有了区块链提供保障就可以解决这个问题。这种利用区块链去中间商的需求听起来顺理成章，但成本优势其实没有特别明显，换句话说，总有靠谱的中间商吧。

# 普及需要区块链提高性能以及人们变得富有
---

中心化社群中只要你有足够的资源就可以作恶，而去中心化使得作恶成本急剧升高，营造了一个人人平等的生态，自己做自己数据的主人。但如同上文中所说的，能否普及取决于大家心目中对此需求的重视程度以及需要付出的成本。

文明大概率是会进步的，很难想象 200 年前的人会如此看重数据隐私，或是言论自由，这得益于生产力的发展，财富的积累让人们变得挑剔，变得更重视长远利益，更害怕小概率风险。而区块链可扩展性的技术突破也会大大降低其使用成本，让大规模普及变得可能，实现真正的 Web3 网络。在此之前，我认为它更适用于一些小众、专业的领域，比如金融、学术，特别是涉及跨国的业务。这些领域对安全和信誉要求很高，对作恶带来的风险是低容忍的。

总之，区块链就是一个保险箱，如果你把它当抽屉用你会很失望，我们也得承认不是人人都需要一个保险箱，虽然在特殊情况下它真的十分有用。
