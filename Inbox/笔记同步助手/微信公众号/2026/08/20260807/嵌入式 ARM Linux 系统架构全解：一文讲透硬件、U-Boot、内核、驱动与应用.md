---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247491452&idx=1&sn=d059cfc6bc115a4accc95f02cd3260aa&chksm=c33bd555ab74fd850abc0666838945480d85ee09ef82e645a41cea3f138614ef775952592ca2&mpshare=1&scene=1&srcid=08079j9xSd3iv7fesgZBXWej&sharer_shareinfo=fa6824a5ab35ce130b2f7c8b951543a0&sharer_shareinfo_first=fa6824a5ab35ce130b2f7c8b951543a0#rd
saved: 2026-08-07 22:25:17
tags:
  - 笔记同步助手
id: c94aae0a-683f-4ebf-9e18-559d44a3d00a
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-08-07 20:31

大家好，我是蟹老板～

在嵌入式系统领域之中，**ARM Linux 已经成为极为重要的一套软件以及硬件技术体系**。我们平时接触到的智能终端、工业控制器、车载娱乐设备、机器人、网络通信设备、物联网网关以及各种各样的边缘计算设备，很多都是建立在 ARM 处理器与 Linux 操作系统之上的。

**硬件给所有软件提供运行的物理基础，Bootloader 负责把 Linux 内核送上运行轨道，设备树向内核描述开发板上的硬件，Linux 内核负责管理 CPU、内存和外设，根文件系统提供用户空间运行环境，应用程序则在这些基础之上完成产品真正的业务功能。**

因此，学习嵌入式 Linux 不能只盯着某一个驱动函数，也不能只会编译应用程序。**要想真正具备独立开发以及排查问题的能力，就必须得建立一种完整的系统思维，把硬件、Bootloader、内核、设备树、驱动、根文件系统以及应用程序连接到一起。**

## 为什么要从整体视角理解 ARM Linux

**嵌入式 ARM Linux 并不是一个单独的软件，也不是一份能够直接烧录到设备中的普通程序。它是一整套由多个层次紧密配合起来的系统。**

一个典型的 ARM Linux 系统可以划分成为下面几个层次：

```
┌────────────────────────────────────┐
│        业务应用与后台服务           │
├────────────────────────────────────┤
│   C/C++ 运行库、BusyBox、系统服务   │
├────────────────────────────────────┤
│            根文件系统               │
├────────────────────────────────────┤
│ Linux 内核、子系统与设备驱动        │
├────────────────────────────────────┤
│      Device Tree 硬件描述           │
├────────────────────────────────────┤
│       U-Boot / SPL / 固件           │
├────────────────────────────────────┤
│ Boot ROM、SoC、DDR、Flash、外设     │
└────────────────────────────────────┘
```

设备从上电到运行应用程序，大致会经历下面的流程：

```
设备上电
   ↓
电源和时钟稳定
   ↓
CPU 解除复位
   ↓
执行芯片内部 Boot ROM
   ↓
加载 SPL
   ↓
初始化 DDR
   ↓
加载完整 U-Boot
   ↓
加载 Linux 内核与设备树
   ↓
Linux 内核初始化
   ↓
加载设备驱动
   ↓
挂载根文件系统
   ↓
启动 PID 1
   ↓
启动系统服务
   ↓
运行产品应用程序
```

**其中任何一个环节出现错误，最终都可能表现成为“系统没有正常启动”。**

但是不同阶段的问题，其排查方式完全不一样。例如**串口完全没有输出，可能是电源、时钟、启动模式或者 Boot ROM 阶段的问题；能够进入 U-Boot 但是内核没有输出，可能是内核地址、设备树或者串口参数错误；内核能够运行但是提示无法挂载根文件系统，则要检查 `root=` 参数、存储驱动和文件系统格式；进入 Shell 后应用无法启动，则可能是动态库、权限或者服务脚本方面的问题。**

本文的目标，就是建立一套完整的嵌入式 ARM Linux 系统认知框架，使得我们能够从全局的角度理解每一层到底承担什么职责，各层之间如何交互，以及出现问题时应当从哪一层开始排查。

## 一、ARM 硬件平台：Linux 系统运行的物理基础

### 1.1 ARM 处理器的基本架构

**ARM 并不单纯代表某一个具体型号的 CPU，而是一系列处理器架构以及处理器 IP 的总称。**

在嵌入式领域之中，经常能够看到 **Cortex-A、Cortex-R 和 Cortex-M** 三类处理器。

#### Cortex-A

**Cortex-A 主要面向需要运行复杂操作系统的应用处理器场景**，例如：

-   • 智能手机；
    
-   • 工业人机界面；
    
-   • Linux 网关；
    
-   • 车载娱乐系统；
    
-   • 边缘计算设备；
    
-   • 高性能嵌入式控制器。
    

**Cortex-A 一般具备 MMU、Cache、较为完整的虚拟内存支持以及较高的处理性能，因此能够运行 Linux、Android 这类复杂操作系统。**

#### Cortex-R

**Cortex-R 主要面向高性能、确定性比较强的实时系统**，例如：

-   • 汽车控制；
    
-   • 存储控制器；
    
-   • 基带处理；
    
-   • 安全关键控制；
    
-   • 硬盘和固态硬盘控制器。
    

它更加重视**低中断延迟、确定性和功能安全。**

#### Cortex-M

**Cortex-M 主要面向微控制器领域**，例如：

-   • 传感器；
    
-   • 电机控制；
    
-   • 低功耗物联网终端；
    
-   • 简单工业控制；
    
-   • 可穿戴设备。
    

**Cortex-M 通常运行裸机程序或者 FreeRTOS、Zephyr 这类实时操作系统，一般不用于运行具备完整虚拟内存管理的标准 Linux。**

Arm 对三个系列的典型定位是：**Cortex-A 面向运行复杂操作系统的应用场景，Cortex-R 面向高性能硬实时系统，Cortex-M 则面向低功耗、确定性以及成本敏感的微控制器应用。**

#### 32 位 ARM 与 64 位 ARMv8-A

传统 ARM Linux 系统大量采用 **ARMv7-A 32 位架构**，常见处理器包括 Cortex-A7、Cortex-A8、Cortex-A9 和 Cortex-A15。

ARMv8-A 引入了 AArch64 执行状态以及 A64 指令集，使处理器能够进行 64 位运算并使用更大的虚拟地址空间。AArch64 中通常具备 31 个通用寄存器，可以通过 `X0～X30` 作为 64 位寄存器访问，也可以通过 `W0～W30` 访问相应的低 32 位。

需要注意的是，芯片采用 ARMv8-A 架构，并不代表系统一定运行 64 位 Linux。有些 ARMv8-A SoC 仍然可以运行 32 位内核和应用程序。

**用户态、内核态与异常级别**

在 AArch64 体系中，处理器通常使用异常级别来区分权限：

```
EL0：普通用户程序
EL1：操作系统内核
EL2：虚拟化管理程序
EL3：安全监控固件
```

**Linux 用户程序通常运行在 EL0，Linux 内核运行在 EL1。KVM Hypervisor 可能使用 EL2，ARM Trusted Firmware 则经常运行在 EL3。**

异常级别越高，通常能够访问的系统寄存器和控制能力越多。普通应用程序不能直接修改页表、中断控制器或者系统控制寄存器，而是必须通过系统调用请求内核完成相应操作。Armv8-A 使用 EL0～EL3 表示不同的权限层次，Linux 用户态和内核态通常分别对应 EL0 与 EL1。

**指令集、寄存器与流水线**

CPU 执行程序的时候，最终执行的是经过编译器生成的机器指令。

AArch64 主要使用 A64 指令集。每一条指令可能完成算术运算、逻辑操作、跳转、加载或者存储。

寄存器是 CPU 内部速度极快的存储单元，用于保存：

-   • 函数参数；
    
-   • 临时变量；
    
-   • 返回值；
    
-   • 栈指针；
    
-   • 程序返回地址；
    
-   • 运算中间结果。
    

流水线则是把指令执行拆解成为多个阶段，使处理器能够同时处理多条处于不同阶段的指令。现代 Cortex-A 处理器还可能具备**乱序执行、分支预测、多级 Cache 和多核一致性**等机制。

驱动程序虽然不需要每天直接编写汇编代码，但是**理解寄存器、Cache、内存顺序以及异常处理，对于分析启动流程、DMA 和中断问题是非常重要的。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/70ab89890c042ebdb398245bbd979055_MD5.jpg]]

### 1.2 SoC 的主要组成部分

**SoC 的全称是 System on Chip，也就是片上系统。**

一颗现代 ARM SoC 往往并不只是包含 CPU，而是把大量功能模块集成在同一块芯片内部：

```
┌─────────────────────────────────┐
│ CPU Cluster                     │
├─────────────────────────────────┤
│ GPU / NPU / DSP / ISP           │
├─────────────────────────────────┤
│ DDR Controller / Interconnect   │
├─────────────────────────────────┤
│ GIC / Timer / DMA Controller    │
├─────────────────────────────────┤
│ UART / I²C / SPI / CAN / USB    │
├─────────────────────────────────┤
│ Ethernet / PCIe / SD / eMMC     │
├─────────────────────────────────┤
│ Clock / Reset / Power Domain    │
└─────────────────────────────────┘
```

### CPU、GPU、NPU 与 DSP

**CPU 负责通用逻辑、操作系统和业务程序。**

**GPU 主要处理图形渲染和高度并行的计算任务。**

**NPU 主要面向神经网络推理，例如图像识别、目标检测以及语音处理。**

**DSP 擅长数字信号处理，常用于音频、通信、雷达以及传感器数据处理。**

Linux 主 CPU 和这些处理器之间可能通过**共享内存、Mailbox、RPMsg、OpenAMP 或厂商自定义接口**进行通信。

### DDR 内存控制器

**DDR 是 Linux 系统运行过程中最主要的内存。**

内核代码、应用程序、文件缓存、DMA 缓冲区以及进程堆栈，大多数都位于 DDR 中。

但是 CPU 上电之后，**DDR 通常并不能立即使用。DDR 控制器需要按照芯片和内存颗粒的参数完成初始化和训练。**

这也正是很多系统需要 SPL 的原因。**SPL 先在片内 SRAM 中运行，完成 DDR 初始化，然后才能够把更大的 U-Boot 和 Linux 内核加载到 DDR。**

### NAND、NOR、eMMC 与 SD 卡

常见启动以及存储介质包括：

-   • SPI NOR Flash；
    
-   • Raw NAND Flash；
    
-   • eMMC；
    
-   • SD 卡；
    
-   • UFS；
    
-   • NVMe。
    

**NOR Flash 支持随机读取，适合存放早期启动代码，但是容量通常比较小。**

**Raw NAND 容量较大，但存在坏块、ECC 和磨损管理问题，常与 UBI、UBIFS 配合使用。**

**eMMC 内部已经集成控制器、坏块管理和基础磨损均衡，对软件表现得类似块设备。**

SD 卡适合开发和调试，但是工业产品还需要考虑**卡片寿命、掉电一致性以及温度范围。**

### 常见外设接口

ARM SoC 中经常集成：

-   • UART：串口通信和启动日志；
    
-   • I²C：传感器、PMIC、EEPROM；
    
-   • SPI：Flash、ADC、显示控制器；
    
-   • USB：主机、设备以及 OTG；
    
-   • CAN：汽车和工业现场总线；
    
-   • Ethernet：网络通信；
    
-   • PCIe：高速网卡、NVMe 和加速卡；
    
-   • GPIO：普通数字输入输出；
    
-   • PWM：电机、背光和蜂鸣器控制；
    
-   • MIPI CSI/DSI：摄像头以及显示接口。
    

**Linux 会通过不同的内核子系统来统一管理这些硬件接口，而不是让应用程序随意直接操作寄存器。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ca98c5a567030faa244f2527dae1c5b8_MD5.jpg]]

### 1.3 嵌入式开发板的启动相关硬件

### 电源管理芯片与复位电路

SoC 和外设往往需要多路不同电压，例如：

```
CPU Core：约 0.8V～1.xV
DDR：1.1V、1.2V 或其他电压
I/O：1.8V 或 3.3V
外设电源：3.3V、5V
```

**电源管理芯片需要按照规定的时序开启各路电源。**

如果电源上升顺序错误，**SoC 可能无法启动，DDR 也可能训练失败。**

复位电路需要保证电源和时钟稳定以后再释放 CPU 复位。外部看门狗、PMIC 和 SoC 内部复位控制器都可能影响系统启动。

### 时钟源与晶振

**CPU、DDR、UART、USB、以太网以及各种外设都需要时钟。**

开发板上通常存在 24MHz、25MHz、32.768kHz 等晶振，再通过 SoC 内部 PLL 产生不同频率。

**如果主晶振不起振，CPU 可能完全无法执行指令。**

**如果 UART 时钟配置错误，串口输出可能出现乱码。**

**如果以太网参考时钟配置错误，PHY 可能无法建立链路。**

### 启动模式配置引脚

SoC 通常会在复位阶段采样一组启动模式引脚，以决定从什么介质启动，例如：

```
BOOT_MODE = 000：eMMC
BOOT_MODE = 001：SD 卡
BOOT_MODE = 010：SPI NOR
BOOT_MODE = 011：USB 下载模式
```

具体定义需要查阅芯片手册。

**启动模式电阻焊接错误，会导致 Boot ROM 从错误介质读取启动镜像。**

### 串口、JTAG 与 SWD

**UART 是嵌入式 Linux 最重要的调试接口之一。**

从 Boot ROM、SPL、U-Boot 到 Linux 内核，都可以通过串口输出日志。

**JTAG 则能够暂停 CPU、读取寄存器、设置断点和访问内存，在早期启动代码完全没有串口输出时非常有价值。**

SWD 更常见于 Cortex-M 微控制器。对于复杂 Cortex-A SoC，通常使用 JTAG 或 CoreSight 调试体系。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1c33431b00a80a3388bd861989b1fbbc_MD5.jpg]]

### 1.4 硬件原理图与 Linux 软件配置的对应关系

**嵌入式 Linux 软件并不是脱离硬件独立存在的。**

**硬件原理图中的每一条重要连接，往往都需要在设备树或者驱动中找到相对应的配置。**

### 芯片引脚与设备树节点

例如原理图中存在一个 I²C 温度传感器：

```
SoC I2C2_SCL ───── Sensor SCL
SoC I2C2_SDA ───── Sensor SDA
Sensor INT    ───── GPIO3_B2
Sensor VCC    ───── 3V3_SENSOR
```

设备树中就可能需要描述：

```
&i2c2 {
    status = "okay";

    sensor@48 {
        compatible = "vendor,temp-sensor";
        reg = <0x48>;

        interrupt-parent = <&gpio3>;
        interrupts = <10 IRQ_TYPE_LEVEL_LOW>;

        vdd-supply = <&vcc_3v3_sensor>;
    };
};
```

### GPIO 编号与引脚复用

一个物理引脚可能支持多种功能：

```
GPIO
UART_TX
I2C_SCL
SPI_CLK
PWM
```

这就是**引脚复用。**

要把引脚用作 UART，就必须在 Pin Controller 中选择 UART 功能，而不是 GPIO 功能。

**如果引脚复用配置错误，驱动即便正确加载，也无法在引脚上看到期望信号。**

### 中断、时钟和电源域

设备能够正常工作，可能同时依赖：

-   • 寄存器地址；
    
-   • 中断号；
    
-   • 中断触发方式；
    
-   • 功能时钟；
    
-   • 总线时钟；
    
-   • 复位信号；
    
-   • 电源域；
    
-   • Regulator；
    
-   • DMA 通道；
    
-   • IOMMU。
    

**设备树缺少其中任何一项，都可能导致驱动 `probe()` 失败或者设备运行不稳定。**

因此，**驱动开发不能只看芯片数据手册，还必须结合 SoC 手册、原理图、设备树以及内核日志进行综合分析。**

## 二、ARM Linux 软件栈：从底层到应用的整体架构

### 2.1 嵌入式 Linux 系统的分层模型

**典型嵌入式 Linux 系统可以划分成为五个主要层次。**

### 硬件层

硬件层包含：

-   • CPU；
    
-   • DDR；
    
-   • Flash；
    
-   • 电源；
    
-   • 时钟；
    
-   • 总线；
    
-   • 外部设备。
    

**它是整个系统运行的物理根基。**

### Bootloader 层

Bootloader 负责：

-   • 完成早期硬件初始化；
    
-   • 初始化 DDR；
    
-   • 读取存储介质；
    
-   • 加载内核和设备树；
    
-   • 设置启动参数；
    
-   • 跳转到 Linux 内核。
    

### Linux 内核层

Linux 内核负责：

-   • CPU 调度；
    
-   • 内存管理；
    
-   • 中断处理；
    
-   • 设备驱动；
    
-   • 文件系统；
    
-   • 网络协议；
    
-   • 进程隔离；
    
-   • 系统调用。
    

### 根文件系统层

根文件系统提供：

-   • Shell；
    
-   • 动态链接器；
    
-   • C 运行库；
    
-   • 基础命令；
    
-   • 配置文件；
    
-   • 设备节点；
    
-   • 系统服务；
    
-   • 应用程序。
    

### 系统服务与应用层

这一层完成产品的业务功能，例如：

-   • 图形界面；
    
-   • 网络服务；
    
-   • 数据采集；
    
-   • 视频处理；
    
-   • 设备控制；
    
-   • 云平台连接。
    

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ccbce3b44e3ad7ba275be33bb445dc3e_MD5.jpg]]

### 2.2 各层之间的职责划分

### Bootloader 为什么不能替代 Linux 内核

Bootloader 也能够访问串口、存储和网络，但是**它的主要目标是初始化硬件并加载操作系统。**

它一般不具备完整的：

-   • 多进程调度；
    
-   • 虚拟内存；
    
-   • 用户权限隔离；
    
-   • 完整网络协议栈；
    
-   • 丰富文件系统；
    
-   • 标准设备模型；
    
-   • 应用运行环境。
    

因此，**U-Boot 可以下载文件、读取 eMMC 和启动网络，但是不适合长期承担复杂产品业务。**

### 内核为什么不能直接提供完整用户交互

**Linux 内核主要负责资源管理和提供系统调用。**

虽然内核能够输出日志，也可以包含部分调试 Shell 功能，但是普通业务逻辑并不应该直接编写到内核中。

应用程序需要使用用户空间运行库、配置文件、命令行、图形库以及各种业务框架，这些都属于根文件系统和用户空间。

### 根文件系统为什么是必要组成

内核启动到最后，需要寻找第一个用户态程序，例如：

```
/init
/sbin/init
/etc/init
/bin/init
/bin/sh
```

如果这些程序不存在，内核就无法进入正常用户空间，通常会出现：

```
Kernel panic - not syncing:
No working init found
```

**initramfs 是一种能够完整承载早期用户空间的根文件系统形式，Linux 内核文档将其描述为一个自包含的根文件系统归档。**

### 应用程序如何访问硬件

**应用程序一般不直接访问物理寄存器，而是通过系统调用进入内核：**

```
应用程序
   ↓ open/read/write/ioctl
系统调用接口
   ↓
Linux 设备驱动
   ↓
MMIO / GPIO / I²C / SPI / DMA
   ↓
硬件设备
```

这样可以实现**权限隔离、资源共享和统一管理。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/369b71a38ee041cb7e9187a55675e4b8_MD5.jpg]]

### 2.3 常见系统组件及其依赖关系

### U-Boot

**U-Boot 是嵌入式 Linux 中使用非常广泛的 Bootloader。**

它可以提供：

-   • 命令行；
    
-   • 存储访问；
    
-   • 网络下载；
    
-   • 环境变量；
    
-   • 镜像校验；
    
-   • Linux 启动；
    
-   • 固件烧录。
    

### Linux Kernel

**Linux 内核负责所有核心资源管理，并通过驱动程序控制硬件。**

### Device Tree

**设备树负责描述硬件**，例如：

-   • 寄存器地址；
    
-   • 中断；
    
-   • GPIO；
    
-   • 时钟；
    
-   • 电源；
    
-   • 总线连接关系。
    

### BusyBox

**BusyBox 把大量常用 Linux 命令集成到一个可执行程序中**，例如：

```
ls
cp
mv
mount
ifconfig
sh
init
mdev
```

这些命令通过 Applet 机制共享同一个 BusyBox 二进制文件，非常适合存储空间有限的嵌入式系统。BusyBox 官方文档说明，一个 BusyBox 程序可以包含大量编译时选中的 Applet，并根据调用名称执行相应功能。

### C 运行库

常见运行库包括：

-   • glibc；
    
-   • musl；
    
-   • uClibc-ng。
    

**应用程序通过 C 运行库调用系统调用、线程、动态链接、网络以及标准输入输出等功能。**

### 初始化系统

常见初始化系统包括：

-   • BusyBox init；
    
-   • SysVinit；
    
-   • systemd；
    
-   • OpenRC。
    

**它们负责启动系统服务、挂载文件系统、创建设备节点以及监督后台进程。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7a2394c579d115d0146b449da054823b_MD5.jpg]]

### 2.4 嵌入式 Linux 与桌面 Linux 的区别

桌面 Linux 和嵌入式 Linux 使用的是同一个 Linux 内核体系，但是**产品目标存在很大差异。**

### 硬件资源差异

桌面计算机可能具备几十 GB 内存以及大容量 SSD，而嵌入式设备可能只有：

```
128MB DDR
16MB NOR Flash
512MB NAND
```

因此**嵌入式系统更重视裁剪和资源控制。**

### 启动速度要求

工业设备和车载设备可能要求几秒内完成界面显示或者关键功能启动。

这就需要优化：

-   • Boot ROM；
    
-   • SPL；
    
-   • U-Boot；
    
-   • 内核初始化；
    
-   • 根文件系统挂载；
    
-   • 服务启动。
    

### 实时性要求

某些嵌入式场景需要较低并且可预测的响应延迟，可能使用：

-   • PREEMPT\_RT；
    
-   • CPU 隔离；
    
-   • IRQ 亲和性；
    
-   • 实时调度策略；
    
-   • 独立 Cortex-R 或 Cortex-M 协处理器。
    

### 升级方式差异

桌面系统可以通过包管理器在线升级，而**嵌入式产品往往采用整镜像、A/B 分区或者 OTA 升级。**

产品还必须考虑**掉电、升级中断以及回滚。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3ff3341db359e2b73707419743fd7b9f_MD5.jpg]]

## 三、Boot ROM 与 Bootloader：系统启动的第一棒

### 3.1 ARM 设备上电后的执行流程

**CPU 上电以后，并不会直接执行 Linux 内核。**

它首先进入复位状态，紧接着从芯片架构规定的复位入口开始执行。

**现代 SoC 通常会先执行内部固化的 Boot ROM。**

### Boot ROM 的作用

**Boot ROM 是芯片制造阶段固化在 SoC 内部的只读程序，普通用户不能随意修改。**

它通常负责：

-   • 读取启动模式引脚；
    
-   • 初始化最少量时钟；
    
-   • 初始化片内 SRAM；
    
-   • 检查启动介质；
    
-   • 验证启动镜像；
    
-   • 加载第一阶段 Bootloader；
    
-   • 进入 USB 或 UART 下载模式。
    

### 启动介质检测顺序

某些芯片会按照固定顺序尝试：

```
eFuse 配置
   ↓
启动模式引脚
   ↓
eMMC Boot Partition
   ↓
SD 卡
   ↓
SPI NOR
   ↓
USB Recovery
```

不同芯片的顺序存在差异，**必须以芯片参考手册为准。**

### 烧录模式

如果正常启动介质不存在有效镜像，Boot ROM 可能进入：

-   • USB Download；
    
-   • UART Download；
    
-   • SDP；
    
-   • MaskROM；
    
-   • Recovery。
    

量产工具往往就是利用这种模式，把 SPL、U-Boot 和系统镜像写入 eMMC 或 NAND。

![](https://relay-1.bijitongbu.site/p/01c6f067ce95367bc07a00c7cb5e62ec.png)

### 3.2 SPL 与多阶段启动机制

完整 U-Boot 可能包含文件系统、网络、USB、命令行以及各种设备驱动，体积比较大。

但是 CPU 刚上电的时候，**DDR 尚未初始化，只有容量很小的片内 SRAM 可以使用。**

因此完整 U-Boot 往往无法直接运行。

### SPL 的主要职责

**SPL 是 Secondary Program Loader，也就是二级程序加载器。**

它通常负责：

-   • 初始化 CPU 基础状态；
    
-   • 初始化时钟；
    
-   • 初始化串口；
    
-   • 初始化 Pinmux；
    
-   -   **初始化 DDR；**
-   • 访问启动介质；
    
-   • 把完整 U-Boot 加载到 DDR；
    
-   • 跳转到 U-Boot Proper。
    

U-Boot 官方文档说明，多阶段启动最初就是因为完整 U-Boot 太大，Boot ROM 无法直接加载；**SPL 的典型任务是配置 SDRAM 并加载完整 U-Boot。**

某些平台还会包含：

```
TPL → VPL → SPL → U-Boot
```

其中 TPL 可以完成比 SPL 更早的初始化，VPL 则可以用于验证和选择不同启动镜像。

### DDR 初始化

**DDR 初始化是 SPL 阶段最为关键的任务之一。**

其过程可能包含：

-   • 配置 DDR PLL；
    
-   • 设置时序参数；
    
-   • 配置内存控制器；
    
-   • 设置 PHY；
    
-   • 执行读写训练；
    
-   • 检测容量；
    
-   • 验证稳定性。
    

DDR 参数错误时，常见表现包括：

-   • SPL 随机死机；
    
-   • 加载 U-Boot 后跳转失败；
    
-   • 内核解压过程中崩溃；
    
-   • 大内存访问时数据错误。
    

![](https://relay-1.bijitongbu.site/p/892eb0ef217e8243c07bef46155dfba9.png)

### 3.3 U-Boot 的核心功能

完整 U-Boot 进入 DDR 后，就能够提供更加丰富的功能。

### 硬件初始化

U-Boot 会继续初始化：

-   • 串口；
    
-   • 定时器；
    
-   • 存储控制器；
    
-   • USB；
    
-   • Ethernet；
    
-   • 显示；
    
-   • PCIe；
    
-   • 环境变量存储。
    

### 加载内核、设备树和根文件系统

U-Boot 可以从多种介质加载镜像：

```
eMMC
SD
NAND
SPI NOR
USB
TFTP
NFS
SATA
NVMe
```

典型过程为：

```
读取 Kernel Image 到 DDR
读取 DTB 到 DDR
读取 Initramfs 到 DDR
设置 bootargs
跳转到内核入口
```

### 调试与烧录

U-Boot 命令行可以执行：

```
md      查看内存
mw      修改内存
mmc     访问 eMMC/SD
sf      访问 SPI Flash
nand    访问 NAND
tftp    网络下载
ping    测试网络
fdt     修改设备树
booti   启动 ARM64 Image
bootz   启动 zImage
bootm   启动 U-Boot 镜像或 FIT
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/b9c9e66e9678e388650bda9769d95ad9_MD5.jpg]]

### 3.4 U-Boot 环境变量与启动命令

### `bootcmd`

**`bootcmd` 是 U-Boot 自动启动时执行的命令。**

例如：

```
setenv bootcmd \
'mmc dev 0; run load_kernel; run load_fdt; run boot_linux'
```

### `bootargs`

**`bootargs` 会作为 Linux 内核命令行传递给内核。**

例如：

```
setenv bootargs \
'console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait rw'
```

其中：

-   -   `console=` 指定控制台；
-   -   `root=` 指定根文件系统；
-   -   `rootwait` 等待存储设备出现；
-   -   `rw` 以可读写方式挂载根文件系统。

### 镜像加载地址

常见变量包括：

```
loadaddr
kernel_addr_r
fdt_addr_r
ramdisk_addr_r
```

**这些地址必须避免互相覆盖，也不能覆盖 U-Boot 自身、栈、设备树或保留内存。**

### `booti`、`bootz` 与 `bootm`

ARM64 系统常使用：

```
booti ${kernel_addr_r} - ${fdt_addr_r}
```

启动 `Image` 格式内核。

32 位 ARM 常见：

```
bootz ${kernel_addr_r} - ${fdt_addr_r}
```

启动 `zImage`。

`bootm` 常用于启动：

-   -   `uImage`；
-   • FIT Image；
    
-   • 某些经过 U-Boot Header 封装的镜像。
    

U-Boot 官方文档将 `booti` 定义为启动平坦或压缩的 Linux `Image`，将 `bootz` 定义为启动 Linux `zImage`，两者都可以同时接收 initrd 和设备树地址。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0457568822f0a691704fb06a7469d1b9_MD5.jpg]]

### 3.5 常见 Bootloader 启动故障

### DDR 初始化失败

表现为：

-   • SPL 没有输出；
    
-   • 输出到某个位置后停止；
    
-   • U-Boot 校验失败；
    
-   • 数据随机异常。
    

排查：

-   • DDR 型号和容量；
    
-   • PCB 走线；
    
-   • 时序参数；
    
-   • 电源；
    
-   • 时钟；
    
-   • DDR Training 日志。
    

### 找不到启动介质

表现为：

```
MMC init failed
No boot device
Bad magic
```

排查：

-   • Boot Strap；
    
-   • eMMC 焊接；
    
-   • SD 卡供电；
    
-   • 引脚复用；
    
-   • 控制器时钟；
    
-   • 镜像写入偏移。
    

### 内核镜像地址冲突

表现为：

-   • 解压覆盖设备树；
    
-   • Initramfs 覆盖内核；
    
-   • 启动后立即异常。
    

应当检查：

```
bdinfo
printenv kernel_addr_r
printenv fdt_addr_r
printenv ramdisk_addr_r
```

### 设备树加载错误

表现为：

```
FDT_ERR_BADMAGIC
Wrong Image Format
Could not find a valid device tree
```

需要检查 **DTB 文件、加载地址和启动命令参数。**

### 启动参数错误

例如 **`root=` 指向错误分区，就会出现无法挂载根文件系统的问题。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/beb47a202280bb5668cc7f97af27dcc7_MD5.jpg]]

## 四、Linux 内核：嵌入式系统的核心管理者

### 4.1 Linux 内核承担的核心职责

**Linux 内核如同整个系统的大管家一般，负责管理所有共享资源。**

### 进程与线程调度

内核决定：

-   • 哪一个线程运行；
    
-   • 在哪一个 CPU 上运行；
    
-   • 运行多长时间；
    
-   • 什么时候被抢占；
    
-   • 实时任务和普通任务如何调度。
    

### 虚拟内存管理

**每一个用户进程都有自己的虚拟地址空间。**

内核负责：

-   • 页表；
    
-   • 内存分配；
    
-   • 缺页异常；
    
-   • 文件映射；
    
-   • Copy-on-Write；
    
-   • 内存回收；
    
-   • Cache 管理。
    

### 中断与异常处理

外设产生中断以后，CPU 会进入内核异常处理流程。

**内核负责识别中断来源，并调用相应设备驱动。**

### 文件系统

Linux 通过 VFS 向应用提供统一接口：

```
open
read
write
close
mmap
```

底层可以是 ext4、UBIFS、SquashFS、NFS 或其他文件系统。

### 网络协议栈

Linux 内核实现：

-   • Ethernet；
    
-   • ARP；
    
-   • IP；
    
-   • TCP；
    
-   • UDP；
    
-   • Netfilter；
    
-   • Routing；
    
-   • Socket。
    

### 设备驱动

**驱动负责把通用内核接口转换成为具体硬件操作。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/761c4f955ff7d40b0d8a6ba2b18faa56_MD5.jpg]]

### 4.2 ARM Linux 内核启动过程

U-Boot 跳转到内核以后，内核会进入架构相关入口。

ARM64 启动协议要求引导程序准备好内核镜像、设备树以及相应寄存器状态，然后把控制权交给内核入口。

内核启动流程可以高度简化成为：

```
内核入口
   ↓
建立早期页表
   ↓
切换异常级别和执行环境
   ↓
解压或重定位内核
   ↓
start_kernel()
   ↓
初始化内存管理
   ↓
初始化中断和定时器
   ↓
初始化调度器
   ↓
初始化驱动模型
   ↓
执行 initcall
   ↓
挂载根文件系统
   ↓
执行 PID 1
```

### `start_kernel()`

**`start_kernel()` 是通用内核初始化的重要入口。**

它会依次初始化：

-   • 日志；
    
-   • CPU；
    
-   • 调度器；
    
-   • 内存；
    
-   • 中断；
    
-   • RCU；
    
-   • 定时器；
    
-   • VFS；
    
-   • 设备模型。
    

### Initcall

内建驱动通常通过不同等级的 Initcall 进行初始化，例如：

```
early_initcall
core_initcall
postcore_initcall
arch_initcall
subsys_initcall
fs_initcall
device_initcall
late_initcall
```

设备、总线和驱动会在这些阶段逐步注册。

### 第一个用户进程

内核完成初始化以后，会尝试执行第一个用户进程。

**成功以后，系统就从纯内核环境进入了用户空间。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/4e60240f80a0f312166d8f3812b1c339_MD5.jpg]]

### 4.3 Linux 内核目录结构

### `arch/arm` 与 `arch/arm64`

保存 ARM 架构相关代码，例如：

-   • 启动入口；
    
-   • 页表；
    
-   • 中断；
    
-   • Cache；
    
-   • SMP；
    
-   • 异常向量；
    
-   • 架构系统调用。
    

### `drivers`

保存设备驱动，例如：

```
drivers/gpio
drivers/i2c
drivers/spi
drivers/net
drivers/mmc
drivers/usb
drivers/pci
drivers/gpu
drivers/media
```

### `fs`

保存 VFS 和文件系统实现。

### `mm`

保存内存管理代码。

### `net`

保存网络协议栈。

### `kernel`

保存调度、信号、锁、定时器和通用核心机制。

### `include`

保存内核头文件和公共接口。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/324a8fbeeca2ca27fd79ba4be2a90124_MD5.jpg]]

### 4.4 内核配置与裁剪

配置内核：

```
make ARCH=arm64 menuconfig
```

指定交叉编译器：

```
make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     menuconfig
```

### 内建与模块

配置项常见状态：

```
[*] 内建
[M] 模块
[ ] 不启用
```

**启动早期必须使用的驱动，应当内建进内核**，例如：

-   • 根文件系统所在存储控制器；
    
-   • 根文件系统类型；
    
-   • 必要时的文件系统解密；
    
-   • 早期串口。
    

**如果根文件系统依赖的驱动被编译为模块，而模块又存放在尚未挂载的根文件系统中，就会形成无法启动的循环依赖。**

### 调试版与生产版

调试版本可能启用：

-   • DEBUG\_KERNEL；
    
-   • LOCKDEP；
    
-   • KASAN；
    
-   • DEBUG\_FS；
    
-   • Dynamic Debug；
    
-   • Ftrace；
    
-   • 更多日志。
    

生产版本则需要综合考虑：

-   • 镜像大小；
    
-   • 启动时间；
    
-   • 性能；
    
-   • 安全性；
    
-   • 日志量；
    
-   • 可维护性。
    

**不能为了减小镜像而删除所有诊断能力，否则量产后的故障将很难定位。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a9d306e7358aa43a0d22a2076d36c931_MD5.jpg]]

### 4.5 内核镜像类型

### `Image`

ARM64 常用未封装内核镜像。

### `zImage`

传统 32 位 ARM 常见的自解压内核镜像。

### `uImage`

在内核镜像外面增加 U-Boot Legacy Image Header，其中可以包含加载地址、入口地址、类型和校验信息。

### FIT Image

FIT 是 Flattened Image Tree。

它可以把以下内容组织在一个镜像中：

-   • 内核；
    
-   • 多个设备树；
    
-   • Initramfs；
    
-   • 固件；
    
-   • Hash；
    
-   • 数字签名；
    
-   • 不同板级配置。
    

**FIT 更适合需要安全启动、多个设备树以及 A/B 系统的产品。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/54b07cf820ac8de78c62557dfa1e3f8c_MD5.jpg]]

## 五、设备树：连接硬件描述与 Linux 驱动

### 5.1 为什么 ARM Linux 需要设备树

早期 ARM Linux 经常为每一块开发板编写板级 C 文件。

随着 SoC 和开发板数量不断增加，内核中出现了大量重复的硬件描述代码。

**设备树把硬件描述从内核源码中分离出来，使得同一套内核可以配合不同 DTB 适配不同开发板。**

Linux 设备树使用模型的核心目的之一，就是**描述运行时不能由硬件自动发现的平台设备以及它们之间的连接关系。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/da9fdb4341f38c054bfb48a9b1bdbc12_MD5.jpg]]

### 5.2 设备树文件体系

### DTS

DTS 是板级设备树源文件，例如：

```
my-board.dts
```

### DTSI

DTSI 是可被多个 DTS 引用的公共片段，例如：

```
soc.dtsi
chip-family.dtsi
```

SoC 级 DTSI 描述芯片内部固定资源，板级 DTS 描述：

-   • 哪些设备启用；
    
-   • 使用哪些引脚；
    
-   • 外接了什么器件；
    
-   • 电源如何连接；
    
-   • GPIO 极性如何配置。
    

### DTB

DTS 经过 DTC 编译成为二进制 DTB：

```
dtc -I dts -O dtb \
    -o my-board.dtb \
    my-board.dts
```

正常内核构建通常由 Makefile 自动调用 DTC。

### U-Boot 传递 DTB

**U-Boot 把 DTB 加载到 DDR，并在启动内核时把设备树地址传入内核。**

**内核解析 DTB 后创建相应的平台设备和硬件资源。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f245fed53efd6397846bffe6f4878385_MD5.jpg]]

### 5.3 设备树的基本语法

示例：

```
uart3: serial@30860000 {
    compatible = "vendor,soc-uart";
    reg = <0x0 0x30860000 0x0 0x10000>;
    interrupts = <GIC_SPI 58 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clk UART3_CLK>;
    status = "disabled";
};
```

板级文件可以覆盖：

```
&uart3 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_uart3>;
    status = "okay";
};
```

### `compatible`

**用于描述设备的硬件兼容性**，例如：

```
compatible = "vendor,chip-model";
```

设备树绑定要求 `compatible` 应当足够具体，并且当硬件编程模型发生不兼容变化时，应使用新的兼容字符串。

### `reg`

**描述设备寄存器地址和长度。**

具体单元数量由父节点的：

```
#address-cells
#size-cells
```

决定。

### `interrupts`

描述中断号和触发方式。

### `clocks`

引用设备所需时钟。

### `gpios`

引用 GPIO 控制器、编号以及有效电平。

### `status`

常见值：

```
okay
disabled
reserved
```

**`status = "disabled"` 的节点一般不会创建可供普通驱动绑定的设备。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/22ee090b06254c6dca402cd512240812_MD5.jpg]]

### 5.4 设备树与驱动匹配机制

驱动中定义匹配表：

```
static const struct of_device_id demo_of_match[] = {
    {
        .compatible = "vendor,demo-device",
    },
    { }
};

MODULE_DEVICE_TABLE(of, demo_of_match);
```

注册平台驱动：

```
static struct platform_driver demo_driver = {
    .probe = demo_probe,
    .remove = demo_remove,
    .driver = {
        .name = "demo-device",
        .of_match_table = demo_of_match,
    },
};
```

**当设备树节点被转换成为 `platform_device`，并且 `compatible` 与驱动匹配成功以后，Driver Core 就会调用 `probe()`。**

Platform Bus 主要用于管理 SoC 中直接挂接到 CPU 地址空间、无法像 PCI 和 USB 那样自行枚举的设备。驱动注册以后，未绑定的平台设备会被检查是否匹配，然后调用相应的 `probe()`。

`probe()` 中可以获取资源：

```
void __iomem *base;
int irq;

base = devm_platform_ioremap_resource(pdev, 0);
if (IS_ERR(base))
    return PTR_ERR(base);

irq = platform_get_irq(pdev, 0);
if (irq < 0)
    return irq;
```

还可以获取：

```
Clock
Reset
GPIO
Regulator
DMA Channel
PHY
IOMMU
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/841c52b487849c9a2638286d9c82fb6f_MD5.jpg]]

### 5.5 常见设备树问题

### 地址与长度错误

**`reg` 地址错误会导致驱动访问错误寄存器，严重时可能直接触发总线异常。**

### GPIO 极性错误

例如硬件复位脚低电平有效：

```
reset-gpios = <&gpio3 10 GPIO_ACTIVE_LOW>;
```

若错误写成 `GPIO_ACTIVE_HIGH`，**设备可能始终处于复位状态。**

### 中断触发方式错误

把边沿中断配置成电平中断，可能造成：

-   • 中断风暴；
    
-   • 丢中断；
    
-   • 中断只触发一次；
    
-   • 中断无法清除。
    

### 时钟、电源和复位资源缺失

**设备寄存器可以访问，并不代表设备内部逻辑已经工作。**

某些设备必须依次执行：

```
打开电源
   ↓
打开总线时钟
   ↓
打开功能时钟
   ↓
解除复位
   ↓
初始化寄存器
```

### 引脚复用冲突

同一个引脚被 UART 和 SPI 同时使用时，最后生效的 Pinmux 配置可能导致另一个外设失效。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5a78eb272f7244d788b2c29114061385_MD5.jpg]]

## 六、Linux 设备驱动：让操作系统真正控制硬件

### 6.1 Linux 驱动程序的作用

**驱动位于硬件和上层软件之间。**

```
应用程序
   ↓
系统调用或子系统接口
   ↓
Linux 驱动
   ↓
寄存器、中断、DMA、总线
   ↓
硬件设备
```

驱动一方面需要理解硬件手册，另一方面还必须遵循 Linux 内核子系统规范。

例如同样是一颗温度传感器，可以接入：

-   • IIO；
    
-   • hwmon；
    
-   • Input；
    
-   • 字符设备。
    

**正确做法一般是优先选择现有内核子系统，而不是为所有设备都自行创建一个字符设备节点。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/917fb4e9b48ef216864474db6f6f2993_MD5.jpg]]

### 6.2 Linux 设备模型

Linux 设备模型的核心对象包括：

```
Device
Driver
Bus
Class
```

### Device

代表一个具体设备实例。

### Driver

代表可以管理某类设备的软件。

### Bus

负责设备发现和设备驱动匹配，例如：

-   • Platform；
    
-   • I²C；
    
-   • SPI；
    
-   • USB；
    
-   • PCIe。
    

### Class

按照功能向用户空间组织设备，例如：

```
/net
/block
/input
/tty
/video4linux
```

**Linux Driver Model 为总线、设备和驱动提供统一的数据模型以及 sysfs 表示。**

### Platform 总线

适合 SoC 内部无法自动枚举的设备。

### I²C 与 SPI

I²C 和 SPI 控制器本身通常是 Platform Device，而挂接在控制器下面的器件则由 I²C 或 SPI 总线管理。

### USB 与 PCIe

USB 和 PCIe 具备较强的自动枚举能力，可以通过协议读取设备标识以及资源信息。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/36ffd19d2094cae3b43559b5dcf9a20d_MD5.jpg]]

### 6.3 常见驱动类型

### 字符设备驱动

按照字节流或控制接口访问，例如：

-   • 串口；
    
-   • 自定义 FPGA；
    
-   • 简单控制设备。
    

### 块设备驱动

按照块进行随机读写，例如：

-   • eMMC；
    
-   • SD；
    
-   • NVMe；
    
-   • SATA 硬盘。
    

### 网络设备驱动

**向网络协议栈注册 `net_device`，而不是提供普通 `/dev` 读写接口。**

### 输入子系统驱动

适用于：

-   • 按键；
    
-   • 触摸屏；
    
-   • 鼠标；
    
-   • 键盘；
    
-   • 遥控器。
    

### DRM 显示驱动

现代 Linux 显示系统一般使用 **DRM/KMS**，而不是只使用传统 Framebuffer。

### V4L2 驱动

适用于摄像头、视频采集以及编解码设备。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/14502500ee00c3690c243a13d01e10f9_MD5.jpg]]

### 6.4 驱动访问硬件的核心机制

### MMIO

设备寄存器通常使用：

```
readl()
writel()
readb()
writeb()
```

访问，**不能简单当作普通内存指针解引用。**

### GPIO

现代驱动推荐使用描述符接口：

```
struct gpio_desc *reset_gpio;

reset_gpio = devm_gpiod_get(dev,
                            "reset",
                            GPIOD_OUT_HIGH);
```

### 中断处理

注册中断：

```
ret = devm_request_threaded_irq(dev,
                                irq,
                                demo_irq,
                                demo_irq_thread,
                                IRQF_ONESHOT,
                                "demo",
                                data);
```

**硬中断处理应当尽量简短，耗时工作可以放到 Threaded IRQ 或 Workqueue。**

### DMA

**DMA 允许设备直接访问内存。**

驱动必须使用：

```
dma_alloc_coherent()
dma_map_single()
dma_map_sg()
```

等 DMA API，**不能直接把 CPU 虚拟地址交给设备。**

### Clock、Reset 和 Regulator

常见操作：

```
clk_prepare_enable(clk);
reset_control_deassert(rst);
regulator_enable(vdd);
```

退出时必须按照合理顺序关闭。

### 锁和等待机制

常见同步机制包括：

-   • Mutex：允许睡眠；
    
-   • Spinlock：不能睡眠；
    
-   • Completion：等待一次事件完成；
    
-   • Wait Queue：等待条件成立；
    
-   • Atomic：简单原子计数；
    
-   • RCU：读多写少的并发场景。
    

![[Inbox/笔记同步助手/微信公众号/2026/08/images/942ce32533ab39315238db55f3612604_MD5.jpg]]

### 6.5 用户空间访问驱动的方式

### `/dev` 设备节点

字符设备可以实现：

```
.open
.read
.write
.unlocked_ioctl
.mmap
.poll
.release
```

应用程序调用：

```
fd = open("/dev/demo", O_RDWR);
read(fd, buf, len);
ioctl(fd, DEMO_START, &arg);
```

### sysfs

sysfs 适合暴露简单属性，例如：

```
enable
temperature
mode
status
```

**sysfs 不是用于大量高速数据传输的接口。它主要用于导出内核对象以及简单属性。**

### procfs

procfs 主要用于进程以及内核状态信息。

**新设备驱动不应当随意把大量控制接口放入 `/proc`。**

### Netlink

适合内核与用户空间之间进行结构化消息通信。

### `mmap`

可以把驱动管理的缓冲区映射到用户空间，减少数据复制。

但是必须严格控制映射范围、Cache 属性和权限。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1fd8724e2c2090dcf7bbe33043e142d7_MD5.jpg]]

### 6.6 驱动调试方法

查看日志：

```
dmesg -w
```

查看模块：

```
lsmod
```

加载模块：

```
insmod demo.ko
```

卸载模块：

```
rmmod demo
```

查看中断：

```
cat /proc/interrupts
```

查看 DebugFS：

```
mount -t debugfs none /sys/kernel/debug
```

打开 Dynamic Debug：

```
echo 'module demo +p' \
  > /sys/kernel/debug/dynamic_debug/control
```

硬件信号问题还需要配合：

-   • 万用表；
    
-   • 示波器；
    
-   • 逻辑分析仪；
    
-   • JTAG；
    
-   • 总线分析仪。
    

例如 I²C 通信失败时，内核日志只能告诉我们控制器返回超时，但是逻辑分析仪可以直接看到：

-   • 有没有 Start；
    
-   • 地址是否正确；
    
-   • 从设备是否 ACK；
    
-   • SCL 是否被拉低；
    
-   • SDA 是否存在毛刺。
    

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9a507707031de1fbc7326d624b687a32_MD5.jpg]]

## 七、根文件系统：Linux 用户空间的运行环境

### 7.1 根文件系统为什么不可缺少

Linux 内核负责资源管理，但是它本身并不包含完整的用户命令、配置文件和业务程序。

**内核初始化完成以后，需要从根文件系统中找到 PID 1。**

根文件系统和普通数据分区的区别在于，**它必须包含系统运行所需要的核心目录、运行库和初始化程序。**

**如果根文件系统不存在或者无法挂载，系统就无法进入正常用户空间。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/416d472aa106e8f936c631a1f1d8ca1c_MD5.jpg]]

### 7.2 根文件系统的标准目录

### `/bin`

保存基础用户命令，例如：

```
sh
ls
cp
mount
```

### `/sbin`

保存系统管理命令和 init。

### `/etc`

保存系统配置，例如：

```
inittab
fstab
passwd
network
init.d
```

### `/dev`

保存设备节点。

### `/proc`

由内核动态提供进程和系统状态。

### `/sys`

由设备模型和内核子系统动态导出设备信息。

### `/lib`

保存共享库、动态链接器以及内核模块。

### `/usr`

保存更多应用程序和资源。

### `/var`

保存运行过程中不断变化的数据，例如日志和数据库。

### `/tmp`

保存临时文件。

### `/mnt` 与 `/media`

用于挂载其他文件系统。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/22f1154c2b9d9a625e8582a14eac1bdb_MD5.jpg]]

### 7.3 BusyBox 的作用

**BusyBox 如同嵌入式 Linux 用户空间中的瑞士军刀一般。**

它把很多命令编译到一个二进制程序：

```
/bin/busybox
```

再建立软链接：

```
/bin/ls   -> busybox
/bin/cp   -> busybox
/bin/sh   -> busybox
/sbin/init -> ../bin/busybox
```

执行 `/bin/ls` 时，BusyBox 根据 `argv[0]` 判断应当运行 `ls` Applet。

编译基本流程：

```
make menuconfig
make CROSS_COMPILE=aarch64-linux-gnu-
make CONFIG_PREFIX=$PWD/rootfs install
```

然后创建：

```
rootfs/
├── bin
├── sbin
├── etc
├── dev
├── proc
├── sys
├── lib
├── usr
├── var
└── tmp
```

BusyBox 官方资料说明，被编译进 BusyBox 的命令以 Applet 形式存在，并可通过 BusyBox 本体或相应软链接进行调用。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/23afc1e3c6ae484d47de53ad84ccc521_MD5.jpg]]

### 7.4 C 运行库的选择

### glibc

优点：

-   • 功能完整；
    
-   • 兼容性广；
    
-   • 生态成熟；
    
-   • 适合复杂应用。
    

缺点：

-   • 体积相对较大；
    
-   • 对超小型系统不一定合适。
    

### musl

优点：

-   • 实现简洁；
    
-   • 静态链接体积通常较小；
    
-   • 设计重视一致性；
    
-   • 适合容器和精简系统。
    

### uClibc-ng

主要面向资源受限的嵌入式 Linux，提供较小的 C 运行库实现。

### 动态链接与静态链接

动态链接程序依赖目标系统中的：

-   • 动态链接器；
    
-   • libc；
    
-   • 其他共享库。
    

静态链接把依赖库合并进可执行文件，部署简单，但是文件体积较大，多个程序之间不能共享代码页。

### 工具链兼容性

**应用程序使用哪一套 C 运行库编译，目标根文件系统就需要具备兼容的运行环境。**

例如使用 glibc 工具链编译的程序，不能简单复制到只有 musl 的根文件系统中运行。

常见错误包括：

```
No such file or directory
```

即使可执行文件明明存在，也可能是因为 **ELF 中指定的动态链接器不存在。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/153932115f4326fe7c6c9a49cbcbc923_MD5.jpg]]

### 7.5 根文件系统的组织形式

### initramfs

initramfs 位于内存中，可以内置进内核，也可以由 Bootloader 单独传入。

它适合：

-   • 最小系统；
    
-   • 救援系统；
    
-   • 早期启动；
    
-   • 挂载加密根文件系统；
    
-   • 执行升级和恢复。
    

### ext4

适用于 eMMC、SD、SATA 和 NVMe 等块设备。

优点是生态成熟、支持日志和大容量存储。

### SquashFS

只读压缩文件系统，适合只读系统镜像。

优点：

-   • 压缩率较高；
    
-   • 镜像不容易被意外修改；
    
-   • 有利于系统完整性。
    

### UBIFS

适用于 Raw NAND 上的 UBI 层，能够处理坏块和磨损均衡。

### OverlayFS

可以把只读底层与可写上层组合起来：

```
SquashFS 只读系统
        +
可写 Data 分区
        ↓
OverlayFS 统一视图
```

适合只读根文件系统以及恢复出厂设置。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/8092cd91178fddb6bb42be1e3360611f_MD5.jpg]]

### 7.6 根文件系统启动失败的常见原因

### `root=` 参数错误

例如实际根分区是：

```
/dev/mmcblk0p3
```

但是 `bootargs` 写成：

```
root=/dev/mmcblk0p2
```

### 找不到 init

常见提示：

```
No working init found
```

检查：

```
/sbin/init
/init
/bin/sh
```

是否存在并具备执行权限。

### 动态链接器缺失

查看程序：

```
readelf -l application | grep interpreter
```

可能显示：

```
/lib/ld-linux-aarch64.so.1
```

**目标根文件系统中必须存在对应文件。**

### 文件系统驱动没有内建

**如果根文件系统为 ext4，但 ext4 只编译成模块，内核又没有 initramfs 提前加载模块，就无法挂载根分区。**

### 存储驱动加载过晚

可使用：

```
rootwait
rootdelay=5
```

但是**根本解决办法是确保控制器、PHY、时钟以及块设备驱动能够在正确阶段初始化。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e421a11153d43a9da6e82b74fe8543a5_MD5.jpg]]

## 八、系统初始化与后台服务：从内核启动到可用系统

### 8.1 Linux 的第一个用户态进程

**PID 1 是 Linux 用户空间中的第一个进程。**

它具备特殊地位：

-   • 是其他用户进程的祖先；
    
-   • 回收孤儿进程；
    
-   • 启动系统服务；
    
-   • 处理关机和重启；
    
-   • 维护系统运行状态。
    

常见 PID 1 包括：

```
BusyBox init
SysVinit
systemd
OpenRC init
自定义 init
```

调试时可以在内核命令行加入：

```
init=/bin/sh
```

这样内核会直接启动 Shell，而不执行正常初始化系统。

但是此时：

-   • 根文件系统可能是只读；
    
-   -   `/proc` 和 `/sys` 可能没有挂载；
-   • 网络没有配置；
    
-   • 后台服务没有启动。
    

**如果 PID 1 退出，内核通常会认为用户空间已经无法继续工作，并触发 Panic。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/381aa636e5356688edb2b6e43f12973a_MD5.jpg]]

### 8.2 常见初始化系统

### BusyBox init

优点：

-   • 体积小；
    
-   • 配置简单；
    
-   • 适合资源受限设备。
    

### SysVinit

按照启动脚本和 Runlevel 管理服务，传统嵌入式系统中比较常见。

### systemd

提供：

-   • 服务依赖；
    
-   • 并行启动；
    
-   • Socket Activation；
    
-   • 日志；
    
-   • 设备管理；
    
-   • Cgroup 管理；
    
-   • 定时任务。
    

适合功能复杂、服务数量较多的产品。

### OpenRC

提供相对轻量级的依赖式服务管理，常见于一些精简发行版。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3ccbe7e3c44bbe955f6f30c174d4a0bd_MD5.jpg]]

### 8.3 系统启动脚本

BusyBox init 常读取：

```
/etc/inittab
```

示例：

```
::sysinit:/etc/init.d/rcS
::respawn:-/bin/sh
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
```

`rcS` 中可以执行：

```
#!/bin/sh

mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

hostname embedded-device

ifconfig lo up
ifconfig eth0 up

mkdir -p /var/run
mkdir -p /var/log

/etc/init.d/S40network start
/etc/init.d/S50sshd start
/etc/init.d/S90application start
```

**BusyBox 的 `inittab` 语法具有自身特点，不应直接套用其他 init 实现的配置格式。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d52a09ca8f3f04fd84bc631f65b57932_MD5.jpg]]

### 8.4 嵌入式系统中的常见服务

### udev 与 mdev

它们负责根据内核 Uevent 动态管理设备节点。

资源受限系统可以使用 BusyBox mdev，复杂系统通常使用 udev 或 systemd-udevd。

### SSH

常见实现：

-   • OpenSSH；
    
-   • Dropbear。
    

Dropbear 体积较小，适合嵌入式系统。

### 日志服务

可以使用：

-   • syslogd；
    
-   • rsyslog；
    
-   • journald；
    
-   • 自定义日志守护进程。
    

**量产产品必须限制日志空间，避免日志写满存储分区。**

### Watchdog 服务

Watchdog 服务需要周期性喂狗。

**一旦关键进程卡死，硬件看门狗就会复位系统。**

### 时间同步

常见方式：

-   • NTP；
    
-   • Chrony；
    
-   • PTP；
    
-   • RTC。
    

对于没有电池 RTC 的设备，断电重启后系统时间可能从默认值开始，需要联网校时。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e11ad9400cdb3bc6cffb66cbb6ecbb04_MD5.jpg]]

### 8.5 启动速度优化

### 精简 Bootloader 等待

例如：

```
setenv bootdelay 0
```

但是完全取消倒计时以后，**应保留可靠的恢复入口。**

### 精简内核

删除不需要的：

-   • 总线；
    
-   • 文件系统；
    
-   • 网络协议；
    
-   • 调试功能；
    
-   • 设备驱动。
    

### 延迟非关键驱动

摄像头、无线网络以及非关键传感器可以在主业务可用以后再初始化。

### 并行启动服务

systemd 可以依据依赖关系并行启动服务。

自定义脚本也可以让互不依赖的服务并行执行，但是需要避免竞态条件。

### 测量启动时间

U-Boot 中可以记录时间戳。

内核可以使用：

```
initcall_debug
```

查看 Initcall 耗时。

用户空间可以使用：

```
systemd-analyze
systemd-analyze blame
```

**真正的启动优化必须先测量，再针对耗时最大的阶段进行处理。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/421dfef266401df0f62bf5be998fbfcc_MD5.jpg]]

## 九、应用程序与系统调用：业务功能如何落地

### 9.1 嵌入式 Linux 应用程序的运行方式

嵌入式应用可以是：

-   • 前台交互程序；
    
-   • 后台守护进程；
    
-   • 网络服务；
    
-   • 图形界面；
    
-   • 数据采集服务；
    
-   • 多媒体处理程序。
    

### 单进程架构

结构简单，但是某个模块崩溃可能导致整个程序退出。

### 多进程架构

隔离性更好，模块之间通过 IPC 通信。

### 多线程架构

共享内存方便，但是需要正确处理：

-   • 锁；
    
-   • 条件变量；
    
-   • 死锁；
    
-   • 数据竞争；
    
-   • 线程退出。
    

![](https://relay-1.bijitongbu.site/p/cf9cabcf92bd912f8b2297a79f28e698.png)

### 9.2 应用程序如何访问系统资源

应用程序调用 C 运行库接口：

```
open();
read();
write();
ioctl();
socket();
mmap();
```

**运行库最终执行系统调用，使 CPU 从 EL0 进入 EL1。**

### 文件描述符

文件、设备、Socket、管道和 Eventfd 都可以通过文件描述符进行管理。

**这体现了 Linux“一切皆文件”的统一接口思想。**

### 设备文件

例如：

```
/dev/ttyS0
/dev/i2c-1
/dev/spidev0.0
/dev/video0
/dev/input/event0
```

### 内存映射

`mmap()` 可以用于：

-   • 映射文件；
    
-   • 共享内存；
    
-   • 映射驱动缓冲区；
    
-   • 零拷贝数据传输。
    

### Socket

网络应用通过 Socket 进行 TCP、UDP、Unix Domain Socket 或其他协议通信。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/421dcb7b69cc100d6676f540a5e28fbe_MD5.jpg]]

### 9.3 进程间通信方式

### 管道

适合具有亲缘关系的进程进行单向字节流通信。

### 命名管道

通过文件路径让无亲缘关系的进程通信。

### 消息队列

适合传递具备边界的结构化消息。

### 共享内存

**效率较高，但是需要 Mutex、Semaphore 或 Futex 协调并发访问。**

### 信号

适合传递简单异步事件，不适合传输大量数据。

### Unix Domain Socket

支持本机进程之间进行流式或者数据报通信，还可以传递文件描述符。

### D-Bus

适合较复杂的系统服务通信和对象化接口。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/231595cca4cf6d0373e72a39b01724c6_MD5.jpg]]

### 9.4 常见嵌入式应用框架

### Qt

适合复杂图形界面，支持：

-   • Widgets；
    
-   • QML；
    
-   • 网络；
    
-   • 多媒体；
    
-   • 线程；
    
-   • 数据库。
    

### LVGL

适合资源相对受限的 MCU 或 Linux Framebuffer/DRM 场景，界面体积较小。

### GStreamer

适合音视频采集、编解码、转换以及传输。

### OpenCV

适合图像处理和计算机视觉。

### MQTT

适合物联网设备和云平台之间进行轻量消息通信。

### Web 服务

嵌入式设备也可以运行：

-   • Nginx；
    
-   • lighttpd；
    
-   • Mongoose；
    
-   • CivetWeb；
    
-   • 自定义 HTTP Server。
    

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0a14fb48f9221ed6e69d29e94314cd56_MD5.jpg]]

### 9.5 应用程序的稳定性设计

### 异常处理

**所有系统调用都需要检查返回值：**

```
ssize_t ret = read(fd, buf, sizeof(buf));

if (ret < 0) {
    if (errno == EINTR)
        ...
}
```

### 内存泄漏

可使用：

-   • Valgrind；
    
-   • AddressSanitizer；
    
-   • LeakSanitizer；
    
-   -   `/proc/<pid>/smaps`；
-   • 长时间压力测试。
    

### 线程同步

需要明确：

-   • 共享对象由谁保护；
    
-   • 锁的顺序；
    
-   • 回调中是否允许阻塞；
    
-   • 线程退出如何通知。
    

### 看门狗与进程拉起

可以由：

-   • systemd；
    
-   • BusyBox init；
    
-   • Supervisor；
    
-   • 自定义守护进程；
    
-   • 硬件 Watchdog。
    

负责重启异常应用。

### 数据持久化和掉电保护

关键数据应当采用：

-   • 临时文件加原子重命名；
    
-   -   `fsync()`；
-   • 双副本；
    
-   • 日志式存储；
    
-   • CRC；
    
-   • 版本号；
    
-   • 掉电检测。
    

**不能只在内存中修改配置，然后假定 `write()` 返回就一定已经写入 Flash。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ad6e2ee0544ce70fcdbd087a726c21db_MD5.jpg]]

## 十、交叉编译与系统构建：把完整系统制作出来

### 10.1 为什么需要交叉编译

开发主机可能是：

```
x86_64 Linux
```

目标设备可能是：

```
ARMv7-A
AArch64
```

**x86\_64 编译器产生的机器代码不能直接在 ARM CPU 上运行。**

因此需要**交叉工具链。**

### Build、Host 与 Target

在构建系统中经常看到：

-   • Build：执行编译过程的平台；
    
-   • Host：编译产生的工具将运行的平台；
    
-   • Target：最终程序所面向的平台。
    

对于普通嵌入式应用交叉编译，可以简单理解为：

```
Build Host：x86_64 PC
Target：ARM 开发板
```

### 工具链组成

典型 AArch64 工具链包含：

```
aarch64-linux-gnu-gcc
aarch64-linux-gnu-g++
aarch64-linux-gnu-ld
aarch64-linux-gnu-as
aarch64-linux-gnu-objcopy
aarch64-linux-gnu-strip
aarch64-linux-gnu-gdb
```

### ABI 与浮点兼容

32 位 ARM 常见 ABI 差异包括：

```
arm-linux-gnueabi
arm-linux-gnueabihf
```

`gnueabihf` 通常代表 Hard Float ABI。

**应用程序、运行库和动态链接器必须使用兼容 ABI。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/54cee937f9d5e9b94c6b2bb93ba5599c_MD5.jpg]]

### 10.2 手动构建 ARM Linux 系统

### 编译 U-Boot

```
make ARCH=arm \
     CROSS_COMPILE=arm-linux-gnueabihf- \
     board_defconfig

make ARCH=arm \
     CROSS_COMPILE=arm-linux-gnueabihf- \
     -j$(nproc)
```

### 编译 ARM64 Linux 内核

```
make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     board_defconfig

make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     -j$(nproc) \
     Image dtbs modules
```

### 安装模块

```
make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     INSTALL_MOD_PATH=$PWD/rootfs \
     modules_install
```

### 编译 BusyBox

```
make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     menuconfig

make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     -j$(nproc)

make ARCH=arm64 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     CONFIG_PREFIX=$PWD/rootfs \
     install
```

### 编译应用程序

```
aarch64-linux-gnu-gcc \
    main.c \
    -o application
```

然后把程序以及依赖库复制到根文件系统。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f23dba2a0a0b1e318f256d4d592da9d1_MD5.jpg]]

### 10.3 自动化构建系统

### Buildroot

Buildroot 可以自动构建：

-   • 交叉工具链；
    
-   • Bootloader；
    
-   • Linux 内核；
    
-   • 根文件系统；
    
-   • 用户软件包；
    
-   • 最终镜像。
    

**Buildroot 官方手册将其定位为用于简化和自动化嵌入式 Linux 系统交叉构建的工具，并能够生成工具链、根文件系统、内核与 Bootloader。**

Buildroot 的特点包括：

-   • 入门相对简单；
    
-   • 配置界面类似内核；
    
-   • 适合单一产品和中小型系统；
    
-   • 构建结果直观；
    
-   • 不以生成完整二进制发行版包仓库为主要目标。
    

### Yocto Project

Yocto Project 建立在 OpenEmbedded Build System 之上，通过 Layer、Recipe 和 Metadata 构建定制发行版。

它更适合：

-   • 多产品线；
    
-   • 多硬件平台；
    
-   • 复杂包依赖；
    
-   • 长期维护；
    
-   • 软件许可证管理；
    
-   • 可复现构建；
    
-   • SDK 生成；
    
-   • 二进制包管理。
    

**Yocto 官方将其描述为一套帮助开发人员为不同硬件架构构建定制 Linux 系统的工具和协作技术体系，其核心构建流程使用 OpenEmbedded Build System。**

### 厂商 SDK

SoC 厂商 SDK 经常包含：

```
U-Boot
Linux Kernel
设备树
驱动补丁
交叉工具链
Buildroot/Yocto
烧录工具
多媒体库
NPU/GPU SDK
示例程序
```

使用厂商 SDK 可以更快让硬件工作，但是也要避免过度依赖未经整理的大量私有补丁。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/b2c62a9c62ef54195062650f5f0b700b_MD5.jpg]]

### 10.4 系统镜像制作与分区设计

典型存储布局：

```
┌──────────────────────┐
│ Bootloader / SPL     │
├──────────────────────┤
│ Environment          │
├──────────────────────┤
│ Boot / Kernel / DTB  │
├──────────────────────┤
│ RootFS A             │
├──────────────────────┤
│ RootFS B             │
├──────────────────────┤
│ Recovery             │
├──────────────────────┤
│ Data                 │
└──────────────────────┘
```

### Boot 分区

保存：

-   • 内核；
    
-   • DTB；
    
-   • Initramfs；
    
-   • FIT；
    
-   • 启动配置。
    

### RootFS

保存系统运行环境。

### Data

保存配置、日志、数据库和用户数据。

### Recovery

保存独立救援系统，用于恢复主系统。

### A/B 分区

准备两套系统：

```
Slot A
Slot B
```

升级时写入当前未运行的 Slot。验证成功后再切换启动标志。

**这样可以避免升级中断造成系统完全无法启动。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0119e02a1c9db45d3122e161ceef795b_MD5.jpg]]

### 10.5 系统烧录与部署

### SD 卡镜像

可以使用：

```
dd if=system.img \
   of=/dev/sdX \
   bs=4M \
   conv=fsync
```

**必须确认设备路径，避免覆盖主机硬盘。**

### USB 烧录

Boot ROM 或 U-Boot 可以提供 USB 下载协议。

### Fastboot

可以写入：

```
boot
system
vendor
userdata
```

等分区。

### U-Boot 网络下载

```
tftp ${loadaddr} Image
tftp ${fdt_addr_r} board.dtb
```

适合开发调试。

### eMMC 与 NAND

eMMC 按块设备方式写入。

Raw NAND 则要处理：

-   • 擦除块；
    
-   • 坏块；
    
-   • ECC；
    
-   • UBI Volume。
    

### 量产烧录

量产工具需要具备：

-   • 自动识别；
    
-   • 并行烧录；
    
-   • 镜像校验；
    
-   • 序列号写入；
    
-   • MAC 地址写入；
    
-   • 安全密钥注入；
    
-   • 烧录日志；
    
-   • 失败重试。
    

## 十一、调试、升级与量产

### 11.1 ARM Linux 启动问题的分层排查

### 上电无输出

首先检查：

```
电源
复位
晶振
启动模式
UART 电平
串口波特率
```

**不能一开始就怀疑 Linux 内核，因为此时 CPU 可能连 Boot ROM 都没有正常执行。**

### Bootloader 未启动

检查：

-   • Boot ROM 是否加载 SPL；
    
-   • 镜像偏移；
    
-   • SPL Header；
    
-   • DDR 初始化；
    
-   • 启动介质；
    
-   • 安全启动验证。
    

### 内核卡死

检查：

-   • 内核加载地址；
    
-   • DTB 地址；
    
-   -   `console=`；
-   -   `earlycon`；
-   • GIC；
    
-   • Timer；
    
-   • 内存保留区域；
    
-   • 设备树；
    
-   • 某个驱动 Initcall。
    

### 根文件系统挂载失败

检查：

-   -   `root=`；
-   -   `rootfstype=`；
-   -   `rootwait`；
-   • 存储设备驱动；
    
-   • 文件系统驱动；
    
-   • 分区表；
    
-   • Initramfs。
    

### 应用未运行

检查：

-   • PID 1；
    
-   • 启动脚本；
    
-   • 程序权限；
    
-   • 动态库；
    
-   • 环境变量；
    
-   • 工作目录；
    
-   • 配置文件；
    
-   • 守护进程日志。
    

### 11.2 常用调试工具

### 串口终端

常用工具：

```
minicom
picocom
screen
PuTTY
MobaXterm
```

### GDB 与 GDB Server

目标机运行：

```
gdbserver :1234 ./application
```

主机运行：

```
aarch64-linux-gnu-gdb application
```

连接：

```
target remote board-ip:1234
```

### `strace`

查看应用系统调用：

```
strace -f -tt ./application
```

它能够快速发现：

-   • 文件不存在；
    
-   • 权限错误；
    
-   • Socket 连接失败；
    
-   • IOCTL 返回错误；
    
-   • 线程阻塞。
    

### `perf`

分析 CPU 热点：

```
perf top
perf record -g ./application
perf report
```

### `ftrace`

分析内核函数、调度延迟和中断。

### 系统资源工具

```
top
free -m
vmstat 1
iostat
pidstat
```

### 网络工具

```
tcpdump
ethtool
ss
ip
ping
iperf3
```

### 11.3 系统性能分析

### CPU 占用率

需要区分：

-   • User；
    
-   • System；
    
-   • IRQ；
    
-   • SoftIRQ；
    
-   • I/O Wait；
    
-   • Idle。
    

**CPU 高不一定是应用计算量大，也可能是中断风暴或者频繁系统调用。**

### 内存

重点观察：

-   • 进程 RSS；
    
-   • Page Cache；
    
-   • Slab；
    
-   • CMA；
    
-   • 内存碎片；
    
-   • OOM；
    
-   • 长期泄漏。
    

### 存储

观察：

-   • 顺序吞吐；
    
-   • 随机 IOPS；
    
-   -   `fsync()` 延迟；
-   • 写放大；
    
-   • Flash 寿命；
    
-   • 掉电一致性。
    

### 网络

观察：

-   • 带宽；
    
-   • RTT；
    
-   • 丢包；
    
-   • 重传；
    
-   • SoftIRQ；
    
-   • 网卡中断；
    
-   • Socket Buffer。
    

### 中断负载

```
watch -n 1 cat /proc/interrupts
```

**如果某个 IRQ 计数高速增长，需要判断设备是否出现中断风暴。**

### 11.4 OTA 升级机制

### 整包升级

直接替换完整系统镜像。

优点是实现简单、结果一致，缺点是下载量较大。

### 差分升级

只传输旧版本与新版本之间的差异。

优点是节省流量，缺点是升级链路和版本管理更加复杂。

### A/B 升级

典型流程：

```
当前运行 Slot A
       ↓
下载更新到 Slot B
       ↓
校验签名和 Hash
       ↓
设置下次启动 Slot B
       ↓
重启并试运行
       ↓
应用上报启动成功
       ↓
确认 Slot B
```

**如果启动失败，Bootloader 自动回退到 Slot A。**

### Bootloader、内核和 RootFS 升级

**Bootloader 升级风险最高，因为 Bootloader 损坏以后，设备可能失去普通恢复能力。**

应当尽量：

-   • 保留不可修改 Boot ROM 恢复入口；
    
-   • 使用冗余 Bootloader；
    
-   • 分阶段升级；
    
-   • 写后校验；
    
-   • 避免在低电量时升级。
    

### 签名和完整性

升级包应当包含：

-   • 版本；
    
-   • 目标硬件型号；
    
-   • Hash；
    
-   • 数字签名；
    
-   • 防回滚版本号；
    
-   • 分区布局信息。
    

**不能只使用 CRC 作为安全验证，因为 CRC 只能检测随机错误，不能防止恶意篡改。**

### 11.5 量产系统的可靠性设计

### 只读根文件系统

把系统程序放在 SquashFS 或只读 ext4 中，可以降低文件系统被破坏的概率。

可写数据放到独立 Data 分区。

### 掉电保护

需要考虑：

-   • 数据库事务；
    
-   • 文件原子更新；
    
-   -   `fsync()`；
-   • 电容保持时间；
    
-   • 掉电检测 GPIO；
    
-   • 日志式文件系统；
    
-   • 双备份配置。
    

### 看门狗

**硬件看门狗应当由真正能够判断系统健康状态的服务喂狗，而不是任何线程无条件定时写寄存器。**

### 日志限额

可以使用：

-   • 日志轮转；
    
-   • 总容量上限；
    
-   • 按时间清理；
    
-   • 内存日志；
    
-   • 关键日志单独持久化。
    

### 安全启动

**安全启动建立信任链：**

```
Boot ROM 公钥根
      ↓
验证 SPL
      ↓
SPL 验证 U-Boot
      ↓
U-Boot 验证 Kernel/FIT
      ↓
内核验证 RootFS 或应用
```

### 固件签名和加密

**签名用于验证来源和完整性。**

**加密用于保护固件内容。**

**二者作用不同，不能用加密替代签名。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/aed0313bbb4f128039953b18e39cdc32_MD5.jpg]]

## 结语：建立 ARM Linux 的完整系统思维

通过对嵌入式 ARM Linux 系统进行层层分析，可以看到，**一个应用程序能够正常运行，绝对不是应用程序自身单独完成的结果。**

从最底层来看，**电源管理芯片需要按照正确顺序给 SoC 和 DDR 供电，晶振需要产生稳定时钟，复位电路需要在合适时间释放 CPU。**

CPU 解除复位以后，芯片内部 Boot ROM 根据启动模式寻找 SPL。**SPL 在容量有限的片内 SRAM 中完成 DDR 初始化，再把完整 U-Boot 加载到 DDR。**

**U-Boot 紧接着加载 Linux 内核、设备树以及可选的 Initramfs，并通过内核命令行告诉 Linux 控制台和根文件系统位于什么位置。**

**Linux 内核启动以后，会建立页表、初始化内存、调度器、中断、时钟、文件系统以及设备模型。设备树负责告诉内核开发板上具备哪些硬件，驱动则根据设备树资源初始化寄存器、中断、GPIO、时钟、复位、电源和 DMA。**

**当内核挂载根文件系统以后，PID 1 开始运行，系统进入用户空间。初始化系统继续挂载 `/proc`、`/sys` 和 `/dev`，配置网络，启动日志、SSH、Watchdog 以及产品应用程序。**

整条启动链路可以重新概括成为：

```
硬件上电
   ↓
Boot ROM
   ↓
SPL 初始化 DDR
   ↓
U-Boot 加载系统
   ↓
Linux 内核管理资源
   ↓
设备树描述硬件
   ↓
驱动控制设备
   ↓
根文件系统提供用户环境
   ↓
PID 1 启动服务
   ↓
应用程序实现业务
```

因此，**学习嵌入式 Linux 时，应当避免只学习某一个孤立部分。**

只会应用开发，遇到驱动、设备树和启动问题时就难以继续向下分析。

只会编写驱动，却不了解根文件系统和应用架构，也很难把驱动真正集成进完整产品。

只会使用 U-Boot 命令，却不了解 DDR、内核以及分区设计，也难以解决复杂启动故障。

更加合理的学习顺序可以是：

```
第一步：理解 ARM 硬件和启动过程
第二步：掌握 U-Boot 与内核启动参数
第三步：学会配置和编译 Linux 内核
第四步：理解设备树和驱动匹配
第五步：掌握常见总线与设备驱动
第六步：构建 BusyBox 根文件系统
第七步：学习应用开发和系统服务
第八步：使用 Buildroot 或 Yocto 自动构建
第九步：研究 OTA、安全启动和量产可靠性
```

最终还需要从开发板实验逐步走向真实产品。

**开发板能够启动，只能说明基本软件链路已经打通。真正能够交付的产品，还必须处理电源波动、Flash 寿命、意外掉电、硬件差异、升级回滚、长期运行、日志管理、安全攻击以及批量生产等问题。**

**当我们能够把硬件、Bootloader、内核、设备树、驱动、根文件系统、系统服务以及应用程序连接成为一个完整整体的时候，就不再只是会使用几个 Linux 命令或者编写某一个驱动函数，而是开始具备真正的嵌入式 ARM Linux 系统开发能力。**

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/1058f24c_1786112710902?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247491452%26idx%3D1%26sn%3Dd059cfc6bc115a4accc95f02cd3260aa%26chksm%3Dc33bd555ab74fd850abc0666838945480d85ee09ef82e645a41cea3f138614ef775952592ca2%26mpshare%3D1%26scene%3D1%26srcid%3D08079j9xSd3iv7fesgZBXWej%26sharer_shareinfo%3Dfa6824a5ab35ce130b2f7c8b951543a0%26sharer_shareinfo_first%3Dfa6824a5ab35ce130b2f7c8b951543a0%23rd&s=obsidian)