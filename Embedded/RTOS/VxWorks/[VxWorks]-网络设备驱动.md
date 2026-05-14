# VxWorks网络设备驱动

```c
usrRoot
    sockLibInit
    usrNetworkInit
	    ...
    	endLibInit
    	...
	    usrNetEndLibInit
    		...
    		muxDevLoad
    			xxxEndLoad
    		muxDevStart
    			xxxEndStart
```





















参考

[VxWorks网络设备的加载及协议栈初始化](https://www.vxworks.net/bsp/29-vxworks-network-device-loading-and-protocol-initialization)

[VxWorks下END网络驱动编写概述](https://www.vxworks.net/bsp/87-general-description-of-end-network-device-driver-design-in-vxworks)