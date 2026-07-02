---
author: 胡胡子的
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU1NzkxNTQ2OA==&mid=2247488754&idx=1&sn=c147d6f59bf495cb026bbc9b52b5a32f&chksm=fdc5154afba8aa989e30e1401dcc3f08cbda8e831ada95103909bd7b559820878be8ee8fa366&mpshare=1&scene=1&srcid=07021J1vqssn9FBP6yig4QJ9&sharer_shareinfo=3c6bd9cf9df5dc2727bb47ae54221598&sharer_shareinfo_first=3c6bd9cf9df5dc2727bb47ae54221598#rd
saved: 2026-07-02 11:40:02
tags:
  - 笔记同步助手
id: 8dd092ee-d2e7-461a-93b9-f9c22ebe8ced
---

公众号名称：linux高性能网络

作者名称：胡胡子的

发布时间：2026-07-02 11:31

---

# 💡 前言

只要你做 RDMA 开发，一定会和ibv\_wr\_opcode打交道。

它决定网卡 HCA 本次 WR（Work Request）到底要做什么：是读写远端内存？发消息？做原子锁？还是回收权限？

很多人学 RDMA 多年依然搞不懂：

✅RDMA\_WRITE 和 SEND 到底差在哪？

✅带 IMM 立即数的意义是什么？

✅单边/双边操作怎么区分？

✅原子操作、MW绑定、RKEY失效分别适用什么场景？

---

# 📌 基础概念速览

WR：开发者下发给网卡的工作请求

opcode：网卡硬件指令，决定数据通路行为

单边操作：远端CPU无感知、无需投递Recv

双边操作：必须远端提前投递Recv WR，成对消耗

---

# 📚 12种操作码总枚举

RDMA 标准 Verbs 一共 12 种 WR 操作码：

enum ibv\_wr\_opcode {

IBV\_WR\_RDMA\_WRITE,

IBV\_WR\_RDMA\_WRITE\_WITH\_IMM,

IBV\_WR\_SEND,

IBV\_WR\_SEND\_WITH\_IMM,

IBV\_WR\_RDMA\_READ,

IBV\_WR\_ATOMIC\_CMP\_AND\_SWP,

IBV\_WR\_ATOMIC\_FETCH\_AND\_ADD,

IBV\_WR\_LOCAL\_INV,

IBV\_WR\_BIND\_MW,

IBV\_WR\_SEND\_WITH\_INV,

IBV\_WR\_TSO,

IBV\_WR\_DRIVER1,

};

---

# 🗺️ QP 支持能力总表（高频必看）

RC（可靠连接）：支持全部功能，存储/数据库主流

UC（不可靠连接）：不支持读、原子操作

UD（数据报）：仅支持普通SEND，无RDMA读写

![[Inbox/笔记同步助手/微信公众号/2026/07/images/36b33e02126d8727f0acf3d6b9687e1b_MD5.jpg]]

---

# 一、单边RDMA读写类（高性能、零远端CPU）

![[Inbox/笔记同步助手/微信公众号/2026/07/images/df52da1602728ee2dba22c80093cdee0_MD5.jpg]]

## 1️⃣ IBV\_WR\_RDMA\_WRITE｜远端内存直写

## 核心能力

本地网卡直接把本地内存数据DMA 写入远端内存。

## 特点

远端无需CPU参与、无需Recv WR

只有本地产生CQE

性能最高、延迟最低

适用场景：大数据推送、流式写入、日志落地

## 2️⃣ IBV\_WR\_RDMA\_WRITE\_WITH\_IMM｜带通知的RDMA写

最常用的工业级组合！

在 RDMA\_WRITE 基础上，附带4字节立即数 imm\_data。

关键区别：

数据包到达远端后，强制触发远端Recv CQE，唤醒远端业务线程。

👉高速传输 + 远端通知一举两得

用途：分片标记、长度传递、消息类型、写完唤醒业务

## 3️⃣ IBV\_WR\_RDMA\_READ｜远端内存读取

主动从远端内存拉取数据到本地。

## 特点

远端全程无感知

仅本地返回数据、生成CQE

场景：读取远端元数据、索引、全局状态

---

# 二、双边SEND消息类（信令、控制通道）

## 4️⃣ IBV\_WR\_SEND｜普通消息发送

传统双边消息模型，类似TCP发送。

必须配对：远端必须提前Post Recv，否则丢包报错。

场景：握手、心跳、信令、控制命令

## 5️⃣ IBV\_WR\_SEND\_WITH\_IMM｜带立即数消息

SEND + 4字节标识，用于区分不同消息类型。

## 6️⃣ IBV\_WR\_SEND\_WITH\_INV｜发消息 + 远端RKEY失效

非常重要的安全操作

消息到达远端后：

先硬件失效远端RKEY，再通知业务

彻底解决：内存释放后，远端残留RKEY非法访问的BUG

场景：内存池归还、缓冲区下线、动态解绑

## 7️⃣ IBV\_WR\_TSO｜网卡大报文分片卸载

仅 RoCEv2 以太网环境支持。

超大报文由网卡硬件自动分片，解放CPU。

---

# 三、硬件原子操作类（分布式无锁核心）

仅 RC QP 支持！！

![[Inbox/笔记同步助手/微信公众号/2026/07/images/c6941c9d2645f75e3efc9123bbe036f6_MD5.jpg]]

## 8️⃣ IBV\_WR\_ATOMIC\_CMP\_AND\_SWP｜CAS比较交换

硬件原子三步：

读取远端原值

对比预期值

相等则替换、不相等不变

场景：分布式锁、状态机、主从切换

## 9️⃣ IBV\_WR\_ATOMIC\_FETCH\_AND\_ADD｜FAA原子累加

远端内存原子 +N，返回旧值。

场景：全局计数器、流水号、分片索引、引用计数

---

# 四、本地内存管理类（无网络流量）

## 🔟 IBV\_WR\_LOCAL\_INV｜本地RKEY失效

纯本地操作，不发包

主动作废本地MR的远程访问权限，防止野访问。

内存安全必用：释放内存前必须INV

## 1️⃣1️⃣ IBV\_WR\_BIND\_MW｜绑定内存窗口

MR = 大块内存

MW = MR 内的一小块动态窗口

BIND\_MW 可以动态授权、回收小段内存访问权限

场景：多租户隔离、动态缓冲区租借

---

# 五、厂商私有扩展

## 1️⃣2️⃣ IBV\_WR\_DRIVER1｜私有硬件指令

无标准定义，由网卡厂商自定义。

业务代码禁止使用，无兼容性！

---

# 🎯 开发场景快速选型（速查）

✅高速写数据 + 远端通知→ RDMA\_WRITE\_WITH\_IMM

✅主动拉取远端数据→ RDMA\_READ

✅握手、心跳、信令→ SEND / SEND\_WITH\_IMM

✅分布式锁、状态更新→ ATOMIC\_CMP\_AND\_SWP

✅全局自增计数器→ ATOMIC\_FETCH\_AND\_ADD

✅安全释放远端内存权限→ SEND\_WITH\_INV

✅本地内存防野访问→ LOCAL\_INV

✅动态细粒度权限控制→ BIND\_MW

---

# ⚠️ 高频踩坑总结

UD QP 不支持读写、原子操作，大数据必须RC

纯 WRITE不会唤醒远端，业务卡死大概率是少了 IMM

原子操作仅64bit、仅RC支持

LOCAL\_INV 本地失效，SEND\_WITH\_INV 远端失效，不能混用

DRIVER1 绝不通用，禁止业务使用

---

# 💬 结语

opcode 是 RDMA 的灵魂，读懂12种操作码，才算真正读懂 RDMA 架构。

后续会持续更新：RDMA 内存模型、WQE、CQE机制、拥塞控制、高性能轮询模型。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/e187c6aa_1782963600197?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU1NzkxNTQ2OA%3D%3D%26mid%3D2247488754%26idx%3D1%26sn%3Dc147d6f59bf495cb026bbc9b52b5a32f%26chksm%3Dfdc5154afba8aa989e30e1401dcc3f08cbda8e831ada95103909bd7b559820878be8ee8fa366%26mpshare%3D1%26scene%3D1%26srcid%3D07021J1vqssn9FBP6yig4QJ9%26sharer_shareinfo%3D3c6bd9cf9df5dc2727bb47ae54221598%26sharer_shareinfo_first%3D3c6bd9cf9df5dc2727bb47ae54221598%23rd&s=obsidian)