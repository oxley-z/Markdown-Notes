---
author: strongerHuang
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUxNjgxMDE4OQ==&mid=2247508729&idx=1&sn=dbc1e25462d2267068008446b309b5ee&chksm=f844c14498708314ab4ae901a276222ea464ffd5d65f21d56f78dcbf161af8042b346e52885e&mpshare=1&scene=1&srcid=0715Z1RnCfOyVByeT5hF8ns9&sharer_shareinfo=0a9160e54d2cefd19c6c4dafeb122fb3&sharer_shareinfo_first=0a9160e54d2cefd19c6c4dafeb122fb3#rd
saved: 2026-07-15 12:16:12
tags:
  - 笔记同步助手
id: c219539f-1f39-4727-8489-578cea457e3c
---

公众号名称：嵌入式专栏

作者名称：strongerHuang

发布时间：2026-07-15 11:45

原文链接：[https://mp.weixin.qq.com/s/3zfLO3RWbOVrOZFkXDiU9w#rd](https://mp.weixin.qq.com/s/3zfLO3RWbOVrOZFkXDiU9w#rd)

**关注+****星标公众****号**，不错过精彩内容

作者 | strongerHuang

微信公众号 | strongerHuang

随着物联网的发展，加上MCU外设/功能越来越丰富、存储资源也越来越多，在线更新MCU固件成了很多嵌入式产品的重要功能。

今天分享几款适用于MCU的Bootloader，看看你们用过哪些？

> **MCUboot**

MCUboot顾名思义，针对MCU的boot，它是一款适用于 32 位微控制器的安全引导加载程序（软件框架）。而且，这款MCUboot开源、并遵循Apache License 2.0开源协议。

开源地址：

> https://github.com/mcu-tools/mcuboot

![[Inbox/笔记同步助手/微信公众号/2026/07/images/7a00b34a4a5866b05117667b48587244_MD5.jpg]]

MCUBoot是一个开源的、跨平台的Bootloader，支持多种ARM Cortex-M系列单片机。MCUboot 提供了安全的固件更新机制，支持加密和签名验证，适用于物联网设备。它不依赖于任何特定的操作系统和硬件，主要跟芯片的Flash结构密切相关。

MCUboot主要特点：

-   完全开源
    
-   多种升级模式
    
-   对固件安全校验
    
-   可异常恢复
    

  

官方网站：

> http://www.trustedfirmware.org/

![[Inbox/笔记同步助手/微信公众号/2026/07/images/aac1cbd7474e928a017ebe7f53fc4beb_MD5.jpg]]

官方提供了许多文档资料，我之前也给大家分享了[MCUboot的几种模式](https://mp.weixin.qq.com/s?__biz=MzUxNjgxMDE4OQ==&mid=2247502589&idx=1&sn=3e51a8238702fcde2b440255a3890eb0&scene=21#wechat_redirect)，感兴趣的同学可以点击进去看下。

> **OpenBLT**

OpenBLT 是一款适用于常见 8 位、16 位、32 位等众多单片机的Bootloader。

默认情况下，它支持RS232、CAN、USB、TCP/IP、Modbus RTU等单片机常见通信协议。并附带易于使用的 MicroBoot PC 工具，用于启动和监控固件更新。同时，还支持直接从 SD 卡执行固件更新。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/278d82116dcf29d52f2ff8c0f2caf555_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/2026/07/images/03fff13a28878956915858a0cbcf2048_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/2026/07/images/cb3fe8dc55e67a8bea92a110f3b1daaf_MD5.jpg]]

开源地址：

> https://github.com/feaser/openblt
> 
> 或
> 
> https://sourceforge.net/projects/openblt/

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8107b403000d317d023832767cc69536_MD5.jpg]]

OpenBLT特点：

-   开源免费，提供完整源代码
    
-   包括用户友好的 PC 下载实用程序
    
-   易于移植到不同的微控制器
    
-   ROM 占用空间小
    
-   高度可配置
    
-   有序且文档齐全的代码
    
-   支持从本地连接的存储（如 SD 卡）进行软件更新
    
-   可扩展以支持额外的存储器，例如串行 EEPROM 或外部Flash
    
-   支持常见的通信接口，如 RS232、CAN、TCP/IP、USB 和 Modbus RTU
    
-   可与 STM32、XMC4、XCM1、Tricore、HCS12 和其他基于 ARM Cortex 的微控制器配合使用
    

  

OpenBLT 遵循 [GNU GPL V3 开源协议](https://mp.weixin.qq.com/s?__biz=MzI4MDI4MDE5Ng==&mid=2247513717&idx=2&sn=976671e58d7b445096cf108e661419f7&scene=21#wechat_redirect)。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ebfb16dd1c339641f8ed448d3543c38e_MD5.jpg]]

官方给了一个OpenBLT的介绍视频，大家可以观看下：

![[Inbox/笔记同步助手/微信公众号/2026/07/images/976a41e94085cefd6b9b9ef0decda495_MD5.jpg]]

> 📹 此处为视频内容（vid: wxv\_3891060838714638345）（上图为封面），未能直接提取，请前往原文查看：[在公众号原文中观看](https://mp.weixin.qq.com/s?__biz=MzUxNjgxMDE4OQ==&mid=2247508729&idx=1&sn=dbc1e25462d2267068008446b309b5ee&chksm=f844c14498708314ab4ae901a276222ea464ffd5d65f21d56f78dcbf161af8042b346e52885e&mpshare=1&scene=1&srcid=0715Z1RnCfOyVByeT5hF8ns9&sharer_shareinfo=0a9160e54d2cefd19c6c4dafeb122fb3&sharer_shareinfo_first=0a9160e54d2cefd19c6c4dafeb122fb3#rd)

>   
> Tiny Bootloader

Tiny Bootloader顾名思义，它是一款微小（轻量级）的Bootloader，适合于8位（AVR）、32位单片机等资源有限的单片机，只需要2KB ROM即可。

开源地址：

> https://github.com/jaz303/tiny\_bootloader

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b32981f09e92f1cc5d112525c70ff5bf_MD5.jpg]]

Tiny Bootloader支持UART、SPI、I2C等常见的通信。源码其实挺简单，定义了一些常见的读写、页大小等。

Tiny Bootloader软件框架如下：

```
// MACRO DEFINITIONS HERE

char bootloader_requested() {
// check if bootloader has been requested (e.g. button press, GPIO low, etc)
return 0;
}

void bootloader_init() {
// get ready to enter bootloader; enable comms channel etc.
}

#include "tiny_bootloader.h"
#define TINY_BOOTLOADER_IMPL
#include "tiny_bootloader.h"

int main() {
if (bootloader_requested()) {
bootloader_init();
bootloader_run(); // defined in tiny_bootloader.h
} else {
asm("JMP 0"); // jump to main program
}
while (1);
}
```

>   
> wolfBoot

wolfBoot 是一款开源的、轻量级的安全Bootloader，它是完全独立的应用程序，适用于32位MCU操作系统或裸机项目。

开源地址：

> https://github.com/wolfSSL/wolfBoot

![[Inbox/笔记同步助手/微信公众号/2026/07/images/dbe71d613a7a48a3bb5bea2ace57048e_MD5.jpg]]

该bootloader由以下组件组成：

-   wolfCrypt，用于验证镜像的签名
    
-   一个极简的硬件抽象层，为支持的目标提供了实现，该目标负责特定 MCU 上的 IAP 闪存访问和时钟设置
    
-   核心引导加载程序
    
-   应用程序用于与引导加载程序 src/libwolfboot.c 交互的小型应用程序库
    

  

这款程序也是号称安全的Bootloader，没有动态内存分配机制，也没有链接到除 wolfCrypt 之外的任何标准 C 库。

\------------ **END**\------------

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f4d802a03204c6e02d423c7ded081c51_MD5.jpg|

![]]

RK3576如何用“零拷贝”改写车载视觉延迟规则？







](https://mp.weixin.qq.com/s?__biz=MzI4MDI4MDE5Ng==&mid=2247534592&idx=1&sn=1bd8a261c8f946f88f222bcb81976230&scene=21#wechat_redirect)

![[Inbox/笔记同步助手/微信公众号/2026/07/images/36f79280e6dd4c9f94ca067fe686a0db_MD5.jpg|

![]]

RT-Thread工业控制全栈方案，即学即用！







](https://mp.weixin.qq.com/s?__biz=MzI4MDI4MDE5Ng==&mid=2247534565&idx=1&sn=ea2a37ec43eb3fcc75f29f803127c9fe&scene=21#wechat_redirect)

![[Inbox/笔记同步助手/微信公众号/2026/07/images/cbd25fb4aeb02c0e19953ac7f669ad27_MD5.jpg|

![]]

AI不是全能的，你没有一点嵌入式基础，最牛逼的AI也不能帮你做完一个项目







](https://mp.weixin.qq.com/s?__biz=MzI4MDI4MDE5Ng==&mid=2247534579&idx=1&sn=46a74348b5ef4ecf3566add2a5fe740c&scene=21#wechat_redirect)

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/8d763838_1784088970907?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUxNjgxMDE4OQ%3D%3D%26mid%3D2247508729%26idx%3D1%26sn%3Ddbc1e25462d2267068008446b309b5ee%26chksm%3Df844c14498708314ab4ae901a276222ea464ffd5d65f21d56f78dcbf161af8042b346e52885e%26mpshare%3D1%26scene%3D1%26srcid%3D0715Z1RnCfOyVByeT5hF8ns9%26sharer_shareinfo%3D0a9160e54d2cefd19c6c4dafeb122fb3%26sharer_shareinfo_first%3D0a9160e54d2cefd19c6c4dafeb122fb3%23rd&s=obsidian)