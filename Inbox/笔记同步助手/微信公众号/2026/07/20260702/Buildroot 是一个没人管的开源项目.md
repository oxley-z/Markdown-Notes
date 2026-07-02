---
author: 11
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzA5NTM3MjIxMw==&mid=2247519698&idx=1&sn=f6cde675344e82eb259d204edf140757&chksm=91bdb613631f9f5953322c5543711deb70ccf0a88c213187de69cc53a22bd53823ea17194d58&mpshare=1&scene=1&srcid=070257YKCDwboXwWDXjCguul&sharer_shareinfo=0583e3103e1823ce8f9ebbfe6b132057&sharer_shareinfo_first=0583e3103e1823ce8f9ebbfe6b132057#rd
saved: 2026-07-02 07:56:25
tags:
  - 笔记同步助手
id: 42e4213d-7e43-4ae6-9189-13ea5619b0a0
---

公众号名称：嵌入式Linux

作者名称：11

发布时间：2026-07-02 07:39

前几天在群里讨论 buildroot，我觉得这种开源项目应该让大家多了解。

今天简单总结一下。

做嵌入式的要没用过 Buildroot，那大概率也听说过它。只是很多人用的时候稀里糊涂——配个 menuconfig，make 一下，镜像就出来了，至于它背后是谁、这个开源项目怎么坚持到现在，很少有人细想。

即便是国内的芯片sdk，也离不开 buildroot 的支持

——

**先说 Buildroot 是干啥的。**

**它是一堆堆堆堆的 Makefile 加补丁，帮你把一套嵌入式 Linux 系统自动交叉编译出来。**

你给它一个目标板子的架构，它能一口气把这几样东西都给你造好：

-   交叉编译工具链
    

-   根文件系统（rootfs）
    

-   Linux 内核镜像
    

-   bootloader
    

你要是手里已经有工具链了，它也能只帮你生成 rootfs，各干各的，不绑死。

它最优雅的是配置方式——用的是跟 Linux 内核一模一样的 **Kconfig**，就是那个 menuconfig、gconfig 的界面。做过内核的兄弟一看就懂，方向键挪来挪去，该开的打开，该关的关。

你实际操作就两行命令：先敲 make raspberrypi\_defconfig 选好板子，再敲一个 make，剩下的它自己跑。编完去 output/images 底下找东西，zImage、rootfs.ext4、还有 .dtb 都在那，烧录的时候 dd 一下或者进 U-Boot tftp 拉就行。基本系统的镜像十几二十分钟就出来，官网自己说的就是 15 到 30 分钟。

整个项目是用 **Make、shell 和 C** 写的，没有 bitbake 那套 recipe、layer 的概念。这也是它跟 Yocto 路子完全不同的地方，咱们后面再说。

——

**那它最早是怎么来的？**

这事得从 **uClibc** 说起。

2001 年末，一群给 uClibc 写代码的人，顺手搞了个东西，主要就是拿 uClibc 来做测试用的。uClibc 是 Erik Andersen 搞出来的一个给嵌入式用的 C 库，对标 glibc，但体积小得多。那会儿嵌入式板子内存紧巴巴的，glibc 塞不进去，uClibc 就是救命的。

这群人做 Buildroot 的初衷特简单：**我写了个 C 库，总得有个环境把它编译、跑起来验证吧？** 就这么，Buildroot 一开始就是 uClibc 的测试台（testbed），不是冲着"做一个构建系统"去的。

后来慢慢有人拿它干正事，2005 年 1 月 12 号，出了第一个正式的 release。

但是——

**从 2006 到 2008 年，这项目基本没人管了。**

没有固定维护者，没人合并补丁，用户想用只能去 SVN 上随便 checkout 一个版本，然后"cross fingers"（交叉手指求好运），赌它能不能用。质量一路往下掉，用的人挺郁闷。

转机在 **2009 年**。

那年的新年，Peter Korsgaard（社区里叫 jacmet）一拍脑袋，在新年决心里把自己推上去当了官方维护者。2 月 12 号，他发了 **Buildroot 2009.02**，把攒了很久没人理的 patch 一口气合了进去。

从那以后，Buildroot 定了条规矩：**每个季度发一个稳定版**，版本号就是年份加月份，比如 2009.02、2026.05。这条规矩到现在都没变。

也是从 2009 年起，一个叫 **Thomas Petazzoni** 的工程师（当时在 Free Electrons，后来的 Bootlin）开始大量往里塞代码，前前后后合了 900 多个 patch，是仅次于维护者本人在项目里贡献最多的人。还有 Yann E. MORIN、Arnout Vandecappelle、Gustavo Zacarias 这些人，都是一路扛过来的。

Petazzoni 干的一件特别实在的事，是把 Buildroot 加包的流程标准化了。

早些年想往里加一个软件，得手写一堆重复的 Makefile。他搞出了一套统一的包基础设施，后来加个包，你只要在 package 目录下写一个 .mk 文件，把版本号、源码地址、编译和安装命令定义好，最后用一行 generic-package 把前面收口，menuconfig 里就能勾出来了。这就是为什么 Buildroot 现在能塞好几千个包，加包的人不用再重复造轮子。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b04771d3fa92b57ff091c50c75de9ae9_MD5.jpg]]

更重要的是，Buildroot 从"只认 uClibc"变成了 **vendor-neutral**——glibc、uClibc-ng、musl 都支持了，不再绑死在 uClibc 这一棵树上。

我把它这几步走成一张表，看得清楚些：

<table style="border-collapse: collapse"><tbody><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">时间</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">发生了什么</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">2001 年末</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">uClibc 开发者发起 Buildroot，当 uClibc 的测试台</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">2005.01.12</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">第一个正式 release</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">2006–2008</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">基本没人维护，质量下滑，用户靠"抽签"式 checkout</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">2009.02</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Peter Korsgaard 接手，发 2009.02，开启季度稳定版</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">2009 起</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Thomas Petazzoni 等大量贡献，摆脱 uClibc 依赖，走向通用</span></span></td></tr></tbody></table>

——

**说到这，得提一下它跟 Yocto 的关系。**

很多人选型的时候在 Buildroot 和 Yocto 之间纠结。其实它俩思路完全不一样。

**Buildroot 的哲学就仨字：简单。** 它是 Make + Kconfig，一次性把镜像生成出来，**没有运行时的包管理**——你想更新？重新生成整个镜像烧进去。这对小系统、资源紧的设备反而是好事，攻击面小，行为可预期。

**Yocto 走的是另一条路。** 它用 BitBake 加一堆 metadata（recipes、layers），能搞出支持 OTA、能长期维护、能合规认证的产品级系统。代价是学起来陡，第一次全量构建动辄一两个钟头。

给你个粗暴的经验法则：

-   板子资源紧、不需要现场升级、想快点出镜像 → **Buildroot**
    

-   要做产品级、多硬件平台、要 OTA 和长期维护 → **Yocto**
    

不是谁好谁坏，是看你的活儿长啥样。

——

**还有个冷知识。**

你手边那台跑着 OpenWrt 的路由器，它最早就是从 Buildroot 分出来的。OpenWrt 早期就是 Buildroot 的一个 fork，后来自己长成了独立的项目，加了 opkg 包管理、LuCI Web 界面那一套。

所以你看，Buildroot 不声不响的，可它养出来的"孩子"满世界都是。

——

**它现在走到哪了？**

回头看一眼最近的版本——2026.05 加了 Arm Neoverse 的支持，能直接给服务器级的 Arm 平台出镜像，还加了 XFS rootfs。从当年只测个 uClibc，到现在能伺支持Arm 服务器、自带好几千个软件包，这跨度够大的。

现在 Buildroot 支持 x86、ARM、MIPS、PowerPC、RISC-V 一堆架构，X.org、Qt、GStreamer 这些都能拉进来，一个 menuconfig 配完，半小时出镜像。

——

**那它到底支持哪些板子、哪些芯片？**

这个看 Buildroot 源码里的 configs 目录就知道了，里面每一个 .defconfig 就是一个官方认的板子。我挑几个有代表性的说说。

芯片原厂这边，Buildroot 现在覆盖得挺全：

-   **Broadcom**
    
    —— 树莓派全家桶，从 Pi Zero 到 Pi 5，光 defconfig 就十几个
    

-   **NXP**
    
    —— i.MX6（wandboard、riotboard、toradex apalis）、i.MX7（warp7）
    

-   **Rockchip**
    
    —— RK3399 的 rockpro64、roc\_pc\_rk3399，RK3328 的 rock64
    

-   **ST**
    
    —— STM32MP1（stm32mp157c\_dk2）、STM32F4 那类 disco 板
    

-   **Xilinx**
    
    —— Zynq-7000 的 microzed
    

-   **Marvell**
    
    —— clearfog、macchiatobin 这些
    

-   **Synopsys**
    
    —— 做 ARC 核的 axs 评估板
    

-   **Intel/Altera**
    
    —— Cyclone V 这种 FPGA 加 ARM 的
    

还有 TI 的 BeagleBone Black，敲一行 make beaglebone\_defconfig 一把就出镜像。

列个表看得清楚：

<table style="border-collapse: collapse"><tbody><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">芯片原厂</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">代表芯片</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Buildroot 里的板子（defconfig）</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Broadcom</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">BCM283x / BCM271x</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">raspberrypi / raspberrypi4 / raspberrypi5</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">NXP</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">i.MX6 / i.MX7</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">wandboard / riotboard / warp7</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Rockchip</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">RK3399 / RK3328</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">rockpro64 / rock64 / roc_pc_rk3399</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">ST</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">STM32MP1 / STM32F4</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">stm32mp157c_dk2 / stm32f469_disco</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Xilinx</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Zynq-7000</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">zynq_microzed</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Marvell</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Armada</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">clearfog / macchiatobin</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Synopsys</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">ARC</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">snps_archs38_hsdk</span></span></td></tr><tr><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Intel/Altera</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">Cyclone V</span></span></td><td style="border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 15px; color: rgb(0, 0, 0)">socrates_cyclone5</span></span></td></tr></tbody></table>

最省事的是 **qemu**——你手头没板子，敲一行 make qemu\_arm\_vexpress\_defconfig，在虚拟机里把系统跑起来看效果，连卡都不用烧。

反正只要是 Cortex-A 这类能跑 Linux 的芯片，Buildroot 基本都有现成的 defconfig 兜底，找不到就自己照着写一个，也不费劲。

——

**几个挺有意思的地方。**

说几个我觉着挺逗的细节。

**第一，它有个"随机配置"的自动测试机。** 你敲一行 make randpackageconfig，它就把几千个包随机开关一遍，生成一个乱七八糟的配置。​Buildroot 官方有个公开的自动构建系统（autobuild.buildroot.org），每天就靠这种随机配置，在不同的架构、不同的工具链上不停编译，看哪个包炸了。​更绝的是，那些 qemu 的板子它不光编译，还真在 qemu 里把系统跑起来，一直等到屏幕上蹦出 "buildroot login:" 才算过。​你在家里随便勾包，背后有台机器替全世界的用户把坑先踩了。

**第二，它每年在 Embedded Linux Conference 上有个保留节目。** Thomas Petazzoni 会做一个叫 "What's new in Buildroot" 的演讲，把这一年加的东西过一遍。从 2014 年讲到现在，年年不落。你要是想知道 Buildroot 往哪走，翻他这个系列演讲最准。

**第三，这项目没有老板。** 不像有些开源项目背后站着一家公司，Buildroot 就是一群志愿者，每天往里提交代码，谁有空谁干。官网上写着 many developers contribute to it daily，不是客套话——你去翻它的提交记录，天天都有人 bump 一个包、修一个编译错误。Peter Korsgaard 当了十几年维护者，靠的就是这股子自发的热乎劲。

**第四，它默认塞给你的根文件系统，是 BusyBox。** 就是那个一个二进制顶几十个常用命令的瑞士军刀。你不做任何配置，编出来的系统里 ls、cp、vi 全靠 busybox 这一个程序撑着。这也是为什么 Buildroot 出来的镜像能压那么小，一颗 8M 的 flash 都装得下。

这些都不是什么大道理，可你把这几桩事串起来看，就明白 Buildroot 为什么能活二十多年还没散伙。

——

**写到最后。**

Buildroot 这项目，从 2001 年一个 C 库的测试台，到今天被无数工业设备拿去量产，靠的就是一件事：**把复杂的事做简单。**

它不像 Yocto 那样宏大，也不像某些商业工具那样花哨。它就是一堆 Makefile，老老实实帮你把系统编出来。

致敬，Peter Korsgaard，和那群从 2001 年就开始折腾它的人。

还有 Thomas Petazzoni，900 多个 patch 砸进去，不是谁都愿意干的。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/602e9211d70cdb0632116a34319857f1_MD5.jpg]]

——

想自己上手折腾的，官方文档和源码都在这：

-   官网：https://buildroot.org/
    

-   文档：https://buildroot.org/docs.html
    

-   源码仓库：https://github.com/buildroot/buildroot
    

![[Inbox/笔记同步助手/微信公众号/2026/07/images/49d1b24202f3940511ce30f20dea4958_MD5.jpg]]

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/aa1d6dc7_1782950183287?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzA5NTM3MjIxMw%3D%3D%26mid%3D2247519698%26idx%3D1%26sn%3Df6cde675344e82eb259d204edf140757%26chksm%3D91bdb613631f9f5953322c5543711deb70ccf0a88c213187de69cc53a22bd53823ea17194d58%26mpshare%3D1%26scene%3D1%26srcid%3D070257YKCDwboXwWDXjCguul%26sharer_shareinfo%3D0583e3103e1823ce8f9ebbfe6b132057%26sharer_shareinfo_first%3D0583e3103e1823ce8f9ebbfe6b132057%23rd&s=obsidian)