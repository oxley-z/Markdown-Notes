---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487459&idx=1&sn=48e904236b2f68d06d0b1b055be84113&chksm=c07c46c408b672a5732785c6063201a34bff227af51f6dd268e34abd8f7898fe4e0d4fc90c6f&mpshare=1&scene=1&srcid=0711zQkFwbLLWmNtojIKr2ra&sharer_shareinfo=a196f4305ad9f6048416059b3a8859c5&sharer_shareinfo_first=a196f4305ad9f6048416059b3a8859c5#rd
saved: 2026-07-11 18:34:10
tags:
  - 笔记同步助手
id: 9221d2ea-a997-440a-84e5-6c8f226c23b6
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-11 18:24

### rdma\_resolve\_addr() 核心一句话

### RoCE‑v2下 基于对端ip地址

### rdma\_resolve\_addr = 计算对端GID + ARP获取对端MAC

### 1\. 整体作用概括

RoCE‑v2是以太网里跑的RDMA，它和IB不一样：IB有专门的GID、LID；RoCE‑v2依靠IP地址定位远端机器。

  

### rdma\_resolve\_addr 核心任务：

通过对端的IP地址，解析出RDMA层需要的 Remote GID + Remote MAC，并且填充到rdma\_addr结构体；后续创建QP的时候，驱动依靠这个结果组装RoCE‑v2数据包的底层头部（GRH头部）。

### 一句话通俗概括：

TCP里gethostbyname把域名转IP； rdma\_resolve\_addr 把对方IP翻译成RoCE‑v2专用的GID和MAC地址。

### 2\. RoCE‑v2基础前提（必须清楚）

1\. RoCE‑v2报文结构：以太网头 + GRH头 + RDMA Payload。

### 2\. GRH（Global Route Header）里面最重要字段：

\- Source GID（本端GID）

\- Destination GID（远端GID）

### 3\. RoCE‑v2的GID格式：

GID = GID\_PREFIX(64bit) + IPv6‑formatted IPv4地址(64bit)

RoCE‑v2强制用IPv6格式存放IPv4地址，遵循： 0:0:0:0:0:0:FFFF:IPv4‑in‑hex ；

所以拿到对方IPv4就能算出对应的GID前缀部分。

4\. 但是只有GID不够，网卡发包还需要对端MAC地址。

获取MAC就需要ARP。

### 3\. rdma\_resolve\_addr完整执行原理（RoCE‑v2场景分步拆解）

函数原型：

int rdma\_resolve\_addr(

struct rdma\_cm\_id \*id,

struct sockaddr \*src\_addr,

struct sockaddr \*dst\_addr,

int timeout\_ms);

## 步骤1：用户传入dst\_addr（对端IPv4）

rdma\_cm（RDMA Connection Manager，内核rdma\_cm模块）接收目标IP。

## 步骤2：内核构造对应的Destination‑GID

RoCE‑v2规则：

\- GID前64位：来自网卡配置的GID Prefix（一般0xfe80::或者管理员配置）

\- 后64位：把IPv4转为IPv4‑mapped‑IPv6格式 ::ffff:a.b.c.d 。

这一步软件层面就能算出GID，不用网络交互。

## 步骤3：内核发起ARP请求拿到远端MAC（关键网络动作）

rdma\_cm内核模块调用内核网络栈：

1\. 查看ARP缓存里有没有目标IP‑MAC映射；

2\. 缓存不存在就发出ARP广播获取对端MAC；

这是RoCE‑v2区别于IB的重点：IB不需要ARP，RoCE‑v2必须ARP。

## 步骤4：内核填充 struct rdma\_addr

解析完成之后，rdma\_cm把下面信息保存进rdma\_cm\_id内部的rdma\_addr：

1\. remote\_gid：GRH头要用；

2\. dst\_mac：填充以太网帧头；

3\. 还有本机GID、本机MAC；

之后后续 rdma\_resolve\_route 再进一步处理路由信息。

## 步骤5：通知上层应用（事件回调）

### rdma\_cm通过事件队列返回：

RDMA\_CM\_EVENT\_ADDR\_RESOLVED ；

应用收到这个事件，代表地址解析结束，之后才可以创建QP、设置QP的属性（ah‑address handle，AH：Address Handle）。

AH对象：就是把GID和MAC打包交给硬件网卡，智能网卡后续发包就依靠AH填充GRH和以太网头部。

### 4\. 和IB模式做对比，更容易理解边界

### 1\. IB模式：

\- rdma\_resolve\_addr依靠SA服务（Subnet‑Agent）查询远端GID和LID；需要和IB交换机通信；

  

### 2\. RoCE‑v2模式：

\- 没有SA，抛弃IB‑SA；

\- 完全依靠ARP获取MAC + 公式计算GID；

所以RoCE‑v2下 rdma\_resolve\_addr = 计算GID + ARP获取MAC。

### 5\. 常见误区（工程里高频踩坑）

### 1\. 误区1：rdma\_resolve\_addr只是用户态函数

实际大部分工作是内核模块 rdma\_cm 完成，用户态只是发起调用并等待事件；

### 2\. 误区2：只有Perftest、rping才调用它

只要使用rdma‑cm建立连接的程序（RC QP连接模式）都会调用；如果是UD或者Raw Verbs不走rdma‑cm，开发者自己手动做ARP和构造GID；

3\. 坑点：ARP失败，rdma\_resolve\_addr超时失败。

\- PFC配置错误、VLAN配置不一致、防火墙拦截ARP都会导致解析超时；

  

### 4\. 另一个关键点：

rdma\_resolve\_addr

只完成IP→GID+MAC；

后面还要调用 rdma\_resolve\_route 生成AH（Address Handle）给到硬件网卡。AH才是下发给硬件的关键结构，CX‑8网卡依靠AH里的GID、MAC组装RoCE‑v2报文。

### 6\. 总结

## 在RoCE‑v2环境下：

1\. rdma\_resolve\_addr 借助rdma‑cm内核模块，基于传入的对端IPv4地址按照RoCE‑v2规则算出远端GID；

2\. 通过ARP协议获取对端MAC地址；

3\. 将GID和MAC信息保存下来，之后生成AH；

4\. 完成RDMA地址解析，为QP创建和RoCE报文组包准备底层信息。

关注我，一起了解更多，谢谢您！

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/1e2a99cf_1783766048612?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487459%26idx%3D1%26sn%3D48e904236b2f68d06d0b1b055be84113%26chksm%3Dc07c46c408b672a5732785c6063201a34bff227af51f6dd268e34abd8f7898fe4e0d4fc90c6f%26mpshare%3D1%26scene%3D1%26srcid%3D0711zQkFwbLLWmNtojIKr2ra%26sharer_shareinfo%3Da196f4305ad9f6048416059b3a8859c5%26sharer_shareinfo_first%3Da196f4305ad9f6048416059b3a8859c5%23rd&s=obsidian)