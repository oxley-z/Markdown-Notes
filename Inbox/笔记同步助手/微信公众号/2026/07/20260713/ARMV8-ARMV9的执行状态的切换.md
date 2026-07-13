---
author: baron
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU4NDg4MzY3OA==&mid=2247515246&idx=1&sn=8ceee794049e63618885d7c58d0c94af&chksm=fc2fad0b3e91587cba9078b32f30a9b30a8a580d9121a2a7a905b74048148302f81887498532&mpshare=1&scene=1&srcid=0713CZf6ctfYkZHr4tOpvUBp&sharer_shareinfo=99bbd3425311916eaf18cf4f615e43ee&sharer_shareinfo_first=99bbd3425311916eaf18cf4f615e43ee#rd
saved: 2026-07-13 08:28:40
tags:
  - 笔记同步助手
id: f7fc5f11-802b-4638-b342-e243ee0726db
---

公众号名称：Arm精选

作者名称：baron

发布时间：2026-07-13 08:08

#### **背景：为什么会有执行状态的切换**

在一个大系统中，我们所说这它是64位的，还是32位的，往往说的是kernel内核。事实上，在这么的一个大系统中，有着多级镜像，并非全都是64位的，也并非全都是32位的。如下一张图，便展示了某SOC系统中常用的一个执行状态

![[Inbox/笔记同步助手/微信公众号/2026/07/images/58f6527c675f3d59187ab1d81c90d508_MD5.jpg]]

#### **Interprocessing ：执行状态切换**

术语：AArch64和AArch32执行状态之间的交互称为interprocessing。

ARMV8/ARMV9的执行状态(Execution state) 有两种：aarch64和aarch32，它们的切换的方式只有两种：

-   (1) reset
    
-   (2) changing Exception level
    

它们的切换规则是：

-   aarch32到aarch64的切换，必需是触发异常，产生的low exception level 到 high exception level的切换。
    
-   aarch64到aarch32的切换，必需是异常返回，产生的high exception level 到 low exception level的切换。
    
-   在 exception level不变的情况下，产生的异常和异常返回，都不能改变excution state.
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/c66c6a339526b6fe15f0d9a72351747a_MD5.jpg]]

再简而言之，总结起来其实就是：

-   high exception level如果是aarch64，那么low exception level 可以是aarch64或aarch32
    
-   high exception level如果是aarch32，那么low exception level 只能是aarch32
    

#### **寄存器介绍**

##### **SCR\_EL3**

如果实现了EL3，那么PE复位后将直接是aarch64  
`SCR_EL3.RW` 将决定着lower exception level的执行状态

![[Inbox/笔记同步助手/微信公众号/2026/07/images/082a938477acd5839c8305a4ca6754c7_MD5.jpg]]

##### **HCR\_EL2**

如果实现了EL2，且EL2的执行状态是aarch64.  
`HCR_EL2.RW` 将决定着EL1的执行状态

![[Inbox/笔记同步助手/微信公众号/2026/07/images/a3067e0f275e43f2df7d6b8932d8f2ba_MD5.jpg]]

##### **PSTATE和SPSR\_ELx**

如果EL1是aarch64，那么SPSR\_EL1.M\[4\] 将决定着EL0的执行状态

![[Inbox/笔记同步助手/微信公众号/2026/07/images/03103225a431c5eb83277228228872dd_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d13af15b627d3c3f75b14437343e68ea_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3d80d232d90cba7d271ef8c134b69da6_MD5.jpg]]

#### 代码导读

![[Inbox/笔记同步助手/微信公众号/2026/07/images/310f16aba113c2924a61988ec1332762_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/2026/07/images/876fabdc7d81171e6cd2603e9bf2406d_MD5.jpg]]

-   如需进群可加我微信邀请进群交流：sami01\_2023
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/bf2b3c9f94e21a5b76cd60a524e65abf_MD5.jpg]]

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/4f4dd0fa_1783902518386?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU4NDg4MzY3OA%3D%3D%26mid%3D2247515246%26idx%3D1%26sn%3D8ceee794049e63618885d7c58d0c94af%26chksm%3Dfc2fad0b3e91587cba9078b32f30a9b30a8a580d9121a2a7a905b74048148302f81887498532%26mpshare%3D1%26scene%3D1%26srcid%3D0713CZf6ctfYkZHr4tOpvUBp%26sharer_shareinfo%3D99bbd3425311916eaf18cf4f615e43ee%26sharer_shareinfo_first%3D99bbd3425311916eaf18cf4f615e43ee%23rd&s=obsidian)