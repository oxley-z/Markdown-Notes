---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487447&idx=1&sn=9ced4d698d144fd9b1f4812e5aa11720&chksm=c057152f0a98c7a1a4972e654825b72aa7205be02c2dee3dc2f19b84af6688f29815bd526178&mpshare=1&scene=1&srcid=0704tAaCKvDSJTsOW7Jqhwbx&sharer_shareinfo=c3de9a3f0b8075d8da369db6376926dd&sharer_shareinfo_first=c3de9a3f0b8075d8da369db6376926dd#rd
saved: 2026-07-04 13:20:52
tags:
  - 笔记同步助手
id: 4485d5e3-640b-4c12-b948-0980b011bff6
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-04 12:59

一、驱动层面：底层共用mlx5\_core，上层分两套独立接口模块  
1\. 内核模块分层（Linux原生/OFED一致）  
1\. mlx5\_core.ko（公共底层，唯一硬件抽象层）  
网卡硬件操作、PCI、FW命令、硬件队列资源池、EQ/CQ/WQ硬件管理、DMA、寄存器访问全部由这一套代码实现，以太网、RDMA共用同一个core底层。  
struct mlx5\_core\_dev 是单卡唯一根对象，管理全局硬件资源池（QP、CQ、EQ、MKEY、MSI-X等）。

2\. mlx5\_en（以太网/内核TCP网络栈）  
基于mlx5\_core封装Linux net\_device ，提供eth0/ensXX网卡、内核TCP/IP、TX/RX队列、NAPI、TC卸载、TLS/IPsec offload。

3\. mlx5\_ib.ko（RDMA/RoCE/IB Verbs层）  
同样基于mlx5\_core，对接内核 ib\_core RDMA子系统，实现IB Verbs、QP、MR、GID、RoCE、IPoIB、用户态libibverbs通路 。  
总结驱动关系  
\- 硬件底层：同一套mlx5\_core驱动代码，操作同一套网卡硬件；  
\- 上层协议入口：两套独立模块，netdev走Linux网络栈，RDMA走IB/RDMA子系统，互不干扰。  
二、硬件队列资源：硬件池全局共享，但业务队列完全隔离、不混用  
1\. 全局硬件资源池（PF/VF硬件统一分配）  
网卡FW预分配全局资源池，以太网、RDMA从同一个硬件池申请资源：  
\- EQ（事件队列）、CQ（完成队列）、WQ（发送/接收队列）、QP（RDMA专属队列对）、MKEY/MR、MSI-X中断向量、DoorBell、DMA内存池。  
资源总量由FW参数（ NUM\_OF\_EQ / NUM\_OF\_QP / NUM\_OF\_CQ ）限制，以太网和RDMA竞争同一限额，一方开大量队列会挤占另一方配额。  
2\. 业务队列完全隔离，绝不混用  
（1）以太网netdev队列（内核TCP专用）  
\- 名词：TXQ/RXQ（内核网卡收发队列）  
\- 硬件本质：普通WQE Work Queue，无QP封装，走以太网硬件转发通道；  
\- 完成机制：绑定独立CQ，EQ中断单独处理；  
\- 用途：内核socket、TCP、UDP、DPDK原生以太网；  
\- 归属：仅mlx5\_en管理，RDMA无法打开/使用netdev TX/RXQ。  
（2）RDMA队列（QP，RDMA专属）  
\- 名词：QP（Queue Pair，SQ发送WQ + RQ接收WQ成对绑定）  
\- 硬件本质：专用RDMA队列对象，带GID、RDMA协议状态机、重传/PSN、RDMA内存保护域；  
\- 完成机制：QP绑定独立CQ，EQ事件区分RDMA/以太网完成；  
\- 用途：RoCEv2、IB、NVMe-oF、NCCL、UCX等RDMA应用；  
\- 归属：仅mlx5\_ib/libmlx5用户态管理，内核TCP完全看不到QP。  
3\. 关键资源隔离细节  
1\. CQ/EQ  
\- 以太网TX/RX各绑定独立CQ；  
\- RDMA每个QP绑定独立CQ；  
\- EQ按类型分流：以太网事件、RDMA事件、FW错误事件分离，可绑定不同MSI-X中断。  
2\. 内存注册MR/MKEY  
\- 内核TCP：使用网卡内置普通DMA缓冲，无MR注册；  
\- RDMA：必须申请MKEY/MR内存注册池，隔离内核/用户态内存访问权限，硬件独立资源池。  
3\. 中断MSI-X  
全局中断池共享，但可配置亲和性隔离：以太网TX/RX中断、RDMA EQ中断分到不同CPU，互不抢占。  
4\. PF/VF/SF隔离  
同一PCI Function（PF/VF/SF）内，以太网与RDMA共享Function硬件限额；不同Function资源完全隔离，VF的QP、TXQ不会占用PF资源 。  
三、数据流路径对比（硬件通路分离）  
1\. 内核TCP（netdev）  
SKB → mlx5\_en TXQ WQE → 以太网硬件转发引擎 → 网线；  
收包：网线 → RXQ → SKB → Linux TCP协议栈。  
2\. RDMA RoCE  
用户态WR → QP SQ WQE → RDMA硬件协议引擎（RoCEv2封装、重传、校验） → 网线；  
收包：网线 → QP RQ → 直接DMA到用户态Buffer，不经过内核TCP栈。  
网卡内部两套硬件转发引擎并行工作，硬件通道物理隔离，只是共享PCIe、FW资源池。  
四、实操现象  
1\. mlxconfig 配置 NUM\_OF\_QP 是RDMA最大QP数量，开大量RDMA会报错QP不足，但不影响以太网TX/RX；

2\. ethtool -l 查看以太网TX/RX队列数，和RDMA QP数量互不显示；

3\. /proc/interrupts 里可区分 mlx5\_compX （以太网CQ中断）、 mlx5\_ib\_eqX （RDMA EQ中断）；

4\. OFED中同时跑iperf3（TCP）+ perftest（RDMA），两者带宽叠加消耗网卡总带宽，但队列资源池相互抢占；

5\. DPDK原生以太网PMD、DPDK RDMA Verbs PMD，底层都依赖mlx5\_core，但分别创建独立TXQ和QP，不能混用队列句柄。  
精简结论  
1\. 驱动底层：同一套mlx5\_core，硬件操作共用；上层以太网(mlx5\_en)、RDMA(mlx5\_ib)是两套独立业务模块。  
2\. 硬件资源池全局共享：EQ/CQ/WQ/QP/MKEY总量统一限额，TCP和RDMA会竞争资源；  
3\. 业务队列完全隔离不互通：TCP用TX/RX普通WQ，RDMA用QP队列对，两套队列硬件通道、管理逻辑、中断完全分开，不能互相借用。

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/8caeec4f_1783142447949?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487447%26idx%3D1%26sn%3D9ced4d698d144fd9b1f4812e5aa11720%26chksm%3Dc057152f0a98c7a1a4972e654825b72aa7205be02c2dee3dc2f19b84af6688f29815bd526178%26mpshare%3D1%26scene%3D1%26srcid%3D0704tAaCKvDSJTsOW7Jqhwbx%26sharer_shareinfo%3Dc3de9a3f0b8075d8da369db6376926dd%26sharer_shareinfo_first%3Dc3de9a3f0b8075d8da369db6376926dd%23rd&s=obsidian)