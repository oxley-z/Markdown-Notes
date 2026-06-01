---
author: 王二小
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk0MTE5MDk0OA==&mid=2247486625&idx=1&sn=5fd4923ee3f0a43a3a92940667840974&chksm=c361b0a7acdb99fc127fb744b0d5b6866d7b846df171c3c8581d6f0d015e3635959456abf5f9&mpshare=1&scene=1&srcid=0530nJ67aCy0IjFtwvEi1Jab&sharer_shareinfo=2b154abadcf115bdfd549eba2be6390c&sharer_shareinfo_first=2b154abadcf115bdfd549eba2be6390c#rd
saved: 2026-05-30 13:33:40
tags:
  - 笔记同步助手
id: 16cc7c49-0a8b-4278-a26b-28ab5addb3a3
---

公众号名称：Linux炉边会

作者名称：王二小

发布时间：2026-05-25 07:22

看一个框架图，有疑问我们可以评论区或者加群讨论

![[Inbox/笔记同步助手/微信公众号/20260530/images/719321fff6fdae549456ae929aa37e4a_MD5.jpg]]

## 第一章 Linux 网络子系统整体架构

### 1.1 netdevice 在 Linux 网络栈中的位置

Linux 网络子系统是内核中最复杂的子系统之一。从应用层 socket 到网卡 DMA 收发，中间涉及协议栈、路由、缓存管理、中断、软中断、内存分配与驱动调度等大量模块。而 netdevice 则是整个 Linux 网络栈中连接“协议层”与“硬件层”的核心抽象。

Linux 内核并不会直接操作具体网卡，而是通过 struct net\_device 抽象网络设备。无论是 PCIe Ethernet、USB 网卡、WiFi、CAN、虚拟网卡、TUN/TAP 还是 loopback，在 Linux 看来本质上都是 net\_device。上层 TCP/IP 协议栈只与 net\_device 交互，而无需关心底层硬件实现细节，用户调用：send(sockfd, buf, len, 0);`数据最终会进入：`

```
Socket Layer
↓
TCP/UDP
↓
IP Layer
↓
Routing
↓
Qdisc
↓
net_device
↓
NIC Driver
↓
DMA Engine
```

其中 net\_device 正是协议栈与驱动之间的分界层。

### 1.2 Linux 网络驱动的分层设计

Linux 网络驱动架构本质上是典型的“分层式 framework”。上层协议栈完全不关心硬件寄存器；底层驱动也无需实现 TCP/IP 协议。Linux 通过统一 net\_device interface 将两者解耦，整个架构主要包括：

```
User Space
↓
Socket API
↓
TCP/IP Stack
↓
Network Core
↓
net_device
↓
netdev_ops
↓
NIC Driver
↓
Hardware
```

其中 network core 位于 net/core/ 目录；协议栈位于 net/ipv4、net/ipv6；驱动通常位于 drivers/net/。Linux 使用 struct net\_device\_ops 定义驱动回调接口，驱动只需实现 ndo\_open、ndo\_start\_xmit 等操作即可接入完整网络栈，这种设计是 Linux 网络子系统高度可扩展的重要原因。无论硬件来自 Intel、Realtek、Broadcom 还是虚拟网卡，都可以无缝接入 Linux 协议栈。

## 第二章 net\_device 核心结构解析

### 2.1 struct net\_device 结构分析

Linux 网络设备核心结构是 struct net\_device。它用于描述一个网络接口的所有状态与能力。

```
struct net_device {
char name[IFNAMSIZ];
unsigned long state;
const struct net_device_ops *netdev_ops;
const struct ethtool_ops *ethtool_ops;
unsigned int mtu;
unsigned char dev_addr[MAX_ADDR_LEN];
struct napi_struct napi;
struct netdev_queue *_tx;
};
```

其中 name 表示网卡名字，例如 eth0；mtu 表示最大传输单元；dev\_addr 保存 MAC 地址；netdev\_ops 定义驱动回调；napi 用于网络轮询；\_tx 表示发送队列。

net\_device 本质上是 Linux 网络栈中的“设备对象”。整个协议栈几乎所有发送与接收路径最终都会围绕 net\_device 展开。

### 2.2 net\_device 的注册流程

网络驱动 probe 成功后，必须向 Linux 注册 net\_device，否则协议栈无法识别当前网卡。

```
ndev = alloc_etherdev(sizeof(struct my_priv));
ndev->netdev_ops = &my_netdev_ops;
register_netdev(ndev);
```

alloc\_etherdev() 用于创建 Ethernet 类型 net\_device，并自动分配 private data。register\_netdev() 则会完成：

```
设备编号分配
sysfs 创建
/proc/net 注册
网络命名空间绑定
RTNL 注册
通知协议栈
```

注册成功后，用户即可通过：

```
ip link show
ifconfig
看到对应网卡。
```

Linux 网络子系统内部维护全局 net\_device 链表。所有网络设备最终都会挂入该 framework 中统一管理。

## 第三章 net\_device\_ops 驱动接口

### 3.1 ndo\_open 与 ndo\_stop

Linux 网络驱动最核心的接口是 struct net\_device\_ops。它定义了网卡生命周期与数据收发操作。

```
struct net_device_ops {
int (*ndo_open)(struct net_device *dev);
int (*ndo_stop)(struct net_device *dev);
netdev_tx_t (*ndo_start_xmit)(
struct sk_buff *skb,
struct net_device *dev);
};
```

其中 ndo\_open 在网卡启动时调用

```
ip link set eth0 up
```

Linux 会进入 dev\_open()，随后调用驱动 ndo\_open。驱动通常在此阶段完成：

```
DMA ring 初始化
IRQ 申请
PHY 启动
NAPI 启用
MAC 配置
queue 启动
```

而 ndo\_stop 则负责关闭网卡、释放中断、停止 DMA 与关闭 queue。

### 3.2 ndo\_start\_xmit 发送路径

ndo\_start\_xmit 是网络驱动最关键的发送接口。协议栈在完成 TCP/IP 封装后，会将 skb 传递给驱动，典型调用路径如下：

```
socket send
↓
TCP/IP
↓
dev_queue_xmit()
↓
qdisc enqueue
↓
ndo_start_xmit()
```

驱动通常会在 ndo\_start\_xmit 中：

```
解析 skb
映射 DMA
填写 TX descriptor
通知 DMA engine
启动发送
```

```
static netdev_tx_t my_xmit(struct sk_buff *skb,
struct net_device *ndev)
{
dma_addr_t dma;
dma = dma_map_single(dev,
skb->data,
skb->len,
DMA_TO_DEVICE);
tx_desc->addr = dma;
tx_desc->len = skb->len;
writel(TX_START, priv->base + REG_TX_CTRL);
return NETDEV_TX_OK;
}
```

当硬件发送完成后，中断处理函数会回收 descriptor 与 skb。

## 第四章 sk\_buff 数据结构解析

### 4.1 skb 在网络栈中的作用

Linux 网络栈所有数据包都使用 sk\_buff 表示。skb 是 Linux 网络子系统最核心的数据结构之一，典型定义：

```
struct sk_buff {
unsigned char *head;
unsigned char *data;
unsigned char *tail;
unsigned char *end;
unsigned int len;
struct net_device *dev;
__be16 protocol;
};
head -> buffer 起始地址
data -> 当前数据起始位置
tail -> 当前数据结束位置
end  -> buffer 结束地址
```

Linux 协议栈会不断在 skb 前部 push header。

```
Application Data
↓
TCP Header Push
↓
IP Header Push
↓
MAC Header Push
```

整个过程几乎无需额外 copy。

### 4.2 skb 生命周期

网络包从 socket 到网卡，再从网卡返回协议栈，整个过程中 skb 会不断流转，发送路径：

```
sock_alloc_send_skb()
↓
TCP/IP Encapsulation
↓
dev_queue_xmit()
↓
Driver TX Ring
↓
DMA Send
↓
dev_kfree_skb()
```

接收路径：

```
RX DMA
↓
IRQ/NAPI
↓
build skb
↓
netif_receive_skb()
↓
IP Layer
↓
Socket Queue
```

Linux 网络性能优化很大程度上都围绕 skb 展开。例如 GRO、GSO、TSO、XDP、本质上都在减少 skb 分配与 copy 开销。

## 第五章 网络接收路径解析

### 5.1 RX DMA Ring 工作机制

现代网卡几乎全部基于 DMA ring 工作。驱动会预先分配大量 RX descriptor，并为每个 descriptor 绑定 skb buffer，典型结构：

```
RX Descriptor Ring
↓
Descriptor[0]
Descriptor[1]
Descriptor[2]
```

每个 descriptor 包含：

```
DMA Address
Buffer Length
Status
Control Bits
```

网卡收到数据包后，会通过 DMA 直接写入对应 buffer。CPU 完全不参与数据搬运。这也是现代网络系统能够实现 10G/40G/100G 吞吐的基础，驱动初始化阶段通常会：

```
skb = netdev_alloc_skb(ndev, RX_BUF_SIZE);
dma = dma_map_single(dev,
skb->data,
RX_BUF_SIZE,
DMA_FROM_DEVICE);
```

随后将 DMA 地址写入 RX descriptor。

### 5.2 NAPI 与软中断机制

Linux 网络接收性能核心在于 NAPI（New API）。早期 Linux 每收到一个包都会触发一次中断，在高流量场景下 CPU 会被 IRQ 打爆，NAPI 采用“中断 + 轮询”混合模式：

```
Packet Arrive
↓
IRQ Trigger
↓
Disable RX IRQ
↓
Schedule NAPI
↓
SoftIRQ Polling
↓
Process Multiple Packets
↓
Enable IRQ
```

驱动通常实现 napi\_poll：

```
static int my_poll(struct napi_struct *napi,
int budget)
{
while (work_done < budget)
rx_packet();
if (done)
napi_complete_done(napi, work_done);
}
```

这种机制极大降低中断频率，是 Linux 网络高吞吐的关键设计。

## 第六章 网络发送路径解析

### 6.1 qdisc 队列调度机制

Linux 网络发送路径中，数据并不会直接进入驱动，而是先进入 qdisc（queue discipline）。qdisc 负责流控、限速、优先级调度与拥塞控制，典型发送路径：

```
Socket
↓
TCP/IP
↓
qdisc enqueue
↓
qdisc dequeue
↓
ndo_start_xmit
```

Linux 默认 qdisc 包括：

```
pfifo_fast
fq_codel
mq
cake
```

这里fq\_codel 能显著降低 bufferbloat，Linux 网络性能不仅取决于驱动，还与 qdisc 密切相关。

### 6.2 TX Completion 与 Queue 管理

发送完成后，网卡会触发 TX complete interrupt。驱动需要回收 descriptor、解除 DMA mapping 并释放 skb，典型流程：

```
dma_unmap_single(dev,
dma,
skb->len,
DMA_TO_DEVICE);
dev_kfree_skb_any(skb);
```

Linux 网络栈同时维护 queue stop/wake 机制：

```
netif_stop_queue(ndev);
netif_wake_queue(ndev);
```

当 TX ring 满时，驱动必须 stop queue，否则协议栈会继续发送 skb 导致 descriptor overflow。待 TX complete 回收部分 descriptor 后，再 wake queue 恢复发送，这也是 Linux 网络 backpressure 的重要组成部分。

## 第七章 PHY 与 MDIO 子系统

### 7.1 PHY 驱动架构

Linux Ethernet 驱动通常分为 MAC driver 与 PHY driver 两部分。MAC 控制 DMA 与 Ethernet controller；PHY 负责电气层。

Linux PHY subsystem 位于：drivers/net/phy/

MAC driver 通常通过 phylib 管理 PHY：

```
phy_connect(ndev,
phy_name,
adjust_link,
interface);
```

PHY driver 负责：(做底层驱动的应该了解这里，这里是标准的操作)

```
Link Detect
Auto Negotiation
Speed Detect
Duplex Detect
EEE
```

MAC 与 PHY 通过 MDIO 总线通信。

### 7.2 MDIO 总线机制

MDIO 是 Ethernet PHY 标准管理总线。Linux 使用 mii\_bus 抽象 MDIO controller，典型操作：

```
mdiobus_read(bus, phyaddr, regnum);
mdiobus_write(bus, phyaddr, regnum, val);
```

PHY 初始化过程中，Linux 会扫描 MDIO bus：

```
Read PHY ID
↓
Match PHY Driver
↓
Register PHY Device
```

随后 PHY 状态变化会通过 state machine 更新 net\_device carrier。

```
netif_carrier_on(ndev);
netif_carrier_off(ndev);
```

这也是用户空间看到：Link is Up和`Link is Down的来源。`

## 第八章 Linux 网络软中断机制

### 8.1 NET\_RX\_SOFTIRQ

Linux 网络接收大量依赖 softirq。网卡 IRQ handler 通常不会直接处理所有数据，而只是 schedule softirq。

```
NIC IRQ
↓
napi_schedule()
↓
NET_RX_SOFTIRQ
↓
net_rx_action()
↓
napi_poll()
```

softirq 运行在中断上下文，能够避免频繁线程调度。这也是 Linux 网络栈高吞吐的重要原因，但 softirq 也存在 CPU starvation 风险。如果网络流量过大，ksoftirqd 可能占满 CPU，导致用户线程延迟升高。

### 8.2 RPS/RFS/XPS 机制

为了提高多核系统网络吞吐，Linux 引入：

```
RPS
RFS
XPS
```

RPS（Receive Packet Steering）允许 RX packet 分发到不同 CPU；RFS（Receive Flow Steering）能够根据 socket 所在 CPU 定向流量；XPS（Transmit Packet Steering）则优化发送路径 CPU 绑定。

```
echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus
```

可以将 RX softirq 分散到多个 CPU，这些机制本质上属于 Linux 网络栈 SMP scalability 优化。

## 第九章 网络驱动中的 DMA 与内存管理

### 9.1 DMA Mapping 机制

现代网卡完全依赖 DMA 完成数据搬运。Linux 驱动必须使用 DMA API，而不能直接使用物理地址，发送路径：

```
dma_map_single(dev,
skb->data,
skb->len,
DMA_TO_DEVICE);
```

接收路径：

```
dma_map_single(dev,
skb->data,
RX_BUF_SIZE,
DMA_FROM_DEVICE);
```

DMA API 会处理：

```
IOMMU Mapping
Cache Sync
DMA Address Translation
```

Linux 网络驱动本质上高度依赖 DMA subsystem。

### 9.2 Page Pool 与 Zero-Copy

现代 Linux 网络栈已经逐渐转向 page-based RX buffer 管理。page\_pool framework 专门用于高性能 RX page recycling，传统 skb alloc/free 成本较高，而 page\_pool 能实现 page reuse：

```
RX DMA
↓
Build skb Fragment
↓
Protocol Stack
↓
Recycle Page
```

Linux XDP 与 high-performance NIC driver 大量使用 page\_pool。

此外 AF\_XDP、io\_uring zerocopy send 等机制，也在进一步减少 skb copy 与 page alloc 开销。这是现代 Linux 网络性能优化的重要方向。

## 第十章 虚拟网络设备架构

### 10.1 TUN/TAP 与 veth

Linux net\_device framework 不仅支持物理网卡，也支持虚拟网络设备，例如：

```
lo
veth
bridge
vxlan
geneve
tun
tap
```

这些设备本质上同样是 net\_device，比如有 TUN/TAP：

```
User Space
↓
/dev/net/tun
↓
net_device
↓
TCP/IP Stack
```

VPN、QEMU、容器网络大量依赖 TUN/TAP。

### 10.2 Linux Bridge 与 Namespace

Linux bridge 本质上也是虚拟 net\_device。它实现二层转发功能：

```
eth0 ←→ bridge ←→ veth0
```

容器网络大量依赖：

```
veth pair
bridge
network namespace
```

整个 Kubernetes Docker 网络架构，本质上建立在 Linux net\_device framework 之上，这也是 Linux 网络子系统极其强大的原因之一。物理网卡与虚拟网络设备全部共享统一抽象。

## 第十一章 ethtool 与调试机制

### 11.1 ethtool\_ops 接口

Linux 网络驱动通常实现 ethtool\_ops，用于支持用户空间调试。

```
static const struct ethtool_ops my_ethtool_ops = {
.get_drvinfo = my_get_drvinfo,
.get_link = ethtool_op_get_link,
.get_ringparam = my_get_ringparam,
};
```

用户空间可通过：ethtool eth0，`ethtool -S eth0查看：`

```
Link Speed
Duplex
Ring Size
Driver Stats
Offload Feature
```

ethtool 是 Linux 网络驱动调试的重要工具。

### 11.2 procfs 与 debugfs

Linux 网络子系统还提供大量调试接口：

```
/proc/net/dev
/proc/interrupts
/sys/class/net/
/sys/kernel/debug/
cat /proc/net/dev
可以查看 RX/TX statistics。
cat /proc/softirqs
可以分析网络 softirq 压力
```

现代高性能网络调试，通常需要结合：perf `ftrace bpftrace ethtool sar这些工具分析。`

从整体架构来看，Linux netdevice framework 的成功，在于它通过统一抽象，将复杂硬件、协议栈与虚拟化网络全部纳入同一模型。它不仅仅是一个驱动框架，更是整个 Linux 网络生态的核心基础设施。

建了一个嵌入式Linux技术群，专门聊难题分析和求职面试，欢迎大家一起加入，共同解决工作中的疑难杂症问题

![[Inbox/笔记同步助手/微信公众号/20260530/images/07c842dbc94c099acfd1438b597d70a4_MD5.jpg]]

---

![[Inbox/笔记同步助手/微信公众号/20260530/images/6ed9e901c79afd2a3d370ce8d40326b3_MD5.jpg|cover_image]]

原创 王二小 Linux炉边会

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/2a440f24_1780119218223?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk0MTE5MDk0OA%3D%3D%26mid%3D2247486625%26idx%3D1%26sn%3D5fd4923ee3f0a43a3a92940667840974%26chksm%3Dc361b0a7acdb99fc127fb744b0d5b6866d7b846df171c3c8581d6f0d015e3635959456abf5f9%26mpshare%3D1%26scene%3D1%26srcid%3D0530nJ67aCy0IjFtwvEi1Jab%26sharer_shareinfo%3D2b154abadcf115bdfd549eba2be6390c%26sharer_shareinfo_first%3D2b154abadcf115bdfd549eba2be6390c%23rd&s=obsidian)