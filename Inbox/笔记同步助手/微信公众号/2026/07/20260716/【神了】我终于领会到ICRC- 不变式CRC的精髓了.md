---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487475&idx=1&sn=a2536c9eb32ebff4283761b244ad2481&chksm=c090157073d5b244853edcfe448dcbaceade53d1d3ac7bd05665451b42cd16dc3cbb1f454544&mpshare=1&scene=1&srcid=0716304XBFBnRXJM1qRxiQhM&sharer_shareinfo=c330d3fd470d4e91be5931d689cd9b97&sharer_shareinfo_first=c330d3fd470d4e91be5931d689cd9b97#rd
saved: 2026-07-16 08:35:51
tags:
  - 笔记同步助手
id: c817ff19-2eeb-442e-8d65-1517e4e431af
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-16 06:54

## 一、核心结论前置

### 1\. IB原生ICRC：只计算 GRH+BTH+扩展头+Payload，完全不碰IP/UDP

  

2\. RoCEv2 标准ICRC（IBTA官方规范）必须包含掩码处理后的 IPv4/IPv6头 + UDP头

3\. RoCEv2 ICRC 整套分为5个硬性步骤：前缀填充 → 可变字段掩码覆盖 → 串拼接 → CRC32标准运算 → 结果比特反转；

4\. 底层迭代依旧是 crc32\_le （输入字节按比特反转迭代，也就是IEEE CRC32）。

## 二、RoCEv2 ICRC 完整5步标准流程

## 步骤1：头部最前端增加8字节全1前缀

### 在所有待校验数据最前面追加：

0xFFFFFFFFFFFFFFFF （64bit，连续64个比特1）

这是RoCEv2对比IB原生ICRC新增的固定前缀，IB没有这8字节前缀。

## 步骤2：IP/UDP可变字段掩码覆盖（RoCE最核心扩展）

三层路由转发时，IP/UDP部分字段会被交换机/路由器改写（TTL、IP校验和、DSCP、UDP校验和）；

如果直接用原始IP/UDP做CRC，转发后CRC会变化，两端校验失败；

因此协议规定：先把路由会改动的字段全部填为0xFF掩码，再参与CRC运算：

### 1\. IPv4头部修改：

\- TTL、IP头校验和、TOS(DSCP+ECN) 全部字节置为 0xFF

### 2\. IPv6头部修改：

\- Traffic Class、Flow Label、HopLimit 掩码填充

### 3\. UDP头部修改：

\- UDP Checksum 两字节固定填 0xFFFF

关键点：RoCEv2 ICRC计算输入，包含「掩码改写后的IP头 + 掩码改写后的UDP头」，再拼接GRH/BTH/DETH/载荷

## 步骤3：拼接完整CRC输入数据流

最终送入CRC计算器的完整字节流顺序：

\[8字节全1前缀\] + \[掩码处理后IPv4/IPv6头\] + \[掩码处理后UDP头\] + BTH + RDMA扩展头(DETH/RETH) + Payload

末尾ICRC自身4字节不参与计算。

## 步骤4：CRC运算参数（和标准IEEE CRC32完全一致）

\- 生成多项式： 0x04C11DB7 （CRC32正向标准多项式）

\- 初始值： 0xFFFFFFFF

\- 输入：每个字节按比特反转输入（等价 crc32\_le 查表实现）

\- 运算结束后整体异或掩码： 0xFFFFFFFF

## 步骤5：最终比特反转得到ICRC值

CRC算出32bit结果后，需要对整个32bit做按位反转，才是报文里最终存放的ICRC；

最后这4字节ICRC在报文尾部以大端字节序存储。

  

## 三、结合UD报文举最简结构例子

一张RoCEv2 UD报文原始帧：

## 以太网头 → IP头 → UDP头 → BTH → DETH → Payload → ICRC

硬件计算ICRC时内部组装顺序：

### 1\. 加8字节 0xFFFFFFFFFFFFFFFF

### 2\. IP头可变字段全部改成0xFF掩码

### 3\. UDP校验和改为0xFFFF

### 4\. 拼接：掩码IP + 掩码UDP + BTH + DETH + Payload

### 5\. 使用标准crc32\_le完整运算（初始0xFFFFFFFF，结尾^0xFFFFFFFF）

### 6\. 32bit结果整体按比特反转

### 7\. 4字节大端存入报文尾部作为ICRC

## 四、为什么要做这么复杂的掩码+前缀设计（设计目的）

### 1\. 跨三层路由ICRC不失效

路由器转发修改TTL、IP校验和，但是我们计算CRC时已经把这些可变位置固定填0xFF，转发前后CRC输入完全一致，两端ICRC校验依旧匹配；

### 2\. 兼容以太网硬件CRC流水线

最开头64个1的前缀，适配以太网FCS硬件流水线架构，网卡硬件可以复用以太网CRC计算单元，降低芯片设计成本；

### 3\. 保证Invariant（不变式）特性

ICRC全称不变式CRC，保证报文在整个以太网转发路径中，只要业务载荷、RDMA头部没被篡改，无论三层设备怎么改写IP/UDP动态字段，ICRC永远不变，实现端到端数据完整性校验。

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/c3871b11_1784162151068?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487475%26idx%3D1%26sn%3Da2536c9eb32ebff4283761b244ad2481%26chksm%3Dc090157073d5b244853edcfe448dcbaceade53d1d3ac7bd05665451b42cd16dc3cbb1f454544%26mpshare%3D1%26scene%3D1%26srcid%3D0716304XBFBnRXJM1qRxiQhM%26sharer_shareinfo%3Dc330d3fd470d4e91be5931d689cd9b97%26sharer_shareinfo_first%3Dc330d3fd470d4e91be5931d689cd9b97%23rd&s=obsidian)