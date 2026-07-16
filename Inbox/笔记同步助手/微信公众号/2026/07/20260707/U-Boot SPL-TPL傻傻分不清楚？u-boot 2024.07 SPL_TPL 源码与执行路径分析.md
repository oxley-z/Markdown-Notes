---
author: Surest
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwOTUxMzQzOA==&mid=2247484225&idx=1&sn=0be7be178a311f1da2d564ed10b79218&chksm=c025b462a6e4847865042003ab64e183400c0597af25c2165acd6fa16c26609388bbd5d5aed2&mpshare=1&scene=1&srcid=0707RNb64Hqjt9w26WPpOVeY&sharer_shareinfo=8d8bdb314dbb128e035d55539e161284&sharer_shareinfo_first=8d8bdb314dbb128e035d55539e161284#rd
saved: 2026-07-07 21:26:23
tags:
  - 笔记同步助手
id: de9f43ca-3dbd-48a3-b18e-b91ce9d7e462
---

公众号名称：摸鱼的日记本

作者名称：Surest

发布时间：2026-07-07 00:48

## 一、概述

U-Boot 从加电到进入操作系统有两条大方向：

```
① NOR/QSPI XIP:   CPU  →  Flash 上 XIP U-Boot  →  DDR U-Boot  →  OS
② 无 XIP 介质:     ROM  →  (TPL)  →  SPL  →  U-Boot proper  →  OS
```

**方向①是最主流的传统嵌入式做法**：CPU 直接从 NOR/QSPI Flash线性取指执行，片内 SRAM 仅作栈用；`board_init_f()` 初始化 DDR 后，`relocate_code()` 将 U-Boot 自身搬到 DDR 继续跑。整个过程没有 SPL/TPL，也无需 BootROM 加载。

**方向②是 eMMC/NAND/SD 等无 XIP 能力介质**采用的方案：BootROM 或 SPL 先把镜像搬进 SRAM/DDR 才能运行。SRAM 大小、DDR 是否需要软件训练、是否引入 ARM Trusted Firmware，决定了走 SPL 单段、SPL+TPL 双段还是 SPL+ATF 组合。

U-Boot 2024.07 版本对 SPL/TPL 框架进行了以下几处收敛：

-   -   `common/spl/spl.c` 作为 SPL 主流程的唯一入口
-   -   `spl_boot_device()` 替代了平台私有的启动设备判断逻辑
-   • TPL 到 SPL 的参数传递统一到 `arch/arm/lib/armv8/tpl.c` 的参数块
    
-   • FIT 加载路径从 `common/spl/spl.c` 独立到 `common/spl/spl_fit.c`
    

本文按源码路径分析 SPL/TPL 的核心结构、执行流程与可修改点。

## 二、编译边界

TPL、SPL、U-Boot proper 三段共享大部分源码目录，具体哪些目标文件被编入哪一段，由 Kconfig 开关 `CONFIG_TPL_BUILD` 与 `CONFIG_SPL_BUILD` 控制：

| 段             | 生效开关                 | 编入范围                                    |
| ------------- | -------------------- | --------------------------------------- |
| TPL           | `CONFIG_TPL_BUILD=y` | `drivers/tpl/`、`arch/xxx/cpu/xxx/tpl.o` |
| SPL           | `CONFIG_SPL_BUILD=y` | `common/spl/`、`drivers/spl/`            |
| U-Boot proper | 两个都为 n               | `common/board.c`及全量驱动                   |

同一份 `board/xxx/board.c` 在三轮编译中会分别产出三份不同的目标文件，`board_init()` 存在三个独立版本。这是分析 SPL/TPL 源码的第一个前提。

三段镜像使用各自的设备树片段：

```
1234
u-boot.dtsi        → U-Boot proper
u-boot-spl.dtsi    → SPL
u-boot-tpl.dtsi    → TPL
```

`u-boot-spl.dtsi` 中的属性在 SPL 编译阶段被 dtc 合并进 SPL 的 DTB；proper 阶段完全看不到。

## 三、核心数据结构

SPL/TPL 框架内涉及四个核心数据结构，其余类型均围绕它们展开。

### 1\. `struct spl_image_info`

定义于 `include/spl.h`，描述被 SPL 加载的下一段镜像：

```
12345678910
struct spl_image_info {
    const char *name;
    uint8_t     os;
    uint32_t    load_addr;
    uint32_t    entry_point;
    uint32_t    size;
    void       *fdt_addr;
    /* ... */
};
```

各类介质加载器（MMC、SPI NOR、NAND、USB、FIT）最终填写的都是同一个 `struct spl_image_info`。

### 2\. `struct spl_boot_device`

定义于 `include/spl.h`，用于绑定启动设备号与对应的加载回调：

```
123456
struct spl_boot_device {
    uint32_t boot_device;
    int (*load)(struct spl_image_info *spl_image,
                struct spl_boot_device *bootdev);
};
```

2024.07 版本在此结构上增加了优先级字段，允许同一类设备挂载多个加载器按优先级依次尝试。

### 3\. `spl_boot_device()`

`common/spl/spl.c` 中的分发函数，返回本次启动使用的设备号。平台代码通过 override 该函数决定实际启动顺序。

### 4\. `struct tpl_params`

`arch/arm/lib/armv8/tpl.c` 中定义的 TPL→SPL 参数块，放置在约定好的物理地址上：

```
123456789
struct tpl_params {
    u32 magic;
    u32 version;
    u64 ddr_size;
    u64 ddr_base;
    u64 fdt_addr;
    /* ... */
};
```

TPL 完成 DDR 训练后向该结构写入 DDR 大小、基址、FDT 位置；SPL 在启动初期从相同地址读回。

四个数据结构围绕 `spl_load_image()` 的调用关系如下图所示。①② 是加载中枢的输入，③ 是它的输出，④ 是 TPL 通过 `tpl_params` 提供的 fdt\_addr 透传给最终镜像描述。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/eb45c1a8b2dbbe13347ee5acdc2dacd7_MD5.jpg]]

## 四、框架分层

SPL/TPL 框架按职责可以划分为五层。上层负责策略，下层贴近硬件。

| 层次 | 职责 | 主要源码位置 |
| --- | --- | --- |
| 平台策略层 | 决定启动顺序、板型识别、内存布局 | `board/xxx/spl.c`、`board/xxx/tpl.c`、`u-boot-spl.dtsi` |
| 核心框架层 | SPL 主流程与镜像加载框架 | `common/spl/spl.c`、`common/spl/spl_fit.c` |
| 设备驱动层 | 各类介质的加载器实现 | `common/spl/spl_mmc.c`、`spl_spi.c`、`spl_nand.c` |
| 架构支撑层 | TPL/SPL 跳转、异常向量、缓存维护 | `arch/arm/lib/armv8/tpl.c`、`arch/arm/cpu/armv8/start.S` |
| 编译构建层 | 决定源码文件的编入范围 | `common/spl/Makefile`、`scripts/Makefile.spl` |

TPL 与 SPL 之间只通过一次函数跳转过渡，两段之间不共享全局变量。信息交换仅通过以下三条路径：

1.  1\. 位于约定物理地址上的参数块（`tpl_params`）
    
2.  2\. ARM64 平台通用的寄存器传参约定（`x0`～`x3`）
    
3.  3\. FDT 的 `chosen` 节点
    

## 五、启动数据流

按 ROM → TPL → SPL → U-Boot proper → OS 顺序，每两个相邻阶段之间箭头上的标签描述本次交接携带的核心数据。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/eeb99e3b058f624688b5048784765c30_MD5.jpg]]

### TPL 阶段的数据生产

1.  4\. 配置 PLL 与时钟树
    
2.  5\. 初始化 DDR PHY 与 controller
    
3.  6\. 训练 DDR，得到有效大小与工作频率
    
4.  7\. 将 DDR 信息写入 `tpl_params` 结构
    
5.  8\. 从 Flash 加载 SPL 到 DDR
    
6.  9\. 跳转到 SPL entry point
    

### SPL 阶段的数据消费

1.  10\. 从固定地址读取 `tpl_params`，确认 DDR 可用
    
2.  11\. 初始化串口
    
3.  12\. 调用 `spl_boot_device()` 决定启动介质
    
4.  13\. 从介质加载 FIT 镜像
    
5.  14\. 解析 FIT，拆出 U-Boot proper、ATF、OP-TEE、FDT 段
    
6.  15\. 将各段搬到 FIT 中声明的加载地址
    
7.  16\. 跳转到 ATF 或 U-Boot proper 入口
    

2024.07 版本中的一个显著变化：SPL 加载 FIT 时不再使用硬编码的 load address，而是从 FIT 中每个 image 节点的 `load` 属性读取。地址布局由 FIT 描述文件承载，源码内不再保留平台专属的偏移量。

  

## 六、SPL 主流程

`common/spl/spl.c` 中 `spl_main()` 是 SPL 的顶层入口：

```
1234567891011121314
void spl_main(void)
{
    struct spl_image_info spl_image = {0};
 
    spl_init();
    spl_board_init();
 
    /* 加载下一段镜像 */
    ret = spl_load_image(&spl_image);
 
    /* 跳到加载好的镜像 */
    spl_jump_to_image(&spl_image);
}
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b697f25e3c0eba394ed9c6f2de8cb250_MD5.jpg]]

### 1\. `spl_init()`

位于 `common/spl/spl.c`，负责 SPL 阶段全局数据的初始化：建立 `gd`（Global Data）结构、初始化 SPL 阶段的 malloc 分配器、注册异常向量。该函数与具体平台无关，所有 SPL 平台复用同一份实现。

### 2\. `spl_board_init()`

平台可选实现的板级初始化函数。默认为空实现（weak symbol），平台在 `board/xxx/spl.c` 中提供强符号覆盖：

```
12
__weak void spl_board_init(void) { }
```

板级电源、时钟树、引脚复用、复位控制等操作在此完成。

### 3\. `spl_boot_device()`

同为 weak 符号，返回本次启动使用的设备号。默认实现通常返回 `BOOT_DEVICE_MMC1`，平台通过 override 决定真实的启动介质：

```
12345678910
u32 spl_boot_device(void)
{
    u32 val = readl(GPIO_STATUS_REG);
 
    if (val & BIT(5))
        return BOOT_DEVICE_SPI;
 
    return BOOT_DEVICE_MMC1;
}
```

### 4\. `spl_load_image()`

根据 `spl_boot_device()` 的返回值查表调用具体的加载器。2024.07 版本的加载器表：

```
1234567891011
static const struct spl_boot_mode spl_boot_mode_list[] = {
#if CONFIG_IS_ENABLED(MMC_SUPPORT)
    { BOOT_DEVICE_MMC1, spl_mmc_load_image },
    { BOOT_DEVICE_MMC2, spl_mmc_load_image },
#endif
#if CONFIG_IS_ENABLED(SPI_FLASH_SUPPORT)
    { BOOT_DEVICE_SPI,  spl_spi_load_image },
#endif
    /* ... */
};
```

加载器执行完毕后，`struct spl_image_info` 中的 `load_addr`、`entry_point`、`fdt_addr` 均已填充完成。

### 5\. `spl_jump_to_image()`

位于 `arch/arm/lib/armv8/spl.c`：

```
1234567891011121314
void spl_jump_to_image(struct spl_image_info *spl_image)
{
    void (*entry)(void *fdt_addr, u64 res0, u64 res1, u64 res2);
 
    /* 刷 D-cache、失效 I-cache，确保新镜像看到一致的内存视图 */
    flush_dcache_all();
    icache_invalidate_all();
 
    entry = (void *)spl_image->entry_point;
 
    /* ARM64 调用约定：x0 = fdt_addr */
    entry(spl_image->fdt_addr, 0, 0, 0);
}
```

跳转前必须刷 D-cache 并失效 I-cache，避免下一段代码执行时使用了缓存中残留的旧数据。FDT 地址必须 8 字节对齐，且不能落在加载过程中会被覆盖的区域。2024.07 新增的 `spl_fit_ensure_fdt_safe()`（`common/spl/spl_fit.c`）在检测到 FDT 与加载区重叠时会自动将其搬移到安全区域。

## 七、平台适配的三个关键切入点

从上述源码结构可以看出，SPL/TPL 的平台适配主要落在以下三个位置。

### 1\. `board/xxx/tpl.c`：DDR 初始化与参数块填充

```
12345678910111213141516171819202122232425
#define TPL_PARAMS_ADDR 0x200000
 
struct tpl_params {
    u32 magic;
    u32 version;
    u64 ddr_size;
    u64 ddr_base;
} __packed;
 
void tpl_board_init(void)
{
    struct tpl_params *params = (void *)TPL_PARAMS_ADDR;
 
    clock_init();
    ddr_init();
 
    params->magic    = 0x54504c50;   /* "TPLP" */
    params->version  = 1;
    params->ddr_size = ddr_probe_size();
    params->ddr_base = CFG_SYS_SDRAM_BASE;
 
    flush_dcache_range((uintptr_t)params,
                       (uintptr_t)params + sizeof(*params));
}
```

`flush_dcache_range()` 是关键：SPL 从 DDR 读取参数块前，TPL 必须将 cache 中的写入回写到 DDR，否则 SPL 读到的会是初始化前的旧值。

### 2\. `board/xxx/spl.c`：启动设备决策

```
123456789101112131415
int spl_board_init(void)
{
    struct tpl_params *params = (void *)TPL_PARAMS_ADDR;
 
    if (params->magic == 0x54504c50)
        gd->ram_size = params->ddr_size;
 
    return 0;
}
 
u32 spl_boot_device(void)
{
    return get_boot_source_from_hw();
}
```

### 3\. `u-boot-spl.dtsi` 与 FIT `.its`：加载地址

后续镜像的加载地址在 2024.07 中优先从 FIT 描述文件读取，其次回落到设备树 `chosen` 节点：

```
1234567
/ {
    chosen {
        u-boot,spl-load-address = <0x40200000>;
        u-boot,spl-entry-point  = <0x40200000>;
    };
};
```

FIT `.its` 中的对应写法：

```
1234567891011121314
images {
    u-boot {
        description = "U-Boot";
        data = /incbin/("u-boot-nodtb.bin");
        type = "standalone";
        arch = "arm64";
        os   = "u-boot";
        compression = "none";
        load  = <0x40200000>;
        entry = <0x40200000>;
        hash-1 { algo = "sha256"; };
    };
};
```

加载地址由 FIT 承载后，配置随镜像分发，源码内无需保留平台相关的偏移量。

## 八、常见启动模式

并非所有平台都会启用完整的 TPL/SPL/proper 三段。下表列出五种常见组合，之后按模式逐一展开其工作流。

| 模式 | 组合 | SRAM 要求 | DDR 训练 | Secure World | 典型平台 |
| --- | --- | --- | --- | --- | --- |
| 1 | 完整 U-Boot 自 Flash XIP + 自重定位 | 仅作栈 KB | U-Boot 自身完成 | 无 | NOR/QSPI XIP 的传统嵌入式平台 |
| 2 | SPL + U-Boot | 64 ～ 256 KB | SPL 完成 | 无 | i.MX8、STM32MP1、AM335x |
| 3 | TPL + SPL + U-Boot | 32 ～ 64 KB | TPL 完成 | 可选 | RK3399、RK3568、RK3588 |
| 4 | SPL + ATF + U-Boot | 64 ～ 256 KB | SPL 完成 | 有 BL31 | i.MX8M、NXP Layerscape |
| 5 | SPL + FIT falcon | 32 ～ 128 KB | SPL 完成 | 无 | 车载 IVI、快启工控 |

选型三条依据：SRAM 是否能容纳整张 U-Boot；DDR 是否需要软件训练；是否使用 ARM Trusted Firmware。三个问题答完，启动模式随之确定，不必默认套用完整 TPL/SPL/proper 三段。

### 8.1 模式 1：XIP + 自重定位

**最主流的启动方式**。CPU 上电后从 NOR/QSPI Flash 的复位向量**直接线性取指执行**——NOR 类 Flash 支持字节寻址与随机读取，具备 XIP（eXecute-In-Place）能力，无需先把镜像搬到 RAM 就能跑。

此模式下**没有 SPL/TPL、没有 BootROM 拷贝**，U-Boot 就地开始运行：

1.  17.  `_start` 建立最小执行环境（禁用中断、异常向量表、栈指针指向片内 SRAM）
2.  18.  `board_init_f()` 在 Flash 上原地执行，初始化 UART、DDR 控制器、DDR PHY
3.  19\. DDR 训练完成并可用后，`relocate_code()` 把 U-Boot 自身从 Flash 拷到 DDR 高地址，同时修正重定位表中的绝对地址引用
    
4.  20\. 跳到 DDR 上的新地址继续执行 `board_init_r()`，进入完整驱动初始化与命令行
    

栈从 SRAM 迁到 DDR 的时机由 `board_init_f()` 中的 `arch_setup_gd()` 与 `relocate_code()` 之间的过渡完成。之所以先用 SRAM 作栈：Flash 上执行阶段 DDR 还没初始化，全局数据、malloc、`gd` 结构都必须放 SRAM 里；一旦 DDR init 完成才可以搬迁。

关键源码路径：

-   • 入口：`arch/arm/cpu/armv8/start.S :: _start`（Flash 上执行）
    
-   • board\_init\_f：`common/board_f.c :: board_init_f()`
    
-   • 自搬移：`arch/arm/lib/relocate_64.S :: relocate_code`
    
-   • board\_init\_r：`common/board_r.c :: board_init_r()`
    
-   • 内存布局：`include/asm-generic/global_data.h :: struct global_data`
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/5ace1c97a6bf1db51cc16ea97afbda19_MD5.jpg]]

### 8.2 模式 2：SPL + U-Boot

老式嵌入式 SoC 的典型模式。BootROM 加载 SPL 到 SRAM或启动区域，SPL 完成 DDR 初始化后从启动介质加载 U-Boot proper 到 DDR。SPL 与 proper 是两个独立编译产物，通过 `spl_image_info` 交接。

关键源码路径：

-   • SPL 入口：`common/spl/spl.c :: spl_main()`
    
-   • 加载器：`common/spl/spl_mmc.c` / `spl_spi.c` / `spl_nand.c`
    
-   • 跳转：`arch/arm/lib/armv8/spl.c :: spl_jump_to_image()`
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/c522f424d7650a92198b21230e559d57_MD5.jpg]]

### 8.3 模式 3：TPL + SPL + U-Boot

SRAM 极小、DDR 需软件训练时使用的三段式启动。TPL 单独负责 DDR 训练与 `tpl_params` 生产；SPL 与 U-Boot 均运行在 DDR 上。此模式是 Rockchip 系列的标准配置。

关键源码路径：

-   • TPL 入口：`arch/arm/cpu/armv8/start.S`（受 `CONFIG_TPL_BUILD` 门控）
    
-   • TPL 主体：`board/xxx/tpl.c :: tpl_board_init()`
    
-   • TPL→SPL 参数：`arch/arm/lib/armv8/tpl.c :: struct tpl_params`
    
-   • SPL 消费参数：`board/xxx/spl.c :: spl_board_init()`
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/6ce9a0b81e046dd8e10ae157f8a7bbd1_MD5.jpg]]

### 8.4 模式 4：SPL + ATF + U-Boot

进入 ARM Trusted Firmware 生态后的标准启动组合。SPL 加载 FIT 后先跳转到 BL31，BL31 在 EL3 secure world 完成 PSCI、SMC handler 初始化，随后通过 `eret` 返回到 EL2 的 U-Boot proper。Linux 运行时通过 SMC 与 BL31 交互实现 CPU 上下线、系统休眠等电源管理动作。

关键源码路径：

-   • FIT 加载：`common/spl/spl_fit.c :: spl_load_simple_fit()`
    
-   • 跳转 BL31：`common/spl/spl_atf.c :: spl_invoke_atf()`
    
-   • BL31 侧：ARM Trusted Firmware 项目 `bl31/`
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/15623a87208a5b2827e4df5217114a97_MD5.jpg]]

### 8.5 模式 5：SPL + FIT falcon 直跳内核

对启动延迟极敏感的场景使用的裁剪方案，也称 falcon 模式。SPL 完全跳过 U-Boot proper，从 FIT 中直接加载 Linux 内核与 FDT，跳入内核入口。此模式下 U-Boot proper 只在开发调试时使用。

关键源码路径：

-   • falcon 判定：`common/spl/spl.c :: spl_start_uboot()`（weak，返回 0 走 falcon）
    
-   • FIT 加载：`common/spl/spl_fit.c :: spl_load_simple_fit()`
    
-   • 跳内核：`arch/arm/lib/armv8/spl.c :: spl_jump_to_image()`
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/e64d80fd340c9934693f36404fb39583_MD5.jpg]]

## 九、源码路径速查

| 主题 | 关键源码 |
| --- | --- |
| SPL 主流程 | `common/spl/spl.c :: spl_main()` |
| 加载器分发 | `common/spl/spl.c :: spl_load_image()` |
| MMC 加载器 | `common/spl/spl_mmc.c :: spl_mmc_load_image()` |
| FIT 加载 | `common/spl/spl_fit.c :: spl_load_simple_fit()` |
| SPL 跳转 | `arch/arm/lib/armv8/spl.c :: spl_jump_to_image()` |
| TPL 入口 | `arch/arm/cpu/armv8/start.S :: reset` |
| TPL 参数块定义 | `arch/arm/lib/armv8/tpl.c :: struct tpl_params` |
| 启动设备决策 | `board/xxx/spl.c :: spl_boot_device()`weak override |
| DDR 初始化 | `board/xxx/tpl.c :: tpl_board_init()` |
| SPL 设备树 | `arch/arm/dts/*u-boot-spl.dtsi` |
| TPL 设备树 | `arch/arm/dts/*u-boot-tpl.dtsi` |
| 编译入口 | `scripts/Makefile.spl`、`common/spl/Makefile` |

## 十、小结

U-Boot 2024.07 的 SPL/TPL 框架收敛出了清晰的抽象层次：`spl_image_info` 作为镜像加载的统一描述、`spl_boot_device()` 作为启动介质的策略入口、`tpl_params` 作为 TPL→SPL 的信息通道。理解这三个对象与它们对应的源码路径，就掌握了 SPL/TPL 的核心结构。

平台适配的工作集中在 `board/xxx/tpl.c`、`board/xxx/spl.c` 与 `u-boot-spl.dtsi` 三处，其他框架代码保持通用。这种收敛使得新平台移植的工作量明显下降，源码结构也更便于阅读与分析。

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/3c0c9904_1783430780479?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwOTUxMzQzOA%3D%3D%26mid%3D2247484225%26idx%3D1%26sn%3D0be7be178a311f1da2d564ed10b79218%26chksm%3Dc025b462a6e4847865042003ab64e183400c0597af25c2165acd6fa16c26609388bbd5d5aed2%26mpshare%3D1%26scene%3D1%26srcid%3D0707RNb64Hqjt9w26WPpOVeY%26sharer_shareinfo%3D8d8bdb314dbb128e035d55539e161284%26sharer_shareinfo_first%3D8d8bdb314dbb128e035d55539e161284%23rd&s=obsidian)