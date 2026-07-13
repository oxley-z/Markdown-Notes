---
author: Surest
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwOTUxMzQzOA==&mid=2247484302&idx=1&sn=044a5e2c010ce97eee7df62ca78065fe&chksm=c0906f4a252ded3192eb9c52a705f524f8dd50ac975ec1403ccbb6cfa172a03b8c265179b6af&mpshare=1&scene=1&srcid=0709qTZI8zegbhvnvJujoBGc&sharer_shareinfo=481637c4bfee8f0acf9c35813e7c7991&sharer_shareinfo_first=481637c4bfee8f0acf9c35813e7c7991#rd
saved: 2026-07-09 10:48:52
tags:
  - 笔记同步助手
id: 7ae457c3-b93e-480b-a052-33c64e6401b4
---

公众号名称：摸鱼的日记本

作者名称：Surest

发布时间：2026-07-08 20:01

  

## 全景介绍

前三篇拆了 U-Boot 的 ARM64 汇编入口、Driver Model 和 SPL 阶段。但 U-Boot 启动到最后一步——`autoboot`——为什么有的板子直接进内核、有的板子停在命令行等操作？这个行为是谁决定的？

答案在**环境变量（env）**。`bootcmd` 环境变量定义了 U-Boot 启动后自动执行的命令，而 env 存在哪里、怎么加载、CRC 怎么校验、修改后又怎么写回——背后是一整套**优先级驱动的存储后端框架**。

env 子系统是 U-Boot 里非常"完整"的一个内部服务：hash table、CRC 校验、后端抽象、默认值兜底、写保护、import/export 格式，一样不缺。把它拆明白，你对嵌入式启动的控制力会上一个台阶。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f21d3903767395db4677f8477690d1d2_MD5.jpg]]

  

上图（图 0 全景运作图）把 env 子系统在 U-Boot 生命周期中的位置一张图讲清楚：从 `board_init_r` 收尾开始，`env_init` 注册后端 driver，`env_load` 从存储把变量读进内存，`env_ready` 标志置位，`autoboot` 倒计时允许打断，最后 `bootcmd` 执行完成跳内核。中间任何阶段用户 `saveenv` 都能反向持久化——这就是本文接下来要拆解的全部内容。

---

## 实际情况

### 环境变量的物理存储

环境变量不是编译进代码里的，而是持久化在非易失存储上。U-Boot 支持的后端多达十几种：

| 存储后端 | Kconfig 宏 | 典型场景 |
| --- | --- | --- |
| MMC / eMMC | `CONFIG_ENV_IS_IN_MMC` | 最常见的嵌入式方案 |
| SPI Flash | `CONFIG_ENV_IS_IN_SPI_FLASH` | NOR Flash 方案 |
| NAND Flash | `CONFIG_ENV_IS_IN_NAND` | 大容量 NAND 方案 |
| FAT 文件系统 | `CONFIG_ENV_IS_IN_FAT` | SD 卡启动的开发模式 |
| ext4 文件系统 | `CONFIG_ENV_IS_IN_EXT4` | eMMC ext4 分区 |
| UBI | `CONFIG_ENV_IS_IN_UBI` | MTD 之上的 UBI 卷 |
| EEPROM | `CONFIG_ENV_IS_IN_EEPROM` | 板载小容量 EEPROM |
| MTD | `CONFIG_ENV_IS_IN_MTD` | 裸 MTD 分区 |
| 远程 | `CONFIG_ENV_IS_IN_REMOTE` | 网络加载（很少见） |
| NVRAM | `CONFIG_ENV_IS_IN_NVRAM` | 带电池备份的 SRAM |
| Nowhere | `CONFIG_ENV_IS_NOWHERE` | 无持久存储，只用默认值 |

一个平台可以同时启用多个后端，U-Boot 按优先级依次尝试。

### 环境变量的内存组织：hash table 深剖

env 在内存中的唯一载体是一张 hash table：

```
1234
struct hsearch_data env_htab = {
    .change_ok = env_flags_validate,
};
```

`hsearch_data` 借鉴自 GNU libc 的 `hcreate_r()`/`hsearch_r()` API，U-Boot 把它裁剪到 `lib/hashtable.c` 里独立维护。核心字段：

```
12345678
struct hsearch_data {
    struct env_entry *table;    // bucket 数组
    unsigned int size;          // bucket 总数（2 的幂）
    unsigned int filled;        // 已用槽位数
    int (*change_ok)(const env_entry_t *__item,
                     const char *newval, enum env_op, int flags);
};
```

每一个槽位是一条 `env_entry`：

```
1234567891011
typedef struct env_entry {
    int used;                   // 该槽是否已占用
    struct env_item {
        const char *key;
        char *data;
        int (*callback)(const char *name, const char *value,
                        enum env_op op, int flags);
        int flags;              // 属性位（write-once 等）
    } entry;
} env_entry_t;
```

**为什么选 hash 而不是数组或链表**：环境变量是随机访问、名字查找为主的场景（`env_get("bootargs")`、`env_get("ipaddr")`），链表 O$N$ 太慢，数组要维护有序又浪费空间。hash 表在 U-Boot 上的实测查找是 O$1$ 均摊，几百个变量都能瞬间命中。

### hash 函数与冲突处理

U-Boot 用的是简单加权 hash（`lib/hashtable.c: hash_string()`）：

```
12345678910111213141516
static unsigned int hash_string(const char *key)
{
    unsigned int hval, g;
    hval = strlen(key);
    while (*key != '\0') {
        hval <<= 4;
        hval += *key++;
        g = hval & 0xf0000000;
        if (g) {
            hval ^= g >> 24;
            hval ^= g;
        }
    }
    return hval;
}
```

冲突处理走**开放定址 + 二次探测**：初始 idx = hash % size，冲突时 idx = (idx + hval2) % size 依次探测，直到找到空位或已存在的同名 key。开放定址的好处是无内存分配、无指针跳转，对 U-Boot 早期还没完全建立堆的阶段特别友好。

### hash table 的容量管理

| 时机 | 行为 |
| --- | --- |
| `hcreate_r(nel, htab)` | 启动阶段一次性分配 nel 个 bucket。U-Boot 默认 `CONFIG_ENV_MIN_ENTRIES=64`，加载真实环境时按导入条数 ×3 重新分配 |
| `hsearch_r(item, ENTER)` | 插入/更新。查槽 → 若已存在则触发 `change_ok` 校验并复写；若为空则占用 |
| 装载因子 > 80% | `hmatch_r`探测代价升高，需要 `hdelete_r` 释放槽位或增大 size |

### import / export：hash table ↔ 字符串

hash table 只是运行时视图，真正持久化的是"扁平字符串"：

```
12
key1=value1\0key2=value2\0key3=value3\0\0
```

两个双向转换函数：

-   -   **himport\_r：解析扁平字符串 → 逐条 `hsearch_r(ENTER)` 写入 hash table**
-   **• **hexport\_r
    
    ：遍历 hash table → 输出扁平字符串（可选按 sep 分隔、按 whitelist 过滤）
    
    `env_load()`
    
    走 himport，`env_save()` 走 hexport。​这两条路径就是 hash table 和物理存储之间的桥梁。​  
    
    ### hash table 的额外能力
    
    ### 关键参数
    
    ```
    1234
    CONFIG_ENV_SIZE=0x2000              # 环境区域大小 (8 KB)
    CONFIG_ENV_OFFSET=0xE0000           # 存储介质偏移
    CONFIG_ENV_OFFSET_REDUND=0xE2000    # 冗余副本偏移（可选）
    ```
    
    冗余环境的意义：存储上维护两份 env + 各自独立的 CRC。​加载时选 CRC 正确的一份、保存时交替写两份，确保任何时候断电都不会丢配置。​
    
    ### 源码位置
    
    | 文件 | 内容 |
    | --- | --- |
    | `env/env.c` | 框架核心：load/save/erase 主流程，后端注册与查找 |
    | `env/common.c` | hashtable 操作：`env_set`、`env_get`、import/export |
    | `env/attr.c` | 变量属性（write-once、change-once 等） |
    | `env/callback.c` | 变量修改时的回调机制 |
    | `env/flags.c` | 变量访问标志验证 |
    | `env/mmc.c`、`env/fat.c`、`env/ext4.c` | 各存储后端实现 |
    | `env/embedded.c` | 编译时内嵌的默认环境变量 |
    | `cmd/nvedit.c` | 用户命令：`printenv`、`setenv`、`saveenv`、`editenv` |
    | `include/env.h` | 对外 API |
    | `include/env_internal.h` | 内部数据结构 |
    
    ---
    
    ## 抽象对象
    
    ### env\_driver — 存储后端驱动
    
    ```
    123456789
    struct env_driver {
        const char *name;
        enum env_location location;
        int (*load)(void);      // 从存储读 → hash table
        int (*save)(void);      // hash table → 存储
        int (*erase)(void);     // 擦除存储
        int (*init)(void);      // 初始化存储介质
    };
    ```
    
    各后端通过 `U_BOOT_ENV_LOCATION()` 宏注册到 linker list `.env_driver` 段。​framework 通过 `_env_driver_lookup(loc)` 拿到对应的 driver。​
    
    ### env\_location — 存储位置枚举
    
    ```
    1234567
    enum env_location {
        ENVL_EEPROM, ENVL_EXT4, ENVL_FAT, ENVL_FLASH,
        ENVL_MMC, ENVL_NAND, ENVL_NVRAM, ENVL_REMOTE,
        ENVL_SPI_FLASH, ENVL_UBI, ENVL_MTD, ENVL_NOWHERE,
        ENVL_SCSI, ENVL_COUNT,
    };
    ```
    
    ### env\_entry — hash table 条目
    
    ```
    1234567
    struct env_entry {
        const char *key;
        char *data;
        int (*callback)(const char *name, const char *value,
                        enum env_op op, int flags);
    };
    ```
    
    ### env\_id — 变更计数器
    
    ```
    123
    static int env_id = 1;
    void env_inc_id(void) { env_id++; }
    ```
    
    每次 `env_set()` 递增。​网络协议栈用它判断环境是否变化——变了就重读 IP 相关变量。​
    
    ### CRC 校验
    
    物理存储布局：
    
    ```
    12345
    +-----------+----------+------+
    |  CRC32    |  flags   | data |
    | (4 bytes) | (1 byte) | ...  |
    +-----------+----------+------+
    ```
    
    `env_load()` → 后端 `load()` → CRC32 校验 → `himport_r()` 导入 hash table。​CRC 不通过 → 使用编译时默认值。​
    
    ![[Inbox/笔记同步助手/微信公众号/2026/07/images/2e950b944e8c171369f3f581e093a646_MD5.jpg]]
    
      
    
    上图（图 1）把 env 子系统的五个角色画在一起：hash table 是内存视图，driver 抽象物理存储，用户命令层负责读写触发，默认值兜底，env\_load / env\_save 是两个具体动作。​理解这张图就能理解后面的所有细节。​
    
    ---
    
    ## 模型
    
    ### 优先级驱动的后端选择
    
    env\_load 优先级策略：
    
    ****

-   -   **回调（callback）**：`ipaddr`、`netmask` 等变量修改时会触发 `net_init()` 重新拉起网络栈，无需重启 U-Boot
-   -   **flags 校验**：`change_ok` 拦截 write-once、change-once 变量的非法修改
-   -   **variable 属性**：`env/flags.c` 通过 `env_flags_get()` 查每个 key 的属性；`serial#` 默认 `w`（write-once）、`ethaddr` 默认 `mc`（MAC change-once）

1.  1\. 从 prio=0（最高优先级）开始尝试
    
2.  2\. CRC 校验通过 → 加载成功，记录 `gd->env_load_prio`
    

-   ****3\. CRC 校验失败 → 记录 best\_prio（有数据但损坏了）****
-   ****4\. 所有后端都失败 → `env_set_default()` 用默认值****
-   ****5\. 下次 `env_save()` 写到 `gd->env_load_prio` 对应的位置****
    
    ****这个设计保证：**环境在哪就读哪、坏了就修、没有就建**——不需要用户手工配置。****
    
    ### ****优先级来源****
    
    ```
    12345
    __weak enum env_location env_get_location(enum env_operation op, int prio)
    {
        return arch_env_get_location(op, prio);
    }
    ```
    
    ****`env_locations[]` 数组按启用的 `CONFIG_ENV_IS_IN_*` 排列。板级代码可以覆盖 `arch_env_get_location()`，实现自定义优先级（比如"生产模式先读 eMMC、开发模式先读 SD FAT"）。****
    
    ### ****默认环境变量的两个来源****
    
    ****两组默认值编译进 `.rodata`，加载失败时直接搬进 hash table。****
    
    ---
    
    ## ****数据流****
    
    ### ****加载流：存储 → CRC → hash table****
    
    ```
    1234567891011121314151617181920
    env_init():
      → 遍历 env_locations[]，调用各后端的 init()
      → 设置 gd->env_has_init 对应位
    
    env_load():
      → 遍历优先级 (0 → N):
          env_driver_lookup(ENVOP_LOAD, prio)
            ↓
          drv->load():
            → 从存储读原始数据 → CRC 校验
            → himport_r() 逐条 key=value → env_htab
            ↓
          CRC 正确 → 返回 0
          CRC 错误 → 返回 -ENOMSG
            ↓
      全部失败 → env_set_default() + gd->env_load_prio = best_prio
    
    env_get_ready():
      gd->flags |= GD_FLG_ENV_READY
    ```
    
    ****`GD_FLG_ENV_READY` 是环境就绪标志——置位后 `env_get()`/`env_set()` 才走 hash table 主路径，之前只能读编译时默认值。****
    
    ****![[Inbox/笔记同步助手/微信公众号/2026/07/images/8cd5f9c6b5312f463d1865f26aaf53cf_MD5.jpg]]
    
      
    
    
    ****
    
    ****图 2 把上面的伪代码画成了状态机：上方主线是成功路径，从 `env_load()` 入口一路走到 `himport_r → env_htab`；下方独立一行是失败与结果分支——`CRC 错` 记录 best\_prio → prio++ 继续尝试或全失败走默认值 → 最终都把值写到 `gd->env_load_prio`。**成功和失败路径最后都汇入 gd->env\_load\_prio，因为它同时是"这次从哪读到"和"下次 saveenv 写到哪"两个含义。******
    
    ### ****保存流：hash table → CRC → 存储****
    
    ```
    123456789101112
    env_save():
      → drv = env_driver_lookup(ENVOP_SAVE, gd->env_load_prio)
      → drv->save():
          → hexport_r() 导出 key=value 字符串
          → 计算 CRC32
          → 写入冗余副本（如有）
          → 写回存储:
             MMC        → mmc_write()
             SPI Flash  → spi_flash_write()
             FAT        → file_fat_write()
             NAND       → nand_write_skip_bad()
    ```
    
    ****![[Inbox/笔记同步助手/微信公众号/2026/07/images/daf254469e774484d3f544661b28cf40_MD5.jpg]]
    
      
    
    
    ****
    
    ****图 3 是 save 的镜像流程。注意几点：****
    
    ### ****bootcmd 执行流****
    
    ```
    12345678910111213
    board_init_r() 最后:
      → autoboot()
          → bootdelay 倒计时
          → 用户按键? → 是: 命令行 / 否: continue
      → run_command_list("run bootcmd")
          ↓
      bootcmd 典型值:
          "ext4load mmc 0:1 ${kernel_addr_r} /boot/Image;
           ext4load mmc 0:1 ${fdt_addr_r} /boot/board.dtb;
           booti ${kernel_addr_r} - ${fdt_addr_r}"
          ↓
      逐命令执行 → 加载内核 → booti 跳转
    ```
    
    ---
    
    ## ****运行****
    
    ### ****初始化时序****
    
    ****在 `board_init_r()` 的 init sequence 中，环境分两步初始化：****
    
    ****第一步: initr\_env — 只做后端的 init（探测存储设备），不读数据第二步: initr\_env\_reloc — env\_relocate → env\_load 真正加载****
    
    ****之所以分两步，是因为 initr\_env时 DM 还没完全就绪，只能做 driver probe；等到 initr\_env\_reloc，MMC/NAND/SPI 存储驱动都能正常工作了，才走 env\_load 主路径。****
    
      
    
    ****![[Inbox/笔记同步助手/微信公众号/2026/07/images/ef02aece6e02e8db4bd4b84a355d7e10_MD5.jpg]]
    
      
    
    
    ****
    
    ****图 4 展示了 env 加载完成之后的最后一段：autoboot 读 bootdelay 倒计时，允许键盘打断，否则执行 bootcmd 展开 的命令链，最后 `booti`****
    
     ****把控制权移交给 Linux 内核。**bootcmd 就是 U-Boot 交付给操作系 统的最后一个动作。**  
    
    ### 常用命令
    
    ```
    123456789
    printenv                    # 查看所有变量
    printenv bootcmd            # 查看单个
    setenv bootdelay 3          # 设置（仅内存）
    setenv bootcmd 'mmc read ${loadaddr} 0x800 0x4000; bootm ${loadaddr}'
    saveenv                     # 持久化
    setenv bootdelay            # 删除变量（值为空即删除）
    env export -c -s 0x2000 0x40000000   # 导出到内存
    env import 0x40000000                # 从内存导入
    ```
    
    ### setexpr — 环境变量的表达式求值
    
    `setenv` 只能把一个字符串塞到 hash table 里，做算术、位运算、字符串拼接就得靠 `setexpr`。它把命令行参数当作**表达式**求值，把结果写回 env。
    
    ```
    123456
    setexpr  [*]  [*]   # 二元运算
    setexpr  [*]                      # 直接赋值
    setexpr  gsub   [str]        # 全局替换
    setexpr  sub    [str]        # 首次替换
    setexpr  fmt   [args...]            # 格式化字符串
    ```
    
    **几个常用玩法**：
    
    ```
    123456789101112131415
    # 1) 简单算术：地址偏移
    setexpr fdt_addr_r $kernel_addr_r + 0x02000000
    # 结果 fdt_addr_r = kernel_addr_r 之上 32MB
    
    # 2) 位运算：从寄存器取字段
    setexpr chipid *0xff100000                    # 取地址 0xff100000 处的一个 word
    setexpr rev $chipid \& 0xff                    # 低 8 位是芯片版本
    
    # 3) 字符串替换：动态构造启动参数
    setenv template "root=PARTUUID=%s ro"
    setexpr rootargs sub %s $uuid $template
    
    # 4) 生成 IP 尾段偏移
    setexpr myip $baseip + $node_id
    ```
    
    ## 取值语义：
    
    -   -   `$name` — 从 env 读变量值（字符串）
    -   -   `*$name` 或 `*0xAddr` — 把值当作内存地址，读该地址的一个 word（32/64 位视平台而定）
    
    setexpr 让 U-Boot 脚本从"死字符串拼接"进化到"运行时可计算"，特别适合：
    
    -   • 板级差异化启动参数（同一份 bootcmd 应对多种硬件版本）
        
    -   • 从 OTP / 芯片 ID 读厂商信息拼接到 kernel cmdline
        
    -   • 动态计算加载地址、分区偏移
        
    
    配置开关：`CONFIG_CMD_SETEXPR=y`，默认在多数 defconfig 中启用。源码 `cmd/setexpr.c`。
    
    ## 注意事项：
    
    -   • 算术溢出不报错，静默截断到 32/64 位
        
    -   -   `gsub`/`sub` 用的是简化正则（`env/regex.c`），不完全兼容 POSIX ERE
    -   • 二元操作数支持 `+`、`-`、`*`、`/`、`%`、`&`、`|`、`^`、`<<`、`>>`；shell 里 `&` 和 `|` 要转义
        
    
    ### 变量访问控制
    
    -   -   **write-once**：只能设置一次（如序列号 `serial#`）
    -   -   **change-once**：出厂后只能改一次（如 MAC `ethaddr`）
    
    配置在 `CONFIG_ENV_FLAGS_LIST_STATIC` 中：`"serial#:wo,ethaddr:oc"`。这套机制通常配合安全启动使用，避免关键身份信息被随意改写。****

-     
    ****
    
    ---
    
    ## 案例
    
    ### QEMU ARM64 virt 环境变量流
    
    QEMU virt 默认 `CONFIG_ENV_IS_IN_FLASH`，存在模拟的 CFI Flash 里。启动日志：
    
    ```
    123
    Loading Environment from Flash... OK
    Hit any key to stop autoboot:  2
    ```
    
    完整链路：
    
    ### 从 MMC FAT 分区加载
    
    开发阶段常用 SD 卡/FAT 存 env：
    
    ```
    12345
    CONFIG_ENV_IS_IN_FAT=y
    CONFIG_ENV_FAT_INTERFACE="mmc"
    CONFIG_ENV_FAT_DEVICE_AND_PART="0:1"
    CONFIG_ENV_FAT_FILE="uboot.env"
    ```
    
    `env/fat.c` 的实现：读 `uboot.env` 文件 → CRC → hash table。直接替换 SD 卡上的文件就能改配置，不用重烧 U-Boot——这是嵌入式开发阶段最好用的一种玩法。
    
    ### 常见问题定位
    
    saveenv 后环境丢失 → 检查 `gd->env_load_prio`、`CONFIG_ENV_OFFSET` 是否和分区表冲突
    
    "bad CRC, using default environment" → 存储上的 env CRC 校验失败，已回退到默认值，saveenv 一次即可修复
    
    bootcmd 不执行 → `printenv bootcmd` 查是否存在，`bootdelay` 是否为 -1（-1 表示禁用 autoboot）
    
    自定义变量重启丢失 → `setenv` 只写内存，必须 `saveenv` 才持久化
    
    sn/mac 改不动 → 查 `CONFIG_ENV_FLAGS_LIST_STATIC`，可能被标为 write-once / change-once
    
    ---
    
    ## 总结
    
    1.  1\. 存储后端抽象 — hash table 是统一"内存视图"，物理存储由 driver 抽象。换存储介质只在 Kconfig 改两个宏。
        
      
    3.  2\. 优先级策略 — 默认值 → 存储上的持久化值 → 运行时 `setenv`。加载时"优先用 CRC 正确的，坏了退默认值"。
        
      
    5.  3\. bootcmd 是唯一交接点 — 它的值定义了完整的启动命令链。理解 `bootcmd` 就理解了 U-Boot 是怎么把控制权交给内核的。
        
    
    **调试口诀**：`printenv` 看现状 → `setenv` 改配置 → `saveenv` 持久化 → 重启验证。一路顺畅就说明存储后端和优先级配置都对了。
    
      
    ****

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/edfafee6_1783565327768?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwOTUxMzQzOA%3D%3D%26mid%3D2247484302%26idx%3D1%26sn%3D044a5e2c010ce97eee7df62ca78065fe%26chksm%3Dc0906f4a252ded3192eb9c52a705f524f8dd50ac975ec1403ccbb6cfa172a03b8c265179b6af%26mpshare%3D1%26scene%3D1%26srcid%3D0709qTZI8zegbhvnvJujoBGc%26sharer_shareinfo%3D481637c4bfee8f0acf9c35813e7c7991%26sharer_shareinfo_first%3D481637c4bfee8f0acf9c35813e7c7991%23rd&s=obsidian)