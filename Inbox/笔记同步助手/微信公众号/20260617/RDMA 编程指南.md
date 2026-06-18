---
author: fengyp
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI2NjIwNzYyOQ==&mid=2649619933&idx=1&sn=22e32d2ab4aa431741879b02fa0a23a4&chksm=f33d73c5e9cf13dd3ba8fd50fd1d3b75148f4ed41c2ef9f315bc0cd0710dc41a91c79a7d14f4&mpshare=1&scene=1&srcid=06177Dw6DIOrMmZFTRx4Kqzb&sharer_shareinfo=00ed3254d7889984261ad276eb240c8f&sharer_shareinfo_first=00ed3254d7889984261ad276eb240c8f#rd
saved: 2026-06-17 09:59:01
tags:
  - 笔记同步助手
id: b8b3bf7d-1b65-4761-aedb-ef73b633f954
---

公众号名称：大话网络通信

作者名称：fengyp

发布时间：2026-06-17 08:10

![[Inbox/笔记同步助手/images/7ef50295e088d2c075cc46bf8aaee198_MD5.jpg]]

从元素到流程，系统梳理 libibverbs 编程的全景视图

## 1\. 全景关系图

RDMA 编程的核心是通过 libibverbs 库在用户态直接操作硬件。所有编程对象之间存在严格的依赖关系——理解这张关系图是掌握 RDMA 编程的第一步。

![[Inbox/笔记同步助手/images/bc6cedb350136565c0a44693ed711955_MD5.jpg]]

关系图的核心逻辑：

-   **ibv\_device**
    
    是物理设备的软件抽象，通过它打开设备获取 **ibv\_context**
    
-   **ibv\_pd（Protection Domain）**
    
    是资源隔离的边界，几乎所有资源都从 PD 创建
    
-   **ibv\_mr**
    
    向 PD 注册内存，返回 lkey/rkey 供硬件寻址
    
-   **ibv\_qp**
    
    包含 **SQ（Send Queue）**和 **RQ（Receive Queue）**，QPN 是对端寻址的依据
    
-   **ibv\_cq**
    
    关联 QP 的完成事件，通过 **ibv\_poll\_cq()**获取 **ibv\_wc**
    
-   **ibv\_send\_wr/ibv\_recv\_wr**
    
    提交到 QP 的 SQ/RQ，通过 **ibv\_sge**描述缓冲区
    
-   **ibv\_srq**
    
    和 **ibv\_ah**是可选资源，分别用于共享接收队列和 UD 模式寻址
    

## 2\. 编程元素

### 2.1 Device & Context

struct ibv\_device \*\*ibv\_get\_device\_list(int \*num\_devices);struct ibv\_context \*ibv\_open\_device(struct ibv\_device \*device);int ibv\_query\_device(struct ibv\_context \*context, struct ibv\_device\_attr \*device\_attr);

`ibv_device`代表系统中的 RDMA 设备（如 mlx5\_0、rxe\_0）。`ibv_get_device_list()`枚举所有可用设备，`ibv_open_device()`打开设备返回 `ibv_context`——这是后续所有操作的基础句柄。

关键点：设备列表使用完后通过 `ibv_free_device_list()`释放，但打开的设备需要单独 `ibv_close_device()`关闭。

### 2.2 Protection Domain (PD)

struct ibv\_pd \*ibv\_alloc\_pd(struct ibv\_context \*context);int ibv\_dealloc\_pd(struct ibv\_pd \*pd);

PD 是 RDMA 资源隔离的基本单元。一个 PD 内部创建的 MR、QP、CQ 等资源只能在该 PD 范围内互相访问。典型使用模式：一个应用或一个连接使用一个 PD。

PD 不具有跨设备能力——它绑定在创建它的设备上。

### 2.3 Memory Region (MR)

struct ibv\_mr \*ibv\_reg\_mr(struct ibv\_pd \*pd, void \*addr, size\_t length, int access);int ibv\_dereg\_mr(struct ibv\_mr \*mr);

RDMA 是 DMA 操作，硬件直接访问内存。因此使用的内存必须提前**注册（register）**——锁定物理页面、建立 DMA 映射、生成 **lkey（local key）**和 **rkey（remote key）**：

-   **lkey**
    
    ：本地操作使用，如 SGE 中指定数据缓冲区
    
-   **rkey**
    
    ：远端操作使用，如 `ibv_post_send()`中 `IBV_WR_RDMA_WRITE`时对端需要本端的 rkey
    

`access`标志常见值：`IBV_ACCESS_LOCAL_WRITE`（本地写）、`IBV_ACCESS_REMOTE_WRITE`（远端写）、`IBV_ACCESS_REMOTE_READ`（远端读）。

页对齐并非 libibverbs 的硬性 API 要求。mlx5 等现代驱动的 `ibv_reg_mr()`内部以 4KB 页为单位建立 MTT（Memory Translation Table）映射，对调用方传入的 `addr`本身无对齐要求——非对齐地址会被 libibverbs 自动向下/向上 round 到 page 边界。但跨页的注册会让相邻 MR 共享同一物理页，可能引发 DMA 重映射冲突；实际生产中应使用 4KB 对齐的预注册内存池（`posix_memalign`或 `hugepage`），通过 SGE 的 `addr`/`length`在池内寻址——这样既避免页冲突，又把"慢操作"限制在初始化阶段。

### 2.4 Completion Queue (CQ)

struct ibv\_cq \*ibv\_create\_cq(struct ibv\_context \*context, int cqe, void \*cq\_context,struct ibv\_comp\_channel \*channel, int comp\_vector);int ibv\_destroy\_cq(struct ibv\_cq \*cq);int ibv\_poll\_cq(struct ibv\_cq \*cq, int num\_entries, struct ibv\_wc \*wc);

CQ 是完成事件的通道。QP 上的每个操作（send/recv/rdma）执行完成后，WC（Work Completion）会被放入关联的 CQ。

关键参数 `cqe`：CQ 的容量，即最多容纳多少个未轮询的 WC。`cqe`必须 ≥ QP 的 `max_send_wr + max_recv_wr`（多个 QP 共享 CQ 时取累加），否则新完成事件可能因 CQ 空间不足而异常——具体行为依赖驱动实现：可能产生 async event、阻塞投递，或让 QP 进入 ERR 状态。生产中应留有余量（如 `cqe = (SQ+RQ) * 2`）。

CQ 可以被多个 QP 共享——这是常见的优化手段。

### 2.5 Queue Pair (QP) — SQ / RQ / QPN

struct ibv\_qp \*ibv\_create\_qp(struct ibv\_pd \*pd, struct ibv\_qp\_init\_attr \*qp\_init\_attr);int ibv\_destroy\_qp(struct ibv\_qp \*qp);int ibv\_modify\_qp(struct ibv\_qp \*qp, struct ibv\_qp\_attr \*attr, int attr\_mask);

QP 是 RDMA 通信的端点，但它的名字已经揭示了本质：**它不是单个队列，而是一对队列**：

-   **SQ（Send Queue）**
    
    ：处理发送请求。提交 `ibv_send_wr`，执行 send/write/read/atomic 操作
    
-   **RQ（Receive Queue）**
    
    ：处理接收请求。提交 `ibv_recv_wr`，接收对端发来的数据
    
-   **QPN（Queue Pair Number）**
    
    ：QP 的唯一标识，通信对端需要知道 QPN 才能向本端发送数据
    

QP 有三种类型：

| 类型 | 可靠性 | 连接方式 | 适用场景 |
| --- | --- | --- | --- |
| **RC (Reliable Connection)** | 可靠、保序 | 一对一连接 | 绝大多数场景 |
| **UC (Unreliable Connection)** | 不可靠、保序 | 一对一连接 | 允许丢包的高性能场景 |
| **UD (Unreliable Datagram)** | 不可靠、不保序 | 多对多 | 多播/控制面 |

`qp_init_attr`中需要指定：`qp_type`、`send_cq`、`recv_cq`、`cap.max_send_wr`、`cap.max_recv_wr`、`cap.max_send_sge`、`cap.max_recv_sge`。

### 2.6 Shared Receive Queue (SRQ)

struct ibv\_srq \*ibv\_create\_srq(struct ibv\_pd \*pd, struct ibv\_srq\_init\_attr \*srq\_init\_attr);int ibv\_destroy\_srq(struct ibv\_srq \*srq);

SRQ 允许多个 QP 共享同一个 RQ。当大量 QP 连接到一个节点时，每个 QP 独立维护 RQ 需要预先 post 大量 recv WR，浪费内存。SRQ 在连接数多的场景下显著降低内存开销。

使用 SRQ 的 QP 创建时，在 `qp_init_attr`中设置 `srq`字段指向 SRQ 实例。

### 2.7 Address Handle (AH)

struct ibv\_ah \*ibv\_create\_ah(struct ibv\_pd \*pd, struct ibv\_ah\_attr \*attr);int ibv\_destroy\_ah(struct ibv\_ah \*ah);

AH 仅在 **UD（Unreliable Datagram）**模式下使用，描述目标端的路由信息（GID、GID type、端口号等）。每个 send WR 都需要指定一个 AH 来标识数据发往谁。

对于 RC/UC 模式，连接在 `modify_qp`的 `IBV_QP_AV`中已指定对端信息，不需要 AH。

### 2.8 Work Request & Scatter-Gather Element (WR / SGE)

struct ibv\_send\_wr {uint64\_t wr\_id;struct ibv\_send\_wr \*next;struct ibv\_sge \*sg\_list;int num\_sge;enum ibv\_wr\_opcode opcode;int send\_flags;uint32\_t imm\_data;          // SEND with immediate 时使用union {struct {                // opcode == IBV\_WR\_RDMA\_{READ,WRITE}uint64\_t remote\_addr;uint32\_t rkey;} rdma;struct {                // opcode == IBV\_WR\_ATOMIC\_\*uint64\_t remote\_addr;uint64\_t compare\_add;uint64\_t swap;uint32\_t rkey;} atomic;struct {                // opcode == IBV\_WR\_SEND（UD 模式）struct ibv\_ah \*ah;uint32\_t remote\_qpn;uint32\_t remote\_qkey;} ud;} wr;};struct ibv\_recv\_wr {uint64\_t wr\_id;struct ibv\_recv\_wr \*next;struct ibv\_sge \*sg\_list;int num\_sge;};struct ibv\_sge {uint64\_t addr;   // 缓冲区虚拟地址uint32\_t length; // 长度uint32\_t lkey;   // MR 的 lkey};

WR 描述一次操作请求，SGE 描述数据缓冲区的位置和长度。

-   `ibv_send_wr`
    
    ：提交到 SQ，opcode 决定操作类型（`IBV_WR_SEND`、`IBV_WR_RDMA_WRITE`、`IBV_WR_RDMA_READ`、`IBV_WR_ATOMIC_CMP_AND_SWP`等）
    
-   `ibv_recv_wr`
    
    ：提交到 RQ，指定接收数据放到哪里
    
-   `wr_id`
    
    ：用户自定义标识，在 WC 中返回，用于匹配完成事件
    
-   SGE 的 `addr`必须指向已注册 MR 的内存区域
    

### 2.9 Work Completion (WC)

struct ibv\_wc {uint64\_t wr\_id;enum ibv\_wc\_status status;enum ibv\_wc\_opcode opcode;uint32\_t qp\_num;uint32\_t byte\_len;...};

WC 由 `ibv_poll_cq()`从 CQ 中取出，表示一次操作已完成。

关键字段：

-   `status`
    
    ：`IBV_WC_SUCCESS`表示成功，其他值表示失败（`IBV_WC_RETRY_EXC_ERR`、`IBV_WC_WR_FLUSH_ERR`等）
    
-   `wr_id`
    
    ：与提交 WR 时设置的 `wr_id`对应，用于识别哪个操作完成了
    
-   `opcode`
    
    ：完成的操作类型
    
-   `byte_len`
    
    ：实际传输的字节数。不同完成类型含义不同：SEND 表示已发送字节数、RECV 表示收到字节数、RDMA WRITE/READ 表示实际传输的字节数
    

## 3\. 编程流程

### 3.1 设备初始化

struct ibv\_device \*\*dev\_list;struct ibv\_context \*ctx;struct ibv\_device\_attr dev\_attr;dev\_list = ibv\_get\_device\_list(&num\_devices);// 选择需要的设备，通常选第一个或按名称匹配ctx = ibv\_open\_device(dev\_list\[0\]);ibv\_query\_device(ctx, &dev\_attr);ibv\_free\_device\_list(dev\_list);

`ibv_query_device()`获取设备属性（最大 QP 数、最大 MR 大小、FW 版本等），用于后续资源创建时评估容量。

### 3.2 创建 PD & 注册 MR

struct ibv\_pd \*pd = ibv\_alloc\_pd(ctx);char \*buf = malloc(BUF\_SIZE);struct ibv\_mr \*mr = ibv\_reg\_mr(pd, buf, BUF\_SIZE,IBV\_ACCESS\_LOCAL\_WRITE |IBV\_ACCESS\_REMOTE\_WRITE);

内存注册是 RDMA 编程中唯一的"慢操作"——需要内核锁页并建立 DMA 映射，单次 `ibv_reg_mr`在繁忙的系统上可能耗时数百微秒到毫秒级。高频操作应在初始化阶段一次性注册大块内存，通过 SGE 的 addr/length 控制访问范围。生产代码里通常采用 **MR 池（memory pool）**模式：

// MR 池的简化示例#define POOL\_SIZE  (4 \* 1024 \* 1024)   // 4 MB#define CHUNK\_SIZE 4096                // 4 KB#define NUM\_CHUNKS (POOL\_SIZE / CHUNK\_SIZE)struct mr\_pool {void  \*base;struct ibv\_mr \*mr;void  \*free\_list\[NUM\_CHUNKS\];int    free\_cnt;};static struct mr\_pool \*pool\_create(struct ibv\_pd \*pd){struct mr\_pool \*p = calloc(1, sizeof(\*p));posix\_memalign(&p->base, 4096, POOL\_SIZE);p->mr = ibv\_reg\_mr(pd, p->base, POOL\_SIZE,IBV\_ACCESS\_LOCAL\_WRITE | IBV\_ACCESS\_REMOTE\_WRITE);// 初始化 free\_listfor (int i = 0; i < NUM\_CHUNKS; i++)p->free\_list\[i\] = (char \*)p->base + i \* CHUNK\_SIZE;p->free\_cnt = NUM\_CHUNKS;return p;}static void \*pool\_alloc(struct mr\_pool \*p) { return p->free\_list\[--p->free\_cnt\]; }static void  pool\_free (struct mr\_pool \*p, void \*ptr) { p->free\_list\[p->free\_cnt++\] = ptr; }

-   启动时分配数 MB 到数 GB 的对齐内存（如 2MB hugepage 或 `posix_memalign`4KB 对齐）
    
-   注册为大块 MR（或按固定 chunk size 切成多个 MR）
    
-   运行时通过空闲链表分配/释放 chunk，使用时填充 SGE 后提交
    

**注意事项**： - MR 池大小应根据业务峰值内存使用量评估，注册后无法扩容 - 跨进程的零拷贝需要把 `mr->rkey`\+ `addr`通过 OOB 通知对端 - 大页（hugepage）能减少页表项数量、提升 DMA TLB 命中率，对高吞吐场景是必备

### 3.3 创建 CQ

struct ibv\_cq \*cq = ibv\_create\_cq(ctx, CQ\_DEPTH, NULL, NULL, 0);

`CQ_DEPTH`通常设置为 QP 的 SQ 和 RQ 深度之和，或更大。CQ 创建后关联到 QP 创建时的 `send_cq`/`recv_cq`参数。

### 3.4 创建 QP

struct ibv\_qp\_init\_attr qp\_attr = {.send\_cq = cq,.recv\_cq = cq,.qp\_type = IBV\_QPT\_RC,.cap = {.max\_send\_wr = 16,.max\_recv\_wr = 16,.max\_send\_sge = 1,.max\_recv\_sge = 1,},};struct ibv\_qp \*qp = ibv\_create\_qp(pd, &qp\_attr);

注意 `send_cq`和 `recv_cq`可以指向同一个 CQ。`max_send_wr`/`max_recv_wr`表示 SQ/RQ 最多能排队的 WR 数量，硬件有上限，`ibv_query_device()`返回的 `max_qp_wr`是硬上限。

### 3.5 带外信息交换

RDMA 通信的双方在建立连接前，需要交换一组元数据——这些信息通过 **带外（out-of-band）**方式传递，常见的 OOB 通道有 TCP socket、librdmacm（CM）、共享文件等。

**需要交换的信息：**

| 信息 | 获取方式 | 用于何处 |
| --- | --- | --- |
| **QPN** | `qp->qp_num` | `modify_qp`的 `dest_qpn` |
| **GID** | `ibv_query_gid()` | `ah_attr.grh.dgid` |
| **LID** | `ibv_query_port()` | `ah_attr.dlid`（仅 IB 链路层，RoCEv2 不需要） |
| **rkey** | `mr->rkey` | RDMA Write/Read 操作，对端凭此访问本端内存 |
| **PSN** | 本端 `rq_psn`必须等于对端 `sq_psn`（反之亦然） | `rq_psn`/ `sq_psn` |

**RoCEv2 vs IB 的关键差异**：RoCEv2 是基于 UDP/IP 的封装，走的是以太网路由（IP + UDP 4791），因此**只用 GID 不用 LID**。LID 仅在原生 InfiniBand 链路层或 RoCEv1 场景使用。如果你的部署全是 RoCEv2，表中的 LID 行可以忽略。

**PSN 安全提示**：学习示例可使用固定 0 方便调试，但生产环境应使用 `/dev/urandom`生成的随机值。PSN 是抵御包重放攻击的关键，固定 PSN 在共享 L2 网络中存在被同子网恶意主机伪造重放包的理论风险。

最常用的 OOB 方式是通过 TCP socket 交换消息：

// 本端发送 QPN 和 GIDsend(sock, &qp->qp\_num, sizeof(uint32\_t), 0);send(sock, &my\_gid, sizeof(union ibv\_gid), 0);// 接收对端 QPN 和 GIDrecv(sock, &remote\_qpn, sizeof(uint32\_t), 0);recv(sock, &remote\_gid, sizeof(union ibv\_gid), 0);

GID 的获取方式：

union ibv\_gid my\_gid;ibv\_query\_gid(ctx, port\_num, gid\_index, &my\_gid);

对于 RoCEv2，`gid_index`与 RDMA 设备绑定的 netdev 强相关：每个 GID entry 对应一个 IP 地址，index 编号由驱动按 netdev 顺序分配。`ibstat`和 `show_gid`命令可以查看当前 GID table，但更可靠的方式是直接遍历 `/sys/class/infiniband//ports/1/gid_attrs/types/`找到类型为 `RoCE v2`的 entry。

**新手最常踩的坑**：`sgid_index`选错时，`RTR`状态转换会**成功返回**，但实际收发数据时 hardware 报 "destination unreachable" 之类的错误——错误延迟到第一次 `ibv_post_send`才暴露，排查起来很费时间。建议在初始化阶段用 `ibv_query_gid()`验证对端 GID 是否非零。

如果使用 RDMA Write/Read 操作，还需要交换 rkey：

// 本端发送 rkey 和地址偏移struct {uint64\_t addr;  // buf 的虚拟地址uint32\_t rkey;  // mr->rkey} local\_info = { (uintptr\_t)buf, mr->rkey };send(sock, &local\_info, sizeof(local\_info), 0);// 收到对端信息后，存入 send\_wr 的 wr.rdma 字段swr.opcode = IBV\_WR\_RDMA\_WRITE;swr.wr.rdma.remote\_addr = remote\_info.addr;swr.wr.rdma.rkey = remote\_info.rkey;

**注意：**OOB 交换的只是元数据而非数据本身。实际通信数据通过 RDMA 硬件直接传输，不经过 OOB 通道。

### 3.6 QP 状态转换

QP 创建后处于 **RESET**状态，需要经过三次 `ibv_modify_qp()`才能开始收发数据。

![[Inbox/笔记同步助手/images/924fcbf2cb915381b9715e948336cfc8_MD5.jpg]]

**RESET → INIT**

struct ibv\_qp\_attr attr = {.qp\_state = IBV\_QPS\_INIT,.port\_num = 1,.pkey\_index = 0,};ibv\_modify\_qp(qp, &attr,IBV\_QP\_STATE | IBV\_QP\_PORT | IBV\_QP\_PKEY\_INDEX);

指定 QP 绑定的端口和 pkey。RC/UC 的 INIT 还需要设置 `qp_access_flags`控制远端访问权限。

**INIT → RTR（Ready to Receive）**

struct ibv\_qp\_attr attr = {.qp\_state = IBV\_QPS\_RTR,.path\_mtu = IBV\_MTU\_1024,.dest\_qpn = remote\_qpn,.rq\_psn = 0,.ah\_attr = {.grh = { .dgid = remote\_gid, .sgid\_index = 0 },.dlid = remote\_lid,.sl = 0,.src\_path\_bits = 0,.is\_global = 1,.port\_num = 1,},};ibv\_modify\_qp(qp, &attr,IBV\_QP\_STATE | IBV\_QP\_AV | IBV\_QP\_PATH\_MTU |IBV\_QP\_DEST\_QPN | IBV\_QP\_RQ\_PSN);

RTR 配置 QP 的接收端信息，核心参数：

-   `dest_qpn`
    
    ：对端的 QPN（通过带外方式交换）
    
-   `rq_psn`
    
    ：接收端的起始包序号
    
-   `ah_attr`
    
    ：地址属性，包含对端 GID/LID。RoCEv2 使用 GID（`is_global=1`）
    
-   `path_mtu`
    
    ：路径 MTU，RC/UC 要求两端一致
    

**RTR → RTS（Ready to Send）**

struct ibv\_qp\_attr attr = {.qp\_state = IBV\_QPS\_RTS,.sq\_psn = 0,.timeout = 14,.retry\_cnt = 7,.rnr\_retry = 7,.max\_rd\_atomic = 1,.min\_rnr\_timer = 12,};ibv\_modify\_qp(qp, &attr,IBV\_QP\_STATE | IBV\_QP\_SQ\_PSN | IBV\_QP\_TIMEOUT |IBV\_QP\_RETRY\_CNT | IBV\_QP\_RNR\_RETRY |IBV\_QP\_MAX\_QP\_DEST\_RD\_ATOMIC | IBV\_QP\_MIN\_RNR\_TIMER);

RTS 激活发送队列，核心参数：

-   `sq_psn`
    
    ：发送端的起始包序号（通常与对端的 `rq_psn`一致）
    
-   `timeout`
    
    ：ACK 超时时间（单位为 4.096μs，公式：`4.096μs * 2^timeout`）
    
-   `retry_cnt`
    
    ：重试次数上限
    
-   `rnr_retry`
    
    ：RNR（Receiver Not Ready）重试次数
    

不同 QP 类型在状态转换时的差异：

| QP 类型 | INIT→RTR | RTR→RTS |
| --- | --- | --- |
| RC | 完整配置 ah\_attr、dest\_qpn、path\_mtu | 需要所有参数 |
| UC | 与 RC 类似，无原子操作相关参数 | 需要所有参数 |
| UD | 只需 dest\_qpn，无需 ah\_attr/path\_mtu | 只需 sq\_psn |

任何状态下发生错误，QP 进入 **ERR**状态。ERR 状态的 QP 需要销毁重建。

### 3.7 提交 WR & 轮询 WC

**post recv（先于 post send）**

struct ibv\_sge sge = {.addr = (uint64\_t)buf,.length = BUF\_SIZE,.lkey = mr->lkey,};struct ibv\_recv\_wr recv\_wr = {.wr\_id = 1,.sg\_list = &sge,.num\_sge = 1,};struct ibv\_recv\_wr \*bad\_wr = NULL;ibv\_post\_recv(qp, &recv\_wr, &bad\_wr);

RQ 必须提前 post recv WR——对端数据到来时 RQ 没有准备好的 receive，会触发 RNR 错误。

**post send**

struct ibv\_sge sge = {.addr = (uint64\_t)buf,.length = send\_len,.lkey = mr->lkey,};struct ibv\_send\_wr send\_wr = {.wr\_id = 2,.sg\_list = &sge,.num\_sge = 1,.opcode = IBV\_WR\_SEND,.send\_flags = IBV\_SEND\_SIGNALED,};struct ibv\_send\_wr \*bad\_wr = NULL;ibv\_post\_send(qp, &send\_wr, &bad\_wr);

RDMA 是异步操作——`ibv_post_send()`返回只表示请求已入队，不代表传输完成。需要通过 `ibv_poll_cq()`获取 WC 确认完成。

**poll CQ**

struct ibv\_wc wc;int count = ibv\_poll\_cq(cq, 1, &wc);if (count > 0) {if (wc.status == IBV\_WC\_SUCCESS) {// 操作成功完成，wr\_id 标识具体操作printf("completed wr\_id=%lu\\n", wc.wr\_id);} else {fprintf(stderr, "CQ error: %s\\n", ibv\_wc\_status\_str(wc.status));}}

**send\_flags 调优要点**

`IBV_SEND_SIGNALED`标志：RC 模式下，如果多个 WR 批量提交，只需最后一个设置 `IBV_SEND_SIGNALED`，CQ 中只产生一个 WC，减少轮询开销和中断压力。批量大小一般取 8～32。

| Flag | 作用 | 适用场景 |
| --- | --- | --- |
| `IBV_SEND_SIGNALED` | 产生 WC | 批量提交的最后 1 个；任何需要确认完成的 WR |
| `IBV_SEND_SOLICITED` | 通知对端产生 WC | 配合对端 `IBV_RECV_RQ`，用于精确应答场景（如 RPC） |
| `IBV_SEND_INLINE` | 数据放 WR 而非 DMA 读 | 小数据（≤ `max_inline_data`，mlx5 通常 192 字节），节省一次 DMA + 提升小包性能 |
| `IBV_SEND_FENCE` | 强制按顺序执行（在该 WR 之前的未完成 WR 全部完成前不执行本 WR） | 读写依赖场景，避免乱序 |

**轮询 vs 事件驱动**

文章中的 `ibv_poll_cq()`是**轮询模式**——CPU 主动询问 CQ 是否有完成。另一种模式是**事件驱动**：

struct ibv\_comp\_channel \*channel = ibv\_create\_comp\_channel(ctx);struct ibv\_cq \*cq = ibv\_create\_cq(ctx, CQ\_DEPTH, NULL, channel, 0);// arm CQ 通知ibv\_req\_notify\_cq(cq, 0);// 等待事件（阻塞，直到至少一个 WC 可用）struct ibv\_cq \*ev\_cq;void \*ev\_ctx;ibv\_get\_cq\_event(channel, &ev\_cq, &ev\_ctx);ibv\_ack\_cq\_events(ev\_cq, 1);  // 必须在下次 req\_notify 前 ack// 然后再 poll 拿具体 WCibv\_poll\_cq(cq, ...);ibv\_req\_notify\_cq(cq, 0);  // 重新 arm

**两种模式的选择**：

-   **轮询模式**
    
    ：低延迟、高吞吐（避免系统调用与上下文切换），但持续占用 CPU
    
-   **事件驱动模式**
    
    ：低 CPU 占用（epoll 友好），适合 QP 数量多、消息密度低的场景
    
-   **生产推荐**
    
    ：高吞吐路径用轮询 + 忙等；管理面/控制面用事件驱动；多 QP 共享 CQ + 事件驱动 + epoll 是常见架构
    

### 3.8 资源清理

资源释放必须遵循严格的**逆向顺序**（创建顺序的逆序）：

ibv\_destroy\_qp(qp);ibv\_destroy\_cq(cq);ibv\_dereg\_mr(mr);ibv\_dealloc\_pd(pd);ibv\_close\_device(ctx);

QP 必须在 CQ 之前销毁——因为 QP 销毁时会 flush 未完成的 WR 到 CQ。同样，MR 必须在 PD 之前注销，PD 必须在设备之前释放。

### 3.9 错误处理与 QP 重建

RDMA 通信中的错误分两类：**通信层错误**（CRC 错误、重传超限、RNR 等）和**编程错误**（非法 opcode、未注册的 MR、权限不足等）。两类错误都会让 QP 进入 **ERR**状态——ERR 状态是终态，QP 只能销毁重建。

#### 异步事件

异步事件是驱动通知应用错误的标准机制：

// 在独立线程中循环处理while (1) {struct ibv\_async\_event event;if (ibv\_get\_async\_event(ctx, &event)) break;if (event.event\_type == IBV\_EVENT\_QP\_ERR) {struct ibv\_qp \*qp = event.element.qp;// 标记 QP 错误，触发上层重连逻辑mark\_qp\_broken(qp);} else if (event.event\_type == IBV\_EVENT\_CQ\_ERR) {// CQ 错误一般不可恢复，需要重建} else if (event.event\_type == IBV\_EVENT\_PORT\_ERR ||event.event\_type == IBV\_EVENT\_LID\_CHANGE) {// 链路层事件：网卡/PHY 异常、SM 重分配 LID// RoCEv2 场景下常见，需要重新建立 QP}ibv\_ack\_async\_event(&event);  // 必须 ack，否则驱动停止投递}

#### 同步处理

每次 `ibv_poll_cq()`拿到 `wc.status != IBV_WC_SUCCESS`的 WC 时，必须处理错误。常见错误码：

| 错误码 | 含义 | 处理建议 |
| --- | --- | --- |
| `IBV_WC_RETRY_EXC_ERR` | 重传超限（对端无响应或链路质量差） | 触发重连 |
| `IBV_WC_RNR_RETRY_EXC_ERR` | RNR 重试超限（对端 RQ 长期空） | 触发重连 |
| `IBV_WC_LOC_PROT_ERR` | 本地权限错误（如本地 WRITE 但 MR 未授权） | 检查 access flags |
| `IBV_WC_REM_INV_REQ_ERR` | 远端请求非法（如 RDMA 写到不可写区域） | 检查 rkey/addr 范围 |
| `IBV_WC_LOC_ACCESS_ERR` | 本地访问 MR 失败 | 检查 MR 有效性 |
| `IBV_WC_WR_FLUSH_ERR` | QP 进入 ERR 时 flush 出来的 WR | 视为正常清理信号 |

#### QP 重建流程

static int qp\_rebuild(struct rdma\_res \*res, uint32\_t remote\_qpn,union ibv\_gid remote\_gid){// 1. 销毁旧的 QPibv\_destroy\_qp(res->qp);// 2. 重新创建 QPstruct ibv\_qp\_init\_attr qia = { /\* 同 rdma\_init 中的配置 \*/ };res->qp = ibv\_create\_qp(res->pd, &qia);if (!res->qp) return -1;// 3. 重新走 RESET → INIT → RTR → RTSreturn rdma\_connect(res, remote\_qpn, remote\_gid);}

**关键点**： - ERR 状态的 QP 仍占用资源，必须显式 destroy - 重建后对端的 QP 也需要重新建立（RTR 状态需要新的路径信息） - 实际生产中通常引入"连接管理器"抽象层，封装 QP 生命周期 - 高频错误（如 1 秒内多次 RNR）应触发告警，可能是对端业务异常

### 3.10 性能调优要点

| 维度 | 调优点 | 建议 |
| --- | --- | --- |
| **MR** | 预注册、hugepage、跨页避免 | 4KB 对齐 + 大块注册，SGE 在池内寻址 |
| **CQ** | 容量、`cqe`留余量、共享 CQ | `cqe = (SQ+RQ) * 2`；多 QP 同类型可共享 |
| **WR 提交** | `IBV_SEND_SIGNALED`批量 | 批量大小 8～32；仅最后 1 个 SIGNALED |
| **小数据** | `IBV_SEND_INLINE` | ≤ `max_inline_data`（mlx5 通常 192B）时启用，省一次 DMA |
| **轮询** | busy-poll / 独占 CPU | 高吞吐路径绑核 + 关闭调度器抢占 |
| **链路** | MTU、PFC/ECN、CPU 亲和性 | 路径 MTU 4096；RoCEv2 需配置无损流（PFC） |
| **inline data** | 阈值 | 通过 `device_attr.max_inline_data`查询 |

#### 性能验证

写完 RDMA 代码后，必须用 perftest 工具集验证带宽和延迟：

\# 双向带宽测试ib\_send\_bw -d mlx5\_0 -x 3     # 客户端，gid\_index=3ib\_send\_bw -d mlx5\_0 -x 3     # 服务端# 单边 RDMA WRITE 带宽ib\_write\_bw -d mlx5\_0 -x 3# 延迟测试ib\_send\_lat -d mlx5\_0 -x 3

100Gbps 网卡的典型预期：

| 操作 | 带宽 | 延迟 |
| --- | --- | --- |
| SEND | ～95 Gbps | ～1-2 μs |
| RDMA WRITE | ～96 Gbps | ～1-2 μs |
| RDMA READ | ～95 Gbps | ～2-3 μs |
| latency 模式 | — | < 1 μs |

如果实测带宽远低于预期，常见原因：

1.  **MTU 没协商到 4096**
    
    ：`ifconfig mtu 9000`或 `ibv_devinfo`检查 active MTU
    
2.  **CPU 亲和性差**
    
    ：poll 线程在多个核间迁移 → 绑核 `taskset -c 0 ib_send_bw`
    
3.  **PCIe 槽位错误**
    
    ：x8 槽位插了 x16 卡 → `lspci -vv`检查协商速率
    
4.  **RoCEv2 拥塞控制未配置**
    
    ：PFC/ECN 没启用会丢包触发重传
    

### 3.11 调试工具

| 工具 | 用途 | 常用命令 | <br> |
| --- | --- | --- | --- |
| `ibstat` | 查看 IB/RoCE 设备状态、端口状态 | `ibstat` | <br> |
| `ibv_devinfo` | 详细设备属性（cap、gid 表大小） | `ibv_devinfo -d mlx5_0` | <br> |
| `ibstatus` | 快速查看链路 UP/DOWN | `ibstatus` | <br> |
| `show_gid`(iproute2) | 查看 GID table | `show_gid` | <br> |
| `perfquery` | 端口性能计数器（CRC 错误、重传） | `perfquery -d mlx5_0 1` | <br> |
| `mlx5dump` | 内部对象 dump（QP、CQ、MTT 等） | `mlx5dump -d mlx5_0 /tmp/dump` | <br> |
| `ib_write_bw`/ `ib_send_bw` | 性能基准 | 见 3.10 节 | <br> |
| `strace` | 跟踪 verbs 系统调用 | `strace -e ibv_xxx ./prog` | <br> |
| `dmesg` | 内核日志（含 RDMA 子系统） | \`dmesg \\ | grep -i mlx5\` |

#### 常见错误排查流程

| 现象 | 排查方向 |
| --- | --- |
| `ibv_modify_qp`返回 `EINVAL` | 检查 `attr_mask`与 `qp_state`是否匹配、参数是否齐全 |
| 收发数据时 QP 进入 ERR | `ibv_query_qp`看 `last_wc_status`；检查 rkey/addr 范围、access flags |
| 性能只有标称值的 50% | 检查 MTU、PCIe 协商速率、CPU 亲和性 |
| RoCEv2 偶发 RNR | 对端 RQ 深度不够；增大 `max_recv_wr`或减少突发发送 |
| 跨 NUMA 性能骤降 | 内存注册绑在远端 NUMA；绑核 + 绑内存 |

## 4\. 完整代码示例

以下是一个 RC Send-Recv 通信的完整流程（伪代码，OOB 交换部分详见 3.5 节）：

\#include \#define BUF\_SIZE 4096#define CQ\_DEPTH  32struct rdma\_res {struct ibv\_context \*ctx;struct ibv\_pd       \*pd;struct ibv\_mr       \*mr;struct ibv\_cq       \*cq;struct ibv\_qp       \*qp;char                 buf\[BUF\_SIZE\] \_\_attribute\_\_((aligned(4096)));};// 初始化设备、PD、MR、CQ、QPstatic int rdma\_init(struct rdma\_res \*res){struct ibv\_device \*\*dev\_list;int num;// Step 1: 设备初始化dev\_list = ibv\_get\_device\_list(&num);res->ctx = ibv\_open\_device(dev\_list\[0\]);ibv\_free\_device\_list(dev\_list);if (!res->ctx) return -1;// Step 2: PDres->pd = ibv\_alloc\_pd(res->ctx);if (!res->pd) return -1;// Step 2: MRres->mr = ibv\_reg\_mr(res->pd, res->buf, BUF\_SIZE,IBV\_ACCESS\_LOCAL\_WRITE |IBV\_ACCESS\_REMOTE\_WRITE);if (!res->mr) return -1;// Step 3: CQres->cq = ibv\_create\_cq(res->ctx, CQ\_DEPTH, NULL, NULL, 0);if (!res->cq) return -1;// Step 4: QPstruct ibv\_qp\_init\_attr qia = {.send\_cq = res->cq, .recv\_cq = res->cq,.qp\_type = IBV\_QPT\_RC,.cap = { .max\_send\_wr = 16, .max\_recv\_wr = 16,.max\_send\_sge = 1, .max\_recv\_sge = 1 },};res->qp = ibv\_create\_qp(res->pd, &qia);if (!res->qp) return -1;return 0;}// QP 状态转换：RESET → INIT → RTR → RTSstatic int rdma\_connect(struct rdma\_res \*res, uint32\_t remote\_qpn,union ibv\_gid remote\_gid){struct ibv\_qp\_attr attr;int flags;// RESET → INITmemset(&attr, 0, sizeof(attr));attr.qp\_state = IBV\_QPS\_INIT;attr.port\_num = 1;attr.pkey\_index = 0;flags = IBV\_QP\_STATE | IBV\_QP\_PORT | IBV\_QP\_PKEY\_INDEX;if (ibv\_modify\_qp(res->qp, &attr, flags)) return -1;// INIT → RTRmemset(&attr, 0, sizeof(attr));attr.qp\_state = IBV\_QPS\_RTR;attr.path\_mtu = IBV\_MTU\_1024;attr.dest\_qpn = remote\_qpn;attr.rq\_psn = 0;attr.ah\_attr.is\_global = 1;attr.ah\_attr.grh.dgid = remote\_gid;attr.ah\_attr.grh.sgid\_index = 1;  // RoCEv2，根据实际配置attr.ah\_attr.port\_num = 1;flags = IBV\_QP\_STATE | IBV\_QP\_AV | IBV\_QP\_PATH\_MTU |IBV\_QP\_DEST\_QPN | IBV\_QP\_RQ\_PSN;if (ibv\_modify\_qp(res->qp, &attr, flags)) return -1;// RTR → RTSmemset(&attr, 0, sizeof(attr));attr.qp\_state = IBV\_QPS\_RTS;attr.sq\_psn = 0;attr.timeout = 14;attr.retry\_cnt = 7;attr.rnr\_retry = 7;attr.max\_rd\_atomic = 1;attr.min\_rnr\_timer = 12;flags = IBV\_QP\_STATE | IBV\_QP\_SQ\_PSN | IBV\_QP\_TIMEOUT |IBV\_QP\_RETRY\_CNT | IBV\_QP\_RNR\_RETRY |IBV\_QP\_MAX\_QP\_DEST\_RD\_ATOMIC | IBV\_QP\_MIN\_RNR\_TIMER;if (ibv\_modify\_qp(res->qp, &attr, flags)) return -1;return 0;}// 数据收发// 注意：post 一个 recv + 一个 send 会产生 2 个 WC，必须循环 poll 直到拿全static int rdma\_send\_recv(struct rdma\_res \*res){struct ibv\_sge sge;struct ibv\_send\_wr swr, \*bad\_swr;struct ibv\_recv\_wr rwr, \*bad\_rwr;struct ibv\_wc wc;int polled, expected = 2;  // 1 recv + 1 send// 先 post recv（必须在 send 之前，否则对端数据到来时 RQ 无可用 buffer）sge.addr = (uint64\_t)res->buf;sge.length = BUF\_SIZE;sge.lkey = res->mr->lkey;memset(&rwr, 0, sizeof(rwr));rwr.wr\_id = 1;rwr.sg\_list = &sge;rwr.num\_sge = 1;if (ibv\_post\_recv(res->qp, &rwr, &bad\_rwr))return -1;// post sendmemset(&swr, 0, sizeof(swr));swr.wr\_id = 2;swr.sg\_list = &sge;swr.num\_sge = 1;swr.opcode = IBV\_WR\_SEND;swr.send\_flags = IBV\_SEND\_SIGNALED;if (ibv\_post\_send(res->qp, &swr, &bad\_swr))return -1;// 循环 poll 直到拿到所有期望的 WC，并对每个 WC 检查 statuswhile (expected > 0) {polled = ibv\_poll\_cq(res->cq, 1, &wc);if (polled < 0) return -1;if (polled == 0) continue;  // 实际应让出 CPU 或用事件驱动if (wc.status != IBV\_WC\_SUCCESS) {fprintf(stderr, "WC error: %s (wr\_id=%lu)\\n",ibv\_wc\_status\_str(wc.status), wc.wr\_id);return -1;}expected--;}return 0;}// 清理（逆序）static void rdma\_cleanup(struct rdma\_res \*res){ibv\_destroy\_qp(res->qp);ibv\_destroy\_cq(res->cq);ibv\_dereg\_mr(res->mr);ibv\_dealloc\_pd(res->pd);ibv\_close\_device(res->ctx);}

## 5\. 总结

-   **RDMA 编程的本质是用户态直通硬件**
    
    。所有元素（PD、MR、CQ、QP、WR、WC）都是围绕"应用 → 用户态驱动 → HCA"这条零拷贝路径设计的，中间没有内核介入。
    
-   **QP 状态机是最大的心智负担**
    
    。RTR/RTS 的参数填写是新手最容易出错的地方——特别是 `ah_attr`中 GID 的配置和 RC/UC/UD 的参数差异。状态转换失败时，检查 `ibv_modify_qp()`的返回值并结合 `errno`定位原因；如果 `RTR`转换成功但收发时报错，重点检查 `sgid_index`、rkey/addr 范围、access flags。
    
-   **资源有严格的依赖链**
    
    。创建顺序决定销毁顺序——正向创建、逆向销毁。这条链上的任何一环出错都可能影响上层资源的可用性，排查时从链头（设备）开始检查。
    
-   **MR 是性能的关键**
    
    。注册是慢操作，生产代码必须预注册大块对齐内存（MR 池 + hugepage），运行时通过 SGE 在池内寻址，把"快慢"严格分离。
    
-   **错误处理不可省略**
    
    。RDMA 通信链路层错误是常态（CRC、重传、RNR），必须用 async event + WC status 双通道监听 QP 状态，ERR 状态下的 QP 要走 destroy + 重建流程。
    
-   **RoCEv2 与 IB 是两套生态**
    
    。RoCEv2 走 UDP/IP 路由，只需 GID 不需 LID，且依赖无损网络（PFC + ECN）；IB 用 LID 走子网管理。混用概念是新手常见错误来源。
    

原创文章 · 作者：fengyp

欢迎转发，转载请联系作者

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/564d04f3_1781661538157?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI2NjIwNzYyOQ%3D%3D%26mid%3D2649619933%26idx%3D1%26sn%3D22e32d2ab4aa431741879b02fa0a23a4%26chksm%3Df33d73c5e9cf13dd3ba8fd50fd1d3b75148f4ed41c2ef9f315bc0cd0710dc41a91c79a7d14f4%26mpshare%3D1%26scene%3D1%26srcid%3D06177Dw6DIOrMmZFTRx4Kqzb%26sharer_shareinfo%3D00ed3254d7889984261ad276eb240c8f%26sharer_shareinfo_first%3D00ed3254d7889984261ad276eb240c8f%23rd&s=obsidian)