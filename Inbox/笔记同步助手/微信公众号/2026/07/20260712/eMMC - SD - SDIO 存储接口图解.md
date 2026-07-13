---
author: Leanu &amp; JOJO
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzcwOTI4Nzc4Ng==&mid=2247484151&idx=1&sn=2591e9fcbfb65fdcb03783723956ea6a&chksm=f476ed1ef6a1aae3223f1ff66365d093b76736ebda5e5e48208baf1bc6291957c837a4d70ee8&mpshare=1&scene=1&srcid=0712agMbCD8b3khIOHxKdFlM&sharer_shareinfo=1cefe6a2a9ab1b399dd3f9de77c41329&sharer_shareinfo_first=1cefe6a2a9ab1b399dd3f9de77c41329#rd
saved: 2026-07-12 23:46:41
tags:
  - 笔记同步助手
id: 8988d9f6-ff8e-4afd-aebd-701ce1cc59c5
---

公众号名称：一口吃懂硬件

作者名称：Leanu & JOJO

发布时间：2026-06-30 08:00

![[Inbox/笔记同步助手/微信公众号/2026/07/images/e955f0825d4e90c90fbcc1101cca5baa_MD5.jpg]]

**嵌入式接口技术专题**

**eMMC / SD / SDIO** **存储接口图解**

_CLK/CMD/DAT · 1/4/8-bit · HS200/HS400 · Boot_ _分区 · SDIO 外设_

嵌入式设备的固件存在哪、用户数据放哪、Wi-Fi 模块又怎么接？答案往往落在同一族接口上：SD、eMMC、SDIO。它们同根同源，都基于MMC/SD 的“命令 + 数据”模型，却各有分工：SD 是可插拔的卡，eMMC是焊在板上的存储芯片，SDIO 则把这套接口借去接外设。这一期，我们把它们的信号、eMMC 的分区、速率模式、SDIO 与读写事务一次讲清。

**01** **同根同源：SD / eMMC / SDIO 一家人**

SD、eMMC、SDIO 都脱胎于MMC/SD 协议族，共用同一套“命令(CMD) + 数据(DAT)”的总线模型，因此理解了一个，另两个就触类旁通。区别在用途：

![[Inbox/笔记同步助手/微信公众号/2026/07/images/c5b8ad5b06e3e9a14f68120956e60797_MD5.jpg]]

_图 1　同根同源：SD 卡 · eMMC · SDIO 都源自 MMC/SD 协议_

SD 卡是可插拔的存储卡，常见于相机、记录仪；eMMC 是把控制器和 NAND 封在一颗 BGA 里、直接焊在板上的存储芯片，是大多数嵌入式系统的启动与主存储介质；SDIO 则复用这套接口去接 Wi-Fi、蓝牙等功能模块。

**02** **SD** **接口信号：CLK / CMD / DAT**

先看最基础的信号线，这也是 eMMC、SDIO 共用的骨架：

![[Inbox/笔记同步助手/微信公众号/2026/07/images/5ba7aa142bd616afcf0e0b84a325d3e4_MD5.jpg]]

_图 2SD 接口信号：CLK · CMD · DAT0–3_

CLK 是主机提供的时钟，CMD 是双向的命令/响应线，DAT 则是数据线。1-bit 模式只用DAT0；4-bit 模式同时用 DAT0–3，一个时钟拍传 4 位、带宽翻四倍。eMMC 进一步把数据线扩展到 8-bit（DAT0–7），带宽更高。

**03** **eMMC****：把存储与分区焊在板上**

eMMC 是嵌入式产品里出镜率极高的“板载硬盘”，它的内部其实是一套完整的存储子系统：

![[Inbox/笔记同步助手/微信公众号/2026/07/images/c992e92f22068de9f63d3b13fb627c7f_MD5.jpg]]

_图 3eMMC：把控制器与 NAND 封进一颗BGA_

一颗 eMMC = Flash 控制器（负责 FTL 映射、ECC 纠错、坏块管理）\+ NAND 存储阵列，对外只露出标准的 eMMC 总线，复杂的闪存管理都被封装屏蔽。逻辑上它还划分了 Boot（可直接从此启动）、RPMB（防重放的安全存储，常放密钥）、User（最大的用户数据区）等分区，非常契合嵌入式系统“启动 + 存储 + 安全”的需求。

**04** **总线速率模式：从 DS 到 HS400**

同样的信号线，靠不同的速率模式拉开了带宽差距：

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f5890eac218217ca09e76db0252cd6fa_MD5.jpg]]

_图 4　总线速率模式：从 DS 到 HS400_

提速主要靠两条路：提高时钟频率，以及在时钟的上升和下降两个边沿都传数据（DDR）。于是从早期的 DS、HS，一路演进到 HS200（单边沿高频）、HS400（8-bit + DDR，约 400 MB/s）。SD 卡侧也有 SDR50/SDR104/UHS 等对应的高速模式。选型时要让主机控制器、eMMC/卡、PCB 走线三者都支持目标模式，才能真正跑到标称速率。

**05** **SDIO****：把卡槽变成外设接口**

SDIO 是这套接口最巧妙的延伸——它对面接的不是存储，而是功能模块：

![[Inbox/笔记同步助手/微信公众号/2026/07/images/a7009a639633d2a407e12880b53b33e3_MD5.jpg]]

_图 5SDIO：用同一套接口去接 Wi-Fi / 蓝牙等外设_

SDIO 复用 SD 的引脚与协议，让主机像访问存储卡一样去访问 Wi-Fi、蓝牙、NFC 等外设模块。很多嵌入式 Wi-Fi 模组正是走 SDIO 接入主控的——一个原本用来插卡的接口，就这样变成了一条通用的高速外设总线。

**06** **一次读写事务：命令 → 响应 → 数据**

最后看协议层，一次读写在总线上是怎么走完的：

![[Inbox/笔记同步助手/微信公众号/2026/07/images/4ece8e9f983bd51691f6abb7e0851b00_MD5.jpg]]

_图 6　一次读写事务：命令 → 响应 → 数据_

主机先在 CMD 线上发出读/写命令（含地址，带 CRC），卡在CMD 线上回一个响应表示接受或报告状态，随后数据在 DAT 线上成块传输（同样带 CRC 校验）：写入时主机→卡，读取时卡→主机。时钟始终由主机提供，CRC 则保证了存储读写的可靠性。

**07** **对比与设计要点**

把三者放在一起，再落到设计要点：

| **维度** | **SD** **卡** | **eMMC** | **SDIO** |
| --- | --- | --- | --- |
| **形态** | 可插拔卡 | 板载 BGA 芯片 | 外设模块 |
| **数据位宽** | 1 / 4-bit | 1 / 4 / 8-bit | 1 / 4-bit |
| **典型用途** | 相机/记录仪存储 | 系统启动+主存储 | Wi-Fi/蓝牙接入 |
| **代表速率** | UHS SDR104 | HS200 / HS400 | 随模块而定 |

板级落地还要注意：

| **设计要点** | **说明** |
| --- | --- |
| **信号完整性** | CLK/DAT 是高速信号，走线短、等长、少过孔；HS200/HS400 对走线与阻抗要求高。 |
| **上拉与电平** | CMD/DAT 按规范加上拉；确认主机与器件 I/O 电平一致（多为 1.8V/3.3V）。 |
| **供电与去耦** | eMMC 写入瞬间电流大，供电要足、去耦电容就近放，避免掉电导致数据损坏。 |
| **分区规划** | 合理规划 Boot/RPMB/User；安全数据放 RPMB，启动镜像放 Boot 分区。 |
| **速率匹配** | 三方（控制器/器件/走线）都支持目标模式才跑得到标称速率，否则自动降速。 |
| **掉电保护** | 存储对掉电敏感，必要时做掉电检测与文件系统层面的保护。 |

**08** **结语**

一套“命令 + 数据”的总线，三种形态——SD 给你可插拔的灵活，eMMC 给你板载的可靠启动与主存储，SDIO 把卡槽变成外设入口。对做嵌入式产品的工程师而言，读懂 CLK/CMD/DAT 的分工、读懂 eMMC 的分区与速率模式、读懂 SDIO 的复用，就能在存储选型、启动设计与外设接入中做出更稳的决定。

系列说明：《嵌入式接口技术专题》已覆盖 USB、韦根、PCIe、CAN/CAN FD、RS-485/Modbus、UART、SPI/I²C、LIN、车载以太网、MIPI、I²S、I3C、JTAG/SWD、eMMC/SD/SDIO 等接口。后续可按需补充更多专题，欢迎留言点题。

**——** **嵌入式接口技术专题 ——**

觉得这篇图解有用？点赞、在看、转发给同样在和存储、启动打交道的工程师朋友。

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/96868d94_1783871199344?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzcwOTI4Nzc4Ng%3D%3D%26mid%3D2247484151%26idx%3D1%26sn%3D2591e9fcbfb65fdcb03783723956ea6a%26chksm%3Df476ed1ef6a1aae3223f1ff66365d093b76736ebda5e5e48208baf1bc6291957c837a4d70ee8%26mpshare%3D1%26scene%3D1%26srcid%3D0712agMbCD8b3khIOHxKdFlM%26sharer_shareinfo%3D1cefe6a2a9ab1b399dd3f9de77c41329%26sharer_shareinfo_first%3D1cefe6a2a9ab1b399dd3f9de77c41329%23rd&s=obsidian)