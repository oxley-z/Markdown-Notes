---
author: hb的小羊
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg2NDkwODQzNw==&mid=2247484443&idx=1&sn=32488407af7e6f22a34a21b8b66005c1&chksm=cff92ce829b9585d7420d2ebd7bcd08c0a8d6fb3b58ccc520aa884e1e112a66ca8226efdec14&mpshare=1&scene=1&srcid=1002YTALqQYVJ8Unj53UCjLP&sharer_shareinfo=2befd985bb515738d063d89c7b61e6c1&sharer_shareinfo_first=2befd985bb515738d063d89c7b61e6c1#rd
saved: 2026-10-02 11:43:33
tags:
  - 笔记同步助手
id: 091bdaa3-5a32-4d42-a38c-cecbd339279d
---

公众号名称：hb的小羊

作者名称：hb的小羊

发布时间：2026-10-02 08:20

lspci -t ：打印PCI设备树，树状结构一眼识别PCIe Switch，判断是否存在PCIe漏斗

参数补充：

lspci -t -vv 同时打印树+详细信息

lspci -tv 带设备名称，可读性更强（推荐日常用）

# 输出符号含义

\- 根端口（Root Port）：CPU内置PCIe根控制器

└─ PCIe Switch：PCIe交换机芯片

├─ 下游端口 → GPU / 网卡 / Retimer

\- -\[0000\] ：PCI域号，一般单主机都是0000

\- 00:00.0 ：BDF 总线:设备.功能号

\- 缩进代表父子关系：父节点 = 上游；子节点 = 下游挂载设备

# 核心判断规则：

多个GPU/网卡，子节点都隶属于同一个PCIe Switch父节点 → 这些设备共享这颗Switch的上行PCIe链路 → 存在PCIe漏斗风险！

# 示例1：有PCIe漏斗拓扑（8GPU挂载同一个Switch）

\-\[0000\]00:00.0 Root Port

└─01:00.0 PCIe Switch （上行x16 PCIe5.0，瓶颈口！）

├─02:00.0 GPU0

├─03:00.0 GPU1

├─04:00.0 GPU2

├─05:00.0 GPU3

├─06:00.0 GPU4

├─07:00.0 GPU5

├─08:00.0 GPU6

└─09:00.0 GPU7

# 解读：

8个GPU全部挂在同一颗PCIe Switch下面，所有GPU的流量上行都要挤 01:00.0 这一条PCIe链路。

# 典型PCIe带宽漏斗场景

# 示例2：无漏斗，两颗Switch拆分（4卡一组）

plaintext

\-\[0000\]00:00.0 Root Port

├─01:00.0 PCIe Switch A

│ ├─02:00.0 GPU0

│ ├─03:00.0 GPU1

│ ├─04:00.0 GPU2

│ └─05:00.0 GPU3

└─10:00.0 PCIe Switch B

├─11:00.0 GPU4

├─12:00.0 GPU5

├─13:00.0 GPU6

└─14:00.0 GPU7

# 解读：

分成两组，各自独立上行通道。A组4卡共享上行；B组4卡共享上行。

漏斗风险下降，但每组内部4卡并发时依然有漏斗。

# 示例3：HGX SXM机型（GPU直连CPU根端口，无Switch）

\-\[0000\]00:00.0 Root Port

├─01:00.0 GPU0

├─02:00.0 GPU1

├─03:00.0 GPU2

└─04:00.0 GPU3

\-\[0000\]80:00.0 Root Port

├─81:00.0 GPU4

├─82:00.0 GPU5

├─83:00.0 GPU6

└─84:00.0 GPU7

# 解读：

GPU直接挂载CPU根端口，中间没有PCIe Switch。

GPU之间通信走NVLink，不走PCIe。✅ 不存在PCIe带宽漏斗。

# 示例4：网卡和GPU共用同一个Switch（坑！RDMA打流必踩）

\-\[0000\]00:00.0 Root Port

└─01:00.0 PCIe Switch

├─02:00.0 GPU0

├─03:00.0 GPU1

└─04:00.0 Thor2 400G网卡

# 解读：

GPU的H2D/D2H流量、Thor2的GPUDirect RDMA流量，争抢同一条Switch上行带宽。

RDMA打流 + GPU同时跑训练，带宽直接撞瓶颈。

最佳实践：高速RDMA网卡，单独挂载CPU根端口，不和GPU共享Switch。

# 配套查看命令组合（运维排查一键执行）

# 1\. 打印PCI树，带设备名称（首选）

lspci -tv

# 2\. 查看设备链路速率与宽度，确认是不是PCIe5.0 x16

lspci -vv | grep -E 'LnkCap|LnkSta'

# 3\. 过滤只看GPU和PCIe Switch

lspci -tv | grep -E 'NVIDIA|Broadcom|PCI bridge'

LnkSta：当前协商的链路速度、宽度；

LnkCap：硬件支持最大链路规格。

如果 LnkSta显示 Speed 8GT/s, Width x8 → 链路降级，和漏斗是两个问题，不要混淆！

# 总结：

树状缩进看父子；同Switch下多设备；

共享一条上行路；并发流量漏斗出；

GPU直连根端口；没有Switch无此苦。

# 常见故障：

1.  Retimer 不会产生漏斗：Retimer只是信号放大中继，不做带宽复用；只有PCIe Switch才会产生带宽漏斗，不要把Retimer和Switch搞混。
    
2.  只看拓扑不能判断瓶颈大小：还要确认上行链路是x16/x8，PCIe4.0还是5.0。
    
3.  单卡跑带宽正常，不代表没有漏斗：漏斗是多设备并发才会暴露的问题。
    

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/d4634dd1_1790912612527?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg2NDkwODQzNw%3D%3D%26mid%3D2247484443%26idx%3D1%26sn%3D32488407af7e6f22a34a21b8b66005c1%26chksm%3Dcff92ce829b9585d7420d2ebd7bcd08c0a8d6fb3b58ccc520aa884e1e112a66ca8226efdec14%26mpshare%3D1%26scene%3D1%26srcid%3D1002YTALqQYVJ8Unj53UCjLP%26sharer_shareinfo%3D2befd985bb515738d063d89c7b61e6c1%26sharer_shareinfo_first%3D2befd985bb515738d063d89c7b61e6c1%23rd&s=obsidian)