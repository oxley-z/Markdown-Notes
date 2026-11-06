---
author: hb的小羊
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg2NDkwODQzNw==&mid=2247484446&idx=1&sn=07bd14cdcf5ebc0bd2036bf972a4d9d7&chksm=cfc88d81fc7726abebc6baa7377dcd2c8ee443c15dc478dd627edf9ea849b9588cd5edfb22fd&mpshare=1&scene=1&srcid=1005MNjOxoDDLMICpKUJnhTx&sharer_shareinfo=42800c112b94dca849bc4e0507489747&sharer_shareinfo_first=42800c112b94dca849bc4e0507489747#rd
saved: 2026-11-06 10:31:08
tags:
  - 笔记同步助手
id: 7a8e1ebe-3b76-41e3-9227-6f6c132ce4a3
---

公众号名称：hb的小羊

作者名称：hb的小羊

发布时间：2026-10-05 06:06

H2D（Host‑to‑Device）：主机内存 → GPU显存；

D2H（Device‑to‑Host）：GPU显存 → 主机内存；

数据走PCIe总线，由GPU内部Copy Engine（DMA拷贝引擎）完成，不占用CUDA核心 。

区分：H2D/D2H是CPU‑GPU之间；D2D是GPU‑GPU（PCIe P2P / NVLink）。

# 核心性能指标

# 1、有效带宽（Effective Bandwidth，单位 GB/s）

计算公式： 带宽 = 传输字节大小 ÷ 传输耗时 ，多次迭代取平均值 。

-   H2D带宽：主机内存拷贝到GPU显存吞吐
    
-   D2H带宽：GPU显存拷贝回主机内存吞吐
    

理论峰值参考（PCIe x16单向理论上限）

| PCIe版本 | 单向理论峰值 | 实测理想值（pinned锁页内存） |
| --- | --- | --- |
| Gen3 x16 | **15.75GB/s** | 12‑14GB/s |
| Gen4 x16 | **31.5GB/s** | 25‑28GB/s |
| Gen5 x16 | 63GB/s | 50‑56GB/s |

  

注意：

1.  必须使用锁页内存(pinned memory)，普通pageable内存带宽会明显下跌；
    
2.  小报文带宽低：PCIe有协议开销，只有大报文（>几MB）才能跑满带宽；
    
3.  H2D、D2H单向带宽不完全相等，硬件微小差异属于正常。
    

# 2、传输时延 Latency（单位 μs / ms）

-   H2D Latency：一次主机到GPU拷贝的耗时
    
-   D2H Latency：一次GPU回传给主机拷贝耗时
    

分为：

1.  单报文时延：固定开销，小数据包占主导；
    
2.  批量平均时延：大数据，带宽主导。
    

小数据包（KB级），时延是瓶颈；大数据（GB级），带宽是瓶颈。

# 3、双向带宽（Bidirectional）

同时跑H2D+D2H，两个方向叠加。GPU有多套Copy Engine，理想下可以双向并行；但PCIe总线是半双工，双向总和不会超过单向理论峰值。

# 4、关键衍生指标（业务场景）

## 1\. 拷贝与计算重叠效率

异步拷贝 cudaMemcpyAsync 配合CUDA Stream，可以让H2D/D2H数据传输 和 GPU kernel计算并行执行，掩盖传输开销。

理想：传输时间被完全掩盖；差：拷贝串行，总耗时=计算+拷贝时间。

## 2\. 小消息开销（overhead）

每次拷贝调用有固定驱动/PCIe开销，大量细碎小拷贝会严重拉低整体性能，即使单测带宽正常。

优化：合并小buffer，减少拷贝调用次数。

## 3\. P95/P99时延抖动

生产环境不只是看平均时延，要看长尾；系统内存压力、NUMA错位、其他设备抢占PCIe，会造成D2H/H2D时延抖动。

# 影响H2D/D2H性能的关键条件

## 1\. Host内存类型

-   cudaMallocHost 锁页(pinned)内存：DMA直接访问，高性能；
    
-   普通malloc可分页内存：CUDA驱动内部临时拷贝一份到锁页缓冲区，带宽下降、CPU占用上升。
    

## 2\. NUMA绑定

GPU绑定的CPU NUMA节点，host内存分配要落在对应NUMA节点；跨NUMA访问H2D/D2H带宽直接腰斩。

## 3\. 同步/异步API

-   cudaMemcpy ：同步拷贝，CPU阻塞等待拷贝完成；
    
-   cudaMemcpyAsync ：异步非阻塞，配合stream实现计算拷贝重叠，生产环境优先。
    

## 4\. PCIe链路状态

确认显卡跑在PCIe x16，不是降速x8/x4；BIOS、主板插槽、硬件故障都会降带宽。

# 测试工具

## 1\. nvbandwidth（NVIDIA官方）

专门测H2D/D2H、D2D单向/双向带宽时延，支持多GPU，工业验收标准工具 。

# H2D D2H单向测试

nvbandwidth -t h2d,d2h

## 2\. cuda示例 bandwidthTest

CUDA toolkit自带示例，简单快速验证H2D/D2H带宽。

## 3\. Nsight‑Systems（nsys）

性能剖析工具，可以抓取真实业务下H2D/D2H的耗时、次数、时延分布，定位细碎拷贝瓶颈。

## 4\. trtexec

TensorRT工具， --includeDataTransfers ，输出推理链路的H2D‑Latency、D2H‑Latency指标 。

注意：

1.  大报文带宽达不到PCIe理论的80%：检查是否pinned内存、NUMA、PCIe链路降速。
    
2.  大报文带宽正常，业务性能很差：业务大量细碎小拷贝，看nsys统计拷贝调用次数。
    
3.  D2H时延抖动大：主机内存压力大，swap、跨NUMA。
    

总结：

H2D/D2H走PCIe，DMA拷贝引擎完成，不耗CUDA核心；

两大核心指标：有效带宽GB/s、传输时延μs；

锁页内存、NUMA绑定、报文大小、异步stream，是决定性能四大关键；

小报文看时延，大报文看带宽；优先用异步拷贝做计算传输重叠。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/290802df_1793932267112?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg2NDkwODQzNw%3D%3D%26mid%3D2247484446%26idx%3D1%26sn%3D07bd14cdcf5ebc0bd2036bf972a4d9d7%26chksm%3Dcfc88d81fc7726abebc6baa7377dcd2c8ee443c15dc478dd627edf9ea849b9588cd5edfb22fd%26mpshare%3D1%26scene%3D1%26srcid%3D1005MNjOxoDDLMICpKUJnhTx%26sharer_shareinfo%3D42800c112b94dca849bc4e0507489747%26sharer_shareinfo_first%3D42800c112b94dca849bc4e0507489747%23rd&s=obsidian)