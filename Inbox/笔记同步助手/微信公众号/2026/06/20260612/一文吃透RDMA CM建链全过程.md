---
author: 胡胡子的
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU1NzkxNTQ2OA==&mid=2247488726&idx=1&sn=506a64bd9d382606d597d19898410470&chksm=fd9522f58dfe32ab88d431704761257a2d9d5abdf512906e739367ef1af7dd245942851ad5c5&mpshare=1&scene=1&srcid=0612IyPfhF0nXiYvhrdefhKH&sharer_shareinfo=3bdf819d0f7e0acad7ff6c2d00d72085&sharer_shareinfo_first=3bdf819d0f7e0acad7ff6c2d00d72085#rd
saved: 2026-06-12 12:03:11
tags:
  - 笔记同步助手
id: 174c4fd1-8282-4485-900e-9b4dfbe68ae6
---

公众号名称：linux高性能网络

作者名称：胡胡子的

发布时间：2026-06-12 10:36

![[../../../../images/7a0d56329731bf03f1dc770cbbc3aa0b_MD5.jpg]]

一文吃透RDMA CM建链全过程

---

# 前言

做高性能网络、分布式存储、云原生高性能通信的同学，几乎绕不开RDMA技术。

很多人能熟练写出RDMA读写代码，却始终搞不懂RDMA底层连接到底是怎么建立的：

RDMA CM到底是干嘛的？和TCP三次握手有什么区别？

地址解析、路由解析先后顺序为什么不能乱？

明明连接已经建立，为什么还是无法收发RDMA数据？

QP状态流转全过程是什么？面试必问状态机怎么背？

今天这篇文章，零废话、全干货、流程一步不拉满，带你彻底搞懂RDMA CM建链全流程。

---

# 一、先搞懂本质：RDMA CM是什么？

## 1\. 通俗类比

把整个RDMA通信比作打电话：

RDMA CM（连接管理器）：负责拨号、确认对方号码、协商通话参数、接通挂断，只负责信令沟通，不传递通话内容

QP队列对：真正传输语音（业务数据）的通道

## 2\. 官方核心定义

RDMA CM 是RDMA体系下的信令管理层（控制面），全程不参与任何业务数据传输，只专心做一件事：管理RDMA全生命周期连接。

## 3\. CM五大核心工作职责

设备发现：识别两端IB/RoCE网卡设备

地址解析：IP和RDMA专属GID/QPN互相转换

安全握手：两端密钥、权限校验

QP状态迁移：驱动通信队列完成状态切换

连接管理：完整实现连接建立、异常重连、连接断开

核心误区提醒：很多新手以为CM会转发数据，大错特错！RDMA所有读写、收发数据全部走硬件网卡直通，CPU无拷贝，CM全程只跑控制信令。

---

# 二、硬核核心：RDMA建链6大完整阶段（时序逐步骤拆解）

整个RDMA建链分为固定6个阶段，客户端主动发起、服务端被动监听，时序严格固定，一步都不能颠倒，下面结合源码接口+底层事件双维度讲解。

## 阶段1：两端初始化基础资源（客户端+服务端都要执行）

连接建立前，双方必须提前开好底层基础资源，此时QP初始状态为RESET（重置态），完全无法通信。

枚举并打开IB设备：ibv\_get\_device\_list() → ibv\_open\_device()

分配保护域PD：ibv\_alloc\_pd()（统一管控QP/CQ内存权限，做隔离）

创建完成队列CQ：ibv\_create\_cq()，接收工作请求完成通知

创建通信队列QP：ibv\_create\_qp()，初始状态锁定RESET

## 阶段2：服务端被动监听，等待客户端接入

服务端作为被动端，提前开启端口监听，阻塞等待连接请求：

创建CM事件通道：rdma\_create\_event\_channel()，统一接收所有连接事件

创建连接标识ID：rdma\_create\_id()，绑定TCP类型RDMA传输

绑定本地IP+端口：rdma\_bind\_addr()

开启端口监听：rdma\_listen()

执行完毕后，服务端进入监听状态，安静等待客户端握手请求。

## 阶段3：客户端主动发起寻址+路由解析

客户端想要连上服务端，必须先做两次关键解析，顺序强制：先地址、后路由，不可调换。

地址解析 rdma\_resolve\_addr()：通过对端IP，解析出RDMA硬件地址GID，触发地址成功/失败事件

路由解析 rdma\_resolve\_route()：计算端到端硬件传输路径、链路MTU、服务等级，打通底层物理通路

## 阶段4：核心握手：两端交换全部连接参数

这一步等同于TCP三次握手，是建链最关键的一步：

客户端调用 rdma\_connect() 发起连接请求

服务端收到CONNECT\_REQUEST连接请求事件

服务端调用 rdma\_accept() 同意连接

两端同时收到ESTABLISHED连接建立事件

两端本次握手会互相同步全部核心通信参数：远端QPN、GID地址、PSN序列号、RKey密钥、MSS最大分段大小等。

## 阶段5：关键分水岭：QP切换至RTS可发送状态

超级高频坑点：收到ESTABLISHED连接成功事件 ≠ 可以发RDMA数据！此时QP依旧处于不可用状态。

必须两端手动修改QP状态：

ibv\_modify\_qp(qp, &attr, IBV\_QP\_STATE, IBV\_QPS\_RTS);

只有QP切换为RTS（Ready To Send，就绪发送态），硬件通道才真正打通。

## 阶段6：正式开始硬件直通数据传输

QP就绪后，即可发起四大类原生RDMA操作，全程CPU无内存拷贝，网卡硬件直接收发：

单边无感知读写：RDMA\_WRITE / RDMA\_READ（存储核心场景）

双边消息收发：RDMA\_SEND / RDMA\_RECV（消息通信场景）

---

# 三、极简版状态机（面试直接背）

面试常问QP完整状态流转，精简版一目了然：

✅ 客户端状态流转

RESET → ADDR\_RESOLVED → ROUTE\_RESOLVED → CONNECTING → ESTABLISHED → RTS → DISCONNECTED

✅ 服务端状态流转

IDLE → LISTEN → CONNECT\_REQUEST → ESTABLISHED → RTS → DISCONNECTED

---

# 四、8行极简时间线（复盘一秒回忆全流程）

两端初始化：设备 → PD → CQ → QP(RESET)

服务端：创建事件通道+ID → 绑定地址 → 开启监听

客户端：解析远端地址 → 解析全网路由

客户端主动发起connect连接

服务端接收请求，执行accept应答

双方同步收到连接建立ESTABLISHED事件

两端修改QP状态，切换为RTS就绪态

正式进行RDMA零拷贝数据传输

---

# 五、问题

Q1：RDMA CM和TCP三次握手区别？

TCP握手同时兼顾控制+数据通路；RDMA CM只做控制面信令握手，业务数据完全走网卡硬件直通，全程不经过CPU，性能远高于TCP。

Q2：地址解析和路由解析能颠倒顺序吗？

不能。必须先通过IP解析硬件GID地址，再基于硬件地址计算传输路由，颠倒会直接建链失败。

Q3：连接ESTABLISHED之后为什么发不了数据？

CM层面连接完成只是信令通路打通，QP还停留在初始状态，必须手动切换为RTS状态，数据通道才能真正启用。

Q4：rdma\_id的作用是什么？

贯穿连接全生命周期的核心句柄，绑定事件通道、QP、远端地址、连接参数，管理整条连接所有资源。

Q5：RDMA建链全过程开销在哪里？

仅前期CM握手、状态切换存在极小CPU开销；数据传输阶段无CPU开销，这也是RDMA低延迟、高吞吐的核心原因。

---

# 文末总结

最后一句话彻底记住RDMA CM：

RDMA CM就是RDMA的信令管家，负责寻址、路由、握手、状态管理，铺好所有前置通路；真正的高性能数据传输，全程和CM无关，完全由网卡硬件独立完成。

后续我们会更新：RDMA断链流程、RC/UC/UD三种传输模式区别、RDMA源码核心逻辑拆解，感兴趣可以关注～

---

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/829f0a87_1781236989540?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU1NzkxNTQ2OA%3D%3D%26mid%3D2247488726%26idx%3D1%26sn%3D506a64bd9d382606d597d19898410470%26chksm%3Dfd9522f58dfe32ab88d431704761257a2d9d5abdf512906e739367ef1af7dd245942851ad5c5%26mpshare%3D1%26scene%3D1%26srcid%3D0612IyPfhF0nXiYvhrdefhKH%26sharer_shareinfo%3D3bdf819d0f7e0acad7ff6c2d00d72085%26sharer_shareinfo_first%3D3bdf819d0f7e0acad7ff6c2d00d72085%23rd&s=obsidian)