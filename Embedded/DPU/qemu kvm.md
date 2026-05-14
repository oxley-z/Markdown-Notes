



## kvm虚拟化技术

### KVM介绍

KVM是Linux开源社区大力支持的虚拟化技术，基于Intel和AMD的硬件虚拟化技术。KVM（Kernel-bashdVirtual Machine，即基于内核的虚拟机），它是用于Linux内核中的虚拟化环境设施，是Linux内核中的一个功能模块，在Linux内核中默认被安装，可以将Linux内核转化为一个[Hypervisor](https://zh.m.wikipedia.org/zh-hans/Hypervisor)[^hypervisor]。

KVM支持页表虚拟化和IOMMU虚拟化技术，页表虚拟化技术用于将GuestOS的页表转换为HostOS运行的环境中，在这种机制下，KVM的HostOS将要接管所有的内存分配(其实也是使用Linux 内核的内存管理能力)。

## QEMU

qemu是一个便捷的跨平台开源计算机模拟器，可以模拟许多硬件体系结构，包括ARM、RISC-V等，qemu可以在不同的机器上运行独自开发的操作系统与软件，经过kqemu加速后，能模拟至接近真是电脑的速度。qemu在GNU/Linux平台上使用较为广泛，并且能够运行于多种操作系统中和不同的CPU体系架构中，允许在虚拟机运行时保存虚拟机的状态，进行实时迁移，进行操作系统级别的调试，也能模拟硬件的相关案例。

![关于qemu的简介](image/qemu%20kvm/143910.png)

### 何为qemu-kvm

所谓kvm技术中，应用到两个技术，分别是：qemu、kvm，其中kvm负责CPU虚拟化+内存虚拟化，实现了CPU和内存的虚拟化，但kvm不能模拟其他设备；qemu是模拟IO设备（网卡、磁盘等），kvm加上qemu之后就能实现真正的服务器虚拟化，故称之为qemu-kvm。由于kvm技术已经相当成熟，并且对很多方面都进行了隔离，但是像网卡、磁盘等设备依然无法虚拟出真是的机器。qemu-kvm补充了kvm技术的不足，而且在性能上对kvm进行了优化。

## Ubuntu20.04部署KVM

查询本机是否支持虚拟化技术。

```bash
 eddy@eddy  ~  cat /proc/cpuinfo | grep -E 'vmx|svm'
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid tsc_known_freq pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti ssbd ibrs ibpb stibp tpr_shadow vnmi ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap xsaveopt arat md_clear flush_l1d arch_capabilities
vmx flags       : vnmi invvpid ept_x_only ept_ad tsc_offset vtpr mtf ept vpid unrestricted_guest ple
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid tsc_known_freq pni pclmulqdq vmx ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti ssbd ibrs ibpb stibp tpr_shadow vnmi ept vpid ept_ad fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap xsaveopt arat md_clear flush_l1d arch_capabilities
vmx flags       : vnmi invvpid ept_x_only ept_ad tsc_offset vtpr mtf ept vpid unrestricted_guest ple
 eddy@eddy  ~ （古董机）
```

若使用VMWare安装的虚拟机则需要在虚拟机设置中开启虚拟化技术。

![image-20221115194735985](image/qemu%20kvm/image-20221115194735985.png)

安装KVM和虚拟机管理器等

> qemu-kvm：完整的虚拟化平台。
>
> libvirtd：用于硬件虚拟化的开源API、守护进程与管理工具。
>
> virt-manager：虚拟机管理器

```bash
eddy@eddy:~$ sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
```

查看服务是否激活

```bash
eddy@eddy:~$ systemctl is-active libvirtd
```

查看是否激活

```bash
eddy@eddy:~$ lsmod | grep kvm
```

启动虚拟化

```bash
eddy@eddy:~$ systemctl start libvirtd
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
启动“libvirtd.service”需要认证。
Authenticating as: eddy,,, (eddy)
Password:
==== AUTHENTICATION COMPLETE ===
eddy@eddy:~$
```

查看状态

```bash
eddy@eddy:~$ systemctl status libvirtd
● libvirtd.service - Virtualization daemon
     Loaded: loaded (/lib/systemd/system/libvirtd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-11-16 15:54:03 CST; 27min ago
TriggeredBy: ● libvirtd-admin.socket
             ● libvirtd.socket
             ● libvirtd-ro.socket
       Docs: man:libvirtd(8)
             https://libvirt.org
   Main PID: 5519 (libvirtd)
      Tasks: 19 (limit: 32768)
     Memory: 13.2M
     CGroup: /system.slice/libvirtd.service
             ├─5519 /usr/sbin/libvirtd
             ├─5701 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/u>
             └─5702 /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/u>

Nov 16 15:54:03 eddy systemd[1]: Started Virtualization daemon.
Nov 16 15:54:14 eddy dnsmasq[5701]: started, version 2.80 cachesize 150
Nov 16 15:54:14 eddy dnsmasq[5701]: compile time options: IPv6 GNU-getopt DBus i18n IDN DHCP DHCPv6 no-Lua TFTP conntra>
Nov 16 15:54:14 eddy dnsmasq-dhcp[5701]: DHCP, IP range 192.168.122.2 -- 192.168.122.254, lease time 1h
Nov 16 15:54:14 eddy dnsmasq-dhcp[5701]: DHCP, sockets bound exclusively to interface virbr0
Nov 16 15:54:14 eddy dnsmasq[5701]: reading /etc/resolv.conf
Nov 16 15:54:14 eddy dnsmasq[5701]: using nameserver 127.0.0.53#53
Nov 16 15:54:14 eddy dnsmasq[5701]: read /etc/hosts - 7 addresses
Nov 16 15:54:14 eddy dnsmasq[5701]: read /var/lib/libvirt/dnsmasq/default.addnhosts - 0 addresses
Nov 16 15:54:14 eddy dnsmasq-dhcp[5701]: read /var/lib/libvirt/dnsmasq/default.hostsfile
lines 1-26/26 (END)
```

配置网桥

```bash
eddy@eddy:~$ sudo apt install bridge-utils
```

修改01-network-manager-all.yaml文件

```bash
eddy@eddy:~$ gedit /etc/netplan/01-network-manager-all.yaml
```

文件内容如下

```bash
# Let NetworkManager manage all devices on this system
network:
  version: 2
  #renderer: NetworkManager
  ethernets:
        ens33:(网卡名称)
            dhcp4: false
            dhcp6: false
  bridges:
        br0:
            addresses: [192.168.159.133/24]（本机IP）
            gateway4: 192.168.159.1（网关）
            nameservers:
                addresses: [192.168.159.2, 114.114.114.114]（DNS）
                search: [msnode]
            interfaces: [ens33]（网卡名称）
```

启用配置，并重启网络

```bash
eddy@eddy:~$ sudo netplan apply
```

报错

```bash
eddy@eddy:~$ netplan apply   
/etc/netplan/01-network-manager-all.yaml:9:5: Invalid YAML: inconsistent indentation:
    bridges:
    ^
eddy@eddy:~$ 
```

对`01-network-manager-all.yam`文件的`bridge`和ethernets对齐。

启用配置，并重启网络

```bash
eddy@eddy:~$ sudo netplan apply
```

ifconfig查看网络

```bash
eddy@eddy:~$ ifconfig
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.159.131  netmask 255.255.255.0  broadcast 192.168.159.255
        inet6 fe80::20c:29ff:fe7c:49dd  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:7c:49:dd  txqueuelen 1000  (以太网)
        RX packets 8  bytes 828 (828.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 39  bytes 4869 (4.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 00:0c:29:7c:49:dd  txqueuelen 1000  (以太网)
        RX packets 32967  bytes 39746636 (39.7 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17563  bytes 1398614 (1.3 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (本地环回)
        RX packets 1908  bytes 140071 (140.0 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1908  bytes 140071 (140.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 52:54:00:52:ad:05  txqueuelen 1000  (以太网)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eddy@eddy:~$ 
```

安装虚拟机

```bash
eddy@eddy:~$ virt-install -n Ubuntu20_VM0 -r 2048 --vcpus=2 --os-variant=ubuntu20.04 --accelerate -c /tmp/ubuntu-20.04.5-live-server-amd64.iso --disk path=/home/vmsets/ubuntu20_04.img,format=qcow2,bus=virtio,size=20 --network bridge=br0 --vnc --vncport=5996 --vnclisten=0.0.0.0
```

报错

```bash
eddy@eddy:~$ virt-install -n Ubuntu20_VM0 -r 2048 --vcpus=2 --os-variant=ubuntu20.04 --accelerate -c /tmp/ubuntu-20.04.5-live-server-amd64.iso --disk path=/home/vmsets/ubuntu20_04.img,format=qcow2,bus=virtio,size=20 --network bridge=br0 --vnc --vncport=5996 --vnclisten=0.0.0.0

开始安装......
正在分配 'ubuntu20_04.img'                                                                                                                                                              |  20 GB  00:00:00     
ERROR    internal error: /usr/lib/qemu/qemu-bridge-helper --use-vnet --br=br0 --fd=30: failed to communicate with bridge helper: Transport endpoint is not connected
stderr=failed to parse default acl file `/etc/qemu/bridge.conf'

正在删除磁盘 'ubuntu20_04.img'                                                                                                                                                            |    0 B  00:00:00     
域安装失败，您可以运行下列命令重启您的域：
'virsh start virsh --connect qemu:///session start Ubuntu20_VM0'
否则请重新开始安装。
eddy@eddy:~$ 
```

创建 /etc/qemu/bridge.conf （qemu用到的网桥配置文件）

```bash
eddy@eddy:~$ mkdir /etc/qemu
eddy@eddy:~$ vim /etc/qemu/bridge.conf
```

键入

```bash
allow br0 (这里的br0跟之前创建的虚拟网桥相同)
allow all
```

更改权限

```bash
eddy@eddy:~$ sudo chmod 777 /etc/qemu/bridge.conf
```

继续创建虚拟机，报错

```bash
eddy@eddy:~$  virt-install -n Ubuntu20_VM0 -r 2048 --vcpus=2 --os-variant=ubuntu20.04 --accelerate -c /tmp/ubuntu-20.04.5-live-server-amd64.iso --disk path=/home/vmsets/ubuntu20_04.img,format=qcow2,bus=virtio,size=20 --network bridge=br0 --vnc --vncport=5996 --vnclisten=0.0.0.0

开始安装......
正在分配 'ubuntu20_04.img'                                                                                                                                                              |  20 GB  00:00:00     
ERROR    internal error: /usr/lib/qemu/qemu-bridge-helper --use-vnet --br=br0 --fd=30: failed to communicate with bridge helper: Transport endpoint is not connected
stderr=failed to create tun device: Operation not permitted

正在删除磁盘 'ubuntu20_04.img'                                                                                                                                                            |    0 B  00:00:00     
域安装失败，您可以运行下列命令重启您的域：
'virsh start virsh --connect qemu:///session start Ubuntu20_VM0'
否则请重新开始安装。
```

解决方法（qemu-bridge-helper需要set uid）

```bash
eddy@eddy:~$  sudo chmod 4755 /usr/lib/qemu/qemu-bridge-helper
```

继续创建虚拟机

<img src="image/qemu%20kvm/image-20221116192241286.png" alt="image-20221116192241286"  />

<img src="image/qemu%20kvm/image-20221116192301851.png" alt="image-20221116192301851"  />

安装Ubuntu Server

![image-20221116192639685](image/qemu%20kvm/image-20221116192639685.png)

安装SSH

输入相应用户名、密码等信息，之后

![image-20221116193320601](image/qemu%20kvm/image-20221116193320601.png)

安装完成选择重启

# 参考

[Ubuntu20.4系统下安装kvm并创建虚拟机_馫馫馫馫香的博客-CSDN博客](https://blog.csdn.net/twotwo22222/article/details/126767604?ops_request_misc=%7B%22request%5Fid%22%3A%22166858422716782425148246%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=166858422716782425148246&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-126767604-null-null.142^v63^control,201^v3^add_ask,213^v2^t3_esquery_v3&utm_term=ubuntu20.04安装kvm&spm=1018.2226.3001.4187)

[如何在 Ubuntu 20.04 上安装 KVM - 武平宁 - 博客园 (cnblogs.com)](https://www.cnblogs.com/dewan/p/16661956.html)





🤑

🎨

😒

[^hypervisor]: 虚拟机监控器（Virtual Machine Monitor，缩写VMM），是用来建立于执行虚拟机的软件。