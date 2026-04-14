# 基础知识

GIC控制器对来自整个系统的所有中断进行编组，确定它们的优先级，并将它们发送到一个核心进行处理。 GIC 主要用于提高处理器效率并启用中断虚拟化。GIC 基于 Arm GIC 架构实现。该架构已从 GICv1 发展到最新版本的 GICv3 和 GICv4。

GIC中断控制器架构的分类：gicv1（已弃用）、gicv2、gicv3、gicv4；

ARM公司中断控制器IP对应架构：
* gic400：支持gicv2架构；
* gic500：支持gicv3架构；
* gic600：支持gicv3架构；
* gic700：支持gicv4架构；

gic的核心功能是对soc中外设的中断源进行管理，并提供给软件，配置以及控制这些中断源。

* 当对应的中断源有效时，gic根据该中断源的配置，决定是否将该中断信号发送给CPU。如果存在多个中断源有效，gic将会进行仲裁，选择高优先级的中断发送给CPU。
* 当CPU接收到gic发送的中断，通过读取gic的寄存器，就可以知道中断来自于哪里，从而可以做出相应的处理。
* 当CPU处理完中断后，会告诉gic（访问gic的寄存器）该中断处理完毕。gic接收到CPU处理完中断后，就将该中断源取消，避免又重新发送中断给CPU及允许中断抢占。

## ARM Core访问gic的方式

ARM Core访问gic寄存器的方式有两种：
1. 使用memory-mapped的方式：SoC在设计时预留一块内存区域给gic，ARM Core通过读写该地址来进行gic寄存器的操作；
2. 通过系统寄存器的方式：通过MSR/MRS（armv8-arch32使用MCR/MRC）来进行读写gic寄存器；

ARM Core访问各版本gic寄存器的方式：
* ARM Core访问gicv2所有寄存器（distributor、cpu interface）都是使用memmory-mapped的方式进行访问；
* <mark style="background: #D2B3FFA6;">ARM Core访问gicv3的distributor/redistributor使用memory-mapped方式进行访问；</mark>
* <mark style="background: #D2B3FFA6;">gicv3的ITS/CPU interface既可以使用memory-mapped方式访问，也可以使用系统寄存器方式访问；</mark>



# gic架构
## gicv2




## gicv3




## gicv4
