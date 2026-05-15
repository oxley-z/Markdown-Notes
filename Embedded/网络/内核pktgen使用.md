---
github.io: true
csdn: false
---
# 内核pktgen使用

## 什么是pktgen

pktgen 是一款 Linux 发包工具，可在内核高速发包。由于`pktgen`是内核内置的，使用内核空间，所以它可以达到高带宽、高发包速率，以此更好地测试路由器、交换机和其他网络设备。

一旦加载`pktgen`模块后，会自动为每一个CPU创建一个内核线程[^1]，并将该线程绑定在对应的CPU上。与此同时，在`/proc/net/pktgen/`目录下，为每一个线程创建一个`kpktgend_X`文件（其中的X为CPU编号），用来控制和监控这些线程，以及一个`pgctrl`文件，用来控制`pktgen`程序。当卸载`pktgen`模块时，`/proc/net/pktgen`文件夹会被自动删除。

同时，当将某个网卡绑定在某个线程时，也会在`/proc/net/pktgen/`目录下自动创建以该网卡名称为名的文件，用于记录此网卡设备的配置信息和运行信息。

测试程序可以配置成一直运行，或者在发送固定数量的包之后结束。也可以使用快捷键Ctrl+C终止运行。

[^1]:<font color=red>注意</font>：一个设备只能属于一个线程。

## 使能 pktgen

使用`pktgen`需要在内核配置文件开启`CONFIG_NET_PKTGEN`，

内核在执行一下命令后：

```bash
$ sudo modprobe pktgen
```

会在文件夹`/proc/net/`下创建一个`pktgen`文件夹

## 使用pktgen发包

在加载pktgen之后会在`proc/net/pktgen`下创建出如下几个文件：

![image-20230915083323445](image/内核pktgen使用/image-20230915083323445.png)

### 添加设备

```bash
$ echo "add_device ens33" > /proc/net/pktgen/kpktgend_0
```

![image-20230915091151661](image/内核pktgen使用/image-20230915091151661.png)

### 设置报文

* 部分配置

```bash
$ echo "pkt_size 64" > /proc/net/pktgen/etn
$ echo "count 1000000" > /proc/net/pktgen/eth3
$ echo "dst_mac aa:bb:cc:dd:ee:ff" > /proc/net/pktgen/eth3
```

![image-20230915091238685](image/内核pktgen使用/image-20230915091238685.png)

* 查看配置结果

```bash
cat /proc/net/pktgen/ens33
```

![image-20230915091552982](image/内核pktgen使用/image-20230915091552982.png)

### 发送报文

```bash
$ echo "start" > /proc/net/pktgen/pgctrl
```

## 查看统计信息

```bash
$ cat /proc/net/pktgen/ens33
```

![image-20230915091751564](image/内核pktgen使用/image-20230915091751564.png)

## 配置命令

### 控制命令

| 名称  | 含义               |
| ----- | ------------------ |
| start | 所有的线程开始发送 |
| stop  | 停止               |

### 线程命令

| 名称               | 含义                                     |
| ------------------ | ---------------------------------------- |
| add_device         | 添加某个端口到某个线程                   |
| rem_device_all     | 删除绑定在某个线程的所有端口             |
| max_before_softirq | 在最多发送多少个数据包后执行do_softirq() |

### 端口命令

| 名称  | 含义               |
| ----- | ------------------ |
| count | 发送数据包的个数，0 表示一直发送 |
| clone_skb | 对每个skb进行多少个复制，0表示不复制。对于Dos等测试必须至零 |
| debug | 调试 |
| frags | 数据包的片段数量 |
| delay | 发送两个数据包之间的延时 |
| src_mac_count | 源mac的数量，从src mac设置的mac开始轮询 |
| dst_mac_count | 目的mac的数量，从dst mac设置的mac开始轮询 |
| pkt_size | 链路包的大小（前去CRC的值） |
| min_pkt_size | 数据包最小值 |
| max_pkt_size | 数据包最大值 |
| udp_src_min | 最小源udp端口号 |
| udp_src_max | 最大源udp端口号 |
| udp_dst_min | 最小目的udp端口号 |
| udp_dst_max | 最大目的udp端口号 |
| flag<br />        IPSRC_RND<br />        TXSIZE_RND<br />        IPDST_RND<br />        UDPSRC_RND<br />        UDPDST_RND<br />        MACSRC_RND<br />        MACDST_RND | PSRC_RND 源IP随机发送 |
| dst_min | 目的IP的最小值 |
| dst_max | 目的IP的最大值 |
| src_min | 源IP最小值 |
| src_max | 源IP最大值 |
| dst_mac | 目的mac |
| src_mac | 源mac |
| clear_counters | 清空计数器，一般程序自动清空 |
| dst6 | 目的IPv6地址 |
| src6 | 源IPv6地址 |
| flows | 并发流的个数 |
| flowlen| 流的长度 |

## pktgen脚本

### 脚本实例

```bash
#!/bin/bash
#modprobe pktgen

function pgset() {
    local result
    echo $1 > $PGDEV
    result=`cat $PGDEV | fgrep "Result: OK:"`
    
    if [ "$result" = "" ]; then
    	cat $PGDEV | fgrep Result:
    fi
}

function pg() {
    echo inject > $PGDEV
    cat $PGDEV
}

# Config Start Here -----------------------------------------------------------
# thread config
# Each CPU has own thread. Two CPU exammple. We add ens33 respectivly.

for i in {0..3}; do
    PGDEV=/proc/net/pktgen/kpktgend_$i
    echo "Removing all devices"
    pgset "rem_device_all"
done

PGDEV=/proc/net/pktgen/kpktgend_2
echo "Adding ens33"
pgset "add_device ens33"
echo "Setting max_before_softirq 10000"
pgset "max_before_softirq 100000"

# device config
# 改为0可以加速
CLONE_SKB="clone_skb 1000000"

# NIC adds 4 bytes CRC
# 一般值越大，可以达到的throughput越大
PKT_SIZE="pkt_size 1500"
# COUNT 0 means forever
#COUNT="count 0"
COUNT="count 1000000"

# ipg is inter packet gap. 0 means maximum speed. 有些设备不支持
IPG="ipg 0"
PGDEV=/proc/net/pktgen/ens33
echo "Configuring $PGDEV"
pgset "$COUNT"
pgset "$CLONE_SKB"
pgset "$PKT_SIZE"

#pgset "$IPG"
#pgset "dst 192.168.253.23"
# 配置发送的源mac
pgset "src_mac 00:05:9a:5c:7a:00"
# 配置接收的目的mac
pgset "dst_mac 0a:00:27:8a:00:07"

# Time to run

PGDEV=/proc/net/pktgen/pgctrl
echo "Running... ctrl^C to stop"
pgset "start"
echo "Done"

# Result can be vieved in /proc/net/pktgen/ens33
```

执行结果

![pktgen](image/内核pktgen使用/pktgen.gif)

### 内核pktgen脚本

在仓库[torvalds/linux/samples/pktgen](https://github.com/torvalds/linux/tree/master/samples/pktgen)中提供了多个pktgen的脚本例子，

![image-20230915105901218](image/内核pktgen使用/image-20230915105901218.png)

### 脚本使用

```bash
./pktgen_sample01_simple.sh -i ens33 -d 127.0.0.1 -t 1 -n 1000 -m 00:15:5d:4b:19:b1 -s 100000
```

![image-20230915110658031](image/内核pktgen使用/image-20230915110658031.png)

### 测试网络PPS性能指标，判断是否丢包

客户端安装pktgen来发包，接收端通过sar命令（linux系统）来查看端口统计信息，通过对比收发两端的速率来判断是否有丢包。

![image-20230915111012837](image/内核pktgen使用/image-20230915111012837.png)

## 其他pktgen命令

### 查看版本

```bash
$ cat pgctrl
Packet Generator for packet performance testing. Version: 2.75
```

### 查看kpktgend进程

```bash
ps -A | grep pkt
   5614 ?        00:00:03 kpktgend_0
   5615 ?        00:00:03 kpktgend_1
   5616 ?        00:00:02 kpktgend_2
   5618 ?        00:00:02 kpktgend_3
   5619 ?        00:00:02 kpktgend_0
   5620 ?        00:00:02 kpktgend_1
   5621 ?        00:00:02 kpktgend_2
   5622 ?        00:00:02 kpktgend_3
```

# 参考

[pktgen | 孙勇峰的部落格 (sunyongfeng.com)](http://sunyongfeng.com/201603/networks/pktgen)

[pktgen——the packet generator within linux kernel-CSDN博客](https://blog.csdn.net/Hey___Man/article/details/98215294)

[kernel.org/doc/Documentation/networking/pktgen.txt](https://www.kernel.org/doc/Documentation/networking/pktgen.txt)

[networking:pktgen Wiki (linuxfoundation.org)](https://wiki.linuxfoundation.org/networking/pktgen)

[pktgen使用指南 - mrwuzs - 博客园 (cnblogs.com)](https://www.cnblogs.com/mrwuzs/p/14696321.html)

[pktgen使用详细教程 - 可酷可乐 - 博客园 (cnblogs.com)](https://www.cnblogs.com/kekukele/p/3709781.html)

[pktgen: A packet traffic generator in kernel space for testing network throughput. – Julio Faracco (wordpress.com)](https://juliofaracco.wordpress.com/2015/06/14/pktgen-a-kernel-space-traffic-generator-for-testing-network-throughput/)

[利用pktgen发包工具做性能测试_pktgen prbs_S100400507的博客-CSDN博客](https://blog.csdn.net/s100400507/article/details/43668557)

[network-testing/pktgen at master · netoptimizer/network-testing (github.com)](https://github.com/netoptimizer/network-testing/tree/master/pktgen) pktgen测试脚本。

[pktgen的使用 (jialeicui.github.io)](http://jialeicui.github.io/blog/pktgen.html)

https://mp.weixin.qq.com/s/D7h2AnlrDrEGU_eGmWGXcA