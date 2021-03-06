# Raft算法相关问题以及解释

本文长期记录一些Raft相关的问题，主要是面试被问到的。主要不是基本原理，而是一些细节和异常情况的优化。



### Q: 节点故障后为何不重置为Candidate，而是Follower? 

正常来说，一个节点在挂了之后重新加入集群，是以Follower的身份，需要等待一定时间才能开始选举。但是为什么不设计成挂了之后以Candidate的身份，一加入就开始选举投票呢？从正确性的角度上来说，这不会影响Raft的正确性：如果新加入的节点立刻开始选举，选举成功即为Leader，失败即为Follower。同时假设Leader挂了，那么这个时候相当于群龙无首，就算立刻把Leader拉起来，也要等待一定的选举时间才能选举出新Leader，开始处理业务。如果拉起来就能立刻选举的话，能有效减少等待选举的时间。既然这种设计不会带来错误，同时又在某些Corner Case上表现不错，为什么工业上不这么设计呢？

A: 主要是从工业实际运行的稳定性上考虑的。虽然分布式系统，crash是常态，但是正常运行的时间始终是占大多数的。如果挂了的节点拉起来之后重置为Candidate，那么会比原来多出很多Leader无缘无故切换的情况，扰乱系统稳定性。因此不推荐这种做法。（小声：那对于原来是Leader的节点，crash之后重置为Candidate立刻开始投票可不可以呢？） 



### Q：多个节点同时选举，能否成功？ 选举时不首先投给自己呢？

在Raft中，虽然采取了随机时间选举，但是假设一下现在多个节点同时开始选举，那么选举是不可能成功的，因为每个节点都会首先把票投给了自己。那么我们突发奇想一下，假设所有节点发起投票时，不首先把票投给自己呢？这样的话，还是有可能选举失败（三节点：A投给B，B投给C，C投给A，每个节点各一票）。同时在其他场景中，如果出现网络分区（分为多数派和少数派），假设多数派节点数为３，少数派为２。那么理论上来说多数派节点可以选举出Leader对外提供服务，但如果选举的时候不投给自己，可能就因为这一票永远无法选举出Leader，因此这种做法不可选。



### Q：Ｍaster在准备Commit某条Log的时候挂了，Raft如何保证一致性？

因为Master在Commit某条Log的时候，这个Log已经被复制到集群中半数以上节点了。根据Raft的选举性质，那么之后再选举出来的Leader，一定会包含这条Log。对于这条Log，是上一个Term的，那么当前Leader不能直接commit，而是要发送AppendEntries RPC来commit这条log。具体原因见论文图８。



### Q: 假设某个Follower节点出现网络分区，由于接收不到Leader的心跳包，所以会不断选举，Term会一直增加。加入原集群后会把原Leader降级为Follower，导致重新选举。但实际上它并不能成为Leader(没有最新日志)，造成disruption。如何解决？

Raft作者博士论文《CONSENSUS: BRIDGING THEORY AND PRACTICE》的第9.6节 "Preventing disruptions when a server rejoins the cluster"提到了PreVote算法的大概实现思路。

主要就是Prevote的思想。把选举也拆成一个两阶段提交的方式，先进行一轮prevote，这一轮prevote并不会更改自己的Term＋１，但是Vote请求里的Term是＋１的。如果收到大多数的赞成票，那么发起真正选举。否则继续等待下一轮。（好奇的是Etcd的Raft应该对Prevote票和Vote票做区分？如果Prevote投过这个Term就不能再投了，那就非常奇怪了。所以应该是区别对待的）



### Q: 假设Ａ, B, C三节点，A为Leader。A和B之间出现分区，但是A, C和B, C之间连接不受影响，会导致Leader频繁切换，有什么方法优化？

A, B, C三节点，只有A和B之间的网络不通，而C是可以和A, B 连通的，有什么办法优化？由于Leader A和B无法连通，B会选举成为新Leader (C 会投给它)。同时Leader A的心跳包会被拒绝，降级为Follower(因为Term的原因)。过一段时间后，A也会经历一样的过程，选举成为Leader，B降级为Follower，如此反复。

应该办法是在系统中去detect这种网络分区，如果发现了的话，要及时修复连接。以及C其实可以观测到频繁的Leader切换，从而报警，或者自己选举成为新Leader。好奇其他算法是怎么做的。







