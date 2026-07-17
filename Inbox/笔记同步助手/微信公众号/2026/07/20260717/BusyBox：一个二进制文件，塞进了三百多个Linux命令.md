---
author: 11
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzA5NTM3MjIxMw==&mid=2247519865&idx=1&sn=e0e89f8f9fbbada6f93a38e34522eacd&chksm=91d04136ff132eed9226491027924c482ad881767f2b78af0df40c6b3558267b67c1b8d148f8&mpshare=1&scene=1&srcid=0717jNhRVW1a3JEgk1IGgKK4&sharer_shareinfo=f1d759931642bcc6c8a30585a3e48665&sharer_shareinfo_first=f1d759931642bcc6c8a30585a3e48665#rd
saved: 2026-07-17 07:53:28
tags:
  - 笔记同步助手
id: 38423bb4-79dc-4f7e-8a96-0f117935ca6c
---

公众号名称：嵌入式Linux

作者名称：11

发布时间：2026-07-17 07:38

大家好，我是写代码的篮球球痴。

搞嵌入式的。。。。。。。，没人离得开 BusyBox。你拿 Buildroot 编一个最精简的系统出来，默认塞给你的 shell 环境就是BusyBox。你家里那台跑 OpenWrt 的路由器，根文件系统也是BusyBox。甚至 docker 里最常用的那个 5MB 的 Alpine Linux 基础镜像，/bin/sh 指向的还是 busybox。

但很多人天天敲 ls、cd、ifconfig，从来没想过这些命令背后站着的其实是同一个程序。

一个叫 BusyBox 的东西。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/02621b006b85026010fa929b96af7389_MD5.jpg]]

OpenWrt 路由器启动画面，BusyBox v1.28.4 自带的 ash shell 在跑——这就是绝大多数人家里那台路由器的「灵魂。。。。。」。

——

BusyBox 把三百多个最常用的 Linux 命令，塞进了一个可执行文件里。

ls、cp、cat、echo、grep、find、mount、ifconfig——你能在 shell 里敲的常见命令，它基本都带了。普通的 Linux 发行版，每个命令都是一个独立的可执行程序，散落在 /bin、/usr/bin 底下。

BusyBox 不一样，就一个二进制文件 busybox，你敲的 ls、cp 那些，全是指向 busybox 的软链接。

程序启动的时候看自己被叫成啥名，就干啥活——这套机制叫 applet（小工具）。所以你在嵌入式板子上看到的 /bin/sh，ls -alh 去查一下，，，，，八成是个指向 busybox 的软链接。

BusyBox 有多小？

只留 20 个最基础的命令，能压到 350KB；全功能默认编译大概 1.8MB。对比一下，完整的 GNU coreutils + bash 吃 100MB 以上，每个命令独立占地方，动辄要好几 MB 内存。一颗 8M 的 nor flash，用 BusyBox 能把整套命令行环境塞进去，还能剩地方给你自己的程序。

这就是它在嵌入式设备里到处都是BusyBox的根根根根原因。。。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/7c95fee0ef51ef11d9fae1e93e78775f_MD5.jpg]]

busybox --help 输出，最后那段 \`\![[Inbox/笔记同步助手/微信公众号/2026/07/images/2ec56c26acdbfde52be6a04693ca2c27_MD5.jpg|\` 开头到结尾就是它塞进来的全部 applet 名字。

——

BusyBox是 1996 年写的。

当时 Bruce Perens 在给 Debian GNU/Linux 写安装软盘。那时候装系统得靠一张软盘引导，软盘容量 1.44MB，塞一个能用的命令行环境都费劲。

他的思路很简单

与其把几十个命令程序一个个塞进软盘，不如写一个大程序，把常用命令全收进去，需要用哪个就调哪个。写完之后这软盘既能当安装盘，也能当急救盘——系统崩了起不来的时候，拿这张盘引导，挂载上硬盘，把系统修好。

写完了，Bruce Perens 就撒手了。

后面维护者换了好几轮。

1996 到 1998 是 Enrique Zanardi 接着维护，主要伺候 Debian 启动软盘。

1998 到 1999，Dave Cinege 把它做成模块化，开始往嵌入式方向带——他当时在搞 Linux Router Project，路由器上资源紧，BusyBox 正好用得上。

最关键的是 Erik Andersen。他从 1999 年一直维护到 2006 年 3 月，BusyBox 今天能塞进三百多个命令，大半是他那几年攒出来的。

这人也是 uClibc 的作者——一个人撑起了嵌入式 Linux 两套地基。

2006 年前后，Rob Landley 接手，后来因为许可证的事跟社区闹掰了，自己另写了一个叫 Toybox 的东西。2006 年 10 月起，Denys Vlasenko 接手，到现在还在维护。

![]]

——

你做嵌入式开发，大概率已经用过 BusyBox，只是没注意。

OpenWrt 路由器的根文件系统默认就是BusyBox。Buildroot 编出来的系统默认也是BusyBox。Alpine Linux 的 /bin/sh 指向的还是BusyBox。这不是巧合——嵌入式设备 flash 小、内存紧，用完整 coreutils + bash 光命令就吃掉大半空间，轮不到你自己的程序。BusyBox 一个二进制顶几百个命令，省下来的空间才是真金白银。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3ca5356bc72b5a06dfafa5338abbf997_MD5.jpg]]

一块 Cirrus Logic EP9302 的 ARM9 开发板，左下角那个黑色接口是 Console 串口——上电之后跑的就是 BusyBox。

而且BusyBox不只是一堆命令的集合。

BusyBox自带 init——就是那个 PID 1 的进程，系统起来第一个跑的就是BusyBox。

自带一个叫 ash 的 shell，你板子上 /bin/sh 指向 busybox 的时候，真正在跑的 shell 就是 ash。它还藏着好几个常驻服务：httpd 能跑网页服务，telnetd 能远程登录，udhcpd 能分配 IP，ntpd 能对时，ftpd 能传文件——全在一个二进制里。

你家里那台路由器的管理页面，背后很可能就是 BusyBox 的 httpd 在撑着。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ac2fcfceeffefca279450131b348726a_MD5.jpg]]

TTYD 跑出来的 Web 终端，正在用 busybox 自带的 iperf3 测网速。背后那个 httpd 撑起来的 Web UI，也是 BusyBox 。

要是嫌命令太多，想自己裁剪，它用的是跟 Linux 内核一样的 menuconfig：

```
make menuconfig
```

方向键挪来挪去，哪些命令要、哪些不要，勾完编出来。跟你配内核一个体验。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8a8ef2351f1c6e17861884e8d9a031a9_MD5.jpg]]

BusyBox 1.27.2 的 make menuconfig 主界面，方向键往下挪能勾出几十个分类（Archival、Coreutils、Editors、Networking 都在里面）。

还有个东西叫"standalone shell"模式。

你不用建那堆软链接，直接进 busybox 自己的 shell，就能调所有内建命令。做 initramfs 早期引导的时候经常这么玩——系统还没挂载好，靠这个模式就能干活。

另外它能跑在没有 MMU 的芯片上。正常 Linux 进程靠 MMU 做内存隔离，但像 ARM7、ColdFire 这类没 MMU 的便宜芯片，靠 uClinux 也能跑 Linux，BusyBox 就是这套系统里默认的命令行。板子越便宜，越离不开它。

——

BusyBox 还打过一场官司，在美国开源法律史上是有位置的。

2007 年 9 月，软件自由法律中心（SFLC）代表 BusyBox 的两位开发者 Erik Andersen 和 Rob Landley，把一家叫 Monsoon Multimedia 的公司告了。

Monsoon 在一款叫 HAVA 的电视转播设备里用了 BusyBox，没按 GPLv2 的要求公开源代码。

这是美国历史上第一桩因为违反 GPL 而打的官司。GPL 从 1991 年出来，大家都说它"强 copyleft"，但真到法庭上管不管用，从来没人试过。

案子很快和解了——Monsoon 乖乖把源码挂上网、交了笔钱、还专门设了个开源合规官。​SFLC 接着又告了好几家：Xterasys、High-Gain Antennas、Verizon（Actiontec 路由器）、Bell Microproducts、Super Micro。​2009 年 12 月一口气告了 Best Buy、JVC、三星等 14 家，其中 Westinghouse 被判三倍赔偿 9 万美元。

Bruce Perens 没参与这些官司，后来还专门发声明，说对当时维护者的一些做法不太满意。

但这场官司之后，所有用开源软件做产品的公司都明白了一件事——GPL 不是废纸，你不公开源码，真有人把你告上法庭。

——

BusyBox 官网的 tagline 就一句话：The Swiss Army Knife of Embedded Linux——嵌入式 Linux 的瑞士军刀。

1996 年一张软盘上的安装工具，到今天全世界的路由器、机顶盒、工控板里都有它。

不做内核，不做框架，不上热搜。就是一个安静的二进制，你敲命令的时候BusyBox能响应，不敲的时候它也不叽叽歪歪。。。。。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/51fddefc_1784246006851?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzA5NTM3MjIxMw%3D%3D%26mid%3D2247519865%26idx%3D1%26sn%3De0e89f8f9fbbada6f93a38e34522eacd%26chksm%3D91d04136ff132eed9226491027924c482ad881767f2b78af0df40c6b3558267b67c1b8d148f8%26mpshare%3D1%26scene%3D1%26srcid%3D0717jNhRVW1a3JEgk1IGgKK4%26sharer_shareinfo%3Df1d759931642bcc6c8a30585a3e48665%26sharer_shareinfo_first%3Df1d759931642bcc6c8a30585a3e48665%23rd&s=obsidian)