---
author: Surest
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwOTUxMzQzOA==&mid=2247484386&idx=1&sn=54434204ebafd52a43f8574cc25fcbc1&chksm=c068af18f97459d4a766f29b9a8d4f76b19519c313def28a494a6f9ad314d3909a0d970b432a&mpshare=1&scene=1&srcid=0717arbs6yh7I14YDpLQUk6M&sharer_shareinfo=c42dbaf3e80c0235b09d328d401a7bc5&sharer_shareinfo_first=c42dbaf3e80c0235b09d328d401a7bc5#rd
saved: 2026-07-17 19:35:27
tags:
  - 笔记同步助手
id: 145ff6bc-4322-4852-a815-018108de2a09
---

公众号名称：摸鱼的日记本

作者名称：Surest

发布时间：2026-07-17 19:30

## 一、概述

### 1.1 flash 子系统要解决的问题

U-Boot 的核心工作是把镜像从存储介质搬到 DRAM 再跳转执行。存储介质在源码里被划分成四条主线：

-   -   **SPI-NOR**：容量 MB 级，字节编程，随机读快，`env` / boot.bin 常见落脚点
-   -   **Raw NAND**（parallel）：GB 级，页编程 + 坏块 + ECC 强依赖
-   -   **SPI-NAND**：Raw NAND 的 SPI 封装，芯片自带 on-die ECC
-   -   **eMMC / SD**：块设备语义，GB-TB 级，芯片内部 FTL 处理坏块和磨损均衡

四种介质硬件语义完全不同，但 U-Boot 上层对它们的需求高度统一：读某段、写某段、擦某段。整个 flash 子系统就是为了把四种物理语义压平到一套接口。

### 1.2 骨-筋-皮三层结构

U-Boot 源码里，flash 子系统按**骨-筋-皮**三层组织：

-   -   **骨**：`struct mtd_info` 是 flash 类介质（SPI-NOR / NAND）的统一门面，提供 `_erase` / `_read` / `_write` 三个函数指针。位于 `include/linux/mtd/mtd.h`。
-   -   **筋**：DM $DriverModel$ 的 uclass 把控制器驱动和芯片驱动串起来：`UCLASS_MTD` / `UCLASS_SPI_FLASH` / `UCLASS_MMC` / `UCLASS_SPI`。
-   -   **皮**：`sf` / `nand` / `mtd` / `mmc` 四条命令树，从控制台落到设备操作。命令入口分别在 `cmd/sf.c` / `cmd/nand.c` / `cmd/mtd.c` / `cmd/mmc.c`。

**eMMC / SD 不走 MTD**：它们是块设备，挂在 `blk_desc` 上，跟 flash 类完全隔离。这条分岔是本文的第一条主线。

### 1.3 一分钟速览

flash 子系统的关键源码入口：

-   • 命令层：`cmd/sf.c` / `cmd/mmc.c` / `cmd/mtd.c` / `cmd/nand.c`
    
-   • 抽象层：`drivers/mtd/mtdcore.c`（MTD 分派）、`drivers/block/blk-uclass.c`（块设备分派）
    
-   • 介质核心：`drivers/mtd/spi/spi-nor-core.c`（SPI-NOR）、`drivers/mtd/nand/raw/`（Raw NAND）、`drivers/mtd/nand/spi/`（SPI-NAND）、`drivers/mmc/mmc.c`（MMC 协议层）
    
-   • 控制器抽象：`drivers/spi/spi-mem.c`（SPI 内存操作）、`drivers/mmc/sdhci.c`（SDHCI 通用层）
    
-   • 环境变量挂钩：`env/sf.c` / `env/mmc.c` / `env/nand.c`
    

后面的章节按这条主线展开：先看源码把 flash 分成几类，再看核心数据结构，再看框架分层怎么把它们串起来，最后落到具体控制器（举例 Zynq）。

---

![[Inbox/笔记同步助手/微信公众号/2026/07/images/db078c00e7ad3ff8dbef6628d7670dc9_MD5.jpg]]

Image

### 1.4 U-Boot 为什么"减负"了 Linux MTD

熟悉 Linux 的读者会立刻发现：U-Boot 的 MTD 层是 Linux `include/linux/mtd/mtd.h` 的裁剪版。同样的 `mtd_info`、同样的 `_read` / `_write` / `_erase` 三个函数指针，但 U-Boot 少掉了不少东西：

-   -   **没有 partition parser 框架**。Linux 的 `cmdlinepart` / `ofpart` / `redboot` 三种 parser 全部拿掉，`mtd` 命令只支持 `add` 手工建分区
-   -   **没有 chardev / sysfs 出口**。Linux 每个 MTD 设备暴露成 `/dev/mtd0`、`/sys/class/mtd/mtd0`，U-Boot 直接用全局链表 + `get_mtd_device_nm` 名字查找
-   -   **没有并发保护**。Linux 的 `_get_device` / `_put_device` 引用计数、`sync` / `lock` / `unlock` 一律省略，因为 U-Boot 是单线程执行
-   -   **没有 fastmap / bit-flip 追踪**。Linux 里 UBI 靠这些做磨损均衡，U-Boot 只做被动读，写路径也不做统计

减负的**动机**很直接：bootloader 只关心"把镜像从存储搬到 DRAM 再交给 kernel"，加载完就交棒退场。所以 MTD 层在 U-Boot 里只保留了**能让上层不感知介质差异**的最小契约——`_read / _write / _erase` 三个函数指针，加一份容量/擦除粒度描述。

这个减负策略反过来解释了 U-Boot flash 子系统的一些看似奇怪的设计：`sf` 和 `nand` 命令绕过 MTD 分派、`env_sf_load` 直接调 `spi_flash_probe`、`spl` 阶段还有更精简的 `drivers/mtd/spi/sf_mtd.c`——都是在**这个契约不够便宜时**开的旁路。

## 二、源码边界：介质分类与目录结构

U-Boot 里 flash 相关的源码分成四个大目录。理解目录划分就理解了框架的第一层分类。

### 2.1 `drivers/mtd/spi/` — SPI-NOR 子树

```
drivers/mtd/spi/
├── sf_probe.c          // UCLASS_SPI_FLASH 驱动骨架
├── sf-uclass.c         // UCLASS_SPI_FLASH 定义
├── spi-nor-core.c      // 介质探测 + 命令组装
├── spi-nor-ids.c       // JEDEC ID 数据库
├── sf_internal.h       // spi_flash 内部结构
└── sandbox.c           // sandbox 测试驱动
```

`spi_nor_scan()` 是这条子树的核心入口：读 JEDEC ID、解析 SFDP、填充 `mtd_info`。

### 2.2 `drivers/mtd/nand/` — NAND 子树

```
drivers/mtd/nand/
├── raw/                // Raw NAND（并行 NAND）
│   ├── nand.c
│   ├── nand_base.c
│   └── <soc>_nand.c    // 各家 SoC 的 NAND 控制器驱动
├── spi/                // SPI-NAND
│   ├── core.c
│   └── <vendor>.c      // Micron / Winbond / Toshiba
└── core.c
```

Raw NAND 和 SPI-NAND 共享 `nand_chip` 抽象，但控制器接口差异大：Raw NAND 走并行地址/数据周期，SPI-NAND 走 `spi_mem_op`。

### 2.3 `drivers/mmc/` — MMC 子树

```
drivers/mmc/
├── mmc.c               // 协议层：CMD0/8/17/18/23...
├── mmc-uclass.c        // UCLASS_MMC 骨架
├── mmc_write.c
├── sdhci.c             // SDHCI 通用层
├── <soc>_sdhci.c       // 各家 SDHCI 主控器驱动
└── <soc>_mmc.c         // 非 SDHCI 主控（如 Rockchip DWMMC）
```

MMC 子树跟 MTD 完全独立。`sdhci.c` 提供通用 SDHCI 寄存器操作，各家主控器驱动只填 `sdhci_ops` 里的 `set_ios` / `set_clock` / `platform_execute_tuning` 就能复用整套逻辑。

### 2.4 `drivers/spi/` — SPI 控制器与 spi-mem

```
drivers/spi/
├── spi-uclass.c        // UCLASS_SPI
├── spi-mem.c           // spi_mem_exec_op：把 cmd+addr+dummy+data 打包
├── spi-mem-nodm.c
└── <soc>_qspi.c        // 各家 QSPI 控制器
```

`spi-mem.c` 是 v2019 引入的新抽象：控制器驱动实现 `mem_ops->exec_op`，上层（SPI-NOR / SPI-NAND）用 `struct spi_mem_op` 表达一次完整操作，屏蔽了老式 `xfer` 回调的碎片化。

### 2.5 命令层

```
cmd/
├── sf.c        // SPI-NOR：sf probe / read / write / erase / update
├── nand.c      // Raw NAND：nand read / write / erase / bad / dump
├── mtd.c       // MTD 统一：mtd list / read / write / erase
└── mmc.c       // MMC：mmc dev / read / write / erase / partconf
```

四个命令入口的分工：

-   -   `sf` — 直接调 `spi_flash_read/write/erase`，绕过 MTD 分派，历史遗留但用得最广
-   -   `nand` — 直接调 `nand_read/write/erase`，绕过 MTD 分派
-   -   `mtd` — 走 `mtd_read/write/erase`，通过 `struct mtd_info->_read` 分派到具体后端
-   -   `mmc` — 走 `blk_dread/blk_dwrite`，通过块设备接口

### 2.6 环境变量挂钩

```
env/
├── sf.c        // env_sf_load / env_sf_save
├── mmc.c       // env_mmc_load / env_mmc_save
├── nand.c
└── env.c       // env_load 分派入口
```

`env_load()` 根据 `CONFIG_ENV_IS_IN_*` 选一个 backend，每个 backend 借上面的 flash 子系统读一段固定 offset。

## 三、核心数据结构

### 3.1 `struct mtd_info` — flash 类的统一门面

```
// include/linux/mtd/mtd.h
struct mtd_info {
    u_char type;               // MTD_NORFLASH / MTD_NANDFLASH / MTD_MLCNANDFLASH ...
    uint32_t flags;
    uint64_t size;             // 总容量
    uint32_t erasesize;        // 擦除单位（NOR: 4KB/64KB, NAND: 128KB）
    uint32_t writesize;        // 编程单位（NOR: 1 byte, NAND: 2KB/4KB page）
    uint32_t oobsize;          // NAND OOB 区大小（NOR 为 0）

    int (*_erase)(struct mtd_info *mtd, struct erase_info *instr);
    int (*_read) (struct mtd_info *mtd, loff_t from, size_t len,
                  size_t *retlen, u_char *buf);
    int (*_write)(struct mtd_info *mtd, loff_t to, size_t len,
                  size_t *retlen, const u_char *buf);

    void *priv;                // 具体驱动私有指针（spi_nor / nand_chip）
};
```

`mtd_info` 是 SPI-NOR / SPI-NAND / Raw NAND 三种介质对上层的统一接口。`_read / _write / _erase` 指向的具体函数由后端填：

| 后端 | `_read` | `_write` | `_erase` |
| --- | --- | --- | --- |
| SPI-NOR | `spi_nor_read` | `spi_nor_write` | `spi_nor_erase` |
| SPI-NAND | `spinand_mtd_read` | `spinand_mtd_write` | `spinand_mtd_erase` |
| Raw NAND | `nand_read` | `nand_write` | `nand_erase` |

`mtd_info` 通过 `mtd_device_register()` 挂到全局链表，`get_mtd_device_nm("nor0")` 按名字取出来。

### 3.2 `struct spi_nor` — SPI-NOR 私有结构

```
// drivers/mtd/spi/sf_internal.h（简化）
struct spi_nor {
    struct mtd_info     mtd;             // 内嵌 mtd_info
    struct spi_slave    *spi;
    const struct flash_info *info;
    u32 page_size;
    u32 addr_width;                       // 3 或 4 字节地址
    u8  erase_opcode;
    u8  read_opcode;
    u8  read_dummy;
    u8  program_opcode;
    enum spi_nor_protocol read_proto;     // 1-1-1 / 1-1-4 / 4-4-4 ...
    // ...
    int (*read_reg)(struct spi_nor *nor, u8 opcode, u8 *buf, int len);
    int (*write_reg)(struct spi_nor *nor, u8 opcode, u8 *buf, int len);
    ssize_t (*read)(struct spi_nor *nor, loff_t from, size_t len, u_char *buf);
    ssize_t (*write)(struct spi_nor *nor, loff_t to, size_t len, const u_char *buf);
    int (*erase)(struct spi_nor *nor, loff_t offs);
};
```

`spi_nor_scan()` 在 probe 时分配这个结构，读 JEDEC ID、匹配 `spi_nor_ids[]`、解析 SFDP，最后填 `mtd` 字段并挂到 MTD 全局链表。

`struct spi_flash` 是 `sf` 命令用的旧 API，是 `spi_nor` 的兼容包装（`include/spi_flash.h`）。

### 3.3 `struct nand_chip` — NAND 私有结构

```
// include/linux/mtd/rawnand.h（简化）
struct nand_chip {
    struct mtd_info mtd;
    void __iomem *IO_ADDR_R;
    void __iomem *IO_ADDR_W;
    uint8_t (*read_byte)(struct mtd_info *mtd);
    void (*write_buf)(struct mtd_info *mtd, const uint8_t *buf, int len);
    void (*cmd_ctrl)(struct mtd_info *mtd, int dat, unsigned int ctrl);
    struct nand_ecc_ctrl ecc;
    int chipsize;
    int pagemask;
    // ...
};
```

Raw NAND 强依赖 `nand_ecc_ctrl`：控制器侧做 BCH / Hamming，或者软件 ECC。SPI-NAND 因为芯片内置 on-die ECC，`nand_chip` 里 ECC 字段大部分留空。

### 3.4 `struct mmc` — MMC / SD 核心结构

```
// include/mmc.h（简化）
struct mmc {
    struct list_head link;
    const struct mmc_ops *ops;   // send_cmd / set_ios / get_cd / get_wp
    void *priv;
    uint version;
    struct mmc_config *cfg;
    int high_capacity;
    uint bus_width;
    uint clock;
    uint tran_speed;
    uint read_bl_len;            // 通常 512
    uint write_bl_len;
    uint erase_grp_size;
    u64 capacity;
    struct blk_desc *block_dev;  // 挂到块设备系统
};

struct mmc_ops {
    int (*send_cmd)(struct udevice *dev, struct mmc_cmd *cmd,
                    struct mmc_data *data);
    int (*set_ios)(struct udevice *dev);
    int (*get_cd)(struct udevice *dev);
    int (*get_wp)(struct udevice *dev);
    int (*execute_tuning)(struct udevice *dev, uint opcode);
};
```

`mmc_ops` 是主控器驱动实现的接口，MMC 协议层（`drivers/mmc/mmc.c`）调 `mmc->ops->send_cmd` 发命令，主控器驱动负责翻译成寄存器操作。

### 3.5 `struct blk_desc` — 块设备描述

```
// include/blk.h（简化）
struct blk_desc {
    enum uclass_id  uclass_id;
    int             devnum;
    unsigned char   part_type;
    lbaint_t        lba;         // 总扇区数
    unsigned long   blksz;       // 通常 512
    int             (*block_read)(struct udevice *dev, lbaint_t start,
                                  lbaint_t blkcnt, void *buffer);
    int             (*block_write)(struct udevice *dev, lbaint_t start,
                                   lbaint_t blkcnt, const void *buffer);
    int             (*block_erase)(struct udevice *dev, lbaint_t start,
                                   lbaint_t blkcnt);
};
```

块设备接口跟 MTD 完全独立。`mmc` 命令走 `blk_dread` → `blk_desc->block_read` → 具体主控器的 `mmc_bread`。

### 3.6 数据结构总览

```
上层命令
    │
    ├── sf   ── spi_flash (兼容层) ── spi_nor ── mtd_info._read = spi_nor_read
    │                                     └── spi_mem_op ── SPI 控制器 exec_op
    │
    ├── nand ── nand_chip  ────────────── mtd_info._read = nand_read
    │
    ├── mtd  ── (分派) ────────── mtd_info._read (spi_nor / spinand / nand)
    │
    └── mmc  ── blk_desc.block_read ── mmc.ops->send_cmd ── SDHCI 寄存器
```

flash 类三条上层命令最终都汇聚到 `mtd_info`，MMC 独走 `blk_desc`。这就是骨-筋-皮里那根**骨**的价值。

---

### 3.7 为什么要三张抽象牌，不能合并成两张

看完前面六个结构体，读者可能会问：既然 `struct spi_nor` / `struct nand_chip` / `struct mmc` 都内嵌了 `mtd_info` 或挂到 `blk_desc`，为什么不能只留 `mtd_info` 一张牌，或者干脆全部用 `blk_desc`？

**答案在读写单位。** 三种介质的物理语义决定了必须暴露不同的操作粒度：

-   -   **SPI-NOR**：写单位 = 1 byte（page program 前不需要预擦），擦单位 = 4KB 或 64KB。可以随机寻址随机读，写要先擦
-   -   **NAND**：写单位 = 2KB / 4KB / 8KB page，且**同一 page 不能写第二次**（需要先擦整个 128KB block）。带 OOB 区做 ECC，读写都得跟 OOB 联动
-   -   **eMMC / SD**：读写单位 = 512 byte block，芯片内部 FTL 隐藏了 flash 的擦除概念，主机看到的是硬盘语义

要是把三种介质压平到同一个 `blk_desc`（只暴露 `block_read` / `block_write`），Raw NAND 的 OOB 就藏不住，UBI 层没法做磨损均衡；反过来要是全部用 `mtd_info` 的 `_read / _write / _erase`，eMMC 上层（fatload / ext4load）就得每次手动算 LBA→byte 偏移，还要面对根本不存在的 `_erase`。

所以三张牌各自守自己的抽象层次：

-   -   `mtd_info` — **flash 类物理语义**：字节地址 + 擦除粒度 + OOB
-   -   `spi_nor` / `nand_chip` — **介质私有细节**：JEDEC 命令码、地址宽度、ECC 引擎
-   -   `blk_desc` — **块设备语义**：LBA + 定长扇区，对上层像硬盘

`spi_nor` 内嵌 `mtd_info` 是**继承关系**（is-a），把私有字段挂在门面结构体后面；`mmc` 通过 `block_dev` 指针挂 `blk_desc` 是**组合关系**（has-a），因为 MMC 协议层跟块设备接口本就是两码事。这两种耦合方式对应了两种抽象目的：MTD 侧要复用 `mtd_read` 分派、MMC 侧要复用块设备栈。

## 四、框架分层：DM uclass 怎么串起来

### 4.1 相关 uclass

flash 子系统涉及的 uclass：

| uclass | 定义位置 | 承载对象 |
| --- | --- | --- |
| `UCLASS_SPI` | `drivers/spi/spi-uclass.c` | SPI 总线控制器（QSPI、DSPI） |
| `UCLASS_SPI_FLASH` | `drivers/mtd/spi/sf-uclass.c` | SPI-NOR 芯片 |
| `UCLASS_MTD` | `drivers/mtd/mtd-uclass.c` | Raw NAND 控制器（SPI-NAND 通过 spi-mem 挂 UCLASS\_SPI） |
| `UCLASS_MMC` | `drivers/mmc/mmc-uclass.c` | MMC 主控器 |
| `UCLASS_BLK` | `drivers/block/blk-uclass.c` | 块设备（MMC / USB / SCSI 都挂这里） |

### 4.2 DT + uclass 的挂载关系

以一片 SPI-NOR 为例，源码里的挂载路径：

```
DT: soc { qspi@e000d000 { flash@0 { compatible = "jedec,spi-nor"; }; }; }

      │ dm_scan_fdt() 递归遍历
      ▼
UCLASS_SPI 驱动（如 zynq_qspi.c）绑定 qspi@e000d000 节点
      │ probe 时创建 spi_slave
      ▼
UCLASS_SPI_FLASH 驱动（sf_probe.c）绑定 flash@0 子节点
      │ probe → spi_nor_scan()
      ▼
分配 struct spi_nor + 填 mtd_info
      │ mtd_device_register()
      ▼
挂到 MTD 全局链表，mtd_probe_devices() 后可用
```

MMC 主控器的挂载更简单：`UCLASS_MMC` 驱动直接创建 `struct mmc` 并注册 `blk_desc`，跳过中间层。

### 4.3 `spi-mem` 抽象

`drivers/spi/spi-mem.c` 是 v2019 引入的关键抽象。它定义了 `struct spi_mem_op`：

```
struct spi_mem_op {
    struct { u8 opcode; u8 buswidth; } cmd;
    struct { u8 nbytes; u8 buswidth; u64 val; } addr;
    struct { u8 nbytes; u8 buswidth; } dummy;
    struct { u8 buswidth; enum spi_mem_data_dir dir;
             unsigned int nbytes; void *buf; } data;
};
```

一次 SPI 存储器操作被拆成 cmd / addr / dummy / data 四段，各段可以独立选择 buswidth（1/2/4/8 线）。控制器驱动只要实现 `mem_ops->exec_op`，SPI-NOR 和 SPI-NAND 都能复用。老式 `xfer` 回调仍然保留兼容，但新驱动都走 spi-mem。

### 4.4 命令层跟 uclass 的挂钩

四条命令在源码里跟 uclass 的挂钩：

-   -   `sf probe <bus> <cs>` → `spi_flash_probe(bus, cs, ...)` → `uclass_get_device_by_seq(UCLASS_SPI, bus, ...)` → 找到子设备
-   -   `mtd list` → 遍历 MTD 全局链表（不走 uclass）
-   -   `mmc dev <n>` → `mmc_get_dev(n)` → `uclass_get_device_by_seq(UCLASS_MMC, n, ...)`
-   -   `nand device <n>` → 遍历静态 `nand_info[]` 数组（DM 迁移未完成）

这也解释了四条命令**为什么长得不一样**：`sf` 和 `mmc` 已经完成 DM 迁移，`mtd` 走全局链表，`nand` 还留有非 DM 遗产。

---

![[Inbox/笔记同步助手/微信公众号/2026/07/images/76059186d198684469260d6d2d6f2e36_MD5.jpg]]

Image

### 4.5 四条命令为什么至今没统一

`sf` / `nand` / `mtd` / `mmc` 明明前三条都操作 flash 介质，为什么源码里保留四个入口？看似冗余，其实是**兼容压力**和**语义差异**的合力。

先看历史。U-Boot 从 2000 年前后有 `nand` 命令，2005 年前后加 `sf`，2010 年后 DM 模型成型，2015 年前后引入 `mtd` 想统一，2019 年 `spi-mem` 抽象成型。四条命令的实现分别对应了四个时代的 U-Boot 内部设施——`nand` 和 `sf` 是"直系派"，直接调各自介质的操作函数；`mtd` 是"统一派"，走 `mtd_read` 分派；`mmc` 独立于 MTD 栈，走块设备接口。

再看**为什么迁移不彻底**：

-   -   **sf**：太老、太广。BSP 厂商、板卡默认脚本、启动日志、教程全部依赖 `sf probe` / `sf read` / `sf erase`。硬要拿掉会打断所有下游用户。所以 `sf` 至今是个薄壳——命令解析后 v2020 起改成走 `mtd_read`，但**用户界面**保留原样
-   -   **nand**：还没完成 DM 迁移。`nand device <n>` 至今遍历静态 `nand_info[]` 数组，不走 `uclass_get_device_by_seq`。原因是 Raw NAND 驱动的 ECC 引擎种类多、老代码沉淀重，DM 化改动量太大，没人接手
-   -   **mtd**：新增的统一入口，是新代码推荐的选择。但要求 `struct mtd_info` 已经通过 `mtd_device_register` 挂进全局链表，SPI-NOR 需要显式 `sf probe` 触发这一步才能被 `mtd list` 看到
-   -   **mmc**：从一开始就走块设备栈，跟 MTD 栈无交集。上层可以 `fatload mmc 0:1 addr file` 直接用文件系统语义，用户体验最舒服

所以四条命令留存到 v2024.07 不是设计冗余，是**渐进式重构还没走完**的中间状态。上层调用者要理解每条命令的实际去处：`sf` 内部现在其实也走 `mtd_read`（v2020 后），只是命令名保留下来做兼容层；`mtd list` 看不到 SPI-NOR 是因为 probe 还没触发；`nand` 是唯一还没进 DM 的老兵。

  

## 五、启动数据流：命令到硬件

拆两条最典型的路径：`sf read` 和 `mmc read`。SPI-NAND 和 Raw NAND 是 `sf read` 的变种。

### 5.1 `sf read` 数据流

用户命令：

```
=> sf probe
=> sf read 0x10000000 0x100000 0x400000
```

调用栈：

```
do_spi_flash_read                      cmd/sf.c
  spi_flash_read                       include/spi_flash.h (inline → mtd_read)
    mtd_read                           drivers/mtd/mtdcore.c
      mtd->_read = spi_nor_read        drivers/mtd/spi/spi-nor-core.c
        spi_nor_read_data
          spi_nor_read_data_op
            spi_mem_exec_op            drivers/spi/spi-mem.c
              mem_ops->exec_op         SPI 控制器驱动
```

关键点：

-   -   `spi_flash_read` 早期直接调 `spi_nor_read`，v2020 后统一走 `mtd_read`
-   -   `spi_mem_exec_op` 会先尝试控制器 `mem_ops->exec_op`，失败才 fallback 到老 `xfer` 回调
-   • 单次 op 的数据量受控制器 FIFO / DMA 限制，`spi-nor-core.c` 里有 `spi_nor_adjust_op_size` 做自动分段
    

**举例：Zynq-7000 QSPI**。控制器驱动 `drivers/spi/zynq_qspi.c` 里 `zynq_qspi_exec_op()` 的做法：把 opcode + address + dummy 组装成 TX 缓冲，写 QSPI Configuration Register 使能 manual start，塞进 63 字节深的 TX FIFO（超出要分段），启动传输后轮询 RX FIFO not empty，最后把 RX FIFO 里的字节搬到 `op->data.buf`。

ZynqMP 上换成 `drivers/spi/zynqmp_gqspi.c`，做法不同：GQSPI 硬件里有一个 **generic FIFO**，CPU 把一系列 32-bit 描述符（`GQSPI_EXP`、`GQSPI_TX_FIFO`、`GQSPI_RX_FIFO`）塞进去，硬件按序执行整个 op。驱动的工作是把 `spi_mem_op` 翻译成描述符序列。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3e41b49ac84fc6bc91860b8a52ff405f_MD5.jpg]]

Image

### 5.2 `mmc read` 数据流

```
=> mmc dev 0
=> mmc read 0x10000000 0x200 0x2000
```

调用栈：

```
do_mmc_read                            cmd/mmc.c
  blk_dread                            drivers/block/blk-uclass.c
    blk_desc->block_read = mmc_bread   drivers/mmc/mmc-uclass.c → mmc_bread
      mmc_read_blocks                  drivers/mmc/mmc.c
        mmc_send_cmd                    (CMD17 / CMD18 + CMD23)
          mmc->cfg->ops->send_cmd
            sdhci_send_command          drivers/mmc/sdhci.c
              → 写 SDHCI_ARGUMENT
              → 写 SDHCI_TRANSFER_MODE (DMA / block count / auto CMD)
              → 写 SDHCI_COMMAND 触发
              → 等待 SDHCI_INT_STATUS 里的 XFER_COMPLETE
              → ADMA2 / SDMA 把数据搬到 buffer
```

关键点：

-   -   `mmc_read_blocks` 会根据主控能力决定用 CMD17（单块）还是 CMD18（多块）+ CMD23（预设块数）
-   -   `sdhci.c` 是通用层，各家主控只填 `sdhci_ops` 里的 `set_ios` / `set_delay` / `platform_execute_tuning`
-   • 高速模式（HS200 / HS400）要 tuning：驱动连续发 CMD21，采样多个相位窗口取中间值
    

**举例：Zynq / ZynqMP SDHCI**。`drivers/mmc/zynq_sdhci.c` 复用 `sdhci.c` 的绝大部分逻辑，只添加三个平台特定回调：`arasan_sdhci_execute_tuning`（HS200 / HS400 phy tuning，读写 SLCR 里的 `SD*_ITAPDLY` / `SD*_OTAPDLY` 微调时钟相位）、`arasan_sdhci_set_tapdelay`（不同速率模式下写入预设 tap 值）、`zynq_sdhci_probe`（解析 DT 的 `arasan,soc-ctl-syscon` 属性拿到 SLCR handle）。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/740abb4d4f6cc8795fd1e6706286082f_MD5.jpg]]

Image

### 5.3 两条路径的对比

| 环节 | SPI-NOR ‵sfread‵ | eMMC ‵mmcread‵ |
| :-- | :-- | :-- |
| 上层接口 | `mtd_read` | `blk_dread` |
| 命令组装 | `spi_mem_op`（cmd/addr/dummy/data 四段） | `mmc_cmd`（opcode + arg + resp） |
| 硬件层 | FIFO / DMA（控制器决定） | ADMA2 / SDMA（SDHCI 规范定义） |
| 单次传输 | 控制器 FIFO 深度限制，分段 | 描述符表决定，一次可 MB 级 |
| 时钟切换 | flash 芯片吃到什么算什么 | 主控 tuning 找相位 |
| 错误处理 | 上层 CRC / OP 重试 | XFER\_COMPLETE / ERROR 中断 |

### 5.4 两条路径的性能推导

对比表看着并列，实际 boot 阶段 eMMC 比 SPI-NOR 快一到两个数量级。数字算一算就明白。

**SPI-NOR（Zynq-7000 QSPI + n25q256a，Quad Read）**：

-   • 时钟：100 MHz、Quad IO（4 线数据）
    
-   • 有效带宽：100 M × 4 / 8 = **50 MB/s** 理论上限
    
-   • 实际打折：每次 op 有 opcode + 24/32-bit address + dummy cycles 的开销；控制器 63 字节 TX FIFO 要分段；CPU 轮询 RX FIFO 忙等
    
-   • 实测 boot 阶段：**8–15 MB/s** 是常见范围。加载一个 20 MB 的 Image 大概 1.5–2.5 秒
    

**eMMC（Zynq SDHCI，HS200）**：

-   • 时钟：200 MHz、8-bit data bus
    
-   • 有效带宽：200 M × 8 / 8 = **200 MB/s** 理论上限
    
-   • 加成：ADMA2 描述符表让 CPU 只需要设置一次，硬件自动完成整段搬运；主控 XFER\_COMPLETE 中断（U-Boot 里降级为轮询，但轮询周期比 SPI 少很多）
    
-   • 实测 boot 阶段：**60–120 MB/s**（HS200），HS400 还能翻倍。加载 20 MB Image 只要 150–300 毫秒
    

**差在哪里**（不是控制器好坏，是**协议本身**）：

1.  1.  **数据线数量**：QSPI 4 线 vs eMMC 8 线，物理带宽差 2×
2.  2.  **命令开销**：SPI 每次操作要发 1 字节 opcode + 3/4 字节地址 + N 字节 dummy，30 字节的 header 才换来一次 burst read；eMMC 一次 CMD18 + CMD23 就能连续读几 MB
3.  3.  **CPU 参与度**：SPI 传输 CPU 得反复读 RX FIFO，DMA 支持在 U-Boot 阶段常常没打开；eMMC 走 ADMA2，CPU 提交一次描述符表就等中断
4.  4.  **时钟切换**：SPI 芯片吃到什么速率就跑什么速率，多数板卡受 PCB 走线限制卡在 50–100 MHz；SDHCI 的 HS200 tuning 可以把时钟推到 200 MHz 并做相位补偿

这也是为什么"boot.bin 放 QSPI + kernel 放 eMMC"成为 ZynqMP 量产板的标配——boot.bin 只有几百 KB，QSPI 的慢无所谓；kernel + rootfs 动辄几十 MB，必须走 eMMC 才有可用体验。

### 5.5 `mtd read` 统一分派

`mtd read` 是后期引入的统一入口，走 `mtd_read` 分派到具体后端：

-   -   `_read = spi_nor_read` → SPI-NOR 走 spi-mem
-   -   `_read = spinand_mtd_read` → SPI-NAND 也走 spi-mem，但需要处理 ECC
-   -   `_read = nand_read` → Raw NAND 走各家控制器的 `read_page`

上层不感知后端类型，这是 MTD 抽象层最直接的收益。

---

## 六、初始化流程：从 `board_init_r` 到设备就绪

`board_init_r()` 在 `common/board_r.c` 里定义了一个 `init_sequence_r[]` 数组，flash 相关的调用点分几批出现。

### 6.1 DM 扫描阶段

```
board_init_r()
  → initr_dm()                       激活 DM 核心
  → dm_extended_scan()               二次扫描 DT
      → 遍历 DT 节点，按 compatible 匹配驱动
      → SPI 控制器节点 → 绑定 UCLASS_SPI 驱动
      → SPI-NOR 子节点 → 绑定 UCLASS_SPI_FLASH 驱动
      → MMC 控制器节点 → 绑定 UCLASS_MMC 驱动
```

DM 只完成 **bind**（分配 udevice、找到驱动），不 probe。probe 延迟到首次使用时触发（lazy probe）。

### 6.2 MTD 探测

```
→ initr_mtd()
      → mtd_probe_devices()          drivers/mtd/mtd-uclass.c
          → uclass_foreach_dev(UCLASS_MTD)
          → device_probe(dev)        触发驱动的 probe 回调
```

只有 `UCLASS_MTD`（Raw NAND / SPI-NAND 控制器）在这里主动 probe。SPI-NOR 走 `UCLASS_SPI_FLASH`，不在这个循环里，靠 `env_load` 或首次 `sf probe` 触发。

### 6.3 MMC 初始化

```
→ initr_mmc()                      common/board_r.c
      → mmc_initialize()             drivers/mmc/mmc.c
          → for each UCLASS_MMC device:
              → mmc_start_init()
                  → CMD0 (reset)
                  → CMD8 (voltage check)
                  → ACMD41 (SD) or CMD1 (eMMC) 上电协商
                  → CMD2 (get CID)
                  → CMD3 (assign RCA)
                  → CMD9 (read CSD)
                  → CMD7 (select card)
              → mmc_complete_init()
                  → 切换总线宽度（4/8-bit）
                  → 切换速率（HS / HS200 / HS400）
                  → tuning（如支持）
              → 注册 blk_desc
```

`mmc_start_init` 是 MMC 上电协议的完整实现。日志里那行 `MMC: mmc@e0100000: 0` 就是这个函数返回成功的标志。

### 6.4 环境变量装载

```
→ initr_env()
      → env_relocate()               把 default env 拷到 heap
      → env_load()                    common/env.c
          → 根据 CONFIG_ENV_IS_IN_* 选 backend
          → env_sf_load / env_mmc_load / env_nand_load
```

`env_sf_load` 在 `env/sf.c`：

```
static int env_sf_load(void)
{
    struct spi_flash *env_flash;
    env_flash = spi_flash_probe(CONFIG_ENV_SPI_BUS,
                                CONFIG_ENV_SPI_CS,
                                CONFIG_ENV_SPI_MAX_HZ,
                                CONFIG_ENV_SPI_MODE);
    // 从 CONFIG_ENV_OFFSET 读 CONFIG_ENV_SIZE 字节
    // CRC 校验 → env_import
}
```

`spi_flash_probe()` 内部会触发前面提到的完整 probe 链：`UCLASS_SPI` → `UCLASS_SPI_FLASH` → `spi_nor_scan` → 填 `mtd_info`。这是第一次实际的 SPI-NOR 硬件访问。

### 6.5 `spi_nor_scan` 的三段握手

```
// drivers/mtd/spi/spi-nor-core.c（简化）
int spi_nor_scan(struct spi_nor *nor)
{
    // 1. 读 JEDEC ID (opcode 0x9F)
    spi_nor_read_id(nor, id);
    info = spi_nor_read_id_match(id);  // 查 spi_nor_ids[]

    // 2. 解析 SFDP (opcode 0x5A) — 可选
    if (info->flags & SPI_NOR_SKIP_SFDP)
        goto skip_sfdp;
    spi_nor_parse_sfdp(nor);            // 覆盖 info 里的默认值

    // 3. Setup：根据 ID + SFDP 结果选定
    //    - read_opcode: 0x03 / 0x0B / 0x6B (Quad Read)
    //    - erase_opcode: 0x20 (4KB) / 0xD8 (64KB)
    //    - addr_width: 3 或 4
    //    - 激活 QE 位 / 进入 4-byte mode
    spi_nor_setup(nor);

    // 4. 填 mtd_info 并注册
    nor->mtd._read  = spi_nor_read;
    nor->mtd._write = spi_nor_write;
    nor->mtd._erase = spi_nor_erase;
    return mtd_device_register(&nor->mtd, NULL, 0);
}
```

JEDEC ID 匹配是**兜底**：所有 SPI-NOR 都必须实现 `0x9F`，返回 3 字节 $manufacturer,memorytype,capacity$。SFDP 是**升级**：JESD216 规范定义的一张表格，flash 里预烧了自己的能力描述，让驱动动态适配未知型号。

**举例：Zynq QSPI 的 4-byte 地址坑**。Zynq-7000 QSPI 控制器默认走 24-bit 地址，对 32MB 以上的 flash（如 Micron n25q256a）必须发 `0xB7`（Enter 4-byte mode）。`spi_nor_setup` 里通过 `spi_nor_set_4byte()` 处理，但 board defconfig 要保证 `SPI_FLASH_BAR` 或 `SPI_FLASH_4B_OPCODES` 打开，否则读取会跨在 24-bit 边界处错乱。

---

![[Inbox/笔记同步助手/微信公众号/2026/07/images/00f7e068bb9d4ab9d0d18fd85ec25ab3_MD5.jpg]]

Image

### 6.6 常见启动失败与分层排错

`board_init_r` 里 flash 相关的初始化点串起来的这条链，也决定了故障的**定位维度**。三类现象各对应一个层次：

**现象 A：SF: Detected 打不出来**（SPI-NOR 没探到）

-   • 定位层：DM 扫描或 `spi_nor_scan`
    
-   • 常见根因：
    

-   • DT 里 `flash@0` 缺 `compatible = "jedec,spi-nor"`——DM 找不到匹配驱动，`UCLASS_SPI_FLASH` 根本没绑定
    
-   • JEDEC ID 0xFF/0xFF/0xFF——硬件层握手失败，可能 QSPI 时钟没开、`SPI_FLASH_BAR` 配置错、板卡电源没上
    
-   • JEDEC ID 匹配了但 `spi_nor_ids[]` 里没这型号，且 SFDP 也没实现——SFDP 也读不出来时 `spi_nor_scan` 返回 `-ENODEV`
    

-   • 排查手法：先看 `dm tree` 里有没有 spi-flash 节点，再手动 `sf probe` 看具体报错
    

**现象 B：Card did not respond to voltage select**（eMMC 上电失败）

-   • 定位层：`mmc_start_init` 里的 CMD1（eMMC）或 ACMD41（SD）
    
-   • 常见根因：
    

-   -   `mmc-ddr-1_8v` / `mmc-hs200-1_8v` 属性配置错，VCCQ 电压没切
-   -   `clock-frequency` 太高，初始化阶段应该用 400 KHz 慢速握手
-   • SDHCI clock domain 的 gate 没打开（Zynq 上是 SLCR 里的 `SDIO*_CLK_CTRL`）
    

-   • 排查手法：`mmc dev 0` 看返回码，`mmc info` 打不出来就是 `mmc_start_init` 挂了
    

**现象 C：sf read 能读但内容乱**（读到的字节不对）

-   • 定位层：`spi_nor_setup` 或控制器 `exec_op`
    
-   • 常见根因：
    

-   • 32MB 以上的 flash 但没开 4-byte 模式，24-bit 地址回卷到 flash 头
    
-   • Quad Enable 位没写进 Status Register，控制器发 Quad Read 但芯片只在 Single mode 应答
    
-   • dummy cycles 数量对不上——SFDP 声明的和实际编程的不一致
    

-   • 排查手法：`sf read` 前先 `md.b <addr> 20` 看开头是不是全 FF，`md.b <addr+0x100000> 20` 看有没有回卷
    

**分层排错的核心思路**：从上到下按调用栈判断——命令进入了吗？分派到正确的 `_read` 了吗？介质探测通过了吗？控制器发对命令了吗？每一步都对应源码里明确的返回码，不用瞎猜。

## 七、平台适配：三个切入点

一款新 SoC 要接入 flash 子系统，源码上有三个切入点。以 Zynq / ZynqMP 为例。

### 7.1 切入点 A：SPI 控制器驱动（挂 `UCLASS_SPI`）

关键文件：

-   -   `drivers/spi/zynq_qspi.c` — Zynq-7000
-   -   `drivers/spi/zynqmp_gqspi.c` — ZynqMP（新架构 GQSPI）

驱动骨架：

```
static const struct spi_controller_mem_ops zynq_qspi_mem_ops = {
    .exec_op = zynq_qspi_exec_op,
};

static const struct dm_spi_ops zynq_qspi_ops = {
    .claim_bus  = zynq_qspi_claim_bus,
    .release_bus= zynq_qspi_release_bus,
    .xfer       = zynq_qspi_xfer,           // 老接口兼容
    .set_speed  = zynq_qspi_set_speed,
    .set_mode   = zynq_qspi_set_mode,
    .mem_ops    = &zynq_qspi_mem_ops,        // 新接口
};

U_BOOT_DRIVER(zynq_qspi) = {
    .name    = "zynq_qspi",
    .id      = UCLASS_SPI,
    .of_match= zynq_qspi_ids,
    .ops     = &zynq_qspi_ops,
    .probe   = zynq_qspi_probe,
    .priv_auto = sizeof(struct zynq_qspi_priv),
};
```

关键点：新驱动同时实现 `xfer` 和 `mem_ops->exec_op`。`spi-mem.c` 优先走 `exec_op`，失败才 fallback 到 `xfer`。

寄存器基址：Zynq-7000 QSPI 是 `0xE000_D000`；ZynqMP GQSPI 是 `0xFF0F_0000`（LPD 域）。控制器差异：Zynq-7000 QSPI 支持 Single SS / Dual Stacked / Dual Parallel 三种模式，`is-stacked` / `is-dual-parallel` DT 属性驱动内部识别；GQSPI 支持 x1/x2/x4/x8，最高 200 MHz。

### 7.2 切入点 B：MMC 主控器驱动（挂 `UCLASS_MMC`）

关键文件：

-   -   `drivers/mmc/zynq_sdhci.c` — Zynq / ZynqMP
-   • 依赖：`drivers/mmc/sdhci.c`（通用 SDHCI 层）
    

驱动骨架：

```
static const struct sdhci_ops arasan_ops = {
    .platform_execute_tuning = arasan_sdhci_execute_tuning,
    .set_delay = arasan_sdhci_set_tapdelay,
    .set_control_reg = &sdhci_set_control_reg,
};

static int arasan_sdhci_probe(struct udevice *dev)
{
    struct sdhci_host *host = ...;
    host->ops = &arasan_ops;
    ret = sdhci_setup_cfg(&plat->cfg, host, max_clk, min_clk);
    upriv->mmc = &host->mmc;
    return sdhci_probe(dev);
}

U_BOOT_DRIVER(arasan_sdhci_drv) = {
    .name    = "arasan_sdhci",
    .id      = UCLASS_MMC,
    .of_match= arasan_sdhci_ids,
    .ops     = &sdhci_ops,
    .bind    = arasan_sdhci_bind,
    .probe   = arasan_sdhci_probe,
    .priv_auto = sizeof(struct arasan_sdhci_priv),
};
```

关键点：Arasan SDHCI 只需要填三个平台特定回调（tuning、tap delay、control register），其余全部复用 `sdhci.c`。这也是为什么加一款新 SoC 支持通常只要几百行代码。

寄存器基址：Zynq-7000 SD0 是 `0xE010_0000`、SD1 是 `0xE010_1000`。ZynqMP 在 `0xFF16_0000` / `0xFF17_0000`。

### 7.3 切入点 C：NAND 控制器驱动（挂 `UCLASS_MTD`）

-   -   `drivers/mtd/nand/raw/zynq_nand.c` — Zynq-7000 SMC-based parallel NAND
-   -   `drivers/mtd/nand/raw/arasan_nfc.c` — ZynqMP Arasan NAND Flash Controller

Raw NAND 驱动比 SPI 复杂，主要因为 ECC。Zynq 的 SMC 支持硬件 BCH，`zynq_nand.c` 里 `zynq_nand_calc_hw_ecc` / `zynq_nand_correct_data` 分别负责编码校验和纠错。

NAND 因为坏块管理 + ECC 复杂度，在 boot 阶段用得越来越少，本文不再展开。

---

### 7.4 一款新 SoC 的移植工作量估算

看完三个切入点，可以反过来估算新 SoC 接入的成本。以做过几款 SoC 的经验：

**SPI 控制器（切入点 A）**：**500–1500 行**

-   • 大头是 `exec_op` 的实现——把 `spi_mem_op` 里的 cmd / addr / dummy / data 翻译成控制器的 FIFO / 描述符操作
    
-   • 时钟设置、复位、GPIO chip-select 补贴百来行
    
-   • 支持 DMA 再加 200–500 行
    
-   • 参考模板：`zynq_qspi.c`（约 800 行）、`cadence_qspi.c`（约 900 行）
    

**MMC 主控器（切入点 B）**：**200–800 行**

-   • 用 SDHCI 兼容主控（大多数 ARM SoC）——填 `sdhci_ops` 里 tuning、tap delay、control register 三个回调即可
    
-   • 非 SDHCI 主控（如 Rockchip DWMMC、MediaTek MSDC）就得从 `mmc_ops.send_cmd` 开始自己实现，工作量翻倍到 1000–2000 行
    
-   • 参考模板：`zynq_sdhci.c`（约 400 行）、`fsl_esdhc.c`（约 1500 行，非 SDHCI）
    

**NAND 控制器（切入点 C）**：**1500–3000 行**

-   • Raw NAND 复杂度主要来自 ECC——BCH 编码/译码要么全靠硬件（500 行接口代码），要么部分软件回退（多加 1000 行）
    
-   • 坏块管理、OOB 布局、ONFI 探测都要处理
    
-   • 参考模板：`zynq_nand.c`（约 1300 行）、`arasan_nfc.c`（约 1700 行）
    

**再加上共同的必做项**（跟切入点无关）：

-   • 设备树 binding 文档（`Documentation/devicetree/bindings/`）
    
-   • board defconfig 里开对应的 `CONFIG_*`
    
-   • 板卡 DTS 里加节点
    
-   • （可选）SPL 阶段简化版驱动，走 `drivers/mtd/spi/sf_mtd.c` 或 `drivers/mmc/*_spl.c`
    

**经验教训**：

-   • 90% 的移植 bug 出在 DT 和时钟配置上，不在驱动逻辑本身。上电顺序、reset 释放、clock gate 是常见的隐性依赖
    
-   • Zynq / ZynqMP 之所以驱动写起来轻——因为 Arasan / Xilinx 的 IP 直接兼容 SDHCI 规范和 SPI-mem 抽象，"造抽象的人当年就参考了这套 IP"
    
-   • 首次点亮新平台的 SPI-NOR 时，建议先用 `sspi` 命令（`cmd/spi.c` 提供的裸 SPI 传输）验证 CS / 时钟 / 数据线通了，再上 `sf probe`
    

## 八、常见启动模式与源码路径

四条命令树在实际 boot 流程里的组合方式。以 Zynq 系为例。

### 8.1 模式 A：SPI-NOR-only

**流程**：BootROM → FSBL（QSPI）→ U-Boot（QSPI）→ `env_sf_load` → `bootcmd` 调 `sf read` 加载 kernel。

**源码走向**：

-   • 环境变量：`env/sf.c::env_sf_load`
    
-   • 命令：`cmd/sf.c::do_spi_flash_read` → `mtd_read` → `spi_nor_read`
    
-   • 硬件：`drivers/spi/zynq_qspi.c::zynq_qspi_exec_op`
    

**典型 defconfig**：

```
CONFIG_ENV_IS_IN_SPI_FLASH=y
CONFIG_ENV_OFFSET=0x600000
CONFIG_ENV_SECT_SIZE=0x10000
CONFIG_ENV_SIZE=0x8000
```

### 8.2 模式 B：SPI-NOR + eMMC 混合

**流程**：BootROM 从 QSPI 读 boot.bin，U-Boot 起来后从 eMMC 加载 kernel / rootfs。ZynqMP 量产板卡最常见的组合。

**源码走向**：

-   • 环境变量可在 eMMC boot partition：`env/mmc.c::env_mmc_load`
    
-   • 命令：`cmd/mmc.c::do_mmc_read` → `blk_dread` → `mmc_bread` → `mmc_send_cmd`
    
-   • 硬件：`drivers/mmc/zynq_sdhci.c` + `drivers/mmc/sdhci.c`
    

**typical bootcmd**：

```
mmc dev 0
ext4load mmc 0:1 0x10000000 /boot/Image
ext4load mmc 0:1 0x11000000 /boot/system.dtb
booti 0x10000000 - 0x11000000
```

**注意**：eMMC 的 boot partition（boot0 / boot1）跟 user data area 分开，写 boot partition 要 `mmc partconf 0 1 1 1` 切进去。BootROM 根据 mode strap 决定从 boot partition 还是 user area 加载。

### 8.3 模式 C：SPI-NAND 或 Raw NAND

**流程**：boot.bin 放在 NAND，通常配合 UBI / UBIFS 使用。

**源码走向**：

-   • 命令：`cmd/mtd.c::do_mtd_io` → `mtd_read` → `spinand_mtd_read` / `nand_read`
    
-   • SPI-NAND：`drivers/mtd/nand/spi/core.c` + `drivers/spi/zynqmp_gqspi.c`
    
-   • Raw NAND：`drivers/mtd/nand/raw/zynq_nand.c`
    

**UBI 挂载栈**：

```
mtd read (page + OOB, ECC 已由 spinand core 处理)
  → UBI (drivers/mtd/ubi/) 管理逻辑擦除块 + 磨损均衡
    → UBIFS (fs/ubifs/) 提供文件系统接口
```

### 8.4 模式 D：QSPI Dual Stacked（Zynq-7000 特色）

**流程**：两片 SPI-NOR 共享 IO，通过 CS 切换。`drivers/spi/zynq_qspi.c` 里 `is_stacked` 字段决定寻址：地址 < 单片容量走 CS0，超过走 CS1。对上层完全透明。

**DT 配置**：

```
&qspi {
    is-dual = <1>;
    is-stacked = <1>;
    num-cs = <2>;
    flash@0 { compatible = "jedec,spi-nor"; reg = <0>; };
    flash@1 { compatible = "jedec,spi-nor"; reg = <1>; };
};
```

### 8.5 四种模式的源码入口对照

| 模式 | 环境变量 backend | boot 命令 | 数据读取 backend |
| --- | --- | --- | --- |
| A: SPI-NOR only | `env/sf.c` | `sf read` | `spi_nor_read` |
| B: SPI-NOR + eMMC | `env/mmc.c` | `ext4load mmc`/ `mmc read` | `mmc_bread`→ `sdhci_send_command` |
| C: NAND | `env/nand.c` | `ubi read`/ `mtd read` | `nand_read`/ `spinand_mtd_read` |
| D: Dual Stacked | `env/sf.c` | `sf read` | `spi_nor_read`（控制器透明分片） |

---

## 九、这套框架的局限

前面八章说清了 U-Boot flash 子系统"能做什么"和"怎么做"，也应该看到"哪里做得不够好"。这些局限决定了它的定位——只是 bootloader 阶段的临时工，永远不会变成 Linux 那么完整的存储栈。

**局限一：SPL 阶段几乎全部旁路**

`board_init_r` 那套 DM + MTD 分派只在 U-Boot proper（第二阶段）跑得起来。SPL / TPL 因为镜像大小限制（几十 KB 到一百多 KB），装不下完整 DM + MTD + spi-mem。所以 SPL 走的是 `drivers/mtd/spi/sf_mtd.c` + `spi_flash_probe` 的裸接口，`env` 装载也是 `env/sf.c` 里的简化路径。上层看到的 `mtd_read` 分派在 SPL 阶段基本不存在。

代价：一份 SPI-NOR 探测逻辑要维护两个版本（SPL 简化版 + U-Boot proper 完整版），当 flash 芯片有新型号加进来时，两处都得改。

**局限二：nand 命令还没进 DM 时代**

前面说过 `nand device <n>` 遍历静态 `nand_info[]` 数组。这不只是"看着不一致"的问题——它意味着 Raw NAND 驱动跟 DM 生命周期脱钩：`bind` / `probe` / `remove` 的常规钩子用不上，DT 里如果有多个 NAND 控制器节点，得手工处理排序和索引。跟 UBI / UBIFS 挂接时还要靠 `nand_info[0]` 硬编码索引，不能靠 DT alias。

想推进 DM 化的补丁十几年前就有人提，一直卡在 Raw NAND 驱动数量太大（几十家 SoC 各写一份）、ECC 引擎接口没有统一抽象上。

**局限三：MTD 和块设备两个栈永不相交**

MMC / SD 走 `blk_desc`，SPI-NOR / NAND 走 `mtd_info`，两个栈之间没有桥。这意味着：

-   -   `fatload mmc 0:1 addr file` 能用文件系统语义读 eMMC，但同样的路子读 SPI-NOR 上的 FAT 分区就得先 `sf read` 到 DRAM 再解析
-   • UBIFS 建在 MTD 上，ext4 建在块设备上——想把 UBI 卷当块设备用得靠 `ubiblock`，U-Boot 里没有对应实现
    
-   • Linux 里的 `mtdblock` 也没有 U-Boot 版本
    

代价是 flash 类介质想用文件系统就得走 UBIFS / JFFS2 的窄路径，或者 `sf read` 到 DRAM 再挂 `fatload`。用户界面上的分裂反映的是骨架层的分裂。

**局限四：没有写路径的 wear-leveling**

`mtd_write` / `spi_nor_write` 是被动接口，写哪个 offset 就写哪个 offset。磨损均衡的责任全部推给上层（UBI 或者用户手动分片）。这在 bootloader 场景没问题——boot.bin / env / kernel 都写不了几次。但如果有人拿 U-Boot 当 flash 编程器往生产板刷镜像，就要小心：反复写同一片区会打空 flash 的擦除寿命。

**局限五：spi-mem 只覆盖 SPI 类介质**

`spi_mem_op` 是个漂亮的抽象——cmd + addr + dummy + data 四段解耦。但这个抽象只对 SPI-NOR 和 SPI-NAND 有意义，Raw NAND 用不上（并行接口没有 opcode 概念），eMMC 用不上（走 CMD 系列）。所以框架里其实并没有"统一的 flash 命令抽象"，只有"统一的 SPI 存储器命令抽象"。

**局限六：跟 Linux MTD 有隐性偏离**

U-Boot 的 `spi-nor-core.c` 在 v2019 从 Linux 移植过来，但**没跟 Linux 保持同步更新**。Linux 上后来加的 `spi-nor-sfdp.c` 拆分、`spi-nor` 变成 kernel module 化的接口、`nvmem` 提供者等特性，U-Boot 侧都没跟。这导致 Linux 能识别的一些新 flash 芯片，U-Boot 未必能识别；反过来 U-Boot 里的一些 workaround 也没回流到 Linux。

理解这些局限的价值：它们不是"缺陷"而是**取舍**。U-Boot 想保持自己 200KB 的镜像上限、想把复杂度留给 Linux，就得接受这些边界。做移植和调优时知道边界在哪里，比无谓地想把 U-Boot 变成半个 Linux 更重要。

---

## 十、小结

flash 子系统在 U-Boot 里的源码组织可以用四句话概括：

1.  1.  **MTD 抽象层是 flash 类介质的骨架**。`struct mtd_info` 用 `_read` / `_write` / `_erase` 三个函数指针统一了 SPI-NOR / SPI-NAND / Raw NAND，上层不感知后端。
2.  2.  **DM uclass 是把驱动串起来的筋络**。`UCLASS_SPI` / `UCLASS_SPI_FLASH` / `UCLASS_MTD` / `UCLASS_MMC` 四个 uclass 覆盖了从总线控制器到芯片驱动的全部层次。
3.  3.  **spi-mem 是 SPI 存储器操作的现代接口**。`struct spi_mem_op` 把 cmd / addr / dummy / data 四段打包，控制器驱动实现 `exec_op`，SPI-NOR 和 SPI-NAND 共用一套硬件路径。
4.  4.  **MMC 独走块设备栈**。`blk_desc.block_read` 是入口，`mmc_bread` 是实现，`sdhci_send_command` 是硬件层。这条栈跟 MTD 永远不交叉。

四条命令树的分工与统一：

-   -   `sf` 和 `nand` 是历史遗留的直系命令，绕过 MTD 分派
-   -   `mtd` 是后期统一入口，走 `mtd_read`，支持所有 flash 类介质
-   -   `mmc` 独立于 MTD 栈，走块设备接口

平台适配三个切入点：

-   • SPI 控制器驱动挂 `UCLASS_SPI`，实现 `mem_ops->exec_op`
    
-   • MMC 主控器驱动挂 `UCLASS_MMC`，通常复用 `sdhci.c` 通用层
    
-   • NAND 控制器驱动挂 `UCLASS_MTD`，需要处理 ECC 和坏块
    

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/36ba71de_1784288118829?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwOTUxMzQzOA%3D%3D%26mid%3D2247484386%26idx%3D1%26sn%3D54434204ebafd52a43f8574cc25fcbc1%26chksm%3Dc068af18f97459d4a766f29b9a8d4f76b19519c313def28a494a6f9ad314d3909a0d970b432a%26mpshare%3D1%26scene%3D1%26srcid%3D0717arbs6yh7I14YDpLQUk6M%26sharer_shareinfo%3Dc42dbaf3e80c0235b09d328d401a7bc5%26sharer_shareinfo_first%3Dc42dbaf3e80c0235b09d328d401a7bc5%23rd&s=obsidian)