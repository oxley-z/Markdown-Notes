---
github.io: true
csdn: false
---
# rxe_cfg

rxe_cfg 是一个用于配置和管理 RXE（RDMA over Converged Ethernet）实例的命令行工具。

## 概述

```bash
$ rxe_cfg [status]					显示可用的以太网设备和配置的 RXE 实例的信息。
$ rxe_cfg start [-p proto]			加载 RXE 模块并配置任何持久实例。
$ rxe_cfg stop						取消配置所有的 RXE 实例并尝试卸载内核模块。
$ rxe_cfg persistent				显示已持久配置的以太网设备列表。
$ rxe_cfg add [-n] ethN				在以太网设备 ethN 上配置一个 RXE 实例。
$ rxe_cfg remove [-n] ethN|rxeN		移除指定的 RXE 实例。
$ rxe_cfg crc enable|disable		启用或禁用 RoCE ICRC计算。
$ rxe_cfg mtu [-f] [rxeN] mtu_size	将所有 RXE 设备的 RoCE MTU 设置为 mtu_size。
```

## 常用参数

```bash
proto 
	以太类型字段。默认值为0x8915。必须更改此值才能在Mellanox ConnectX适配器上使用RXE。
ethN   
	/sys/class/net 中列出的网络设备名称。 仅支持以太网设备； IE. VLAN 的eth0或eth0.1234。
rxeN 
	/sys/class/infiniband/ 中列出的RXE设备名称。 例如 rxe0 或 rxe1。
mtu_size
	RoCE mtu。 对于 RoCE，mtu表示不包括标头的有效负载，可能的值有：256、512、1024、2048 和 4096。		
```

## 常用命令

```bash
[status]
	状态命令打印有关可用以太网设备和配置的 RXE 实例的信息表。 如果未提供任何选项，则状态显示为默	 认值。
start [-p proto]
	start命令加载 RXE 模块并配置任何持久实例。 如果包含 -p proto 选项，RXE 模块将配置为使用 	 Ethertype = proto。 （这允许在设备上测试 RXE，例如 Mellanox ConnectX，这些设备已经在硬	件中支持默认的 RoCE 以太网类型。）
stop
	stop 命令取消所有 RXE 实例的配置并尝试卸载内核模块。
persistent
	显示已持久配置的以太网设备列表。
add [-n] ethN
	add 命令将在以太网设备 ethN（例如 eth0）上配置 RXE 实例。
	RXE 模块必须已通过 rxe_cfg start 加载。
	
	默认行为是将 ethN 添加到持久配置文件中，并且下次运行 rxe_cfg start 时将配置相同的 RXE 设	 备。 如果包含 -n 选项，则设备不会添加到持久性文件中。
remove [-n] ethN|rxeN
	删除命令将删除指定的 RXE 实例。 该参数必须与当前活动的 ethN 或 rxeN 名称匹配。
	
	如果包含 -n 选项，RXE 设备将被删除，但不会从持久状态中删除。 因此，下次运行 rxe_cfg start 	 时将重新创建它。
mtu [-f] [rxeN] mtu_size
	mtu 命令会将所有 RXE 设备的 RoCE MTU 设置为 mtu_size，前提是底层以太网 MTU 足够大。 如	果以太网MTU不够大，RXE将使用适合的最大MTU； 驱动程序会记住请求的 RoCE MTU，并且如果以太网 	   MTU 稍后更改为请求的 MTU，则会增加当前活动的 MTU。
	
	如果包含 -f 选项，则基础以太网 MTU 将在必要时增加到最小大小，以适应 RoCE MTU 的大小。
	如果指定了 rxeN 实例，则只有该实例才会受到该命令的影响。
crc enable|disable
	crc 命令将启用或禁用 RoCE ICRC 计算。 与硬件 RoCE NIC 通信需要有效的 ICRC，但当 RXE 实	 例与另一个 RXE 实例通信时，禁用 CRC 时性能会更好。

	此选项对于所有RXE实例都是全局的。
```

## 文件

```bash
[PREFIX]/etc/rxe.conf
	RXE 配置文件。 包含持久 RXE 实例的列表。 可以通过删除此文件来删除所有持久 RXE 实例（请注意，	  这将在下一个“rxe_cfg start”时生效 - 要删除主动配置的实例，您必须“rxe_cfg stop”）。
```

## 配置Soft-RoCE

以 `root` 用户身份，输入以下命令显示 RXE 的当前配置状态：

```bash
$ sudo rxe_cfg 
  Name   Link  Driver  Speed  NMTU  IPv4_addr  RDEV  RMTU  
  ens33  yes   e1000
```

要载入 RXE 内核模块并启动 RXE，以 `root` 用户身份输入：

```bash
$ sudo rxe_cfg start
  Name   Link  Driver  Speed  NMTU  IPv4_addr  RDEV  RMTU          
  ens33  yes   e1000                           rxe0  1024  (3) 
```

查看rdma_rxe驱动

```bash
$ lsmod |grep rdma_rxe
rdma_rxe              118784  0
ip6_udp_tunnel         16384  1 rdma_rxe
udp_tunnel             16384  1 rdma_rxe
ib_uverbs             131072  2 rdma_rxe,rdma_ucm
ib_core               307200  8 rdma_cm,rdma_rxe,rpcrdma,iw_cm,ib_iser,rdma_ucm,ib_uverbs,ib_cm
```

在通过以太网接口添加新 RXE 设备之前，应打开对应的接口并分配有有效的 IP 地址。

添加新的 RXE 设备

```bash
$ sudo rxe_cfg -n add ens33
```

```bash
$ sudo rxe_cfg status
  Name   Link  Driver  Speed  NMTU  IPv4_addr  RDEV  RMTU          
  ens33  yes   e1000                           rxe0  1024  (3)
```

查看设备

```bash
$ ibv_devices 
    device          	   node GUID
    ------          	----------------
    rxe0            	020c29fffea41f73
```

删除 RXE 设备

```bash
$ rxe_cfg remove ens33
```

# 参考

[Ubuntu Manpage: rxe_cfg - rxe configuration tool for RXE (Soft RoCE)](https://manpages.ubuntu.com/manpages/focal/en/man8/rxe_cfg.8.html)

[13.3. 配置 Soft-RoCE Red Hat Enterprise Linux 7 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/networking_guide/sec-configuring_soft-_roce)
