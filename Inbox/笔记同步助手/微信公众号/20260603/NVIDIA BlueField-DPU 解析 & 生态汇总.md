---
author: AI闲谈
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247491450&idx=1&sn=89e62a3664b3cc63596d7cfbdefe27e8&chksm=c255aa4a970202b00a2a5dd1ae71c949208e5cd2000d841f29fc4217fef6d7eaa0a69326b65d&mpshare=1&scene=1&srcid=0603xjsvf3fXea40tVrD0nKi&sharer_shareinfo=473583c0b7846e1b17ad90f1a2965324&sharer_shareinfo_first=473583c0b7846e1b17ad90f1a2965324#rd
saved: 2026-06-03 08:42:33
tags:
  - 笔记同步助手
id: a2e65b26-33ed-42ca-8514-b665aaa79ba4
---

公众号名称：AI闲谈

作者名称：AI闲谈

发布时间：2026-05-02 20:00

**

## 一、引言

本文全面梳理了 NVIDIA BlueField-DPU 的硬件代际、关键子系统（DPA / GDS / NVMe-oF RDMA）、网络互联、AI Factory 与大模型场景下的收益、CMX 上下文记忆存储等。信息比较多可能存在误差，以 NVIDIA 官方 Datasheet、DOCA SDK 等为准。

相关内容可以参考：

-   [Google TPU 系列之 TPUv8 全面解析](https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247491376&idx=1&sn=6fc3f56263911bb4baf6052106d15fb9&scene=21#wechat_redirect)
    
-   [全面解析 Google TPU 演进：从 TPUv1 到 TPUv7](https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247490894&idx=1&sn=224ac79fdaeaa86d2c58cf6005943e82&scene=21#wechat_redirect)
    
-   [全面解析 NVIDIA 最新硬件：Vera/Rubin/Rubin Ultra/NVL72/NVL144/LPX 等](https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247491311&idx=1&sn=7d995b502ce317b4ff5b3bd65b6c237c&scene=21#wechat_redirect)
    
-   [NVIDIA GTC2026 详细解读和分析](https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247491269&idx=1&sn=2bb86a97873105ef7e0e522a13ddaeb4&scene=21#wechat_redirect)
    
-   [全面解析 Amazon 自研 Trainium 系列芯片：从 Inferentia1 到 Trainium3](https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247491003&idx=1&sn=8a309bea743838681814cec4bfb19ae2&scene=21#wechat_redirect)
    
-   [全面梳理 AMD CDNA 架构 GPU：MI325X 等 8 种 A/GPU 介绍](https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247488222&idx=1&sn=282545e3e3c796edac8fe47b2918bfc7&scene=21#wechat_redirect)
    
-   [万卡 GPU 集群互联：硬件配置和网络设计](https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247486775&idx=1&sn=abf7af24181cf5189e113fb161cc8d30&scene=21#wechat_redirect)
    
-   [HPN 7.0：阿里云新一代万卡集群网络架构](https://mp.weixin.qq.com/s?__biz=Mzk0ODU3MjcxNA==&mid=2247487094&idx=1&sn=f0a94bff3b3cc6e88cb95c8f82551e0c&scene=21#wechat_redirect)
    

## 二、各代 BlueField DPU 对比

### 2.1 DPU 概念与 BlueField 平台定位

DPU（Data Processing Unit） 是继 CPU、GPU 之后的"第三颗主芯片"。它将网络、存储、安全、虚拟化等"基础设施税"（Infrastructure Tax，业内估计占数据中心 CPU 周期的 20%–30%）从 Host CPU 卸载到专用 SoC。

NVIDIA 的 BlueField 系列（NVIDIA BlueField Networking Platform）源自 2019 年收购 Mellanox 后的产品线，融合了 Mellanox ConnectX 系列网卡的硬件加速引擎、Arm 多核 SoC、PCIe Switch 与 NVIDIA 的 DOCA 软件栈，已成为云、HPC 与 AI Factory 的事实标准 DPU 之一。

BlueField 的核心价值为：

-   Offload：把 OVS、vRouter、NVMe-oF Initiator/Target、TLS/IPsec、防火墙等从 Host CPU 卸载，降低 CPU 负载。
    

-   Accelerate：硬件加速 RDMA、加密、压缩、正则匹配、流分类、Packet Pacing 等。
    
-   Isolate：Host 与基础设施控制面（Tenant vs. Operator）物理隔离，零信任架构的天然落点。
    

进入大模型时代后，BlueField 进一步承担了“AI Factory 的基础设施 OS”角色，成为 GPU 计算单元之外、为 AI 数据流水线提供网络、存储、安全、KV Cache 服务的核心芯片。

### 2.2 BlueField 详细数据与硬件结构

#### 2.2.1 BlueField-1（2017，Mellanox 时代）

如下图所示为 BlueField-1 的硬件架构图（NVIDIA® Mellanox® BlueField® Data Processing Unit (DPU)）：

-   网络：集成 ConnectX-5，峰值带宽 100Gb/s，EDR/HDR100 InfiniBand。
    
-   Arm 子系统：最多 16 个 Armv8 A72 核（部分 SKU 8 核）。
    
-   PCIe：Gen3/Gen4 ×16（同时可作为 PCIe Switch，下挂 NVMe SSD，构成 BlueField Storage Controller / JBOF Reference Design）。
    
-   内存：板载 DDR4。
    
-   典型用例：NVMe over Fabrics (NVMe-oF) 全闪存阵列 (AFA)、JBOF 存储控制器、服务器缓存 (memcached)、分离式机架存储及 Scale-Out 扩展直连存储。
    
-   意义：奠定了"NIC + Arm SoC + PCIe Switch"的 DPU 形态，但当时尚未配 DOCA，需要使用 Mellanox OFED + 自研 SDK。
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/5a24b61030e1c1e72d1cde3d0bdf7c83_MD5.jpg]]

#### 2.2.2 BlueField-2（2020）

如下图所示为 BlueField-2 的硬件架构图（nvidia bluefield-2 dpu）：

-   网络：集成 ConnectX-6 Dx，单/双口，最高 2×100 GbE 或 1×200 Gb/s（Ethernet 与 HDR100 InfiniBand）。
    
-   Arm 子系统：8 颗 Armv8 A72 核，4 MB L2。
    
-   PCIe：Gen4 ×8/16（支持 PCIe Switch + EP/RC 模式切换）。
    
-   内存：16/32 GB 板载 DDR4。
    
-   加速器：
    

-   RDMA / RoCEv2（ConnectX-6 Dx ASAP² 引擎）
    
-   硬件加密：AES-XTS-256/512、AES-GCM、TLS/IPsec
    
-   正则表达式加速（RegEx）
    
-   SHA-256、压缩/解压（DEFLATE）
    
-   SR-IOV、VirtIO-net
    

-   NVMe SNAP™：把远端 NVMe-oF 目标在 Host 侧伪装成本地 PCIe NVMe 盘。
    
-   形态：HHHL 单宽/双宽 PCIe 卡，OCP 3.0 SFF 卡，部分有集成 BMC 形态。
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/457a401f7d8ca7d231b38183efc4b116_MD5.jpg]]

#### 2.2.3 BlueField-3（2022 量产，2023 大规模出货）

如下图所示为 BlueField-2 的硬件架构图（NVIDIA BLUEFIELD-3 DPU）：

-   网络：集成 ConnectX-7，单口 400 Gb/s（或 2×200 GbE / 4×100 GbE / NDR200 IB）。
    
-   Arm 子系统：16 个 Armv8.2 A78 核，8 MB L2， 16 MB 共享 L3。
    
-   PCIe：Gen5 ×32（可拆 2×16 或 1×16 + Switch），支持 PCIe Switch 模式，下挂 GPU、NVMe。
    
-   内存：16 GB 板载 DDR5（5600 MT/s，支持 ECC）。
    
-   DPA（Datapath Accelerator）：16 个超线程 RISC-V 执行单元 × 每核 16 hw 线程 = 256 个硬件线程，专用于 Dataplane 包处理、拥塞控制（PCC）、SHARP collectives、storage-init 等微内核。
    
-   加速器：
    

-   200/400 Gb/s 线速 RoCEv2、Adaptive Routing、Out-of-Order RDMA Write
    
-   GPUDirect RDMA、GPUDirect Storage
    
-   NVMe-oF（TCP / RDMA / FC-NVMe）initiator+target 卸载
    
-   SNAP 4.x：NVMe / VirtIO-blk / VirtIO-net 设备仿真
    
-   公钥 / 对称加密、TLS
    
-   压缩 / 解压、Regex
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/9c4662cc34d5f3800e44a62393421503_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/20260603/images/a55022336f59885dbdc75f4ed4a48c90_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/20260603/images/2a5658426e242c0ca4a709d5908c60f1_MD5.jpg]]

#### 2.2.4 BlueField-4（2025 GTC 公布，2026 出货）

NVIDIA 目前还未提供 BlueField-4 的硬件架构（NVIDIA BlueField-4 DPU datasheet）：

-   核心定位：AI Factory 的 "operating system processor"，与 ConnectX-9、Spectrum-X 第二代、Rubin/Vera-Rubin GPU 同代发布。
    
-   架构融合：把 NVIDIA Grace CPU 子系统与 ConnectX-9 网络引擎直接整合到一颗 SoC，构成全片上一致性内存 + 网络 + 加速器域。
    
-   网络：单卡 800 Gb/s，支持 800G Ethernet（Spectrum-X Gen2）与 800G InfiniBand（Quantum-X800 / XDR）。
    
-   Arm 子系统：基于 Grace 系列 Arm Neoverse V2，64 核心（同源于独立 Grace CPU），114MB 共享 L3 Cache。
    
-   内存：128GB 板载 LPDDR5X 统一内存。
    
-   存储：512GB 可插拔 SSD。
    
-   算力倍增（官方）：
    

-   6x 通用计算性能（vs. BlueField-3）
    
-   支持 4x 更大规模的 AI Factory
    
-   800 Gb/s 网络带宽 是 BF-3 的 2x
    

-   DPA：第二代 Datapath Accelerator，16 核心，256 线程。
    
-   PCIe：Gen6（与 Rubin/Vera 平台对齐）。
    
-   DOCA Microservices 原生支持：容器化、可热插拔的 Networking / Security / Storage / AI Data 服务在 BF-4 上以微服务方式运行。
    

### 2.3 关键差异对比

**

**

### ![[Inbox/笔记同步助手/微信公众号/20260603/images/d5aed30e7cc76e1c37638732a9b661ce_MD5.jpg]]

## 三、配套 GPU、服务器与参考设计

### 3.1 与 NVIDIA GPU 的搭配（典型）

**

**

### ![[Inbox/笔记同步助手/微信公众号/20260603/images/fdc4fc9311d3d87a7493ad093f26512d_MD5.jpg]]

### 3.2 NVIDIA DGX/HGX/MGX 系统

在 NVIDIA 的 DGX 系统中通常会配置 BlueField DPU，而在 HGX、MGX 平台通常为选配：

-   DGX A100：每节点 8 x ConnectX-6 + 2 x BlueField-2（管理 + 存储面）。
    
-   DGX H100 / H200：标配 8 x ConnectX-7 SuperNIC + 2 x BlueField-3，提供 8×400 Gb/s 计算面与独立存储/管理面。
    
-   DGX GB200 NVL72 / GB300 NVL72：72×GB200/B300 + Spectrum-X 后端；BlueField-3 用作 in-rack 网络/存储/安全 offload。
    
-   HGX Reference：OEM 自由配置 ConnectX/BlueField；HGX H100 8-GPU baseboard 推荐每 GPU 配 1× BF-3 SuperNIC。
    
-   MGX 模块化平台（2023+）：BlueField 作为可选 Infra 卡，与 Grace、Hopper、L40S、Blackwell 自由组合。
    

### 3.3 OEM/ODM 服务器生态

支持 BlueField-3/4 的主流厂商（NVIDIA-Certified）：

-   Dell PowerEdge XE 系列
    
-   HPE ProLiant Compute、Cray XD
    
-   Lenovo ThinkSystem SR / ThinkAgile
    
-   Supermicro AS / SYS / GPU SuperServer
    
-   Cisco UCS X-Series（与 Cisco Hypershield 集成）
    
-   ASUS、Gigabyte、Pegatron、ASRock Rack
    
-   NVIDIA RTX PRO Server（2025）：Blackwell + BlueField-3，企业 AI Factory 验证设计的标配模块
    

存储侧 JBOF（Just a Bunch of Flash，全闪存阵列） / All-Flash Array 厂商：

-   VAST Data、DDN、Pure Storage、NetApp、Hitachi Vantara、IBM、WekaIO、Hammerspace、Dell PowerScale、Kioxia、Samsung、Micron 等都有基于 BlueField-2/3 的产品或参考设计；
    
-   BlueField-4 则被用于 NVIDIA STX 与 CMX（Context Memory Storage）平台。
    

JBOF 使用 DPU 相比传统的 x86 CPU 方案具有显著优势，比如具有更高的集成度、更低的时延、更高的吞吐量、更低的功耗等：

![[Inbox/笔记同步助手/微信公众号/20260603/images/afa8676f2b721ed1cd14c4165f67be95_MD5.jpg]]

## 四、DOCA 软件栈与 DPA（Datapath Accelerator）

### 4.1 DOCA 总览

DOCA（Data Center Infrastructure-on-a-Chip Architecture）是 BlueField 的官方 SDK + Runtime，对标 GPU 上的 CUDA（NVIDIA DOCA Software Framework）。包括：

-   DOCA Drivers：MLNX\_OFED / DOCA-OFED（rdma-core, mlx5, ib-verbs）。
    
-   DOCA Libraries：Flow（ASAP² OVS offload）、RDMA、Comch、DMA、SHA、AES-GCM、Compress、Erasure Coding、Telemetry、App Shield、Device Emulation、GPUNetIO、UROM、Rivermax、PCC，以及 DOCA Memos（KV Cache 共享，BF-4 起，详见 §10）。
    
-   DOCA Services / Microservices：BlueField Service Function Chains，K8s 友好的容器化基础设施服务（Firewall、IDS、Telemetry、HBN host-based networking、Memos KV Router 等）。BF-4 起官方主推"DOCA Microservices"运行时，与 NIM 推理微服务对齐。
    
-   DOCA Runtime / BFB Image：BlueField OS 镜像，基于 Ubuntu/Alma，预装 DOCA 全栈。
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/536f483da11af7cc74e4b25b0aef72c1_MD5.jpg]]

### 4.2 DPA（Datapath Accelerator）

DPA（Datapath Accelerator）是 BlueField-3 新引入的嵌入子系统，之前的 SmartNIC 只能提供控制面的处理能力，而 DPA 可以进一步加速某些数据包和 I/O 处理工作负载中需要高性能访问 NIC 引擎的任务。DPA 子系统的特点是拥有众多可并行工作的执行单元，以克服延迟问题（如访问 Host 内存）并提供更高的整体吞吐量（DPA Subsystem - NVIDIA Docs）。具体来说：

-   作用：把 ConnectX 的 ASIC 数据面与 Arm 控制面之间的"中间路径"做软件可编程化，专门跑：
    

-   拥塞控制算法（DOCA PCC，例如 Programmable CC for RoCE，可自定义 DCQCN 替代）
    
-   集合通信加速（与 SHARP 配合）
    
-   自定义 packet processing（DOCA FlexIO / DPA Verbs）
    
-   Storage offload 微内核（NVMe-oF target/initiator 数据面）
    
-   CMX 中的 KV Cache 路由 / 一致性检查 / 加密（BF-4）
    

-   编程模型：C 语言 + DOCA DPA / FlexIO SDK，host 侧 cross-compile（dpaCC 是 Clang/LLVM-for-RISC-V 的封装），runtime 动态加载 ELF 到 DPA，触发方式包括 doorbell、completion、Wire-IRQ。线程上下文与 host 隔离，多进程 + 隔离地址空间。
    
-   价值：相比 FPGA 提供更友好的 C/C++ 编程性与可调试性；相比 Arm 通用核，单线程延迟与并行度都更适合包级处理。
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/03005e5acd5c04908a25d1420cee5111_MD5.jpg]]

其硬件架构如下图所示：自研 RISC-V 多线程处理器，BF-3 含 16 个执行单元，每个 16 个硬件线程，也就是共 256 个硬件线程，由轻量 RTOS 协同调度（cooperative run-to-completion + watchdog），具有专用 L1/L2/L3 + Memory Aperture，可直接访问 NIC 加速器（加密、压缩、SHA、DMA）和 host 内存。

![[Inbox/笔记同步助手/微信公众号/20260603/images/48a9d5b8106f69f6a9615393fca3c3c9_MD5.jpg]]

## 五、GPUDirect Storage（GDS）深入解析

### 5.1 设计动机：CPU bounce buffer 是 GPU IO 的瓶颈

传统 IO 路径：

-   多一次拷贝，增加延迟与功耗；
    
-   受限于 CPU ↔ GPU PCIe 单通道带宽（DGX-2 上 SysMem → GPU 仅 12–12.5 GB/s × 4 PCIe tree ≈ 50 GB/s 上限）；
    
-   Page fault / mmap 隐式路径会造成抖动和长尾延迟。
    

Storage (NVMe / NVMe-oF) - DMA -> CPU SysMem (bounce buffer) - cudaMemcpy -> GPU HBM

  

GDS 提供的直接路径：

-   CPU 仅参与控制面；
    
-   在 DGX-2 实测中，本地 NVMe → GPU 13.3 GB/s（x4 tree = 53.3 GB/s）、外部 NIC NVMe-oF 10.5 GB/s（x8 = 84 GB/s）、RAID 14 GB/s（x8 = 112 GB/s），叠加可达 ～215 GB/s，逼近 GPU PCIe 峰值 230 GB/s，是 SysMem 路径的 ～4x（GPUDirect Storage: A Direct Path Between Storage and GPU Memory | NVIDIA Technical Blog）。
    
-   这些带宽是可加和的：本地 NVMe + 外部 NIC + RAID 同时使用，可以让 GPU 同时从多源拉数据。
    

Storage (本地 NVMe / 远端 NVMe-oF / NIC RDMA) - DMA/RDMA -> GPU HBM

### 5.2 软件栈与 cuFile API

包括：

-   libcufile.so（用户态）：cuFile API 的实现，已合入 CUDA Toolkit 一并发行。
    
-   nvidia-fs.ko（内核态，GPLv2）：实现 DMA callback、GPU 物理地址翻译、与文件系统驱动联动；CUDA 12.8+ 起本地 NVMe / DOCA SNAP 路径走 Linux 上游 PCI P2PDMA（Linux ≥ 6.2 + OpenRM 570.x），不再强依赖 nvidia-fs.ko，对 OS 兼容性显著改善。
    
-   cuFile 核心 API 家族：
    

-   同步：cuFileDriverOpen/Close、cuFileHandleRegister/Deregister、cuFileBufRegister/Deregister、cuFileRead/Write
    
-   异步（CUDA Stream 语义）：cuFileReadAsync/WriteAsync、cuFileStreamRegister/Deregister
    
-   批量：cuFileBatchIOSetUp/Submit/GetStatus/Cancel/Destroy
    

-   关键要点：
    

-   GPU buffer 必须先经过 BAR 注册才能被第三方设备 DMA。cuFileBufRegister 显式注册；不注册时库内部用 bounce buffer + 拷贝（功能可用，性能下降）。
    
-   4 KB 对齐（buffer 地址、file offset、size）才能走最优路径；不对齐会 fallback 到 GPU bounce buffer / POSIX read-modify-write。
    

### 5.3 系统调优

-   关闭 PCIe ACS（Access Control Services）：否则 P2P 会被强制经过 Root Complex，无法绕开 CPU。gdscheck -p 可查看启用 ACS 的 PCIe switch，也可以通过 lspci -vvv | grep -i "acsctl" 查看。
    
-   关闭 IOMMU（x86）：
    

-   BIOS 关闭 intel\_iommu / amd\_iommu，否则 P2P 流量被 Root Port 路由，限制 PCIe switch 下的吞吐（可以使用 cat /proc/cmdline 查看）。
    
-   Grace（aarch64）平台需开启 IOMMU 但禁用 passthrough。
    

-   NIC ↔ GPU PCIe 拓扑：尽量同 PCIe switch 下，至少同 CPU socket，避免 QPI/UPI 跨 socket。
    
-   NIC 配置：
    

-   ConnectX-5/6/7/8 必须 IB 或 RoCEv2 模式；
    
-   MLNX\_OFED ≥ 5.4 或 DOCA ≥ 2.9。
    

-   /etc/cufile.json关键项：
    

-   max\_direct\_io\_size\_kb：默认 16384（16 MB），更大 IO 块减少 IO stack 调用次数；
    
-   max\_device\_cache\_size / per\_buffer\_cache\_size：内部 bounce buffer 池（默认 128 MB / 1 MB）；
    
-   allow\_compat\_mode：unaligned IO fallback 到 POSIX；
    
-   properties.rdma\_dynamic\_routing 与 rdma\_dynamic\_routing\_order。
    

-   CUDA 上下文：尽量让计算 kernel 跑在 primary context，避免 D2D 拷贝被其他 context 抢占（Runtime API 默认就是 primary context）。
    
-   避免 fork() 在 cuFileDriverOpen 之后：行为未定义。
    

### 5.4 Dynamic Routing（关键性能优化）

当 GPU 与 Storage NIC 不在同一 PCIe Root Complex 下时，跨 Root Port 的 P2P 延迟高、吞吐被限制。cuFile 的动态路由按 4 种策略选择最优路径：

-   GPU\_MEM\_NVLINKS：经同 PCIe tree 内的代理 GPU + NVLink 中继到目标 GPU；
    
-   GPU\_MEM：经代理 GPU + PCIe（无 NVLink）；
    
-   SYS\_MEM：fallback 到 host pinned memory；
    
-   P2P：跨 Root Port 直接 P2P（兜底）。
    

通过 cufile.json 的 properties.rdma\_dynamic\_routing\_order 配置，并以 gdscheck -p 校验。Lustre / WekaFS / NFS 还需在 fs.\<type>.mount\_table.\<mount>.rdma\_dev\_addr\_list 列出 NIC IP，否则 fallback 到 SYS\_MEM。

### 5.5 公开的性能数据（GDS vs. 非 GDS）

**

**

### ![[Inbox/笔记同步助手/微信公众号/20260603/images/94cf92e610f2056b02ba8aa8ac04b1d1_MD5.jpg]]

值得注意：

-   即使带宽收益较小（如 Ethernet vs IB 仅 1.17×），CPU 利用率收益通常 4x–6x，对推理服务的延迟稳定性影响巨大。
    
-   IB 与 Ethernet（RoCEv2）功能等价，IB 在大 IO 下略有 1.17× 性能优势。
    
-   多源叠加是 GDS 独有特性：本地 NVMe + 远端 NVMe-oF（IB） + 远端 NVMe-oF（Ethernet）可同时跑满，在 CPU bounce 路径下因 Root Complex 拥塞而无法实现。
    

### 5.6 与 BlueField 的协同

-   BlueField 提供 RDMA 网络栈与 NVMe-oF target/initiator 卸载，是 GDS 远端存储路径的关键 enabler。
    
-   BlueField 上的 NVMe SNAP 把远端 NVMe-oF 在 host 侧仿真成本地 PCIe NVMe 盘，使 GDS 客户端代码无需改动；CUDA 12.8 起 SNAP 路径直接走 P2PDMA，无需 nvidia-fs.ko patch。
    
-   BlueField-3 的 PCIe Gen5 ×32 + 400 Gb/s 网络让 GDS 在远端能跑到接近本地 NVMe 的水平；BF-4 再加 800 Gb/s + Gen6 PCIe，使 NVMe-oF 与 HBM 之间的延迟逼近本地。
    
-   DOCA Memos（BF-4）将 KV Cache 直接以 RDMA + cuFile 风格的 API 暴露给 GPU，是 GDS 抽象层在"AI 原生存储"方向的延伸。
    

## 六、网络互联方案：InfiniBand、Spectrum-X、SuperNIC

### 6.1 InfiniBand（HPC / 紧耦合 AI）

-   Quantum-2（NDR 400G，2021）：与 ConnectX-7 / BF-3 配合，DGX H100 SuperPOD 默认骨干。
    
-   Quantum-X800（XDR 800G，2024）：与 ConnectX-8、Blackwell 同代。
    
-   特性：SHARP（in-network reductions，AllReduce 加速 2×+）、Adaptive Routing、Self-Healing Network、SHIELD（链路层 FEC）。
    

### 6.2 Spectrum-X（专为 AI 的 Ethernet）

-   Spectrum-4（51.2 Tb/s 交换芯片）+ BlueField-3 SuperNIC 组合，构成业内首个"为 AI 优化的以太网"。（NVIDIA Spectrum-X Ethernet Platform for AI Networking）
    
-   关键：
    

-   Adaptive Routing + Packet Spraying：每包独立路径，配合 SuperNIC 端的 RDMA Out-of-Order 重排。
    
-   Direct Data Placement (DDP)：硬件保证 Out-of-Order 包到 GPU 内存正确位置。
    
-   Zero-Touch RoCE Congestion Control：基于 In-band Telemetry。
    
-   在大规模 AllReduce 上 vs. 标准 RoCE 提升 1.6x–1.7x 有效带宽。
    

-   Spectrum-X 第二代（2025）：800G，与 BlueField-4 / ConnectX-9 / Rubin 配套，是 CMX 的网络底座。
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/73d4e3e8df94680305a83dc51cbb2f47_MD5.jpg]]

### 6.3 SuperNIC vs. DPU 区别

**

**

### ![[Inbox/笔记同步助手/微信公众号/20260603/images/e099e7410f479e823d461d87fb96a764_MD5.jpg]]

DGX H100 / HGX 节点：8× SuperNIC（计算面）+ 2× DPU（存储/管理面）是参考设计；Rubin 架构将切换到 ConnectX-9 SuperNIC + BF-4 DPU。

### 6.4 NVLink / NVSwitch

NVLink/NVSwitch 是 GPU 内部 scale-up 互联（一个 NVL72 域内 GPU 之间），BlueField/InfiniBand/Ethernet 是 scale-out 互联，二者是上下层关系。BlueField 不在 NVLink 域内，但 GDS 的动态路由可借用 NVLink 在 GPU 之间中继 RDMA buffer。

## 七、CMX（Context Memory Storage） 上下文记忆存储平台

### 7.1 定位与动机

NVIDIA CMX（Context Memory Storage Platform）是 2026 年 1 月（CES）正式发布、由 BlueField-4 驱动的 AI 原生存储平台，专为长上下文、多轮、Agentic AI 推理设计。 NVIDIA 的官方表述：

"CMX is an AI-native context tier for long-context, multi-turn, and agentic AI inference. Powered by NVIDIA BlueField-4, it extends GPU memory with a shared, pod-level context tier optimized for ephemeral KV cache."

它解决的根本问题：

-   KV Cache 不能长期存在 GPU 上（GPU 是稀缺贵资源），但又必须以接近 GPU 内存的速度被读写；
    
-   传统块/文件存储延迟太高、协议太重、CPU 占用高；
    
-   需要一个共享、低延迟、高带宽、专为 KV 优化的"第四层内存"——位于 GPU HBM 与传统对象存储之间。
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/8f60cc5c3058ca9ebaefe234a0770036_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/20260603/images/f2920366da3a00e53be0352d1f1b609d_MD5.jpg]]

### 7.2 体系结构（端到端 Co-Design）

**

**

### ![[Inbox/笔记同步助手/微信公众号/20260603/images/96d9c6e61b9d9c7f73d4583d096be063_MD5.jpg]]

四层 Co-Design：

NVIDIA BlueField-4（处理器层）:

-   集成 Grace CPU + ConnectX-9，6x BF-3 算力，800 Gb/s 网络；
    
-   直接管理 NVMe SSD（NVMe SNAP / NVMe-oF target）；
    
-   在硬件上 offload KV cache 的 placement、metadata、data integrity、encryption、压缩；
    
-   支持 service function chaining，可同时跑 Memos、Storage、Security 微服务。
    

NVIDIA DOCA Memos（SDK 层 / 数据面）:

-   BF-4 + CMX 优化的 SDK，专为 KV cache 共享/路由/重用而生；
    
-   暴露简单 key-value API（put / get / lookup / evict），把以太网附着的 flash 变成 pod 级共享缓存；
    
-   硬件加速 integrity（CRC/SHA）+ encryption（AES-XTS / AES-GCM），让应用保持无状态而 CMX 处理 KV 路由；
    
-   与 NIXL（NVIDIA Inference Exchange Library）和 Dynamo serving 紧耦合。
    

NVIDIA Spectrum-X Ethernet（网络层）:

-   提供 800 Gb/s 低延迟 RDMA 互联，用于跨节点的 KV 共享；
    
-   Adaptive Routing + lossless RoCE + 高级拥塞控制，最小化 jitter 与 tail latency，确保多租户下"可重复"的性能；
    
-   让 CMX 可以以 pod 为单位 scale 到多个 rack 而保持高吞吐 + 低尾延迟。
    

NVIDIA Dynamo（推理服务编排层）:

-   分布式推理服务框架，把 CMX 与底层 context tier 暴露成 pod 内"无缝"的 KV 抽象；
    
-   实现 KV-aware request placement：把请求路由到已经持有相关 KV 的实例（reuse）；
    
-   提升 tokens/s、降低 TTFT，支持多轮 / 多 Agent 上下文重用；
    
-   与 NIM 推理微服务、TensorRT-LLM、vLLM、SGLang 都可对接。
    

### 7.3 关键能力与性能

来自 NVIDIA 官方公开资料（CES 2026 / GTC DC 2025）：

-   Rubin cluster-level KV cache capacity：rack-scale 共享 KV，单 pod 容量可达 TB→PB 级。
    
-   5× higher tokens/s：相比传统存储方案。
    
-   5× greater power efficiency：把更多 data center 能耗预算让给 GPU 计算。
    
-   Hardware-accelerated KV placement by BF-4：消除 metadata 开销、减少数据搬运、保证安全隔离访问。
    
-   Smart KV sharing across AI nodes：DOCA Memos + NIXL + Dynamo 协同。
    
-   Spectrum-X RDMA：低 jitter、高带宽 RDMA 访问 AI-native KV。
    
-   Multi-turn / Agentic 持久上下文：对长会话型 Agent 系统提供"短期记忆 + 长期记忆"统一基础设施。
    

### 7.4 与 GDS 的关系

-   GDS / cuFile：解决 GPU↔storage 的直连访问（文件/块语义）；
    
-   CMX / DOCA Memos：解决 GPU↔KV cache tier 的直连访问（KV 语义）；
    
-   两者底层都靠 BlueField + RDMA + GPUDirect 系列；
    
-   CMX 可视为 GDS 思想在 KV / Object 抽象上的延伸：把"GPU 直接读写远端存储"扩展到"GPU 直接读写 pod 级 KV 缓存"。
    
-   在同一个 AI Factory 中，GDS 仍然负责模型权重、训练数据、checkpoint；CMX 负责推理 KV 与 Agent 上下文记忆，互补而非替代。
    

### 7.5 部署形态与 STX

-   NVIDIA STX（Storage Reference Architecture）：模块化 AI 存储参考设计；CMX 可视为 STX 的一种"上下文记忆"配置实例。
    
-   典型 pod：1× AI 计算 rack（Rubin GPU + BF-4 SuperNIC）+ N× CMX 存储 rack（BF-4 + NVMe JBOF）+ Spectrum-X 二代 800G fabric。
    
-   API：
    

-   对外：DOCA Memos KV API + NIXL + Dynamo；
    
-   对内：NVMe-oF over RoCE / IB。
    

### 7.6 实践使用模式

-   冷启动 / 模型切换：从 CMX 拉模型权重，BF-4 SNAP 仿真本地盘 + GDS 直达 HBM；
    
-   Decode 阶段：每生成一个 token 后，把溢出的 KV 异步推到 CMX；下一轮请求若命中相同 prefix（system prompt / 多轮历史），由 Dynamo 路由到对应实例并直接拉 KV；
    
-   Agent 多 hop：跨工具调用、跨 Agent 切换的上下文持久化在 CMX，所有 Agent 实例可见；
    
-   多租户：BF-4 提供租户级 KV 隔离与加密，租户之间不可见。
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/ae9ab885bc2ff0935730d937438098d0_MD5.jpg]]

## 八、大模型 / 视频生成 / 大规模推理场景下的应用收益

### 8.1 LLM / VLM 预训练（千亿 ～ 万亿参数）

预训练阶段的 IO 瓶颈集中在三件事：

-   大规模 Tokenized Dataset Streaming：多 PB 级 tokenized data（Common Crawl/WebDataset/Mosaic shards/Parquet）需要持续灌入数千 GPU。
    
-   Checkpoint Save / Load：千亿参数 + Optimizer State 单点可达 1–5 TB，按 step 节奏写盘。
    
-   集合通信（AllReduce/AllGather）：跨节点 East-West 流量，受拥塞控制与路由质量直接影响 MFU。
    

BlueField + GDS + Spectrum-X 的收益：

-   数据加载（Data Loader）：通过 cuFile + NVMe-oF + BlueField，每 GPU 拉数据带宽从 SysMem 路径的 ～6 GB/s 提升到 13–20 GB/s（本地+远端叠加），消除 CPU DataLoader 抖动对 step time 的影响。RAPIDS cuDF / NVIDIA DALI / WebDataset 已原生集成 cuFile。
    
-   Checkpoint：原生支持 cuFile Stream Async + Batch，Checkpoint write 速度提升 3–8×，配合 BlueField 上的 DOCA Compress / AES-XTS，可同时做压缩 + 落盘加密；checkpoint 时间窗压缩，意味着 checkpoint 间隔可以更短（更少的丢失训练时间），是 megascale 训练（>10K GPU）必备能力。
    
-   集合通信：BlueField-3 SuperNIC + Spectrum-X 在 RoCE 上跑出 1.6–1.7× 标准 RoCE 的有效带宽，对应 LLM 训练端到端 MFU（Model FLOPs Utilization）提升 5–10%（根据模型与 batch size）。
    
-   DPA 可编程拥塞控制：超大集群的 incast、long-tail 拥塞用自定义 PCC 算法（如 HPCC/Annapurna 类）替代 DCQCN，长尾延迟下降。
    
-   故障域隔离：BF DPU 跑独立 OS 与 telemetry，控制面与训练面分离，单 GPU 节点重启不会影响 BF-managed 存储面。
    

### 8.2 视频生成 / 多模态预训练（Sora 类、Veo 类、世界模型）

视频/多模态模型（Sora、Veo、Cosmos、世界模型 + diffusion transformer）在数据维度的特殊性：

-   样本极大：单条 720p/1080p 视频片段经 VAE encode 后 latent token 序列动辄 10⁵–10⁶，原始素材 PB 级；
    
-   预处理流水：视频解码、关键帧提取、resize、tokenization 全部在 GPU 上做（NVIDIA DALI / VPF / NVDEC）；
    
-   多模态对齐：文本-视频-音频-3D 多源融合，IO pattern 高度随机。
    

BlueField + GDS 的收益：

-   NVDEC + GDS：视频帧从 NVMe-oF / 对象存储 → 直达 GPU HBM → NVDEC 硬解 → VAE encode，全程不经过 CPU，吞吐提升 ～9x（NVIDIA 官方在视频分析负载上的实测）。
    
-   对象存储加速：与 MinIO、Cloudian、Hammerspace、VAST 等 S3 兼容存储集成，BlueField 上跑 RDMA-S3 / NFSoRDMA 客户端，把对象路径直接接入 cuFile。
    
-   多并发 stream：cuFile Batch + Stream Async 配合 NVDEC 多 instance，单 H100 节点可并发解码上百路视频。
    
-   视频生成推理：视频/Sora 类模型推理时既有"长上下文"（时间维度的 latent KV）又有大模型 KV Cache，BF-4 + CMX 可以把 spatial-temporal KV Cache offload 到 pod 级共享存储。
    

### 8.3 大规模推理部署（LLM Serving、Agentic、Long-Context）

大规模推理是 BlueField-4 + CMX 真正的"杀手级场景"。它解决的核心矛盾：模型越大、上下文越长、并发越高、Agent 多轮越深，KV Cache 占用越爆炸；但 KV Cache 存在 GPU HBM 里，GPU 既贵又稀缺。

KV Cache 经济学（以 Llama-3 405B class 为例）：

-   单 token KV ≈ 2 × layers × heads × head\_dim × bytes\_per\_elem；405B 模型每 token KV ≈ 数十 KB。
    
-   128K 上下文、batch=64 的并发推理，KV Cache 单 instance ≈ 几十 GB；多轮 Agent / RAG 场景下，Cache 增长到 TB 级。
    
-   把 KV 完全留在 HBM：浪费 GPU；丢弃后重算（recompute）：燃烧 prefill 算力，TTFT 升高。
    

BlueField + GDS + CMX 的解法：

![[Inbox/笔记同步助手/微信公众号/20260603/images/5a9b9f7f620f1c33ae6bd4acd37e8718_MD5.jpg]]

NVIDIA 官方在 CMX + Dynamo + Rubin 组合上披露的关键指标：

-   5x tokens/s（vs. 传统存储后端做 KV cache 的方案）
    
-   5x 更好的功率效率
    
-   TTFT（Time to First Token）显著下降：因 KV 已在 pod 内可寻址，不需重算
    
-   支持 Rubin 集群级 KV Cache 容量：多 rack 共享上下文层
    
-   可用时间：BlueField-4 与 CMX 平台 2026 年下半年进入早期可用，作为 NVIDIA Vera-Rubin 平台一部分。
    

![[Inbox/笔记同步助手/微信公众号/20260603/images/c4aa278dd9d4724e15008ac9a2b9deae_MD5.jpg]]

### 8.4 RAG / 向量检索 / Agent

-   向量数据库（Milvus、Weaviate、FAISS、cuVS）：索引规模常达几亿到几十亿向量，IO 要求接近 OLTP；通过 GDS + NVMe-oF + BlueField 直接把 ANN 查询数据 DMA 到 GPU，搜索 QPS 与延迟都优于 SysMem 路径。
    
-   Agent 多轮记忆：长期记忆/工具调用历史本质上是带 metadata 的 KV，CMX 提供"短期 KV + 长期 context memory"的统一抽象，DOCA Memos 把 key-value API 暴露给应用层，BlueField-4 硬件做 placement、加密与去重。
    
-   NIXL（NVIDIA Inference eXchange Library）+ Dynamo：CMX 通过 NIXL 与 Dynamo 紧耦合，在 serving 层做 KV-affinity 路由（"把请求送到 KV 已在的实例"），避免跨 pod 拉 KV。
    

### 8.5 收益小结

**

**

### ![[Inbox/笔记同步助手/微信公众号/20260603/images/5e1391cddbfdf060429b3cc9ab5a7e0d_MD5.jpg]]

## 九、部署模式与运维要点

-   BFB 镜像安装：bfb-install + rshim（USB/PCIe over USB）刷写 BlueField OS。
    
-   DPU 模式切换：mlxconfig -d \<dev> set INTERNAL\_CPU\_MODEL=…，区分 Embedded/Separated host。
    
-   驱动栈：Host 装 MLNX\_OFED 或 DOCA-OFED；DPU 内已集成。
    
-   Kubernetes：NVIDIA Network Operator + DOCA Operator + SR-IOV / Multus / NVIDIA-Aerial CNI。
    
-   观测：DOCA Telemetry Service（DTS）→ Prometheus / OTel；NetQ for fabric。
    
-   安全基线：开启 Secure Boot + RoT，固件签名验证，PSP/Confidential Computing attestation；BF-4 引入 BlueField Advanced Secure Trusted Resource Architecture。
    
-   GDS 系统调优：参考 §6.4（IOMMU / ACS / NIC affinity / cufile.json）。
    
-   常见坑：
    

-   PCIe Gen5/Gen6 信号完整性对 PCB / Riser 要求高，OEM 验证必看。
    
-   GDS 在多路径 NVMe（multipath）/ RAID 上需额外补丁，CUDA 12.8 之前依赖 nvidia-fs.ko 对 nvme.ko 的私有 patch。
    
-   Dynamic Routing 需要正确填写 rdma\_dev\_addr\_list，否则会回退 SYS\_MEM 损失带宽。
    
-   DCQCN 调参：在 Spectrum-X 域更推荐用 NVIDIA Zero-Touch RoCE，混合域需谨慎。
    
-   大模型 Checkpoint：cuFile Async + 4 MB+ 块大小 + buffer register；避免循环 register/deregister。
    

## 十、参考链接

### 10.1 NVIDIA 官方主页 / Datasheet

-   BlueField 平台主页：https://www.nvidia.com/en-us/networking/products/data-processing-unit/
    
-   BlueField-3 Datasheet：​https://resources.nvidia.com/en-us-accelerated-networking-resource-library/datasheet-nvidia-bluefield
    
-   BlueField-4 Datasheet：https://resources.nvidia.com/en-us-accelerated-networking-resource-library/bluefield-4-dpu-datasheet
    
-   BlueField-2 Datasheet：​https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/documents/datasheet-nvidia-bluefield-2-dpu.pdf
    
-   BlueField-3 SuperNIC：https://www.nvidia.com/en-us/networking/products/ethernet/supernic/
    
-   Spectrum-X：https://www.nvidia.com/en-us/networking/spectrumx/
    
-   Quantum-X800 IB：https://www.nvidia.com/en-us/networking/products/infiniband/quantum-x800/
    

### 10.2 DOCA / DPA / Memos

-   DOCA SDK 文档：https://docs.nvidia.com/doca/sdk/index.html
    
-   DPA Subsystem：https://docs.nvidia.com/doca/sdk/dpa-subsystem/index.html
    
-   DOCA DPA Library：https://docs.nvidia.com/doca/sdk/DOCA-DPA/index.html
    
-   DOCA PCC（可编程拥塞控制）：https://docs.nvidia.com/doca/sdk/DOCA-PCC/index.html
    
-   DOCA 开发者主页（含 Memos）：https://developer.nvidia.com/networking/doca
    

### 10.3 GPUDirect Storage / Magnum IO

-   GDS Overview Guide：https://docs.nvidia.com/gpudirect-storage/overview-guide/index.html
    
-   cuFile API Reference：https://docs.nvidia.com/gpudirect-storage/api-reference-guide/index.html
    
-   GDS Best Practices Guide：https://docs.nvidia.com/gpudirect-storage/best-practices-guide/index.html
    
-   GDS 配置 / Configuration Guide：https://docs.nvidia.com/gpudirect-storage/configuration-guide/index.html
    
-   O\_DIRECT Requirements Guide：https://docs.nvidia.com/gpudirect-storage/o-direct-guide/index.html
    
-   Magnum IO 开发者主页：https://developer.nvidia.com/magnum-io
    
-   Dev Blog: GPUDirect Storage（原始）：https://developer.nvidia.com/blog/gpudirect-storage/
    
-   Dev Blog: Magnum IO Storage Partnerships：​https://developer.nvidia.com/blog/accelerating-io-in-the-modern-data-center-magnum-io-storage-partnerships/
    
-   MagnumIO GitHub：https://github.com/NVIDIA/MagnumIO
    
-   gds-nvidia-fs GitHub：https://github.com/NVIDIA/gds-nvidia-fs
    

### 10.4 BlueField-4 / CMX / STX / AI Factory

-   BF-4 STX Storage：​https://nvidianews.nvidia.com/news/nvidia-launches-bluefield-4-stx-storage-architecture-with-broad-industry-adoption
    
-   BF-4 AI-Native Storage：​https://nvidianews.nvidia.com/news/nvidia-bluefield-4-powers-new-class-of-ai-native-storage-infrastructure-for-the-next-frontier-of-ai
    
-   Blog: BF-4 AI Factory：https://blogs.nvidia.com/bluefield-4-ai-factory
    
-   CMX 主页：https://www.nvidia.com/en-us/data-center/ai-storage/cmx/
    
-   CMX Solution Overview：https://resources.nvidia.com/en-us-ai-storage/cmx-solution-overview
    
-   Dev Blog: Introducing BF-4-powered Inference Context Memory Storage：​https://developer.nvidia.com/blog/introducing-nvidia-bluefield-4-powered-inference-context-memory-storage-platform-for-the-next-frontier-of-ai/
    
-   STX 主页：https://www.nvidia.com/en-us/data-center/ai-storage/stx/
    
-   AI Data Platform：https://www.nvidia.com/en-us/data-center/ai-data-platform/
    
-   AI Factory Validated Design：https://www.nvidia.com/en-us/solutions/ai-factories/validated-design/
    
-   NVIDIA Dynamo（推理编排）：https://developer.nvidia.com/dynamo
    
-   Rubin 平台：https://www.nvidia.com/en-us/data-center/technologies/rubin/
    

### 10.5 客户案例

-   OCI 全面采用 BlueField：​https://nvidianews.nvidia.com/news/oracle-cloud-infrastructure-chooses-nvidia-bluefield-data-center-acceleration-platform
    
-   BlueField for HPC：https://www.nvidia.com/en-us/networking/products/data-processing-unit/hpc/
    
-   BlueField + Cybersecurity：https://www.nvidia.com/en-us/solutions/ai/cybersecurity/
    
-   合作伙伴 AI Inference 案例集合：https://blogs.nvidia.com/blog/category/enterprise/intelligent-networking/
    

### 10.6 社区 / 开发者资源

-   NVIDIA DOCA Developer Forum：https://forums.developer.nvidia.com/c/infrastructure/369/
    
-   NVIDIA Networking 文档总入口：https://docs.nvidia.com/networking/index.html
    
-   NVIDIA Academy（培训）：https://academy.nvidia.com/en/
    

**

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/39a9f107_1780447350896?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk0ODU3MjcxNA%3D%3D%26mid%3D2247491450%26idx%3D1%26sn%3D89e62a3664b3cc63596d7cfbdefe27e8%26chksm%3Dc255aa4a970202b00a2a5dd1ae71c949208e5cd2000d841f29fc4217fef6d7eaa0a69326b65d%26mpshare%3D1%26scene%3D1%26srcid%3D0603xjsvf3fXea40tVrD0nKi%26sharer_shareinfo%3D473583c0b7846e1b17ad90f1a2965324%26sharer_shareinfo_first%3D473583c0b7846e1b17ad90f1a2965324%23rd&s=obsidian)