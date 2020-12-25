# DDC上的数据库系统设计

## 什么是DDC？

DDC 又称 Disaggregated Data Centers，是可能的下一代云计算中心的架构。现在的云服务基本上是monolithic的server，也就是一台节点既包含计算，存储等等，没有办法分开。DDC从概念上来说，就是把计算资源，存储资源更加集中化。资源的索取全部通过网络来进行。

好处：
1. 允许运维人员独立升级资源（原句: DDCs allow operators to upgrade and expand each resource independently）。这点比较好理解，因为你原先的调度单位就是一台服务器，那么一般来说可选的内存大小，CPU资源等等都有固定的模式，很难做到定制。当你把资源disaggregate之后，就可以选：加CPU增强计算能力？加内存增强remote memory能力？加存储扩充容量上限？随着调度单位的缩小，运维人员在upgrade CPU资源的时候，就不用同时upgrade内存资源。

2. 提升资源利用率。现在你购买云服务，很多时候美元办法做到随心所欲的定制你的配置。比如你想买个7 cores, 100 GB RAM, 3GPUs的服务，DDC能做到差不多刚好就给你这么多。但是现在如果你要在AWS上买，满足这个配置的是p3.8xlarge (32 cores, 244 GB of RAM, and 4 GPUs)。。可以对比下看看浪费了多少核心数和内存。本质问题还是没有办法缩小scale的粒度，做到精准配置资源。

挑战：
1. 如何降低通信延迟
2. 如何应对减小的内存
3. 如何设立更完备的failure机制，来应对CPU和内存的independent failure。