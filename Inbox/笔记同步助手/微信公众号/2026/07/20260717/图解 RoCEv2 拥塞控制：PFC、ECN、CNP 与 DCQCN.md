---
author: 李家旺
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUyNTY0NzI4OA==&mid=2247484419&idx=1&sn=1ab7580fd4928a74518c871acda763a3&chksm=fb63fa743bba4918bc314ad5807faa2c1a881846ec8da2e44e742576a6fbe253adb71686a724&mpshare=1&scene=1&srcid=0717ouTBOqGO2ZJt76xkWRKe&sharer_shareinfo=fb64d4f9137c4ae3f52e75047f889635&sharer_shareinfo_first=fb64d4f9137c4ae3f52e75047f889635#rd
saved: 2026-07-17 08:59:29
tags:
  - 笔记同步助手
id: dd6130c8-3019-4077-8afe-febe6187296c
---

公众号名称：AI算力学习手记

作者名称：李家旺

发布时间：2026-07-17 00:00

> 本文介绍 PFC、ECN、CNP 与 DCQCN 的工作机制、协作流程，以及训练吞吐下降时的排查方法。

在 AI 集群中，GPU 算力并不总是性能瓶颈。一次 AllReduce、AllGather 或 MoE All-to-All，只要网络中的某个出口发生拥塞，就可能让大量 GPU 等待通信，最终表现为：

-   NCCL 带宽忽高忽低；
    

-   单个训练 Step 偶发变慢；
    

-   平均吞吐尚可，但 P99/P999 延迟明显升高；
    

-   链路没有 Down，光模块也没有误码，作业却像“网络不稳定”；
    

-   严重时出现 RDMA 重传、QP 超时或训练任务失败。
    

RoCEv2 使用 UDP/IP 承载 RDMA，可以运行在三层 Leaf-Spine 网络中，但它对拥塞和丢包非常敏感。生产网络通常组合使用三种不同层次的机制：ECN 标记拥塞，DCQCN 调整源端 RNIC 的发送速率，PFC 在队列接近危险水位时逐跳暂停流量。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/13ff9b6d86a2da6ad1acf966d62ce86e_MD5.jpg]]

01

为什么 AI 流量特别容易造成拥塞？

1\. AI 流量不是平稳的小水流

大模型训练会周期性进入集合通信阶段。多个 GPU 可能在非常接近的时间同时发送数据，流量具有三个典型特征：

-   突发性强：计算阶段结束后，许多 GPU 同时开始通信；
    

-   大流多：梯度、激活值和专家 Token 会形成持续的高带宽流；
    

-   同步性强：一个 Rank 变慢，其他 Rank 也可能在同步点等待。
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/905c61f1be1b8714584200f15a1cc93e_MD5.jpg]]

2\. Incast 会快速打满一个出口队列

假设 8 台服务器各以 100 Gbit/s 向同一个 100 Gbit/s 出口发送：

```
8 × 100 Gbit/s 输入
↓
同一个交换机出口
↓
100 Gbit/s
```

交换机瞬间收到的流量远高于出口能力，多出来的报文只能进入队列。即使平均带宽没有超过网络容量，微秒级的同步突发也可能让缓存快速上涨。

队列增长通常经历以下过程：

```
正常水位
↓
达到 ECN 门限：开始标记拥塞
↓
达到 PFC XOFF 门限：暂停直接上游的指定优先级
↓
缓存耗尽：丢包
```

拥塞控制需要在缓存溢出前抑制队列增长，不要求队列始终保持为空。

02

三种机制分别解决什么问题？

| 机制 | 工作范围 | 谁发现/触发 | 谁采取动作 | 主要作用 |
| --- | --- | --- | --- | --- |
| PFC | 二层、逐跳 | 下游端口或交换机 | 直接相邻的上游设备 | 暂停某个优先级，尽量避免丢包 |
| ECN | 三层、端到端信号的一部分 | 拥塞交换机 | 给经过的 IP 报文标记 CE | 在丢包前暴露拥塞 |
| CNP | 端到端反馈报文 | 接收端 RNIC | 通知发送端 RNIC | 把 CE 信号带回流量源 |
| DCQCN | 端到端速率控制 | 发送端 RNIC | 降低或恢复对应流的发送速率 | 从源头消除持续拥塞 |

RoCEv2 数据报文封装在 UDP/IP 中，标准目标端口通常是 4791。排障时可以在交换机 ACL、流量计数器或镜像抓包中使用 udp dst port 4791 快速定位 RoCEv2 流量。该端口可能因设备或历史配置而不同；RNIC 硬件卸载也可能使主机上的 tcpdump 看不到全部 RDMA 报文，因此不能只凭主机抓包判断流量是否存在。

持续拥塞需要通过 DCQCN 降低源端发送速率。PFC 只能暂时阻止报文继续进入拥塞队列，无法增加出口带宽。

03

PFC：按优先级执行的逐跳暂停

PFC 全称 Priority-based Flow Control，来自数据中心桥接（DCB）体系。它与传统的全链路 Pause 最大的区别是：PFC 可以只暂停一个或若干个 802.1p 优先级，而不暂停整条链路上的所有流量。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8c20f4c1e32ae0b38db3cec6a61379b3_MD5.jpg]]

1\. PFC 如何工作？

当交换机发现某个无损队列达到 XOFF 门限时，会向直接相连的上游设备发送 PFC Pause 帧：

![[Inbox/笔记同步助手/微信公众号/2026/07/images/63e6a2cd230e6cf742d112d3182ae87d_MD5.jpg]]

上游收到 Pause 后，暂时停止发送对应优先级的流量。队列下降到 XON 恢复条件后，下游再允许上游继续发送。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/e9957cc6e0bc2783703fb2c03064c3fb_MD5.jpg]]

PFC 的几个关键词：

-   逐跳：Pause 只作用于直连邻居，不会直接通知最初的发送端；
    

-   按优先级：可以只暂停 RoCE 所在优先级；
    

-   响应快：通常由交换机 ASIC 和网卡硬件完成；
    

-   只是兜底：它没有让发送源理解“为什么拥塞”，也没有从根本上降低业务注入速率。
    

2\. Buffer 是什么？

这里的 Buffer（缓存），是交换机 ASIC 临时存放报文的高速存储空间，不是服务器内存，也不是磁盘缓存。当多个入口同时向同一个出口发送，报文到达速度暂时超过出口转发速度时，来不及立即发出的报文会先进入队列 Buffer：

```
多个入口同时到达
↓
交换机端口 / TC 队列 Buffer 暂存报文
↓
出口按线路速率逐个转发
```

Buffer 的作用是吸收短时突发和速率差，但容量有限。队列持续增长时，会依次触发 ECN 标记、PFC Pause，最终在缓存耗尽后丢包。交换机可能采用入口缓存、出口缓存、共享缓存和队列专用缓存等不同架构，因此“整机总缓存很大”不等于某个 RoCE 队列可以使用全部容量。

Buffer 可以吸收短时突发，但无法长期承受高于出口转发能力的输入流量。

3\. Headroom 是什么？

Headroom 是 Buffer 中专门为 PFC 停止过程预留的安全空间。它不是日常排队容量，而是在队列达到 XOFF、交换机已经发出 Pause 后，用来接住上游尚未来得及停发的在途报文。

交换机发出 Pause 后，上游不会瞬间停止。在 Pause 帧传播、设备处理和发送流水线停止之前，仍有一批在途报文继续到达。

因此，无损队列必须预留 Headroom，用来吸收：

-   链路传播期间的在途数据；
    

-   对端响应 Pause 之前已经发出的数据；
    

-   本地端口和 ASIC 流水线中的数据；
    

-   最大帧、线缆长度和设备实现带来的余量。
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/64fced90be752c110a1391b2bdefb1cc_MD5.jpg]]

链路越快、线缆越长、设备响应越慢，需要吸收的在途数据通常越多。Headroom 不能照抄另一种交换机或另一种速率的配置，应该使用厂商针对 ASIC、端口速率、MTU 和线缆长度给出的计算或模板。

> 数量级直觉，不是配置公式：假设从交换机决定发送 Pause，到上游真正停止发送的总延迟预算为 3 μs，那么在 100 Gbit/s 链路上，这段时间仍可能到达：  
> 100 Gbit/s × 3 μs ÷ 8 ≈ 37.5 KB  
> 这只是停止延迟对应的在途数据量。实际 Headroom 还要考虑最大帧、ASIC流水线、缓存 cell 对齐、安全余量和厂商实现，不能直接把 37.5 KB 当作生产配置值。

两者的关系可以简化为：

```
普通 Buffer：吸收日常排队和微突发
Headroom：    发出 PFC Pause 后，吸收尚未停下的在途报文
```

Headroom 太小，Pause 已经发出仍可能溢出丢包；预留过大，则会挤占共享 Buffer，降低其他端口或队列吸收突发的能力。

4\. PFC 的局限与风险

PFC 能减少因拥塞造成的丢包，但不是越多越好。

队头阻塞（HOL Blocking）

同一优先级内，只要其中一个目的方向拥塞，其他本来不拥塞的流量也可能被一起暂停。

拥塞扩散

一个下游端口持续发 Pause，上游队列也会积压；上游又可能继续向更上游发 Pause，最终把局部热点扩散到更大范围。

PFC 风暴或死锁

错误的优先级映射、环形依赖、故障网卡持续发送 Pause，可能导致大量端口长期暂停。此时链路仍是 Up，但有效吞吐接近零，排障时很容易误判。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b5a50b36d896f30bc79d495ee5d39824_MD5.jpg]]

因此，生产网络通常需要：

-   只对计划中的 RoCE 优先级启用 PFC；
    

-   控制面和管理流量放在非 PFC 优先级；
    

-   启用 PFC watchdog、storm detection 或厂商同类保护；
    

-   监控 Pause 帧次数之外，还要监控 Pause 持续时间。
    

  

04

ECN：在丢包之前给报文做标记

ECN 全称 Explicit Congestion Notification。对于 RoCEv2，它使用 IP 头中的两个 ECN 位传递拥塞信息。

常见取值为：

| ECN 位 | 名称 | 含义 |
| --- | --- | --- |
| 00 | Not-ECT | 发送方不声明支持 ECN |
| 01 / 10 | ECT | 发送方声明报文支持 ECN |
| 11 | CE | 报文经过的设备发现了拥塞 |

1\. ECN 标记过程

发送端发出 ECT 报文。交换机检测到 RoCE 队列超过 ECN 门限后，不必立即丢包，而是把部分或全部经过报文的 ECN 字段改成 CE：

CE 标记不会阻止报文继续转发，而是向端点传递路径拥塞信号。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ec7dfb3bb8c8641f5ba75766750ea33a_MD5.jpg]]

2\. ECN 门限需要成套配置

交换机常见的 ECN 配置包括：

-   最小门限 Kmin；
    

-   最大门限 Kmax；
    

-   标记概率；
    

-   绝对门限或动态门限；
    

-   生效的 Traffic Class/Queue。
    

达到 ECN 门限不等于立即给每个报文标记 CE。一种常见策略是：队列低于 Kmin 不标记；从 Kmin 到 Kmax 按概率标记，并随队列水位升高逐步提高标记概率；超过 Kmax 后高概率或全部标记。具体行为取决于交换机 ASIC、NOS 和配置模式。

只有 ECT(01) 或 ECT(10) 报文能够被改写为 CE(11)。对于 Not-ECT(00) 报文，交换机不能用 CE 表示拥塞，可能按照 WRED 配置执行丢弃。

排障时应确认以下配置是否匹配：

1.  ECN 是否作用于真正承载 RoCE 的队列；
    
2.  ECN 是否早于 PFC 介入；
    
3.  门限是否与交换机缓存、端口速率和 DCQCN 参数成套设计；
    
4.  ECT 报文是否被正确标记，而不是直接被 WRED 丢弃。
    

> 正常情况下，ECN 应早于 PFC 触发。PFC 频繁触发通常表示端到端控速没有及时抑制队列增长。

05

CNP：把拥塞消息送回发送端

交换机只负责标记 CE，并不直接控制源端 RNIC 的发送速率。

接收端 RNIC 收到带 CE 标记的 RoCEv2 报文后，会生成 CNP（Congestion Notification Packet，拥塞通知报文），发回发送端：

```
数据方向：发送端 RNIC ──→ 拥塞交换机 ──→ 接收端 RNIC
标记 CE
反馈方向：发送端 RNIC ←──────── CNP ────── 接收端 RNIC
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ccd90d0c2c708f4b4089be81b276a6bd_MD5.jpg]]

CNP 是通知，不是 Pause：

-   它不会要求链路上的某个端口立即停止一个优先级；
    

-   它告诉发送端某条 RoCE 流遇到了拥塞；
    

-   发送端收到 CNP 后，由拥塞控制算法决定降速幅度和恢复方式。
    

实际部署中，CNP 常被映射到独立的高优先级队列，以免拥塞反馈本身被数据流堵住。但它不应被错误地塞进一个可能长期被 PFC Pause 的队列，否则会出现“越拥塞，反馈越回不来”的问题。

06

DCQCN：在发送端完成闭环调速

DCQCN 全称 Data Center Quantized Congestion Notification，是 RoCEv2 中广泛使用的端到端拥塞控制算法。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/0928c9a93ca0623e0e470e77aa1f5298_MD5.jpg]]

它把系统分成三个角色：

| 角色 | 所在位置 | 作用 |
| --- | --- | --- |
| CP（Congestion Point） | 拥塞交换机 | 检测排队并标记 CE |
| NP（Notification Point） | 接收端 RNIC | 识别 CE，返回 CNP |
| RP（Reaction Point） | 发送端 RNIC | 根据 CNP 调整发送速率 |

1\. DCQCN 如何降速与恢复

发送端 RNIC 会根据 CNP 估计拥塞程度，更新目标速率和当前速率。拥塞持续时继续抑制发送；拥塞缓解后，再按照定时器或已发送字节数逐步恢复。

原始 DCQCN 的降速公式可以简化为：

```
新速率 = 当前速率 × (1 - α / 2)
```

α 是发送端维护的拥塞程度估计值，通常在 0～1 之间。α 越大，降速幅度越大。例如当前速率为 100 Gbit/s、α = 0.4，则新速率约为：

```
100 × (1 - 0.4 / 2) = 80 Gbit/s
```

因此，DCQCN 不是每次固定降低某个百分比，而是根据拥塞反馈动态计算降速幅度。不同 RNIC、固件和驱动的实现参数可能有所不同。

影响 DCQCN 调速行为的参数包括：

-   初始速率和最低速率；
    

-   收到 CNP 后的降速强度；
    

-   拥塞程度估计的平滑权重；
    

-   无新 CNP 时的恢复周期；
    

-   加性恢复、快速恢复等阶段的步长；
    

-   CNP 生成和限速相关参数。
    

参数过于激进，吞吐可能剧烈振荡；参数过于保守，队列降不下来，PFC 会频繁触发。

2\. DCQCN 为什么不能完全取代 PFC？

ECN、CNP 和源端降速形成闭环需要时间。遇到极强微突发时，交换机队列可能在反馈抵达发送端之前就逼近缓存上限。

PFC 的价值是在这段反馈延迟内提供快速的逐跳保护。反过来，只有 PFC 而没有有效的 ECN/DCQCN，持续拥塞就会不断触发 Pause，容易造成拥塞扩散。

07

三者是如何配合的？

以多个 GPU 节点同时向一个出口发送为例：

```
1. 多个发送端同时突发
↓
2. 交换机出口队列上涨
↓
3. 达到 ECN 门限，交换机标记 CE
↓
4. 接收端 RNIC 返回 CNP
↓
5. 发送端 DCQCN 降速
↓
6. 拥塞缓解后，发送端逐步恢复速率
若 3～5 的反馈尚未来得及生效，队列继续上涨：
7. 达到 PFC XOFF 门限
↓
8. 交换机逐跳 Pause 指定优先级
↓
9. 队列下降到恢复水位后解除 Pause
```

推荐的逻辑关系是：

```
ECN 标记门限 < PFC XOFF 门限 < 实际丢包水位
```

三者之间还要留出足够安全距离：ECN 需要时间完成端到端反馈，PFC XOFF 之后需要 Headroom 吸收在途报文。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/262186e788d5f5c2a06d0be192c2de15_MD5.jpg]]

这只是设计原则，不是可以直接下发的数值公式。共享缓存架构、端口速率、扇入比、MTU、线缆长度、ASIC 单元大小和厂商实现都会影响最终配置。

08

优先级映射：最容易被忽略的基础

服务器与交换机的优先级映射不一致，是 RoCE 故障的常见原因。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/9f2a9fb885d8586f01436c46c037ffa6_MD5.jpg]]

一个典型链路可能包含：

```
应用/NCCL
↓
RNIC 根据 DSCP 或 PCP 标记报文
↓
交换机 Trust DSCP/PCP
↓
映射到 Switch Priority
↓
映射到 Traffic Class / Queue
↓
该队列应用 ECN、PFC 和 Buffer 配置
```

需要逐跳核对：

-   RoCEv2 数据报文使用的 DSCP/PCP；
    

-   端口的 Trust 模式；
    

-   DSCP/PCP → Priority → TC/Queue 映射；
    

-   PFC 启用的 Priority；
    

-   ECN 生效的 TC/Queue；
    

-   CNP 使用的 Priority 和调度方式；
    

-   主机、Leaf、Spine、边界设备是否保持一致。
    

只要中间一跳重写 DSCP、Trust 模式错误或队列映射不一致，就可能出现：服务器认为流量属于无损类，交换机却把它放进普通有损队列。

09

AI 运维应该监控哪些指标？

单看端口利用率远远不够。400G 端口的 1 毫秒平均利用率可能并不高，但一个微秒级突发已经足以触发 ECN 或 PFC。

交换机侧

| 指标 | 说明 | 需要警惕的现象 |
| --- | --- | --- |
| ECN marked packets | 被标记 CE 的报文数/速率 | 持续高位且伴随吞吐下降 |
| PFC Tx | 本端向上游发送 Pause | 说明本端下游方向出现积压 |
| PFC Rx | 本端收到下游 Pause | 本端被下游反压 |
| PFC pause duration | 处于暂停状态的累计时间 | 比单纯帧数更能反映影响程度 |
| Queue occupancy | 队列当前/峰值水位 | 长期高水位或频繁触顶 |
| Buffer max usage | Buffer 峰值使用量 | 接近 Headroom 或共享池上限 |
| Queue drops / discards | 队列丢包 | 无损类出现丢包通常需要立即调查 |
| WRED/ECN counters | 标记与丢弃动作 | 确认 ECT 报文被标记而非误丢弃 |

RNIC 与主机侧

不同厂商和驱动的计数器命名不同，常用入口包括：

```
ethtool -S <netdev>
rdma link show
rdma statistic show
perfquery                 # InfiniBand 场景常用，RoCE 需结合具体驱动
ls /sys/class/infiniband/<device>/ports/1/hw_counters/
# NVIDIA/Mellanox ConnectX（需安装相应驱动工具或 MFT）
mlnx_qos -i <netdev>          # 查看 QoS、Trust、PFC 等配置
mst start                     # 启动 MFT 设备访问服务
mst status -v                 # 查找 PCI BDF 与 MST 设备名
mlxlink -d <PCI_BDF或MST设备> # 查看链路、FEC、误码等物理层信息
```

mlxreg 可做寄存器级查询和调试，但也具备修改设备寄存器的能力。生产环境中只应按 NVIDIA 文档或厂商支持人员的明确步骤使用，本文不提供寄存器写入示例。show\_gvmid.sh 并非所有 ConnectX 软件栈都提供的标准命令，因此没有把它列为通用排障入口。

重点寻找：

-   CNP 发送和接收；
    

-   ECN/CE 相关计数；
    

-   RoCE 重传、序列错误和 retry exceeded；
    

-   PFC Rx/Tx 和暂停时长；
    

-   端口丢包、discard、CRC/FCS、符号错误；
    

-   RDMA QP/CQ 错误。
    

作业侧

-   NCCL bus bandwidth 和 algorithm bandwidth；
    

-   Collective 的 P50/P95/P99 延迟；
    

-   单 Step 时间和长尾；
    

-   不同 Rank 的完成时间差异；
    

-   GPU SM 利用率下降是否与通信阶段重合；
    

-   故障是否集中在固定的 Leaf、Spine、Rail 或目的节点。
    

把作业时间线、RNIC 计数器和交换机队列遥测放到同一个时间轴上，通常比单独看任何一个告警更有效。

10

常见现象与判断方向

| 现象 | 可能原因 | 优先检查 |
| --- | --- | --- |
| ECN 标记上升，CNP 上升，吞吐基本稳定 | DCQCN 正常工作，网络正在主动控速 | 队列是否回落、PFC 是否很少触发 |
| ECN 为零，但 PFC 频繁触发 | ECN 门限过高、未启用或队列映射错误 | ECN 配置、Trust、TC 映射 |
| ECN/CNP 很多，队列仍长期很高 | DCQCN 参数偏慢、源端未正确响应或持续过载 | RNIC CC 状态、CNP Rx、速率恢复参数 |
| PFC Tx/Rx 很高，端口无丢包但作业卡顿 | Pause 传播、HOL 阻塞或 PFC 风暴 | Pause duration、传播路径、watchdog |
| 无 PFC、无 ECN，但有队列丢包和重传 | 流量进入了错误队列，或功能未生效 | DSCP/PCP、Trust、Queue counters |
| 只有一个 Leaf/Rail 反复出现 PFC | 局部热点、ECMP 哈希碰撞、链路降速或拓扑不对称 | 路径、端口速率、ECMP、故障链路 |
| CNP 发出很多，但发送端 CNP Rx 很少 | CNP 路径、优先级或 ACL 有问题 | CNP 队列、路由、ACL、丢包计数 |
| PFC 后仍有无损队列丢包 | Headroom 不足、XOFF 太晚或 Pause 未被对端接收 | Headroom、线缆长度、PFC Rx/Tx、MTU |

注意：计数器非零不等于故障。ECN 标记本来就是主动拥塞控制的一部分。判断时应该关注速率、持续时间、基线变化以及它与作业长尾的相关性。

11

推荐排障顺序

第一步：先排除物理层和链路问题

检查端口是否降速、FEC 是否异常、CRC/FCS、symbol error、光功率和链路 flap。物理误码与拥塞丢包的处理方向完全不同。

这些指标分别表示：

| 指标 | 是什么 | 异常时通常说明什么 |
| --- | --- | --- |
| FEC | Forward Error Correction，接收端利用冗余编码修复少量比特错误 | corrected 持续快速增长表示链路质量变差；uncorrectable 表示错误已经无法修复，可能直接丢包 |
| CRC/FCS | 以太网帧完整性校验；接收端计算结果与帧尾校验值不一致时记错 | 报文在物理传输中损坏，常见于模块、光纤、DAC/AOC 或端口异常 |
| Symbol Error | 接收端识别到了无效的物理层编码符号 | 信号衰减、噪声、线缆质量、模块兼容性或端口硬件存在问题 |
| 光功率 | 光模块的发送功率 Tx Power 和接收功率 Rx Power，通常以 dBm 表示 | Rx 过低常见于光纤衰减、接头脏污或弯折；过高可能使接收器过载，需要与模块规格范围比较 |
| Link Flap | 端口在 Up 和 Down 状态之间反复切换 | 模块接触不良、光纤故障、FEC 模式不匹配、端口故障或对端重启 |

其中，少量 FEC corrected 并不必然代表故障，关键是观察增长速率和历史基线；CRC/FCS、FEC uncorrectable、链路 flap 持续增加则应优先处理。不要在物理层错误仍在增长时直接调整 ECN、PFC 或 DCQCN 参数。

```
FEC / CRC / Symbol Error / 光功率 / Link Flap → 物理链路方向
ECN / CNP / PFC / Queue Occupancy / Buffer Drop → 拥塞控制方向
```

第二步：确认 RoCE 流量进入了正确队列

从 RNIC 到 Leaf、Spine 再到对端，逐跳核对 DSCP/PCP、Trust、Priority、TC 和 Queue。不要只看配置命令成功，要看实际队列计数是否增长。

第三步：判断拥塞控制链路是否闭环

参照第七节的拥塞控制闭环，依次核对交换机 ECN/CE、接收端 CNP Tx、发送端 CNP Rx、发送速率和队列水位。哪个信号没有按预期变化，就重点检查对应环节。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d3f9b162f4774b38e938f6288f0257ae_MD5.jpg]]

第四步：检查 PFC 是否只是短暂兜底

少量 PFC 不一定异常；持续 Pause、Pause 时长快速增长或多跳扩散才是危险信号。结合队列水位判断 XOFF 是否过早或过晚。

第五步：关联具体通信模式和路径

用 NCCL Tests 或业务的 Collective 指标复现，确认问题出现在 AllReduce、All-to-All，还是某个固定源/目的组合。再结合 ECMP、Rail 和交换机遥测定位热点。

第六步：最后再调门限和算法参数

先确认物理层、优先级映射和功能闭环正确，再修改 ECN、PFC 或 DCQCN 参数。一次只改一组参数，保留变更前后的队列、CNP、PFC 和作业性能数据。

12

总结

RoCEv2 拥塞控制包含以下四个环节：

1.  交换机通过 ECN 提前标记拥塞；
    
2.  接收端 RNIC 通过 CNP 把拥塞反馈给发送端；
    
3.  发送端 DCQCN 降低对应流的发送速率，并在拥塞缓解后逐步恢复；
    
4.  如果端到端反馈尚未生效，PFC 在队列接近危险水位时逐跳暂停，保护缓存不被打爆。
    

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/8bface9f_1784249967016?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUyNTY0NzI4OA%3D%3D%26mid%3D2247484419%26idx%3D1%26sn%3D1ab7580fd4928a74518c871acda763a3%26chksm%3Dfb63fa743bba4918bc314ad5807faa2c1a881846ec8da2e44e742576a6fbe253adb71686a724%26mpshare%3D1%26scene%3D1%26srcid%3D0717ouTBOqGO2ZJt76xkWRKe%26sharer_shareinfo%3Dfb64d4f9137c4ae3f52e75047f889635%26sharer_shareinfo_first%3Dfb64d4f9137c4ae3f52e75047f889635%23rd&s=obsidian)