---
github.io: true
csdn: false
---
# DPDK及其上层应用的安装使用

DPDK是Intel公司开发的一款高性能的网络驱动组件，旨在为数据面应用程序提供一个简单方便的，完整的，快速的数据包处理解决方案，主要技术有用户态、轮询取代中断、零拷贝、网卡RSS、访存DirectIO等。

## DPDK

### 安装

DPDK需要根据系统版本选择，版本的选择可查看[DPDK官方文档](https://doc.dpdk.org/guides/index.html)，由于当前内核为`Ubuntu20.04.5`故选用dpdk版本为20.11版本，文档中也提供相应的安装方式。

软件包

```bash
$ sudo apt-get install meson
$ sudo apt-get install python3-pyelftools	
$ sudo apt-get install pkg-config
```

安装步骤

```bash
$ git clone git://dpdk.org/dpdk-stable
$ cd dpdk-stable
$ git checkout 20.11
$ meson build
$ cd build
$ ninja
$ ninja install
```

检查是否安装成功：

```bash
$ pkg-config --modversion libdpdk
20.11.2
```

`dpdk-20.11` 默认会编译 app 目录下的所有应用，比如 testpmd，编译完成后生成的可执行文件在 `./build/app` 目录下面：

```bash
$ ls -l build/app
...
-rwxr-xr-x 1 root root 16949456 12月 13 09:56 dpdk-pdump
-rwxr-xr-x 1 root root 16950176 12月 13 09:56 dpdk-proc-info
-rwxr-xr-x 1 root root 16949984 12月 13 09:56 dpdk-test-acl
-rwxr-xr-x 1 root root 17088760 12月 13 09:56 dpdk-test-bbdev
-rwxr-xr-x 1 root root 16950688 12月 13 09:56 dpdk-test-cmdline
-rwxr-xr-x 1 root root 16984424 12月 13 09:56 dpdk-test-compress-perf
-rwxr-xr-x 1 root root 17024104 12月 13 09:56 dpdk-test-crypto-perf
-rwxr-xr-x 1 root root 17058480 12月 13 09:56 dpdk-test-eventdev
-rwxr-xr-x 1 root root 16949448 12月 13 09:56 dpdk-test-fib
-rwxr-xr-x 1 root root 16970928 12月 13 09:56 dpdk-test-flow-perf
-rwxr-xr-x 1 root root 16965288 12月 13 09:56 dpdk-test-pipeline
-rwxr-xr-x 1 root root 17826888 12月 13 09:56 dpdk-testpmd
-rwxr-xr-x 1 root root 16948944 12月 13 09:56 dpdk-test-regex
-rwxr-xr-x 1 root root 16948936 12月 13 09:56 dpdk-test-sad
...
```

### 相关命令

查看网卡绑定情况

```bash
./dpdk-stable-20.11.9/usertools/dpdk-devbind.py -s
```

![image-20230921134405249](image/DPDK及其上层应用的使用/image-20230921134405249.png#pic_center)

## TestPMD安装使用

TestPMD是一个使用DPDK软件包分发的参考应用程序。其主要目的是在网络接口的以太网端口之间转发数据包。

![image-20230921134040659](image/DPDK及其上层应用的使用/image-20230921134040659.png)

### 编译igb_uio驱动

```bash
$ git clone http://dpdk.org/git/dpdk-kmods
$ cd dpdk-kmods/linux/igb_uio
$ make
```

### 加载uio、igb_uio

```bash
$ modprobe uio
$ insmod ./dpdk-kmods/linux/igb_uio/igb_uio.ko intr_mode=legacy
```

### 配置大页内存

```bash
$ echo 512  >  /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

### 运行TestPMD

#### 网卡驱动绑定至igb_uio

注意：Mellanox网卡不需要绑定可以直接检测到设备，其他网卡需要进行绑定。

```bash
$ ifconfig ens33 down
$ ifconfig ens34 down
$ dpdk-devbind.py -b igb_uio <网卡名称 or 网卡的BDF号>
```

#### 运行

```bash
$ cd dpdk-stable-21.11.2/build/app/
$ sudo ./dpdk-testpmd -l 0-3 -n 4 -- -i --portmask=0x3 --nb-cores=2
```

![image-20230921113227538](image/DPDK及其上层应用的使用/image-20230921113227538.png)

`-l `参数用来设置用到的逻辑核。0核用来管理命令行，1、2核用来转发数据包。`-n` 参数用来指定使用的内存通道数。--（破折号）分开了EAL参数和应用程序参数。端口掩码指定使用的网卡端口。`--nb-cores`用来指定两个核作为数据的转发。

#### 解绑

```bash
$ lspci | grep Eth
$ dpdk-devbind.py -u 02:02.0 02:03.0
$ modprobe e1000
$ ./dpdk-stable-20.11.9/usertools/dpdk-devbind.py -b e1000 0000:02:01.0
$ ifconfig ens33 up
$ ifconfig ens34 up
```

### TestPMD的相关使用

#### 启动转发

```bash
$ testpmd> start
```

#### 停止转发

```bash
$ testpmd> stop
```

#### 查看当前所有在用端口的相关统计

```bash
$ testpmd> show port stats all
```

#### 查看port信息

```bash
$ testpmd> show port info 0
```

#### 使用多核

```bash
testpmd> set nbcore 2
 
testpmd> show config fwd
 
io packet forwarding - ports=2 - cores=2 - streams=2 NUMA support disabled, MP over anonymous pages disabled
Logical Core 13 (socket 1) forwards packets on 1 streams:P=0/Q=0 (socket 0) -> TX P=1/Q=0 (socket 0) peer=02:00:00:00:00:01
Logical Core 14 (socket 1) forwards packets on 1 streams:P=1/Q=0 (socket 0) -> TX P=0/Q=0 (socket 0) peer=02:00:00:00:00:00
```

#### 更改转发模式

[TestPMD 的转发模式](http://doc.dpdk.org/guides/testpmd_app_ug/testpmd_funcs.html#set-fwd)包括：

* <font color=red>io</font>（输入/输出模式）：通常也被称为 IO 模式，是最常用的转发模式，也是 TestPMD 启动时的默认模式。在该模式下，CPU 从一个端口接收数据包（Rx），并将其发送到另一个端口（Tx）。如果需要的话，一个端口可同时用于接收和发送。

* <font color=red>rxonly</font>（收包模式）：在此模式下，TestPMD 会轮询 Rx 端口的数据包，然后直接释放而不发送，以这种方式充当数据包接收器。

* <font color=red>txonly</font>（发包模式）：在此模式下，TestPMD 生成 64Byte 的 IP 数据包，并从 Tx 端口发送出去。不接收数据包，仅作为数据包的发送源。

* <font color=red>mac</font>：在转发报文前修改报文的 src MAC 地址和 dst MAC 地址。默认的行为是将 src MAC 地址设置为转发接口的 MAC 地址，将 dst MAC 地址设置为预设的 MAC 地址，可以通过 eth-peer 或 eth-peers-configfile 命令参数，在程序启动时配置预设 MAC 地址。目前还不支持对 src MAC 地址进行预设。

* <font color=red>macswap</font>：MAC 交换转发模式，在转发报文之前交换报文的 src MAC 地址和 dst MAC 地址。

* <font color=red>flowgen</font>：多数据流生成模式，根据不同的 dst IP 地址生成一组数据流，同时终结接收到的流量。

* <font color=red>csum</font>：根据报文的 offload 标记，通过硬件或软件方法修改报文的 checksum 字段。

* <font color=red>icmpecho</font>：接收一组报文，查找是否有 ICMP echo 请求，如果有，就回应 ICMP echo reply。

* <font color=red>ieee1588</font>：演示对 Rx 和 Tx 基于 L2 IEEE1588 V2 PTP 时间同步特性，需要配置 CONFIG_RTE_LIBRTE_IEEE1588=y。

* <font color=red>5tswap</font>：交换L2，L3，L4的源和目的地 (如果存在)。

  ​				L2交换以太网的源地址和目的地址，与macswap相同。

  ​				L3交换IP (v4和v6) 的源地址和目的地址。

  ​				L4交换传输层的源端口和目的端口 (TCP和UDP)。

* <font color=red>shared-rxq</font>：仅接收共享Rx队列。从`mbuf`中解析数据包源端口，并相应地更新流统计信息。

```bash
testpmd> set fwd rxonly
testpmd> start
```

#### 检查转发配置

```bash
$ testpmd> show config fwd
```

![image-20230921113320721](image/DPDK及其上层应用的使用/image-20230921113320721.png)

表明 TestPMD 正使用默认的 IO 转发模式，`logical Core 1` 将对`Port 0`接收到的数据包转发到`Port 1`上发送出去；

### pktgen-dpdk

DPDK-Pktgen其实就是DPDK的一个应用，它类似于linux原生的pktgen，通过自己构造数据包，然后发送。而DPDK-Pktgen做的更强大，他可以通过用lua脚本或者json编辑自己的测试过程，同时输出自己关心的数据，比如发送，接收的数据包数量，流量带宽等等。

#### 安装

```bash
$ git clone git://dpdk.org/apps/pktgen-dpdk
$ cd pktgen-dpdk
$ git checkout pktgen-21.03.1
$ meson build
$ cd build
$ ninja
$ ls -l app/pktgen  //编译出来的可执行程序
-rwxr-xr-x 1 root root 477672 10月 19 12:59 app/pktgen
```

##### 基本使用方法

```bash
Usage: ./app/pktgen [EAL options] -- [-h] [-v] [-P] [-G] [-T] [-f cmd_file] [-l log_file] [-s P:PCAP_file] [-m <string>] 
```

pktgen 支持的命令行参数还是挺多的，具体可以通过 pktgen --help 来详细查看。不过它像大多数基于 dpdk 的测试程序一样，命令行参数主要有两部分组成：

- [EAL options](![image-20230921164303167](image/DPDK%E5%8F%8A%E5%85%B6%E4%B8%8A%E5%B1%82%E5%BA%94%E7%94%A8%E7%9A%84%E4%BD%BF%E7%94%A8/image-20230921164303167.png))
- pktgen 自有参数

其中，最重要的一个参数就是 -l 参数，用它来指定使用的核列表，比如：-l 1,2 或者 -l 1-2，表示使用核 1 和核 2。值得注意的是，pktgen 至少要指定两个核，因为 pktgen 需要一个核与用户进行交互，比如响应测试过程中用户的输入。

pktgen 自有参数部分最重要的是 -m 参数，用它来指定网口与核之间的对应关系，比如：-m 2.0，表示让核 2 来处理网口 0。值得注意的是，若要指定多个对应关系（使用多个网卡和多个核），则需多次使用 -m 参数。如果要收包，最好也指定一下 -P 参数，表示让所有网口进入混杂模式，以便接收到所有数据包。

```bash
./build/app/pktgen -l 1-3 -- -P -m 2.0 -m 3.1
```

```bash
Pktgen:/> set 0 size 64 // 将网口 0 所发包的帧大小设置 64 字节
Pktgen:/> set 0 src ip 192.168.0.1/32 // 将网口 0 所发包的源 IP 设置为 192.168.0.1
Pktgen:/> set 0 dst ip 192.168.0.2 // 将网口 0 所发包的目的 IP 设置为 192.168.0.2
Pktgen:/> set 0 count 100000000  // 将网口 0 的发包数设置为 100000000
Pktgen:/> set 0 rate 20 // 将网口 0 的发包速率设置为最大速率的 20%
Pktgen:/> start 0 // 启动网口 0 的发包过程
Pktgen:/> stop 0 // 停止网口 0 的发包过程
Pktgen:/> stp // 停止所有网口的发包过程
Pktgen:/> clr // 清除屏幕上的收发包统计信息
Pktgen:/> quit // 退出 pktgen 程序
```

#### 常见错误

##### ERROR: Dependency "libdpdk" not found

```bash
Found pkg-config: /usr/bin/pkg-config (1.4.2)
Found CMake: /usr/bin/cmake (3.18.2)
Run-time dependency libdpdk found: NO (tried pkgconfig and cmake)

meson.build:62:0: ERROR: Dependency "libdpdk" not found, tried pkgconfig and cmake
```

显示没有找到dpdk相关的库，原因在于dpdk编译时生成的库并没有放入/usr/lib与/lib中。dpdk相关的库数量非常多，如果直接拖动到这两个文件夹中，不方便管理。因此可以采用ld.so的方式进行类似"快捷方式"的连接，dpdk的库不需要移动位置就可以被其他的程序调用。

**解决方法**：重新安装dpdk

##### error: pcap library not found! 

```bash
app/meson.build:36:0: ERROR: C library 'pcap' not found
```

解决方法

```bash
$ sudo apt-get install libsqlite3-0 libpcap0.8
$ sudo apt-get install libpcap-dev
```

##### Pogram sphinx-build found: NO

```bash
Pogram sphinx-build found: NO
Build targets in project: 8
```

解决方法

```bash
sudo apt-get install sphinx-doc sphinx-common
```

##### librte_kvargs.so.21: cannot open shared object file: No such file or directory

```bash
./app/pktgen: error while loading shared libraries: librte_kvargs.so.21: cannot open shared object file: No such file or directory
```

解决方法

```bash
$ find ./ -name librte_kvargs.so.21
./dpdk-stable-20.11.9/build/lib/librte_kvargs.so.21
$ sudo ln -s ./dpdk-stable-20.11.9/build/lib/librte_kvargs.so.21 /usr/local/lib/librte_kvargs.so.21
$ ldconfig -v
```

### DPDK日志系统

根据日志的优先级高低， DPDK将日志分为 8 个级别（宏定义如下）。 数字越小级别越高， 其中`RTE_LOG_DEBUG`的值为 8, 这是日志的最低级别， 意味着输出的日志信息最多，因为高于这个级别的日志的信息都会被输出；相应地， `RTE_LOG_EMERG` 为日志的最高级别，只输出严重错误的信息 。

```c
/* Can't use 0, as it gives compiler warnings */
#define RTE_LOG_EMERG    1U  /**< System is unusable.               */
#define RTE_LOG_ALERT    2U  /**< Action must be taken immediately. */
#define RTE_LOG_CRIT     3U  /**< Critical conditions.              */
#define RTE_LOG_ERR      4U  /**< Error conditions.                 */
#define RTE_LOG_WARNING  5U  /**< Warning conditions.               */
#define RTE_LOG_NOTICE   6U  /**< Normal but significant condition. */
#define RTE_LOG_INFO     7U  /**< Informational.                    */
#define RTE_LOG_DEBUG    8U  /**< Debug-level messages.             */
```



```bash
$ ./build/app/dpdktestpmd --log-level="pmd,net.mqnic.nint":8 --log-level=8
```

命令中，带有命令选项`--log-level="pmd.net.mqnic.init":8` 的同时，还加上`--log-level=8`把
全局级别设置成了`RTE_LOG_CRIT (数字8) ` 。

# 参考

[DPDK系列之二：pktgen-dpdk向testpmd发送数据的测试_cloudvtech的博客-CSDN博客](https://blog.csdn.net/cloudvtech/article/details/80359825)

[NVIDIA BlueField-2 DPU 和 DPDK 开发入门指南 - NVIDIA 技术博客](https://developer.nvidia.com/zh-cn/blog/nvidia-bluefield-2-dpu-dpdk/)

## DPDK

[DPDK Tools User Guides — Data Plane Development Kit 17.11.10 documentation](https://doc.dpdk.org/guides-17.11/tools/index.html)

[DPDK 配置和优化指南 (amperecomputing.com)](https://amperecomputing.com/zh-CN/tuning-guides/DPDK-setup-and-tuning-guide)

## TestPMD

[云服务器 使用 DPDK 测试-最佳实践-文档中心-腾讯云 (tencent.com)](https://cloud.tencent.com/document/product/213/56300) 包含使用TestPMD测试带宽及吞吐率。

[Testpmd Application User Guide — Data Plane Development Kit 17.11.10 documentation (dpdk.org)](https://doc.dpdk.org/guides-17.11/testpmd_app_ug/index.html) TestPMD官方手册

## pktgen-dpdk

[dpdk-pktgen 的安装和基本使用-CSDN博客](https://blog.csdn.net/choumin/article/details/120884509)

[9. EAL parameters — Data Plane Development Kit 23.07.0 documentation (dpdk.org)](https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html)