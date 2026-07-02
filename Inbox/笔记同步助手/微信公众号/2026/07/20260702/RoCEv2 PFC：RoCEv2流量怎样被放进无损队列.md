---
author: 宋昭威
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzY4MjI4NTM4OA==&mid=2247484001&idx=1&sn=42fe60b74c2e0196b2d920c3b0c2cec6&chksm=f24f41ecaa730debbe4fb7309911998889ee02f18a221d557329f54bf1044389b44cfd2b0d6d&mpshare=1&scene=1&srcid=0702gDFINVGI8447E6DkaFJM&sharer_shareinfo=13dce2db90ce64f8b8cc3b1f64389a54&sharer_shareinfo_first=13dce2db90ce64f8b8cc3b1f64389a54#rd
saved: 2026-07-02 07:58:05
tags:
  - 笔记同步助手
id: f2f67da4-37fa-48c6-b687-9e1a09a5a9df
---

公众号名称：宋昭威的个人笔记

作者名称：宋昭威

发布时间：2026-06-29 16:41

# 引言

上一篇 EVPN Multihoming 解决的是"服务器同时接入两台 Leaf，Fabric 怎么理解这件事"。接入段的身份、成员关系、转发责任，都已经通过 EVPN 控制面统一表达。

但 AI 后端网络在接入段之上，还有一层问题更紧迫：RoCEv2 流量怎么被识别、怎么被分类、怎么被放进无损队列。

这篇聊 PFC，Priority-based Flow Control。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/2e2c7bf096ae71e4b96a70f50f10a7e5_MD5.jpg]]

## RDMA 怕丢包，但以太网天生会丢包

RoCEv2 把 RDMA 承载到 UDP/IP 之上，使 RDMA 可以跨三层网络转发。数据中心可以用普通以太网跑 RDMA，不用再专门铺一张 InfiniBand 网络。

但 RDMA 对丢包极度敏感。

TCP 丢包可以重传，应用层感受不太到。RDMA 不一样，一旦丢包，NIC 会触发 go-back-N 重传，严重时直接超时断连。对于 AI 训练里的集合通信（AllReduce、AllGather），一次尾延迟抖动就可能拖慢整个同步阶段，几百张 GPU 一起等。

可传统以太网就是会丢包的。交换机缓存满了就 tail drop，不区分你是 RDMA 流量还是普通业务流量。

早期有人用 802.3x Pause 来解决，整条链路暂停。这等于把高速公路上所有车道同时封死，RDMA 流量和普通 HTTP 流量挤在同一条链路上，一暂停全停，代价太大。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f667daf1ee77d35d4599e9864c9bcf07_MD5.jpg]]

于是问题就变成了：能不能只暂停"需要无损保护的那部分流量"，让其他流量正常通行？

PFC 就是为了解决这个问题。

## 服务器打了标记，交换机凭什么信任它？

要让交换机"只暂停部分流量"，第一步是让交换机知道哪些流量需要被特殊对待。

RoCEv2 跑在 IP 体系里，IP 头里有 DSCP 字段。服务器侧需要给 RoCEv2 报文打上特定的 DSCP 值（比如 DSCP 26），告诉网络"这批报文是 RDMA 流量，需要无损保护"。

交换机收到报文以后，需要在 Trust DSCP 模式下读取这个 DSCP 值，并把它映射到对应的 Switch Priority。比如 DSCP 24-31 映射到 Switch Priority 3。

这一步是整条链路的起点。

如果服务器侧 RoCEv2 报文没有被正确打标，仍然使用 DSCP 0，那么在 Trust DSCP 模式下它会被映射到 Priority 0。Priority 0 通常不是无损 priority，后面所有 PFC 配置都保护不到它。

很多人第一次部署 RoCEv2 时踩过这个坑：交换机侧 PFC 配得整整齐齐，服务器侧 DSCP 没打对，流量根本没进无损队列，PFC 形同虚设。

**服务器侧 DSCP 决定报文进入哪个 Switch Priority，PFC Priority 决定哪些 Switch Priority 可以获得无损暂停保护。** 这两个环节必须对上。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/c0088612d5a55bbbe593016dc9d25d80_MD5.jpg]]

## PFC 不是把端口变成无损端口，而是把 priority 变成无损 priority

PFC 的全称是 Priority-based Flow Control，IEEE 802.1Qbb 定义它的时候目标很明确：让以太网链路可以按 traffic class 逐个暂停，而不是像 802.3x Pause 那样整条链路停转。

名字里最关键的词不是 Flow Control，而是 Priority。

放到 RoCE 模板里，`PFC Priority: 3` 表达的就是这个含义：Switch Priority 3 被配置为 PFC 保护对象。如果 RoCEv2 流量进入 Priority 3，当对应入口缓存达到触发水位时，交换机可以对该 priority 发送 PFC Pause，要求对端暂停发送这个 priority 的流量。

同一条物理链路上，Priority 0 的普通业务流量不受影响，该转发继续转发。

`TX Status: Enabled` 和 `RX Status: Enabled` 分别说明两个方向。TX 表示本端可以向直连对端发送 Pause，RX 表示本端收到对端 Pause 后会响应暂停。这里的 Pause 是逐跳生效的链路级动作，只影响相邻两端之间的对应 priority。

所以 PFC 保护的不是端口，是 priority。一个端口上可以有 8 个 priority，只有被标记为 PFC Priority 的那几个才会被暂停保护。RoCEv2 流量必须进入正确的 priority，PFC 才能保护到它。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/2e2c7bf096ae71e4b96a70f50f10a7e5_MD5.jpg]]

## PFC 真正盯住的不是速率，而是缓存水位

PFC 不是在端口流量跑到 100G、200G 或 400G 时自动发送 Pause，也不是在某个速率阈值上踩刹车。PFC 盯住的是入口无损缓存的占用水位。

当流量进入某个启用 PFC 的 priority 的交换机，而下游暂时来不及转发时，这类报文会占用对应的 ingress lossless buffer。缓存占用继续上升，达到 xoff 水位以后，交换机就向直连对端发送该 priority 的 PFC Pause。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/75dd2e703d47211d838e20bbdd1a268a_MD5.jpg]]

xoff 可以理解为触发暂停的高水位。它不能太低，太低会导致链路过早 Pause，降低有效吞吐；也不能太高，太高会让交换机在发出 Pause 后仍然来不及吸收已经在路上的数据。

这里有一个关键变量：端口速率越高、线缆越长，从 Pause 发出到对端真正停下来的"在途数据"越多。一条 400G 链路用 5 米铜缆和用 30 米光缆，需要的缓存余量完全不一样。无损缓存分为静态和动态两部分，动态部分的参数和端口速度、电缆长度相关。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d2290f0430812ebc8e9beb6ce5cf5371_MD5.jpg]]

触发暂停以后，缓存水位不会刚低于 xoff 就立刻恢复发送，否则队列会在临界点附近反复 Pause 和 Resume，产生振荡。xon-offset 的作用就是拉开触发水位和恢复水位之间的距离，让缓存下降到更安全的位置后再恢复。

dynamic\_th 则描述 buffer profile 如何使用共享缓存余量，控制动态计算可用阈值的比例。

把这几个参数串起来，PFC 的暂停-恢复过程就清楚了：缓存涨到 xoff，发 Pause，对端停发，缓存回落到 xoff - xon-offset，发恢复帧，对端继续发送。

## PFC 什么时候踩刹车，频繁踩刹车为什么危险

把前面的分类和水位连起来，PFC 触发条件就很清楚了：某个已经开启 PFC 的 priority，其入口无损缓存占用达到 xoff，交换机向直连对端发送该 priority 的 PFC Pause。

一个典型过程：RoCEv2 报文从服务器发出，携带 DSCP 26，交换机在 Trust DSCP 模式下把它映射到 Priority 3，Priority 3 已经开启 PFC 并绑定 lossless buffer profile。​当 Priority 3 对应入口无损缓存达到 xoff，交换机向服务器发送 Priority 3 的 PFC Pause，服务器收到后暂停发送 Priority 3 流量，缓存水位回落到恢复边界后，发送恢复帧。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3a6a5801e85db9de66c4ea2232485723_MD5.jpg]]

这套机制的作用是在缓存溢出前先按 priority 踩一脚刹车，降低 RoCEv2 流量因瞬时拥塞被丢弃的风险。对于 RDMA 流量，丢包意味着重传、超时和性能抖动，训练或存储业务会直接感受到尾延迟变化。

但 PFC 不是越多越好。

如果 PFC 计数持续高速增长，说明网络里存在持续热点、微突发、下游拥塞或阈值设置不合适。频繁 Pause 会让上游发送端反复停顿，还可能把局部拥塞影响传递给相邻链路，这就是所谓的 PFC Storm，拥塞从一个点扩散到整条路径。

对于 AI 后端网络里的集合通信流量，少数链路抖动就可能拖慢一组同步通信阶段。PFC 更像是无损队列的兜底防线，不是常态化调速工具。

## 怎么确认 PFC 真的在保护 RoCEv2 流量

验证 PFC 不能只看"有没有配置该命令"，更可靠的做法是沿着链路逐段检查。

先看模板整体状态。重点看 Trust Mode 是否为 DSCP，DSCP 到 Switch Priority 的映射是否符合预期，PFC Priority 是否包含 RoCEv2 要使用的 priority，TX/RX Status 是否 Enabled，PFC Profile 是否绑定到对应 Switch Priority。

再看某个接口、某个队列的运行状态。如果 RoCEv2 规划使用 Priority 3，就重点看 queue 3 的相关计数。

最后看 PFC Pause 计数。Tx PFC 表示本端向对端发过 Pause，说明本端对应 priority 的入口缓存曾经达到暂停条件；Rx PFC 表示本端收到过对端的 Pause，说明对端要求本端暂停发送对应 priority 的流量。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/143a316dd9bdadca2cf0cb66c8c62101_MD5.jpg]]

这里有两个容易误判的地方。PFC 计数为 0 不一定是坏事，可能只是当前没有拥塞；PFC 计数持续增长也不一定代表配置正确，可能只是流量进入了某个 PFC priority 并持续触发 Pause。要判断 PFC 是否真的在保护 RoCEv2，需要确认三件事：RoCEv2 报文是否进入了预期 priority，PFC 是否只在对应 priority 上触发，触发频率是否和业务压力、链路设计匹配。

## 小结

回到整条链路，PFC 的逻辑可以串成一条线：

服务器侧 RoCEv2 报文携带 DSCP → 交换机 Trust DSCP 后把报文映射到 Switch Priority → RoCE 模板对指定 Priority 开启 PFC → 该 Priority 绑定 lossless buffer profile → 入口缓存达到 xoff 后，交换机对直连对端发送 PFC Pause → 缓存回落到恢复边界后，再恢复发送。

这条链路里任何一个环节断开，都会让"交换机已经配置 RoCE 模板"这件事失去意义。DSCP 没有打对，流量进不了 PFC 保护的无损 priority；PFC Priority 没有开，priority 无法获得暂停保护；xoff、xon-offset 和 dynamic\_th 不理解，PFC 就只能照抄模板，问不出为什么不同速率、线缆长度和突发模型需要不同水位。

PFC 不是 RoCEv2 的全部，也不是无损网络的万能开关。它更像是兜底刹车，真正保护的是被正确分类后的 priority。后面可以再分别聊 DCBX、ECN/DCQCN 和 PFC Watchdog，这些话题都要建立在 PFC 链路已经搞清楚的基础之上。

## 参考来源

1.  IEEE 802.1, _802.1Qbb - Priority-based Flow Control_
    
2.  InfiniBand Trade Association, _RoCEv2 Specification Release_
    
3.  NVIDIA Docs, _Ethernet Network - RoCE Configuration_
    
4.  NVIDIA Networking Docs, _RDMA over Converged Ethernet (RoCE)_
    
5.  SONiC / AsterNOS Command Reference, QoS RoCE / Buffer Profile / PFC Counters
    

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/1784e6c0_1782950283535?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzY4MjI4NTM4OA%3D%3D%26mid%3D2247484001%26idx%3D1%26sn%3D42fe60b74c2e0196b2d920c3b0c2cec6%26chksm%3Df24f41ecaa730debbe4fb7309911998889ee02f18a221d557329f54bf1044389b44cfd2b0d6d%26mpshare%3D1%26scene%3D1%26srcid%3D0702gDFINVGI8447E6DkaFJM%26sharer_shareinfo%3D13dce2db90ce64f8b8cc3b1f64389a54%26sharer_shareinfo_first%3D13dce2db90ce64f8b8cc3b1f64389a54%23rd&s=obsidian)