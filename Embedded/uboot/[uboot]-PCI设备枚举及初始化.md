# uboot下PCI/PCIe设备枚举及初始化

```c
board_init_f
/* uboot-2019-07\common\board_r.c */
board_init_r
    initcall_run_list(init_sequence_r)
    	initr_pci
    		/* uboot-2019-07\drivers\pci\pci-uclass.c */
    		pci_init
    			pci_uclass_pre_probe
    			pci_uclass_post_probe
    				pci_bind_bus_devices
    					pci_find_and_bind_driver
    				pci_auto_config_devices
    					/* uboot-2019-07\drivers\pci\pci_auto_common.c */
    					pciauto_config_init
    					/* uboot-2019-07\drivers\pci\pci_auto.c */
    					dm_pciauto_config_device
    						case PCI_CLASS_BRIDGE_PCI:
								dm_pciauto_setup_device
                            	dm_pci_hose_probe_bus
                                    dm_pciauto_prescan_setup_bridge
                            case PCI_CLASS_BRIDGE_CARDBUS:
								dm_pciauto_setup_device
                            default:
								dm_pciauto_setup_device
```

* board_init_r：启动阶段的框架函数，负责进入 U-Boot 的初始化序列。

* initcall_run_list(init_sequence_r)：执行回调函数，最终触发各子系统的初始化。

* initr_pci：pci子系统初始化。

* pci_init：初始化 pci-uclass。

* pci_uclass_pre_probe：探测pci设备前的准备工作（例如为控制器设置初始区域）。

* pci_uclass_post_probe：探测pci设备后的配置工作（例如记录已分配区域、打印信息）。

* pci_bind_bus_devices：遍历总线上的 PCI 设备节点，为每个发现的设备调用绑定函数（把驱动和设备实例关联到 udevice 数据结构上）。

* pci_find_and_bind_driver：对单个 PCI 设备，查找合适的驱动并把该设备绑定成一个 udevice（建立设备数据结构、初始化 platdata 等

* pci_auto_config_devices：启动自动配置流程：初始化自动分配器（pciauto_config_init），然后对设备执行自动分配/配置（例如分配 BAR、设置桥的窗口等）。

* pciauto_config_init：初始化用于自动分配的区域描述（pci_region），收集平台/控制器提供的资源窗口，建立 allocator 的初始状态（供后续分配使用）。

* dm_pciauto_config_device：对单个设备进行自动配置的控制函数：

* case PCI_CLASS_BRIDGE_PCI：当遇到桥（bridge）时，递归扫描其从属总线：
		dm_pciauto_setup_device
		dm_pci_hose_probe_bus

* dm_pciauto_prescan_setup_bridge：在进入下级总线枚举前，为下级设备预留并暴露合适的地址窗口，确保后续对下级设备的 BAR 分配能在这些窗口内进行，从而保证地址分配正确与连贯。

* case PCI_CLASS_BRIDGE_CARDBUS：
  	dm_pciauto_setup_device

* default：真正对设备的 BAR/ROM 等配置空间进行操作：
  	dm_pciauto_setup_device



# 参考

[Rockchip_Developer_Guide_UBoot_Nextdev_CN.pdf](https://download.t-firefly.com/product/Board/RK356X/Document/Developer/Rockchip_Developer_Guide_UBoot_Nextdev_CN.pdf)
