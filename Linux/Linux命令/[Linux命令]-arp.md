---
github.io: true
csdn: false
---
# arp

| 参数 | 含义 |
| ---- | ---- |
| -a | 显示 arp 缓冲区的所有条目； |
| -d | 从 arp 缓冲区中删除指定主机的 arp 条目； |
| -D | 使用指定接口的硬件地址； |
| -e | 以 Linux 的显示风格显示 arp 缓冲区中的条目； |
| -f | 设置主机的 IP 地址与 MAC 地址的静态映射; |
| -g | 显示ARP缓存列表; |
| -H | 指定 arp 指令使用的地址类型； |
| -i | 指定要操作 arp 缓冲区的网络接口； |
| -n | 以数字方式显示 arp 缓冲区中的条目； |
| -v | 显示详细的 arp 缓冲区条目，包括缓冲区条目的统计信息； |
| -s        | 设置指定的主机的 IP 地址与 MAC 地址的静态映射；       |
| --version | 显示版本信息. |

## 显示本机arp缓存中所有记录

```bash
[root@linuxcool ~]# arp   
Address                  HWtype  HWaddress           Flags Mask            Iface
_gateway                 ether   00:50:56:ff:bf:31   C                     ens33
```

## 以数字方式显示指定主机arp缓存条目

```bash
[root@linuxcool ~]# arp -n 192.168.198.2
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.198.2            ether   00:50:56:ff:bf:31   C                     ens33
```

## 删除指定主机网卡上的arp条目

```bash
[root@linuxcool ~]# arp -i ens33 -d 192.168.198.2
```

## 添加对指定的IP地址和MAC地址进行静态映射条目

```bash
[root@linuxcool ~]# arp -s 192.168.198.2 00:50:56:ff:bf:31
```



# 参考

[arp命令 – 管理系统arp缓存信息 – Linux命令大全(手册) (linuxcool.com)](https://www.linuxcool.com/arp)
