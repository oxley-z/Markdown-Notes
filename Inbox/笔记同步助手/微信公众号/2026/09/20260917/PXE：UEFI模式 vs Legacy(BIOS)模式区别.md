---
author: hb的小羊
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg2NDkwODQzNw==&mid=2247484154&idx=1&sn=5a91c4b9eda45ef972a318b347ebd626&chksm=cf0193a31a43c2bd7266389409b8ca6987d773daa6970830e0395f411607c69fd504a77da98f&mpshare=1&scene=1&srcid=0917AcoySzzQU82ClNR2BIq7&sharer_shareinfo=ecd91e8c206c5e03f05ed5e648f25747&sharer_shareinfo_first=ecd91e8c206c5e03f05ed5e648f25747#rd
saved: 2026-09-17 11:48:00
tags:
  - 笔记同步助手
id: b096920d-fdce-4d95-996a-c5445a7f0c67
---

公众号名称：hb的小羊

作者名称：hb的小羊

发布时间：2026-09-17 06:08

PXE是预启动执行环境，网卡通过网络获取引导程序，实现网络装机。

-   Legacy(BIOS)：传统MBR BIOS启动，基于BIOS中断服务，网卡ROM固件（Option ROM）；
    
-   UEFI：新一代固件，使用UEFI驱动，支持GPT分区，不再依赖传统Option‑ROM。
    

# 核心差异对比表

| 对比项 | Legacy BIOS PXE | UEFI PXE |
| --- | --- | --- |
| 底层固件 | 传统BIOS，MBR分区表 | UEFI固件，GPT分区表 |
| 网卡固件 | Option‑ROM（传统网卡ROM） | UEFI‑Driver（EFI驱动） |
| PXE协议栈 | PXE 2.1，基于TFTP + BIOS中断 | UEFI PXE，UEFI网络栈，TFTP，支持IPv4/IPv6 |
| 磁盘分区 | 只能识别MBR分区，最大磁盘2TB | GPT分区，支持>2TB磁盘，EFI系统分区ESP |
| 引导文件 | pxelinux.0（传统PXE引导器） | （EFI引导文件） |
| DHCP配置 | DHCP 67端口，option 67指向pxelinux.0 | DHCP option 67指向EFI的efi文件；部分环境需要option 60 |
| 安全 | 无Secure‑Boot | 支持Secure‑Boot；开启后，EFI引导镜像需要签名，未签名无法启动 |
| 网卡支持 | 老网卡Option‑ROM；部分网卡不支持UEFI | 需要网卡提供UEFI驱动；BF‑DPU/ConnectX网卡同时支持两种模式 |
| 启动流程 | BIOS → Option‑ROM PXE → TFTP下载pxelinux.0 → 加载内核 | UEFI固件 → UEFI网络驱动 → TFTP下载bootx64.efi → EFI引导器 |

重要点：

# 1、引导文件不一样（最容易踩坑）

# 1\. Legacy BIOS PXE

-   下载： pxelinux.0 （Linux PXE引导程序）
    
-   配置文件： pxelinux.cfg/ 目录下配置
    
-   只支持MBR磁盘。
    

# 2\. UEFI PXE

-   下载： bootx64.efi （64位EFI引导程序）
    
-   配置文件： grub.cfg ，使用EFI grub
    

注意：不能把pxelinux.0用于UEFI PXE，会直接启动失败。

# 2、DHCP服务器配置差异

# Legacy BIOS PXE

option 66: tftp服务器IP

option 67: pxelinux.0

# UEFI PXE

option 66: tftp服务器IP

option 67: /EFI/boot/bootx64.efi

部分UEFI环境需要设置 option 60=PXEClient 。

同一个DHCP服务器可以同时服务Legacy和UEFI客户端，通过DHCP的class识别区分不同客户端返回不同的bootfile。

# 3、Secure‑Boot（UEFI独有）

-   UEFI开启Secure‑Boot：EFI引导镜像必须被微软/厂商签名，自制未签名bootx64.efi会拒绝启动。
    
-   Legacy BIOS完全没有Secure‑Boot机制。
    

# 4、磁盘分区限制

-   Legacy BIOS：只能MBR，单分区最大2TB。
    
-   UEFI：GPT，支持大于2TB磁盘，必须存在ESP(EFI系统分区)。
    

# 5、BF‑DPU（BlueField）场景重点

BF‑3/2的OOB‑Net口、NIC网口都支持PXE，同时支持Legacy BIOS PXE 和 UEFI PXE。

-   服务器BIOS设置为Legacy，网卡走Legacy PXE；
    
-   BIOS设置为UEFI，网卡走UEFI PXE；
    

坑点：

1.  服务器BIOS启动模式改变，PXE引导文件必须同步更换，否则PXE循环报错。
    
2.  UEFI Secure‑Boot开启，DPU做PXE引导时，EFI镜像必须签名。
    
3.  DPU的OOB‑Net（带外网口）也支持PXE，可实现DPU自身的网络装机。
    

# 6、一些故障：

1.  Legacy PXE：PXE启动，获取IP，下载pxelinux.0失败/报错；
    
2.  UEFI PXE：获取IP成功，但是找不到 bootx64.efi ，直接退出PXE；
    
3.  混用：把pxelinux.0给UEFI客户端，PXE会报 No boot device available ；
    
4.  UEFI Secure‑Boot开启，bootx64.efi未签名，PXE启动直接失败。
    

# 7、简单记忆

-   Legacy PXE：MBR + pxelinux.0，老BIOS，无Secure‑Boot
    
-   UEFI PXE：GPT + bootx64.efi，新固件，支持Secure‑Boot
    

生产环境建议：服务器统一使用UEFI PXE，Legacy仅做兼容老旧硬件。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/0c337298_1789616879248?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg2NDkwODQzNw%3D%3D%26mid%3D2247484154%26idx%3D1%26sn%3D5a91c4b9eda45ef972a318b347ebd626%26chksm%3Dcf0193a31a43c2bd7266389409b8ca6987d773daa6970830e0395f411607c69fd504a77da98f%26mpshare%3D1%26scene%3D1%26srcid%3D0917AcoySzzQU82ClNR2BIq7%26sharer_shareinfo%3Decd91e8c206c5e03f05ed5e648f25747%26sharer_shareinfo_first%3Decd91e8c206c5e03f05ed5e648f25747%23rd&s=obsidian)