# netperf测试

## 目录

1. [批量网络流量性能测试](#net)
   1. [TCP_STREAM测试](#TCP_STREAM)
   2. [UDP_STREAM](#UDP_STREAM)

2. [测试请求/应答网络流量测试](#net2)
   1. [TCP_RR](TCP_RR)
   
   2. [TCP_CRR](TCP_CRR)
   
---

## <a id=net>批量网络流量性能测试</a>

根据使用传输协议的不同，批量数据传输又分为TCP批量传输和UDP批量传输。

> 服务端

服务端启动服务执行

```bash
eddy@eddy:~$ netserver
Unable to start netserver with  'IN(6)ADDR_ANY' port '12865' and family AF_UNSPEC
```

默认情况下 netserver 开启端口号为 12865，可以通过以下命令指定服务端口（如果服务端指定端口，则客户端也需要指定服务器端口）

```bash
eddy@eddy:~$ sudo netserver -D -p 4659
```

查看netperf帮助

```bash
eddy@eddy:~$ netperf -h

Usage: netperf [global options] -- [test options]

Global options:
    -a send,recv      Set the local send,recv buffer alignment
    -A send,recv      Set the remote send,recv buffer alignment
    -B brandstr       Specify a string to be emitted with brief output
    -c [cpu_rate]     Report local CPU usage
    -C [cpu_rate]     Report remote CPU usage
    -d                Increase debugging output
    -D time,[units] * Display interim results at least every time interval
                      using units as the initial guess for units per second
                      A negative value for time will make heavy use of the
                      system's timestamping functionality
    -f G|M|K|g|m|k    Set the output units
    -F lfill[,rfill]* Pre-fill buffers with data from specified file
    -h                Display this text
    -H name|ip,fam *  Specify the target machine and/or local ip and family
    -i max,min        Specify the max and min number of iterations (15,1)
    -I lvl[,intvl]    Specify confidence level (95 or 99) (99)
                      and confidence interval in percentage (10)
    -j                Keep additional timing statistics
    -l testlen        Specify test duration (>0 secs) (<0 bytes|trans)
    -L name|ip,fam *  Specify the local ip|name and address family
    -o send,recv      Set the local send,recv buffer offsets
    -O send,recv      Set the remote send,recv buffer offset
    -n numcpu         Set the number of processors for CPU util
    -N                Establish no control connection, do 'send' side only
    -p port,lport*    Specify netserver port number and/or local port
    -P 0|1            Don't/Do display test headers
    -r                Allow confidence to be hit on result only
    -s seconds        Wait seconds between test setup and test start
    -S                Set SO_KEEPALIVE on the data connection
    -t testname       Specify test to perform
    -T lcpu,rcpu      Request netperf/netserver be bound to local/remote cpu
    -v verbosity      Specify the verbosity level
    -W send,recv      Set the number of send,recv buffers
    -v level          Set the verbosity level (default 1, min 0)
    -V                Display the netperf version and exit
    -y local,remote   Set the socket priority
    -Y local,remote   Set the IP_TOS. Use hexadecimal.
    -Z passphrase     Set and pass to netserver a passphrase

For those options taking two parms, at least one must be specified;
specifying one value without a comma will set both parms to that
value, specifying a value with a leading comma will set just the second
parm, a value with a trailing comma will set just the first. To set
each parm to unique values, specify both and separate them with a
comma.

* For these options taking two parms, specifying one value with no comma
will only set the first parms and will leave the second at the default
value. To set the second value it must be preceded with a comma or be a
comma-separated pair. This is to retain previous netperf behaviour.
```

查看netserver帮助

```bash
eddy@eddy:~$ netserver -h

Usage: netserver [options]

Options:
    -h                Display this text
    -D                Do not daemonize
    -d                Increase debugging output
    -f                Do not spawn chilren for each test, run serially
    -L name,family    Use name to pick listen address and family for family
    -N                No debugging output, even if netperf asks
    -p portnum        Listen for connect requests on portnum.
    -4                Do IPv4
    -6                Do IPv6
    -v verbosity      Specify the verbosity level
    -V                Display version information and exit
    -Z passphrase     Expect passphrase as the first thing received

eddy@eddy:~$
```



### <a id=TCP_STREAM>TCP_STREAM测试</a>

Netperf缺省情况下进行TCP批量传输，即-t TCP_STREAM，用来测试进行TCP批量传输时的网络性能。

测试过程中，netperf向netserver发送批量的TCP数据分组，以确定数据传输过程中的吞吐量。

服务端

```bash
 eddy@eddy:~$ sudo netserver -D -p 4659
```

客户端

```bash
eddy@eddy:~$ sudo netperf -t TCP_STREAM -H 192.168.159.131 -l 5 -p 4659
MIGRATED TCP STREAM TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.159.131 () port 0 AF_INET : demo
Recv   Send    Send
Socket Socket  Message  Elapsed
Size   Size    Size     Time     Throughput
bytes  bytes   bytes    secs.    10^6bits/sec

131072  16384  16384    5.02      867.98
eddy@eddy:~$
```



### <a id=UDP_STREAM>UDP_STREAM测试</a>

UDP_STREAM用来测试进行UDP批量传输时的网络性能。

测试UDP的网络性能时，测试分组的大小不得大于socket的发送与接收缓冲大小，否则netperf会报出错提示。

服务端

```bash
eddy@eddy:~$ sudo netserver -D -p 4659
```

客户端

```bash
eddy@eddy:~$ sudo netperf -t UDP_STREAM -H 192.168.159.131 -l 5 -p 4659
MIGRATED UDP STREAM TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.159.131 () port 0 AF_INET : demo
Socket  Message  Elapsed      Messages
Size    Size     Time         Okay Errors   Throughput
bytes   bytes    secs            #      #   10^6bits/sec

212992   65507   5.00         3275      0     343.22
212992           5.00         3229            338.40

eddy@eddy:~$
```





## <a id=net2>测试请求/应答网络流量测试</a>

### <a id=TCP_RR>TCP_RR</a>

TCP_RR 方式的测试对象是多次 TCP request 和 response 的交易过程，但是它们发生在同一个 TCP 连接中，这种模式常常出现在数据库应用中。数据库的 client 程序与 server 程序建立一个 TCP 连接以后，就在这个连接中传送数据库的多次交易过程。

5秒内TCP_RR数据

```bash
eddy@eddy:~$ sudo netperf -t TCP_RR -H 192.168.159.131 -l 5
MIGRATED TCP REQUEST/RESPONSE TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.159.131 () port 0 AF_INET : demo : first burst 0
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size     Size    Time     Rate
bytes  Bytes  bytes    bytes   secs.    per sec

16384  131072 1        1       5.00     1669.24
16384  131072
eddy@eddy:~$
```

可使用`-- -r <Request size>,<Resp size>`更改请求和相应大小



### <a id=TCP_CRR>TCP_CRR</a>

5秒内TCP_CRR数据

```bash
eddy@eddy:~$ sudo netperf -t TCP_CRR -H 192.168.159.131 -l 5
MIGRATED TCP Connect/Request/Response TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.159.131 () port 0 AF_INET : demo
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size     Size    Time     Rate
bytes  Bytes  bytes    bytes   secs.    per sec

16384  131072 1        1       5.00      458.35
16384  131072
eddy@eddy:~$
```

可使用`-- -r <Request size>,<Resp size>`更改请求和相应大小

### <a id=UDP_RR>UDP_RR</a>

```bash
eddy@eddy:~$ sudo netperf -t UDP_RR -H 192.168.159.131 -l 5
MIGRATED UDP REQUEST/RESPONSE TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.159.131 () port 0 AF_INET : demo : first burst 0
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size     Size    Time     Rate
bytes  Bytes  bytes    bytes   secs.    per sec

212992 212992 1        1       5.00     1684.71
212992 212992
eddy@eddy:~$
```



可使用`-- -r <Request size>,<Resp size>`更改请求和相应大小

```bash
eddy@eddy:~$ sudo netperf -t UDP_RR -H 192.168.159.131 -l 5 -- -r 128,8192
MIGRATED UDP REQUEST/RESPONSE TEST from 0.0.0.0 (0.0.0.0) port 0 AF_INET to 192.168.159.131 () port 0 AF_INET : demo : first burst 0
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size     Size    Time     Rate
bytes  Bytes  bytes    bytes   secs.    per sec

212992 212992 128      8192    5.00     1690.62
212992 212992
eddy@eddy:~$
```



# 参考

[第 4 章 Netperf 网络测试 (brinnatt.com)](https://www.brinnatt.com/projects/第-4-章-netperf-网络测试/#41、Netperf_安装)

[【山外笔记-工具框架】Netperf网络性能测试工具详解教程-云社区-华为云 (huaweicloud.com)](https://bbs.huaweicloud.com/blogs/228744)