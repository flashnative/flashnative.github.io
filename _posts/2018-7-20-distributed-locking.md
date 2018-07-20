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


#### Reference
- [1] [how to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
