---
layout:     post
title:      "谈谈对分布式锁的理解"
subtitle:   ""
date:       2018-07-20 12:00:00
author:     "cheney"
header-img: "img/mountain.jpg"
catalog: true
tags:
    - Distributed System
---

作为一个分布式系统工程师,在系统设计中总是避免不了要碰到分布式锁的问题,但通常情况下大家都尽量选择避开或者依赖外部服务,因为潜意识中大家都认为分布式锁是一个非常容易出错,很难实现正确的东西,就跟很多时候大家会尽量避免使用 `paxos`,不过近些年由于 `raft`/`pacificA` 等协议的铺垫,大家已经越来越能够接受自己去实现一个 `consensus algorithm` 了,但很大程度上,分布式锁还是有点洪水猛兽的意思.

但事实上,在设计分布式系统的时候,要避开分布式锁的概念是几乎不可能的.甚至在很多时候可能我们不会意识到某些地方实际需要的其实是一个分布式锁,在不明所以的情况下,就可能会给出一个自认为安全的能够解决问题的方案,正如我们知道的,一个算法在理论上能正确工作和他在实现中是否正常工作,是两码事,更别提没意识到要使用某种分布式算法的时候,给出的方案是否可靠就更难以想想了.

##  Lease Type
我在最近重构我们某个存储业务的时候,发现系统多处使用到了[租约(`lease`)](https://www.semanticscholar.org/paper/Leases%3A-An-Efficient-Fault-Tolerant-Mechanism-for-Gray-Cheriton/148ec401da7d5859a9488c0f9a34200de71cc824),然后我发现这里对租约的使用有两种方式,可以总结为要求精确和不要求精确两种.后来我在 `Martin Kleppmann` 的一篇文章<sup>[1]</sup>中发现他更精确地定义了这两种类型:

> Efficiency: Taking a lock saves you from unnecessarily doing the same work twice (e.g. some expensive computation). If the lock fails and two nodes end up doing the same piece of work, the result is a minor increase in cost (you end up paying 5 cents more to AWS than you otherwise would have) or a minor inconvenience (e.g. a user ends up getting the same email notification twice).

> Correctness: Taking a lock prevents concurrent processes from stepping on each others’ toes and messing up the state of your system. If the lock fails and two nodes concurrently work on the same piece of data, the result is a corrupted file, data loss, permanent inconsistency, the wrong dose of a drug administered to a patient, or some other serious problem.

在这篇驳斥 [redlock](https://redis.io/topics/distlock)<sup>2</sup> 的文章中,`Martin` 用 `redis` 的例子来解释了这两种类型的差异.事实上在实际的业务系统里这两种类型也是很常见的用法,比如我正在重构的系统.为此,我需要阐述一下经过简化的我们的存储系统,我们的系统是一个类似 `HBase+HDFS` 的系统,示意图如下:

![layout](https://note.youdao.com/yws/api/personal/file/WEBde5711bfdbe7bc80a3d1370546d6769b?method=download&shareKey=4c8bdfad8a8746ccc067a1632b7ed32f)

### Efficiency Case
假设客户端需要读取一行业务数据(可以理解为表格存储),则通过 `Client` 来访问 `TabletSvr` 上的某个表格,具体访问哪个 `TabletSvr` 则是由存储在 `TabletCenter` 中的路由规则决定的,很自然的,为了避免每个读写请求都需要询问 `TabletCenter` 来获取路由,通常各个 `Client` 就会缓存一份路由规则在本地.到这里我们发现一个经典的分布式问题是: `cache consistency`.我们必须在业务场景中考虑这里对于缓存一致性的要求是怎样的,我们的业务设计为,对某一个 `key` 的写入只允许发生在一个 `TabletSvr` 上,并且写入在 `TabletSvr` 上的语义是 `write-through`,会直接穿透到底层存储但是成功后会在本 `TabletSvr` 上进行缓存,所以一个写入完成后在各个 `TabletSvr` 上都是可以读取到数据的.所以我们允许对同一个 `key` 的读取请求因为不正确的路由信息而到了一个并非它应该被读取的 `TabletSvr` 上,那么就必须从存储层加载该数据来返回给客户端,此时尽管性能差一些,但是不影响业务正确性,正对应了上面总结的租约用途的第一种,即为了提升 `Efficiency`.我们再看写入,因为写入只允许在一个 `TabletSvr` 上进行,所以如果因为不正确的租约导致出现了类似双主的问题,即两个客户端各自认为两台不同的 `TabletSvr` 在服务他们写入的同一个 `key`,当他们的请求达到这两台 `TabletSvr` 上之后,如果此前该节点并未打开某个 `tablet`,则到存储服务进行写打开(`open for write`),由于存储服务本身对外提供了 `single writer` 语义,所以同一时刻只可能有一个客户端打开写成功.这么看的话,在 `Client` 和 `TabletSvr` 这一层如果租约信息不够准确,并不会产生问题.

像上面描述的缓存使用场景是很常见的一种服务端技术,相信很多系统都在使用.很幸运(really?),我们这里用到的场景并没有缓存一致性的要求是一种对缓存一致性,那么如果业务需要缓存一致性的时候,依靠租约该如何解决呢?我们在下面进行分析,从上面的描述相信大家也看到了,我们把一个问题的解决方案给推迟了,但还是不可避免,即 `single writer` 功能.

### Correctness Case
`single writer` 的语义非常明确,即对某个文件的写入操作在任意时刻只允许有一个写入者,这是因为在并发写的时候文件的写入结果是不确定的,轻者产生数据不一致,严重的话则是数据损毁(比如某些关键元数据不一致导致数据服务读取),所以我们在 `FileStoreLayer` 实现了 `single writer`,使用的方式依然是租约,具体实现下文还会分析,我们先来看下文件存储这一层的大概架构(当然,也是经过简化了的):

![store](https://note.youdao.com/yws/api/personal/file/WEB0452aee7e3ddf4d0ce2c416b9565e9cb?method=download&shareKey=6cba6cd664a64ac1ed4d94bbe4c955b1)

`RootServer` 在这里是集群的大脑,类似于 `GFS` 架构的 `master`,主要负责的功能有两块:
- session 管理: 这部分就是实现了所谓的 `single writer`,每个写入者都需要在这里维持一个 `session` 的 `lease`,在这个 `lease` 的有效期内只有它的拥有者可以写入.
- 索引管理: 管理某个文件所在的 `FileServer` 分布,决定 `I/O` 请求最终处理的位置.

当 `TabletSvr` 要处理一个写请求时,首先获取或者检查写入该文件的租约是否有效,如果有效则根据索引信息得到 `FileServer` 的信息,之后将请求发往指定的 `FileServer` 进行处理.那么问题来了,租约机制能够保证真正的 `single writer` 吗?来看看我们给到的租约实现方案:

#### false lease implementation
1. 向 `RootServer` 获取租约,并定期对该租约进行续约.
2. 假设某 `TabletSvr` 节点 `A` 在 `t1` 时刻发起租约续约请求,该请求在 `t2` 时刻返回到节点 `A`.节点 `A` 更新租约的有效期.
3. 在租约有效期内都允许对该租约保护的文件进行写入.

这里需要注意的一个问题是,`t2` 与 `t1` 之间消耗的时间是不可预设的,我们来看三个时间点上,各个对应节点如果服务挂起(`pause/hang`) 导致的问题,假定客户端的租约长度为 `σ`,处于保守考虑那么剩余租约有效期必须短于 `σ - (t2-t1)`, 而不能是 `t1 + σ`,正如下图所示:

![timeline](https://note.youdao.com/yws/api/personal/file/WEB3254228db50af1a294a53626ec6dd79b?method=download&shareKey=0240fff2a923526e6a6ce3d90fba533e)

但真正的问题在于,租约机制是一个非常依赖时钟的系统,正如 `Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency`<sup>3</sup> 这篇论文提到的:

> Leases depend on well-behaved clocks. In particular,a server clock that advances too quickly can cause errors because it may allow a write before the term of a lease held by a previous client has expired at that client. Sim- ilarly, if a client clock fails by advancing too slowly, it may continue using a lease which the server regards as having expired.

在分布式系统中如果需要对物理时钟有要求,通常都具有风险(当然,像 `spanner` 那样通过原子钟实现的 `bound drift clock` 也是一种方式),因为这种要求尽管在绝大多数情况下都是满足的,但不能保证一个分布式协议的 `safety` 要求.这一点也是在 2016 年的 `redlock` 大讨论中<sup>4</sup> `martin` 和 `antirez` 产生重大分歧的点.

所以到此为止,我们在这个要求绝对正确的租约场景下,实现的方式是达不到要求的.

到此为止我一直在谈的是租约机制,可能和分布式锁看起来没有直接联系.但是考虑到分布式锁服务按照实现可以有两类:
- 可以解决死锁问题: 即当锁持有者在释放锁之前 `crash` 的话,这把锁将最终能够释放,以防止后续对其保护的资源访问的饥饿问题.通常实现上会对锁加一个超时时间,在超时时间过后如果锁依然没有被访问(`keepalive`),锁将被强制释放.
- 不能解决死锁问题: 每把锁都需要通过显式的释放来解决.这种实现的缺点在 `chubby`<sup>5</sup> 这篇论文中总结了几点经典理由,这里不再赘述.

可以看到,分布式锁依然面临由于解决死锁问题而引入的租约机制,可以这么说,解决了租约问题,就解决了分布式锁的一个大难点.

#### fencing token
我们先借 `Designing Data-Intensive Applications`<sup>6</sup> 中的两张图来详细解释下分布式锁可能产生的双主问题.通常我们使用分布式锁的方式是这样的:

```
std::string id;
bool succ = LockService.Acquire(&id);
if (!succ) return;

//do something...
try {
    StorageService.Write("xxx")
} finally {
    LockService.Release(id);
}
```

这段代码的问题在于,在获取锁成功之后和处理 `Write` 操作之前,进程有可能被挂起,比如做一个长时间的 `GC`,当挂起结束后程序会继续进行 `Write` 操作,而此时有可能这把锁的有效期已经过了,而且已经在另一个客户端上被获取使用,那么就会导致两个客户端都产生了写操作到存储服务上,而其中只有一个是处于被锁保护下进行的,数据就可能发生并发操作导致损坏.这个过程的示意图是这样的:

![unsafe-write](https://note.youdao.com/yws/api/personal/file/WEBc17cdc6c6a7269246de1839e5d007686?method=download&shareKey=ea2523cd6fa61163362d2a178998a0d8)

即使你认为 `GC` 这种事情只会发生在某些编程语言上,上述描述的场景也可能很难逃避.比如当第一个客户端的请求发出后,并没有直接到达存储端,而是在网络栈上迷失,这有可能花费很长时间才会最终送达,这个时间并不能保证比锁有效期短.`GitHub` 就曾经发生过这样一例事件<sup>7</sup>.

解决这个问题的方式是,使用 `fencing token` 机制,如下图描述:

![fencing token](https://note.youdao.com/yws/api/personal/file/WEB29be0d0226c2970df7a7d8f27579f2cc?method=download&shareKey=52c089aa7284ead6e2794f7480a33a4b)

当每个客户端申请锁的时候,都会产生一个 `token`,并且该 `token` 保证是严格递增的.当请求因为乱序到达存储端时,存储端根据在本地记录的已经处理过的最大 `token`,拒绝已经过期的请求.看起来很完美,但是还是有一个问题,假设上图中的两个客户端的数据包都发生了网络迷失,但是依然是按顺序到达的,即 `token = 33` 的请求先到,此时两个请求实际上都会被处理.考虑到这样一个处理序列:

```
t1: client1 got token = 33 and paused and lease expired
t2: client2 got token = 34 and read the data
t3: client2 do something with read-modify-write
t4: client1 resumed from paused and sent its write to the storage and processed
t5: client2 sent its write to storage and overwrite the previous data
```

我们可以看到这里发生了一次 `Lost Update`,即使 `client2` 在合法的锁保护下,依然产生了丢失更新,这是有些系统无法接受的.我曾经和 `martin` 讨论了下这个问题,他认为应该使用 `causality` 来解决并发写的冲突.我对这个问题的解决方法是,在申请 `token` 的时候让实际资源处理的系统(比如我们这里的 `FileServer`)也参与进来,将这个 `token` 传递给 `FileServer` 之后再返回锁操作.

至此我们得到了一些结论:
- 由于 `asynchronuse system model` (无法预测的时钟动作,无法预测的网络延迟,无法预测的进程挂起)在分布式场景下不可避免,必须引入 `fencing token`,由最终的服务端根据 `fencing token` 进行裁决.但是 `fencing token` 并不是完美的,可能产生丢失更新.而且对于 `fencing token` 的处理需要在服务端实现 `CAS`,并不是所有系统都支持该操作,这对系统选型有一定的指导意义.
- 更为关键的问题是,似乎不能依赖一个外部的锁来做某个资源的互斥访问?那么所有最终处理被锁保护的资源的服务,似乎否应该自己用一致性协议实现锁服务?

上述第二个结论令人感到不安,这个时候大家很容易想到要看的的一篇 `paper`,没错,就是 `chubby`,我们来看看 `chubby` 是怎么干的.

#### google chubby
`chubby` 对于每一个锁都会关联一个 `sequencer`,按我理解这就是我们所说的 `fencing token`了,当请求客户端获取到锁之后会将此 `sequencer` 和请求一起发往服务端,服务端需要检查此 `sequencer` 是否依然有效,如果有效则处理.这个机制我认为从功能上看是完全没有问题的,但是性能可能会是问题,因为服务端可能需要频繁查询 `chubby cell` 来确认某个请求的租约是否还有效,论文提及到这可以使用 `lock cache` 来解决,这个我们后面还会提到.

我们注意到 `chubby` 声称自己是一个 `coarse-grained lock`,这意味者获取锁操作这个请求是很少的.
> instead, we expect coarse-grained use. For example, an
application might use a lock to elect a primary, which
would then handle all access to that data for a considerable
time, perhaps hours or days.

但是并不意味着检查 `sequencer` 这个操作的请求数也不多,因为当某个节点通过 `chubby` 选举出来作为 `primary` 负责处理某个共享数据之后,所有这个节点经手的请求都需要被最终处理请求的节点来校验发起方是否还是合法的 `primary`.`chubby` 在这里的解决方案是使用本地 `cache`,不幸的是它的 `cache` 有效性依然是基于租约实现的.

#### Reference
- [1] [how to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [2] [distlock](https://redis.io/topics/distlock)
- [3] [Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency](https://web.eecs.umich.edu/~mosharaf/Readings/Leases.pdf)
- [4] [is redlock safe?](http://antirez.com/news/101)
- [5] [The Chubby lock service for loosely-coupled distributed systems](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/chubby-osdi06.pdf)
- [6] [DDIA](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321)
- [7] [GitHub Downtime Issue](https://blog.github.com/2012-12-26-downtime-last-saturday/)
