---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487480&idx=1&sn=1e3bfac301a092bdaa5c68cc02b242d5&chksm=c000049cd390911822ba45fd423b866aff08135723a79967aabdbb643e3f68313e7648e85196&mpshare=1&scene=1&srcid=0716az28InZrMnsuIAx79XLn&sharer_shareinfo=e2712a5f6e6dd5f655800da41fbd44e9&sharer_shareinfo_first=e2712a5f6e6dd5f655800da41fbd44e9#rd
saved: 2026-07-16 11:37:36
tags:
  - 笔记同步助手
id: c1b59ba2-9811-4d81-8dc4-4d05833582a6
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-16 11:30

### 分类1：UD 不可靠数据报（最常用，无连接）

## 1）UD Send Only（0x64）标准UD报文

外层封装 + BTH(12B) + DETH(8B) + Payload + ICRC

## 2）UD Send with Immediate（0x65）带立即数

外层封装 + BTH + DETH + ImmDt(4B) \+ Payload + ICRC

### UD必带DETH，QKey存放在DETH，是UD鉴权关键；无RETH/AETH

### UD报文的payload最大只能一个mtu

  

分类2：RC 可靠连接QP（AI集群主流：Send / RDMA Write / RDMA Read / ACK）

### 1\. Send 双边收发语义（最基础，需要两端Post Recv WR）

#### ① Send Only 单包小包

外层 + BTH + Payload + ICRC

#### ② Send with Imm 带立即数

外层 + BTH + ImmDt + Payload + ICRC

#### ③ 大数据分片：Send First / Middle / Last

### BTH标记分片标记，仅Last包可带ImmDt，无额外扩展头

### 2\. RDMA Write 单边写（远端无需提前Post Recv）

#### ① Write First / Write Only（首包/整包）

外层 + BTH + RETH(16B) + Payload + ICRC

### RETH携带远端VA、RKey，告知网卡写入远端哪块MR内存

#### ② Write Middle / Write Last

外层 + BTH + Payload + ICRC（RETH只在第一个分片出现）

### 3\. RDMA Read 单边读（分请求包+应答包两个报文）

#### （1）Read Request 读请求报文（发往对端）

外层 + BTH + RETH(16B) + ICRC

### RETH标明想要读取的远端地址、RKey、长度，无载荷Payload

#### （2）Read Response 读返回报文（对端回复）

外层 + BTH + AETH(8B) \+ Payload + ICRC

### AETH携带ACK序列号，用于RC可靠重传排序

### 4\. RC ACK 纯应答报文（无数据，流量控重传）

外层 + BTH + AETH(8B) \+ ICRC

无Payload，仅用来确认PSN接收完成、滑动窗口前移、RNR通知

### 分类3：UC 不可靠连接QP（极少使用）

结构和RC几乎完全一致：支持Send、RDMA Write；不支持RDMA Read

无ACK重传机制，丢包不会自动重传；头部格式同RC对应报文。

### 分类4：特殊控制报文（拥塞、异常通知）

### 1\. CNP 拥塞通知报文（RoCE拥塞控制标准包）

外层+IP/UDP掩码处理 + BTH + CNP载荷 + ICRC

用于ECN拥塞标记，通知发送端降速，头部无额外扩展头

### 2\. RNR NAK 报文（接收队列满重试通知）

外层 + BTH + AETH(NAK标记) \+ ICRC

接收端RQ无WR时回复，触发发送端RNR重试计数上涨

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/00403ef5_1784173055311?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487480%26idx%3D1%26sn%3D1e3bfac301a092bdaa5c68cc02b242d5%26chksm%3Dc000049cd390911822ba45fd423b866aff08135723a79967aabdbb643e3f68313e7648e85196%26mpshare%3D1%26scene%3D1%26srcid%3D0716az28InZrMnsuIAx79XLn%26sharer_shareinfo%3De2712a5f6e6dd5f655800da41fbd44e9%26sharer_shareinfo_first%3De2712a5f6e6dd5f655800da41fbd44e9%23rd&s=obsidian)