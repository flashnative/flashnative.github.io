---
layout:     post
title:      "Percolator分布式事务模型原理与应用"
subtitle:   ""
date:       2020-11-18 18:00:00
author:     "cheney"
header-img: "img/mountain.jpg"
catalog: true
tags:
    - distributed system
    - google
    - percolator
    - distributed transaction
---

## Percolator 模型

Percolator[1] 是 Google 发表在 OSDI‘2010 上的论文 *Large-scale Incremental Processing Using Distributed Transactions and Notifications* 中介绍的分布式事务模型。论文的用意是为了帮助 Google Search 提升网页的索引速度，提升网页搜索的 freshness，由于基于 Map-Reduce 的处理框架总是需要遍历整个数据集，对于增量更新的场景是不合适的，所以他们提出了一个增量处理的框架。这篇论文的重点是 Transaction 和 Notification，但是本文仅关注前者。

Percolator 本身是一个依赖 BigTable 作为数据存储引擎的分布式 Key-Value 数据库，是一个典型的通过 2PC 进行分布式事务协调的实现，简单来说是依靠 Timestamp-Order 实现了快照级别的事务。创新点在于，依靠对时间戳的使用和BigTable的单行事务，实现了跨行事务，更进一步说，由于某一行数据可以位于不同的 BigTable 的 TabletServer 之上，所以该事务也是跨节点的。

事务实现时主要参与者有三个：Percolator 客户端、BigTable TabletServer、Timestamp Oracle(TSO)。按照 2PC 的要求分为两个步骤：PreWrite 和 Commit。

## 代码实现

Percolator 论文中提供了一个详细的伪代码实现，我们将对其进行详细分析。

### 事务结构

一个事务允许对多个cell进行读写，其中cell可以是跨行的。写请求在客户端仅仅是先放入缓存中，在此阶段中并不会和 BigTable 进行交互，所以这里用一个数组作为写操作缓存。读操作较为复杂，我们将在介绍完整个写流程之后再回到这一部分。

在一个事务初始化时将会从 TSO 中获取一个时间戳，该时间戳实际上就是 MVCC 的版本号，所有在本事务中修改的数据都会tag上这个版本号，这也是通常的做法，唯一需要保证的就是 TSO 分配的时间戳必须是不可重复并且是递增的，即具备 time order，毕竟事务的冲突判断依赖时间序。

```
1 class Transaction {
2 struct Write { Row row; Column col; string value; };
3 vector<Write> writes ; 
4 int start ts ;
5 
6 Transaction() : start ts (oracle.GetTimestamp()) {} 
7 void Set(Write w) { writes .push back(w); } 
8 bool Get(Row row, Column c, string* value) { ... }
```

### PreWrite 操作

在客户端完成一系列的读写请求的提交后，Percolator准备提交操作，将这些请求提交到 BigTable 去，整个过程分为两个阶段，第一阶段即这里的 PreWrite。PreWrite 将整个事务里涉及的写操作分为两类，一类是 primary记录，一类是 secondary 记录。其中 primary 记录只有一条，而后者可以是多条的，实际上 primary 的选择可以是随机的，论文中固定用第一条操作的记录作为 primary 记录。 

有必要提前介绍下每行涉及的几个重要列(Percolator还会有notify、acknowledge等列用于消息通知，但由于分布式事务功能用不上，这里不作介绍)：

- write：数据列，一个写事务最终被成功提交后，相应的数据部分是存储于这个列中的
- data：临时数据列，MVCC 写过程中会将被修改的数据写入该列，视最终事务的结果是成功commit还是roll-backward、roll-forward来决定如何解释 data 列数据
- lock：锁所在的列，某个事务在进行修改时会通过写入该列来锁住该行，在目前的实现下只要有发现某行上存在锁（任意时刻的锁）即需要终止本事务（对于残留的锁的处理后文会详细描述）

```
26 // Prewrite tries to lock cell w, returning false in case of conflict. 
27 bool Prewrite(Write w, Write primary) { 
28   Column c = w.col;
29   bigtable::Txn T = bigtable::StartRowTransaction(w.row);
30 
31  // Abort on writes after our start timestamp . . .
32  if (T.Read(w.row, c+"write", [start ts , ∞])) return false;
33  // . . . or locks at any timestamp.
34  if (T.Read(w.row, c+"lock", [0, ∞])) return false;
35
36  T.Write(w.row, c+"data", start ts , w.value);
37  T.Write(w.row, c+"lock", start ts ,
38                  {primary.row, primary.col}); // The primary’s location.
39  return T.Commit();
40}
```

每次被 PreWrite 操作的行都需要带上 primary 行，这是为了在后续处理遗留的 lock 列时便于确定事务是否进行 roll-backward 还是 roll-forward。 

第29行启动一个单行事务，注意这部分逻辑都是在客户端执行的，为了在 commit 阶段能够判断是否有冲突的修改，这里很可能获取了该行在 BigTable 服务端记录的一个内部数据版本号。

第32行进入事务后先读取被修改行的 write 列，查看该列上是否已经有一个在事务开始之后被commit的数据，如果有就需要终止本事务，因为本事务中观察到的数据在事务过程中可能被外部其他事务给修改了，所以本事务未观察到新写入的值，Percolator认为这是不安全的所以进行回滚。另外，Percolator比较激进地认为一行上一旦出现一把锁都是不允许本事务继续进行执行的，后面我们会看到 TiKV 在实现时也有类似的要求，这一定程度上可以简化处理逻辑。

当发现冲突并不存在之后，事务开始真正更新这行数据，时间戳即为事务开始时从 TSO 获取到的 `start_ts`,写完 data 列之后将该行对应的 lock 列也更新，注意 lock 列中的值填写的是 primary 记录，后续还需要依赖这个重定位来确定 secondary 记录是应该 roll-backword 还是 roll-forward。

待所有记录的 PreWrite 操作完成后，事务进入到 Commit 阶段。

### Commit 操作

Commit 操作分为两个阶段，第一个阶段是提交 primary 记录，第二个阶段则是提交 secondary 记录，一旦 primary 记录被提交完成，整个事务就完成了，所有该事务内修改的数据也具有可见性。如果 secondary 没有提交完成前客户端或者服务端就发生了crash，Percolator保证在读操作的实现时依然可以看到整个事务的完整性，并执行 roll-forward 将所有记录恢复以得到数据 integrity。

```
41 bool Commit() { 
42     // PreWrite logic...
47
48     int commit ts = oracle .GetTimestamp();
49 
50     // Commit primary first.
51     Write p = primary;
52     bigtable::Txn T = bigtable::StartRowTransaction(p.row);
53     if (!T.Read(p.row, p.col+"lock", [start ts , start ts ]))
54       return false; // aborted while working
55     T.Write(p.row, p.col+"write", commit ts,
56         start ts ); // Pointer to data written at start ts .
57     T.Erase(p.row, p.col+"lock", commit ts);
58     if (!T.Commit()) return false; // commit point
59 
60     // Second phase: write out write records for secondary cells.
61     for (Write w : secondaries) {
62       bigtable::Write(w.row, w.col+"write", commit ts, start ts );
63       bigtable::Erase(w.row, w.col+"lock", commit ts);
64     }
65   return true;
66 } 
67 } // class Transaction
```

第48行向 TSO 申请一个时间戳用于进行 commit 时使用，这是 MVCC 的经典套路。

第52行针对 primary 记录开启一个单行事务，第53行检查 primary 记录上的锁是否还在(时间戳是精确的`start_ts`)，这是因为在之前的 PreWrite 阶段和这里的 Commit 第一阶段并不是原子的，primary 记录设置的锁可能会在 PreWrite 完成后被清除掉。这个清除操作主要是因为Percolator对于残留的lock进行lazy cleanup时导致的，比如如下的场景：

- Txn0 启动，打算修改row("AAA")，row("BBB")
- Txn0 执行完对“AAA”和“BBB” 的PreWrite，还未执行Commit第一阶段
- Txn1 启动，打算修改 row("BBB")，对“BBB”执行lock检查发现有锁，此时启动lazy cleanup查看“BBB”中记录的primary的lock记录是否存在，发现是存在的说明这个primary记录对应的事务还没有Commit，可以对其进行回滚，于是删除其lock
- Txn1 打算执行 Commit，发现primary记录的lock被人清除了，于是事务失败

为了应对这种情况（在RocksDB中可以依靠GetForUpdate[2]来实现），Commit 第一阶段的事务提交前会先检查这个lock是否存在，只有存在时才可以继续提交，由于这个保证是由 BigTable 单行事务提供的，只要第一阶段顺利提交完成，就说明这个 lock 之前还没被人清除，是安全的。确认primary记录安全之后，就可以更新write列了，其value就是对应修改的data列，并清除掉对应行的lock。

**注：Percolator代码中的Erase使用的是commit_ts，按我个人理解应该是start_ts，不知道是否是作者的typo，还请读者一起讨论。**

在primary记录提交完成后整个事务事实上已经可以认为完成了，所有数据已经具有对外可见性，这是通过对读操作的实现来保证的，所以后面不再叙述对secondary记录的Commit操作。

#### Get 操作

```
8 bool Get(Row row, Column c, string* value) { 
9   while (true) {
10    bigtable::Txn T = bigtable::StartRowTransaction(row);
11    // Check for locks that signal concurrent writes.
12    if (T.Read(row, c+"lock", [0, start ts ])) {
13    // There is a pending lock; try to clean it and wait
14      BackoffAndMaybeCleanupLock(row, c);
15      continue;
16    }
17 
18    // Find the latest write below our start timestamp.
19    latest write = T.Read(row, c+"write", [0, start ts ]);
20    if (!latest write.found()) return false; // no data
21    int data ts = latest write.start timestamp();
22    *value = T.Read(row, c+"data", [data ts, data ts]);
23    return true;
24  }
25}
```

Get操作是比较直观的，启动一个事务然后查看要去读的行上是否由锁，需要注意的是，如果存在的锁的时间戳在 `start_ts` 之后的话，是允许的，因为按照快照隔离级别的要求，当前事务对于一个未来事务修改的数据可以是不可见的，但是如果在 `start_ts` 之前残留的锁，Get操作就会进行等待和cleanup操作，大致的含义就是如果多次等待之后发现该锁还存在，就可以按照论文中提及的lazy cleanup将这个锁给清除。

确定没有冲突的锁之后，读事务在第19行先看了下在快照隔离事务允许的时间戳之内是否有合法的write记录，如果有这就是我们要读取的最新的数据，从这个write列中读取出写入这个记录的事务修改的数据所在的data时间戳，利用该时间戳获取到相应数据即可。

这里额外提一句，这类读操作对版本号的使用实际上使得一个点查的数据操作在实际实现中会产生大量的Scan操作，在CockrouchDB的博客[3]中有对该问题的阐述。

#### 对BigTable单行事务的要求

Percolator是依赖BigTable的单行事务构建的，它除了保证事务内的所有写操作能够原子执行之外，还需要保证在事务过程中进行判断时依赖的条件在事务被Commit时也是依然成立的。比如在Percolator执行Commit的第一阶段，当主记录写入write列成功时，必须保证整个过程中lock列都是存在的。

对于使用RocksDB的应用来说，RocksDB提供了GetForUpdate接口来做这一保证，保证一个condition在事务整体过程中的一致，一旦该condition被修改事务将无法提交。

## Percolator 在 TiKV 中的应用

TiKV是较为知名的一个使用Percolator来实现分布式事务的项目，所以我大概分析了下代码，也参看了官方源码分析的一些文章[4]。最开始的一个疑惑是TiKV在实现PreWrite阶段做判断时，读操作都是从一个snapshot进行的，这是使用RocksDB的存储引擎的惯用做法，但是按照前面对单行事务的要求，依靠snapshot并不能满足我们的要求。

比如在Commit主记录阶段，从snapshot中看到的lock还存在，此时执行一个写事务修改write列，并不能保证这个lock在外部没有被清理，看得我百思不得其解，后来发现原来 TiKV 在进入到事务之前，还在其 Scheduler 中为每一个行提供了一个latch来防并发，这个latch限制了对于一个行的修改同时只有一个事务可以进行，这样就可以保证我们的要求了。

## Percolator模型的其他应用

由于 Percolator 实现了一个快照隔离级别的事务，我们可以利用这个功能做一些特性。比如在一个分布式文件系统中做快照，这个文件系统可以在对索引实现中使用Percolator模型保证索引在多节点上修改的一致性。通知在应用层提交一个快照请求时，只需要从 TSO 获取一个时间戳来表示快照时间点，这个时间点前后都可以保证整个文件系统索引的内部一致性。

## Reference

- [1] [Percolator](https://research.google/pubs/pub36726/)
- [2] [RocksDB Transactions](https://github.com/facebook/rocksdb/wiki/Transactions)
- [3] [Why we built CockroachDB on top of RocksDB](https://www.cockroachlabs.com/blog/cockroachdb-on-rocksd/)
- [4] [TiKV 源码解析系列文章（十二）分布式事务](https://zhuanlan.zhihu.com/p/77846678)
