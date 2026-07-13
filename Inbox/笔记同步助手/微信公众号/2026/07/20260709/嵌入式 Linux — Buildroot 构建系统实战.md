---
author: chenxi
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkxMjM0MzMwNQ==&mid=2247483712&idx=1&sn=2ff07c1a80cc2421212f682041c3f08e&chksm=c0efade7b8a9bed1a3a2a10d1cd52923120d859283351ec3bbdfe9569fa8b377e88a29ce40b6&mpshare=1&scene=1&srcid=0709GOsrsB6JURoX3wXIVOLT&sharer_shareinfo=f71c776afc37d7d05170ca660074f08c&sharer_shareinfo_first=f71c776afc37d7d05170ca660074f08c#rd
saved: 2026-07-09 11:10:10
tags:
  - 笔记同步助手
id: 0caf4943-e6db-4354-8de0-5612c2425bc9
---

公众号名称：chenxi 的笔记

作者名称：chenxi

发布时间：2026-06-05 22:06

> 板块：嵌入式 Linux

---

从零搭一套嵌入式 Linux 系统，最麻烦的往往不是写驱动，而是把 Bootloader、内核、设备树、根文件系统这几件事凑在一起，让板子真的能跑起来。手动一个个编译不是不行，但光是工具链版本、库依赖、各组件之间的兼容性，就很折腾。

Buildroot 干的就是这件事。给它一份配置，工具链、内核、BusyBox、第三方库、根文件系统，全部帮你编出来，最后打成能直接烧的镜像。折腾配置的时间比手动编译少很多，踩坑也集中得多。

---

## 一、先搞清楚 output/ 里有什么

很多人第一次用 Buildroot，编完之后对着 `output/` 目录发呆——这么多文件夹，到底烧哪个？先把结构搞清楚：\[¹\]

```
buildroot/
└── output/
    ├── images/       ← 最终产物，这里的东西才是要烧到板子上的
    │   ├── zImage
    │   ├── imx6ull-14x14-evk.dtb
    │   ├── rootfs.ext4
    │   ├── rootfs.tar      # 给 NFS 启动用
    │   └── sdcard.img      # 完整 SD 卡镜像，含分区表和 U-Boot
    │
    ├── build/        ← 每个包各自的编译目录
    │   ├── linux-4.19.232/
    │   ├── busybox-1.31.1/
    │   └── ...
    │
    ├── host/         ← 交叉工具链 + 目标架构的 sysroot
    │   ├── bin/      # arm-linux-gnueabihf-gcc 等工具在这里
    │   └── arm-buildroot-linux-gnueabihf/sysroot/
    │
    ├── staging/      ← host/sysroot 的符号链接，历史遗留
    │
    └── target/       ← 根文件系统骨架，但不能直接用
```

`staging/` 现在只是指向 `host//sysroot/` 的符号链接，Buildroot 早期它是独立存在的，后来合并进了 `host/`，留着这个链接是为了向后兼容。编译依赖某个库的应用时，头文件和 `.so` 都在这里找。

`target/` 看起来像根文件系统，实际上不能直接用。Buildroot 不以 root 运行，所以 `/dev` 下没有设备节点，文件权限也不完整（比如 BusyBox 的 setuid 位）。要用根文件系统，得用 `images/` 里打包好的镜像，不要直接挂 `target/`。\[¹\]

`sdcard.img` 不是 Buildroot 天然一定会生成的文件。Buildroot 能生成内核镜像、设备树、Bootloader 和根文件系统镜像；如果板级配置里额外配置了 `genimage`、`post-image` 脚本和分区布局，才会把这些产物进一步组织成完整的 SD 卡镜像，例如 `sdcard.img`。

所以判断"烧哪个文件"不能只看目录名，要看当前 defconfig 到底生成了什么：有 `sdcard.img` 就可以整卡烧录；只有 `zImage`、`.dtb`、`rootfs.ext4` 时，就需要手动分区烧录，或者补一套镜像打包脚本。

> **\[¹\]** 参考：Buildroot 用户手册 "Output directory structure"，https://buildroot.org/downloads/manual/manual.html

---

## 二、从 defconfig 开始

Buildroot 内置了很多板子的 defconfig，先搜一下有没有现成的：

```
make list-defconfigs | grep imx6
```

这里有两种情况：

**用板卡厂商的 SDK**（比如 100ask 的 `100ask_imx6ull-sdk`），厂商在 Buildroot 里预置了自己的 defconfig，`make list-defconfigs` 里能直接看到：

```
100ask_imx6ull_mini_ddr512m_systemV_core_defconfig
100ask_imx6ull_pro_ddr512m_systemV_core_defconfig
100ask_imx6ull_pro_ddr512m_systemV_qt5_defconfig
```

直接用厂商的 defconfig 是更稳妥的起点，它已经针对具体板子的内存大小、外设配置、分区方案做过适配：

```
make 100ask_imx6ull_pro_ddr512m_systemV_core_defconfig
```

**用主线 Buildroot 或厂商 SDK 里找不到对应板子的 defconfig 时**，可以看看有没有相近的作为起点。以 iMX6 系列为例，100ask 的 SDK（基于 Buildroot 2020.02.x）里能搜到 `imx6ulevk_defconfig`（iMX6UL，单 L），但没有专门针对 iMX6ULL（双 L）的主线 defconfig——这两款芯片名字只差一个字母，很容易混淆。iMX6ULL 建议直接用厂商 SDK，或者在 `imx6ulevk_defconfig` 基础上按自己的板子修改。\[²\]

加载 defconfig 之后会生成 `.config`，后续所有调整都在这个文件上进行，defconfig 本身不再变动——除非你主动执行 `make savedefconfig` 把当前配置同步回去。

> **\[²\]** 参考：Buildroot 官方仓库，https://gitlab.com/buildroot.org/buildroot；100ask iMX6ULL SDK，https://github.com/100askTeam/100ask\_imx6ull-sdk

---

## 三、提前准备好内核和 U-Boot 源码

Buildroot 默认会按配置去下载内核和 U-Boot 源码。网络慢、源码仓库访问不稳定，或者厂商 SDK 版本比较老时，这一步很容易卡住。更实用的做法是：**在 Buildroot 同级目录提前准备好内核源码、U-Boot 源码和板级定制目录**，然后让 Buildroot 走本地路径。

比如开发目录可以这样放：

```
workspace/
├── buildroot/                  ← Buildroot 主目录
├── linux-4.9.88/               ← 提前准备好的内核源码
├── linux-4.19.232/             ← 也可以放其它版本，按项目选择
├── u-boot-2016.03/             ← U-Boot 源码
├── u-boot-2020.04/             ← 其它 U-Boot 版本
└── my-project/                 ← BR2_EXTERNAL 板级配置，后面会讲
```

这里的重点不是必须叫 `linux-4.9.88` 或 `linux-4.19.232`，而是让源码目录和 `buildroot/` 保持并列。这样配置路径时可以统一写成：

```
$(TOPDIR)/../linux-4.9.88
$(TOPDIR)/../u-boot-2016.03
```

`$(TOPDIR)` 代表 Buildroot 根目录，所以 `$(TOPDIR)/../linux-4.9.88` 就是 Buildroot 同级目录下的内核源码。这样做比写死 `/home/xxx/workspace/linux-4.9.88` 更方便打包，不依赖绝对路径。

## 方法一：把源码压缩包放进 dl/ 目录

Buildroot 下载的源码包会缓存在 `dl/` 目录。如果只是想避免重复下载，可以把提前下载好的压缩包放进去：

```
# 先看 Buildroot 已经下载过哪些包
ls buildroot/dl/

# 把提前准备好的源码包放进去，文件名要和 Buildroot 期望的一致
cp linux-4.9.88.tar.xz buildroot/dl/
cp u-boot-2016.03.tar.bz2 buildroot/dl/
```

这种方式适合"源码不怎么改，只是避免下载"的场景。文件名得完全一致，包括版本号和压缩格式。不确定文件名时，可以先跑一次 `make`，从报错里的下载 URL 反推出文件名。

## 方法二：在配置里指定本地源码路径（仅适用于部分厂商 SDK）

某些厂商 Buildroot SDK，特别是基于较老版本的，会保留 `Custom local directory` 这类选项。它的思路是直接指定本地源码目录：

```
BR2_LINUX_KERNEL_CUSTOM_LOCAL=y
BR2_LINUX_KERNEL_CUSTOM_LOCAL_PATH="$(TOPDIR)/../linux-4.9.88"
```

U-Boot 类似：

```
BR2_TARGET_UBOOT_CUSTOM_LOCAL=y
BR2_TARGET_UBOOT_CUSTOM_LOCAL_PATH="$(TOPDIR)/../u-boot-2016.03"
```

需要说清楚：**`BR2_LINUX_KERNEL_CUSTOM_LOCAL` 这个选项从来不在主线 Buildroot 里，是部分厂商 SDK 自行加入的私有机制。** 如果你用的是 100ask 等厂商 SDK，可以先在 `make menuconfig` 里搜索 `CUSTOM_LOCAL`，能搜到才能用；如果搜不到，直接跳到方法三。

## 方法三：用 override-srcdir 指向本地源码，适合改内核/驱动时用

这是主线 Buildroot 推荐的方式，也是厂商 SDK 新版本通常支持的方式。先建一个本地覆盖配置文件：

```
mkdir -p board/myboard
vim board/myboard/local.mk
```

写入：

```
LINUX_OVERRIDE_SRCDIR = $(TOPDIR)/../linux-4.9.88
UBOOT_OVERRIDE_SRCDIR = $(TOPDIR)/../u-boot-2016.03
```

然后在 Buildroot 配置里指定这个文件：

```
BR2_PACKAGE_OVERRIDE_FILE="board/myboard/local.mk"
```

或者放到 `BR2_EXTERNAL` 目录里：

```
BR2_PACKAGE_OVERRIDE_FILE="$(BR2_EXTERNAL_MY_PROJECT_PATH)/board/myboard/local.mk"
```

这种方式有两个好处：

1.  1\. 内核源码可以提前放在 `buildroot/` 同级目录，比如 `linux-4.9.88/`、`linux-4.19.232/`。
    
2.  2\. 源码目录可以继续作为独立 git 仓库维护，不需要塞进 Buildroot 源码树。
    

实际开发时，可以按这个节奏走：

```
# 改了内核源码后，重新同步/编译内核
make linux-rebuild

# 如果发现改动没有被重新同步，直接清掉内核构建目录后再编
make linux-dirclean && make linux
```

不要把本地源码理解成"Buildroot 永远直接在原目录里编译"。Buildroot 通常会把源码 rsync 到 `output/build/` 下再构建，所以当你频繁改源码时，`override-srcdir` 比单纯放 `dl/` 压缩包更适合。

## 方法四：100ask SDK 的处理方式

100ask 的 SDK 里，内核和 U-Boot 源码通常已经打包在 SDK 中，或者通过脚本单独下载。很多 defconfig 已经配好了对应路径，加载厂商 defconfig 后可以先直接编译，不要急着改路径。

如果编译报错显示找不到内核源码，再回头检查 `.config` 里这些项：

```
grep -E "LINUX_KERNEL|UBOOT|OVERRIDE" .config
```

确认它到底是从网络下载、从 `dl/` 取压缩包，还是引用了 Buildroot 同级目录下的本地源码。

---

## 四、menuconfig：真正需要动的那些选项

```
make menuconfig
```

界面和内核的 menuconfig 完全一样，方向键导航，空格选中取消，`/` 搜索，`?` 看帮助。重点关注这几块：

**Target options**：确认目标架构。iMX6ULL 是 Cortex-A7，架构选 ARM，ABI 选 EABIhf，浮点选 NEON/VFPv4。选错了二进制能跑起来，但浮点运算会走软浮点模拟，性能差很多，或者直接 Illegal instruction 崩掉。

**Toolchain**：工具链来源。两个选择：用 Buildroot 自带的 crosstool-NG 自动构建（第一次要花不少时间），或者指定外部预编译工具链（比如 Linaro 的）。用外部工具链能省掉首次编译工具链那 15～30 分钟，但版本要和内核匹配。

**System configuration**：`Init system` 选 BusyBox（对应上一篇讲的 rcS 启动方式）；`getty options` 里串口和波特率要和板子一致，iMX6ULL 通常是 `ttymxc0`，115200；`Root password` 留空的话登录直接回车进去，调试方便但生产环境别这么用。

**Kernel**：如果用本地源码，部分厂商 SDK 里能找到 `Custom local directory` 选项；主线 Buildroot 或较新的 SDK 通过 `BR2_PACKAGE_OVERRIDE_FILE` 配 `LINUX_OVERRIDE_SRCDIR`。设备树文件名填内核源码里 DTS 的名称，不带 `.dts` 后缀，比如 `imx6ull-14x14-evk`。

**Target packages**：选需要装进根文件系统的软件包。dropbear 是轻量级 SSH，调试必装；i2c-tools、can-utils 按实际需要选。每多选一个包编译时间都会增加，按需勾选。

**Filesystem images**：`ext4` 适合需要读写的分区，`squashfs` 只读但压缩率高。同时勾上 `tar` 格式，后面配 NFS 启动会用到。

**Bootloaders**：如果用本地 U-Boot 源码，部分厂商 SDK 可以选 `Custom local directory`；主线 Buildroot 或较新版本用 `UBOOT_OVERRIDE_SRCDIR` 指向 Buildroot 同级目录下提前准备好的 U-Boot 源码。

---

## 五、保存配置，别让改动丢失

`make menuconfig` 改了一堆，记得 `savedefconfig`

```
# 把当前 .config 精简后存成 defconfig（只保留和默认值不同的项）
make savedefconfig

# 默认存到 buildroot 根目录的 defconfig 文件
# 指定路径：
make savedefconfig BR2_DEFCONFIG=configs/myboard_defconfig
```

内核配置独立处理：

```
make linux-menuconfig          # 改内核配置
make linux-update-defconfig    # 存回 BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE 指定的路径
```

BusyBox 同理：`make busybox-menuconfig` 改，`make busybox-update-config` 存。

---

## 六、编译

```
make -j$(nproc)
```

第一次全量编译（包含工具链），通常 1～3 小时，主要看 CPU 核心数。如果内核和 U-Boot 已经提前放在 Buildroot 同级目录，并通过本地路径或 `override-srcdir` 引用，可以省掉不少下载和解压等待时间。

编译中途挂掉，先看最后的报错是哪个包：

```
make 2>&1 | tail -50

# 单独重新编内核（不清源码目录，只重跑 make 步骤）
make linux-rebuild

# 彻底清掉内核编译目录重来
make linux-dirclean && make linux
```

网络问题导致某个包下载失败，把源码压缩包手动放到 `dl/` 目录，Buildroot 检测到就跳过下载。文件名要和 Buildroot 期望的一致，如果还不确定就先跑一次 `make` 看报错里的 URL，从 URL 里拿文件名。

\---\`\`\`\`

## 七、post-image 和 genimage：把编译产物组织成可烧录镜像

**Buildroot 负责编译各个组件，但不一定自动帮你拼成完整的 SD 卡镜像。**

如果你的配置只生成这些文件：

```
output/images/
├── zImage
├── imx6ull-14x14-evk.dtb
├── rootfs.ext4
└── rootfs.tar
```

这说明 Buildroot 已经把内核、设备树和根文件系统编出来了，但还没有把它们组织成一张带分区表的卡镜像。这个时候有两种做法：

1.  1\. 手动给 SD 卡分区，然后分别烧 U-Boot、内核、设备树和 rootfs。
    
2.  2\. 写 `post-image.sh` + `genimage.cfg`，让 Buildroot 在编译结束后自动生成 `sdcard.img`。
    

实际项目里更推荐第二种，因为它能把"分区大小、U-Boot 偏移、boot 分区内容、rootfs 分区"固化下来，别人拿到工程后直接 `make` 就能得到同样的镜像。

### 1\. 在配置里启用 post-image 脚本

在 defconfig 里加上类似配置：

```
BR2_PACKAGE_HOST_GENIMAGE=y
BR2_ROOTFS_POST_IMAGE_SCRIPT="$(BR2_EXTERNAL_MY_PROJECT_PATH)/board/myboard/post-image.sh"
```

如果你没有用 `BR2_EXTERNAL`，也可以写成 Buildroot 源码树内的相对路径：

```
BR2_ROOTFS_POST_IMAGE_SCRIPT="board/myboard/post-image.sh"
```

`post-image` 脚本是在文件系统镜像、内核镜像、Bootloader 镜像都生成之后执行的。Buildroot 会把 `output/images` 的路径作为第一个参数传给脚本，同时也会导出 `BINARIES_DIR`、`TARGET_DIR`、`BUILD_DIR` 等环境变量。

### 2\. 编写 post-image.sh

示例：

```
#!/bin/sh
set -e

BOARD_DIR="$(dirname "$0")"
GENIMAGE_CFG="${BOARD_DIR}/genimage.cfg"
GENIMAGE_TMP="${BUILD_DIR}/genimage.tmp"

rm -rf "${GENIMAGE_TMP}"

genimage \
    --rootpath "${TARGET_DIR}" \
    --tmppath "${GENIMAGE_TMP}" \
    --inputpath "${BINARIES_DIR}" \
    --outputpath "${BINARIES_DIR}" \
    --config "${GENIMAGE_CFG}"
```

给脚本加执行权限：

```
chmod +x board/myboard/post-image.sh
```

### 3\. 编写 genimage.cfg

下面是一个简化示例，假设 boot 分区放 `zImage` 和 `.dtb`，rootfs 使用 `rootfs.ext4`，U-Boot 镜像文件叫 `u-boot.imx`：

```
image boot.vfat {
    vfat {
        files = {
            "zImage",
            "imx6ull-14x14-evk.dtb"
        }
    }
    size = 32M
}

image sdcard.img {
    hdimage {
    }

    partition u-boot {
        in-partition-table = "no"
        image = "u-boot.imx"
        offset = 1K
    }

    partition boot {
        partition-type = 0xC
        bootable = "true"
        image = "boot.vfat"
    }

    partition rootfs {
        partition-type = 0x83
        image = "rootfs.ext4"
        size = 512M
    }
}
```

这份 `genimage.cfg` 只能作为模板。不同芯片、不同启动 ROM、不同板卡厂商对 U-Boot 偏移、镜像名、分区布局可能不一样，比如有的板子用 `u-boot.imx`，有的用 `flash.bin`，有的还要单独放 SPL。按自己的板卡启动要求修改。

最终生成后，目录会变成：

```
output/images/
├── zImage
├── imx6ull-14x14-evk.dtb
├── rootfs.ext4
├── boot.vfat
└── sdcard.img
```

到这一步，`sdcard.img` 才是可以直接整卡烧录的完整镜像。

---

## 八、烧录

```
lsblk  # 先确认 SD 卡设备名

sudo dd if=output/images/sdcard.img of=/dev/sdX bs=4M status=progress
sudo sync
```

烧完插板子上，串口 115200 8N1，上电看 U-Boot 日志，然后内核，最后登录提示。

有些 defconfig 不生成 `sdcard.img`，只有 `rootfs.ext4` 和 `zImage`，这时要手动分区后分别烧，或者自己写 genimage 配置。

---

## 九、rootfs overlay：加文件最干净的方式

想往根文件系统里加文件——自定义 inittab、开机脚本、应用程序——用 overlay，不用改 Buildroot 任何代码。\[³\]

按根文件系统的目录结构建 overlay 目录，放对应文件：

```
mkdir -p board/myboard/rootfs-overlay/etc/init.d
mkdir -p board/myboard/rootfs-overlay/usr/bin

cp S99myapp board/myboard/rootfs-overlay/etc/init.d/
chmod +x board/myboard/rootfs-overlay/etc/init.d/S99myapp
cp myapp board/myboard/rootfs-overlay/usr/bin/
```

在 menuconfig 的 `System configuration → Root filesystem overlay directories` 填路径，或直接加进 `.config`：

```
BR2_ROOTFS_OVERLAY="board/myboard/rootfs-overlay"
```

重新 `make`，overlay 里的文件合并进根文件系统，同名文件覆盖，新文件追加。`.git`、`.svn` 和 `～` 结尾的文件会被自动排除。\[³\]

> **\[³\]** 参考：Buildroot 用户手册 "Customizing the generated target filesystem"，https://buildroot.org/downloads/manual/manual.html#rootfs\-custom

---

## 十、post-build 脚本：动态处理用这个

overlay 放静态文件，动态处理（写版本号、替换占位符、修复权限）用 post-build 脚本，在 `System configuration → Custom scripts to run before creating filesystem images` 里填路径：

```
#!/bin/sh
# board/myboard/post-build.sh
# $1 是 output/target/ 的路径

TARGET_DIR="$1"

# 写固件版本号
echo "fw_version=$(git describe --tags --dirty 2>/dev/null || echo 'unknown')" \
    > "${TARGET_DIR}/etc/fw_version"

# overlay 复制过来的文件有时权限会丢
chmod +x "${TARGET_DIR}/etc/init.d/S99myapp"

# 替换配置文件里的占位符
sed -i "s/__HOSTNAME__/${BR2_TARGET_GENERIC_HOSTNAME}/" \
    "${TARGET_DIR}/etc/myapp.conf"
```

---

## 十一、BR2\_EXTERNAL：板级配置和 Buildroot 源码分开放

把 overlay、defconfig、post-build 脚本都放在 Buildroot 目录里，升级 Buildroot 版本时会有麻烦。用 `BR2_EXTERNAL` 把板级定制单独放一个 repo，和上面的本地内核源码目录并列：\[³\]

```
workspace/
├── buildroot/
├── linux-4.19.232/
├── u-boot-2020.04/
└── my-project/               ← BR2_EXTERNAL，单独一个 repo
    ├── external.mk           ← 必须有（可以为空）
    ├── Config.in             ← 必须有（可以为空）
    ├── external.desc
    ├── configs/
    │   └── myboard_defconfig
    └── board/
        └── myboard/
            ├── rootfs-overlay/
            ├── post-build.sh
            ├── post-image.sh
            ├── genimage.cfg
            ├── kernel.config
            └── uboot.config
```

`external.desc`：

```
name: MY_PROJECT
desc: My custom board support
```

使用：

```
# 第一次指定 BR2_EXTERNAL 路径
make BR2_EXTERNAL=/path/to/my-project myboard_defconfig

# 之后 Buildroot 会记住路径，直接 make 就行
make menuconfig
make
```

defconfig 里引用 external tree 路径和本地源码路径：

```
BR2_ROOTFS_OVERLAY="$(BR2_EXTERNAL_MY_PROJECT_PATH)/board/myboard/rootfs-overlay"
BR2_ROOTFS_POST_BUILD_SCRIPT="$(BR2_EXTERNAL_MY_PROJECT_PATH)/board/myboard/post-build.sh"
BR2_PACKAGE_HOST_GENIMAGE=y
BR2_ROOTFS_POST_IMAGE_SCRIPT="$(BR2_EXTERNAL_MY_PROJECT_PATH)/board/myboard/post-image.sh"
BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="$(BR2_EXTERNAL_MY_PROJECT_PATH)/board/myboard/kernel.config"

# 部分厂商 SDK 支持 CUSTOM_LOCAL（主线 Buildroot 无此选项）
# BR2_LINUX_KERNEL_CUSTOM_LOCAL_PATH="$(TOPDIR)/../linux-4.9.88"
# BR2_TARGET_UBOOT_CUSTOM_LOCAL_PATH="$(TOPDIR)/../u-boot-2016.03"

# 主线 Buildroot 及较新版本 SDK 推荐用 override-srcdir
BR2_PACKAGE_OVERRIDE_FILE="$(BR2_EXTERNAL_MY_PROJECT_PATH)/board/myboard/local.mk"
```

`$(TOPDIR)` 是 Buildroot 根目录，`$(BR2_EXTERNAL_MY_PROJECT_PATH)` 是 external tree 根目录，两个变量一起用，路径就不会写死。

---

## 十二、常用命令速查

```
# 配置
make menuconfig                  # 图形配置
make savedefconfig               # 保存当前配置为 defconfig
make list-defconfigs             # 列出所有内置 defconfig

# 编译
make -j$(nproc)                  # 全量编译
make linux                       # 只编内核
make linux-rebuild               # 重跑内核 make（不清源码目录）
make linux-dirclean              # 清掉内核编译目录，彻底重来
make linux-menuconfig            # 改内核配置
make linux-update-defconfig      # 把内核配置存到指定路径
make busybox-menuconfig          # 改 BusyBox 配置
make busybox-update-config       # 保存 BusyBox 配置
make -rebuild           # 重新编某个包
make -dirclean          # 清掉某个包的编译目录

# 清理
make clean       # 清掉 output/（含工具链编译结果），保留 .config 和 dl/
make distclean   # 在 clean 基础上再删掉 .config，dl/ 两个命令都不动

# 调试
make V=1                         # 打印详细编译命令
make graph-depends               # 生成包依赖关系图（需要安装 graphviz）
make show-targets                # 列出当前配置要编译哪些目标
```

`make clean` 会清掉整个 `output/` 包括工具链编译结果，但不删 `dl/` 和 `.config`。`make clean` 之后重新编，工具链要重新编译，但不用重新下载。`make distclean` 再额外删掉 `.config`，`dl/` 两个命令都不会动。另外，如果开启了 ccache，`make clean` 和 `make distclean` 都**不会**清空 ccache 缓存，需要单独处理。

---

## 十三、NFS 启动：调试阶段的救命稻草

每次改完应用、重新编译镜像、烧 SD 卡、上电测试，有些耗时。调试阶段用 NFS 挂根文件系统，改完重启板子就能看到效果。

## 开发机配置 NFS：

```
sudo apt install nfs-kernel-server

sudo mkdir -p /nfsroot/myboard
sudo tar xf output/images/rootfs.tar -C /nfsroot/myboard

echo "/nfsroot/myboard *(rw,sync,no_subtree_check,no_root_squash)" \
    | sudo tee -a /etc/exports
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

## U-Boot 里改 bootargs：

```
setenv bootargs console=ttymxc0,115200 root=/dev/nfs \
    nfsroot=192.168.1.100:/nfsroot/myboard,v3,tcp \
    ip=192.168.1.101:192.168.1.100:192.168.1.1:255.255.255.0::eth0:off \
    rw
saveenv
```

`nfsroot` 里的 `v3,tcp` 建议显式指定，不加的话有些内核版本默认用 UDP，偶发性不稳定但报错不明显。`rootwait` 这个参数是等块设备（SD 卡、eMMC）就绪用的，NFS 启动不走块设备枚举流程，不需要加；带上也不会报错，内核会忽略，但没必要。

重启板子，根文件系统从开发机挂过来，在 `/nfsroot/myboard` 里直接改文件，重启立刻生效——比每次烧卡省去的时间是肉眼可见的。

---

## 小结

Buildroot 的日常工作流：

```
make _defconfig    → 选基础配置
make menuconfig           → 调整成你要的样子
make savedefconfig        → 存起来
make -j$(nproc)           → 编
post-image/genimage       → 组织 sdcard.img
dd sdcard.img             → 烧
```

本地源码（`dl/` 缓存、部分厂商 SDK 的 `CUSTOM_LOCAL`、主线常用的 `override-srcdir`）解决源码获取和开发调试问题；overlay 和 post-build 脚本处理根文件系统定制；post-image + genimage 负责把内核、设备树、U-Boot 和 rootfs 组织成可烧录镜像；BR2\_EXTERNAL 把这些板级定制从 Buildroot 源码里分离出来。这几件事想清楚之后，换板子、升 Buildroot 版本，基本上是照着走就行，不会再觉得无从下手。

---

## 参考资料

1.  1\. Buildroot 用户手册 — https://buildroot.org/downloads/manual/manual.html
    
2.  2\. Buildroot 官方仓库 — https://gitlab.com/buildroot.org/buildroot
    
3.  3\. Bootlin 嵌入式 Linux 培训讲义（Buildroot 章节）— https://bootlin.com/doc/training/buildroot/
    
4.  4\. 100ask iMX6ULL SDK — https://github.com/100askTeam/100ask\_imx6ull-sdk
    
5.  5\. Gateworks Buildroot 使用笔记 — https://trac.gateworks.com/wiki/buildroot
    

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/04c7d54e_1783566609837?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkxMjM0MzMwNQ%3D%3D%26mid%3D2247483712%26idx%3D1%26sn%3D2ff07c1a80cc2421212f682041c3f08e%26chksm%3Dc0efade7b8a9bed1a3a2a10d1cd52923120d859283351ec3bbdfe9569fa8b377e88a29ce40b6%26mpshare%3D1%26scene%3D1%26srcid%3D0709GOsrsB6JURoX3wXIVOLT%26sharer_shareinfo%3Df71c776afc37d7d05170ca660074f08c%26sharer_shareinfo_first%3Df71c776afc37d7d05170ca660074f08c%23rd&s=obsidian)