# qemu搭建arm64 uboot环境

## 环境搭建

### 虚拟机

```bash
$ uname -a
Linux test 5.15.0-101-generic #111~20.04.1-Ubuntu SMP Mon Mar 11 15:44:43 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

### 内核及根文件系统源码

uboot源码：[uboot2022.10](https://ftp.denx.de/pub/u-boot/)

### 交叉编译工具链

```bash
$ sudo apt-get install gcc-aarch64-linux-gnu
```

### qemu

```bash
$ sudo apt-get install qemu-system-arm
```

### 其他依赖包

```bash
$ sudo apt-get install gcc g++ libncurses5-dev build-essential kernel-package libssl-dev libc6-dev bin86 flex bison qttools5-dev libelf-dev
```

## uboot编译

解压uboot

```bash
$ tar -xvf u-boot-2022.10.tar.bz2
```

生成配置文件 `.config`

```bash
$ make qemu_arm64_defconfig CROSS_COMPILE=aarch64-linux-gnu- O=./object
make[1]: 进入目录“/home/test/zjzhe/uboot/u-boot-2022.10/object”
  GEN     Makefile
  HOSTCC  scripts/kconfig/conf.o
  HOSTLD  scripts/kconfig/conf
#
# configuration written to .config
#
make[1]: 离开目录“/home/test/zjzhe/uboot/u-boot-2022.10/object”
```

如果需要修改配置文件中的一些选项，可以通过以下界面操作选择

```bash
$ make menuconfig ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- O=./object
```

编译

```bash
$ make CROSS_COMPILE=aarch64-linux-gnu- O=./object -j4
...
  CC      examples/standalone/hello_world.o
  CC      examples/standalone/stubs.o
  LD      examples/standalone/libstubs.o
  LD      examples/standalone/hello_world
  OBJCOPY examples/standalone/hello_world.srec
  OBJCOPY examples/standalone/hello_world.bin
  LD      u-boot
  OBJCOPY u-boot.srec
  OBJCOPY u-boot-nodtb.bin
  SYM     u-boot.sym
  RELOC   u-boot-nodtb.bin
  DTC     arch/arm/dts/qemu-arm.dtb
  COPY    u-boot.bin
  DTC     arch/arm/dts/qemu-arm64.dtb
  SHIPPED dts/dt.dtb
make[1]: 离开目录“/home/test/zjzhe/uboot/u-boot-2022.10/object”
```

## qemu启动uboot

### 不带网络配置

qemu启动uboot

```bash
$ qemu-system-aarch64 -machine virt -cpu cortex-a72 -m size=1G -kernel ./object/u-boot -nographic
```

### 网络配置

#### 搭建网桥

```bash
sudo brctl addbr br0
sudo ifconfig br0 192.168.2.46
```

#### qemu_ifup

```bash
$ sudo vim /etc/qemu_ifup
```

键入以下内容

```bash
#! /bin/sh
# Script to bring a network (tap) device for qemu up.
# The idea is to add the tap device to the same bridge
# as we have default routing to.

# in order to be able to find brctl
echo sudo tunctl -u $(id -un) -t $1
sudo tunctl -u $(id -un) -t $1
echo sudo ifconfig $1 0.0.0.0 promisc up
sudo ifconfig $1 0.0.0.0 promisc up
echo sudo brctl addif br0 $1
sudo brctl addif br0 $1
echo brctl show
brctl show
sudo ifconfig br0 192.168.2.74 # 这里设置的是网桥br0的地址
```

#### qemu_ifdown

```bash
$ sudo vim /etc/qemu_ifup
```

键入以下内容

```bash
#! /bin/sh
# Script to shut down a network (tap) device for qemu.
# Initially this script is empty, but you can configure,
# for example, accounting info here.
echo sudo brctl delif br0 $1
sudo brctl delif br0 $1
echo sudo tunctl -d $1
sudo tunctl -d $1
echo brctl show
brctl show
```

#### qemu启动uboot

```bash
$ sudo qemu-system-aarch64 -machine virt -cpu cortex-a72 -m size=1G -kernel ./object/u-boot -nographic -nic tap,ifname=tap0

sudo tunctl -u root -t tap0
sudo: tunctl：找不到命令
sudo ifconfig tap0 0.0.0.0 promisc up
sudo brctl addif br0 tap0
brctl show
bridge name	bridge id		STP enabled	interfaces
br0		8000.7a3954aecd60	no		tap0


U-Boot 2022.10 (Oct 29 2025 - 15:19:29 +0800)

DRAM:  1 GiB
Core:  47 devices, 13 uclasses, devicetree: board
Flash: 64 MiB
Loading Environment from Flash... *** Warning - bad CRC, using default environment

In:    pl011@9000000
Out:   pl011@9000000
Err:   pl011@9000000
Net:   eth0: virtio-net#32
Hit any key to stop autoboot:  0 
=> 

```

> 此处 `sudo: tunctl：找不到命令` 可安装 `uml-utilities`软件包，但是安装后，配置网络后无法ping通主机。

在控制台分别输入以下命令，配置网络

```bash
$ setenv ipaddr   192.168.160.77
$ setenv serverip 192.168.160.74
```

```bash
=> setenv serverip 192.168.160.74
=> setenv ipaddr   192.168.160.77
=> ping 192.168.160.74
Using virtio-net#32 device
host 192.168.160.74 is alive
=>
```

# 常见问题

## 退出qemu

```bash
CTRL+A->X
```

## sudo: tunctl：找不到命令

```bash
$ sudo apt install uml-utilities
```

# 参考

[[(转)mkimage制作linux内核映像 即uImage是怎么制作的](https://www.cnblogs.com/lihaiping/archive/2013/04/20/uImage.html)](https://www.cnblogs.com/lihaiping/archive/2013/04/20/uImage.html)

