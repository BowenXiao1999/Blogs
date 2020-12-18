# Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases


https://awsmedia.awsstatic-china.com/blog/2017/aurora-design-considerations-paper.pdf

阅读前置：2PC

## Abstract

Aurora是亚马逊的OLTP数据库。最大的亮点在于把Redo Processsing融合进特制的存储引擎（Purpose for Aurora）。这样做的原因主要是亚马逊内部认为现代存储系统的瓶颈在于网络I/O而不是CPU计算。好处有：

1. 减少网络流量
2. 容错机制
3. 自愈存储



## Introduction

计算与存储分离是现代系统设计的大趋势。Aurora的Kernel设计如下

database tier（query processor, undo management, locking, buffer cache, access methods, transactions）

storage tier (redo logging, durable storage, crash recovery, backup)

Aurora的三点优势：

1. 存储系统跨数据中心，提供自愈能力
2. 减少网络流量，只写redo log
3. 把备份和redo recovery转换为异步操作





## Durability At Scale

Aurora的quorum模型，segment storage，以及它们是如何增强可用性和减少抖动的。



## The Log is the database

如果用传统的数据库来写入segemented 和 replicated的数据，会造成写放大的问题。Aurora通过offload log to storage来解决。

### The burden of amplified Writes

![image-20200815003103560](https://raw.githubusercontent.com/BowenXiao1999/Picture/master/img/20200815003103.png)

如图，需要写入这么多文件。

### Offloading Redo Processing to Storage

> In Aurora, the only writes that cross the network are redo log records. No pages are ever written from the database tier, not for background writes, not for checkpointing, and not for cache eviction. Instead, the log applicator is pushed to the storage tier where it can be used to generate database pages in background or on demand. 

由于存储和计算的分离，Database 可以很快的start up (不过read request不是还是要apply log么。。)



### Storage Service Design Points

![image-20200814134051973](https://raw.githubusercontent.com/BowenXiao1999/Picture/master/20200814134052.png)

减少foreground write request的latency，只有1是synchronous的。



## The LOG MARCHES FORWARD

1. Crash Recovery如何使用redo log才能更快
2. 解释一些读写流程
3. 故障恢复的步骤

### 4.1 Solution sketch: Asynchronous Processing

1. 不同数据副本之间quorum交流，fill gap
2. 计算与存储分离，在restart的时候，storage可以do its own recovery

### 4.2 Normal Operation

Aurora的事务控制

1. 

### 4.3 Recovery





## LESSONS LEARNED







## RELATED WORK

1. Decoupling storage from compute

   

2. 