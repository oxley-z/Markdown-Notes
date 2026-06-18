---
author: fengyp
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI2NjIwNzYyOQ==&mid=2649619891&idx=1&sn=c41325d2c6275253e0a50e966185685f&chksm=f3e43b952ccb8279947e629faa2cc1ebf380a89d1db135e56b0c6eab7ff1d6d55718600e8400&mpshare=1&scene=1&srcid=0611f1Kt1mOj0YaNiSuwKCUT&sharer_shareinfo=58ef5bdd9d11798d7d461ee08a58b41a&sharer_shareinfo_first=58ef5bdd9d11798d7d461ee08a58b41a#rd
saved: 2026-06-11 19:06:53
tags:
  - 笔记同步助手
id: d0e361b7-35fe-49d6-8d18-8983e093e1a9
---

公众号名称：大话网络通信

作者名称：fengyp

发布时间：2026-06-11 14:00

RoCEv2 RDMA Hello World 程序 — 基于 raw libibverbs API

## 1\. 架构概要

整个程序围绕 **RDMA 通信的核心流程**展开，设计为单二进制同时支持 server/client 模式：

+-----------------------+          TCP (带外)          +-----------------------+|  linux01 (server)     |◄────── 交换 QP 信息 ──────►|  linux02 (client)     ||                       |                              |                       ||  rxe0 (192.168.64.4)  |◄────── RDMA SEND ─────────►|  rxe\_0 (192.168.64.5) |+-----------------------+                              +-----------------------+

**为什么不用 rdma\_cm？**SoftRoCE 上 `rdma_bind_addr`无法绑定到指定 IP（EADDRNOTAVAIL，这个问题解决了，下篇文章讲），因为 RXE 设备没有在 rdma\_cm 内核模块中注册地址映射。改用 raw ibverbs 配合 TCP 交换连接信息是更通用的做法。

## 2\. 数据结构

### \`struct qp\_info\` — 通过 TCP 交换的 QP 连接信息

struct qp\_info {uint16\_t      lid;      // LID (Local ID) — RoCEv2 下固定为 0uint32\_t      qpn;      // QP Number — 远端 QP 的唯一标识uint32\_t      psn;      // Packet Sequence Number — 初始包序号union ibv\_gid gid;      // GID (Global ID) — 128 位，RoCEv2 下为 IPv4 映射格式};

`union ibv_gid`是 128 位的全局标识符。RoCEv2 的 GID index 1 填充为 IPv4 映射格式 `::ffff:192.168.x.x`。

### \`struct rdma\_res\` — 本地 RDMA 资源

struct rdma\_res {struct ibv\_context \*ctx;   // 设备上下文（对应一个 RNIC 或 RXE）struct ibv\_pd      \*pd;    // Protection Domain — 保护域，隔离不同应用的资源struct ibv\_cq      \*cq;    // Completion Queue — 完成事件队列struct ibv\_qp      \*qp;    // Queue Pair — 包含 SQ(发送) + RQ(接收)struct ibv\_mr      \*mr;    // Memory Region — 注册给 RDMA 使用的内存区char                buf\[MSG\_SIZE\]; // 收发共用缓冲区};

这是 RDMA 编程中最核心的 5 个对象，关系如下：

ibv\_context (设备)└── ibv\_pd (保护域)├── ibv\_mr (内存注册)├── ibv\_qp (队列对)│       ├── SQ (Send Queue)  ── 投递 ibv\_send\_wr│       └── RQ (Recv Queue)  ── 投递 ibv\_recv\_wr└── ibv\_cq (完成队列) ◄── SQ/RQ 的完成事件汇聚于此

## 3\. 资源管理

### 打开 RXE 设备 — \`open\_rxe\_device()\`

static int open\_rxe\_device(struct ibv\_context \*\*ctx)

-   调用 `ibv_get_device_list()`枚举系统上所有 RDMA 设备
    
-   依次 `ibv_open_device()`打开，成功即返回第一个可用的设备
    
-   在 SoftRoCE 环境下，列表包含 `rxe0`或 `rxe_0`
    
-   用 `ibv_free_device_list()`释放设备列表
    

### 分配资源 — \`res\_alloc()\`

static struct rdma\_res \*res\_alloc(void)

按顺序创建 4 个对象，任何一个失败则反向清理：

ibv\_alloc\_pd()    → Protection Domainibv\_create\_cq()   → Completion Queue（16 个槽位）ibv\_reg\_mr()      → 注册 buf 到 RDMA 硬件（支持本地写 + 远端写）

-   `IBV_ACCESS_LOCAL_WRITE`
    
    — 允许本端 CPU 写
    
-   `IBV_ACCESS_REMOTE_WRITE`
    
    — 允许远端 RDMA 写（SEND/RECV 需要）
    

### 释放资源 — \`res\_free()\`

逆序销毁，每个指针都做 NULL 检查：

ibv\_destroy\_qp() → ibv\_dereg\_mr() → ibv\_destroy\_cq() → ibv\_dealloc\_pd() → ibv\_close\_device()

### 创建 QP — \`create\_qp()\`

struct ibv\_qp\_init\_attr attr = {.send\_cq  = r->cq,.recv\_cq  = r->cq,           // 收发共用同一个 CQ.cap = {.max\_send\_wr  = 8,       // 最多 8 个待处理发送 WR.max\_recv\_wr  = 8,       // 最多 8 个待处理接收 WR.max\_send\_sge = 1,       // 每个 WR 最多 1 个 SGE.max\_recv\_sge = 1,},.qp\_type = IBV\_QPT\_RC,       // Reliable Connection — 可靠连接};

**IBV\_QPT\_RC**是最常用的 QP 类型，保证有序、可靠、不重复交付，类似 TCP。

## 4\. TCP 辅助通道

### \`tcp\_listen()\` / \`tcp\_accept()\` / \`tcp\_connect()\`

标准的 TCP socket 封装，用于在 RDMA 数据传输之前交换 QP 元数据。因为 RDMA 连接建立需要双方互知 QPN、GID、PSN 等参数，而这些信息无法通过 RDMA 链路本身传递（链路还没建好），所以需要一个**带外通道（out-of-band）**。

### \`tcp\_exchange()\`

write(fd, local, sizeof(\*local));  // 发送本地 QP 信息read(fd, peer, sizeof(\*peer));     // 接收远端 QP 信息

信息交换顺序决定了 connection 的对称性——这里 client 和 server 在 TCP 层面角色不同，但交换的内容是完全对称的。

## 5\. RDMA 数据收发

### Work Request 机制

RDMA 的核心操作模型：**Work Request → Work Queue → Work Completion**

App 调用 ibv\_post\_send/ibv\_post\_recv↓ 投递 WRSend Queue / Recv Queue↓ 硬件执行ibv\_poll\_cq 轮询 CQ 获取 Work Completion (WC)

### \`post\_recv()\`

struct ibv\_sge sge = {.addr   = (uintptr\_t)r->buf,  // 缓冲区地址.length = MSG\_SIZE,           // 长度.lkey   = r->mr->lkey,        // 内存区域本地 key（注册 MR 时分配）};struct ibv\_recv\_wr wr = {.wr\_id   = 1,                 // 用户自定义 ID，完成时通过 WC 返回.sg\_list = &sge,.num\_sge = 1,};ibv\_post\_recv(r->qp, &wr, &bad);

-   `lkey`
    
    是 MR 注册后得到的本地标识符，硬件用它来验证内存访问权限
    
-   `wr_id`
    
    在 poll CQ 时原样返回，用来区分是哪个 WR 完成了（本例 send 用 2，recv 用 1）
    
-   SGE (Scatter/Gather Element) 描述了一段内存，RDMA 硬件直接从这段内存读写
    

### \`post\_send()\`

struct ibv\_send\_wr wr = {.wr\_id      = 2,.opcode     = IBV\_WR\_SEND,     // SEND 操作（无 remote key 要求）.send\_flags = IBV\_SEND\_SIGNALED, // 完成时生成 CQ 事件.sg\_list    = &sge,.num\_sge    = 1,};ibv\_post\_send(r->qp, &wr, &bad);

-   **IBV\_WR\_SEND**
    
    — 最基础的 RDMA 操作，类似 TCP 的 send。数据从本端 SGE 送到对端预先 post 的 RECV 缓冲区。
    
-   **IBV\_SEND\_SIGNALED**
    
    — 必须设置，否则操作完成后 CQ 上不会生成完成事件，`wait_cq()`将永远等不到结果。
    

### \`wait\_cq()\`

for (int i = 0; i < 1000; i++) {int n = ibv\_poll\_cq(r->cq, 1, &wc);if (n > 0) {if (wc.status == IBV\_WC\_SUCCESS) return 0;// 处理错误...}usleep(2000);  // 2ms 间隔，总共等 2 秒}

-   `ibv_poll_cq`
    
    是**非阻塞**的，返回 0 表示当前没有完成事件
    
-   轮询间隔 2ms，超时 2 秒（1000 次）
    
-   WC 的 `status`字段必须为 `IBV_WC_SUCCESS`，否则表示传输失败
    

## 6\. QP 状态机

这是整个程序最核心也最难理解的部分。一个 RC QP 必须严格经过以下状态转换：

RESET ──► INIT ──► RTR (Ready To Receive) ──► RTS (Ready To Send)

### RESET → INIT

attr.qp\_state        = IBV\_QPS\_INIT;attr.pkey\_index      = 0;        // 分区键索引，Ethernet/RoCE 下为 0attr.port\_num        = 1;        // IB 端口号，SoftRoCE 固定为 1attr.qp\_access\_flags = IBV\_ACCESS\_LOCAL\_WRITE | IBV\_ACCESS\_REMOTE\_WRITE;

INIT 状态允许 QP 接收 post\_recv 投递的 WR，但还不能发送或接收网络数据。

### INIT → RTR

attr.qp\_state           = IBV\_QPS\_RTR;attr.path\_mtu           = IBV\_MTU\_1024;   // 与 ibv\_devinfo 的 active\_mtu 匹配attr.dest\_qp\_num        = peer.qpn;       // 远端的 QP 号attr.rq\_psn             = peer.psn;       // 远端期望的接收 PSNattr.max\_dest\_rd\_atomic = 1;              // 最大未完成的读/原子操作数attr.ah\_attr.is\_global  = 1;              // 需要 GRH (Global Routing Header) — RoCEv2 必须attr.ah\_attr.grh.dgid   = peer.gid;       // 远端 GIDattr.ah\_attr.grh.sgid\_index = 1;          // 本地 GID index 1 = RoCEv2 IPv4attr.ah\_attr.grh.hop\_limit  = 1;attr.ah\_attr.port\_num   = 1;

**这是最关键的一步。**RTR 状态配置了到对端 QP 的路由信息（GID、QPN）和传输参数（MTU、PSN）。

**GID index 1 为什么是 RoCEv2？**

| Index | GID 格式 | 协议 |
| --- | --- | --- |
| 0 | `fe80::34f0:e0ff:fe65:26dd` | RoCEv1 (IPv6 link-local) |
| 1 | `::ffff:192.168.64.4` | **RoCEv2**(IPv4 可路由) |

RoCEv1 用 EtherType 0x8915 封装，不可路由。RoCEv2 用 UDP 端口 4791，可路由。GID index 1 的 IPv4 映射格式明确标识了 RoCEv2。

### RTR → RTS

attr.qp\_state   = IBV\_QPS\_RTS;attr.timeout    = 14;       // 本地 ACK 超时（4.096μs \* 2^timeout ≈ 134ms）attr.retry\_cnt  = 7;        // 重试次数attr.rnr\_retry  = 7;        // RNR (Receiver Not Ready) 重试次数attr.sq\_psn     = local.psn; // 本端 SQ 的初始 PSN

RTS 状态表示 QP 已经完全就绪，可以发送和接收 RDMA 数据。

### 状态机总图

Client QP                           Server QP─────────                           ─────────RESET                               RESET│                                  ││ create\_qp()                      │ create\_qp() (on CONNECT\_REQUEST)▼                                  ▼INIT ───────────────────────────── INIT│                                  ││ qp\_transition\_rtr()              │ qp\_transition\_rtr()▼                                  ▼RTR ────────────────────────────── RTR│                                  ││ qp\_transition\_rts()              │ qp\_transition\_rts()▼                                  ▼RTS ───────── RDMA ─────────────── RTS│            SEND                  ││══════════════════════════════════►││            SEND                  ││◄══════════════════════════════════│

## 7\. Server 完整流程

### 步骤分解

1\. res\_alloc()        ── 打开 RXE 设备, 创建 PD/CQ/MR2. create\_qp()        ── 创建 RC QP (处于 RESET 状态)3. ibv\_query\_gid()    ── 读取 GID index 1 (RoCEv2)4. tcp\_listen()       ── 监听 TCP 端口 185155. tcp\_accept()       ── 等待客户端 TCP 连接6. tcp\_exchange()     ── 交换 QP info (QPN, PSN, GID)7. qp\_transition\_init() ── RESET → INIT8. post\_recv()        ── 投递 RECV WR (等待客户端消息)9. qp\_transition\_rtr()  ── INIT → RTR (配置对端路由)10. qp\_transition\_rts() ── RTR → RTS (QP 就绪)11. wait\_cq()         ── 等待 RECV 完成12. printf("received: %s")  ── 打印客户端消息13. post\_send()       ── 回复 "Hello from server!"14. wait\_cq()         ── 等待 SEND 完成15. 清理               ── 关闭 TCP/RDMA 资源

### 为什么 post\_recv 在 RTR/RTS 之前？

post\_recv 可以在 INIT 或之后任意状态投递。WR 会挂在 RQ 上，等 QP 进入 RTR 状态后自动生效。**先 post\_recv 再转 RTR**确保当远端数据到达时，接收缓冲区已经就位。

## 8\. Client 完整流程

### 步骤分解

1\. res\_alloc()        ── 打开 RXE 设备, 创建 PD/CQ/MR2. create\_qp()        ── 创建 RC QP3. ibv\_query\_gid()    ── 读取 GID index 14. tcp\_connect()      ── 连接服务器 TCP 端口5. tcp\_exchange()     ── 交换 QP info6. qp\_transition\_init() ── RESET → INIT7. qp\_transition\_rtr()  ── INIT → RTR8. qp\_transition\_rts() ── RTR → RTS9. post\_recv()        ── 投递 RECV (准备接收服务器的回复)10. post\_send()       ── 发送 "Hello from client!"11. wait\_cq()         ── 等待 SEND 完成 (WC status == SUCCESS)12. wait\_cq()         ── 等待 RECV 完成 (服务器回复)13. printf("received: %s")  ── 打印服务器回复14. 清理

### 为什么 client 需要 2 次 wait\_cq()？

client 先后 post 了 1 个 RECV 和 1 个 SEND（带 SIGNALED 标志），所以 CQ 上会产生 **2 个完成事件**：

| 顺序 | wr\_id | opcode | 说明 |
| --- | --- | --- | --- |
| 1 | 2 (SEND) | SEND | 本端发送完成 |
| 2 | 1 (RECV) | RECV | 对端回复到达 |

先 call wait\_cq 拿到 SEND 完成，再 call 一次拿到 RECV 完成。

## 9\. 与 TCP 的对比

| 概念 | TCP | RDMA (ibverbs) |
| --- | --- | --- |
| 端点标识 | IP:Port | GID + QPN |
| 连接建立 | connect/accept 握手 | TCP 交换 QP info + 手动 QP 状态机 |
| 发送 | write/send | ibv\_post\_send (WR 入队) |
| 接收 | read/recv | ibv\_post\_recv (预先注册缓冲区) |
| 完成通知 | select/epoll 事件 | ibv\_poll\_cq (轮询 CQ) |
| 数据拷贝 | CPU 参与，内核→用户 | DMA 直接到用户缓冲区 (zero-copy) |
| 可靠性 | TCP 协议层保证 | QP 硬件层保证 (RC 模式) |

## 10\. MTU 和超时参数

### MTU (Path MTU)

attr.path\_mtu = IBV\_MTU\_1024;

`ibv_devinfo`中 `active_mtu: 1024`表示两端协商后的 MTU。可选的 MTU 值：256 / 512 / 1024 / 2048 / 4096。RXE 当前协商为 1024。

### Timeout

attr.timeout = 14;

本地 ACK 超时 = `4.096μs × 2^timeout`。timeout=14 时约 67ms。如果对端在超时时间内没有回复 ACK，QP 会触发重试或进入 error 状态。

### Retry Count

attr.retry\_cnt = 7;attr.rnr\_retry = 7;

-   `retry_cnt`
    
    — 发送端重试次数（7 次）
    
-   `rnr_retry`
    
    — 对端 RNR (Receiver Not Ready) 时的重试次数（7 = 无限）
    

## 11\. 错误处理模式

程序使用 **goto + out 标签**的统一清理模式：

int ret = 1;// ... 初始化资源 ...if (some\_error) {perror("...");goto out;  // 跳转到统一的清理代码}// ... 正常处理 ...ret = 0;out:// 销毁所有已分配的资源return ret;

优点：

-   避免每处错误都写一遍 cleanup
    
-   资源按分配顺序逆序释放
    
-   `ret = 1`
    
    （失败）/ `0`（成功）作为返回值
    

## 12\. 编译和运行

### 编译

gcc -Wall -Wextra -o rdma\_hello rdma\_hello.c -libverbs

-   `-libverbs`
    
    — 链接 InfiniBand verbs 库（包含 ibv\_\* API）
    
-   不需要 `-lrdmacm`（本程序不使用 rdma\_cm）
    

### 运行

\# 服务端（linux01）./rdma\_hello -s -p 18515# 客户端（linux02）./rdma\_hello -c 192.168.64.4 -p 18515

### 预期输出

\=== server (linux01) ===                              === client (linux02) ===server: opening RDMA device...                        client: opening RDMA device...device: rxe0                                          device: rxe\_0server: waiting for TCP connection on port 18515...   client: connecting to 192.168.64.4:18515...tcp: client from 192.168.64.5:54198                 client: exchanging QP info...server: exchanging QP info...                           local QPN: 17, peer QPN: 33local QPN: 33, peer QPN: 17                         client: QP connectedserver: QP connected                                  client: send completedserver: received: Hello from client!                  client: received: Hello from server!server: reply sent

## 13\. 扩展阅读

-   RDMA Core 官方文档
    
-   libibverbs API 参考
    
-   InfiniBand Architecture Specification
    
-   SoftRoCE — Linux Kernel
    

原创文章 · 作者：fengyp

欢迎转发，转载请联系作者

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/77e31327_1781176012304?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI2NjIwNzYyOQ%3D%3D%26mid%3D2649619891%26idx%3D1%26sn%3Dc41325d2c6275253e0a50e966185685f%26chksm%3Df3e43b952ccb8279947e629faa2cc1ebf380a89d1db135e56b0c6eab7ff1d6d55718600e8400%26mpshare%3D1%26scene%3D1%26srcid%3D0611f1Kt1mOj0YaNiSuwKCUT%26sharer_shareinfo%3D58ef5bdd9d11798d7d461ee08a58b41a%26sharer_shareinfo_first%3D58ef5bdd9d11798d7d461ee08a58b41a%23rd&s=obsidian)