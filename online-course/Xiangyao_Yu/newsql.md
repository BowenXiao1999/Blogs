# What's Really New with NewSQL

这是Andy Pavlo 15年发在SIGMOD上的论文。主要是一篇综述。个人感觉这篇文章的主要亮点在于分析SOTA和对Trend一些预测，也重点读了一下这部分。

接下来的每个英文标题的小节一般是Newsql的一种技术，会去讲Newsql在这些技术上有什么新鲜玩意。

## Main Memory Storage
以前的数据库设计，由于内存的昂贵价格，都采用了Disk-Based的结构。使用Block的硬盘如HDD, SDD做持久性存储，在内存中使用buffer pool manager来管理内存页的换进换出。

由于科技的进步，现在通过把所有数据库的所有数据放进main memory成了现实。尽管如此，我还不是不太相信仅凭内存就能实现数据库的所有功能，只不过现在的设计方向从disk-oriented转移到了memory-oriented。

对于memory-oriented的数据库，一个比较难的问题是：如何handle大于当前内存的数据库（俗称Larger than memory database）。这需要设计内存Evict策略，比如H-Store是用一种叫anti-caching的方式，把evicted的tuple移到disk上然后再内存对应位置插入tombstone。当要查询evicted的tuple时（也即查到tombstone标记），立刻起一个线程去disk上搬回来。这样的麻烦之处在于：还是要是用某种tracking算法来决定哪些数据应该被evicted并且已经被evicted（这不就是buffer pool manager嘛。。）。

EPFL有个小组用OS的Virtual Paging的方式（做在了VoltDB）来做上面说的这个tracking工作。我的理解是避免了重复造轮子，应用了OS那一套东西（但是感觉越高效的数据库越应该抛弃OS...）。

memory-oriented数据库要减少memory使用量，这需要比较好的tracking算法，及时清理过期数据。但是我们知道，索引结构占用的空间也是很大的。有时候一条记录删除了，在索引结构里面我们还是要保留，这对于有很多索引表的schema是不太友好的（太占空间）。于是我们通过给每个index表配个BloomFilter来减少内存占用量（所以index表还是放在disk上了？）。比如Microsoft的Project Siberia。（感觉这里的优化方式也很普通，为了减少索引结构占用量用BF不是常规操作嘛。。）

后来我还搜到了，有人说main-memory的亮点不在于内存，在于column storage（比如MemSQL）。我把他的答案节选贴在这。

> 1. in-memory数据库的最大卖点其实不是把数据放在内存里. 2. 最大的变化是把传统关系型数据库(RDBMS)里表的存储方式从行存储变为列存储. 列存储的好处是:
2a. 数据的压缩率可以很大, 因为往往很多应用表里一列的数据冗余度很大
2b. 对于OLAP系统来说, 求和, 平均等aggregation的查询在列上操作效率非常高所以列存储已经成为所有所谓in-memory数据库的标准配置

https://www.zhihu.com/question/19883454/answer/28274134。

所以说难怪比较难看到纯TP的Main-Memory数据库，因为只有行存才能保证操作效率，但是行存又不好压缩。其实Pavlo也就提了一嘴，以后有空再研究。

## Partition/Sharding
这一章Pavlo行文思路是：分布式数据库引出了数据Partition，数据Partition带来了分布式事务和分布式查询，这些都是早就有了的概念。那NewSQL新在哪里呢？分几个小节来谈一下。

### Why Sharding?
为什么要Sharding？因为我们当前的OLTP应用形态决定的。举例，我们可以把同一个顾客的信息，他的订单历史，账户信息通过Sharding的方式放在一台节点上，这样我们在同时更新这些信息的时候，能避免网络开销（单节点完成）。如果我们做不到这点的话，分布式事务带来的开销可能让分布式数据库的推行难上加难（效率低）。

### Heterogenous Architecture
为了保证分布式系统的可扩展性，我们在添加节点的时候，不希望re-partition整个数据库。

有一帮人搞出了heterogeneous架构，比如NuoDB。NuoDB里有2种节点，storage managers (SM) 和 transaction engines (TEs)。SM只有存储能力没有计算能力，TE是维护分片的内存缓存。当查询来临时，某个TE负责从其他TE/SM处获取数据。同时用了load-balancing的技巧，所以一起使用的数据经常放在同一台TE上。

相似的还有MemSQL，两者的区别在于“如何减少查询时从storage node带来的计算量”。NuoDB通过在TE维护缓存来做到这点，但是这样来看TE就不是无状态的了，所以跟TiDB的计算节点还是不一样。而MemSQL通过算子下推（或者叫Near Data Processing）来减少。

读完后，我想这不就是计算和存储分离嘛？我是比较支持MemSQL这种架构的，感觉存储层下推部分计算是趋势。

### Live Migration
NewSQL比较新的一个feature就是它能做到partition的live migration。第一种方式是粗粒度的"vitural partitions"（没读明白），第二种通过细粒度的range partition（比如PD in TiDB）。

## Concurrency Control (CC)
Pavlo总结说这个领域没什么新意了，看来千万不要作死往这个领域做。

首先最早实现CC的就是靠锁，比如2PL。但是这带来了很多问题，比如要怎么处理死锁，而且效率也比较低。

由于处理死锁带来的复杂度，人们开始提出新的CC协议，出现了TO(Timestamp Ordering)，这种架构往往意味着分布式MVCC，在CRDB, HANA等新型数据库里很常见。个人觉得对发号器的要求也非常高。基本不需要处理死锁，即使update同一个tuple事务也不一定会冲突（冲突了回滚就行）。“只要比较数据的写入时间戳（即写入该数据的事务的提交时间戳）和 Snapshot 的读时间戳，即可判断出可见性。”


但更多数据库使用的CC算法，还是2PL + MVCC的结合。最著名的就是InnoDB了。但这里面Spanner是特立独行的存在，因为其他所有数据库都只能从软件上去设计CC，但是谷歌能从硬件上做文章。但是这种一般都是依靠GTM（Global Transaction Manager）。GTM是个比TSO (Time Stamp Oracle) 难处理的单点问题，所以TiDB没有选择GTM，也因此导致了它在某些事务可见性上的设计和MySQL不同，以及不能完全兼容MySQL。


还有一种比较新型的算法是VoltDB使用的。它保证每个Partition同时只有一个事务在进行处理。这是通过TO + 单线程来实现的。通过TO，对待处理事务进行排序，**一个个执行**。由于只有一个事务执行，因此不必担心锁冲突的问题，速度很快。但是很明显，这样做是不太对的：当你的事务span多个partition的时候，任意时刻只有一个partition会工作，其他都会被你当前事务锁住而无法处理其他的事务。

对于CC，个人觉得@Eric Fu这篇文章讲的很好，我搬运下
https://ericfu.me/timestamp-in-distributed-trans/

## Secondary Indexes
二级索引在单机情况下很好支持，因为它只有一个节点，但是在分布式的情况下却难了。

对于一个分布式数据库，要设计二级索引，先问2个问题：
1. 这个索引结构存放在哪里？
2. 当事务操作可能改变索引结构时，系统怎么维护索引？

这样就诞生了2大流派：partition-based, replication-based。看名字很好理解，partition-based就是像数据表结构一样水平拆分成多个，分散地存放在多个节点；replication-based就是每个节点存放一个索引副本。

它们的优缺点也很好分析：partition-based的话，查询起来不太容易，可能会有较多网络消耗（因为事先无法知道要查询的索引数据在哪个节点上），但是update起来很简单，不必考虑副本一致性问题；replication-based的话，查询起来很容易，因为很容易找到含有对应索引数据的节点，但是当事务操作时，往往需要一个分布式事务原子性地改变所有副本的索引数据。

Clustrix是一种two-layered的实现。首先一层是replication的数据，分散在所有节点，维护的是 \[索引数据\] -> \[对应节点\] 的映射关系。第二层就是partition-based，查询查到在哪个节点后，直接操作对应的索引结构。感觉就是权衡吧，也不能说很新，结合2-layered思想的系统多了去了。

对于不提供secondary-index的分布式数据库，可能要工程师自己部署一个Memcached，坏处是增加了复杂度（要自己维护cache一致性等问题）。

## Replication

数据冗余是分布式数据库提供高可用和高容错率的常用手段之一。Newsql往往把这种特性隐藏起来，不需要用户有额外的心智负担。Newsql为了达到副本数据的高度一致性（可能为了只读副本的查询），往往必须忍受2PC带来的网络开销。Nosql提供的较低级别的一致性，很多是为了躲避Replication这个老大难的问题。这些都是老生常谈了。

总的来说，Replication有2种方式: 一种是Active-Active，即所有副本同时并行地执行Query，同时达到下一个状态。一种是Active-Passive，master执行完后把结果状态传递给replica节点进行同步。大部分Newsql都是第二种，因为第一种太理想化，需要使用确定性的并发控制算法。而大部分DBMS没有办法做到每个Query在不同节点上的查询是完全same order的。

当然，也有实现了确定性并发控制算法的DBMS (AkA Deterministic DBMS)，至于怎么实现的，留个坑之后研究。其实这引出了一个本质问题：为什么数据库需要共识算法？因为数据库没办法保证从一个状态执行相同的Query能共同到达下一个状态。Deterministic DBMS就是为此而生的：减少Agreement Protocol （譬如通常的 two-phase commit）的使用。

> Calvin 是一个 deterministic database，也就是说，对于需要处理的事务，Calvin 会在全局确定好事务的顺序，并按照这个顺序执行。这些事务的执行顺序，我们可以认为是一个全局的有序 log，并且 Calvin 会通过复制机制来备份到不同的副本上面。所有的副本都是可以并行的顺序执行这些 log 的。这样，即使一个副本当掉，另外副本也仍然能执行并且 commit 事务。

感觉就是实现了一个中间件，这个中间件是一个事务调度器，能在事务开始前给每个副本定下一个执行的log，使得所有的副本能并发的执行。

## Crash Recovery

Newsql的Crash Recovery比传统的单机数据库相比，除了要保证数据不丢失，还要尽可能地减少down time。Master挂了之后，除了要load checkpoint and WAL from local storage，还要pull new logs starting from new master。这种实现一般是physical的log，因为只有这样同步起来才会比run sql要快很多。

感觉这一部分也是老生常谈，没啥好讲的。

## Future Trends

看看Pavlo在16年是怎么预测的。他觉得随着IT业的发展，数据特点也在变化。现在的数据更加讲究时效性，越新的数据越有分析价值。因此他觉得HTAP + Online Analytics的方式是主流。他相当看衰big OLAP 仓库，觉得它们的末日马上就要来了，以后公司会把TP和AP请求跑在同一台数据库上。然后顺带分析了一下HTAP的实现，大概就是3种。

1. Deploy 2套数据库，通过CDC或者ETL update数据，delay以分钟甚至小时计算。
2. Lambda架构，一套batch processing system + 一套stream processing system
3. 一套HTAP，融合TP技术(lock free, in-memory storage)和AP技术(column store, vectorized execution)

## 总结
我认为如果以数据库从业者的角度来看的话，这篇文章的价值不大，当然，我这里没法从2016年的角度去思考。有很多分布式数据库的设计在现在看来都是很正常的，不过在当时应该是一篇不错的综述了。

## Ref
https://blog.csdn.net/Gexrior/article/details/80917508

