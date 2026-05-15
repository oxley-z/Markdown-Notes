---
github.io: false
csdn: false
---
# ifconfig

## 显示网络设备信息（激活状态的）

![image-20230919133206486](image/%5BLinux%E5%91%BD%E4%BB%A4%5D-ifconfig/image-20230919133206486.png)

## 设置IP和掩码

```bash
$ ifconfig eth0 192.168.1.4 netmask 255.255.255.0
```

## 修改MAC地址

```bash
$ ifconfig eth0 hw ether 00:AA:BB:CC:dd:EE
```

## 启用混杂模式

混杂允许网络接口访问和查看网络中的所有数据包。您可以使用该ifconfig命令在特定网络设备上启用和禁用混杂。

```bash
$ ifconfig eth0 promisc
```

## 禁用混杂模式

```bash
$ ifconfig eth0 -promisc
```











# 参考

[ifconfig 命令 - IBM 文档](https://www.ibm.com/docs/zh/aix/7.3?topic=i-ifconfig-command)