---
author: baron
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247515096&idx=1&sn=673229755dedae52a3a7bfcd95378459&chksm=fcf9f95b47dabae3452ec9c8d4f48e6ed430c8db7f304dad5a43a977063093f840a24d671cd4&mpshare=1&scene=1&srcid=05279648psLyxeT8iLhjyZtw&sharer_shareinfo=18890fc195b3ee7d05f22f261e744816&sharer_shareinfo_first=18890fc195b3ee7d05f22f261e744816#rd
saved: 2026-05-27 08:39:20
tags:
  - 笔记同步助手
id: be410d40-cb23-45a1-ab4e-e0afc35c6ea9
---

公众号名称：Arm精选

作者名称：baron

发布时间：2026-05-27 07:58

原文链接：[https://appoxpc9kws6959.h5.xiaoeknow.com](https://appoxpc9kws6959.h5.xiaoeknow.com)

点击左上方蓝色“Arm精选”，选择“设为星标”

  

#### 前言

为什么要学习虚拟化？作为一名安全领域的渣渣 ，有必要去学习虚拟化技术，因为它也属于ARM安全架构的一部分。![](https://relay-1.bijitongbu.site/p/8a3eb3fb882c1c73a2dbcbe5d8080c65.png)

​

#### 1.概述

看完之后你将学会：

​

-   两种类型的hypervisor（Type 1 Hypervisor、Type 2 Hypervisor），以及他们如何映射到Arm异常等级。
    
-   解释operation trapped以及如何模拟operation
    
-   能够列出hypervisor能够产生的虚拟异常以及产生这些异常的机制
    

#### 2.虚拟化介绍

下文的hypervisor泛指：用于创建、管理、调度虚拟机的软件。

​

##### 2.1 虚拟化为什么重要

虚拟化是一种广泛使用的技术，几乎支持所有现代云计算和企业基础架构。开发人员使用虚拟化在一台机器上运行多个操作系统 (OS)，并在不破坏主计算环境的情况下测试软件。

虚拟化在服务器系统中很流行，大多数服务器级处理器都要求支持虚拟化。这是因为虚拟化为数据中心提供了非常理想的功能，包括：

​

-   隔离性(Isolation)
    
-   高可用性(High Availability)：虚拟机迁移
    
-   负载均衡(Workload balancing)：
    
-   沙箱(Sandboxing)
    

##### 2.2 hypervisors的两种类型

-   **standalone 或 Type 1 hypervisors** Hypervisor直接在硬件上运行，并完全控制硬件平台及其所有资源，包括 CPU 和物理内存。其上的虚拟机可以运行一个或多个完整的guest os.![](https://relay-1.bijitongbu.site/p/e13817aaf5e38c254de2d468b8d840cb.png)
    
-   **hosted 或 Type 2 hypervisors** (1)完全控制硬件平台及其所有资源，包括 CPU 和物理内存 (2)如果您以前使用过 Virtual Box 或 VMware Workstation 等软件，那么这就是您正在运行的虚拟机管理程序类型。​操作系统（称为主机操作系统）安装在平台上，管理程序在主机操作系统内运行，利用现有功能来管理硬件。​管理程序然后可以托管虚拟机，这些虚拟机本身运行操作系统。​我们将其称为来guest os。​  ![](https://relay-1.bijitongbu.site/p/22af057007dbe3ccef75b2f0a40ae92d.png) Arm 平台上最常用的两个开源虚拟机管理程序是 Xen（type 1）和 KVM（type 2）。我们将使用这些管理程序来说明本文中的一些要点。当然除此之外还有许多其他的开源或专业的hypervisor。
    

##### 2.3 全虚拟化和半虚拟化

**VM 的经典定义**是一个独立的、隔离的计算环境，与真实的物理机无法区分。尽管可以在基于 Arm 的系统上完全模拟真实机器，但这通常不是一件有效的事情。因此，这种模拟并不经常进行。例如，模拟真实的以太网设备很慢，因为每次访问由guest os执行的模拟寄存器都必须由hypervisor在软件中处理。这种处理可能比访问物理设备上的寄存器要昂贵得多。

通常用于提高性能的首选替代方案是 **enlighten Guest OS**。通过让guest os知道它正在 VM 中运行，并通过提供在hypervisor中模拟并从Guest OS访问时具有良好性能的虚拟设备 ，guest os可以获得良好的性能，即使对于IO。

严格来说，**全系统虚拟化**模拟真实的物理机。另一方面，Xen（开源项目）推广了术语“paravirtualization”，其中guest os的核心部分被修改为在虚拟硬件平台而不是物理机上运行。进行此修改是为了提高性能。

今天，在大多数具有虚拟化硬件支持的架构上，包括 Arm，Guest OS 大部分都未经修改地运行。guest os认为它在真实硬件上运行，除了块存储和网络等 I/O 外围设备的驱动程序，它们使用**半虚拟化**设备和设备驱动程序。这种半虚拟化 I/O 设备的例子是 Virtio 和 Xen PV Bus。

​

##### 2.4 虚拟机和虚拟CPUs

了解虚拟机 (VM) 和虚拟 CPU (vCPU) 之间的区别很重要。一个 VM 将包含一个或多个 vCPU，如下图所示：![](https://relay-1.bijitongbu.site/p/edada91250eed14090be3af5f936b11b.png)当我们查看本文中的其他一些章节时，VM 和 vCPU 之间的区别将变得很重要。例如，内存页面可能会分配给 VM，因此该 VM 中的所有 vCPU 都可以访问该内存页。但是，虚拟中断是针对特定的 vCPU，并且只能转到该 vCPU。

注意：严格来说，我们应该指的是虚拟处理元素 (vPE)，而不是 vCPU。请记住，处理元件 (PE) 是实现 Arm 架构的机器的通用术语。本指南使用 vCPU 而不是 vPE，因为 vCPU 是大多数人熟悉的术语。但是，在体系结构规范中，使用术语 vPE

​

#### 3.AArch64中的虚拟化

运行在EL2之上的软件可以访问并控制虚拟化功能：

​

-   Stage 2 转换
    
-   EL1/0指令和寄存器访问
    
-   产生虚拟化异常
    

非安全状态和安全状态的异常等级如下所示：![](https://relay-1.bijitongbu.site/p/bf3cefa09081bb52121122d41bbaf36f.png)在图中，Secure EL2 显示为灰色。这是因为之前的话是没有S-EL2的(armv8.4及其之后是有了的)。这在有关安全虚拟化的部分中进行了讨论

架构中还有一些功能支持：

​

-   Secure virtualization
    
-   Hosted, or Type 2, hypervisors
    
-   Nested virtualization(嵌套虚拟化)
    

#### 4\. Stage 2 translation

##### 4.1 What is stage 2 translation?

Stage 2 translation允许hypervisor 控制虚拟机 (VM) 中的内存视图。具体来说，它允许管理程序控制 VM 可以访问哪些内存映射系统资源，以及这些资源出现在 VM 的地址空间中的位置。

这种控制内存访问的能力对于isolation和sandboxing很重要。Stage 2 translation 可用于确保 VM 只能看到分配给它的资源，而看不到分配给其他 VM 或hypervisor的资源。

对于内存地址翻译，stage 2 translation是转换的第二个stage。为了支持这一点，需要一组新的页表，称为 Stage 2 表，如下所示：![](https://relay-1.bijitongbu.site/p/de63fc87a9f832a3c922cf3f3477e240.png)操作系统 (OS) 控制一组从虚拟地址空间映射到它认为是物理地址空间的转换表。然而，这个过程会经历第二次到真实物理地址空间的转换。第二阶段由hypervisor控制。

操作系统控制的转换称为第 1 阶段转换，hypervisor控制的转换称为第 2 阶段转换。操作系统认为是物理内存的地址空间称为中间物理地址 (IPA) 空间。

注意：有关地址翻译是如何工作的，请参考MMU相关博文

用于第 2 阶段的转换表的格式与用于第 1 阶段的格式非常相似。但是，某些属性在第 2 阶段的处理方式不同，像type、normal或device被直接存放到table entry中，而不是通过 MAIR\_ELx 寄存器。

​

##### 4.2 VMIDs

每个 VM 都分配有一个虚拟机标识符 (VMID) 。VMID 用于标记translation lookaside buffer(TLB) 条目，以标识每个entry属于哪个 VM。此标记允许多个不同 VM 的翻译同时出现在 TLB 中

VMID 存储在 VTTBREL2 中，可以是 8 位或 16 位。VMID 由 VTCREL2.VS 位控制。对 16 位 VMID 的支持是可选的，并且是在 Armv8.1-A 中添加的。

注意：EL2 和 EL3 翻译机制的翻译没有用 VMID 标记，因为它们不受第 2 阶段翻译的约束。

​

##### 4.3 VMID interaction with ASIDs

TLB entry也可以使用地址空间标识符 (ASID) 进行标记。操作系统为应用程序分配了一个 ASID，该应用程序中的所有 TLB 条目都使用该 ASID 进行标记。这意味着不同应用程序的 TLB entry能够共存于 TLB 中，而没有一个应用程序使用属于不同应用程序的 TLB entry的可能性。

每个 VM 都有自己的 ASID 命名空间。例如，两个 VM 可能都使用 ASID 5，但它们将它们用于不同的事情。ASID 和 VMID 的组合很重要。

​

##### 4.4 Attribute combining and overriding (属性组合和覆盖)

第 1 阶段和第 2 阶段映射都包含属性，如类型和访问权限。内存管理单元 (MMU) 将两个阶段的属性结合起来，给出最终的有效值。MMU 通过选择更具限制性的阶段来完成此操作，如下所示：![](https://relay-1.bijitongbu.site/p/51493f00a052bd2587dc55847febff8c.png)在此示例中，device比normal限制更多。因此，结果类型为 device。如果我们颠倒示例，结果将是相同的.

这种组合属性的方法适用于大多数用例，但有时hypervisor 可能想要覆盖此行为。例如，在 VM 的早期启动期间。对于这些情况，有一些控制位会覆盖normal行为：

​

-   HCR\_EL2.CD. This makes all stage 1 attributes Non-cacheable.
    
-   HCR\_EL2.DC. This forces stage 1 attributes to be Normal, Write-Back Cacheable.
    
-   HCREL2.FWB. This allows stage 2 to override the stage 1 attribute, instead of regular attribute combining （Note: HCREL2.FWB was introduced in Armv8.4-A）
    

##### 4.5 Emulating Memory-mapped Input/Output (MMIO)

与物理机上的物理地址空间一样，VM 中的 IPA 空间包含用于访问内存和外围设备的区域，如下所示：![](https://relay-1.bijitongbu.site/p/760923dab5b2e3cf6ee84c354dcb949a.png)VM 可以使用peripheral regions访问physical peripherals（通常称为直接分配的外围设备）和virtual peripherals。

virtual peripherals由hypervisor在软件中完全模拟，如下图所示：![](https://relay-1.bijitongbu.site/p/5538e1e0f0a90716d90662ec42f072b4.png)

assigned peripheral是已分配给 VM 并映射到其 IPA 空间的真实物理设备。这允许在 VM 中运行的软件直接与外围设备交互。

virtual peripheral是hypervisor将在软件中模拟的外围设备。相应的第 2 阶段table entries 将被标记为错误。VM 中的软件认为它可以直接与外设对话，但每次访问都会触发第 2 阶段fault，hypervisor将在异常处理程序中模拟外设访问。

要模拟外设，管理程序不仅需要知道访问了哪个外设，还需要知道访问了该外设中的哪个寄存器、访问是读还是写、访问的大小以及用于传输数据的寄存器 .

异常模型引入了 FARELx 寄存器。在处理stage 1 fault时，这些寄存器报告触发异常的虚拟地址。虚拟地址对hypervisor没有帮助，因为hypervisor通常不知道来guest os如何配置其虚拟地址空间。对于stage 2 fault，还有一个额外的寄存器 HPFAREL2，它报告中止地址的 IPA。由于 IPA 空间由hypervisor控制，因此它可以使用此信息来确定需要模拟的寄存器。

异常模型显示了 ESR\_ELx 寄存器如何报告有关异常的信息。对于触发stage 2 fault的单个通用寄存器加载或存储，提供了额外的综合信息。该信息包括访问的大小和源或目标寄存器，并允许管理程序确定对虚拟外围设备进行的访问类型。

下图说明了捕获异常然后模拟访问的过程：![](https://relay-1.bijitongbu.site/p/f5b0fbf8146a7ca5f671267eb9563ac0.png)这个过程包含以下步骤：

​

-   (1) VM 中的软件尝试访问虚拟外围设备。在本例中，这是一个虚拟 UART 的接收 FIFO。
    
-   (2) 此访问在stage 2 translation时被阻止，导致路由到 EL2 的abort异常向量表 在abort异常向量表的路基中，读取ESREL2获取访问的字节数、目标寄存器以及它是加载还是存储; 读取HPFAREL2获取IPA地址
    
-   (3) hypervisor使用来自 ESREL2 和 HPFAREL2 的信息来识别访问的虚拟外设寄存器。此信息允许hypervisor模拟操作。然后通过 ERET 返回到 vCPU
    

##### 4.6 System Memory Management Units (SMMUs)

到目前为止，我们已经考虑了来自处理器的不同类型的访问。系统中的其他主机，例如 DMA 控制器，可能会分配给 VM 使用。我们还需要一些方法来将stage 1的保护扩展到这些 master。

一个带有不使用虚拟化的 DMA 控制器的系统，如下图所示：![](https://relay-1.bijitongbu.site/p/c908bc9c1a1738866647f65f9a53e325.png)DMA 控制器将通过驱动程序进行编程，通常在内核空间中。该内核空间驱动程序可以确保操作系统级别的内存保护不被破坏。这意味着一个应用程序无法使用 DMA 访问它不应该看到的内存。

我们再看一张有VM的系统框图：![](https://relay-1.bijitongbu.site/p/d7c7b470b7d460c602c8d6853c0df475.png)在这个系统中，hypervisor使用stage 2来提供 VM 之间的隔离。软件查看内存的能力受到hypervisor控制的stage 2的限制。（其实就是说guest os无法为DMA提供真是的Physical Address， guest os看到的物理地址都是IPA）

允许 VM 中的驱动程序直接与 DMA 控制器交互会产生两个问题：

​

-   (1)、**隔离**：DMA 控制器不受stage 2转换的约束，可用于破坏 VM 的沙箱。
    
-   (2)、**地址空间**：通过两个阶段的转换，内核认为的 PA 是 IPA。DMA 控制器仍然需要看到 PA，因此内核和 DMA 控制器看到的内存是不同的。为了克服这个问题，管理程序可以捕获 VM 和 DMA 控制器之间的每一次交互，提供必要的转换。当内存碎片化时，此过程效率低下且有问题。
    

捕获和模拟驱动程序访问的另一种方法是扩展stage 2机制以覆盖其他主机，例如我们的 DMA 控制器。发生这种情况时，这些主设备还需要一个 MMU。这被称为系统内存管理单元（SMMU，有时也称为 IOMMU） --- SMMU就这么诞生了

![](https://relay-1.bijitongbu.site/p/27cecfb87b7b7ea612d2474379450141.png)

管理程序将负责对 SMMU 进行编程，以便上游主机（在我们的示例中为 DMA）看到与分配给它的 VM 相同的内存视图。

这个过程解决了我们发现的两个问题。SMMU 可以强制执行 VM 之间的隔离，确保不能使用外部主节点来破坏沙箱。SMMU 还为 VM 中的软件和分配给 VM 的外部主控提供一致的内存视图。

虚拟化并不是 SMMU 的唯一用例。还有许多其他情况不在本文的范围内。

​

#### 5 Trapping and emulation of instructions

有时，管理程序需要模拟虚拟机 (VM) 内的操作。例如，VM 内的软件可能会尝试配置与电源管理或缓存一致性相关的低级处理器控制。通常，您不想让 VM 直接访问这些控件，因为它们可能会被用来打破隔离或影响系统中的其他 VM。

当执行给定的操作（例如读取寄存器）时，捕获会导致异常。管理程序需要能够在 VM 中捕获操作（例如配置低级别控制的操作）并模拟它们，而不影响其他 VM。

该体系结构包括 trap operations ，供您捕获 VM 中的操作并模拟它们。Trapped后，执行通常被允许的特定操作会导致更高级别的异常。管理程序可以使用这些trapped产生的异常来模拟 VM 中的操作

例如，执行等待中断 (WFI) 指令通常会使 CPU 进入低功耗状态。通过置位 TWI 位，如果 HCR\_EL2.TWI==1，则在 EL0 或 EL1 处执行 WFI 将导致 EL2 异常, 然后进入EL2来模拟这个WFI指令

注意：Trapped不仅仅用于虚拟化。还有 EL3 和 EL1 控制的traps。但是，traps对虚拟化软件特别有用。本文仅讨论通常与虚拟化相关的traps。

在我们的 WFI 示例中，操作系统通常会执行 WFI 作为空闲循环的一部分。对于 VM 中的guest os，管理程序可以捕获此操作并改为调度不同的 vCPU，如下图所示：![](https://relay-1.bijitongbu.site/p/a7b1e4619fcd4f53020df7508e1a1d4a.png)

​

##### 5.1 Presenting virtual values of registers

另一个使用traps的例子是呈现寄存器的虚拟值。例如，IDAA64MMFR0EL1 报告对处理器中与内存系统相关的功能的支持。操作系统可能会在启动过程中读取此寄存器，以确定要启用内核中的哪些功能。管理程序可能希望向客户操作系统提供一个不同的值，称为虚拟值。

为此，管理程序启用覆盖寄存器读取的陷阱。对于trapped异常，管理程序确定触发了哪个trap，然后模拟该操作。在此示例中，管理程序使用 IDAA64MMFR0EL1 的虚拟值填充目标寄存器，如下所示：![](https://relay-1.bijitongbu.site/p/3ecfa0100fcf11a76362802226a76ed0.png)traps也可以用作lazy context switching的一部分。例如，操作系统通常会在启动期间初始化内存管理单元 (MMU) 配置寄存器（TTBREL1、TCREL1 和 MAIR\_EL1），然后不会再次对它们重新编程。管理程序可以使用它来优化其上下文切换例程，方法是仅在上下文切换时恢复寄存器而不保存它们。

然而，操作系统可能会做一些不寻常的事情并在启动后重新编程寄存器。为避免这导致任何问题，管理程序可以设置 `HCR_EL2.TVM` trap。此设置会导致对 MMU 相关寄存器的任何写入都会在 EL2 中生成trap，从而允许管理程序检测它是否需要更新其保存的这些寄存器的副本。

解释：这两段其实就是说，对于系统寄存器，在每次切换的时候会lazy context switching。但是对于MMU寄存器，可能会在运行时被修改，所以针对MMU寄存器的控制，可以设置 `HCR_EL2.TVM`，这样的话在guest os写MMU寄存器的时候，就会产生trap异常）​注: lazy context switching 和 context switching的区别：（1）、前者，开机的时候记录下每个vcpu的context，当vcpu切换到VM时，则恢复这个vContext （2）、后者，每次的VM切换，都伴随着vContext的save和restore

注意：该体系结构使用术语**trapping** 和**routing**来表示独立但又相关的概念。回顾一下，当执行给定的操作（例如读取寄存器）时，trapped会导致异常。routing是指异常生成后将采取的异常级别。

​

##### 5.2 MIDR and MPIDR

使用traps来虚拟化操作需要大量计算。该操作向 EL2 生成trapped异常，管理程序确定所需的操作，对其进行模拟，然后返回给guest os。诸如 IDAA64MMFR0EL1 之类的功能寄存器不被操作系统频繁访问。这意味着当将对这些寄存器的访问捕获到管理程序中以模拟读取时，计算是可以接受的。

对于更频繁访问的寄存器，或在性能关键代码中，你可能希望避免此类计算频繁。这些寄存器及其值的示例包括：

​

-   MIDR\_EL1. The type of processor, for example Cortex-A53
    
-   MPIDR\_EL1. The affinity, for example core 1 of processor 2
    

管理程序可能希望来guest os查看这些寄存器的虚拟值，而不必捕获每个单独的访问。对于这些寄存器，该架构提供了一种捕获的替代方法：

​

-   VPIDREL2. This is the value to return for EL1 reads of MIDREL1.
    
-   VMPIDREL2. This is the value to return for EL1 reads of MPIDREL1
    

管理程序可以在进入 VM 之前设置这些寄存器。如果在 VM 中运行的软件读取 MIDREL1 或 MPIDREL1，硬件将自动返回虚拟值，无需trapped。

注意：VMPIDREL2 和 VPIDREL2 没有定义的复位值。在第一次进入 EL1 之前，它们必须通过启动代码进行初始化。这一点尤其重要。

​

#### 6 Virtualizing exceptions

系统中的硬件使用中断向软件发送事件信号。例如，GPU 可能会发送一个中断来表示它已完成刷完一帧。

使用虚拟化的系统更为复杂。某些中断可能由管理程序本身处理。其他中断可能来自分配给虚拟机 (VM) 的设备，需要由该 VM 内的软件处理。此外，中断所针对的 VM 在收到中断时可能未运行

这意味着需要一些机制来支持管理程序处理 EL2 中的某些中断。还需要将其他中断转发到特定 VM 或 VM 中特定虚拟 CPU (vCPU) 的机制。

为了启用这些机制，该架构包括对虚拟中断的支持：vIRQ、vFIQ 和 vSErrors。这些虚拟中断的行为类似于它们的物理中断（IRQ、FIQ 和 SError），但只能在 EL0 和 EL1 中执行时发出信号。在 EL2 或 EL3 中执行时不可能接收到虚拟中断。

注意：回顾一下，Armv8.4-A 中引入了对安全状态虚拟化的支持。对于要在安全 EL0/1 中发出信号的虚拟中断，需要支持和启用安全 EL2。否则虚拟中断不会在安全状态下发出信号。

​

##### 6.1 Enabling virtual interrupts

要将虚拟中断发送到 EL0/1，管理程序必须在 HCREL2 中设置相应的路由位。例如，要启用 vIRQ 信号，管理程序必须设置 HCREL2.IMO。此设置将物理 IRQ 异常路由到 EL2，并启用向 EL1 发送虚拟异常信号.

虚拟中断按中断类型进行控制。理论上，VM 可以配置为接收物理 FIQ 和虚拟 IRQ。在实践中，这是少见的。VM 通常配置为仅接收虚拟中断。

​

##### 6.2 Generating virtual interrupts

有两种产生虚拟中断的机制：

​

-   （1）、在内核内部，使用HCR\_EL2 中的控件。
    
-   （2）、使用 GICv2 或更高版本的中断控制器。
    

先看第一种。HCR\_EL2 中有三个位控制虚拟中断的生成：

​

-   VI = Setting this bit registers a vIRQ.
    
-   VF = Setting this bit registers a vFIQ.
    
-   VSE = Setting this bit registers a vSError
    

设置这些位之一相当于中断控制器将中断信号**断言**到 vCPU。生成的虚拟中断受制于 PSTATE 屏蔽，就像常规中断一样。

这种机制使用简单，但缺点是它只提供了一种产生中断本身的方式。然而，**管理程序需要模拟 VM 中中断控制器的操作**。回顾一下，软件中的捕获和模拟操作涉及开销，最好避免频繁操作（如中断）。

第二种选择是使用 Arm 的通用中断控制器 (GIC) 来生成虚拟中断。从 Arm GICv2 开始，GIC 可以通过提供物理 CPU 接口和虚拟 CPU 接口来发出物理和虚拟中断信号，如下图所示：![](https://relay-1.bijitongbu.site/p/4f4147a4ebe9836be4340aa3374f312d.png)这两个接口是相同的，除了一个表示物理中断，另一个表示虚拟中断。管理程序可以将虚拟 CPU 接口映射到 VM，允许该 VM 中的软件直接与 GIC 通信。这种方式的好处是hypervisor只需要设置虚拟接口，不需要模拟。这种方法减少了执行需要被困在 EL2 的次数，因此减少了虚拟化中断的开销。

注意：虽然 Arm GICv2 可用于 Armv8-A 设计，但更常见的是使用 GICv3 或 GICv4。

​

##### 6.3 Example of forwarding an interrupt to a vCPU

到目前为止，我们已经了解了如何启用和生成虚拟中断。让我们看一个示例，该示例显示了将虚拟中断转发到 vCPU。在此示例中，我们将考虑已分配给 VM 的物理外围设备，如下图所示：![](https://relay-1.bijitongbu.site/p/c069a8b525076ba4ce3efa74d92657a1.png)上图示例的步骤为：

​

-   (1)、 物理外设产生中断，发送信号到 GIC。
    
-   (2)、 GIC 生成物理中断异常，IRQ 或 FIQ，通过 HCR\_EL2.IMO/FMO 的配置路由到 EL2。管理程序识别外围设备并确定它已分配给 VM。它检查中断应该转发到哪个 vCPU。
    
-   (3)、 hypervisor配置GIC将物理中断作为虚拟中断转发给vCPU。然后 GIC 将置位 vIRQ 或 vFIQ 信号，但处理器在 EL2 中执行时将忽略该信号。
    
-   (4)、 管理程序将控制权交还给 vCPU。
    
-   (5)、 现在处理器在 vCPU（EL0 或 EL1）中，可以从 GIC 获取虚拟中断。此虚拟中断受 PSTATE 异常掩码的约束。
    

该示例显示了作为虚拟中断转发的物理中断。。对于虚拟外设，管理程序可以创建虚拟中断，而无需将其链接到物理中断。

​

##### 6.4 Interrupt masking and virtual interrupts

在异常模型中，我们在 PSTATE 中引入了中断屏蔽位，IRQ 为 PSTATE.I，FIQ 为 PSTATE.F，SError 为 PSTATE.A。在虚拟化环境中运行时，这些掩码的工作方式略有不同。

例如，对于 IRQ，我们已经看到设置 HCR\_EL2.IMO 做了两件事：

​

-   Routes physical IRQs to EL2
    
-   Enables signaling of vIRQs in EL0 and EL1
    

此设置还会更改应用 PSTATE.I 掩码的方式。而在 EL0 和 EL1 中，如果 HCR\_E2.IMO==1，PSTATE.I 对 vIRQ 而非 pIRQ 起作用。

​

#### 7 Virtualizing the Generic Timers

Arm 架构包括通用定时器，它是每个处理器中可用的标准化定时器集。通用定时器由一组比较器组成，用于与公共系统计数进行比较。当比较器的值等于或小于系统计数时，比较器产生中断。在下图中，我们可以看到系统中的通用定时器（橙色），及其比较器和计数器模块的组件。![](https://relay-1.bijitongbu.site/p/c9a14f785714811137842a17077d3b04.png)下图显示了一个示例系统，该系统具有托管两个虚拟 CPU (vCPU) 的管理程序：![](https://relay-1.bijitongbu.site/p/4f2c1d58196d3c7c250dff6d9f081b9f.png)注意：在示例中，我们忽略了运行管理程序以在 vCPU 之间进行上下文切换的开销。

在 4 毫秒的物理时间或wall-clock time,之后，每个 vCPU 已经运行了 2 毫秒。如果 vCPU0 在 T=0 时将其比较器设置为在 3ms 后生成中断，您是否希望中断触发？或者，您是否想要在虚拟时间 2 毫秒（即 vCPU 所经历的时间）或wall-clock time的 2 毫秒之后中断？

Arm 架构提供了两种能力，具体取决于虚拟化的用途。让我们看看它是如何做到的。

在 vCPU 上运行的软件可以访问两个计时器：

​

-   **EL1 Physical Timer** EL1 物理计时器与系统计数器模块生成的计数进行比较。使用此计时器可提wall-clock time
    
-   **EL1 Virtual Timer** EL1 虚拟计时器与虚拟计数进行比较。​虚拟计数是物理计数减去偏移量。​管理程序在寄存器中指定当前调度的 vCPU 的偏移量。​这允许它在未安排 vCPU 运行时隐藏时间的流逝： ![](https://relay-1.bijitongbu.site/p/ff318a16c643a80572c2de32df3baef6.png) 为了说明这个概念，我们可以扩展前面的例子，如下图所示： ![](https://relay-1.bijitongbu.site/p/7d3ccc597baf6a82b9be766ab2001f38.png) 在 6 毫秒的时间内，每个 vCPU 运行 3 毫秒。​虚拟机管理程序可以使用偏移寄存器来呈现仅显示 vCPU 运行时间的虚拟计数。​或者管理程序可以将偏移量保持为 0，这意味着虚拟时间与物理时间相同。​
    

注意：示例显示系统计数的频率为 1ms。实际上，这个频率值是不太可能的。我们建议您将系统计数设置为使用 1MHz 到 50MHz 之间的频率

​

#### 8 Virtualization Host Extensions

下图显示了我们在虚拟化异常部分中查看的软件框图和异常级别的简化版本：

![](https://relay-1.bijitongbu.site/p/2a89bcb4aeda7324e6f71cbf3d8041f0.png)

可以看到独立虚拟机管理程序如何映射到 Arm 异常级别。管理程序在 EL2 上运行，虚拟机 (VM) 在 EL0/1 上运行。

注意：DynamIQ 处理器（Cortex-A55、Cortex-A75 和 Cortex-A76）支持虚拟化主机扩展 (VHE)。 事实上armv8.1 都支持VHE

​

##### 8.1 Running the Host OS at EL2

VHE 由 HCR\_EL2 中的两位控制。这些位可以总结为：

​

-   E2H：控制是否启用VHE。
    
-   TGE：启用VHE 时，控制EL0 是Guest 还是Host
    

下表总结了典型设置：

| **Executing in:** | **E2H** | **TGE** |
| --- | --- | --- |
| **Guest kernel (EL1)** | 1 | 0 |
| **Guest application (EL0)** | 1 | 0 |
| **Host kernel (EL2)** | 1 | 1\* |
| **Host application (EL0)** | 1 | 1 |

（星号\*）对于从 VM 退出到管理程序的异常，TGE 最初将为 0。软件将在运行主机内核的主要部分之前必须设置该位。

可以在下图中看到这些典型设置：![](https://relay-1.bijitongbu.site/p/282c6768cf96af280b7861ed284ccc20.png)

​

##### 8.2 Virtual address space

下图显示了在引入 VHE 之前 EL0/EL1 的虚拟地址空间的样子：![](https://relay-1.bijitongbu.site/p/56e2d42702d27fcd288a4493084565f7.png)正如内存管理中所讨论的，EL0/1 有两个区域。按照惯例，上部区域称为内核空间，下部区域称为用户空间。但是，EL2 在地址范围的底部只有一个区域。这种差异是因为，传统上，管理程序不会托管应用程序。这意味着管理程序不需要在内核空间和用户空间之间进行拆分。

注意：内核空间分配给上层区域，用户空间分配给下层区域，只是一个约定。它不是 Arm 架构强制要求的。

EL0/1 虚拟地址空间也支持地址空间标识符 (ASID)，但 EL2 不支持。这是因为管理程序通常不会托管应用程序。

为了让我们的主机操作系统在 EL2 中高效执行，我们需要添加第二个区域和 ASID 支持。设置 HCREL2.E2H 解决了这些问题，如下图所示：![](https://relay-1.bijitongbu.site/p/77fd65e95d7c2757273dc821e1031114.png)而在 EL0 中，HCREL2.TGE 控制使用哪个虚拟地址空间：EL1 空间或 EL2 空间。使用哪个空间取决于应用程序是在 Host OS (TGE==1) 还是 Guest OS (TGE==0) 下运行。

​

##### 8.3 Re-directing register accesses （重新定位寄存器访问）

我们在虚拟化通用定时器部分看到启用 VHE 会改变 EL2 虚拟地址空间的布局。但是，我们仍然有 MMU 的配置问题。这是因为我们的内核会尝试访问xxxEL1 寄存器，如 TTBR0EL1，而不是 xxxEL2 寄存器，如 TTBR0EL2。

要在 EL2 上运行相同的操作，我们需要将访问从 EL1 寄存器重定向到 EL2 等效项。设置 E2H 将执行此操作，以便对 xxxEL1 系统寄存器的访问被重定向到它们的 EL2 等效项。此重定向如下图所示：![](https://relay-1.bijitongbu.site/p/74e0f37fe35c0be8abddbec18d39d300.png)然而，这种重定向给我们留下了一个新问题。管理程序仍然需要访问真正的EL1 寄存器，以便它可以实现任务切换。为了解决这个问题，引入了一组带有 xxxEL12 或 xxxEL02 后缀的新寄存器别名。当在 EL2 上使用时，E2H==1，它们可以访问 EL1 寄存器以进行上下文切换。您可以在下图中看到这一点：![](https://relay-1.bijitongbu.site/p/340abc6ff43725213c531e7e6da4f373.png)

​

##### 8.4 Exceptions

通常，HCREL2.IMO/FMO/AMO 位控制物理异常是路由到 EL1 还是 EL2。在 TGE ==1 的 EL0 中执行时，所有物理异常都被路由到 EL2，除非它们被 SCREL3 路由到 EL3。无论 HCR\_EL2 路由位的实际值如何，情况都是如此。这是因为应用程序作为主机操作系统的子操作系统执行，而不是客户操作系统。因此，任何异常都应路由到在 EL2 中运行的主机操作系统

​

#### 9 Nested virtualization (嵌套虚拟化)

理论上，管理程序可以在虚拟机 (VM) 中运行。这个概念叫做嵌套虚拟化：![](https://relay-1.bijitongbu.site/p/8bbf45e0bac80ea6423ae4bfa1f1a0cc.png)

​

#### 10 Secure virtualization

虚拟化是在 Armv7-A 中引入的。当时Hyp模式，相当于AArch32中的EL2，只在Non-secure状态下可用。引入 Armv8.4-A 时，添加了对处于安全状态的 EL2 的支持作为可选功能。

当处理器支持安全 EL2 时，需要使用 SCR\_EL3.EEL2 位从 EL3 启用处理器。设置该位允许进入 EL2，并允许在安全状态下使用虚拟化功能。

在安全虚拟化可用之前，EL3 通常用于托管安全状态切换软件和平台固件的混合体。这是因为我们喜欢尽量减少 EL3 中的软件量，让 EL3 更容易安全。安全虚拟化允许我们将平台固件移动到 EL1。虚拟化为平台固件和可信内核提供单独的安全分区。下图说明了这一点：![](https://relay-1.bijitongbu.site/p/5636749751dda8103a70e619591ff983.png)

​

##### 10.1 Secure EL2 and the two Intermediate Physical Address spaces

Arm 架构定义了两个物理地址空间：安全和非安全。在非安全状态下，虚拟机 (VM) 的stage 1转换的输出始终是非安全的。因此，stage 1需要处理单个中间物理地址 (IPA) 空间。

在安全状态下，VM 的stage 1转换可以输出安全和非安全地址。转换表描述符中的 NS 位控制是输出安全地址空间还是非安全地址空间。如下图所示，这意味着stage 2有两个 IPA 空间，安全和非安全：![](https://relay-1.bijitongbu.site/p/4f43d1ee56650c98d53538159899535e.png)与stage 1不同，stage 2条目中没有 NS 位。对于特定的 IPA 空间，所有转换都会产生安全物理地址或非安全物理地址。该转换由寄存器位控制。通常，非安全 IPA 转换为非安全 PA，而安全 IPA 转换为安全 PA。

​

-   如需进群可加我微信邀请进群交流：sami01\_2023
    

![](https://relay-1.bijitongbu.site/p/5e96b9f771ee8348f2759423910f2faa.png)

  

---

![cover_image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/72OMRpZ5hV0BEYwibnxJTuzC2OVYgDMVQf7n7c0bEUj4vtLmgYDfdEBicq0e3FImufibZrsXWM8gtA4CefcibeZc8w/0?wx_fmt=jpeg)

Original baron Arm精选

Read more

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/6fc92f6a_1779842357379?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU4NDg4MzY3OA%3D%3D%26mid%3D2247515096%26idx%3D1%26sn%3D673229755dedae52a3a7bfcd95378459%26chksm%3Dfcf9f95b47dabae3452ec9c8d4f48e6ed430c8db7f304dad5a43a977063093f840a24d671cd4%26mpshare%3D1%26scene%3D1%26srcid%3D05279648psLyxeT8iLhjyZtw%26sharer_shareinfo%3D18890fc195b3ee7d05f22f261e744816%26sharer_shareinfo_first%3D18890fc195b3ee7d05f22f261e744816%23rd&s=obsidian)