---
layout:     post
title:      "f4:facebook的温数据存储系统paper review"
subtitle:   ""
date:       2019-12-24 22:00:00
author:     "cheney"
header-img: "img/mountain.jpg"
catalog: true
tags:
    - Distributed System
    - facebook
    - f4
    - storage system
---

受到 the morning paper 作者最近的一封邮件影响，觉得定期分享一些读论文的感受应该是很有帮助的，也有助于有兴趣的朋友一起讨论，所以决定尝试着写一写，看看能不能坚持下去。

第一篇论文想讲讲facebook的f4，这是一篇发表在OSDI'14上的论文，迄今也已经过去5年了，所以有很多概念现如今大家已经都非常熟悉了，这篇文章还是有必要读一读，因为facebook从需求出发直到系统落地的方法论很值得学习，当然它的架构演变的过程也很有趣。

## 引子

facebook的存储基础设施的演进是非常典型的业务驱动，早期他们使用商用NAS产品来存储数据，很快随着规模的上涨，这一方案带来的经济/人力性价比越来越低，终于facebook决定自己搞一套存储系统来应对日益增长的业务对存储系统的要求，这就是haystack，这个系统的论文发布在OSDI'2010上。

haystack的设计原理还是比较直观的，之前我负责的一套存储系统也是依据此模型进行构建的，包括淘宝的TFS也是类似的思想。haystack作为一款热数据存储引擎，功能的重点是以低延迟抗住读写请求，并提供较高的系统吞吐。而facebook公司业务决定了他们会有大量的小文件(图片、头像等)，基于本地文件系统实现的存储模型，在遇到大量的文件元数据时面临的寻址开销非常大，一次普通的IO由于inode查询等磁盘开销可能会被放大多倍，导致读取性能低下。在删除带来的磁盘碎片更会加剧对性能的影响，为此，haystack的主要思路是：

- 通过将大量业务层的小文件(blob)聚合为一个大文件(volume)存放在文件系统中，降低文件系统元数据的量级。而这些volume的量级非常大(~100GB)，可以很好的利用磁盘的追加写特性，并且在回收时降低磁盘碎片
- 通过将小文件的索引(记录每个小文件在volume中的偏移等信息)全量加载进内存中加速索引查询

这样一优化下来，每个blob的读取最多会涉及一次磁盘IO，很好得解决了热数据的性能问题。这套系统也运行得很好，在f4出来之前已经在线上运行了大约7年。

后来facebook的同志就发现了问题，数据的存储成本太高了，因为facebook的业务具有很强的时间效应，随着时间推移大量的数据访问频度会降低(温数据)，而这些温数据具有长尾效应，在整个系统中是越来越多，换句话说，haystack作为一个热数据系统，其中大多数数据其实访问的频度很低，这属于典型的占着茅坑不拉屎啊。还有一个问题是，haystack作为一个抗热数据流量的系统，它必须是多副本的，因为多副本才可以进行流量分摊，但是作为温数据显然不需要这个特性，但它们依然享受着多副本的待遇，这可不就是素位尸餐。

于是facebook的同志们不答应了，很显然，这里的blob随着时间区分出了两种访问模型，它们是如此的不同以至于应该要分而治之(计算机领域最伟大的思想又发挥作用了)。

## 温热分离

我觉得facebook包括之前看到的很多国外公司发的paper都有一个很好的习惯，必须用数据做验证。当然这也是有成熟业务的公司的优势，他们有一个可供他们研究的数据集。facebook也一样，既然觉得数据存在温热的区别，那必须验证一下，通过对现有集群的benchmark和数据收集，证据是很明确的，线上大多数业务的数据都具有热度随时间消退的特性:

![image](https://note.youdao.com/yws/api/personal/file/WEBfe0791f444a8cee6f8a8de7fffb13c85?method=download&shareKey=1594df9aec35d0deb042079996d1283b)

第二个问题随之而来，既然数据有温热的分界，那么分界在哪里呢？就是说要找到一个时间点，在这个时间点之后，数据会大概率趋于温数据。我没太看懂这里的数据分析逻辑(如果有看到本文的同学欢迎告诉我下，感谢)，但总之facebook的兄弟们找到了这个时间--三个月，三个月后除了头像数据以外，大部分数据都成了非高频访问。

好的，现在有充分的证据支持要做一套新的系统来存放温数据了，那么，这套系统需要满足什么条件呢？至少有两个条件是不能商量，必须满足的：

- 省钱，这套系统必须省钱，为一些长尾的不常访问的数据没必要让他们具有在线访问能力。当然了也不能像低频存储那样以天为单位访问，但可以容忍比在线访问稍慢一些
- 数据冗余不能降低，我们知道之前的haystack的副本冗余是3.6并且具备跨地域容灾能力(有一个从副本位于独立地域，每块磁盘做RAID6提供1.2的冗余度)

第一点还是比较容易满足的，EC,EC还是EC。这似乎已经成了标配，不知道在2019年的今天，facebook这套系统有没有做新的改进，但在这篇论文中他们采用了经典的 Reed-Solomon coding(10,4)，这意味着副本冗余度为1.4。当时f4出来的时候，微软的WAS(windows azure storage)采用的LRC方式也已经出现了，不过facebook还是用了比较经典的方式，毕竟，稳才是第一位的。这个1.4的冗余度是在单地域实现的，如果要实现跨地域数据容灾，就需要做一个对等拷贝，那整体就是2.8的冗余度，对比之前的3.6收效不是非常大。不过f4引入了XOR coding的方式将整体冗余度降至2.1。

## 统一的blob存储

facebook的存储设施演进非常清晰，明确的需求引领系统的改变。在引入了f4之后整个blob存储系统演化为下图：

![image](https://d3i71xaburhd42.cloudfront.net/4b1b767ea79f350ee52bf347406e9b1ee1d4c68f/4-Figure6-1.png)

这是一套温热共存的系统，并且由于router tier的存在，存储内部从haystack过度到haystack+f4的架构对外层是完全透明的。所有的创建会进入到haystack，并且由它抗住大部分的读操作和删除操作。当某个volume达到温热分界时间点(实际上并不是所有volume内的blob都达到该条件的)之后，该volume从unlock变为locked状态，不再允许执行写入，并将volume移动至f4。

关于『移动』这个操作实际上并没有看到更多细节，但从高效的角度来说，我还是觉得应该不至于将这个volume拷贝至f4，然后f4再进行格式转换至EC的方式，可能这里存在一个中转网关，f4会到haystack系统进行读取然后直接转换。也可能是在haystack集群上可以访问f4的HDFS集群，到期后直接将某个volume写入到HDFS完成数据的传输。

### f4内部

volume在f4内部如何存储，说起来也比较直观。一个volume会被顺序进行切分，每n个block(通常为1GB)为一组，这n个block根据EC算法生成k个parity块，于是这`n+k`个block组成了一个strip。由于论文说f4是基于facebook内部的HDFS进行构建的，我有理由相信这些block就是直接存储于HDFS中(但是副本数应该是1)。

![image](https://d3i71xaburhd42.cloudfront.net/4b1b767ea79f350ee52bf347406e9b1ee1d4c68f/6-Figure7-1.png)

其次，每个volume依然有一个index文件，我甚至认为该index文件是和haystack中的index文件内容一样的，因为并不需要做转换。那么要从f4中读取一个blob的流程是什么样的呢？首先要知道f4中各功能模块的组成部分，处在关键IO链路上的其实只有 storage node：

![image](https://d3i71xaburhd42.cloudfront.net/4b1b767ea79f350ee52bf347406e9b1ee1d4c68f/7-Figure8-1.png)

storage node对外暴露两组接口，一组是查询某个blob的索引信息的Index API,该接口可以查询到blob在哪个物理block中以及相应的大小和crc等。另一组是File API，负责读取相应数据。其中每个volume都有一个storage node负责，这个负责的关系存储在一个外部系统中:

> The volume-to-storage-node assignment is maintained by a separate system that is out of the scope of this paper.

但总的来说router tier应该能从请求参数中得到volume id并将请求转给负责该volume id的storage node，该storage node(primary node)具有此volume对应的index文件的内存映像(于启动时加载，我认为该信息是存储在HDFS上的并且index文件被哪个storage node处理是弱依赖关系，可能由某个外部系统协调)。primary node得出该blob的偏移后还需要知道这个偏移对应的block信息，这是有location-map记录的，location-map记录了volume->strip->block(logical)的信息，并且通过HDFS的name node得知该logical block的实际物理存储节点，这些信息通过Index API返回给router tier。

router tie收到回应后会向实际数据所在的storage node调用File API获取实际数据。在遇到一些故障的情况下，router tier可能需要依靠backoff node来进行数据重建才能满足前端要求。值得注意的是，在backoff node重建blob信息时是以blob为单位而不是block为单位的，这可以大大降低故障情况下数据构建的开销。而整个blok的丢失重建则是由rebuilder node负责的。

### 跨地域容灾

按照之前所说，f4的跨地域数据冗余度是2.1，它是通过对两个互不相同的地域的block进行XOR操作，并将结果存储与第三个地域来实现的。这个操作也是比较直观的，但有一点值得注意，即XOR地域的block不仅存储block数据，还会将block对应的volume的index文件也进行存储，因为我们知道，index文件本身只在地域内做了三副本，如果地域失效就无法进行索引查询了。

> These XOR blocks are stored with normal triple-replicated index files for the volumes.

![image](https://d3i71xaburhd42.cloudfront.net/4b1b767ea79f350ee52bf347406e9b1ee1d4c68f/8-Figure9-1.png)

## 总结

f4是一个非常朴素的系统，但是它是构建在坚实的业务分析基础和合理的前期架构组合的基础上的，所以能够在19个月的时间里完成上线并承载60PB+的数据，这是一个值得尊敬的工程实践。它通过对业务访问形态的分析出发，严格论证了猜想然后实现系统，并且最大程度保持与既有系统的融合，降低工程复杂度。通过XOR的创新将存储冗余度降至2.1但依然保持跨地域数据安全，并且降低了长尾温数据对haystack系统的影响，有效降低运营成本，值得学习。
