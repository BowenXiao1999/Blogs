# DDC上的数据库系统设计

## 什么是DDC？

DDC 又称 Disaggregated Data Centers，是可能的下一代云计算中心的架构。现在的云服务基本上是monolithic的server，也就是一台节点既包含计算，存储等等，没有办法分开。DDC从概念上来说，就是把计算资源，存储资源更加集中化。资源的索取全部通过网络来进行。现在越来越多的工业级数据库开始采用这种设计思想，比如PolarDB。

## DDC的好处和挑战
好处：
1. 允许运维人员独立升级资源（原句: DDCs allow operators to upgrade and expand each resource independently）。这点比较好理解，因为你原先的调度单位就是一台服务器，那么一般来说可选的内存大小，CPU资源等等都有固定的模式，很难做到定制。当你把资源disaggregate之后，就可以选：加CPU增强计算能力？加内存增强remote memory能力？加存储扩充容量上限？随着调度单位的缩小，运维人员在upgrade CPU资源的时候，就不用同时upgrade内存资源。

2. 提升资源利用率。现在你购买云服务，很多时候美元办法做到随心所欲的定制你的配置。比如你想买个7 cores, 100 GB RAM, 3GPUs的服务，DDC能做到差不多刚好就给你这么多。但是现在如果你要在AWS上买，满足这个配置的是p3.8xlarge (32 cores, 244 GB of RAM, and 4 GPUs)。。可以对比下看看浪费了多少核心数和内存。本质问题还是没有办法缩小scale的粒度，做到精准配置资源。

挑战：
1. 如何降低通信延迟。可以预想到，在这种架构上，数据库必定涉及大量数据在计算节点和内存节点之间的转移。如何减少这部分latency？

2. 如何应对越来越少的working set memory(compute node的内存)? 

3. 如何设立更完备的failure机制，来应对CPU和内存的independent failure。

## DDC的架构

![image-20201227094352495](https://raw.githubusercontent.com/BowenXiao1999/Picture/master/img/20201227094352.png)

DDC的架构还是挺简单的。我们把一个单元称为一个Rack，一个Rack里面有compute, memory, storage。它们在硬件上各有各的特点，从图上就可以看出来。compute节点的计算资源比较丰富，同时有小部分内存，memory和storage节点都有部分计算能力，同时有比较强大的内存和持久化存储的能力。它们通过Rack MMU连接，同时通过ToR Switch和其他Rack连接。



## Parallel Hash Join算子在DDC上的运行过程



以最简单的Parallel Hash Join算子为例，假设A为大表，B已经建好hash index。Parallel Hash Join涉及2个步骤：

1. Partition。这一步是各个节点fetch大表table A，然后计算部分tuples的hash值，根据这个值分区，发送到另外一个计算节点。
2. Join。另外的计算节点拿到后，和对应的hash index做Join，最后做聚合。

![image-20201227101345305](https://raw.githubusercontent.com/BowenXiao1999/Picture/master/img/cloud_native/RDBMS_DDC20201227101345.png)

关注图a，这是普通的数据库设计。假设A是大表，x节点现在要fetch A的columns来做比较。在initial scan中，x节点会先在local memory找，找不到就发request到memory node，memory node找不到就发request到storage node，最终拿到数据。x拿到数据后要做partition，所以Hash A -> send tuples to y。B表的hash index也是同样的fetch步骤，然后一起做Join。在这个过程中，如果compute node的内存不够用了，就会暂时发给loacl memory存着。整体work flow就是如此。可以看到，正如前面所说，这个过程有大量网络通信，延迟很高。具体有多差大家可以看论文，反正当表大于计算节点的local memory的时候，大概都慢10倍。



除了performance challenges，还有reliability challenges。

1. 如果计算节点fail了，内存节点上属于这个节点的内存怎么释放？以前是不会有这个问题的，因为一个节点fail了，代表内存和CPU一起挂。
2. 如果内存节点fail了，计算节点上还有cache，这一部分的recovery怎么做？怎么恢复到原来的状态？



## 解决办法

为了解决上面的这些挑战，作者提出了一些设计思想，主要是Near Data Processing的运用。



为了解决performance:

1. 给storage node添加Scan-Hash原语以及给memory node添加memory grant原语。具体可以看图b。当请求到达storage node之后，它计算好hash后直接把tuples传递到对应节点的remote memory，而不是返回给compute node来计算这个hash。这样极大地减少了延迟。注意，传递到remote memory后，我们还需要memory grant partition，因为这个时候这部分内存还属于x，我们要把它交给y。所以这个时候要更改内存使用权限(x -> y)。有点像编程语言里的所有权这个概念？



为了解决reliability:

1. 防范compute node failures。做一个中心化的heart beat detector，检测每个compute node的心跳。同时给每个任务加一个 "done" flag。如果某个节点挂了，这个中心化的detecor就用二分查找去找remote memory里的"done"，根据这个来进行恢复。

2. read/write replication。读/写的时候，经过多个节点。挺好理解的办法，但是感觉管理多副本状态一致性还是一个难题。

   ![image-20201227103645703](https://raw.githubusercontent.com/BowenXiao1999/Picture/master/img/cloud_native/RDBMS_DDC20201227103645.png)用了这种办法后，你怎么保证不同的memory里的数据状态是一致的？如果不一致，那么你读的时候要怎么知道哪个才是正常的数	据？共识的问题一直是老大难。

3. 用上面这种办法解决memory node failures的问题。





