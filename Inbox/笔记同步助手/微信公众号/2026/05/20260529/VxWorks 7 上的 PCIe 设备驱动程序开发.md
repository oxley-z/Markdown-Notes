---
author: VxWorks Club
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzIzOTQzOTIxOA==&mid=2247487842&idx=1&sn=f974257a01eb1e4b16bb93d794eff0ce&chksm=e8a076275d5a8981b45b785a1effe5b77e52bad3f327fa1eeb709b1b5c5770457e59a292d41b&mpshare=1&scene=1&srcid=0529uUD4gJiKqGIyZl7rGFjS&sharer_shareinfo=e52bf7737f21959f370eb55f73cf2e4b&sharer_shareinfo_first=e52bf7737f21959f370eb55f73cf2e4b#rd
saved: 2026-05-29 08:35:56
tags:
  - 笔记同步助手
id: 868b113b-31a9-4429-ab46-1e7acdaa7558
---

公众号名称：北南南北

作者名称：VxWorks Club

发布时间：2026-05-29 02:10

原文链接：[https://www.vxworks7.com/bsp/pcie-device-driver-development-on-vxworks-7/](https://www.vxworks7.com/bsp/pcie-device-driver-development-on-vxworks-7/)

> 更多VxWorks文章，请点击"阅读原文"

PCIe 设备驱动程序开发是航空航天、国防、工业自动化、医疗仪器、FPGA 加速以及高速网络等现代嵌入式系统中的核心需求。

在 VxWorks 7 中，Wind River 通过引入 VxBus 2.0 框架、改进的 SMP 支持、设备树（Device Tree）集成以及增强的 DMA 基础设施，使驱动程序开发走向现代化。这些改进显著简化了可扩展且具移植性的 PCIe 驱动程序实现。

本指南提供了在 VxWorks 7 上进行 PCIe 驱动程序开发的全面教程，包括：

-   • PCIe 架构基础
    
-   • VxBus 驱动程序架构
    
-   • PCIe 枚举
    
-   • BAR 映射
    
-   • 中断处理
    
-   • DMA 操作
    
-   • MSI/MSI-X
    
-   • 设备树集成
    
-   • SMP 安全同步
    
-   • 用户空间访问模式
    
-   • 性能优化
    
-   • 完整的代码示例
    

---

## 🧩 PCIe 架构基础

在实现 PCIe 驱动程序之前，理解硬件架构至关重要。

一个 PCIe 端点（Endpoint）设备通常包含以下组件：

| 组件 | 描述 |
| :-- | :-- |
| Vendor ID（厂商 ID） | 制造商标识符 |
| Device ID（设备 ID） | 设备型号标识符 |
| BARs（基地址寄存器） | 用于 MMIO 的基地址寄存器 |
| Configuration Space（配置空间） | PCIe 配置寄存器 |
| MSI/MSI-X | 中断交付机制 |
| DMA Engine（DMA 引擎） | 高速内存传输引擎 |
| PCIe Capabilities（PCIe 能力） | 高级 PCIe 特性支持 |

典型的 PCIe 拓扑结构：

```
CPU
 └── Root Complex（根复合体）
      └── PCIe Switch（PCIe 开关）
            ├── 端点设备 A
            ├── 端点设备 B
            └── FPGA 端点
```

PCIe 通信是基于内存映射和数据包的，并针对低延迟数据传输进行了高度优化。高性能应用几乎总是依赖 DMA，而不是程序控制 I/O（PIO）。

---

## 🏗️ VxWorks 7 驱动程序架构

VxWorks 7 使用 VxBus 框架进行驱动程序开发。

传统的 VxWorks 与 BSP 紧密耦合的驱动程序难以扩展和维护。VxBus 引入了更清晰的抽象模型，具有以下特点：

-   • 动态设备探测（Probing）
    
-   • 可移植的驱动程序架构
    
-   • SMP 安全的初始化
    
-   • 设备树支持
    
-   • 统一的资源管理
    
-   • 标准化的驱动程序注册
    

一个典型的 VxBus PCIe 驱动程序生命周期包括：

```
Probe()（探测）
Attach()（挂载）
Interrupt Service Routine()（中断服务程序）
DMA Handling（DMA 处理）
Detach()（卸载）
```

VxBus 模型使得驱动程序可以在多个 BSP 和硬件平台之间复用。

---

## 📁 PCIe 驱动程序源码布局

一个常见的 VxWorks PCIe 驱动程序目录结构：

```
myPcieDrv/
├── myPcieDrv.c
├── myPcieDrv.h
├── Makefile
├── component.cdf
└── hwconf.c
```

对于较大的项目，通常会将以下内容分离到专属的模块中：

-   • DMA 处理
    
-   • ISR 管理
    
-   • 寄存器访问
    
-   • 用户 API
    
-   • 设备树解析
    

---

## 📚 所需的头文件

典型的 PCIe 驱动程序需要以下头文件：

```
#include 
#include 
#include 
#include 

#include 
#include 
#include 

#include 
#include 
#include 
#include 

#include 
#include 
```

这些头文件提供了对以下内容的访问支持：

-   • VxBus 基础设施
    
-   • PCIe 配置 API
    
-   • 同步原语
    
-   • 中断管理
    
-   • DMA 缓存操作
    

---

## 🧠 设备上下文结构体

每个 PCIe 设备实例都需要一个软件上下文结构体。

```
typedef struct
{
    VXB_DEV_ID     pDev;

    void * bar0Base;
    void * bar1Base;

    VXB_RESOURCE * pResBar0;
    VXB_RESOURCE * pResBar1;
    VXB_RESOURCE * pResIrq;

    UINT32         irq;

    SEM_ID         dmaSem;
    SEM_ID         devSem;

    void * dmaBuffer;
    PHYS_ADDR      dmaPhys;

    UINT32         dmaSize;

} MY_PCIE_CTRL;
```

该结构体维护了：

-   • BAR 映射
    
-   • IRQ 资源
    
-   • DMA 缓冲区
    
-   • 同步对象
    
-   • 设备特定的运行时状态
    

软件上下文通常使用以下方式存储：

```
vxbDevSoftcSet(pDev, pCtrl);
```

---

## 🔍 PCIe 设备识别

假设 FPGA 端点使用以下 PCIe 标识符：

```
#define MY_VENDOR_ID    0x1234
#define MY_DEVICE_ID    0x5678
```

探测（probe）程序使用这些值来确定驱动程序是否与硬件匹配。

---

## ⚙️ Probe 函数实现

探测函数用于验证 PCIe 配置空间的信息。

```
LOCAL STATUS myPcieProbe
    (
    VXB_DEV_ID pDev
    )
{
    UINT16 vendorId;
    UINT16 deviceId;

    vxbPciConfigRead16(pDev,
                       PCI_CFG_VENDOR_ID,
                       &vendorId);

    vxbPciConfigRead16(pDev,
                       PCI_CFG_DEVICE_ID,
                       &deviceId);

    if ((vendorId == MY_VENDOR_ID) &&
        (deviceId == MY_DEVICE_ID))
    {
        printf("PCIe device matched\n");
        return OK;
    }

    return ERROR;
}
```

探测程序应当保持轻量，并避免分配资源。

---

## 🗂️ BAR 映射和 MMIO 访问

PCIe BAR 暴露了内存映射的硬件区域。

BAR 使用示例：

| BAR | 用途 |
| :-- | :-- |
| BAR0 | 控制寄存器 |
| BAR1 | DMA 引擎 |
| BAR2 | 共享内存 |

BAR 映射示例：

```
LOCAL STATUS myMapBars
    (
    MY_PCIE_CTRL * pCtrl
    )
{
    pCtrl->pResBar0 =
        vxbResourceAlloc(pCtrl->pDev,
                         VXB_RES_MEMORY,
                         0);

    if (pCtrl->pResBar0 == NULL)
        return ERROR;

    pCtrl->bar0Base =
        (void *)vxbResourceVirtAdrsGet(
                    pCtrl->pResBar0);

    printf("BAR0 = %p\n", pCtrl->bar0Base);

    return OK;
}
```

未能正确映射 BAR 通常会导致从寄存器读取到：

```
0xFFFFFFFF
```

的数据。

---

## 🧾 寄存器访问宏

寄存器访问宏可以简化 MMIO 操作：

```
#define REG_READ32(base, offset) \
    (*(volatile UINT32 *)((UINT8 *)(base) + (offset)))

#define REG_WRITE32(base, offset, value) \
    (*(volatile UINT32 *)((UINT8 *)(base) + (offset)) = (value))
```

示例寄存器映射：

```
#define REG_STATUS         0x00
#define REG_CONTROL        0x04
#define REG_DMA_SRC        0x08
#define REG_DMA_DST        0x0C
#define REG_DMA_SIZE       0x10
#define REG_DMA_START      0x14
#define REG_INT_STATUS     0x18
#define REG_INT_ENABLE     0x1C
```

保持寄存器定义的集中化可以提高可维护性和硬件可移植性。

---

## 🚀 设备初始化

设备初始化在 BAR 映射和中断设置完成后配置硬件状态。

```
LOCAL STATUS myDeviceInit
    (
    MY_PCIE_CTRL * pCtrl
    )
{
    REG_WRITE32(pCtrl->bar0Base,
                REG_CONTROL,
                0x1);

    REG_WRITE32(pCtrl->bar0Base,
                REG_INT_ENABLE,
                0x1);

    return OK;
}
```

初始化通常包括：

-   • 复位控制
    
-   • DMA 引擎初始化
    
-   • 启用中断
    
-   • 清空 FIFO
    
-   • 链路验证
    

---

## ⚡ 中断处理

PCIe 设备支持几种中断模型：

-   • 传统的 INTx
    
-   • MSI
    
-   • MSI-X
    

在现代 SMP 系统中，强烈推荐使用策 MSI 和 MSI-X，因为它们避免了中断共享的限制。

ISR 示例：

```
LOCAL void myPcieIsr
    (
    void * arg
    )
{
    MY_PCIE_CTRL * pCtrl =
        (MY_PCIE_CTRL *)arg;

    UINT32 status;

    status = REG_READ32(pCtrl->bar0Base,
                        REG_INT_STATUS);

    REG_WRITE32(pCtrl->bar0Base,
                REG_INT_STATUS,
                status);

    if (status & 0x1)
    {
        semGive(pCtrl->dmaSem);
    }
}
```

ISR 应当保持精简，并将繁重的处理工作延迟到工作任务（worker tasks）中执行。

---

## 🔌 中断注册

中断资源通过 VxBus API 进行分配。

```
LOCAL STATUS mySetupInterrupt
    (
    MY_PCIE_CTRL * pCtrl
    )
{
    pCtrl->pResIrq =
        vxbResourceAlloc(pCtrl->pDev,
                         VXB_RES_IRQ,
                         0);

    if (pCtrl->pResIrq == NULL)
        return ERROR;

    vxbIntConnect(pCtrl->pDev,
                  pCtrl->pResIrq,
                  myPcieIsr,
                  pCtrl);

    vxbIntEnable(pCtrl->pDev,
                 pCtrl->pResIrq);

    return OK;
}
```

在卸载和热插拔移除期间，正确的中断清理同样重要。

---

## 📦 DMA 基础

在高带宽系统中，基于 PIO 的传输会成为瓶颈。

PCIe DMA 工作流程：

```
CPU 分配缓冲区
    ↓
物理地址发送给 FPGA
    ↓
FPGA 执行 DMA
    ↓
产生中断
    ↓
驱动程序唤醒任务
```

DMA 在以下场景中是必不可少的：

-   • FPGA 加速
    
-   • 高速网络
    
-   • 视频流水线
    
-   • 存储系统
    
-   • 数据采集平台
    

---

## 🧮 DMA 缓冲区分配

DMA 缓冲区必须是缓存安全的（cache-safe）且物理上可访问。

```
LOCAL STATUS myAllocDma
    (
    MY_PCIE_CTRL * pCtrl
    )
{
    pCtrl->dmaSize = 0x10000;

    pCtrl->dmaBuffer =
        cacheDmaMalloc(pCtrl->dmaSize);

    if (pCtrl->dmaBuffer == NULL)
        return ERROR;

    pCtrl->dmaPhys =
        CACHE_DMA_VIRT_TO_PHYS(
            pCtrl->dmaBuffer);

    printf("DMA virt=%p phys=0x%llx\n",
           pCtrl->dmaBuffer,
           (unsigned long long)pCtrl->dmaPhys);

    return OK;
}
```

DMA 缓冲区通常应当满足：

-   • 缓存行对齐（cache-line aligned）
    
-   • 页对齐（page aligned）
    
-   • 预先分配
    
-   • 尽可能复用
    

---

## 🔄 启动 DMA 传输

DMA 传输示例：

```
LOCAL STATUS myStartDma
    (
    MY_PCIE_CTRL * pCtrl
    )
{
    cacheFlush(DATA_CACHE,
               pCtrl->dmaBuffer,
               pCtrl->dmaSize);

    REG_WRITE32(pCtrl->bar0Base,
                REG_DMA_DST,
                (UINT32)pCtrl->dmaPhys);

    REG_WRITE32(pCtrl->bar0Base,
                REG_DMA_SIZE,
                pCtrl->dmaSize);

    REG_WRITE32(pCtrl->bar0Base,
                REG_DMA_START,
                1);

    return OK;
}
```

在执行向外（outbound）的 DMA 之前：

```
cacheFlush()
```

必须被执行以确保内存一致性。

---

## ⏳ 等待 DMA 完成

DMA 的完成通常依赖于中断驱动的同步。

```
LOCAL STATUS myWaitDma
    (
    MY_PCIE_CTRL * pCtrl
    )
{
    if (semTake(pCtrl->dmaSem,
                sysClkRateGet() * 5)
        == ERROR)
    {
        printf("DMA timeout\n");
        return ERROR;
    }

    cacheInvalidate(DATA_CACHE,
                    pCtrl->dmaBuffer,
                    pCtrl->dmaSize);

    return OK;
}
```

在执行向内（inbound）的 DMA 之后：

```
cacheInvalidate()
```

可确保废弃过期的缓存行。

---

## 🧱 完整的 Attach 程序

挂载（attach）程序负责初始化所有的驱动程序资源。

```
LOCAL STATUS myPcieAttach
    (
    VXB_DEV_ID pDev
    )
{
    MY_PCIE_CTRL * pCtrl;

    pCtrl = vxbMemAlloc(sizeof(MY_PCIE_CTRL));

    if (pCtrl == NULL)
        return ERROR;

    memset(pCtrl, 0, sizeof(*pCtrl));

    pCtrl->pDev = pDev;

    vxbDevSoftcSet(pDev, pCtrl);

    pCtrl->dmaSem =
        semBCreate(SEM_Q_FIFO,
                   SEM_EMPTY);

    pCtrl->devSem =
        semMCreate(SEM_Q_PRIORITY |
                   SEM_INVERSION_SAFE);

    if (myMapBars(pCtrl) != OK)
        return ERROR;

    if (myAllocDma(pCtrl) != OK)
        return ERROR;

    if (mySetupInterrupt(pCtrl) != OK)
        return ERROR;

    if (myDeviceInit(pCtrl) != OK)
        return ERROR;

    printf("PCIe driver attached\n");

    return OK;
}
```

生产级驱动程序还应该包括用于错误处理的健壮的清理路径。

---

## 🛠️ 驱动程序注册

VxBus 驱动程序通过驱动程序表注册其方法。

```
LOCAL VXB_DRV_METHOD myMethods[] =
{
    { VXB_DEVMETHOD_CALL(vxbDevProbe),
      (FUNCPTR)myPcieProbe },

    { VXB_DEVMETHOD_CALL(vxbDevAttach),
      (FUNCPTR)myPcieAttach },

    VXB_DEVMETHOD_END
};

LOCAL VXB_DRV myPcieDrv =
{
    { NULL },
    "myPcieDrv",
    "Custom PCIe Driver",
    VXB_BUSID_PCI,
    0,
    0,
    myMethods,
    NULL
};

VXB_DRV_DEF(myPcieDrv)
```

此结构使 PCIe 枚举过程中能够自动发现驱动程序。

---

## 🌲 设备树集成

VxWorks 7 支持基于扁平设备树（FDT）的硬件配置。

DTS 节点示例：

```
pcie@0x80000000
{
    compatible = "vendor,my-pcie";
    reg = <0x80000000 0x1000>;
    interrupts = <32>;
};
```

设备树集成简化了：

-   • 硬件可移植性
    
-   • BSP 维护
    
-   • 多平台支持
    

---

## 🔒 SMP 同步

现代嵌入式系统通常是多核的。

潜在的 SMP 问题包括：

-   • 并发寄存器访问
    
-   • 中断竞争
    
-   • DMA 所有权冲突
    
-   • 共享缓冲区损坏
    

互斥锁示例：

```
semTake(pCtrl->devSem, WAIT_FOREVER);

/* 临界区 */

semGive(pCtrl->devSem);
```

VxBus 专门针对支持 SMP 安全的驱动程序开发而设计。

---

## 🧭 PCIe 配置空间访问

驱动程序经常需要直接访问 PCIe 配置空间。

```
UINT16 command;

vxbPciConfigRead16(pDev,
                   PCI_CFG_COMMAND,
                   &command);

command |= PCI_CMD_MASTER_ENABLE;

vxbPciConfigWrite16(pDev,
                    PCI_CFG_COMMAND,
                    command);
```

典型的配置包括启用：

-   • 总线主控（Bus mastering）
    
-   • 内存解码
    
-   • 中断交付
    

---

## 📡 MSI 启用验证

基础 PCIe 状态检查示例：

```
UINT16 status;

vxbPciConfigRead16(pDev,
                   PCI_CFG_STATUS,
                   &status);

printf("PCI status = 0x%x\n", status);
```

调试 MSI 问题时，请验证：

-   • 存在 MSI 能力（capability）
    
-   • 中断向量分配
    
-   • PCIe 命令寄存器配置
    
-   • 中断屏蔽状态
    

---

## 🖥️ 用户空间访问接口

应用程序往往需要对设备寄存器或 DMA 缓冲区进行受控访问。

辅助 API 示例：

```
STATUS myReadReg
    (
    MY_PCIE_CTRL * pCtrl,
    UINT32 offset,
    UINT32 * value
    )
{
    *value = REG_READ32(pCtrl->bar0Base,
                        offset);

    return OK;
}
```

生产系统通常暴露：

-   • IOCTL 接口
    
-   • 共享内存通道
    
-   • 零拷贝缓冲区
    
-   • 消息队列
    

---

## 🐞 PCIe 驱动程序调试

实用的 VxWorks Shell 命令：

```
-> vxbDevShow
-> vxbPciShow
-> devs
-> i
```

调试日志输出示例：

```
printf("BAR0=%p IRQ=%d\n",
       pCtrl->bar0Base,
       pCtrl->irq);
```

WindView 可以帮助分析：

-   • ISR 延迟
    
-   • 调度行为
    
-   • DMA 时序
    
-   • SMP 竞态冲突
    
-   • 中断风暴
    

---

## ⚠️ 常见的 PCIe 驱动程序问题

| 问题 | 典型原因 |
| :-- | :-- |
| BAR 读取返回 `0xFFFFFFFF` | BAR 未映射 |
| DMA 数据损坏 | 缓存一致性问题 |
| ISR 从不触发 | MSI 未启用 |
| 系统挂起/死机 | 无效的 DMA 地址 |
| 枚举失败 | 错误的 Vendor/Device ID |
| SMP 竞态条件 | 缺少同步机制 |

大多数 PCIe 驱动程序故障都与同步、DMA 一致性或资源初始化顺序有关。

---

## 🚄 PCIe 性能优化

### 使用 DMA

基于 PIO 的传输会严重限制吞吐量。

### 优先选择 MSI-X

MSI-X 在多核系统中提供了更好的可扩展性。

### 对齐 DMA 缓冲区

```
memalign(64, size);
```

### 批量 DMA 传输

大块的 DMA 数据块可以显著提高吞吐效率。

### 降低中断频率

在高速率吞吐系统中，中断合并（Interrupt coalescing）可以提高 CPU 利用率。

---

## 🧬 FPGA PCIe 系统架构

典型的 FPGA PCIe 集成：

```
VxWorks CPU
    ↓
PCIe 根复合体（Root Complex）
    ↓
FPGA 端点（Endpoint）
    ├── DMA 引擎
    ├── 控制寄存器
    ├── DDR 缓冲区
    └── 中断发生器
```

高性能 FPGA 系统利用优化的 DMA 架构，可以实现每秒数百 MB 或数 GB 的吞吐量。

---

## 🧵 推荐的驱动程序设计模式

推荐的架构：

```
ISR
 ↓
信号量
 ↓
工作任务（Worker Task）
 ↓
DMA 完成
 ↓
应用程序通知
```

该模型在保持确定性行为的同时，将 ISR 延迟降至最低。

---

## 👷 工作任务（Worker Task）示例

```
LOCAL void myWorkerTask
    (
    MY_PCIE_CTRL * pCtrl
    )
{
    while (1)
    {
        semTake(pCtrl->dmaSem,
                WAIT_FOREVER);

        printf("DMA completed\n");

        /* 处理数据 */
    }
}
```

工作任务应当处理：

-   • DMA 后处理
    
-   • 缓冲区管理
    
-   • 应用程序通知
    
-   • 重试处理
    

---

## 🧠 缓存一致性管理

DMA 和 CPU 缓存必须保持同步。

在 DMA 输出（OUT）之前：

```
cacheFlush(DATA_CACHE, buffer, size);
```

在 DMA 输入（IN）之后：

```
cacheInvalidate(DATA_CACHE, buffer, size);
```

缓存一致性 Bug 是最难诊断的 PCIe 驱动程序问题之一。

---

## 🔌 热插拔支持

PCIe 支持运行时的设备插入和移除。

驱动程序应当妥善处理：

-   • 设备消失
    
-   • 中断销毁
    
-   • DMA 关闭
    
-   • 资源释放
    
-   • 任务终止
    

不完整的清理往往会导致内核不稳定。

---

## 🛡️ PCIe 安全性考虑

PCIe 设备具有直接内存访问能力。

驱动程序应当验证：

-   • DMA 大小
    
-   • DMA 地址范围
    
-   • 用户请求
    
-   • 中断源
    
-   • 寄存器访问
    

永远不要假定端点硬件是完全可信的。

---

## 🧪 高级 PCIe 主题

高级 VxWorks PCIe 特性包括：

-   • SR-IOV
    
-   • 分散-聚集（Scatter-gather）DMA
    
-   • MSI-X 向量表
    
-   • NUMA 感知 DMA
    
-   • IOMMU 集成
    
-   • 零拷贝网络
    
-   • 点对点（Peer-to-peer）PCIe
    
-   • 共享内存传输
    

这些能力在高性能多核系统中变得越来越重要。

---

## 🧰 示例 Makefile

```
CPU=ARMARCH8
TOOL=gnu

OBJS = myPcieDrv.o

all:
    $(CC) -c myPcieDrv.c
```

较大的项目通常会与 VxWorks 构建系统和组件框架（Component Framework）进行集成。

---

## 📌 结论

在 VxWorks 7 上进行 PCIe 设备驱动程序开发结合了多个学科：

-   • 实时系统工程
    
-   • 软硬件集成
    
-   • 中断架构
    
-   • DMA 优化
    
-   • SMP 同步
    
-   • 底层内存管理
    

与传统的与 BSP 耦合的驱动程序模型相比，VxBus 2.0 提供了一个明显更清晰、更具扩展性的架构。

对于高性能 FPGA、网络、存储和工业系统，掌握 DMA、中断处理和同步是构建生产级 PCIe 解决方案的核心。

推荐的学习路径是：

1.  1\. BAR 访问
    
2.  2\. 中断处理
    
3.  3\. DMA 传输
    
4.  4\. MSI/MSI-X
    
5.  5\. SMP 同步
    
6.  6\. 分散-聚集（Scatter-gather）DMA
    
7.  7\. 零拷贝架构
    
8.  8\. 多设备扩展
    

一旦掌握了这些概念，开发人员就能够构建出在现代嵌入式平台上维持极高吞吐量的、具确定性且低延迟的 PCIe 系统。

  

---

![[images/12500ec49da506de5a78e409056591bf_MD5.jpg|cover_image]]

原创 VxWorks Club 北南南北

阅读原文

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/8afa8bac_1780014954153?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIzOTQzOTIxOA%3D%3D%26mid%3D2247487842%26idx%3D1%26sn%3Df974257a01eb1e4b16bb93d794eff0ce%26chksm%3De8a076275d5a8981b45b785a1effe5b77e52bad3f327fa1eeb709b1b5c5770457e59a292d41b%26mpshare%3D1%26scene%3D1%26srcid%3D0529uUD4gJiKqGIyZl7rGFjS%26sharer_shareinfo%3De52bf7737f21959f370eb55f73cf2e4b%26sharer_shareinfo_first%3De52bf7737f21959f370eb55f73cf2e4b%23rd&s=obsidian)