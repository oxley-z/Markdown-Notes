---
github.io: true
csdn: false
---
# Linux内核基础知识

![Linux_kernel_map](image/[Linux]-内核基础知识/Linux_kernel_map.png)

<center>Linux Kernel Map</center>

![1200px-Simplified_Structure_of_the_Linux_Kernel.svg](image/[Linux]-内核基础知识/1200px-Simplified_Structure_of_the_Linux_Kernel.svg.png)

# KConfig文件

Kconfig用于内核的配置，主要用来完成config条目界面，一般用来控制一个条件/目录的编译动作，`make menuconfig`显示的菜单信息便来自`.config`文件。

## Kconfig语法

| 关键字             | 内容                                                         |
| ------------------ | ------------------------------------------------------------ |
| `config`           | `config`是构成`Kconfig`的基本单元，其中定义了配置项的详细信息。 |
| `menu/endmenu`     | `menu`的作用可以理解为一个目录，可以把一部分相关配置包含到一个`menu`中，这样有利于配置的分类显示，`menu`中可以嵌套包含`menu`。 |
| `menuconfig`       | `menuconfig`有点类似 `menu`，但区别就在于 `menu` 后面多了一个 `config`，这个 `menu` 是可以配置的，前面比 `menu` 类型多了一个方框，通过空格可以修改这个配置项的选中状态。若选中进入该菜单项时和普通的 `menu` 是一样的。 |
| `if/endif`         | `if/endif` 搭配使用，说明配置项是位于 `if` 和 `endif` 中。`if/endif` 中的部分就是 `MODULES`子目录显示的内容。如果选中了 `MODULE`，那么 `if` 和 `endif` 中的内容可以显示。如果没有定义，就只能进入一个空目录。 |
| `choice/endchoice` | `choice` 的作用，定义了一组选择项，从中选择一个。该配置项的类型只能是 `bool` 或者`tristate`。 |
| `comment`          | 定义了在配置过程中显示给用户的注释，其中唯一可选的属性是依赖关系 `depends on`。 |
| `source`           | `source` 中用来读取指定的配置文件，通过 `source` 就可以将各目录下的 `Kconfig` 文件层层嵌套进`source`来，类似 `c` 语言中的 `include`。 |



## config条目

* prompt：提示信息，显示在配置界面中的信息。

* 变量类型：
  * bool - 二值型变量（y/n）
  * tristate - 三值型变量（y/m/n）
  * string - 字符串变量
  * hex - 十六进制变量
  * int - 整型变量
* default：config条目的默认值
* depends on：依赖项，表示当前条目必须在依赖的项选中后方能出现
* select：饭想依赖，当前项被选中，反向依赖的项目自动被选中
* help：帮助信息

## menu条目

表示一个菜单

以menu开头，endmenu结束，中间的内容即是菜单的内容。

![image-20230308161616689](image/[Linux]-内核基础知识/image-20230308161616689.png)

## source条目

表示需要去调用其他位置的Kconfig文件

## choice条目

多选一选项，以choice开始，endchoice结束，中间的内容即是多选一选项中的内容。

![image-20230308163907590](image/[Linux]-内核基础知识/image-20230308163907590.png)

## menuconfig条目

表示带选项的菜单，通常与if/endif联用，表示选中的条目才会出现。

![image-20230308163451056](image/[Linux]-内核基础知识/image-20230308163451056.png)

## comment条目

文字信息，不可选

`make menuconfig` 配置完成之后，配置结果（变量的值）会保存在内核源码顶层目录的`.config`文件中，Makefile编译内核时回去读取`.config`文件中的变量，来控制编译行为。

![image-20230308164749688](image/[Linux]-内核基础知识/image-20230308164749688.png)

```makefile
obj-y 编译进内核
obj-m 编译成模块
obj-n 不编译
```

# Kbuild

`Kbuild`即`Kernel build`，是内核的一个编译系统。

**简单来说，`Kbuild`是对`Makefile`语法功能的扩展。**

例如，`kbuild`定义了很多有用的变量如`obj-m`、`obj-y`、`-objs`等等，用户只要为这些变量赋值，`kbuild`会自动把代码编译到内核或者编译成模块。

`Linux`内核使用`kbuild`编译时，会扫描两次`Linux`的`Makefile`：首先读取顶层的`Makefile`文件，然后根据读取到的内容第二次读取`kbuild`的`Makefile`文件来编译`Linux`内核。

内核的`Makefile`包括五部分：

| 文件                     | 内容                                                         |
| ------------------------ | ------------------------------------------------------------ |
| 顶层`Makefile`           | 顶层`Makefile` 文件定义两个主要的目标：`vmlinux`（内核映像）和所有的模块，顶层`Makefile` 文件根据内核配置，通过递归编译内核代码树子目录建立这两个文件。 |
| `.config`                | 配置完成后生成`.config`文件，`Kbuild`中的`Makefile`会读取该文件的配置，比如个、各目录中`Makefile`会查看某一个项配置的是`y`还是`n`。 |
| `arch/$(ARCH)/Makefile`  | 顶层`Makefile`文件引用一个名为`arch/$(ARCH)/Makefile`的文件，这个文件提供了与平台相关的一些信息。 |
| `scripts/Makefile.*`     | `scripts/Makefile.*`包含了所有的定义和规则，与`Makefile`文件一起编译出内核映像。 |
| 各目录中的`Makefile`文件 | 每个子目录都有一个`Makefile`文件，子目录`Makeifle`文件根据上级目录`Makefile`文件命令启动编译。这些`Makefile`使用`.config`文件配置数据构建各种文件列表，并使用这些文件列表编译内嵌或模块目标文件。 |

# Makefile

顶层的`Makefile`是编译的入口，下面分析一下顶层`Makefile`文件。

顶层`Makefile`会向各子目录的`Makefile`传递一些信息，有些变量，例如`vmlinux-dirs`，不仅在顶层`Makefile`中定义并且赋值，而且在`arch/*/Makefile`还作了扩充。

| 变量         | 内容                                                         |
| ------------ | ------------------------------------------------------------ |
| 内核版本     | 版本信息定义了荡秋千内核的版本，`VERSION=4`，`PATCHLEVEL=4`，`SUBLEVEL=4`，`EXATAVERSION=`，他们共同构成内核的发行版本号`4.4.4`。 |
| CPU体系结构  | 用`ARCH`定义目标`CPU`的体系结构，比如`ARCH:=arm`等。         |
| 源码路径     | 很多`scripts/Makefile.*`规则被很多不同目录下的`Makefile`所共享，所以要针对不同的路径操作不同的文件，就需要定义但概念的相对路径。 |
| 递归目录定义 | `init-y`、`core-y`、`libs-y`、`drivers-y`、`net-y`。         |
| 编译选项     | 如`CPP`、`CC`、`AS`、`LD`、`AR`、`KBUILD_CFLAGS`、`LDFLAGS_vmlinux`等编译参数。 |

# 编译安装内核

## 下载内核

Linux内核下载[The Linux Kernel Archives](https://www.kernel.org/)

## 安装依赖

```bash
sudo apt-get install gcc g++ libncurses5-dev build-essential kernel-package libssl-dev libc6-dev bin86 flex bison qttools5-dev libelf-dev
```

## 更改.config

```bash
#拷贝现有ubuntu的.config至编译内核的目录
cp -v /boot/config-$(uname -r) .config
```

```bash
make menuconfig
```

1. 打开`.config`更改`CONFIG_SYSTEM_TRUSTED_KEYS `和`CONFIG_SYSTEM_REVOCATION_KEYS`

```bash
CONFIG_SYSTEM_TRUSTED_KEYS=''
CONFIG_SYSTEM_EXTRA_CERTIFICATE_SIZE=2048
CONFIG_SYSTEM_REVOCATION_KEYS=""
```

2. 更改`CONFIG_DEBUG_INFO_BTF`

```bash
CONFIG_DEBUG_INFO_BTF=n
```

3. 注释掉CONFIG_X86_X32

```bash
#CONFIG_X86_X32=y
```



## 编译内核

```bash
#基于文本选单的配置界面，字符终端下推荐使用
make menuconfig

make all -j$(nproc)

# 编译模块
make -j$(nproc) modules
```

## 编译安装

### 首先安装模块

这里加上`INSTALL_MOD_STRIP=1`是为了避免内核启动时卡在 [loading initial ramdisk](#ramdisk)

```bash
sudo make INSTALL_MOD_STRIP=1 modules_install
```

### 安装内核

```bash
make bzImage
sudo make install
```

### 更改引导

```bash
sudo update-initramfs -c -k 5.10.0
```

### 更改`grub`

修改文件

```bash
sudo vi /etc/default/grub
```

```bash
#GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=10 
```

更新

```bash
sudo update-grub
```

### 重启

```bash
reboot
```

在`GRUB`页面选择`Advanced options for Ubuntu`

![image-20230515104210488](image/[Linux]-内核基础知识/image-20230515104210488.png)

选择需要启动的内核版本

![image-20230517111217516](image/[Linux]-内核基础知识/image-20230517111217516.png)




<center>安装新内核之前的内核版本</center>

![image-20230517110513053](image/[Linux]-内核基础知识/image-20230517110513053.png)

<center>安装新内核之后</center>

[若重启未看到选择内核的页面参考](#GRUB)

## 在线安装

```bash
sudo apt-cache search linux-image
#查询列表中可更新的内核

sudo apt-get install linux-image-4.15.0-39-generic  

sudo apt-get install linux-headers-4.15.0-39-generic      

sudo apt-get install linux-modules-4.15.0-39-generic	   

sudo apt-get install linux-modules-extra-4.15.0-39-generic    

```



## 其他操作

### 清理内核源目录

```bash
# make mrproper会删除配置的.config以及其他备份
sudo make mrproper 
# make clean会删除编译过程中生成的中间文件和内核镜像文件
sudo make clean
```

### 卸载安装的内核

#### 获取所有安装的内核版本

```bash
dpkg --get-selections | grep linux
```

选择要卸载的版本，卸载下列安装包

```bash
sudo apt remove linux-image-<版本>-generic
sudo apt remove linux-headers-<版本>
sudo apt remove linux-headers-<版本>-generic
sudo apt remove linux-modules-<版本>-generic
```



卸载源码版本

```bash
sudo rm /boot/vmlinuz-5.10.0
sudo rm /boot/initrd.img-5.10.0 
sudo rm /boot/System.map-5.10.0 
sudo rm /boot/config-5.10.0 
sudo rm -rf /lib/modules/5.10.0 
```

更新启动引导

```bash
sudo update-grub
```

### 修改内核配置菜单实现对新加入内核源码的控制

1. 将源码拷贝到内核对应的文件夹下
2. 为配置界面添加控制新加入源代码的内容

	在加入源码的目录下创建`Kconfig`文件，便添加相应控制内容

3. 修改上层`Kconfig`文件，包含新加入源码的`Kconfig`文件。
4. 修改`Makefile`文件
5. 修改上一级目录的Makefile

```bash
make[1]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
make: *** [Makefile:1868: certs] Error 2
```

解决方法[(104) how to fix the error "No rule to make target 'debian canonical certs pem', needed by 'certs x509 cer - YouTube](https://www.youtube.com/watch?v=7G3UkugVhOw)：

```bash
nano .config
查找到debian/canonical-certs.pem和debian/canonical-revoked-certs.pem字符串删除
```

```bash
BTF: .tmp_vmlinux.btf: pahole (pahole) is not available
```

解决方法

```bash
sudo apt install dwarves
```



## 常见问题

### 1. Module.symvers is missing

```bash
WARNING: Symbol version dump "Module.symvers" is missing.
         Modules may not have dependencies or modversions.
         You may get many unresolved symbol warnings.
```

执行

```bash
make modules_prepare
```

### 2. No rule to make target ‘debian/canonical-certs.pem‘, needed by ‘certs/x509_certificate_list‘

可以修改config：

vim .config

修改`CONFIG_SYSTEM_TRUSTED_KEYS`，将其置空： CONFIG_SYSTEM_TRUSTED_KEYS=""*也可能需要设置*CONFIG_SYSTEM_REVOCATION_KEYS为空。

### 3. FAILED:load BTF from vmlinux:No such file or directory

vim .config

修改`CONFIG_DEBUG_INFO_BTF`，将其置n

### 4. 安装完内核之后无法显示`GRUB`用以选择内核启动<a name="GRUB"/>

进入命令行之后执行

```bash
sudo vi /etc/default/grub
```

注释掉

```bash
# GRUB_TIMEOUT_STYLE=hidden
```

修改

```bash
GRUB_TIMEOUT=10
 
GRUB_COMLINE_LINUX_DEFAULT="text"
```

![image-20230515103459264](image/[Linux]-内核基础知识/image-20230515103459264.png)

执行

```bash
sudo update-grub
```

![image-20230515103559801](image/[Linux]-内核基础知识/image-20230515103559801.png)

### 5. 内核启动卡在`loading initial ramdisk`<a name="ramdisk"/>

编译内核过程中，当安装内核模块时未使用

```bash
INSTALL_MOD_STRIP=1
```

标注，会导致`initrd`文件过大，Ubuntu 20.04所用的Grub 2.04无法支持过大的initrd文件（如500M），导致内核启动时卡在`“loading initial ramdisk”（Can’t allocate initrd）`。

可行的办法：

安装模块时加上`INSTALL_MOD_STRIP=1`

```bash
sudo make INSTALL_MOD_STRIP=1 modules_install
```

### 6. sign-file: : No such file or directory

报错信息：

![image-20230516082845572](image/[Linux]-内核基础知识/image-20230516082845572.png)

解决方法：

 将`.config`的 `CONFIG_MODULE_SIG_KEY="cert/signing_key.pem"`恢复后解决。

```bash
CONFIG_MODULE_SIG_KEY="cert/signing_key.pem"
```

### 7. zstd: not found

```bash
ZSTD22  arch/x86/boot/compressed/vmlinux.bin.zst 
/bin/sh: 1: zstd: not found 
make[2]: *** [arch/x86/boot/compressed/Makefile:134：arch/x86/boot/compressed/vmlinux.bin.zst] 错误 127 
make[2]: *** 正在删除文件“arch/x86/boot/compressed/vmlinux.bin.zst” make[1]: *** [arch/x86/boot/Makefile:115：arch/x86/boot/compressed/vmlinux] 错误 2
```

解决方法

在配置内核时更改`Kernel compression mode`为`LZMA`

```bash
General setup  ---> 
	Kernel compression mode (LZMA)  ---> 
```

保存后重新编译安装即可。

### 8. ERROR: Kernel configuration is invalid.

```bash
  ERROR: Kernel configuration is invalid.
         include/generated/autoconf.h or include/config/auto.conf are missing.
         Run 'make oldconfig && make prepare' on kernel src to fix it.
```

原因为内核未编译，

## 单独编译Linux内核的某一个模块

找到对应的模块文件夹，找到需要编译的文件，确认编译的config文件

```bash
make CONFIG_INFINIBAND=m -C <源码> M=<模块文件夹> modules
```

# qemu+gdb调试linux内核

## 安装qemu

```bash
sudo apt install qemu
```

## 配置调试版内核

对内核进行调试需要解析符号信息，所以得编译一个调试版内核。

```bash
cd linux-5.15
make menuconfig
```

这里需要开启内核参数`CONFIG_DEBUG_INFO`和`CONFIG_GDB_SCRIPTS`。GDB提供了Python接口来扩展功能，内核基于Python接口实现了一系列辅助脚本，简化内核调试，开启`CONFIG_GDB_SCRIPTS`参数就可以使用了。

```bash
Kernel hacking  --->
    [*] Kernel debugging
    Compile-time checks and compiler options  --->
        [*] Compile the kernel with debug info
        [*]   Provide GDB scripts for kernel debugging
```

## 构建initramfs根文件系统

Linux系统启动阶段，boot loader加载完内核文件vmlinuz后，内核紧接着需要挂载磁盘根文件系统，但如果此时内核没有相应驱动，无法识别磁盘，就需要先加载驱动，而驱动又位于`/lib/modules`，得挂载根文件系统才能读取，这就陷入了一个两难境地，系统无法顺利启动。于是有了initramfs根文件系统，其中包含必要的设备驱动和工具，boot loader加载initramfs到内存中，内核会将其挂载到根目录`/`,然后运行`/init`脚本，挂载真正的磁盘根文件系统。

这里借助[BusyBox](https://www.busybox.net/)构建极简initramfs，提供基本的用户态可执行程序。

编译BusyBox，配置`CONFIG_STATIC`参数，编译静态版BusyBox，编译好的可执行文件`busybox`不依赖动态链接库，可以独立运行，方便构建initramfs。

```bash
cd busybox-1.28.0
make menuconfig
```

```bash
Settings  --->
    [*] Build static binary (no shared libs)
```

```bash
make -j 20
make install
```

会安装在`_install`目录:

```bash
ls _install
bin  linuxrc  sbin  usr
```

创建initramfs，其中包含BusyBox可执行程序、必要的设备文件、启动脚本`init`。这里没有内核模块，如果需要调试内核模块，可将需要的内核模块包含进来。`init`脚本只挂载了虚拟文件系统`procfs`和`sysfs`，没有挂载磁盘根文件系统，所有调试操作都在内存中进行，不会落磁盘。

```bash
mkdir initramfs
cd initramfs
cp ../_install/* -rf ./
mkdir dev proc sys
sudo cp -a /dev/{null, console, tty, tty1, tty2, tty3, tty4} ./dev/
rm linuxrc
vim init
chmod a+x init
ls
bin   dev  init  proc  sbin  sys   usr
```

init文件的内容

```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
exec /bin/sh
```

打包initramfs

```bash
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
```

## 调试

```bash
cd busybox-xxx
qemu-system-x86_64 -s -kernel /path/to/vmlinux -initrd initramfs.cpio.gz -nographic -append "console=ttyS0"
```

* `-s` 是`-gdb tcp::1234`的缩写，监听1234端口，在GDB中通过`target remote localhost:1234`连接；
* `-kernel` 指定编译好的调试内核`vmlinux`路径；
* `initrd` 指定制作好的`initramfs`；
* `-nographic`取消图形输出窗口，试qemu成简单的命令行程序。
* `-append "console=ttyS0"`将输出重定向到console，将会显示在标准输出stdio。

启动后的根目录

```bash
/ ls
bin   dev  init  proc  root  sbin  sys   usr
```

# 参考

[使用QEMU和GDB调试Linux内核 | Consen](https://consen.github.io/2018/01/17/debug-linux-kernel-with-qemu-and-gdb/)

[Linux下使用内核源码单独编译某一模块 - tycoon3 - 博客园 (cnblogs.com)](https://www.cnblogs.com/dream397/p/13984263.html)

[内核错误: No rule to make target ‘debian/canonical-certs.pem‘, needed by ‘certs/x509_certificate_list‘_no rule to make target 'debian/canonical-certs.pem_蓝天居士的博客-CSDN博客](https://blog.csdn.net/phmatthaus/article/details/124353775)

[Linux 内核 下载 编译 安装 2021 ubuntu_yaoxinJJJ的博客-CSDN博客](https://blog.csdn.net/yaoxinJJJ/article/details/115433638)

[内核Module.symvers文件揭秘 - Linux内核编程 | 宅学部落 (zhaixue.cc)](https://www.zhaixue.cc/kernel/kernel-module_sysmvers.html)

[如何编译安装Linux内核 - LightningStar - 博客园 (cnblogs.com)](https://www.cnblogs.com/harrypotterjackson/p/11846222.html)

[ubuntu上更新和卸载Linux内核 - 广漠飘羽 - 博客园 (cnblogs.com)](https://www.cnblogs.com/gmpy/p/12533548.html)

[关于Ubuntu内核(更新和卸载内核、取消自动更新) · Issue #1 · chiwent/blog (github.com)](https://github.com/chiwent/blog/issues/1)

[自行编译内核，启动内核卡在“loading initial ramdisk”_启动卡在initrd_奇妙之二进制的博客-CSDN博客](https://blog.csdn.net/HandsomeHong/article/details/125157372)

[linux——编译内核（ubuntu18.04+linux-5.6.4）](https://blog.csdn.net/bdwdwks/article/details/106294349)

[编译内核 make modules_install报错SSL error:02001002:system library:fopen:No such file or directory_ssl: error:02001002_酱山楂的博客-CSDN博客](https://blog.csdn.net/weixin_52592969/article/details/127815302)

[中文翻译 — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/translations/zh_CN/index.html) Linux内核中文文档

[[arch/x86/boot/compressed/vmlinux.bin.lzma] Error 1_sustwct的博客-CSDN博客](https://blog.csdn.net/sustwct/article/details/7109251)
