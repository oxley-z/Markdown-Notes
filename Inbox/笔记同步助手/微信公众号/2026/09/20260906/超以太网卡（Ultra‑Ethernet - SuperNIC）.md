---
author: hb的小羊
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg2NDkwODQzNw==&mid=2247483928&idx=1&sn=749490edde58259877b06a15fe124880&chksm=cf6a4e7fd7eb5d264f29f50f7d8894856e6f217d345ce598068e8e35dc8402a4ae40d0bd0e02&mpshare=1&scene=1&srcid=0906f2NaJiviKPbBZBZPqfi5&sharer_shareinfo=5752024ff93b90ebf6f16982a20d8130&sharer_shareinfo_first=5752024ff93b90ebf6f16982a20d8130#rd
saved: 2026-09-06 12:36:08
tags:
  - 笔记同步助手
id: d3f263db-fbab-4f5d-a176-cb63a57a5e55
---

公众号名称：hb的小羊

作者名称：hb的小羊

发布时间：2026-09-06 10:25

1\. SuperNIC：NVIDIA定义，面向AI集群的超级网卡（ConnectX‑8/9），搭配Spectrum‑X交换机，增强版RoCE网卡 。

2\. Ultra‑Ethernet（UEC超以太网）：开放联盟标准（博通、AMD、联想等），新一代传输协议UET，替代传统RoCE‑v2，不依赖PFC无损网络，面向十万卡AI集群 。

日常运维口语“超以太网卡”，泛指支持Ultra‑Ethernet(UET)协议的新一代AI加速网卡。

## 一、传统RoCE‑v2痛点（为什么要超以太网）

传统RoCE‑v2大规模集群短板：

1\. 强依赖PFC优先级流控无损网络；PFC配置错误容易死锁、全网震荡，调试难度极高。

2\. QP队列对连接模型，集群规模上万卡，QP资源耗尽，扩展性受限。

3\. ECMP等价路由报文必须保序，不能包粒度多路径，带宽利用率上不去。

4\. 丢包之后Go‑Back‑N回退重传，尾部延迟暴涨，NCCL All‑Reduce性能跳水。

超以太网UET核心目标：在普通以太网（有损网络）上，拿到接近IB InfiniBand的RDMA性能，摆脱PFC强依赖，支持十万卡规模集群。

## 二、超以太网UET核心技术特性

### 1\. 不需要PFC无损网络

依靠链路层重传LLR、选择性重传，网络允许丢包，硬件快速恢复，规避PFC死锁灾难，运维复杂度大幅下降。

### 2\. 报文级多路径 Packet‑Spray（包喷射）

同一个数据流的不同数据包可以走交换机多条不同路径，真正负载均衡，带宽利用率90‑98%；传统RoCE只能流粒度负载均衡。

### 3\. 支持乱序交付

接收端硬件直接把乱序报文写入GPU内存，不需要网卡内部重排序缓存，降低延迟、节省网卡缓存资源 。

### 4\. 无连接传输模型PDC

抛弃传统RDMA QP队列对；不需要预先建立大量连接，大规模集群资源不再成为瓶颈，支持数万‑十万卡扩展 。

### 5\. 全新拥塞控制NSCC

专门针对AI incast（多对一并发涌入）场景优化；大模型all‑reduce通信下拥塞响应更快。

### 6\. 原生安全加密

硬件AES加密、防重放攻击，端到端安全卸载。

向下兼容普通以太网IP协议；旧交换机不需要全部替换，但完整UET能力需要新一代支持UEC的交换机。

## 三、SuperNIC（NVIDIA） VS Ultra‑Ethernet（UEC开放标准）

![[Inbox/笔记同步助手/微信公众号/2026/09/images/9839f11ea98078ba26f584399fcf288d_MD5.jpg]]

重点：SuperNIC ≠ Ultra‑Ethernet网卡，很多人把两者混为一谈。SuperNIC是NVIDIA硬件产品；Ultra‑Ethernet是行业开放协议标准。

## 四、超以太网卡对比传统RoCE HCA网卡

![[Inbox/笔记同步助手/微信公众号/2026/09/images/b281aac3e6bd41d384378abe1ce7b258_MD5.jpg]]

  

## 五、硬件代表型号

### 1\. NVIDIA SuperNIC

\- ConnectX‑8 SuperNIC：PCIe6.0，内置PCIe交换，双口400G Ethernet / 单口800G IB

\- ConnectX‑9 SuperNIC：单端口800G以太网，面向下一代AI工厂

### 2\. Ultra‑Ethernet（UEC联盟）

\- Broadcom Thor Ultra 800G：业界首款正式UEC标准800G超以太网卡，PCIe6.0，OCP/PCIe形态 。

## 六、运维层面关键点

### 1\. 驱动版本是核心

传统MLNX‑OFED不完全支持UET超以太网协议；UEC网卡需要新一代驱动栈，libfabric作为上层接口。

### 2\. 交换机必须支持对应特性

\- SuperNIC：要Spectrum‑X交换机，才能开启自适应路由、拥塞遥测；普通交换机只能跑基础RoCE。

\- UEC超以太网：需要支持UET协议交换机；老交换机只能跑普通RoCE/TCP，发挥不出超以太网全部能力。

### 3\. 不是插上就能用，端到端整套协同

网卡、交换机、驱动、通信库（NCCL/CCL、MPI）必须配套；只换网卡，交换机不变，拿不到UET的性能收益。

### 4\. 向下兼容

超以太网卡依然支持传统RoCE‑v2、普通TCP，可以混合旧业务，支持平滑升级。

## 七、优缺点总结

优点

1\. 大规模集群不再被PFC死锁困扰，运维难度下降。

2\. 报文级多路径，网络带宽利用率大幅提升。

3\. 集群可扩展上限从万卡迈向十万卡级别，适配下一代大模型AI工厂。

4\. 网络轻微丢包场景依然可以高性能RDMA，降低对布线、光模块苛刻要求。

缺点

1\. 整套软硬件生态较新，成熟度不如多年打磨的RoCE‑v2、IB。

2\. 需要新一代网卡+交换机，硬件升级成本高。

3\. 老工具链（旧OFED、旧NCCL版本）不支持，全套软件栈需要升级。

## 八、选型:

\- 万卡以内规模：成熟方案选CX7 RoCE HCA + PFC无损，生态成熟，工具链完备。

\- 万卡向十万卡演进、想摆脱PFC噩梦：评估 Ultra‑Ethernet(UEC)超以太整套方案。

\- NVIDIA体系，Spectrum‑X交换机：选 SuperNIC（CX8/CX9）增强RoCE方案。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/e4012440_1788669366155?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg2NDkwODQzNw%3D%3D%26mid%3D2247483928%26idx%3D1%26sn%3D749490edde58259877b06a15fe124880%26chksm%3Dcf6a4e7fd7eb5d264f29f50f7d8894856e6f217d345ce598068e8e35dc8402a4ae40d0bd0e02%26mpshare%3D1%26scene%3D1%26srcid%3D0906f2NaJiviKPbBZBZPqfi5%26sharer_shareinfo%3D5752024ff93b90ebf6f16982a20d8130%26sharer_shareinfo_first%3D5752024ff93b90ebf6f16982a20d8130%23rd&s=obsidian)