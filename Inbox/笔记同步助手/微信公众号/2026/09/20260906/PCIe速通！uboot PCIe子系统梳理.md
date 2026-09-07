---
author: Surest
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwOTUxMzQzOA==&mid=2247484405&idx=1&sn=2a4109c629876d7f91d694103fc522ce&chksm=c0ee4e5289f28c1e03da49b865051c464b6c6ac6490d180c939a7c156318b19cfd88bd7744c5&mpshare=1&scene=1&srcid=0906JdlTGrdSrJOB4BYhcnum&sharer_shareinfo=2a2f872376b9660807b7acb1a73062a3&sharer_shareinfo_first=2a2f872376b9660807b7acb1a73062a3#rd
saved: 2026-09-06 12:39:31
tags:
  - 笔记同步助手
id: 775e9276-4729-4f99-b8a0-42ada31dcb37
---

公众号名称：摸鱼的日记本

作者名称：Surest

发布时间：2026-08-05 19:11

> 之前的文章按源码视角拆过 SPL/TPL、环境变量、bootm/booti/bootelf、DM 模型、autoboot、Flash 子系统、文件系统、USB。这一篇拆 U-Boot 里的 **PCIe 子系统**：控制器初始化、深度优先枚举、BAR 静态分配、驱动匹配绑定，以及它们跟具体控制器驱动的挂钩方式。代码基于 **U-Boot v2024.07**。平台适配部分以常见的 DesignWare PCIe 控制器为落地例子。

---

## 一、概述

### 1.1 PCIe 子系统要解决的问题

U-Boot 里 PCIe 最常见的三个场景：接 NVMe SSD 做启动盘，接 PCIe 网卡做网络启动，接 PCIe USB 控制器扩展外设。跟 Linux 内核完整 PCIe 子系统不同，U-Boot 只做最核心的四件事：

-   • 初始化主机控制器，建立配置空间映射
    
-   • 枚举总线上的所有设备，建好 `udevice` 树
    
-   • 给每个设备的 BAR 分配地址空间
    
-   • 按 Vendor ID / Device ID / Class Code 匹配合适的 U-Boot 驱动
    

U-Boot 砍掉了所有非启动必须的特性：热插拔（启动时一次枚举完就不再扫）、MSI/MSI-X 中断（大部分控制器用 polling）、AER 错误处理、SR-IOV、PCIe 电源管理。代码精简到只有 Linux PCIe 的十分之一不到，走读起来非常清晰。

### 1.2 U-Boot PCIe 和 Linux 的核心差异

| 特性 | U-Boot | Linux |
| --- | --- | --- |
| 枚举方式 | 启动时一次枚举完，不支持热插拔 | 运行时动态枚举，支持热插拔 |
| 中断 | 大部分控制器 polling 模式，不用 MSI/MSI-X | 完整中断支持，MSI/MSI-X 是标配 |
| BAR 分配 | 一次性静态分配，不支持运行时重映射 | 动态分配/重映射，考虑碎片整理 |
| 地址空间 | 从 controller 的 `mem_start` 顺序往后塞 | 从 `[mem, bus_addr]` 资源树中按需分配 |
| 错误处理 | 简单错误检查，有错直接打日志返回 | 完整 AER 高级错误报告与恢复 |
| 高级特性 | 几乎不支持 SR-IOV / ASPM / 电源管理 | 全支持 |
| 代码量 | ～3000 行（含所有控制器驱动） | ～50万行 |

U-Boot 的设计目标就是"能认出设备，能做 IO 读 kernel"就行。复杂特性全砍，但核心四步（初始化 → 枚举 → 分配 BAR → 匹配驱动）跟 Linux 逻辑完全一致，看完 U-Boot 版再去看 Linux 版，思路是通的。

### 1.3 一分钟速览

PCIe 子系统的关键源码入口：

-   -   **uclass 框架**：`drivers/pci/pci-uclass.c` — 枚举引擎、BAR 分配、驱动匹配
-   -   **核心公共函数**：`drivers/pci/pci.c` — 配置空间读写、总线号管理
-   -   **DesignWare 通用驱动**：`drivers/pci/pcie_dw.c` — 大多数 ARM SoC 的 PCIe IP
-   -   **厂商控制器驱动**：`drivers/pci/pcie_layerscape.c` / `pcie_tegra.c` / `pcie_mediatek.c` 等
-   -   **公共头文件**：`include/pci.h` — 核心数据结构、寄存器定义、枚举常量

后面的章节按这条主线展开：先看源码目录怎么组织的，再看核心数据结构，再看框架分层怎么把控制器和驱动串起来，最后落到具体控制器的适配。

### 1.4 U-Boot 为什么"减负"了 Linux PCIe

熟悉 Linux 的读者会立刻发现：U-Boot 的 PCIe 层是 Linux `drivers/pci/` 的极简版。同样的 `pci_controller`、同样的 256 字节配置空间、同样的 Type 0 / Type 1 Header、同样的深度优先枚举算法——但 U-Boot 少掉了非常多的东西：

-   -   **没有 resource tree**。Linux 里 `struct resource` 构建了一棵 I/O + MEM + BUS 的资源树，BAR 分配是往树里插节点，支持动态冲突检测和重分配。U-Boot 直接一个 `mem_start` + `mem_size`，从前往后塞，没有冲突检测也没有重分配。
-   -   **没有 hotplug 框架**。Linux 的 `pciehp` / `acpiphp` / `shpchp` 三套热插拔驱动全部拿掉。U-Boot 只在 `pci_init` 调一次 `pci_hose_scan_bus`，扫完就完了。
-   -   **没有 MSI 中断管理**。Linux 的 `msi.c` / `msi-irqdomain.c` 管理整个 MSI 中断域。U-Boot 绝大部分驱动是 polling 模式，MSI 寄存器虽然硬件上有，但驱动里不读不写。
-   -   **没有 AER / DPC 错误处理**。PCIe Advanced Error Reporting 和 Downstream Port Containment 是 Linux 里很重的两个模块，U-Boot 里完全不存在。配置空间写错了就是错了，打一行 `printf` 就结束。
-   -   **没有 ASPM / PM 电源管理**。Active State Power Management 让 Linux 的 PCIe 链路在空闲时切到 L0s/L1 省电。U-Boot 不需要省电，链路一直拉在 L0 全速。

减负的**动机**很直接：bootloader 的任务是把 kernel 从存储搬到 DRAM，搬完就交棒退场。PCIe 设备的寿命在 U-Boot 阶段总共就几秒钟，热插拔、省电、错误恢复这些全是浪费代码空间。所以 U-Boot PCIe 只保留了**能让上层认出设备、读写 BAR** 的最小契约——`read_config` / `write_config` 两个函数指针，加上一份总线号和 BAR 空间的描述。

---

## 二、源码边界：目录结构与控制器分类

### 2.1 `drivers/pci/` — PCIe 子树

```
drivers/pci/
├── pci-uclass.c         // PCIe uclass 核心：枚举引擎、BAR 分配、驱动匹配（约 1200 行）
├── pci.c                // 配置空间读写、总线号管理、公共辅助函数（约 400 行）
├── pci_rom.c            // Option ROM 支持（x86 传统，ARM 侧几乎不用）
├── pci_compat.c         // 老式 pci_read_config_* 兼容接口
├── pcie_dw.c            // DesignWare PCIe 控制器通用层（约 600 行）
├── pcie_dw_common.c     // DesignWare 公共寄存器操作
├── pcie_dw_meson.c      // Amlogic Meson DesignWare 定制
├── pcie_dw_rockchip.c   // Rockchip DesignWare 定制
├── pcie_dw_ti.c         // TI Keystone DesignWare 定制
├── pcie_layerscape.c    // NXP Layerscape 独立控制器
├── pcie_layerscape_gen4.c // Layerscape Gen4
├── pcie_tegra.c         // NVIDIA Tegra 控制器
├── pcie_mediatek.c      // MediaTek 控制器
├── pcie_brcmstb.c       // Broadcom STB 控制器
├── pcie_xilinx.c        // Xilinx AXI PCIe Bridge
└── ...                   // 更多厂商控制器
```

核心文件只有两个：`pci-uclass.c` 和 `pci.c`。所有控制器的差异封装在 `read_config` / `write_config` 两个回调里，上层枚举逻辑不感知硬件细节。

### 2.2 控制器驱动分类

按 IP 来源，控制器的接入方式可以分为两类：

**第一类：DesignWare 系（占比最高）**

Synopsys DesignWare PCIe 是 ARM SoC 里最普遍的 PCIe IP。`pcie_dw.c` 提供通用层：DBI（Data Bus Interface）寄存器读写、ATU（Address Translation Unit）配置、链路训练。各家厂商定制驱动只需要实现：

-   -   `dw_pcie_ops.read_dbi` / `write_dbi` — DBI 空间的访问方式（有的走内存映射，有的走间接寄存器）
-   • 平台的 clock / reset / PHY 初始化
    
-   • DT binding 里 `compatible` 的匹配
    

**第二类：自研控制器**

像 NXP Layerscape、NVIDIA Tegra 这些有自己的 PCIe IP，不走 DesignWare 通用层，直接实现 `dm_pci_ops.read_config` / `write_config`。

无论哪一类，对 uclass 来说接口完全一样：就是 `struct dm_pci_ops` 里的几个函数指针。

### 2.3 `include/pci.h` — 公共头文件

```
include/pci.h           // struct pci_controller、struct pci_device_id、寄存器偏移宏
include/pci_ids.h       // Vendor ID / Device ID 宏定义（Intel/NVIDIA/Broadcom 等）
```

`pci.h` 定义了整个 PCIe 子系统的核心数据结构。后续章节会逐一展开。

---

## 三、核心数据结构

### 3.1 `struct pci_controller` — 主机控制器实例

每个 PCIe 主机控制器一个实例，管理一条 PCIe 总线域的根：

```
// include/pci.h（简化）
struct pci_controller {
    struct udevice *bus;         // 指向根总线对应的 udevice
    struct list_head list;        // 全局控制器链表节点

    /* 配置空间访问方法 — 所有控制器差异封装在这两个函数指针里 */
    int (*read_config)(const struct udevice *dev, int offset,
                       ulong *valuep, enum pci_size_t size);
    int (*write_config)(const struct udevice *dev, int offset,
                        ulong value, enum pci_size_t size);

    /* BAR 分配的地址空间范围 — 从 DT 的 ranges 属性解析出来 */
    phys_addr_t mem_start;       // Memory Space 起始物理地址
    phys_addr_t mem_size;        // Memory Space 大小
    phys_addr_t io_start;        // I/O Space 起始物理地址（ARM 侧通常不用）
    phys_addr_t io_size;         // I/O Space 大小

    /* 总线号范围 */
    int first_busno;             // 根总线号，一般是 0
    int last_busno;              // 最大可分配总线号
    int current_busno;           // 当前分配到哪个总线号

    /* 设备链表 */
    struct list_head devices;    // 挂在控制器下的所有设备
};
```

核心就是 `read_config` / `write_config` 两个方法。所有控制器——不管是 DesignWare 还是自研——差异只在这两个函数的实现里。上层枚举逻辑（`pci_hose_scan_bus`）完全不关心你的控制器是哪种 IP，它只调 `read_config(dev, PCI_VENDOR_ID, &val, PCI_SIZE_16)` 读 Vendor ID。

### 3.2 `struct pci_device_id` — 驱动匹配表

跟 Linux 几乎一样，驱动定义自己的匹配表，枚举到设备就按表匹配：

```
// include/pci.h
struct pci_device_id {
    unsigned int vendor;         // Vendor ID，如 0x144d (Samsung)、0x8086 (Intel)
    unsigned int device;         // Device ID
    unsigned int subvendor;      // Subsystem Vendor ID
    unsigned int subdevice;      // Subsystem Device ID
    unsigned int class;          // Class Code，如 0x010802 (NVMe)
    unsigned int class_mask;     // Class 掩码，0 表示不匹配 class
    unsigned long driver_data;   // 驱动私有数据，probe 时使用
};
```

匹配规则是两级优先级：

1.  ## 1\. 精确匹配：Vendor ID + Device ID 完全一致 → 最高优先级
    
2.  2.  **类码匹配**：Vendor ID 没命中时，按 class & class\_mask 匹配。比如 NVMe 类码 0x010802，所有 NVMe SSD 都会被 NVMe 驱动认领

这个表的使用方式在第五章展开。

### 3.3 `struct pci_dev` — 设备实例

枚举到每个 PCIe 设备都会分配一个 `udevice`，其私有数据就是 `struct pci_dev`：

```
// udevice 的 platdata / priv 中存储
struct pci_dev {
    struct udevice *dev;         // 反向指针，指向所属的 udevice
    unsigned int vendor;         // Vendor ID（从配置空间读出来缓存）
    unsigned int device;         // Device ID
    unsigned int class;          // Class Code（3 字节合并：Base + Sub + Prog-IF）
    unsigned int busno;          // 总线号
    unsigned int devfn;          // device << 3 | function
    unsigned int hdr_type;       // Header Type：0 = 端点，1 = PCI-PCI 桥
    struct pci_bar bars[6];      // 6 个 BAR 的地址和大小
};
```

枚举阶段把配置空间的关键字段全读出来缓存，后面驱动 probe 直接用，不用再读配置空间。`bars[6]` 数组是重点——PCIe 每个 function 最多 6 个 BAR，BAR 分配的结果就存这里。

### 3.4 `struct pci_bar` — BAR 描述符

```
struct pci_bar {
    phys_addr_t addr;            // 分配后的物理地址
    phys_addr_t size;            // BAR 的大小（2 的幂对齐）
    int type;                    // PCI_BAR_IO 或 PCI_BAR_MEM
    bool prefetch;               // 是否可预取
};
```

BAR 分配的过程在第五章展开。

---

## 四、框架分层：骨-筋-皮

PCIe 子系统的源码组织可以用**骨-筋-皮**三层来理解：

### 4.1 骨 — `pci-uclass.c`：枚举引擎 + BAR 分配 + 驱动匹配

`drivers/pci/pci-uclass.c` 是整个 PCIe 子系统的核心骨架，约 1200 行，承担了三个核心职责：

**职责一：总线枚举**

```
// pci-uclass.c
int pci_hose_scan_bus(struct pci_controller *hose, int busno)
{
    // 从总线号 busno 开始，遍历 32 个 device × 8 个 function
    for (dev = 0; dev < PCI_MAX_DEVICES; dev++) {
        for (func = 0; func < PCI_MAX_FUNCTIONS; func++) {
            // 读 Vendor ID，0xffff 表示设备不存在
            pci_read_config_word(dev, func, PCI_VENDOR_ID, &vendor);
            if (vendor == 0xffff || vendor == 0x0000)
                continue;

            // 分配 udevice，读配置空间
            pci_bind_device(dev, func, &child_dev);

            // 读 Header Type，如果是桥（Type 1），递归枚举下游
            pci_read_config_byte(dev, func, PCI_HEADER_TYPE, &hdr);
            if ((hdr & 0x7f) == PCI_HEADER_TYPE_BRIDGE) {
                hose->current_busno++;
                pci_hose_scan_bus(hose, hose->current_busno);
            }
        }
    }
    return 0;
}
```

逻辑非常直接：双层循环扫所有 devfn → 读 Vendor ID 判断有无设备 → 有就分配 udevice → 是桥就递归。

**职责二：BAR 分配**

```
int pci_auto_assign_bars(struct pci_controller *hose)
{
    // 第一遍：只算大小，不分配地址
    // 遍历所有设备，读每个 BAR 的 size
    // 第二遍：从 mem_start 开始，按大小对齐顺序分配
    // 第三遍：把分配好的地址写回 BAR 寄存器
}
```

因为是一次性静态分配，没有碎片整理，算法极其简单。

**职责三：驱动匹配**

```
// 枚举完成后，DM 框架自动触发 probe
// 每个 PCI 驱动的 probe 函数里调用 pci_match_id() 做匹配
const struct pci_device_id *pci_match_id(
    const struct pci_device_id *ids, struct udevice *dev)
{
    struct pci_dev *pdev = dev_get_priv(dev);
    while (ids->vendor || ids->class) {
        if (ids->vendor && ids->vendor != pdev->vendor)
            goto next;
        if (ids->device && ids->device != pdev->device)
            goto next;
        if ((ids->class ^ pdev->class) & ids->class_mask)
            goto next;
        return ids;  // 匹配成功
next:
        ids++;
    }
    return NULL;
}
```

先精确匹配 Vendor + Device，再按 class 匹配，跟 Linux 逻辑完全一致。

### 4.2 筋 — DM uclass：把控制器和驱动串起来

PCIe 子系统用了两个 DM uclass：

| uclass | ID | 绑定对象 | probe 做什么 |
| --- | --- | --- | --- |
| `UCLASS_PCI` | 根总线 | `pci-uclass.c` 里的 `pci_uclass_post_probe` | 触发 `pci_hose_scan_bus` 枚举 |
| `UCLASS_PCI_GENERIC` | 枚举到的 PCIe 设备 | `pci-uclass.c` 里的通用绑定 | 读配置空间填 `pci_dev`，后续按类码匹配驱动 |

串联方式：

```
DM scan (board_init_r)
  → 遍历 DT 节点
    → compatible = "snps,dw-pcie" → 绑定 pcie_dw 驱动（UCLASS_PCI）
    → 驱动 probe：
        → 初始化硬件（PHY、clock、reset）
        → 分配 pci_controller，填 read_config / write_config
        → 注册到 uclass
        → uclass post_probe 触发 pci_hose_scan_bus
          → 枚举到设备 → 分配 udevice（UCLASS_PCI_GENERIC）
            → 读配置空间 → 填 pci_dev
            → DM 尝试匹配 PCI 设备驱动
              → NVMe 驱动（UCLASS_NVME）匹配 class=0x010802
              → 网卡驱动（UCLASS_ETH）匹配 class=0x020000
              → ...
```

### 4.3 皮 — pci 命令树

![[Inbox/笔记同步助手/微信公众号/2026/09/images/6f46a56aa73d2f21f60f04fe0dfa5652_MD5.jpg]]

```
cmd/pci.c               // pci command：pci enum / pci bar / pci regions / pci header
```

U-Boot 控制台可以直接操作 PCIe：

```
=> pci enum              # 手动触发枚举（如果自动枚举被禁用）
=> pci 1                 # 显示 bus 1 上的所有设备
=> pci regions           # 显示 BAR 分配情况
=> pci header 0.0.0      # dump bus 0 dev 0 func 0 的配置空间头
```

`pci header` 是最常用的调试命令，能直接看 Type 0 / Type 1 Header 的原始内容，排查设备识别问题非常有用。

---

## 五、启动数据流：从控制器复位到驱动 probe

走一遍完整 PCIe 初始化流程，分四个阶段，每阶段给出关键函数和调用栈。

### 5.1 阶段一：主机控制器初始化

```
board_init_r()
  → init_sequence_r[] 中触发 DM scan
    → dm_extended_scan()                    扫描 DT，绑定驱动
    → 遇到 compatible = "snps,dw-pcie"      匹配 pcie_dw_rockchip_probe（举例）
        → 解析 DT 的 reg / clocks / resets / phys
        → 开时钟、解复位、初始化 PHY
        → 建立配置空间映射（DBI 寄存器 ioremap）
        → 分配 struct pci_controller
        → 解析 DT 的 ranges 属性：
            → mem_start / mem_size（Memory Space）
            → io_start / io_size（I/O Space，ARM 侧通常留空）
        → 设置 first_busno = 0, last_busno = 255
        → 填写 read_config / write_config（指向控制器驱动实现）
        → 返回 0，probe 完成
```

控制器 probe 完成后，`pci_controller` 已注册到 `UCLASS_PCI` uclass。`pci_uclass_post_probe` 被 DM 框架回调，触发下一阶段的枚举。

### 5.2 阶段二：深度优先枚举总线

```
pci_uclass_post_probe()
→ pci_hose_scan_bus(hose, hose->first_busno)
 → for dev in 0..31:                          // 扫 32 个 device
   → for func in 0..7:                       // 每个扫 8 个 function
    → pci_read_config(dev, func, PCI_VENDOR_ID, &vendor, PCI_SIZE_16)
      → if vendor == 0xffff: continue        // 没设备，跳过

        → pci_bind_device(dev, func, &child_dev)
            → 分配 udevice（UCLASS_PCI_GENERIC）
            → 读 PCI_VENDOR_ID / PCI_DEVICE_ID / PCI_CLASS_REVISION
            → 读 PCI_HEADER_TYPE
            → 填 struct pci_dev 所有字段
           → 如果是多功能设备（hdr_type & 0x80），标记后续 function 也要扫

        → if hdr_type == PCI_HEADER_TYPE_BRIDGE:  // PCI-PCI 桥
            → hose->current_busno++               // 分配新总线号
            → 写 Secondary Bus Number 寄存器
            → pci_hose_scan_bus(hose, hose->current_busno)  // 递归扫下游
            → 同上逻辑，扫下游总线上的所有设备
```

深度优先枚举的核心：根总线 → 扫到桥 → 分配新总线号 → 递归扫下游 → 下游扫完返回 → 继续扫根总线下的下一个 devfn。这跟 Linux `pci_scan_child_bus` 的逻辑完全一样。

枚举完成时，所有 PCIe 设备（不管在几级桥后面）都有了对应的 `udevice` 和 `struct pci_dev`。

### 5.3 阶段三：BAR 静态分配

```
pci_uclass_post_probe()
  → pci_auto_assign_bars(hose)
    → // 第一遍：读每个 BAR 的 size
    → for each device:
        → for bar_idx in 0..5:
            → pci_read_bar_size(dev, bar_idx, &size)
                → 写入 BAR 寄存器 0xFFFFFFFF
                → 读回 BAR 寄存器的值
                → size = ～(读回值 & BAR_MASK) + 1   // 经典 BAR sizing 算法
            → bar.size = size
            → bar.type = (读回值 & 1) ? PCI_BAR_IO : PCI_BAR_MEM

    → // 第二遍：从前往后分配地址
    → cur_mem_addr = hose->mem_start
    → for each device:
        → for bar_idx in 0..5:
            → if bar.type == PCI_BAR_MEM:
                → bar.addr = ALIGN(cur_mem_addr, bar.size)  // 按 BAR 大小对齐
                → cur_mem_addr = bar.addr + bar.size

    → // 第三遍：写回 BAR 寄存器
    → for each device:
        → for bar_idx in 0..5:
            → pci_write_bar(dev, bar_idx, bar.addr)
```

BAR sizing 算法是 PCI 规范标准做法：写全 1 到 BAR 寄存器，读回来的最低几位会被硬件清 0（表示 BAR 大小），取反加 1 就是 BAR 的容量。比如写 0xFFFFFFFF，读回 0xFFFF0000 → size = ～0xFFFF0000 + 1 = 0x10000 = 64KB。

分配完 BAR，所有设备都有了自己独占的一段物理地址空间，驱动 probe 时直接用。

### 5.4 阶段四：驱动匹配 probe

```
DM 框架在枚举完成后触发所有未 probe 设备的 probe
  → device_probe(nvme_dev)
    → nvme_probe(nvme_dev)                   // drivers/nvme/nvme.c
      → pci_match_id(nvme_id_table, nvme_dev)  // 匹配 NVMe 驱动表
        → vendor = 0x144d (Samsung)            // 匹配！
      → 从 pci_dev->bars[0] 拿 BAR0 地址        // NVMe 寄存器在 BAR0
      → ioremap → iowrite 操作 BAR0                // 读写 NVMe 控制器寄存器
      → 建 Admin Queue → Identify Controller      // NVMe 初始化...
```

pci\_match\_id 匹配规则的优先级：

1.  1.  **Vendor + Device 精确匹配**（最高优先级）：驱动表里有 `{0x144d, 0xa808}` → 三星 NVMe SSD 精确命中
2.  2.  **Class Code 匹配**（兜底匹配）：比如 NVMe 驱动表里设 `class = 0x010802, class_mask = 0xffffff` → 所有 NVMe 设备都会匹配

这样即使你的 SSD 型号不在驱动表里，只要是 NVMe 类码，驱动就能认领。

### 5.5 完整调用栈汇总

![[Inbox/笔记同步助手/微信公众号/2026/09/images/9a04fe5274509be55bb7500d72a9e146_MD5.jpg]]

```
board_init_r()
  initr_dm()
    dm_init_and_scan()
      dm_extended_scan()                       // DT → udevice 绑定
        pcie_dw_rockchip_bind()                 // 绑定 UCLASS_PCI 驱动
  initr_pci()
    pci_init()
      uclass_foreach_dev(UCLASS_PCI)
        device_probe(controller_dev)
          pcie_dw_rockchip_probe()              // 硬件初始化
            → 开 clock / 解 reset / 初始化 PHY
            → 映射 DBI 寄存器
            → 分配 struct pci_controller
            → 解析 ranges → mem_start/mem_size
          pci_uclass_post_probe()               // uclass 回调
            pci_hose_scan_bus(hose, 0)          // ★ 深度优先枚举
              pci_bind_device()                 // 分配 udevice + pci_dev
              → 是桥？→ 递归 pci_hose_scan_bus
            pci_auto_assign_bars(hose)          // ★ BAR 静态分配
              → sizing → 分配 → 写回
          // DM 触发设备驱动 probe
          device_probe(nvme_dev)                 // NVMe 驱动 probe
            nvme_probe()                         // 读 BAR0 → 初始化 NVMe
```

整个流程没有中断、没有热插拔、没有动态重分配，全是同步顺序执行。哪一步出错直接加 `printf` 就能定位，不需要复现竞态条件。

---

## 六、平台适配

### 6.1 切入点 A：DesignWare PCIe 控制器

大多数 ARM SoC 用 Synopsys DesignWare PCIe IP。通用层在 `drivers/pci/pcie_dw.c`：

```
// drivers/pci/pcie_dw.c（简化骨架）
static int pcie_dw_probe(struct udevice *dev)
{
    struct pcie_dw *dw = dev_get_priv(dev);

    // 1. 映射 DBI 空间（Data Bus Interface，PCIe 核心寄存器）
    dw->dbi_base = dev_remap_addr_index(dev, 0);

    // 2. 映射 ATU 空间（Address Translation Unit，地址转换）
    dw->atu_base = dev_remap_addr_name(dev, "atu");

    // 3. 调用厂商定制的初始化
    if (dw->ops->init)
        dw->ops->init(dw);

    // 4. 链路训练
    pcie_dw_link_up(dw);           // 等 LTSSM 进入 L0

    // 5. 设置 ATU，建立配置空间访问窗口
    pcie_dw_setup_atu(dw, ...);    // 把 host 侧地址映射到 PCIe 侧的 config space

    // 6. 初始化 pci_controller
    hose->read_config = pcie_dw_read_config;   // DesignWare 通用实现
    hose->write_config = pcie_dw_write_config;
    hose->first_busno = 0;
    hose->last_busno = dw->max_busno;          // 通常 255
    // mem_start / mem_size 从 DT ranges 解析

    return 0;
}

static const struct dm_pci_ops pcie_dw_ops = {
    .read_config  = pcie_dw_read_config,
    .write_config = pcie_dw_write_config,
};

U_BOOT_DRIVER(pcie_dw) = {
    .name    = "pcie_dw",
    .id      = UCLASS_PCI,
    .ops     = &pcie_dw_ops,
    .probe   = pcie_dw_probe,
    .priv_auto = sizeof(struct pcie_dw),
};
```

各家厂商的定制驱动只需要填 `dw->ops` 里的几个回调：

| 回调 | 作用 | 举例（Rockchip） |
| --- | --- | --- |
| `init` | 平台特定的时钟、复位、PHY 初始化 | 开 PCIe 的 PCLK / ACLK，拉高 PERST# |
| `read_dbi` | DBI 寄存器的读方式 | 内存映射直接读（memcpy）还是间接寄存器 |
| `write_dbi` | DBI 寄存器的写方式 | 同上 |

以 Rockchip RK3588 为例，`drivers/pci/pcie_dw_rockchip.c` 约 500 行，核心就是实现这三个回调，加上 DT compatible 匹配。

### 6.2 切入点 B：ATU 的 outbound 映射

DesignWare PCIe 的 ATU（Address Translation Unit）是把 host 侧的地址空间映射到 PCIe 侧的地址空间的关键硬件。理解 ATU 对调试"PCIe 设备能识别但 BAR 访问不通"非常重要。

ATU 有 outbound 和 inbound 两个方向。U-Boot 阶段只需要 outbound：

-   -   **Outbound Memory**：CPU 访问 `0xA000_0000` → ATU 翻译成 PCIe Memory Write/Read，目标地址 `0x0000_0000`（PCIe 侧）
-   -   **Outbound Configuration**：CPU 访问 `0xB000_0000` → ATU 翻译成 Configuration Type 0/1 Request

`pcie_dw_setup_atu` 的典型调用：

```
// 建立 Config Space 访问窗口
pcie_dw_setup_atu(dw, PCIE_ATU_REGION0, PCIE_ATU_TYPE_CFG0,
                  dw->cfg_base,     // host 侧基址（例如 0xB000_0000）
                  0,                // PCIe 侧基址（configuration 总从 0 开始）
                  dw->cfg_size);    // 窗口大小

// 建立 Memory Space 访问窗口
pcie_dw_setup_atu(dw, PCIE_ATU_REGION1, PCIE_ATU_TYPE_MEM,
                  hose->mem_start,  // host 侧基址
                  0,                // PCIe 侧基址
                  hose->mem_size);  // 窗口大小
```

**踩坑提醒**：ATU region 数量有限（DW 是 8/16 个），每个 region 的 size 必须是 2 的幂且不小于 4KB。如果你的 `mem_size` 不是 2 的幂，ATU 配置会失败，BAR 访问就会挂。

### 6.3 新 SoC 移植工作量估算

接入一款新 SoC 的 PCIe 控制器：

| 类型 | 估算代码量 | 说明 |
| --- | --- | --- |
| DesignWare 系（有通用层） | **300–800 行** | 主要填 clock/reset/PHY 初始化 + `read_dbi`/`write_dbi` |
| 自研控制器（无通用层） | **800–2000 行** | 需要从零实现 `read_config` / `write_config`、链路训练、ATU |

加上必做项：DT binding 文档、defconfig 开 `CONFIG_PCI=y`、DTS 加节点。DesignWare 系通常一两天就能搞定。

---

## 七、运行：常见问题排查

### 7.1 分层排错

PCIe 子系统从硬件到驱动的完整调用链有四个层次，每个层次都有对应的故障现象和排查方法。

**现象 A：\*\*\*\*`pci enum`\*\*** 一个设备都看不到\*\*

**定位层：控制器初始化（阶段一）**

| 检查点 | 具体操作 |
| --- | --- |
| 控制器 probe 是否成功 | 看 boot log 有没有 `pcie@xxx: probe success` 或类似字样 |
| PHY 有没有 link 上 | 读控制器的 LTSSM 状态寄存器，是不是在 L0。通常 `pcie_dw_link_up` 会超时等待 DBI 里的 `PCIe Link Up` bit |
| PERST# 有没有释放 | 有些板卡 PERST# 接在 GPIO 上，驱动里要拉高才能让设备解除复位。看 DT 里有没有 `reset-gpios` |
| 参考时钟有没有给 | PCIe 需要 100MHz 参考时钟，有些板卡用 SoC 内部时钟输出，有些用外部晶振。量一下 PCIe REFCLK 有没有信号 |
| DT 配置对不对 | `reg` 属性有没有写对，`ranges` 有没有配。`ranges` 的格式是 `<pci_addr cpu_addr size>`，写错就会导致配置空间映射到错误地址 |

**排查手法**：在 `pcie_xxx_probe` 里加打印，看 DBI 映射地址、PHY init 返回码、link up 等待超时没有。`md.l <DBI基址+偏移> 1` 直接读 DBI 寄存器看链路状态。

**现象 B：能看到设备，但 BAR 全是 0**

**定位层：配置空间访问（阶段二/三）**

| 检查点 | 具体操作 |
| --- | --- |
| 配置空间读写是否正常 | 手动读几次 BAR 寄存器：`pci header 0.0.0` 看 BAR 有没有值。如果全 0，可能配置空间映射有问题 |
| ATU 配置对不对 | DesignWare 系检查 ATU outbound 窗口：region 类型是不是 `CFG0`、大小是不是 2 的幂、host 侧基址跟 DT 里 `ranges` 的配置空间段是否一致 |
| Byte Enable 有没有接对 | 配置空间访问要支持 byte/halfword/word 粒度，硬件上 C/BE# 信号映射错了就会读回 0 |

**排查手法**：`pci header 0.0.0 | pci header 1.0.0 | ...` 逐级看每个 bus 的第一个设备。如果 bus 0 正常但 bus 1 就全 0，大概率是 ATU 没给下游总线建配置空间窗口。

**现象 C：BAR 分配失败，地址不够**

**定位层：BAR 分配（阶段三）**

-   -   `mem_size` 设小了。所有设备的 BAR 加起来超过预留空间。改 DT 的 `ranges` 里 MEM 段的 size
-   • 某些设备的 BAR 特别大（比如 GPU 要 256MB），超出了控制器能管理的范围
    

**排查手法**：`pci regions` 看当前分配情况，`pci bar 0.0.0` 看每个设备的 BAR 大小，加起来跟 `ranges` 里的 MEM size 对比。

**现象 D：驱动匹配不上**

**定位层：驱动匹配（阶段四）**

-   • 设备 Vendor/Device ID 没在驱动表里（`pci_device_id`）
    
-   • Class Code 不对或 class\_mask 匹配规则写反了
    
-   • 驱动没编译进去（defconfig 里没开对应的 `CONFIG_*`）
    

**排查手法**：`pci header <bus.dev.func>` 看 Vendor ID / Device ID / Class Code 的原始值，跟驱动表一一对比。手动在驱动表里加一行 `{.vendor = 0xYOUR_VENDOR, .device = 0xYOUR_DEVICE}` 测试。

**现象 E：访问 BAR 地址直接卡死**

**定位层：BAR 地址映射**

-   • BAR 分配的地址不在 SoC 可访问范围内。比如 SoC DRAM 控制器只映射了 0–2GB，但 BAR 被分配到了 3GB，CPU 访问就会挂
    
-   • MMU 没开对应地址段的映射。U-Boot 如果开了 MMU，BAR 地址段需要加映射
    
-   • ATU 的 Memory outbound 窗口没配置或者配置错了
    

**排查手法**：打印 BAR 分配的地址，确认在 SoC address map 的合法范围。用 `md.l <BAR地址> 1` 试读，挂了就说明地址映射有问题。

### 7.2 调试 PCIe 最实用的三条命令

```
=> pci enum                            # 手动枚举，看扫描到几个设备
=> pci 1                               # 显示 bus 1 上的所有设备
=> pci header 1.0.0                    # dump bus 1 dev 0 func 0 的配置空间
```

`pci header` 的输出示例：

```
PCIe Header for bus 1, dev 0, func 0:
  vendor ID = 0x144d
  device ID = 0xa808
  class code = 0x010802 (Non-Volatile memory controller: NVMe)
  header type = 0x00 (Normal)
  BAR0 = 0xA0000000 (64-bit, prefetchable, size 16 KB)
  ...
```

看到这个输出，Vendor ID / Device ID / Class Code / BAR0 都有值，说明从枚举到 BAR 分配整条链都走通了。接下来就是驱动 probe 的事了。

---

## 八、这套框架的局限

前面七章说清了 U-Boot PCIe 子系统"能做什么"和"怎么做"，也该看到"哪里做得不够好"。

**局限一：不支持热插拔**

`pci_hose_scan_bus` 只在启动时调用一次。运行中插上新设备不会重新枚举，因为 U-Boot 没有 hotplug 中断处理——PCIe 的 Hot-Plug Controller 寄存器 U-Boot 根本不读。如果想加运行时枚举，需要自己调 `pci enum` 命令手动触发。但在 bootloader 场景下，这个"局限"其实是故意设计——减少代码量和复杂度。

**局限二：BAR 分配无冲突检测**

Linux 的 BAR 分配走 `pci_assign_unassigned_resources`，有完整的冲突检测和重试机制。U-Boot 的 `pci_auto_assign_bars` 就是从头往后塞，完全不管两个设备的 BAR 会不会重叠。如果你的 DT 里 `ranges` 配错了，或者设备 BAR 太多，完全可能把两个设备分配到同一段地址。好在 U-Boot 阶段 PCIe 设备数量极少（通常 1–3 个），实践中很少触发这个边界情况。

**局限三：只支持 Type 0 / Type 1 Header**

PCIe 规范还有 Type 2（CardBus Bridge）Header，U-Boot 完全不支持。不过 CardBus 是 2000 年代的技术，现在板卡上基本见不到，实践中不是问题。

**局限四：没有 MSI/MSI-X 中断管理**

U-Boot 里大部分 PCIe 驱动是 polling 模式。想用中断得自己实现一套最简的中断控制器映射，把 MSI 地址和数据寄存器配好。如果真要加网络启动（netboot），某些网卡驱动需要 MSI 中断才能正常工作，需要额外适配。

**局限五：与 Linux PCIe 的同步滞后**

U-Boot 的 PCIe 框架跟 Linux 是独立演进的。Linux 上后来加的 `pci_epf`（Endpoint Function Framework）、`pci/host` 通用控制器抽象等，U-Boot 侧都没有。不过 U-Boot 设计上就不需要 Endpoint 模式——它是 host 端，需要的是 root complex 功能。

---

## 九、总结

U-Boot PCIe 子系统的源码组织可以用四句话概括：

1.  1.  **`pci-uclass.c`**\*\* 是整个子系统的骨架\*\*。所有枚举、BAR 分配、驱动匹配的逻辑都在这里，约 1200 行，精简但完整体现了 PCIe 枚举标准流程。
2.  2.  **控制器差异封装在 ****`read_config`**** / ****`write_config`**** 两个函数指针里**。不管是 DesignWare 还是自研控制器，上层不用感知硬件细节。
3.  3.  **枚举算法是标准深度优先**。从根总线开始，扫到桥就递归，跟 Linux 逻辑完全一致，只是不支持热插拔。
4.  4.  **BAR 分配是静态的、一次性的**。从 `mem_start` 开始顺序分配，没有碎片整理，没有动态重映射。

四条流程串起整个子系统：

-   -   **控制器初始化** → PHY 训练 → link up → ATU 配置
-   -   **深度优先枚举** → 扫 devfn → 读 Vendor ID → 分配 udevice → 桥？→ 递归
-   -   **BAR 静态分配** → sizing → 对齐 → 顺序分配 → 写回寄存器
-   -   **驱动匹配 probe** → Vendor + Device 精确匹配 → Class Code 兜底 → 驱动初始化设备

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/72732786_1788669569842?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwOTUxMzQzOA%3D%3D%26mid%3D2247484405%26idx%3D1%26sn%3D2a4109c629876d7f91d694103fc522ce%26chksm%3Dc0ee4e5289f28c1e03da49b865051c464b6c6ac6490d180c939a7c156318b19cfd88bd7744c5%26mpshare%3D1%26scene%3D1%26srcid%3D0906JdlTGrdSrJOB4BYhcnum%26sharer_shareinfo%3D2a2f872376b9660807b7acb1a73062a3%26sharer_shareinfo_first%3D2a2f872376b9660807b7acb1a73062a3%23rd&s=obsidian)