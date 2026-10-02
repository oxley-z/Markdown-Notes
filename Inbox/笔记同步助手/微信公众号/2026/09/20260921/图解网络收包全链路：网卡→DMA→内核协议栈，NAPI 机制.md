---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s/voYrg8bIT9KS7ufFi1U1eg
saved: 2026-09-21 20:48:09
tags:
  - 笔记同步助手
id: ea0ae443-bd27-430f-bf22-e984a4f798dd
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-09-21 20:28

大家好，我是蟹老板～

在 Linux 系统之中，网络收包乃是一个极为基础同时又非常重要的过程。我们平时在应用程序里面调用一个 `recv()`，看起来仅仅只是从 Socket 当中读取了一段数据，可实际上在这一个 `recv()` 背后，网络数据早已经经历了网卡、DMA、RX Ring、中断、NAPI、软中断、`sk_buff`、IP 协议栈、TCP/UDP 以及 Socket 接收队列等一系列处理。

今天这一篇文章就从一个真正的网络包出发，将 Linux 网络收包完整链路彻底拆开来看。

阅读前，我们先把整个过程简单记一下，这样文章读起来更轻松：

```
物理链路
   ↓
网卡 PHY / MAC
   ↓
DMA
   ↓
RX Ring Buffer
   ↓
硬中断 IRQ
   ↓
NAPI
   ↓
NET_RX_SOFTIRQ
   ↓
sk_buff / GRO
   ↓
Linux TCP/IP 协议栈
   ↓
Socket Receive Queue
   ↓
recv()/read()
   ↓
用户进程
```

真正搞清楚这一条链路之后，很多所谓的“Linux 网络性能玄学”，其实都会变得非常清晰。

## 一、一个网络包的完整一生

对于大多数应用开发人员而言，对网络的认识一般是从 TCP/IP 协议栈开始的。

一个外网数据包，从物理线路抵达服务器，再被业务进程读取，要经历一整套层层流转的链路，任何一个环节卡住、溢出、处理不及时，都会丢包、延时、抖动。

我把完整链路从头到尾捋清楚，每个节点都是线上可能出问题的关键点：

物理链路/光模块/网线 → 网卡PHY/MAC层校验处理 → DMA硬件搬运数据 → 网卡RX环形缓冲区缓存数据包 → 硬件中断触发CPU响应 → NAPI机制+软中断批量处理 → 内核协议栈逐层解析剥离 → Socket接收队列缓存数据 → 用户进程调用recv/read读取数据

是不是瞬间通透了？

平时我们用tcpdump抓包、看netstat指标、调优网卡参数，本质上都是在干预、观测这条链路里的某个环节。你看不懂指标，就是因为没对应上链路节点。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/04d1dc0edc07ddf894aff8b8ae2957f5_MD5.jpg]]

## **二、一个网络包的“一生”**

### **2.1 收包全链路关键节点**

为了后面能够更加容易地理解，我们先把一个网络包进入 Linux 服务器之后的大致过程给梳理清楚。

假定客户端向服务器发送一个 TCP 数据包。

这个包首先经过网线或者光纤来到服务器的物理网卡。

网卡的 PHY 层负责处理物理信号，将线路上的电气信号转换成为数字数据；MAC 层随后识别以太网帧，完成目的 MAC 地址、帧长度以及 FCS/CRC 等相关检查。

当数据帧被网卡确认可以接收之后，网卡并不会要求 CPU 一个字节一个字节地把数据读取出来。

相反，驱动程序早就准备好了 RX Ring，并且在其中告诉网卡：

```
有新的数据包到来的时候，
请把数据 DMA 到这些内存缓冲区。
```

网卡按照 RX 描述符提供的 DMA 地址，将数据直接写入系统内存。

写完以后，网卡更新对应描述符的状态，并且通常借助 MSI/MSI-X 中断通知 CPU：

```
有新的 RX 数据需要处理。
```

CPU 进入网卡中断处理函数。中断处理函数不会长时间处理数据包，而是快速完成必要操作并调度 NAPI。

之后通常由 `NET_RX_SOFTIRQ` 驱动网络接收处理，NAPI 的 `poll()` 回调开始批量清理 RX Ring。

驱动从 RX Ring 中获得已经接收完成的数据缓冲区，将其组织成为 Linux 网络协议栈所使用的 `sk_buff`，或者先经过 XDP、GRO 等相关处理，再提交进入协议栈。

之后 Ethernet 层判断上层协议，IP 层进行路由以及 Netfilter 等处理，TCP/UDP 层再根据源地址、目标地址以及端口号等信息寻找对应的 Socket。

最终数据进入 Socket Receive Queue。

如果此时应用程序正阻塞在：

```
recv(fd, buf, sizeof(buf), 0);
```

内核就能够唤醒相应的进程。

用户进程重新得到 CPU 之后，再把 Socket 接收队列中的数据复制到用户空间缓冲区。

至此，一个网络包才算真正走完了从网卡到应用程序的收包过程。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/fa71effbcf6e0ed417d8ffd4ef9fa9de_MD5.jpg]]

### **2.2 一句话记住整个收包过程**

整个 Linux 网络收包链路，可以用一句非常容易记忆的话进行概括：

> **DMA 搬包，中断通知，NAPI 轮询，协议栈解析，Socket 排队，应用读取。**

如果再展开一点就是：

```
网卡收帧
   ↓
DMA 写内存
   ↓
IRQ 通知 CPU
   ↓
NAPI 批量收包
   ↓
SoftIRQ 执行网络处理
   ↓
构造 skb
   ↓
TCP/IP 协议栈
   ↓
Socket Receive Queue
   ↓
唤醒进程
   ↓
recv/read
```

后面的所有内容，实际上都是围绕这一条主线展开。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/3e0c98c06357c4e647445a9230d6124c_MD5.jpg]]

## **三、第一站：网卡如何接收数据帧**

所有网络数据，最先接触的不是操作系统，更不是内核，而是服务器的物理网卡。很多人忽略这一点，导致排查丢包永远卡在内核层面，殊不知包在网卡阶段就已经被丢掉了。

网卡内部不是一个简单的传输工具，它自带一套完整的硬件处理逻辑，核心就是PHY、MAC、FIFO三大模块，还有硬性的CRC校验机制。

### **3.1 PHY、MAC、FIFO、CRC 校验**

一个数据包真正进入 Linux 内核之前，首先得经过网卡硬件。

现代以太网卡内部并不是单纯存在着一个“收包模块”，而是包含 PHY、MAC、FIFO、DMA Engine、RX/TX Queue、Interrupt Controller 等众多硬件单元。

PHY 也就是 Physical Layer，负责物理层相关的工作。

如果使用普通电口网卡，那么 PHY 负责将网线上的模拟电信号转换成为数字信号。如果使用光纤，那么光模块首先完成光电转换，再将数据交给相应的 PHY/MAC 逻辑。

MAC 层主要负责处理以太网帧。

一个典型 Ethernet Frame 大致可以理解成：

```
+-----------+-----------+----------+---------+-----------+
| DST MAC   | SRC MAC   | EtherType| Payload | FCS/CRC   |
+-----------+-----------+----------+---------+-----------+
```

目的 MAC 用来确定这个帧是不是发送给当前主机。

EtherType 则可以告诉后面的协议栈：

```
0x0800 → IPv4
0x86DD → IPv6
0x0806 → ARP
```

而帧尾部的 FCS 则用于检测传输过程中是否出现错误。

正常情况下，要是网卡发现一个 Ethernet Frame 的 FCS 错误，那么这个帧通常就会在网卡硬件层直接被丢弃，甚至根本没有机会进入 Linux TCP/IP 协议栈。

因此有一些网络故障，实际上发生在比 `tcpdump` 更低的位置。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/fcd812e013cc256288bc9a823d188822_MD5.jpg]]

### **3.2 以太网帧到达网卡后发生什么**

数据帧来到网卡以后，大致经历这样几个阶段：

```
线路信号
   ↓
PHY 解码
   ↓
MAC 接收 Ethernet Frame
   ↓
目的 MAC / VLAN / FCS 等检查
   ↓
RSS Hash / Flow Classification
   ↓
选择 RX Queue
   ↓
DMA 写入主机内存
```

这里一个非常重要的问题出现了：

网卡究竟应该把数据放到哪个 RX Queue？

早期网卡可能只有一条接收队列。

但是现代服务器一般都是：

```
16 核
32 核
64 核
甚至 128 核以上
```

如果全部网络包都放到一个 RX Queue，再全部交给一个 CPU 来处理，那么这个 CPU 很快就会成为整个网络系统的瓶颈。

因此现代网卡普遍使用多队列机制。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/1d1eeb5a9e9d2beccc341bd699fd00c4_MD5.jpg]]

### **3.3 多队列网卡与 RSS：为什么收包不只靠一个 CPU**

RSS 全称 Receive Side Scaling。

它所做的事情可以简单理解成为：

```
对网络 Flow 做 Hash
           ↓
       得到 Hash 值
           ↓
     选择一个 RX Queue
```

例如网卡可以针对 TCP/IPv4 流的：

```
源 IP
目的 IP
源端口
目的端口
```

计算 Hash。

随后根据 Hash 结果，把不同的 Flow 分配到：

```
RX Queue 0
RX Queue 1
RX Queue 2
RX Queue 3
...
```

每一个 RX Queue 又能够关联不同的 MSI-X 中断以及 CPU。

因此就可能形成：

```
RX Queue 0 → IRQ 50 → CPU0
RX Queue 1 → IRQ 51 → CPU1
RX Queue 2 → IRQ 52 → CPU2
RX Queue 3 → IRQ 53 → CPU3
```

这样网络接收的处理压力就能够分散到多个 CPU 上面。

Linux 内核官方文档也将 RSS 定义成通过硬件接收队列把流量分布到多个 CPU 的扩展机制，而 RPS、RFS 则属于在软件层进一步进行 CPU 分流以及 Flow Steering 的办法。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/1882f199001f4785495c43c78111e5f6_MD5.jpg]]

### **3.4 网卡侧可能丢包的位置**

不要认为所有丢包都是 TCP/IP 协议栈造成的。

网络包甚至还没有进入协议栈之前就可能已经消失了。

例如：

```
PHY 错误
FCS/CRC 错误
网卡 FIFO Overflow
RX Queue Buffer 不足
DMA 来不及完成
PCIe 带宽不足
RX Descriptor 耗尽
```

这些情况都可能导致网卡直接丢包。

因此排查 Linux 网络丢包的时候：

```
ethtool -S eth0
```

往往是一个非常重要的起点。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/4b1b9ac84ad5bbcc0becebb0c25e2ef2_MD5.jpg]]

## **四、第二站：DMA——为什么收包不靠 CPU 拷贝**

讲完网卡硬件，很多人会好奇。网卡收到数据包后，数据是怎么到服务器内存里的？难道是CPU一点点拷贝过来的吗？

如果真的靠CPU拷贝，那百万并发场景下，CPU直接被拷贝操作耗死，根本没资源处理业务逻辑。这就是DMA存在的意义。

### **4.1 DMA 是什么，为什么需要 DMA**

DMA 全称 Direct Memory Access，也就是直接内存访问。

要是没有 DMA，那么假设网卡收到了一个 1500 字节的数据包，CPU 可能需要不断读取网卡寄存器或者 FIFO，然后把数据一点一点地搬到系统内存之中。

大致就会变成：

```
网卡
 ↓
CPU 读取
 ↓
CPU 写内存
```

这显然会严重浪费 CPU。

因此现代网卡基本都会使用 DMA。

DMA 机制可以让网卡直接和内存进行数据传输：

```
┌──── CPU
        │
网卡 ──DMA──→ 内存
```

CPU 不需要亲自搬运每一个字节。

CPU 主要负责的是：

```
准备缓冲区
告诉网卡 DMA 地址
处理完成事件
运行协议栈
```

真正的数据搬运则由网卡的 DMA Engine 来完成。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/e09a24f513fdfc3367e1c1825d493109_MD5.jpg]]

### **4.2 RX Ring 描述符**

那么网卡怎么知道数据应该 DMA 到哪一块内存呢？

这里就需要 RX Ring。

RX Ring 可以把它理解成一个循环队列。

里面放置的是一系列 RX Descriptor。

概念上一个 Descriptor 可以理解成为：

```
struct rx_desc {
    dma_addr_t buffer_addr;
    uint16_t length;
    uint16_t status;
};
```

实际不同网卡的描述符格式会存在非常大的差异，但是核心思想基本相同。

描述符主要告诉网卡：

```
这个内存地址可以用
这个 Buffer 有多大
当前 Buffer 的状态是什么
```

例如：

```
RX Ring

+-------+-------+-------+-------+-------+
| Desc0 | Desc1 | Desc2 | Desc3 | Desc4 |
+-------+-------+-------+-------+-------+
    ↓       ↓       ↓       ↓
  Buffer  Buffer  Buffer  Buffer
```

驱动程序会提前准备很多可以用来接收数据的内存缓冲区，并将这些 Buffer 做 DMA 映射，再把 DMA 地址写入 RX Descriptor。

网卡收到数据以后，就顺着 Ring 找一个可用 Descriptor，然后将数据 DMA 到相应的 Buffer。

现代驱动为了提高 RX 内存管理效率，还经常会结合 page pool 等机制来重复利用接收页面，而不是每收到一个包都进行一次昂贵的普通内存分配。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/6b3707a29facd5ece44af70af38e7086_MD5.jpg]]

### **4.3 网卡通过 DMA 把数据写入内核内存**

假设：

```
Desc 100
```

指向一块 DMA Buffer：

```
0x12340000
```

网卡接收到一个 1280 字节的数据包之后，就能够直接执行类似：

```
NIC DMA Engine
        ↓
0x12340000
```

这样的数据写入操作。

数据进入内存以后，网卡再更新 Descriptor，例如记录：

```
Packet Length
Checksum Status
RSS Hash
VLAN Information
Receive Complete
```

随后驱动程序在 NAPI poll 当中读取这些描述符，便可以知道：

```
哪些包已经接收完成
包有多大
校验和是否已经由硬件检查
属于哪个 Flow
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/8d6a54e71facac20ce02b6401513030c_MD5.jpg]]

### **4.4 DMA 映射与 Cache 一致性简单理解**

这里还有一个比较容易被忽视的问题，就是 DMA 地址和 CPU 虚拟地址并不是完全相同的概念。

驱动程序一般需要通过 DMA API 将某块内存映射成为设备能够访问的 DMA 地址。

从逻辑上来看：

```
CPU Virtual Address
        ↓
DMA Mapping
        ↓
DMA Address
        ↓
NIC
```

某些体系结构还需要考虑 CPU Cache 和设备 DMA 之间的一致性问题。

例如 CPU Cache 里面可能存在一份旧数据，而网卡已经通过 DMA 修改了内存。

因此 Linux DMA API 会负责建立相应的内存访问规则以及同步机制。

对于普通应用开发者而言，并不需要深入到每一种 CPU 架构的 Cache Coherence 细节，但是需要知道一点：

> **网卡 DMA 的目标并不是随随便便一个普通指针，而是经过 DMA API 管理的设备可访问内存。**

![[Inbox/笔记同步助手/微信公众号/2026/09/images/5efde0551c807f67aea3a1cc67367129_MD5.jpg]]

### **4.5 DMA 并不代表整个网络链路零拷贝**

这里还有一个非常常见的误区。

使用 DMA 并不意味着：

```
网络收包 = 零拷贝
```

DMA 解决的是：

```
网卡 → 主机内存
```

这一阶段不要 CPU 亲自搬运的问题。

但正常 Socket 接收路径最后通常仍然存在：

```
Kernel Buffer
     ↓ copy_to_user
User Buffer
```

也就是说普通：

```
recv(fd, buf, len, 0);
```

一般依然需要把数据从内核空间复制到用户空间。

因此更加准确的说法应该是：

> **DMA 减少了网卡到内存阶段的 CPU 数据搬运，但是并不意味着传统 Linux Socket 收包链路已经实现了全程零拷贝。**

![[Inbox/笔记同步助手/微信公众号/2026/09/images/514b5dc080c103a0fbbd201f3c08b3ae_MD5.jpg]]

## **五、第三站：硬中断——CPU，有包到了**

DMA把数据包成功写入内核内存之后，CPU是怎么知道有新包到达的？总不能让CPU一直轮询检测内存吧？太浪费性能了。

这里就用到了硬件中断机制，也就是我们常说的硬中断。

### **5.1 网卡收到包后如何通知 CPU**

网卡已经通过 DMA 把数据写进内存以后，还有一个问题：

CPU 怎么知道有新数据？

总不能让 CPU 一直这样：

```
while (1) {
    check_rx_ring();
}
```

这样会把 CPU 完全浪费在轮询上面。

因此传统方式就是使用硬件中断。

网卡完成一批 RX 工作以后，通过：

```
Legacy INTx
MSI
MSI-X
```

等中断机制通知 CPU。

现代多队列网卡大量使用 MSI-X，因为它可以为不同的 RX/TX Queue 提供不同的中断向量。

如此一来：

```
Queue0 → IRQ → CPU0
Queue1 → IRQ → CPU1
Queue2 → IRQ → CPU2
```

就成为了可能。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/aadefe0b6905cd8e22e7d7479ab6ffcd_MD5.jpg]]

### **5.2 中断上半部：必须快进快出**

Linux 硬中断上下文里面存在一个极为重要的原则：

> **不能长时间工作。**

假设网卡每秒收到几百万个 Packet。

如果每一次中断都在硬中断上下文当中把：

```
Ethernet
IP
TCP
Socket
```

全部处理完，那么 CPU 就会长时间停留在硬中断之中。

这会严重影响系统其他任务。

因此网卡中断处理函数通常仅仅完成：

```
确认中断
屏蔽/抑制相应 RX 中断
调度 NAPI
退出硬中断
```

例如从概念上来看：

```
irqreturn_t nic_irq_handler(...)
{
    disable_rx_irq();

    napi_schedule_irqoff(&queue->napi);

    return IRQ_HANDLED;
}
```

真正大量的数据包处理工作被推迟到 NAPI 阶段完成。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/2ebc2fef7212e1778d521a4aa2eed95e_MD5.jpg]]

### **5.3 中断合并 / 中断缓解**

如果：

```
1 个 Packet = 1 次 IRQ
```

那么万兆、25G、100G 网络情况下，中断数量可能会变得非常恐怖。

假设每秒 100 万个数据包，那么理论上就可能产生非常高的 IRQ 处理压力。

因此现代网卡普遍具有 Interrupt Coalescing，也就是中断合并或者中断缓解机制。

它的基本思想乃是：

```
不要每收到 1 个包就中断一次
而是收到一批包以后再通知 CPU
```

例如：

```
收到 32 个 Packet
或者
等待 50 微秒
```

以后再产生一次 IRQ。

如此一来就能够显著减少中断数量。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/bc8ea1b9373eb9367ec0353319555cda_MD5.jpg]]

### **5.4 高收包率下的“中断风暴”**

纯中断模型最大的麻烦就在高 PPS 场景。

如果数据包源源不断：

```
Packet
 ↓
IRQ
 ↓
Packet
 ↓
IRQ
 ↓
Packet
 ↓
IRQ
```

那么 CPU 会不断在：

```
用户态
内核态
硬中断
```

之间发生上下文切换。

甚至 CPU 大量时间都消耗在处理中断，而真正用于协议栈以及应用程序的时间反而变少。

这就是为什么 Linux 后来引入 NAPI。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/9558e1464f71dae19459043e9f7fa84c_MD5.jpg]]

### **5.5 为什么需要 NAPI**

NAPI 最重要的思想其实非常简单：

> **流量小的时候依靠事件/中断及时响应；流量大的时候依靠轮询批量处理。**

也就是说，不再坚持：

```
一个包 → 一个中断
```

而是转变成为：

```
IRQ
 ↓
调度 NAPI
 ↓
一次 poll 处理很多 Packet
```

这样既保留了低流量情况下中断响应迅速的优势，又减少了高流量情况下产生大量中断的问题。

Linux 当前的 NAPI 文档也明确说明，正常工作方式通常是设备先通过中断通知主机，由主机调度 NAPI 实例，而 NAPI 的处理通常运行在软件中断上下文；现代内核同时也支持 threaded NAPI 等其他模式。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/fa3461ad3598dcf560f16cfc86a7d4a3_MD5.jpg]]

## **六、第四站：NAPI 机制——中断 + 轮询的混合艺术**

NAPI绝对是Linux网络收包的核心精髓，没有之一。搞懂NAPI，你就搞定了80%的内核收包性能问题。

很多人觉得NAPI很难，其实核心逻辑一句话就能讲透：**空闲时用中断通知，繁忙时用轮询批量捞包**。

### **6.1 NAPI 的核心思想**

理解 NAPI 可以先记住一个简化版流程：

```
第一个事件
   ↓
IRQ
   ↓
关闭/抑制 RX IRQ
   ↓
调度 NAPI
   ↓
poll 批量处理 Packet
   ↓
Ring 清空
   ↓
重新打开 IRQ
```

所以很多资料会简单说：

> **第一个包靠中断，后面的包靠轮询。**

这句话非常方便记忆，但是它并不是严谨的逐包描述。

准确来说，NIC 产生接收事件以后，中断处理函数调度某一个 NAPI 实例，随后该 NAPI 实例的 `poll()` 在指定 budget 下批量处理设备事件。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/0710832ee8bdd46c86334e238b5f8d9d_MD5.jpg]]

### **6.2 napi\_struct、poll 与 budget**

NAPI 最核心的结构之一就是：

```
struct napi_struct
```

驱动通常会为 RX Queue 或者 RX/TX Queue Pair 配置 NAPI。

例如概念上可能存在：

```
netif_napi_add(netdev, &rxq->napi, driver_poll);
```

随后定义：

```
static int driver_poll(struct napi_struct *napi, int budget)
{
    ...
}
```

这里的：

```
poll
```

就是驱动真正批量处理 RX 数据的重要入口。

而：

```
budget
```

可以把它理解为一次 NAPI Poll 最多允许处理多少工作量，尤其是 RX Packet 数量。

为什么需要 budget？

因为 Linux 不能让一块流量非常大的网卡无限占用 CPU。

假设：

```
Queue0 永远有 Packet
```

如果不做限制，那么：

```
driver_poll()
```

可能永远退不出去。

所以内核会限制单次 poll 的工作量，让不同设备以及系统任务之间能够获得相对合理的 CPU 时间。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/b179d737443f0a6d5dd7752b1e9a8176_MD5.jpg]]

### **6.3 IRQ → NET\_RX\_SOFTIRQ → NAPI poll**

整个过程可以简化理解成为：

```
NIC
 │
 │ MSI-X
 ↓
Hard IRQ
 │
 │ napi_schedule()
 ↓
NET_RX_SOFTIRQ
 │
 ↓
net_rx_action()
 │
 ↓
NAPI poll()
```

硬中断只完成快速操作。

之后 NAPI 被加入待处理列表，并触发网络 RX SoftIRQ。

SoftIRQ 执行过程中，网络子系统开始调用相应 NAPI 实例的 `poll()`。

这时候才真正开始大量清理 RX Ring。

所以要是你执行：

```
cat /proc/softirqs
```

发现：

```
NET_RX
```

数值非常高，那么通常意味着当前机器正在进行大量网络接收处理。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/1b70fd89471295a8f1b18743b0fac4c0_MD5.jpg]]

### **6.4 NAPI poll 如何处理 RX Ring**

NAPI poll 大致可以理解成为：

```
while (work_done < budget) {

    desc = get_next_rx_desc();

    if (!packet_ready(desc))
        break;

    process_rx_packet(desc);

    refill_rx_buffer(desc);

    work_done++;
}
```

当然真实驱动会复杂得多。

里面还可能存在：

```
DMA Sync
XDP
page_pool
Checksum Offload
VLAN
RSS Hash
SKB Build
GRO
RX Statistics
```

但是核心流程依然是：

```
读取已经完成的 Descriptor
            ↓
拿到 Packet Buffer
            ↓
构建网络数据结构
            ↓
向协议栈提交 Packet
            ↓
补充新的 RX Buffer
```

Ring 本质上就是被不断消费以及补充。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/b7d21f636de270baad72542b9539e852_MD5.jpg]]

### **6.5 什么情况下重新开启中断**

这里是 NAPI 最关键的逻辑之一。

假设：

```
budget = 64
```

这一次 poll 只处理了：

```
20 个 Packet
```

然后 RX Ring 就空了。

这说明目前已经没有更多工作。

驱动就可以完成 NAPI，例如通过：

```
napi_complete_done(napi, work_done);
```

随后重新打开相应的硬件中断。

Linux NAPI 文档特别强调，驱动通常应该在 `napi_complete_done()` 成功以后再重新 unmask 相应 IRQ。

但是另外一种情况：

```
budget = 64
work_done = 64
```

这说明这一次把 budget 全部吃完了，Ring 很可能还有数据。

此时通常不会马上恢复中断，而是让 NAPI 后续继续得到执行机会。

因此 NAPI 本质上就是在：

```
Interrupt Mode
```

和：

```
Polling Mode
```

之间进行非常巧妙的切换。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/b330f77573dd1482953fd88a43381b01_MD5.jpg]]

### **6.6 多队列下的 NAPI 与 CPU 亲和性**

现代服务器里面经常会形成：

```
RX Queue0 → MSI-X IRQ0 → NAPI0 → CPU0
RX Queue1 → MSI-X IRQ1 → NAPI1 → CPU1
RX Queue2 → MSI-X IRQ2 → NAPI2 → CPU2
RX Queue3 → MSI-X IRQ3 → NAPI3 → CPU3
```

大多数网卡往往会让：

```
Queue Pair
Interrupt
NAPI
```

形成比较直接的对应关系，不过这并不是内核强制要求的固定一一映射。官方 NAPI 文档也指出，常见网卡通常会使用类似“中断—NAPI—队列对”的组合，但是具体映射仍然取决于驱动实现。

所以 Linux 网络性能优化里面一个极其重要的主题就是：

```
IRQ Affinity
```

也就是把不同网卡队列产生的中断合理地分散到不同 CPU。

## **七、第五站：从 RX Ring 到 sk\_buff**

NAPI轮询从RX Ring取出数据包后，不能直接丢给协议栈处理。内核需要对原始数据包进行封装、补全元数据，这就诞生了Linux网络最核心的结构体——sk\_buff。

所有内核协议栈处理、Socket队列存储、业务读取的数据包，全部都是sk\_buff结构。

### **7.1 驱动 poll 取出描述符**

现在 Packet 已经躺在 DMA Buffer 里面了。

NAPI poll 找到对应 RX Descriptor 以后，就能够获取：

```
Packet Buffer
Packet Length
Checksum State
RSS Hash
VLAN Tag
Queue ID
```

等等信息。

随后驱动会把这些硬件世界中的信息转换成为 Linux 网络协议栈能够理解的形式。

而 Linux 网络协议栈当中极其重要的一个结构就是：

```
struct sk_buff
```

通常简称：

```
skb
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/efdd66d08ea662a4891e61b874c9f865_MD5.jpg]]

### **7.2 sk\_buff 是什么**

`sk_buff` 可以理解成为 Linux 网络协议栈内部的数据包“身份证 + 包装盒”。

它不仅仅保存数据。

还保存大量元数据，例如：

```
Packet Length
Network Device
Protocol
Checksum State
VLAN
Hash
MAC Header Offset
Network Header Offset
Transport Header Offset
Socket
Timestamp
```

等等。

概念上可以理解成：

```
struct sk_buff
        │
        ├── data
        ├── len
        ├── protocol
        ├── dev
        ├── hash
        ├── ip_summed
        └── ...
              ↓
        Packet Data
```

不过现代高性能网卡驱动并不一定是：

```
malloc 一个 skb
再 memcpy Packet
```

很多情况下会直接利用 DMA 接收页面构建 skb，从而尽量避免没有必要的数据复制。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/5f4261c02afac5bcd4a8a4eedfef0d75_MD5.jpg]]

### **7.3 填充协议头、长度、Hash 等信息**

驱动构造 skb 以后，还需要填写很多信息。

例如：

```
skb->dev
```

代表这个包是从哪个网络设备进入的。

RSS Hash 可以记录到 skb 的 Hash 元数据里面。

硬件如果已经完成 RX Checksum Verification，也能够通过 skb 的 checksum 状态告诉协议栈：

```
这个校验硬件已经验证过了
```

这样 CPU 就没有必要再次完整计算一遍。

这也是现代网卡 Hardware Offload 能够减少 CPU 开销的重要原因之一。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/37200857e9f1b151da233f9409b64bcd_MD5.jpg]]

### **7.4 GRO：进入协议栈前先批量一下**

GRO 全称：

```
Generic Receive Offload
```

其作用可以简单理解成为：

> **把属于同一条 Flow 的多个较小 Packet，在条件允许的时候合并成为较大的 skb，再交给协议栈处理。**

例如网卡连续收到：

```
1500B
1500B
1500B
1500B
```

如果每一个 Packet 都分别经过：

```
IP
TCP
Socket
```

那么协议栈需要处理很多次。

GRO 可以在满足条件的时候把多个 Packet 聚合起来，从而减少协议栈每包处理的固定开销。

Linux 内核文档将 GRO 看成 GSO 的接收方向对应机制，目标就是在能够保持协议语义的情况下聚合接收数据。

驱动经常会调用类似：

```
napi_gro_receive(napi, skb);
```

把数据交给 GRO 以及后续网络协议栈。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/a4ea73e384521401002df5d46349b218_MD5.jpg]]

### **7.5 napi\_gro\_receive 与 netif\_receive\_skb**

从概念上来看，驱动构造完成 skb 后，最终必须把它交给通用网络接收逻辑。

常见路径会涉及：

```
napi_gro_receive()
```

或者某些场景下：

```
netif_receive_skb()
```

之后 Packet 才正式从：

```
网卡驱动世界
```

进入：

```
Linux 通用网络协议栈世界
```

至此，硬件相关的收包工作基本告一段落。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/f3f44f4d1970c1693801f7d08debd224_MD5.jpg]]

## **八、第六站：内核协议栈逐层处理**

数据包进入内核协议栈后，会严格按照**从下到上、逐层解析、层层剥壳**的逻辑处理，每一层只处理自己的协议头，剥离后交给上一层。这也是TCP/IP分层模型的落地实现。

### **8.1 链路层：Ethernet、VLAN、ARP**

数据进入通用网络协议栈以后，首先需要处理二层相关信息。

Ethernet Driver 会提供相应的协议类型。

典型情况下会涉及：

```
eth_type_trans()
```

Linux 根据 Ethernet Header 里面的 EtherType 判断上层协议。

例如：

```
ETH_P_IP
ETH_P_IPV6
ETH_P_ARP
```

如果是 VLAN，还需要进一步处理：

```
802.1Q
802.1ad
```

等 Tag 信息。

如果是 ARP，那么 Packet 会进入 ARP 的处理路径。

如果是 IPv4，则继续进入 IPv4 Receive Path。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/563e13857b5aef561ea622b1e7716079_MD5.jpg]]

### **8.2 网络层：IP 校验、路由、Netfilter**

IPv4 数据进入协议栈以后，会继续完成一系列工作。

例如：

```
检查 IP Header
检查数据包是否合法
处理分片
Netfilter
路由查找
判断 LOCAL / FORWARD
```

如果目标 IP 就是当前机器，那么 Packet 会进入：

```
LOCAL_IN
```

相关的本地接收路径。

如果机器充当路由器，那么 Packet 则有可能进入 Forward Path。

iptables/nftables 等防火墙机制也会在相关 Netfilter Hook 上参与数据包处理。

所以：

```
网卡收到 Packet
```

并不代表：

```
TCP 一定能够收到 Packet
```

中间还有可能被：

```
Netfilter
Routing
Policy
```

等逻辑所丢弃。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/90427f2ef02f98d405e3e2a41cec695f_MD5.jpg]]

**8.3 传输层：TCP / UDP 查找 Socket**

IP Header 当中还有一个非常重要的字段：

```
Protocol
```

例如：

```
6  → TCP
17 → UDP
```

如果是 TCP，就进入 TCP Receive Path。

如果是 UDP，就进入 UDP Receive Path。

传输层会根据：

```
源 IP
源 Port
目的 IP
目的 Port
Protocol
```

等信息寻找对应的 Socket。

对于已经建立连接的 TCP 来说，本质上就是根据 Flow 信息寻找对应的 TCP Socket。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/b796f1a9b8db777fe54020f24b9fa8f2_MD5.jpg]]

### **8.4 TCP 收包时到底做了什么**

TCP 并不是把 Payload 直接扔进 Socket 就结束了。

它还需要处理大量协议逻辑。

例如：

```
Sequence Number
ACK
Receive Window
Out-of-order Queue
Retransmission
SACK
TCP State
Congestion Control
Timestamp
FIN/RST
```

假设收到的数据序号刚好是：

```
期待的下一个 Seq
```

那么数据可以正常进入接收队列。

要是收到的是乱序数据：

```
SEQ = 5000

但是当前期待：

SEQ = 3000
```

TCP 还可能需要暂时保存乱序数据，等待中间缺失的数据到达以后再进行重组。

所以一个 TCP Packet 从网卡进入以后，实际上可能经历非常复杂的状态机处理。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/0408e35180e9b314270d549132133f72_MD5.jpg]]

### **8.5 数据最终进入 Socket 接收队列**

协议处理完成以后，真正需要交给应用程序的数据最终会被关联到目标 Socket 的接收缓冲机制。

我们可以简单理解成为：

```
TCP/IP Stack
      ↓
Socket Receive Buffer
      ↓
Application
```

这个时候 Packet 才算真正来到了应用程序的“门口”。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/8920197c45910cc38bd510a08295e0e9_MD5.jpg]]

## **九、第七站：应用层如何拿到数据**

很多人以为数据进Socket队列就万事大吉了，其实最后一步也会出问题。Socket队列满、进程不唤醒、读取不及时，照样丢包超时。

### **9.1 Socket Receive Queue**

假设应用程序已经建立 TCP 连接：

```
int connfd = accept(...);
```

并调用：

```
recv(connfd, buf, sizeof(buf), 0);
```

要是此时 Socket 没有数据，那么线程通常会进入阻塞状态。

当 TCP 数据到达以后，内核把数据放入 Socket 接收相关队列，并更新 Socket 状态。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/eafab305bbf9d01a951eecefc616f59d_MD5.jpg]]

### **9.2 唤醒阻塞进程与 epoll**

如果某个进程阻塞在：

```
recv()
```

那么数据到达以后就能够触发相应的唤醒逻辑。

如果应用程序使用的是：

```
epoll
```

那么 Socket 从：

```
没有数据
```

变成：

```
可以读取
```

就可能触发相应的 readiness 通知。

于是：

```
epoll_wait()
```

返回。

程序随后再调用：

```
recv()
```

真正读取数据。

因此 epoll 并不是负责“把 Packet 收进来”。

它主要是在告诉应用程序：

> **这个 FD 现在已经有条件可以继续处理了。**

![[Inbox/笔记同步助手/微信公众号/2026/09/images/7fc76a88b5765b31242282b9489c7399_MD5.jpg]]

### **9.3 recv/read：数据复制到用户空间**

普通 Socket 接收最后还有一个非常重要的过程：

```
Kernel Memory
      ↓
copy_to_user
      ↓
User Buffer
```

应用程序：

```
char buf[4096];

recv(fd, buf, sizeof(buf), 0);
```

这里的 `buf` 位于用户空间。

而 TCP 接收到的数据原本位于内核管理的网络缓冲区里面。

所以普通 `recv()` 路径一般需要把数据复制到用户空间。

也正因为如此，对于极高性能网络程序而言：

```
Kernel → User
```

这一次复制同样可能成为重要开销。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/9e661922ad154c288cd299d14adb52b5_MD5.jpg]]

### **9.4 mmap、AF\_XDP、DPDK 又是在解决什么**

既然传统 Socket Path 很长：

```
NIC
↓
Driver
↓
skb
↓
TCP/IP
↓
Socket
↓
copy_to_user
↓
Application
```

那么高性能网络框架自然会思考：

能不能把其中某些环节减少掉？

比如 PACKET\_MMAP 可以让 AF\_PACKET 使用共享 Ring 映射到用户空间，从而减少传统逐包系统调用以及数据复制相关的开销。Linux 内核文档也明确指出，`PACKET_MMAP` 使用可配置环形缓冲区并通过 `mmap()` 映射给用户进程，常被抓包程序使用。

AF\_XDP 更进一步，可以配合 XDP 和 UMEM，把 Packet 高效地送入用户态 RX Ring。其核心结构里面同样存在 RX、TX、FILL、COMPLETION 等 Ring。

DPDK 则更加激进。

典型 DPDK 程序可以绕开传统 Linux 内核协议栈的大量处理，采用用户态 Poll Mode Driver：

```
NIC
 ↓
DMA
 ↓
HugePage
 ↓
Userspace Poll
```

因此传统 Socket、AF\_XDP 和 DPDK，可以理解成为不同程度上的：

```
性能
   ↕
通用性
   ↕
开发复杂度
```

之间的权衡。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/f7607deec70c810d16e2a0c5c957cba9_MD5.jpg]]

## 十、性能优化地图：收包链路有哪些调优点

理解了完整链路以后，Linux 网络性能优化就不会再变成到处随便修改 sysctl。

因为每一个参数实际上都对应着收包链路上的某一个位置。

### **10.1 多队列、RSS、RPS、RFS**

首先是流量到底由哪个 CPU 处理。

RSS 属于网卡硬件分流。

可以理解成为：

```
Flow Hash
   ↓
RX Queue
   ↓
CPU
```

RPS 则是 Receive Packet Steering。

它是在软件层对 Packet 进一步分配 CPU。

RFS 也就是 Receive Flow Steering，则进一步考虑：

```
处理这个 Flow 的应用线程主要在哪个 CPU 上运行？
```

尽量让网络处理与应用运行具有更好的 CPU Cache Locality。

Linux 内核把 RSS、RPS、RFS 都归入 Networking Stack Scaling 机制当中。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/1dd27043adfaeb4181a099eaa0943b74_MD5.jpg]]

### **10.2 中断亲和性与 NAPI budget**

查看网卡 IRQ：

```
cat /proc/interrupts
```

可能看到：

```
eth0-TxRx-0
eth0-TxRx-1
eth0-TxRx-2
eth0-TxRx-3
```

如果所有 IRQ 都集中在一个 CPU，那么很可能出现：

```
CPU0 100%
CPU1 10%
CPU2 8%
CPU3 7%
```

这种严重不均衡的情况。

因此合理配置：

```
IRQ Affinity
RSS Queue
RPS
```

与此同时，NAPI budget 会影响单次软中断处理网络 Packet 的工作量。

数值太小可能导致 Packet 积压。

数值过大又可能导致网络处理长时间占据 CPU。

因此这种参数并不是越大越好，而是需要结合实际业务的 PPS、延迟要求以及 CPU 使用率综合判断。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/72fdd851b0b9c6add692a746c7c4973e_MD5.jpg]]

### **10.3 RX Ring 大小**

可以使用：

```
ethtool -g eth0
```

查看 Ring 参数。

某些网卡允许使用：

```
ethtool -G eth0 rx 4096
```

等办法调整 RX Ring 大小。

当 CPU 偶尔来不及处理 Packet 时，更大的 RX Ring 能够提供一定的缓冲余量。

Linux 官方网卡驱动文档同样给出了通过增大 RX Descriptor Ring 来缓解接收缓冲区耗尽导致丢包的调优思路。

但是 Ring 也不是无限大就一定更好。

Ring 过大意味着：

```
更多内存占用
更深排队
潜在更高延迟
```

所以依旧需要做权衡。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/d503b8eb0f4c5e142e4a861fc28b5e42_MD5.jpg]]

### **10.4 GRO / LRO / TSO**

这里需要区分方向。

GRO：

```
Receive
```

负责把多个接收 Packet 进行适当聚合。

TSO：

```
Transmit
```

则让系统能够把大的 TCP 数据块交给网卡，由网卡最后完成 segmentation。

LRO 也是接收方向的大包聚合，但是通常更偏硬件实现以及特定场景，语义限制与 GRO 并不完全相同。

Linux 内核文档将 TSO、GSO、GRO 等统一归入 Segmentation Offloads 体系。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/286086ef96ea24f0801e92da09f559c9_MD5.jpg]]

### **10.5 XDP / eBPF：为什么越早处理越省 CPU**

传统数据包：

```
NIC
 ↓
Driver
 ↓
skb
 ↓
Protocol Stack
 ↓
Netfilter
 ↓
Socket
```

要经过很多步骤。

可是如果一个 Packet 最终只需要：

```
DROP
```

那么完全没有必要让它一路进入 TCP/IP 协议栈。

XDP 的核心优势就在于：

> **尽量在收包路径非常早的位置处理 Packet。**

例如：

```
NIC RX
 ↓
Driver / XDP
 ↓
XDP_DROP
```

恶意流量甚至无需创建完整 skb，更不需要进入上层协议栈。

因此 XDP 非常适合：

```
DDoS 防护
Load Balancing
Packet Filtering
High Performance Forwarding
```

等场景。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/e7ff817f7dbc9daef4ca1bc7e76cad3a_MD5.jpg]]

### **10.6 网络性能最值得看的几个指标**

不要只看：

```
top
```

网络收包性能通常需要联合查看：

```
ethtool -S eth0
```

查看 NIC 层统计。

```
cat /proc/interrupts
```

查看网卡 IRQ 分布。

```
cat /proc/softirqs
```

查看 `NET_RX` SoftIRQ。

```
mpstat -P ALL 1
```

查看不同 CPU 的负载以及 softirq 使用情况。

```
nstat
```

查看协议栈统计。

```
ss -lnt
```

查看 Socket 相关队列。

真正排查问题的时候，必须将：

```
硬件
驱动
CPU
协议栈
Socket
应用
```

串起来分析。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/b374511f662dbcecd8b260baf4ef9e13_MD5.jpg]]

## 十一、排障实战：包到底丢在哪

网络排障最麻烦的一件事情就是：

> **我们明明知道“丢包了”，可是根本不知道这个包是在什么地方丢的。**

掌握完整链路之后，就能够按照层次去排查。

### **11.1 第一层：网卡有没有把包收进来**

首先执行：

```
ip -s link show eth0
```

以及：

```
ethtool -S eth0
```

重点关注网卡驱动提供的 RX 统计。

不同网卡驱动的字段名称并不统一，但是经常能够看到类似：

```
rx_errors
rx_dropped
rx_missed
rx_crc_errors
rx_no_buffer
```

如果：

```
rx_crc_errors
```

不断增加，那么问题很可能在：

```
物理链路
光模块
网线
交换机端口
PHY
```

附近。

### **11.2 第二层：RX Ring 是不是满了**

如果网卡收包速度：

```
NIC RX Rate
```

大于 CPU 清理 RX Ring 的速度：

```
NAPI Drain Rate
```

那么 RX Ring 最终可能被填满。

形成：

```
NIC
 ↓
RX Ring
 ↓
满
 ↓
新 Packet 无 Buffer
 ↓
DROP
```

这时候某些网卡统计可能出现：

```
rx_no_buffer
rx_missed
rx_dropped
```

等持续上升。

可以结合：

```
ethtool -g eth0
```

查看 Ring 大小。

### **11.3 第三层：是不是 SoftIRQ / CPU 来不及**

执行：

```
cat /proc/softirqs
```

重点观察：

```
NET_RX
```

然后：

```
mpstat -P ALL 1
```

如果发现某个 CPU 的：

```
%soft
```

非常高，那么很可能该 CPU 正在大量处理网络 SoftIRQ。

再结合：

```
cat /proc/interrupts
```

就能够判断 是不是大量网卡 IRQ 被打到一个 CPU 上。

典型问题可能就是：

```
RSS Queue 不合理
IRQ Affinity 不合理
CPU 不均衡
```

### **11.4 第四层：Linux 协议栈有没有丢**

继续可以查看：

```
nstat
```

以及：

```
netstat -s
```

这里可以观察：

```
IP
TCP
UDP
```

相关的错误以及丢包统计。

例如 UDP 接收缓冲不足、TCP 某些异常、IP 层错误，都可能在这里留下线索。

### **11.5 第五层：Socket 队列是不是堵住了**

执行：

```
ss -lnt
```

或者查看具体连接：

```
ss -ntp
```

就能够看到：

```
Recv-Q
Send-Q
```

如果 Recv-Q 持续非常大，就可能说明：

```
内核已经收到数据
但是应用程序消费速度不够
```

形成：

```
NIC
 ↓
Kernel
 ↓
Socket Receive Queue
 ↓
应用读太慢
 ↓
Queue 堆积
```

这时候再疯狂调整网卡 Ring，实际上并不能解决真正的问题。

真正的问题可能是：

```
应用线程阻塞
GC
锁竞争
业务处理过慢
epoll 使用不当
线程数量不足
```

### **11.6 tcpdump 到底抓的是哪一层**

这一点非常重要。

很多人会错误的认为：

```
tcpdump 抓到 Packet
= 应用一定应该收到 Packet
```

Linux 下 tcpdump 通常通过 libpcap 使用 AF\_PACKET 等接口从设备层附近捕获数据。Packet Socket 工作在设备驱动对应的 OSI Layer 2 层级，能够获得进入主机的数据帧。

因此可能出现：

```
NIC 收到
 ↓
tcpdump 抓到了
 ↓
Netfilter DROP
 ↓
TCP 没收到
 ↓
Application 没收到
```

所以：

> **tcpdump 能抓到包，只能证明这个包已经进入了主机相应的抓包路径附近，并不能证明最终一定进入目标 Socket。**

反过来还有一种情况。

如果包在：

```
网卡 PHY
FCS
RX FIFO
RX Descriptor
```

这些位置就已经丢失，那么普通 tcpdump 根本就看不到。

因此排障的时候要形成这样的思维：

```
ethtool 看硬件
        ↓
/proc/interrupts 看 IRQ
        ↓
/proc/softirqs 看 NAPI/SoftIRQ 压力
        ↓
nstat/netstat 看协议栈
        ↓
ss 看 Socket
        ↓
应用日志看业务
```

这才是真正完整的网络收包排障链路。

## **十二、总结：一张图记住 Linux 网络收包全链路**

学习到这里，再回头来看 Linux 网络收包，整个过程其实已经比较清晰了。

我们可以将它画成下面这样：

```
Linux 网络收包完整链路

                    网络
                     │
                     ▼
             光模块 / 网线 / PHY
                     │
                     ▼
                 NIC MAC
                     │
          CRC / VLAN / RSS Hash
                     │
                     ▼
                 RX Queue
                     │
                     │ DMA
                     ▼
              DMA RX Buffer
                     │
                     ▼
                RX Ring
                     │
                     │ MSI-X
                     ▼
                 Hard IRQ
                     │
             napi_schedule()
                     │
                     ▼
            NET_RX_SOFTIRQ
                     │
                     ▼
                NAPI poll
                     │
             清理 RX Descriptor
                     │
                     ▼
              XDP（可选）
                     │
                     ▼
            sk_buff / GRO
                     │
                     ▼
              Ethernet Layer
                     │
                     ▼
               IP / Netfilter
                     │
                     ▼
                TCP / UDP
                     │
                     ▼
            Socket Receive Queue
                     │
              wakeup / epoll
                     │
                     ▼
                recv / read
                     │
                     ▼
               用户态程序
```

整个链路里面，有三个机制尤其值得记住。

第一个就是 **DMA**。

DMA 解决的是：

```
网卡怎样高效地把 Packet 放进内存
```

第二个就是 **中断 + NAPI + SoftIRQ**。

它解决的是：

```
CPU 怎样既及时知道有 Packet，
又不会因为高 PPS 被中断彻底淹没
```

第三个就是 **Linux 分层协议栈 + Socket**。

它解决的是：

```
一个 Ethernet Frame
怎样最终找到某一个用户进程里面的 Socket
```

所以只需要记住这一句话：

> **网卡负责收帧，DMA 负责搬包，IRQ 负责通知，NAPI 负责批量收包，skb 负责承载数据，协议栈负责解析，Socket 负责排队，recv 负责交给应用。**

再进一步进行浓缩，就是：

```
网卡
 ↓
DMA
 ↓
RX Ring
 ↓
IRQ
 ↓
NAPI
 ↓
SoftIRQ
 ↓
SKB
 ↓
TCP/IP
 ↓
Socket
 ↓
Application
```

这条链路，就是 Linux 网络接收路径里面最需要建立起来的一张“脑内地图”。

而在搞懂这一条链路以后，下一步就可以继续深入另外几个非常重要的问题。

例如 Linux 网络发送路径：

```
send()
 → Socket
 → TCP/IP
 → qdisc
 → Driver
 → TX Ring
 → DMA
 → NIC
```

以及更高性能的数据面机制：

```
XDP
eBPF
AF_XDP
DPDK
io_uring
```

到了这个时候，我们再去看为什么网卡需要多队列、为什么服务器需要调整 IRQ Affinity、为什么 `ksoftirqd` 会占满 CPU、为什么 RX Ring 会导致丢包、为什么 GRO 能够提高吞吐、为什么 XDP 比 iptables 更适合极早期丢包，以及为什么 DPDK 可以做到极高 PPS，都会比单纯背诵网络参数容易理解得多。

因为它们本质上全部是在同一条链路之上，对不同阶段进行优化。

而真正掌握 Linux 网络性能的第一步，也正是先把这一条：**网卡 → DMA → RX Ring → 中断 → NAPI → SoftIRQ → 协议栈 → Socket → 应用**，完整地装进脑子里面。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/6b114ced_1789994881990?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FvoYrg8bIT9KS7ufFi1U1eg&s=obsidian)