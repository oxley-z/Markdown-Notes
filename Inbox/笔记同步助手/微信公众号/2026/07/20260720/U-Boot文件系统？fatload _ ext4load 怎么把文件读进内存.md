---
author: Surest
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwOTUxMzQzOA==&mid=2247484395&idx=1&sn=1f9af13f63e8b5e13465a338b58f31b4&chksm=c0c6c9a68625ff6aaf08e30cef50d9730bf06c2621cdfc028e03ebe49c149dded2792e5ac5b4&mpshare=1&scene=1&srcid=07201FqcWeO0LRxs0lGCiAUN&sharer_shareinfo=28db4652f321f78cff8e94c1b830ba25&sharer_shareinfo_first=28db4652f321f78cff8e94c1b830ba25#rd
saved: 2026-07-20 20:50:22
tags:
  - 笔记同步助手
id: 79911e36-6174-4790-bf77-c657179aa4cc
---

公众号名称：摸鱼的日记本

作者名称：Surest

发布时间：2026-07-20 19:30

## 全景介绍

前六篇拆了 U-Boot 的 ARM64 汇编入口、Driver Model、SPL 阶段、环境变量、bootm/booti/bootelf 命令族，以及 flash 子系统。到这一步，镜像已经能从 SPI-NOR / eMMC / NAND 读进 DRAM 了。但有个问题一直没答：为什么 bootcmd 里几乎都是 `fatload mmc 0:1 0x10000000 /boot/Image` 这种带**文件名**的命令，而不是裸的 `mmc read 0x10000000 0x800 0x8000`？

答案是**文件系统子系统**。`fatload` / `ext4load` / `load` / `ls` / `save` 这套命令让 U-Boot 能用「读某个文件」的语义去访问存储，而不用手工算扇区偏移。它在 flash 子系统那层「读某扇区」的抽象之上，又搭了一层「读某文件」的抽象。

这一篇拆 U-Boot 的文件系统子系统：`fs/fs.c` 分派层怎么把 `fatload` / `ext4load` 统一起来、FAT 和 ext4 两个后端怎么解析磁盘结构、写路径为什么比读路径危险得多，以及它跟前一篇 flash 子系统的 `blk_desc` 怎么挂钩。代码基于 **U-Boot v2024.07**。数据流部分以 SD 卡 FAT 分区加载 kernel 作为落地例子。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/4a0d08124f8c10fa748403dc9a1ee103_MD5.jpg]]

Image

上图（图 0 全景运作图）把文件系统子系统在一次 `fatload` 里的完整位置画清楚：用户敲 `fatload mmc 0:1 addr /boot/Image` 后，命令层落到 `fs/fs.c` 的统一入口 `do_load`，分派层按 `fs_type` 选中 FAT 后端，`fat_set_blk_dev` 先读引导扇区认出文件系统，`fat_read_file` 解析目录项和簇链，每一次实际的扇区读都往下走 `disk_read` → `blk_dread` → `mmc_bread`，最后落到前一篇讲的 flash 子系统。整篇文章就是把这条链一节一节拆开。

---

## 实际情况

### U-Boot 支持哪些文件系统

U-Boot 不是要做一个完整的操作系统，它只需要「把 kernel、dtb、initramfs 这几个文件从某个分区读出来」。所以它支持的文件系统是一个精选子集，且大部分只做只读：

| 文件系统 | Kconfig 宏 | 读 / 写 | 典型场景 |
| --- | --- | --- | --- |
| FAT12/16/32 | `CONFIG_FS_FAT` | 读 + 写 | SD 卡 boot 分区，最常用 |
| ext2/3/4 | `CONFIG_FS_EXT4` | 读 + 写（写默认关） | eMMC rootfs 分区 |
| SquashFS | `CONFIG_FS_SQUASHFS` | 只读 | 压缩只读 rootfs |
| UBIFS | `CONFIG_CMD_UBIFS` | 只读 | NAND 上的 UBI 卷 |
| Btrfs | `CONFIG_FS_BTRFS` | 只读 | 少数发行版 rootfs |
| exFAT | `CONFIG_FS_EXFAT` | 只读 | 大容量 SD 卡 |
| erofs | `CONFIG_FS_EROFS` | 只读 | 压缩只读镜像 |

规律很清楚：**FAT 是唯一读写都稳定的文件系统**，ext4 写路径能用但默认关闭，其余全是只读。原因在后面数据流章节会讲透：写文件系统要维护元数据一致性，这在 bootloader 里既危险又用得少。

### 三组命令：老命令、新命令、统一命令

U-Boot 里访问文件系统的命令分三代，都还在源码里共存：

| 代际 | 命令 | 特点 |
| --- | --- | --- |
| 按 FS 命名（老） | `fatload`/ `fatls` / `fatwrite`、`ext4load` / `ext4ls` / `ext4write` | 名字里绑死 FS 类型，源码在 `cmd/fat.c` / `cmd/ext4.c` |
| 统一命令（新） | `load`/ `ls` / `save` / `size` | 不带 FS 名，自动探测后端，源码在 `cmd/fs.c` |
| 特定协议 | `ubifsload`/ `sqfsload` | 单独实现，不走通用分派 |

`fatload` 和 `load` 最终都走 `fs/fs.c` 的同一套分派逻辑。区别只在：`fatload` 硬编码了 `fs_set_blk_dev` 时传 `FS_TYPE_FAT`，`load` 传 `FS_TYPE_ANY` 让分派层自己试。所以下面拆源码，拆的就是这套分派层加两个后端。

### 源码目录结构

```
fs/
├── fs.c              // 分派层：fstype_info[] + fs_read/fs_write/fs_ls
├── fat/
│   ├── fat.c         // FAT 只读：解析 BPB、目录项、簇链
│   └── fat_write.c   // FAT 写：分配簇、更新 FAT 表和目录项
├── ext4/
│   ├── ext4fs.c      // ext4 只读入口
│   ├── ext4_common.c // superblock / inode / extent 解析
│   └── ext4_write.c  // ext4 写：分配 block、更新 bitmap 和 journal
├── squashfs/
├── ubifs/
└── btrfs/

cmd/
├── fs.c              // load / ls / save / size 统一命令
├── fat.c             // fatload / fatls / fatwrite
└── ext4.c            // ext4load / ext4ls / ext4write
```

分派层 `fs/fs.c` 只有几百行，主要的实现量在 `fat/` 和 `ext4/` 两个后端目录。

---

## 抽象对象

### `struct fstype_info` — 文件系统的统一门面

跟前一篇 flash 子系统里 `mtd_info` 用三个函数指针统一介质的思路完全一样，文件系统层用 `fstype_info` 统一各种 FS：

```
// fs/fs.c（简化）
struct fstype_info {
    int fstype;                    // FS_TYPE_FAT / FS_TYPE_EXT / ...
    char *name;                    // "fat" / "ext4" / "squashfs"
    bool null_dev_desc_ok;         // 允许无块设备（如 sandbox）

    int  (*probe)(struct blk_desc *fs_dev_desc,
                  struct disk_partition *fs_partition);
    int  (*ls)(const char *dirname);
    int  (*exists)(const char *filename);
    int  (*size)(const char *filename, loff_t *size);
    int  (*read)(const char *filename, void *buf, loff_t offset,
                 loff_t len, loff_t *actread);
    int  (*write)(const char *filename, void *buf, loff_t offset,
                  loff_t len, loff_t *actwrite);
    void (*close)(void);
    int  (*uuid)(char *uuid_str);
    int  (*opendir)(const char *filename, struct fs_dir_stream **dirsp);
    int  (*readdir)(struct fs_dir_stream *dirs, struct fs_dirent **dentp);
    void (*closedir)(struct fs_dir_stream *dirs);
    int  (*ln)(const char *filename, const char *target);
};
```

`probe` 负责认出「这个分区是不是我这种 FS」，`read` / `write` / `ls` 是核心操作。每种 FS 只要填好这张表，就能接入统一的 `load` / `ls` / `save` 命令。

这里用函数指针表而不是 `switch (fstype)` 一路分下去，是有工程理由的。FAT 和 ext4 的读文件逻辑完全不同，但对上层 `do_load` 来说只关心「给我一个文件名，把数据填进这个 buffer」。函数指针表把「选哪个后端」和「后端怎么干」这两件事解耦：`do_load` 只认 `fstype_info` 这张表，加一种新文件系统就是往 `fstypes[]` 里加一条，命令层一行都不用改。SquashFS、Btrfs 这些后来加进来的后端，就是这么挂上去的。

这套抽象的代价是调试时多一层跳转。你在 `do_load` 里下断点，看到的是 `info->read(...)`，得先知道当前 `fs_type` 是什么，才能定位到底进了 `fat_read_file` 还是 `ext4_read_file`。`fs_type` 是 `fs/fs.c` 里的一个静态全局变量，`fs_set_blk_dev` 认盘成功时写进去，读文件时再读出来选后端。调试文件系统问题，第一件事就是确认这个 `fs_type` 认对了没有——认错盘的话，后面全是错的。

### `fstypes[]` — 后端注册数组

所有 FS 后端登记在一个静态数组里：

```
// fs/fs.c
static struct fstype_info fstypes[] = {
#ifdef CONFIG_FS_FAT
    {
        .fstype    = FS_TYPE_FAT,
        .name      = "fat",
        .probe     = fat_set_blk_dev,
        .close     = fat_close,
        .ls        = fs_ls_generic,
        .read      = fat_read_file,
        .write     = file_fat_write,
        .exists    = fat_exists,
        .size      = fat_size,
        .opendir   = fat_opendir,
        .readdir   = fat_readdir,
        .closedir  = fat_closedir,
    },
#endif
#ifdef CONFIG_FS_EXT4
    {
        .fstype    = FS_TYPE_EXT,
        .name      = "ext4",
        .probe     = ext4fs_probe,
        .close     = ext4fs_close,
        .ls        = ext4fs_ls,
        .read      = ext4_read_file,
        .write     = ext4_write_file,
        .uuid      = ext4fs_uuid,
        ...
    },
#endif
    ...
    {
        .fstype    = FS_TYPE_ANY,      // 兜底：探测失败时的哨兵项
        .name      = "unsupported",
        ...
    },
};
```

数组末尾那条 `FS_TYPE_ANY` 是哨兵。当 `load` 命令不指定 FS 类型时，分派层从头遍历数组、逐个调 `probe`，第一个认领成功的就是当前分区的 FS。

### 三层数据结构的传递

一次 `fatload` 要在三个数据结构之间传递上下文：

| 数据结构 | 来自哪一层 | 作用 |
| --- | --- | --- |
| `struct fstype_info` | 文件系统层 | 选中哪个后端、调哪组回调 |
| `struct disk_partition` | 分区层（`disk/part.c`） | 分区起始 LBA、大小、类型 |
| `struct blk_desc` | 块设备层（前一篇） | `block_read`落到具体主控 |

`disk_partition` 是本文相对前一篇新增的一层：flash 子系统只讲到「读设备的某个扇区」，但一块 SD 卡上有 MBR / GPT 分区表，文件系统只在某个分区内部。`fs_set_blk_dev("mmc", "0:1")` 里的 `0:1` 就是「设备 0 的分区 1」，分区层负责把分区内偏移换算成设备全局 LBA。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/bfc22fced87b8fba5cad55284e5607cf_MD5.jpg]]

Image

上图（图 1 三层门面结构）把文件系统层的抽象骨架画出来：顶上是三代命令（fatload / ext4load / load），中间 `fs/fs.c` 的 `fstypes[]` 数组把它们分派到 FAT / ext4 / SquashFS 各后端，每个后端通过 `probe` 认领分区、通过 `read` / `write` 干活；再往下经过 `disk_partition` 分区换算，最后落到 `blk_desc` 的块设备接口。这张图跟前一篇 flash 子系统的骨架图接在一起，就是 U-Boot 存储栈的完整两层。

---

## 模型

### FAT 的磁盘布局模型

要理解 `fat_read_file` 怎么找到文件，先得有 FAT 磁盘布局的心智模型。FAT32 分区从头到尾分成四段：

```
┌─────────────────┐  分区起始 LBA
│  Boot Sector    │  BPB：每扇区字节数、每簇扇区数、FAT 数量、根目录簇号
│  (BPB)          │
├─────────────────┤
│  FAT #1         │  文件分配表：簇号 → 下一簇号 的链表数组
│  FAT #2         │  备份副本
├─────────────────┤
│  Data Region    │  文件数据本体，按「簇」为单位分配
│   ├─ 根目录     │  目录项数组：文件名 + 起始簇号 + 文件大小
│   ├─ 子目录     │
│   └─ 文件数据   │
└─────────────────┘
```

三个关键概念：

-   -   **簇（cluster）**：FAT 分配的最小单位，通常 4KB～32KB（`每簇扇区数 × 512`）。一个文件占用若干个簇
-   -   **FAT 表**：一个巨大的数组，下标是簇号，值是「这个文件的下一个簇号」。读文件就是顺着这条簇链跳
-   -   **目录项**：32 字节固定长度，记文件名、起始簇号、文件大小。找文件先在目录项数组里按名字匹配

读一个文件的动作因此被拆成两步：**先查目录项拿到起始簇号和大小，再顺着 FAT 表的簇链把数据块一段段读出来**。

### ext4 的磁盘布局模型

ext4 比 FAT 复杂一个量级，因为它是为运行中的 Linux 设计的完整文件系统：

```
┌──────────────┐  分区起始
│  Boot Block  │  前 1KB 保留（历史遗留给引导代码）
├──────────────┤
│  Superblock  │  总 block 数、inode 数、block 大小、特性标志位
├──────────────┤
│  Block Group Descriptors │  每个 block group 的元数据位置
├──────────────┤
│  Block Group 0       │
│   ├─ Block Bitmap    │  哪些数据块被占用
│   ├─ Inode Bitmap    │  哪些 inode 被占用
│   ├─ Inode Table    │  inode 数组：权限、大小、数据块指针
│   └─ Data Blocks         │
├──────────────┤
│  Block Group 1 ...       │
└──────────────┘
```

ext4 找文件的路径比 FAT 长：

1.  1\. 读 **superblock** 拿到 block 大小、inode 表位置
    
2.  2\. 从根目录 inode（固定是 inode \#2）出发，读根目录的数据块
    
3.  3\. 在目录数据块里按名字匹配，拿到目标文件的 **inode 号**
    
4.  4\. 读该 inode，拿到文件大小和**数据块索引**
    
5.  5\. 按索引读数据块
    

第 4 步的「数据块索引」是 ext4 和早期 ext2 的关键分歧：

-   -   **ext2/3 用间接块**：inode 里 12 个直接指针 + 1 个一级间接 + 1 个二级 + 1 个三级
-   -   **ext4 用 extent tree**：inode 里存一棵 B+ 树，每个 extent 描述「一段连续的物理块」，大文件效率高很多

U-Boot 两种都支持，但 extent tree 只稳定处理单层（`eh_depth == 0`），这个限制在案例章节会踩到。

两种布局差这么多，根子在设计目标不同。FAT 是给软盘、U 盘这种「插上读几个文件就拔」的场景设计的，结构越简单越好认，一张 FAT 表就是全部索引，代价是没有权限、没有日志、大盘上簇链长了查找慢。ext4 是给一块常年挂载、多进程并发读写的系统盘设计的，superblock 记全局、block group 把盘切成多个自治区域减少锁争用、extent tree 让大文件的块索引不至于膨胀。对 U-Boot 来说，它要的恰恰是 FAT 那种「简单好认」——启动时读几个固定文件，不需要权限、不需要并发、更不需要日志。ext4 那套为运行态设计的复杂度，在 bootloader 里全是负担。这就是为什么量产板几乎都把 kernel/dtb 放在一个小 FAT 分区，rootfs 才用 ext4：各取所长。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/1fb14ac377e80508bff60e1925412b05_MD5.jpg]]

上图（图 2 FAT vs ext4 布局对比）把两种文件系统的磁盘结构并排画出来：左边 FAT 的四段式（BPB / FAT 表 / 目录项 / 数据簇），配一条「查目录项→顺簇链读」的两步查找箭头；右边 ext4 的分组式（superblock / block group / inode 表 / 数据块），配一条「superblock→根 inode→目录→目标 inode→extent→数据块」的五步查找箭头。两相对照就能看出为什么 FAT 简单稳定、ext4 强大但复杂。

---

## 数据流

拆两条最典型的路径：`fatload` 读文件（正常路径）和 `ext4write` 写文件（危险路径）。SquashFS / UBIFS 是 `fatload` 只读路径的变种，不单独展开。

### `fatload` 读数据流

用户命令：

```
=> mmc dev 0
=> fatload mmc 0:1 0x10000000 /boot/Image
```

调用栈：

```
do_fat_fsload                          cmd/fat.c
  do_load                              cmd/fs.c（统一入口）
    fs_set_blk_dev("mmc", "0:1", FS_TYPE_FAT)   fs/fs.c
      blk_get_device_part_str          disk/part.c 解析 "0:1" 拿分区
      fat_set_blk_dev                  fs/fat/fat.c
        disk_read(0, 1, buf)           读分区第 0 扇区（BPB）
          fs_devread → blk_dread
            blk_desc->block_read = mmc_bread   前一篇的 flash 子系统
        解析 BPB：每簇扇区数、FAT 起始、根目录簇号
    fs_read("/boot/Image", addr, 0, 0, &actread)
      fat_read_file
        do_fat_read_at                 解析路径逐级找目录项
          get_dentfromdir              在目录簇里按名字匹配
          get_fatent                   读 FAT 表拿簇链
          get_cluster                  按簇链读数据块
            disk_read → blk_dread → mmc_bread
```

三个关键点：

-   -   **FAT 层不直接碰 mmc**，只调 `disk_read` / `disk_write`。这两个函数是块设备语义的薄壳，把「分区内第 N 扇区」换算成「设备全局 LBA」再交给 `blk_dread`。所以同一份 FAT 代码能跑在 MMC / USB / NVMe / SATA 上，跟介质无关
-   -   **长文件名（LFN）处理**：FAT32 的长文件名拆在多个 32 字节目录项里，`get_dentfromdir` 要拼接这些项。`/boot/Image` 这种短名走 8.3 快路径，中文名或超长名走 LFN 慢路径
-   -   **读放大**：每读一个新簇都要先查一次 FAT 表（可能触发额外的扇区读），文件碎片化严重时读放大明显。`get_fatent` 里有一个单扇区的 FAT 缓存缓解这个问题

这条链最值得琢磨的是分层带来的读放大。表面上 `fatload` 就是「把文件读进来」，实际每读一个新簇，`get_cluster` 都要回头问一次 `get_fatent`「下一簇在哪」，而 `get_fatent` 可能触发一次额外的扇区读去查 FAT 表。文件在磁盘上连续时，簇号也连续，FAT 表落在同一个缓存扇区里，读放大不明显；文件碎片化严重时，簇链在 FAT 表里跳来跳去，每跳一次可能多一次 IO。启动阶段读的都是刚烧进去的 kernel/dtb，基本连续，所以这个放大平时看不出来，但如果你在一个用了很久、反复删改的 FAT 分区上 `fatload` 一个大文件，速度会明显掉下来。这时候不是 FAT 代码慢，是这块盘该整理碎片了。

### `ext4write` 写数据流

写 ext4 比读 FAT 复杂一个量级，因为要维护多份元数据的一致性：

```
=> ext4write mmc 0:2 0x10000000 /boot/newfile 0x100000
```

调用栈：

```
do_ext4_write                          cmd/ext4.c
  ext4_register_devices
  ext4fs_open                          打开/创建目标文件
  ext4fs_write                         fs/ext4/ext4_write.c
    ext4fs_read_superblock             读 superblock
    ext4fs_get_parent_inode            解析父目录 inode
    ext4fs_get_new_inode               分配一个空闲 inode
      → 扫 inode bitmap 找空位
    ext4fs_allocate_blocks             从 block bitmap 分配数据块
      → 扫 block bitmap
      → 更新 block group descriptor 的 free count
    ext4fs_write_file                  写数据块 → blk_dwrite → mmc_bwrite
    ext4fs_update_parent_dentry        在父目录加一条目录项
    ext4fs_update_journal              追加 journal 记录（若开了 journal）
    ext4fs_update_super_block          回写 superblock 的 free 计数
    ext4fs_update_bmap                 回写 block/inode bitmap
```

一次写要触碰 **superblock、block bitmap、inode bitmap、inode table、目录项、journal、数据块**七类结构，任何一处写完就断电，文件系统就可能不一致。这正是为什么 ext4 写在 U-Boot 里默认关闭。

写路径脆在哪，值得展开一下。这七类结构不是一次原子写下去的，而是一步步分别回写。假设写到 `ext4fs_update_parent_dentry` 之后、`ext4fs_update_bmap` 之前断电：目录项里已经有了新文件的名字，指向那个新 inode，但 inode bitmap 还没标记这个 inode 被占用。下次 Linux 挂载时，fsck 会发现「有目录项引用了一个 bitmap 说空闲的 inode」，这就是典型的元数据不一致。Linux 内核靠 journal 把这一串写打包成一个事务来避免这个问题——要么整批生效，要么整批回滚。而 U-Boot 的 ext4 写只是追加 journal 记录，并没有实现完整的事务重放机制，所以它给不了这个原子性保证。这不是 bug，是 bootloader 场景下的取舍：完整实现 journal 事务要拖进一大块内核级代码，而 U-Boot 里写 ext4 的需求本来就少。既然给不了强保证，索性默认就把这个功能关掉，逼你显式打开、自己担风险。

### 两条路径的对比

| 环节 | `fatload`（读） | `ext4write`（写） |
| --- | --- | --- |
| 上层入口 | `fs_read` | `fs_write` |
| 要读的元数据 | BPB + FAT 表 + 目录项 | superblock + bitmap + inode + 目录 |
| 要写的元数据 | 无（只读） | 七类结构全要回写 |
| 一致性风险 | 无 | 中途断电即损坏 |
| 底层接口 | `disk_read`→ `blk_dread` | `blk_dwrite`（读改写混合） |
| 默认开关 | 默认开 | `CONFIG_CMD_EXT4_WRITE`默认关 |

### `load` 统一命令的自动探测

`load` 命令跟 `fatload` 唯一的区别在 `fs_set_blk_dev` 时传 `FS_TYPE_ANY`：

```
// fs/fs.c（简化）
int fs_set_blk_dev(const char *ifname, const char *dev_part_str, int fstype)
{
    // 拿到分区
    blk_get_device_part_str(ifname, dev_part_str,
                            &fs_dev_desc, &fs_partition, 1);

    // 遍历 fstypes[]，逐个 probe
    for (info = fstypes; info < fstypes + ARRAY_SIZE(fstypes); info++) {
        if (fstype != FS_TYPE_ANY && info->fstype != fstype)
            continue;
        if (!info->probe(fs_dev_desc, &fs_partition)) {
            fs_type = info->fstype;      // 认领成功
            return 0;
        }
    }
    return -1;
}
```

`probe` 的实现就是「读关键扇区看魔数」：`fat_set_blk_dev` 读 BPB 看 `0x55AA` 签名和 `FAT` 字样，`ext4fs_probe` 读 superblock 看 `0xEF53` 魔数。第一个认领成功的后端就是当前分区的 FS。所以 `load mmc 0:1 addr file` 用户不用记具体 FS 名，脚本能跨 FAT / ext4 复用同一条命令。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/84698ee65564d264077272789351cbc0_MD5.jpg]]

Image

上图（图 3 fatload 读数据流）把一次 `fatload mmc 0:1 addr /boot/Image` 的完整调用栈画成竖向主线：命令层 do\_fat\_fsload → 分派层 do\_load → fs\_set\_blk\_dev 认盘 → fat\_read\_file 解析。右侧分叉出「查目录项」和「顺簇链读」两个子流程，每个子流程的底部都汇聚到 disk\_read → blk\_dread → mmc\_bread。这张图让读者看清「读文件」这个动作是怎么一层层落到「读扇区」的。

---

## 运行

### 初始化：文件系统层什么时候就绪

跟 env 子系统不同，文件系统层**没有独立的初始化阶段**。它是完全按需触发的，只有当 bootcmd 里实际执行 `fatload` / `load` 时，`fs_set_blk_dev` 才会去 probe。

这个设计的原因是文件系统层依赖块设备层，而块设备层本身是 lazy probe 的（前一篇讲过）：

```
board_init_r()
  → initr_mmc()              初始化 MMC 控制器，注册 blk_desc
  → ...
  → run_main_loop()
      → autoboot_command()
          → run_command("fatload mmc 0:1 ...")
              → 此时才第一次触发文件系统层
              → fs_set_blk_dev → fat_set_blk_dev → probe
```

所以文件系统层的「运行」严格跟随 bootcmd 的执行节奏。一个典型的 SD 卡启动 bootcmd：

```
mmc dev 0
fatload mmc 0:1 0x10000000 /boot/Image
fatload mmc 0:1 0x11000000 /boot/system.dtb
fatload mmc 0:1 0x12000000 /boot/initramfs.cpio.gz
booti 0x10000000 0x12000000 0x11000000
```

三条 `fatload` 各触发一次完整的「认盘 + 解析 + 读」流程，认盘结果不缓存跨命令，每条 `fatload` 都重新读一次 BPB。这在启动阶段无所谓，因为总共就几条命令。

每条 `fatload` 都重新认一次盘，看着有点浪费，但这个设计在 bootloader 里是对的。缓存认盘结果意味着要维护「这块盘还是不是上次那块」的状态——用户中途可能 `mmc dev 1` 切了设备、可能把 SD 卡拔了换一张。U-Boot 是个交互式环境，命令之间没有强约束，缓存状态一旦和实际不符，读出来的就是错盘的数据，这种错误比多读一次 BPB 危险得多。启动阶段总共就几条 `fatload`，重复认盘的开销可以忽略，换来的是每条命令都从干净状态开始，不依赖上一条命令留下的上下文。这是拿一点性能换确定性，符合 bootloader「宁可慢一点也别出错」的基调。

调试文件系统读不出来的问题，排查顺序也跟着这个结构走：先 `mmc part` 确认分区表认得出来（块设备层 OK），再 `ls mmc 0:1 /` 确认文件系统认得出来（`probe` 成功、`fs_type` 对），最后才是 `fatload` 具体文件。三步任何一步断了，问题就锁在那一层，不用一路瞎猜。

### `ls` 和 `size` 命令

除了 load，文件系统层还提供两个查询命令，走同一套分派：

-   -   **ls mmc 0:1 /boot**：`do_ls` → `fs_ls` → `info->ls` → `fs_ls_generic`（FAT/ext4 共用）→ 遍历目录项打印
-   -   **size mmc 0:1 /boot/Image**：`do_size` → `fs_size` → `info->size`，只解析元数据拿文件大小，不读数据，常用于给 `tftpboot` 之前预留内存

`fs_ls_generic` 是个有意思的复用点：它用 `opendir` / `readdir` / `closedir` 三个回调遍历目录，FAT 和 ext4 各自实现这三个，`ls` 逻辑本身完全共享。这跟 Linux VFS 的 `readdir` 抽象是同一个套路，只是精简版。

### 环境变量后端复用文件系统层

上一篇 env 子系统里讲过 `CONFIG_ENV_IS_IN_FAT` / `CONFIG_ENV_IS_IN_EXT4`，这两个 env 后端就是**复用了本篇的文件系统层**：

```
// env/fat.c（简化）
static int env_fat_load(void)
{
    file_fat_read(CONFIG_ENV_FAT_FILE, buf, CONFIG_ENV_SIZE);
    // CONFIG_ENV_FAT_FILE 默认 "uboot.env"
    return env_import(buf, 1, H_EXTERNAL);
}
```

`env_fat_load` 直接调 `file_fat_read` 从 FAT 分区读 `uboot.env` 文件，再交给 `env_import` 解析。所以 env 子系统的 FAT/ext4 后端不用自己碰扇区，站在文件系统层肩膀上。这也解释了为什么前一篇讲 env 时说「借上面的 flash 子系统读一段」，中间隔着的就是本篇的文件系统层。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b90aae898825b5ad7b05046720ac1134_MD5.jpg]]

Image

上图（图 4 按需触发时序）把文件系统层在启动时序里的位置画出来：board\_init\_r 阶段只初始化 MMC 控制器注册 blk\_desc，文件系统层此时完全没动；直到 run\_main\_loop 里 autoboot 执行 bootcmd，第一条 fatload 才触发 fs\_set\_blk\_dev 去 probe。图右侧标出 env\_fat\_load 也是同一个入口的复用者。整张图的核心是「文件系统层没有独立初始化，完全按 bootcmd 节奏被动运行」。

---

## 总结

U-Boot 文件系统子系统的源码组织可以用四句话概括：

1.  1.  **fs/fs.c 是文件系统的分派层**。`fstype_info` 用 `probe` / `read` / `write` / `ls` 一组函数指针统一了 FAT / ext4 / SquashFS，跟前一篇 `mtd_info` 统一介质是同一个套路，只是这次统一的是文件系统。
2.  2.  **fatload 和 load 殊途同归**。老命令 `fatload` 硬编码 `FS_TYPE_FAT`，新命令 `load` 传 `FS_TYPE_ANY` 让分派层遍历后端自动探测。两者最终都走 `fs_read` → 后端 `read` 回调。
3.  3.  **文件系统层站在块设备层肩膀上**。FAT / ext4 后端不直接碰 `mmc`，只调 `disk_read` / `disk_write`，由分区层换算 LBA 再落到前一篇的 `blk_dread` → `mmc_bread`。介质无关性就是这么来的。
4.  4.  **读稳写险**。FAT 读写都稳，ext4 读能用（但 extent 深层树有坑），ext4 写要触碰七类元数据、还有 journal 覆盖风险，默认关闭。

存储栈的两层抽象到这里就补全了：

-   • 前一篇 flash 子系统：把 SPI-NOR / NAND / eMMC / SD 压平成「读某扇区」
    
-   • 本篇文件系统子系统：把扇区转换读某文件
    

`fatload mmc 0:1 addr /boot/Image` 这条命令，就是这两层抽象叠在一起的产物：上面走 `fs_read` 解析文件，下面走 `blk_dread` + `mmc_bread` 读扇区。看懂它的完整路径，就等于把 U-Boot 的整个存储栈走通了一遍。

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/3e446835_1784551820603?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwOTUxMzQzOA%3D%3D%26mid%3D2247484395%26idx%3D1%26sn%3D1f9af13f63e8b5e13465a338b58f31b4%26chksm%3Dc0c6c9a68625ff6aaf08e30cef50d9730bf06c2621cdfc028e03ebe49c149dded2792e5ac5b4%26mpshare%3D1%26scene%3D1%26srcid%3D07201FqcWeO0LRxs0lGCiAUN%26sharer_shareinfo%3D28db4652f321f78cff8e94c1b830ba25%26sharer_shareinfo_first%3D28db4652f321f78cff8e94c1b830ba25%23rd&s=obsidian)