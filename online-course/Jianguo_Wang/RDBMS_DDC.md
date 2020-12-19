# DDC上的数据库系统设计

## 什么是DDC？

DDC 又称 Disaggregated Data Centers，是可能的下一代云计算中心的架构。现在的云服务基本上是monolithic的server，也就是一个服务既包含计算，存储等等，没有办法分开。DDC从概念上来说，就是把计算资源，存储资源更加集中化，资源的索取，调度全部通过网络来进行。

好处：
1. 更加容易扩展。
2. 提升资源利用率。

挑战：
1. 如何降低通信延迟
2. 如何应对减小的内存
3. 如何设立更完备的failure机制，来应对CPU和内存的independent failure。