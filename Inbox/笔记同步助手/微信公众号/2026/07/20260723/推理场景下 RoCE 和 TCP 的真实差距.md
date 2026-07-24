---
author: 疯狂的兔子tommy
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk0MzY3NTQzOQ==&mid=2247488910&idx=1&sn=7c322f2014f06bd08b5fa1aff24b42a7&chksm=c26fe91d4bbccb785083c9645439e6f91b8b17dfd81246b2335e27c3b5cb14aa23cb7aeea8e7&mpshare=1&scene=1&srcid=0723bldu9Ff8pKGzMRsK0gnA&sharer_shareinfo=d247d82e2a4ad51ca13207c4fb36f23e&sharer_shareinfo_first=d247d82e2a4ad51ca13207c4fb36f23e#rd
saved: 2026-07-23 08:37:56
tags:
  - 笔记同步助手
id: b194728b-2b66-42c5-811e-726698a96a4f
---

公众号名称：算力网络架构手记

作者名称：疯狂的兔子tommy

发布时间：2026-07-23 08:27

这里是「算力网络架构手记」北京

  

-   **专注 AI 集群网络架构与性能优化**
    
-   **深入 GPU×RoCE×NCCL×K8s 跨层瓶颈**
    
-   **只拆真实问题，不写概念科普**
    

---

  

👇 试听内容

![[Inbox/笔记同步助手/微信公众号/2026/07/images/e64daaf443994cf14af78c141757622c_MD5.jpg]]

> 📹 视频内容（上图为封面），请前往原文观看：[在公众号原文中观看](https://mp.weixin.qq.com/s?__biz=Mzk0MzY3NTQzOQ==&mid=2247488910&idx=1&sn=7c322f2014f06bd08b5fa1aff24b42a7&chksm=c26fe91d4bbccb785083c9645439e6f91b8b17dfd81246b2335e27c3b5cb14aa23cb7aeea8e7&mpshare=1&scene=1&srcid=0723bldu9Ff8pKGzMRsK0gnA&sharer_shareinfo=d247d82e2a4ad51ca13207c4fb36f23e&sharer_shareinfo_first=d247d82e2a4ad51ca13207c4fb36f23e#rd)

👉 AI 训练网络全路径拆解 → **私信：****AI网络**

👉 AI 推理网络全路径拆解 → **私信：****推理**

**👉** AI 网络架构工程指南手册 → **私信：****工程指南**

**👉 AI 算力网络架构系统（真机实验环境） → 私信：系统**

👉 日常工作1对1答疑 → **私信：****答疑**

---

00

RoCE or TCP？

  

> 推理到底该用 RoCE，还是 TCP？

这个问题听起来像是在问“协议选型”。

但真实工程里，它不是一个简单的协议问题。

因为推理服务里根本不是一种流量。

有的是用户请求。  
有的是调度控制。  
有的是 KV Cache 迁移。  
有的是 Decode 阶段 TP 通信。  
有的是后台缓存回填。  
有的是 P/D 分离后的状态搬运。

这些流量对网络的要求完全不同。

所以，如果你只问：

> RoCE 快，还是 TCP 快？

这个问题就已经问偏了。

更准确的问题应该是：

> **哪一段推理链路，值得为低延迟、低 CPU 开销、GPU 直通和 RDMA 数据面付出 RoCE 的复杂度？**

这才是 RoCE 和 TCP 在推理场景下的真实差距。

01

RoCE 和 TCP 的差距，在“谁更适合热路径”

TCP 能传。  
RoCE 也能传。

但它们适合的路径不一样。

在推理服务里，TCP 更适合：

-   API 请求接入
    
-   网关到调度器
    
-   控制面通信
    
-   管理流量
    
-   普通服务间 RPC
    
-   对延迟极致要求不高的数据流
    

  

RoCE 更适合：

-   多 GPU / 多节点 TP 通信
    
-   Decode 高频同步
    
-   P/D 分离下的 KV Cache 迁移
    
-   GPU 到 GPU 的大块状态搬运
    
-   对 CPU 开销、尾延迟、传输效率非常敏感的东西向热路径
    

  

所以不要把 TCP 和 RoCE 看成“谁替代谁”。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/5f73428847b2b23ef29b335cc7abc189_MD5.jpg]]

更真实的关系是：

> **TCP 是推理服务的通用网络底座。**

> **RoCE 是推理热路径上的高性能数据通道。**

  

因为很多系统不是“全 TCP”或“全 RoCE”。

而是：

-   南北向用户访问走 TCP
    
-   控制面走 TCP / HTTP / gRPC
    
-   东西向热数据走 RoCE
    
-   NCCL / RDMA 数据面走 RoCE
    
-   后台冷数据视情况走 TCP 或 RoCE
    

  

真正成熟的推理网络，不是盲目追求单一协议。

而是先分清：哪条路径是热路径，哪条路径是普通路径。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/17dfee61e185a82d0d75d44206ccb61e_MD5.jpg]]

02

TCP 的优势

TCP 在推理服务里非常重要。

甚至可以说，没有 TCP，大多数推理服务根本没法完整运行。

用户访问 API，用 TCP。  
HTTP / gRPC / WebSocket / SSE，多数跑在 TCP 上。  
服务发现、调度、控制面、监控，也大量依赖 TCP。

TCP 的优势很明确：

-   成熟
    
-   通用
    
-   可靠
    
-   运维简单
    
-   跨网络环境适应性强
    
-   和云原生体系结合好
    
-   对应用开发友好
    

  

在南北向访问上，TCP 非常合适。

用户请求不是直接把 KV Cache 发给 GPU。  
用户发的是文本、请求参数、上下文 ID、控制信息。

这些流量更关心：

-   连接稳定
    
-   请求可达
    
-   超时控制
    
-   重传可靠
    
-   服务治理
    
-   限流熔断
    
-   网关能力
    

这些正是 TCP 和上层服务框架擅长的地方。

所以，TCP 在推理系统里的角色不是“低端替代品”。

它是服务化架构的基础。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/385266b3becd9fd3225c5d4728e2a52a_MD5.jpg]]

  

> **TCP 很适合推理服务的外层。**

> **但它不一定适合推理内部最热、最重、最怕抖的 GPU 数据路径。**

  

问题出在这里：

当你把 TCP 用到推理热路径时，它的代价就开始明显。

比如：

-   经过内核网络栈
    
-   socket buffer
    
-   CPU 参与更多
    
-   copy / 协议栈开销更高
    
-   尾延迟更容易受系统调度影响
    
-   多流并发时 CPU 和内核路径压力更明显
    

对于普通 RPC，这些可以接受。

但对于 Decode TP 同步、KV Cache 大块迁移、GPU-GPU 状态搬运，这些开销就会变得敏感。

尤其是 Decode 阶段。

每个 token 都可能有同步点。  
一次小抖动，就可能拖慢当前 token。  
当前 token 慢，后面 token 全部顺延。

这时候，TCP 的“通用性”并不等于“热路径最优”。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/c0e2512e9b9096d2ac3c54aa0bef118e_MD5.jpg]]

03

RoCE 的优势

RoCE 的核心价值，不是“比 TCP 更高级”。

而是它改变了数据搬运方式。

RoCE 基于 RDMA 思路，可以让网卡在完成必要建连、内存注册、队列管理之后，通过 DMA 直接读写注册内存。

在合适条件下，它可以减少：

-   CPU 数据搬运参与
    
-   内核协议栈开销
    
-   多次 copy
    
-   高并发下的 CPU 抖动
    
-   热路径上的软件处理延迟
    

  

如果配合 GPUDirect RDMA，GPU 显存和网卡之间的数据路径会更直接。

这对于推理内部热路径非常关键。

比如：

### 1）KV Cache 迁移

P/D 分离下，Prefill 生成 KV Cache 后，需要送到 Decode。

这不是普通文本。

这是模型上下文状态。

KV 一旦跨节点迁移，越接近 GPU-to-GPU 高效搬运，越有价值。

RoCE 的优势就在这里更明显。

### 2）Decode 阶段 TP 通信

Decode 是逐 token 生成。

如果模型跨多 GPU / 多节点 TP，每个 token 都可能触发同步通信。

这类通信对尾延迟非常敏感。

RoCE 可以减少 CPU 和内核路径参与，使数据面更接近高性能网络直通。

### 3）NCCL 数据面

NCCL 在多 GPU 通信中通常更偏向使用 IB / RoCE 这类 RDMA 能力。

因为 collective 通信最怕：

-   某个 rank 慢
    
-   某条路径抖
    
-   某次同步被拖住
    

RoCE 能让高性能数据面更稳定地支撑 NCCL。

  

> **RoCE 的价值，不是让所有推理流量都变快。**

> **而是让 GPU 热路径上的状态搬运和同步通信少绕路、少耗 CPU、少抖动。**

  

![[Inbox/笔记同步助手/微信公众号/2026/07/images/42e11f52dd5d1bb594f7e567efd7a0fa_MD5.jpg]]

当然，RoCE 不是免费午餐。

它需要：

-   正确的 NIC / 驱动 / 固件
    
-   RDMA 栈
    
-   GID / MTU / 路由
    
-   PFC / ECN / DCQCN
    
-   DSCP / priority / TC 映射
    
-   交换机 buffer 与队列策略
    
-   主机 GPU-NIC 亲和
    
-   NCCL 选路对齐
    

这些复杂度，是 RoCE 的代价。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/90abca20ba0bb5ad18667d379f9a7490_MD5.jpg]]

所以 RoCE 的选择逻辑不是：

> RoCE 快，所以全都用 RoCE。

而是：

> 哪些路径值得为 RoCE 的复杂度买单？

04

真实差距一：延迟差距，不只看平均值，要看 p99 和同步等待

在推理场景下，RoCE 和 TCP 的差距，不能只看平均延迟。

更要看：

-   p95
    
-   p99
    
-   p999
    
-   token latency
    
-   TTFT
    
-   Decode 抖动
    
-   NCCL collective time
    

  

为什么？

因为推理是在线服务。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/7c10a335f972e0b77d62441e42854a87_MD5.jpg]]

平均值好看，用户不一定满意。  
只要 p99 难看，用户就会觉得系统不稳。

尤其 Decode 阶段是逐 token 串行依赖。

如果一次通信慢了：

-   当前 token 慢
    
-   后续 token 顺延
    
-   流式输出卡顿
    
-   用户感知明显
    

  

在 TCP 路径下，热数据可能更容易受到：

-   内核调度
    
-   CPU 抢占
    
-   socket buffer 排队
    
-   软件协议栈处理
    
-   多连接竞争
    

这些因素影响尾延迟。

RoCE 的优势在于：它让热数据路径更直接，减少 CPU 和内核路径抖动对数据面的影响。

这不是说 RoCE 一定永远低延迟。

如果 RoCE QoS 配错，PFC 频繁 pause，ECN 不生效，队列堆深，RoCE 一样会抖。

但在配置正确、路径干净、QoS 闭环打通的情况下，RoCE 更适合承载对 p99 敏感的东西向热通信。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/a9dce277bd579f983c6ee85772564018_MD5.jpg]]

  

> **推理里真正重要的，不是平均通信有多快。**

> **而是最慢那 1% 的通信会不会拖住 token。**

05

真实差距二：CPU 开销差距，在高并发推理里会变成系统稳定性差距

很多人比较 RoCE 和 TCP，只看网络带宽。

这不够。

在推理服务里，CPU 也很关键。

因为 CPU 不只跑网络。

它还要负责：

-   调度器
    
-   tokenizer / detokenizer
    
-   请求编排
    
-   batch 组织
    
-   KV metadata 管理
    
-   日志与监控
    
-   服务框架
    
-   连接管理
    
-   推理运行时控制逻辑
    

如果大量热路径数据都通过 TCP 走内核协议栈，CPU 压力会增加。

这会带来一种隐蔽后果：

**不是链路先满，而是 CPU 先抖。**

CPU 一抖，推理服务就可能出现：

-   调度延迟增加
    
-   batch 组织变慢
    
-   token 输出节奏变差
    
-   网络收发不稳定
    
-   p99 被拉长
    

RoCE 的价值在这里也很明显。

它把高性能数据搬运更多交给 NIC / RDMA 数据面，减少 CPU 在热路径 payload 搬运上的参与。

这样 CPU 可以更多用于：

-   调度
    
-   控制
    
-   服务逻辑
    
-   请求管理
    
-   metadata 处理
    

这在高并发推理里很重要。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8b35e616b47dc0ac26c7dcf70a42a52b_MD5.jpg]]

  

> **TCP 的代价不只在网络上。**

> **它还可能把热路径数据搬运压力转嫁给 CPU，最终影响调度和 p99。**

  

当然，也不能反过来神化 RoCE。

RoCE 也需要 CPU：

-   QP / CQ 管理
    
-   建连控制
    
-   CQ polling
    
-   内存注册
    
-   错误处理
    
-   NCCL / runtime 协调
    

  

但它的核心优势是：数据 payload 搬运路径更轻。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/530429a82df87149179a5113a55b73a9_MD5.jpg]]

尤其在 GPU-GPU 大块状态和高频同步通信中，这个差距会更明显。

06

真实差距三：带宽差距，在 KV Cache 迁移里最容易被放大

推理场景下，用户请求本身通常不是大流量。

真正大的，往往是内部状态。

最典型就是 KV Cache。

P/D 分离后，Prefill 生成 KV Cache。  
Decode 要使用 KV Cache。  
如果 P 和 D 不在同一个节点，就要跨网络搬。

这时 RoCE 和 TCP 的差距会变得更明显。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/81cd9f276fa32cef83cf3b0e5c642b67_MD5.jpg]]

因为 KV Cache 迁移具有几个特点：

-   数据块较大
    
-   通常在首 token 前
    
-   延迟敏感
    
-   可能多个请求同时发生
    
-   目的端容易形成 Incast
    
-   源端和目的端都可能出现队列压力
    

如果使用 TCP 迁移，路径通常会引入更多 CPU 和内核开销。

如果并发多、KV 大、请求随机，TCP 路径可能更容易出现：

-   CPU 忙
    
-   socket buffer 排队
    
-   拷贝开销上升
    
-   迁移时间变长
    
-   TTFT 被拉高
    

RoCE 更适合这种东西向大块状态搬运。

尤其在 GPU 直连、RDMA 数据面、网络 QoS 打通后，可以更好地支撑 KV 迁移。

但注意：

RoCE 也不是只要带宽大就够。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/762c5171f21698e09cfd057a62ce0d73_MD5.jpg]]

如果多个 P 节点同时把 KV 打向少数 D 节点，RoCE 一样会遇到：

-   Leaf 队列升高
    
-   ECN marked 增加
    
-   DCQCN 降速
    
-   PFC pause 风险
    
-   D 侧 NIC 接不住
    

所以差距的关键不是“RoCE 不会拥塞”。

而是：

**在同样需要搬大块 KV 的前提下，RoCE 的数据面效率和 CPU 开销更适合热路径。**

  

> **KV Cache 迁移不是普通文件传输。**

> **它挡在 Decode 之前，慢了就是 TTFT 慢。**

  

07

真实差距四：NCCL / TP 通信里，RoCE 更接近主流高性能路径

大模型推理如果使用 TP，尤其是跨节点 TP，就绕不开 collective 通信。

可能包括：

-   AllReduce
    
-   AllGather
    
-   ReduceScatter
    
-   P2P
    
-   其他模型并行通信
    

NCCL 在这些通信里通常会优先利用高性能网络能力。

在可用时，IB / RoCE 这类 RDMA 路径通常比 TCP socket 路径更适合做数据面。

NCCL 通信关心的是：

-   吞吐
    
-   延迟
    
-   rank 间同步稳定性
    
-   多 GPU 并发
    
-   多通道传输
    
-   GPU-NIC 拓扑亲和
    

  

TCP socket 能跑 NCCL，但它更像兜底路径或通用路径。

在高性能推理或训练场景里，如果 NCCL 只能走 TCP，通常意味着：

-   性能上限下降
    
-   CPU 参与增加
    
-   多节点通信延迟更高
    
-   p99 更难控制
    
-   大规模扩展更吃力
    

尤其 Decode 阶段。

每个 token 都可能经过 TP 同步。  
如果这个同步跑在 TCP 上，哪怕平均还能接受，尾延迟也更容易被放大。

RoCE 的价值是：

让 NCCL 的数据面更接近高性能 RDMA 传输。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/5ac2e7d14181ff75cf38894fba97ad21_MD5.jpg]]

  

> **TCP 能作为 NCCL 的通路。**

> **但 RoCE 更适合承载 NCCL 的高性能数据面。**

  

不过这里也要强调：RoCE 用不好，NCCL 一样慢。

典型问题包括：

-   NCCL 没选到正确 HCA
    
-   GID index 错
    
-   RoCE QoS 没闭环
    
-   PFC / ECN 映射错
    
-   多轨没用起来
    
-   GPU-NIC 跨 NUMA
    
-   交换机队列不稳
    

  

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ae112c161749bb6cd9daa5313b312dcb_MD5.jpg]]

所以 RoCE 和 NCCL 的组合，不是“自动快”。

它是：潜力更高，但要求更严格。

08

真实差距五：可靠性与运维复杂度

如果只看性能，很容易倾向 RoCE。

但真实工程不能只看性能。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/180f42d40909a385188f36e744d6819b_MD5.jpg]]

TCP 的优势是：

-   对网络丢包更宽容
    
-   拥塞控制成熟
    
-   运维经验丰富
    
-   跨三层网络简单
    
-   故障排查工具多
    
-   服务框架原生支持
    
-   对交换机配置要求低
    

  

RoCE 的挑战是：

-   对丢包更敏感
    
-   需要 PFC / ECN / DCQCN 配合
    
-   需要 QoS 映射正确
    
-   buffer 水线复杂
    
-   多轨和 GID 规划复杂
    
-   主机与交换机必须一起调
    
-   一处配置错，可能表现成全局训练/推理抖动
    

  

所以 RoCE 的真实性能差距，建立在一个前提上：

**你能把 RoCE 环境配置对。**

如果配置不对，RoCE 可能比 TCP 更难排障。

比如：

-   TCP 慢了，你还能靠重传和拥塞控制兜住
    
-   RoCE 队列和 PFC 错了，可能直接造成 pause 扩散、尾延迟抖动、NCCL 卡顿
    

  

所以选型时要看团队能力。

不是所有推理流量都值得上 RoCE。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/9bdfa3f4a7685b55b40f3c0906aa6e15_MD5.jpg]]

  

> **TCP 的优势是工程宽容度。**

> **RoCE 的优势是性能上限。**

> **你要的是省心，还是要的是热路径极限性能？这两件事不能混为一谈。**

  

09

推理场景下，哪些流量应该优先 RoCE？哪些可以继续 TCP？

这里给一个比较工程化的判断。

  

### 优先考虑 RoCE 的流量

第一类：Decode 阶段 TP 通信

因为它在每个 token 的关键路径上。  
它怕尾延迟，怕 rank 慢，怕同步等待。

第二类：P/D 分离下的 KV Cache 迁移

因为它挡在 Decode 开始之前。  
慢了直接拉高 TTFT。

第三类：GPU-GPU 状态搬运

尤其是跨节点、跨多卡、需要高吞吐和低 CPU 开销的数据路径。

第四类：NCCL collective 数据面

尤其是多节点模型并行、推理并发高、需要稳定 p99 的场景。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/5333e15e3da859faf49d4d1772c4fa67_MD5.jpg]]

### 可以继续使用 TCP 的流量

  

第一类：用户 API 请求和流式返回

HTTP / gRPC / SSE / WebSocket 继续用 TCP 非常合理。

第二类：调度控制面

调度器、服务发现、健康检查、控制 RPC 通常不需要 RoCE。

第三类：管理监控流量

日志、指标、元数据同步、运维流量，通常 TCP 更合适。

第四类：冷数据后台传输

不在当前 token 关键路径上的冷数据，可以根据成本和复杂度选择 TCP 或低优先级网络路径。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/666ff992067620d781af7606d67e1a1b_MD5.jpg]]

  

> **RoCE 应该优先给热路径。**

> **TCP 应该继续承担服务外层、控制面和通用通信。**

  

10

一个实用判断框架：别问“RoCE 还是 TCP”，先问这 5 个问题

选 RoCE 还是 TCP，不要靠信仰。

先问 5 个问题。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/4b658d5fd50c0f241bc4c19366921d7e_MD5.jpg]]

  

### 问题 1：这条流量在不在 token 关键路径上？

如果在，比如 Decode TP 同步、KV 到位前等待，那 RoCE 价值更高。

如果不在，比如后台日志、管理流量，TCP 足够。

### 问题 2：这条流量是不是大块状态搬运？

如果是 KV Cache、GPU 状态、模型并行中间数据，RoCE 更有意义。

如果只是小控制 RPC，TCP 更合理。

### 问题 3：这条流量对 CPU 开销敏感吗？

如果 CPU 已经承担调度、tokenizer、服务框架，热路径数据再压 CPU，会影响系统稳定性。

此时 RoCE 更有价值。

### 问题 4：这条流量是否容易形成 Incast？

如果大量 P 节点同时向少数 D 节点发 KV，即使用 RoCE，也必须配合调度、限速、ECN/PFC。

RoCE 不是免疫拥塞。

### 问题 5：你的团队能否运维好 RoCE 闭环？

如果不能保证：

-   QoS 映射
    
-   ECN/PFC
    
-   DCQCN
    
-   buffer
    
-   GID
    
-   NCCL 选路
    
-   主机拓扑
    

那 RoCE 的理论优势可能兑现不了。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/0f3e5e79c0cef9650700d75a04673c80_MD5.jpg]]

  

> **协议选型不是问“哪个更快”。**

> **而是问“这条流量是否值得为更高性能付出更高复杂度”。**

  

11

文末升维

  

> **TCP 适合推理服务的通用连接和控制面。**

> **RoCE 适合推理系统内部的 GPU 热数据面。**

  

也就是说：

-   TCP 胜在通用、成熟、可靠、好运维
    
-   RoCE 胜在低 CPU 开销、低延迟、高吞吐、适合 RDMA 数据面
    
-   TCP 更适合南北向和控制面
    
-   RoCE 更适合东西向热路径
    
-   TCP 可以跑很多流量，但热路径可能吃 CPU 和尾延迟
    
-   RoCE 可以带来更高性能，但前提是 QoS、路由、GID、NCCL、主机拓扑全部闭环
    

  

所以 RoCE 和 TCP 的真实差距，不是教科书里一句：

> RDMA 比 TCP 快。

而是工程里的这句话：

> 当推理链路进入 GPU 热路径、KV 状态搬运、TP 同步和 p99 敏感区时，RoCE 的价值才真正拉开。

  

```
最合理的结论不是：
全部上 RoCE。

也不是：
TCP 足够了。

而是：
- TCP 管服务。
- RoCE 管热路径。

这才是推理场景下 RoCE 和 TCP 最真实的差距。
```

  

---

  

👉 AI 训练网络全路径拆解 → **私信：****AI网络**

👉 AI 推理网络全路径拆解 → **私信：****推理**

**👉** AI 网络架构工程指南手册 → **私信：****工程指南**

**👉 AI 算力网络架构系统（真机实验环境） → 私信：系统**

👉 日常工作1对1答疑 → **私信：****答疑**

  

---

  

> **如果你也在做 AI 集群架构**
> 
> **欢迎关注「算力网络架构手记」**
> 
> **长期拆解真实算力网络问题**

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/aad99da2_1784767075449?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk0MzY3NTQzOQ%3D%3D%26mid%3D2247488910%26idx%3D1%26sn%3D7c322f2014f06bd08b5fa1aff24b42a7%26chksm%3Dc26fe91d4bbccb785083c9645439e6f91b8b17dfd81246b2335e27c3b5cb14aa23cb7aeea8e7%26mpshare%3D1%26scene%3D1%26srcid%3D0723bldu9Ff8pKGzMRsK0gnA%26sharer_shareinfo%3Dd247d82e2a4ad51ca13207c4fb36f23e%26sharer_shareinfo_first%3Dd247d82e2a4ad51ca13207c4fb36f23e%23rd&s=obsidian)