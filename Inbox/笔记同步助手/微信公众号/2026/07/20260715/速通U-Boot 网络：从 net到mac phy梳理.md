---
author: Surest
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwOTUxMzQzOA==&mid=2247484377&idx=1&sn=b57a04b2a6e33e07760d2bfda5ea85cb&chksm=c035ee510e2f1619c6418bda77cfb30c1ef44d030cf0342eeff6ffaed478d1063c894e35482a&mpshare=1&scene=1&srcid=0715c0kXhwHnpqukOi7MzNRO&sharer_shareinfo=965b3190d22d2d61ba77aa79248012ba&sharer_shareinfo_first=965b3190d22d2d61ba77aa79248012ba#rd
saved: 2026-07-15 07:56:38
tags:
  - 笔记同步助手
id: b0b8976e-36ff-4973-990c-0710b8801ed4
---

公众号名称：摸鱼的日记本

作者名称：Surest

发布时间：2026-07-14 19:00

> **参考资料：** U-Boot 源码 `net/net.c`、`net/eth-uclass.c`、`net/tftp.c`、`net/bootp.c`、`net/arp.c`、`drivers/net/phy/*`、`include/net.h`、`include/phy.h`、`doc/README.net`
> 
> **主题：** U-Boot 网络子系统的整体架构、驱动模型、协议栈实现，以及 mii/mdio/PHY 层的诊断与调试
> 
> **适用读者：** 做 board bringup、写 eth 驱动、调试 tftpboot 失败的驱动/内核工程师

## 一、全景介绍

### 1.1 为什么 U-Boot 要有网络

嵌入式设备做 bring-up 的时候有一段黄金时间：SoC 上电能跑，DDR 训练过了，串口打得出字，但 eMMC/UFS 里空空如也。这时候要把 kernel、rootfs、devicetree 塞进去，插 SD 卡太慢，串口 xmodem 更慢，**网络传输是效率最高的一条**：

-   -   **tftpboot**：从开发主机拉 kernel/dtb/initrd 到 DRAM，直接 booti 起来
-   -   **dhcp**：一条命令拿地址、拿 tftp server、拿 bootfile，配合 pxe 做批量 bringup
-   -   **nfsboot**：kernel 起来后 rootfs 挂主机的 NFS export，改 userspace 不用重烧
-   -   **update**：产线烧录 fitimage、A/B 分区切换后从服务器拉新镜像

前面几篇讲了 `_start` → `board_init_f` → SPL/TPL → env → DM → bootm/booti/bootelf，中间从 DRAM 装载镜像这一步一直被简化为 `tftpboot 0x40000000 Image` 一行命令。这一篇把这行命令背后的网络子系统展开。

### 1.2 U-Boot 网络子系统的骨架

U-Boot 的网络子系统很轻，源码就在 `net/` 目录下，加起来大约 8000 行，比 Linux 内核网络栈少两个数量级。它的架构分三层：

-   -   **命令层**：`cmd/net.c`、`cmd/mii.c`、`cmd/dhcp.c` 提供用户命令
-   -   **协议层**：`net/net.c` 主循环 + `arp.c` / `bootp.c` / `dhcp.c` / `tftp.c` / `ping.c` / `nfs.c` 各协议实现
-   -   **驱动层**：`net/eth-uclass.c`（DM eth uclass）+ `drivers/net/*.c`（各家 MAC 驱动）+ `drivers/net/phy/*.c`（PHY 层）

三层之间用 **eth\_ops 接口** 和 **net\_loop() 回调机制** 解耦：命令层调 `net_loop()` 传入 protocol，协议层调 `eth_send()/eth_recv()` 收发包，驱动层实现 `eth_ops` 的 5 个函数。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/828112dfdbf7a59495f61ec71cdf7712_MD5.jpg]]

_图 1 · U-Boot 网络子系统三层分工_

## 二、三个关键设计取舍

U-Boot 网络子系统有三个关键设计取舍决定了它今天的样子。

### 2.1 取舍一：半协议栈，没有 TCP

U-Boot 只实现下列协议：

-   -   **L2**：Ethernet framing、ARP、ICMP echo（ping）
-   -   **L3**：IPv4（无路由，只走 gateway 一次跳转）
-   -   **L4**：UDP（TFTP、DHCP、NFS 都基于 UDP）

> ⚠️ **故意不做 TCP。** 原因：TCP 需要状态机、重传、拥塞控制，代码量至少多 3～4 倍；bringup 场景下 TFTP+UDP 已经足够，靠 lock-step ack 实现可靠性（发一块等一块 ack，不追求带宽）。想要 TCP 就上 Linux。

代价是 TFTP 传大文件慢：以太网理论 1Gbps，TFTP 实测常常只有 100～300Mbps，因为每 512 字节要停下来等一次 ack。lwIP 之类的做法可以塞进 U-Boot 但主线不接受，会破坏 boot loader 保持简单的原则。

### 2.2 取舍二：基于回调而非线程

U-Boot 是单线程的，没有 kernel、没有调度器，网络怎么异步收包？

答案：**主循环 + handler 回调**。

```
// net/net.c 简化
int net_loop(enum proto_t protocol)
{
    // 1. 初始化 protocol
    switch (protocol) {
    case TFTPGET:
        tftp_start(protocol);       // 内部会 net_set_udp_handler(tftp_handler)
        break;
    case DHCP:
        bootp_reset();
        dhcp_state = INIT;
        bootp_request();            // 内部会 net_set_udp_handler(dhcp_handler)
        break;
    ...
    }

    // 2. 主循环轮询
    for (;;) {
        // 让 driver 收一次包（非阻塞）
        eth_rx();

        // 用户按了 Ctrl-C？
        if (ctrlc()) {
            net_state = NETLOOP_FAIL;
            break;
        }

        // 超时了？（tftp_timeout_handler、arp_timeout_check 等）
        if (time_since_last >= timeout_ms)
            timeout_handler();

        // 收到需要处理的包，protocol handler 已经在 eth_rx() 内部被调
        // handler 会更新 net_state
        if (net_state != NETLOOP_CONTINUE)
            break;
    }
    return net_state;
}
```

`eth_rx()` 拿到包之后调用 `net_process_received_packet()`，后者按 EtherType 分派（ARP / IPv4），IPv4 再按 protocol 字段分派（ICMP / UDP），UDP 再按 dest port 分派到当前注册的 handler（`net_set_udp_handler` 设置的那个）。

> 💡 **优点：** 驱动只需要提供 poll 式 recv，不用中断、不用线程、不用锁。  
> **缺点：** 延迟只能靠 poll 频率保证，好在 U-Boot 阶段没人在乎微秒延迟。

### 2.3 取舍三：eth\_ops 只有 5 个函数

DM 化后，`struct eth_ops` 定义在 `include/net.h`：

```
struct eth_ops {
    int (*start)(struct udevice *dev);                              // 启动网卡
    int (*send)(struct udevice *dev, void *packet, int length);     // 发一帧
    int (*recv)(struct udevice *dev, int flags, uchar **packetp);   // 收一帧（非阻塞）
    int (*free_pkt)(struct udevice *dev, uchar *packet, int length);// recv 完释放
    void (*stop)(struct udevice *dev);                              // 停止网卡

    /* 可选：设置 MAC 地址、读 PHY 寄存器 */
    int (*write_hwaddr)(struct udevice *dev);
    int (*read_rom_hwaddr)(struct udevice *dev);
    int (*set_promisc)(struct udevice *dev, bool enable);
};
```

写一个 U-Boot 网卡驱动比写 Linux 网卡驱动简单一个数量级，5 个函数几百行代码搞定。当然功能也弱：不做 checksum offload、不做多队列、不做 GSO/GRO。

---

## 三、三大核心数据结构

> **一句话概括：U-Boot 网络 = eth uclass + protocol handler + PHY 子系统三件套。**

### 3.1 中心一：struct udevice（eth uclass）

DM 里每个网卡是一个 `struct udevice`，属于 uclass `UCLASS_ETH`：

```
// net/eth-uclass.c
struct eth_uclass_priv {
    struct udevice *current;    // 当前使用的网卡（多网口时决定 tftp 走哪个）
};

struct eth_pdata {              // 平台数据，每个 dev 一份
    phys_addr_t iobase;         // MAC 寄存器基址
    unsigned char enetaddr[6];  // MAC 地址
    int phy_interface;          // RGMII / RMII / SGMII / MII
    int max_speed;              // 10/100/1000
    void *priv_pdata;
};
```

一个板子多网口的话，`eth list` 会列出所有 `udevice`；`setenv ethact eth1` 切换当前网口。

### 3.2 中心二：struct phy\_device（PHY 子系统）

PHY 独立于 MAC，用 MDIO 总线通信。U-Boot 抽象成 `struct phy_device`：

```
// include/phy.h
struct phy_device {
    struct mii_dev *bus;        // 所在的 MDIO 总线
    struct phy_driver *drv;     // PHY 驱动
    struct udevice *dev;        // 对应的 udevice（如果 DM 化了）
    int addr;                   // MDIO addr（0～31）
    int phy_id;                 // OUI + model + rev，读 reg 2/3 得到
    int flags;
    u32 advertising;
    u32 supported;
    int autoneg;
    int speed;
    int duplex;
    int link;
    int interface;              // PHY_INTERFACE_MODE_RGMII 等
    ...
};
```

**PHY 驱动分两级**：

-   -   `drivers/net/phy/phy.c`：通用逻辑（`genphy_config_aneg`、`genphy_read_status`）
-   -   `drivers/net/phy/{atheros,marvell,realtek,ti,...}.c`：厂商特殊寄存器（比如 Micrel KSZ9031 的 RGMII 延迟寄存器）

MAC 驱动在 `probe()` 里通过 `phy_connect(bus, addr, dev, interface)` 拿到 `phy_device` 指针，之后所有链路层动作（协商、读速度、读 link 状态）都通过它。

### 3.3 中心三：struct mii\_dev（MDIO 总线）

MDIO 是 IEEE 802.3 clause 22/45 定义的两线串行总线（MDC + MDIO），MAC 通过它读写 PHY 内部寄存器：

```
// include/miiphy.h
struct mii_dev {
    struct list_head link;
    char name[MDIO_NAME_LEN];
    void *priv;
    int (*read)(struct mii_dev *bus, int addr, int devad, int reg);
    int (*write)(struct mii_dev *bus, int addr, int devad, int reg, u16 val);
    int (*reset)(struct mii_dev *bus);
    struct phy_device *phymap[32]; // 每条 MDIO 总线上最多 32 个 PHY
    u32 phy_mask;
    int reset_delay_us;
};
```

`mdio list` 命令就是遍历全局 `mii_bus_list` 打印每条总线上挂了哪些 PHY。**一条 MDIO 总线可以挂多个 PHY**（比如两个网口共享一条总线），所以命令要指定 phy addr。

PHY 内部寄存器按 clause 22 是 5 位地址（0～31），常用的：

-   -   **reg 0 (BMCR)**：控制寄存器，复位、power down、autoneg enable、speed、duplex
-   -   **reg 1 (BMSR)**：状态寄存器，link up 位、autoneg complete 位
-   -   **reg 2/3 (PHY ID)**：识别 PHY 厂商和型号
-   -   **reg 4/5 (ANAR/ANLPAR)**：本地能力/对端能力广告
-   -   **reg 9/10 (1000-T ctrl/status)**：千兆能力（Master/Slave）

> 后面 mii/mdio 调试全靠这几个寄存器。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/378b5efbe07fb00f261a247771fcb4e9_MD5.jpg]]

_图 4 · PHY 状态迁移与诊断寄存器_

## 四、两大核心模型

网络子系统有两个核心模型：**net\_loop 状态机** 和 **PHY 状态迁移**。

### 4.1 net\_loop 状态机

`net_state` 是 `net_loop()` 的核心变量，可能取值：

```
enum net_loop_state {
    NETLOOP_CONTINUE,   // 继续 poll
    NETLOOP_RESTART,    // 重启当前命令（比如收到 DHCP OFFER 但 lease 过短）
    NETLOOP_SUCCESS,    // 成功，返回给上层命令
    NETLOOP_FAIL,       // 失败（超时、Ctrl-C、协议错误）
};
```

一次典型的 tftpboot 状态迁移：

```
net_loop(TFTPGET) → tftp_start() 发 RRQ → NETLOOP_CONTINUE
  → 每收到 DATA #k，tftp_handler() 发 ACK #k
  → 收到最后一块（< 512B）→ net_state = NETLOOP_SUCCESS
  → net_loop() 返回
```

如果中间任何一步超时，`tftp_timeout_handler` 会重发上一个 ACK（或 RRQ）；重试次数超过 `TIMEOUT_COUNT`（默认 10）就设 `NETLOOP_FAIL`。

DHCP 状态机更长（INIT → SELECTING → REQUESTING → BOUND），逻辑在 `net/bootp.c`。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/984c50113ea2fde642a04f8f8a1299f5_MD5.jpg]]

_图 2 · net\_loop() 状态机与 handler 分发_

### 4.2 PHY 状态迁移

PHY 从上电到 link up 的状态迁移是网络调试的核心：

```
PHY 复位（BMCR bit15）→ Power up（BMCR bit11=0）
  → 检测介质（看 RX signal）→ Autoneg（BMCR bit12=1 触发）
  → Link up（BMSR bit2=1）→ 数据收发
```

三条最典型的调试路径：

1.  1.  **link 起不来** → 看 BMSR bit2；BMSR 读回 0xFFFF 通常是 MDIO 走不通
2.  2.  **autoneg 卡住** → 看 BMSR bit5（Autoneg Complete）、ANLPAR 是否非零（对端广告收到没）
3.  3.  **link up 但 tftp 打不通** → 看速度/双工是否协商成 MAC 期望的（RGMII 时钟依赖速度）

---

## 五、完整数据流：tftpboot 一条命令的旅程

从命令行输入 `tftpboot 0x40000000 Image` 开始，完整数据流分四段。

### 5.1 阶段一：命令解析 + 协议初始化

`cmd/net.c` 里的 `do_tftpb()`：

```
// cmd/net.c 简化
int do_tftpb(struct cmd_tbl *cmdtp, int flag, int argc, char *const argv[])
{
    // 解析参数：load_addr、filename
    ...
    return netboot_common(TFTPGET, cmdtp, argc, argv);
}

// net/net.c
int netboot_common(enum proto_t proto, ...)
{
    // 1. 从 env 拿 ipaddr / serverip / netmask / gatewayip
    net_ip = string_to_ip(env_get("ipaddr"));
    net_server_ip = string_to_ip(env_get("serverip"));
    net_netmask = string_to_ip(env_get("netmask"));

    // 2. 拿当前 ethact
    eth_init();     // 内部走 eth_ops.start()，激活 MAC + PHY

    // 3. 进 net_loop
    ret = net_loop(proto);

    // 4. 清理
    eth_halt();     // eth_ops.stop()
    return ret;
}
```

> ⚠️ `eth_init()` 里最容易踩坑：会调 `phy_startup()` 阻塞等 PHY link up。如果 PHY 状态异常，这里会卡 `Waiting for PHY auto negotiation to complete... TIMEOUT`。

### 5.2 阶段二：ARP 学习

第一次给 serverip 发包之前，U-Boot 不知道对方 MAC。走 ARP：

```
// net/arp.c 简化
void arp_request(void)
{
    // 组包
    struct arp_hdr *arp = ...;
    arp->ar_op = htons(ARPOP_REQUEST);
    arp->ar_sha = my_mac;
    arp->ar_sip = my_ip;
    arp->ar_tha = 00:00:00:00:00:00;
    arp->ar_tip = target_ip;

    // 广播出去
    memset(dst_mac, 0xff, 6);
    eth_send(pkt, len);

    // 设 udp_handler = NULL、arp_wait_reply_ip = target_ip
    // 等 ARP reply
}

void arp_receive(struct in_addr sender_ip, struct ether_addr sender_mac)
{
    if (sender_ip == arp_wait_reply_ip) {
        // 记到 arp_wait_packet_ethaddr、arp_wait_packet_ip
        // 触发上层重发原来那个想发但没 MAC 的包
        net_arp_wait_reply_received();
    }
}
```

> ⚠️ 如果 ARP 超时（默认 5 秒），U-Boot 会打印 `ARP Retry count exceeded; starting again`。**这是 tftp 失败的第一大根因**——netmask 错、serverip 不在同一子网、或者干脆物理层不通。

### 5.3 阶段三：TFTP RRQ/DATA/ACK 循环

ARP 拿到 MAC 后，`tftp_send()` 发 RRQ，之后 Client / Server 之间就进入"发一块 DATA、回一个 ACK"的 lock-step 循环，直到收到一块 < 512B 的 DATA，Client 置 `net_state = NETLOOP_SUCCESS`。

每收到一个 DATA，`tftp_handler()` 会：

1.  1\. 检查 block 号连续
    
2.  2.  `memcpy` 到 `load_addr + (block-1)*512`
3.  3\. 发 ACK
    
4.  4\. block 号 < 512 字节 → 传输完成，`net_state = NETLOOP_SUCCESS`
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/9d5999c93bdc9b5fc2a456be5b99f96e_MD5.jpg]]

_图 3 · tftpboot 完整泳道：ARP → RRQ → DATA/ACK 循环_

`tftp put`（TFTPPUT）反过来：Client 发 WRQ、然后交替发 DATA / 收 ACK。

> ⚠️ **block 号 16 位限制：** 默认 max 65535 × 512B ≈ 32MB。超过要开 `CONFIG_TFTP_BLOCKSIZE_LARGE` 或 windowsize option。

### 5.4 阶段四：DHCP 四次握手（可选）

如果用 `dhcp` 命令而不是 `tftpboot`，第一步会先做 DHCP 四次握手拿地址，OFFER / ACK 里带 `siaddr`（tftp server IP）和 `bootfile`（要拉的镜像名），U-Boot 收到后直接接着 tftpboot，一条命令即起系统——这是 PXE 网络启动的基础。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b40eba317f4c5bcfbff49a6c892487c4_MD5.jpg]]

_图 5 · DHCP 四次握手（U-Boot dhcp 命令内部）_

---

## 六、实战：QEMU 搭调试环境

### 6.1 Windows 宿主 + 真实 tftpd

调 U-Boot 网络最省心的场景：Windows 装 QEMU、装 tftpd64（图形化 tftpd + dhcpd）、再装 tap-windows6（OpenVPN 附带的 TAP 驱动）。这样跑起来的三件事全部真实：真实二层链路、真实 TFTP server、能用 Wireshark 直接抓 tap 网卡上的包。

> 💡 **思路：** QEMU 走 `-netdev tap` 把虚拟网卡接到 Windows 的 tap 网卡上，Windows 主机在同一网段起 tftpd64，U-Boot 里的 `serverip` 就是 Windows 侧 tap 网卡地址。整条链路完全走 Ethernet，不走 QEMU 内建 SLIRP，遇到 ARP/DHCP/TFTP 任何问题都可以在主机侧 Wireshark 里看到真实报文。

**（1） 装 TAP 驱动**：装完 OpenVPN 或单独装 tap-windows6，`ncpa.cpl` 里会多出一个 `TAP-Windows Adapter V9`。改个直观的名字，比如 `tap0`。给它配一个静态 IP，比如 `192.168.100.1 / 255.255.255.0`，网关留空。

**（2） 装并配好 tftpd64**：勾选 TFTP Server（可以顺便勾 DHCP Server 省得 U-Boot 里手动填地址），Server interfaces 选刚才那个 tap0，Base directory 指向存放 `Image` / `u-boot.bin` / `dtb` 的目录。DHCP 段起 `192.168.100.10～192.168.100.20`，`Def. router` 填 `192.168.100.1`（tap0 自己），`Boot File` 填要拉的镜像名（如果要 PXE 自动拉的话）。

### （3） 拉起 QEMU（PowerShell / cmd）：

```
qemu-system-aarch64.exe ^
  -M virt -cpu cortex-a53 -m 512M -nographic ^
  -bios u-boot.bin ^
  -netdev tap,id=n1,ifname=tap0,script=no,downscript=no ^
  -device virtio-net-device,netdev=n1
```

`ifname=tap0` 就是刚才在网卡界面改的名字；`script=no` 让 QEMU 不去调 Linux 那套 `/etc/qemu-ifup`，避免在 Windows 上找不到脚本报错。

### （4） U-Boot 里配环境：

```
=> setenv ipaddr    192.168.100.100
=> setenv netmask   255.255.255.0
=> setenv serverip  192.168.100.1
=> tftpboot 0x40000000 Image
```

DHCP 打开的话直接 `dhcp` 也可以，tftpd64 会把 `ipaddr` / `serverip` / `bootfile` 都塞回来。

**（5） 抓包**：Wireshark 选那张 tap0 网卡直接抓；tftpd64 主界面也会实时打印每一个 RRQ / ACK 事件，出问题时先看它有没有收到 RRQ，就能判断出是链路层不通、ARP 不通还是 TFTP 层协议不对。

> 💡 **Linux 宿主替代方案：** 宿主是 Linux 的话把上面第一、二步换成 `ip tuntap add dev tap0 mode tap user $(whoami)` + 起 `tftpd-hpa`，QEMU 命令完全一致。

### 6.2 环境变量清单

网络相关 env 一览（`printenv | grep -E 'ip|eth|serverip|autoload|bootfile|gatewayip|netmask'`）：

| env 名 | 作用 |
| --- | --- |
| `ipaddr` | 本机 IP |
| `serverip` | tftp/nfs server IP |
| `netmask` | 掩码，决定 ARP 走本网段还是走 gateway |
| `gatewayip` | 默认网关，跨网段时用 |
| `ethaddr`/ `eth1addr` / ... | 各网口 MAC（一般烧到 OTP/EFUSE，boot 时读进来） |
| `ethact` | 当前使用的网口名（多网口切换） |
| `autoload` | `yes`时 dhcp 拿到地址自动 tftp bootfile；`no` 只拿地址 |
| `bootfile` | dhcp 自动拉的文件名 |
| `netretry` | 网络重试策略（`no` / `once` / `always`） |
| `tftpblocksize` | TFTP block 大小，默认 1468（受 MTU 影响） |
| `tftptimeout` | ms 级重试间隔 |

> 调试时最常改的三个：
> 
> ```
> ipaddr
> serverip
> netmask
> ```
> 
> 。改完直接下条命令就生效，不用 saveenv。

### 6.3 常用网络命令速查

```
=> net list           # 列出所有 eth udevice
   eth0 : ethernet@a003e00 00:11:22:33:44:55 active
   eth1 : ethernet@a004000 00:11:22:33:44:56

=> setenv ethact eth1 # 切当前网口

=> ping 192.168.1.1   # ICMP 测试
=> tftpboot 0x40000000 Image
=> nfs   0x40000000 192.168.1.1:/srv/nfs/Image
=> dhcp
=> dhcp 0x40000000 tftp://192.168.1.1/Image
```

---

## 七、mii / mdio / PHY 调试（核心章节）

> ⚠️ 网络调不通、link 起不来、协商速率对不上——所有物理层问题都可以用 `mii` / `mdio` 这两个命令诊断。**这一节是本文的核心。**

### 7.1 先分清 mii 和 mdio 是两回事

历史上 U-Boot 有两套命令：

-   -   **mii**：老命令（`cmd/mii.c`），基于 legacy `miiphy_*` 接口，只支持 clause 22
-   -   **mdio**：新命令（`cmd/mdio.c`），基于 DM 化的 `mii_dev` 接口，支持 clause 22 和 clause 45（10G/内部 PMA）

> 💡 **新板子建议直接用 mdio。** 老板子 driver 没 DM 化时才用 `mii`。两者功能重叠，寄存器结果一致，只是接口不同。

### 7.2 mii 命令用法

```
=> mii help
mii device                          - list available devices
mii device <devname>                - set current device
mii info    <addr>                  - display MII PHY info
mii read    <addr> <reg>            - read PHY reg
mii write   <addr> <reg> <data>     - write PHY reg
mii dump    <addr> <reg>            - dump named PHY reg fields
mii list                            - list all PHYs on all busses
```

典型工作流：

```
=> mii info
PHY 0x00: OUI = 0x0885, Model = 0x22, Rev = 0x01,
          10baseT, HDX, 100baseTX, HDX, FDX
          link up, speed 1000, full duplex

=> mii read 0 1          # 读 BMSR
0x796d

=> mii dump 0 1
0.  (796d)                 -- PHY control register --
    (8000:0000) 0.15    =     0    reset
    (4000:0000) 0.14    =     0    loopback
    (2040:2000) 0. 6,13 =   b01    speed selection = 100 Mbps
    (1000:1000) 0.12    =     1    A/N enable
    ...
```

`mii dump` 输出人性化，直接告诉你每个 bit 什么意思——**不用查数据手册**。

### 7.3 mdio 命令用法

```
=> mdio help
mdio list                                - list all bus/PHY
mdio read  <bus> <addr>[.<devad>] <reg>  - read
mdio write <bus> <addr>[.<devad>] <reg> <data>
mdio rx    <bus>                          - reset PHYs on bus
```

devad 是 clause 45 才用的（比如 `.1` = PMA/PMD）。clause 22 省略即可。

```
=> mdio list
mdio@fd07b000:
  0 - Realtek RTL8211F <--> ethernet@fd07b000

=> mdio read mdio@fd07b000 0 1
Reading from bus mdio@fd07b000
PHY at address 0:
1 - 0x796d
```

### 7.4 症状 → 诊断路径

#### 症状 A：mii info 全是 0 或 0xFFFF

```
=> mii info
PHY 0x00: OUI = 0xFFFF, Model = 0x3F, ...
   link down
```

**结论：MDIO 走不通。** 三种可能：

1.  1.  **PHY addr 错**：设备树里 `reg = <0>` 但实际 PHY 在 addr 3。用 `mii list` 扫全网段，或者直接对 addr 0～31 都 `mii read N 2`，看哪个不是 0xFFFF：
        
        ```
        => for i in 0 1 2 3 4 5; do echo -n addr $i:; mii read $i 2; done
        ```
        
2.  2.  **MDIO 时钟没开**：MAC 驱动 `probe()` 里应该 `clk_enable(mdio_clk)`。SoC datasheet 会说明 MDIO 时钟频率上限（一般 2.5MHz），太快 PHY 采样失败。
3.  3.  **PHY 没上电或复位**：GPIO 控制的 PHY reset 脚没释放。DTS 里 `phy-reset-gpios` 是否声明、release-delay 够不够。

#### 症状 B：BMSR link bit 一直是 0

```
=> mii read 0 1
0x7849       # bit2 = 0，link down
```

再看 autoneg 状态：

```
=> mii read 0 1
0x7849
=> mii dump 0 1
1.  (7849) ...
    (0020:0000) 1. 5    =     0    Auto-negotiation complete
    (0004:0000) 1. 2    =     0    Link status
```

两个 bit 都是 0，说明协商没起来。看两端广告：

```
=> mii read 0 4          # ANAR - 本地广告
0x01e1                   # 100Base-TX FDX, 100Base-TX HDX, 10Base FDX, 10Base HDX, 802.3

=> mii read 0 5          # ANLPAR - 对端广告
0x0000                   # 对端啥都没广告 → 对端 PHY 或线缆有问题
```

ANLPAR = 0 通常是**没有对端**（网线没插好、对端设备没开、接的是不支持自动协商的老 hub）。ANLPAR 有值但 link 起不来通常是**能力交集为空**（一端只 10M-HDX、另一端只 100M-FDX）。

#### 症状 C：link up 了但 tftp 打不通

```
=> mii info
   link up, speed 1000, full duplex

=> ping 192.168.1.1
Using ethernet@... device
ping failed; host 192.168.1.1 is not alive
```

link 有了，包发不出去/收不进来。查几个方向：

1.  1.  **RGMII/RMII delay 错了**：千兆 RGMII 要 clock skew（2ns 左右），有的 PHY 内置延迟、有的靠 PCB trace、有的靠 MAC。三方对不上包会全丢或校验错。诊断：先降速到 100M `mii write 0 0 0x2100`（强制 100M-FDX），如果 100M 通、1000M 不通，就是延迟没配对。
2.  2.  **PHY 内部延迟寄存器**：以 Micrel KSZ9031 为例，clause 22 之外还有 MMD 3.4 / 3.5 / 3.6 / 3.8 控制 RGMII skew：
        
        ```
        => mii write 0 0x0d 0x0002    # 选中 MMD device 2
        => mii write 0 0x0e 0x0004    # register address 4
        => mii write 0 0x0d 0x4002    # data mode
        => mii write 0 0x0e 0x0000    # 写 clock skew
        ```
        
        麻烦但常用。DTS 里 `rxc-skew-ps` / `txc-skew-ps` 属性走的就是这套流程。
        
3.  3.  **MAC 侧速度没跟上 PHY**：PHY 协商成 1000M 但 MAC 还配在 100M。检查 MAC 驱动的 `adjust_link()` 有没有在 PHY link change 时同步更新 MAC speed 寄存器。

#### 症状 D：Waiting for PHY auto negotiation to complete... TIMEOUT

> ⚠️ 这条最常见。`eth_init()` 里 `phy_startup()` 阻塞等 autoneg 完成（`BMSR bit5 = 1`），默认 4 秒超时。

排查步骤：

```
=> mii read 0 0         # BMCR
=> mii read 0 1         # BMSR
=> mii read 0 4         # ANAR
=> mii read 0 5         # ANLPAR
```

组合看：

-   • BMCR bit12 = 0 → autoneg disable，手动强制模式；正常应该是 1
    
-   • BMSR bit2 = 1 但 bit5 = 0 → 强制模式成功，autoneg 未启动
    
-   • BMSR bit2 = 0 且 bit5 = 0 → 物理层没起来（回到症状 A/B）
    
-   • 反复读 BMSR bit5，慢慢从 0 变 1 但过程超过 4s → 协商真的慢（某些老 PHY）：加 `CONFIG_PHY_AN_TIMEOUT` 或 boot 时先 sleep 一段再 phy\_startup
    

### 7.5 PHY 驱动加载确认

有时候通用 phy 驱动能识别但厂商特殊 fixup 不生效（比如 RGMII 延迟）。看当前 PHY 是不是走对了 driver：

```
=> mdio list
mdio@...:
  0 - Micrel KSZ9031 <-->  ethernet@...    ✓ 厂商驱动
```

如果显示 `Generic PHY` 就要检查 `drivers/net/phy/Kconfig` 里对应的 `CONFIG_PHY_MICREL` / `CONFIG_PHY_ATHEROS` 有没有 `y`，以及 PHY ID 是否被驱动的 phy\_driver 表覆盖（reg 2/3 读出来的值要匹配 driver 里的 uid mask）。

### 7.6 把常用诊断拼成一条命令

每次问题冒出来，最少需要看这几个信息才能定位到层：MDIO 总线是否通、PHY 是否被识别、BMCR / BMSR 里 autoneg 与 link 是什么状态、本地和对端广告是否重叠、千兆能力是否对齐。把这一串按固定顺序读出来，是把网络故障从"不通"收敛到"哪一层不通"的最短路径。

> 💡 **思路：** 用 U-Boot 的 env 变量把 `mii list` → `mii info` → `mii dump 0 0/1` → `mii read 0 4/5/9/10` 串成一条命令，遇到问题 `run` 一下，输出直接对照上面"症状 A/B/C/D"四类做定位。哪一层的信号异常，就往哪一层深挖，不用每次现敲。

---

## 八、总结

U-Boot 网络子系统做的三件事：**收发 UDP 包、协商 PHY 速率、把命令行接到 protocol handler**。三件事对应三层设计：

-   -   **命令层** = `cmd/net.c`、`cmd/mii.c`、`cmd/mdio.c`：用户接口
-   -   **协议层** = `net_loop()` + handler 回调：ARP / ICMP / UDP / DHCP / TFTP / NFS
-   -   **驱动层** = eth\_ops 五函数 + MDIO/PHY 独立子系统

按故障层次划分，调试手段也分三档：

| 症状 | 关注点 | 关键命令/寄存器 |
| --- | --- | --- |
| `ARP Retry count exceeded` | 三层配置：ip/netmask/serverip | `printenv`、`ping` |
| `Waiting for PHY autoneg... TIMEOUT` | PHY autoneg / link | `mii info`、`mii dump 0 1` |
| link up 但 tftp 卡 | RGMII delay / MAC-PHY 速率不同步 | `mii read 0 0/1/9/10`、降速试 100M |
| `mii info`全 0xFFFF | MDIO 走不通 | `mii list`、检查 PHY addr / 时钟 / reset |
| `Generic PHY`而不是厂商 driver | PHY ID 未匹配 | `mii read 0 2/3`、`CONFIG_PHY_*` |

> 🔑 **MDIO / PHY 那一层是 U-Boot 网络调试最容易出问题也最容易被忽视的一层：** MAC 驱动写得再对，PHY 上不来一切白搭。养成 `mii info` + `mii dump` 三步走的诊断习惯，比看几百行内核网络栈更快定位 bringup 阶段的问题。

到这一篇为止，U-Boot 系列的骨架就完整了：从 `_start` 汇编入口，走 SPL/TPL DDR bringup，进 DM 设备模型，读 env 决定 bootcmd，用网络子系统拉 kernel / dtb，最后走 bootm / booti / bootelf 交给操作系统。

---

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/344c3db9_1784073396665?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwOTUxMzQzOA%3D%3D%26mid%3D2247484377%26idx%3D1%26sn%3Db57a04b2a6e33e07760d2bfda5ea85cb%26chksm%3Dc035ee510e2f1619c6418bda77cfb30c1ef44d030cf0342eeff6ffaed478d1063c894e35482a%26mpshare%3D1%26scene%3D1%26srcid%3D0715c0kXhwHnpqukOi7MzNRO%26sharer_shareinfo%3D965b3190d22d2d61ba77aa79248012ba%26sharer_shareinfo_first%3D965b3190d22d2d61ba77aa79248012ba%23rd&s=obsidian)