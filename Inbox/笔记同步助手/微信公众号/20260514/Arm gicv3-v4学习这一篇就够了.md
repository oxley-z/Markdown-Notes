---
author: baron
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247515086&idx=1&sn=edbf705d86344c82f9440dd3b72230ce&chksm=fc707160d146a45684efc94ab51f6ee2fb8f10dcdc248863e7741d5c2383c8acd608d3f1e8cc&mpshare=1&scene=1&srcid=0514jBtYVU5kwOBDDARCpFmx&sharer_shareinfo=4b133b939040b8f0a69cf1f4fb4ef74c&sharer_shareinfo_first=4b133b939040b8f0a69cf1f4fb4ef74c#rd
saved: 2026-05-14 08:38:12
tags:
  - 笔记同步助手
id: f09bd348-6fa6-4d3f-bf09-94079a32a137
---

公众号名称：Arm精选

作者名称：baron

发布时间：2026-05-14 08:07

#### 1 简介

本文概述了 `ArmCoreLinkGenericInterruptController(GIC)v3andv4`的功能。 描述了 GICv3 兼容中断控制器的操9作。 它还描述了如何配置 GICv3 中断控制器以在裸机环境中使用。

中断是给处理器的信号，告诉处理器发生了需要处理的事件。 中断通常由外围设备生成。

例如，系统可能使用通用异步接收器/发送器 (UART) 接口与外部世界进行通信。 当 UART 接收到数据时，它需要一种机制来告诉处理器新数据已经到达并准备好进行处理。 UART 可以使用的一种机制是生成中断以向处理器发送信号。

小型系统可能只有几个中断源和一个处理器。 然而，更大的系统可能有更多潜在的中断源和处理器。 GIC 执行中断管理、优先级和路由等关键任务。 GIC 对来自整个系统的所有中断进行编组，对它们进行优先级排序，并将它们发送到要处理的核心。 GIC 主要用于提高处理器效率和启用中断虚拟化。

GIC 是基于 Arm GIC 架构实现的。 这种架构已经从 GICv1 发展到最新版本的 GICv3 和 GICv4。 Arm 拥有多个通用中断控制器，可为所有类型的 Arm Cortex 多处理器系统提供一系列中断管理解决方案。 这些控制器的范围从用于 CPU 内核数较少的系统的最简单的 GIC-400 到用于高性能和多芯片系统的 GIC-600。 GIC-600AE 针对高性能 ASIL B 到 ASIL D 系统增加了额外的安全功能。

​

#### 2 gic的版本号

-   gic400，支持gicv2架构版本。
    
-   gic500，支持gicv3架构版本。
    
-   gic600，支持gicv3架构版本
    
-   gic700, 支持gicv4.1架构版本 ![](https://relay-1.bijitongbu.site/p/5496664ff127236370d93d44b3ccb0a3.png) 本文涵盖了大多数 Armv8-A 和 Armv8-R 设计使用的 Arm CoreLink GICv3 和 GICv4。
    

Arm CoreLink GICv3 和 GICv4 自发布以来也进行过小更新：

​

-   GICv3.1 添加了对额外有线中断、安全虚拟化和内存系统资源分区和监控 (MPAM) 的支持
    
-   GICv4.1 扩展虚拟化支持以涵盖虚拟软件生成中断 (SGI) 的直接注入
    

![](https://relay-1.bijitongbu.site/p/4d430bd5323ef9627542c4fdcc9ea003.png)

#### 3 gic模型

通用中断控制器 (GIC ：Generic Interrupt Controller) 接收来自外设的中断，对它们进行优先级排序，然后将它们传送到适当的处理器内核。 下图显示了一个 GIC 从 n 个不同的外设获取中断，并将它们分配给两个不同的处理器。![](https://relay-1.bijitongbu.site/p/d3700926921b7a7102be34c3529da391.png)GIC 是 Arm Cortex-A 和 Arm Cortex-R 配置文件处理器的标准中断控制器。 GIC 提供了一种灵活且可扩展的中断管理方法，支持具有单核的系统到具有数百个核的大型多芯片设计。

​

#### 4 gic的基础知识

在本节中，我们将了解 Arm CoreLink GICv3 和 v4 中断控制器的基本操作

​

##### 4.1. 中断类型

GIC 分为不同类型的中断源：

​

-   Shared Peripheral Interrupt (SPI) ： 共享中断
    
-   Private Peripheral Interrupt (PPI) ： 私有中断
    
-   Software Generated Interrupt (SGI) ： 软件产生中断
    
-   Locality-specific Peripheral Interrupt (LPI)
    

每个中断源都由一个 ID 号标识，称为 INTID。 前面列表中介绍的中断类型就是根据 INTID 的范围定义的：![](https://relay-1.bijitongbu.site/p/18e761e6317190b9dedc44db844db889.png)

​

###### 4.1.1 外设给gic的中断信号

传统上，中断是使用专用硬件信号从外设发送到中断控制器的，如下图所示：![](https://relay-1.bijitongbu.site/p/394460c227e41a1c8561f715cfe495ce.png)GICv3 支持这种模型，另外还支持基于消息的中断。 基于消息的中断 message-based interrupt是通过写入中断控制器中的寄存器来设置和清除的中断。![](https://relay-1.bijitongbu.site/p/5166ab878e5bcb3ae7d7cc55297233a4.png)使用消息将中断从外设转发到中断控制器消除了对每个中断源的专用信号的要求。 这对于大型系统的设计人员来说可能是一个优势，其中可能有数百甚至数千个信号可能通过 SoC 路由并汇聚到中断控制器上。

中断是作为消息发送还是使用专用信号对中断处理代码处理中断的方式几乎没有影响。 可能需要对外围设备进行一些配置。 例如，可能需要指定中断控制器的地址。

在 Arm CoreLink GICv3 中，SPI 可以是消息信号中断。 LPI 始终是消息信号中断。 不同的寄存器用于不同的中断类型，如下表所示（ **Message-based interrupt registers**）：![](https://relay-1.bijitongbu.site/p/16330f236c8ff135aa5247f6b524bf3c.png)

​

##### 4.2. 中断的状态

中断控制器为每个 SPI、PPI 和 SGI 中断源维护一个状态机。 这个状态机由四个状态组成：![](https://relay-1.bijitongbu.site/p/a71d0c20ec2e0d3fb2b1594ba4894761.png)

​

-   **Inactive** The interrupt source is not currently asserted.
    
-   **Pending** The interrupt source has been asserted, but the interrupt has not yet been acknowledged by a PE.
    
-   **Active** The interrupt source has been asserted, and the interrupt has been acknowledged by a PE.
    
-   **Active and Pending** An instance of the interrupt has been acknowledged, and another instance is now pending.
    

(注：如果是LPI，则 没有Active或Active and Pending状态)

中断的生命周期取决于它是否配置为电平触发( level-sensitive )或边沿触发(edge-triggered)：

​

-   对于电平触中断，中断输入的上升沿导致中断变为挂起状态，并且中断保持有效，直到外设取消中断信号。
    
-   对于边沿触发中断，中断输入的上升沿会导致中断变为挂起状态，但中断不会保持有效。
    

###### 4.2.1 点评中断

如下展示了电平触发中断的信号的示例：![](https://relay-1.bijitongbu.site/p/aad2a70ff45593f5a097613b7471abed.png)

​

###### 4.2.2 边沿中断

如下展示了边沿触发中断的信号的示例：![](https://relay-1.bijitongbu.site/p/dacfb14cf27a9719ebad68b041a0891e.png)

​

##### 4.3. Target interrupts ：中断affinity

Arm 架构为每个 PE 分配一个层次标识符，称为affinity。 GIC 使用affinity来标记特定内核的中断。

affinity是一个 32 位值，分为四个字段： `...`

**PE的affinity值是从MPIDREL1读取的。**不同级别关联的affinity含义由特定处理器和 SoC 定义。 例如，Arm Cortex-A53 和 Arm Cortex-A57 处理器使用： `...`后来的设计，如 Arm Cortex-A55 和 Arm Cortex-A76 处理器中使用的设计，使用： `...所有可能的节点都存在于单个实现中的可能性很小。 例如，用于移动设备的 SoC 可能具有如下布局：`

```
0.0.0.[0:3]Cores0to3of aCortex-A53 processor
0.0.1.[0:1]Cores0to1of aCortex-A57 processor
```

``注意：AArch32 状态和 Armv7-A 只能支持三个级别的关联。 这意味着使用 AArch32 状态的设计仅限于关联级别 3 (0.x.y.z) 的单个节点。 GICDTYPER.A3V 表示中断控制器是否可以支持多个三级节点。  ##### 4.4. 中断的安全模型  Arm CoreLink GICv3 架构支持 Arm TrustZone 技术。 必须通过软件为每个 INTID 分配一个组和安全设置。 GICv3 支持三种设置组合，如下表所示：![](https://relay-1.bijitongbu.site/p/047f240696647f1c5311b3d1f2471cd0.png)group 0 中断始终作为 FIQ 发出信号。 group 1中断以 IRQ 或 FIQ 的形式发出信号，具体取决于 PE 的当前安全状态和异常级别，如下所示：![](https://relay-1.bijitongbu.site/p/10ea528d3f5bb19b0c33ff3912bf5b76.png)这些规则旨在补充 AArch64 安全状态和异常级别的路由控制。 下图显示了一个简化的软件框图，以及在 EL0 处执行时发出不同类型中断信号时是怎样处理的：![](https://relay-1.bijitongbu.site/p/bf2b98c0cb11bb1a9dbb53ac2ef9b969.png)在这个例子中，IRQ 被路由到 EL1 (SCREL3.IRQ==0) 而 FIQ 被路由到 EL3 (SCREL3.FIQ==1) 。 考虑到上述规则，在 EL1 或 EL0 执行时，当前安全状态的组 1 中断被视为 IRQ。  其他安全状态的中断触发 FIQ，并将异常处理到 EL3。 这允许在 EL3 执行的软件执行必要的上下文切换。  ​  ###### 4.4.1 中断分组的配置  在配置中断控制器时，软件控制 INTID 到中断组的分配。 只有在安全状态下执行的软件才能将 INTID 分配给中断组。 通常，只有在安全状态下执行的软件才能访问安全中断的设置和状态：group 0 和secure group 1。  可以启用从非安全状态到安全中断设置和状态的访问。 这使用 `GICD_NSACRn` 和 `GICR_NSACR` 寄存器为每个 INTID 单独控制。  注意：LPI 始终被视为non-secure group 1中断。  ​  ###### 4.4.2 gic安全状态的配置  GICv3 支持 Arm TrustZone 技术，但 TrustZone 的使用是可选的。 这意味着您可以将实现配置为具有单个安全状态或两个安全状态：  ​  -   GICD_CTLR.DS == 0 Two Security states, Secure and Non-secure, are supported.      -   GICD_CTLR.DS == 1 Only a single Security state is supported.       将 GIC 配置为使用与连接的 PE 相同数量的安全状态。 通常，这意味着 GIC 在连接到 Arm Cortex-A 配置文件处理器时将支持两种安全状态，当连接到 Arm Cortex-R 处理器时将支持一种安全状态。  ​  ##### 4.5. gic的组件和寄存器介绍  GICv3 中断控制器的寄存器接口分为三类：  ​  -   Distributor interface      -   Redistributor interface      -   CPU interface       下面是一个GICV3模块的结构框图：![](https://relay-1.bijitongbu.site/p/1835df7c99be6d7cff01d325ead0f8bb.png)一般情况下，Distributor 和 Redistributor 用于配置中断，CPU 接口用于处理中断。  ​  ###### 4.5.1 Distributor (GICD_*) 介绍  Distributor registers是memory-mapped，用于SPI中断。它的作用主要有：  ​  -   SPI的中断优先级和路由      -   启用和禁用 SPI      -   设置每个SPI的优先级      -   每个SPI的路由信息      -   将每个 SPI 设置为电平敏感或边沿触发      -   生成message-signaled SPI      -   控制 SPI 的active 和pending 状态      -   确定每个安全状态中使用的模型：affinity routing或 legacy       ###### 4.5.2 Redistributors (GICR_*)介绍  一个Redistributor per connected连接一个 core，它的作用主要有：  ​  -   启用和禁用 SGI 和 PPI      -   设置 SGI 和 PPI 的优先级      -   将每个 PPI 设置为电平敏感或边沿触发      -   将每个 SGI 和 PPI 分配给一个中断组      -   控制 SGI 和 PPI 的状态      -   控制内存中支持相关中断属性和 LPI 挂起状态的数据结构的基地址      -   为连接的PE提供电源管理支持       ###### 4.5.3 CPU interfaces (ICC*ELn)介绍  每一个Core都有一个cpu interface，它是通过memory-mapped或系统寄存器的方式来方式，它的作用为：  ​  -   提供通用控制和配置以启用中断处理      -   确认中断：Acknowledge an interrupt      -   执行优先级降低和中断停用： priority drop 和deactivation      -   为 PE 设置中断优先级掩码      -   定义PE的抢占策略      -   确定 PE 的pending中断的最高优先级       在 Arm CoreLink GICv3 中，CPU 接口寄存器作为系统寄存器访问：ICC*ELn。  如果要通过系统寄存器的方式来访问，那么必现enable 系统寄存器访问的这个开关，通过 `ICC_SRE_ELn`寄存器来控制。  ​  ###### 4.5.4 GICD寄存器  ![](https://relay-1.bijitongbu.site/p/3f34db6c1df62c33c324856dbbd4d76f.png)![](https://relay-1.bijitongbu.site/p/93f5b87a61f66f39787e2a192ca6fd2d.png)  ​  ###### 4.5.5 GICR寄存器  ![](https://relay-1.bijitongbu.site/p/965047964575076c6d304a4389c098d7.png)![](https://relay-1.bijitongbu.site/p/50344a9b209c1e9bc379a4e822bf9b34.png)![](https://relay-1.bijitongbu.site/p/f8aa32fbf3826c1daf405459bcb0efbe.png)  ​  ###### 4.5.6 GICR寄存器– SGI/PPI  ![](https://relay-1.bijitongbu.site/p/bb49e5b2894365f70d6bd295bf5cdbe9.png)  ###### 4.5.7 cpu interface寄存器  ![](https://relay-1.bijitongbu.site/p/0bb24f25585206e518551859e12a33bc.png)  #### 5 gic寄存器的配置  对于Locality-specific Peripheral Interrupts (LPIs)中断，它和Shared Peripheral Interrupts (SPIs)、Private Peripheral Interrupt (PPIs)、Software Generated Interrupts (SGIs)有太多的不同的地方，基本也用不到，本文就不详细介绍了。  大多数使用 GICv3 中断控制器的系统都是多核系统，也可能是多处理器系统。 有些设置是全局的，这意味着会影响所有连接的 PE。 其他设置特定于单个 PE。  然后我们看看全局设置，然后是每个 PE 的设置。  ​  ##### 5.1. 针对全局的配置  `GICD_CTLR` 必须配置为启用中断组并设置路由模式如下：  ​  -   Enable Affinity routing (ARE bits): GICD_CTLR 中的 ARE 位控制 GIC 是在 GICv3 模式还是legacy模式下运行。 legacy模式提供与 GICv2 兼容。 本文假设 ARE 位设置为 1，以便使用 GICv3 模式。      -   Enables: GICD_CTLR 包group 0、secure group 1 和non-secure group 的单独启用位： (1)、EnableGrp1S ： enables distribution of Secure Group 1 interrupts. (2)、EnableGrp1NS ： enables distribution of Non-secure Group 1 interrupts. (3)、EnableGrp0 ： enables distribution of Group 0 interrupts.       注意：Arm CoreLink GIC-600 不支持legacy操作，并且 ARE 位永久设置为 1。  ​  ##### 5.2. 针对每个PE的配置  下面再来看针对单个PE的配置  ​  ###### 5.2.1 Redistributor配置  每一个Core都对应一个Redistributor![](https://relay-1.bijitongbu.site/p/1a460efdc1ee754eccc489020bde537b.png)Redistributor 包含一个名为 `GICR_WAKER` 的寄存器，用于记录连接的 PE 是在线还是离线。 中断仅转发给 GIC 认为在线的 PE。 复位时，所有 PE 都被视为离线。  要将连接的 PE 标记为在线，软件必须：  ​  -   Clear GICR_WAKER.ProcessorSleep to 0.      -   Poll GICR_WAKER.ChildrenAsleep until it reads 0.       软件在配置 CPU 接口之前执行这些步骤很重要，否则行为可能是不可预测的。  当 PE 离线 (GICR_WAKER.ProcessorSleep==1) 时，针对 PE 的中断将导致唤醒请求信号被断言。 通常，该信号将进入系统的电源控制器。 然后电源控制器打开 PE。 唤醒时，该 PE 上的软件将清除 ProcessorSleep 位，允许转发唤醒 PE 的中断。  ​  ###### 5.2.2 CPU interface配置  CPU 接口负责向它所连接的 PE 传递中断异常。 要启用 CPU 接口，软件必须配置以下内容：  (1)、**Enable System register access**通过设置 `ICC_SRE_ELn.SRE` 就可以enable系统寄存器访问的方式. 注意：许多最新的 Arm Cortex 处理器不支持legacy操作，并且 SRE 位固定为1。 在这些处理器上，可以跳过此步骤。  (2)、**Set Priority Mask and Binary Point registers**CPU 接口包含优先屏蔽寄存器 (ICCPMREL1) 和Binary Point registers (ICCBPRnEL1)。 优先级掩码设置中断必须具有的最低优先级才能转发到 PE。 Binary Point registers用于优先分组和抢占。  (3)、**Set EOI mode**CPU 接口中 ICCCTLREL1 和 ICCCTLREL3 中的 EOI mode 位控制如何处理中断的完成。  (4)、**Enable signaling of each interrupt group**每个中断组必须先使能，然后该组的中断才会通过 CPU interface转发到 PE。 要使能中断组，软件必须针对 Group 1 中断写入 ICCIGRPEN1EL1 寄存器，针对 Group 0 中断写入 ICCIGRPEN0EL1 寄存器。 ICCIGRPEN1EL1 是根据安全状态banked，这意味着 ICCGRPEN1EL1 控制当前安全状态的group 1。 在 EL3，软件可以使用 ICCIGRPEN1EL3 访问Group 1 使能寄存器。  ​  ###### 5.2.3 PE配置  -   Routing controls - SCREL3 、 HCREL2      -   Interrupt masks - PSTATE      -   Vector table - VBAR_ELn       ![](https://relay-1.bijitongbu.site/p/7c861770dbe11c84f63806cacb3b47ac.png)  ###### 5.2.4 SPI, PPI and SGI的配置  -   SPIs are configured through the Distributor, using the GICD_* registers.      -   PPIs and SGIs are configured through the individual Redistributors, using the GICR_* registers       对于每一个中断，软件必需配置的：  ​  -   Priority: GICDIPRIORITYn, GICRIPRIORITYn      -   Group: GICDIGROUPn, GICDIGRPMODn, GICRIGROUPn, GICRIGRPMODn      -   Edge-triggered or level-sensitive: GICDICFGRn, GICRICFGRn      -   Enable: GICDISENABLERn, GICDICENABLER, GICRISENABLERn, GICRICENABLERn       ![](https://relay-1.bijitongbu.site/p/7a6f3120009beb9028d9d11b866d523d.png)  ###### 5.2.5 Arm CoreLink GICv3.1 and the extended INTID ranges  ###### 5.2.6 Setting the target PE for SPIs  #### 6 中断处理  ##### 6.1. Routing a pending interrupt to a PE  -   Check that the group associated with the interrupt is enabled      -   Check that the interrupt is enabled      -   Check the routing controls to decide which PEs can receive the interrupt. routing is controlled by GICD_IROUTERn，An SPI can target one specific PE, or any one of the connected PEs      -   Check the interrupt priority and priority mask to decide which PEs are suitable to handle the interrupt Each PE has a Priority Mask register, ICCPMREL1, in its CPU interface      -   Check the running priority to decide which PEs are available to handle the interrup Only an interrupt with a higher priority than the running priority can preempt the current interrupt       ##### 6.2. Taking an interrupt  ![](https://relay-1.bijitongbu.site/p/0d97db78f2bb85ff8e1803355665c7b2.png)![](https://relay-1.bijitongbu.site/p/1272a57bf1ca72fa556d218f707e37ce.png)  ​  ###### 6.2.1 Example of interrupt handling  ##### 6.3. 运行优先级和抢占：Running priority and preemption  ![](https://relay-1.bijitongbu.site/p/4f5518355b089a6a9ef07c12fbf02c8b.png)  ##### 6.4. 结束一个中断  -   Priority drop - 将中断优先级降到中断产生之前的值      -   Deactivation - 将中断从active变成inactive       在gicv3中，drop和deactivation通常是一起打开的。  ICCCTLRELn.EOImode = 1: 通过写ICCEOIR0EL1、ICCEOIR1EL1让drop and deactivation同时生效 ICCCTLRELn.EOImode = 0: 通过写ICCEOIR0EL1、ICCEOIR1EL1让drop生效，写ICCDIREL1让deactivation生效，这在虚拟化中会用到.  大多数的软件系统中 EOIMode==0，而下hypervisor的系统中 EOIMode==1  ​  ##### 6.5. Checking the current state of the system  ###### 6.5.1 Highest priority pending interrupt and running priority  ###### 6.5.2 State of individual INTIDs  ![](https://relay-1.bijitongbu.site/p/98826b3744e5d17930d91de37d8e14c2.png)  #### 7 Sending and receiving Software Generated Interrupts  ##### 7.1. Generating SGIs  ![](https://relay-1.bijitongbu.site/p/b64a0f2e91326762a8dd88d0e14aa9ef.png)  -   PE在secure执行时，可以产生secure和non-secure的SGI;      -   PE在non-secure执行时，也是可以产生secure的SGI，但是取决于GICR_NSACR寄存器的配置，该寄存器只能在secure中读写 ![](https://relay-1.bijitongbu.site/p/ebd3bd74c7452c1ee5c72192d5c375f4.png)        ##### 7.2. 对比GICv3和GICv2  在gicv2中，SGI INTIDs对于originating PE和the target PE是banked 在gicv3中，SGI仅仅对target PE是banked  在gicv2中同时收到两个SGI=5中断，两个中断都会被PE处理。 而在gicv3上，由于originating不是banked，所有前一个SGI=5中断将会丢失。PE只能收到一个![](https://relay-1.bijitongbu.site/p/d75bc65e9980a85018db051269c52976.png)  ​  #### 8 Appendix: Legacy operation  -   When ARE==0, affinity routing is disabled (legacy operation)      -   When ARE==1, affinity routing is enabled (GICv3 operation) ![](https://relay-1.bijitongbu.site/p/6b0c2d750a79ac2abe85403931c7fa0e.png)       -   ![](https://relay-1.bijitongbu.site/p/5e96b9f771ee8348f2759423910f2faa.png)       -   如需进群可加我微信邀请进群交流：sami01_2023         ``

---

![cover_image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/72OMRpZ5hV2RHqpHLO3e1wJL5Nrjt7fxAzy8mmMJdicoZ0JH9Qm0xNtxJx0j8nuMEK5BLlXyWN1d66oRT6OkicxQ/0?wx_fmt=jpeg)

原创 baron Arm精选

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/2f85d670_1778719091030?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU4NDg4MzY3OA%3D%3D%26mid%3D2247515086%26idx%3D1%26sn%3Dedbf705d86344c82f9440dd3b72230ce%26chksm%3Dfc707160d146a45684efc94ab51f6ee2fb8f10dcdc248863e7741d5c2383c8acd608d3f1e8cc%26mpshare%3D1%26scene%3D1%26srcid%3D0514jBtYVU5kwOBDDARCpFmx%26sharer_shareinfo%3D4b133b939040b8f0a69cf1f4fb4ef74c%26sharer_shareinfo_first%3D4b133b939040b8f0a69cf1f4fb4ef74c%23rd&s=obsidian)