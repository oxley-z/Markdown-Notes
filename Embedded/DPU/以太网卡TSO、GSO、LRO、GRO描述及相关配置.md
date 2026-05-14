# 以太网卡TSO、GSO、LRO、GRO描述及相关配置

## 硬件包拆分与合并
### TSO（TCP Segmentation Offload ）

TSO针对TCP在发送端网卡上的拆包

### UFO（UDP Fragmentation Offload ）

已弃用UFO [segmentation-offloads](https://www.kernel.org/doc/Documentation/networking/segmentation-offloads.txt)

![image-20221205151526276](image/Bluefiled2配置TSO、GSO/image-20221205151526276.png)

UFO针对UDP在发送端网卡上的拆包

### LRO（Large Receive Offload ）

LRO针对TCP在接收端网卡的组包

TSO、UFO、LRO基于网卡特性，可在网卡上进行包的合并与拆分，减轻CPU的负荷。

## 软件包拆分与合并

### GSO（Generic Segmentation Offload）

软件实现包拆分，若网卡不支持分片、重组offload能力（如TSO、UFO、LRO）的情况下，GSO推迟数据分片直至数据发送到网卡驱动之前进行分片后再发往网卡。

拆包过程：首先查询网卡是否支持 TSO 功能，如果硬件支持 TSO 则使用网卡的硬件分片能力执行分片；如果网卡不支持 TSO 功能，则将分片的执行，延缓到了将数据推送到网卡的前一刻执行。

| TSO  | GSO  | 分片阶段          |
| ---- | ---- | ----------------- |
| off  | on   | GSO、网卡驱动阶段 |
| on   | on   | TSO、网卡硬件阶段 |

### GRO（Generic Receive Offload）

软件实现包合并，GRO将大量的小报文合并为少量的大报文，再将合并后的大报文提交给OS协议栈处理。

同时GSO、GRO不仅支持TCP和UDP包，还可支持vxlan和gre。

| LRO  | GRO  | 阶段         |
| ---- | ---- | ------------ |
| off  | off  | TCP阶段      |
| on   | off  | 网卡硬件阶段 |
| off  | on   | 网卡驱动阶段 |

**GRO**（**G**eneric **R**eceive **O**ffloading）是 **LRO**（**L**arge **R**eceive **O**ffload，多数是在 NIC 上实现的一种硬件优化机制）的一种软件实现，从而能让所有 NIC 都支持这个功能。网络上大部分 MTU 都是 1500 字节，开启 Jumbo Frame 后能到 9000 字节，如果发送的数据超过 MTU 就需要切割成多个数据包。通过合并「足够类似」的包来减少传送给网络协议栈的包数，有助于减少 CPU 的使用量。GRO 使协议层只需处理一个 header，而将包含大量数据的整个大包送到用户程序。如果用 `tcpdump` 抓包看到机器收到了不现实的、非常大的包，这很可能是系统开启了 GRO。

GRO 和「硬中断合并」的思想类似，不过阶段不同。「硬中断合并」是在中断发起之前，而 GRO 已经在处理软中断中了。

**napi_gro_receive** 就是在收到数据包的时候合并多个数据包用的，如果收到的数据包需要被合并，**napi_gro_receive** 会很快返回。当合并完成后会调用 **napi_skb_finish** ，将因为数据包合并而不再用到的数据结构释放。最终会调用到 **netif_receive_skb** 将数据包交到上层网络栈继续处理。**netif_receive_skb** 就是数据包从 Ring Buffer 出来后到上层网络栈的入口。

## Bluefiled2网卡配置TSO、GSO、LRO、GRO

安装`ethtool`工具

```bash
sudo apt install ethtool
```

查询TSO

```bash
sudo ethtool -k eth<x>
```

eg

```bash
sudo ethtool -k enps4f0np0
```

设置TSO

```bash
sudo ethtool -K eth<x> [rx on|off] [tx on|off] [sg on|off] [tso on|off] [lro on|off]
```

eg

```bash
sudo ethtool -K enps4f0np0 tso on
```

`注意`：ethtool命令的相关配置不可持续，在计算机下电后会被清除；

# 参考

[Ethernet Driver Usage and Configuration - MLNX_EN v5.8-1.0.1.1 LTS - NVIDIA Networking Docs](https://docs.nvidia.com/networking/display/MLNXENv581011/Ethernet+Driver+Usage+and+Configuration) 迈络思网卡配置手册。

[关于网卡特性TSO、UFO、GSO、LRO、GRO_51CTO博客_网卡的特性](https://blog.51cto.com/u_15333820/3459534)

[网卡TSO、UFO、GSO、LRO、GRO和RSS介绍_小雨的网络文档的技术博客_51CTO博客](https://blog.51cto.com/shutdownsky/1441761)

[SmartNIC — TSO、GSO、LRO、GRO 技术_范桂飓的博客-CSDN博客](https://blog.csdn.net/Jmilk/article/details/118858113)

[TSO和GSO_测试里的小虾米的技术博客_51CTO博客](https://blog.51cto.com/u_14009921/5757723)

[TSO、UFO、GSO、LRO、GRO和RSS介绍-飞鸿无痕-ChinaUnix博客](http://blog.chinaunix.net/uid-20639775-id-3529535.html)

[网络虚拟化中的 offload 技术：LSO/LRO、GSO/GRO、TSO/UFO、VXLAN_yeasy的博客-CSDN博客](https://blog.csdn.net/yeasy/article/details/19204639)

[https://www.kernel.org/doc/Documentation/networking/segmentation-offloads.txt](https://www.kernel.org/doc/Documentation/networking/segmentation-offloads.txt) Linux内核offload的描述

[Linux GRO流程分析_gro linux_zhenghuaduo的博客-CSDN博客](https://blog.csdn.net/zgy666/article/details/106989856) 内核GRO流程分析。
