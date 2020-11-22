# What's Really New with NewSQL

这是Andy Pavlo 15年发在SIGMOD上的论文。主要是一篇综述。个人感觉这篇文章的主要亮点在于分析SOTA和对Trend一些预测，也重点读了一下这部分。

## Main Memory Storage
以前的数据库设计，由于内存的昂贵价格，都采用了Disk-Based的结构。使用Block的硬盘如HDD, SDD做持久性存储，在内存中使用buffer pool manager来管理内存页的换进换出。

由于科技的进步，现在通过把所有数据库的所有数据放进main memory成了现实。尽管如此，我还不是不太相信仅凭内存就能实现数据库的所有功能。我认为，只不过现在的设计方向从disk-oriented转移到了memory-oriented。

对于memory-oriented的数据库，一个比较难的问题是：如何handle大于当前内存的数据库。这需要设计内存Evict策略（个人感觉怎么和disk-oriented还是差不多。。）比如H-Store是用一种叫anti-caching的方式，把evicted的tuple移到disk上然后再内存对应位置插入tombstone。当要查询evicted的tuple时（也即查到tombstone标记），立刻起一个线程去disk上搬回来。这样的麻烦之处在于：还是要是用某种tracking算法来决定哪些数据应该被evicted（这不就是buffer pool manager嘛。。）。

EPFL有个小组用OS的Virtual Paging的方式（做在了VoltDB）来做上面说的这个tracking工作。我的理解是避免了重复造轮子，应用了OS那一套东西（但是感觉越高效的数据库越应该抛弃OS...）。

memory-oriented数据库要减少memory使用量，于是我们需要比较好的tracking算法，及时清理过期数据。但是我们知道，index占用的空间也是很大的。有时候一条记录删除了，在index里面我们还是要保留，这对于有很多数据库索引表的设计是不太友好的。于是我们通过给每个index表配个BloomFilter来减少内存占用量（所以index表还是放在disk上了？）。比如Microsoft的Project Siberia。

后来我还搜到了，有人说main-memory的亮点不在于内存，在于column storage（比如MemSQL）。我把他的答案节选贴在这。

> 1. in-memory数据库的最大卖点其实不是把数据放在内存里. 2. 最大的变化是把传统关系型数据库(RDBMS)里表的存储方式从行存储变为列存储. 列存储的好处是:2a. 数据的压缩率可以很大, 因为往往很多应用表里一列的数据冗余度很大2b. 对于OLAP系统来说, 求和, 平均等aggregation的查询在列上操作效率非常高所以列存储已经成为所有所谓in-memory数据库的标准配置

https://www.zhihu.com/question/19883454/answer/28274134。

这里其实Pavlo也就提了一嘴，以后有空再研究。

## Partition/Sharding
这一章Pavlo行文思路是：分布式数据库引出了数据Partition，数据Partition带来了分布式事务和分布式查询，这些都是早就有了的概念。那NewSQL新在哪里呢？

### Why Sharding?
为什么要Sharding？因为我们当前的OLTP应用形态决定的。举例，我们可以把同一个顾客的信息，他的订单历史，账户信息通过Sharding的方式放在一台节点上，避免网络开销。如果我们做不到这点的话，分布式事务带来的开销可能让分布式数据库的推行难上加难。

### Heterogenous Architecture
为了保证分布式系统的可扩展性，我们在添加节点的时候，不希望re-partition整个数据库。

有一帮人搞出了heterogeneous，比如NuoDB。NuoDB里有2种节点，storage managers (SM) 和 transaction engines (TEs)。SM只有存储能力没有计算能力，TE是维护分片的内存缓存。当查询来临时，某个TE负责从其他TE/SM处获取数据。同时用了load-balancing的技巧，所以一起使用的数据经常放在同一台TE上。

相似的还有MemSQL，两者的区别在于“如何减少查询时从storage node带来的计算量”。NuoDB通过在TE维护缓存来做到这点，但是这样来看TE就不是无状态的了。而MemSQL通过算子下推（或者叫Near Data Processing）来减少。

读完后，我想这不就是计算和存储分离嘛？我是比较支持MemSQL这种架构的，感觉存储层下推部分计算是趋势。

### Live Migration
NewSQL比较新的一个feature就是它能做到partition的live migration。第一种方式是粗粒度的"vitural partitions"（没读明白），第二种通过细粒度的range partition（比如PD in TiDB）。

## Concurrency Control (CC)
Pavlo总结说这个领域没什么新意了，看来千万不要作死往这个领域做。

首先最早实现CC的就是靠锁，比如2PL。但是这带来了很多问题，比如要怎么处理死锁。

由于处理死锁带来的复杂度，人们开始提出新的CC协议，出现了TO(Timestamp Ordering)，这种架构往往意味着分布式MVCC，在CRDB, HANA等新型数据库里很常见。对发号器的要求也非常高。基本不需要处理死锁，即使update同一个tuple事务也不一定会冲突（冲突了回滚就行）。这里来分析一下CRDB是怎么做的。




但更多，更经典的还是2PL + MVCC的结合。最著名的就是InnoDB了。但这里面Spanner是特立独行的存在，因为GPS和原子钟的高度精确性，它能保证非常强的一致性。

* Distributed 2PL怎么做到？
* Distributed MVCC怎么做到？
* 拿锁是要去lock manager拿嘛？

## Secondary Indexes


## Replication


## Crash Recovery

## Future Trends
