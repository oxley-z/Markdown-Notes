---
author: hb的小羊
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg2NDkwODQzNw==&mid=2247484412&idx=1&sn=cadc31c3ee081dea6f567860dece97f3&chksm=cf3d7f0b88bd1b721ca645cba744b96ad61c6343e77fb53b17e80f3386fe19bb3fb9e656afec&mpshare=1&scene=1&srcid=0930QWoLrcWKRgmJKZvMcfc5&sharer_shareinfo=f116b00ad5d7c645cab22ed99cbdfae4&sharer_shareinfo_first=f116b00ad5d7c645cab22ed99cbdfae4#rd
saved: 2026-09-30 08:25:54
tags:
  - 笔记同步助手
id: 3ed93bbf-0f86-49bf-8a63-6ae88f32c5b6
---

公众号名称：hb的小羊

作者名称：hb的小羊

发布时间：2026-09-30 06:06

场景： insmod / modprobe 报错，模块加载不成功，dmesg报错误。

# 前置检查

## 1\. 确认内核版本匹配

uname -r

查看ko文件编译时的内核版本，必须和运行内核完全一致。

内核版本不匹配是最常见原因，会报 Invalid module format 。

## 2\. 确认模块是否已经内置进内核

zcat /proc/config.gz | grep 模块名

\- =y ：编译进内核，不能作为模块加载/卸载，直接报错。

\- =m ：编译为模块，可以正常加载。

## 3\. 查看模块依赖

modinfo 模块名

看 depends: 字段，列出依赖的其他模块。

modprobe 会自动加载依赖； insmod 不会，会直接缺依赖报错。

# 第一步：看内核日志（最重要）

所有驱动加载错误都会输出到dmesg，优先看这里

## \# 实时看日志

dmesg -w

## \# 过滤模块相关报错

dmesg | grep -iE 'module|xxx模块名|error|fail'

常见报错关键词：

\- Invalid module format ：内核版本不匹配

\- Unknown symbol in module ：缺少依赖模块，符号找不到

\- No such device ：硬件不存在，或者硬件没有识别到

\- Operation not permitted ：权限不足，需要root执行

\- Module license tainting ：许可证不兼容，内核taint标记，不一定崩溃，但会告警

# 第二步：区分两种加载命令的问题

## 1\. insmod /path/xxx.ko

\- 不会自动加载依赖，缺依赖直接失败

\- 必须给完整ko文件路径

\- 建议优先用 modprobe 模块名

## 2\. modprobe 模块名

\- 自动处理依赖；如果失败，执行：

## \# 更新模块依赖缓存

depmod -a

新增ko文件后，一定要执行 depmod -a ，否则modprobe找不到模块。

# 第三步：常见错误场景与处理

场景1： Unknown symbol in module 未知符号

原因：缺少依赖模块，依赖的ko没有加载。

解决：

## \# 查看依赖列表

modinfo 模块名

## \# 手动加载所有依赖模块，再加载目标模块

modprobe 依赖模块1

modprobe 依赖模块2

modprobe 目标模块

## 场景2： Invalid module format

原因：ko编译内核版本 ≠ 当前运行内核版本。

解决：重新编译ko，使用当前运行内核的头文件编译。

## 场景3： No such device

模块加载了，但是硬件设备不存在。

\- 检查硬件： lspci / lsusb 看设备是否存在

\- 设备没被识别，驱动就算加载也无法工作。

## 场景4： Module is in use 卸载失败

模块正在被设备/进程占用，无法卸载。

## 1\. 先停止使用该驱动的业务进程

## 2\. 若为PCI设备，可以解绑驱动：

## \# 查看pci设备

lspci -k

## \# 解绑示例，替换你的pci地址

echo "0000:03:00.0" > /sys/bus/pci/drivers/xxx\_driver/unbind

不要随便用 rmmod -f 强制卸载，容易内核panic。

## 场景5：模块加载成功，但设备不工作

1\. lsmod 看到模块已经加载；

2\. lspci -k 看设备是否绑定到该驱动；

3\. 如果驱动加载了，但没有绑定设备，说明硬件识别异常，看dmesg。

# 第四步：内核taint污染标记（模块许可证）

cat /proc/sys/kernel/tainted

非GPL许可证的闭源驱动，会标记taint，不一定代表加载失败，只是告警。

部分内核配置会拒绝加载非GPL模块。

# 第五步：完整排查流水线

## 1\. uname -r 确认内核版本，确认ko编译版本匹配

## 2\. depmod -a 更新模块依赖

## 3\. modinfo 模块名 查看依赖、许可证

## 4\. modprobe 模块名 尝试加载

## 5\. dmesg -w 看报错日志

## 6\. lsmod | grep 模块名 确认是否加载成功

## 7\. lspci -k / lsusb -k 确认硬件与驱动绑定关系

## 8\. 卸载： modprobe -r 模块名

# 补充：模块的位置

\- 模块存放目录： /lib/modules/$(uname -r)/

\- 新增ko放到对应目录后，必须执行 depmod -a，modprobe才能识别。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/8098862f_1790727953487?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg2NDkwODQzNw%3D%3D%26mid%3D2247484412%26idx%3D1%26sn%3Dcadc31c3ee081dea6f567860dece97f3%26chksm%3Dcf3d7f0b88bd1b721ca645cba744b96ad61c6343e77fb53b17e80f3386fe19bb3fb9e656afec%26mpshare%3D1%26scene%3D1%26srcid%3D0930QWoLrcWKRgmJKZvMcfc5%26sharer_shareinfo%3Df116b00ad5d7c645cab22ed99cbdfae4%26sharer_shareinfo_first%3Df116b00ad5d7c645cab22ed99cbdfae4%23rd&s=obsidian)