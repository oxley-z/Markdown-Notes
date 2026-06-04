---
author: 点击关注👉👉
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzIxMTgxNzQ2OA==&mid=2247504180&idx=1&sn=ec19c6f7a0ef89b99db059061309ae9a&chksm=96151e4d2b835a6c24e4435b76c669370082eab7ac36a3e74f38e52ae729b03ad2c4f94c0d3c&mpshare=1&scene=1&srcid=0604il8AssunTebzXrdERaLE&sharer_shareinfo=e3c0f87f76025d56e5406f3b6d9d7410&sharer_shareinfo_first=e3c0f87f76025d56e5406f3b6d9d7410#rd
saved: 2026-06-04 08:31:08
tags:
  - 笔记同步助手
id: 1e25b19e-7bdf-4560-a342-0164c0f46eb2
---

公众号名称：浩道Linux

作者名称：点击关注👉👉

发布时间：2026-06-04 08:00

最近甲方项目上遇到不少问题，每次叫甲方客户协助把抓包数据给我，他都是直接回复一句：我不会Linux下抓包！

这让我既无奈又拿他没办法，谁让他是甲方呢！不过我可不惯着他，我对他说：不会抓包，我可以手把手教会你！于是我就把下文整理好的tcpdump抓包技巧丢给他学习，从此他不再来找我了，会自己抓包分析了！

讲真的，这些技能是真的实用，如果在看的你还不会，也抓紧学习下吧！

# 前言

tcpdump 是 Linux 下最常用的网络抓包工具，地位相当于 Windows 下的 Wireshark。它能够截获网络数据包、过滤数据包、按多种条件显示，是排查网络问题、分析协议行为、定位故障根因的必备工具。

很多人只会简单的 `tcpdump port 80`，遇到复杂场景就束手无策。实际上 tcpdump 功能非常强大，配合 shell 脚本和管道可以实现复杂的过滤和分析逻辑。

本文从实际运维场景出发，介绍 20 个高频使用的 tcpdump 命令组合，覆盖基础抓包、过滤条件、输出格式、数据分析、常见故障排查等场景。

## 1 tcpdump 基础

### 1.1 工作原理

tcpdump 依赖 libpcap 库实现数据包捕获。它通过 AF\_PACKET socket 读取网卡驱动收到的数据包，然后在用户空间进行过滤和格式化输出。

```
网络 -> 网卡驱动 -> AF_PACKET socket -> libpcap -> tcpdump -> 用户输出
```

**需要的权限**：root 权限或 CAP\_NET\_RAW capability

```
# 检查当前用户是否有抓包权限
tcpdump -D

# 如果报错 "you don't have permission to capture on that device"
# 可以使用 sudo
sudo tcpdump -i eth0

# 或者给 tcpdump 添加 capabilities
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
```

### 1.2 常用选项速览

```
-i     指定网卡
-c         抓取指定数量包后退出
-n                不解析域名
-nn               不解析域名和端口
-v, -vv, -vvv     增加详细程度
-X                以十六进制和ASCII显示数据包内容
-XX               显示以太网头部
-e                显示数据链路层头部
-s       抓取前多少字节（0表示完整抓取）
-w          写入文件而不是显示
-r          从文件读取
-C     写入文件时按大小分卷
-G        写入文件时按时间分卷
```

### 1.3 指定网卡

```
# 列出所有可用网卡
tcpdump -D

# 常见输出
# 1.eth0
# 2.any (Pseudo-device that captures on all interfaces)
# 3.lo [Loopback]

# 抓取任意网卡（所有网卡）
tcpdump -i any

# 抓取指定网卡
tcpdump -i eth0

# 抓取本地回环
tcpdump -i lo

# Docker 容器网卡
tcpdump -i docker0

# 指定 VLAN
tcpdump -i eth0.100
```

## 2 基础抓包命令

### 2.1 抓取所有流量

```
# 抓取 eth0 网卡的前 100 个包
tcpdump -i eth0 -c 100

# 抓取所有网卡的包（any）
tcpdump -i any -c 100

# 持续抓包直到 Ctrl+C
tcpdump -i eth0

# 抓包并保存到文件
tcpdump -i eth0 -w /tmp/capture.pcap

# 保存并同时显示（tee 效果）
tcpdump -i eth0 -w /tmp/capture.pcap &
tcpdump -r /tmp/capture.pcap | tail -f
```

### 2.2 基本过滤表达式

**过滤主机**：

```
# 抓取指定 IP 的包
tcpdump -i eth0 host 192.168.1.100

# 抓取源或目标为指定 IP 的包
tcpdump -i eth0 src 192.168.1.100
tcpdump -i eth0 dst 192.168.1.100

# 排除指定 IP
tcpdump -i eth0 not host 192.168.1.100

# 抓取两个主机之间的包
tcpdump -i eth0 host 192.168.1.100 and 192.168.1.200
```

**过滤端口**：

```
# 抓取指定端口
tcpdump -i eth0 port 80

# 抓取源或目标端口
tcpdump -i eth0 src port 80
tcpdump -i eth0 dst port 80

# 排除端口
tcpdump -i eth0 not port 22

# 抓取端口范围
tcpdump -i eth0 portrange 80-443

# 抓取多个端口
tcpdump -i eth0 port 80 or port 443
tcpdump -i eth0 port 80 or 443  # 简写
```

**过滤网络**：

```
# 抓取指定网段的包
tcpdump -i eth0 net 192.168.1.0/24

# 排除网段
tcpdump -i eth0 not net 192.168.1.0/24

# 组合过滤
tcpdump -i eth0 net 192.168.1.0/24 and port 80
```

### 2.3 协议过滤

```
# 抓取 TCP 包
tcpdump -i eth0 tcp

# 抓取 UDP 包
tcpdump -i eth0 udp

# 抓取 ICMP 包
tcpdump -i eth0 icmp

# 抓取 ARP 包
tcpdump -i eth0 arp

# 抓取 IP 包
tcpdump -i eth0 ip

# 抓取 IPv6 包
tcpdump -i eth0 ip6

# 抓取 DNS 包（通常 UDP 53）
tcpdump -i eth0 "udp port 53"

# 抓取 SSH 包
tcpdump -i eth0 "tcp port 22"

# 抓取 HTTP 包
tcpdump -i eth0 "tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)"
```

## 3 输出格式控制

### 3.1 显示控制

```
# 不解析 IP 和端口（显示原始数字）
tcpdump -i eth0 -nn port 80

# 不解析域名（只解析端口）
tcpdump -i eth0 -n port 80

# 详细显示（显示 TTL、ID、长度等）
tcpdump -i eth0 -v port 80

# 更详细
tcpdump -i eth0 -vv port 80

# 最详细（显示 payload 内容）
tcpdump -i eth0 -vvv port 80

# 显示时间戳
tcpdump -i eth0 -tttt port 80
# 输出格式：2025-01-15 10:30:45.123456

# 显示简短时间（不含日期）
tcpdump -i eth0 -tt port 80

# 显示微秒级时间戳
tcpdump -i eth0 -ttttu port 80
```

### 3.2 内容显示

```
# 显示数据链路层头部（MAC 地址）
tcpdump -i eth0 -e port 80

# 以十六进制和 ASCII 显示
tcpdump -i eth0 -X port 80

# 以十六进制和 ASCII 显示（包含以太网头部）
tcpdump -i eth0 -XX port 80

# 只显示包的头部（不显示 payload）
tcpdump -i eth0 -s 68 port 80

# 抓取完整包（snaplen=0）
tcpdump -i eth0 -s 0 port 80

# 显示 ASCII 字符串
tcpdump -i eth0 -A port 80

# 显示十六进制和 ASCII 混合
tcpdump -i eth0 -X port 80
```

### 3.3 保存与读取

```
# 保存到文件
tcpdump -i eth0 -w /tmp/capture.pcap

# 保存时同时显示
tcpdump -i eth0 -w /tmp/capture.pcap &

# 从文件读取
tcpdump -r /tmp/capture.pcap

# 读取时过滤
tcpdump -r /tmp/capture.pcap 'tcp[tcpflags] & tcp-syn != 0'

# 保存到文件并压缩
tcpdump -i eth0 -w - | gzip > /tmp/capture.pcap.gz

# 读取压缩文件
gunzip -c /tmp/capture.pcap.gz | tcpdump -r -

# 文件分卷（每个文件 100MB）
tcpdump -i eth0 -C 100 -w /tmp/capture_%Y%m%d_%H%M%s.pcap

# 按时间分卷（每 5 分钟一个新文件）
tcpdump -i eth0 -G 300 -w /tmp/capture_%Y%m%d_%H%M.pcap

# 从多个文件读取
tcpdump -r /tmp/capture1.pcap -r /tmp/capture2.pcap
```

## 4 TCP 协议专用过滤

### 4.1 TCP 标志位过滤

TCP 头部的第 13 个字节（offset 12）是 Flags 字段：

```
+---+---+---+---+---+---+---+---+
| U | A | P | R | S | F |   |   |
+---+---+---+---+---+---+---+---+
  7   6   5   4   3   2   1   0

F = FIN (1)
S = SYN (2)
R = RST (4)
P = PSH (8)
A = ACK (16)
U = URG (32)
E = ECE (64)
C = CWR (128)
```

```
# 抓取 SYN 包（连接建立）
tcpdump -i eth0 'tcp[13] & 2 != 0'
tcpdump -i eth0 'tcp[tcpflags] == tcp-syn'

# 抓取 SYN-ACK 包
tcpdump -i eth0 'tcp[13] == 18'

# 抓取 ACK 包
tcpdump -i eth0 'tcp[13] & 16 != 0'
tcpdump -i eth0 'tcp[tcpflags] == tcp-ack'

# 抓取 FIN 包（连接关闭）
tcpdump -i eth0 'tcp[13] & 1 != 0'
tcpdump -i eth0 'tcp[tcpflags] == tcp-fin'

# 抓取 RST 包（重置连接）
tcpdump -i eth0 'tcp[13] & 4 != 0'
tcpdump -i eth0 'tcp[tcpflags] == tcp-rst'

# 抓取 PSH-ACK 包（数据传输）
tcpdump -i eth0 'tcp[13] == 24'

# 抓取所有握手和挥手包
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'

# 抓取只有 SYN 没有 ACK 的包（SYN flood）
tcpdump -i eth0 'tcp[13] == 2'
```

### 4.2 TCP 序列号与窗口

```
# 抓取特定序列号的包
tcpdump -i eth0 'tcp[4:4] = 12345'

# 抓取特定确认号的包
tcpdump -i eth0 'tcp[8:4] = 54321'

# 抓取窗口大小大于 0 的包（排除 zero window）
tcpdump -i eth0 'tcp[tcpflags] & tcp-ack != 0 and tcp[14:2] > 0'

# 抓取 zero window 警告包
tcpdump -i eth0 'tcp[14:2] = 0 and tcp[tcpflags] & tcp-ack != 0'
```

### 4.3 TCP 选项过滤

```
# 抓取带 MSS 选项的 SYN 包
tcpdump -i eth0 'tcp[tcpflags] == tcp-syn and tcp[20] == 2 and tcp[21] == 4'

# 抓取带 Window Scale 选项的 SYN 包
tcpdump -i eth0 'tcp[tcpflags] == tcp-syn and tcp[20] == 3 and tcp[21] == 3'

# 抓取带 SACK 选项的包
tcpdump -i eth0 'tcp[tcpflags] & tcp-ack != 0 and tcp[tcpflags] & tcp-syn == 0 and tcp[21] >= 5'

# 抓取带 Timestamp 选项的包
tcpdump -i eth0 'tcp[tcpflags] == tcp-syn and tcp[20] == 8 and tcp[21] == 10'
```

## 5 实战场景

### 5.1 分析 HTTP 请求

```
# 抓取 HTTP 请求
tcpdump -i eth0 -nn -A 'tcp[((tcp[12:1] & 0xf0) >> 2):2] = 0x4745' 2>/dev/null
# 0x4745 是 "GE" 的十六进制（GET 请求）

# 抓取 HTTP 首部
tcpdump -i eth0 -nn -A 'tcp port 80 and tcp[tcpflags] & tcp-push != 0' 2>/dev/null | head -50

# 抓取完整 HTTP 会话
tcpdump -i eth0 -nn -X 'tcp port 80'

# 抓取 HTTP 响应
tcpdump -i eth0 -nn -A 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x48545450' 2>/dev/null
# 0x48545450 是 "HTTP" 的十六进制

# 提取 HTTP Host 首部
tcpdump -i eth0 -nn -A 'tcp port 80' 2>/dev/null | grep -i "host:"
```

### 5.2 分析 DNS 查询

```
# 抓取 DNS 查询和响应
tcpdump -i eth0 -nn -v 'udp port 53'

# 抓取 DNS 查询
tcpdump -i eth0 -nn -A 'udp port 53 and udp[10:2] = 0x0100' 2>/dev/null

# 抓取 DNS 响应
tcpdump -i eth0 -nn -A 'udp port 53 and udp[10:2] = 0x8180' 2>/dev/null

# 抓取特定域名的 DNS 查询
tcpdump -i eth0 -nn -v 'udp port 53 and ip[2:2] > 40' 2>/dev/null | grep "example.com"
```

### 5.3 分析 SSH 会话

```
# 抓取 SSH 连接建立
tcpdump -i eth0 -nn 'tcp port 22 and tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'

# 抓取 SSH 数据传输
tcpdump -i eth0 -nn -X 'tcp port 22 and tcp[tcpflags] & tcp-ack != 0'

# 分析 SSH 延迟
tcpdump -i eth0 -nn -tttt 'tcp port 22' 2>/dev/null | awk '{print $1, $NF}'

# 抓取 SSH keepalive
tcpdump -i eth0 -nn -v 'tcp port 22 and tcp[tcpflags] & tcp-ack != 0 and tcp[tcpflags] & tcp-psh != 0 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x01010500' 2>/dev/null
```

### 5.4 分析 MySQL 连接

```
# 抓取 MySQL 握手包
tcpdump -i eth0 -nn -X 'tcp port 3306 and tcp[tcpflags] & tcp-syn != 0'

# 抓取 MySQL 查询
tcpdump -i eth0 -nn -A 'tcp port 3306 and tcp[tcpflags] & tcp-push != 0' 2>/dev/null | grep -E "(SELECT|INSERT|UPDATE|DELETE|COMMIT)"

# 抓取 MySQL 响应
tcpdump -i eth0 -nn -A 'tcp port 3306 and tcp[tcpflags] & tcp-ack != 0' 2>/dev/null | head -100

# 完整 MySQL 会话抓包
tcpdump -i eth0 -nn -w /tmp/mysql.pcap 'tcp port 3306'
```

### 5.5 分析 Redis 连接

```
# 抓取 Redis 命令
tcpdump -i eth0 -nn -X 'tcp port 6379 and tcp[tcpflags] & tcp-push != 0' 2>/dev/null

# 提取 Redis 命令
tcpdump -i eth0 -nn -A 'tcp port 6379' 2>/dev/null | strings | grep -E "^(GET|SET|DEL|HSET|HGET|EXPIRE|LRANGE)"

# 分析 Redis 响应时间（需要结合时间戳）
tcpdump -i eth0 -nn -tttt 'tcp port 6379' 2>/dev/null
```

## 6 性能与资源控制

### 6.1 限制抓包大小

```
# 设置 snaplen（抓取每个包的前多少字节）
tcpdump -i eth0 -s 68   # 只抓头部
tcpdump -i eth0 -s 100  # 抓头部+少量数据
tcpdump -i eth0 -s 0    # 抓完整包

# snaplen=0 等价于 262144 字节

# 限制显示的行数
tcpdump -i eth0 -c 100  # 只抓100个包

# 限制抓包时间
timeout 60 tcpdump -i eth0 -w /tmp/capture.pcap
```

### 6.2 文件大小控制

```
# 每个文件最大 100MB
tcpdump -i eth0 -C 100 -w /tmp/capture.pcap

# 文件数量上限（达到后覆盖）
tcpdump -i eth0 -C 100 -W 10 -W /tmp/capture.pcap

# 按时间分卷（每 5 分钟一个文件）
tcpdump -i eth0 -G 300 -w /tmp/capture_%Y%m%d_%H%M.pcap

# 按大小和时间双重分卷
tcpdump -i eth0 -G 300 -C 50 -w /tmp/capture.pcap
```

### 6.3 缓冲区设置

```
# 设置读取缓冲区大小（MB）
tcpdump -i eth0 -B 4096

# 设置 snapshots 缓冲区
tcpdump -i eth0 -s 0 -w /tmp/capture.pcap &

# 调整系统缓冲区
echo 16777216 > /proc/sys/net/core/rmem_max
echo 16777216 > /proc/sys/net/core/rmem_default
```

## 7 高级技巧

### 7.1 表达式组合

```
# 逻辑运算符
tcpdump -i eth0 'host 192.168.1.100 and port 80'
tcpdump -i eth0 'host 192.168.1.100 or host 192.168.1.200'
tcpdump -i eth0 'not host 192.168.1.100'
tcpdump -i eth0 'host 192.168.1.100 and not port 22'

# 复杂组合
tcpdump -i eth0 '(host 192.168.1.100 or host 192.168.1.200) and (port 80 or port 443)'

# 抓取非本地网络的 HTTP 请求
tcpdump -i eth0 'not net 192.168.0.0/24 and not net 10.0.0.0/8 and port 80'
```

### 7.2 BPF 高级过滤

```
# 抓取 IP 碎片
tcpdump -i eth0 'ip[6:2] & 0x1fff != 0'

# 抓取 TTL 小于 10 的包
tcpdump -i eth0 'ip[8] < 10'

# 抓取特定 DSCP 值
tcpdump -i eth0 'ip[1] & 0xfc == 0x80'# DSCP EF

# 抓取特定 IP 协议
tcpdump -i eth0 'ip[9] = 6'    # TCP
tcpdump -i eth0 'ip[9] = 17'   # UDP
tcpdump -i eth0 'ip[9] = 1'    # ICMP

# 抓取特定 IP 长度
tcpdump -i eth0 'ip[2:2] > 600'

# 抓取 TCP payload 大于 0 的包
tcpdump -i eth0 'tcp[tcpflags] & tcp-ack != 0 and (ip[2:2] - ((ip[0]&0xf)<<2) - ((tcp[12]&0xf0)>>2)) > 0'
```

### 7.3 结合其他工具

```
# 实时显示 HTTP 请求数统计
tcpdump -i eth0 -nn 'tcp port 80' 2>/dev/null | awk '{print $5}' | sort | uniq -c | sort -rn

# 实时显示 TOP 10 请求 IP
tcpdump -i eth0 -nn 'tcp port 80' 2>/dev/null | awk '{print $3}' | cut -d. -f1-4 | sort | uniq -c | sort -rn | head -10

# 统计 TCP 状态
watch -n 1 "ss -tan | awk '{print \$1}' | sort | uniq -c | sort -rn"

# 结合 strings 提取敏感信息（慎用）
tcpdump -i eth0 -nn -A 'tcp port 80' 2>/dev/null | strings | grep -iE "(password|passwd|pwd|token|secret)"
```

### 7.4 与 Wireshark 配合

```
# tcpdump 抓包，Wireshark 分析
tcpdump -i eth0 -nn -s 0 -w /tmp/capture.pcap 'tcp port 8080'

# 实时流式分析
tcpdump -i eth0 -nn -s 0 -w - 'tcp port 8080' | wireshark -k -i -

# 远程抓包，本地分析
# 远程执行
ssh root@server "tcpdump -i eth0 -nn -s 0 -w - 'tcp port 8080'" > /tmp/capture.pcap

# 导出特定连接
tcpdump -r /tmp/capture.pcap 'tcpdump -w output.pcap "host 192.168.1.100"'
```

## 8 常见故障排查

### 8.1 网络延迟分析

```
# 抓取时间戳
tcpdump -i eth0 -nn -tttt port 8080 2>/dev/null

# 分析请求-响应时间
# 1. 找到客户端请求的 SYN
# 2. 找到服务端响应的 SYN-ACK
# 3. 计算时间差

# 抓取每个包的往返时间
tcpdump -i eth0 -nn -tttt 'tcp port 80' 2>/dev/null | awk '
/SYN/ {syn=$2; syn_ip=$5}
/SYN.*ACK/ {print "SYN->SYN-ACK:", syn, $2, $5}
/ACK.*8080/ {print "SYN-ACK->ACK:", $2, $5}
'
```

### 8.2 连接重置排查

```
# 抓取 RST 包
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0'

# 抓取 RST 并显示上下文
tcpdump -i eth0 -nn -c 1000 'tcp[tcpflags] & (tcp-rst|tcp-ack) != 0' 2>/dev/null

# 分析 RST 来源
# 1. 客户端 RST：客户端认为连接有问题
# 2. 服务端 RST：服务端拒绝连接或应用崩溃

# 抓取连接后的 RST
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0 and tcp[tcpflags] & tcp-ack != 0'
```

### 8.3 丢包分析

```
# 抓取 TCP 重传
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-syn != 0' 2>/dev/null | wc -l
# 如果 SYN 重传率高，说明网络丢包

# 抓取重复 ACK
tcpdump -i eth0 -nn 'tcp[tcpflags] == tcp-ack and tcp[26:4] = 0x01000000' 2>/dev/null
# 0x01000000 是重复 ACK 的特征

# 抓取 SACK 块
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-ack != 0 and tcp[tcpflags] & tcp-syn == 0' 2>/dev/null | grep "SACK"
```

### 8.4 SYN Flood 排查

```
# 抓取所有 SYN
tcpdump -i eth0 -nn 'tcp[tcpflags] == tcp-syn'

# 统计每秒 SYN 数量
tcpdump -i eth0 -nn 'tcp[tcpflags] == tcp-syn' 2>/dev/null | awk '{print $1}' | cut -d. -f1 | uniq -c

# 查看半连接队列
ss -ltn state syn-recv

# 查看 SYN cookie 是否启用
cat /proc/sys/net/ipv4/tcp_syncookies

# 如果 SYN 队列满，可能需要
sysctl -w net.ipv4.tcp_max_syn_backlog=4096
sysctl -w net.core.somaxconn=4096
```

### 8.5 MTU 问题排查

```
# 抓取 IP 分片
tcpdump -i eth0 -nn 'ip[6:2] & 0x4000 != 0'

# 抓取分片包
tcpdump -i eth0 -nn 'ip[6:2] & 0x1fff != 0'

# 查看是否有 ICMP fragmentation needed
tcpdump -i eth0 -nn 'icmp[icmptype] == 3 and icmp[icmpcode] == 4'
# type=3, code=4 是 "fragmentation needed but DF set"

# 抓取 ICMP 包
tcpdump -i eth0 -nn icmp
```

## 9 实用脚本

### 9.1 实时网络监控脚本

```
#!/bin/bash
# filename: net_monitor.sh
# 实时监控网络流量，发现异常告警

INTERFACE=${1:-eth0}
THRESHOLD=${2:-100}

echo"=== 网络监控开始 ==="
echo"监控网卡: $INTERFACE"
echo"告警阈值: $THRESHOLD SYN/秒"
echo""

# 清理旧文件
rm -f /tmp/tcpdump_*.pcap

# 捕获 SIGINT
trap'echo "正在停止..."; pkill -P $$; exit 0' INT

# 持续监控
whiletrue; do
    # 抓包 1 秒统计
    SYN_COUNT=$(timeout 1 tcpdump -i $INTERFACE -c 1000 'tcp[tcpflags] == tcp-syn' 2>/dev/null | wc -l)

    echo"$(date '+%Y-%m-%d %H:%M:%S') SYN数: $SYN_COUNT"

    if [ $SYN_COUNT -gt $THRESHOLD ]; then
        echo"警告: SYN 数量超过阈值，可能存在 SYN Flood 攻击"
        echo"开始抓包保存..."
        timeout 60 tcpdump -i $INTERFACE -nn -w /tmp/tcpdump_$(date +%Y%m%d_%H%M%S).pcap 'tcp[tcpflags] == tcp-syn' &
    fi

    sleep 5
done
```

### 9.2 HTTP 请求分析脚本

```
#!/bin/bash
# filename: http_analysis.sh
# 分析 HTTP 请求和响应

PORT=${1:-80}
OUTPUT="/tmp/http_analysis_$(date +%Y%m%d_%H%M%S).txt"

echo"=== HTTP 流量分析 ===" | tee $OUTPUT
echo"监控端口: $PORT" | tee -a $OUTPUT
echo"开始时间: $(date)" | tee -a $OUTPUT
echo"" | tee -a $OUTPUT

# 抓取 HTTP 请求
echo"=== 最近 20 个 HTTP 请求 ===" | tee -a $OUTPUT
timeout 30 tcpdump -i eth0 -nn -A "tcp port $PORT and tcp[tcpflags] & tcp-push != 0" 2>/dev/null | \
    grep -E "^(GET|POST|PUT|DELETE|HEAD|OPTIONS|PATCH)" | \
    head -20 | tee -a $OUTPUT

echo"" | tee -a $OUTPUT

# 统计请求类型
echo"=== 请求类型统计 ===" | tee -a $OUTPUT
timeout 30 tcpdump -i eth0 -nn -c 1000 "tcp port $PORT" 2>/dev/null | \
    awk '/GET/ {get++; next} /POST/ {post++; next} /PUT/ {put++; next} /DELETE/ {del++; next} END {print "GET:", get, "POST:", post, "PUT:", put, "DELETE:", del}' | \
    tee -a $OUTPUT

echo"" | tee -a $OUTPUT

# 统计目标 IP
echo"=== 目标 IP TOP 10 ===" | tee -a $OUTPUT
timeout 30 tcpdump -i eth0 -nn -c 1000 "tcp port $PORT" 2>/dev/null | \
    awk '{print $5}' | cut -d. -f1-4 | sort | uniq -c | sort -rn | head -10 | tee -a $OUTPUT

echo"" | tee -a $OUTPUT
echo"分析完成，保存到: $OUTPUT"
```

### 9.3 TCP 连接状态分析脚本

```
#!/bin/bash
# filename: tcp_state_analysis.sh
# 分析 TCP 连接状态

echo"=== TCP 连接状态分析 ==="
echo"分析时间: $(date)"
echo""

# 当前连接状态统计
echo"=== 当前各状态连接数 ==="
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

echo""

# TIME_WAIT 详情
echo"=== TIME_WAIT 详情 TOP 20 ==="
ss -tan state time-wait | awk 'NR>1 {print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -20

echo""

# CLOSE_WAIT 详情
echo"=== CLOSE_WAIT 详情 ==="
CLOSE_WAIT=$(ss -tan state close-wait | awk 'NR>1 {print $4}' | wc -l)
echo"CLOSE_WAIT 连接数: $CLOSE_WAIT"

if [ $CLOSE_WAIT -gt 100 ]; then
    echo"警告: CLOSE_WAIT 连接数过高，请检查应用"
    ss -tan state close-wait | awk 'NR>1 {print $4, $5}' | head -20
fi

echo""

# SYN_RECVD 详情（可能存在攻击）
echo"=== SYN_RECVD 详情 ==="
SYN_RECV=$(ss -tan state syn-recv | awk 'NR>1 {print $4}' | wc -l)
echo"SYN_RECVD 连接数: $SYN_RECV"

if [ $SYN_RECV -gt 1000 ]; then
    echo"警告: SYN_RECVD 连接数过高，可能存在 SYN Flood 攻击"
fi

echo""

# Established 连接 TOP
echo"=== Established 连接 TOP 20 ==="
ss -tan state established | awk 'NR>1 {print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -20
```

### 9.4 丢包检测脚本

```
#!/bin/bash
# filename: packet_loss_detect.sh
# 检测网络丢包情况

INTERFACE=${1:-eth0}
TARGET=${2:-8.8.8.8}

echo"=== 网络丢包检测 ==="
echo"目标: $TARGET"
echo"检测时间: $(date)"
echo""

# ping 检测丢包
echo"=== Ping 检测 ==="
ping -c 100 -i 0.2 $TARGET | tail -5

echo""

# tcpdump 检测 TCP 重传
echo"=== TCP 重传检测 ==="
echo"抓取 10 秒 TCP 流量..."

# 统计 TCP 重传（需要抓包后分析）
timeout 10 tcpdump -i $INTERFACE -c 10000 'tcp[tcpflags] & tcp-ack != 0' 2>/dev/null > /tmp/tcp_check.txt

SYN_COUNT=$(grep -c "SYN" /tmp/tcp_check.txt)
FIN_COUNT=$(grep -c "FIN" /tmp/tcp_check.txt)
RST_COUNT=$(grep -c "RST" /tmp/tcp_check.txt)
TOTAL=$(wc -l < /tmp/tcp_check.txt)

echo"总包数: $TOTAL"
echo"SYN 包: $SYN_COUNT"
echo"FIN 包: $FIN_COUNT"
echo"RST 包: $RST_COUNT"

if [ $RST_COUNT -gt 10 ]; then
    echo"警告: RST 包过多，可能存在连接问题"
fi

echo""

# netstat 丢包统计
echo"=== 系统丢包统计 ==="
netstat -s | grep -iE "(segments retransmited|segment lost|lists overflow)" | head -10

rm -f /tmp/tcp_check.txt
```

## 10 总结

### 10.1 常用命令速查表

| 场景 | 命令 |
| --- | --- |
| 抓取所有包 | `tcpdump -i eth0` |
| 抓取指定主机 | `tcpdump -i eth0 host 192.168.1.100` |
| 抓取指定端口 | `tcpdump -i eth0 port 80` |
| 保存到文件 | `tcpdump -i eth0 -w file.pcap` |
| 读取文件 | `tcpdump -r file.pcap` |
| 不解析域名 | `tcpdump -i eth0 -nn` |
| 显示包内容 | `tcpdump -i eth0 -X` |
| 抓取 SYN 包 | `tcpdump -i eth0 'tcp[tcpflags] == tcp-syn'` |
| 抓取 RST 包 | `tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0'` |
| 抓取 HTTP | `tcpdump -i eth0 -A 'tcp port 80'` |
| 抓取 ICMP | `tcpdump -i eth0 icmp` |
| 按大小分卷 | `tcpdump -i eth0 -C 100 -w file.pcap` |

### 10.2 最佳实践

1.  **生产环境抓包前先评估影响**：大流量抓包可能产生额外 CPU 负载
    
2.  **使用 snaplen 限制**：不需要完整包时，使用 `-s 68` 或 `-s 100` 减少资源占用
    
3.  **结合过滤器**：先过滤再抓包，减少无效数据
    
4.  **及时保存分析结果**：抓包文件可能很大，分析后及时清理
    
5.  **注意敏感数据**：抓包会捕获明文密码等敏感信息，确保抓包文件安全
    
6.  **使用 -nn 加速**：生产环境避免 DNS 反向解析
    
7.  **使用 any 时注意负载**：多网卡环境使用 any 会增加负载
    

### 10.3 常见错误

```
# 错误：直接 tcpdump 不指定网卡
tcpdump  # 可能抓不到任何东西

# 正确：指定网卡
tcpdump -i eth0

# 错误：抓包时未设置 snaplen 导致文件过大
tcpdump -i eth0 -w huge.pcap

# 正确：根据需要设置 snaplen
tcpdump -i eth0 -s 100 -w large.pcap

# 错误：表达式缺少引号
tcpdump -i eth0 host 192.168.1.100 and port 80  # shell 解析可能出错

# 正确：表达式加引号
tcpdump -i eth0 'host 192.168.1.100 and port 80'

# 错误：忘记 -nn 导致解析缓慢
tcpdump -i eth0 port 80  # 缓慢

# 正确：加 -nn 加速
tcpdump -i eth0 -nn port 80
```

tcpdump 是网络排查的瑞士军刀，熟练掌握各种过滤组合能够在故障排查时事半功倍。建议在测试环境多练习各种过滤条件，理解 BPF 语法的原理，才能在生产环境中快速定位问题。

## 更多精彩

关注公众号**「****浩道Linux****」**

**浩道Linux****，专注于****Linux系统****的相关知识、****网络通信****、****网络安全****、****Python相关****知识以及涵盖IT行业相关技能的学习，**理论与实战结合，真正让你在学习工作中真正去用到所学。同时也会分享一些面试经验，助你找到高薪offer，让我们一起去学习，一起去进步，一起去涨薪！期待您的加入～～～**关注回复“资料”可****免费获取学习资料****（含有电子书籍、视频等）。**

**点赞 分享****服务常在🌷 点个**在看 **永不宕机🍀****![[Inbox/笔记同步助手/微信公众号/20260604/images/374dcfa56729b6cece6602f2bb3b3529_MD5.gif||21]]**

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/61e4da37_1780533067079?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIxMTgxNzQ2OA%3D%3D%26mid%3D2247504180%26idx%3D1%26sn%3Dec19c6f7a0ef89b99db059061309ae9a%26chksm%3D96151e4d2b835a6c24e4435b76c669370082eab7ac36a3e74f38e52ae729b03ad2c4f94c0d3c%26mpshare%3D1%26scene%3D1%26srcid%3D0604il8AssunTebzXrdERaLE%26sharer_shareinfo%3De3c0f87f76025d56e5406f3b6d9d7410%26sharer_shareinfo_first%3De3c0f87f76025d56e5406f3b6d9d7410%23rd&s=obsidian)