# Lock Free Objects　总结
source: https://www.cnblogs.com/catch/p/3161403.html

## 什么是无锁
很多人觉得无锁其实就是没有mutex等等锁来设计数据结构，但实际上它代表的是一种编程理念：**如何在理论上无限并发的情况下保证程序不会永远阻塞？**。举个例子：
```c
while (X == 0)
{
    X = 1 - X;
}
```
这段代码在多线程情况下是有可能被无限阻塞的（具体原因自己可以思考），所以这也不是无锁的。


## 无锁队列的实现难点

### 节点分配
考虑用链表来实现无锁队列。当我们要Push的时候，我们不可避免会new一个新节点。如果这个new本身是有锁的——事实上，标准库的new就是有锁的——那我们的无锁其实是假的无锁。解决方法是可以自己实现一套内存分配机制，或者使用TLS(Thread Local Storage)。？顾名思义，TLS是用线程的内存资源。所有分配操作在线程内进行，不和其他线程竞争。缺点也有，是可移植性比较差。

### Memory Reclamation
这个可能是最难的问题: 对于多线程编程，如何确定我们访问的内存是合法的。在多线程下，任何你的访问内存的操作都要如履薄冰。

对于这个问题，解决的办法也比较粗暴：不允许释放链表的节点。我们先分配好全部的内存，并且不允许释放相关的节点。


### ABA问题
假设有一个这样的队列存在，node1 -> node2 -> node3

假设线程 1，开始执行 dequeue。我们知道实现dequeue要首先把队首元素head取出，并且把下一个元素next设置成新的next。为了支持并发读写，我们这里使用cas(q->head, head, next)来完成替换。假设线程１将要执行到这一步的时候被换出去， head, next 分别指向 node1, node2，什么事都还没发生。

另一个线程 2，开始执行dequeue，然后它成功把 node1, node2, node3 都pop 出去了，此时队列为空！

然后又切换到线程 3，它要执行 enqueue 操作，此时它会在 27 行分配一个节点，坏事来了，如果十分巧合的情况下，它分配得到了线程 1 所 free 掉的 node1，enqueue 之后，队列成了如下样子：

node1-> NULL

线程 3 执行完后，如果恰好又切换回线程 1，此时，对线程 1 来说，它无法分辨出q->head 和head的区别（因为线程3恰好分配了和原来内存地址相同的节点）, 因此 cas 会成功！但是 next 呢？next 指向的是被释放掉的 node2! 严重问题！

以上问题可以这么抽象：
1. 进程P1在共享变量中读到值为A
2. P1被抢占了，进程P2执行
3. P2把共享变量里的值从A改成了B，再改回到A，此时被P1抢占。
4. P1回来看到共享变量里的值没有被改变，于是继续执行。

这个就是无锁世界里所谓的 ABA 问题，这个问题就是我为什么最开始想尝试用数组来实现无锁操作的原因之二，而且，ABA 问题不容易解决。

目前相关的解决方案主要有：
1. 通过软件解决（Hazard Pointers）
2. 硬件支持 (CAS2)

这里主要讲一下第二种解决方案。可以看到上面的ABA问题，难点是在于传统cas没有办法判断2次的head(node1)是不一样的，那我们就通过自定义Pointer，这个新Pointer包含了原来Pointer指向的位置＋一个tag。我们只要保证每次新加入一个Node的时候，Tag不一样就行了。本质cas2上是通过原子性地比较更多信息来确定cas是否执行。


## Ref
