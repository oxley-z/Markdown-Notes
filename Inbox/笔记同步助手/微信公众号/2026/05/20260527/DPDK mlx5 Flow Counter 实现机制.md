---
author: GavinHu
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzODIxNjk5Mg==&mid=2247484845&idx=1&sn=7bdeac7aaa2a45ee9bb3208a164754db&chksm=c31b4998c674f2fc8eddd2624a5b7f3965eb97418f0b22f4189817aab2b92fe2f2f25dc0711f&mpshare=1&scene=1&srcid=05276pKJQkenOfXS5YxmraVC&sharer_shareinfo=3ac37ab8840a20db9507bfbed6d3e6e6&sharer_shareinfo_first=3ac37ab8840a20db9507bfbed6d3e6e6#rd
saved: 2026-05-27 11:10:05
tags:
  - 笔记同步助手
id: 4c586735-58b5-4e83-905b-de4218c96294
---

公众号名称：GavinHu的工作记忆

作者名称：GavinHu

发布时间：2026-05-27 00:00

MLX5 的 flow counter 看起来只是 rte\_flow 上挂一个 COUNT action，再通过 rte\_flow\_query() 读取 hits/bytes，但真正实现起来远比接口表面复杂。

原因在于：​硬件 counter 资源是按 bulk 分配的，单个 alloc/query 成本很高；查询本质是 FW mailbox 操作，无法让应用在百万级 counter 规模下逐条同步读取；同时 fast path 上又不能接受加锁和频繁控制面交互。

因此，mlx5 的设计思路从一开始就不是“应用查一个 counter，驱动就去查一次硬件”，而是​把 counter 系统做成一套后台批量查询、前台读取内存快照、代际安全复用的子系统​。 代码里长期同时存在两套 counter 管理体系：一套面向旧的 SWS/DV 路径，另一套面向新的 HWS 路径。

count action 的使用示例：

testpmd> flow create 0 ingress \\

pattern eth / ipv4 src is 10.0.0.1 dst is 10.0.0.2 / \\

udp src is 1234 dst is 5678 / end \\

actions count identifier 1 / queue index 0 / end

testpmd> flow query 0 0 count

​

# 底层抽象

两套实现共享同一个硬件抽象：DEVX 的 ALLOC\_FLOW\_COUNTER 和 QUERY\_FLOW\_COUNTER。

一次 ALLOC\_FLOW\_COUNTER 返回的是一个 DCS（Devx Counter Set） 对象，FW 给出这批 counter 的起始 flow\_counter\_id，后续每个具体 counter 的硬件 ID 都由 dcs->id + offset 推导出来。

查询也有两种模式：

如果只查一个 counter，就走同步接口直接返回 hits/bytes；

如果要查一整段 counter，就打开 dump\_to\_memory，让 FW 直接把整批 {hits, bytes} DMA 到一块预注册好的 MR 中。 mlx5 整个高性能 counter 框架，实际上就是围绕这个“​批量 DMA 到内存​”能力建立起来的。

​

# SWS 体系

在 SWS（legacy DV）路径中，counter 按 pool 管理，每个 pool 固定 512 个 counter。 外部看到的是一个 32-bit cnt\_idx，内部则通过 pool\_index + offset 映射到具体 pool 中的 counter；每个 pool 还保存两份统计快照 raw/raw\_hw，配合 query\_gen 形成 ping-pong 机制。

分配路径的核心是 flow\_dv\_counter\_alloc()。​它先尝试从全局 free list 中拿一个空闲 counter；如果空了，就触发一次 bulk alloc，向 FW 申请一个 DCS，对应 512 个 counter，第 0 个立即返回给本次调用，其余 511 个挂回 free list 备用。​第一次使用某个 counter 时，还会 lazy 创建 DV count action，真正把它变成 steering rule 可引用的句柄。

SWS 最关键的设计是：应用查询不直接访问硬件。 驱动通过 rte\_eal\_alarm\_set() 定时器周期性轮询 pool，​每次只查一个 pool（所有pool 在一秒内查完），把 512 个 counter 的 hits/bytes 一次性 DMA 到 raw\_hw；异步完成后再由completion handler 把 raw\_hw 切换成新的 raw，应用前台只需读内存中的快照。​ 换句话说，rte\_flow\_query() 在 SWS 正常路径下本质只是一次内存 load，再减去 alloc 时记录的基准值。

之所以要有 query\_gen，是因为 counter 不能在 free 后立刻重用。​如果某个 counter 刚释放，还没等覆盖它的那一轮 pool query 完成，就重新分配给下一个 flow，那么下一次 query 读到的 hits 里仍然混有上一任使用者留下的数据。​因此，SWS 用 ping-pong list 明确表达“这个 counter 已经 free，但还得等一轮 query 之后才能重新进入 free list”。

Counter 的生命周期如下：

device init

\-> init sws\_cmng, no pools yet

flow create with COUNT

\-> allocate from global free list

\-> if empty, create 512-counter pool

\-> assign one counter to flow

\-> create count action if first use

\-> store baseline

\-> start query alarm

periodic alarm

\-> async query one pool

\-> HW/FW writes stats to raw\_hw

\-> DevX completion event

completion handler

\-> publish raw\_hw as pool->raw

\-> recycle old raw buffer

\-> reclaim counters released before this query

rte\_flow\_query

\-> read latest pool->raw snapshot

\-> subtract baseline

\-> optionally update baseline on clear

flow destroy

\-> remove HW flow

\-> put counter into pool recycle list

\-> after a later query completion, move it back to global free list

device close

\-> cancel alarm

\-> destroy DevX counter objects/actions

\-> free pools and stats memory

​

# HWS 体系

到了 HWS（ConnectX-7+ 的 Hardware Steering），counter 子系统几乎被重写了一遍，目标是支持 16M+ counter、全程 lockfree alloc/free，以及更高吞吐的查询路径。 这里最大的变化是：查询不再依赖 mailbox completion 通道，而是改成 ASO WQE，由硬件队列直接发起批量查询。

HWS 的 counter ID 也不再直接暴露 FW counter id，而是由 PMD 自己编码成 32-bit 值。​高位复用 rte\_flow indirect action 的 TYPE，表示它是一个 count handle；中间两位表示 DCS index；低 24 位表示该 counter 在 DCS bulk 中的 offset，因此单 pool 最多可以覆盖 16M 个 counter。​这种编码的好处是：应用可以直接把这个 ID 当 indirect action handle 用，驱动再内部解码成 pool 内部下标。

在数据结构上，HWS 用三条 rte\_ring 管理 counter 生命周期：free\_list 表示从未用过的 counter，wait\_reset\_list 表示已 free 但还不能复用的 counter，reuse\_list 表示已完成一轮 query、可以重新分配的 counter。​在此基础上，每个 queue 还有一个本地 qcache，用 zero-copy ring API 实现无锁缓存。

alloc fast path 的核心函数是 mlx5\_hws\_cnt\_pool\_get()。​它优先从本 queue 的 qcache 中取 counter；如果缓存空了，就批量从 reuse\_list 或 free\_list 中拉取；如果只能看到 wait\_reset\_list 非空，则直接返回 -EAGAIN，告诉上层稍后再试。​这套设计的关键不是“永远拿得到 counter”，而是“永远不把还没完成 reset 的 counter 交给新使用者”。

free fast path 同样完全无锁。 mlx5\_hws\_cnt\_pool\_put() 只负责清状态、记录 query\_gen\_when\_free，再把 counter 放回 qcache 或 wait\_reset\_list；真正把它推进到 reuse\_list，是在后续 service 线程完成一轮 query 之后。Service 线程与 ASO 查询

HWS 不再使用 SWS 的 alarm 定时器，而是由一个 pin 到 service core 的专用线程周期性扫描所有 cpool。​每轮服务的第一步，是调用 mlx5\_aso\_cnt\_query()，通过 4 条专用 SQ 并行发送 ASO WQE，把整片 counter 的 {hits, bytes} DMA 到 raw\_mng->raw 中。​由于 ASO 通道的吞吐远高于 mailbox，这一代设计才具备支撑 16M 级 counter 的基础。

查询完成后，service 线程做两件事：一是 query\_gen++，告诉 alloc 路径“上一代 free 的 counter 现在已经被刷新，可以重用了”；二是把 wait\_reset\_list 整段搬到 reuse\_list。​这就是 HWS 版的“代际安全复用”机制，它和 SWS 的 ping-pong list 本质是同一个思想，只是表达方式更适合 lockfree ring 和大规模并发场景。

另一个细节是读快照的一致性问题。 因为 ASO DMA 写 stats 和应用读 stats 是并发的，HWS 不直接信任单次 load，而是使用一个经典的“双读比较”技巧：连续读两次 hits/bytes，只有两次结果完全一致时才认为读到了稳定快照。 这样可以避免应用恰好在 DMA 更新过程中，读到半新半旧的数据。

​

# 查询为什么能这么快

无论 SWS 还是 HWS，rte\_flow\_query() 的高性能都来自同一个原则：​应用查询只是读内存中的快照，再减去 reset 基准值。真正和硬件 FW 打交道的查询都被后台线程或定时器批量完成了，前台 never enters mailbox。

这也是为什么 mlx5 counter 支持 reset 几乎不需要额外开销。 reset 并不是通知硬件把 counter 清零，而只是更新 PMD 自己记录的 cnt->reset.hits/bytes 基准；以后查询时返回“当前快照 - 基准”，逻辑上看起来像被 reset，底层硬件计数其实一直在累加。

​

# Aging 如何复用 counter

HWS 还有一个很巧妙的设计：aging 和 counter 复用同一份 query 结果。 每个 counter 可以带一个 age\_idx，service 线程在拿到这一轮 DMA stats 后，顺手检查 hits 是否变化；如果某个 flow 对应的 hits 长时间不变，就认为它 aged out。

这样做的收益很直接：aging 不需要再单独发一轮 FW query。 对大量 flow 的场景来说，这等于把“计数”和“老化判断”合并在同一份后台采样里，节省了大量控制面开销。

​

# 设计主线

如果把 mlx5 flow counter 的实现抽象成几条主线，其实非常清晰。

分配不进 mailbox：mailbox 只在 pool 扩张时发生，单个 counter 的 alloc/free 全在 PMD 内部 list/ring 完成。

应用读 = 内存读：前台永远读快照，后台统一批量 query。

基于 generation 的安全重用：counter free 后必须等覆盖它的那轮 query 完成，才能真正复用。

aging 复用同一份 stats：hits 同时承担“计数值”和“心跳”两种角色。

HWS 的升级重点：从 mailbox 升级到 ASO WQE，从全局 spinlock 升级到 per-queue zero-copy ring，把可扩展规模从百万级推进到 16M 级。

一句话总结，mlx5 counter 的本质不是“如何查询一个 counter”，而是“如何把海量 counter 的分配、查询、复用和 aging 都变成后台批处理，而前台只做无锁内存读取”。

---

![[images/67b7766fe534bc8e30affabd45ed2b12_MD5.jpg|cover_image]]

原创 GavinHu GavinHu的工作记忆

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/6bc8f7b0_1779851403659?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzODIxNjk5Mg%3D%3D%26mid%3D2247484845%26idx%3D1%26sn%3D7bdeac7aaa2a45ee9bb3208a164754db%26chksm%3Dc31b4998c674f2fc8eddd2624a5b7f3965eb97418f0b22f4189817aab2b92fe2f2f25dc0711f%26mpshare%3D1%26scene%3D1%26srcid%3D05276pKJQkenOfXS5YxmraVC%26sharer_shareinfo%3D3ac37ab8840a20db9507bfbed6d3e6e6%26sharer_shareinfo_first%3D3ac37ab8840a20db9507bfbed6d3e6e6%23rd&s=obsidian)