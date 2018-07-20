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

作为一个分布式系统工程师,多年来在系统设计中总是尽量避免使用所谓的分布式锁,因为潜意识中大家都认为分布式锁是一个非常容易出错,很难实现正确的东西,就跟很多时候大家会尽量避免使用 `paxos`,不过近些年由于 `raft`/`pacificA` 等协议的铺垫,大家已经越来越能够接受自己去实现一个 `consensus algorithm` 了,但很大程度上,分布式锁还是有点洪水猛兽的意思.

不过事实上,在设计分布式系统的时候,要避开分布式锁的概念是几乎不可能的.甚至在很多时候可能我们不会意识到某些地方实际需要的其实是一个分布式锁,在不明所以的情况下,就可能会给出一个自认为安全的能够解决问题的方案,正如我们知道的,一个算法在理论上能正确工作和他在实现中是否正常工作,是两码事,更别提没意识到要使用某种分布式算法的时候,给出的方案是否可靠就更难以想想了.

##  Lease Type
我在最近重构我们某个存储业务的时候,发现系统多处使用到了[租约(`lease`)](https://www.semanticscholar.org/paper/Leases%3A-An-Efficient-Fault-Tolerant-Mechanism-for-Gray-Cheriton/148ec401da7d5859a9488c0f9a34200de71cc824),然后我发现这里对租约的使用有两种方式,可以总结为要求精确和不要求精确两种.后来我在 `Martin Kleppmann` 的一篇文章<sup>[1]</sup>中发现他更精确地定义了这两种类型:

> Efficiency: Taking a lock saves you from unnecessarily doing the same work twice (e.g. some expensive computation). If the lock fails and two nodes end up doing the same piece of work, the result is a minor increase in cost (you end up paying 5 cents more to AWS than you otherwise would have) or a minor inconvenience (e.g. a user ends up getting the same email notification twice).

> Correctness: Taking a lock prevents concurrent processes from stepping on each others’ toes and messing up the state of your system. If the lock fails and two nodes concurrently work on the same piece of data, the result is a corrupted file, data loss, permanent inconsistency, the wrong dose of a drug administered to a patient, or some other serious problem.

在这篇驳斥 [redlock](https://redis.io/topics/distlock) 的文章中,`Martin` 用 `redis` 的例子来解释了这两种类型的差异.事实上在实际的业务系统里这两种类型也是很常见的用法,比如我正在重构的系统.为此,我需要阐述一下经过简化的我们的存储系统,我们的系统是一个类似 `HBase+HDFS` 的系统,示意图如下:

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

但真正的问题在于,租约机制是一个非常依赖时钟的系统,正如 `Leases: An Efficient Fault-Tolerant Mechanism for Distributed File Cache Consistency` 这篇论文提到的:

> Leases depend on well-behaved clocks. In particular,a server clock that advances too quickly can cause errors because it may allow a write before the term of a lease held by a previous client has expired at that client. Sim- ilarly, if a client clock fails by advancing too slowly, it may continue using a lease which the server regards as having expired.

在分布式系统中如果需要对物理时钟有要求,通常都具有风险(当然,像 `spanner` 那样通过原子钟实现的 `bound drift clock` 也是一种方式),因为这种要求尽管在绝大多数情况下都是满足的,但不能保证一个分布式协议的 `safety` 要求.

所以到此为止,我们在这个要求绝对正确的租约场景下,实现的方式是达不到要求的.

到此为止我一直在谈的是租约机制,可能和分布式锁看起来没有直接联系.但是考虑到分布式锁服务按照实现可以有两类:
- 可以解决死锁问题: 即当锁持有者在释放锁之前 `crash` 的话,这把锁将最终能够释放,以防止后续对其保护的资源访问的饥饿问题.通常实现上会对锁加一个超时时间,在超时时间过后如果锁依然没有被访问(`keepalive`),锁将被强制释放.
- 不能解决死锁问题: 每把锁都需要通过显式的释放来解决.这种实现的缺点在 `chubby`<sup>2</sup> 这篇论文中总结了几点经典理由,这里不再赘述.

可以看到,分布式锁依然面临由于解决死锁问题而引入的租约机制,可以这么说,解决了租约问题,就解决了分布式锁的一个大难点.

#### Reference
- [1] [how to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [2] [The Chubby lock service for loosely-coupled distributed systems](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/chubby-osdi06.pdf)
