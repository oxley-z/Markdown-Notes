# RDMA_Verbs编程

---
# Verbs API

广义的Verbs API分为两部分：[IB_VERBS](#IB_VERBS)、[RDMA_CM](#RDMA_CM)。

## IB_VERBS

接口以ibv_xx（用户态）或者ib_xx（内核态）作为前缀，是最基础的编程接口；

## RDMA_CM

以`rdma_`为前缀的接口，主要分为两个功能：

### CMA（Connection Management Abstraction）

在Socket和Verbs API基础上实现，用于CM建链并交换信息的一组接口。CM建链是在Socket基础上封装为QP实现，从用户的角度来看，是在通过AP交换之后数据交换所需要的QPN，Key等信息，减少了代码量，让RDMA编程更加简单，其接口类似于Socket。

```c
rdma_listen()
rdma_connect()
rdma_accept()
```

### CM_VERBS

RDMA_CM本质上是建立在libibverbs上的一层包装，主要用于管理连接，使通信的双方能够确定彼此的GID和QPN信息，从而可以进行后续的交互处理。

RDMA_CM也可以用于数据交换，相当于在Verbs API上有封装了一套数据交换接口。

狭义的Verbs API指以ibv_\*为前缀的用户态Verbs接口，以及以ib_\*开头的内核rdma子系统，分别用于用户态和内核态的rdma应用。

# Linux内核RDMA子系统

代码仓库 [https://git.kernel.org/pub/scm/linux/kernel/git/rdma/rdma.git/](https://git.kernel.org/pub/scm/linux/kernel/git/rdma/rdma.git/)

代码位于内核driver/infiniband/目录下，包括核心代码和各厂商的驱动代码；

Linux内核RDMA子系统邮件订阅 [[Majordomo Lists at VGER.KERNEL.ORG](http://vger.kernel.org/vger-lists.html#linux-rdma)]([Majordomo Lists at VGER.KERNEL.ORG](http://vger.kernel.org/vger-lists.html#linux-rdma))

## 用户态和内核态驱动的关系

Linux系统将RDMA驱动分为内核态和驱动态两个大的组成部分，主要是为了**安全和性能**两个角度的平衡。

从安全角度来说，RDMA网卡作为一个硬件设备，允许其根据网络请求直接操纵本地主机的内存，这其实是一个非常危险的操作。不同于软件通过CPU访问内存时有MMU对内存访问进行权限控制，对于硬件设备而言，其对本机内存的访问是直接在物理地址上进行最底层的读写访问（虽然一些现代处理器配备有IOMMU来为外设提供类似MMU的保护能力，但在高性能网络中为了提升性能，可能并不会开启）。因此，RDMA网卡在正式开始工作之前，必须对其进行一些设置，告知网卡哪些地址范围是可以访问的，哪些地址范围是不能访问的。

此外，由于RDMA协议中传递的内存地址是虚拟地址，而硬件访问内存时需要的是物理地址，因此在使用网卡之前也需要对网卡配置一个类似于CPU上MMU所使用的页表一样的东西用于帮助RDMA网卡自己实现VA到PA的转换，如果这个转换关系表填错了，则会导致本机内存被写乱的风险。

可以看出，对于网卡一些重要配置操作，必须是安全可靠的，也就是不能让用户为所欲为的配置，要实现这一点，只能通过内核态的驱动程序来实现，即用户对网卡的重要配置操作，都需要通过执行系统调用来陷入内核，由内核确认操作合法后再交由硬件执行。

另外一个方面，RDMA网络追求高性能和低延迟，这意味着不可能所有的操作都陷入内核去完成，对于应用的收发数据这种简单的数据传输，最好是直接由用户态发起对硬件进行操作。

### 内核态驱动程序

Linux内核驱动程序的RDMA子系统代码位于内核源码中的drivers/infiniband目录下，只需要正常克隆Linux内核的源码仓库即可获得到内核态驱动程序的源码。内核态源码的目录结构大致如下：

```text
├── core
├── hw
│   ├── erdma
│   ├── hns
│   ├── mlx5
│   └── ......
├── sw
│   ├── rdmavt
│   ├── rxe
│   └── siw
└── ulp
```

- **core目录**：放置RDMA子系统的核心代码，包含对于各种特权级Verbs行为的处理、创建与用户态交互的设备文件接口等工作，可以说这个core目录提供了一个RDMA设备驱动开发的抽象框架，该目录下的代码与具体的硬件设备无关。
- **hw目录**：与各个硬件厂商相关的硬件适配层代码，用于将core目录下RDMA子系统的抽象控制信息翻译给各家硬件特有的控制协议。从狭义的角度来说，这个目录下的代码才是真正的“硬件驱动程序”。
	- mlx5：存放Mellanox系列网卡驱动，其特点是功能完善且复杂；
	- hns：存放华为海思旗下网卡驱动程序，代码量试中，包含两个版本的硬件设计。
- **sw目录**：软件模拟的RDMA协议栈，在没有真实的RDMA设备时，可以使用该目录下的驱动程序将普通网卡模拟成一块RDMA网卡进行使用，当然其性能无法与真实硬件相媲美，通常仅用于在没有真实RDMA网卡情况下的调试、测试等用途。

### 用户态驱动程序

---
# Verbs相关软件库
## rdma-core

rdma-core指**开源RDMA用户态软件协议栈**，仓库位置位于：[https://github.com/linux-rdma/rdma-core](https://github.com/linux-rdma/rdma-core)，包含用户态框架、各厂商用户态驱动、API帮助手册以及开发自测试工具等。

安装步骤：

1. 下载rdma-core源码并安装依赖包：

```bash
cd ~ 
git clone https://github.com/linux-rdma/rdma-core.git
sudo apt-get install build-essential cmake gcc libudev-dev libnl-3-dev libnl-route-3-dev ninja-build pkg-config valgrind python3-dev cython3 python3-docutils pandoc
bash build.sh
```

安装完成后即可使用常用的ib相关命令，例如：`ib_devices`。

```bash
$ ibv_devices
    device                 node GUID
    ------              ----------------
    mlx5_0              1070fd0300ddb752
    mlx5_1              1070fd0300ddb753

```

### [libibverbs](https://downloads.openfabrics.org/verbs/README.html)

libibverbs库是针对各种RDMA设备的硬件特定库和工具，

libibverbs库的本质是RDMA硬件能力的用户空间接口。它直接与内核中的RDMA子系统通信，提供了对队列对、完成队列、内存区域等核心硬件资源的创建、配置和管理能力。

这个库的设计哲学是最小抽象原则——它尽可能少地在硬件能力之上添加额外的抽象层。每个ibv_前缀的函数调用几乎都能在硬件中找到对应的操作。

### librdmacm

与libibverbs的底层哲学不同，librdmacm采用了高层抽象设计。它的核心目标是将复杂的RDMA连接建立过程简化为类似传统网络编程的模型。

librdmacm并不替代libibverbs，而是在其基础上构建了一个连接管理层。这个库处理了RDMA连接建立过程中最复杂的部分：地址解析、路由发现、QP状态转换协商等。更重要的是，它引入了事件驱动模型，让开发者能够以异步方式处理连接事件，这在构建高性能服务器时至关重要。

代码维护仓库：[ofiwg/librdmacm (github.com)](https://github.com/ofiwg/librdmacm)


![image-20221110184609504](image/[RDMA]-Verbs编程/image-20221110184609504.png)

图来自[RDMA应用程序系列——rping程序简介 - 墨天轮 (modb.pro)](https://www.modb.pro/db/485335)

[构建参考](https://blog.csdn.net/qq_36537040/article/details/115769105)

安装步骤：

1. 下载librdmacm源码并安装依赖包

```bash
cd ~
git clone https://github.com/ofiwg/librdmacm.git
cd librdmacm
sudo apt-get install autoconf automake gettext libtool libibverbs*
```

2. 编译安装librdmacm

```bash
cd ~/librdmacm
./autogen.sh
./configure
make
```

#### 测试RDMA CM 建链

##### udaddy

服务端

```bash
udaddy
```

客户端

```bash
udaddy -s 192.168.159.131
```

![image-20221110204141868](image/[RDMA]-Verbs编程/image-20221110204141868.png)

return 0表示正常退出。

##### rdma_server，rdma_client

服务端

```bash
rdma_server
```

客户端

```bash
rdma_client -s 192.168.159.132
```

![image-20221110204353372](image/[RDMA]-Verbs编程/image-20221110204353372.png)

##### ib_send_bw性能测试

客户端

```bash
ib_send_bw
```

客户端

```bash
ib_send_bw -d rxe0
```

![image-20221110205008079](image/[RDMA]-Verbs编程/image-20221110205008079.png)

##### rping

服务端

```bash
rping -s -C 101 -v
```

客户端

```bash
rping -c -a 192.168.159.132 -C 10 -v
```

![image-20221110205121422](image/[RDMA]-Verbs编程/image-20221110205121422.png)

##### ucmatose

服务端

```bash
ucmatose
```

客户端

```bash
ucmatose -s 192.168.159.132
```

![image-20221110205229486](image/[RDMA]-Verbs编程/image-20221110205229486.png)

# Verbs编程

## Hello World步骤

1. Get the device list；
2. Open the requested device
3. Query the device capabilites
4. Allocate a Protection Domain to contain your resources
5. Register a memory region
6. Create a Completion Queue（CQ）
7. Create a Queue Pair（QP）
8. Bring up a QP
9. Post work requests and poll for completion
10. Cleanup



1. 获取设备列表；
2. 打开请求的设备
3. 查询设备能力
4. 分配保护域以包含您的资源
5. 注册内存区
6. 创建完成队列(CQ)
7. 创建队列对(QP)
8. 提出一个QP
9. 发布工作请求并轮询完成
10. 清理

![3bba5d9bae953ee8ca3519516cbfdec5.png](image/[RDMA]-Verbs编程/3bba5d9bae953ee8ca3519516cbfdec5.png)

## QP状态迁移

### RST（Reset）

复位状态。当一个QP通过Create QP创建好之后就处于这个状态，相关的资源都已经申请好了，但是这个QP目前什么都做不了，其无法接收用户下发的WQE，也无法接受对端某个QP的消息。

### INIT（Initialized）

已初始化状态。这个状态下，用户可以通过Post Receive给这个QP下发Receive WR，但是接收到的消息并不会被处理，会被静默丢弃；如果用户下发了一个Post Send的WR，则会报错。

### RTR（Ready to Receive）

准备接收状态。在INIT状态的基础上，RQ可以正常工作，即对于接收到的消息，可以按照其中WQE的指示搬移数据到指定内存位置。此状态下SQ仍然不能工作。

### RTS（Ready to Send）

准备发送状态。在RTR基础上，SQ可以正常工作，即用户可以进行Post Send，并且硬件也会根据SQ的内容将数据发送出去。进入该状态前，QP必须已于对端建立好链接。

### SQD（Send Queue Drain）

SQ排空状态。顾名思义，该状态会将SQ队列中现存的未处理的WQE全部处理掉，这个时候用户还可以下发新的WQE下来，但是这些WQE要等到旧的WQE全处理之后才会被处理。

### SQEr（Send Queue Error）

SQ错误状态。当某个Send WR发生完成错误（即硬件通过CQE告知驱动发生的错误）时，会导致QP进入此状态。

### ERR（Error）

即错误状态。其他状态如果发生了错误，都可能进入该状态。Error状态时，QP会停止处理WQE，已经处理到一半的WQE也会停止。上层需要在修复错误后再将QP重新切换到RST的初始状态。

## 典型应用


# 参考链接

[jcxue/RDMA-Tutorial: A tutorial on RDMA based programming using code examples (github.com)](https://github.com/jcxue/RDMA-Tutorial)

[RDMA简介 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/336793481)

[用戶端作業 - IBM 說明文件](https://www.ibm.com/docs/zh-tw/aix/7.1?topic=cm-client-operation)

[How To Enable, Verify and Troubleshoot RDMA (force.com)](https://mymellanox.force.com/mellanoxcommunity/s/article/How-To-Enable-Verify-and-Troubleshoot-RDMA) RDMA CM测试参考

[RDMA over Converged Ethernet (RoCE) - MLNX_OFED v5.2-1.0.4.0 - NVIDIA Networking Docs](https://docs.nvidia.com/networking/pages/viewpage.action?pageId=39284930)RDMA CM测试参考

[rping 指令 - IBM 說明文件](https://www.ibm.com/docs/zh-tw/aix/7.1?topic=commands-rping-command)

[新人エンジニアの赤面ブログ 『Mellanox 社製品を使ってみましょう！(1)～NIC 後編～ 』 - 半導体事業 - マクニカ (macnica.co.jp)](https://www.macnica.co.jp/business/semiconductor/articles/mellanox/138/) 迈络思驱动安装。

[rdma_cm(7) — librdmacm-dev — Debian testing — Debian Manpages](https://manpages.debian.org/testing/librdmacm-dev/rdma_cm.7.en.html) librdmacm API文档。

[RDMA read and write with IB verbs | The Geek in the Corner (wordpress.com)](https://thegeekinthecorner.wordpress.com/2010/09/28/rdma-read-and-write-with-ib-verbs/) 编程示例。

[RDMA 编程完整学习路线图](https://www.cnblogs.com/clnchanpin/p/19510176)

[RDMAmojo](https://www.rdmamojo.com/) 优质博客

