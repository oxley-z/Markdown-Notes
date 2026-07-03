# wireshark使用

## wireshark过滤

### 过滤值符号

| 英文          | 符号  | 描述                        |
| ----------- | --- | ------------------------- |
| eq          | ==  | 等于 ip.src == 192.168.3.1  |
| ne          | !=  | 不等于 ip.src != 192.168.3.1 |
| gt          | >   | 大于 frame.len > 10         |
| lt          | <   | 小于 frame.len < 10         |
| ge          | >=  | 大于等于 frame.len ge 0x100   |
| le          | <=  | 小于等于 frame.len le 0x100   |
| contains    |     | 包含                        |
| matches     | ~   | 正则匹配                      |
| bitwise_and | &   | 位与操作                      |

### 多个表达式组合

| 英文 | 符号 | 描述                                  |
| ---- | ---- | ------------------------------------- |
| and  | &&   | 逻辑与 ip.src == 192.168.3.1 and xxxx |
| or   | \|\| | 逻辑或                                |
| xor  | ^^   | 逻辑异或                              |
| not  | !    | 逻辑非                                |

### IP过滤

#### 对源地址和目的地址过滤

```bash
 过滤源地址
 ip.src_host eq 192.168.31.106          #相当于
 ip.src_host == 192.168.31.106

 过滤目的地址
 ip.dst_host eq 192.168.31.106          #相当于
 ip.dst_host == 192.168.31.106

 过滤源地址或者目的地址
 ip.addr eq 192.168.31.106              #相当于
 ip.addr == 192.168.31.106
```

如果想排除以上的数据包，只需要将其用括号囊括，然后使用 "!" 即可

```bash
!(ip.addr == 192.168.0.1)
```



> 注意:在Filter编辑框中，收入过虑规则时，如果语法有误，框会显红色，如正确，会是绿色。

#### 针对端口进行过滤

```bash
例子:
tcp.port eq 80                    #不管端口是来源的还是目标的都显示，相当于
tcp.port == 80

tcp.port eq 80 or udp.port eq 80

tcp.srcport == 80                 #只显tcp协议的源端口80
tcp.dstport == 80                 #只显tcp协议的目标端口80

udp.port eq 15000
tcp.port >= 1 and tcp.port <= 80  #过滤端口范围
```

#### 过滤MAC地址

```bash
eth.addr eq B4-6D-83-2B-27-D5 //过滤来源MAC和目标MAC都等于A0:00:00:04:C5:84的

eth.src eq B4-6D-83-2B-27-D5 // 过滤来源mac
eth.dst == B4-6D-83-2B-27-D5 // 过滤目标mac
```

#### 针对http的请求类型进行过滤

```bash
http.request.method == "GET"
http.request.method == "POST"
```

#### 过滤协议

```bash
tcp
udp
arp
icmp
http
smtp
ftp
dns
msnms
ip
ssl
oicq
bootp
```

#### 包长度过滤

```bash
udp.length == 26 这个长度是指udp本身固定长度8加上udp下面那块数据包之和
tcp.len >= 7   指的是ip数据包(tcp下面那块数据),不包括tcp本身
ip.len == 94 除了以太网头固定长度14,其它都算是ip.len,即从ip本身到最后
frame.len == 119 整个数据包长度,从eth开始到最后
```

#### TCP参数过滤

```bash
tcp.flags 显示包含TCP标志的封包。
tcp.flags.syn == 0x02     显示包含TCP SYN标志的封包。
tcp.window_size == 0 && tcp.flags.reset != 1
```

