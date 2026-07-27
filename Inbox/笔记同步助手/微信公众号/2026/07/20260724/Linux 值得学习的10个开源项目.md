---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247490558&idx=1&sn=6a1ce49560e9d52d89f518d70df7e301&chksm=c3c06476673b495d42da464709a5d6fd699b21a566b0dca8594d8d9770fbbecfe93badefea23&mpshare=1&scene=1&srcid=07243IL5miZrjnyLD3XhMLv8&sharer_shareinfo=c4792cdc17407c063b524b4d3a1e5beb&sharer_shareinfo_first=c4792cdc17407c063b524b4d3a1e5beb#rd
saved: 2026-07-24 20:45:38
tags:
  - 笔记同步助手
id: 522fca35-36f7-4f93-974f-0201076e183e
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-07-24 20:34

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d5673b61529048fc2392d6b1f5e4601a_MD5.gif]]

大家好，我是蟹老板～

很多人学习 Linux，最先接触的是命令，再往后，可能会写 Shell、部署服务、配置防火墙、制作 Docker 镜像。

这些当然都很重要。

但如果想真正理解 Linux，仅仅会使用命令还不够。

因为一个完整的 Linux 系统，背后至少包含：

-   • 内核如何管理进程、内存、文件和设备
    
-   • C 程序如何从 ELF 入口运行到 `main()`
    
-   • 根文件系统里的基础命令从哪里来
    
-   • 交叉编译工具链和系统镜像如何生成
    
-   • 芯片上电后如何一步步启动 Linux
    
-   • SSH 如何完成认证、加密和远程会话
    
-   • 网络数据包如何经过过滤、转发和 NAT
    
-   • 容器如何利用 Namespace 和 Cgroup 实现隔离
    
-   • Docker 如何管理镜像、容器和运行时
    
-   • PID 1 如何组织整个用户空间
    

**而深度学习开源项目，是建立 Linux 系统能力最快、最稳的一条路。**

你能从这些项目中学习到：

-   • 系统调用如何穿过用户态和内核态
    
-   • 用户空间基础设施如何组织
    
-   • 启动链路如何分层
    
-   • 配置系统如何驱动构建
    
-   • 网络协议和防火墙如何建模
    
-   • 容器运行时如何组合内核能力
    
-   • 服务管理如何抽象成依赖图
    

下面分享 10 个 Linux 方向非常值得深度学习的开源项目。

## 一、10个值得深度复刻的开源项目

### 1.1 Linux Kernel

Linux Kernel 是 Linux 操作系统的核心，管理着所有硬件、系统资源和基础服务。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b6f2ec1f76e136c7b43feeec6436a4bf_MD5.jpg]]

```
GitHub地址：https://github.com/torvalds/linux
```

> 开源协议：GPL-2.0

> GitHub星标：240 k+

> 复刻建议：不要整仓复刻，重点复刻“一个简单字符设备驱动 + procfs接口 + 内核模块编译框架”

**功能特性：**

-   • 进程调度、内存管理、文件系统、网络协议栈
    
-   • 设备驱动框架（字符设备、块设备、网络设备）
    
-   • 硬件抽象与架构支持
    
-   • 内核模块动态加载机制
    
-   • procfs、sysfs、debugfs 等内核接口
    
-   • 支持数十种硬件架构
    

**复刻能学到啥：**

Linux Kernel 最大的学习价值，不是“它又实现了一个操作系统”，而是它把硬件管理做成了一个完整的抽象体系。

传统裸机程序里，我们经常这样写：

```
#define LED_BASE 0x40021000
*(volatile uint32_t *)(LED_BASE + 0x14) |= (1 << 5);
```

换一个芯片，代码全部重写。

而 Linux 的思路是硬件不应该被业务代码直接操作，而应该通过设备驱动模型 + 文件接口统一管理。字符设备驱动核心就是：

```
struct file_operations {
    int (*open)(struct inode *, struct file *);
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    int (*release)(struct inode *, struct file *);
};
```

应用层通过 `open("/dev/myled", O_RDWR)` 操作硬件，驱动层负责具体寄存器读写。业务逻辑和硬件操作彻底分离。

**复刻目标：**

不要一开始看内核调度或内存管理。更建议先复刻一个迷你内核模块：

-   • 实现一个简单的字符设备
    
-   • 实现 `open` / `read` / `write` / `release` 回调
    
-   • 在 `/proc` 下创建一个接口
    
-   • 编写 Makefile 编译内核模块
    
-   • 用 `insmod` / `rmmod` 加载卸载
    

这就是 Linux“驱动工程化”的起点。

### 1.2 musl libc

musl 是一个 MIT 许可证的 C 标准库实现，面向 Linux syscall API，适合在各种部署环境中使用。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/e17b7d946d04a24fdad5a0a85297f353_MD5.jpg]]

```
地址：https://git.musl-libc.org/cgit/musl
```

> 开源协议：MIT

> GitHub星标：无GitHub地址

> 复刻建议：复刻一个简化版 libc：实现 `malloc`、`printf`、`strlen`、`memcpy` + 几个 syscall 封装

**功能特性：**

-   • 完整 C 标准库实现
    
-   • 轻量级代码，低运行时开销
    
-   • 高效静态和动态链接支持
    
-   • 静态链接生成的二进制远比 glibc 小
    
-   • 完整 `.a` 库仅 462KB（glibc 为 2MB）
    
-   • 强故障安全保证
    

**复刻能学到啥：**

很多嵌入式 Linux 开发者习惯了这样写：

```
printf("hello world\n");
malloc(1024);
```

但很少有人想过：`printf` 怎么把字符串送到终端？`malloc` 怎么从内核要内存？

musl 的价值在于：它把 C 库实现得极其精简和清晰。

核心流程是：

```
printf → vfprintf → write → syscall(SYS_write)
malloc → brk/mmap → syscall(SYS_brk)
```

你可以顺着一行代码从用户态追到内核态。

musl 的 `malloc` 实现非常值得学习——它不像 glibc 那样有复杂的 per-thread arena，而是用一套简单但正确的内存分配器。对于嵌入式场景，musl 的静态链接优势巨大：一个 `hello world` 静态链接 musl 只有几十 KB，而 glibc 静态链接动辄几 MB。

**复刻目标：**

建议实现一个 mini libc：

-   • 实现 `strlen`、`strcpy`、`memcpy`、`memset`
    
-   • 实现简化版 `printf`（支持 `%d`、`%s`、`%x`）
    
-   • 实现简化版 `malloc` / `free`（基于 brk）
    
-   • 封装几个 syscall（`write`、`read`、`brk`）
    
-   • 写一个测试程序，用你自己的 libc 编译运行
    

做完这个，你会真正理解“应用程序和内核之间隔着一层什么”。

### 1.3 BusyBox

BusyBox 是一个将数百个常见 Unix/Linux 命令整合到单个可执行文件中的轻量级工具集。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/da3ad9d8fcff0b597664547af3799d80_MD5.jpg]]

```
地址：https://busybox.net/
```

> 开源协议：GPL-2.0

> 复刻建议：复刻一个迷你 BusyBox：实现多调用机制 + 3～5 个常用命令（`ls`、`cat`、`echo`、`ps`）

**功能特性：**

-   • 将 ls、cat、grep、awk、sed、tar、wget 等 300+ 命令集成到一个二进制文件
    
-   • 体积通常只有 1MB 左右
    
-   • 提供约 400 个常用命令的精简实现
    
-   • 包含 shell 环境
    
-   • 广泛用于嵌入式系统、急救 shell、最小 Linux 发行版
    

**复刻能学到啥：**

很多嵌入式 Linux 系统的根文件系统里，/bin/sh 指向的就是 busybox。你家里那台跑 OpenWrt 的路由器、Alpine Linux 的 Docker 镜像，底层都是 BusyBox。

BusyBox 最值得学的不是“某个命令怎么实现”，而是它的**多调用机制（multicall）**。

普通程序是这样的：

```
int main(int argc, char **argv) {
    // 每个程序都有自己的 main
}
```

BusyBox 是这样的：

```
int main(int argc, char **argv) {
    applet = find_applet(argv[0]);  // 根据命令名找对应的函数
    applet->main(argc, argv);       // 执行对应命令
}
```

`ls` 和 `cat` 不是两个独立程序，而是同一个二进制里的两个函数。通过软链接（`/bin/ls -> /bin/busybox`），系统调用 `ls` 时实际执行的是 BusyBox，它根据 `argv[0]` 决定执行哪个命令。

这套机制的精髓在于：**用一张函数表驱动程序行为**。

```
struct applet {
    const char *name;
    int (*main)(int argc, char **argv);
};
```

**复刻目标：**

建议实现一个 mini BusyBox：

-   • 实现 `applet` 结构体和查找表
    
-   • 实现 `ls`（简化版：列出文件名）
    
-   • 实现 `cat`（简化版：打印文件内容）
    
-   • 实现 `echo`（打印参数）
    
-   • 实现多调用入口
    
-   • 用软链接测试
    

做完这个，你会理解为什么嵌入式 Linux 能在几 MB 的 Flash 里跑起来。

### 1.4 Buildroot

Buildroot 是一个简单、高效、易用的工具，通过交叉编译生成嵌入式 Linux 系统。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/405da2a07ff39ddae65579ad8097f3c8_MD5.jpg]]

```
官方网站：https://buildroot.org/
```

> 开源协议：GPL-2.0

> 复刻建议：不要复刻整个 Buildroot，重点复刻“Kconfig 配置系统 + 一个最小构建流水线”

**功能特性：**

-   • 自动化生成嵌入式 Linux 系统
    
-   • 集成内核、Bootloader、根文件系统的编译和打包
    
-   • 基于 Kconfig 和 Makefile 的配置系统
    
-   • 支持数百个软件包选择
    
-   • 无需 root 权限即可构建
    
-   • 为大量开发板提供基础配置
    

**复刻能学到啥：**

很多初学者构建嵌入式 Linux 是这样干的：

```
1. 下载内核源码 → 配置 → 编译
2. 下载 BusyBox 源码 → 配置 → 编译
3. 手动创建 /dev、/etc、/proc 等目录
4. 手动拷贝所有文件到根文件系统
5. 自己做文件系统镜像
```

第一次做还能忍，做第二次就开始崩溃。

Buildroot 的价值在于：**它把“构建系统”这件事自动化了**。

核心思想是：

```
menuconfig → .config → Makefile 递归构建 → output/images/
```

每个软件包都有一个 `.mk` 文件描述怎么下载、怎么配置、怎么编译、怎么安装。

```
# 一个简化的包描述
FOO_VERSION = 1.2.3
FOO_SOURCE = foo-$(FOO_VERSION).tar.gz
FOO_SITE = https://example.com/download
FOO_DEPENDENCIES = bar
define FOO_BUILD_CMDS
    $(MAKE) -C $(@D)
endef
```

这背后是一个非常重要的软件思想：**用声明式描述替代命令式脚本**。

**复刻目标：**

建议实现一个 mini Buildroot：

-   • 实现 Kconfig 风格的配置（用 `make menuconfig` 或简化版）
    
-   • 实现一个包描述框架（下载 → 解压 → 编译 → 安装）
    
-   • 构建一个包含 BusyBox + 内核的最小系统
    
-   • 生成可烧录的镜像文件
    

做完这个，你会对“构建系统”有全新的理解，而不是只会敲 `make`。

### 1.5 U-Boot

U-Boot 是嵌入式系统的引导加载程序，支持 PowerPC、ARM、MIPS 等多种处理器架构。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/527a3c848d7fa7754184308b99b8c95b_MD5.jpg]]

```
GitHub地址：https://github.com/u-boot/u-boot
```

> 开源协议：GPL-2.0+

> GitHub星标：-

> 复刻建议：复刻一个迷你 Bootloader：SPL 阶段 + 内存初始化 + 内核加载跳转

**功能特性：**

-   • 硬件初始化（CPU、DDR 内存、存储控制器）
    
-   • 从存储设备加载内核和设备树
    
-   • 交互式命令行
    
-   • 支持网络下载、镜像烧录
    
-   • 环境变量配置
    
-   • 支持多种文件系统和启动介质
    

**复刻能学到啥：**

U-Boot 不是简单的“启动加载程序”，它是一套轻量化的嵌入式固件系统。

很多开发者把 U-Boot 当成黑盒：上电 → 自动启动内核。但真正值得学的是：**从 CPU 上电到跳转到内核，中间发生了什么？**

简化版的启动流程是：

```
ROM Code → SPL (Secondary Program Loader) → U-Boot proper → Kernel
```

SPL 阶段在片内 SRAM 中运行，负责初始化 DDR 内存，然后把 U-Boot 从 Flash 加载到 DDR。U-Boot proper 在 DDR 中运行，负责加载内核和设备树，最后跳转。

更关键的是 **U-Boot 的重定位机制**：它可能先在一个地址运行，后面把自己搬到另一个地址继续跑。这种“自搬运”能力是理解链接脚本、内存布局的绝佳教材。

**复刻目标：**

建议实现一个 mini Bootloader：

-   • 实现汇编入口（设置栈、清零 BSS）
    
-   • 实现 C 环境初始化
    
-   • 实现简单串口驱动（用于打印）
    
-   • 实现从 Flash 读取内核镜像
    
-   • 实现跳转到内核入口
    

做完这个，你会真正理解“嵌入式系统是怎么活过来的”。

### 1.6 OpenSSH

OpenSSH 是 SSH 协议最流行、最广泛部署的开源实现，由 OpenBSD 项目开发。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/9634fb7291eaa958d425a02a622ade73_MD5.jpg]]

```
GitHub地址：https://github.com/openssh/openssh-portable
```

> 开源协议：BSD 风格

> GitHub星标：3.9 k

> 复刻建议：不要复刻完整 SSH，重点复刻“非对称认证流程 + 加密通道协商 + 会话管理”

**功能特性：**

-   • SSH 协议版本 2 的完整实现
    
-   • 客户端 `ssh` 和服务端 `sshd`
    
-   • 文件传输工具 `scp` 和 `sftp`
    
-   • 密钥生成 `ssh-keygen`
    
-   • 运行时密钥存储 `ssh-agent`
    
-   • 支持 PAM 等系统原生认证和审计
    

**复刻能学到啥：**

OpenSSH 最值得学的不是“怎么用 SSH 登录”，而是**安全通信协议的设计**。

传统的 telnet 是明文传输，密码和命令全部裸奔。OpenSSH 解决的是一整套问题：

-   • 如何在不安全的网络上安全地交换密钥？
    
-   • 如何验证对方身份（防止中间人攻击）？
    
-   • 如何加密双向通信？
    
-   • 如何管理会话和复用连接？
    

SSH 的握手流程大致是：

```
Client → Server: 协议版本协商
Client → Server: 密钥交换（Diffie-Hellman）
Server → Client: 主机密钥签名验证
Client → Server: 用户认证（密码/公钥）
Client ↔ Server: 加密会话建立
```

其中公钥认证的流程尤其精妙：客户端用私钥签名一个 challenge，服务端用公钥验证。这背后是**非对称加密 + 数字签名**的经典应用。

**复刻目标：**

建议实现一个 mini SSH 认证流程（在 localhost 上跑）：

-   • 实现一个简化版密钥交换
    
-   • 实现公钥/私钥生成和验证
    
-   • 实现 challenge-response 认证
    
-   • 实现会话密钥派生
    
-   • 用 AES 加密一条消息做验证
    

做完这个，你会真正理解“安全的远程登录”是怎么实现的。

### 1.7 iptables / nftables

iptables 是 Linux 内核包过滤系统的用户空间配置工具。nftables 是其新一代替代品，自 Linux 内核 3.13 起可用。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/5e51ca4705b4e8d09ea94bba6cec9182_MD5.jpg]]

```
项目地址：https://www.netfilter.org/
```

> 开源协议：GPL-2.0

> 复刻建议：复刻一个简化版规则引擎：“表 + 链 + 规则”数据结构 + 匹配执行

**功能特性：**

-   • 网络数据包过滤和分类
    
-   • 4 个表：filter、nat、mangle、raw
    
-   • 5 个链：PREROUTING、INPUT、FORWARD、OUTPUT、POSTROUTING
    
-   • 支持 NAT、包标记、流量控制
    
-   • 规则可增量变更
    
-   • nftables 比 iptables 执行更高效、存储更紧凑
    

**复刻能学到啥：**

iptables 最值得学的不是“怎么配防火墙规则”，而是**规则匹配引擎的设计**。

它的核心架构是：

```
表（功能维度）→ 链（钩子点）→ 规则（匹配 + 动作）
```

一个数据包进入网卡后，依次经过多个 Netfilter 钩子点。每个钩子点上挂着一组规则，规则按顺序匹配：

```
struct rule {
    struct match *matches;   // 匹配条件（IP、端口、协议等）
    struct target *target;   // 动作（ACCEPT、DROP、REJECT、DNAT 等）
    struct rule *next;
};

struct chain {
    char *name;
    struct rule *rules;
    struct chain *next;
};
```

数据包来了就遍历规则链表，匹配就执行动作，不匹配就继续。这就是典型的**策略模式 + 责任链模式**在 C 语言中的实现。

nftables 更进一步，引入了**集合（set）**和**映射（map）**，规则表达更灵活，存储更紧凑。

**复刻目标：**

建议实现一个 mini 防火墙引擎：

-   • 定义 `table` / `chain` / `rule` 数据结构
    
-   • 实现规则匹配（源 IP、目的 IP、端口）
    
-   • 实现动作执行（ACCEPT / DROP）
    
-   • 在用户态模拟数据包处理流程
    
-   • 实现一个简化版规则解析器
    

做完这个，你会对“网络数据包处理”有本质理解，而不是只会背 iptables 命令。

### 1.8 runc

runc 是一个符合 OCI 容器运行时规范的命令行工具，用于在 Linux 上创建和运行容器。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d3eb66386ea52cc56f130db9a91d5439_MD5.jpg]]

```
GitHub地址：https://github.com/opencontainers/runc
```

> 开源协议：Apache-2.0

> GitHub星标：13.4k+

> 复刻建议：复刻一个迷你容器运行时：namespace 隔离 + cgroup 限制 + rootfs 切换

**功能特性：**

-   • OCI 运行时规范的参考实现
    
-   • CLI 工具，可在 Linux 上生成和运行容器
    
-   • 设计上尽可能精简
    
-   • 是 Docker 等容器引擎的底层工作组件
    
-   • 支持 cgroups、namespace、rootfs
    
-   • 支持 seccomp 等安全特性
    

**复刻能学到啥：**

很多人用 Docker 多年，以为容器是“轻量级虚拟机”。实际上，容器就是一组 Linux 内核特性的组合：

-   -   **Namespace**：隔离视图（PID、NET、IPC、MNT、UTS、USER）
-   -   **Cgroup**：限制资源（CPU、内存、IO）
-   -   **chroot / pivot\_root**：切换根文件系统

runc 的核心就做三件事：

```
1. 创建 namespace（clone 时指定 CLONE_NEW* 标志）
2. 配置 cgroup（写入 /sys/fs/cgroup/ 下的文件）
3. 切换 rootfs（pivot_root 到容器的文件系统）
```

简化版的容器启动流程：

```
clone(child_func, stack, CLONE_NEWPID | CLONE_NEWNET | SIGCHLD, NULL);
// 在子进程中：
set_cgroup_limits();
pivot_root("/path/to/rootfs");
execve("/bin/sh", ...);
```

这就是容器的本质——**不是虚拟化，是隔离**。

**复刻目标：**

建议实现一个 mini runc：

-   • 用 `clone()` 创建新进程，带 PID namespace
    
-   • 用 `unshare()` 隔离网络 namespace
    
-   • 用 `pivot_root()` 切换根文件系统（准备一个最小 rootfs）
    
-   • 用 cgroup 限制 CPU 和内存
    
-   • 最后在容器内执行 `/bin/sh`
    

做完这个，Docker 对你来说不再是黑盒。

### 1.9 Docker (Moby)

Moby 是 Docker 创建的开源项目，提供容器化所需的“乐高积木”式组件集。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d3cff9bfbcb00b317d46bc8da7301313_MD5.jpg]]

```
GitHub地址：https://github.com/moby/moby
```

> 开源协议：Apache-2.0

> GitHub星标：71.9k+

> 复刻建议：不要复刻整仓，重点复刻“容器生命周期管理 + 镜像分层存储 + 客户端-服务端架构”

**功能特性：**

-   • 容器构建工具、容器运行时、容器镜像仓库等组件
    
-   • 模块化架构，组件可替换
    
-   • 镜像分层存储（UnionFS）
    
-   • 客户端-服务端（C/S）架构
    
-   • 跨平台支持
    
-   • 安全默认配置
    

**复刻能学到啥：**

Docker 最值得学的不是 `docker run`，而是**镜像分层 + 容器生命周期管理**。

镜像分层是 Docker 的核心创新：每一层是一个只读文件系统，多个层通过 UnionFS 叠加成完整视图。

```
FROM alpine:3.19          # 层1: 基础镜像
RUN apk add python3       # 层2: 安装 Python
COPY app.py /app/         # 层3: 拷贝应用
```

每一层只记录变化，多个容器共享同一基础层，极大节省存储空间。

Docker 的架构是典型的 C/S 模式：

```
docker CLI → Docker daemon (dockerd) → containerd → runc → 容器进程
```

你敲的每一个 `docker` 命令，都经过这条链。

更重要的是理解**容器生命周期状态机**：

```
created → running → paused → stopped → deleted
```

每个状态转换都有对应的操作和清理逻辑。

**复刻目标：**

建议实现一个 mini Docker：

-   • 实现 C/S 架构（一个 CLI 发命令，一个 daemon 执行）
    
-   • 实现镜像分层（用 overlayfs 叠加多个目录）
    
-   • 实现容器生命周期（create → start → stop → delete）
    
-   • 底层调用你自己的 mini runc 来运行容器
    
-   • 实现 `docker ps` 查看运行中的容器
    

做完这个，你会对“容器平台”有完整的理解。

### 1.10 systemd

systemd 是 Linux 系统的系统和服务管理器。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/a4626caee2f39cb0db6bf48bc8f4b0c0_MD5.jpg]]

```
GitHub地址：https://github.com/systemd/systemd
```

> 开源协议：LGPL-2.1+ / GPL-2.0

> GitHub星标：16.5 k+

> 复刻建议：不要整仓复刻，重点复刻“unit 文件解析 + 依赖图构建 + 并行启动调度”

**功能特性：**

-   • 系统和服务管理
    
-   • Unit 文件配置（service、socket、timer、mount 等）
    
-   • 依赖关系管理和并行启动
    
-   • 套接字激活（socket activation）
    
-   • 日志管理（journald）
    
-   • 系统状态快照和恢复
    

**复刻能学到啥：**

传统 Linux init 系统（SysVinit）靠一堆 shell 脚本按顺序启动。脚本之间靠编号决定顺序：`S01xxx` 先于 `S02xxx`。系统启动慢、依赖管理弱、出问题难调试。

systemd 的价值在于：**把“启动服务”从脚本升级为声明式配置 + 依赖图调度**。

核心概念是 Unit：

```
[Unit]
Description=My Web Server
After=network.target

[Service]
ExecStart=/usr/bin/my-server
Restart=always

[Install]
WantedBy=multi-user.target
```

systemd 解析所有 Unit 文件，构建依赖图，然后**并行启动**没有依赖关系的服务。

背后是一套**拓扑排序**算法：

```
// 简化理解
build_dependency_graph();
topological_sort();
for each unit in sorted_order:
    if unit.dependencies_ready():
        start_unit(unit);
```

socket activation 更是精妙：服务还没启动，systemd 先监听端口；第一个请求到达时再启动服务。启动延迟被“隐藏”了。

**复刻目标：**

建议实现一个 mini systemd：

-   • 解析简化版 Unit 文件（`[Unit]` + `[Service]`）
    
-   • 构建依赖图
    
-   • 实现拓扑排序
    
-   • 并行启动无依赖的服务
    
-   • 实现服务状态监控（running / failed / exited）
    

做完这个，你会真正理解“现代 Linux 系统是怎么启动的”。

## 二、推荐学习顺序

这 10 个项目不建议按照文章顺序从头啃到尾。

可以按这个顺序来：先建立用户空间和系统镜像概念，再深入内核、网络和容器。

| 阶段 | 项目 | 核心收获 |
| :-- | :-- | :-- |
| 入门 | BusyBox | 多调用程序、命令分发、公共代码复用 |
| 入门 | musl libc | 程序启动、系统调用、C 运行时 |
| 进阶 | Buildroot | 交叉编译、构建依赖、Rootfs 生成 |
| 进阶 | U-Boot | 上电启动、SPL、镜像加载、设备树 |
| 深入 | Linux Kernel | 系统调用、VFS、驱动、内核对象模型 |
| 深入 | OpenSSH | 安全协议、状态机、认证、多路复用 |
| 深入 | iptables/nftables | Netfilter、规则引擎、连接跟踪、NAT |
| 高级 | systemd | Unit 模型、依赖图、进程与资源管理 |
| 高级 | runc | Namespace、Cgroup、Rootfs、OCI |
| 高级 | Docker | 镜像系统、容器控制面、运行时编排 |

我的建议是：**先用户空间，后内核空间；先单机系统，后网络和容器。**

不要一上来就啃 Linux Kernel 整仓，也不要一上来就研究 systemd 全部 Unit 类型。真正有效的方式是拆出最小闭环。

具体来说：

-   • 学 BusyBox，是理解 Linux 基础命令怎么组织
    
-   • 学 musl，是理解程序怎么进入和离开内核
    
-   • 学 Buildroot，是理解 Linux 系统怎么被构建出来
    
-   • 学 U-Boot，是理解 Linux 系统怎么被启动起来
    
-   • 学 Kernel，是理解操作系统能力从哪里来
    
-   • 学 OpenSSH，是理解安全远程协议怎么设计
    
-   • 学 nftables，是理解数据包在内核里怎么流动
    
-   • 学 systemd，是理解用户空间服务怎么组织
    
-   • 学 runc，是理解容器怎么被真正创建
    
-   • 学 Docker，是理解容器能力怎么被产品化
    

## 三、复刻方法论

Linux 项目的源码规模通常非常大。

如果只是打开源码目录，从第一个文件开始读，很容易陷入两个问题：

-   • 每个函数都认识，但不知道整体在做什么
    
-   • 看了很多分支，却没有跑通一条完整流程
    

更有效的方式，是围绕一个可观测的最小闭环展开。

### 1\. 先把项目跑起来

不要一开始就读源码。

先通过官方文档完成：

-   • 编译
    
-   • 安装
    
-   • 启动
    
-   • 运行一个最小示例
    
-   • 观察正常输出
    

Linux 项目推荐优先使用：

-   • QEMU
    
-   • 虚拟机
    
-   • 容器
    
-   • 独立测试机
    

例如：

-   • Linux Kernel 使用 QEMU 启动
    
-   • Buildroot 生成 QEMU 镜像
    
-   • U-Boot 使用 Sandbox 或 QEMU
    
-   • BusyBox 放进 Initramfs
    
-   • runc 放进虚拟机测试
    
-   • nftables 在 Network Namespace 中测试
    

### 2\. 只追一条主线

不要试图一次看懂所有功能。

例如研究 Docker 时，只追踪：

```
docker run
   ↓
API 请求
   ↓
Daemon 创建容器
   ↓
containerd
   ↓
runc
   ↓
clone
   ↓
execve
```

研究 systemd 时，只追踪：

```
systemctl start
   ↓
加载 Unit
   ↓
生成 Job
   ↓
依赖处理
   ↓
fork/exec
   ↓
状态更新
```

研究 Kernel 字符设备时，只追踪：

```
用户 read()
   ↓
系统调用
   ↓
VFS
   ↓
驱动 read
   ↓
copy_to_user
```

先把一条主线走通，再研究异常分支。

### 3\. 同时观察用户态和内核态

Linux 学习最大的特点，是很多功能横跨用户态和内核态。

建议组合使用：

-   -   `strace`：观察系统调用
-   -   `ltrace`：观察动态库调用
-   -   `readelf`：分析 ELF
-   -   `objdump`：查看反汇编和段信息
-   -   `gdb`：调试用户态程序
-   -   `gdbserver`：远程调试
-   -   `perf`：分析性能和调用栈
-   -   `ftrace`：跟踪内核函数
-   -   `bpftrace`：动态观察内核事件
-   -   `tcpdump`：查看网络数据包
-   -   `ip netns`：构建隔离网络实验

不要只看代码。

**让系统把真实执行路径展示出来。**

### 4\. 画四种图

研究一个 Linux 项目时，建议至少画出四类图。

#### 模块图

回答：

> 项目由哪些模块构成？

#### 时序图

回答：

> 一次请求从哪里开始，经过哪些组件？

#### 状态机图

回答：

> 对象有哪些状态，什么事件会改变状态？

#### 数据结构关系图

回答：

> 核心结构体之间如何引用？

例如研究 runc，可以画：

```
OCI Config
   ↓
Container
   ↓
Process
   ↓
Namespace
   ↓
Cgroup
   ↓
Rootfs
```

研究 systemd，可以画：

```
Manager
   ↓
Unit
   ↓
Job
   ↓
Dependency
   ↓
Process
   ↓
Cgroup
```

画图的目的不是好看，而是把源码中的隐含关系显式化。

### 5\. 从测试代码理解边界

成熟项目的测试代码非常有价值。

因为测试通常会直接告诉你：

-   • 正常输入是什么
    
-   • 错误输入是什么
    
-   • 模块边界在哪里
    
-   • 哪些行为必须保持兼容
    
-   • 哪些异常场景曾经出现过问题
    

阅读一个复杂函数之前，可以先搜索：

```
这个函数在哪里被测试？
这个数据结构有哪些构造样例？
这个错误码在什么情况下出现？
```

很多时候，测试比注释更接近真实设计意图。

### 6\. 删除复杂功能，保留核心机制

复刻不是抄写。

真正有效的复刻应该主动删除：

-   • 平台兼容代码
    
-   • 历史兼容代码
    
-   • 大量错误分支
    
-   • 性能优化
    
-   • 冷门功能
    
-   • 完整安全机制
    
-   • 复杂配置语法
    

只保留：

```
核心数据结构
    +
主状态机
    +
最小接口
    +
一个可运行案例
    +
一个异常恢复案例
```

比如：

-   • 复刻 Kernel，不是复刻所有驱动，而是接通一次系统调用
    
-   • 复刻 OpenSSH，不是自己写密码算法，而是复刻协议状态机
    
-   • 复刻 nftables，不是实现所有表达式，而是复刻规则执行模型
    
-   • 复刻 Docker，不是做完整平台，而是接通镜像到进程的生命周期
    
-   • 复刻 systemd，不是实现所有 Unit，而是复刻依赖图和服务状态
    

### 7\. 给自己的复刻版本加入故障

系统工程能力，很多时候不是从正常流程中练出来的。

需要主动模拟：

-   • 文件不存在
    
-   • 内存不足
    
-   • 子进程异常退出
    
-   • 网络断开
    
-   • 数据包乱序
    
-   • 镜像校验失败
    
-   • 服务依赖失败
    
-   • 启动过程中断电
    
-   • 容器 Rootfs 挂载失败
    
-   • Cgroup 配置失败
    

然后观察：

-   • 状态是否一致
    
-   • 资源是否释放
    
-   • 错误是否可定位
    
-   • 系统是否可以重试
    
-   • 重启后能否恢复
    

**正常流程决定功能能不能跑，异常流程决定系统能不能用。**

## 四、总结

学习优秀开源项目，不是为了重复造轮子。

而是为了理解：

> **一个系统级轮子，需要经过怎样的抽象、分层、状态管理和异常处理，才能稳定地运转。**

当你能够把这些项目的核心机制拆出来，用自己的代码重新实现一个最小闭环时，你就不再只是“会使用 Linux”。

你开始真正具备：

**理解 Linux、调试 Linux，以及设计 Linux 系统软件的能力。**

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/2bcfbd6e_1784897137401?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247490558%26idx%3D1%26sn%3D6a1ce49560e9d52d89f518d70df7e301%26chksm%3Dc3c06476673b495d42da464709a5d6fd699b21a566b0dca8594d8d9770fbbecfe93badefea23%26mpshare%3D1%26scene%3D1%26srcid%3D07243IL5miZrjnyLD3XhMLv8%26sharer_shareinfo%3Dc4792cdc17407c063b524b4d3a1e5beb%26sharer_shareinfo_first%3Dc4792cdc17407c063b524b4d3a1e5beb%23rd&s=obsidian)