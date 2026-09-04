---
author: 疯狂的兔子tommy
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk0MzY3NTQzOQ==&mid=2247489525&idx=1&sn=68f0b66ce178557dc649d7f847478d0f&chksm=c286ccc99fc1f94b63611a809f4615bb737ba925e82eb05b10c0d229b0e97c94fb48dabfc810&mpshare=1&scene=1&srcid=0831TKSvO3B2kOPcp1rZH27E&sharer_shareinfo=4bb0d1341f913ccc3a8153dba704d2b7&sharer_shareinfo_first=4bb0d1341f913ccc3a8153dba704d2b7#rd
saved: 2026-08-31 08:38:41
tags:
  - 笔记同步助手
id: 13a26118-a7d4-46ad-8f52-ef2b87be32db
---

公众号名称：算力网络架构手记

作者名称：疯狂的兔子tommy

发布时间：2026-08-31 08:08

这里是**「算力网络架构手记」****北京**

  

-   **专注 AI 集群网络架构与性能优化**
    
-   **深入 GPU×RoCE×NCCL×K8s 跨层瓶颈**
    
-   **只拆真实问题，不写概念科普**
    

---

  

👇 试听内容

![](https://findermp.video.qq.com/251/20304/stodownload?encfilekey=rjD5jyTuFrIpZ2ibE8T7YmwgiahniaXswqzAoNNwiaBib5cZoMupOic5Gz3NDh6fbsZAiaRTjbJD5DJGgOkXLibiagcBic8H3oJBKVpyH8ml6OFWqHgPggBBCicvmTKXg&token=cztXnd9GyrGGXeic6lBU9RRAFJsCPHOq8kqvVGDwhsKVibKRzuCVB9fWLDMjT1wEQddf1jHE62xQrC8a3waIY4CtAT1CnR4Ns0PibUiayBxMyaxeClOtJwkYwEGIa9RwHxoibAmVmT8HS2GZAee40hhkeYxJgp2FiaLV5ddk7Z2uat8HefyFiak4uwbodgWB2Feh0yzq2tXGeeY28TCDjVAicQNhJcFDn2lYHwUxicW2SFsa3Niak&hy=SH&idx=1&m=&scene=2&uzid=1&wxampicformat=503&picformat=200)

> 📹 视频内容（上图为封面），请前往原文观看：[在公众号原文中观看](https://mp.weixin.qq.com/s?__biz=Mzk0MzY3NTQzOQ==&mid=2247489525&idx=1&sn=68f0b66ce178557dc649d7f847478d0f&chksm=c286ccc99fc1f94b63611a809f4615bb737ba925e82eb05b10c0d229b0e97c94fb48dabfc810&mpshare=1&scene=1&srcid=0831TKSvO3B2kOPcp1rZH27E&sharer_shareinfo=4bb0d1341f913ccc3a8153dba704d2b7&sharer_shareinfo_first=4bb0d1341f913ccc3a8153dba704d2b7#rd)

  

---

<u>00</u>

GID 表不是小细节，它决定 RoCE QP 用什么身份上路

很多人在排 RoCE 的时候，只盯几个东西：

-   网卡是不是 Up
    
-   光模块有没有误码
    
-   MTU 对不对
    
-   PFC 有没有开
    
-   ECN 有没有打
    
-   CNP 有没有涨
    
-   NCCL 有没有走 NET/IB
    

这些都重要。

但还有一张表，很多人一直忽略。

它就是：

**GID Table**

GID，全称是 **Global Identifier**。

在 InfiniBand 里，GID 是全局标识。  
在 RoCE 里，它承担了一个更工程化的角色：

把 RDMA 世界里的“源 GID”，和以太网世界里的 **IP、netdev、RoCE 类型、VLAN** 关联起来。

```
NVIDIA RoCE 文档明确说明：RoCE 端口的 GID 表项会在 NIC 端口关联的 Ethernet 设备配置 IP 地址时创建；每个表项包含 GID value、GID type 和 network device；GID 表也通过 sysfs 暴露给用户空间。
```

所以，不要把 GID Index 当成一个随便填的数字。

它背后代表的是：

-   哪张 HCA
    
-   哪个 Port
    
-   哪个 netdev
    
-   哪个 IP
    
-   哪种 RoCE 类型
    
-   是否带 VLAN
    
-   QP 出去时用什么源身份
    

  

> GID 表是 RoCE 的“身份索引表”：QP 选错 GID Index，就可能等于拿错身份证上路。

这也是为什么有些 RoCE 问题看起来像网络不通，实际根因只是 GID Index 选错了。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d50a1c1c0bd363704c4b46e3cd529b92_MD5.jpg]]

<u>01</u>

为什么 RoCE 必须有 GID？它不是已经有 IP 了吗？

这是第一个常见问题。

RoCEv2 不是跑在 IP/UDP 上吗？

既然已经有源 IP、目的 IP，为什么还要 GID？

因为 RoCE 不是普通 UDP 应用。

它是把 RDMA 传输语义放到以太网上运行。

上层 RDMA 栈仍然要面对：

-   QP
    
-   Address Vector
    
-   源 GID
    
-   目的 GID
    
-   RoCE Type
    
-   路径信息
    
-   GID Index
    

  

```
RoCEv1 使用专用 EtherType；RoCEv2 则使用 IP header 和 UDP header，UDP 目的端口为 4791，UDP 源端口可作为不透明 flow identifier，帮助网络设备做 ECMP 等转发优化。
```

所以在 RoCEv2 里，IP 的确在报文线上出现了。

但 RDMA Verbs / QP 层仍然需要一个 GID 来表达源端和目的端的 RDMA 地址身份。

可以这样理解：

-   IP 是以太网/IP 网络看到的地址
    
-   GID 是 RDMA 栈用来表达路径和端点身份的地址对象
    
-   GID Index 是本地 HCA GID 表里的索引
    
-   QP 需要通过这个索引找到自己应该使用的源 GID
    

  

> RoCEv2 用 IP/UDP 封装，不代表 RDMA 栈不再需要 GID。

这就是很多人混淆的地方。

你看到的是 IP。  
HCA 和 Verbs 看到的是 GID、GID Index 和 QP 属性。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/92da2e2c54b4911b53e5e76b5dac06da_MD5.jpg]]

<u>02</u>

GID 表到底从哪里来？

很多人以为：

GID 表是网卡固件里固定写死的。

不完全是。

```
在 RoCE 环境中，GID 表会随着 netdev 上的 IP 地址变化而变化。
NVIDIA 文档说明，RoCE 端口的 GID 表项会在对应 Ethernet device 配置 IP 地址时创建；表项字段包括 GID value、GID type 和 network device。
```

这意味着：

你给 eth10 配一个 IP。  
对应 RDMA Port 的 GID 表里就可能出现相关 GID 项。

你给 eth10.100 配一个 VLAN 子接口，并配置 IP。  
GID 表里也可能出现与这个 VLAN netdev 关联的 GID 项。

你删除 IP。  
GID 表也可能发生变化。

你新增 IPv6。  
GID 表里也可能增加对应 IPv6 GID。

所以 GID 表不是静态背景信息。

它和 Linux 侧网络配置强相关。

  

> RoCE 的 GID 表，是 RDMA HCA 对 Linux netdev/IP/VLAN 状态的一种映射结果。

这也解释了一个现场现象：

昨天 `NCCL_IB_GID_INDEX=3` 能跑。  
今天改了 IP、加了 VLAN、换了网卡命名，突然不行了。

不是 RoCE 莫名其妙。

而是 GID 表可能已经变了。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/37c7efa60918a53b771730cecbd4df3c_MD5.jpg]]

<u>03</u>

一张 GID 表里到底有什么？

一个典型 GID 表，不只是一个 GID 值列表。

它至少要看四个字段：

-   **Index**
    
-   **GID Value**
    
-   **Type**
    
-   **Netdev**
    

  

```
NVIDIA 文档中给出的 RoCE GID 表字段也是这几个核心项：GID value、GID type、network device，并指出 GID value 可以从 /sys/class/infiniband/{device}/ports/{port}/gids/{index}读取，GID type 可以从gid_attrs/types/{index}读取，关联 netdev 可以从gid_attrs/ndevs/{index} 读取。
```

工程上最常见的显示方式是：

```
show_gids
```

你可能看到类似：

```
DEV     PORT  INDEX  GID                                      IPv4         VER  DEV
mlx5_0  1     0      fe80:0000:0000:0000:....                 --           v1   eth10
mlx5_0  1     1      fe80:0000:0000:0000:....                 --           v2   eth10
mlx5_0  1     2      0000:0000:0000:0000:0000:ffff:0a0a:000b  10.10.0.11   v1   eth10
mlx5_0  1     3      0000:0000:0000:0000:0000:ffff:0a0a:000b  10.10.0.11   v2   eth10
```

注意这里有几个细节。

同一个 GID Value，可能出现两次。

一次是 RoCE v1。  
一次是 RoCE v2。

同一个 netdev，可能有多个 GID：

-   一个是 Link-local
    
-   一个是 IPv4-mapped IPv6
    
-   一个是 IPv6
    
-   一个是 VLAN 子接口对应的 GID
    

  

同一个 HCA port，也可能对应多个 netdev。

所以不能看到 `mlx5_0` 就结束。

还要看：

-   Index 是多少
    
-   Type 是 v1 还是 v2
    
-   netdev 是 eth10 还是 eth10.100
    
-   IPv4 是否是你要的源 IP
    
-   是否对应正确 VLAN
    

  

> 看 GID 表不能只看 GID 值，要同时看 Index、Type、netdev 和 IP/VLAN。

这才是真正能用于排障的 GID 表。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ee3480c0db55bc6601724a6e4a2cff93_MD5.jpg]]

<u>04</u>

RoCEv1 和 RoCEv2，在 GID 表里为什么会同时出现？

很多环境里，你会看到同一个 IP 对应两个 GID Index：

-   一个 Type 是 v1
    
-   一个 Type 是 v2
    

  

这不是重复。

这是两种不同 RoCE 封装。

```
RoCEv1 使用 Ethernet 专用 EtherType，不能跨三层 IP 路由。
RoCEv2 使用 IP/UDP 封装，可以运行在三层 IP 网络里。NVIDIA 文档也把 RoCEv1 描述为专用 EtherType 0x8915，把 RoCEv2 描述为带 IP header 和 UDP header 的可路由封装。
```

所以同一个 GID Value，对应 v1 和 v2，含义并不一样：

-   选择 v1 Index，QP 会使用 RoCEv1 类型。
    
-   选择 v2 Index，QP 会使用 RoCEv2 类型。
    

  

```
NVIDIA 文档明确说明，在修改 RC/UC QP 从 INIT 到 RTR 时，Address Vector 需要指定端口 GID 表中的 source GID index；该 index 对应的 GID type 会用于设置 QP 的 RoCE 类型。
```

这句话非常关键。

它说明：

GID Index 不只是选一个地址。  
还在选 RoCE 类型。

  

> 选错 GID Index，可能不是 IP 选错，而是 RoCEv1/v2 类型选错。

如果你的网络是 RoCEv2 三层网络，却选到了 v1 GID Index，后面建连失败并不奇怪。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7235b0725c74d9536ae0203aa8a7737e_MD5.jpg]]

<u>05</u>

VLAN 子接口为什么也会出现在 GID 表里？

RoCE 网络里经常会有 VLAN。

比如：

```
eth10      → 未打 VLAN
eth10.100  → VLAN 100
eth10.200  → VLAN 200
```

如果你在 `eth10.100` 上配置了 IP，那么 GID 表里可能出现与 `eth10.100` 关联的 GID。

```
NVIDIA 文档的示例说明，端口 GID 表中的某些 index 属于 VLAN 子接口；与这些 GID Index 关联的 QP 发送出的包会携带对应 VLAN Header。
```

这意味着：

GID Index 也可能间接决定：

-   报文是否带 VLAN
    
-   带哪个 VLAN
    
-   使用哪个 netdev
    
-   走哪个二层广播域
    

  

所以在带 VLAN 的 RoCE 环境中，如果 GID Index 选错，可能出现：

-   QP 使用了未打 VLAN 的 GID
    
-   报文没有带预期 VLAN
    
-   交换机端口丢弃
    
-   PFC/ECN 优先级映射不生效
    
-   对端根本收不到
    

  

或者反过来：

-   你本来想走裸接口
    
-   却选到了 VLAN 子接口 GID
    
-   包被打上 VLAN
    
-   对端或交换机不接受
    

  

> 在 RoCE 里，GID Index 选错，有时就是 VLAN 选错。

这就是为什么排查时一定要看 `Netdev` 列。

不要只看 `mlx5_0`。  
也不要只看 IP。  
要看它是不是你真正想用的那个 netdev。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/2d31c4a21caeb6cc24282c353d08d24a_MD5.jpg]]

<u>06</u>

RDMA-CM 为什么特别依赖 GID 表？

很多应用不是手工 `ibv_modify_qp()`。

它们用 RDMA-CM 建连。

RDMA-CM 的体验更像 socket：

-   给一个对端 IP
    
-   让它帮你解析地址
    
-   解析路由
    
-   创建连接
    

  

但在 RoCE 里，RDMA-CM 不能凭空建连。

它仍然要选择本地源 GID。

```
NVIDIA 文档说明，RDMA-CM 主动端只需要传入被动端 IP；RDMA-CM 会决定使用哪个源 GID，并从 GID 表中获取它。由于同一个 GID value 可能有多个实例，查找还要考虑 GID type。
```

这就解释了很多现场问题。

应用传的是：

```
10.13.0.25
```

你以为它自然走：

```
eth13 / mlx5_3 / RoCEv2 / VLAN 300
```

  

但 RDMA-CM 可能根据本机路由、源地址、GID 表、RoCE type 配置，最终选了别的 GID。

于是出现：

地址能 ping。  
但 RDMA-CM connect 失败。

或者：

server 监听的是 eth13 的地址。  
client 解析后却选到了 eth10 相关 GID。

  

> RDMA-CM 不是绕开 GID 表，而是自动帮你从 GID 表里选一个源身份。

自动选择有便利性。

也有风险。

尤其在多网卡、多 IP、多 VLAN、多 VRF 的 GPU 服务器里，如果主机路由没有规划好，RDMA-CM 很可能帮你“自动选错”。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0d83f4db811f5ec79b8586053a95fbdc_MD5.jpg]]

<u>07</u>

手写 Verbs 程序为什么必须关心 `sgid_index`？

如果你不用 RDMA-CM，而是自己写 Verbs 建连，就更要关心 GID Index。

RC QP 从 INIT 转 RTR 时，需要填 Address Vector。

在 RoCE 场景里，AV 中有一个关键字段：

```
sgid_index
```

也就是 Source GID Index。

```
NVIDIA 文档明确说明，修改 RC/UC QP 从 INIT 到 RTR 时，Address Vector 需要指定 port GID table 里的 source GID index，该 index 对应的 GID type 会决定 QP 的 RoCE type。
```

这意味着：

你写错 `sgid_index`，QP 属性就错。

可能表现为：

-   `ibv_modify_qp()` 返回 Invalid Argument
    
-   QP 建立失败
    
-   本端认为已经发起连接
    
-   对端没有收到预期类型报文
    
-   抓包发现 RoCE 类型不对
    
-   VLAN 不对
    
-   源 IP 不对
    

  

所以不要把 `-x <gid_index>` 当成 perftest 的“可有可无参数”。

在 RoCE 多 GID 环境里，它非常关键。

比如：

```
ib_write_bw -d mlx5_3 -i 1 -x 3 <server-ip>
```

这里的 `-x 3` 很可能就是在告诉 perftest：

使用 GID Index 3。

如果 Index 3 是：

```
RoCEv2 + eth13 + 10.13.0.11
```

```
那是你想要的。
```

如果 Index 3 是：

```
RoCEv1 + eth13
```

或者：

```
RoCEv2 + eth13.100
```

那就完全不一样。

  

> 手写 Verbs 或 perftest 里，`sgid_index` 不是装饰参数，而是 QP 的 RoCE 身份选择器。

  

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3d055643bffcabd032ebf8f8fdf2b3ac_MD5.jpg]]

<u>08</u>

GID、GUID、MAC、IP：这几个“身份”千万别混

RoCE 排障时，经常会把几个身份混在一起。

### GUID

GUID 是 RDMA 设备或端口的全球唯一标识。  
它更像硬件身份证。

### MAC

MAC 是以太网二层地址。  
交换机二层转发看它。

### IP

IP 是三层地址。  
RoCEv2 会使用 IP/UDP 封装。

### GID

GID 是 RDMA 地址体系里的全局标识。  
在 RoCE 里，它和 netdev、IP、RoCE type、VLAN 等形成映射关系。

所以不能这样说：

“GID 就是 IP。”

更不能这样说：

“GID 就是 MAC。”

```
对于 IPv4 RoCEv2，GID value 常以 IPv4-mapped IPv6 地址形式呈现，例如 ::ffff:10.10.0.11 这种格式；NVIDIA 文档也说明 IPv4 GID 是 IPv4-mapped IPv6 地址，而 IPv6 GID 则是 IPv6 地址本身。
```

但它仍然是 RDMA GID 表项中的值。

不是普通应用直接使用的 IP socket 地址。

> GUID 是设备身份，MAC 是二层身份，IP 是三层身份，GID 是 RDMA 端点身份。

这几个身份之间可以有关联。

但绝不能画等号。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1a4e1cb167c1c7a863c412c8cd638b45_MD5.jpg]]

<u>09</u>

为什么 NCCL 经常因为 GID Index 出问题？

NCCL 走 RoCE 时，底层需要建立 RDMA 通信。

这时就绕不开：

-   HCA
    
-   Port
    
-   GID
    
-   GID Index
    
-   RoCE Type
    
-   Traffic Class
    
-   Rail
    

  

```
NCCL 官方环境变量文档明确说明：NCCL_IB_HCA 用于选择 RDMA HCA 接口，NCCL_IB_GID_INDEX 定义 RoCE 模式下使用的 Global ID index。
```

所以，很多 NCCL 问题：

-   不是算法问题
    
-   不是 Ring 不行
    
-   不是 Tree 不行
    
-   不是 GPU 不行
    

而是建连时选错了 GID。

```
NVIDIA NCCL 网络排障文档也明确提到，RoCE 中常见问题之一就是选择了错误的 GID index；这可能导致 ibv_modify_qp failed with error Invalid argument，旧版本需要通过 show_gids 查看并设置 NCCL_IB_GID_INDEX。
```

但这里还要补一个现代版本的注意点：

```
从 NCCL 2.21 及以后版本开始，NCCL 支持动态选择 GID；官方排障文档说明，在 NCCL 2.21 及以后，通常不应再手工设置 NCCL_IB_GID_INDEX，而应让 NCCL 动态选择，并可通过地址族、地址范围和 RoCE 版本变量约束选择结果。
```

  

> 旧 NCCL 时代，GID Index 可能要手工指定；新 NCCL 时代，更重要的是让动态选择的范围和你的网络规划一致。

所以现在排 NCCL，不是永远加：

```
-x NCCL_IB_GID_INDEX=3
```

  

而是先要知道：

-   当前 NCCL 版本是多少？
    
-   它是否已经支持动态 GID 选择？
    
-   动态选择是否被错误环境变量覆盖？
    
-   GID 表里哪个 index 才对应正确 IP、netdev、RoCEv2 和 VLAN？
    
-   是否应该用 NCCL\_IB\_ADDR\_FAMILY、NCCL\_IB\_ADDR\_RANGE、
    

`NCCL_IB_ROCE_VERSION_NUM` 做约束？

![[Inbox/笔记同步助手/微信公众号/2026/08/images/116b08cb257160a3e277c10987fc9a8d_MD5.jpg]]

<u>10</u>

GID 表和多 Rail：为什么 mlx5\_3 对了，GID 仍可能错？

多 Rail GPU 服务器里，经常这样规划：

```
mlx5_0 / eth10 → Rail0
mlx5_1 / eth11 → Rail1
mlx5_2 / eth12 → Rail2
mlx5_3 / eth13 → Rail3
```

你设置了：

```
NCCL_IB_HCA=mlx5_3
```

然后以为一切都对了。

不一定。

因为 HCA 选对，只表示你选中了 `mlx5_3` 这个 RDMA 设备。

但这个设备的 GID 表里可能有多个 Index：

-   Link-local v1
    
-   Link-local v2
    
-   IPv4 eth13 v1
    
-   IPv4 eth13 v2
    
-   IPv4 eth13.300 v1
    
-   IPv4 eth13.300 v2
    
-   IPv6 v2
    

  

你真正想用的可能是：

```
mlx5_3 / port1 / GID Index 5 / RoCEv2 / eth13.300 / 10.13.0.11
```

但程序实际选到的可能是：

```
mlx5_3 / port1 / GID Index 3 / RoCEv2 / eth13 / 10.13.0.11
```

```
或者更糟：
```

```
mlx5_3 / port1 / GID Index 2 / RoCEv1 / eth13
```

于是就会出现：

HCA 对。  
Port 对。  
但报文的 RoCE 类型、VLAN 或源地址不对。

  

> 多 Rail 里，HCA 只是第一层选择；GID Index 才决定这个 HCA 以哪个 netdev/IP/VLAN 身份发包。

这就是为什么排查时要建立映射表：

```
Rail → HCA → Port → netdev → IP → VLAN → GID Index → RoCE Type
```

```
缺任何一列，都可能误判。
```

  

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7ef4b7e8d68bfc03e520dce703b67162_MD5.jpg]]

<u>11</u>

GID 表和 VRF/容器：设备给了，netdev 没给，也可能失败

在 Kubernetes、容器和虚拟化环境里，GID 问题更隐蔽。

你可能把 RDMA 设备暴露给了容器：

```
mlx5_3
```

容器里也能看到：

```
ibv_devinfo
```

但这还不够。

如果对应的 netdev、IP 或网络命名空间关系不正确，RoCE 的 GID 解析可能仍然出问题。

为什么？

因为 RoCE GID 表项与 network device 关联。NVIDIA 文档明确指出，GID 表项里的 network device 是该 GID 所关联 IP 地址所在的 Ethernet device。

这意味着：

```
RDMA Device 和 netdev 不能割裂看。
```

在容器里可能出现：

-   RDMA 设备在
    
-   但对应 netdev 不在容器 Network Namespace
    
-   或者 IP 在宿主机
    
-   或者 GID 表看到的是宿主机 netdev
    
-   或者应用在容器内查不到正确接口
    
-   或者 Multus/macvlan/SR-IOV VF 与 RDMA device 没有对应上
    

  

> RoCE 容器化不能只问“有没有 /dev/infiniband”，还要问“这个 RDMA 设备对应的 netdev、IP 和 GID 是否在同一个可用网络语境里”。

这也是为什么容器里的 NCCL、UCX、vLLM、Mooncake 等 RDMA 应用，必须同时核对：

-   RDMA device
    
-   netdev
    
-   IP
    
-   GID Table
    
-   Network Namespace
    
-   VRF
    
-   路由表
    
-   Pod 的接口配置
    

不是设备能看见就一定能建连。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5438d59f6c55481cdd21946a50f9c33d_MD5.jpg]]

<u>12</u>

GID Index 选错，会表现成哪些奇怪现象？

GID 问题最折磨人的地方，是它经常不报“GID 错”。

它会伪装成很多别的问题。

### 现象一：`ibv_modify_qp` 失败

尤其是：

```
Invalid argument
```

这可能是 QP 属性、AV、GID Index 或 RoCE Type 不匹配。

NCCL 官方排障文档就把错误 GID index 与 `ibv_modify_qp failed with error Invalid argument` 联系在一起。

### 现象二：`rping` 失败，但 ping 正常

ping 只验证 IP 可达。  
RDMA-CM 还要选源 GID、GID type 和 RDMA device。

### 现象三：`ib_write_bw` 不指定 `-x` 失败，指定后成功

说明默认选择的 GID 不是你想用的。

### 现象四：VLAN 网络不通

可能选到了非 VLAN netdev 对应 GID。

### 现象五：跨三层不通

可能选到了 RoCEv1 GID，而不是 RoCEv2 GID。

### 现象六：NCCL 某些节点能跑，某些节点 Hang

可能节点之间 GID 表顺序不一致，或者某些节点 IP/VLAN 配置多了少了。

### 现象七：换了网卡、改了 IP、重启后故障变化

可能 GID 表被重新生成，Index 对应关系变了。

  

> GID 错误经常不会显示为“GID 错误”，而是表现为 QP Modify、RDMA-CM、NCCL 初始化或 VLAN/RoCE 类型异常。

所以现场排障要主动查 GID。

不要等日志里写出来。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1d39418e335274e725b09dd8eaa27259_MD5.jpg]]

<u>13</u>

怎么查 GID 表？

现场最常用命令是：

```
show_gids
```

但为了确认底层对应关系，建议再看 sysfs：

  

### 1）看 GID 值

```
cat /sys/class/infiniband/mlx5_3/ports/1/gids/3
```

### 2）看 GID 类型

```
cat /sys/class/infiniband/mlx5_3/ports/1/gid_attrs/types/3
```

### 3）看关联 netdev

```
cat /sys/class/infiniband/mlx5_3/ports/1/gid_attrs/ndevs/3
```

这些路径正是 NVIDIA RoCE 文档列出的 GID table sysfs 暴露位置。

  

### 4）看 RDMA 设备到 netdev 映射

```
ibdev2netdev
rdma link
```

### 5）看 netdev 上的 IP 和 VLAN

```
ip -br addr show eth13
ip -d link show eth13
ip -d link show eth13.300
```

### 6）看 GID 是否是你想要的 IPv4-mapped IPv6

例如：

```
0000:0000:0000:0000:0000:ffff:0a0d:000b
```

对应：

```
10.13.0.11
```

### 7）确认 Type 是 v2

如果你的网络是 RoCEv2，必须确认对应 Index 的 Type 是：

```
v2
```

> GID 表排查的核心动作，是把 Index 反查成一句人话：这个 Index 到底代表哪张网卡、哪个 IP、哪个 VLAN、哪种 RoCE。

  

![[Inbox/笔记同步助手/微信公众号/2026/08/images/11e5444e268036423403b0ec25883ad9_MD5.jpg]]

<u>14</u>

怎么验证 GID 真的选对了？

只看表不够。

还要用测试验证。

### 第一层：确认端口状态

```
ibstat
ibv_devinfo
rdma link
```

```
确认：
```

-   HCA 存在
    
-   Port Active
    
-   Link layer 是 Ethernet
    
-   netdev 对应正确
    

```
NCCL 官方排障文档也建议在跑 NCCL 前，用 ibstatus、ibstat 等确认端口、速率、link layer 和 active 状态。
```

### 第二层：确认 GID 表

```
show_gids
```

重点看：

-   Index
    
-   RoCE Version
    
-   IP
    
-   netdev
    
-   VLAN
    

### 第三层：用 perftest 指定 GID Index

服务端：

```
ib_write_bw -d mlx5_3 -i 1 -x 3
```

客户端：

```
ib_write_bw -d mlx5_3 -i 1 -x 3 <server-ip>
```

如果换一个 Index 后结果完全不同，就说明 GID 选择很关键。

### 第四层：用 rping 验证 RDMA-CM

服务端：

```
rping -s -a <server-rail3-ip> -V -C 10
```

客户端：

```
rping -c -a <server-rail3-ip> -S <client-rail3-ip> -V -C 10
```

```
NCCL 网络排障文档也推荐在 RoCE 连接问题中用 rping 验证节点间连接，并通过 show_gids 核查 GID 选择。
```

  

### 第五层：再跑 NCCL

```
NCCL_DEBUG=INFO
NCCL_DEBUG_SUBSYS=NET,INIT
```

查看：

-   Socket 接口
    
-   HCA
    
-   GID
    
-   RoCE Version
    
-   连接建立日志
    

  

> 正确的 GID 不是“看起来像”，而是必须通过 Verbs、RDMA-CM 和 NCCL 三层验证。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1043da3c39b062ab72c9e81a12946b98_MD5.jpg]]

<u>15</u>

GID 表规划，应该纳入 GPU 集群交付文档

很多项目交付时，会整理：

-   服务器清单
    
-   GPU 型号
    
-   网卡型号
    
-   交换机端口
    
-   线缆编号
    
-   VLAN
    
-   IP 地址
    
-   NCCL 测试结果
    

  

但很少有人把 GID 表纳入交付。

这是不够的。

对于 RoCE GPU 集群，建议每台机器都形成一张映射表：

```
GPU ID
NUMA
PCIe Path
HCA
Port
Netdev
IP
VLAN
VRF
GID Index
RoCE Type
Switch Port
Rail
```

为什么要这么细？

因为 NCCL 性能、GDR、RoCE 建连、多 Rail 均衡，都依赖这些关系同时正确。

比如：

```
GPU0 离 mlx5_0 最近
mlx5_0 对应 eth10
eth10 在 Rail0
Rail0 对应 VLAN 100
GID Index 3 是 eth10 的 RoCEv2 IPv4 GID
NCCL 应该在某个 Channel 上使用 mlx5_0
交换机端口也应该对应 Rail0
```

如果这些关系没有记录，一旦后续换线、换 IP、加 VLAN、换内核、升级 NCCL，就很难判断问题到底出在哪里。

  

> RoCE 交付不是只交 IP 表，还应该交 GID 表、HCA 表和 GPU-NIC-Rail 对应表。

这才是算力网络真正能持续运维的基础。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9231887901e651b558115c14b06ba5a2_MD5.jpg]]

<u>16</u>

GID 表是 RoCE 的身份账本

RoCE 网络里那张被忽视的“身份证”，到底是什么？

就是 GID 表。

它不是一个小参数。  
不是一串看不懂的 IPv6。  
不是只有写 RDMA 程序的人才需要关心。

它决定了：

-   QP 用哪个源 GID
    
-   用 RoCEv1 还是 RoCEv2
    
-   对应哪个 netdev
    
-   对应哪个 IP
    
-   是否带 VLAN
    
-   RDMA-CM 选择哪个源身份
    
-   NCCL 走 RoCE 时能不能正确建连
    
-   多 Rail 是否真的走到预期 HCA 和 Rail
    

  

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ef3df9e0f7161aa69c126c66c9445b7a_MD5.jpg]]

  

> GID 表是 RoCE 把 RDMA 世界和 Ethernet/IP 世界连接起来的身份索引表。

> GID Index 不只是一个编号，它背后绑定了 GID Value、RoCE Type、netdev、IP 和 VLAN。

> HCA 选对不代表 GID 选对；GID 选错，QP 可能从一开始就拿错了网络身份证。

> RDMA-CM 会自动从 GID 表里选源 GID，但自动选择依赖主机路由、netdev 和 RoCE Type 配置，不能盲目信任。

> NCCL 旧版本可能需要手工指定 `NCCL_IB_GID_INDEX`；新版本更多依赖动态选择，但仍要保证 GID 表、地址范围和 RoCE 版本规划正确。

> 多 Rail RoCE 的验收，不能只看 8 张网卡 Up，而要证明每条 Rail 的 HCA、netdev、IP、VLAN、GID Index 和交换机端口完全对应。

  

```
# 技术参考
## 本文关于 RoCEv1 使用专用 EtherType、RoCEv2 使用 IP/UDP 封装、UDP 目的端口 4791，以及 UDP 源端口可作为 flow identifier 辅助 ECMP 的描述，参考 NVIDIA MLNX_OFED RoCE 官方文档。
## 本文关于 RoCE GID 表项在 Ethernet device 配置 IP 地址时创建、表项包含 GID value、GID type 和 network device，以及 GID 表通过 sysfs 暴露的描述，参考 NVIDIA MLNX_OFED RoCE 官方文档。
## 本文关于同一 GID value 可以有 RoCEv1 和 RoCEv2 两种类型、IPv4 GID 以 IPv4-mapped IPv6 形式表示、VLAN 子接口对应 GID 会导致报文携带 VLAN Header 的描述，参考 NVIDIA MLNX_OFED RoCE 官方文档。
## 本文关于 RC/UC QP 从 INIT 到 RTR 时需要在 Address Vector 中指定 source GID index，并由该 index 的 GID type 决定 QP 的 RoCE 类型的描述，参考 NVIDIA MLNX_OFED RoCE 官方文档。
## 本文关于 RDMA-CM 会根据被动端 IP 自动决定源 GID，并从 GID 表中获取 GID 的描述，参考 NVIDIA MLNX_OFED RoCE 官方文档。
## 本文关于 NCCL 的 NCCL_SOCKET_IFNAME、NCCL_IB_HCA、NCCL_IB_GID_INDEX、NCCL_IB_ADDR_FAMILY、NCCL_IB_ADDR_RANGE 和 NCCL_IB_ROCE_VERSION_NUM 的描述，参考 NVIDIA NCCL 2.30.7 环境变量文档。
## 本文关于 RoCE 中错误 GID index 可能导致 ibv_modify_qp failed with error Invalid argument，以及 NCCL 2.21 及以后版本通常动态选择 GID、不应继续手工设置 NCCL_IB_GID_INDEX 的描述，参考 NVIDIA NCCL 2.30.7 网络排障文档。
```

---

  

👉 AI 训练网络全路径拆解 → **私信：****AI网络**

👉 AI 推理网络全路径拆解 → **私信：****推理**

**👉 AI 算力网络架构系统（真机实验环境） → 私信：系统**

**👉** AI 算力网络架构专家（真机实验环境） **→ 私信：专家**

**👉** AI 网络架构工程指南手册 **→ **私信：******工程指南**

👉 日常工作1对1答疑 → **私信：****答疑**

  

---

  

> **如果你也在做 AI 集群架构**
> 
> **欢迎关注「算力网络架构手记」**
> 
> **长期拆解真实算力网络问题**

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/f1f81efd_1788136719836?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk0MzY3NTQzOQ%3D%3D%26mid%3D2247489525%26idx%3D1%26sn%3D68f0b66ce178557dc649d7f847478d0f%26chksm%3Dc286ccc99fc1f94b63611a809f4615bb737ba925e82eb05b10c0d229b0e97c94fb48dabfc810%26mpshare%3D1%26scene%3D1%26srcid%3D0831TKSvO3B2kOPcp1rZH27E%26sharer_shareinfo%3D4bb0d1341f913ccc3a8153dba704d2b7%26sharer_shareinfo_first%3D4bb0d1341f913ccc3a8153dba704d2b7%23rd&s=obsidian)