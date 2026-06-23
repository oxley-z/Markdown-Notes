---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487415&idx=1&sn=1f9a3b9369249bbe361830048d7e51cc&chksm=c043f424ba16a4fa8956b366fa1207637983bb9f61fa0c39dab3e4da69d38d2916b1e009028a&mpshare=1&scene=1&srcid=0613JAOyfruAawUw5b61bXuS&sharer_shareinfo=d3c2781bde5744a90be32dc5972e182b&sharer_shareinfo_first=d3c2781bde5744a90be32dc5972e182b#rd
saved: 2026-06-13 09:41:16
tags:
  - 笔记同步助手
id: 4be4c3ac-fa6a-4c9c-b74c-52a9a0df2072
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-06-09 07:27

途分为 连通性验证、性能基准、调试抓包、硬件固件、拓扑诊断、上层应用压测、系统调试 7大类，覆盖 RoCE/InfiniBand/Soft-RoCE(rxe)。

## 一、连通性快速验证工具（最常用，对应 rping）

### 1\. rping（rdma-core 自带，RoCE/IPoRDMA 首选）

\- 作用：基于 RDMA CM + IP 像 ping 一样测试两端RDMA可达性，支持RC/UD连接，输入IP即可，无需LID/GID，RoCE环境标配 。

\- 用法：

\# 服务端

rping -s

\# 客户端

rping -c 10 192.170.50.101

### 2\. ibv\_\*\_pingpong（libibverbs-utils）

底层verbs直通测试，验证QP创建、内存注册、收发通路：

\- ibv\_rc\_pingpong ：RC可靠连接（业务最常用）

\- ibv\_uc\_pingpong ：UC不可靠连接

\- ibv\_ud\_pingpong ：UD无连接组播

\- ibv\_srq\_pingpong ：共享接收队列SRQ功能验证

### 3\. ibping（infiniband-diags，纯IB子网用LID）

IB专用ping，依赖子网管理器SM，用端口LID通信，不依赖IP，仅InfiniBand适用，RoCE基本不用 。

### 4\. ucmatose / udaddy

RDMA CM 连接深度测试：验证RDMA CM建立、断开、多流并发、大数据传输，排查QP资源、GID解析问题。

### 5\. rdma\_server / rdma\_client

简易RDMA读写收发测试，轻量ping-pong，快速验证RDMA Write/Read基础功能。

## 二、性能基准测试套件 perftest（工业标准带宽/时延工具）

直接调用libibverbs，无内核协议栈开销，测Send/Write/Read/Atomic四大RDMA原语，分 \_bw (带宽)、 \_lat (延迟)两类工具 ：

## 带宽测试

\- ib\_send\_bw ：Send/Recv 消息收发带宽

\- ib\_write\_bw ：RDMA Write 远程写带宽（AI集群主流）

\- ib\_read\_bw ：RDMA Read 远程读带宽

\- ib\_atomic\_bw ：原子操作（FetchAdd/CAS）吞吐

## 延迟测试

\- ib\_send\_lat 、 ib\_write\_lat 、 ib\_read\_lat 、 ib\_atomic\_lat

## 补充工具

\- raw\_ethernet\_bw/lat ：裸以太网层RoCE性能（绕过IP）

\- run\_perftest\_loopback ：本机回环自测HCA硬件

## 三、多协议通用压测 qperf

同时支持 TCP/UDP/RDMA，一套命令对比传统TCP与RDMA带宽、延迟、CPU占用，适合做性能对比基线，无需记忆perftest多套命令 。

\# 服务端

qperf

\# 客户端测RDMA write延迟

qperf 192.170.50.101 rdma\_write\_lat

## 四、硬件信息 & 网卡状态查询（排查基础环境）

## libibverbs-utils

\- ibv\_devices ：列出本机所有RDMA网卡HCA

\- ibv\_devinfo ：查看HCA固件、GID、端口状态、MTU、RoCE/IB模式、硬件能力（最核心排查命令）

## infiniband-diags

\- ibstat / ibstatus ：端口速率、状态、LID、链路层

\- iblinkinfo ：全网链路拓扑、速率、错误计数

\- ibhosts ：列出IB子网所有节点

## mstflint（Mellanox/NVIDIA ConnectX专用）

mst status 、 mstflint 、 mlxconfig ：查看网卡硬件参数、开启RDMA、PFC/DCB配置、固件烧录、寄存器读写。

## 五、IB/ RoCE 网络拓扑 & 故障诊断

### 1\. ibnetdiscover ：扫描全网IB交换机、HCA，输出完整拓扑

2\. ibdiagnet ：全网巡检，统计CRC错误、PFC丢包、端口误码，输出诊断报告（集群巡检必备）

### 3\. ibtrace ：IB流量追踪，类似tcpdump抓IB报文

4\. rdma （rdma-core主工具）：统一管理rxe软RDMA、查看RDMA子系统、GID表、CM状态

## 六、抓包、系统调试工具（你提到的 ptrace 归类此处）

### 1\. ptrace（系统调试，非RDMA专用）

Linux原生进程追踪工具，用于调试RDMA用户态程序（追踪verbs调用、内存注册、QP操作、崩溃堆栈）；搭配 strace 追踪libibverbs系统调用：

strace -e trace=ibv\_ ./ib\_write\_bw

### 2\. tcpdump / wireshark（RoCE抓包）

RoCEv2走以太网，可用tcpdump抓取RoCE报文，wireshark支持RDMA/IB协议解析，排查PFC、ECN、GID、GRH头问题。

### 3\. perf

CPU/中断/内存性能剖析：测RDMA程序CPU占用、HCA中断绑核效率、大页内存使用、内核瓶颈。

### 4\. ibv\_asyncwatch

监控RDMA异步事件：端口up/down、QP错误、GID失效、CM断开，定位链路闪断问题。

## 七、上层应用 & 存储RDMA专用测试工具

### 1\. NVMe-oF：

nvmetcli 、 nvme rdma ，存储RDMA块设备压测

### 2\. Ceph RDMA：

ceph\_perf\_msgr\_client/server ，Ceph Messenger RDMA性能基准

### 3\. GPU Direct RDMA：

nd\_rping 、 nd\_write\_bw （Windows NetworkDirect，GPU跨机RDMA）

4\. NCCL：nccl-tests，AI集群多卡多机AllReduce通信测试（大模型训练专用RDMA测试）

## 八、软件RDMA (Soft-RoCE/rxe) 配套工具

\- rxe\_cfg （旧）/ rdma rxe （新rdma-core）：配置软RDMA网卡、创建rxe设备、管理IP映射GID

\- rping、perftest全套工具均兼容rxe，无需额外适配。

### 快速选型指南

### 1\. 仅验证两端RoCE通不通 → rping

2\. 测真实业务RDMA Write带宽/延迟 → ib\_write\_bw / ib\_write\_lat

### 3\. 对比TCP与RDMA性能差异 → qperf

### 4\. 查网卡GID、端口是否UP、固件版本 → ibv\_devinfo

### 5\. 追踪RDMA程序系统调用、崩溃调试 → strace + ptrace

### 6\. 全网集群错误巡检（IB/RoCE交换机）→ ibdiagnet

### 7\. AI多机集合通信测试 → nccl-tests

  

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/ce36cacb_1781314875249?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487415%26idx%3D1%26sn%3D1f9a3b9369249bbe361830048d7e51cc%26chksm%3Dc043f424ba16a4fa8956b366fa1207637983bb9f61fa0c39dab3e4da69d38d2916b1e009028a%26mpshare%3D1%26scene%3D1%26srcid%3D0613JAOyfruAawUw5b61bXuS%26sharer_shareinfo%3Dd3c2781bde5744a90be32dc5972e182b%26sharer_shareinfo_first%3Dd3c2781bde5744a90be32dc5972e182b%23rd&s=obsidian)