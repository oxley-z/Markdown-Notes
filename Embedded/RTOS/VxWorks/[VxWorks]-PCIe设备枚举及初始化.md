# VxWorks系统下PCIe设备枚举及初始化

PCI设备枚举重要函数调用过程如下：

```c
/* vxworks-6.9\target\src\hwif\busCtlr\ppc440gpPci.c */
ppc440gpPciInstInit
    vxbBusAnnounce
    pciConfigLibInit
    sysPciHostBridgeInit
    ppc440gpPciAutoConfig
    	pciAutoConfigLibInit
    	...	/* 自动分配前的构建好 pCookie 中的策略/资源池/回调表 */
    	/* vxworks-6.9\target\src\drv\pci\pciAutoConfigLib.c */
    	pciAutoCfg
    		pciAutoCfgFunc
    			pciAutoListCreate
    				pciAutoDevProbe
    					pciAutoBusProbe
    			pciAutoFuncConfigAll
    				pciAutoDevConfig
    					pciAutoFuncConfig
    						pciAutoFuncDisable
    							pciAutoRegConfig
    					case ((PCI_CLASS_BRIDGE_CTLR << 8) + PCI_SUBCLASS_CARDBUS_BRIDGE):
    					pciAutoCardBusConfig
    					case (( PCI_CLASS_BRIDGE_CTLR << 8) + PCI_SUBCLASS_P2P_BRIDGE ):
    					pciAutoBusConfig
    				pciAutoFuncEnable
    vxbPciBusTypeInit
```

* vxbBusAnnounce：调用 `vxbBusAnnounce(pDev, VXB_BUSID_PCI)` 的作用是向 vxBus 子系统“声明/注册”当前设备实例为一个 PCI 总线控制器（bus controller），让 vxBus 把它当作父级总线来管理并为其执行后续的总线初始化与设备枚举/绑定流程。

* pciConfigLibInit：初始化 PCI 基础库。
* sysPciHostBridgeInit：初始化 PCI 桥，使其既可以作为 PCI 主设备也可以作为从设备运行。，配置inbound、outbound等操作。
* ppc440gpPciAutoConfig：硬件资源分配，计算并写入 BAR、IRQ 等配置空间字段，确保硬件上的地址窗口与中断路由被设置好。
  * pciAutoConfigLibInit：初始化PCI自动初始化库，返回供后续 pciAutoConfigLib 函数使用的 cookie。
  * pciAutoCfg：硬件资源自动分配。
    * pciAutoListCreate：探测所有PCI Func并创建 PCI 探测列表。
      * pciAutoDevProbe：探测单个 PCI 总线并添加每个检测到的 PCI Func添加到Func列表。
        * pciAutoBusProbe：如果找到 PCI-PCI 桥接器，pciAutoBusProbe()被调用来探测该总线。pciAutoBusProbe() 依次调用该函数以递归方式直到所有总线都被探测到。
    * pciAutoFuncConfigAll：初始化指定范围内的所有 PCI Func。
      * pciAutoDevConfig：为 PCI Func分配内存和 I/O 空间。
        * pciAutoFuncConfig：将内存和/或 I/O 空间分配给单个Func。
          * pciAutoFuncDisable：对于单个 PCI Func清除 I/O、mem、master 和 ROM 空间使能位。
            * pciAutoRegConfig：将 PCI 空间分配给单个 PCI 基址寄存器。
        * case ((PCI_CLASS_BRIDGE_CTLR << 8) + PCI_SUBCLASS_CARDBUS_BRIDGE):
          pciAutoCardBusConfig：为单个 PCI-Cardbus 桥设置内存和 I/O 寄存器。
        * case (( PCI_CLASS_BRIDGE_CTLR << 8) + PCI_SUBCLASS_P2P_BRIDGE ):
          pciAutoBusConfig：为单个 PCI-PCI 桥设置内存和 I/O 寄存器。
      * pciAutoFuncEnable：执行最终配置并启用Func。

* vxbPciBusTypeInit：进入 vxBus 的 PCI 总线类型初始化,遍历PCI总线，将硬件状态采集到 vxBus 的软件数据结构并启动驱动层。

