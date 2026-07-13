---
author: Surest
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwOTUxMzQzOA==&mid=2247484162&idx=1&sn=207576d497a4b27e4307b6fa17158a1d&chksm=c0a83ea0af6604b8bf1595623dadd88d05668d206f9610e6e63c95b7ef8d28735c9624f51374&mpshare=1&scene=1&srcid=07096eDMLVVE0OVYxHDER0uK&sharer_shareinfo=e0e23647358879ec3d98cb8ce420184b&sharer_shareinfo_first=e0e23647358879ec3d98cb8ce420184b#rd
saved: 2026-07-09 11:09:53
tags:
  - 笔记同步助手
id: 4f10b22e-03b0-495a-bc51-560f22f70655
---

公众号名称：摸鱼的日记本

作者名称：Surest

发布时间：2026-07-01 19:25

## 全景介绍

前两篇把 QEMU ARM64 的启动链路走到了 `board_init_r`：汇编入口、异常等级、relocation、C 入口都已经能对上源码。继续往下看时，真正开始影响外设访问的不是某个单独的串口函数，而是一套更底层的驱动骨架：**U-Boot Driver Model，简称 DM**。

在 `board_init_r` 的 initcall 表里，会看到这样一段调用：

```
12
initr_of_live → initr_dm → board_init → initr_dm_devices → serial_initialize → dm_announce
```

这几步决定了后面的串口、timer、GPIO、I2C、MMC、网络设备怎样从设备树节点变成可调用的运行时对象。U-Boot 不是 Linux，但这里已经能看到一个很清晰的设备模型：

-   • 设备树描述硬件位置和匹配字符串；
    
-   • driver 描述一类硬件的初始化逻辑；
    
-   • uclass 把同类设备收在一起；
    
-   • udevice 代表某一个设备在当前启动阶段的运行时状态；
    
-   • probe 把这些静态信息真正落实到硬件初始化。
    

读 DM 时不要先陷入某个驱动文件，先建立这张骨架图：

-   -   **DT 节点先 bind 成 udevice**
-   -   **probe 时调用 driver 回调**
-   -   **上层再通过 uclass 和 ops 使用设备**

这条线理顺以后，`serial_puts()` 为什么能输出、MMC 为什么能被枚举、网络设备为什么要等总线和 PHY 初始化，都能按同一个模型解释。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/1eae99d0cb7ab9af6042b769fc599579_MD5.jpg]]

## 实际情况：board\_init\_r 里 DM 到底做了什么

`board_init_r` 阶段已经完成 relocation，U-Boot 镜像搬到了 DDR 里的最终运行地址。这个阶段最容易忽略的一点是：**pre-reloc 阶段建立过的 DM 对象不能简单继续沿用**。旧对象里的很多指针指向 relocation 前的地址，继续使用会带来不可控的问题。

所以 `initr_dm` 做的不是“继续初始化一点设备”，而是把 DM 根重新搭起来：

```
123456789
static int initr_dm(void)
{
    gd->dm_root_f = gd->dm_root;    /* 保存 pre-reloc 阶段的 root，主要用于调试 */
    gd->dm_root = NULL;             /* 清空 post-reloc root */
 
    dm_init_and_scan(false);        /* 重新初始化 DM，并扫描设备树 */
    return 0;
}
```

这一步之后，DM 的主流程大致展开成：

```
1234567891011121314
dm_init()
  ├─ 创建 root driver
  ├─ device_bind_by_name() 创建虚拟根设备
  └─ device_probe(dm_root) 激活 root
 
dm_scan()
  ├─ dm_scan_plat()        扫描 platform data
  ├─ dm_extended_scan()    扫描 FDT 顶层节点
  │   ├─ lists_bind_fdt(root, chosen)
  │   ├─ lists_bind_fdt(root, aliases)
  │   └─ lists_bind_fdt(root, /)
  ├─ dm_scan_other()       平台特定扫描
  └─ dm_probe_devices()    probe bootph-all / PRE_RELOC 设备
```

`dm_extended_scan()` 会从 FDT 根节点往下递归，遇到可用节点后读取 `compatible`，再去 U-Boot 的 driver linker list 里找匹配项。找到以后调用 `device_bind_with_driver_data()`，创建对应的 `udevice`。

这里要区分两个动作：

-   -   **bind**：创建运行时对象，把设备放进 DM 树和 uclass 链表；
-   -   **probe**：分配/填充 plat、priv，打开电源域、配置 pinctrl、调用 driver 的 probe，真正初始化硬件。

也就是说，`dm_scan()` 之后很多设备只是“被登记了”，并不一定已经可用。比如串口通常会在后面的 `serial_initialize()` 中通过 `uclass_probe_all(UCLASS_SERIAL)` 被统一 probe。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/503aba7db08eeeea27744b2f1d99d791_MD5.jpg]]

## 抽象对象：driver、uclass、udevice 三件事

DM 的源码初看容易乱，是因为几个结构体名字都像“驱动”。实际读代码时可以按职责拆开。

### struct driver：一类硬件怎么初始化

`struct driver` 描述的是某一类具体硬件的驱动实现。以 ns16550 串口为例：

```
123456789101112
U_BOOT_DRIVER(ns16550_serial) = {
    .name       = "ns16550_serial",
    .id         = UCLASS_SERIAL,
    .of_match   = ns16550_serial_ids,
    .of_to_plat = ns16550_serial_of_to_plat,
    .probe      = ns16550_serial_probe,
    .remove     = ns16550_serial_remove,
    .ops        = &ns16550_serial_ops,
    .priv_auto  = sizeof(struct ns16550),
    .plat_auto  = sizeof(struct ns16550_plat),
};
```

几个字段非常关键：

-   -   `id` 决定这个设备属于哪个 uclass，例如 `UCLASS_SERIAL`；
-   -   `of_match` 用来和设备树的 `compatible` 匹配；
-   -   `of_to_plat` 把 DT 属性解析到 `plat`；
-   -   `probe` 是真正初始化硬件的入口；
-   -   `ops` 是上层访问设备时使用的操作函数表；
-   -   `priv_auto`、`plat_auto` 告诉 DM 自动分配多大的私有数据和平台数据。

`U_BOOT_DRIVER()` 宏不是普通注册函数，它会把 driver 对象放进特定 linker list 段里。扫描设备树时，DM 遍历这个 driver 列表做匹配。

### struct uclass / uclass\_driver：同类设备怎么管理

uclass 是设备分类。所有串口设备进入 `UCLASS_SERIAL`，所有 GPIO 控制器进入 `UCLASS_GPIO`，所有块设备又进入自己的分类。

简化后可以这样理解：

```
12345678910111213141516
struct uclass {
    void *priv_;
    struct uclass_driver *uc_drv;
    struct list_head dev_head;      /* 当前 uclass 下所有设备 */
};
 
struct uclass_driver {
    const char *name;
    enum uclass_id id;
    int (*post_bind)(struct udevice *dev);
    int (*pre_probe)(struct udevice *dev);
    int (*post_probe)(struct udevice *dev);
    int per_device_auto;
    uint32_t flags;
};
```

以串口 uclass 为例，`UCLASS_DRIVER(serial)` 会设置 `DM_UC_FLAG_SEQ_ALIAS`，表示串口编号可以来自设备树 `aliases`，例如：

```
1234
aliases {
    serial0 = &uart0;
};
```

这样 bind 串口设备时，DM 可以给对应 `udevice` 设置 `seq_ = 0`。后面 `serial0`、默认 console 设备、`uclass_first_device()` 的行为都和这个编号有关。

### struct udevice：某个设备当前的运行时状态

`udevice` 是 driver 的运行时实例。设备树里有一个 `serial@9000000`，匹配到 `ns16550_serial` driver 后，就会创建一个对应的 `udevice`。

常见字段可以这样读：

```
123456789101112131415
struct udevice {
    const struct driver *driver;
    const char *name;
    void *plat_;                    /* of_to_plat 填充 */
    void *priv_;                    /* driver 私有状态 */
    struct udevice *parent;
    struct uclass *uclass;
    void *uclass_priv_;
    struct list_head uclass_node;
    struct list_head child_head;
    struct list_head sibling_node;
    int seq_;
    ofnode node_;
};
```

这几个数据区的所有权要分清：

-   -   `plat_`：来自设备树/平台配置，通常由 `of_to_plat` 填充；
-   -   `priv_`：驱动自己的运行时状态，由 driver 使用；
-   -   `uclass_priv_`：uclass 给每个设备附加的分类私有数据；
-   -   `parent_priv_`：父总线为子设备维护的私有状态。

DM 的设计并不复杂，但它把“谁拥有哪块数据”分得比较清楚。读驱动时只要看 `dev_get_plat()`、`dev_get_priv()`、`dev_get_uclass_priv()` 分别拿到什么，就能判断这块数据属于哪一层。

## 模型：从设备树扫描到 udevice 树

DM 的初始化可以分成两个阶段看。

### pre-reloc：只让最小设备先活起来

`board_init_f` 阶段也会用到 DM，但这时环境还很受限，malloc、FDT、DDR 地址都不是最终状态。通常只有标了 `bootph-*` 或 `DM_FLAG_PRE_RELOC` 的设备会在这个阶段被 probe，比如早期串口、timer、少量 clock/reset 相关设备。

这个阶段的目标很克制：让 U-Boot 能继续启动、能打印日志、能完成必要的早期硬件准备。

### post-reloc：重建完整 DM 树

进入 `board_init_r` 后，`initr_dm` 重新创建 root device，然后扫描设备树。扫描过程中，`lists_bind_fdt()` 会做几件事：

1.  1\. 检查节点是否可用，`status = "disabled"` 的节点跳过；
    
2.  2\. 读取 `compatible` 字符串；
    
3.  3\. 遍历 linker list 里的所有 driver；
    
4.  4\. 比较 driver 的 `of_match` 表；
    
5.  5\. 匹配成功后创建 `udevice`；
    
6.  6\. 如果是总线设备，再继续处理子节点。
    

这里有个工程上很重要的边界：**有 compatible 但没有 U-Boot driver 的节点，通常不会让启动直接失败**。很多设备树节点只给内核使用，U-Boot 阶段不一定需要驱动它们。DM 扫描时会尽量只绑定 U-Boot 能识别、当前阶段需要处理的设备。

## 数据流：device\_probe 把静态对象变成可用设备

bind 完成以后，设备仍然只是一个对象。真正让硬件可用的是 `device_probe(dev)`。

简化后的调用链可以写成：

```
12345678910111213141516171819202122232425262728293031
int device_probe(struct udevice *dev)
{
    if (dev_get_flags(dev) & DM_FLAG_ACTIVATED)
        return 0;
 
    device_of_to_plat(dev);             /* DT → plat */
 
    if (dev->parent)
        device_probe(dev->parent);      /* 递归 probe 父设备 */
 
    dev_or_flags(dev, DM_FLAG_ACTIVATED);
 
    dev_power_domain_on(dev);
    pinctrl_select_state(dev, "default");
    dev_iommu_enable(dev);
    device_get_dma_constraints(dev);
 
    uclass_pre_probe_device(dev);
 
    if (dev->parent && dev->parent->driver->child_pre_probe)
        dev->parent->driver->child_pre_probe(dev);
 
    clk_set_defaults(dev, CLK_DEFAULTS_PRE);
 
    if (drv->probe)
        drv->probe(dev);
 
    uclass_post_probe_device(dev);
    return 0;
}
```

这段代码里最值得注意的是父设备递归 probe。比如一个 I2C EEPROM 的父设备是 I2C controller。调用 `device_probe(eeprom)` 时，DM 会先 probe I2C controller。只有控制器时钟、pinctrl、寄存器都初始化好了，子设备才有可能通过总线访问 EEPROM。

这就是 U-Boot DM 里的依赖管理：**子设备不需要手写先初始化父总线，device\_probe 会沿 parent 链自动补齐**。

但这里也有 U-Boot 和 Linux 的明显差异。Linux 里常见 `-EPROBE_DEFER`，依赖没准备好可以延后重试；U-Boot 的启动路径更短，通常没有完整的 probe deferral 机制。父设备 probe 失败，子设备往往就是本轮启动不可用。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ea5c240c96dc7ac5050e2430be029f3e_MD5.jpg]]

## 运行：serial\_initialize 为什么能一次拉起串口

串口是理解 DM 最好的例子，因为它贯穿整个启动链路：早期日志需要串口，进入命令行也需要串口，`printf()` 最后还是要落到具体 UART 寄存器。

在 post-reloc 阶段，`serial_initialize()` 会触发串口 uclass 下设备的 probe。典型路径是：

```
12345
serial_initialize()
  → uclass_get(UCLASS_SERIAL)
  → uclass_probe_all(UCLASS_SERIAL)
  → device_probe(serial@9000000)
```

probe 过程中，ns16550 driver 的 `of_to_plat` 会读取设备树：

```
1234567
uart0: serial@9000000 {
    compatible = "ns16550a";
    reg = <0x0 0x9000000 0x0 0x1000>;
    clock-frequency = <24000000>;
    current-speed = <115200>;
};
```

然后填充 `struct ns16550_plat`：

-   -   `base` 指向 UART MMIO 基地址，例如 `0x9000000`；
-   -   `clock` 来自 `clock-frequency`；
-   -   `reg_shift`、寄存器宽度等来自平台默认或 DT 属性。

接着 `ns16550_serial_probe()` 取得 `plat` 和 `priv`：

```
12345678910111213
int ns16550_serial_probe(struct udevice *dev)
{
    struct ns16550_plat *plat = dev_get_plat(dev);
    struct ns16550 *com_port = dev_get_priv(dev);
 
    reset_get_bulk(dev, &reset_bulk);
    reset_deassert_bulk(&reset_bulk);
 
    com_port->plat = plat;
    ns16550_init(com_port, -1);
    return 0;
}
```

`ns16550_init()` 会配置波特率、数据位、FIFO 等寄存器。到这里，`udevice` 才真正从“已绑定”变成“可用”。

## 案例：serial\_puts$Hello$ 的完整路径

把上面的对象串起来，`serial_puts("Hello")` 的路径会变成：

```
12345678910
serial_puts("Hello")
  → _serial_puts(dev, "Hello")
    → for each char c
      → _serial_putc(dev, c)
        → serial_get_ops(dev)->putc(dev, c)
          → ns16550_serial_putc(dev, 'H')
            → ns16550_writeb(port, THR, 'H')
              → writeb('H', 0x9000000)
                → QEMU 捕获 MMIO 写，终端显示字符
```

这里有两个关键点。

第一，上层代码并不直接知道 `0x9000000` 这个地址。地址来自设备树，经过 `of_to_plat` 存到 `plat`，再由 driver 在 probe 和 ops 里使用。

第二，上层也不关心底层 UART 是 ns16550、dw-apb-uart 还是 PL011。它只拿到 `UCLASS_SERIAL` 下的设备，再通过 `dm_serial_ops` 调用 `putc`。硬件差异被收敛在具体 driver 的 `probe` 和 `ops` 实现里。

这就是 DM 的价值：它没有 Linux 设备模型那么完整，也没有复杂的热插拔和 runtime PM，但对 bootloader 来说已经足够。它让 U-Boot 在很短的启动路径里，用统一方式处理设备树、驱动匹配、设备分类和硬件初始化。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/313781b241e7368e4eb0430d1bc19dcc_MD5.jpg]]

## 总结

读 U-Boot DM，可以把脑图压缩成四句话：

1.  7.  **设备树给硬件描述**：`compatible`、`reg`、`clock-frequency`、`aliases` 决定设备怎么被找到；
2.  8.  **driver 给初始化逻辑**：`of_match` 匹配节点，`of_to_plat` 解析配置，`probe` 初始化硬件，`ops` 提供操作入口；
3.  9.  **uclass 给分类入口**：同类设备挂到同一个链表，上层按 `UCLASS_xxx` 找设备；
4.  10.  **udevice 给运行时状态**：一个节点对应一个运行时实例，里面挂着 plat、priv、uclass\_priv、parent、seq 等状态。

和 Linux 设备模型相比，U-Boot DM 更轻，也更直接：

| 维度 | U-Boot DM | Linux 设备模型 |
| --- | --- | --- |
| probe 失败 | 通常不自动重试 | 可通过 probe deferral 重试 |
| 生命周期 | bind → probe → 使用 → remove | probe → runtime PM → suspend/resume → remove |
| 电源管理 | 以启动期初始化为主 | 完整 runtime PM / system sleep |
| 引用计数 | 轻量对象管理 | kobject / device 引用计数更完整 |
| 匹配方式 | `U_BOOT_DRIVER(.of_match) + linker list` | bus / driver / device / of\_match 多层模型 |

调试 U-Boot 外设问题时，可以按这个顺序看：

-   • 设备树节点是否 `status = "okay"`；
    
-   -   `compatible` 是否能匹配到 U-Boot 里的 `U_BOOT_DRIVER`；
-   • 设备是否完成 bind，可以看 `dm tree`；
    
-   • 设备是否完成 probe，可以看 `dm uclass`、`dm devres` 或启动日志；
    
-   -   `of_to_plat` 是否正确解析 `reg`、clock、reset、pinctrl；
-   • driver 的 `probe` 是否因为父设备、clock、reset、power domain 失败而返回错误。
    

对应源码入口主要在这些文件：

```
12345678
common/board_r.c              board_init_r initcall 表
drivers/core/root.c           dm_init_and_scan / dm_scan
drivers/core/lists.c          lists_bind_fdt / compatible 匹配
drivers/core/device.c         device_bind / device_probe
drivers/core/uclass.c         uclass_get / uclass_probe_all
drivers/serial/serial-uclass.c 串口 uclass 行为
drivers/serial/ns16550.c      ns16550 driver 实现
```

一个结论：DM 的主线不是某个驱动怎么写，而是设备树节点怎样变成 udevice，udevice 怎样被 probe，probe 后怎样通过 uclass/ops 被上层使用。沿着这条线看，`board_init_r` 里的驱动骨架就不再是一串散乱的 initcall，而是一套清晰的启动期设备模型。

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/3890e1a6_1783566591291?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwOTUxMzQzOA%3D%3D%26mid%3D2247484162%26idx%3D1%26sn%3D207576d497a4b27e4307b6fa17158a1d%26chksm%3Dc0a83ea0af6604b8bf1595623dadd88d05668d206f9610e6e63c95b7ef8d28735c9624f51374%26mpshare%3D1%26scene%3D1%26srcid%3D07096eDMLVVE0OVYxHDER0uK%26sharer_shareinfo%3De0e23647358879ec3d98cb8ce420184b%26sharer_shareinfo_first%3De0e23647358879ec3d98cb8ce420184b%23rd&s=obsidian)