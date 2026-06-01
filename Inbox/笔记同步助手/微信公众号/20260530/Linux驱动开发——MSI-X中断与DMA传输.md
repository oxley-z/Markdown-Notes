---
author: F学社
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg5Mjk3NjU0MA==&mid=2247486183&idx=1&sn=96a3fd6bbfc784ee142641aa3cf97351&chksm=c10727279723116fd697ff2d41f602fcf0e7d3cda772009e39944b3e00919fe4f4ca4bb8441e&mpshare=1&scene=1&srcid=0530p03MlEAwfsKQkz5mEHlc&sharer_shareinfo=035a3e0fdbd9ba7d1cf813f31ba2f3c8&sharer_shareinfo_first=035a3e0fdbd9ba7d1cf813f31ba2f3c8#rd
saved: 2026-05-30 12:45:40
tags:
  - 笔记同步助手
id: 0c9465f1-c091-49fa-b603-9e84b029a97d
---

公众号名称：F学社

作者名称：F学社

发布时间：2026-05-27 07:30

原文链接：[https://docs.qq.com/document/DU01CWHR4ZVRRRmZ5](https://docs.qq.com/document/DU01CWHR4ZVRRRmZ5)

上一期讲完XDMA架构，兴冲冲打开Xilinx官方驱动，改完描述符和通道配置准备上线。

可真正上板加载驱动，接连碰到**MSI-X注册失败、读到的数据全是0、CPU被中断占满卡死**……这些官方手册一笔带过甚至完全不提的问题，也是驱动开发最折磨人的环节。

**「IRQ handler took too long」「系统卡死」「数据全是0」**

这几个问题，全是坑

> 📌 **前置知识**：本文假设你已经熟悉XDMA基本概念（见上一期《XDMA架构——SG DMA怎么工作的》），并具备Linux内核模块基础。

---

## 第一个真实痛苦：MSI-X注册失败，系统根本进不去

你照着手册写：

ret = pci\_enable\_msix(pfdev, entries, 32);

if (ret) {

dev\_err(&pfdev->dev, "MSI-X enable failed: %d\\n", ret);

return ret;

}

一加载驱动——

text

pcilib: 0000:01:00.0: can't enable MSI-X for rc

系统直接报错，驱动加载失败。

**为什么？**

BIOS把MSI-X的路由给别的设备了。PCIe RC（Root Complex）有硬件限制，**一个设备能用的MSI-X向量数由BIOS分配，不是你想用多少就用多少**。

**三步排查：**

\# 1. 先看系统到底给了多少MSI-X向量

cat /proc/interrupts | grep -i xdma

\# 2. 看PCIe设备的MSI-X能力

lspci -vvv -s 01:00.0 | grep -A 10 MSI-X

\# 3. 如果向量数不够，试试减少请求数（4～8个是最稳妥的选择）

```
pci_enable_msix_range
```

通常**4～8个向量**是最稳妥的选择，覆盖主要场景就够了。32个向量在某些服务器平台上会触发BIOS限制。

---

## 第二个真实痛苦：中断来了，但数据全是0

驱动加载成功了，XDMA也启动了，MSI-X也触发了。但用户态读到数据全是0。

你在isr里加了打印：

static irqreturn\_t xdma\_isr(int irq, void \*dev\_id)

{

pr\_err("%s: IRQ fired!\\n", \_\_func\_\_);

return IRQ\_HANDLED;

}

打印出来了，说明中断确实来了。但数据呢？

**问题在哪？**

你忘了**映射DMA缓冲区到用户态**。

很多教程里的mmap实现是错的：

// ❌ 错误做法：直接映射BAR

static int xdma\_mmap(struct file \*filp, struct vm\_area\_struct \*vma)

{

struct xdma\_dev \*dev = filp->private\_data;

// 直接映射BAR——PCIe BAR的延迟很高

if (remap\_pfn\_range(vma, vma->vm\_start,

pci\_resource\_start(pfdev, 0) >> PAGE\_SHIFT,

vma->vm\_end - vma->vm\_start,

vma->vm\_page\_prot))

return -EAGAIN;

return 0;

}

PCIe BAR的读延迟是**200～300ns**（见PCIe协议篇），你这样映射后，用户态读4字节就要等300ns。这不是零拷贝，这是零优化。

**正确做法：用dma\_alloc\_coherent分配专用DMA缓冲区：**

// ✅ 正确做法

static int xdma\_mmap(struct file \*filp, struct vm\_area\_struct \*vma)

{

struct xdma\_dev \*dev = filp->private\_data;

unsigned long size = vma->vm\_end - vma->vm\_start;

// 映射的是DMA一致的专用内存缓冲区，不是BAR

if (remap\_pfn\_range(vma, vma->vm\_start,

dev->host\_buf\_dma >> PAGE\_SHIFT,

size, vma->vm\_page\_prot))

return -EAGAIN;

return 0;

}

// 分配DMA缓冲区（probe里）

dev->host\_buf = dma\_alloc\_coherent(&pfdev->dev, DMA\_BUF\_SIZE,

&dev->host\_buf\_dma, GFP\_KERNEL);

XDMA的C2H通道**直接把数据写进这块DMA内存**，mmap映射后用户态读的就是这块内存。零拷贝、延迟低、带宽高。

---

## 第三个真实痛苦：中断风暴，系统卡死

跑了一会儿，系统突然变卡。`top`一看，一个CPU核100%在处理中断。

这就是**中断风暴（Interrupt Storm）**。

原因：描述符完成后，**中断状态寄存器没有被清除**。

// ❌ 错误：读完状态就完了，没有写回去清除

static irqreturn\_t xdma\_isr(int irq, void \*dev\_id)

{

struct xdma\_dev \*dev = dev\_id;

u32 status = readl(dev->bar + XDMA\_C2H\_STATUS);

// 读取状态后直接返回，硬件认为中断未处理

// 硬件会一直触发同一个中断

return IRQ\_HANDLED; // 中断标志没有清！

}

💡**XDMA中断状态寄存器是写1清除（Write-1-to-Clear）**，读出来的是当前状态，写对应位1清除该中断。

**正确做法：写1清零：**

static irqreturn\_t xdma\_isr(int irq, void \*dev\_id)

{

struct xdma\_dev \*dev = dev\_id;

// 寄存器偏移基于常规XDMA配置，实际项目请以PG195手册为准

\#define XDMA\_C2H\_INT\_STATUS 0x1048

\#define XDMA\_C2H\_DONE\_PIDC 0x1050

u32 status = readl(dev->bar + XDMA\_C2H\_INT\_STATUS);

if (!(status & 0x1)) // 检查C2H完成中断

return IRQ\_NONE;

// 写1清零

writel(status, dev->bar + XDMA\_C2H\_INT\_STATUS);

// 读取完成的描述符索引

u32 done\_idx = readl(dev->bar + XDMA\_C2H\_DONE\_PIDC);

done\_idx &= 0xFFFF;

// 备注：0xFF为掩码，适用于描述符数量≤255场景，数量更大请调整掩码位数

set\_bit(done\_idx & 0xFF, &dev->completed\_mask);

wake\_up\_interruptible(&dev->waitq);

// 重要原则：中断服务程序禁止耗时操作，复杂业务请转交工作队列/用户态处理

return IRQ\_HANDLED;

}

另外还有个常见问题：**每个描述符都触发中断**。如果是1MHz采样率、每描述符1KB数据，32个描述符全满的时间约32ms，中断频率约30次/秒，这还好。但如果描述符很小（4KB）且每个都触发中断，中断频率会很高，CPU扛不住。

**优化方案：用轮询（poll）替代高频中断**

// 用户态用poll/epoll替代interrupt

static \_\_u32 xdma\_poll(struct file \*filp, poll\_table \*wait)

{

struct xdma\_dev \*dev = filp->private\_data;

\_\_u32 mask = 0;

poll\_wait(filp, &dev->waitq, wait);

if (dev->completed\_mask != 0)

mask |= POLLIN | POLLRDNORM;

return mask;

}

用户态：

int fd = open("/dev/xdma0", O\_RDWR);

struct pollfd pfd = { .fd = fd, .events = POLLIN };

while (1) {

poll(&pfd, 1, 1000); // 等待数据，超时1s

if (pfd.revents & POLLIN) {

// 有数据了，读取

int idx = find\_first\_bit(&completed\_mask);

process\_buffer(idx);

}

}

第四个真实痛苦：probe里申请的内存，remove里忘了释放

static void xdma\_remove(struct pci\_dev \*pfdev)

{

struct xdma\_dev \*dev = pci\_get\_drvdata(pfdev);

// ❌ 漏了dma\_free\_coherent——内核内存泄漏

// ❌ 漏了pci\_disable\_msix

// ❌ 漏了free\_irq

iounmap(dev->bar);

pci\_disable\_device(pfdev);

kfree(dev); // 只有这个

}

一加载卸载驱动几十次，系统内存就没了。

**完整remove：**

static void xdma\_remove(struct pci\_dev \*pfdev)

{

struct xdma\_dev \*dev = pci\_get\_drvdata(pfdev);

if (!dev) return;

// 停止DMA（C2H控制寄存器偏移0x1000，写0停止）

writel(0, dev->bar + 0x1000);

// 释放中断

for (int i = 0; i < NUM\_MSIX; i++)

free\_irq(dev->msix\_entries\[i\].vector, dev);

pci\_disable\_msix(pfdev);

// 释放DMA缓冲区（不能忘！）

if (dev->host\_buf)

dma\_free\_coherent(&pfdev->dev, DMA\_BUF\_SIZE,

dev->host\_buf, dev->host\_buf\_dma);

// 释放描述符环（如果有）

if (dev->desc\_ring)

dma\_free\_coherent(&pfdev->dev, dev->desc\_ring\_size,

dev->desc\_ring, dev->desc\_ring\_dma);

// 释放BAR映射

if (dev->bar)

iounmap(dev->bar);

pci\_disable\_device(pfdev);

kfree(dev);

}

调试清单：按这个顺序查

当你遇到“数据读不到/中断不触发/系统卡死”，按这个顺序排查：

□ dmesg | grep -i xdma ← 先看内核日志，有没有报错

□ cat /proc/interrupts | grep xdma ← 看中断有没有注册上

□ lspci -vvv -s 01:00.0 | grep -i msi ← 看MSI-X向量数

□ readl(bar + XDMA\_C2H\_INT\_STATUS) ← 直接读寄存器看状态

□ 描述符的src\_addr/dst\_addr是不是64B对齐

□ 描述符的control字段有没有设置CONTROL\_COMPLETE\_EN（中断使能）

□ MSI-X中断有没有正确清除（写1清零）

□ DMA缓冲区有没有正确分配（dma\_alloc\_coherent）

□ remove里所有资源是否都有对应的释放

关键代码：完整的probe+remove

c

📥 \*\*源码下载\*\*由于公众号正文篇幅有限，代码请点击阅读原文

## 总结

| 痛苦点 | 根因 | 标准解法 |
| --- | --- | --- |
| MSI-X 注册失败 | BIOS/PCIe RC 限制最大向量数 | 降低申请向量至4～8个，用命令行排查硬件能力 |
| 读取数据全0 | 错误映射 BAR 空间，未使用 DMA 一致性内存 | 用`dma_alloc_coherent`分配缓冲区，mmap映射该内存 |
| 中断风暴、CPU 100% | 中断状态未执行「写1清零」；中断频率过高 | ISR内主动写回状态寄存器清中断；高频场景改用 poll/epoll |
| 内核内存泄漏 | remove 函数未成对释放资源 | probe申请的中断、MSI-X、DMA内存、BAR映射，在 remove 中全部释放 |

---

**关注我**，下一期讲用户态开发——如何正确读取DMA缓冲区、poll机制、ring buffer设计，写出一个工业级的FPGA驱动用户态程序。

---

![[Inbox/笔记同步助手/微信公众号/20260530/images/0a94f29f330507aef8f65ba831804847_MD5.jpg|cover_image]]

Original F学社 F学社

Read more

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/c4d5ab55_1780116337392?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg5Mjk3NjU0MA%3D%3D%26mid%3D2247486183%26idx%3D1%26sn%3D96a3fd6bbfc784ee142641aa3cf97351%26chksm%3Dc10727279723116fd697ff2d41f602fcf0e7d3cda772009e39944b3e00919fe4f4ca4bb8441e%26mpshare%3D1%26scene%3D1%26srcid%3D0530p03MlEAwfsKQkz5mEHlc%26sharer_shareinfo%3D035a3e0fdbd9ba7d1cf813f31ba2f3c8%26sharer_shareinfo_first%3D035a3e0fdbd9ba7d1cf813f31ba2f3c8%23rd&s=obsidian)