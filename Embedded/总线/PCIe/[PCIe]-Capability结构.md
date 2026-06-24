# PCIe Capability结构

## Capability结构

### Power Management-电源管理能力（0x01）

该结构是所有PCIe设备Func所必须的，该功能在 [PCI Bus Power Management Interface Specification  Revision 1.2](https://lekensteyn.nl/files/docs/PCI_Power_Management_12.pdf) 有明确定义。对于PCIe来说，其结构与传统PCI相同。

#### 电源管理状态

电源管理状态被定义为不同的、可区分的功耗节省级别。这些状态通过特定的状态编号来表示。按照惯例，PCI总线的电源状态以“B”为前缀，后面跟着电源管理的编号（0~3），编号越大，所实现的功耗节省效果越明显。同样，对于PCI Func来说，其电源管理状态以“D”作为前缀，后面跟着相同的状态编号（0~3）。电源管理状态的编号越大，实现的功耗节省效果也越明显。

#### PCI Func状态

在电源管理系统中，每个PCI Func最多可定义四种电源状态，这些状态包括 D0 到 D3，其中D0为最大功率状态，而D3为最小功率状态。D1和D2则是介于D0（开启）和D3（关闭）之间的中间功率节省状态。

虽然这些电源状态的概念对于系统中的所有Func都是普遍适用的，但转换到给定电源管理状态时的含义或预期功能行为取决于Func的类型（或类别）。

### MSI（Message Signaled Interrupts）-消息信号中断能力（0x05）







## Extended Capabilities结构



