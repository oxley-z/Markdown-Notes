---
author: 帅呆
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzAwMDM4NTUyNw==&mid=2652277717&idx=1&sn=c3afdbb34ba49adc78a2f4102b7e0427&chksm=803a979147a85ed303a237dcf89217e85494dd2c7e08f45074a5897e5a81ef833fa87ab710b8&mpshare=1&scene=1&srcid=0713gHqojWwDkRnx1hOpH71z&sharer_shareinfo=8f2121eabb4d0173020f30c63db2b8cc&sharer_shareinfo_first=8f2121eabb4d0173020f30c63db2b8cc#rd
saved: 2026-07-13 07:59:58
tags:
  - 笔记同步助手
id: 657d377f-67cd-4aa1-85e0-ce8754ae4022
---

公众号名称：SSDFans

作者名称：帅呆

发布时间：2026-07-13 07:41

![[Inbox/笔记同步助手/微信公众号/2026/07/images/fee591199d76c800263ff451dc338016_MD5.gif]]

点击蓝字

关注我们

因为公众号平台更改了推送规则。记得点下右下角的大拇指“赞”和红心“推荐”。这样每次新文章推送，就会第一时间出现在订阅号列表里。

因为公众号平台更改了推送规则。记得点右下角的大拇指“赞”和红心“推荐”。这样每次新文章推送，就会第一时间出现在订阅号列表里。

在AI加速器快速发展的时代，High Bandwidth Memory（HBM）、High Bandwidth Flash（HBF）和High Bandwidth Storage（HBS）正逐步构建起层次分明且协同工作的内存架构。HBM提供极致的运行速度，HBF实现海量模型存储容量，而HBS则将高带宽堆叠式内存引入移动设备，使AI能够覆盖更广泛的应用场景。

未来的大型AI系统预计将越来越多地采用HBM作为热数据层、HBF作为冷数据层的组合，而边缘智能将利用HBS将高带宽存储扩展到全新的应用场景。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/1eee533455f68bc75f0f1491fbbc3166_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/2026/07/images/0d83f0fc08661282519442d6f33102ec_MD5.jpg]]

HBM：AI训练的“超高速工作内存”

HBM采用堆叠式DRAM架构，通过TSV技术实现超高带宽。在大型模型训练过程中，计算核心必须以极低延迟访问参数和中间特征。在此背景下，HBM如同一个紧邻GPU的“超高速工作台”。

性能跃升

HBM3E产品已实现高达1.2 TB/s的带宽，约为DDR5内存的五倍，有效突破了AI计算中的“内存墙”，支持高度并行的大模型工作负载。预计到2026年，HBM3E仍将作为主流解决方案，堆叠层数从8个芯片增至12个芯片，性能持续提升。

能效优化

与GDDR6相比，HBM的功耗降低了40%至50%，显著缓解了热管理难题，并降低了AI集群的能源成本。因此，它特别适用于高密度数据中心部署。

空间压缩

通过垂直集成，最多可将32个DRAM芯片集成到14毫米\*14毫米的紧凑封装中，显著提升集成密度。

主要供应商

SK海力士（HBM2E、HBM3和HBM3E的主要供应商）、三星、美光

HBF：AI推理的“大容量高速文件柜”

HBF基于NAND闪存，通过堆叠和高密度封装技术实现接近HBM的带宽水平，同时具备更大的存储容量。在AI推理工作负载中，HBF作为模型权重的主要存储库。它并非取代HBM，而是作为位于HBM之上的大容量高速缓存层。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/c7529aff05c72c87ba432fa61b15beb1_MD5.jpg]]

主要优势

在容量方面，单个HBF堆叠最高可达512 GB，而八层堆叠配置可提供4 TB的总容量，是HBM4（64 GB）的8至16倍。Kioxia已展示出一款原型模块，支持5 TB容量和64 GB/s的带宽。

从成本角度来看，与DRAM相比，NAND闪存每吉字节的成本仅为DRAM的十分之一到二十分之一，具有极高的性价比。在性能方面，其带宽可达1.6 TB/s至3.2 TB/s，可与HBM3相媲美，并远超传统SSD，后者通常峰值性能约为7,000 MB/s。

技术创新

HBF采用12至16层3D NAND芯片的垂直堆叠结构，并通过TSV技术实现互连。结合专用逻辑芯片和插接芯片，该架构形成了一种高密度的互连网络，可实现对多个NAND阵列的并行访问。基于SK海力士的238层3D NAND技术，12层HBF堆叠可实现有效2,866层结构，总容量达768 GB。

代表供应商

三星、SK海力士、铠侠、美光、CXMT

HBS：面向移动设备的“混合高带宽存储系统”

HBS将移动DRAM和NAND闪存集成到新一代存储器中，旨在提升智能手机和平板电脑的AI性能。该技术支持最多16个DRAM和NAND模块的堆叠，并通过VFO技术实现互联，从而提高数据处理速度。

SK海力士目前正在研发的VFO技术采用铜线替代铜柱。DRAM芯片以阶梯状堆叠方式排列，缝隙中注入环氧树脂并固化，以稳定结构。随后，垂直的柱状导线和再分布层将堆叠结构与基板连接起来。

VFO将FOWLP与DRAM堆叠技术相结合。通过引入垂直互连，显著缩短了多个DRAM层之间的信号传输路径，使布线长度减少至传统内存设计的四分之一以下，并提升了4.9%的能效。尽管热输出增加了约1.4%，但整体封装厚度却减少了27%。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f1b957477d228c5ce96dbfdd88aab3af_MD5.jpg]]

主要供应商

三星（LPDDR + UFS 集成解决方案领先企业），SK海力士（移动DRAM + NAND堆叠封装），美光（移动堆叠存储解决方案）

HBM vs HBF vs HBS：AI训练、推理与边缘AI的内存架构

实际上，未来的大型AI系统预计将采用多层级内存架构，其中HBM作为热存储，HBF作为冷或温区存储，而HBS则满足边缘AI的需求。

在真实AI系统中它们如何协同工作

-   AI训练（数据中心GPU）：HBM紧邻计算核心，负责处理活跃参数和中间张量，其中延迟和带宽至关重要。
    
-   AI推理（大模型）：HBF作为模型权重的高容量、高带宽存储层级，在分层内存系统中位于HBM之上，显著减少对较慢SSD的依赖。
    
-   Edge / On-device AI HBS 将 DRAM 和 NAND 集成于紧凑且节能的封装中，支持在智能手机、平板电脑及其他移动平台上的本地推理。
    

HBM vs HBF vs HBS：基于AI工作负载场景的对比

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ee3d7fda63a907ed14e274303a89b498_MD5.jpg]]

AI系统的内存层次结构

现代AI系统依赖于分层的内存架构，以在带宽、容量、延迟和成本之间取得平衡。​在层次结构的顶层，HBM作为热存储，与GPU或AI加速器紧密耦合，为训练和推理过程中的活跃参数及中间张量提供超高速带宽和超低延迟。​在HBM之下，HBF作为大容量、高带宽的存储层级，用于存储海量模型权重，减少频繁向较慢的外置存储设备（如SSD）传输数据。​在边缘端，HBS将移动式DRAM和NAND闪存集成到紧凑且节能的封装中，在严格的功耗和外形尺寸限制下实现高带宽的数据访问。​这些层级共同构成一个可扩展且具备工作负载感知能力的内存层次结构，使AI系统能够在数据中心和边缘环境中高效支持大模型训练、高吞吐量推理以及设备端智能应用。​

原文链接：

https://www.lovechip.com/blog/hbm-vs-hbf-vs-hbs

<table style="border-collapse: collapse"><tbody><tr><td colspan="2" data-colwidth="191,0" width="191,0" style="border: 1px solid \#ddd; padding: 6px 10px"><div style="text-align: center; color: rgb(95, 156, 239); font-size: 24px"><p style="color: rgb(0, 0, 0)"><strong><span>高端微信群介绍</span></strong></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>创业投资群</span></strong></em></span></p></div></div><br></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: left; font-size: 14px; color: rgb(0, 138, 175)"><p style="color: rgb(0, 0, 0)"><span>AI、IOT、芯片创始人、投资人、分析师、券商</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>闪存群</span></strong></em></span></p></div></div><br></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: left; font-size: 14px; color: rgb(0, 138, 175)"><p style="color: rgb(0, 0, 0)"><span>覆盖5000多位全球华人闪存、存储芯片精英</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>云计算群</span></strong></em></span></p></div></div><br></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: center; font-size: 14px; color: rgb(0, 138, 175)"><p style="text-align: left; color: rgb(0, 0, 0)"><span>全闪存、软件定义存储SDS、超融合等公有云和私有云讨论</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>AI芯片群</span></strong></em></span></p></div></div><br></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: center; font-size: 14px; color: rgb(0, 138, 175)"><p style="text-align: left; color: rgb(0, 0, 0)"><span>讨论AI芯片和GPU、FPGA、CPU异构计算</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>5G群</span></strong></em></span></p></div></div><br></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: center; font-size: 14px; color: rgb(0, 138, 175)"><p style="text-align: left; color: rgb(0, 0, 0)"><span>物联网、5G芯片讨论</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>第三代半导体群</span></strong></em></span></p></div></div></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: center; font-size: 14px; color: rgb(0, 138, 175)"><p style="text-align: left; color: rgb(0, 0, 0)"><span>氮化镓、碳化硅等化合物半导体讨论</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><em><strong><span>存</span></strong></em><span>储芯片群</span></strong></em></span></p></div></div></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: center; font-size: 14px; color: rgb(0, 138, 175)"><p style="text-align: left; color: rgb(0, 0, 0)"><span>DRAM、NAND、3D XPoint等各类存储介质和主控讨论</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>汽车电子群</span></strong></em></span></p></div></div></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: center; font-size: 14px; color: rgb(0, 138, 175)"><p style="text-align: left; color: rgb(0, 0, 0)"><span>MCU、电源、传感器等汽车电子讨论</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>光电器件群</span></strong></em></span></p></div></div></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: center; font-size: 14px; color: rgb(0, 138, 175)"><p style="text-align: left; color: rgb(0, 0, 0)"><span>光通信、激光器、ToF、AR、VCSEL等光电器件讨论</span></p></div></td></tr><tr><td data-colwidth="191" width="191" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="color: rgb(0, 0, 0)"><div style="color: rgb(0, 0, 0)"><p style="text-align: center; color: rgb(0, 0, 0)"><span style="text-decoration: underline; color: rgb(95, 186, 203)"><em><strong><span>渠道群</span></strong></em></span></p></div></div></td><td data-colwidth="NaN" width="NaN" style="border: 1px solid \#ddd; padding: 6px 10px"><div powered-by="xiumi.us" style="text-align: center; font-size: 14px; color: rgb(0, 138, 175)"><p style="text-align: left; color: rgb(0, 0, 0)"><span>存储和芯片产品报价、行情、渠道、供应链</span></p></div></td></tr></tbody></table>

![[Inbox/笔记同步助手/微信公众号/2026/07/images/23b10d4531dc46ece2ce97b4bdc99096_MD5.jpg]]

**< 长按识别二维码添加好友 >**

**加入上述群聊**

![[Inbox/笔记同步助手/微信公众号/2026/07/images/0e5864d0298ec1fe7ed9969d89f54c45_MD5.jpg]]

****长按并关注****

带你走进万物存储、万物智能、  

万物互联信息革命新时代

![[Inbox/笔记同步助手/微信公众号/2026/07/images/2100d10a7fbd8db8cc461665efdf72d6_MD5.jpg]]

微信号：SSDFans

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/66742167_1783900796165?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzAwMDM4NTUyNw%3D%3D%26mid%3D2652277717%26idx%3D1%26sn%3Dc3afdbb34ba49adc78a2f4102b7e0427%26chksm%3D803a979147a85ed303a237dcf89217e85494dd2c7e08f45074a5897e5a81ef833fa87ab710b8%26mpshare%3D1%26scene%3D1%26srcid%3D0713gHqojWwDkRnx1hOpH71z%26sharer_shareinfo%3D8f2121eabb4d0173020f30c63db2b8cc%26sharer_shareinfo_first%3D8f2121eabb4d0173020f30c63db2b8cc%23rd&s=obsidian)