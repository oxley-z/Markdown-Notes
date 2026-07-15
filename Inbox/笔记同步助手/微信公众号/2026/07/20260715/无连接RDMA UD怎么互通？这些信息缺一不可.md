---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487471&idx=1&sn=f29d36de517deaf1942b0dae6dc0f01e&chksm=c0ef5578cdd5f105bc020f52b2a05adc8a85756f5a5eada79afe5b36505ca83223e8a5e62a0c&mpshare=1&scene=1&srcid=07153AxTZpz3olSALfHtiid6&sharer_shareinfo=a4d68ef34325e2aba026ed0e5d9e05c5&sharer_shareinfo_first=a4d68ef34325e2aba026ed0e5d9e05c5#rd
saved: 2026-07-15 13:04:29
tags:
  - 笔记同步助手
id: 0664a04f-2898-4885-a76c-b79bb9352a25
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-15 12:35

## 一、RoCEv2 UD 报文 PKey、QKey 作用分层详解

### 1\. PKey（Partition Key，16bit，BTH 头部固定字段）

层级：RoCE 子网分区隔离（全局粗粒度权限）

1\. 所有类型QP（RC/UC/UD）强制校验，并非UD独有；

2\. 报文结构：BTH 16bit固定位置携带PKey，网卡硬件第一层校验；

### 3\. 核心作用：

\- 网络分层隔离：同一物理交换机/网卡划多个业务分区，只有两端QP配置相同PKey才能互通；

\- 管控多播组：UD多播组绑定指定PKey，跨分区节点无法加入、接收多播；

\- 基础安全过滤：PKey不匹配直接硬件丢包，不占用接收队列资源。

4\. 特性：一张网卡端口所有QP共享同一份允许的PKey白名单。

### 2\. QKey（Queue Key，32bit，UD专属校验字段）

层级：单QP细粒度访问密钥，仅UD生效（RC/UC无校验）

1\. 报文来源：发送端在 ibv\_send\_wr 中填充目标QP的QKey，下发网卡；报文内部携带，不在GRH/BTH静态头部；

2\. 校验时机：PKey校验通过、网卡根据Dst QPN匹配到UD QP后，执行第二层校验；

### 3\. 核心作用（UD无连接特性必备）：

\- 防DoS：UD无握手，任意主机都能向你的QPN发包；QKey不匹配直接丢弃，避免恶意报文耗尽RQ接收WR；

\- 同分区QP隔离：同一PKey下，多张UD QP通过不同QKey隔离业务，A业务无法投递消息到B业务QP；

\- 访问鉴权：相当于QP的“访问密码”，只有持有正确QKey的发送端才能投递消息。

4\. 特性：每个UD QP独立配置，互不干扰。

### 3\. RoCEv2 UD收包完整过滤流程

以太网帧 → GRH解析 → BTH解析 →

#### ① 校验PKey（失败直接丢包）→

#### ② 根据Dst QPN查找本地UD QP上下文 →

#### ③ 校验QKey（失败直接丢包）→

合法报文送入RQ，生成WC完成事件。

## 二、两个UD QP正常通信，必须互相同步的全部信息

RoCEv2 UD 无连接、无CM握手，所有参数必须业务层手动同步（TCP/配置中心/共享配置文件传递）

### 1\. 网卡寻址信息（定位远端网卡端口）

1\. 远端GID：RoCEv2全局端口标识，填充GRH头部， rdma\_resolve\_addr 解析；

2\. AH地址句柄：本地内核缓存远端L2 MAC、GRH、路由信息，由 rdma\_resolve\_route 生成，发包复用。

### 2\. QP标识（定位远端目标队列）

远端QPN：目标UD QP硬件编号，网卡依靠QPN索引QP上下文。

### 3\. 权限校验密钥（硬件两层过滤必备）

1\. PKey：两端QP必须配置完全相同的分区密钥；

2\. 目标QP的QKey：发送WR必须填写接收端UD QP绑定的QKey。

### 4\. 传输协商参数（收发匹配，否则报文异常/截断）

1\. QP MTU：两端统一，UD不支持硬件分片，超长报文上层需手动拆分；

2\. 最大收发资源： max\_recv\_wr 、 max\_recv\_sge ，发送端报文不能超出对方接收承载上限；

3\. 流控参数：rnr\_retry\_count、rnr\_timeout（UD极少触发RNR，但参数不一致会产生异常计数器）。

### 5\. 可选自定义业务信息

private\_data：无CM握手，业务自定义元数据（版本、业务ID、加密密钥等）只能通过业务消息交互传递。

### 极简清单（发送端必须获取对端全部字段）

### 1\. 远端GID

### 2\. 远端QPN

### 3\. 对端UD QP的QKey

### 4\. 统一的PKey

### 5\. 协商一致的MTU

### 6\. 对方接收队列最大WR/SGE限制

  

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/4a770547_1784091867762?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487471%26idx%3D1%26sn%3Df29d36de517deaf1942b0dae6dc0f01e%26chksm%3Dc0ef5578cdd5f105bc020f52b2a05adc8a85756f5a5eada79afe5b36505ca83223e8a5e62a0c%26mpshare%3D1%26scene%3D1%26srcid%3D07153AxTZpz3olSALfHtiid6%26sharer_shareinfo%3Da4d68ef34325e2aba026ed0e5d9e05c5%26sharer_shareinfo_first%3Da4d68ef34325e2aba026ed0e5d9e05c5%23rd&s=obsidian)