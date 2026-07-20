---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487489&idx=1&sn=072ef15865162f17ddc352a8eefd621d&chksm=c0436ad44bcaa7967ff732b73664e544577b7623f7bbe495a7e1fb0d518dff8cc205b2421734&mpshare=1&scene=1&srcid=0718QJCFDNYpZMZHLMWXuNSS&sharer_shareinfo=e7e3ce4e49f5c2c9cbee331700ef2ec6&sharer_shareinfo_first=e7e3ce4e49f5c2c9cbee331700ef2ec6#rd
saved: 2026-07-18 20:37:53
tags:
  - 笔记同步助手
id: b7d675de-6069-4f68-af64-57311f8beac6
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-18 19:41

前置关键点：  
1\. 只有RC/DC类型QP才会产生ACK，UD不可靠QP没有ACK报文；  
2\. ACK全部由RNIC硬件自动生成发送，用户态软件不会手动构造ACK包；  
3\. ACK是累积确认：一个ACK里的 E\_PSN 代表「该PSN及之前所有数据包全部连续收齐」，不用逐包回ACK节省带宽；  
4\. CX8/mlx5网卡遵循IBTA标准，ACK触发分为强制必发场景、累积延时触发、异常场景、特殊操作强制应答四大类。  
一、必须立刻发送ACK的强制场景（无延迟）  
1\. 数据包头部 AckReq = 1（发送端主动索要ACK）  
发送方在BTH头部置位 AckReq=1 ，接收网卡收到后立刻回送ACK，用于：  
\- 发送窗口即将耗尽，急需滑动接收窗口Credit；  
\- 测量RTT、探测链路是否存活；  
\- 单小包RPC场景，每包强制确认降低时延。  
2. RDMA READ、ATOMIC原子操作（必须显式回ACK）  
\- RDMA Read：远端需要把本地数据打包发回请求端，最后一个响应包自带AETH（ACK信息），等同于应答；  
\- Atomic（CAS/FetchAdd）：原子操作执行完毕，硬件必须单独发送ACK告知请求方操作完成；  
这两类业务不允许延迟合并ACK，必须即时应答。  
3\. 消息最后分片（Last包）  
一个大消息被拆分为多个报文（First/Middle/Last），接收端收到Last分片时，硬件会触发ACK，确认整条消息完整接收完毕。  
二、累积合并ACK（吞吐优化，最常见场景）  
为了避免海量小包每个都回ACK造成ACK风暴，网卡采用计数+超时双阈值合并应答：  
1\. 计数阈值：ack\_req\_freq  
QP创建时配置参数 ack\_req\_freq ，规则：  
累计收到 2^{ack\_req\_freq} 个有效数据包，硬件自动打包一个累积ACK发出。  
例：freq=2 → 每收到4个包统一回1个ACK；高频大流量默认freq=3/4，大幅减少ACK小包数量。  
2\. 超时定时器：ACK延迟计时器  
收到数据包后启动硬件微秒级定时器（CX8默认几us～十几us）；  
定时器超时前即使包数量没达到计数阈值，也必须发一次ACK，防止发送端超时重传。  
三、异常/流控场景立刻发送ACK/NAK  
1\. 收到乱序包（PSN不连续，丢包判定）  
期望PSN=100，却收到PSN=102，网卡立即回复NAK否定应答，告知发送方缺失101号包，触发Go-Back-N重传。  
2\. 接收RQ窗口即将耗尽（接收credit不足）  
接收端RQ剩余WR数量很少，硬件主动回带窗口信息的ACK，让发送端感知接收容量，降速避免报文被丢弃，也是PFC无损网络配套的流控联动机制。  
3\. 校验失败、PKey/QKey/MR权限错误  
报文校验非法，硬件回复带错误码的ACK/异步事件，告知对端终止QP或重传。  
四、连接层面隐性ACK（非数据ACK）  
1\. rdma\_accept建连完成：两端QP状态切到RTS，握手阶段的控制报文也携带隐性确认；  
2\. QP错误异步事件上报（AEQ事件）本质也是管理层面的应答通知。  
五、不会单独发ACK的典型情况  
1\. RDMA Write/ Send 的中间分片（Middle包）：单纯攒包，不触发ACK，等计数/超时/last包统一应答；  
2\. UD QP：无ACK机制，完全依靠上层业务做确认；  
3\. 接收报文格式非法被直接丢弃，不会返回任何ACK。  
总结  
1\. 对方标了AckReq → 立刻ACK  
2\. Read/Atomic操作 → 必须即时ACK  
3\. 攒够 2^N 个包 或 超时到点 → 合并ACK  
4\. PSN乱序丢包 → 立即NAK  
5\. 消息末尾Last分片 → 强制ACK  
6\. 接收缓冲区快满 → 主动ACK通告窗口

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/eed068a8_1784378271805?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487489%26idx%3D1%26sn%3D072ef15865162f17ddc352a8eefd621d%26chksm%3Dc0436ad44bcaa7967ff732b73664e544577b7623f7bbe495a7e1fb0d518dff8cc205b2421734%26mpshare%3D1%26scene%3D1%26srcid%3D0718QJCFDNYpZMZHLMWXuNSS%26sharer_shareinfo%3De7e3ce4e49f5c2c9cbee331700ef2ec6%26sharer_shareinfo_first%3De7e3ce4e49f5c2c9cbee331700ef2ec6%23rd&s=obsidian)