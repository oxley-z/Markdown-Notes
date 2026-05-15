---
github.io: true
csdn: false
---
# Linux网络性能调优

## 增加环缓冲，减少丢包率

**接收环缓冲**在设备驱动程序和网络接口控制器(NIC)之间共享。网卡分配一个传输(TX)和接收(RX)环缓冲。环缓冲的名称意味着它是循环缓冲区，如果溢出会导致新到的数据覆盖现有数据。

内核使用 RX 环缓冲来存储传入的数据包，直到它们可以被网卡设备驱动程序处理。设备驱动程序排空 RX 环，通常使用 SoftIRQ，将传入的数据包置于名为 `sk_buff` 或 `skb` 的内核数据结构中，从而通过内核并开始其拥有相关套接字的应用程序。

内核使用 TX 环缓冲来保存为有线目的的传出数据包。这些环缓冲位于堆栈的底部，是可能会发生数据包丢弃的关键点，进而会对网络性能造成负面影响。

### 配置流程

#### 查看接口数据包丢包情况

```bash
$ ethtool -S <eth name>
```

#### 显示最大环缓冲大小

```bash
$ ethtool -g ens33
Ring parameters for ens33:
Pre-set maximums:
RX:		4096
RX Mini:	0
RX Jumbo:	0
TX:		4096
Current hardware settings:
RX:		256
RX Mini:	0
RX Jumbo:	0
TX:		256
```

#### 更改缓冲环大小

```bash
$ ethtool -G ens33 rx 512
$ ethtool -G ens33 tx 512
```

## 调整网络设备back队列避免丢包

### 流程

#### 确定是否需要调整 backlog 队列

显示 `/proc/net/softnet_stat` 文件中的计数器

```bash
$ cat /proc/net/softnet_stat
0000005b 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
00000070 00000000 00000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001
00000042 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000002
000006aa 00000000 00000008 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000003
```

相关的列有：

- 第一列：收到的帧总数；
- 第二列：因为完整积压队列而丢弃的帧数量；
- 最后列：CPU 内核数；

#### 调整back队列

如果 `/proc/net/softnet_stat` 文件的第二列中的值随着时间递增，则需要增加 backlog 队列的大小

##### 显示当前的后端队列大小

```bash
$ sysctl net.core.netdev_max_backlog
net.core.netdev_max_backlog = 1000
```

##### 配置当前的后端队列大小

```bash
$ sudo sysctl net.core.netdev_max_backlog=2000
net.core.netdev_max_backlog = 2000
```

监控 `/proc/net/softnet_stat` 文件中的第二列，如果值仍然增加，请再次加倍`net.core.netdev_max_backlog` 值。重复此过程，直到数据包丢弃计数器不再增加。

## 增加NIC传输队列长度，减少传输错误的数量

内核在处理传输队列前将数据包存储在传输队列中，默认长度为1000，若出现传输错误的现象，可适当增加队列长度。

### 流程

#### 显示当前传输队列长度

```bash
$ ip link show ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:ed:ed:dc brd ff:ff:ff:ff:ff:ff
    altname enp2s1
```

在本例中，`ens33` 接口的传输队列长度(`qlen`)是 `1000`。

#### 监控网络接口软件传输队列丢弃的数据包计数器

```bash
$ tc -s qdisc show dev ens33
qdisc fq_codel 0: root refcnt 2 limit 10240p flows 1024 quantum 1514 target 5.0ms interval 100.0ms memory_limit 32Mb ecn 
 Sent 114904 bytes 1291 pkt (dropped 0, overlimits 0 requeues 0) 
 backlog 0b 0p requeues 0
  maxpacket 74 drop_overlimit 0 new_flow_count 1 ecn_mark 0
  new_flows_len 0 old_flows_len 0
```

上述命令的含义如下：

* qdisc fq_codel 0：使用 fq_codel 排队调度器，编号为 0；
* root refcnt 2：根节点的引用计数为 2，表示有两个对象引用了该根节点；
* limit 10240p： 队列的包限制为 10240 个包
* flows 1024：支持的最大流数为 1024
* quantum 1514：每个包的量子大小为 1514 字节
* target 5.0ms：目标延迟为 5 毫秒
* interval 100.0ms：调度间隔为 100 毫秒
* memory_limit 32Mb：内存限制为 32 兆字节
* ecn：启用 Explicit Congestion Notification (ECN)。

发送统计信息：

* Sent 114904 bytes：
* 1291 pkt：发送的包数量为 1291
*  (dropped 0, overlimits 0 requeues 0)：丢弃的包数量为 0，超出限制的包数量为 0，重新排队的包数量为 0。

其它统计信息：

* backlog 0b 0p requeues 0：队列中的积压数据量为 0 字节，包数为 0，重新排队的次数为 0；
* maxpacket 74：最大包大小为 74 字节；
* drop_overlimit 0：超出限制的包丢弃数为 0；
* new_flow_count 1：新流的数量为 1；
* ecn_mark 0：使用 ECN 标记的包数量为 0；
* new_flows_len 0：新流的长度为 0；
* old_flows_len 0：旧流的长度为 0。

#### 创建分配脚本

创建位置`/etc/NetworkManager/dispatcher.d/99-set-tx-queue-length-up`

```bash
#!/bin/bash
# Set TX queue length on ens33 to 2000

if [ "$1" == "ens33" ] && [ "$2" == "up" ] ; then
    ip link set dev ens33 txqueuelen 2000
fi
```

#### 添加可执行权限

```bash
$ chmod +x /etc/NetworkManager/dispatcher.d/99-set-tx-queue-length-up
```

查看

```bash
$ nmcli connection show
NAME        UUID                                  TYPE      DEVICE 
有线连接 1  073b4158-b270-3f87-8f8a-044cd630c248  ethernet  ens33
```

#### 重新载入 NetworkManager 连接

注意：这里如果是中文需要对空格进行转义。

```bash
$ nmcli connection up 有线连接\ 1
```

### 验证

#### 显示传输队列长度

```bash
$ ip -s link show ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 2000
    link/ether 00:0c:29:ed:ed:dc brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast   
    9613606    7887     0       0       0       0       
    TX: bytes  packets  errors  dropped carrier collsns 
    366475     3052     0       0       0       0       
    altname enp2s1
```

#### 监控丢弃的数据包计数器

```bash
$ tc -s qdisc show dev ens33
```

如果丢弃的计数器仍然增加，请再次加倍传输队列长度。重复此过程，直到计数器不再增加。

## 调整IRQ平衡

### 配置硬件中断亲和性

当网络接口控制器(NIC)接收传入数据时，它会使用直接内存访问(DMA)将数据复制到内核缓冲区中。然后，NIC 通过触发硬中断来通知内核有关此数据。这些中断由中断处理程序处理，这些处理程序执行最少的工作，因为它们已中断另一个任务，处理程序无法中断自己。硬中断可能会牺牲 CPU 用量，特别是在使用内核锁定时。

然后，硬中断会使处理器将大多数数据包接收程序留给软件中断请求(SoftIRQ)进程。内核可以更频繁地调度这些进程。

用户可根据调整硬件中断与CPU的亲和性来提高性能，具体调整方法可查看[[Linux内核网卡中断分配不均调优 - 凛冬将至 (eddyvv.github.io)](https://eddyvv.github.io/2023/09/08/Linux内核网卡中断分配不均调优/)]。

### irqbalance服务

`irqbalance`项目主页：[Irqbalance](http://irqbalance.github.io/irqbalance/)

`irqbalance` 用于优化中断分配，它会自动收集系统数据以分析使用模式，并依据系统负载状况将工作状态置于 `Performance mode` 或 `Power-save mode`。在具有和不使用非统一内存访问(NUMA)架构的系统上，`irqbalance` 服务根据系统状况有效地在 CPU 内核间平衡中断。`irqbalance` 服务在后台运行，每 10 秒监控 CPU 负载。当 CPU 的负载过高时，该服务会将中断移到其他 CPU 内核，使处理器更有效地处理负载。

处于 `Performance mode` 时，`irqbalance` 会将中断尽可能均匀地分发给各个 `CPU core`，以充分利用 CPU 多核，提升性能。
处于 `Power-save mode` 时，`irqbalance` 会将中断集中分配给第一个 CPU，以保证其它空闲 CPU 的睡眠时间，降低能耗。

如果 `irqbalance` 没有运行，则 CPU 内核 0 通常会处理大多数中断。即使在负载中，这个 CPU 内核可能会变得忙碌，试图处理系统中所有硬件的工作负载。因此，可能会丢失或延迟中断或中断的工作。这可能导致网络和存储性能较低、数据包丢失和其他问题。

# 参考

[第 34 章 调整网络性能 Red Hat Enterprise Linux 8 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/tuning-the-network-performance_monitoring-and-managing-system-status-and-performance)

[Linux 调优篇 ：虚拟化调优（irqbalance 网卡中断绑定）* 贰_查里王的博客-CSDN博客](https://blog.csdn.net/tony_vip/article/details/113793313)

[【性能】中断绑定和查看|irqbalance 中断负载均衡|CPU瓶颈_irqbalance 配置-CSDN博客](https://blog.csdn.net/bandaoyu/article/details/129234911)

[Irqbalance/irqbalance: The irqbalance source tree - The new official site for irqbalance (github.com)](https://github.com/Irqbalance/irqbalance)

[中断NUMA绑核-系统配置-优化建议指导-常用操作-鲲鹏性能分析插件-2.5.5-鲲鹏开发套件插件工具（IntelliJ）-鲲鹏DevKit历史版本-文档首页-鲲鹏社区 (hikunpeng.com)](https://www.hikunpeng.com/document/detail/zh/kunpengdevkithistory/intellijhistory/2.5.5/kunpengideintellij_10_0329.html)