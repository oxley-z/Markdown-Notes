---
author: 胡胡子的
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU1NzkxNTQ2OA==&mid=2247488720&idx=1&sn=86b24f752536a9f357553a28902dad1a&chksm=fd466c58227c6c87a8f01d079fe04c2ddb54746c89fea2d6fc061d3efbe46d384f2f69a888dd&mpshare=1&scene=1&srcid=0610UtHO1hE6JKYHIGzzbM1P&sharer_shareinfo=a9aa87c969027f2c9fc68d744de4ee07&sharer_shareinfo_first=a9aa87c969027f2c9fc68d744de4ee07#rd
saved: 2026-06-10 12:00:45
tags:
  - 笔记同步助手
id: 2f73e89d-c69c-4abc-874c-9786fa9dc7e0
---

公众号名称：linux高性能网络

作者名称：胡胡子的

发布时间：2026-06-10 11:33

# 并发与队列模型

在 DPDK 以及各类高并发底层框架中，rte\_ring（无锁环形队列） 是解决多核之间数据传输瓶颈的“大杀器”。

传统的队列在多线程并发访问时，必须使用互斥锁（Mutex）或自旋锁（Spinlock）。但锁是高性能的杀手：加锁会导致线程挂起、上下文切换 ，自旋锁会导致 CPU 核心空转，在数千万 PPS（每秒数据包数）的场景下，这种开销是无法接受的。

rte\_ring 能够做到完全不需要互斥锁就能实现超高性能的并发，其核心秘密在于：精妙的环形数组指针设计、硬件级的原子指令（CAS） 以及对现代 CPU 缓存一致性（Cache Line）的极致压榨。

![[../../../../images/7b855ba3ddc63774b8cfe58d96a98866_MD5.jpg]]

# 2.1 rte\_ring

rte\_ring 底层是一个固定大小的数组（容量通常是 2 的幂次方，便于通过位与 & 操作代替取模 %）。它没有使用传统的 head 和 tail，而是维护了两对独立的 32 位无符号递增序列号（指针）：

![[../../../../images/53eaca5c8eaab96ab22e4f85aa8613c1_MD5.jpg]]

核心精髓：这四个指针都是永远向后递增的（由于是 32 位无符号整数，溢出后会自动回绕到 0，刚好完美契合环形数组）。数组的实际槽位索引（idx）是通过 pointer & (size - 1) 计算出来的。

# 2.2 SPSC

在 SPSC 模式下，只有一个线程写，一个线程读。这是最纯粹、性能最高的场景。由于根本没有竞争，它甚至不需要任何复杂的原子指令，只需要保证内存可见性（Memory Order）。

入队（Enqueue）操作：

1.  生产者读取 prod.head 和 cons.tail，计算出当前队列里还有多少剩余空间。
    
2.  空间足够，生产者直接把数据写入 prod.head 指向的槽位。
    
3.  写入完成后，生产者直接执行 prod.tail = prod.tail + 1（在 C 语言中配合内存屏障/Release 语义，确保数据先落盘，指针后更新）。
    
4.  此时，消费者看到了 prod.tail 往前推进了，就知道有新数据可以读了。
    

在这个过程中，生产者只修改 prod 指针，消费者只修改 cons 指针，双方互不干扰，完全不需要锁。

# 2.3 MPMC

当有多个线程同时入队（Multi-Producer）时，真正的挑战来了：如何保证两个生产者不会把数据写到同一个槽位里？

rte\_ring 采用了基于硬件级原子指令 CAS（Compare-And-Swap ，比较并交换） 的乐观锁策略。我们以两个生产者（A 和 B）同时尝试入队 1 个对象为例：

第一步：抢占位置（Local Move）

-   此时队列的 prod.head = prod.tail = 1000。
    
-   生产者 A 和 生产者 B 同时读到了 prod.head = 1000。
    
-   它们各自在本地计算：如果我抢成功了，下一次的 head 应该是 1001。
    

第二步：硬件决斗（CAS 比较并交换）

-   两个生产者同时调用 CPU 的 CAS 原子指令（在 x86 上是 cmpxchg），尝试把全局的 prod.head 从 1000 改为 1001。
    
-   决斗结果：硬件保证只能有一个人成功。
    

假设 生产者 A 成功了：全局 prod.head 瞬间变成 1001。A 欢天喜地，拿到了槽位 1000 的“写字楼入驻权”。

生产者 B 失败了：CAS 返回失败。B 发现全局 prod.head 已经变成 1001 了，它不会气馁，而是立即原地自旋重试（Loop）。它重新读取当前的全局 prod.head (1001)，再次发起 CAS 尝试将其改为 1002。这一次它成功了，拿到了槽位 1001 的写入权

第三步：写入数据（Write Data ）

此时，全局 prod.head 已经被推进到了 1002。

生产者 A 开始往槽位 1000 写数据，生产者 B 开始往槽位 1001 写数据。它们在各自的独立槽位上写，互不干扰！

第四步：安全提交（Wait & Move Tail）

这是 rte\_ring 最经典、最精妙的一步。

数据写完后，A 和 B 都需要更新 prod.tail，因为只有 prod.tail 推进了，消费者才能看到并读取数据。

规则：必须严格按照抢到位置的顺序来更新 tail。

-   生产者 B 虽然先写完了（槽位1001），但它回头一看：当前的全局 prod.tail 还是 1000。B 知道：“哦，在我前面的 A 还没写完呢，我不能越过它提交。” 于是 B 就在原地等待（自旋）。
    
-   随后，生产者 A 写完了（槽位1000），它发现全局 prod.tail == 1000 正好满足。A 直接把 prod.tail 推进到 1001。
    
-   正在等待的 B 瞬间看到 prod.tail 变成 1001 了，它立马接力，把 prod.tail 推进到 1002。入队圆满结束！
    

问：为什么这样能做到“高并发”？

可能会问：多生产者模式下，失败的线程不是还要“自旋重试”吗？这和加锁有什么区别？

区别在于锁的粒度和层次完全不同：

1\. 避免了内核态挂起（零上下文切换）：

传统互斥锁如果拿不到锁，线程会被操作系统挂起并切出 CPU，触发高昂的内核上下文切换开销（微秒级）。而 rte\_ring 的自旋完全发生在用户态，利用硬件指令决胜负（纳秒级）。

2\. 极短的临界区：

传统的锁，锁住的是“从寻找空位 -> 写入数据 -> 更新指针”的全过程。

而 rte\_ring 用 CAS 锁住的，仅仅是“抢占 head 指针”的那一个微小的瞬间（一步 CPU 指令）。一旦抢到了位置，多线程在写入实际数据（这通常是最耗时的）时，是完全并行、互不干扰的！

3\. 缓存行对齐（防伪共享）：

在结构体定义中，DPDK 使用了 \_\_rte\_cache\_aligned 关键字。生产者控制的 prod 指针和消费者控制的 cons 指针被强制隔开在不同的 CPU Cache Line（缓存行）中。

这样，当生产者疯狂修改 prod 指针时，永远不会导致消费者的 L1/L2 缓存中的 cons 指针失效（避免了经典的 False Sharing 伪共享 问题），让多核 CPU 的独立缓存发挥出最大威力。

对于"避免了内核态挂起"这一特性提问：那这样系统不是一个核只能绑定一个线程了，还能跑多任务吗？

答：是的，在这种极致追求性能的场景下，一个 CPU 核心通常只绑定一个线程（即单核单线程），并且这个核心不再跑传统意义上的“多任务”。

这种设计在 DPDK 中被称为 轮询模式（PMD, Poll Mode Driver） 和 核绑定（CPU Affinity）。

![[../../../../images/57986a4bda60175fe58752e615d9aeb3_MD5.jpg]]

![[../../../../images/4fa00494aafb51afca4b8585672fdb49_MD5.jpg]]

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/95f89f2d_1781064043323?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU1NzkxNTQ2OA%3D%3D%26mid%3D2247488720%26idx%3D1%26sn%3D86b24f752536a9f357553a28902dad1a%26chksm%3Dfd466c58227c6c87a8f01d079fe04c2ddb54746c89fea2d6fc061d3efbe46d384f2f69a888dd%26mpshare%3D1%26scene%3D1%26srcid%3D0610UtHO1hE6JKYHIGzzbM1P%26sharer_shareinfo%3Da9aa87c969027f2c9fc68d744de4ee07%26sharer_shareinfo_first%3Da9aa87c969027f2c9fc68d744de4ee07%23rd&s=obsidian)