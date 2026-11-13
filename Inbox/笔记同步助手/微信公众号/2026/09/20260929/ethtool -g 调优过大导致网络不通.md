---
author: hb的小羊
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg2NDkwODQzNw==&mid=2247484418&idx=1&sn=d5c3368c9380f2d1216dcecdd8222f4a&chksm=cfaddf76cdaebe91b90996ee8e2a223f6e8f548d1b17363fdc5dcb1da54939b8261a20f55075&mpshare=1&scene=1&srcid=0929eHM0srAMXPgjj936SmGV&sharer_shareinfo=2daab21059be786bf648d2a1c90fa02e&sharer_shareinfo_first=2daab21059be786bf648d2a1c90fa02e#rd
saved: 2026-09-29 08:29:38
tags:
  - 笔记同步助手
id: 763905d2-5e7b-49e6-8e0c-da010f40a6a3
---

公众号名称：hb的小羊

作者名称：hb的小羊

发布时间：2026-09-29 06:29

# ethtool -g 查看/设置 RX/TX ring（网卡环形缓冲区，接收/发送队列）

问题现象：把 rx‑ring / tx‑ring 设置得太大，网卡直接异常，ping不通、业务断流，但网卡没有down，dmesg可能出现网卡报错。

# 现象

1.  执行 ethtool -G eth0 rx 8192 tx 8192 这类超大值
    
2.  网卡仍然UP，但是完全收不到报文，ping不通，丢包100%
    
3.  dmesg 出现类似：
    

ixgbe: eth0: Rx ring size 8192 exceeds maximum supported size

ixgbe: eth0: Could not set ring parameters

部分网卡不会直接报错误，只是静默不工作；部分驱动会直接把队列置为异常状态。

重点：每个网卡硬件有最大ring size上限，不是想设多大就多大。

不同网卡型号上限不一样：

-   老10G网卡：rx最大一般 4096
    
-   25G/40G/100G网卡：常见最大 8192 / 16384
    
-   超过硬件上限，设置会失败，网卡队列异常，网络直接挂掉。
    

# 紧急恢复操作

## 1、先看网卡支持的最大ring值

ethtool -g eth0

输出里：

RX ring parameters:

Pre-set maximums:

RX: 4096

RX Mini: 0

RX Jumbo: 0

TX: 4096

Pre‑set maximums 就是硬件允许的最大值，不能超过这个数字！

## 2、恢复为网卡默认值（最快修复）

# 重置rx/tx ring到驱动默认值

ethtool -G eth0 rx default tx default

## 3、如果执行完依旧异常，重启网卡

ip link set eth0 down

ip link set eth0 up

生产环境会断业务；如果是bond，需要操作bond主接口，不要操作slave。

## 4、如果还是不行，重新加载网卡驱动（谨慎）

modprobe -r 驱动名

modprobe 驱动名

# 为什么调太大会把网络搞挂

硬件寄存器上限：网卡硬件环形队列有物理最大数量，超过上限，驱动无法初始化队列，接收/发送逻辑直接失效。

内存开销暴涨：ring越大，网卡预分配的DMA内存越大。超大ring会占用大量系统内存，部分网卡驱动会出现内存分配失败，队列直接挂死。

延迟增加：即便没超过上限，ring过大，报文在队列堆积，会增大网络延迟，不适合低延迟业务。

不是越大越好！

-   高吞吐大流量：适当调大；
    
-   低延迟业务：ring要偏小；
    
-   绝对不能超过 Pre‑set maximums。
    

# 正确调优步骤

1.  先查看 ethtool -g eth0 ，记下 Pre‑set maximums
    
2.  取值不要超过硬件最大上限，建议预留余量，不要顶满上限
    

# 示例：最大rx=4096，设置3072，不要直接写4096

ethtool -G eth0 rx 3072 tx 3072

3\. 修改完成后，测试连通性，ping、iperf验证，确认没有丢包。

4\. 持久化：写进网卡配置文件，重启网络生效。

# 常见问题:

1.  直接写死很大数值，不看网卡硬件上限。不同型号网卡最大ring不一样，不能照搬别人配置。
    
2.  bond网卡，修改slave子网卡ring，不修改bond主接口。
    
3.  设置完不验证，直接上业务。
    
4.  部分网卡：rx‑jumbo ring 是巨帧专用，普通流量不要乱设置。
    

# 总结:

1.  ethtool -G 修改rx/tx ring（网卡环形缓冲区）； ethtool -g 查看。
    
2.  有硬件上限 Pre‑set maximums，超过上限会导致网卡队列异常、网络静默挂掉。
    
3.  修复： ethtool -G eth0 rx default tx default ，再重启网卡。
    
4.  ring不是越大越好，过大带来高内存占用、延迟升高。
    

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/4d927dac_1790641776972?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg2NDkwODQzNw%3D%3D%26mid%3D2247484418%26idx%3D1%26sn%3Dd5c3368c9380f2d1216dcecdd8222f4a%26chksm%3Dcfaddf76cdaebe91b90996ee8e2a223f6e8f548d1b17363fdc5dcb1da54939b8261a20f55075%26mpshare%3D1%26scene%3D1%26srcid%3D0929eHM0srAMXPgjj936SmGV%26sharer_shareinfo%3D2daab21059be786bf648d2a1c90fa02e%26sharer_shareinfo_first%3D2daab21059be786bf648d2a1c90fa02e%23rd&s=obsidian)