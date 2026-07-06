---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487451&idx=1&sn=dacbcb49e8dee741454298396f4cc7ce&chksm=c084075bce1c4972b37b08bdbaf26599aa369f8a4582738909801fe2dc7b2de55ade57e76500&mpshare=1&scene=1&srcid=0705fEYIKOSB0YSwrcGLH9hB&sharer_shareinfo=ce4bf0c3294469b04981e2b1328a908a&sharer_shareinfo_first=ce4bf0c3294469b04981e2b1328a908a#rd
saved: 2026-07-05 19:17:27
tags:
  - 笔记同步助手
id: 9212d8d7-813b-4589-b45c-7e12d799069b
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-05 18:00

## 一、两种传输路径（有无GDR天差地别）

## 1）传统普通RDMA（无GDR，双拷贝）

### 数据路径（跨节点GPU通信）：

本地GPU显存 → cudaMemcpy D2H → 主机内存 → RNIC RDMA发网络

远端RNIC → 远端主机内存 → cudaMemcpy H2D → 远端GPU显存

问题：两次内存拷贝、占用PCIe带宽、CPU参与数据搬运、延迟高。

## 2）GPUDirect RDMA（GDR，零主机内存中转）

底层依赖 PCIe P2P（对等访问）：RNIC直接通过PCIe读写GPU显存，绕过主机内存做中转缓冲区 。

  

### 数据路径：

本地GPU显存 ←PCIe P2P→ RNIC → 网络 → 远端RNIC ←PCIe P2P→ 远端GPU显存

## 重要区分：

\- 数据面：没有主机内存拷贝，CPU不搬数据；

\- 控制面（默认GDR）：依然需要CPU提交RDMA WR（工作请求）到网卡SQ/RQ队列；

\- 更进一步：GPUDirect Async(GDA/GDAKI)：CUDA Kernel直接提交WR，控制面也绕开CPU。

## 二、底层硬件与软件前提

## 硬件条件

1\. GPU：Kepler及以上NVIDIA数据卡（A100/H100/B200等）；

2\. RNIC：Mellanox ConnectX-5/6/7（支持PCIe P2P）；

3\. 关键约束：GPU与RNIC必须挂在同一个PCIe Root Complex下（同一个上游PCIe交换机），否则P2P不通，GDR无法启用 ；

4\. 开启IOMMU、PCIe P2P内核选项。

## 软件核心机制

1\. CUDA分配显存（ cudaMalloc ）；

2\. 通过NVIDIA驱动拿到显存的P2P token、DMA-BUF fd；

3\. RDMA verbs驱动调用特殊接口（ reg\_user\_mr\_dmabuf ），把GPU显存直接注册为RDMA的MR（内存区域），生成Lkey/Rkey；

4\. 内核打通地址映射：RNIC的DMA引擎可以拿到显存对应的IOVA地址，直接发起PCIe DMA读写显存；

5\. ODP（On-Demand Paging）新方案：不用预先Pin整个显存，缺页时动态挂载显存页面，降低资源开销 。

## 三、完整工作流程（标准GDR，CPU负责控制）

### 1\. 初始化阶段（CPU执行一次）

\- 分配GPU设备内存；

\- 向NVIDIA驱动获取显存P2P访问令牌；

\- RDMA驱动将显存注册成MR，获得访问密钥；

\- 建立QP（队列对）连接。

### 2\. 发送流程（CPU提交指令，数据面无CPU参与）

1\. GPU Kernel完成计算，数据留在显存；

2\. CPU在用户态构造RDMA Send/Write WR，填入GPU显存虚拟地址 + MR密钥，投递到RNIC发送队列SQ；

3\. RNIC硬件从SQ取出WR，通过PCIe P2P直接DMA读取GPU显存；

4\. RNIC封装数据包，通过RoCE/IB发给对端；

5\. 数据全程没有进入主机内存。

### 3\. 接收流程

1\. 对端RNIC收到报文；

2\. RNIC通过PCIe P2P直接把数据DMA写入远端GPU显存；

3\. 传输完成后，RNIC生成CQ（完成队列）事件通知CPU；

4\. CPU通知GPU Kernel可以读取显存数据。

一句话总结：CPU只管发“快递订单（WR）”，RNIC硬件自己上门去GPU显存取货/送货，不用经过CPU内存仓库。

## 四、进阶：GPUDirect Async (GDA/GDAKI)

标准GDR只是优化数据面，控制指令还是CPU发。

GDA允许：

\- 在GPU显存中分配RDMA队列（SQ/RQ/CQ）；

\- CUDA Kernel直接向网卡提交WR；

\- 控制路径也绕开CPU；

常用于超大规模NCCL AllReduce，进一步降低CPU开销。

## 五、典型应用场景

### 1\. 大模型分布式训练（NCCL核心场景）

多机多卡AllReduce，GDR是NCCL最高带宽方案，避免D2H/H2D拷贝，大幅提升集群有效算力；

2\. GPU推理集群（多机GPU之间交换张量）；

3\. HPC科学计算多节点GPU通信；

4\. BlueField DPU场景：DPU内置RNIC，和GPU做GDR互通。

## 六、容易混淆的3个GPUDirect技术

1\. GPUDirect P2P：同一节点内GPU ↔ GPU直接互通（NVLink优先，其次PCIe P2P）；

2\. GPUDirect RDMA(GDR)：GPU ↔ RNIC网卡直接互通（跨节点通信）；

3\. GPUDirect Storage(GDS)：GPU ↔ NVMe存储直接互通，不经过主机内存。

## 七、常见限制

1\. PCIe拓扑约束：GPU和网卡不能跨Root Complex；

2\. 虚拟化下SR-IOV需要特殊配置，vGPU对GDR支持有限；

3\. 传统verbs必须注册MR，占用资源；ODP缓解该问题；

4\. Windows原生不支持GDR，仅Linux。

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/8dd04e48_1783250246214?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487451%26idx%3D1%26sn%3Ddacbcb49e8dee741454298396f4cc7ce%26chksm%3Dc084075bce1c4972b37b08bdbaf26599aa369f8a4582738909801fe2dc7b2de55ade57e76500%26mpshare%3D1%26scene%3D1%26srcid%3D0705fEYIKOSB0YSwrcGLH9hB%26sharer_shareinfo%3Dce4bf0c3294469b04981e2b1328a908a%26sharer_shareinfo_first%3Dce4bf0c3294469b04981e2b1328a908a%23rd&s=obsidian)