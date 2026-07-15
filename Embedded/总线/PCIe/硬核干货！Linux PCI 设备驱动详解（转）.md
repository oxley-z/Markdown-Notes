---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247486966&idx=1&sn=2040c9e2306d27772a74767b31820c9c&chksm=c387d1685a2c081005d935b2003e6fef19c2702eaa927013b5d726421dd4adc428f3681ba552&mpshare=1&scene=1&srcid=0713jCSul0zIHOCq6yAa4H4Q&sharer_shareinfo=09399eb99f57d6a9f8de941cf6d8ae44&sharer_shareinfo_first=09399eb99f57d6a9f8de941cf6d8ae44#rd
saved: 2026-07-13 23:22:43
tags:
  - 笔记同步助手
id: a525c3e3-1a77-461c-8765-80cc44f54eed
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-06-11 20:53

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/d5673b61529048fc2392d6b1f5e4601a_MD5.gif)

大家好，我是蟹老板～

从业十余年，我越来越觉得，有些东西真的绕不过去。

字符设备可以先会个大概，平台设备可以边写边查。GPIO、I2C、SPI 这些外设驱动，哪怕一开始不太懂，也能靠着板级代码、设备树和示例驱动慢慢摸出来。

但 PCI 不一样。

PCI 这玩意儿，表面看是一个总线驱动问题，实际背后牵着一整套硬件体系。设备怎么被发现，资源怎么分配，CPU 怎么访问设备寄存器，设备怎么直接搬内存，中断怎么通知 CPU，虚拟化怎么把设备直通给虚拟机。

每一个问题单独拿出来，都够把你问倒。

但越往后做项目我越明白一个道理：**所有高速外设驱动，本质上都是PCIe驱动**。

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111004191.jpg)

## 一、PCI/PCIe 子系统基础

当我们开始一个PCIe设备驱动的开发时，通常会面对设备树或BIOS/UEFI已经为我们初始化好的硬件。我们的程序大多是作为Linux内核的一部分在运行。

那就得先搞清楚一件事：我们写的这个驱动，到底是在跟什么打交道？或者说，我们的设备在Linux内核这个大千世界里，是如何被找到、被识别、然后被我们“认领”的？

你写的 PCI 驱动，只是这条链路最后的一环。

### 1.1 PCI 总线架构概述

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111004296.jpg)

PCI这玩意儿，全称Peripheral Component Interconnect，外围组件互连。说白了就是**CPU跟外设之间的一条高速公路**。

把外设挂到一套标准总线上，CPU 不需要为每一种设备设计一套专用访问方式。设备按照统一格式暴露**配置空间**，系统通过统一机制发现设备、分配资源、建立地址映射。

传统 PCI 是**并行总线**。多个设备共享总线，仲裁、时序、信号完整性都比较麻烦。

PCIe 是**点对点串行连接**。每个设备通过链路连接到 Root Complex 或 Switch。它不再是老式共享并行总线，而更像一个分层交换网络。

你可以把 PCIe 拓扑想成这样。

```
CPU
 |
Root Complex
 |
 +-- PCIe Switch
 |     +-- Endpoint 1  网卡
 |     +-- Endpoint 2  采集卡
 |
 +-- Endpoint 3  NVMe SSD
```

**Root Complex** 是 CPU 和 PCIe 世界之间的桥。

**Switch** 负责扩展更多下游端口。

**Endpoint** 就是真正的设备，比如网卡、NVMe、GPU、FPGA 卡。

这个模型一旦记住，后面很多东西就通了。Linux 扫描 PCI 总线，其实就是从 Root Complex 往下走，遇到 Bridge 就继续扫下一层，遇到 Endpoint 就创建对应的 **`pci_dev`**。

### 1.2 PCI、PCI-X、PCIe 的关系

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111004377.jpg)

PCI 是老前辈。

PCI-X 是 PCI 的增强版。

PCIe 是后来真正成为主流的高速串行总线。

它们硬件形态差异挺大，但 Linux 驱动开发中，很多 API 仍然沿用了 PCI 这套抽象。

所以你会看到，明明设备是 PCIe 网卡，驱动里还是叫 **`struct pci_driver`**，设备还是 **`struct pci_dev`**，配置空间还是 PCI configuration space，**BAR** 还是 BAR。

这不是历史包袱吗？

是，也不是。

从驱动作者角度看，这种统一抽象反而省事。你不需要每次都纠结底层到底是传统 PCI 还是 PCIe。大多数时候，驱动关心的是**设备 ID、配置空间、BAR、中断、DMA**。这些核心概念没有变。

PCIe 真正新增的东西更多体现在链路层、事务层、扩展能力、错误处理、虚拟化能力上，比如 AER、ASPM、SR-IOV、ATS、PASID。

**PCI、PCI-X、PCIe三者硬件不兼容，但软件协议完全兼容**。

### 1.3 Root Complex、Switch、Endpoint 与 PCIe 拓扑

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111004453.jpg)

Root Complex 它一头连 CPU 和内存系统，一头连 PCIe 总线。CPU 发起对设备寄存器的访问，通常会经过 Root Complex 转成 PCIe 事务。设备发起 DMA，也会通过 Root Complex 访问主机内存。

Switch 像 PCIe 世界里的交换机。它有一个上游端口，多个下游端口。下游可以挂 Endpoint，也可以继续挂 Switch。

Endpoint 是最终设备。网卡、NVMe、GPU、加速卡、FPGA 板卡都属于 Endpoint。

Linux 里面每条 PCI 总线由 **`struct pci_bus`** 表示，每个设备由 `struct pci_dev` 表示。Bridge 后面会出现新的 bus number。你用 **`lspci -t`** 可以看到这种树状结构。

```
lspci -t
```

输出大概类似这样。

```
-[0000:00]-+-00.0
           +-01.0-[01]----00.0
           +-02.0-[02-04]----00.0-[03-04]--+-00.0
           |                                \-01.0
           \-1f.0
```

新手看这个可能头疼，没关系，你先记住一句：**PCIe 不是一根线挂到底，它是拓扑，PCIe的拓扑结构，说白了就是一棵树。**

这个拓扑决定了设备在哪条 bus 上，也决定了资源窗口怎么分配，中断怎么路由，甚至影响 DMA 和 NUMA 亲和性。

### 1.4 Bus、Device、Function 与 BDF 地址

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111004524.jpg)

每个 PCI 设备都有一个地址，常见格式是 **BDF**。

```
Bus:Device.Function
```

比如：

```
0000:03:00.0
```

这里的含义是：

```
Domain  0000
Bus     03
Device  00
Function 0
```

严格一点说，完整格式是：

```
Domain:Bus:Device.Function
```

Domain 也叫 PCI segment。普通机器上常见是 `0000`。大系统或者多 host bridge 场景可能有多个 domain。

Device 是总线上的设备号。Function 是功能号。**一个物理设备可以有多个 function**。比如一个多功能网卡可能 function 0 是网络功能，function 1 是管理功能。或者某些设备同时暴露音频、视频、控制功能。

Linux 里面，`pci_dev` 里就保存了这些信息。

写驱动的时候，你不一定直接处理 BDF，但调试时它非常重要。因为日志、sysfs、lspci 都围绕这个地址展开。

```
lspci -s 03:00.0 -vvv
```

这条命令基本是 PCI 驱动调试的老朋友。说老实话，我当年用它用到快有肌肉记忆。

### 1.5 PCI 配置空间与设备标识

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111004640.jpg)

PCI 设备为什么能被系统自动发现？

关键就在**配置空间**。

每个 PCI 设备都有一段标准配置空间。里面放着 **Vendor ID、Device ID、Class Code、BAR、Command Register、Status Register、Capabilities Pointer** 等信息。

系统启动时，固件和内核会扫描可能存在的 bus/device/function，读取配置空间。如果读到了合法的 Vendor ID，就说明这个位置有设备。

几个字段必须熟。

-   • Vendor ID 表示厂商。
    
-   • Device ID 表示具体设备。
    
-   • Class Code 表示设备类型。
    
-   • Subsystem Vendor ID 和 Subsystem Device ID 常用于板卡厂商区分。
    
-   • Revision ID 表示硬件版本。
    
-   • BAR 描述设备需要的地址资源。
    
-   • Command Register 控制设备是否响应 Memory Space、I/O Space、Bus Master 等访问。
    

Linux 驱动匹配主要靠 **Vendor ID 和 Device ID**。你写 `pci_device_id` 表的时候，基本就是告诉内核，我这个驱动能支持哪些设备。

```
static const struct pci_device_id demo_pci_ids[] = {
    { PCI_DEVICE(0x1234, 0x5678) },
    { 0, }
};
MODULE_DEVICE_TABLE(pci, demo_pci_ids);
```

这里的 `0x1234` 是 Vendor ID，`0x5678` 是 Device ID。真实项目里你得换成设备自己的 ID。

### 1.6 PCI 枚举流程与 Linux 设备模型

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111004856.jpg)

PCI 枚举可以理解成系统给设备“上户口”。

Linux 启动时会扫描 PCI 总线。发现设备后，创建 `pci_dev`。配置资源，建立 sysfs 节点。等驱动注册进来后，驱动核心会根据 ID table 尝试匹配。匹配成功，就调用驱动的 **`probe()`**。

路径大概是这样。

```
扫描 PCI 总线
   |
读取配置空间
   |
发现设备
   |
创建 struct pci_dev
   |
加入 Linux 设备模型
   |
驱动注册 struct pci_driver
   |
ID table 匹配
   |
调用 probe
```

这个过程有个很重要的点，你的驱动不是主动去找硬件，你的驱动是把自己注册给 PCI 核心，PCI 核心帮你匹配设备，匹配上才调用你的 probe。

这就是 Linux 设备模型的味道。

很多驱动代码里只有一个 **`module_pci_driver()`**，看起来像魔幻，其实它背后就是注册一个 `pci_driver`。

```
module_pci_driver(demo_pci_driver);
```

### 1.7 PCIe 分层、链路训练与事务层

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111004943.jpg)

PCIe 是分层协议，常见分法是**事务层、数据链路层、物理层**。

事务层负责生成和处理 **TLP**。TLP 可以是 Memory Read、Memory Write、Configuration Read、Configuration Write、Completion 等。

数据链路层负责可靠传输，比如序号、ACK/NAK、重传。

物理层负责电气信号、Lane、速率、编码、**链路训练**。

链路训练是 PCIe 设备和对端协商链路宽度、速率等参数的过程，是物理层的事情。比如一个设备物理支持 x8，但实际插在 x4 插槽里，它最后可能就是 x4 运行。再比如设备支持 Gen4，但主板或者 BIOS 设置限制，它可能降到 Gen3。

这就是为什么调性能时要看：

```
lspci -s 03:00.0 -vvv | grep -i speed
```

你以为设备是 PCIe Gen4 x8，实际跑成 Gen3 x4。带宽少了一大截。然后软件同学背锅。硬件同学一句“你驱动是不是没优化好”，你血压直接上来。

有点东西，真的。

## 二、Linux PCI 驱动模型

讲完硬件和枚举，我们回到驱动。

Linux PCI 驱动的本质是三件事。

**让驱动和设备匹配上。** **拿到设备资源并初始化硬件。** **在设备离开时把资源干净释放掉。**

你看，听起来很简单。实际写起来，还是有很多坑等着你踩的。

### 2.1 Linux 设备驱动模型基础

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005036.jpg)

Linux 设备模型里，**bus、device、driver** 是三大核心。

-   • bus 负责管理一种总线类型。
    
-   • device 表示具体设备。
    
-   • driver 表示能驱动某类设备的软件。
    

PCI 子系统注册了 PCI bus type。扫描到的 PCI 设备挂到这个 bus 上。PCI 驱动也注册到这个 bus 上。

匹配过程由驱动核心完成。

```
pci_bus_type
   |
   +-- pci_dev
   +-- pci_driver
```

当 `pci_dev` 和 `pci_driver` 匹配成功，内核就调用驱动的 `probe()`。

这套模型有点绕，但好处很大。驱动不用关心设备什么时候出现，也不用自己遍历全系统 PCI 设备。它只要声明自己支持哪些 ID，然后实现 probe/remove。

### 2.2 `pci_bus`、`pci_dev`、`pci_driver` 的关系

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005115.jpg)

`pci_bus` 表示一条 PCI 总线。

`pci_dev` 表示总线上的一个 function，**不是一个物理卡**。这个区别很重要。多功能设备会有多个 `pci_dev`。

`pci_driver` 表示一个 PCI 驱动。

常见结构体大概这样。

```
struct pci_driver {
    struct list_head node;
    const char *name;
    const struct pci_device_id *id_table;
    int  (*probe)(struct pci_dev *dev, const struct pci_device_id *id);
    void (*remove)(struct pci_dev *dev);
    void (*shutdown)(struct pci_dev *dev);
    int  (*suspend)(struct pci_dev *dev, pm_message_t state);
    int  (*resume)(struct pci_dev *dev);
};
```

不同内核版本结构体成员会有差异，但核心不变。

你真正写驱动时，最常碰到的就是：

```
static struct pci_driver demo_pci_driver = {
    .name     = "demo_pci",
    .id_table = demo_pci_ids,
    .probe    = demo_probe,
    .remove   = demo_remove,
};
```

PCI 驱动就是从这里长出来的。

### 2.3 `pci_device_id` 与 ID Table 匹配机制

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005230.jpg)

PCI 驱动怎么知道自己支持哪个设备？

靠 **`pci_device_id`** 表。

```
static const struct pci_device_id demo_pci_ids[] = {
    { PCI_DEVICE(0x1234, 0x5678) },
    { 0, }
};
```

**`PCI_DEVICE()`** 这个宏会填 Vendor ID 和 Device ID。

也可以更复杂一点，用 class code 匹配，或者匹配 subsystem ID。

```
static const struct pci_device_id demo_pci_ids[] = {
    {
        .vendor = 0x1234,
        .device = 0x5678,
        .subvendor = PCI_ANY_ID,
        .subdevice = PCI_ANY_ID,
    },
    { 0, }
};
```

很多人调驱动时发现 probe 不进，第一反应是内核坏了。别急，先看 ID 对不对。

```
lspci -nn
```

你会看到类似：

```
03:00.0 Processing accelerators [1200]: VendorName DeviceName [1234:5678]
```

方括号里的 `[1234:5678]` 就是你要匹配的 ID。

如果 ID 写错，probe 永远不会来。你在 probe 里打印一万行都没用。这个坑我也踩过，后来学乖了，先看 `lspci -nn`。

### 2.4 `MODULE_DEVICE_TABLE` 的作用

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005319.jpg)

很多新手会问，既然 id\_table 已经写在驱动里了，为什么还要加：

```
MODULE_DEVICE_TABLE(pci, demo_pci_ids);
```

这个宏不是给 probe 运行时看的。它主要用于生成模块别名信息，**让用户态工具可以根据设备 ID 自动加载对应模块**。

比如热插设备出现时，内核发 uevent，用户态根据 modalias 找到合适模块加载。

你可以用这个命令查看模块别名：

```
modinfo demo_pci.ko
```

如果缺了 `MODULE_DEVICE_TABLE`，手动 `insmod` 可能还能工作，但自动加载可能不行。

这就是 Linux 驱动里常见的细节。少一行代码，功能不是立刻炸，而是在某个看似无关的场景里阴你一下。

### 2.5 `pci_register_driver()` 与驱动注册流程

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005423.jpg)

驱动注册可以显式写：

```
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

也可以用宏简化：

```
module_pci_driver(demo_pci_driver);
```

这个宏本质还是注册和注销。

驱动注册后，PCI 核心会拿驱动的 id\_table 去匹配已有设备。如果设备已经在系统里，probe 会立刻被调用。如果设备后插入，热插拔流程中也会触发匹配。

这就是为什么模块一加载，probe 可能马上打印日志。

```
insmod demo_pci.ko
dmesg -w
```

如果没有日志，你就得查三件事。

1.  1\. 设备是否存在。
    
2.  2\. ID 是否匹配。
    
3.  3\. 驱动是否真的加载成功。
    

别上来就怀疑硬件。硬件同学听了会笑，你自己也会多走弯路。

### 2.6 probe、remove、shutdown 生命周期

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005517.jpg)

PCI 驱动生命周期绕不开三个函数。

-   -   **probe** 负责设备初始化。
-   -   **remove** 负责设备移除和资源释放。
-   -   **shutdown** 负责系统关机或重启前处理设备状态。

probe 里通常做这些事：

```
使能 PCI 设备
申请 BAR 资源
映射 MMIO
设置 DMA mask
分配 DMA 缓冲区
申请中断
初始化硬件
注册上层设备接口
```

remove 里反过来。

```
停止硬件
释放中断
释放 DMA 资源
解除 MMIO 映射
释放 BAR
禁用设备
```

你会发现，probe 和 remove 像镜像，但不是简单镜像。因为 probe 可能失败在中间任何一步。失败后也必须释放已经申请成功的资源。

这就是 PCI 驱动最考验基本功的地方。

正常路径谁都会写。 **异常路径才暴露水平。**

## 三、PCI 设备初始化与资源管理

PCI 驱动真正开始干活，是从 probe 开始。

probe 不是一个普通初始化函数，它是驱动和设备绑定后的入口。你手里拿到一个 `struct pci_dev *pdev`，然后要把这个硬件从“系统发现了”推进到“驱动可用”。

这中间有一套固定套路。

### 3.1 probe 函数什么时候被调用

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005598.jpg)

probe 被调用的前提是**设备和驱动匹配成功**。

设备可能早就被内核扫描到了，只是没有驱动。你加载模块时，驱动注册进 PCI 核心，PCI 核心发现这个驱动支持某个已有设备，于是调用 probe。

也可能驱动已经加载，设备后来热插入。设备出现后被枚举，匹配到驱动，probe 也会被调用。

**probe 不是你主动调用的。**

所以 probe 里不要写那种假设系统状态很干净的代码。它可能发生在启动阶段，也可能发生在运行时热插拔阶段。资源申请失败、设备状态异常、固件配置奇怪，这些都可能出现。

### 3.2 `pci_enable_device()` 使能设备

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005667.jpg)

probe 里常见第一步是：

```
ret = pci_enable_device(pdev);
if (ret)
    return ret;
```

这个函数会**使能设备**，让设备可以响应 I/O 或 Memory 访问。它还会处理一些平台相关细节。

很多设备如果不调用 `pci_enable_device()`，你后面直接访问 BAR，可能就炸。不是每个平台都炸，有的平台看起来能跑，有的平台直接异常。这种最烦，因为它不是稳定复现。

如果设备要发起 DMA，还要调用：

```
pci_set_master(pdev);
```

这一步会开启 **Bus Master 能力**。没有 Bus Master，设备没资格主动访问主机内存。DMA 当然也玩不起来。

这俩函数别省。省出来的不是性能，是事故。

### 3.3 `pci_request_regions()` 申请 BAR 资源

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005743.jpg)

PCI 设备通过 BAR 暴露资源。驱动访问之前，需要申请这些资源，避免多个驱动乱抢。

```
ret = pci_request_regions(pdev, "demo_pci");
if (ret)
    goto err_disable_device;
```

也可以只申请某一个 BAR：

```
ret = pci_request_region(pdev, bar, "demo_pci");
```

**`pci_request_regions()`** **并不是映射 MMIO。** 它只是告诉内核，这些 PCI 资源我这个驱动要用了，别让别人碰。

资源管理里最怕“我以为”。你以为没人用。内核不这么认为。规范申请，规范释放。

释放时对应：

```
pci_release_regions(pdev);
```

### 3.4 BAR 类型

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005801.jpg)

BAR 是 Base Address Register。

设备通过 BAR 告诉系统，我需要一段地址空间。这个地址空间可能是 I/O port，也可能是 MMIO，也可能是 ROM。

**常见类型有三种**：

1.  1.  **I/O BAR**：使用 I/O port 空间。老设备常见，现在少了。
2.  2.  **Memory BAR**：使用内存映射 I/O。现代 PCIe 设备主要用这个。
3.  3.  **ROM BAR**：指向扩展 ROM，比如网卡 PXE ROM、显卡 BIOS 之类。普通驱动较少直接碰它。

驱动里判断 BAR 类型可以用：

```
resource_size_t start = pci_resource_start(pdev, bar);
resource_size_t len   = pci_resource_len(pdev, bar);
unsigned long flags   = pci_resource_flags(pdev, bar);

if (flags & IORESOURCE_MEM) {
    /* MMIO BAR */
}

if (flags & IORESOURCE_IO) {
    /* I/O port BAR */
}
```

现代 PCIe 设备驱动，大多数是映射 Memory BAR，然后用 `readl()`、`writel()` 访问设备寄存器。

### 3.5 32-bit BAR、64-bit BAR、Prefetchable BAR

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005913.jpg)

Memory BAR 还分 32-bit 和 64-bit。

-   -   **32-bit BAR**：表示地址在 32 位范围内。
-   -   **64-bit BAR**：可以放到 4GB 以上地址空间。

对大 BAR 设备，比如 GPU、加速卡、某些 FPGA，64-bit BAR 很常见。

**Prefetchable** 表示这个内存区域可以被预取。一般用于设备内存、framebuffer 这类区域。普通控制寄存器 BAR 通常不是 prefetchable，因为寄存器访问有副作用，乱预取就危险了。

驱动不一定要手动解析这些位，因为 Linux PCI 核心已经完成资源分配。但调试时看到这些信息，你要知道它在说什么。

```
lspci -s 03:00.0 -vvv
```

你可能看到：

```
Region 0: Memory at f7200000 (64-bit, non-prefetchable) [size=64K]
Region 2: Memory at 8000000000 (64-bit, prefetchable) [size=256M]
```

Region 0 常常是控制寄存器。Region 2 可能是大块设备内存。具体得看硬件手册。

### 3.6 `pci_resource_start()` / `pci_resource_len()`

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111005980.jpg)

这两个函数非常常用。

```
bar_start = pci_resource_start(pdev, 0);
bar_len   = pci_resource_len(pdev, 0);
```

-   -   **`pci_resource_start()`** 返回 BAR 的起始物理资源地址。
-   -   **`pci_resource_len()`** 返回 BAR 的长度。

注意，**这个地址不是普通内核虚拟地址。不能直接解引用。**

错误写法如下。

```
u32 val = *(u32 *)(bar_start + 0x100);
```

这就是在给自己挖坑。

正确方式是先映射，再通过 I/O accessor 访问。

### 3.7 `pci_iomap()` / `ioremap()` 映射 MMIO

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006050.jpg)

映射 BAR 可以用：

```
void __iomem *mmio;

mmio = pci_iomap(pdev, 0, 0);
if (!mmio) {
    ret = -ENOMEM;
    goto err_release_regions;
}
```

**`pci_iomap()`** 会根据 BAR 类型做合适映射。很多 PCI 驱动喜欢用它。

也可以手动用 **`ioremap()`**。

```
mmio = ioremap(bar_start, bar_len);
```

现在很多驱动还会用 managed 版本，比如 `pcim_iomap_regions()`、`pcim_iomap_table()`。这类 devres/pcim 接口可以在 detach 时自动释放，减少错误路径代码。它很香，但新手最好先理解手动释放流程，不然你连资源生命周期都没建立起来。

映射后访问寄存器：

```
u32 val = readl(mmio + 0x100);
writel(val | 0x1, mmio + 0x100);
```

**`__iomem`** 这个标记不是装饰品。它提醒你，这不是普通内存。

### 3.8 设备私有结构体设计

一个正经 PCI 驱动，通常会定义**设备私有结构体**。

```
struct demo_dev {
    struct pci_dev *pdev;
    void __iomem *bar0;
    resource_size_t bar0_len;

    int irq;
    void *dma_cpu;
    dma_addr_t dma_handle;
    size_t dma_size;

    spinlock_t lock;
};
```

probe 里分配它：

```
struct demo_dev *ddev;

ddev = kzalloc(sizeof(*ddev), GFP_KERNEL);
if (!ddev)
    return -ENOMEM;

ddev->pdev = pdev;
pci_set_drvdata(pdev, ddev);
```

remove 里取出来：

```
struct demo_dev *ddev = pci_get_drvdata(pdev);
```

这个结构体就是你的驱动上下文。BAR 地址、DMA buffer、中断号、锁、队列、状态位都放这里。

**别到处用全局变量。** 别问为什么。问就是多卡设备会让你做人。

当系统里插两张一样的 PCIe 卡时，全局变量驱动直接露馅。你以为自己在写驱动，其实是在写只能跑一次的 demo。

### 3.9 probe 失败路径与资源回滚

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006287.jpg)

**probe 失败路径与资源回滚** 这是驱动代码最能看出水平的地方。

一个普通 probe 可能这样。

```
static int demo_probe(struct pci_dev *pdev,
                      const struct pci_device_id *id)
{
    struct demo_dev *ddev;
    int ret;

    ddev = kzalloc(sizeof(*ddev), GFP_KERNEL);
    if (!ddev)
        return -ENOMEM;

    ddev->pdev = pdev;
    pci_set_drvdata(pdev, ddev);

    ret = pci_enable_device(pdev);
    if (ret)
        goto err_free;

    ret = pci_request_regions(pdev, "demo_pci");
    if (ret)
        goto err_disable;

    pci_set_master(pdev);

    ddev->bar0 = pci_iomap(pdev, 0, 0);
    if (!ddev->bar0) {
        ret = -ENOMEM;
        goto err_release;
    }

    ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    if (ret) {
        ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
        if (ret)
            goto err_iounmap;
    }

    return 0;

err_iounmap:
    pci_iounmap(pdev, ddev->bar0);
err_release:
    pci_release_regions(pdev);
err_disable:
    pci_disable_device(pdev);
err_free:
    kfree(ddev);
    return ret;
}
```

这个代码不华丽，但它像个成年人写的。

每一步失败，都释放前面已经成功申请的资源。remove 也是对应释放。

```
static void demo_remove(struct pci_dev *pdev)
{
    struct demo_dev *ddev = pci_get_drvdata(pdev);

    pci_iounmap(pdev, ddev->bar0);
    pci_release_regions(pdev);
    pci_disable_device(pdev);
    kfree(ddev);
}
```

真实项目里还会有 IRQ、DMA、workqueue、上层设备注册。资源越多，错误路径越容易乱。建议你写 probe 的时候别一口气写完。每加一个资源，就把失败路径和 remove 补上。

不然你迟早会在卸载驱动时遇到 kernel panic。那个画面，谁看谁沉默。

## 四、MMIO 与设备寄存器访问

PCI 设备驱动绕不开寄存器。

设备手册上通常会写：

```
BAR0 + 0x0000  Device ID Register
BAR0 + 0x0004  Control Register
BAR0 + 0x0008  Status Register
BAR0 + 0x0010  DMA Address Low
BAR0 + 0x0014  DMA Address High
BAR0 + 0x0018  DMA Length
BAR0 + 0x001C  DMA Start
```

你要做的，就是把 BAR 映射到内核虚拟地址，然后用正确方式读写这些寄存器。

### 4.1 CPU 如何访问 PCI 设备寄存器

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006407.jpg)

**MMIO** 的意思是 Memory-Mapped I/O。

设备寄存器被映射到 CPU 的地址空间。CPU 发起一次读写，看起来像访问某个地址。实际上这个访问会经过总线桥，转成 PCIe Memory Read 或 Memory Write 事务，最后到达设备。

这和普通内存访问完全不是一回事。

普通内存在 DRAM 上，缓存策略、乱序执行、预取都可能参与。

设备寄存器有副作用。读一次状态寄存器可能清除中断。写一次控制寄存器可能启动 DMA。你当然不希望编译器或者 CPU 自作聪明乱合并、乱缓存、乱重排。

所以 Linux 提供了专门的 **I/O accessor**。

### 4.2 寄存器读函数：`readb/readw/readl/readq`

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006507.jpg)

读取寄存器时，用：

```
u8  v8  = readb(base + REG8);
u16 v16 = readw(base + REG16);
u32 v32 = readl(base + REG32);
u64 v64 = readq(base + REG64);
```

最常见的是 **`readl()`**，因为很多寄存器是 32 位。

注意 offset 要和硬件手册对齐。该按 4 字节访问就别按字节乱读。某些设备对访问宽度有要求。你用 `readb()` 读一个只支持 32 位访问的寄存器，设备可能不认。

### 4.3 寄存器写函数：`writeb/writew/writel/writeq`

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006569.jpg)

写寄存器也一样。

```
writel(0x1, base + REG_CONTROL);
```

典型启动 DMA 的伪代码：

```
#define REG_DMA_ADDR_LO  0x100
#define REG_DMA_ADDR_HI  0x104
#define REG_DMA_LEN      0x108
#define REG_DMA_START    0x10c

static void demo_start_dma(struct demo_dev *ddev,
                           dma_addr_t dma_addr,
                           u32 len)
{
    writel(lower_32_bits(dma_addr), ddev->bar0 + REG_DMA_ADDR_LO);
    writel(upper_32_bits(dma_addr), ddev->bar0 + REG_DMA_ADDR_HI);
    writel(len, ddev->bar0 + REG_DMA_LEN);

    /* 确保前面的寄存器写入顺序 */
    wmb();

    writel(1, ddev->bar0 + REG_DMA_START);
}
```

这里的 **`wmb()`** 不一定每个设备都需要，但你得有这个意识。设备寄存器访问顺序不是“我代码这么写，硬件就一定这么看”。这句话难听，但是真的。

### 4.4 MMIO 访问顺序与内存屏障

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006667.jpg)

现代 CPU 会乱序执行。编译器也可能重排。设备访问又不能随便乱序。

Linux 的 I/O 访问函数一般已经包含一定顺序语义，但复杂场景里，尤其 DMA buffer 和设备寄存器配合时，你还是需要理解**内存屏障**。

比如你先填好 DMA 描述符，再通知设备去读。

```
desc->addr = dma_addr;
desc->len  = len;
desc->ctrl = DESC_OWNED_BY_DEVICE;

wmb();

writel(DOORBELL, ddev->bar0 + REG_DOORBELL);
```

没有屏障会怎样？

设备可能先看到 doorbell，然后去读描述符。结果描述符内容还没对设备可见。于是设备读到旧数据，DMA 飞到奇怪地址，系统炸。你还在那儿看代码，觉得自己没写错。

这种 bug 最恶心。不是每次复现。压力一大就出现。换个 CPU 架构更明显。x86 上可能隐藏了，ARM 上直接暴露。

所以啊，驱动要有硬件可见性的脑子。

### 4.5 posted write 问题

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006507.jpg)

PCIe 写操作通常可能是 **posted write**。意思是 CPU 这边写出去了，不一定等设备真正处理完才返回。

你写了一个寄存器，不代表设备已经看到。更不代表设备已经完成动作。

如果你需要确保之前的写到达设备，常见做法是**读回一个寄存器**。读操作通常会迫使前面的 posted write 刷出去。

```
writel(1, ddev->bar0 + REG_RESET);
readl(ddev->bar0 + REG_STATUS);
```

这个读回不是为了拿值。有时候就是为了 flush。

当然，具体要看设备手册。有些 reset 后需要延迟，有些需要轮询状态位，有些需要等待中断。不能靠玄学。

我以前见过一个驱动，写 reset 后立刻写配置寄存器。大部分机器没事，某一批服务器上设备偶发初始化失败。后来加了读回和状态轮询，才消停。你说离谱吧？PCIe 就是这么“讲道理但不讲情面”。

### 4.6 为什么不能直接解引用 MMIO 地址

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006770.jpg)

再强调一次，**不能直接把 MMIO 地址当普通指针**。

错误示例：

```
u32 val = *(u32 *)(ddev->bar0 + REG_STATUS);
*(u32 *)(ddev->bar0 + REG_CONTROL) = 1;
```

问题很多。

1.  1\. 编译器不知道这是 I/O 内存。
    
2.  2\. 访问顺序可能不符合预期。
    
3.  3\. 访问宽度可能不对。
    
4.  4\. 架构相关行为可能炸。
    
5.  5\. 静态检查也会骂你。
    

正确写法：

```
u32 val = readl(ddev->bar0 + REG_STATUS);
writel(1, ddev->bar0 + REG_CONTROL);
```

`__iomem` 的存在就是为了提醒你，别乱来。

## 五、PCI 中断处理

设备不能总让 CPU 轮询。

DMA 完成了，设备需要通知 CPU。 错误发生了，设备需要通知 CPU。 队列有新事件了，设备需要通知 CPU。

这就需要**中断**。

PCI 中断从传统 **INTx** 到 **MSI**，再到 **MSI-X**，演进非常明显。新设备基本都会用 MSI-X，尤其网卡、NVMe 这种多队列设备。

### 5.1 INTx 传统中断与共享中断

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111006947.jpg)

INTx 是传统 PCI 中断机制。它有 INTA、INTB、INTC、INTD 四根逻辑中断线。**多个设备可能共享中断**。

共享中断意味着什么？

你的中断处理函数被调用，不一定是你的设备触发的。你必须读取设备状态寄存器判断。

```
static irqreturn_t demo_irq_handler(int irq, void *data)
{
    struct demo_dev *ddev = data;
    u32 status;

    status = readl(ddev->bar0 + REG_INT_STATUS);
    if (!(status & DEMO_INT_PENDING))
        return IRQ_NONE;

    writel(status, ddev->bar0 + REG_INT_CLEAR);

    return IRQ_HANDLED;
}
```

如果是共享中断，却不判断设备状态，直接返回 `IRQ_HANDLED`，你就是在坑系统。

INTx 还有一个问题。它是电平触发，设备如果没有正确清中断，可能**中断风暴**。CPU 被打爆，系统卡得像 PPT。

### 5.2 MSI 原理与使用方式

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007083.jpg)

MSI 是 Message Signaled Interrupt。

它不再依赖传统中断线，而是**设备通过一次内存写事务向特定地址写入特定数据，触发 CPU 中断**。

听起来是不是有点怪？

设备发个写请求，CPU 就收到中断。现代系统就是这么玩。

MSI 的优点很明显。

-   • 不需要共享传统中断线。
    
-   • 中断归属更明确。
    
-   • 和 DMA 完成顺序更好配合。
    
-   • 多核系统上更容易分配向量。
    

Linux 驱动里，不建议再用老接口单独开 MSI。更常见的是用：

```
ret = pci_alloc_irq_vectors(pdev, 1, 1,
                            PCI_IRQ_MSI | PCI_IRQ_INTX);
```

这里允许 MSI，也允许 fallback 到 INTx。现实世界里，不是所有平台、BIOS、设备都能稳定支持 MSI。你写驱动别太理想主义。

获取 Linux IRQ 号：

```
irq = pci_irq_vector(pdev, 0);
```

然后注册：

```
ret = request_irq(irq, demo_irq_handler, 0, "demo_pci", ddev);
```

释放时：

```
free_irq(irq, ddev);
pci_free_irq_vectors(pdev);
```

顺序别乱。

### 5.3 MSI-X 原理与多队列设备

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007215.jpg)

**MSI-X** 比 MSI 更适合高性能设备。

MSI 通常要求连续向量。MSI-X 更灵活，每个 vector 可以单独配置。多队列网卡、NVMe 控制器都喜欢 MSI-X。

为什么多队列设备依赖 MSI-X？

因为**每个队列可以绑定一个中断向量，再把不同向量分配到不同 CPU**。这样收包、发包、完成队列可以并行处理。

网卡就是典型例子。

```
RX Queue 0 -> IRQ 32 -> CPU0
RX Queue 1 -> IRQ 33 -> CPU1
RX Queue 2 -> IRQ 34 -> CPU2
RX Queue 3 -> IRQ 35 -> CPU3
```

如果所有队列都挤到一个中断上，性能会很难看。你 DMA 再快，中断处理跟不上，也白搭。

### 5.4 `pci_alloc_irq_vectors()`

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007330.jpg)

推荐方式是：

```
ret = pci_alloc_irq_vectors(pdev, min_vecs, max_vecs,
                            PCI_IRQ_MSIX | PCI_IRQ_MSI | PCI_IRQ_INTX);
if (ret < 0)
    goto err;

nvec = ret;
```

**返回值是实际分配到的 vector 数量。不是 0 表示成功。** 这个细节别写错。

错误写法：

```
ret = pci_alloc_irq_vectors(...);
if (ret)
    return ret;
```

这就完蛋了。因为成功时可能返回 1、4、8。

正确写法是判断小于 0。

```
if (ret < 0)
    return ret;
```

这个坑挺常见。看着不像大问题，实际能让驱动永远初始化失败。

如果你想让内核帮忙做中断亲和性分散，可以带上 **`PCI_IRQ_AFFINITY`**。

```
ret = pci_alloc_irq_vectors(pdev, 1, max_vecs,
                            PCI_IRQ_MSIX | PCI_IRQ_MSI |
                            PCI_IRQ_AFFINITY);
```

### 5.5 `request_irq()` 注册中断处理函数

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007429.jpg)

拿到 vector 后，要注册中断。

```
for (i = 0; i < nvec; i++) {
    int irq = pci_irq_vector(pdev, i);

    ret = request_irq(irq, demo_irq_handler, 0,
                      "demo_pci", ddev);
    if (ret)
        goto err_free_irqs;
}
```

如果多个 vector 对应多个队列，通常每个队列有自己的上下文。

```
ret = request_irq(irq, demo_queue_irq_handler, 0,
                  q->name, q);
```

中断 handler 里拿到的 `dev_id` 就是 `q`。

释放时必须传同一个 `dev_id`。

```
free_irq(irq, q);
```

这个参数不只是摆设。共享中断场景下，它用于区分具体 handler。传错会出事。

### 5.6 顶半部与底半部

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007478.jpg)

**中断处理要快。**

**顶半部**就是硬中断 handler。它应该尽量短，做必须马上做的事情。

-   • 比如读取中断状态。
    
-   • 清除设备中断。
    
-   • 记录事件。
    
-   • 唤醒下半部。
    

复杂处理不要塞在硬中断里。比如大块数据解析、内存分配、可能睡眠的操作，都不适合放在硬中断上下文。

**下半部**可以用 tasklet、workqueue、threaded IRQ、NAPI 等机制。

-   • 网卡驱动里常见 NAPI。
    
-   • 普通设备可以用 threaded IRQ。
    
-   • 需要睡眠的处理可以放 workqueue。
    

### 5.7 tasklet、workqueue、threaded IRQ

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007588.jpg)

-   -   **tasklet**：运行在软中断上下文，不能睡眠。现在很多新驱动会避免使用 tasklet，转向更合适的机制。
-   -   **workqueue**：在进程上下文，可以睡眠。适合耗时或可能阻塞的工作。
-   -   **threaded IRQ**：很适合设备中断处理拆分。你可以把快速确认放到 top half，把复杂逻辑放到 thread\_fn。

```
ret = request_threaded_irq(irq,
                           demo_irq_top,
                           demo_irq_thread,
                           IRQF_ONESHOT,
                           "demo_pci",
                           ddev);
```

top half：

```
static irqreturn_t demo_irq_top(int irq, void *data)
{
    struct demo_dev *ddev = data;
    u32 status = readl(ddev->bar0 + REG_INT_STATUS);

    if (!status)
        return IRQ_NONE;

    writel(status, ddev->bar0 + REG_INT_CLEAR);
    ddev->irq_status = status;

    return IRQ_WAKE_THREAD;
}
```

thread\_fn：

```
static irqreturn_t demo_irq_thread(int irq, void *data)
{
    struct demo_dev *ddev = data;

    demo_handle_event(ddev, ddev->irq_status);

    return IRQ_HANDLED;
}
```

这类代码可读性也好。后面排查问题，自己不会被自己气死。

### 5.8 中断亲和性与负载均衡

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007668.jpg)

多队列设备最怕**所有中断打到一个 CPU**。

查看中断：
 
```
cat /proc/interrupts
```

设置中断亲和性：

```
cat /proc/irq/32/smp_affinity
echo 2 > /proc/irq/32/smp_affinity
```

当然，生产环境里不一定手工 echo，可能由 irqbalance、驱动、系统策略共同决定。

对高性能网卡和 NVMe，**IRQ 亲和性要和队列、CPU、NUMA 节点一起看**。设备在 NUMA node 0，结果中断和应用线程跑到 node 1，数据绕远路，性能自然下去。

很多性能问题不是代码慢，是拓扑没对齐。

这话我说过很多次，还是有人不信。直到压测数据糊脸上。

## 六、PCI DMA 机制

PCI 驱动最容易让人头皮发麻的部分来了。

**DMA**。

如果说 BAR 解决的是 CPU 如何访问设备，那么 DMA 解决的是**设备如何访问内存**。

高性能 PCIe 设备不可能靠 CPU 一次次搬数据。网卡收包、NVMe 读写、采集卡传图、GPU 访问 buffer，都要靠 DMA。

### 6.1 为什么 PCI 设备需要 DMA

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007730.jpg)

没有 DMA 会怎样？

设备要传数据，只能 CPU 读设备寄存器，再写内存。或者 CPU 从内存读，再写设备。数据量小还行，数据量一大 CPU 直接累死。

DMA 让设备自己访问主机内存。

```
设备 <----PCIe----> 内存
        DMA
```

CPU 只负责准备 buffer、告诉设备 DMA 地址、启动传输、处理中断。

这就是高性能 I/O 的基本模式。

### 6.2 Bus Master DMA 的含义

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007844.jpg)

设备要发起 DMA，必须具备 **Bus Master** 能力。驱动里通常调用：

```
pci_set_master(pdev);
```

这会在 PCI Command Register 里打开 Bus Master Enable 位。

**没有它，设备不能作为总线主设备发起事务。** 你的 DMA 描述符写得再漂亮也没用。

有些人会问，为什么 `pci_enable_device()` 不帮我全做了？

因为不是所有设备都应该被允许主动访问内存。Bus Master 是一个很敏感的能力。驱动明确调用，逻辑更清楚。

### 6.3 DMA 地址、物理地址、虚拟地址的区别

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111007963.jpg)

这里是新手重灾区。

-   -   **CPU 虚拟地址**：是内核代码访问内存用的地址。
-   -   **物理地址**：是 CPU 看到的物理内存地址。
-   -   **DMA 地址**：是设备看到的总线地址。

它们不一定一样。

尤其有 IOMMU 时，设备看到的 DMA 地址可能是 IOVA。它经过 IOMMU 翻译后才到真正物理内存。

**所以别拿****`virt_to_phys()`****糊弄 PCI DMA。现代驱动应该使用 DMA API。**

正确方式是：

```
dma_addr_t dma_handle;
void *cpu_addr;

cpu_addr = dma_alloc_coherent(&pdev->dev, size,
                              &dma_handle, GFP_KERNEL);
```

-   -   `cpu_addr` 给 CPU 用。
-   -   `dma_handle` 给设备用。

设备寄存器里写的是 `dma_handle`，不是 `cpu_addr`，也不是你自己算出来的物理地址。

这点千万别错。

### 6.4 Streaming DMA 与 Coherent DMA

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008044.jpg)

Linux DMA API 里常见两类。

1.  1.  **Coherent DMA（一致性DMA）**
2.  2.  **Streaming DMA（流式DMA）**

Coherent DMA 通过 **`dma_alloc_coherent()`** 分配。CPU 和设备看到的数据保持一致性，适合描述符环、控制块、频繁被 CPU 和设备共享的小块内存。

Streaming DMA 通过 **`dma_map_single()`****、****`dma_map_sg()`** 映射已有 buffer。适合一次性或阶段性传输，比如网络包、块 I/O 数据。

简单说：

-   • 描述符环常用 coherent。
    
-   • 数据 buffer 常用 streaming。
    

当然真实驱动里会更复杂，但这个入门判断够用了。

### 6.5 `dma_set_mask_and_coherent()`

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008120.jpg)

设备不是一定能访问所有内存。

老设备可能只能访问 32 位地址。现代设备可能支持 64 位 DMA。

**驱动需要告诉内核设备的 DMA 寻址能力。**

```
ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
if (ret) {
    ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
    if (ret)
        return ret;
}
```

这段代码很常见。

如果设备只支持 32 位 DMA，而你没有设置 mask，内核可能给你一个设备访问不到的 DMA 地址。设备一访问，轻则传输失败，重则系统异常。

这个 bug 一般还挺隐蔽，因为小内存机器上可能没事，大内存机器上就炸。

以前有个项目就是这样。开发机 16GB 跑得好好的，客户服务器 256GB 就随机 DMA 错误。最后发现设备只支持 32-bit DMA，驱动没正确设 mask。那天我盯着日志看了很久，感觉自己像个傻子。

### 6.6 `dma_alloc_coherent()`

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008189.jpg)

分配一致性 DMA 内存：

```
ddev->dma_size = 4096;

ddev->dma_cpu = dma_alloc_coherent(&pdev->dev,
                                   ddev->dma_size,
                                   &ddev->dma_handle,
                                   GFP_KERNEL);
if (!ddev->dma_cpu)
    return -ENOMEM;
```

释放：

```
dma_free_coherent(&pdev->dev,
                  ddev->dma_size,
                  ddev->dma_cpu,
                  ddev->dma_handle);
```

典型用途是 DMA 描述符环。

```
struct demo_desc {
    __le64 addr;
    __le32 len;
    __le32 flags;
};
```

CPU 填描述符，设备读取描述符。设备写回状态，CPU 再读取状态。

Coherent 不代表你完全不用考虑顺序。它解决的是缓存一致性问题，不是所有内存排序问题。该用屏障的地方还得用。

### 6.7 `dma_map_single()` / `dma_unmap_single()`

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008259.jpg)

Streaming DMA 常用：

```
dma_addr_t dma_addr;

dma_addr = dma_map_single(&pdev->dev, buf, len, DMA_TO_DEVICE);
if (dma_mapping_error(&pdev->dev, dma_addr))
    return -EIO;
```

**方向很重要**。

-   -   `DMA_TO_DEVICE`：表示 CPU 写好 buffer，设备读取。
-   -   `DMA_FROM_DEVICE`：表示设备写 buffer，CPU 后面读取。
-   -   `DMA_BIDIRECTIONAL`：表示双向。

传输完成后：

```
dma_unmap_single(&pdev->dev, dma_addr, len, DMA_TO_DEVICE);
```

-   • 别忘了 unmap。
    
-   • 别方向写反。
    
-   • 别 map 后 CPU 又乱改 buffer。
    

Streaming DMA 的规则比 coherent 更严格。**CPU 和设备对 buffer 的所有权要清楚。**

比如设备正在 DMA 写入 buffer，你 CPU 同时读这个 buffer。你读到什么？看命。驱动不能靠运气写。

### 6.8 Scatter-Gather DMA

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008325.jpg)

实际 I/O buffer 很可能不是物理连续的。

**Scatter-Gather（SG）DMA** 允许设备一次处理多个片段。

Linux 里常用 `scatterlist`。

```
int nents;

nents = dma_map_sg(&pdev->dev, sglist, sglen, DMA_TO_DEVICE);
if (nents == 0)
    return -EIO;
```

设备如果支持 SG，就可以读取多个 DMA 段，避免拷贝到一块连续大 buffer。

网卡、存储设备都离不开 SG。没有 SG，性能会差很多，内存分配也更痛苦。

SG 复杂在两个地方。

1.  1\. 一个是描述符格式由硬件定义。你要把 `sg_dma_address()` 和 `sg_dma_len()` 填到设备描述符里。
    
2.  2\. 另一个是错误路径。映射了一半失败、队列提交失败、设备 reset，都要正确 unmap。
    

### 6.9 IOMMU 对 DMA 的影响

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008409.jpg)

**IOMMU** 是 DMA 世界里的内存管理单元。

它能把设备发出的 DMA 地址翻译到真实物理地址，还能做权限隔离。虚拟化、设备直通、安全隔离都靠它。

有 IOMMU 后，设备看到的地址通常是 IOVA，不是真实物理地址。

**这也是为什么驱动必须用 DMA API。** DMA API 会处理底层是否存在 IOMMU、缓存一致性、地址限制等问题。

如果你绕开 DMA API，直接把物理地址塞给设备，在没有 IOMMU 的老平台上可能凑巧能跑。在打开 IOMMU 的服务器上，直接翻车。

驱动代码最怕“我这里能跑”。你这里能跑，不代表用户那里能跑，更不代表生产环境能跑。

### 6.10 DMA 缓存一致性与内存屏障

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008474.jpg)

**缓存一致性**是 DMA 的核心问题。

-   • CPU 写了数据，设备什么时候能看到？
    
-   • 设备写了数据，CPU 什么时候能看到？
    
-   • 描述符状态位谁来更新？
    
-   • doorbell 写寄存器之前，描述符是否已经准备好？
    

这些问题没有处理好，驱动会出现各种玄学 bug。

对 streaming DMA，map/unmap API 会处理必要的缓存同步。某些场景还要用：

```
dma_sync_single_for_device()
dma_sync_single_for_cpu()
```

比如一个 buffer 反复在 CPU 和设备之间切换所有权。

伪代码是这样。

```
/* CPU 准备数据 */
memcpy(buf, data, len);

/* 交给设备前同步 */
dma_sync_single_for_device(&pdev->dev, dma_addr, len, DMA_TO_DEVICE);

/* 通知设备 */
writel(1, ddev->bar0 + REG_START);
```

设备写完后：

```
/* 设备完成，CPU 准备读取前同步 */
dma_sync_single_for_cpu(&pdev->dev, dma_addr, len, DMA_FROM_DEVICE);

process_data(buf);
```

别嫌麻烦。DMA 代码一旦错，调试成本比多写几行同步代码高得多。

## 七、电源管理、热插拔与错误恢复

按理说文章写到 DMA 和 IRQ 就该可以了。

但实际项目里，驱动能加载不算赢。**能长期稳定运行，能 suspend/resume，能热插拔，能 reset 后恢复，才算有点样子。**

PCI 设备不是永远处在完美状态。系统会休眠，设备会掉电，链路会错误，用户会热拔插。你不处理，问题迟早找你。

### 7.1 PCI 电源状态 D0–D3

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008573.jpg)

PCI 设备有电源状态。

-   -   **D0**：正常工作。
-   -   **D1、D2**：中间低功耗状态，不是所有设备支持。
-   -   **D3**：更深的低功耗状态，又分 hot 和 cold。

驱动需要在系统电源管理流程中保存设备状态、停止 DMA、关闭中断、恢复后重新初始化设备。

以前桌面 Linux 休眠唤醒后网卡没了，声卡没了，显卡花屏，很多都和电源管理处理不完善有关。现在成熟多了，但自研 PCIe 设备驱动还是经常踩。

### 7.2 Runtime PM

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008652.jpg)

**Runtime PM** 是运行时电源管理。

设备空闲时进入低功耗状态，需要用时再唤醒。

这对移动设备、嵌入式、低功耗服务器都很重要。对高性能 PCIe 卡，也可能用于节能。

不过我建议新手别一开始就硬上 Runtime PM。先把基本 probe/remove、DMA、IRQ 跑稳。否则你会在“设备到底醒没醒”里迷路。

Runtime PM 的麻烦在于状态交错。用户打开设备、关闭设备、中断到来、工作队列运行、系统 suspend，全都可能影响引用计数和设备状态。写错了很难查。

### 7.3 suspend / resume 回调

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008742.jpg)

系统挂起时，驱动要做几件事。

1.  1\. 停止上层 I/O。
    
2.  2\. 禁止设备继续 DMA。
    
3.  3\. 关闭或屏蔽中断。
    
4.  4\. 保存必要寄存器状态。
    
5.  5\. 让设备进入低功耗。
    

恢复时反过来。

1.  1\. 重新使能设备。
    
2.  2\. 恢复配置空间和私有寄存器。
    
3.  3\. 重新初始化队列。
    
4.  4\. 重新启用中断。
    
5.  5\. 恢复上层 I/O。
    

如果设备硬件 reset 后寄存器全丢，你 resume 里不能只恢复 PCI 配置空间，还得重新写设备私有寄存器。

很多人以为 suspend/resume 只是走个形式。不是。尤其自定义 FPGA PCIe 设备，恢复后内部状态可能干干净净，像刚上电。你不重新配置，它就沉默。

### 7.4 PCI Hotplug 机制

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008845.jpg)

PCIe 支持 **热插拔**，尤其服务器、背板、外接 PCIe 设备场景。

热插拔意味着 remove 可能在运行时发生。

你驱动里如果还有未完成 DMA、未关闭队列、用户进程还在 mmap、workqueue 还没 flush，设备就被拔了。然后呢？

然后你就开始看 oops。

remove 里必须停止硬件，阻止新的 I/O，等待已有工作完成，释放资源。

顺序很重要。

```
阻止新请求
停止设备 DMA
屏蔽中断
同步中断处理
注销上层接口
释放 DMA
解除映射
释放 PCI 资源
```

不能先释放 MMIO，再让工作队列访问寄存器。不能先 free 私有结构体，再让中断 handler 用它。听起来像废话，但线上事故很多就是这种废话引起的。

### 7.5 remove 流程与资源释放

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008920.jpg)

一个带 IRQ 和 DMA 的 remove 大概像这样。

```
static void demo_remove(struct pci_dev *pdev)
{
    struct demo_dev *ddev = pci_get_drvdata(pdev);

    demo_stop_hw(ddev);

    if (ddev->irq >= 0)
        free_irq(ddev->irq, ddev);

    pci_free_irq_vectors(pdev);

    if (ddev->dma_cpu)
        dma_free_coherent(&pdev->dev,
                          ddev->dma_size,
                          ddev->dma_cpu,
                          ddev->dma_handle);

    if (ddev->bar0)
        pci_iounmap(pdev, ddev->bar0);

    pci_release_regions(pdev);
    pci_disable_device(pdev);

    kfree(ddev);
}
```

真实驱动里还要处理 mutex、spinlock、workqueue、timer、kthread、字符设备、网络设备等资源。

我个人习惯是 **probe 申请什么，remove 就按相反顺序释放**。每加一项资源，都在 remove 和错误路径补一项。不要相信自己后面会记得。你不会。加班到晚上十一点时，谁都不会。

### 7.6 AER 错误上报机制

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111008964.jpg)

**AER（Advanced Error Reporting）** 是 PCIe 高级错误上报。PCIe 用它上报链路和事务错误。

比如 Completion Timeout、Unsupported Request、Malformed TLP 等。

高可靠系统里，AER 很重要。设备或者链路出现错误时，系统可以记录、通知、甚至尝试恢复。

驱动可以实现错误处理回调，参与恢复流程。比如 reset\_prepare、slot\_reset、resume 之类的逻辑。

不过这块不建议入门阶段展开太深。你先知道它不是玄学日志就行。`dmesg` 里看到 AER 错误，不要只盯着驱动代码，也要查硬件链路、BIOS 设置、插槽、电源、信号质量。

PCIe 问题有时候真不是软件锅。虽然最后通常还是软件同学先背。

### 7.7 设备 reset 与错误恢复

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009066.jpg)

设备 reset 后，驱动要能重新建立状态。

PCIe Function Level Reset、Hot Reset、Secondary Bus Reset 都可能让设备状态变化。

自定义设备尤其要小心。你 reset 设备后，BAR 还在，但**设备内部寄存器可能回到默认值**。DMA 引擎停止了，队列指针清零了，中断 mask 也变了。

驱动恢复要考虑：

```
停止 I/O
等待 DMA 静止
执行 reset
重新初始化寄存器
重建 DMA 队列
重新启用中断
恢复上层服务
```

reset 做不好，设备会进入半死不活状态。看起来还在 PCI 总线上，`lspci` 也能看到，但业务就是不跑。这个状态比彻底消失还烦。

## 八、PCIe 高级特性

这一章不需要每个点都吃透，但你得知道它们在干嘛。因为现代 PCIe 设备，尤其网卡、GPU、NVMe、DPU、AI 加速卡，很难完全绕开这些高级能力。

### 8.1 PCIe Link、Lane 与速率

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009142.jpg)

**PCIe Link** 由多个 **Lane** 组成。

常见宽度有 x1、x2、x4、x8、x16。每条 Lane 是一对收发差分信号。Lane 越多，带宽越高。

速率从 Gen1、Gen2、Gen3、Gen4、Gen5 到更新代际不断提升。驱动开发者不需要记所有编码细节，但调性能一定要看**实际协商速率和宽度**。

```
lspci -s 03:00.0 -vvv | grep -E "LnkCap|LnkSta"
```

你会看到设备能力和当前状态。

```
LnkCap: Port #0, Speed 16GT/s, Width x8
LnkSta: Speed 8GT/s, Width x4
```

这就说明设备能力是 16GT/s x8，但当前只跑 8GT/s x4。为什么？插槽、主板、BIOS、线缆、Retimer、设备兼容性，都可能。

**软件性能优化前，先确认硬件链路没缩水。** 这个真不是矫情。

### 8.2 Max Payload Size 与 Max Read Request Size

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009216.jpg)

-   -   **Max Payload Size**：决定单个 TLP 最大数据负载。
-   -   **Max Read Request Size**：决定一次 Memory Read 请求最大读多大。

这两个参数会影响 PCIe 传输效率。

-   • Payload 太小，协议开销占比高。
    
-   • Read Request 太小，大块读会拆成很多请求。
    

但也不是越大越好。拓扑中所有设备、Switch、Root Complex 都有限制。配置不当可能引发兼容问题。

一般驱动不随便乱改这些参数，除非你很清楚平台和设备行为。高性能驱动可能会关注它们，但普通驱动先别手痒。

### 8.3 SR-IOV 中的 PF 与 VF

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009305.jpg)

**SR-IOV** 是 PCIe 虚拟化里的重要特性。

一个物理设备可以暴露一个 **PF（Physical Function，物理功能）** 和多个 **VF（Virtual Function，虚拟功能）**。

-   • PF 是完整功能，通常由宿主机驱动管理。
    
-   • VF 是轻量虚拟功能，可以分配给虚拟机或容器使用。
    

网卡是典型场景。一个物理网卡开出多个 VF，每个虚拟机拿一个 VF，性能接近直通。

驱动开发里，PF 驱动和 VF 驱动职责不同。PF 负责资源管理、VF 创建、配置、复位等。VF 驱动只处理自己被分配到的功能。

你写普通 PCIe 设备驱动可以先不支持 SR-IOV。但如果你的设备面向云、虚拟化、DPU，这块迟早要碰。

### 8.4 VFIO 与设备直通

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009389.jpg)

**VFIO** 提供了一套把设备安全暴露给用户态的框架，常用于虚拟机设备直通。

它依赖 IOMMU 做隔离。**没有 IOMMU，用户态直接控制设备 DMA 是非常危险的。** 设备可以 DMA 到任意内存，那还得了。

VFIO 的思路是，把设备放进受 IOMMU 保护的容器里，用户态可以访问设备资源和处理中断，但 DMA 被限制在允许范围内。

这也是为什么现代虚拟化环境里，经常会看到 VFIO、IOMMU group、device passthrough 这些词。

从驱动作者角度看，VFIO 有时意味着你的内核驱动不会绑定设备。设备可能被绑定到 `vfio-pci`。这时候你自己的驱动 probe 不进，不一定是 ID 错，而是设备已经被 VFIO 接管了。

排查时看：

```
lspci -k -s 03:00.0
```

它会告诉你当前 kernel driver 是谁。

### 8.5 ACS、ARI、ATS、PRI、PASID

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009432.jpg)

-   -   **ACS**：用于访问控制和隔离，和 IOMMU group、P2P、虚拟化安全相关。
-   -   **ARI**：扩展 Function 编号能力，让一个设备支持更多 function。
-   -   **ATS**：允许设备缓存地址转换结果，减少 IOMMU 翻译开销。
-   -   **PRI**：允许设备在地址缺页时请求页面。
-   -   **PASID**：用于区分不同进程地址空间，常见于共享虚拟地址场景。

不搞虚拟化和高端加速设备，你可能很少直接写这些。但看到日志和 `lspci -vvv` 输出时，别以为它们是无关噪音。

现代 PCIe 已经不是简单外设总线。它越来越像 CPU、内存、设备、虚拟机之间的高速互联系统。

### 8.6 Peer-to-Peer DMA

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009550.jpg)

**Peer-to-Peer（P2P）DMA** 指一个 PCIe 设备直接和另一个 PCIe 设备传数据，不经过主机内存中转。

比如 NVMe 到 GPU，或者采集卡到 GPU。

听起来很美。少一次拷贝，性能爆炸。

现实呢？

拓扑限制很多。Root Complex、Switch、ACS、IOMMU、驱动支持都影响能不能做。很多平台会把 P2P 流量强制绕回主机，或者根本不支持。

所以 P2P DMA 不是你代码里写个地址就完事。它需要平台、设备、内核框架一起配合。

如果你做高性能数据通路，这块值得研究。如果只是普通 PCIe 控制卡，知道概念就够了。

## 九、调试、测试与性能优化

PCI 驱动调试有个特点。

**你不能只看代码。**

你得看设备是否枚举。  
看配置空间。  
看 BAR。  
看当前绑定的驱动。  
看中断有没有来。  
看 DMA 地址有没有错。  
看链路速率。  
看 sysfs。  
看 dmesg。

只盯着 C 代码，容易把自己绕晕。

### 9.1 `lspci` 常用命令

最常用：

```
lspci
```

看 ID：

```
lspci -nn
```

看详细信息：

```
lspci -vvv -s 03:00.0
```

看拓扑：

```
lspci -t
```

看当前绑定驱动：

```
lspci -k -s 03:00.0
```

如果 probe 不进，我一般第一步就是：

```
lspci -nn -k -s 03:00.0
```

**确认三件事。**

**设备在不在。**  
**Vendor ID 和 Device ID 是什么。**  
**当前绑定的是哪个驱动。**

**别一上来加 printk。先确认设备世界里的事实。**

### 9.2 `setpci` 读写配置空间

`setpci` 可以读写 PCI 配置空间。

读 Vendor ID：

```
setpci -s 03:00.0 00.w
```

读 Command Register：

```
setpci -s 03:00.0 04.w
```

不过写配置空间要非常小心。你可能把设备写挂，甚至影响系统稳定。

调试时可以读，写之前最好确认自己知道每一位含义。**别在生产机器上随便玩。真的会出事。**

### 9.3 `/sys/bus/pci/devices/` 目录分析

sysfs 是 PCI 调试宝库。

设备路径类似：

```
/sys/bus/pci/devices/0000:03:00.0/
```

里面有很多文件。

```
vendor
device
class
resource
config
enable
driver
irq
numa_node
remove
rescan
```

查看 Vendor ID：

```
cat /sys/bus/pci/devices/0000:03:00.0/vendor
```

查看 resource：

```
cat /sys/bus/pci/devices/0000:03:00.0/resource
```

查看 NUMA node：

```
cat /sys/bus/pci/devices/0000:03:00.0/numa_node
```

解绑驱动：

```
echo 0000:03:00.0 > /sys/bus/pci/drivers/demo_pci/unbind
```

绑定驱动：

```
echo 0000:03:00.0 > /sys/bus/pci/drivers/demo_pci/bind
```

重新扫描：

```
echo 1 > /sys/bus/pci/rescan
```

这几个命令调驱动很有用。**但也别在业务机器上乱搞。你 unbind 一个正在工作的网卡，远程连接没了，那场面很安静。**

### 9.4 `dmesg` 与 `dev_err/dev_info`

驱动日志不要全用 `printk()`。

有设备上下文时，用：

```
dev_info(&pdev->dev, "device probed\n");
dev_err(&pdev->dev, "failed to request regions: %d\n", ret);
dev_dbg(&pdev->dev, "bar0 mapped at %p\n", ddev->bar0);
```

这样日志里会带设备信息，调多卡时特别重要。

**你要知道是哪张卡报错。不是“某个 demo\_pci 失败了”。**

看日志：

```
dmesg -w
```

如果日志太多：

```
dmesg | grep demo_pci
```

我的习惯是 probe 每个关键步骤失败都打 `dev_err()`。成功路径别刷屏，关键状态用 `dev_info()`。高频路径不要乱打印，不然性能和日志都炸。

### 9.5 dynamic debug

`dev_dbg()` 默认可能不输出。可以通过 dynamic debug 打开。

```
echo 'file demo_pci.c +p' > /sys/kernel/debug/dynamic_debug/control
```

这比到处改代码加 printk 强多了。

生产环境里也更安全。需要时打开，不需要时关闭。

```
echo 'file demo_pci.c -p' > /sys/kernel/debug/dynamic_debug/control
```

如果你的驱动要长期维护，建议把调试日志分层写好。**别全靠临时 printk。临时东西最后都会变成历史包袱，哈。**

### 9.6 ftrace 跟踪 probe 与 IRQ

ftrace 可以跟踪函数调用。

比如跟踪某个驱动 probe：

```
echo function > /sys/kernel/debug/tracing/current_tracer
echo demo_probe > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

也可以跟 IRQ、调度、workqueue 事件。

PCI 驱动里如果怀疑某个路径没走到，ftrace 很好用。比在每个函数里加日志干净。

### 9.7 perf 分析中断与 DMA 性能

性能问题不要靠感觉。

看中断分布：

```
cat /proc/interrupts
```

看 CPU 消耗：

```
perf top
```

看某个进程：

```
perf record -g -p <pid>
perf report
```

如果中断处理函数、softirq、copy、锁竞争占比很高，你就知道该往哪里看。

DMA 性能也不是只看设备带宽。队列深度、buffer 大小、SG 段数、中断合并、NUMA、cache line、内存分配都会影响。

我以前有次优化采集卡，硬件理论带宽很高，实际吞吐低。最后发现不是 DMA 慢，是每次 DMA 完都打一次中断，CPU 被中断打傻了。开中断合并后，吞吐直接上去了。那一刻我有点想给之前的自己道歉。

### 9.8 PCIe 带宽、DMA 效率与中断合并

性能优化要分层看。

链路层面，看 PCIe 速率和宽度。  
DMA 层面，看 buffer 大小、队列深度、SG、IOMMU 开销。  
中断层面，看中断频率、亲和性、合并策略。  
CPU 层面，看 NUMA、cache miss、锁竞争。

中断合并很常见。设备不必每完成一个包或一个描述符就打断 CPU。可以累计一定数量或延迟一小段时间再中断。

吞吐会上去。  
延迟可能上升。

**这是权衡。**

**网络驱动、存储驱动都离不开这种权衡。别迷信一个参数打天下。低延迟场景和高吞吐场景配置完全不同。**

### 9.9 NUMA 亲和性与 IRQ 绑定

多路服务器上，PCIe 设备挂在哪个 CPU socket 下非常重要。

查看设备 NUMA node：

```
cat /sys/bus/pci/devices/0000:03:00.0/numa_node
```

如果设备在 node 0，但业务线程跑在 node 1，DMA buffer 分配在 node 1，中断也打到 node 1，那数据路径就绕了。

理想情况是：

```
设备所在 NUMA node
   |
本地内存
   |
本地 CPU
   |
本地 IRQ
```

**对网卡、NVMe、GPU 这类高吞吐设备，NUMA 亲和性很关键。**

很多应用层同学看不到这一层。他们只说“驱动性能不行”。你把 IRQ、线程、内存绑对后，性能上去了，他们又说“今天机器状态不错”。听着有点气，但也习惯了。

## 十、实战案例

前面讲了很多概念，现在来串成一个最小驱动。

这个驱动不对应真实硬件，但结构接近真实 PCI 驱动。你可以把它当模板，再根据设备手册填寄存器和业务逻辑。

### 10.1 最小 PCI 驱动模板

```
#include <linux/module.h>
#include <linux/pci.h>
#include <linux/interrupt.h>
#include <linux/dma-mapping.h>

#define DRV_NAME "demo_pci"

#define DEMO_VENDOR_ID 0x1234
#define DEMO_DEVICE_ID 0x5678

#define REG_STATUS     0x00
#define REG_CONTROL    0x04
#define REG_INT_STATUS 0x08
#define REG_INT_CLEAR  0x0c

struct demo_dev {
    struct pci_dev *pdev;
    void __iomem *bar0;

    int irq;

    void *dma_cpu;
    dma_addr_t dma_handle;
    size_t dma_size;
};

static const struct pci_device_id demo_ids[] = {
    { PCI_DEVICE(DEMO_VENDOR_ID, DEMO_DEVICE_ID) },
    { 0, }
};
MODULE_DEVICE_TABLE(pci, demo_ids);
```

这个结构很普通。ID table、私有结构体、寄存器偏移。

**别小看这些基础。一个大驱动也是从这些东西长出来的。**

### 10.2 BAR 映射与寄存器读写示例

probe 里的 BAR 初始化：

```
static int demo_map_bar(struct demo_dev *ddev)
{
    struct pci_dev *pdev = ddev->pdev;
    unsigned long flags;

    flags = pci_resource_flags(pdev, 0);
    if (!(flags & IORESOURCE_MEM)) {
        dev_err(&pdev->dev, "BAR0 is not MMIO\n");
        return -ENODEV;
    }

    ddev->bar0 = pci_iomap(pdev, 0, 0);
    if (!ddev->bar0) {
        dev_err(&pdev->dev, "failed to iomap BAR0\n");
        return -ENOMEM;
    }

    dev_info(&pdev->dev, "BAR0 mapped\n");
    return 0;
}
```

寄存器读写：

```
static void demo_hw_start(struct demo_dev *ddev)
{
    u32 val;

    val = readl(ddev->bar0 + REG_CONTROL);
    val |= 0x1;
    writel(val, ddev->bar0 + REG_CONTROL);

    readl(ddev->bar0 + REG_STATUS);
}

static void demo_hw_stop(struct demo_dev *ddev)
{
    u32 val;

    val = readl(ddev->bar0 + REG_CONTROL);
    val &= ～0x1;
    writel(val, ddev->bar0 + REG_CONTROL);

    readl(ddev->bar0 + REG_STATUS);
}
```

最后那个 `readl()` 是为了让 posted write 刷出去。具体项目里要根据设备手册写。

### 10.3 DMA 缓冲区申请与数据传输示例

初始化 DMA：

```
static int demo_dma_init(struct demo_dev *ddev)
{
    struct pci_dev *pdev = ddev->pdev;
    int ret;

    ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    if (ret) {
        ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
        if (ret) {
            dev_err(&pdev->dev, "no suitable DMA mask\n");
            return ret;
        }
    }

    ddev->dma_size = 4096;
    ddev->dma_cpu = dma_alloc_coherent(&pdev->dev,
                                       ddev->dma_size,
                                       &ddev->dma_handle,
                                       GFP_KERNEL);
    if (!ddev->dma_cpu)
        return -ENOMEM;

    memset(ddev->dma_cpu, 0, ddev->dma_size);

    dev_info(&pdev->dev, "DMA buffer cpu=%p dma=%pad\n",
             ddev->dma_cpu, &ddev->dma_handle);

    return 0;
}
```

释放：

```
static void demo_dma_free(struct demo_dev *ddev)
{
    struct pci_dev *pdev = ddev->pdev;

    if (!ddev->dma_cpu)
        return;

    dma_free_coherent(&pdev->dev,
                      ddev->dma_size,
                      ddev->dma_cpu,
                      ddev->dma_handle);

    ddev->dma_cpu = NULL;
}
```

启动 DMA 的寄存器逻辑要看设备手册。伪代码：

```
#define REG_DMA_ADDR_LO 0x100
#define REG_DMA_ADDR_HI 0x104
#define REG_DMA_LEN     0x108
#define REG_DMA_START   0x10c

static void demo_kick_dma(struct demo_dev *ddev)
{
    writel(lower_32_bits(ddev->dma_handle),
           ddev->bar0 + REG_DMA_ADDR_LO);
    writel(upper_32_bits(ddev->dma_handle),
           ddev->bar0 + REG_DMA_ADDR_HI);
    writel(ddev->dma_size,
           ddev->bar0 + REG_DMA_LEN);

    wmb();

    writel(1, ddev->bar0 + REG_DMA_START);
}
```

**这里写给设备的是 `dma_handle`，不是 `dma_cpu`。再说一遍，因为这个错太常见。**

### 10.4 MSI/MSI-X 中断注册示例

中断处理函数：

```
static irqreturn_t demo_irq(int irq, void *data)
{
    struct demo_dev *ddev = data;
    u32 status;

    status = readl(ddev->bar0 + REG_INT_STATUS);
    if (!status)
        return IRQ_NONE;

    writel(status, ddev->bar0 + REG_INT_CLEAR);

    dev_dbg(&ddev->pdev->dev, "irq status=0x%x\n", status);

    return IRQ_HANDLED;
}
```

申请中断：

```
static int demo_irq_init(struct demo_dev *ddev)
{
    struct pci_dev *pdev = ddev->pdev;
    int ret;

    ret = pci_alloc_irq_vectors(pdev, 1, 1,
                                PCI_IRQ_MSIX |
                                PCI_IRQ_MSI |
                                PCI_IRQ_INTX);
    if (ret < 0) {
        dev_err(&pdev->dev, "failed to alloc irq vectors: %d\n", ret);
        return ret;
    }

    ddev->irq = pci_irq_vector(pdev, 0);

    ret = request_irq(ddev->irq, demo_irq, 0, DRV_NAME, ddev);
    if (ret) {
        dev_err(&pdev->dev, "failed to request irq: %d\n", ret);
        pci_free_irq_vectors(pdev);
        return ret;
    }

    return 0;
}
```

释放：

```
static void demo_irq_free(struct demo_dev *ddev)
{
    if (ddev->irq >= 0)
        free_irq(ddev->irq, ddev);

    pci_free_irq_vectors(ddev->pdev);
}
```

### 10.5 自定义 PCIe 设备驱动完整流程

完整 probe：

```
static int demo_probe(struct pci_dev *pdev,
                      const struct pci_device_id *id)
{
    struct demo_dev *ddev;
    int ret;

    dev_info(&pdev->dev, "probe start\n");

    ddev = kzalloc(sizeof(*ddev), GFP_KERNEL);
    if (!ddev)
        return -ENOMEM;

    ddev->pdev = pdev;
    ddev->irq = -1;
    pci_set_drvdata(pdev, ddev);

    ret = pci_enable_device(pdev);
    if (ret) {
        dev_err(&pdev->dev, "pci_enable_device failed: %d\n", ret);
        goto err_free;
    }

    ret = pci_request_regions(pdev, DRV_NAME);
    if (ret) {
        dev_err(&pdev->dev, "pci_request_regions failed: %d\n", ret);
        goto err_disable;
    }

    pci_set_master(pdev);

    ret = demo_map_bar(ddev);
    if (ret)
        goto err_regions;

    ret = demo_dma_init(ddev);
    if (ret)
        goto err_iounmap;

    ret = demo_irq_init(ddev);
    if (ret)
        goto err_dma;

    demo_hw_start(ddev);

    dev_info(&pdev->dev, "probe success\n");
    return 0;

err_dma:
    demo_dma_free(ddev);
err_iounmap:
    pci_iounmap(pdev, ddev->bar0);
err_regions:
    pci_release_regions(pdev);
err_disable:
    pci_disable_device(pdev);
err_free:
    kfree(ddev);
    return ret;
}
```

remove：

```
static void demo_remove(struct pci_dev *pdev)
{
    struct demo_dev *ddev = pci_get_drvdata(pdev);

    dev_info(&pdev->dev, "remove\n");

    demo_hw_stop(ddev);
    demo_irq_free(ddev);
    demo_dma_free(ddev);

    if (ddev->bar0)
        pci_iounmap(pdev, ddev->bar0);

    pci_release_regions(pdev);
    pci_disable_device(pdev);

    kfree(ddev);
}
```

驱动注册：

```
static struct pci_driver demo_pci_driver = {
    .name     = DRV_NAME,
    .id_table = demo_ids,
    .probe    = demo_probe,
    .remove   = demo_remove,
};

module_pci_driver(demo_pci_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("demo");
MODULE_DESCRIPTION("Demo PCI driver");
```

这就是一个最小但有骨架的 PCI 驱动。

当然，它还不是完整产品。真实驱动还要加字符设备、netdev、block layer、ioctl、mmap、队列、锁、并发保护、电源管理、错误恢复。**但主干已经在了。**

### 10.6 网卡驱动框架拆解

PCI 网卡驱动通常不是只注册 PCI 驱动。probe 里还会分配并注册 `net_device`。

大概流程：

```
PCI probe
   |
pci_enable_device
   |
映射 BAR
   |
设置 DMA mask
   |
分配 net_device
   |
初始化 TX/RX ring
   |
申请 MSI-X
   |
注册 netdev
```

网卡数据路径会涉及 NAPI、skb、DMA mapping、TX/RX ring、中断合并、RSS、多队列。

PCI 在这里承担的是设备发现和资源管理入口。真正网络语义由 Linux 网络子系统接管。

所以别把 PCI 驱动理解成“业务驱动全部内容”。**PCI 只是总线入口。**

### 10.7 NVMe 驱动关键流程

NVMe 也是 PCIe 设备。

它通过 BAR 暴露控制寄存器，通过 DMA 队列提交命令和完成项，通过 MSI-X 通知完成。

核心路径大概是：

```
PCI probe
   |
映射控制寄存器
   |
设置 DMA
   |
创建 admin queue
   |
启用 controller
   |
创建 IO queue
   |
申请 MSI-X
   |
接入 block layer
```

NVMe 的高性能来自队列模型。多个 submission queue 和 completion queue 可以绑定多个 CPU，减少锁竞争，提高并行度。

你看，讲到最后还是那几样。

BAR 访问控制寄存器。  
DMA 搬命令和数据。  
MSI-X 通知完成。  
多队列配合 CPU 并行。

**PCI 驱动的底层逻辑在不同设备里反复出现。**

### 10.8 常见问题排查清单

probe 没进。

```
设备是否存在
Vendor ID / Device ID 是否正确
MODULE_DEVICE_TABLE 是否存在
设备是否已绑定其他驱动
驱动是否加载成功
```

BAR 映射失败。

```
BAR 是否存在
BAR 类型是否是 IORESOURCE_MEM
pci_request_regions 是否失败
资源是否被其他驱动占用
```

MMIO 读写异常。

```
是否调用 pci_enable_device
是否正确 ioremap 或 pci_iomap
offset 是否正确
访问宽度是否符合手册
设备是否处于 reset 或低功耗状态
```

DMA 不工作。

```
是否调用 pci_set_master
DMA mask 是否正确
写给设备的是不是 dma_addr_t
方向是否正确
是否忘记 unmap 或 sync
IOMMU 是否启用
```

中断不来。

```
设备侧中断是否打开
pci_alloc_irq_vectors 是否成功
request_irq 是否成功
设备是否真的产生中断
INTx 是否需要清状态
MSI/MSI-X 是否被平台禁用
```

remove 崩溃。

```
是否先停止硬件
是否还有 DMA 未完成
是否还有 workqueue/timer/kthread 使用私有结构
free_irq 的 dev_id 是否一致
资源释放顺序是否反了
```

性能不达标。

```
PCIe 链路速率和宽度是否正常
DMA buffer 大小是否合理
是否每次小包都触发中断
中断是否集中在一个 CPU
NUMA 是否对齐
IOMMU 是否带来明显开销
是否存在锁竞争
```

**驱动开发很多时候不是你不会写，而是你不知道该往哪查。**

## 附录

PCI 驱动常用 API 速查表

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009602.jpg)

PCI 配置空间标准寄存器图

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009696.jpg)

### BAR、DMA、中断核心概念对照表

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009796.jpg)

**把这几个概念串起来，PCI 驱动主线就清楚了。**

**CPU 通过 BAR 控制设备。**  
**设备通过 DMA 搬数据。**  
**设备通过 IRQ 通知 CPU。**

驱动就是中间那个管家。管资源，管顺序，管状态，管异常。

### INTx、MSI、MSI-X 对比表

![](image/硬核干货！Linux%20PCI%20设备驱动详解（转）/IMG-20260715111009908.jpg)

### lspci / setpci / sysfs 调试命令清单

```
lspci
lspci -nn
lspci -vvv -s 03:00.0
lspci -k -s 03:00.0
lspci -t

setpci -s 03:00.0 00.w
setpci -s 03:00.0 04.w

cat /sys/bus/pci/devices/0000:03:00.0/vendor
cat /sys/bus/pci/devices/0000:03:00.0/device
cat /sys/bus/pci/devices/0000:03:00.0/resource
cat /sys/bus/pci/devices/0000:03:00.0/irq
cat /sys/bus/pci/devices/0000:03:00.0/numa_node

echo 0000:03:00.0 > /sys/bus/pci/drivers/demo_pci/unbind
echo 0000:03:00.0 > /sys/bus/pci/drivers/demo_pci/bind

echo 1 > /sys/bus/pci/rescan
```

如果你只记一条排查线，我建议这样走。

```
lspci 看设备
lspci -nn 看 ID
lspci -k 看绑定驱动
sysfs 看资源
dmesg 看 probe 日志
/proc/interrupts 看中断
DMA 出问题再查 mask、IOMMU、地址和同步
```

**Linux PCI 驱动不是一个 API 问题。它是一条链路。**

> 设备要先被枚举。
> 
> 驱动要能匹配。
> 
> BAR 要能映射。
> 
> 寄存器要能访问。
> 
> DMA 要能搬数据。
> 
> 中断要能通知。
> 
> 异常路径要能收场。

这就是 PCI 驱动。有点烦，也真的有点东西。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/0c897d1b_1783956159345?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247486966%26idx%3D1%26sn%3D2040c9e2306d27772a74767b31820c9c%26chksm%3Dc387d1685a2c081005d935b2003e6fef19c2702eaa927013b5d726421dd4adc428f3681ba552%26mpshare%3D1%26scene%3D1%26srcid%3D0713jCSul0zIHOCq6yAa4H4Q%26sharer_shareinfo%3D09399eb99f57d6a9f8de941cf6d8ae44%26sharer_shareinfo_first%3D09399eb99f57d6a9f8de941cf6d8ae44%23rd&s=obsidian)