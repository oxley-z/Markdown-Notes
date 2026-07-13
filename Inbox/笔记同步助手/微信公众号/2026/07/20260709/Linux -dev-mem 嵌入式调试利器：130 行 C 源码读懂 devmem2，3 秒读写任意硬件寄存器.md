---
author: LabHub
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI5NzQxNzU0Ng==&mid=2247488842&idx=1&sn=d3ce769fd4c5fd9963c3af5a551ccf62&chksm=ed0f45826631c73fb4587b96ddd06d8718187dfda41b69869d6dcf380f19297b72d5faa12009&mpshare=1&scene=1&srcid=07091Qju3PHz2QtxgqysOTAT&sharer_shareinfo=d3242f9a1af2d5b9c992e9d3edf10336&sharer_shareinfo_first=d3242f9a1af2d5b9c992e9d3edf10336#rd
saved: 2026-07-09 15:17:57
tags:
  - 笔记同步助手
id: bae4ffa6-5046-4a86-ba7a-bf6c3abd6c6e
---

公众号名称：LabHub

作者名称：LabHub

发布时间：2026-07-09 13:28

```
// 新板子上电，GPIO 不受控制。
// 内核驱动还没写——或者驱动写错了。
// 你怀疑：寄存器地址对吗？值真的写进去了吗？
//
// 答案在这里：
devmem2 0x50002014 w 0x400
```

新板子亮了，你刚焊完第一块 prototype ，满怀期待地上电——串口没输出， LED 不闪， GPIO 完全不听使唤。你查了三遍 pinmux ，手写了寄存器地址计算，把数据手册翻到第 28 章确认了时钟门控。还是没有反应。

这种时候很难不暴躁。你不确定是硬件的问题还是软件的问题——甚至可能两边都有坑。板子是你焊的，代码是你写的，锅甩不出去。

你开始怀疑：我写的寄存器值，到底有没有真正进到那个物理地址里？

**传统做法**：改内核驱动，加 `printk`，编译，烧写，重启。一轮 5 分钟。 10 个寄存器就是将近一个小时。时间就这么烧掉了。这效率搁谁身上都绷不住——懂的都懂。

**一条命令就够了**。 `devmem2` 让你在用户空间——注意，不是内核空间——直接读写任意物理地址。就像用 `printf` 调试用户程序一样调试硬件寄存器。

> ✏️ **编辑建议**：在这里加一句你第一次用 devmem2 的场景——是哪块板子？哪个寄存器让你调了大半天？

## 为什么你需要这个工具

嵌入式 Linux 驱动开发有一个经典死循环：

改代码 → 编译 → 烧写 → 重启 → 观察 → 发现不对 → 改代码 → ...

这个循环的每一步都是时间。而有经验的嵌入式工程师都知道——**大部分调试时间不是在"写代码"，而是在"确认硬件到底收到了什么值"**。

你说你要配一个 I2C 控制器的时钟分频寄存器。数据手册写得很清楚：地址 0xFF1A0000 ， bit\[2:0\] 是分频比。你写了 `writel(4, 0xFF1A0000)`， I2C 还是不通。

是地址写错了？是时钟没开？是 pinmux 没配对？

你**不知道**。你唯一能做的，就是再加一行 `printk`，再等一轮编译。

用 `devmem2`，这个循环变成：**敲命令 → 看到结果。 3 秒**。

这个差距，不是快了多少，是根本改变了调试的交互模式。就像你调试用户程序有 GDB ，调试硬件寄存器却没有——`devmem2` 就是那个缺失的"寄存器 GDB"。

## 130 行源码，核心 35 行——逐行拆

`devmem2` 是 **Jan-Derk Bakker**（荷兰代尔夫特理工大学）在 2000 年为 LART 计算板写的， GPL v2 。整份代码 **130 行**，其中核心逻辑——从 `open("/dev/mem")` 到 `munmap`——不到 35 行。

先说全景。这份代码做的事情，本质上只有三步：

```
用户空间 (devmem2)          内核空间                  硬件
──────────────────────────────────────────────────────────
open("/dev/mem")       →   drivers/char/mem.c
mmap(phys_addr)        →   remap_pfn_range()       →  物理 MMIO 寄存器
*(virt_addr)           →   直接总线访问              →  读/写寄存器
```

每一步背后都有一层内核机制在干活。下面拆开讲。

### 第一步：打开 `/dev/mem`——这不是一个文件

```
if((fd = open("/dev/mem", O_RDWR | O_SYNC)) == -1) FATAL;
```

看起来就是普通 `open`，但你打开的不是磁盘上的文件。`/dev/mem` 是内核注册的一个**字符设备**，源码在 `drivers/char/mem.c`。它的 `open` 回调做了一件事：检查调用者有没有 `CAP_SYS_RAWIO`。有就返回一个 fd ，没有就拒绝。

**这意味着你不需要 root** 。 只要进程有 `CAP_SYS_RAWIO`——容器、 systemd service 都可以给——就能用。很多工程师以为 `/dev/mem` 是 root-only ，其实不是。这是内核的能力模型，比 UID 更细粒度。

但真正的细节在 `O_SYNC`。不加这个 flag 会怎样？

现代 CPU 有 store buffer 。你执行 `*(volatile unsigned long *)addr = 0x400`，这个写操作可能先在 L1 cache 里待着，过几十个时钟周期才真正到达总线。对于普通内存，这是性能优化。对于硬件寄存器，这是灾难——你以为已经拉高了 GPIO ，实际上电平还没变。

`O_SYNC` 告诉内核：这个 fd 上的 `mmap` 必须用 uncached 映射。映射完成后，每次指针解引用都是一次真正的总线事务。代价是慢了几个数量级，但你调试的时候不在乎性能。

**这里有一个坑**。 如果你忘了 `O_SYNC`，读操作可能返回的是 cache 里的旧值——你以为寄存器是 0x0 ，其实是刚才那轮 DMA 已经把新数据写进去了。这个问题在 ARM Cortex-A 上尤其常见，因为 ARM 的 cache coherency 模型比 x86 更松散。

### 第二步：`mmap`——页表级别的物理映射

```
map_base = mmap(0, MAP_SIZE,
                PROT_READ | PROT_WRITE, MAP_SHARED,
                fd, target & ～MAP_MASK);
```

这是整个工具最核心的一行。逐参数拆：

**`target & ～MAP_MASK`**：`MAP_MASK` 是 `4095`（ 0xFFF ）。`～MAP_MASK` 就是把低 12 位清零。你要访问的物理地址是 `0x50002014`，`mmap` 的 offset 参数却是 `0x50002000`——因为 `mmap` 的最小映射单位是 4KB 页。你不能只映射一个寄存器，你必须映射包含它的整页。

**`MAP_SHARED`**：这把映射标记为"共享"。对于 `/dev/mem`，这意味着页表项直接指向物理地址，而不是创建一个 copy-on-write 副本。如果你用 `MAP_PRIVATE`——你写的值只会留在你的进程地址空间里，硬件永远看不到。

**那内核这端做了什么**？

`mmap` 系统调用落到 `drivers/char/mem.c` 的 `mmap_mem()`：

1.  先调 `valid_mmap_phys_addr_range()`——检查你要映射的物理地址是否在允许范围内。这就是 `CONFIG_STRICT_DEVMEM` 起作用的地方：如果开了这个 config ，只有 `/proc/iomem` 里登记的 MMIO 区域才能通过，系统 RAM 会被拒绝。
2.  然后调 `phys_mem_access_prot()`——根据架构设置页表属性。 x86 上设 `PCD`（ Page Cache Disable ）位， ARM 上设 `nGnRE` 内存属性。
3.  最后 `remap_pfn_range()` 在进程页表中填入 PTE 。**没有任何内存分配、没有数据拷贝**。 映射完成后， CPU 的 MMU 直接把虚拟地址翻译到物理地址。

这跟内核驱动里的 `ioremap()` 是**完全同一个底层机制**。区别只在于：`ioremap` 把物理地址映射到**内核虚拟空间**（`vmalloc` 区域），`devmem2` 的 `mmap` 映射到**用户虚拟空间**。内核态和用户态的页表是两套，但页表项指向的是同一块物理地址。

> **\*\*关键理解\*\*：\`ioremap\` 和 \`devmem2\` 的 \`mmap\`，就像从两个不同的门进入同一个房间。门不一样，房间是同一个。**

### 第三步：`virt_addr = map_base + offset`

```
virt_addr = map_base + (target & MAP_MASK);
```

这一行的作用简单到让人不安。`target & MAP_MASK` 提取物理地址的低 12 位（ 0x50002014 → 0x014 ）。`map_base + 0x014` 就是你要操作的寄存器的虚拟地址。

为什么能直接加？因为 `mmap` 返回的虚拟地址和物理地址在**页内偏移量上是一致的**。内核在建立映射时保持了这种一致性——这不是巧合，是 POSIX 规范要求的。

但有一个架构陷阱。在 MIPS 和一些古老的 PowerPC 上，内核可能返回一个"有偏移"的虚拟地址——页内偏移和物理地址不一致。这就是为什么你在一些古老的嵌入式 Linux 代码里看到 `bus_to_virt()` 宏。 x86 和 ARM 上不用担心这个——但知道这件事的存在，能帮你理解为什么这段 20 年前的代码能在今天的所有主流平台上直接编译运行。

### 第四步：指针解引用读写寄存器

```
// 读
switch(access_type) {
    case 'b': read_result = *((unsigned char *) virt_addr);  break;
    case 'h': read_result = *((unsigned short *) virt_addr); break;
    case 'w': read_result = *((unsigned long *) virt_addr);  break;
}

// 写（如果命令行给了 data 参数）
if(argc > 3) {
    writeval = strtoul(argv[3], 0, 0);
    switch(access_type) {
        case 'b': *((unsigned char *) virt_addr) = writeval;  break;
        case 'h': *((unsigned short *) virt_addr) = writeval; break;
        case 'w': *((unsigned long *) virt_addr) = writeval;  break;
    }
}
```

这是整个工具最让人不安的地方。没有 `ioread32()`。没有 `writel()`。没有 `volatile`。就是 `*(unsigned long *)addr`。

它之所以能工作，是因为前面两个决策——`O_SYNC` 和 `MAP_SHARED`——已经把所有的硬件保证都提供了：  
\- **不要 volatile**：因为 `O_SYNC` 下的 `mmap` 已经让每次解引用绕过 cache ， volatile 在这里是多余的  
\- **不要 memory barrier**：对 MMIO 的单次读写不需要 barrier——barrier 是用来保证多个操作的顺序的， devmem2 一次只做一个操作  
\- **不要 `ioread32`**：那些内核函数做的事和这里完全一样——解引用一个映射好的指针。它们的存在是为了给内核驱动提供可移植的 API ，不是因为有额外的硬件语义

一个你读源码时不会注意到、但实际非常重要的事：**`unsigned long` 在你的平台上到底是几字节**？

32 位 ARM 上是 4 字节， 64 位 ARM 上是 8 字节。如果你在 AArch64 板上用 'w' 模式读一个 32 位寄存器——没问题，因为 `unsigned long` 是 8 字节但寄存器只占 4 字节，高位自动填 0 。但如果你用 'w' 模式写——你会覆盖相邻的 4 字节。这是 `devmem2` 最经典的坑。

原作者在 `usage` 里写了 `[b]yte, [h]alfword, [w]ord`——他一直用 32 位的 word 语义，因为 LART 是 32 位 ARM 板。你在 64 位平台上应该用 'h'（ 2 字节）和 'w'（ 4 字节）谨慎搭配。或者给 `devmem2` 加一个 'd'（ double word ， 8 字节）模式。

## 在板子上跑起来

看懂源码，实操验证：

```
wget https://bootlin.com/pub/mirror/devmem2.c

arm-linux-gnueabihf-gcc -o devmem2 devmem2.c

scp devmem2 root@raspberrypi:/usr/local/bin/

ssh root@raspberrypi
devmem2 0x3F200000 w

devmem2 0x3F200000 w 0x00001000
```

如果你用的是全志/瑞芯微/ST 的板子， GPIO 基址不同，步骤完全一样。**不需要 dts ，不需要内核模块，不需要重启**。 就一条命令。

> ✏️ **编辑建议**：在这里加你常用的一块板子的 GPIO 基址和一次实际的调试经历。

## 该说的坑要说清楚

`devmem2` 的设计哲学就是"不拦你"——它不检查地址是否合法、不关心有没有别人在用这个寄存器、不管 cache coherency 。这是它的优势（零摩擦），也是它的弱点。

**STRICT\_DEVMEM** 。 Linux 5.x+ 默认开了 `CONFIG_STRICT_DEVMEM`，只允许 `mmap` 访问 `/proc/iomem` 里登记的 MMIO 范围。你想用它读系统 RAM 里其他进程的数据？不行。调试硬件寄存器？完全没问题。如果需要在调试板上绕过限制，启动参数加 `iomem=relaxed`——**但只在调试板上用**。

**没有同步**。 devmem2 不管 SMP 、不管中断、不管竞态。读写的那个寄存器如果恰好被内核驱动也在操作——结果不保证。不要在生产环境用它改寄存器，除非你知道那个寄存器此刻没有任何人在碰。

**ARM64 的 nGnRE 属性**。 ARMv8 要求设备内存必须用 non-Gathering 、 non-Reordering 、 Early Write Acknowledge 属性映射。`devmem2` 本身不处理这个——它依赖内核的 `phys_mem_access_prot()` 正确设置。如果你遇到 bus error （ SIGBUS ），检查 `/proc/iomem` 确认目标地址确实被标记为 MMIO 区域。

**64 位的 unsigned long 陷阱**。 上文已经讲了——AArch64 上用 'w' 模式读一个 32 位寄存器没问题，写会覆盖相邻 4 字节。这是源码级的坑，不是内核的锅。

`ioremap` 保证了对齐、 cache 策略、 barrier 语义。`devmem2` 把这层保护全部剥掉了。它适合**调试和快速验证**——写完驱动之后再回来用 `devmem2` 做 sanity check 也很有用。但它不替代驱动。

> **\*\*一句话总结\*\*：\`devmem2\` 做的事情本质上只有三个系统调用——\`open\`、\`mmap\`、指针解引用。内核的 \`remap\_pfn\_range\` 替你做完了所有脏活。 130 行源码，核心 35 行， 20 年没过时。**
> 
> 如果你在调试硬件寄存器，能在 3 秒内读写任意物理地址——而不是等 5 分钟编译重启——调试效率不是一个数量级。

源码在那里， 20 多年了，一个 C 文件，没有任何依赖。你可以 fork 它、魔改它、集成到你的自动化测试脚本里。

但只要 `/dev/mem` 还在，只要内核还允许 `mmap` 物理地址，这件事就会一直成立。

硬件是物理的。寄存器是真实的。你只是需要一条命令去**确认**它。

`devmem2` 就是那条命令。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/b46f5a19_1783581474972?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI5NzQxNzU0Ng%3D%3D%26mid%3D2247488842%26idx%3D1%26sn%3Dd3ce769fd4c5fd9963c3af5a551ccf62%26chksm%3Ded0f45826631c73fb4587b96ddd06d8718187dfda41b69869d6dcf380f19297b72d5faa12009%26mpshare%3D1%26scene%3D1%26srcid%3D07091Qju3PHz2QtxgqysOTAT%26sharer_shareinfo%3Dd3242f9a1af2d5b9c992e9d3edf10336%26sharer_shareinfo_first%3Dd3242f9a1af2d5b9c992e9d3edf10336%23rd&s=obsidian)