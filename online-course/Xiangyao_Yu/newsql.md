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

## 
