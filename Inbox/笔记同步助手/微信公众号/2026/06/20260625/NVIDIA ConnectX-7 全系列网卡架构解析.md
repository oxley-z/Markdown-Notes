---
author: 智算交付笔记
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzYzNzgyMDU4NA==&mid=2247485530&idx=1&sn=1dda35096b667e3095812ebe0c217206&chksm=f1c2c2a58cb653dc30c25c43d59c57f5267de46decb54be0aa96688f35d7276b3a120dd51ef7&mpshare=1&scene=1&srcid=0625vPzQPMfrTRcjnGdPidM3&sharer_shareinfo=429eac7a34e84f484f6f1e6a0244ca1b&sharer_shareinfo_first=429eac7a34e84f484f6f1e6a0244ca1b#rd
saved: 2026-06-25 15:43:29
tags:
  - 笔记同步助手
id: c8dc8354-cc11-47dd-8634-52ee10cb488b
---

公众号名称：智算交付笔记

作者名称：智算交付笔记

发布时间：2026-06-22 07:20

### ![[Inbox/笔记同步助手/微信公众号/2026/06/images/fe21b17c3b2f510f75834deec9fdf101_MD5.jpg]]

为了帮助大家在面对CX7选型时不踩坑，本文将结合 NVIDIA **ConnectX-7** 官方的主流型号，从**外观形态、****端口****速率、规格编码**等多个维度来详细的解析。

### 一、 外观形态与接口类型

NVIDIA ConnectX-7 网卡在外观上针对不同的服务器架构和机架空间，主要提供了两大主流形态：标准 PCIe 插卡式 和 **OCP** **3.0 规格卡**。同时，为了支撑 400G 的带宽，其物理接口也迎来了全面升级：

## OSFP接口

**![[Inbox/笔记同步助手/微信公众号/2026/06/images/0b2f0614fe65bfe5b6205805d8239af4_MD5.jpg]]**

-   **外观特征**：物理体积比传统 QSFP 略大，接口内部拥有 8 个通道。
    
-   **用途**：主要用于 InfiniBand NDR 400Gb/s 架构。不仅支持单口 400G 直连，还能通过一分二线缆连接两路 200G 设备。
    

## QSFP112 接口

**![[Inbox/笔记同步助手/微信公众号/2026/06/images/a48326215ad450ac2bde3e74f873a758_MD5.jpg]]**

-   **外观特征**：体积与传统的 QSFP56 相同，但内部升级为 4 条 112G PAM4 高速通道。
    
-   **用途**：实现单口 400G 或双口高密度部署。
    

## SFP56 接口

![[Inbox/笔记同步助手/微信公众号/2026/06/images/e13717f0728dffd8780a6d7bcaeaea5f_MD5.jpg]]

-   **外观特征**：小型可插拔接口，通常呈多端口（如 4 端口）高密度并排排列。
    
-   **用途**：单通道支持 50G 速率，四端口并联可提供极高的端口密度。
    

### 二、 型号分类

根据官方主流的 OPN，我们可以将 CX7 家族划分为四大类：

#### 1\. 单端口 OSFP 适配器卡

**代表** **OPN****：**`MCX75310AAS-NEAT` / `MCX75310AAC-NEAT`（400G 速率）

-   `MCX75310AAS-HEAT`（200G 速率）
    
-   `MCX75510AAS-NEAT`（400G 速率） / `MCX75510AAS-HEAT`（200G 速率）
    

![[Inbox/笔记同步助手/微信公众号/2026/06/images/1ed54b1cd44e0cf86b2dbf39ddb98000_MD5.jpg]]

以 `NEAT` 结尾的型号原生支持 **NDR** **400****Gb****/s**，通常通过 VPI 技术同时兼容 InfiniBand 和以太网协议。而 `HEAT` 结尾的型号则工作在 **HDR** **200Gb/s** 速率，适用于高性价比过渡方案。

#### 2\. 双端口 QSFP112 适配器卡

**代表** **OPN****：**`MCX755106AS-HEAT` / `MCX755106AC-HEAT`

-   `MCX713106AS-CEAT` / `MCX713106AC-CEAT`
    
-   `MCX713106AS-VEAT` / `MCX713106AC-VEAT`
    

![[Inbox/笔记同步助手/微信公众号/2026/06/images/dfd764eefa74417051f86b9d5558a1f5_MD5.jpg]]

其中，`MCX755` 开头的型号属于功能更全面的 VPI/高级版本，最高支持单口 200G（HEAT）；而 `MCX713` 开头则专注于纯以太网生态，分别提供 100G（CEAT）和 200G（VEAT）等不同速率等级。

#### 3\. 单端口 QSFP112 适配器卡

-   **代表** **OPN****：**`MCX715105AS-WEAT`
    

![[Inbox/笔记同步助手/微信公众号/2026/06/images/cd3130b67065132e063f0cf045b54fd1_MD5.jpg]]

这是纯以太网架构的网卡，采用单个 QSFP112 接口，支持高达 400Gb/s的以太网吞吐。

另外`MCX715105AS-WEAT`型号支持端口拆分，默认情况是一个 400GbE 的以太网端口；

![[Inbox/笔记同步助手/微信公众号/2026/06/images/33dc29c0114bed9b592a5b1327deeb6e_MD5.jpg]]

在交换机配置后，可以以支持四个 100GbE 以太网端口。配置如下图所示。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/0b6e29ed8a36922f4151f159281c7c40_MD5.jpg]]

使用以下命令：

```
mlxconfig -d  set MODULE_SPLIT_M0[0]=1 MODULE_SPLIT_M0[1]=2 MODULE_SPLIT_M0[2]=3 MODULE_SPLIT_M0[3]=4 MODULE_SPLIT_M0[4..15]=FF
mlxconfig -d  set MODULE_SPLIT_M0[0]=1 MODULE_SPLIT_M0[1]=2 MODULE_SPLIT_M0[2]=3 MODULE_SPLIT_M0[3]=4 MODULE_SPLIT_M0[4..15]=FF
```

#### 4\. 四端口 SFP56 适配器卡

**代表** **OPN****：**

-   纯数据版：`MCX713104AC-ADAT` / `MCX713104AS-ADAT`
    

![[Inbox/笔记同步助手/微信公众号/2026/06/images/0ca1fab175118132ed4550cc9c4df067_MD5.jpg]]

-   带脉冲同步版：`MCX713114TC-GEAT`
    

![[Inbox/笔记同步助手/微信公众号/2026/06/images/737f9ddc15fba8ad61f331d79dfae298_MD5.jpg]]

在一块板卡上提供了 4 个 SFP56 端口（每个端口 50G，总带宽 200G）。特别值得注意的是 **`MCX713114TC-GEAT`**，它配备了 **PPS****输入/输出接口**。 这是专门为**5G** **通信基站****、金融****高频交易****、数据同步广播**等对时间同步精度要求达到“纳秒级”的场景设计的。

### 三、 规格编码解码

通过拆解上面提到的具体型号，我们可以总结出 CX7 的解码公式：

以 **`MCX75310AAS-NEAT`** 和 **`MCX713106AC-CEAT`** 为例：

**1\. 前缀：**`MCX7` 代表 ConnectX-7 系列。

### 2\. 第五位数字（网络协议）：

-   **`5`**代表 **VPI**：既能跑 InfiniBand（IB），也能切换跑标准以太网（Ethernet）。如 `MCX753...`。
    
-   **`1`**代表 **Ethernet****（纯****以太网****）**：专为以太网生态设计，不支持 IB。如 `MCX713...`。
    

### 3\. 第八、九位字符（端口数量与接口物理类型）：

-   **`0A`** = **单****端口** **OSFP**（Single-port OSFP）
    
-   **`05`** = **单****端口** **QSFP112**（Single-port QSFP112）
    
-   **`06`** = **双端口 QSFP112**（Dual-port QSFP112）
    
-   **`04`** **/** **`14`** = **四****端口** **SFP56**（Quad-port SFP56）
    

### 4\. 第十、十一位字母（安全特性与结构件）：

-   **`AS`** = 支持 **Secure Boot（安全启动）**，配备硬件信任根，出厂带标准卡固定扣/普通档片。
    
-   **`AC`** = 同样具备安全特性，但在特定 OCP 3.0 或插卡型号中，代表采用了**不同的卡扣/拉环机制**，采购时需与服务器机箱背板严格匹配。
    
-   **`TC`** = 特种卡，如带有 PPS 时钟同步物理接口的版本。
    

### 5\. 横杠后的四位字母：

1.  **`NEAT`** = **NDR** **速率 (400** **Gb****/s)** \* **`WEAT`** = **400 Gb/s** **以太网****速率**
    
2.  **`HEAT`** = **HDR** **/ 200** **Gb****/s 速率**
    
3.  **`VEAT`** = **200** **Gb****/s** **以太网****速率**
    
4.  **`CEAT`** = **100** **Gb****/s** **以太网****速率**
    
5.  **`ADAT`** **/** **`GEAT`** = 分别对应四端口并联下的特定以太网吞吐聚合速率。
    

### 四、 CX7全系速查表

为了让大家更直观地对比，我们提供一张完整的CX7网卡速查表：

<table style="border-collapse: collapse"><tbody><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">订购编码 (</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">OPN</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">)</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">端口</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">形态</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">端口</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">数量</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">最高协议速率</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">协议属性</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">特殊属性 / 适用场景</span></span></strong></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX75310AAS-NEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">OSFP</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">1</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">400G (</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">NDR</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">)</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">VPI (IB/ETH)</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">标准安全版 / 大模型 AI 训练集群</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX75310AAC-NEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">OSFP</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">1</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">400G (</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">NDR</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">)</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">VPI (IB/ETH)</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">特殊固定扣版本 / 适配特定 AI 服务器机卡</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX75310AAS-HEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">OSFP</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">1</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">200G (</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">HDR</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">)</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">VPI (IB/ETH)</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">200G 骨干网与高性能存储过渡方案</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX75510AAS-NEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">OSFP</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">1</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">400G (</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">NDR</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">)</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">VPI (高级版)</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">高性能计算、复杂网络多租户隔离卸载</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX75510AAS-HEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">OSFP</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">1</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">200G (</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">HDR</span></span></strong><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">)</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">VPI (高级版)</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">200G 速率下的高级计算与虚拟化卸载</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX715105AS-WEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">QSFP112</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">1</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">400G</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">纯以太网</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">400G 高速以太网、高性能单路分布式存储</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX755106AS-HEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">QSFP112</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">2</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">200G</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">VPI (IB/ETH)</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">标准双口冗余 / 兼顾 IB 与以太网的高密节点</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX755106AC-HEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">QSFP112</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">2</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">200G</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">VPI (IB/ETH)</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">特殊物理卡扣版 / 适用于定制化机架服务器</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX713106AS-CEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">QSFP112</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">2</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">100G</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">纯以太网</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">标准双口 100G / 云数据中心通用计算节点</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX713106AC-CEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">QSFP112</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">2</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">100G</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">纯以太网</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">特殊物理卡扣版 / 数据中心高密部署</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX713106AS-VEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">QSFP112</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">2</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">200G</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">纯以太网</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">标准双口 200G / 现代企业级高速以太核心网</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX713106AC-VEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">QSFP112</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">2</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">200G</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">纯以太网</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">特殊物理卡扣版 / 200G 高速以太网节点</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX713104AS-ADAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">SFP56</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">4</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">50G每端口</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">纯以太网</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">四口高密度接入 / 适用于复杂拓扑交换上联</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX713104AC-ADAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">SFP56</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">4</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">50G每端口</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">纯以太网</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">四口高密度 / 对应特殊插槽固定结构</span></span></td></tr><tr><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><strong><span><span style="font-size: 16px; color: rgb(0, 0, 0)">MCX713114TC-GEAT</span></span></strong></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">SFP56</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">4</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">高密特定速率</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">纯以太网</span></span></td><td style="font-size:10pt; border: 1px solid \#ddd; padding: 6px 10px"><span><span style="font-size: 16px; color: rgb(0, 0, 0)">自带 PPS 输入/输出 / 金融高频交易、5G 基站时钟同步</span></span></td></tr></tbody></table>

### 五、 选型建议

在选型 ConnectX-7 网卡时，建议遵循以下“三步法”：

**1\. 如果目标是搭建 AI 智算中心、跑大模型分布式训练，优先选择以** `MCX75` 开头的 **VPI 万能卡**；如果是云服务或存储网络，选择以 `MCX71` 开头的**纯****以太网****卡**能大幅节省预算。

**2\. 单口 OSFP（**`AA`）是 AI 算力的标配；双口 QSFP112（`06`）是实现双链路冗余。同时，务必核对好后缀的 `AS` 与 `AC`，确保网卡的物理固定结构与服务器的机箱、拉环组件完美兼容。

**3\. 如果您的业务涉及电信 5G 核心网、金融交易等，直接锁定带有 PPS 硬件时钟同步接口的特种型号（如** `MCX713114TC-GEAT`）。

欢迎大家点赞和转发，谢谢！

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/60c30c0b_1782373406187?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzYzNzgyMDU4NA%3D%3D%26mid%3D2247485530%26idx%3D1%26sn%3D1dda35096b667e3095812ebe0c217206%26chksm%3Df1c2c2a58cb653dc30c25c43d59c57f5267de46decb54be0aa96688f35d7276b3a120dd51ef7%26mpshare%3D1%26scene%3D1%26srcid%3D0625vPzQPMfrTRcjnGdPidM3%26sharer_shareinfo%3D429eac7a34e84f484f6f1e6a0244ca1b%26sharer_shareinfo_first%3D429eac7a34e84f484f6f1e6a0244ca1b%23rd&s=obsidian)