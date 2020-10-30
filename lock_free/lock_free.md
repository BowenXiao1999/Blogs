# Lock Free Objects　总结
source: https://www.cnblogs.com/catch/p/3161403.html

## 无锁队列的实现难点

### ABA问题
假设有一个这样的队列存在，node1 -> node2 -> node3

假设线程 1，开始执行 dequeue。我们知道实现dequeue要首先把队首元素head取出，并且把下一个元素next设置成新的next。为了支持并发读写，我们这里使用cas(q->head, head, next)来完成替换。假设线程１将要执行到这一步的时候被换出去， head, next 分别指向 node1, node2，什么事都还没发生。

另一个线程 2，开始执行dequeue，然后它成功把 node1, node2, node3 都pop 出去了，此时队列为空！

然后又切换到线程 3，它要执行 enqueue 操作，此时它会在 27 行分配一个节点，坏事来了，如果十分巧合的情况下，它分配得到了线程 1 所 free 掉的 node1，enqueue 之后，队列成了如下样子：

node1-> NULL

线程 3 执行完后，如果恰好又切换回线程 1，此时，对线程 1 来说，它无法分辨出q->head 和head的区别（因为线程3恰好分配了和原来内存地址相同的节点）, 因此 cas 会成功！但是 next 呢？next 指向的是被释放掉的 node2! 严重问题！

这个就是无锁世界里所谓的 ABA 问题，这个问题就是我为什么最开始想尝试用数组来实现无锁操作的原因之二，而且，ABA 问题不容易解决。

