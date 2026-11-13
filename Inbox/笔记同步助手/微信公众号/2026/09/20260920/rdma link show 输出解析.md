---
author: hb的小羊
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg2NDkwODQzNw==&mid=2247484336&idx=1&sn=d638487251c4f11036424d372af225db&chksm=cffaebac8b35bbaf8242586f51046c98e6004e6d11e6bbfb3009fa2fbf16b14034216b854310&mpshare=1&scene=1&srcid=0920jIrjVJ7LcxwnOER1Opfw&sharer_shareinfo=d52faa6a3c8e61fa27014b3bd1eea250&sharer_shareinfo_first=d52faa6a3c8e61fa27014b3bd1eea250#rd
saved: 2026-09-20 19:03:59
tags:
  - 笔记同步助手
id: 2ad9883f-620f-409c-b126-5faec05025d5
---

公众号名称：hb的小羊

作者名称：hb的小羊

发布时间：2026-09-19 08:56

rdma link show 是内核rdma‑core自带工具，不需要MLNX‑OFED；用来查看RDMA设备‑网卡映射、端口状态，是 ibdev2netdev 的系统原生替代工具 。

前提：RDMA内核模块已经加载（openibd启动，或者udev自动加载）。

# 基础命令

# 查看全部RDMA链路

rdma link show

# 只看指定RDMA设备，例如mlx5\_0

rdma link show mlx5\_0

# 只看指定端口，mlx5\_0的端口1

rdma link show mlx5\_0/1

# RoCE(以太网RDMA)典型输出

link mlx5\_0/1 state ACTIVE physical\_state LINK\_UP netdev ens1f0

link mlx5\_1/1 state ACTIVE physical\_state LINK\_UP netdev ens1f1

# IB(InfiniBand)典型输出

link mlx5\_0/1 state ACTIVE physical\_state LINK\_UP netdev ib0

# 字段逐个解释

link mlx5\_0/1 state ACTIVE physical\_state LINK\_UP netdev ens1f0

## 1\. mlx5\_0/1

-   mlx5\_0 ：RDMA设备名（和 ibv\_devinfo 看到一致）
    
-   /1 ：物理端口号（网卡端口1）
    

## 2\. state ACTIVE — RDMA端口逻辑状态

-   ACTIVE ：正常可用，RDMA verbs可以使用（ib\_write\_bw、nccl‑test可以跑）
    
-   INIT ：初始化中，还没就绪
    
-   ARMED ：IB子网还没完成配置（IB网络常见）
    
-   DOWN ：RDMA端口不可用
    

## 3\. physical\_state LINK\_UP — 物理链路状态

-   LINK\_UP ：物理层链路已经起来，光模块/光纤/交换机端口已经协商成功
    
-   POLLING ：正在训练链路，光模块没插好、光纤故障、交换机端口down
    
-   DISABLED ：端口被固件禁用（VPI模式错误）
    

重点区分：

-   state ：RDMA协议层状态
    
-   physical\_state ：物理光链路状态
    

有可能物理 LINK\_UP ，但RDMA state是 DOWN ，代表RDMA驱动/配置异常。

## 4\. netdev ens1f0

-   RoCE：对应系统以太网网卡名（ens1f0/ethX）
    
-   IB：对应IPoIB虚拟网卡名（ib0）
    

这就是和 ibdev2netdev 对应的映射关系。

# 对比 ibdev2netdev

| 项目 | ibdev2netdev | rdma link show |
| --- | --- | --- |
| 来源 | MLNX‑OFED自带 | 内核rdma‑core原生，系统自带 |
| 是否需要OFED | 必须 | 不需要 |
| 输出内容 | RDMA设备↔网卡，链路UP/DOWN，还能显示PCI、固件版本 | 只输出设备、端口、state、physical\_state、netdev |
| 新版RHEL9 | 已移除 | 可用 |

小提示： rdma link show 不显示固件、PCI地址；想要看PCI、固件，还是需要 ibdev2netdev -v （装OFED）或者 ibv\_devinfo 。

# 常见故障现象

现象1：命令执行无输出

1.  RDMA内核模块没加载；
    
2.  openibd stop；
    
3.  网卡没有被识别；
    

排查： lsmod | grep mlx5\_core ，确认模块加载。

现象2： physical\_state POLLING

物理层没通：光纤松动、光模块损坏、交换机端口down、光功率异常。

现象3： physical\_state LINK\_UP ，但是 state DOWN

物理链路正常，但RDMA端口逻辑异常。

-   RoCE：网卡端口模式不是ETH模式（VPI配置错误）
    
-   IB：子网管理器opensm没有正常工作
    

现象4：有 netdev ，但是 state=DOWN

RDMA功能不可用，即使以太网网卡可以ping通， ib\_write\_bw 也无法跑流。

# 配套常用组合排查命令

# 1.查看RDMA链路映射

rdma link show

# 2.查看HCA详细信息，速率、链路层（ETH/IB）

ibv\_devinfo

# 3.查看GID（RoCE关键，GID是否生成）

rdma tool gids show

# 4.查看网卡物理链路

ip link show ens1f0

# 总结

rdma link show 是内核原生工具，替代ibdev2netdev，不需要MLNX‑OFED；输出字段：RDMA设备/端口、state(RDMA逻辑状态)、physical\_state(物理链路)、netdev(对应网卡)。

state ACTIVE+physical\_state LINK\_UP代表RDMA端口完全就绪；physical\_state POLLING代表物理链路故障；state DOWN即使网口ping通，RDMA也无法工作。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/020c21be_1789902238006?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg2NDkwODQzNw%3D%3D%26mid%3D2247484336%26idx%3D1%26sn%3Dd638487251c4f11036424d372af225db%26chksm%3Dcffaebac8b35bbaf8242586f51046c98e6004e6d11e6bbfb3009fa2fbf16b14034216b854310%26mpshare%3D1%26scene%3D1%26srcid%3D0920jIrjVJ7LcxwnOER1Opfw%26sharer_shareinfo%3Dd52faa6a3c8e61fa27014b3bd1eea250%26sharer_shareinfo_first%3Dd52faa6a3c8e61fa27014b3bd1eea250%23rd&s=obsidian)