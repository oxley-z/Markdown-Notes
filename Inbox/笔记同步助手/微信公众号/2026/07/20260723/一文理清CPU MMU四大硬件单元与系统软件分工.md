---
author: 智能控制设计
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxNDQ5MTI2OA==&mid=2247487517&idx=1&sn=63f77b7927f61f66630d7d1faec881a8&chksm=c049aae35229d06552e67140ea02c620f4409cdb580155ceefa6e2dba59221b082795904ade1&mpshare=1&scene=1&srcid=0723ThSL2YEuxfUsQy8RJRNj&sharer_shareinfo=085cb366177872b95db2ca193410542f&sharer_shareinfo_first=085cb366177872b95db2ca193410542f#rd
saved: 2026-07-23 08:18:35
tags:
  - 笔记同步助手
id: 36875f3e-d0e4-49f8-8a1d-d6feaade2f3d
---

公众号名称：智能控制设计

作者名称：智能控制设计

发布时间：2026-07-23 07:16

## 一、Linux 的 MMU 硬件由 4 个核心模块组成

### 1\. TLB（地址缓存）

缓存最近的虚拟→物理地址映射，避免每次查表，决定内存速度。

### 2\. 页表遍历器（Page Walker）

TLB 没命中时，硬件自动遍历四级页表，不用 CPU 参与。

### 3\. 权限检查单元

硬件直接校验：读写权限、用户/内核隔离、NX 禁止执行，非法访问直接报错。

### 4\. 页表基址寄存器（CR3/TTBR）

保存当前进程页表起始地址，切换进程只需要改这个寄存器。

## 二、Linux 软件必须做的 5 件核心配合工作（缺一不可）

### 1\. 内核提前建好页表，开机开启 MMU

没页表、不开 MMU，系统只能跑物理地址，无虚拟内存、无隔离。

### 2\. 给每个进程维护一套独立页表

进程隔离的根本：

不同进程虚拟地址一样，但物理内存完全独立，靠内核页表区分。

### 3\. 缺页异常处理（最关键）

虚拟地址有、物理内存没有时，CPU 报异常，Linux 负责：

\- 分配物理内存

\- 拷贝文件/堆内存

\- 写时复制 COW

\- 非法访问杀进程

没有软件缺页处理，MMU 完全无法工作。

### 4\. 手动刷新 TLB

改了页表必须刷 TLB，否则硬件用旧地址 → 内存错乱、死机。

### 5\. 进程切换时切换页表基址

调度程序切换 CR3，瞬间切换整个地址空间，实现进程隔离。

### 终极一句话总结

MMU 硬件负责地址翻译和权限拦截

Linux 软件负责建页表、填映射、处理缺页、刷缓存、切换地址空间

关注我，一起了解更多，谢谢您！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/8df30b3a_1784765914563?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxNDQ5MTI2OA%3D%3D%26mid%3D2247487517%26idx%3D1%26sn%3D63f77b7927f61f66630d7d1faec881a8%26chksm%3Dc049aae35229d06552e67140ea02c620f4409cdb580155ceefa6e2dba59221b082795904ade1%26mpshare%3D1%26scene%3D1%26srcid%3D0723ThSL2YEuxfUsQy8RJRNj%26sharer_shareinfo%3D085cb366177872b95db2ca193410542f%26sharer_shareinfo_first%3D085cb366177872b95db2ca193410542f%23rd&s=obsidian)