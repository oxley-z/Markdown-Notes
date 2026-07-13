---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487345&idx=1&sn=d92ef74e7aea5f8bf3c2144aabe8ce61&chksm=c0f189f845ca645f8949801005ba1961d2bd7d861c34a576e02cc74abdf44aff0963283c67c8&mpshare=1&scene=1&srcid=0709ZKxBznQaFYmQeTkXQKbW&sharer_shareinfo=c2d5fdfd6093bf6205874761d97d5fb8&sharer_shareinfo_first=c2d5fdfd6093bf6205874761d97d5fb8#rd
saved: 2026-07-09 21:22:24
tags:
  - 笔记同步助手
id: 8c92533a-5557-4cf1-a5a4-0b0d51f73218
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-05-17 07:07

嵌入式没有 BIOS，U-Boot 直接充当 Bootloader 全程，整个流程分为 6 个标准阶段，下面按上电顺序讲清楚。  
一、整体启动链路（上电 → Linux 跑起来）  
1\. 芯片上电 → 执行 BootROM（芯片内置固化程序）  
2\. BootROM 初始化最小硬件，加载 U-Boot SPL 到片内 RAM  
3\. U-Boot SPL 初始化 DDR，加载 U-Boot 主程序 到 DDR  
4\. U-Boot 主程序初始化外设，进入命令行/自动启动  
5\. U-Boot 加载 Linux 内核 + 设备树到内存  
6\. U-Boot 跳转到内核入口，移交控制权，Linux 启动  
二、分阶段详细步骤与作用  
1\. 阶段1：BootROM（芯片原厂固化，不可修改）  
执行位置：芯片内部 ROM  
执行时机：上电第一个执行的代码

主要作用  
\- 初始化基础时钟、串口、看门狗  
\- 从启动介质（SD/NAND/EMMC/Flash）读取 U-Boot SPL  
\- 把 SPL 加载到片内 SRAM 执行  
\- 这一步失败，设备直接黑屏无输出  
2\. 阶段2：U-Boot SPL（Secondary Program Loader 次级引导）  
执行位置：片内 SRAM（DDR 还没初始化，跑不了大程序）

核心作用  
\- 初始化 DDR 内存控制器，让外部 DDR 可用  
\- 初始化 Flash/EMMC，能正常读写存储  
\- 把 完整 U-Boot 主镜像 加载到 DDR  
\- 跳转到 U-Boot 主程序入口  
SPL 很小，一般几十 KB，只做“初始化内存”这一件事。  
3\. 阶段3：U-Boot 主程序 前置初始化（board\_init\_f）  
执行位置：DDR

作用  
\- 初始化串口、定时器、GPIO  
\- 检测内存大小、Flash 分区  
\- 初始化堆、栈，准备运行环境  
4\. 阶段4：U-Boot 主程序 完整初始化（board\_init\_r）  
进入 U-Boot 命令行环境前

作用  
\- 初始化网口、USB、SPI、I2C  
\- 读取环境变量（bootcmd、bootargs、bootdelay）  
\- 配置自动启动倒计时（默认3秒）  
此时可以：  
\- 按任意键进入 U-Boot 命令行  
\- 执行命令：

printenv 、 setenv 、 tftp 、 mmc 、 bootm  
5\. 阶段5：自动启动（bootcmd 执行）  
  
默认倒计时结束后自动执行  
典型流程：  
1\. 从 EMMC/Flash 读取 zImage 内核到内存  
2\. 读取设备树 dtb 到内存  
3\. 配置启动参数 bootargs （传给 Linux 内核）  
4\. 执行 bootm 命令启动内核  
6\. 阶段6：移交控制权给 Linux 内核（bootm）  
U-Boot 最后一步：  
\- 关闭 MMU、关闭 Cache（部分架构）  
\- 设置寄存器：内核入口地址、设备树地址、机器ID  
\- 直接跳转执行 Linux 内核，U-Boot 彻底退出，不再运行  
三、U-Boot 核心作用（总结）  
1\. 硬件初始化  
初始化 DDR、Flash、网口、串口，为 Linux 准备硬件环境。  
2\. 加载 Linux 内核与设备树  
从 Flash/SD/网络把内核读到内存，是嵌入式启动必经环节。  
3\. 传递启动参数 bootargs  
告诉 Linux：根文件系统位置、控制台、波特率、分区信息。  
4\. 设备树传递  
把 dtb 设备树文件传给内核，实现驱动与硬件解耦。  
5\. 调试与升级  
支持 tftp 网络烧录、mmc 读写、分区擦除、系统升级。  
6\. 应急启动  
内核损坏时，可进入 U-Boot 修复，不用拆机。  
四、极简一句话流程  
BootROM → 加载 SPL → SPL 初始化 DDR → 加载 U-Boot 主程序 → U-Boot 加载内核与设备树 → 跳转到 Linux

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/b948483a_1783603344107?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487345%26idx%3D1%26sn%3Dd92ef74e7aea5f8bf3c2144aabe8ce61%26chksm%3Dc0f189f845ca645f8949801005ba1961d2bd7d861c34a576e02cc74abdf44aff0963283c67c8%26mpshare%3D1%26scene%3D1%26srcid%3D0709ZKxBznQaFYmQeTkXQKbW%26sharer_shareinfo%3Dc2d5fdfd6093bf6205874761d97d5fb8%26sharer_shareinfo_first%3Dc2d5fdfd6093bf6205874761d97d5fb8%23rd&s=obsidian)