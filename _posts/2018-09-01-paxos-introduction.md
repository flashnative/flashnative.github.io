---
layout:     post
title:      "分布式一致性协议 Paxos 入门浅析"
subtitle:   ""
date:       2018-09-1 23:00:00
author:     "cheney"
header-img: "img/mountain.jpg"
catalog: true
tags:
    - Distributed System
---

我相信对于每一个做分布式系统开发的程序员来说,一致性算法都是没有办法绕过去的.从事分布式系统开发也有一段时间了,萦绕我心头的一个遗憾是,总是未能去理解广为人知的 `Paxos` 协议.从另一方面来说,这影响到一个分布式系统从业者的信心,终于我决定尝试理解下 `Paxos` 协议,并且记录下其中的一些理解.

---

一致性协议其实在工作中用到的不少,我们系统也用到了 `raft`<sup>1</sup>算法,最近在阿里云发表的 `polarfs`<sup>2</sup> 论文中也提到了一种基于 `raft` 的改进,即所谓的 `parallel raft`,业界关于 `raft` 的讨论也是不胜枚举.还有一些其他算法,比如 `zookeeper` 使用的 `zab`<sup>3</sup> 和较少在工业界使用的 `viewstamped replication`<sup>4</sup> 等,如果你去看这些算法,总避免不了会看到有人总结到,他们都是一种 `Paxos` 在特定场景下的衍生版本或者是 `Paxos` 的另一种表现形式(这里需要为 `VR` 正名一下,他和 `Paxos` 是几乎同一时期独立发表的成果,可惜不够那么广为人知).既然这样,那么对于探究下这些算法的祖先,就变得非常重要了.

长期以来,大家对 `Paxos` 是谈虎色变,直至最近我和同事讨论时,这类想法也依然很主流.那么 `Paxos` 到底难在了哪里了?我发现的一个误区是,很多人包括之前的我,认为 `Paxos` 的难点在于难以理解,事实上根据我自己的经验来看,这是错误的,相反, `Paxos` 算法比起 `zab` 要好理解得多,比起 `raft` 则是非常的简洁优雅.真正的难点在于实现 `Paxos` 上,所以大家不要怕,`Paxos` 协议真的是一个难者不会,会者不难的东西,让我们花点精力先理解一下朴素 `Paxos` 的原理.由于 `Paxos` 的变种和优化非常多,后续可能还会有一系列的文章来继续深入理解这个协议.

---

# Paxos 背景介绍

`Paxos` 难懂主要是起源于它的初次发表(`the part-time parliament`<sup>5</sup>)是以一种寓言体发表的,不够简单明了.所以在2001年的时候, `Lamport` 老爷子觉得你们实在太过愚蠢,竟然这么简单的东西都理解不了,怒发了一篇 `paxos made simple`<sup>6</sup>.我也是先看的这篇论文开始的,上来综述的一句话就给了我一巴掌:

> The Paxos algorithm, when presented in plain English, is very simple.

当时我心想,大牛就是大牛啊,这个逼装得可以，大家都说这么难理解的东西竟然说 `very simple`,只能怀疑自己的智力问题了.但是相信我,也相信老爷子,理解 `classic paxos` 确实是非常简单的,从几条引理的推导来看,甚至是唯一的合理结果,逻辑上非常自然.网络上有人非常形象地总结该算法的精髓为:连续利用两次抽屉原理,真的是非常准确.

后面我会来讲述到底如何去理解 `Paxos`.

## 为何需要一致性算法?
那么问题来了,为什么我们需要一致性算法呢?大家都知道,目前知道的对抗单结点的可用性故障,数据可靠性故障唯一的方式就是多副本,不论是以什么方式,什么形式的多副本(主备强一致,`quorum`,`Erasure Coding`),这也是分布式系统的基石.


数据在多节点上进行多副本存放,一个自然的问题就是副本数据如何保证对外具有一致性,一致性协议最终的目的就是要保证各个副本数据对外具有一致性.那么假设我们有三个副本,按照先后顺序往这三个副本写入同一份数据,是否就可以得到我们要的一致性呢?答案肯定是不可以的,因为分布式系统运行的系统模型我们成为称为系统.异步系统意味着网络是不可靠的,可能出现数据包的延迟,乱序,重复等等,但是一个到达的数据包是不会损坏的(`non-Byzantine Fault`),而且各个节点的时钟是不确定同步的,可能很快,也可能很慢.在这种情况下就需要一致性算法来保证提交的有序性和对外一致性,`Paxos` 并不是唯一能做到这一点的算法,经典的两阶段提交(`2PC`)也可以做到,但是`2PC`有一个致命缺点是无法容忍节点宕机,节点出现宕机时整个事务只能无限期阻塞.而 `Paxos` 则可以在集群拥有 `majority` 节点存活的情况下依然正常运行.

通常我们实现模块的时候会采用状态机模型,即给定固定的输入,状态机会给出一个固定的输出,多个节点上都会运行这类 `deterministic` 的状态机,所以集群数据的一致性转换为输出的一致性,进而转化为输入的一致性,而 `Paxos` 这类算法的作用就是为状态机提供一个一致的输入,这在实际工程中表现为提供一致的 `Log`. 由 `Paxos` 决议来决定在某一个日志项中的内容,并保证在所有节点上最终是一致的.

# Paxos 算法分析

我总结 `Paxos` 的核心为一个承诺以及一个不改变(hmm...有点新闻联播的既视感).我们先看一个直观的例子,有两个并行的请求 `A`和 `B` 分别要更新同一个值,可以看到,在三个副本上,由于没有任何措施, `B` 的请求可以覆盖掉 `A` 已经完成的写入,并且 `A` 还顺利返回,并没有意识到这个问题,这就导致两个客户端对系统状态的理解产生了不一致.于是, `Paxos` 用直观的方式解决了这个问题,在此我们简单认为,由于 `Paxos` 的一个承诺的存在,即承诺了接受 `A` 请求那么就会直接拒绝 `B` 的写入.一个不改变是指,下图中如果最终 `B` 写成功,那么这个值将再也无法被修改,它将在历史中永久存在,就像订在区块链的记录里一样.好了,这是用一个直观的例子讲解了下我总结的核心观点,下面我们来看下真正的 `Paxos` 算法.

![图1](https://note.youdao.com/yws/api/personal/file/WEBac1bc1e33f625c8b61e5910824c9aaa5?method=download&shareKey=2e16bbe35e2dd37443c0b9ff911d6729)


## 角色分类

 `Paxos` 将参与整个算法的节点按照在逻辑上执行的不同功能归三类,分别是 `proposer`, `acceptor` 和 `learner`.我们知道, `Paxos` 算法的目的是选出一个值,使这个值被大家所认可(大家先不要纠结这个值是个什么玩意,现在就当他是任意的东西,比如一个字符串,一张图片或者一个网页).而这里的 `proposer` 的主要工作就是提出值选举的方案(`proposal`),每个 `proposal` 包含一个自增的编号 `id` 和它提议的值(`value`). `acceptor` 也比较好理解,就是参与这个提案的审批,决定是否批准(`accept`)提案.比较不怎么直观的是 `leaner`,我看到也有不少不同角度的理解,不过从 `made paxos simple` 原文来看,最正常的一种理解就是当一个值被批准后,将这个值同步到集群中其他节点去,这个被同步的角色就是 `learner`,因为它学习到了最新的值信息.
 
 ## 算法概述
 
 好了,废话不多说,直接看算法,整个算法分为两个阶段(我会将原论文的英文解释也附上,因为我感觉英文好像更直观一些,手动狗头):
- 阶段一: `PREPARE`

提议者将方案广播给集群中的 `acceptor`, 这个动作称为 `Prepare`,`acceptor` 收到 `Prepare` 请求后,检查能否做出承诺,如果新的请求的方案编号比历史上自己承诺过的任何方案编号都大,那么就承诺(`promise`)该请求并记录最近承诺的方案编号,否则予以拒绝(`Reject`).注意这里有一个关键操作是,如果本节点上的值曾经被批准(`accepted`)过,将其最新的值在`promise` 中将会带回.

>  Phase 1.(a) A proposer selects a proposal number 'n' and sends a prepare request with number 'n' to a majority of acceptors. (b) If an acceptor receives a prepare request with number 'n' greater than that of any prepare request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than 'n' and with the highest-numbered proposal (if any) that it has accepted.

- 阶段二: `PROPOSE`
 
当 `proposer` 确认已经有超过半数节点(`majority`)向它返回了 `promise` 之后,即可发起 `Accept` 请求(如果收到任意一个`Reject`则重启新一轮的决议).`Accept` 请求中携带的值有两种来源: ①之前 `Prepare` 阶段从各个节点收集回来的值,选择其中版本号最大的值(由于这个值是在各个 `Acceptor` 节点上被批准的,所以之前是得到了majority批准的,但不一定已经写入到集群中,我们后面的例子会讲解这种情况),此种情况写入的值通常被称为 `constrained value`. ② 前一阶段没有值返回,则自己选择一个值,此种情况写入的值被称为 `uncontrained value`.`acceptor` 收到 `Accept` 请求后和第一步一样,查看自己是否又因为过于开放,承诺了其他更高级的提案,如果是那就拒绝这次请求(`nack`).否则就写入本次方案(批准决议),并向 `learner` 广播该决定.

> (a) If the proposer receives a response to its prepare requests (numbered 'n') from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered 'n' with a value 'v', where 'v' is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals. (b) If an acceptor receives an accept request for a proposal numbered 'n', it accepts the proposal unless it has already responded to a prepare request having a number greater than 'n'.

每一个两阶段的过程称为一个实例(`instance`),实例编号即为上面说的决议方案编号.

整个方案非常简单,以至于无需使用伪代码.但是是通过什么逻辑得出了一定要做这么两步就可以得到一个一致性算法,我还是建议阅读一下 `Lamport` 的论文原文看看推导.下面我们来看几个实际的例子看看这个两阶段是怎么工作的.

## paxos example

假设有 `A`, `B`, `C` 这几个节点,刚开始各个节点上对应的 `value`(还是的,先不要纠结这里的值到底是什么玩意,后文会有解释) 都是 `NULL`.每个被批准的值会最终写入到节点上,记录为<`n`, `v`>,表示在第 `n` 次实例写入的值为 `v`.

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   0  |   0  |   0  |
| lastAccepted | NULL | NULL | NULL |

### 示例(1)

某个 `proposer` 发出一个提案<1,'@'>,表示提案版本号为 `1`, 值为 `@`,在 `prepare` 阶段被三个节点均接受,状态修改为

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   1  |   1  |   1  |
| lastAccepted | NULL | NULL | NULL |

然后 `proposer` 收到半数的 `promise` 回应后进行 `propose`,由于所有节点之前没有批准过任何值,所以 `proposer` 可以自由选择需要写入的值,此时为 `@`,假设所有 `acceptor` 都顺利地将值进行了写入,集群的状态变更为:

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   1  |   1  |   1  |
| lastAccepted | <1,'@'> | <1, '@'> | <1, '@'> |

这时候如果 `learn` 来学习,学习到的只能是 `@`,所以对外一致性确定地表现为 `@`.

### 示例(2)

接着上例,一个新的 `proposer` 进行一轮新的决议,决议为 `2, `#`,假设只有 `A` 节点收到并返回了 `promise`,节点状态变更为

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   2  |   1  |   1  |
| lastAccepted | <1,'@'> | <1, '@'> | <1, '@'> |

由于无法收到超过半数节点的同意,重新进行决议,决议编号升级为 `3`,重新发起 `prepare`,这次 `B`, `C` 节点都收到请求并返回了 `promise`,节点状态变更为

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   2  |   3  |   3  |
| lastAccepted | <1,'@'> | <1, '@'> | <1, '@'> |

注意这时候 `proposer` 收到了 `majority` 节点的回应,并且回应中携带了之前各个节点上被批准的值,为<1, '@'>,根据 `paxos` 对 `proposer` 的规则限定,它写回的值只能是 `@`,节点状态最终为:

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   2  |   3  |   3  |
| lastAccepted | <1,'@'> | <3, '@'> | <3, '@'> |

那么这个 `#` 的值难道就一直写不进去了?下文我们会探讨这个问题.另一点是,我们看到一个值可以被多轮决议批准,但它一定是同一个值.

### 示例(3)

假设集群的进过一轮实例运行后,得到的集群状态如下,这是有可能的,当在 `propose` 阶段只有一个节点成功写入批准的值的话.

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   1  |   1  |   0  |
| lastAccepted | <1,'@'> | NULL | NULL |

此时虽然 `@` 被节点 `A` 批准,但是 `learn` 学习时通过读取一个 `quorum` 无法读到超半数节点对一个值的统一,所以对外表现的值为 `NULL`,即还未有任何值被写入.

假设 `proposer` 发起一轮新的决议 <2, `#`>,并且最终只被节点 `B` 批准,节点状态最终为

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   2  |   2  |   0  |
| lastAccepted | <1,'@'> | <2, '#'> | NULL |

此时节点中 `@`, `#` 都未得到 `majority` 的写入,所以集群的值是不确定的,如果新的一轮决议得到的 `promise` 来自 `A`, `B`, 则集群最终表现为

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   3  |   3  |   0  |
| lastAccepted | <3,'#'> | <3, '#'> | NULL |

此时集群确定 `#` 被超半数写入, `learn` 将学习到这个值.

反之,如果是 `promise` 来自 `A`, `C`,集群最终的状态表现为

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   3  |   2  |   3  |
| lastAccepted | <3,'@'> | <2, '#'> | <3, '@'> |

此时集群确定 `@` 被超半数写入, `learn` 将学习到这个值.

## Live Lock Problem

好了,通过上述示例的演示,相信大家对于基本的决议过程已经有所了解,不难看出,由于两阶段操作的交替进行, `Paxos` 将会遇到如下一个问题,即两个 `proposer` 交替进行 `prepare` 和 `propose`,将会导致两者一直无法决议成功,比如初始时集群状态为:

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   0  |   0  |   0  |
| lastAccepted | NULL | NULL | NULL |

`proposer(1)` 提交决议<1, `@`> 后变更为下图状态:

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   1  |   1  |   1  |
| lastAccepted | NULL | NULL | NULL |

并且 `propose(1)` 准备进行 `propose` 但还未进行,此时 `proposer(2)` 也发起一个决议为 <2, `#`>,由于具有更高的提案号,所以所有节点都决定 `promise` 它,集群变为:

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   2  |   2  |   2  |
| lastAccepted | NULL | NULL | NULL |

此时,`proposer(1)` 的 `propose` 也到达了,但是各节点发现它的提案号 `1` 低于 `lastPromised` 的提案号,于是 `Nack` 予以拒绝.`proposer(1)` 于是更新自己的提案号为 `3` 并继续提交 <3, `@`>, 这次成功将集群状态变更为:

| 节点 | A    | B    | C     |
| ---- | ---- | ---- | ----- |
| lastPromised |   3  |   3  |   3  |
| lastAccepted | NULL | NULL | NULL |

此时,我们的 `proposer(2)` 终于进入到了 `propose` 阶段,发起请求到各节点,和之前类似得到了一个 `Nack`,于是它也升级自己的提案号进行重试,如果没有任何干预,很可能系统将永远无法进行下去,我们称这个现象为 `Paxos` 的 `Live Lock` 问题.
解决 `Live Lock` 的方案就是选出一个 `proposer` 作为 `leader` 来进行提交,以此降低冲突概率,提升提交效率.值得注意的是,这里的 `leader` 角色是一个出于性能优化而引入的角色,而非出于正确性的考虑,这一点和 `raft` 是由很大区别的, `raft` 花了大量精力来保证在同一个 `term` 中只有一个 `leader` 存在.

## so what the fuck value is?

我相信好多人第一次读 `paxos` 的论文觉得不好理解是因为缺乏一个现实的场景,比如即使我们上述的所有示例都理解了,也没有回答一个问题,到底什么是 `value`?或者说一个更严重的问题是,为何一个 `value` 一旦写入后就不可以修改,通常用 `Paxos` 实现的是一个
分布式存储系统,你告诉我一个数据写入后不可以修改,这不是开玩笑么?

这时候让我们转变一下思想,比如在很多系统中,接受的数据操作方式也是 `append-only` 的,比如在各类 `LSM-Tree` 原理的存储引擎中,明明每个操作都是追加,但最终却可以将系统中的值修改掉.对了!这就是我们的状态机模型在这里的作用了,我们说
给定一个状态机,给定特定的起始状态和输入就可以得到固定的输出,假如这里的状态机就是一个我们的 `KV` 数据库示例,它接受的就可以是一个输入日志,每个日志的操作可以是新增,删除和覆盖旧值,这就做到了使用一个 `append` 的日志来做到对系统状态的修改.所以显而易见地, `Paxos` 经过转换,它每次决议
是在决议日志中某一个日志项记录的内容是什么.

# 总结
对于 `Paxos` 其实还有非常多的内容可以讲,比如工程上用得比较多的 `multi-paxos` 等等变形,此篇文章暂时作为一个入门介绍,至少先通过本文让大家对 `Paxos` 有一个基本的理解,并且至少可以认为初步理解了朴素 `Paxos` 算法的一些核心理念.后续我将会通过分析一些实际系统(比如微信开源的 `phxpaxos`<sup>8</sup>) 和其他经典论文来更深入地介绍 `paxos` 协议族的相关信息.


# Reference
- [1] [raft](https://raft.github.io/raft.pdf)
- [2] [polarfs](http://www.vldb.org/pvldb/vol11/p1849-cao.pdf)
- [3] [Zab: High-performance broadcast for primary-backup systems](https://ieeexplore.ieee.org/document/5958223/)
- [4] [Viewstamped Replication: A New Primary Copy Method to Support Highly-Available Distributed Systems](https://dl.acm.org/citation.cfm?id=62549)
- [5] [the part-time parliament](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf)
- [6] [paxos made simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [7] [paxos made live](https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf)
- [8] [paxosstore](http://www.vldb.org/pvldb/vol10/p1730-lin.pdf)
