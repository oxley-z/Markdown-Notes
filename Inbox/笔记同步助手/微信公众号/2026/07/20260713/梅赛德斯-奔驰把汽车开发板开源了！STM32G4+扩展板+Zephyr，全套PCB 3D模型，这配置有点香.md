---
author: 晓宇
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI4NTQ4NTA3NA==&mid=2247518228&idx=1&sn=e18323177087a6b4e2588897ac48b69b&chksm=ea65e0c6dd4163977a4e465c6e0a32e69f072e34be3dd6a578e42c2c52504c118b4b974a37a7&mpshare=1&scene=1&srcid=0713dzsakLHUjppZi8kPEP1I&sharer_shareinfo=16620813156ba2e542bd19470c56ca52&sharer_shareinfo_first=16620813156ba2e542bd19470c56ca52#rd
saved: 2026-07-13 13:16:07
tags:
  - 笔记同步助手
id: 7e89ce20-bad7-46a5-afc9-3cab1cfcc771
---

公众号名称：芯片之家

作者名称：晓宇

发布时间：2026-07-13 12:15

很多人认为，汽车厂商对于核心研发平台都会严格保密。但最近，梅赛德斯-奔驰却反其道而行之，正式开源了一套汽车快速开发平台（ARDEP）。

他们正式发布了一套名为 **ARDEP（Automotive Rapid Development Platform，汽车快速开发平台）** 的开源硬件平台，**不仅开放了PCB、原理图、固件源码**，连开发文档都全部放到了 GitHub 上，并采用 **Apache 2.0** 开源协议。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/fe6bf20b524f4bd31b1faf5f476ee4ec_MD5.jpg]]

对于汽车电子、嵌入式开发工程师来说，这算是一份非常有参考价值的开源项目。

---

## 一块专门为汽车电子打造的开发板

ARDEP V2 主板采用的是 **ST STM32G474VE** MCU。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d0535ed307a2435b672a9ab4d3b584d0_MD5.jpg]]

整个PCB带很漂亮的3D模型，大家直接导出来化为己有也是非常香啊。

这颗芯片基于 **Arm Cortex-M4F** 内核，主频可达 **170MHz**，除了浮点运算单元之外，还集成了 **CORDIC** 和 **FMAC** 数学硬件加速器，非常适合电机控制、实时控制等应用。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ab9173a031f50b69fc4c35420a3725a8_MD5.jpg]]

除此之外，开发板还集成了汽车领域最常用的通信接口：

-   双路 CAN-FD
    
-   一路 LIN
    
-   USB Type-C
    
-   板载调试器
    
-   丰富的 GPIO、UART、SPI、I²C、ADC、DAC 接口
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/36c1ed149f98d0feb6ff85ec2d409a79_MD5.jpg]]

板载供电也充分考虑了汽车环境，可直接支持 **5V～48V** 输入，并提供反接、过压、ESD 等保护。

## 配套了一块工业级 PowerIO 扩展板

相比普通 MCU 开发板，更有意思的是它还提供了一块 **PowerIO Shield**。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/fa9e4bc6256b85368a8db72133012aac_MD5.jpg]]

扩展板的PCB同样漂亮，这块扩展板提供：

-   6 路高边输出，每路最高 3A
    
-   6 路 48V 数字输入
    
-   支持过流、过温检测
    
-   每一路故障均可通过 LED 和 I²C 上报
    

对于继电器、电磁阀、灯光、风扇、电机等汽车负载，都可以直接进行控制，非常适合作为 ECU 原型验证平台。

## 软件也完全开源

ARDEP 的软件部分同样十分完整。

整个平台基于 **Zephyr RTOS** 开发，已经集成了：

-   UDS（ISO 14229）汽车诊断协议
    
-   CAN 通信
    
-   LIN 通信
    
-   ADC、DAC、GPIO 驱动
    
-   CAN 转 USB Bridge
    
-   USB/CAN 固件升级（DFU）
    

官方还提供了大量示例程序，可以直接作为汽车项目开发参考。

https://github.com/mercedes-benz/ardep/

## 连硬件资料都全部公开

最值得关注的是，这并不是简单开放几个 Demo。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ad88a5e80dc92fb083a04d9deff5e7d6_MD5.jpg]]

该开源硬件项目托管在梅赛德斯-奔驰账户下的GitHub 代码库中。代码库包含 KiCad 和 Altium 格式的硬件设计文件、PDF 原理图、固件源代码、库文件、示例程序以及全面的开发文档。所有作品均以“Copyright: Mercedes-Benz AG”为版权所有，并根据 Apache 2.0 许可证发布。

## 已经用于奔驰 FlexCAR 项目

> ![[Inbox/笔记同步助手/微信公众号/2026/07/images/a3adde06bf4b3a477b00c86510b9ca79_MD5.jpg]]
> 
> 梅赛德斯-奔驰的 FlexCAR 滚动底盘概念车
> 
> 据了解，ARDEP 是 **梅赛德斯-奔驰** 与 **Frickly Systems GmbH** 共同开发的项目。
> 
> 其中奔驰负责 GitHub 开源维护，并已经将这套平台应用在 **FlexCAR** 开源研究车辆（Rolling Chassis）项目中，Frickly Systems 则负责硬件和软件工程开发。
> 
> 目前官方尚未正式发售开发板，后续预计将通过众筹方式上市，因此价格暂未公布。
> 
> ![[Inbox/笔记同步助手/微信公众号/2026/07/images/8eda1c6872ad77f25663b304978c8c8b_MD5.jpg]]
> 
> ARDEP平台安装在FlexCAR滚动底盘上
> 
> 过去，汽车 ECU 开发平台大多价格昂贵，而且资料封闭；如今奔驰直接把一整套汽车开发平台开源，无论是对于汽车电子工程师，还是学习 CAN、LIN、UDS、Zephyr 等技术的开发者，**都具有很高的参考价值**。
> 
> 如果你最近正准备学习汽车电子、车载通信或 ECU 开发，不妨研究一下这个项目，也许能从中获得不少设计思路。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/4043c6ae_1783919765484?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI4NTQ4NTA3NA%3D%3D%26mid%3D2247518228%26idx%3D1%26sn%3De18323177087a6b4e2588897ac48b69b%26chksm%3Dea65e0c6dd4163977a4e465c6e0a32e69f072e34be3dd6a578e42c2c52504c118b4b974a37a7%26mpshare%3D1%26scene%3D1%26srcid%3D0713dzsakLHUjppZi8kPEP1I%26sharer_shareinfo%3D16620813156ba2e542bd19470c56ca52%26sharer_shareinfo_first%3D16620813156ba2e542bd19470c56ca52%23rd&s=obsidian)