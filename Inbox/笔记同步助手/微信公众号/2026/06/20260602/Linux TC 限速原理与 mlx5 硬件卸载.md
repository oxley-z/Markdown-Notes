---
author: GavinHu
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzODIxNjk5Mg==&mid=2247484909&idx=1&sn=ba24ea98a028811af6682e158cf5d02e&chksm=c3a841a3f31a7b455dd84605e09ff62156a84ecb517a6b7648aea604437392c044636b60faa6&mpshare=1&scene=1&srcid=0602e5sVZ7A4W7ktTnNbuRYC&sharer_shareinfo=0d66a8445c88f6486ab9d963cf671c77&sharer_shareinfo_first=0d66a8445c88f6486ab9d963cf671c77#rd
saved: 2026-06-02 14:15:36
tags:
  - 笔记同步助手
id: 92d85a47-b7b0-415a-af1b-8a38e7d8ab13
---

公众号名称：GavinHu的工作记忆

作者名称：GavinHu

发布时间：2026-06-02 00:00

Linux TC（Traffic Control）是内核的流量控制子系统，用于实现限速、整形、优先级调度和策略丢包。其架构以qdisc-class-filter三层树形结构组织，参考【1】。

---

# 令牌桶工作原理

令牌桶（Token Bucket）是 TC 限速的基础 ：

![[images/7c125f296d3d3b84a4d01c1a6f71bad7_MD5.jpg]]

令牌以固定速率注入桶（rate = 目标速率）每发送 1 字节消耗 1 个令牌。

桶的最大容量 = burst（允许的瞬时突发量）

时间片内： 报文到达 → 检查桶内令牌数

≥ 报文字节数 → 消耗令牌，conform（放行）

< 报文字节数 → 令牌不足，exceed（超限）→ drop/等待

rate 决定长期平均速率，burst 决定允许的短时突发大小。

# egress 限速（SW Traffic Shaping）

## Traffic Shaping

通过令牌桶（Token Bucket）控制出方向（egress）发送速率，多余流量缓冲在队列中等待，适合软限速。

报文到达 → 令牌桶有令牌 → 立即发送 → 令牌桶空 → 进队列等待令牌补充（不丢包）

特点：平滑突发、不丢包，但增加延迟，只能作用于egress（出方向）。

![[images/b3664c4bd3fc8e53c0848d8ecce1b12e_MD5.jpg]]

---

## 完整配置步骤（SW egress HTB + flower 限速）  

对 egress 限速需用 **HTB（Hierarchical Token Bucket）**：

\# 出方向根 qdisc（HTB）

tc qdisc add dev p0 root handle 1: htb default 10

\# 限速 class：上限 10Gbps

tc class add dev p0 parent 1: classid 1:10 htb \\

rate 10gbit burst 100mb

\# flower filter 将匹配流量引入该 class

tc filter add dev p0 parent 1: \\

protocol ip priority 100 \\

flower \\

src\_ip 10.0.0.0/24 \\

flowid 1:10

---

# Ingress 限速 tc police （mlx5）

## Policing

通过令牌桶测量速率，超出部分直接 drop，不缓冲，适合入方向（ingress）强制限速 ：

报文到达 → 令牌桶有令牌 → conform → ok（放行） → 令牌桶空 → exceed → drop（丢弃）

特点：丢包，不增加延迟。

mlx5 支持将 tc police action卸载到 ConnectX eSwitch 硬件，实现zero-CPU 限速。

传统软件 police 路径： 网卡 → 内核协议栈 → tc ingress hook → police action（CPU 执行）→ 放行/丢弃

mlx5 HW offload 路径： 网卡 eSwitch → 硬件 flow table 中的 meter 规则 → 放行/丢弃 （完全不经过 CPU，延迟 ～纳秒级）

---

# 完整配置步骤（mlx5 ingress 限速）

第一步：切换 switchdev 模式（HW offload 前提）

# 切换网卡到 switchdev（eSwitch）模式

# devlink dev eswitch set pci/0000:03:00.0 mode switchdev

# \# 开启硬件 TC offload 能力

# ethtool -K p0 hw-tc-offload on

# ethtool -K pf0vf0 hw-tc-offload on # VF representor 也要开

第二步：挂载 ingress qdisc

# tc qdisc add dev p0 ingress handle ffff:

# 注：ffff: 是 ingress qdisc 的内核保留固定句柄，不可更改。

第三步：添加 flower filter + police action

\# 对所有入方向 IP 流量限速 10Gbps

tc filter add dev p0 ingress protocol ip flower skip\_sw dst\_ip 10.0.0.0/24 ip\_proto tcp action police rate 10gbit burst 1mb conform-exceed pipe/drop action gact pass

第四步：验证是否 HW offload 成功

\# 查看 filter，确认出现 in\_hw 标志

tc -s -d filter show dev p0 ingress

\# 示例输出：

filter protocol ip pref 49148 flower chain 0

filter protocol ip pref 49148 flower chain 0 handle 0x1

eth\_type ipv4

ip\_proto tcp

dst\_ip 10.0.0.0/24

skip\_sw

in\_hw in\_hw\_count 1

action order 1: police 0x5 rate 10Gbit burst 1046250b mtu 2Kb action pipe/drop overhead 0b linklayer ethernet

ref 1 bind 1 installed 27 sec used 27 sec

Action statistics:

Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0)

backlog 0b 0p requeues 0

used\_hw\_stats delayed

action order 2: gact action pass

random type none pass val 0

index 3 ref 1 bind 1 installed 27 sec used 27 sec

Action statistics:

Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0)

backlog 0b 0p requeues 0

used\_hw\_stats delayed

\# in\_hw in\_hw\_count 1 ← 说明已卸载到硬件

\# 查看 police action 统计（含硬件命中计数）

tc -s -d actions list action police

\# 查看网卡硬件计数器

ethtool -S p0 | grep -i drop

\# 删除 ingress qdisc（同时删除其下所有 filter）

tc qdisc del dev p0 ingress

# 常见场景速查

| 场景 | 方向 | 推荐方式 | mlx5 HW offload |
| --- | --- | --- | --- |
| 限制某 VF 入流量 | ingress（VF rep） | tc police + flower | ✓ |
| 限制某 IP 入流量 | ingress（物理口） | tc police + flower | ✓ |
| 出方向流量整形 | egress | HTB + flower | ✗（软件执行） |
| 多租户 QoS 隔离 | ingress per VF | tc police per VF rep | ✓ |
| OVS OpenFlow Meter | ingress/egress | OVS meter → 自动映射 tc police | ✓（CX-6 Dx+） |

---

# 关键约束

1.  skip\_sw 走HW卸载，只支持tc police；对egress HTB/TBF会报错。对于egress 硬件限速有强需求可以考虑vQoS：[MLX5 限速原理详解：vQoS、Meter、PPS/BPS 限速](https://mp.weixin.qq.com/s?__biz=MzkzODIxNjk5Mg==&mid=2247484840&idx=1&sn=52812393917dd25a4fa2e6bc5a0aca8d&scene=21#wechat_redirect)
    
2.  mlx5 硬件要求 burst > 0。
    
3.  内核版本 ≥ 5.12：tc police HW offload 的最低内核要求。
    
4.  需 ConnectX-5 及以上：CX-4 不支持 tc police offload。
    
5.  mtu 建议显式设置：使用 jumbo frame（9000）时不设置会导致大包无法匹配。
    

参考：

1.  https://geek-blogs.com/blog/linux-tc-qdisc/
    

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/cbbd8c4a_1780380934562?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzODIxNjk5Mg%3D%3D%26mid%3D2247484909%26idx%3D1%26sn%3Dba24ea98a028811af6682e158cf5d02e%26chksm%3Dc3a841a3f31a7b455dd84605e09ff62156a84ecb517a6b7648aea604437392c044636b60faa6%26mpshare%3D1%26scene%3D1%26srcid%3D0602e5sVZ7A4W7ktTnNbuRYC%26sharer_shareinfo%3D0d66a8445c88f6486ab9d963cf671c77%26sharer_shareinfo_first%3D0d66a8445c88f6486ab9d963cf671c77%23rd&s=obsidian)