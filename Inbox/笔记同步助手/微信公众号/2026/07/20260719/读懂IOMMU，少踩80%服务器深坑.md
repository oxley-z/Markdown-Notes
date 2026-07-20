---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487497&idx=1&sn=26fc8da85ef1525aeb9aac3b75bd6756&chksm=c0974320a3ede325cd554146806865c6520f80b7dace6ce5d476134db5b1b65817161c50f233&mpshare=1&scene=1&srcid=0719FSMxMuGDbn0foQtp0KAJ&sharer_shareinfo=4b4002f520d31fb35f9a6c840160c68f&sharer_shareinfo_first=4b4002f520d31fb35f9a6c840160c68f#rd
saved: 2026-07-19 09:16:42
tags:
  - 笔记同步助手
id: 4b44ecfc-eb07-4fa7-91bb-e1402e8686c9
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-19 07:14

## 一、IOMMU是什么

CPU自带一个MMU：负责让软件用虚拟内存地址访问真实物理内存，做地址翻译+内存隔离。

IOMMU = 给网卡、显卡、SSD、FPGA这些外接硬件IO设备专用的MMU，是主板/CPU里集成的一块硬件小芯片。

简单比喻：

CPU是办公室员工，MMU帮员工把自编的房间号翻译成真实库房位置；

显卡、网卡这些外接设备是外来货车，IOMMU就是库房门口的登记岗，货车报一个自编编号，IOMMU翻译成库房真实地址，同时拦住违规闯入。

## 二、IOMMU核心两大作用

## 作用1：硬件内存隔离（最重要）

不开IOMMU：显卡/网卡可以直接读写服务器整块物理内存。

如果一台虚拟机直通了一张网卡，一旦虚拟机中毒、驱动崩溃，这个网卡能随便篡改宿主机系统内存，整台服务器直接死机、数据泄露。

开启IOMMU：给每一个硬件单独划定一片内存区域，网卡/显卡只能读写分配给自己的内存，越界访问会被硬件直接拦截。

结果：虚拟机炸了不会连累主机，不同硬件、不同业务彻底隔离，安全拉满。

## 作用2：地址翻译，解决两大现实问题

### 1\. 老旧硬件兼容大内存

老PCIe设备最多只能识别4G以内内存，服务器插几百G内存就寻址失败；

IOMMU做地址中转翻译，把高位内存映射成硬件能识别的地址，老硬件也能正常使用大容量内存。

### 2\. 拆分硬件分组，自由硬件直通

主板PCIe总线会把多个设备捆在同一个分组里，不开IOMMU时，一组里所有硬件必须整体分给一台虚拟机，没法单独拿一张网卡分给另一个机器。

开启IOMMU后可以拆分分组，多张网卡、多张显卡可以拆开，分别直通给不同虚拟机，硬件利用率大幅提高。

## 补充额外实用价值

高速RDMA网卡、GPU做数据DMA高速搬运时，IOMMU硬件完成地址转换，全程几乎不占用CPU，几乎没有性能损耗，还能减少内存地址冲突导致的丢包、报错、RNR重传异常，提升高性能集群稳定性。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/c0a733ce_1784423801685?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487497%26idx%3D1%26sn%3D26fc8da85ef1525aeb9aac3b75bd6756%26chksm%3Dc0974320a3ede325cd554146806865c6520f80b7dace6ce5d476134db5b1b65817161c50f233%26mpshare%3D1%26scene%3D1%26srcid%3D0719FSMxMuGDbn0foQtp0KAJ%26sharer_shareinfo%3D4b4002f520d31fb35f9a6c840160c68f%26sharer_shareinfo_first%3D4b4002f520d31fb35f9a6c840160c68f%23rd&s=obsidian)