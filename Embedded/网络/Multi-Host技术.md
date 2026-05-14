# Multi-Host技术

Multi-Host技术是一种可扩展的计算机系统设计，它允许多个主机（或服务器）共享同一组计算资源，从而实现更高的计算性能和更高的系统可用性。这种技术通常用于数据中心、云计算和高性能计算等领域。

在一个Multi-Host系统中，多个主机可以共享同一组计算资源，如存储、网络连接和处理器等。这些主机可以通过高速互联总线或网络相互连接，并通过软件来管理和分配计算资源。多个主机共享计算资源的好处是可以提高系统可扩展性和可用性，同时也可以实现更高的计算性能。

通常，Multi-Host系统采用虚拟化技术来管理和分配计算资源。虚拟化技术可以将物理计算资源抽象成多个虚拟计算资源，并将它们分配给不同的主机使用。这样，每个主机就可以访问和使用它所需的计算资源，并且可以在需要时动态地增加或减少这些资源。



![Multi-host演示](image/Multi-Host技术/Multi-host演示.gif)



# Multi-Host资料

## Intel FM10K Multi-Host资料

[Intel Ethernet Multi Host Controller Communication - YouTube](https://www.youtube.com/watch?v=yOgD5dmBogo) Intel Multi-Host讲解

[Adapter User Guide for Intel® Ethernet Adapters](https://www.intel.com/content/www/us/en/download/19373/adapter-user-guide-for-intel-ethernet-adapters.html) FM10000 用户手册

[Driver for Intel® Ethernet Multi-host Controller FM10000 Series](https://www.intel.com/content/www/us/en/download/19047/driver-for-intel-ethernet-multi-host-controller-fm10000-series.html) FM10000 驱动

[Muti-host和MR-IOV-lvyilong316-ChinaUnix博客](http://m.blog.chinaunix.net/uid-28541347-id-5850920.html) Multi-Host与MR-IOV

Linux FM10K driver [torvalds/linux/drivers/net/ethernet/intel/fm10k]([linux/drivers/net/ethernet/intel/fm10k at v5.4 · torvalds/linux (github.com)](https://github.com/torvalds/linux/tree/v5.4/drivers/net/ethernet/intel/fm10k/))

[各种各样的Multi-Host - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/153428851)

# 参考

[各种各样的Multi-Host - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/153428851)

[黑科技：Mellanox Multi-Host技术打通数据中心任督二脉-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/205847)

[Microchip’s New PCIe 4.0 PCIe Switches: 100 lanes, 174 GBps --- Microchip的新型PCIe 4.0 PCIe交换机：100通道，174 GBps (anandtech.com)](https://www.anandtech.com/show/15821/microchips-new-pcie-40-pcie-switches-100-lanes-174-gbps)

[Muti-host和MR-IOV-lvyilong316-ChinaUnix博客](http://linuxsystem.blog.chinaunix.net/uid-28541347-id-5850920.html)



