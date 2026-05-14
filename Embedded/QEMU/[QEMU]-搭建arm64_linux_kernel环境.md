# qemu搭建arm64_linux_kernel环境

## 环境搭建

### 虚拟机

```bash
$ uname -a
Linux test 5.15.0-101-generic #111~20.04.1-Ubuntu SMP Mon Mar 11 15:44:43 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

### 内核及根文件系统源码

内核源码：linux-5.10 [The Linux Kernel Archives](https://www.kernel.org/)

busybox：[busybox-1.32.0](https://busybox.net/downloads/)

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

## 编译Image内核

解压内核文件后，设置`config`文件：

```bash
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
```

上面使用的defconfig文件在目录arch/arm64/configs/defconfig，如果需要增减kernel CONFIG 直接修改这个文件即可。

编译内核

```bash
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image -j8
...
  LD      vmlinux.o
  MODPOST vmlinux.symvers
  MODINFO modules.builtin.modinfo
  GEN     modules.builtin
  LD      .tmp_vmlinux.kallsyms1
  KSYMS   .tmp_vmlinux.kallsyms1.S
  AS      .tmp_vmlinux.kallsyms1.S
  LD      .tmp_vmlinux.kallsyms2
  KSYMS   .tmp_vmlinux.kallsyms2.S
  AS      .tmp_vmlinux.kallsyms2.S
  LD      vmlinux
  SORTTAB vmlinux
  SYSMAP  System.map
  OBJCOPY arch/arm64/boot/Image
```

若之前编译过，需要重新编译，执行如下命令

```bash
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- clean
```

编译模块(module ko)

```bash
$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules -j8
```

或者使用以下命令

```bash
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-linux-gnu-
$ make defconfig
$ make all -j8
```

## 根文件系统制作

### busybox配置编译

```bash
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-linux-gnu-
$ make menuconfig
```

```bash
Settings  ---> 
[*] Build static binary (no shared libs)    //静态编译
[*] Build with debug information       //可选，带调试信息，方便后续调试
```

```bash
$ make && make install
```

当前目录下的`_install`目录就是最终的根文件系统。

```bash
$ ls ./_install/
bin  linuxrc  sbin  usr
```

### rootfs image制作

```bash
$ qemu-img create rootfs.img 512m
$ mkfs.ext4 rootfs.img
```

挂载rootfs.img到rootfs目录

```bash
$ mkdir rootfs
$ sudo mount rootfs.img rootfs
```

拷贝busybox的编译结果_install，同时创建关键目录（挂载虚拟文件系统需要）

```bash
$ sudo cp -rf _install/*  rootfs
$ cd rootfs
$ sudo mkdir proc sys dev etc etc/init.d
$ sudo vim etc/init.d/rcS
```

rcS中的内容，主要是挂载常使用的虚拟文件系统

```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t debugfs none /sys/kernel/debug
```

然后给rcS增加执行权限

```bash
$ sudo chmod +x  etc/init.d/rcS
```

最后umount rootfs

```bash
$ cd ../
$ sudo umount rootfs
```

## qemu启动kernel

qemu启动时也需要配置一个启动脚本加载kernel和rootfs

将前面编译生成的Image(arch/arm64/boot/Image)、rootfs.img拷贝到一个目录

创建shell执行脚本，输入以下内容

```bash
#!/bin/bash

qemu-system-aarch64 \
-machine virt,virtualization=true,gic-version=3 \
-nographic \
-m size=1024M \
-cpu cortex-a72 \
-smp 2 \
-kernel Image \
-drive format=raw,file=rootfs.img \
-append "root=/dev/vda rw"
```

对脚本添加可执行权限，之后直接执行脚本即可。

```bash
$ ./run.sh 
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd083]
[    0.000000] Linux version 5.10.0 (test@test) (aarch64-linux-gnu-gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, GNU ld (GNU Binutils for Ubuntu) 2.34) #1 SMP PREEMPT Wed Oct 29 09:42:10 CST 2025
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 32 MiB at 0x000000007e000000
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x7ddf5b00-0x7ddf7fff]
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv0.2 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] percpu: Embedded 23 pages/cpu s56216 r8192 d29800 u94208
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: GIC system register CPU interface
[    0.000000] CPU features: detected: EL2 vector hardening
[    0.000000] CPU features: detected: Spectre-v2
[    0.000000] CPU features: detected: Spectre-v4
[    0.000000] CPU features: detected: ARM errata 1165522, 1319367, or 1530923
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 258048
[    0.000000] Policy zone: DMA
[    0.000000] Kernel command line: root=/dev/vda rw
[    0.000000] Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
[    0.000000] Inode-cache hash table entries: 65536 (order: 7, 524288 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 963352K/1048576K available (14208K kernel code, 2796K rwdata, 7592K rodata, 5952K init, 518K bss, 52456K reserved, 32768K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU event tracing is enabled.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=2.
[    0.000000] 	Trampoline variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GICv3: GIC: Using split EOI/Deactivate mode
[    0.000000] GICv3: 224 SPIs implemented
[    0.000000] GICv3: 0 Extended SPIs implemented
[    0.000000] GICv3: Distributor has no Range Selector support
[    0.000000] GICv3: 16 PPIs implemented
[    0.000000] GICv3: CPU0: found redistributor 0 region 0:0x00000000080a0000
[    0.000000] random: get_random_bytes called from start_kernel+0x32c/0x4f0 with crng_init=0
[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
[    0.000171] sched_clock: 56 bits at 62MHz, resolution 16ns, wraps every 4398046511096ns
[    0.006956] Console: colour dummy device 80x25
[    0.008705] printk: console [tty0] enabled
[    0.011378] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)
[    0.011559] pid_max: default: 32768 minimum: 301
[    0.012647] LSM: Security Framework initializing
[    0.013770] Mount-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.013840] Mountpoint-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.052616] rcu: Hierarchical SRCU implementation.
[    0.057418] EFI services will not be available.
[    0.058298] smp: Bringing up secondary CPUs ...
[    0.062028] Detected PIPT I-cache on CPU1
[    0.062632] GICv3: CPU1: found redistributor 1 region 0:0x00000000080c0000
[    0.062828] CPU1: Booted secondary processor 0x0000000001 [0x410fd083]
[    0.065594] smp: Brought up 1 node, 2 CPUs
[    0.065692] SMP: Total of 2 processors activated.
[    0.065772] CPU features: detected: 32-bit EL0 Support
[    0.065834] CPU features: detected: CRC32 instructions
[    0.065873] CPU features: detected: 32-bit EL1 Support
[    0.118679] CPU: All CPU(s) started at EL2
[    0.119089] alternatives: patching kernel code
[    0.133914] devtmpfs: initialized
[    0.142878] KASLR disabled due to lack of seed
[    0.146628] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.146822] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.149511] pinctrl core: initialized pinctrl subsystem
[    0.158770] DMI not present or invalid.
[    0.165298] NET: Registered protocol family 16
[    0.176813] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
[    0.176996] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.177324] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.177584] audit: initializing netlink subsys (disabled)
[    0.179444] audit: type=2000 audit(0.144:1): state=initialized audit_enabled=0 res=1
[    0.183802] thermal_sys: Registered thermal governor 'step_wise'
[    0.183882] thermal_sys: Registered thermal governor 'power_allocator'
[    0.184628] cpuidle: using governor menu
[    0.185457] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.186158] ASID allocator initialised with 65536 entries
[    0.189332] Serial: AMBA PL011 UART driver
[    0.230753] 9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 48, base_baud = 0) is a PL011 rev1
[    0.251064] printk: console [ttyAMA0] enabled
[    0.280851] HugeTLB registered 1.00 GiB page size, pre-allocated 0 pages
[    0.281046] HugeTLB registered 32.0 MiB page size, pre-allocated 0 pages
[    0.281265] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    0.281365] HugeTLB registered 64.0 KiB page size, pre-allocated 0 pages
[    0.295683] cryptd: max_cpu_qlen set to 1000
[    0.309958] ACPI: Interpreter disabled.
[    0.314370] iommu: Default domain type: Translated 
[    0.315733] vgaarb: loaded
[    0.317048] SCSI subsystem initialized
[    0.319795] usbcore: registered new interface driver usbfs
[    0.320659] usbcore: registered new interface driver hub
[    0.321262] usbcore: registered new device driver usb
[    0.322769] pps_core: LinuxPPS API ver. 1 registered
[    0.322948] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.323138] PTP clock support registered
[    0.323565] EDAC MC: Ver: 3.0.0
[    0.328131] FPGA manager framework
[    0.328857] Advanced Linux Sound Architecture Driver Initialized.
[    0.340413] clocksource: Switched to clocksource arch_sys_counter
[    0.341757] VFS: Disk quotas dquot_6.6.0
[    0.342207] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.344202] pnp: PnP ACPI: disabled
[    0.387661] NET: Registered protocol family 2
[    0.393075] tcp_listen_portaddr_hash hash table entries: 512 (order: 1, 8192 bytes, linear)
[    0.393882] TCP established hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.394605] TCP bind hash table entries: 8192 (order: 5, 131072 bytes, linear)
[    0.395200] TCP: Hash tables configured (established 8192 bind 8192)
[    0.396682] UDP hash table entries: 512 (order: 2, 16384 bytes, linear)
[    0.397039] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes, linear)
[    0.398562] NET: Registered protocol family 1
[    0.405193] RPC: Registered named UNIX socket transport module.
[    0.405428] RPC: Registered udp transport module.
[    0.405535] RPC: Registered tcp transport module.
[    0.405704] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.406235] PCI: CLS 0 bytes, default 64
[    0.414571] hw perfevents: enabled with armv8_pmuv3 PMU driver, 5 counters available
[    0.415672] kvm [1]: IPA Size Limit: 44 bits
[    0.422436] kvm [1]: GICv3: no GICV resource entry
[    0.422774] kvm [1]: disabling GICv2 emulation
[    0.423125] kvm [1]: GIC system register CPU interface enabled
[    0.424444] kvm [1]: vgic interrupt IRQ9
[    0.426229] kvm [1]: Hyp mode initialized successfully
[    0.451271] Initialise system trusted keyrings
[    0.453780] workingset: timestamp_bits=42 max_order=18 bucket_order=0
[    0.465524] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.469549] NFS: Registering the id_resolver key type
[    0.470301] Key type id_resolver registered
[    0.470600] Key type id_legacy registered
[    0.471336] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    0.473055] 9p: Installing v9fs 9p2000 file system support
[    0.520284] Key type asymmetric registered
[    0.520860] Asymmetric key parser 'x509' registered
[    0.521616] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 245)
[    0.522395] io scheduler mq-deadline registered
[    0.523015] io scheduler kyber registered
[    0.543478] pl061_gpio 9030000.pl061: PL061 GPIO chip registered
[    0.549685] pci-host-generic 4010000000.pcie: host bridge /pcie@10000000 ranges:
[    0.550771] pci-host-generic 4010000000.pcie:       IO 0x003eff0000..0x003effffff -> 0x0000000000
[    0.551390] pci-host-generic 4010000000.pcie:      MEM 0x0010000000..0x003efeffff -> 0x0010000000
[    0.551866] pci-host-generic 4010000000.pcie:      MEM 0x8000000000..0xffffffffff -> 0x8000000000
[    0.553423] pci-host-generic 4010000000.pcie: ECAM at [mem 0x4010000000-0x401fffffff] for [bus 00-ff]
[    0.554928] pci-host-generic 4010000000.pcie: PCI host bridge to bus 0000:00
[    0.555390] pci_bus 0000:00: root bus resource [bus 00-ff]
[    0.555703] pci_bus 0000:00: root bus resource [io  0x0000-0xffff]
[    0.556024] pci_bus 0000:00: root bus resource [mem 0x10000000-0x3efeffff]
[    0.556342] pci_bus 0000:00: root bus resource [mem 0x8000000000-0xffffffffff]
[    0.558577] pci 0000:00:00.0: [1b36:0008] type 00 class 0x060000
[    0.561839] pci 0000:00:01.0: [1af4:1000] type 00 class 0x020000
[    0.562428] pci 0000:00:01.0: reg 0x10: [io  0x0000-0x001f]
[    0.562880] pci 0000:00:01.0: reg 0x20: [mem 0x00000000-0x00003fff 64bit pref]
[    0.563318] pci 0000:00:01.0: reg 0x30: [mem 0x00000000-0x0007ffff pref]
[    0.564054] pci 0000:00:02.0: [1af4:1001] type 00 class 0x010000
[    0.564433] pci 0000:00:02.0: reg 0x10: [io  0x0000-0x007f]
[    0.564893] pci 0000:00:02.0: reg 0x20: [mem 0x00000000-0x00003fff 64bit pref]
[    0.567578] pci 0000:00:01.0: BAR 6: assigned [mem 0x10000000-0x1007ffff pref]
[    0.568196] pci 0000:00:01.0: BAR 4: assigned [mem 0x8000000000-0x8000003fff 64bit pref]
[    0.568881] pci 0000:00:02.0: BAR 4: assigned [mem 0x8000004000-0x8000007fff 64bit pref]
[    0.569293] pci 0000:00:02.0: BAR 0: assigned [io  0x1000-0x107f]
[    0.569651] pci 0000:00:01.0: BAR 0: assigned [io  0x1080-0x109f]
[    0.575454] EINJ: ACPI disabled.
[    0.594318] virtio-pci 0000:00:01.0: enabling device (0000 -> 0003)
[    0.597264] virtio-pci 0000:00:02.0: enabling device (0000 -> 0003)
[    0.617754] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    0.623712] SuperH (H)SCI(F) driver initialized
[    0.624724] msm_serial: driver initialized
[    0.627606] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    0.651867] loop: module loaded
[    0.658185] virtio_blk virtio1: [vda] 1048576 512-byte logical blocks (537 MB/512 MiB)
[    0.658541] vda: detected capacity change from 0 to 536870912
[    0.681780] megasas: 07.714.04.00-rc1
[    0.687904] physmap-flash 0.flash: physmap platform flash device: [mem 0x00000000-0x03ffffff]
[    0.689773] 0.flash: Found 2 x16 devices at 0x0 in 32-bit bank. Manufacturer ID 0x000000 Chip ID 0x000000
[    0.690448] Intel/Sharp Extended Query Table at 0x0031
[    0.691116] Using buffer write method
[    0.691700] physmap-flash 0.flash: physmap platform flash device: [mem 0x04000000-0x07ffffff]
[    0.692559] 0.flash: Found 2 x16 devices at 0x0 in 32-bit bank. Manufacturer ID 0x000000 Chip ID 0x000000
[    0.693223] Intel/Sharp Extended Query Table at 0x0031
[    0.693737] Using buffer write method
[    0.693982] Concatenating MTD devices:
[    0.694076] (0): "0.flash"
[    0.694123] (1): "0.flash"
[    0.694218] into device "0.flash"
[    0.724187] libphy: Fixed MDIO Bus: probed
[    0.729898] tun: Universal TUN/TAP device driver, 1.6
[    0.740825] thunder_xcv, ver 1.0
[    0.741190] thunder_bgx, ver 1.0
[    0.741543] nicpf, ver 1.0
[    0.743606] hclge is initializing
[    0.744097] hns3: Hisilicon Ethernet Network Driver for Hip08 Family - version
[    0.744741] hns3: Copyright (c) 2017 Huawei Corporation.
[    0.745436] e1000: Intel(R) PRO/1000 Network Driver
[    0.745807] e1000: Copyright (c) 1999-2006 Intel Corporation.
[    0.746312] e1000e: Intel(R) PRO/1000 Network Driver
[    0.746657] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    0.747093] igb: Intel(R) Gigabit Ethernet Network Driver
[    0.747548] igb: Copyright (c) 2007-2014 Intel Corporation.
[    0.748044] igbvf: Intel(R) Gigabit Virtual Function Network Driver
[    0.748506] igbvf: Copyright (c) 2009 - 2012 Intel Corporation.
[    0.749670] sky2: driver version 1.30
[    0.751932] VFIO - User Level meta-driver version: 0.3
[    0.755333] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    0.755733] ehci-pci: EHCI PCI platform driver
[    0.756189] ehci-platform: EHCI generic platform driver
[    0.756699] ehci-orion: EHCI orion driver
[    0.757068] ehci-exynos: EHCI Exynos driver
[    0.758746] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    0.759251] ohci-pci: OHCI PCI platform driver
[    0.759785] ohci-platform: OHCI generic platform driver
[    0.760505] ohci-exynos: OHCI Exynos driver
[    0.762146] usbcore: registered new interface driver usb-storage
[    0.773735] rtc-pl031 9010000.pl031: registered as rtc0
[    0.774832] rtc-pl031 9010000.pl031: setting system clock to 2025-10-29T03:29:33 UTC (1761708573)
[    0.777183] i2c /dev entries driver
[    0.786861] sdhci: Secure Digital Host Controller Interface driver
[    0.787508] sdhci: Copyright(c) Pierre Ossman
[    0.788701] Synopsys Designware Multimedia Card Interface Driver
[    0.790865] sdhci-pltfm: SDHCI platform and OF driver helper
[    0.794550] ledtrig-cpu: registered to indicate activity on CPUs
[    0.798221] usbcore: registered new interface driver usbhid
[    0.798538] usbhid: USB HID core driver
[    0.811261] NET: Registered protocol family 17
[    0.821113] 9pnet: Installing 9P2000 support
[    0.821903] Key type dns_resolver registered
[    0.823548] registered taskstats version 1
[    0.823932] Loading compiled-in X.509 certificates
[    0.834102] input: gpio-keys as /devices/platform/gpio-keys/input/input0
[    0.843297] ALSA device list:
[    0.843614]   No soundcards found.
[    0.847844] uart-pl011 9000000.pl011: no DMA platform data
[    0.887280] EXT4-fs (vda): mounted filesystem with ordered data mode. Opts: (null)
[    0.887849] VFS: Mounted root (ext4 filesystem) on device 254:0.
[    0.897978] devtmpfs: mounted
[    0.932990] Freeing unused kernel memory: 5952K
[    0.933919] Run /sbin/init as init process

Please press Enter to activate this console. 
/ # 

```

QEMU参数说明（更多可参考：[Standard options](https://www.qemu.org/docs/master/system/invocation.html)）：

* -M：指定模拟的开发板，可通过qemu-system-aarch64 M help查看，截至书稿时，只有virt支持64-bit ARM。
* -cpu：指定模拟的cpu，可通过qemu-system-aarch64 -M virt --cpu help查看，这里选择cortex-a72。
* -smp：指定cpu核数量，启动后可以使用nproc命令核对。
* -m：指定内存大小，virt 可支持超大内存，启动后可以使用free -h命令核对。
  -kernel：指定启动的内核镜像；
* -initrd：指定启动的内存文件系统；
* -append：传递给内核的启动参数；启动后可使用cat /proc/cmdline命令核对。
* -nographic：启动字符界面（不启动图形界面），输出重定向到宿主机命令行，与参数 console=ttyAMA0 组合使用；

## qemu使用uboot引导arm64 kernel

uboot的构建过程见[[uboot]-qemu搭建uboot环境](../uboot/[uboot]-qemu搭建uboot环境.md)

### Image文件放置

挂载rootfs将编译出来的Image文件放置在`rootfs.img`中。

```bash
$ sudo mount rootfs.img rootfs
$ sudo mkdir ./rootfs/boot/
$ sudo cp ../linux-5.10/arch/arm64/boot/Image rootfs/boot/
$ ls ./rootfs
bin  boot  dev  etc  linuxrc  lost+found  proc  sbin  sys  usr
```

可使用tree命令查看rootfs文件夹的文件

```bash
$ tree ./rootfs
./rootfs
├── bin
│   ├── arch -> busybox
│   ├── ash -> busybox
│   ├── base64 -> busybox
│   ├── busybox
│   ├── cat -> busybox
│   ├── chattr -> busybox
│   ├── chgrp -> busybox
│   ├── chmod -> busybox
│   ├── chown -> busybox
│   ├── conspy -> busybox
│   ├── cp -> busybox
│   ├── cpio -> busybox
│   ├── cttyhack -> busybox
│   ├── date -> busybox
│   ├── dd -> busybox
│   ├── df -> busybox
│   ├── dmesg -> busybox
│   ├── dnsdomainname -> busybox
│   ├── dumpkmap -> busybox
│   ├── echo -> busybox
│   ├── ed -> busybox
│   ├── egrep -> busybox
│   ├── false -> busybox
│   ├── fatattr -> busybox
│   ├── fdflush -> busybox
│   ├── fgrep -> busybox
│   ├── fsync -> busybox
│   ├── getopt -> busybox
│   ├── grep -> busybox
│   ├── gunzip -> busybox
│   ├── gzip -> busybox
│   ├── hostname -> busybox
│   ├── hush -> busybox
│   ├── ionice -> busybox
│   ├── iostat -> busybox
│   ├── ipcalc -> busybox
│   ├── kbd_mode -> busybox
│   ├── kill -> busybox
│   ├── link -> busybox
│   ├── linux32 -> busybox
│   ├── linux64 -> busybox
│   ├── ln -> busybox
│   ├── login -> busybox
│   ├── ls -> busybox
│   ├── lsattr -> busybox
│   ├── lzop -> busybox
│   ├── makemime -> busybox
│   ├── mkdir -> busybox
│   ├── mknod -> busybox
│   ├── mktemp -> busybox
│   ├── more -> busybox
│   ├── mount -> busybox
│   ├── mountpoint -> busybox
│   ├── mpstat -> busybox
│   ├── mt -> busybox
│   ├── mv -> busybox
│   ├── netstat -> busybox
│   ├── nice -> busybox
│   ├── nuke -> busybox
│   ├── pidof -> busybox
│   ├── ping -> busybox
│   ├── ping6 -> busybox
│   ├── pipe_progress -> busybox
│   ├── printenv -> busybox
│   ├── ps -> busybox
│   ├── pwd -> busybox
│   ├── reformime -> busybox
│   ├── resume -> busybox
│   ├── rev -> busybox
│   ├── rm -> busybox
│   ├── rmdir -> busybox
│   ├── rpm -> busybox
│   ├── run-parts -> busybox
│   ├── scriptreplay -> busybox
│   ├── sed -> busybox
│   ├── setarch -> busybox
│   ├── setpriv -> busybox
│   ├── setserial -> busybox
│   ├── sh -> busybox
│   ├── sleep -> busybox
│   ├── stat -> busybox
│   ├── stty -> busybox
│   ├── su -> busybox
│   ├── sync -> busybox
│   ├── tar -> busybox
│   ├── touch -> busybox
│   ├── true -> busybox
│   ├── umount -> busybox
│   ├── uname -> busybox
│   ├── usleep -> busybox
│   ├── vi -> busybox
│   ├── watch -> busybox
│   └── zcat -> busybox
├── boot
│   └── Image
├── dev
├── etc
│   └── init.d
│       └── rcS
├── linuxrc -> bin/busybox
├── lost+found [error opening dir]
├── proc
├── sbin
│   ├── acpid -> ../bin/busybox
│   ├── adjtimex -> ../bin/busybox
│   ├── arp -> ../bin/busybox
│   ├── blkid -> ../bin/busybox
│   ├── blockdev -> ../bin/busybox
│   ├── bootchartd -> ../bin/busybox
│   ├── depmod -> ../bin/busybox
│   ├── devmem -> ../bin/busybox
│   ├── fbsplash -> ../bin/busybox
│   ├── fdisk -> ../bin/busybox
│   ├── findfs -> ../bin/busybox
│   ├── freeramdisk -> ../bin/busybox
│   ├── fsck -> ../bin/busybox
│   ├── fsck.minix -> ../bin/busybox
│   ├── fstrim -> ../bin/busybox
│   ├── getty -> ../bin/busybox
│   ├── halt -> ../bin/busybox
│   ├── hdparm -> ../bin/busybox
│   ├── hwclock -> ../bin/busybox
│   ├── ifconfig -> ../bin/busybox
│   ├── ifdown -> ../bin/busybox
│   ├── ifenslave -> ../bin/busybox
│   ├── ifup -> ../bin/busybox
│   ├── init -> ../bin/busybox
│   ├── insmod -> ../bin/busybox
│   ├── ip -> ../bin/busybox
│   ├── ipaddr -> ../bin/busybox
│   ├── iplink -> ../bin/busybox
│   ├── ipneigh -> ../bin/busybox
│   ├── iproute -> ../bin/busybox
│   ├── iprule -> ../bin/busybox
│   ├── iptunnel -> ../bin/busybox
│   ├── klogd -> ../bin/busybox
│   ├── loadkmap -> ../bin/busybox
│   ├── logread -> ../bin/busybox
│   ├── losetup -> ../bin/busybox
│   ├── lsmod -> ../bin/busybox
│   ├── makedevs -> ../bin/busybox
│   ├── mdev -> ../bin/busybox
│   ├── mkdosfs -> ../bin/busybox
│   ├── mke2fs -> ../bin/busybox
│   ├── mkfs.ext2 -> ../bin/busybox
│   ├── mkfs.minix -> ../bin/busybox
│   ├── mkfs.vfat -> ../bin/busybox
│   ├── mkswap -> ../bin/busybox
│   ├── modinfo -> ../bin/busybox
│   ├── modprobe -> ../bin/busybox
│   ├── nameif -> ../bin/busybox
│   ├── pivot_root -> ../bin/busybox
│   ├── poweroff -> ../bin/busybox
│   ├── raidautorun -> ../bin/busybox
│   ├── reboot -> ../bin/busybox
│   ├── rmmod -> ../bin/busybox
│   ├── route -> ../bin/busybox
│   ├── run-init -> ../bin/busybox
│   ├── runlevel -> ../bin/busybox
│   ├── setconsole -> ../bin/busybox
│   ├── slattach -> ../bin/busybox
│   ├── start-stop-daemon -> ../bin/busybox
│   ├── sulogin -> ../bin/busybox
│   ├── swapoff -> ../bin/busybox
│   ├── swapon -> ../bin/busybox
│   ├── switch_root -> ../bin/busybox
│   ├── sysctl -> ../bin/busybox
│   ├── syslogd -> ../bin/busybox
│   ├── tc -> ../bin/busybox
│   ├── tunctl -> ../bin/busybox
│   ├── udhcpc -> ../bin/busybox
│   ├── uevent -> ../bin/busybox
│   ├── vconfig -> ../bin/busybox
│   ├── watchdog -> ../bin/busybox
│   └── zcip -> ../bin/busybox
├── sys
└── usr
    ├── bin
    │   ├── [ -> ../../bin/busybox
    │   ├── [[ -> ../../bin/busybox
    │   ├── awk -> ../../bin/busybox
    │   ├── basename -> ../../bin/busybox
    │   ├── bc -> ../../bin/busybox
    │   ├── beep -> ../../bin/busybox
    │   ├── blkdiscard -> ../../bin/busybox
    │   ├── bunzip2 -> ../../bin/busybox
    │   ├── bzcat -> ../../bin/busybox
    │   ├── bzip2 -> ../../bin/busybox
    │   ├── cal -> ../../bin/busybox
    │   ├── chpst -> ../../bin/busybox
    │   ├── chrt -> ../../bin/busybox
    │   ├── chvt -> ../../bin/busybox
    │   ├── cksum -> ../../bin/busybox
    │   ├── clear -> ../../bin/busybox
    │   ├── cmp -> ../../bin/busybox
    │   ├── comm -> ../../bin/busybox
    │   ├── crontab -> ../../bin/busybox
    │   ├── cryptpw -> ../../bin/busybox
    │   ├── cut -> ../../bin/busybox
    │   ├── dc -> ../../bin/busybox
    │   ├── deallocvt -> ../../bin/busybox
    │   ├── diff -> ../../bin/busybox
    │   ├── dirname -> ../../bin/busybox
    │   ├── dos2unix -> ../../bin/busybox
    │   ├── dpkg -> ../../bin/busybox
    │   ├── dpkg-deb -> ../../bin/busybox
    │   ├── du -> ../../bin/busybox
    │   ├── dumpleases -> ../../bin/busybox
    │   ├── eject -> ../../bin/busybox
    │   ├── env -> ../../bin/busybox
    │   ├── envdir -> ../../bin/busybox
    │   ├── envuidgid -> ../../bin/busybox
    │   ├── expand -> ../../bin/busybox
    │   ├── expr -> ../../bin/busybox
    │   ├── factor -> ../../bin/busybox
    │   ├── fallocate -> ../../bin/busybox
    │   ├── fgconsole -> ../../bin/busybox
    │   ├── find -> ../../bin/busybox
    │   ├── flock -> ../../bin/busybox
    │   ├── fold -> ../../bin/busybox
    │   ├── free -> ../../bin/busybox
    │   ├── ftpget -> ../../bin/busybox
    │   ├── ftpput -> ../../bin/busybox
    │   ├── fuser -> ../../bin/busybox
    │   ├── groups -> ../../bin/busybox
    │   ├── hd -> ../../bin/busybox
    │   ├── head -> ../../bin/busybox
    │   ├── hexdump -> ../../bin/busybox
    │   ├── hexedit -> ../../bin/busybox
    │   ├── hostid -> ../../bin/busybox
    │   ├── id -> ../../bin/busybox
    │   ├── install -> ../../bin/busybox
    │   ├── ipcrm -> ../../bin/busybox
    │   ├── ipcs -> ../../bin/busybox
    │   ├── killall -> ../../bin/busybox
    │   ├── last -> ../../bin/busybox
    │   ├── less -> ../../bin/busybox
    │   ├── logger -> ../../bin/busybox
    │   ├── logname -> ../../bin/busybox
    │   ├── lpq -> ../../bin/busybox
    │   ├── lpr -> ../../bin/busybox
    │   ├── lsof -> ../../bin/busybox
    │   ├── lspci -> ../../bin/busybox
    │   ├── lsscsi -> ../../bin/busybox
    │   ├── lsusb -> ../../bin/busybox
    │   ├── lzcat -> ../../bin/busybox
    │   ├── lzma -> ../../bin/busybox
    │   ├── man -> ../../bin/busybox
    │   ├── md5sum -> ../../bin/busybox
    │   ├── mesg -> ../../bin/busybox
    │   ├── microcom -> ../../bin/busybox
    │   ├── mkfifo -> ../../bin/busybox
    │   ├── mkpasswd -> ../../bin/busybox
    │   ├── nc -> ../../bin/busybox
    │   ├── nl -> ../../bin/busybox
    │   ├── nmeter -> ../../bin/busybox
    │   ├── nohup -> ../../bin/busybox
    │   ├── nproc -> ../../bin/busybox
    │   ├── nsenter -> ../../bin/busybox
    │   ├── nslookup -> ../../bin/busybox
    │   ├── od -> ../../bin/busybox
    │   ├── openvt -> ../../bin/busybox
    │   ├── passwd -> ../../bin/busybox
    │   ├── paste -> ../../bin/busybox
    │   ├── patch -> ../../bin/busybox
    │   ├── pgrep -> ../../bin/busybox
    │   ├── pkill -> ../../bin/busybox
    │   ├── pmap -> ../../bin/busybox
    │   ├── printf -> ../../bin/busybox
    │   ├── pscan -> ../../bin/busybox
    │   ├── pstree -> ../../bin/busybox
    │   ├── pwdx -> ../../bin/busybox
    │   ├── readlink -> ../../bin/busybox
    │   ├── realpath -> ../../bin/busybox
    │   ├── renice -> ../../bin/busybox
    │   ├── reset -> ../../bin/busybox
    │   ├── resize -> ../../bin/busybox
    │   ├── rpm2cpio -> ../../bin/busybox
    │   ├── runsv -> ../../bin/busybox
    │   ├── runsvdir -> ../../bin/busybox
    │   ├── rx -> ../../bin/busybox
    │   ├── script -> ../../bin/busybox
    │   ├── seq -> ../../bin/busybox
    │   ├── setfattr -> ../../bin/busybox
    │   ├── setkeycodes -> ../../bin/busybox
    │   ├── setsid -> ../../bin/busybox
    │   ├── setuidgid -> ../../bin/busybox
    │   ├── sha1sum -> ../../bin/busybox
    │   ├── sha256sum -> ../../bin/busybox
    │   ├── sha3sum -> ../../bin/busybox
    │   ├── sha512sum -> ../../bin/busybox
    │   ├── showkey -> ../../bin/busybox
    │   ├── shred -> ../../bin/busybox
    │   ├── shuf -> ../../bin/busybox
    │   ├── smemcap -> ../../bin/busybox
    │   ├── softlimit -> ../../bin/busybox
    │   ├── sort -> ../../bin/busybox
    │   ├── split -> ../../bin/busybox
    │   ├── ssl_client -> ../../bin/busybox
    │   ├── strings -> ../../bin/busybox
    │   ├── sum -> ../../bin/busybox
    │   ├── sv -> ../../bin/busybox
    │   ├── svc -> ../../bin/busybox
    │   ├── svok -> ../../bin/busybox
    │   ├── tac -> ../../bin/busybox
    │   ├── tail -> ../../bin/busybox
    │   ├── taskset -> ../../bin/busybox
    │   ├── tcpsvd -> ../../bin/busybox
    │   ├── tee -> ../../bin/busybox
    │   ├── telnet -> ../../bin/busybox
    │   ├── test -> ../../bin/busybox
    │   ├── tftp -> ../../bin/busybox
    │   ├── time -> ../../bin/busybox
    │   ├── timeout -> ../../bin/busybox
    │   ├── top -> ../../bin/busybox
    │   ├── tr -> ../../bin/busybox
    │   ├── traceroute -> ../../bin/busybox
    │   ├── traceroute6 -> ../../bin/busybox
    │   ├── truncate -> ../../bin/busybox
    │   ├── ts -> ../../bin/busybox
    │   ├── tty -> ../../bin/busybox
    │   ├── ttysize -> ../../bin/busybox
    │   ├── udhcpc6 -> ../../bin/busybox
    │   ├── udpsvd -> ../../bin/busybox
    │   ├── unexpand -> ../../bin/busybox
    │   ├── uniq -> ../../bin/busybox
    │   ├── unix2dos -> ../../bin/busybox
    │   ├── unlink -> ../../bin/busybox
    │   ├── unlzma -> ../../bin/busybox
    │   ├── unshare -> ../../bin/busybox
    │   ├── unxz -> ../../bin/busybox
    │   ├── unzip -> ../../bin/busybox
    │   ├── uptime -> ../../bin/busybox
    │   ├── users -> ../../bin/busybox
    │   ├── uudecode -> ../../bin/busybox
    │   ├── uuencode -> ../../bin/busybox
    │   ├── vlock -> ../../bin/busybox
    │   ├── volname -> ../../bin/busybox
    │   ├── w -> ../../bin/busybox
    │   ├── wall -> ../../bin/busybox
    │   ├── wc -> ../../bin/busybox
    │   ├── wget -> ../../bin/busybox
    │   ├── which -> ../../bin/busybox
    │   ├── who -> ../../bin/busybox
    │   ├── whoami -> ../../bin/busybox
    │   ├── whois -> ../../bin/busybox
    │   ├── xargs -> ../../bin/busybox
    │   ├── xxd -> ../../bin/busybox
    │   ├── xz -> ../../bin/busybox
    │   ├── xzcat -> ../../bin/busybox
    │   └── yes -> ../../bin/busybox
    └── sbin
        ├── addgroup -> ../../bin/busybox
        ├── add-shell -> ../../bin/busybox
        ├── adduser -> ../../bin/busybox
        ├── arping -> ../../bin/busybox
        ├── brctl -> ../../bin/busybox
        ├── chat -> ../../bin/busybox
        ├── chpasswd -> ../../bin/busybox
        ├── chroot -> ../../bin/busybox
        ├── crond -> ../../bin/busybox
        ├── delgroup -> ../../bin/busybox
        ├── deluser -> ../../bin/busybox
        ├── dhcprelay -> ../../bin/busybox
        ├── dnsd -> ../../bin/busybox
        ├── ether-wake -> ../../bin/busybox
        ├── fakeidentd -> ../../bin/busybox
        ├── fbset -> ../../bin/busybox
        ├── fdformat -> ../../bin/busybox
        ├── fsfreeze -> ../../bin/busybox
        ├── ftpd -> ../../bin/busybox
        ├── httpd -> ../../bin/busybox
        ├── i2cdetect -> ../../bin/busybox
        ├── i2cdump -> ../../bin/busybox
        ├── i2cget -> ../../bin/busybox
        ├── i2cset -> ../../bin/busybox
        ├── i2ctransfer -> ../../bin/busybox
        ├── ifplugd -> ../../bin/busybox
        ├── inetd -> ../../bin/busybox
        ├── killall5 -> ../../bin/busybox
        ├── loadfont -> ../../bin/busybox
        ├── lpd -> ../../bin/busybox
        ├── mim -> ../../bin/busybox
        ├── nanddump -> ../../bin/busybox
        ├── nandwrite -> ../../bin/busybox
        ├── nbd-client -> ../../bin/busybox
        ├── nologin -> ../../bin/busybox
        ├── ntpd -> ../../bin/busybox
        ├── partprobe -> ../../bin/busybox
        ├── popmaildir -> ../../bin/busybox
        ├── powertop -> ../../bin/busybox
        ├── rdate -> ../../bin/busybox
        ├── rdev -> ../../bin/busybox
        ├── readahead -> ../../bin/busybox
        ├── readprofile -> ../../bin/busybox
        ├── remove-shell -> ../../bin/busybox
        ├── rtcwake -> ../../bin/busybox
        ├── sendmail -> ../../bin/busybox
        ├── setfont -> ../../bin/busybox
        ├── setlogcons -> ../../bin/busybox
        ├── svlogd -> ../../bin/busybox
        ├── telnetd -> ../../bin/busybox
        ├── tftpd -> ../../bin/busybox
        ├── ubiattach -> ../../bin/busybox
        ├── ubidetach -> ../../bin/busybox
        ├── ubimkvol -> ../../bin/busybox
        ├── ubirename -> ../../bin/busybox
        ├── ubirmvol -> ../../bin/busybox
        ├── ubirsvol -> ../../bin/busybox
        ├── ubiupdatevol -> ../../bin/busybox
        └── udhcpd -> ../../bin/busybox
```

取消镜像挂载

```bash
$ sudo umount rootfs
```

### qemu配置uboot启动

```bash
$ qemu-system-aarch64 -machine virt,virtualization=true,gic-version=3 -nographic -m size=1024M -cpu cortex-a72 -smp 2 -bios ./uboot/u-boot-2022.10/object/u-boot.bin -drive format=raw,file=./linux/run/rootfs.img,format=raw,id=hd0,if=none -device virtio-blk-device,drive=hd0


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
starting USB...
No working controllers found
USB is stopped. Please issue 'usb start' first.
scanning bus for devices...

Device 0: unknown device

Device 0: QEMU VirtIO Block Device
            Type: Hard Disk
            Capacity: 512.0 MB = 0.5 GB (1048576 x 512)
... is now current device
** No partition table - virtio 0 **
Couldn't find partition virtio 0:1

Device 0: unknown device
starting USB...
No working controllers found
BOOTP broadcast 1
DHCP client bound to address 10.0.2.15 (3 ms)
Using virtio-net#32 device
TFTP from server 10.0.2.2; our IP address is 10.0.2.15
Filename 'boot.scr.uimg'.
Load address: 0x40200000
Loading: *
TFTP error: 'Access violation' (2)
Not retrying...
BOOTP broadcast 1
DHCP client bound to address 10.0.2.15 (0 ms)
Using virtio-net#32 device
TFTP from server 10.0.2.2; our IP address is 10.0.2.15
Filename 'boot.scr.uimg'.
Load address: 0x40400000
Loading: *
TFTP error: 'Access violation' (2)
Not retrying...
=> 
```

### 加载Image

在uboot命令行中输入

```bash
> load virtio 0 0x40200000 boot/Image
```

设置启动参数

```bash
> setenv bootargs "root=/dev/vda rw"
```

或者

```bash
> setenv bootargs "root=/dev/vda rw console=ttyAMA0,115200 init=/linuxrc"
```

启动Linux

```bash
> booti 0x40200000 - $fdtcontroladdr
## Flattened Device Tree blob at 7edd3db0
   Booting using the fdt blob at 0x7edd3db0
   Loading Device Tree to 000000007eccc000, end 000000007edcefff ... OK

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd083]
[    0.000000] Linux version 5.10.0 (test@test) (aarch64-linux-gnu-gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, GNU ld (GNU Binutils for Ubuntu) 2.34) #1 SMP PREEMPT Wed Oct 29 09:42:10 CST 2025
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 32 MiB at 0x000000007cc00000
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x7fdf5b00-0x7fdf7fff]
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000040000000-0x000000007fffffff]
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv0.2 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] percpu: Embedded 23 pages/cpu s56216 r8192 d29800 u94208
[    0.000000] Detected PIPT I-cache on CPU0
[    0.000000] CPU features: detected: GIC system register CPU interface
[    0.000000] CPU features: detected: EL2 vector hardening
[    0.000000] CPU features: detected: Spectre-v2
[    0.000000] CPU features: detected: Spectre-v4
[    0.000000] CPU features: detected: ARM errata 1165522, 1319367, or 1530923
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 258048
[    0.000000] Policy zone: DMA
[    0.000000] Kernel command line: root=/dev/vda rw console=ttyAMA0,115200 init=/linuxrc
[    0.000000] Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
[    0.000000] Inode-cache hash table entries: 65536 (order: 7, 524288 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 964372K/1048576K available (14208K kernel code, 2796K rwdata, 7592K rodata, 5952K init, 518K bss, 51436K reserved, 32768K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=2, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU event tracing is enabled.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=2.
[    0.000000] 	Trampoline variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=2
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] GICv3: GIC: Using split EOI/Deactivate mode
[    0.000000] GICv3: 224 SPIs implemented
[    0.000000] GICv3: 0 Extended SPIs implemented
[    0.000000] GICv3: Distributor has no Range Selector support
[    0.000000] GICv3: 16 PPIs implemented
[    0.000000] GICv3: CPU0: found redistributor 0 region 0:0x00000000080a0000
[    0.000000] random: get_random_bytes called from start_kernel+0x32c/0x4f0 with crng_init=0
[    0.000000] arch_timer: cp15 timer(s) running at 62.50MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x1cd42e208c, max_idle_ns: 881590405314 ns
[    0.000276] sched_clock: 56 bits at 62MHz, resolution 16ns, wraps every 4398046511096ns
[    0.012314] Console: colour dummy device 80x25
[    0.016984] Calibrating delay loop (skipped), value calculated using timer frequency.. 125.00 BogoMIPS (lpj=250000)
[    0.017331] pid_max: default: 32768 minimum: 301
[    0.019704] LSM: Security Framework initializing
[    0.021729] Mount-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.021864] Mountpoint-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.094299] rcu: Hierarchical SRCU implementation.
[    0.102378] EFI services will not be available.
[    0.105751] smp: Bringing up secondary CPUs ...
[    0.114581] Detected PIPT I-cache on CPU1
[    0.115392] GICv3: CPU1: found redistributor 1 region 0:0x00000000080c0000
[    0.115674] CPU1: Booted secondary processor 0x0000000001 [0x410fd083]
[    0.120014] smp: Brought up 1 node, 2 CPUs
[    0.120170] SMP: Total of 2 processors activated.
[    0.120375] CPU features: detected: 32-bit EL0 Support
[    0.120565] CPU features: detected: CRC32 instructions
[    0.120661] CPU features: detected: 32-bit EL1 Support
[    0.204550] CPU: All CPU(s) started at EL2
[    0.205323] alternatives: patching kernel code
[    0.243196] devtmpfs: initialized
[    0.265421] KASLR disabled due to lack of seed
[    0.274065] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.274413] futex hash table entries: 512 (order: 3, 32768 bytes, linear)
[    0.280048] pinctrl core: initialized pinctrl subsystem
[    0.301551] DMI not present or invalid.
[    0.318530] NET: Registered protocol family 16
[    0.342186] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
[    0.342545] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.344036] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.344643] audit: initializing netlink subsys (disabled)
[    0.348923] audit: type=2000 audit(0.280:1): state=initialized audit_enabled=0 res=1
[    0.361922] thermal_sys: Registered thermal governor 'step_wise'
[    0.362064] thermal_sys: Registered thermal governor 'power_allocator'
[    0.363304] cpuidle: using governor menu
[    0.364988] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.366799] ASID allocator initialised with 65536 entries
[    0.373949] Serial: AMBA PL011 UART driver
[    0.461283] 9000000.pl011: ttyAMA0 at MMIO 0x9000000 (irq = 47, base_baud = 0) is a PL011 rev1
[    0.517701] printk: console [ttyAMA0] enabled
[    0.596521] HugeTLB registered 1.00 GiB page size, pre-allocated 0 pages
[    0.597028] HugeTLB registered 32.0 MiB page size, pre-allocated 0 pages
[    0.597597] HugeTLB registered 2.00 MiB page size, pre-allocated 0 pages
[    0.597929] HugeTLB registered 64.0 KiB page size, pre-allocated 0 pages
[    0.624093] cryptd: max_cpu_qlen set to 1000
[    0.658361] ACPI: Interpreter disabled.
[    0.669418] iommu: Default domain type: Translated 
[    0.672458] vgaarb: loaded
[    0.675673] SCSI subsystem initialized
[    0.681337] usbcore: registered new interface driver usbfs
[    0.682544] usbcore: registered new interface driver hub
[    0.683439] usbcore: registered new device driver usb
[    0.688925] pps_core: LinuxPPS API ver. 1 registered
[    0.689278] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.690009] PTP clock support registered
[    0.691322] EDAC MC: Ver: 3.0.0
[    0.701643] FPGA manager framework
[    0.703905] Advanced Linux Sound Architecture Driver Initialized.
[    0.736623] clocksource: Switched to clocksource arch_sys_counter
[    0.740009] VFS: Disk quotas dquot_6.6.0
[    0.741524] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.749327] pnp: PnP ACPI: disabled
[    0.838847] NET: Registered protocol family 2
[    0.849867] tcp_listen_portaddr_hash hash table entries: 512 (order: 1, 8192 bytes, linear)
[    0.850812] TCP established hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.852242] TCP bind hash table entries: 8192 (order: 5, 131072 bytes, linear)
[    0.854631] TCP: Hash tables configured (established 8192 bind 8192)
[    0.858571] UDP hash table entries: 512 (order: 2, 16384 bytes, linear)
[    0.859814] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes, linear)
[    0.864114] NET: Registered protocol family 1
[    0.875888] RPC: Registered named UNIX socket transport module.
[    0.876830] RPC: Registered udp transport module.
[    0.877698] RPC: Registered tcp transport module.
[    0.878422] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    0.879618] PCI: CLS 0 bytes, default 64
[    0.899605] hw perfevents: enabled with armv8_pmuv3 PMU driver, 5 counters available
[    0.901990] kvm [1]: IPA Size Limit: 44 bits
[    0.911754] kvm [1]: GICv3: no GICV resource entry
[    0.913190] kvm [1]: disabling GICv2 emulation
[    0.914348] kvm [1]: GIC system register CPU interface enabled
[    0.916955] kvm [1]: vgic interrupt IRQ9
[    0.920251] kvm [1]: Hyp mode initialized successfully
[    0.966113] Initialise system trusted keyrings
[    0.970482] workingset: timestamp_bits=42 max_order=18 bucket_order=0
[    0.999625] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    1.010594] NFS: Registering the id_resolver key type
[    1.013532] Key type id_resolver registered
[    1.014155] Key type id_legacy registered
[    1.016026] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    1.020199] 9p: Installing v9fs 9p2000 file system support
[    1.109828] Key type asymmetric registered
[    1.110580] Asymmetric key parser 'x509' registered
[    1.111782] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 245)
[    1.112675] io scheduler mq-deadline registered
[    1.113082] io scheduler kyber registered
[    1.161804] pci-host-generic 4010000000.pcie: host bridge /pcie@10000000 ranges:
[    1.163640] pci-host-generic 4010000000.pcie:       IO 0x003eff0000..0x003effffff -> 0x0000000000
[    1.164948] pci-host-generic 4010000000.pcie:      MEM 0x0010000000..0x003efeffff -> 0x0010000000
[    1.165556] pci-host-generic 4010000000.pcie:      MEM 0x8000000000..0xffffffffff -> 0x8000000000
[    1.167731] pci-host-generic 4010000000.pcie: ECAM at [mem 0x4010000000-0x401fffffff] for [bus 00-ff]
[    1.171465] pci-host-generic 4010000000.pcie: PCI host bridge to bus 0000:00
[    1.172506] pci_bus 0000:00: root bus resource [bus 00-ff]
[    1.172973] pci_bus 0000:00: root bus resource [io  0x0000-0xffff]
[    1.173596] pci_bus 0000:00: root bus resource [mem 0x10000000-0x3efeffff]
[    1.173951] pci_bus 0000:00: root bus resource [mem 0x8000000000-0xffffffffff]
[    1.176987] pci 0000:00:00.0: [1b36:0008] type 00 class 0x060000
[    1.185194] pci 0000:00:01.0: [1af4:1000] type 00 class 0x020000
[    1.187256] pci 0000:00:01.0: reg 0x10: [io  0x1000-0x101f]
[    1.191292] pci 0000:00:01.0: reg 0x20: [mem 0x10000000-0x10003fff 64bit pref]
[    1.192583] pci 0000:00:01.0: reg 0x30: [mem 0x10080000-0x100fffff pref]
[    1.199443] pci 0000:00:01.0: BAR 6: assigned [mem 0x10000000-0x1007ffff pref]
[    1.200571] pci 0000:00:01.0: BAR 4: assigned [mem 0x8000000000-0x8000003fff 64bit pref]
[    1.201598] pci 0000:00:01.0: BAR 0: assigned [io  0x1000-0x101f]
[    1.215261] EINJ: ACPI disabled.
[    1.298258] Serial: 8250/16550 driver, 4 ports, IRQ sharing enabled
[    1.310831] SuperH (H)SCI(F) driver initialized
[    1.313131] msm_serial: driver initialized
[    1.321543] cacheinfo: Unable to detect cache hierarchy for CPU 0
[    1.390487] loop: module loaded
[    1.417673] virtio_blk virtio0: [vda] 1048576 512-byte logical blocks (537 MB/512 MiB)
[    1.418570] vda: detected capacity change from 0 to 536870912
[    1.471182] megasas: 07.714.04.00-rc1
[    1.483862] physmap-flash 0.flash: physmap platform flash device: [mem 0x00000000-0x03ffffff]
[    1.490533] 0.flash: Found 2 x16 devices at 0x0 in 32-bit bank. Manufacturer ID 0x000000 Chip ID 0x000000
[    1.493771] Intel/Sharp Extended Query Table at 0x0031
[    1.495882] Using buffer write method
[    1.497994] physmap-flash 0.flash: physmap platform flash device: [mem 0x04000000-0x07ffffff]
[    1.499988] 0.flash: Found 2 x16 devices at 0x0 in 32-bit bank. Manufacturer ID 0x000000 Chip ID 0x000000
[    1.501761] Intel/Sharp Extended Query Table at 0x0031
[    1.503287] Using buffer write method
[    1.504850] Concatenating MTD devices:
[    1.505689] (0): "0.flash"
[    1.506029] (1): "0.flash"
[    1.506338] into device "0.flash"
[    1.586642] libphy: Fixed MDIO Bus: probed
[    1.591006] tun: Universal TUN/TAP device driver, 1.6
[    1.609644] thunder_xcv, ver 1.0
[    1.610145] thunder_bgx, ver 1.0
[    1.610482] nicpf, ver 1.0
[    1.615740] hclge is initializing
[    1.616441] hns3: Hisilicon Ethernet Network Driver for Hip08 Family - version
[    1.617294] hns3: Copyright (c) 2017 Huawei Corporation.
[    1.618154] e1000: Intel(R) PRO/1000 Network Driver
[    1.618414] e1000: Copyright (c) 1999-2006 Intel Corporation.
[    1.619190] e1000e: Intel(R) PRO/1000 Network Driver
[    1.619462] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    1.619923] igb: Intel(R) Gigabit Ethernet Network Driver
[    1.620139] igb: Copyright (c) 2007-2014 Intel Corporation.
[    1.620754] igbvf: Intel(R) Gigabit Virtual Function Network Driver
[    1.621079] igbvf: Copyright (c) 2009 - 2012 Intel Corporation.
[    1.622310] sky2: driver version 1.30
[    1.627905] VFIO - User Level meta-driver version: 0.3
[    1.638894] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    1.640489] ehci-pci: EHCI PCI platform driver
[    1.641539] ehci-platform: EHCI generic platform driver
[    1.642634] ehci-orion: EHCI orion driver
[    1.643479] ehci-exynos: EHCI Exynos driver
[    1.644379] ohci_hcd: USB 1.1 'Open' Host Controller (OHCI) Driver
[    1.644967] ohci-pci: OHCI PCI platform driver
[    1.645675] ohci-platform: OHCI generic platform driver
[    1.646436] ohci-exynos: OHCI Exynos driver
[    1.649309] usbcore: registered new interface driver usb-storage
[    1.668917] rtc-pl031 9010000.pl031: registered as rtc0
[    1.670960] rtc-pl031 9010000.pl031: setting system clock to 2025-10-29T09:21:26 UTC (1761729686)
[    1.674957] i2c /dev entries driver
[    1.697260] sdhci: Secure Digital Host Controller Interface driver
[    1.698014] sdhci: Copyright(c) Pierre Ossman
[    1.700628] Synopsys Designware Multimedia Card Interface Driver
[    1.705231] sdhci-pltfm: SDHCI platform and OF driver helper
[    1.714635] ledtrig-cpu: registered to indicate activity on CPUs
[    1.722851] usbcore: registered new interface driver usbhid
[    1.723759] usbhid: USB HID core driver
[    1.750142] NET: Registered protocol family 17
[    1.755088] 9pnet: Installing 9P2000 support
[    1.756519] Key type dns_resolver registered
[    1.761346] registered taskstats version 1
[    1.762170] Loading compiled-in X.509 certificates
[    1.775930] ALSA device list:
[    1.776877]   No soundcards found.
[    1.786250] uart-pl011 9000000.pl011: no DMA platform data
[    1.885599] EXT4-fs (vda): recovery complete
[    1.889106] EXT4-fs (vda): mounted filesystem with ordered data mode. Opts: (null)
[    1.890048] VFS: Mounted root (ext4 filesystem) on device 254:0.
[    1.896188] devtmpfs: mounted
[    1.979359] Freeing unused kernel memory: 5952K
[    1.983803] Run /linuxrc as init process

Please press Enter to activate this console. 
/ # 
/ #
```





# 常见问题

## 清理kernel编译残留

```bash
# 删除大多数的编译生成文件， 但是会保留内核的配置文件.config， 还有足够的编译支持来建立扩展模块
make clean

# 删除所有的编译生成文件， 还有内核配置文件， 再加上各种备份文件
make mrproper

# 包含mrproper的所有清理内容 + 额外移除编辑器备份文件和补丁文件
make distclean
```

## arm64架构的uImage制作

当前的arm64体系架构已经不支持zImage和uImage的编译目标，可使用mkimage工具给不经压缩的Image镜像加上uboot头部信息，生成uImage启动镜像，由u-boot来启动。

## qemu virt通用虚拟平台硬件信息

[Hardware configuration information for bare-metal programming](https://www.qemu.org/docs/master/system/arm/virt.html#hardware-configuration-information-for-bare-metal-programming)

# 参考

[qemu搭建arm64 linux kernel环境](https://zhuanlan.zhihu.com/p/667525514)

[QEMU学习之路（7）— ARM64 启动Linux](https://blog.csdn.net/qq_38113006/article/details/147159555)

[qemu模拟vexpress编译linux内核(三)](https://zhuanlan.zhihu.com/p/686780668)

[qemu 之 uboot、linux 启动](https://blog.csdn.net/feitingfj/article/details/133964595)

[ARM64平台的QEMU环境搭建](https://zhuanlan.zhihu.com/p/1925233204899602729)

