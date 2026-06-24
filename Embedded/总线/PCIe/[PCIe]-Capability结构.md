# PCIe Capability结构

## Capability结构

### Power Management-电源管理能力（0x01）

该结构是所有PCIe设备Func所必须的，该功能在 [PCI Bus Power Management Interface Specification  Revision 1.2](https://lekensteyn.nl/files/docs/PCI_Power_Management_12.pdf) 有明确定义。对于PCIe来说，其结构与传统PCI相同。

#### 电源管理状态

电源管理状态被定义为不同的、可区分的功耗节省级别。这些状态通过特定的状态编号来表示。按照惯例，PCI总线的电源状态以“B”为前缀，后面跟着电源管理的编号（0~3），编号越大，所实现的功耗节省效果越明显。

### MSI（Message Signaled Interrupts）-消息信号中断能力（0x05）







## Extended Capabilities结构



