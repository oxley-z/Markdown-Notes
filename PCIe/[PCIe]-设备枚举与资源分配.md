# PCIe设备枚举与资源分配

## Linux内核

`Linux kernel`支持两种PCI扫描方式：`ACPI`的方式和`非ACPI`的方式(Legacy mode)。系统默认使用ACPI的方式枚举PCIE/PCI设备。如果用户想使用非ACPI的方式枚举PCIE/PCI设备，可以通过在kernel 启动参数中加入`pci=noacpi`。

### 枚举步骤

PCI枚举是个不断递归调用发现新设备的过程，PCI枚举简单来说主要包括下面几个步骤：

* 从Bus 0开始，利用深度优先算法（DFS）遍历整个PCI设备树。从Root Complex出发，寻找设备和桥。；
* 递归的过程中通过读取BARs，计算所需的空间大小，记录所有MMIO和IO的需求情况并予以满足，由软件分配一段未使用的地址空间，并将其写入BAR。
* 设置其他参数（必要的Capabilities等）；
* 处理桥设备：
  * 如果发现设备是桥（Header Type = 1），则为其下游分配一个新的总线号。
  * 将该总线号写入桥的`Secondary Bus Number`寄存器。
  * 递归枚举：将这个新总线作为当前总线，重复扫描步骤，扫描下游设备。
  * 扫描完该分支后，将最后分配的总线号写入桥的`Subordinate Bus Number`寄存器。

* 当所有总线都被扫描后，整个PCIe树被构建，所有设备资源分配完毕。

上述过程结束后，一颗完整的资源分配树建立完成。

详细设备枚举过程一般分为以下几步

1. **设备扫描**：当Host主机启动时，PCIe主机控制器开始对PCIe总线进行扫描，以识别连接到PCIe总线上的所有设备；
2. **设备检测**：如果检测到一个设备，主机控制器会为该设备分配一个唯一的设备号并初始化该设备；
3. **设备初始化**：如果监测到一个设备，主机控制器会为该设备分配唯一的设备号，并初始化该设备，其中可能包括对某些寄存器的初始化以及为设备分配资源，如内存地址以及中断。
4. **功能级枚举**：如果PCIe设备支持多个功能，系统将为每个功能分配唯一的功能号，每个功能号都有自己的一组相关寄存器。
5. **设备空间访问**：系统通过配置事务（Configuration Transactions）访问PCIe设备的配置空间。配置空间包含了设备的重要信息，如设备ID、厂商ID、BAR（Base Address Registers）、中断信息等。
6. **BAR分配**：基地址寄存器（Base Address Registers，BARs）用于告诉系统如何寻址设备的内存和I/O资源。系统将分配适当的地址范围给BAR，确保设备能够在地址空间中正确寻址其资源。
7. **中断分配**： 系统可能会为设备分配中断向量，以便设备能够通过中断通知主机系统关于特定事件的发生。
8. **驱动加载**： 一旦PCIe设备被枚举和初始化，系统将加载适当的驱动程序，以便与设备进行通信和控制。

### ACPI方式枚举PCIe/PCI设备

```c
acpi_init
	-->acpi_scan_init
	    -->acpi_pci_root_init
    		-->acpi_scan_add_handler_with_hotplug(&pci_root_handler, "pci_root");
				-->struct acpi_scan_handler pci_root_handler
                    -->acpi_pci_root_add
                    	-->pci_acpi_scan_root
                    		-->acpi_pci_root_create
                    			-->pci_create_root_bus
                    			-->pci_scan_child_bus
    	-->acpi_pci_link_init
```

具体见[\[Linux\]-内核PCIe设备枚举流程分析.md](../Linux/Linux内核学习/[Linux]-内核PCIe设备枚举流程分析.md)

## BootLoader中对PCI/PCIe设备的枚举

### UEFI对PCI/PCIe的支持

UEFI对于PCI总线的支持包括：

1. 提供分配PCI资源的协议；
2. 提供访问PCI设备的协议；
3. 提供PCI枚举器，枚举PCI总线上的设备以及分配设备所需的资源；
4. 提供各种Lib，方便驱动程序访问PCI/PCIe配置空间或者MMIO/IO空间。

### UBOOT对PCI/CPIe设备的枚举

详见[\[uboot\]-PCI设备枚举及初始化.md](../uboot/[uboot]-PCI设备枚举及初始化.md)

```c
pci_uclass_post_probe
    -->pci_bind_bus_devices	/* 绑定指定 PCI 总线上的设备。遍历总线上的设备，读取其配置空间信息，并根据设备树或驱动程序匹配绑定设备，同时更新设备的父平台数据。 */
    -->pci_auto_config_devices
    	-->pciauto_config_init	/* 在所有可用的 PCI 资源区域中，分别找出 I/O、内存、可预取内存三类中最大的那一块，并保存指针，方便后续分配。 */
       	-->dm_pciauto_config_device	/* 配置一个 PCI 设备以使其可以正常使用。如果该设备是桥设备，还会探测其下游设备。 */
    		-->dm_pciauto_setup_device	/* 为指定的 PCI 设备根据类型和资源区域（内存、预取内存或 I/O）分配bar地址空间。该函数还会设置设备的命令状态寄存器以启用必要的功能，并处理特定设备（如 VGA）的特殊配置需求。 */
```

[uboot PCI相关命令](../uboot/[uboot]-uboot命令.md#pci命令)

# 参考

[PCIe学习笔记——4.设备的枚举过程](https://zhuanlan.zhihu.com/p/4993687465)

[PCIe总线-Linux内核PCIe设备枚举流程分析（十三）](https://blog.csdn.net/u011037593/article/details/142031554)

[PCIe枚举过程](https://zhuanlan.zhihu.com/p/4552982975)
