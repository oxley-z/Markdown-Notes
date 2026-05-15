---
github.io: true
csdn: false
---
# Linux拥塞控制算法配置

在Linux用户态可以**通过参数查看当前使用的拥塞控制算法、当前可支持的拥塞控制算法**。如下表所示是两个参数以及含义。

| 参数                                      | 含义                     |
| ----------------------------------------- | ------------------------ |
| net.ipv4.tcp_congestion_control           | 当前运行的拥塞控制算法   |
| net.ipv4.tcp_available_congestion_control | 当前可支持的拥塞控制算法 |

```bash
#查看当前运行的拥塞控制算法
$ sysctl net.ipv4.tcp_congestion_control
#查看当前可支持的拥塞控制算法
$ sysctl net.ipv4.tcp_available_congestion_control
```

## 查看Linux系统中所有已实现的拥塞控制算法模块：

![image-20230803094446632](image/Linux拥塞控制算法配置/image-20230803094446632.png#pic_center)

## 安装特定拥塞控制算法

```bash
$ modprobe -a tcp_vegas
```

## 动态切换拥塞控制算法

当前使用的拥塞控制算法外**还可以动态切换拥塞控制算法。如下所示将默认的cubic拥塞控制算法切换为bbr拥塞控制算法。**

```bash
$ echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
$ echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
$ sysctl -p
kernel.shmmax = 9223372036854775807
kernel.shmall = 1152921504606846720
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

# 参考

[Linux内核网络-拥塞控制系列(一） (qq.com)](https://mp.weixin.qq.com/s/J_EE4oDxP7MmD7qlp9hL1w)