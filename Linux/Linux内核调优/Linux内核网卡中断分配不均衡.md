---
github.io: true
csdn: false
---
# Linux内核网卡中断分配不均衡调优

## 查看CPU具体中断情况

`/proc/interrupts` 文件中可以看到各个 CPU 上的中断情况。

```bash
cat /proc/interrupts

# 持续查看
watch -d cat /proc/interrupts

# 持续查看并检索
watch -n 1 'egrep "CPU|ens33" /proc/interrupts'
```

![watch_interrupts](image/Linux%E5%86%85%E6%A0%B8%E7%BD%91%E5%8D%A1%E4%B8%AD%E6%96%AD%E5%88%86%E9%85%8D%E4%B8%8D%E5%9D%87%E8%A1%A1/watch_interrupts.gif)

`/proc/irq/[irq_num]/smp_affinity_list` 可以查看指定中断当前绑定的 CPU。

使用以下命令可查看对应中断绑定的CPU核：

```bash
cat /proc/interrupts | grep ens33 | cut -d: -f1 | while read i; do echo -ne irq":$i\t bind_cpu: "; cat /proc/irq/$i/smp_affinity_list; done | sort -n -t' ' -k3
```

输出

![image-20230908140038965](image/Linux%E5%86%85%E6%A0%B8%E7%BD%91%E5%8D%A1%E4%B8%AD%E6%96%AD%E5%88%86%E9%85%8D%E4%B8%8D%E5%9D%87%E8%A1%A1/image-20230908140038965.png)

### 查找对应网卡中断触发情况

```bash
cat /proc/interrupts | grep ens33 | tr -s ' ' '\t'|cut -f 1-15
```

输出

![image-20230908140105116](image/Linux%E5%86%85%E6%A0%B8%E7%BD%91%E5%8D%A1%E4%B8%AD%E6%96%AD%E5%88%86%E9%85%8D%E4%B8%8D%E5%9D%87%E8%A1%A1/image-20230908140105116.png)

如输出所示，中断的处理大部分被CPU3所处理。

## 中断绑定

中断绑定即设置中断的`CPU Affinity`，让中断只在指定CPU核心上进行响应。

Kernel 2.4开始支持把不同的硬件中断请求(IRQ)分配到特定的CPU上，绑定技术被称为`SMP IRQ Affinity`。

### /proc/irq/[irq_num]/smp_affinity

`smp_affinity`文件用于存放CPU位掩码（16进制），修改该文件中的值可以改变 CPU 和某中断的亲和性。

#### 使用CPU列表手动修改网卡各队列中断绑定

```bash
echo "2" > /proc/irq/19/smp_affinity
```

### /proc/irq/[irq_num]/smp_affinity_list

该文件存放的是 CPU 列表（十进制）。注意，CPU 核心个数用表示编号从 0 开始，如 CPU0, CPU1 等。

#### 使用CPU列表手动修改网卡各队列中断绑定

```bash
echo 2 > /proc/irq/19/smp_affinity_list
```

查询中断分布

![watch_interrupts_1](image/Linux%E5%86%85%E6%A0%B8%E7%BD%91%E5%8D%A1%E4%B8%AD%E6%96%AD%E5%88%86%E9%85%8D%E4%B8%8D%E5%9D%87%E8%A1%A1/watch_interrupts_1.gif)

## 网络数据处理监控

### /proc/net/softnet_stat

`/proc/net/softnet_stat`用于网络数据处理监控。

```bash
cat /proc/net/softnet_stat
0000002c 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0000007d 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
0000001e 00000000 00000002 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
00000073 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

`/proc/net/softnet_stat`的每一行对应一个`struct softnet_data`数据结构，每个CPU一个，值之间用一个空格分隔，并以十六进制显示。

* 第一个值，`sd->processed`，是处理的网络帧数。如果您使用以太网绑定，这可能会超过接收到的网络帧总数。有些情况下，以太网绑定驱动程序会触发网络数据重新处理，同一数据包将使 `sd->processed` 计数增加不止一次。
* 第二个值，`sd->dropped`，是因处理队列没有空间而丢弃的网络帧数。稍后再谈。
* 第三个值，`sd->time_squeeze`，（如我们所见）是 `net_rx_action` 循环因消耗预算或达到时间限制而终止的次数，但仍然可以完成更多工作。如前所述，增加 `budget` 可以帮助减少这种情况。
* 接下来的 5 个值始终为 0。
* 第九个值，`sd->cpu_collision`，是在发送数据包尝试获取设备锁时发生冲突的次数。本文讨论的是接收，因此下面不会看到这个统计量。
* 第十个值，`sd->received_rps`，是唤醒此 CPU 通过处理器间中断处理数据包的次数。
* 最后一个值，`flow_limit_count`，是达到流量限制的次数。流量限制是可选的 `Receive Packet Steering`功能，稍后会探讨到该特性。

## 批量配置中断亲和性

```bash
 1 #!/bin/bash
  2 
  # nic_queue_name为网卡使用队列的名称
  3 nic_queue_name=$1
  4 array=($(cat /proc/interrupts | grep $nic_queue_name | awk '{gsub(/:/, "", $1); printf "%s%s", (NR==1 ? "" : " "), $1}'))
  5 array_len=${#array[@]}
  6 
  7 # 获取CPU核心数
  8 core_count=$(nproc)
  9 
 10 # 计算要输出的核心数（0到core_count-1）
 11 output_cores=$(seq 0 $((core_count-1)))
 12 
 13 # 将输出核心数与数组长度取模，以循环使用
 14 output_cores=($output_cores)
 15 output_core_count=${#output_cores[@]}
 16 
 17 for ((index=0; index<array_len; index++)); do
 18   irq="${array[index]}"
 19   core="${output_cores[index % output_core_count]}"
 20   echo "$core" > /proc/irq/$irq/smp_affinity_list
 21 done
```

# 参考

[译｜Monitoring and Tuning the Linux Networking Stack: Receiving Data (cyningsun.com)](https://www.cyningsun.com/04-24-2023/monitoring-and-tuning-the-linux-networking-stack-recv-cn.html#proc-net-dev-使用)

[关于网卡中断不均衡问题及其解决方案 - Weir's Note (leeweir.github.io)](https://leeweir.github.io/posts/irq-not-balance/)

[Monitoring and Tuning the Linux Networking Stack: Sending Data | Packagecloud Blog](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data/)

[Linux性能优化（十六）——中断绑定_51CTO博客_linux 中断架构](https://blog.51cto.com/quantfabric/2594344)

[Linux 多核下绑定硬件中断到不同 CPU（IRQ Affinity）-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/64868)

[[译1\]linux网络栈监控及调优：数据接收-电子工程专辑 (eet-china.com)](https://www.eet-china.com/mp/a203948.html)

[第 34 章 调整网络性能 Red Hat Enterprise Linux 8 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/tuning-the-network-performance_monitoring-and-managing-system-status-and-performance)