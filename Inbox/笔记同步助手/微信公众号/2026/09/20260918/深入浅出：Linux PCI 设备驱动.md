---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247495132&idx=1&sn=446cd15e7756a298a49443956b2fd0af&chksm=c3ff1f85f740c36b5e1b82e0d25e327215c73546f36b415b48cdafb9887ecc12418bb1ddb947&mpshare=1&scene=1&srcid=0918bGWziQayn6iCrrre90Sa&sharer_shareinfo=85151e264dfd80f8cf28ebb92dcf1b0b&sharer_shareinfo_first=85151e264dfd80f8cf28ebb92dcf1b0b#rd
saved: 2026-09-18 21:21:07
tags:
  - 笔记同步助手
id: 99f0d789-bb09-42a0-b79f-9d84d9009f37
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-09-18 20:50

大家好，我是蟹老板～

在 Linux 系统之中，PCI 子系统乃是连接处理器、内存以及各种高速外设的重要桥梁。我们平时使用的网卡、显卡、NVMe 固态硬盘、采集卡、FPGA 加速卡、AI 推理卡以及 RAID 控制器，大多数都是通过 PCI 或者 PCI Express 总线接入计算机系统的。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/d56474b81532c06a3636153555ddeea7_MD5.jpg]]

从应用程序的角度来看，这些设备可能分别表现为网络接口、块设备、字符设备或者计算加速设备。但是从 Linux 内核的角度来看，它们首先都是一个个 PCI 设备。内核需要发现这些设备，读取它们的配置空间，为设备分配地址资源，匹配相应的驱动程序，建立 DMA 映射，配置中断，最终才会把设备注册到网络、存储、图形或者其他上层子系统之中。

因此，学习PCI 驱动开发，并不是简单地实现一个 `probe()` 函数，然后读取几个硬件寄存器就可以了。而是要搞懂 Linux 设备驱动模型、PCI 配置空间、BAR 资源、MMIO、中断、DMA、IOMMU、电源管理、热插拔以及错误恢复等一整套机制。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/6c9a23d30a8fc928536afdb8b4ed9690_MD5.jpg]]

## 一、PCI/PCIe 子系统基础

### 1.1 PCI 总线架构概述

PCI 的全称是 Peripheral Component Interconnect，也就是外设组件互连。传统 PCI 使用的是并行共享总线结构。多个设备共享地址线、数据线以及控制信号，同一时刻只能由一个总线主设备占用总线进行传输。其结构可以简化成为：

```
┌── PCI 设备 0
CPU/Host Bridge├── PCI 设备 1
               ├── PCI 设备 2
               └── PCI Bridge
```

多个设备共享同一条总线，必须通过总线仲裁机制决定当前由哪一个设备发起事务。PCI 设备既可以作为事务的接收方，也可以取得总线控制权，主动访问系统内存。后者就是经常所说的 Bus Master，也就是总线主控能力。随着设备带宽不断提高，共享并行总线逐渐出现信号完整性、时钟频率以及设备数量方面的限制。因此后续出现了 PCI-X 和 PCI Express。PCI Express 虽然在软件层面继续保留 PCI 配置空间、设备分类以及 BAR 等编程模型，但是在物理结构上已经从共享并行总线发展成为高速串行、点对点、分组交换的互连结构。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/5c2e1c6f1dd11a93b990a7589352b662_MD5.jpg]]

### 1.2 PCI、PCI-X、PCIe 的关系

PCI、PCI-X 和 PCIe 虽然名称相似，但硬件结构并不相同。

传统 PCI 是共享并行总线，常见位宽包括 32 位和 64 位，多个设备共享同一组总线信号；

PCI-X 可以理解为面向服务器场景的传统 PCI 扩展版本，它仍然采用并行总线，只是在总线频率和传输效率上进行了提升，曾广泛用于服务器网卡、SCSI 控制器和存储设备。PCI-X 并不是 PCI Express，二者最根本的区别在于：PCI-X 仍属于共享并行总线，而 PCI Express 已经发展为点对点高速串行互连。

PCI Express 通常简称 PCIe，它使用 Lane（通道）组成链路，常见链路宽度包括 x1、x2、x4、x8、x16。每条 Lane 都包含一组发送差分信号和一组接收差分信号，因此 PCIe 可以同时进行双向传输。虽然物理层发生了巨大变化，但 PCIe 在软件层面仍继承了传统 PCI 的 Bus、Device、Function 编址、配置空间、Vendor ID、Device ID、BAR、Command/Status 寄存器、Class Code 以及设备驱动匹配机制。正因为保留了这些软件兼容性，Linux 内核才能使用统一的 PCI 子系统管理传统 PCI、PCI-X 和 PCIe 设备。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/864236e36f2667ae6ceb5e0425d71cb0_MD5.jpg]]

### 1.3 Root Complex、Switch、Endpoint 与 PCIe 拓扑

一个典型的 PCIe 拓扑主要由 Root Complex、Root Port、Switch 和 Endpoint 构成。

```
CPU / Memory
                             │
                       Root Complex
                             │
                         Root Port
                             │
                         PCIe Switch
                    ┌────────┼────────┐
                    │        │        │
                Endpoint Endpoint Endpoint
                 网卡      NVMe      FPGA
```

Root Complex 是处理器和 PCIe 系统之间的连接点。它负责把 CPU 或内存系统产生的访问转换成为 PCIe 事务，同时也负责把 PCIe 设备发起的 DMA 请求送入内存系统。在 x86 系统中，Root Complex 通常集成在 CPU 或芯片组内部。在嵌入式 SoC 中，PCIe Root Complex 控制器通常是 SoC 的一个硬件模块。Root Port 是 Root Complex 向下连接 PCIe 设备或者 PCIe Switch 的端口。在 Linux 中，Root Port 本身也会表现为一个 PCI 设备，并且可能提供热插拔、AER、PME 等 PCIe Port Service。

PCIe Switch 具备一个上行端口和多个下行端口。它的作用和网络交换机存在一定相似之处，会依据事务中的地址或者路由信息，把 TLP 转发到相应的端口。Endpoint 就是真正执行具体业务功能的设备，例如：网卡；显卡；NVMe 控制器；FPGA 加速卡；视频采集卡；AI 推理卡。PCI-SIG 的 PCIe 基础资料将 Root Complex、Switch 和 Endpoint 描述为 PCIe 拓扑中的核心组件；PCIe 本质上是一种 Link-based 点对点互连。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/bbce26d75dfe95c02b82ee8907afd315_MD5.jpg]]

### 1.4 Bus、Device、Function 与 BDF 地址

Linux 通常使用 BDF 地址来唯一标识一个 PCI Function。BDF 分别代表：

```
Bus     总线号
Device  设备号
Function 功能号
```

常见显示形式为：

```
0000:03:00.0
```

可以拆解成为：

```
0000  PCI Domain
03    Bus
00    Device
0     Function
```

其中 Domain 用来区分多个独立的 PCI Segment。传统 PCI 编址中：Bus 字段通常为 8 位；Device 字段为 5 位；Function 字段为 3 位。因此一条传统 PCI 总线上最多可以描述 32 个 Device，每一个 Device 最多可以包含 8 个 Function。一个物理设备可以只包含一个 Function，也可以是多功能设备。例如某一块网卡可能包含：

```
03:00.0  以太网控制器
03:00.1  RDMA 功能
03:00.2  管理功能
```

需要注意的是，PCIe 的 ARI 扩展可以提供更加灵活的 Function 编号方式，尤其适用于 SR-IOV 中数量较多的虚拟功能。在 Linux 中可以通过 `lspci` 查看 BDF，示例输出：

```
03:00.0 Ethernet controller: Intel Corporation ...
04:00.0 Non-Volatile memory controller: ...
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/6df785c4886d9752084a2cc1a7e1e6ae_MD5.jpg]]

### 1.5 PCI 配置空间与设备标识

PCI 配置空间是操作系统发现和管理 PCI 设备的重要接口。传统 PCI Function 通常具备 256 字节配置空间，而 PCIe 又把配置空间扩展到 4096 字节。标准配置空间的前面部分大致如下：

```
偏移       字段
------------------------------------------------
0x00       Vendor ID
0x02       Device ID
0x04       Command
0x06       Status
0x08       Revision ID
0x09       Programming Interface
0x0A       Subclass
0x0B       Base Class
0x0C       Cache Line Size
0x0D       Latency Timer
0x0E       Header Type
0x10       BAR0
0x14       BAR1
...
0x2C       Subsystem Vendor ID
0x2E       Subsystem Device ID
0x34       Capability Pointer
0x3C       Interrupt Line
0x3D       Interrupt Pin
```

Vendor ID 用来标识设备厂商，Device ID 用来标识具体设备型号。例如驱动可以使用 `PCI_DEVICE(PCI_VENDOR_ID_INTEL, 0x1234)` 声明自己支持某一个 Intel PCI 设备。

Class Code 用来描述设备类别，例如：

```
0x01  存储控制器
0x02  网络控制器
0x03  显示控制器
0x06  桥设备
```

PCI 和 PCIe 的可选能力通过 Capability 链表以及 Extended Capability 链表进行描述，例如：Power Management；MSI；MSI-X；PCI Express Capability；AER；SR-IOV；ATS；PASID。Linux 提供了 `pci_find_capability()` 和 `pci_find_ext_capability()` 等接口来查找设备能力。PCI Core 会负责安全地遍历配置空间中的能力链表。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/0731bdc33edee279d08ab066a745fa39_MD5.jpg]]

### 1.6 PCI 枚举流程与 Linux 设备模型

系统启动以后，Linux PCI Core 会扫描可用的 PCI Host Bridge 和 Root Bus。其主要过程可以简化成为：

```
发现 PCI Host Bridge
        ↓
创建 Root Bus
        ↓
读取 Bus 0 上各个 Device/Function 的配置空间
        ↓
发现普通 Endpoint 或 PCI Bridge
        ↓
继续扫描 Bridge 下游总线
        ↓
读取 BAR 和设备能力
        ↓
分配或确认地址资源
        ↓
创建 struct pci_dev
        ↓
注册到 Linux Device Model
        ↓
匹配 pci_driver
        ↓
调用 probe()
```

Linux 不是依靠驱动程序主动遍历总线来寻找设备，而是先由 PCI Core 枚举设备并创建 `struct pci_dev`。当设备被注册到设备模型以后，PCI 总线的匹配函数会检查当前已经注册的驱动。要是某一个 `pci_driver` 的 ID Table 能够匹配该设备，就会调用相应的 `probe()`。同样，要是驱动在设备枚举完成以后才被加载，驱动注册时，设备模型也会遍历尚未绑定驱动的 PCI 设备并尝试匹配。Linux Driver Core 的绑定流程规定，设备或驱动注册时都会在对应总线上寻找匹配对象，匹配成功以后才调用 `probe()`。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/49cb1c4a8be989fffa26e65d83f34ad2_MD5.jpg]]

### 1.7 PCIe 分层、链路训练与事务层简介

PCIe 体系通常被划分为三个主要层次：

```
事务层 Transaction Layer
数据链路层 Data Link Layer
物理层 Physical Layer
```

事务层负责构造和处理 TLP，也就是 Transaction Layer Packet。常见事务包括：Memory Read；Memory Write；Configuration Read；Configuration Write；Completion；Message；Atomic Operation。CPU 对设备 MMIO 寄存器进行读取时，Root Complex 可能产生 Memory Read TLP。设备进行 DMA 写入时，则会产生 Memory Write TLP。

数据链路层负责链路级可靠性，例如：序列号；LCRC；ACK/NAK；链路级重放。这里的重传属于相邻 PCIe 组件之间的链路级重放，并不等同于 TCP 那种端到端重传。

物理层负责：Lane；电气信号；编码；链路训练；速率协商；Lane 宽度协商。设备上电或者复位以后，链路双方会通过 LTSSM 完成 Detect、Polling、Configuration 等状态，最终进入 L0 正常工作状态。Linux 普通设备驱动一般不需要直接实现链路训练，但可以通过 PCIe Capability 中的 Link Status 读取当前链路速率和宽度。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/2731e74a4b5b865409886e993f58f877_MD5.jpg]]

## 二、Linux PCI 驱动模型

### 2.1 Linux 设备驱动模型基础

Linux 设备驱动模型主要围绕下面三个对象展开：

```
Bus
Device
Driver
```

Bus 负责定义设备与驱动的匹配规则。Device 代表系统中已经存在的某一个硬件对象。Driver 则描述某一类设备的控制方法。在 PCI 子系统之中，它们大致对应：

```
PCI Bus       → struct pci_bus
PCI Device    → struct pci_dev
PCI Driver    → struct pci_driver
```

`struct pci_dev` 内部包含通用的 `struct device dev;`，而 `struct pci_driver` 内部包含 `struct device_driver driver;`。

这使得 PCI 设备和 PCI 驱动能够接入统一的 Linux Driver Core。因此，PCI Core 负责 PCI 特有的配置空间、BAR 和中断等功能，而 Driver Core 负责通用的注册、匹配、引用计数、sysfs 以及电源管理框架。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/627fdb9ee7cf240bf9730b1d7b340782_MD5.jpg]]

### 2.2 `pci_bus`、`pci_dev`、`pci_driver` 的关系

`pci_bus` 表示一条 PCI 总线。它会记录：总线编号；父总线；上游桥；该总线上的设备链表；总线资源范围；配置空间访问操作。

`pci_dev` 代表一个 PCI Function，而不一定代表一整块物理卡。它包含：BDF 信息；Vendor ID；Device ID；Class Code；BAR 资源；IRQ；PCIe Capability；电源状态；DMA 能力；所绑定的驱动；通用 `struct device`。

`pci_driver` 代表一类 PCI 设备驱动。其精简结构可以表示为：

```
/* pci_driver 描述一类 PCI 驱动，核心是设备 ID 表和各类生命周期回调。 */
struct pci_driver {
    const char *name;
    const struct pci_device_id *id_table;

    int  (*probe)(struct pci_dev *dev,
                  const struct pci_device_id *id);
    void (*remove)(struct pci_dev *dev);
    void (*shutdown)(struct pci_dev *dev);

    const struct pci_error_handlers *err_handler;
    int (*sriov_configure)(struct pci_dev *dev, int num_vfs);

    struct device_driver driver;
};
```

它们之间的关系可以表示成为：

```
pci_bus
   │
   ├── pci_dev A ──匹配── pci_driver A
   ├── pci_dev B ──匹配── pci_driver B
   └── pci_dev C ──尚未绑定驱动
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/78310cafe47c88c31fbf8f96b0de593a_MD5.jpg]]

### 2.3 `pci_device_id` 与 ID Table 匹配机制

PCI 驱动需要通过 `pci_device_id` 数组声明自己支持哪些设备。例如：

```
/* 声明驱动支持的 Vendor ID / Device ID，数组末尾必须用全零项结束。 */
static const struct pci_device_id demo_pci_ids[] = {
    {
        PCI_DEVICE(0x1234, 0x5678)
    },
    {
        PCI_DEVICE(0x1234, 0x5679)
    },
    { }
};
```

其中：

```
0x1234  Vendor ID
0x5678  Device ID
```

也可以同时匹配子系统厂商和子系统设备：

```
/* 精确匹配设备 ID，同时限定子系统厂商和子系统设备 ID。 */
{
    .vendor      = 0x1234,
    .device      = 0x5678,
    .subvendor   = 0x1111,
    .subdevice   = 0x2222,
}
```

还可以按照设备类别进行匹配：

```
/* 按 Class Code 匹配设备类别；适合类别驱动，但要避免误匹配不兼容硬件。 */
{
    .class      = PCI_CLASS_NETWORK_ETHERNET << 8,
    .class_mask = 0xffff00,
}
```

大多数设备驱动优先使用 Vendor ID 和 Device ID，因为单纯依靠 Class Code 可能匹配到并不兼容的硬件。数组末尾必须保留一个全零成员 `{ }`，用来表示 ID Table 结束。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/c7852f3971f3cec3454308fccd6ccad8_MD5.jpg]]

### 2.4 `MODULE_DEVICE_TABLE` 的作用

驱动中经常可以看到 `MODULE_DEVICE_TABLE(pci, demo_pci_ids);`，它并不是负责内核运行时匹配的唯一关键。运行时匹配主要依靠 `pci_driver.id_table`。

而 `MODULE_DEVICE_TABLE` 的重要作用，是把设备匹配信息导出到内核模块的别名元数据中。用户空间的设备管理程序能够依据 PCI 设备的 modalias，查找哪个内核模块支持这个设备，并自动执行模块加载。其流程可以简化成为：

```
PCI 设备被发现
    ↓
内核生成 MODALIAS
    ↓
用户空间设备管理程序接收到事件
    ↓
在模块别名中查找匹配项
    ↓
加载相应 .ko
    ↓
pci_register_driver()
    ↓
调用 probe()
```

如果驱动被直接编译进内核，缺少 `MODULE_DEVICE_TABLE` 通常不会影响内核内部的 ID Table 匹配；但是对于模块自动加载而言，这个宏非常重要。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/e39cce6b2533b5057ffa21b05f4b3d44_MD5.jpg]]

### 2.5 `pci_register_driver()` 与驱动注册流程

PCI 驱动一般通过 `pci_register_driver(&demo_pci_driver);` 向 PCI Core 注册。实际开发中更常见的是 `module_pci_driver(demo_pci_driver);`。

这个宏会自动生成模块初始化和退出函数，其逻辑相当于：

```
/* 模块加载时注册 PCI 驱动，模块卸载时注销驱动。 */
static int __init demo_init(void)
{
    return pci_register_driver(&demo_pci_driver);
}

static void __exit demo_exit(void)
{
    pci_unregister_driver(&demo_pci_driver);
}

module_init(demo_init);
module_exit(demo_exit);
```

驱动注册之后，PCI Core 会把它加入 PCI 驱动链表，并遍历当前尚未绑定驱动的设备。如果 ID Table 匹配成功，就会进入：

```
pci_device_probe()
        ↓
pci_match_device()
        ↓
driver probe
```

驱动卸载的时候，`pci_unregister_driver()` 会解绑该驱动所管理的所有设备，并为每一个设备调用 `remove()`。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/93f18301764525f9f32faaea29e29504_MD5.jpg]]

### 2.6 probe、remove、shutdown 生命周期

`probe()` 在设备与驱动匹配成功以后调用。它的任务通常包括：使能 PCI 设备；配置 DMA；申请 BAR；映射寄存器；分配私有结构体；初始化硬件；分配中断；注册上层接口。

`remove()` 在以下情况中可能被调用：驱动模块被卸载；PCIe 设备被热拔出；用户通过 sysfs 移除设备；设备被解绑；SR-IOV VF 被关闭。`remove()` 必须停止新请求，等待或者取消正在执行的 DMA，关闭中断，然后按照与 `probe()` 相反的顺序释放资源。

`shutdown()` 通常在系统关机或者重启阶段调用。它和 `remove()` 的目的不完全相同。系统即将关机时，内核未必需要把所有软件对象完整注销，但是驱动应当确保设备不会继续 DMA、继续产生中断或者在下一次启动时保持危险状态。Linux Driver Core 中的 `probe()` 会在任务上下文调用，驱动可以在其中分配内存、睡眠并完成硬件初始化。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/76450393d7bee3a76e654fffb0811946_MD5.jpg]]

## 三、PCI 设备初始化与资源管理

### 3.1 probe 函数什么时候被调用

`probe()` 必须同时满足两个条件才会调用：1）PCI 设备已经被 PCI Core 枚举并注册；2）某一个 PCI 驱动的匹配表能够匹配该设备。设备先出现、驱动后加载时，驱动注册过程会触发匹配。驱动先注册、设备后热插入时，设备注册过程也会触发匹配。因此 `probe()` 并不是模块加载函数本身，而是设备与驱动完成绑定以后，针对每一个匹配设备单独执行的初始化函数。一块机器上存在两块相同型号的 PCIe 卡时，同一个驱动的 `probe()` 会被调用两次，每一次接收到不同的 `struct pci_dev *`。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/1bb07637d9f6d1f9e7ebe93db5b8b679_MD5.jpg]]

### 3.2 `pci_enable_device()` 使能设备

PCI 设备在被发现以后，不一定已经允许进行 I/O、MMIO 或 Bus Master DMA。驱动一般首先调用：

```
/* 第一步先使能 PCI 设备，使 BAR 对应的 I/O 或 Memory Space 可以被访问。 */
int ret;

// 使能 PCI Function，允许内核访问设备的 I/O/MMIO 资源
ret = pci_enable_device(pdev);
if (ret)
    return ret;
```

这个函数会完成和平台相关的设备使能工作，并根据设备资源设置 PCI Command 寄存器中的 I/O Space 或 Memory Space 访问能力。如果驱动只使用 Memory BAR，也可以根据具体需求使用 `pci_enable_device_mem(pdev);`。

需要注意的是，`pci_enable_device()` 并不会自动让设备具备 Bus Master DMA 能力。要允许设备主动发起 DMA，还需要调用 `pci_set_master(pdev);`。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/0c221f10e1e016471f08d6293b24bef7_MD5.jpg]]

### 3.3 `pci_request_regions()` 申请 BAR 资源

设备的 BAR 地址已经由固件或 PCI Core 分配，但是驱动仍然需要声明自己正在使用这些资源。常用接口是：

```
/* 申请设备的 BAR 资源所有权，防止其他驱动重复占用。 */
// 声明驱动独占设备 BAR 资源
ret = pci_request_regions(pdev, "demo_pci");
if (ret)
    goto err_disable_device;
```

它的作用和普通内存区域的所有权申请类似，可以防止其他驱动或者内核模块重复占用同一段 PCI 资源。只申请指定 BAR 时，可以使用：

```
/* 只申请指定编号的 BAR 资源。 */
pci_request_region(pdev, bar, "demo_pci");
```

设备卸载时需要使用 `pci_release_regions(pdev);` 或 `pci_release_region(pdev, bar);` 释放相应资源。

推荐驱动在映射 BAR 以前先申请资源，而不是直接对一个没有取得所有权的地址执行 `ioremap()`。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/d9f17f4b72e3f073284828f3a5de529e_MD5.jpg]]

### 3.4 BAR 类型：I/O BAR、Memory BAR、ROM BAR

BAR 的全称是 Base Address Register。它告诉操作系统设备需要多大的地址空间，以及这段地址空间属于 I/O Port 还是 Memory Space。

I/O BAR 表示端口 I/O 空间。在 x86 中可以使用：

```
inb()
inw()
inl()
outb()
outw()
outl()
```

进行访问。现代 PCIe 设备大多数以 Memory BAR 为主。

Memory BAR 会被映射到 CPU 的物理地址空间。CPU 访问这段地址时，事务会被 Root Complex 转换成为 PCIe Memory Read 或 Memory Write。Memory BAR 中通常包含：控制寄存器；状态寄存器；Doorbell；描述符窗口；设备本地内存窗口。

ROM BAR 用来暴露设备固件或者 Option ROM。普通功能驱动通常不会在日常工作路径中映射 ROM BAR。需要读取 ROM 时，应当使用 PCI 子系统提供的 ROM 映射接口，而不是把它当作普通 BAR 随意访问。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/d4a11008a4755ddd5c94649f62876ef8_MD5.jpg]]

### 3.5 32-bit BAR、64-bit BAR、Prefetchable BAR

32 位 Memory BAR 的地址位于 32 位地址范围以内。

64 位 BAR 需要连续占用两个 BAR 寄存器：

```
BAR0  保存地址低 32 位
BAR1  保存地址高 32 位
```

这意味着一个 64 位 BAR 会消耗两个 BAR 槽位。驱动一般不需要手动组合高低地址，直接使用 `pci_resource_start()` 就可以得到内核解析完成的资源地址。

Prefetchable 表示设备允许系统对该区域进行预取、合并或者更加积极的缓存优化。它一般适用于设备内存或者 Framebuffer，而不是具备读取副作用的控制寄存器。设备寄存器 BAR 通常是 Non-Prefetchable，因为某些状态寄存器被读取以后可能清零，或者不同访问顺序具备特殊意义。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/1ff34009152ce4fff82580bd17302cc0_MD5.jpg]]

### 3.6 `pci_resource_start()` / `pci_resource_len()`

驱动可以读取某个 BAR 的起始地址和长度：

```
/* 读取 BAR0 的起始地址、长度和资源类型。 */
resource_size_t start;
resource_size_t len;
unsigned long flags;

start = pci_resource_start(pdev, 0);
len   = pci_resource_len(pdev, 0);
flags = pci_resource_flags(pdev, 0);
```

判断是不是 Memory BAR：

```
/* 确认该 BAR 是 Memory BAR，否则驱动无法按 MMIO 方式访问。 */
if (!(flags & IORESOURCE_MEM))
    return -ENODEV;
```

判断是不是 I/O BAR：

```
/* 判断资源是否属于传统 I/O Port 空间。 */
if (flags & IORESOURCE_IO)
    ...
```

需要注意，驱动一般不应自行向 BAR 写入全 1 来探测长度。PCI Core 在枚举和资源分配阶段已经完成了资源解析。驱动直接使用 `pci_resource_*()` 接口即可。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/1315acae8e35d92fb94e271123161e54_MD5.jpg]]

### 3.7 `pci_iomap()` / `ioremap()` 映射 MMIO

取得 BAR 资源以后，还不能把 PCI 物理地址直接当作普通指针进行访问。驱动需要建立内核虚拟地址映射：

```
/* 把 BAR0 映射到内核虚拟地址空间，后续通过 readl()/writel() 访问。 */
void __iomem *bar0;

// 映射 BAR0；长度 0 表示映射整个 BAR
bar0 = pci_iomap(pdev, 0, 0);
if (!bar0) {
    ret = -ENOMEM;
    goto err_release_regions;
}
```

第三个参数为最大映射长度。传入 0 通常表示映射整个 BAR。卸载时使用 `pci_iounmap(pdev, bar0);` 解除映射。

也可以取得物理地址后手动调用：

```
/* 也可以手动取得 BAR 物理范围再调用 ioremap()，但 PCI 驱动通常优先 pci_iomap()。 */
bar0 = ioremap(pci_resource_start(pdev, 0),
               pci_resource_len(pdev, 0));
```

但是对于 PCI 设备而言，优先使用 `pci_iomap()` 更加合适，因为它能够根据 BAR 类型选择相应的映射方式。Linux 还提供了资源托管接口，例如：

```
pcim_enable_device()
pcim_iomap_regions()
devm_kzalloc()
devm_request_irq()
```

它们可以减少部分错误回滚代码。但是驱动仍然必须正确停止硬件、DMA 和中断，不能认为使用托管接口以后就不需要设计生命周期。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/97ec8be5971abdd3414d4f670d98e8df_MD5.jpg]]

### 3.8 设备私有结构体设计

驱动通常需要为每一个设备实例分配一个私有结构体：

```
/* 每个 PCI Function 都应拥有独立的私有数据结构，保存该设备实例的运行状态。 */
struct demo_device {
    struct pci_dev *pdev;

    void __iomem *bar0;
    resource_size_t bar0_len;

    int irq;
    spinlock_t lock;

    void *dma_virt;
    dma_addr_t dma_handle;
    size_t dma_size;

    bool running;
};
```

在 `probe()` 中分配：

```
/* 为当前 PCI 设备实例分配并清零私有数据。 */
struct demo_device *ddev;

// 分配并清零设备私有结构体
ddev = kzalloc(sizeof(*ddev), GFP_KERNEL);
if (!ddev) {
    ret = -ENOMEM;
    goto err_iounmap;
}
```

然后通过 `pci_set_drvdata(pdev, ddev);` 保存到 `pci_dev`。

其他回调中取回：

```
/* 在其他回调中从 pci_dev 取回 probe() 阶段保存的私有数据。 */
struct demo_device *ddev;

// 取回 probe() 阶段保存的设备私有数据
ddev = pci_get_drvdata(pdev);
```

如此一来，多个相同设备可以各自拥有独立的寄存器映射、DMA 缓冲区、中断号以及锁。

### 3.9 probe 失败路径与资源回滚

`probe()` 任何一步失败，都必须释放前面已经成功申请的资源。资源释放应当严格按照申请顺序的相反方向进行。例如：

```
/* probe() 任一步失败，都按“后申请、先释放”的顺序回滚资源。 */
static int demo_probe(struct pci_dev *pdev,
                      const struct pci_device_id *id)
{
    int ret;

    // 使能 PCI Function，允许内核访问设备的 I/O/MMIO 资源
    ret = pci_enable_device(pdev);
    if (ret)
        return ret;

    // 声明驱动独占设备 BAR 资源
    ret = pci_request_regions(pdev, "demo_pci");
    if (ret)
        goto err_disable;

    // 映射 BAR0；长度 0 表示映射整个 BAR
    bar0 = pci_iomap(pdev, 0, 0);
    if (!bar0) {
        ret = -ENOMEM;
        goto err_regions;
    }

    ret = demo_alloc_dma();
    if (ret)
        goto err_iounmap;

    ret = demo_request_irq();
    if (ret)
        goto err_dma;

    return 0;

err_dma:
    demo_free_dma();
err_iounmap:
    // 解除 BAR0 的 MMIO 映射
    pci_iounmap(pdev, bar0);
err_regions:
    // 释放此前申请的 BAR 资源
    pci_release_regions(pdev);
err_disable:
    // 关闭设备的 PCI 访问能力
    pci_disable_device(pdev);
    return ret;
}
```

这种方式虽然标签比较多，但是它能够明确说明每一步资源的所有权。PCI 驱动最危险的问题之一，就是 `probe()` 中途失败以后留下仍在运行的 DMA、仍然打开的中断或者没有释放的 BAR 映射。

## 四、MMIO 与设备寄存器访问

### 4.1 CPU 如何访问 PCI 设备寄存器

PCI 设备的 MMIO BAR 被分配到系统物理地址空间以后，CPU 访问这段地址时，并不一定真正访问 DRAM。Root Complex 会根据地址范围判断这次访问属于 PCIe 设备，然后把它转换成为 PCIe TLP。例如：

```
/* 向 BAR0 中的控制寄存器写入 1，触发一次 MMIO 写操作。 */
writel(0x1, ddev->bar0 + REG_CONTROL);
```

其逻辑过程可以理解成为：

```
CPU 执行 MMIO 写
      ↓
地址命中 PCIe MMIO 窗口
      ↓
Root Complex 生成 Memory Write TLP
      ↓
Switch 路由
      ↓
Endpoint 接收
      ↓
设备控制寄存器被写入
```

MMIO 映射地址必须使用：

```
void __iomem *
```

进行标记。这个标记可以帮助编译器检查工具发现驱动是否错误地把设备地址当作普通内存地址使用。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/d0201d28d773ae2ec216edf0b834b079_MD5.jpg]]

### 4.2 `readb/readw/readl/readq`

Linux 提供不同访问宽度的 MMIO 读取接口：

```
/* 使用 Linux MMIO Accessor 按寄存器宽度读取设备寄存器。 */
u8  value8  = readb(addr);
u16 value16 = readw(addr);
u32 value32 = readl(addr);
u64 value64 = readq(addr);
```

它们分别表示：

```
readb  8 位
readw  16 位
readl  32 位
readq  64 位
```

`readq()` 并不是在所有 32 位体系结构上都能够天然完成原子的 64 位设备访问。设备要求特定高低位顺序时，需要使用架构和设备协议允许的 64 位访问辅助接口。还存在：

```
ioread8()
ioread16()
ioread32()
```

它们可以提供更加总线无关的 I/O 访问方式。

### 4.3 `writeb/writew/writel/writeq`

相应的写入接口为：

```
/* 使用 Linux MMIO Accessor 按寄存器宽度写设备寄存器。 */
writeb(value8, addr);
writew(value16, addr);
writel(value32, addr);
writeq(value64, addr);
```

例如：

```
/* 定义寄存器偏移，并通过 writel()/readl() 完成 32 位 MMIO 访问。 */
#define DEMO_REG_CONTROL 0x0000
#define DEMO_REG_STATUS  0x0004

writel(DEMO_CTRL_START,
       ddev->bar0 + DEMO_REG_CONTROL);

// 读取设备状态寄存器，确认中断或硬件状态
status = readl(ddev->bar0 + DEMO_REG_STATUS);
```

驱动必须依据硬件手册规定的寄存器宽度进行访问。如果设备规定某个寄存器必须使用 32 位写入，就不能为了方便而使用四次 `writeb()`。

### 4.4 MMIO 访问顺序与内存屏障

编译器、CPU、互连以及设备都可能对访问进行一定程度的重排序。假设驱动需要先准备 DMA 描述符，再通知设备：

```
/* 错误示例：直接写 Doorbell 前没有显式保证 DMA 描述符字段的可见顺序。 */
desc->addr = dma_addr;
desc->len  = data_len;
desc->flags = DESC_VALID;

// 敲 Doorbell，通知设备开始处理已经准备好的描述符/队列
writel(DOORBELL_START, doorbell);
```

必须保证描述符内容在 Doorbell 写入以前已经对设备可见。常见方式为：

```
/* 正确做法：先写描述符，再用 dma_wmb() 保证设备先看到描述符内容，最后写 Doorbell。 */
desc->addr = dma_addr;
desc->len  = data_len;
desc->flags = DESC_VALID;

// DMA 写屏障：保证前面的描述符内容先对设备可见
dma_wmb();

// 敲 Doorbell，通知设备开始处理已经准备好的描述符/队列
writel(DOORBELL_START, doorbell);
```

当 CPU 读取设备更新的 DMA 描述符时，也可能需要使用：

```
/* 在读取设备写回的数据之前，建立 DMA 读屏障。 */
// DMA 读屏障：保证之后读取的是设备已经写回的最新字段
dma_rmb();
```

需要特别注意，即便内存由 `dma_alloc_coherent()` 分配，也不代表 CPU 指令之间完全不需要内存屏障。Coherent 保证的是 CPU 和设备对内存内容的可见性关系，不代表编译器和 CPU 不会调整多个普通存储操作的先后顺序。Linux DMA 文档明确提醒，Coherent DMA 内存仍然可能需要适当的内存屏障。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/834da20b5cca0c30390f0882ca05be69_MD5.jpg]]

### 4.5 posted write 问题

PCIe Memory Write 通常属于 Posted Write。所谓 Posted，是指 CPU 把写请求提交给互连以后，不需要等待目标设备返回完成报文，就可以继续执行。因此：

```
/* PCIe Memory Write 通常是 Posted Write；函数返回并不代表写已经抵达设备。 */
writel(value, reg);
```

返回时，数据可能只是进入 Root Complex 或 PCIe Bridge 的写缓冲区，并不一定已经真正到达设备。如果驱动在写入以后必须确认设备已经看到该操作，可以对一个安全寄存器执行读取：

```
/* 通过一次安全的 MMIO 读取，等待 Completion，从而刷新前面的 Posted Write。 */
writel(value, control_reg);

/* 读取同一设备的安全寄存器，刷新 posted write */
readl(status_reg);
```

读操作需要等待 Completion 返回，因此通常能够把前面的 Posted Write 推送到设备。但是刷新读取必须选择硬件手册允许读取的寄存器。不能读取具备清零副作用的状态寄存器，也不能在设备已经掉线或者 AER 隔离期间盲目访问 MMIO。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/d894e2241087318df9b2f84dd0c2655a_MD5.jpg]]

### 4.6 为什么不能直接解引用 MMIO 地址

下面的写法是错误的：

```
/* 错误示例：不要把 __iomem 地址强转成普通 volatile 指针后直接解引用。 */
volatile u32 *reg = (volatile u32 *)addr;

*reg = value;
```

原因主要包括：1）普通指针访问不具备 Linux I/O 访问器规定的体系结构语义；2）可能缺少必要的字节序转换；3）可能缺少 I/O 顺序保证；4）某些体系结构的 I/O 地址无法像普通内存一样直接访问；5）`volatile` 只能限制部分编译器优化，不能替代内存屏障；6）静态检查工具无法正确判断地址空间。因此必须使用：

```
/* 正确方式是使用内核提供的 I/O Accessor。 */
readl()
writel()
ioread32()
iowrite32()
```

Linux 的总线无关设备 I/O 文档明确要求通过 I/O Accessor 访问 MMIO，而不是直接把设备映射地址作为普通内存指针解引用。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/ce24f9f80950618081dfbaff44be85c2_MD5.jpg]]

## 五、PCI 中断处理

### 5.1 INTx 传统中断与共享中断

INTx 是传统 PCI 中断机制，包含：

```
INTA#
INTB#
INTC#
INTD#
```

INTx 使用电平触发方式，并且可能被多个设备共享。驱动注册共享中断时，需要使用：

```
/* 为传统 INTx 共享中断注册处理函数；dev_id 用于区分共享同一 IRQ 的设备。 */
request_irq(irq,
            demo_irq_handler,
            IRQF_SHARED,
            "demo_pci",
            ddev);
```

其中 `dev_id` 不能随意传入 `NULL`，因为内核需要依靠它区分共享同一个 IRQ 的不同处理程序。共享中断处理函数必须首先读取设备状态，判断中断是不是由自己的设备产生：

```
/* 共享中断处理函数必须先确认中断确实来自本设备，否则返回 IRQ_NONE。 */
static irqreturn_t demo_irq_handler(int irq, void *data)
{
    struct demo_device *ddev = data;
    u32 status;

    // 读取设备状态寄存器，确认中断或硬件状态
    status = readl(ddev->bar0 + DEMO_REG_INT_STATUS);

    if (!(status & DEMO_INT_PENDING))
        return IRQ_NONE;

    // 按设备定义写回状态位，通常用于清除已处理的中断
    writel(status, ddev->bar0 + DEMO_REG_INT_STATUS);

    return IRQ_HANDLED;
}
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/5fd32c636e53a3eec5241a1f703e39c6_MD5.jpg]]

### 5.2 MSI 原理与使用方式

MSI 的全称是 Message Signaled Interrupt。它不再使用单独的物理中断线，而是由 PCI 设备向一个特殊地址执行 Memory Write。写入的数据中包含中断向量信息。其过程可以理解成为：

```
设备需要产生中断
      ↓
设备发起特殊 Memory Write
      ↓
中断控制器接收写事务
      ↓
目标 CPU 收到中断
```

MSI 的优点包括：不需要共享传统 INTx 线路；避免部分共享中断判断开销；可以更加灵活地路由到 CPU；Memory Write 和前面的 DMA 写入具备更加明确的顺序关系。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/9177c65c941f6d5118041139ffc1cf09_MD5.jpg]]

### 5.3 MSI-X 原理与多队列设备

MSI-X 是比 MSI 更加灵活的中断机制。MSI-X 支持更多中断向量，并且每一个向量都可以独立配置。设备中通常存在 MSI-X Table 和 Pending Bit Array，它们位于某一个 BAR 空间之中。多队列网卡可以为每一个接收或发送队列分配单独的 MSI-X 向量：

```
RX Queue 0 → IRQ 100 → CPU 0
RX Queue 1 → IRQ 101 → CPU 1
RX Queue 2 → IRQ 102 → CPU 2
RX Queue 3 → IRQ 103 → CPU 3
```

这样就能够减少多个 CPU 对同一个队列和同一把锁的争用。NVMe 控制器同样经常为多个 I/O Queue 配置独立的 MSI-X 向量。Linux 内核的 MSI 指南说明，MSI-X 相比 MSI 支持更多中断，并允许每个向量被独立配置。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/0e1a0821475885ea698d25ffeb621e5c_MD5.jpg]]

### 5.4 `pci_alloc_irq_vectors()`

现代 PCI 驱动推荐使用 `pci_alloc_irq_vectors()` 统一申请 INTx、MSI 或 MSI-X。例如：

```
/* 统一申请中断向量，允许 MSI-X → MSI → INTx 的兼容退化。 */
int nr_vectors;

// 向 PCI Core 申请可用的中断向量
nr_vectors = pci_alloc_irq_vectors(
    pdev,
    1,
    8,
    PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_LEGACY);

if (nr_vectors < 0)
    return nr_vectors;
```

这个调用表示：至少需要 1 个向量；最多申请 8 个向量；优先允许 MSI-X；MSI-X 不可用时可以退化为 MSI；MSI 也不可用时可以退化为 INTx。可以通过 `pci_irq_vector(pdev, vector_index)` 获取某一个 Linux IRQ 编号，并使用 `pci_free_irq_vectors(pdev)` 释放向量。

必须先释放通过 `request_irq()` 注册的处理程序，再调用 `pci_free_irq_vectors()`。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/e818684b7c3b37c0c8ef05f2ac320813_MD5.jpg]]

### 5.5 `request_irq()` 注册中断处理函数

完整流程如下：

```
/* 先分配向量，再取 Linux IRQ 号，最后向内核注册中断处理函数。 */
// 向 PCI Core 申请可用的中断向量
ret = pci_alloc_irq_vectors(pdev, 1, 1,
                            PCI_IRQ_MSIX |
                            PCI_IRQ_MSI |
                            PCI_IRQ_LEGACY);
if (ret < 0)
    goto err_dma;

// 把第 0 个 PCI 中断向量转换为 Linux IRQ 号
ddev->irq = pci_irq_vector(pdev, 0);

// 向 Linux IRQ 子系统注册中断处理函数
ret = request_irq(ddev->irq,
                  demo_irq_handler,
                  0,
                  "demo_pci",
                  ddev);
if (ret)
    goto err_vectors;
```

卸载时：

```
/* 卸载时先注销 IRQ Handler，再释放 PCI 中断向量。 */
// 注销 IRQ Handler；必须使用与注册时相同的 dev_id
free_irq(ddev->irq, ddev);
// 释放 pci_alloc_irq_vectors() 申请的所有向量
pci_free_irq_vectors(pdev);
```

如果实际退化到了共享 INTx，还需要根据驱动设计传入 `IRQF_SHARED`。更常见的做法是根据分配结果以及设备能力确定请求方式。Linux PCI 驱动文档要求先通过 `pci_alloc_irq_vectors()` 分配向量，再使用 `pci_irq_vector()` 得到可传递给 `request_irq()` 的 Linux IRQ。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/0413f266d148c96731e092b4ac4efc7e_MD5.jpg]]

### 5.6 顶半部与底半部

中断处理通常被划分为顶半部和底半部。

顶半部就是硬中断处理函数。它应当尽量快速完成：读取中断状态；确认中断来源；清除或屏蔽设备中断；保存最少量状态；调度后续处理。硬中断上下文不能执行可能睡眠的操作。  
底半部主要承接耗时工作，应当把这些工作推迟到更加合适的上下文执行，例如：Workqueue；Threaded IRQ；NAPI；Softirq；Tasklet。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/8f39c2105327ce9975fe3d9066877941_MD5.jpg]]

### 5.7 tasklet、workqueue、threaded IRQ

Tasklet 运行在软中断上下文，不能睡眠。老驱动中比较常见，但是新驱动通常更倾向使用 Workqueue、Threaded IRQ 或者特定子系统机制。

Workqueue 运行在线程上下文，可以执行允许睡眠的操作，例如通过 `schedule_work(&ddev->work)` 调度工作，适合处理固件、设备复位、较慢的状态更新以及需要使用 Mutex 的操作。

Threaded IRQ 可以使用 `request_threaded_irq()` 注册硬中断顶半部和线程化处理函数：

```
/* Threaded IRQ：顶半部快速确认事件，耗时工作交给内核中断线程。 */
// 注册顶半部 + 线程化底半部中断处理
ret = request_threaded_irq(
    irq,
    demo_irq_top,
    demo_irq_thread,
    IRQF_ONESHOT,
    "demo_pci",
    ddev);
```

顶半部可以返回 `IRQ_WAKE_THREAD`，然后由内核唤醒中断线程执行耗时操作。  
NAPI 常用于网络驱动。网络驱动通常不会为每一个数据包完整执行一次中断处理，而是通过 NAPI 在高负载下切换为轮询，以降低中断风暴。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/230fa0d997b383fd509c4b71645d9c81_MD5.jpg]]

### 5.8 中断亲和性与负载均衡

中断亲和性决定某个 IRQ 可以在哪些 CPU 上运行。查看中断：

```
# 查看当前系统全部 IRQ 及其命中次数。
cat /proc/interrupts
```

查看亲和性：

```
# 查看指定 IRQ 允许在哪些 CPU 上运行。
cat /proc/irq/<IRQ>/smp_affinity_list
```

设置到 CPU 4：

```
# 把指定 IRQ 的亲和性设置到 CPU 4。
echo 4 > /proc/irq/<IRQ>/smp_affinity_list
```

多队列设备需要综合考虑：IRQ 所在 CPU；应用程序所在 CPU；DMA 内存所在 NUMA 节点；网卡队列；RSS；NVMe I/O Queue；irqbalance。单纯把所有中断绑定到 CPU 0，可能导致 CPU 0 成为瓶颈。但是把中断均匀分散到全部 CPU，也不一定最优，因为跨 NUMA 访问以及缓存迁移可能带来额外开销。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/cd4ff83d5ab00c7e059bfce61b0a40cd_MD5.jpg]]

## 六、PCI DMA 机制

### 6.1 为什么 PCI 设备需要 DMA

如果设备每传输一个字节，都需要 CPU 读取设备寄存器再复制到内存，CPU 开销会非常大。DMA 允许设备直接访问系统内存。例如网卡接收数据时：

```
网卡接收到以太网帧
       ↓
设备读取 RX 描述符
       ↓
设备把数据 DMA 到内存缓冲区
       ↓
设备更新完成状态
       ↓
产生 MSI-X 中断
       ↓
驱动处理数据包
```

CPU 不需要逐字节搬运数据，只需要准备描述符和缓冲区，并在完成后处理结果。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/865d99ebc063eb7b5fbbff4a7c950226_MD5.jpg]]

### 6.2 Bus Master DMA 的含义

Bus Master 表示 PCI 设备可以主动发起总线事务，驱动通常需要调用 `pci_set_master(pdev);`。

它会设置 PCI Command 寄存器中的 Bus Master Enable 位。如果没有打开这个位，即便设备内部 DMA 引擎已经启动，它也可能无法向系统内存发起 PCIe Memory Read 或 Memory Write。设备关闭或者错误恢复时，有些驱动会使用 `pci_clear_master(pdev);` 阻止设备继续发起 DMA。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/91d3d021ce780c5efe9aae60103afc94_MD5.jpg]]

### 6.3 DMA 地址、物理地址、虚拟地址的区别

DMA 驱动开发中最容易混淆的是 CPU 虚拟地址、CPU 物理地址和 DMA 地址。CPU 虚拟地址是内核代码访问内存时使用的地址，例如 `void *cpu_addr`；CPU 物理地址是 DRAM 在处理器物理地址空间中的地址；DMA 地址则是设备发起 DMA 时真正使用的地址，通常以 `dma_addr_t` 表示。存在 IOMMU 时，DMA 地址往往是 IOVA，而不是 CPU 物理地址，其转换关系可以表示为：

```
CPU 虚拟地址
      ↓ 页表
CPU 物理地址
      ↓ IOMMU
设备 DMA 地址 / IOVA
```

因此驱动不能使用 `virt_to_phys()` 替代 DMA API。设备真正应当使用的是 DMA API 返回的 `dma_addr_t`。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/9593291698a6ea9b130a3e38137347e6_MD5.jpg]]

### 6.4 Streaming DMA 与 Coherent DMA

Linux DMA API 主要提供两种常见模型。

Coherent DMA 常用于：DMA 描述符环；命令队列；Completion Queue；设备和 CPU 频繁共享的小型控制结构。通过 `dma_alloc_coherent()` 申请。CPU 和设备通常能够持续观察对方的更新，不需要每次使用前后进行显式 Cache 同步。

Streaming DMA 常用于：网络数据包；存储数据块；大容量传输缓冲区；临时 DMA 数据。通过 `dma_map_single()`、`dma_map_page()` 或 `dma_map_sg()` 建立映射。一次传输完成后通常需要取消映射，或者在长期保持映射时使用 `dma_sync_*()` 在 CPU 和设备之间切换所有权。Linux DMA 文档把 Streaming Mapping 描述为通常针对单次或者阶段性 DMA 传输建立的映射，而 Coherent Allocation 则适合持续共享的控制结构。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/85be2b0f4e1eb003b6dae11fbf2fc231_MD5.jpg]]

### 6.5 `dma_set_mask_and_coherent()`

设备支持的 DMA 地址位数可能不同。驱动应当在 `probe()` 中声明：

```
/* 优先协商 64 位 DMA 地址能力，不支持时再退化到 32 位。 */
// 声明设备和平台可接受的 DMA 地址位宽
ret = dma_set_mask_and_coherent(&pdev->dev,
                                DMA_BIT_MASK(64));
if (ret) {
    // 声明设备和平台可接受的 DMA 地址位宽
    ret = dma_set_mask_and_coherent(&pdev->dev,
                                    DMA_BIT_MASK(32));
    if (ret)
        goto err_regions;
}
```

这段代码表示：1）优先尝试 64 位 DMA；2）设备或平台不支持时退化为 32 位；3）32 位也不支持时，驱动无法正常工作。不能因为设备是 PCIe 设备，就默认它一定支持 64 位 DMA。某些设备内部 DMA 地址寄存器只有 32 位。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/1f201694922fa5f1f16d907a3f688f13_MD5.jpg]]

### 6.6 `dma_alloc_coherent()`

申请 Coherent DMA 内存：

```
/* 申请一块 CPU 与设备持续共享的 Coherent DMA 内存。 */
ddev->dma_size = 4096;

// 申请 Coherent DMA：同时得到 CPU 虚拟地址和设备 DMA 地址
ddev->dma_virt = dma_alloc_coherent(
    &pdev->dev,
    ddev->dma_size,
    &ddev->dma_handle,
    GFP_KERNEL);

if (!ddev->dma_virt) {
    ret = -ENOMEM;
    goto err_iounmap;
}
```

其中：

```
dma_virt    CPU 使用的虚拟地址
dma_handle  写入设备寄存器的 DMA 地址
```

配置设备：

```
/* 把 dma_alloc_coherent() 返回的 DMA 地址拆成高低 32 位写入设备寄存器。 */
// 写入 DMA 地址低 32 位
writel(lower_32_bits(ddev->dma_handle),
       ddev->bar0 + DEMO_DMA_ADDR_LOW);

// 写入 DMA 地址高 32 位
writel(upper_32_bits(ddev->dma_handle),
       ddev->bar0 + DEMO_DMA_ADDR_HIGH);
```

释放：

```
/* 释放 Coherent DMA 内存时，大小、CPU 地址和 DMA 地址必须与申请时一致。 */
dma_free_coherent(&pdev->dev,
                  ddev->dma_size,
                  ddev->dma_virt,
                  ddev->dma_handle);
```

![[Inbox/笔记同步助手/微信公众号/2026/09/images/7e44f20d45af8e1026e98473c226d4b7_MD5.jpg]]

### 6.7 `dma_map_single()` / `dma_unmap_single()`

对于普通内存缓冲区：

```
/* 把普通内存临时映射成设备可访问的 Streaming DMA 地址。 */
void *buffer;
dma_addr_t dma_addr;

buffer = kmalloc(size, GFP_KERNEL);
if (!buffer)
    return -ENOMEM;

// 建立 Streaming DMA 映射，返回设备真正应使用的 DMA 地址
dma_addr = dma_map_single(&pdev->dev,
                          buffer,
                          size,
                          DMA_TO_DEVICE);

if (dma_mapping_error(&pdev->dev, dma_addr)) {
    kfree(buffer);
    return -EIO;
}
```

设备完成传输以后：

```
/* DMA 完成后撤销 Streaming Mapping，把映射关系交还 DMA 子系统。 */
// 撤销 Streaming DMA 映射
dma_unmap_single(&pdev->dev,
                 dma_addr,
                 size,
                 DMA_TO_DEVICE);
```

DMA 方向包括：

```
DMA_TO_DEVICE
DMA_FROM_DEVICE
DMA_BIDIRECTIONAL
```

方向不能随意填写，它会影响 Cache 同步和 IOMMU 权限。下面这些内存通常不能直接使用 `dma_map_single()`：栈内存；普通 `vmalloc()` 连续虚拟内存；生命周期不足的临时对象。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/0b802d8cae1d89b453039fdb0d59d972_MD5.jpg]]

### 6.8 Scatter-Gather DMA

一份逻辑上连续的数据，在物理内存中可能分散在多个页面。Scatter-Gather DMA 允许设备通过多个 DMA Segment 完成一次逻辑传输。驱动先构造：

```
/* scatterlist 用来描述逻辑连续、物理上可能分散的内存片段。 */
struct scatterlist *sgl;
```

然后调用：

```
/* 把 scatterlist 映射成设备真正可用的 DMA Segment。 */
// 把 SG 列表转换为 DMA Segment，返回值可能小于原始条目数
nr_dma = dma_map_sg(&pdev->dev,
                    sgl,
                    nr_entries,
                    DMA_TO_DEVICE);
```

需要注意，返回的 `nr_dma` 可能小于原始 `nr_entries`，因为 DMA 层可能把相邻条目合并。设备描述符中应当使用：

```
/* 设备描述符必须使用 DMA API 生成的地址和长度，而不是原始物理地址。 */
sg_dma_address(sg)
sg_dma_len(sg)
```

而不是普通的页面物理地址。完成以后：

```
/* 解除 SG 映射时传入原始条目数 nr_entries，而不是合并后的 nr_dma。 */
dma_unmap_sg(&pdev->dev,
             sgl,
             nr_entries,
             DMA_TO_DEVICE);
```

取消映射时传入的是原始条目数量，而不是合并后的 DMA Segment 数量。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/47f05534055b70cfa1c7ac5fe302aedb_MD5.jpg]]

### 6.9 IOMMU 对 DMA 的影响

IOMMU 位于 PCIe 设备和系统内存之间。它可以把设备看到的 DMA 地址转换成为实际物理地址：

```
设备发起 DMA 到 IOVA 0x100000
            ↓
IOMMU 查页表
            ↓
转换到物理内存 0x8000100000
```

IOMMU 的作用包括：隔离设备 DMA；防止设备访问未授权内存；支持虚拟机设备直通；为不连续物理页面提供连续 IOVA；支持地址重映射；支持中断重映射。因此在开启 IOMMU 的系统中，DMA 地址并不等于物理地址，驱动必须始终通过 DMA API 获取地址。VFIO 之所以能够较安全地把 PCI 设备交给用户空间或虚拟机，很大程度上依赖 IOMMU 对 DMA 地址范围进行隔离。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/d6e32852aadf8b254bf610ca3fdefb21_MD5.jpg]]

### 6.10 DMA 缓存一致性与内存屏障

DMA 缓存一致性主要解决：

```
CPU Cache 中的数据
设备实际访问的内存
二者是不是一致
```

在非一致性体系结构中，Streaming DMA Mapping 可能需要 DMA API 执行 Cache Clean 或 Invalidate。驱动不能自行假定平台是 x86，就省略 DMA API。对于设备描述符，常见所有权转换方式为：

```
/* 提交描述符时先写数据字段，再用 dma_wmb()，最后交出所有权。 */
desc->addr = dma_addr;
desc->len = len;

/* 先让地址和长度对设备可见 */
// DMA 写屏障：保证前面的描述符内容先对设备可见
dma_wmb();

/* 最后交出所有权 */
desc->flags = DESC_OWNED_BY_DEVICE;
```

接收完成时：

```
/* 确认设备完成后先执行 dma_rmb()，再读取设备写回的其他字段。 */
if (desc->flags & DESC_DONE) {
    // DMA 读屏障：保证之后读取的是设备已经写回的最新字段
    dma_rmb();

    /* 现在再读取设备写回的长度和状态 */
    len = desc->actual_len;
}
```

`dma_wmb()` 和 `dma_rmb()` 解决的是 DMA 观察顺序问题。而：

```
/* MMIO Accessor 负责设备寄存器访问；它与 DMA 内存屏障解决的是不同问题。 */
writel()
readl()
```

解决的是设备 MMIO 操作。两者不能简单互相替代。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/f37c934d53061ea8571a80317d16e66e_MD5.jpg]]

## 七、电源管理、热插拔与错误恢复

### 7.1 PCI 电源状态 D0–D3

PCI 设备常见电源状态包括：

```
D0      完全工作状态
D1      可选的轻度低功耗状态
D2      可选的更深低功耗状态
D3hot   配置空间仍可访问的低功耗状态
D3cold  设备供电可能已经被切断
```

D0 又可以区分初始化前后的状态。D3cold 时设备可能完全失去寄存器和配置状态，恢复时接近一次重新上电。驱动不能假定所有设备都支持 D1、D2，也不能在没有保存状态和停止 DMA 的情况下直接进入 D3。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/64cdb89aa963a01d59b387654bd9abdb_MD5.jpg]]

### 7.2 Runtime PM

Runtime PM 允许设备在系统正常运行期间，因为暂时空闲而进入低功耗状态。驱动可以定义：

```
/* 定义 Runtime PM 三个核心回调：挂起、恢复和空闲判断。 */
static const struct dev_pm_ops demo_pm_ops = {
    .runtime_suspend = demo_runtime_suspend,
    .runtime_resume  = demo_runtime_resume,
    .runtime_idle    = demo_runtime_idle,
};
```

在 `pci_driver` 中设置：

```
/* 把 PM 回调挂接到 pci_driver 内部的通用 device_driver。 */
.driver = {
    .name = "demo_pci",
    .pm = &demo_pm_ops,
},
```

使用设备前，可以调用：

```
/* 访问设备前增加 Runtime PM 使用计数，并确保设备已经恢复到工作状态。 */
ret = pm_runtime_resume_and_get(&pdev->dev);
```

使用完成以后：

```
/* 访问结束后记录最后忙碌时间，并允许设备按 autosuspend 策略进入低功耗。 */
pm_runtime_mark_last_busy(&pdev->dev);
pm_runtime_put_autosuspend(&pdev->dev);
```

Runtime Suspend 通常需要：1）停止新请求；2）等待正在执行的 DMA；3）关闭中断；4）保存设备状态；5）让硬件进入低功耗模式。Runtime Resume 则按照相反顺序恢复。Linux PM Core 提供 `runtime_suspend`、`runtime_resume` 和 `runtime_idle` 三类主要 Runtime PM 回调，并负责这些操作之间的同步。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/d6d87c95f2bd454f98cfe8d8a19461e5_MD5.jpg]]

### 7.3 suspend / resume 回调

系统进入 Suspend-to-RAM 或休眠时，会触发系统级电源管理回调。典型结构为：

```
/* 系统休眠前停止 I/O、关闭中断并保存 PCI 状态；恢复时按相反顺序重建。 */
static int demo_suspend(struct device *dev)
{
    struct pci_dev *pdev = to_pci_dev(dev);
    struct demo_device *ddev = pci_get_drvdata(pdev);

    demo_stop_io(ddev);
    demo_disable_irqs(ddev);

    // 保存 PCI 配置空间状态，为系统恢复做准备
    pci_save_state(pdev);
    // 关闭 Bus Master，阻止设备继续主动 DMA
    pci_clear_master(pdev);

    return 0;
}

static int demo_resume(struct device *dev)
{
    struct pci_dev *pdev = to_pci_dev(dev);
    struct demo_device *ddev = pci_get_drvdata(pdev);

    // 恢复休眠前保存的 PCI 配置空间状态
    pci_restore_state(pdev);
    // 打开 Bus Master Enable，允许设备主动发起 DMA
    pci_set_master(pdev);

    demo_reinit_hw(ddev);
    demo_enable_irqs(ddev);

    return 0;
}
```

实际驱动应当优先结合 PCI Core 和 PM Core 所提供的辅助接口，避免重复执行已经由总线层完成的电源状态切换。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/38ba04c239481667fec6afcb63b8d1b1_MD5.jpg]]

### 7.4 PCI Hotplug 机制

PCIe 支持热插拔的场景包括：服务器 PCIe 热插槽；Thunderbolt；ExpressCard；虚拟机动态添加设备；SR-IOV VF 动态创建；用户通过 sysfs 触发移除和重新扫描。设备被热拔出以后，驱动不能继续访问 BAR，也不能继续等待设备状态寄存器变化。因此驱动需要：为设备断开做好准备；避免无限轮询寄存器；及时停止 DMA；正确处理全部为 `0xffffffff` 的 MMIO 读取结果；在 `remove()` 中取消工作队列和定时器；防止用户接口继续提交请求。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/fbce6f8c76708a1481ea78be564f0f99_MD5.jpg]]

### 7.5 remove 流程与资源释放

一个典型的 `remove()` 可以按照下面的顺序执行：

```
/* remove() 必须先停止新请求和 DMA，再释放 IRQ、DMA、BAR 等底层资源。 */
static void demo_remove(struct pci_dev *pdev)
{
    struct demo_device *ddev = pci_get_drvdata(pdev);

    /* 1. 阻止新的上层请求 */
    ddev->running = false;
    demo_unregister_interface(ddev);

    /* 2. 停止硬件和 DMA */
    demo_stop_device(ddev);
    // 关闭 Bus Master，阻止设备继续主动 DMA
    pci_clear_master(pdev);

    /* 3. 同步并关闭中断 */
    demo_mask_interrupts(ddev);
    synchronize_irq(ddev->irq);
    // 注销 IRQ Handler；必须使用与注册时相同的 dev_id
    free_irq(ddev->irq, ddev);
    // 释放 pci_alloc_irq_vectors() 申请的所有向量
    pci_free_irq_vectors(pdev);

    /* 4. 释放 DMA */
    dma_free_coherent(&pdev->dev,
                      ddev->dma_size,
                      ddev->dma_virt,
                      ddev->dma_handle);

    /* 5. 解除 BAR 映射 */
    // 解除当前设备 BAR0 的 MMIO 映射
    pci_iounmap(pdev, ddev->bar0);

    /* 6. 释放 PCI 资源 */
    // 释放此前申请的 BAR 资源
    pci_release_regions(pdev);
    // 关闭设备的 PCI 访问能力
    pci_disable_device(pdev);

    // 最后释放设备私有结构体
    kfree(ddev);
}
```

必须先停止设备，再释放设备仍然可能访问的 DMA 内存。否则设备可能继续向已经被分配给其他对象的内存进行 DMA，引发数据破坏。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/9f4f98a0b66981626c504b2a88a59227_MD5.jpg]]

### 7.6 AER 错误上报机制

AER 的全称是 Advanced Error Reporting。PCIe AER 能够报告更加详细的链路和事务错误，例如：Correctable Error；Non-Fatal Error；Fatal Error；Unsupported Request；Completion Timeout；Malformed TLP；Poisoned TLP。驱动可以定义：

```
/* 注册 PCIe AER 错误恢复各阶段的回调函数。 */
static const struct pci_error_handlers demo_err_handlers = {
    .error_detected = demo_error_detected,
    .mmio_enabled   = demo_mmio_enabled,
    .slot_reset     = demo_slot_reset,
    .resume         = demo_error_resume,
};
```

然后设置：

```
/* 把 AER 错误处理表挂到 pci_driver。 */
.err_handler = &demo_err_handlers,
```

AER 恢复大致按照下面的流程进行：

```
发现 PCIe 错误
      ↓
error_detected()
      ↓
停止新 I/O
      ↓
必要时恢复 MMIO
      ↓
mmio_enabled()
      ↓
必要时进行 Slot Reset
      ↓
slot_reset()
      ↓
重新初始化设备
      ↓
resume()
```

如果错误不可恢复，驱动应当返回相应状态，并把设备视为永久失效。Linux PCI Error Recovery 文档明确规定了 `error_detected()`、`mmio_enabled()`、`slot_reset()` 和 `resume()` 等阶段，并要求驱动在错误通知后停止新的设备 I/O。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/5ebe704c35d29f0bb6640b86e32d6ce0_MD5.jpg]]

### 7.7 设备 reset 与错误恢复

PCIe 设备可能支持多种 Reset：Function Level Reset；Power Management Reset；Secondary Bus Reset；Slot Power Cycle；设备自定义软复位。Linux 提供：

```
/* 请求 PCI Core 对当前 Function 执行支持的复位流程。 */
// 请求对当前 PCI Function 进行功能级/可用方式的复位
pci_reset_function(pdev);
```

尝试对 Function 进行复位。复位以前必须：1）停止上层请求；2）停止 DMA；3）屏蔽中断；4）保存必要的软件状态；5）确认复位不会影响共享资源。复位以后通常需要：1）恢复配置空间；2）重新设置 DMA Mask；3）重新设置 Bus Master；4）重新配置设备寄存器；5）重建队列；6）重新打开中断；7）恢复业务。对于多功能设备、PF/VF 或共享硬件模块，一个 Function 的复位可能影响其他 Function，因此不能在不了解设备复位边界的情况下随意执行。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/4094480f25b9589e044a99d772bdabf0_MD5.jpg]]

## 八、PCIe 高级特性

### 8.1 PCIe Link、Lane 与速率

PCIe Link 由一个或者多个 Lane 组成。每一条 Lane 都可以同时发送和接收。常见速率如下：

| **代际原始传输率主要编码** | <br> | <br> |
| :-- | :-- | :-- |
| PCIe 1.x | 
## 2.5 GT/s

 | 8b/10b |
| PCIe 2.x | 

## 5.0 GT/s

 | 8b/10b |
| PCIe 3.0 | 

## 8.0 GT/s

 | 128b/130b |
| PCIe 4.0 | 

## 16.0 GT/s

 | 128b/130b |
| PCIe 5.0 | 

## 32.0 GT/s

 | 128b/130b |
| PCIe 6.0 | 

## 64.0 GT/s

 | PAM4、Flit、FEC |
| PCIe 7.0 | 128.0 GT/s | PAM4、Flit |

PCIe 6.0 引入 PAM4、固定大小 Flit 和 FEC，以达到每 Lane 64 GT/s；PCIe 7.0 又进一步提升到 128 GT/s。可以查看链路状态：

```
# 查看指定 PCI 设备的详细能力和当前链路状态。
lspci -s 03:00.0 -vv
```

输出中可能出现：

```
LnkCap: Speed 16GT/s, Width x8
LnkSta: Speed 8GT/s, Width x4
```

这说明设备能力是 16 GT/s、x8，但是当前实际只训练到 8 GT/s、x4。这可能由以下原因造成：插槽只提供较少 Lane；信号质量问题；BIOS 配置；转接卡；Lane 分叉配置；链路降速；硬件故障。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/b9ddee70d511bf68634840f4ff217616_MD5.jpg]]

### 8.2 Max Payload Size 与 Max Read Request Size

MPS 是 Max Payload Size。它限制一个 PCIe TLP 中能够携带的最大有效载荷。常见值包括：

```
128B、256B、512B、1024B、2048B、4096B
```

更大的 MPS 可以降低每个数据字节所承担的 TLP Header 开销，但也可能增加单个事务占用链路以及缓冲资源的时间。

MRRS 是 Max Read Request Size。它限制设备一次 Memory Read Request 可以请求多大的数据。增大 MRRS 有可能提高大块 DMA 读取效率，但是实际效果受到 Root Complex、Switch、设备内部队列以及系统拓扑影响。不能仅凭“数值越大，性能越高”来盲目设置。整个路径上的组件必须能够正确支持相应配置。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/c3a4053d2a1d58dbf354c1d727bc9598_MD5.jpg]]

### 8.3 SR-IOV：PF 与 VF

SR-IOV 的全称是 Single Root I/O Virtualization。一个支持 SR-IOV 的设备可以提供：PF：Physical Function；VF：Virtual Function。PF 拥有完整的管理能力，可以创建和配置 VF。VF 是轻量级 PCI Function，可以分配给虚拟机或者容器工作负载。驱动可以调用：

```
/* 由 PF 驱动创建指定数量的 SR-IOV VF。 */
// 由 PF 创建指定数量的 Virtual Function
pci_enable_sriov(pdev, num_vfs);
```

启用 VF，也可以通过 sysfs：

```
# 通过 sysfs 创建 8 个 SR-IOV VF。
echo 8 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs
```

关闭：

```
# 通过 sysfs 关闭当前 PF 创建的全部 VF。
echo 0 > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs
```

从 Linux PCI Core 的角度来看，VF 会像动态出现的 PCI 设备一样被枚举、匹配和绑定驱动。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/4161f3b92eadfa21325ee710ce45223a_MD5.jpg]]

### 8.4 VFIO 与设备直通

VFIO 是一种把设备安全暴露给用户空间的框架。它常用于：KVM 虚拟机 PCIe 直通；DPDK；用户态存储驱动；用户态加速器驱动。VFIO 会结合 IOMMU Group 管理设备隔离。一个 IOMMU Group 中的设备是否能够被独立直通，取决于硬件拓扑、ACS 和 IOMMU 隔离边界。典型流程包括：

```
设备从原生驱动解绑
        ↓
绑定 vfio-pci
        ↓
VFIO 检查 IOMMU Group
        ↓
用户空间映射 BAR
        ↓
用户空间配置 DMA 地址空间
        ↓
注册中断 Eventfd
        ↓
用户空间直接控制设备
```

VFIO 的目标是在 IOMMU 保护下，让用户空间获得设备 MMIO、中断和 DMA 能力，同时限制设备只能访问被授权的内存。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/a33f7647ee0409db9e53db13df3564c5_MD5.jpg]]

### 8.5 ACS、ARI、ATS、PRI、PASID 简介

ACS 用来控制 PCIe 事务的路由和隔离。它对 IOMMU Group 划分、设备直通以及 P2P 路由非常重要。

ARI 扩展 Function 编号能力，能够支持一个 Device 下更多 Function，常用于 SR-IOV。

ATS 允许设备缓存地址转换结果。设备可以减少每次 DMA 都访问 IOMMU 转换结构所带来的开销。

PRI 允许设备在访问尚未建立映射的地址时，向系统发起页面请求。

PASID 可以给设备发起的事务附带进程地址空间标识。它使多个进程或者多个地址空间能够共享同一个设备，同时由 IOMMU 区分具体上下文。PASID、ATS 和 PRI 是共享虚拟地址、加速器进程隔离以及现代异构计算的重要基础。PCI-SIG 对 PASID 的定义要求设备、Completer 以及路径中的路由组件共同具备相应能力。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/70d957ff494a5b3d41fe73dbd34cebd1_MD5.jpg]]

### 8.6 Peer-to-Peer DMA 简介

普通 DMA 路径通常是：

```
设备 A → 系统内存 → 设备 B
```

P2P DMA 希望实现：

```
设备 A → 设备 B 的 BAR 或设备内存
```

例如：NVMe 直接向 GPU 内存传输；网卡直接向加速卡传输；FPGA 之间直接交换数据。P2P DMA 可以减少系统内存中转和 CPU 参与。但是它受到 PCIe 拓扑限制。某些 Root Complex 不支持把一个下游设备的事务路由到另一个下游设备，IOMMU 和 ACS 也可能影响路由。Linux 提供 `pci_p2pdma` 相关接口，驱动不能简单把另一个设备的 BAR 地址直接交给 DMA API。Linux 文档指出，P2P DMA 是否可行与设备是否位于可路由的 PCI 拓扑中密切相关。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/760a3d7974db77cdce8ed01dbfc0637e_MD5.jpg]]

## 九、调试、测试与性能优化

### 9.1 `lspci` 常用命令

查看全部 PCI 设备：

```
# 列出系统中的 PCI 设备。
lspci
```

显示数字 ID：

```
# 显示 PCI 数字 Vendor/Device ID。
lspci -nn
```

显示驱动绑定情况：

```
# 同时显示数字 ID 和当前绑定的内核驱动。
lspci -nnk
```

查看详细能力：

```
# 显示更详细的 PCI/PCIe 能力信息。
lspci -vv
```

查看指定设备：

```
# 只查看指定 BDF 设备的详细信息。
lspci -s 03:00.0 -vv
```

查看配置空间：

```
# 读取传统 256 字节 PCI 配置空间。
lspci -s 03:00.0 -xxx
```

查看 PCIe 扩展配置空间：

```
# 读取 PCIe 扩展配置空间；需要足够权限。
sudo lspci -s 03:00.0 -xxxx
```

读取完整配置空间可能会触发某些异常设备的硬件问题，生产环境中应当谨慎使用。`lspci` 和 `setpci` 属于 pciutils 工具集。

### 9.2 `setpci` 读写配置空间

读取 Vendor ID：

```
# 读取配置空间 0x00 偏移处的 16 位 Vendor ID。
setpci -s 03:00.0 00.w
```

读取 Command 寄存器：

```
# 读取 PCI Command 寄存器。
setpci -s 03:00.0 COMMAND
```

读取 Status：

```
# 读取 PCI Status 寄存器。
setpci -s 03:00.0 STATUS
```

写配置空间必须非常谨慎。例如错误修改 Command、BAR、Bridge Control 或 Link Control，可能导致：设备从系统中消失；DMA 被关闭；链路被复位；总线下游全部设备掉线；系统崩溃；数据损坏。因此 `setpci` 更适合进行受控实验和配置空间读取，不应在不了解寄存器语义的情况下直接写入。

### 9.3 `/sys/bus/pci/devices/` 目录分析

每一个 PCI Function 通常对应一个目录：

```
/sys/bus/pci/devices/0000:03:00.0/
```

常见文件包括：

```
vendor
device
subsystem_vendor
subsystem_device
class
revision
resource
config
irq
numa_node
local_cpulist
enable
remove
reset
driver
iommu_group
sriov_numvfs
sriov_totalvfs
```

查看 BAR：

```
# 查看设备所有 BAR 的资源范围和标志。
cat /sys/bus/pci/devices/0000:03:00.0/resource
```

查看 NUMA：

```
# 查看 PCI 设备所在的 NUMA 节点。
cat /sys/bus/pci/devices/0000:03:00.0/numa_node
```

解绑驱动：

```
# 把指定 BDF 设备从 demo_pci 驱动解绑。
echo 0000:03:00.0 \
  > /sys/bus/pci/drivers/demo_pci/unbind
```

重新绑定：

```
# 把指定 BDF 设备重新绑定到 demo_pci 驱动。
echo 0000:03:00.0 \
  > /sys/bus/pci/drivers/demo_pci/bind
```

移除设备：

```
# 从内核 PCI 设备模型中移除该设备。
echo 1 \
  > /sys/bus/pci/devices/0000:03:00.0/remove
```

重新扫描：

```
# 触发 PCI 总线重新扫描，重新发现可见设备。
echo 1 > /sys/bus/pci/rescan
```

对于支持复位的设备，sysfs 中还可能存在 `reset` 文件。Linux 的 PCI sysfs ABI 提供 `remove`、`reset`、`resource` 和 SR-IOV 等控制入口。

### 9.4 `dmesg` 与 `dev_err/dev_info`

PCI 驱动应当使用设备日志接口：

```
/* 优先使用 dev_*() 日志接口，让日志自动带上设备上下文。 */
dev_info(&pdev->dev,
         "BAR0 start=%pa len=%pa\n",
         &start, &len);

dev_err(&pdev->dev,
        "failed to allocate DMA buffer\n");

dev_warn(&pdev->dev,
         "link width is lower than expected\n");

dev_dbg(&pdev->dev,
        "interrupt status=0x%08x\n",
        status);
```

相比直接使用：

```
/* printk() 可以输出内核日志，但设备驱动中通常更推荐 dev_*()。 */
printk()
```

`dev_*()` 会自动包含设备名称和 BDF，方便在多设备环境中定位问题。查看日志：

```
# 从内核日志中筛选 PCI 相关信息。
dmesg -T | grep -i pci
```

持续观察：

```
# 持续跟踪新产生的内核日志。
dmesg -w
```

### 9.5 dynamic debug

如果驱动使用 `dev_dbg()` 或 `pr_debug()`，可以通过 Dynamic Debug 动态开启日志：

```
# 动态开启 demo_pci 模块中的 dev_dbg()/pr_debug() 输出。
echo 'module demo_pci +p' \
  > /sys/kernel/debug/dynamic_debug/control
```

按照文件开启：

```
# 只开启 demo_pci.c 文件中的 Dynamic Debug 输出。
echo 'file demo_pci.c +p' \
  > /sys/kernel/debug/dynamic_debug/control
```

关闭：

```
# 关闭 demo_pci 模块的 Dynamic Debug 输出。
echo 'module demo_pci -p' \
  > /sys/kernel/debug/dynamic_debug/control
```

这样不需要重新编译驱动，就可以在运行过程中打开调试信息。

### 9.6 ftrace 跟踪 probe 与 IRQ

查看可用函数：

```
# 确认 pci_device_probe 是否可被 ftrace 跟踪。
grep pci_device_probe \
  /sys/kernel/debug/tracing/available_filter_functions
```

跟踪 PCI Probe：

```
# 启用 function_graph，并只重点跟踪 pci_device_probe 调用链。
echo function_graph \
  > /sys/kernel/debug/tracing/current_tracer

echo pci_device_probe \
  > /sys/kernel/debug/tracing/set_graph_function

echo 1 \
  > /sys/kernel/debug/tracing/tracing_on
```

读取结果：

```
# 读取 ftrace 已采集的跟踪结果。
cat /sys/kernel/debug/tracing/trace
```

跟踪 IRQ 事件：

```
# 打开 IRQ Handler 进入/退出两个 tracepoint。
echo 1 \
  > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable

echo 1 \
  > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
```

还可以使用：

```
# 用 trace-cmd 记录 IRQ Handler 的进入和退出事件。
trace-cmd record \
  -e irq:irq_handler_entry \
  -e irq:irq_handler_exit
```

观察中断处理时间和频率。

### 9.7 perf 分析中断与 DMA 性能

观察热点：

```
# 实时观察全系统 CPU 热点。
perf top -a
```

记录系统热点：

```
# 采集 30 秒全系统调用栈，再进入 perf report 分析。
perf record -a -g -- sleep 30
perf report
```

记录 IRQ Tracepoint：

```
# 使用 perf 记录 IRQ 进入/退出事件，分析中断频率和处理开销。
perf record \
  -a \
  -e irq:irq_handler_entry \
  -e irq:irq_handler_exit \
  -- sleep 10
```

如果中断处理函数占用大量 CPU，需要判断：中断频率是不是过高；中断合并是不是没有启用；每次中断处理的数据量是不是太少；是否应当使用 NAPI 或轮询；MSI-X 向量是不是分配不足；中断是否全部集中在单个 CPU。

### 9.8 PCIe 带宽、DMA 效率与中断合并

PCIe 理论带宽不等于应用层有效带宽。实际效率受到以下因素影响：PCIe 代际；Link Width；编码开销；TLP Header；DLLP；LCRC；MPS；MRRS；DMA 请求大小；Scatter-Gather 数量；设备内部队列深度；中断频率；CPU 处理速度；NUMA；IOMMU；内存带宽。大量小 DMA 请求会产生较高的描述符和 TLP Header 开销。中断合并可以让设备在积累一定数量的完成事件或者经过一定时间后，再产生一次中断。优点是降低 CPU 中断开销，缺点是增加单次请求完成延迟。因此需要在吞吐量和时延之间进行权衡。

### 9.9 NUMA 亲和性与 IRQ 绑定

在多路服务器中，PCIe Root Complex 通常属于某一个 NUMA Node。查看设备 NUMA 节点：

```
# 查看 PCI 设备所在 NUMA 节点。
cat /sys/bus/pci/devices/0000:03:00.0/numa_node
```

查看设备邻近 CPU：

```
# 查看与该 PCI 设备距离较近的 CPU 列表。
cat /sys/bus/pci/devices/0000:03:00.0/local_cpulist
```

理想情况下：中断运行在设备所在 NUMA 节点的 CPU；应用程序运行在相同节点；DMA 缓冲区从相同节点分配；队列和 CPU 建立稳定映射。如果设备位于 NUMA Node 0，但是应用程序和中断长期运行在 Node 1，那么 DMA 数据可能需要跨 Socket 互连传输，增加时延并占用互连带宽。

## 十、实战案例

### 10.1 最小 PCI 驱动模板

下面给出一个最小 PCI 驱动：

```
/* 最小 PCI 驱动：完成 ID 匹配、设备使能、BAR 资源申请和 Bus Master 配置。 */
// 内核模块基础宏和模块元数据接口
#include <linux/module.h>
// Linux PCI Core 的数据结构和 API
#include <linux/pci.h>

#define DEMO_VENDOR_ID 0x1234
#define DEMO_DEVICE_ID 0x5678

static int demo_probe(struct pci_dev *pdev,
                      const struct pci_device_id *id)
{
    int ret;

    dev_info(&pdev->dev, "PCI device matched\n");

    // 使能 PCI Function，允许内核访问设备的 I/O/MMIO 资源
    ret = pci_enable_device(pdev);
    if (ret) {
        dev_err(&pdev->dev,
                "pci_enable_device failed: %d\n",
                ret);
        return ret;
    }

    // 声明驱动独占设备 BAR 资源
    ret = pci_request_regions(pdev, "demo_pci");
    if (ret) {
        dev_err(&pdev->dev,
                "pci_request_regions failed: %d\n",
                ret);
        // 关闭设备的 PCI 访问能力
        pci_disable_device(pdev);
        return ret;
    }

    // 打开 Bus Master Enable，允许设备主动发起 DMA
    pci_set_master(pdev);

    dev_info(&pdev->dev,
             "device initialized successfully\n");

    return 0;
}

static void demo_remove(struct pci_dev *pdev)
{
    // 关闭 Bus Master，阻止设备继续主动 DMA
    pci_clear_master(pdev);
    // 释放此前申请的 BAR 资源
    pci_release_regions(pdev);
    // 关闭设备的 PCI 访问能力
    pci_disable_device(pdev);

    dev_info(&pdev->dev, "device removed\n");
}

static const struct pci_device_id demo_ids[] = {
    { PCI_DEVICE(DEMO_VENDOR_ID, DEMO_DEVICE_ID) },
    { }
};

// 导出 PCI 模块别名，方便用户空间根据 modalias 自动加载模块
MODULE_DEVICE_TABLE(pci, demo_ids);

static struct pci_driver demo_driver = {
    .name     = "demo_pci",
    .id_table = demo_ids,
    .probe    = demo_probe,
    .remove   = demo_remove,
};

// 自动生成模块 init/exit，分别注册和注销 PCI 驱动
module_pci_driver(demo_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Linux Driver Developer");
MODULE_DESCRIPTION("Minimal PCI Driver Example");
```

### 10.2 BAR 映射与寄存器读写示例

增加私有结构体：

```
/* 私有结构体保存当前 PCI 设备和 BAR0 的 MMIO 映射地址。 */
struct demo_device {
    struct pci_dev *pdev;
    void __iomem *bar0;
};
```

在 `probe()` 中：

```
/* 在 probe() 中分配私有数据、映射 BAR0，并把私有数据挂到 pci_dev。 */
struct demo_device *ddev;

// 分配并清零设备私有结构体
ddev = kzalloc(sizeof(*ddev), GFP_KERNEL);
if (!ddev) {
    ret = -ENOMEM;
    goto err_regions;
}

ddev->pdev = pdev;

// 映射 BAR0 到内核虚拟地址
ddev->bar0 = pci_iomap(pdev, 0, 0);
if (!ddev->bar0) {
    ret = -ENOMEM;
    goto err_free;
}

// 把私有结构体保存到 pci_dev，供 remove()/PM/IRQ 等路径取回
pci_set_drvdata(pdev, ddev);
```

寄存器访问：

```
/* 通过寄存器偏移访问设备 ID、控制寄存器和状态寄存器。 */
#define DEMO_REG_ID       0x0000
#define DEMO_REG_CONTROL  0x0004
#define DEMO_REG_STATUS   0x0008

u32 id;
u32 status;

id = readl(ddev->bar0 + DEMO_REG_ID);

writel(DEMO_CTRL_ENABLE,
       ddev->bar0 + DEMO_REG_CONTROL);

// 读取设备状态寄存器，确认中断或硬件状态
status = readl(ddev->bar0 + DEMO_REG_STATUS);
```

### 10.3 DMA 缓冲区申请与数据传输示例

```
/* 完整演示 DMA Mask 协商、Coherent DMA 分配、地址下发以及启动 DMA。 */
#define DEMO_DMA_SIZE 4096

// 声明设备和平台可接受的 DMA 地址位宽
ret = dma_set_mask_and_coherent(&pdev->dev,
                                DMA_BIT_MASK(64));
if (ret) {
    // 声明设备和平台可接受的 DMA 地址位宽
    ret = dma_set_mask_and_coherent(&pdev->dev,
                                    DMA_BIT_MASK(32));
    if (ret)
        goto err_iounmap;
}

// 申请 Coherent DMA：同时得到 CPU 虚拟地址和设备 DMA 地址
ddev->dma_virt = dma_alloc_coherent(
    &pdev->dev,
    DEMO_DMA_SIZE,
    &ddev->dma_handle,
    GFP_KERNEL);

if (!ddev->dma_virt) {
    ret = -ENOMEM;
    goto err_iounmap;
}

memset(ddev->dma_virt, 0, DEMO_DMA_SIZE);

// 写入 DMA 地址低 32 位
writel(lower_32_bits(ddev->dma_handle),
       ddev->bar0 + DEMO_REG_DMA_LOW);

// 写入 DMA 地址高 32 位
writel(upper_32_bits(ddev->dma_handle),
       ddev->bar0 + DEMO_REG_DMA_HIGH);

writel(DEMO_DMA_SIZE,
       ddev->bar0 + DEMO_REG_DMA_LEN);

// DMA 写屏障：保证前面的描述符内容先对设备可见
dma_wmb();

writel(DEMO_DMA_START,
       ddev->bar0 + DEMO_REG_DMA_CTRL);
```

释放时：

```
/* 驱动退出时释放 Coherent DMA 缓冲区。 */
dma_free_coherent(&pdev->dev,
                  DEMO_DMA_SIZE,
                  ddev->dma_virt,
                  ddev->dma_handle);
```

### 10.4 MSI/MSI-X 中断注册示例

```
/* 中断处理函数读取状态、确认来源、清中断，并通知等待 DMA 完成的线程。 */
static irqreturn_t demo_irq(int irq, void *data)
{
    struct demo_device *ddev = data;
    u32 status;

    // 读取设备状态寄存器，确认中断或硬件状态
    status = readl(ddev->bar0 + DEMO_REG_INT_STATUS);

    if (!status)
        return IRQ_NONE;

    // 按设备定义写回状态位，通常用于清除已处理的中断
    writel(status,
           ddev->bar0 + DEMO_REG_INT_STATUS);

    complete(&ddev->dma_done);

    return IRQ_HANDLED;
}
```

申请向量：

```
/* 为设备申请最多 4 个 MSI-X/MSI 向量，并注册第 0 个向量的处理函数。 */
// 向 PCI Core 申请可用的中断向量
ret = pci_alloc_irq_vectors(pdev,
                            1,
                            4,
                            PCI_IRQ_MSIX |
                            PCI_IRQ_MSI);
if (ret < 0)
    goto err_dma;

ddev->nr_vectors = ret;
// 把第 0 个 PCI 中断向量转换为 Linux IRQ 号
ddev->irq = pci_irq_vector(pdev, 0);

// 向 Linux IRQ 子系统注册中断处理函数
ret = request_irq(ddev->irq,
                  demo_irq,
                  0,
                  "demo_pci",
                  ddev);
if (ret)
    goto err_vectors;
```

释放：

```
/* 先释放 IRQ Handler，再释放中断向量。 */
// 注销 IRQ Handler；必须使用与注册时相同的 dev_id
free_irq(ddev->irq, ddev);
// 释放 pci_alloc_irq_vectors() 申请的所有向量
pci_free_irq_vectors(pdev);
```

### 10.5 自定义 PCIe 设备驱动完整流程

一个自定义 FPGA 或 PCIe Endpoint 驱动的完整流程可以概括成为：

```
1. PCI Core 枚举 Endpoint
2. 读取 Vendor ID 和 Device ID
3. 匹配 pci_device_id
4. 调用 probe()
5. pci_enable_device()
6. dma_set_mask_and_coherent()
7. pci_request_regions()
8. pci_set_master()
9. pci_iomap()
10. 分配设备私有结构体
11. 读取版本和能力寄存器
12. 复位设备
13. 分配 DMA 描述符和数据缓冲区
14. 配置 DMA Queue
15. pci_alloc_irq_vectors()
16. request_irq()
17. 打开设备中断
18. 注册字符设备或其他上层接口
19. 接收用户请求
20. 填写 DMA 描述符
21. dma_wmb()
22. 写 Doorbell
23. 设备执行 DMA
24. 设备触发 MSI-X
25. 驱动处理中断和完成队列
26. 返回结果给用户空间
```

### 10.6 网卡驱动框架拆解

PCIe 网卡驱动的 `probe()` 通常包含：

```
使能 PCI 设备
    ↓
设置 DMA Mask
    ↓
申请和映射 BAR
    ↓
分配 struct net_device
    ↓
创建 TX/RX Ring
    ↓
配置 MSI-X
    ↓
初始化 NAPI
    ↓
设置 net_device_ops
    ↓
register_netdev()
```

发送路径大致为：

```
应用程序发送数据
    ↓
协议栈生成 skb
    ↓
ndo_start_xmit()
    ↓
dma_map_single / dma_map_page
    ↓
填写 TX Descriptor
    ↓
dma_wmb()
    ↓
写 TX Doorbell
    ↓
网卡读取数据并发送
    ↓
TX Completion
    ↓
解除 DMA 映射
    ↓
释放 skb
```

接收路径大致为：

```
驱动预先准备 RX Buffer
    ↓
网卡 DMA 写入数据
    ↓
网卡触发 MSI-X
    ↓
中断屏蔽队列中断
    ↓
调度 NAPI
    ↓
轮询 RX Completion
    ↓
构造 skb
    ↓
交给网络协议栈
    ↓
补充新的 RX Buffer
    ↓
重新打开中断
```

### 10.7 NVMe 驱动关键流程

NVMe PCI 驱动的核心初始化大致包括：

```
PCI 设备匹配
    ↓
使能设备和 DMA
    ↓
映射 NVMe Controller BAR
    ↓
读取 CAP、VS 等寄存器
    ↓
禁用控制器
    ↓
创建 Admin Submission Queue
    ↓
创建 Admin Completion Queue
    ↓
配置 AQA、ASQ、ACQ
    ↓
启用控制器
    ↓
Identify Controller
    ↓
创建 I/O Completion Queue
    ↓
创建 I/O Submission Queue
    ↓
配置 MSI-X
    ↓
注册 blk-mq
    ↓
扫描 Namespace
    ↓
创建块设备
```

NVMe 为每个 CPU 或硬件队列建立独立的 Submission Queue 和 Completion Queue，并结合 MSI-X 与 blk-mq 降低锁竞争，提高并发 I/O 能力。

### 10.8 常见问题排查清单

设备没有出现在 `lspci` 时，检查：设备是否上电；PCIe Reset 是否释放；Reference Clock 是否存在；LTSSM 是否进入 L0；BIOS 是否关闭插槽；Lane 分叉是否正确；Root Port 是否存在；链路是否因为信号问题无法训练。

`lspci` 能看到但驱动不 Probe 时，检查：Vendor ID 和 Device ID；`pci_device_id` 是否正确；ID Table 是否以全零成员结束；驱动模块是否加载；设备是否已经绑定其他驱动；`driver_override` 是否被设置；`dmesg` 是否存在 Probe 错误。

`pci_request_regions()` 失败时，检查：BAR 是否已经被其他驱动占用；设备是否仍然绑定旧驱动；BAR 资源是否存在冲突；固件是否正确分配地址。

MMIO 读取全部为 `0xffffffff` 时，检查：设备是否掉线；链路是否断开；设备是否进入 D3cold；BAR 是否正确映射；是否发生 AER Fatal Error；设备复位是否尚未完成。

DMA 没有数据时，检查：是否调用 `pci_set_master()`；DMA Mask 是否正确；是否使用 DMA API 返回的地址；地址高低位是否写反；描述符是否执行 `dma_wmb()`；DMA 方向是否正确；IOMMU 是否报告 Fault；设备队列是否真正启动。

中断没有触发时，检查：`pci_alloc_irq_vectors()` 返回值；`pci_irq_vector()`；`request_irq()`；设备内部中断 Mask；MSI/MSI-X Enable；中断状态位；`/proc/interrupts`；中断是否绑定到了离线 CPU。

---

# 附录 A：PCI 驱动常用 API 速查表

| **API作用** | <br> |
| :-- | :-- |
| `pci_register_driver()` | 注册 PCI 驱动 |
| `pci_unregister_driver()` | 注销 PCI 驱动 |
| `pci_enable_device()` | 使能 PCI 设备 |
| `pci_disable_device()` | 禁用 PCI 设备 |
| `pci_set_master()` | 打开 Bus Master |
| `pci_clear_master()` | 关闭 Bus Master |
| `pci_request_regions()` | 申请全部常用 BAR 资源 |
| `pci_release_regions()` | 释放 BAR 资源 |
| `pci_resource_start()` | 获取 BAR 起始地址 |
| `pci_resource_len()` | 获取 BAR 长度 |
| `pci_resource_flags()` | 获取 BAR 类型标志 |
| `pci_iomap()` | 映射 PCI BAR |
| `pci_iounmap()` | 解除 PCI BAR 映射 |
| `pci_set_drvdata()` | 保存设备私有数据 |
| `pci_get_drvdata()` | 获取设备私有数据 |
| `pci_alloc_irq_vectors()` | 分配 INTx/MSI/MSI-X |
| `pci_irq_vector()` | 获取 Linux IRQ |
| `pci_free_irq_vectors()` | 释放中断向量 |
| `pci_save_state()` | 保存配置空间状态 |
| `pci_restore_state()` | 恢复配置空间状态 |
| `pci_reset_function()` | 复位 PCI Function |
| `pci_enable_sriov()` | 启用 SR-IOV VF |
| `pci_disable_sriov()` | 关闭 SR-IOV VF |

---

# 附录 B：PCI 配置空间标准寄存器图

```
0x00  ┌──────────────────────────────┐
      │ Device ID      │ Vendor ID  │
0x04  ├──────────────────────────────┤
      │ Status         │ Command    │
0x08  ├──────────────────────────────┤
      │ Class Code / Revision ID     │
0x0C  ├──────────────────────────────┤
      │ Header / Latency / Cache     │
0x10  ├──────────────────────────────┤
      │ BAR0                         │
0x14  ├──────────────────────────────┤
      │ BAR1                         │
0x18  ├──────────────────────────────┤
      │ BAR2                         │
0x1C  ├──────────────────────────────┤
      │ BAR3                         │
0x20  ├──────────────────────────────┤
      │ BAR4                         │
0x24  ├──────────────────────────────┤
      │ BAR5                         │
0x28  ├──────────────────────────────┤
      │ CardBus CIS Pointer          │
0x2C  ├──────────────────────────────┤
      │ Subsystem Device/Vendor      │
0x30  ├──────────────────────────────┤
      │ Expansion ROM BAR            │
0x34  ├──────────────────────────────┤
      │ Capability Pointer           │
0x3C  ├──────────────────────────────┤
      │ Interrupt Pin / Line         │
0x40  ├──────────────────────────────┤
      │ PCI Capability Structures    │
      │                              │
0x100 ├──────────────────────────────┤
      │ PCIe Extended Capabilities   │
      │ AER / SR-IOV / ATS / PASID   │
0xFFF └──────────────────────────────┘
```

---

# 附录 C：BAR、DMA、中断核心概念对照表

| **概念本质驱动主要接口** | <br> | <br> |
| :-- | :-- | :-- |
| BAR | 设备暴露的地址资源 | `pci_resource_*()` |
| MMIO | CPU 访问设备寄存器 | `readl()`、`writel()` |
| DMA | 设备直接访问内存 | `dma_map_*()` |
| Coherent DMA | 持续共享控制内存 | `dma_alloc_coherent()` |
| Streaming DMA | 阶段性数据传输 | `dma_map_single()` |
| INTx | 传统共享中断 | `request_irq(...IRQF_SHARED)` |
| MSI | 消息中断 | `pci_alloc_irq_vectors()` |
| MSI-X | 多向量消息中断 | `pci_alloc_irq_vectors()` |
| IOMMU | DMA 地址转换和隔离 | DMA API / VFIO |

---

# 附录 D：INTx、MSI、MSI-X 对比表

| **特性INTxMSIMSI-X** | <br> | <br> | <br> |
| :-- | :-- | :-- | :-- |
| 触发方式 | 物理中断线 | Memory Write | Memory Write |
| 是否共享 | 可以共享 | 通常不共享 | 每向量独立 |
| 向量数量 | 通常较少 | 多个，但受能力限制 | 支持更多 |
| 独立 Mask | 受实现限制 | 支持情况有限 | 每向量独立 |
| 多队列适配 | 较差 | 一般 | 最好 |
| 推荐场景 | 兼容退化 | 普通设备 | 网卡、NVMe、加速器 |

---

# 附录 E：probe/remove 标准代码模板

```
/* 标准 probe/remove 模板：重点观察初始化顺序和错误路径的逆序回滚。 */
static int demo_probe(struct pci_dev *pdev,
                      const struct pci_device_id *id)
{
    struct demo_device *ddev;
    int ret;

    // 使能 PCI Function，允许内核访问设备的 I/O/MMIO 资源
    ret = pci_enable_device(pdev);
    if (ret)
        return ret;

    // 声明设备和平台可接受的 DMA 地址位宽
    ret = dma_set_mask_and_coherent(&pdev->dev,
                                    DMA_BIT_MASK(64));
    if (ret) {
        // 声明设备和平台可接受的 DMA 地址位宽
        ret = dma_set_mask_and_coherent(&pdev->dev,
                                        DMA_BIT_MASK(32));
        if (ret)
            goto err_disable;
    }

    // 声明驱动独占设备 BAR 资源
    ret = pci_request_regions(pdev, "demo_pci");
    if (ret)
        goto err_disable;

    // 打开 Bus Master Enable，允许设备主动发起 DMA
    pci_set_master(pdev);

    // 分配并清零设备私有结构体
    ddev = kzalloc(sizeof(*ddev), GFP_KERNEL);
    if (!ddev) {
        ret = -ENOMEM;
        goto err_regions;
    }

    ddev->pdev = pdev;
    // 把私有结构体保存到 pci_dev，供 remove()/PM/IRQ 等路径取回
    pci_set_drvdata(pdev, ddev);

    // 映射 BAR0 到内核虚拟地址
    ddev->bar0 = pci_iomap(pdev, 0, 0);
    if (!ddev->bar0) {
        ret = -ENOMEM;
        goto err_free;
    }

    ret = demo_dma_init(ddev);
    if (ret)
        goto err_iounmap;

    ret = demo_irq_init(ddev);
    if (ret)
        goto err_dma;

    ret = demo_hw_init(ddev);
    if (ret)
        goto err_irq;

    ret = demo_register_interface(ddev);
    if (ret)
        goto err_hw;

    return 0;

err_hw:
    demo_hw_stop(ddev);
err_irq:
    demo_irq_exit(ddev);
err_dma:
    demo_dma_exit(ddev);
err_iounmap:
    // 解除当前设备 BAR0 的 MMIO 映射
    pci_iounmap(pdev, ddev->bar0);
err_free:
    // 最后释放设备私有结构体
    kfree(ddev);
err_regions:
    // 关闭 Bus Master，阻止设备继续主动 DMA
    pci_clear_master(pdev);
    // 释放此前申请的 BAR 资源
    pci_release_regions(pdev);
err_disable:
    // 关闭设备的 PCI 访问能力
    pci_disable_device(pdev);

    return ret;
}

static void demo_remove(struct pci_dev *pdev)
{
    struct demo_device *ddev = pci_get_drvdata(pdev);

    demo_unregister_interface(ddev);
    demo_hw_stop(ddev);
    demo_irq_exit(ddev);
    demo_dma_exit(ddev);

    // 解除当前设备 BAR0 的 MMIO 映射
    pci_iounmap(pdev, ddev->bar0);

    // 关闭 Bus Master，阻止设备继续主动 DMA
    pci_clear_master(pdev);
    // 释放此前申请的 BAR 资源
    pci_release_regions(pdev);
    // 关闭设备的 PCI 访问能力
    pci_disable_device(pdev);

    // 最后释放设备私有结构体
    kfree(ddev);
}
```

---

# 附录 F：lspci / setpci / sysfs 调试命令清单

```
# PCI 驱动常用排查命令合集；每条命令前保留用途说明。
# 查看全部 PCI 设备
lspci

# 显示数字 ID 和驱动
lspci -nnk

# 查看指定设备详细信息
lspci -s 03:00.0 -vv

# 查看 PCIe 链路
lspci -s 03:00.0 -vv | grep -E 'LnkCap|LnkSta'

# 查看配置空间
lspci -s 03:00.0 -xxx

# 读取 Command 寄存器
setpci -s 03:00.0 COMMAND

# 查看 BAR 资源
cat /sys/bus/pci/devices/0000:03:00.0/resource

# 查看 NUMA 节点
cat /sys/bus/pci/devices/0000:03:00.0/numa_node

# 查看邻近 CPU
cat /sys/bus/pci/devices/0000:03:00.0/local_cpulist

# 查看驱动
readlink /sys/bus/pci/devices/0000:03:00.0/driver

# 解绑驱动
echo 0000:03:00.0 \
  > /sys/bus/pci/drivers/demo_pci/unbind

# 绑定驱动
echo 0000:03:00.0 \
  > /sys/bus/pci/drivers/demo_pci/bind

# 移除设备
echo 1 \
  > /sys/bus/pci/devices/0000:03:00.0/remove

# 重新扫描 PCI 总线
echo 1 > /sys/bus/pci/rescan

# 查看中断
cat /proc/interrupts

# 查看 IOMMU Group
readlink \
  /sys/bus/pci/devices/0000:03:00.0/iommu_group

# 查看 SR-IOV 能力
cat \
  /sys/bus/pci/devices/0000:03:00.0/sriov_totalvfs

# 启用 8 个 VF
echo 8 \
  > /sys/bus/pci/devices/0000:03:00.0/sriov_numvfs

# 查看 PCI 和 AER 日志
dmesg -T | grep -Ei 'pci|pcie|aer|iommu|dma'
```

---

通过对 Linux PCI 设备驱动进行系统化分析，我们可以看到，一个 PCIe 设备从被系统发现到真正投入工作，需要经过多个层次的紧密配合。PCI Core 首先完成总线扫描、配置空间读取以及 `pci_dev` 创建。Driver Core 紧接着完成设备和驱动的匹配。驱动的 `probe()` 再完成设备使能、BAR 申请、MMIO 映射、DMA 配置以及中断注册。在设备真正工作以后，CPU 通过 MMIO 控制设备，设备通过 DMA 访问系统内存，并通过 MSI-X 向 CPU报告完成事件。IOMMU 对 DMA 地址进行转换和隔离，电源管理框架负责设备低功耗状态，AER 则负责 PCIe 错误上报与恢复。

因此，一个完整的 PCI 驱动分析思路应当是：

```
先观察 PCIe 拓扑和配置空间
          ↓
确认设备与驱动是否正确匹配
          ↓
检查 BAR 和 MMIO
          ↓
检查 Bus Master 和 DMA 映射
          ↓
检查中断向量和处理流程
          ↓
检查电源状态、IOMMU 和 AER
          ↓
最后分析队列、NUMA 和性能
```

真正可靠的 PCI 驱动，不能只是“能够读写寄存器”。它还必须保证：设备能够正确初始化；错误路径能够完整回滚；DMA 不会访问非法内存；中断不会在资源释放后继续运行；热拔出时不会继续访问失效 BAR；Suspend 和 Resume 后可以恢复工作；AER 错误发生后可以安全停止或恢复；多队列和 NUMA 环境下具备合理性能。只有把 PCI 配置空间、Linux 设备模型、MMIO、中断、DMA 以及错误恢复连接到一起，才能够真正理解 Linux PCI 驱动的完整工作机制。

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/08028c28_1789737662238?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247495132%26idx%3D1%26sn%3D446cd15e7756a298a49443956b2fd0af%26chksm%3Dc3ff1f85f740c36b5e1b82e0d25e327215c73546f36b415b48cdafb9887ecc12418bb1ddb947%26mpshare%3D1%26scene%3D1%26srcid%3D0918bGWziQayn6iCrrre90Sa%26sharer_shareinfo%3D85151e264dfd80f8cf28ebb92dcf1b0b%26sharer_shareinfo_first%3D85151e264dfd80f8cf28ebb92dcf1b0b%23rd&s=obsidian)