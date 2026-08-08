---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487550&idx=1&sn=5f78874afa0247be4aa2260eeca0acac&chksm=c03844993bfec6f0879642744964e5b851b29a2b5d13e9579d3fffb1c1b1cb6ba280b812042d&mpshare=1&scene=1&srcid=0729GNLn1fAWKmEdB9Y1AB84&sharer_shareinfo=d1dc6b7e5dcfe975775cb372761d3e8c&sharer_shareinfo_first=d1dc6b7e5dcfe975775cb372761d3e8c#rd
saved: 2026-07-29 13:24:10
tags:
  - 笔记同步助手
id: 66598ac6-5b0e-41b6-82b6-0d30a37bd563
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-29 12:21

rdma-core中 libibverbs 是通用抽象层，各个厂商实现Provider动态库（.so插件），例如 libmlx5.so 、 libirdma.so 、 libefa.so 。

应用调用 ibv\_get\_device\_list() 枚举网卡，整个匹配流程依靠sysfs、设备标识、驱动名称三级匹配。

### 完整匹配流程（按执行顺序）

## 阶段1：内核侧，网卡驱动完成注册（匹配源头）

1\. 网卡硬件初始化，内核厂商驱动（ mlx5\_core.ko ）识别PCI设备ID（VID/DID）。

2\. 驱动向Linux RDMA子系统 ib\_core 注册RDMA设备。

3\. ib\_core在/sys/class/infiniband/ 生成设备目录，例如 /sys/class/infiniband/mlx5\_0

关键文件（匹配的核心依据）：

/sys/class/infiniband/mlx5\_0/driver → 内容： mlx5\_core

/sys/class/infiniband/mlx5\_0/device/vendor PCI厂商ID

/sys/class/infiniband/mlx5\_0/device/device PCI设备ID

### 核心：每一个RDMA设备，在内核导出了【内核驱动名称】

MLX网卡固定为 mlx5\_core ，Intel RDMA网卡为 irdma ，AWS EFA为 efa 。

## 阶段2：libibverbs加载所有Provider插件

libibverbs启动时，扫描插件目录：

/usr/lib64/rdma-core/providers/

加载目录下所有厂商so： libmlx5.so 、 libirdma.so ……

每个Provider内置一张匹配表，硬编码写死：

provider名称 ↔ 内核驱动名称

示例 libmlx5.so 内部配置：

driver\_name = "mlx5\_core"

libirdma.so 内部：

driver\_name = "irdma"

## 阶段3：枚举设备 + 一对一匹配（核心逻辑）

1\. ibv\_get\_device\_list() 遍历 /sys/class/infiniband/ 下所有RDMA设备

2\. 对每一个网卡设备，读取 /sys/xxx/driver 文件，拿到内核驱动名字串（例如 mlx5\_core ）

### 3\. libibverbs轮询所有已加载的Provider插件：

如果Provider注册的driver\_name == 设备的内核驱动名称 → 匹配成功！

4\. 匹配成功后，libibverbs将该Provider绑定到此ibv\_device。

5\. 后续 ibv\_open\_device() 打开设备时，底层调用对应provider的接口实现。

极简一句话：

内核导出网卡对应的驱动名字，Provider插件内置支持的驱动名字，字符串相等即匹配。

### 补充两种特殊匹配方式

## 方式1：Direct Verbs场景（MLX5特有）

普通verbs依靠uverbs通道匹配；Direct Verbs模式下，Provider除了核对驱动名称，还会读取PCI vid/did，确认硬件型号，启用mmap用户态直接访问硬件寄存器的私有能力。

## 方式2：兼容旧设备、多Provider优先级

一台服务器多张不同厂商网卡：

\- CX8（mlx5\_core）→ 匹配libmlx5.so

\- Intel E810（irdma）→匹配libirdma.so

互不干扰，同进程可以同时使用多个不同provider。

## 关键数据链路简图

  

网卡硬件 → 内核mlx5\_core.ko → /sys/class/infiniband/mlx5\_0/driver = mlx5\_core

↓

libibverbs读取driver名称

↓

遍历providers：libmlx5.so 声明支持 "mlx5\_core"

↓

匹配成功，绑定libmlx5.so为此设备驱动实现

### 高频误区澄清

### 1\. ❌ 依靠PCI VID直接匹配provider

✅ 标准流程不直接用PCI ID匹配，标准匹配key是【内核驱动名称字符串】。PCI ID仅作为Direct Verbs私有扩展校验。

### 2\. ❌ Provider会和内核驱动版本自动协商

✅ 无自动协商；如果rdma-core版本太旧，provider不支持当前网卡固件/内核驱动新特性，匹配依然成功，但调用新接口会报错。

### 3\. ❌ RoCE / IB设备匹配逻辑不一样

✅ RoCEv2和IB网卡使用同一套匹配逻辑，区分传输协议是上层功能，不影响provider匹配流程。

### 4\. ❌ 修改网卡名称（mlx5\_0 → mlx5\_10）会破坏匹配

✅ 不会！匹配依据是 driver 文件内容，不是设备名mlx5\_x。

## 实操验证命令

\# 查看所有RDMA设备对应的内核驱动名称

ls /sys/class/infiniband/\*/driver -l

\# 查看系统加载的providers

ls /usr/lib64/rdma-core/providers/

\# ibv\_devinfo可以看到设备绑定的provider

ibv\_devinfo -d mlx5\_0

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/47655a25_1785302649535?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487550%26idx%3D1%26sn%3D5f78874afa0247be4aa2260eeca0acac%26chksm%3Dc03844993bfec6f0879642744964e5b851b29a2b5d13e9579d3fffb1c1b1cb6ba280b812042d%26mpshare%3D1%26scene%3D1%26srcid%3D0729GNLn1fAWKmEdB9Y1AB84%26sharer_shareinfo%3Dd1dc6b7e5dcfe975775cb372761d3e8c%26sharer_shareinfo_first%3Dd1dc6b7e5dcfe975775cb372761d3e8c%23rd&s=obsidian)