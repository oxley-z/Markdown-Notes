---
github.io: true
csdn: false
---
# ethtool

## 常用参数

```bash
-a 查看网卡中 接收模块RX、发送模块TX和Autonegotiate模块的状态：启动on 或 停用off。
-A 修改网卡中 接收模块RX、发送模块TX和Autonegotiate模块的状态：启动on 或 停用off。
-c display the Coalesce information of the specified ethernet card 查看硬中断合并策略。
-C Change the Coalesce setting of the specified ethernet card 修改硬中断合并策略。
-g Display the rx/tx ring parameter information of the specified ethernet card 查看RingBuffer的大小。
-G change the rx/tx ring setting of the specified ethernet card 修改RingBuffer的大小。
-i 显示网卡驱动的信息，如驱动的名称、版本等。
-d 显示register dump信息, 部分网卡驱动不支持该选项。
-e 显示EEPROM dump信息，部分网卡驱动不支持该选项。
-E 修改网卡EEPROM byte。
-k 显示网卡Offload参数的状态：on 或 off，包括rx-checksumming、tx-checksumming等。
-K 修改网卡Offload参数的状态。
-p 用于区别不同ethX对应网卡的物理位置，常用的方法是使网卡port上的led不断的闪；N指示了网卡闪的持续时间，以秒为单位。
-r 如果auto-negotiation模块的状态为on，则restarts auto-negotiation。
-S 显示NIC- and driver-specific 的统计参数，如网卡接收/发送的字节数、接收/发送的广播包个数等。
-t 让网卡执行自我检测，有两种模式：offline or online。
-s 修改网卡的部分配置，包括网卡速度、单工/全双工模式、mac地址等。
-l 查看网卡队列数。
-L：修改网卡队列数。
```

## 查看网口基本信息

```bash
$ ethtool ens33
Settings for ens33:
	Supported ports: [ TP ]
	Supported link modes:   10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	Supported pause frame use: No
	Supports auto-negotiation: Yes
	Supported FEC modes: Not reported
	Advertised link modes:  10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	Advertised pause frame use: No
	Advertised auto-negotiation: Yes
	Advertised FEC modes: Not reported
	Speed: 1000Mb/s
	Duplex: Full
	Port: Twisted Pair
	PHYAD: 0
	Transceiver: internal
	Auto-negotiation: on
	MDI-X: off (auto)
Cannot get wake-on-lan settings: Operation not permitted
	Current message level: 0x00000007 (7)
			       drv probe link
	Link detected: yes
```

## 查询网口驱动相关信息 `-i`

```bash
$ ethtool -i ens33
driver: e1000
version: 5.10.0
firmware-version: 
expansion-rom-version: 
bus-info: 0000:02:01.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: no
```

## 查询网口收发包统计信息 `-S`

```bash
$ ethtool -S ens33
NIC statistics:
     rx_packets: 278          # 接收的数据包的数量
     tx_packets: 245          # 发送的数据包的数量
     rx_bytes: 200034         # 接收的字节数
     tx_bytes: 47526          # 发送的字节数
     rx_broadcast: 0          # 接收的广播数据包的数量
     tx_broadcast: 0          # 发送的广播数据包的数量
     rx_multicast: 0          # 接收的多播数据包的数量
     tx_multicast: 0          # 发送的多播数据包的数量
     rx_errors: 0             # 接收时发生的错误数量
     tx_errors: 0             # 发送时发生的错误数量
     tx_dropped: 0            # 发送时丢弃的数据包数量
     multicast: 0             # 接收的多播数据包数量
     collisions: 0            # 发送时发生的冲突数量
     rx_length_errors: 0      # 接收时长度错误的数量
     rx_over_errors: 0        # 接收时溢出错误的数量
     rx_crc_errors: 0         # 接收时 CRC 错误的数量
     rx_frame_errors: 0       # 接收时帧错误的数量
     rx_no_buffer_count: 0    # 接收时缓冲区不足的数量
     rx_missed_errors: 0      # 接收时丢失错误的数量
     tx_aborted_errors: 0     # 发送时中止错误的数量
     tx_carrier_errors: 0     # 发送时载波错误的数量
     tx_fifo_errors: 0        # 发送时 FIFO 错误的数量
     tx_heartbeat_errors: 0   # 发送时心跳错误的数量
     tx_window_errors: 0      # 发送时窗口错误的数量
     tx_abort_late_coll: 0    # 发送时延迟冲突中止的数量
     tx_deferred_ok: 0        # 发送时延迟成功的数量
     tx_single_coll_ok: 0     # 发送时单次冲突成功的数量
     tx_multi_coll_ok: 0      # 发送时多次冲突成功的数量
     tx_timeout_count: 0      # 发送时超时的数量
     tx_restart_queue: 0      # 发送时重新启动队列的数量
     rx_long_length_errors: 0 # 接收时长帧长度错误的数量
     rx_short_length_errors: 0 # 接收时短帧长度错误的数量
     rx_align_errors: 0        # 接收时对齐错误的数量
     tx_tcp_seg_good: 5        # 发送的 TCP 分段成功的数量
     tx_tcp_seg_failed: 0      # 发送的 TCP 分段失败的数量
     rx_flow_control_xon: 0     # 接收的流量控制 XON 的数量
     rx_flow_control_xoff: 0    # 接收的流量控制 XOFF 的数量
     tx_flow_control_xon: 0     # 发送的流量控制 XON 的数量
     tx_flow_control_xoff: 0    # 发送的流量控制 XOFF 的数量
     rx_long_byte_count: 200034 # 接收的长帧字节数
     rx_csum_offload_good: 266  # 接收校验和卸载成功的数量
     rx_csum_offload_errors: 0  # 接收校验和卸载错误的数量
     alloc_rx_buff_failed: 0    # 分配接收缓冲区失败的数量
     tx_smbus: 0                # 发送的 SMBus 操作的数量
     rx_smbus: 0                # 接收的 SMBus 操作的数量
     dropped_smbus: 0           # 丢弃的 SMBus 操作的数量
```

rx_fifo_errors 如果不为 0（在 ifconfig 中体现为 overruns 指标增长），就表示有包因为 Ring Buffer 装不下而被丢弃了。增大 Ring Buffer 的大小可以缓解丢包问题。修改命令如下：

```bash
$ ethtool -G eth0 rx 4096 tx 4096
```

增大队列长度可以解决偶发的瞬时丢包问题。不过会引入新的问题，那就是排队的包过多会导致网络包的延时增加。

## 显示网卡 offload 参数的状态 `-k`

```bash
$ ethtool -k ens33 
Features for ens33:
rx-checksumming: off
tx-checksumming: on
	tx-checksum-ipv4: off [fixed]
	tx-checksum-ip-generic: on
	tx-checksum-ipv6: off [fixed]
	tx-checksum-fcoe-crc: off [fixed]
	tx-checksum-sctp: off [fixed]
scatter-gather: on
	tx-scatter-gather: on
	tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: on
	tx-tcp-segmentation: on
	tx-tcp-ecn-segmentation: off [fixed]
	tx-tcp-mangleid-segmentation: off
	tx-tcp6-segmentation: off [fixed]
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: on
tx-vlan-offload: on [fixed]
ntuple-filters: off [fixed]
receive-hashing: off [fixed]
highdma: off [fixed]
rx-vlan-filter: on [fixed]
vlan-challenged: off [fixed]
tx-lockless: off [fixed]
netns-local: off [fixed]
tx-gso-robust: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-gso-partial: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: off [fixed]
tx-gso-list: off [fixed]
fcoe-mtu: off [fixed]
tx-nocache-copy: off
loopback: off [fixed]
rx-fcs: off
rx-all: off
tx-vlan-stag-hw-insert: off [fixed]
rx-vlan-stag-hw-parse: off [fixed]
rx-vlan-stag-filter: off [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: off [fixed]
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
tls-hw-tx-offload: off [fixed]
tls-hw-rx-offload: off [fixed]
rx-gro-hw: off [fixed]
tls-hw-record: off [fixed]
rx-gro-list: off
macsec-hw-offload: off [fixed]
```

### 配置网卡 offload 参数 `-K`

```bash
$ ethtool -K eth0 rx-checksum on|off
$ ethtool -K eth0 tx-checksum-ip-generic on|off
$ ethtool -K eth0 tso on|off
$ ethtool -K eth0 ufo on | off
$ ethtool -K eth0 gso on | off
$ ethtool -K eth0 ntuple on | off
```

## 队列管理

### 查看多队列网卡的队列使用情况 `-l`

```bash
$ ethtool -l eth0
Channel parameters for eth0:
Pre-set maximums:
RX:             0
TX:             0
Other:          1
Combined:       8
Current hardware settings:
RX:             0
TX:             0
Other:          1
Combined:       8
```

#### 修改队列数量 `-L`

```bash
$ ethtool -L eth0 combined 4
```

### 查看队列大小 `-g`

```bash
$ sudo ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:   4096
RX Mini:  0
RX Jumbo: 0
TX:   4096
Current hardware settings:
RX:   512
RX Mini:  0
RX Jumbo: 0
TX:   512
```

#### 修改队列大小 `-G`

```bash
$ sudo ethtool -G eth0 rx 4096
```

### 查看队列权重 `-x`

```bash
$ sudo ethtool -x eth0
RX flow hash indirection table for eth3 with 2 RX ring(s):
0: 0 1 0 1 0 1 0 1
8: 0 1 0 1 0 1 0 1
16: 0 1 0 1 0 1 0 1
24: 0 1 0 1 0 1 0 1
```

第一列是哈希值索引，是该行的第一个哈希值；冒号后面的是每个哈希值对于的 queue，例如，第一行分别是哈希值 0，1，2，3，4，5，6，7，对应的 packet 应该分别被放到 RX queue 0，1，0，1，0，1，0，1。

#### 修改队列权重 `-X`

在前两个 RX queue 之间均匀的分发（接收到的包）

```bash
$ sudo ethtool -X eth0 equal 2
```

用 `ethtool -X` 设置自定义权重：

```bash
$ sudo ethtool -X eth0 weight 6 2
```

以上命令分别给 rx queue 0 和 rx queue 1 不同的权重：6 和 2，因此 queue 0 接收到的数量更多。注意 queue 一般是和 CPU 绑定的，因此这也意味着相应的 CPU 也会花更多的时间片在收包上。

### 查看网卡队列绑定信息

```bash
# 注意修改网卡名称eth0
for i in `cat /proc/interrupts | grep eth0 \
 | awk -F: '{print $1}'`; do echo $i && cat /proc/irq/$i/smp_affinity ; done
```

### 在文件系统查看生效的网卡队列数量

```bash
$ ls /sys/class/net/eth0/queues
```

## 硬中断合并

发生硬中断时，CPU 会消耗一部分性能来处理上下文切换，以便处理完中断后恢复原来的工作，如果网卡每收到一个包就触发硬中断，频繁中断会使 CPU 工作效率变低。如果能适当降低中断的频率，多攒几个包一起发出硬中断，会使 CPU 的工作效率提升。虽然降低中断频率能使得收包并发量提高，但是会使一些包的延迟增大。

### 查看硬中断合并策略 `-c`

```bash
ethtool -c ens33
Coalesce parameters for ens33:
Adaptive RX: off  TX: off
stats-block-usecs: 0
sample-interval: 0
pkt-rate-low: 0
pkt-rate-high: 0

rx-usecs: 3
rx-frames: 0
rx-usecs-irq: 0
rx-frames-irq: 0

tx-usecs: 0
tx-frames: 0
tx-usecs-irq: 0
tx-frames-irq: 0

rx-usecs-low: 0
rx-frames-low: 0
tx-usecs-low: 0
tx-frames-low: 0

rx-usecs-high: 0
rx-frames-high: 0
tx-usecs-high: 0
tx-frames-high: 0
```

- **Adaptive RX**：自适应中断合并，网卡驱动自己判断啥时候合并；
- **rx-usecs**：每当过这么长时间过后，触发一个 RX interrupt 硬中断；
- **rx-frames**：每当累计收到这么多个帧后，触发一个 RX interrupt 硬中断；

### 修改硬中断合并策略 `-C`

```bash
$ ethtool -C eth0 adaptive-rx on
```















# 参考

[Linux系统下查看网卡相关数据_ethtool bus-info_香烟头的博客-CSDN博客](https://blog.csdn.net/yunlianglinfeng/article/details/120458288)