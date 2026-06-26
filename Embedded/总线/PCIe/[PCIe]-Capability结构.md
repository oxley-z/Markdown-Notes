
# Capability结构

## Power Management-电源管理能力（0x01）

该结构是所有PCIe设备Func所必须的，该功能在 [PCI Bus Power Management Interface Specification  Revision 1.2](https://lekensteyn.nl/files/docs/PCI_Power_Management_12.pdf) 有明确定义。对于PCIe来说，其结构与传统PCI相同。

### 电源管理状态

电源管理状态被定义为不同的、可区分的功耗节省级别。这些状态通过特定的状态编号来表示。按照惯例，PCI总线的电源状态以“B”为前缀，后面跟着电源管理的编号（0~3），编号越大，所实现的功耗节省效果越明显。同样，对于PCI Func来说，其电源管理状态以“D”作为前缀，后面跟着相同的状态编号（0~3）。电源管理状态的编号越大，实现的功耗节省效果也越明显。

#### PCI Func状态

在电源管理系统中，每个PCI Func最多可定义四种电源状态，这些状态包括 D0 到 D3，其中D0为最大功率状态，而D3为最小功率状态。D1和D2则是介于D0（开启）和D3（关闭）之间的中间功率节省状态。

虽然这些电源状态的概念对于系统中的所有Func都是普遍适用的，但转换到给定电源管理状态时的含义或预期功能行为取决于Func的类型（或类别）。

##### D0状态

所有Func都必须支持D0状态。D0分为两个不同的子状态：“<mark style="background: #FF5582A6;">未初始化</mark>”（uninitialized）子状态和“<mark style="background: #FF5582A6;">激活</mark>”（Active）子状态。当组件退出常规复位后，该组件的所有Func都进入D0<sub>未初始化状态</sub>。当Func完成FLR复位时，它也进入D0<sub>未初始化状态</sub>。配置完成后，Func进入D0<sub>激活状态</sub>，这是PCIe Func完全可操作的状态。只要该Func的“内存空间使能”（Memory Space Enable）、“I/O 空间使能”（I/O Space Enable）或“总线主控使能”（Bus Master Enable）位中的任意一个或组合被置位（Set），该功能就会进入 D0<sub>活动状态</sub>。

###### D0<sub>uninitialized</sub>状态

正常复位或软件控制PCIe设备从D3<sub>hot</sub>状态迁移至D0状态时，进入D0<sub>uninitialized</sub>；此时PCIe设备相关寄存器恢复到默认状态，在该状态下的特性如下：  

1. 只响应配置相关事务的处理；  
2. 命令寄存器使能位全部恢复到了默认状态，这意味着它不能发起Memory或I/O读写事务。

###### D0<sub>Active</sub>状态

一旦PCIe相关配置寄存器被软件配置和启用，它将处于D0激活状态，并完全运行。

##### D1状态

D1状态的支持是可选的，在D1状态下，Func不得在链路上发起任何请求TLP（PME消息除外）。配置请求和消息请求是处于D1状态的Func所接受的仅有的TLP类型。所有其他接收到的请求必须作为“不支持的请求”（Unsupported Requests）进行处理，而所有接收到的完成包（Completions）则可选择作为“意外完成包”（Unexpected Completions）进行处理。如果在D1状态下检测到由接收到的TLP（例如’不支持的请求“）引起的错误，且启用了错误报告功能，则必须将链路恢复至L0状态（若尚未处于L0状态）并发送错误消息。如果在D1状态下检测到由非接收TLP事件（例如”完成超时“）引起的错误，则必须在Func被重新配置回D0状态时发送错误信息。

##### D2状态

D2状态的支持是可选的，当处于D2状态时，Func不得在链路上发起任何TLP请求（PME消息除外）配置请求和消息请求是Func在D2状态下唯一接受的TLP类型。所有其他接收到的请求必须作为”不支持的请求“（Unsupported Requests）处理，而所有接收到的完成包则可选择作为”意外完成包“（Unexpected Completions）处理。如果在D2状态下检测到由接收到的TLP（例如”不支持的请求“）引起的错误，且已启用错误报告功能，则必须将链路恢复至L0状态（若尚未处于L0状态）并发送错误消息。如果在D2状态下检测到由非接收TLP时间（例如”完成超时“）引起的错误，则必须在Func被重新配置回D0状态时发送错消息。

##### D3状态

D3状态必须支持（同时包括D3<sub>cold</sub>和D3<sub>hot</sub>状态）。该状态时PCIe设备的功耗最小。

如果PMCSR中的No_Soft_Reset字段已被置位，则处于D3<sub>hot</sub>状态的Func必须维护功能上下文。当Func从D3<sub>hot</sub>转换到D0状态后，系统软件无需对其进行重新初始化（该Func将处于D0<sub>active</sub>状态）。如果No_Soft_Reset字段被清除，则Func在D3<sub>hot</sub>状态下无需保持其功能上下文，这并不保证功能上下文一定会被清楚，软件绝不能依赖此行为。因此，在这种情况下，由于功能在转换到 D0 后将处于 D0<sub>uninitialized</sub>状态，系统软件必须对其进行完全的重新初始化。

###### D3<sub>hot</sub>状态

在D3<sub>hot</sub>状态下，Func仅接受配置请求和消息请求这两种TLP。其他收到的请求必须作为”不支持的请求“处理。并且所有接收到的完成报文（Completions）可选择性的作为”非预期完成报文“（Unexpected Completions，UC）进行处理。如果在D3<sub>hot</sub>状态下检测到由接收到的TLP引起的错误，并且已启用错误报告，则链路如果尚不在L0状态，则必须返回到L0状态，并发送一条错误消息。

###### D3<sub>cold</sub>状态

当主电源被移除时，Func将转换到D3<sub>cold</sub>状态。上电顺序及其关联的冷复位（Cold Reset）会使Func从D3<sub>cold</sub>状态转换到D3<sub>uninitialized</sub>状态，并且硬件将为Func恢复上电默认值，这与初始上电时完全相同。此时软件必须对Func进行完全的初始化以重新建立所有的功能上下文，从而完成将Func恢复至D3<sub>active</sub>状态的过程。

#### 总线电源状态

总线的电源管理状态可以通过其在特定时刻的某些属性来表征，例如是否供电、时钟速度以及允许何种类型的总线活动。这些状态被称为 B0、B1、B2 和 B3。

#### 链路电源状态

PCIe协议定义了以下物理层Link状态，这些状态也对应了LTSSM状态，L-state由下游组件的D-state决定，除L3外其他状态的LinkUp都仍为1。

- L0 - Fully Active状态;
- L0s - 低功耗模式，仅支持ASPM方式，硬件自动发起的，软件无法控制，单向进入，如USP有大量数据，DSP没有数据的时候DSP可以独立进入L0s；
- L1 - 低功耗模式，支持PCI-PM和ASPM两种方式；L1子状态可以关闭参考时钟、Tx common mode电路，Rx electric idle detect电路更加省电；
- L2 -可选低功耗模式，仅支持PCI-PM方式，关闭参考时钟、关闭PLL、关闭Main Power, 但是需要保留Aux Power;
- L3 - 可选低功耗模式，仅支持PCI-PM方式，处于所有power都off的状态，非LTSSM状态；

省电顺序：L0<L0s<L1<L2<L3，越省电的状态recovery到L0正常工作状态的时间就会越长。  
L1和L2/3 Ready的进入需要在L0状态下协商进行。

## MSI（Message Signaled Interrupts）-消息信号中断能力（0x05）







# Extended Capabilities结构



