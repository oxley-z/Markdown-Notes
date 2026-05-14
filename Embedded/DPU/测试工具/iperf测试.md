# Iperf网络测试工具

## iperf

Iperf是一款基于TCP/IP和UDP/IP的网络性能测试工具，可以用来测量网络带宽和网络质量，提供网络延迟抖动、数据包丢失率、最大传输单元等统计信息。

安装

```bash
sudo apt install iperf
```

iperf源码仓库

[measurement tool (github.com)](https://github.com/esnet/iperf)

iperf官网

[iPerf - The TCP, UDP and SCTP network bandwidth measurement tool](https://iperf.fr/)

帮助手册

```bash
eddy@eddy:~$ iperf -h
Usage: iperf [-s|-c host] [options]
       iperf [-h|--help] [-v|--version]

Client/Server:
  -b, --bandwidth #[kmgKMG | pps]  bandwidth to send at in bits/sec or packets per second
  -e, --enhancedreports    use enhanced reporting giving more tcp/udp and traffic information
  -f, --format    [kmgKMG]   format to report: Kbits, Mbits, KBytes, MBytes
  -i, --interval  #        seconds between periodic bandwidth reports
  -l, --len       #[kmKM]    length of buffer in bytes to read or write (Defaults: TCP=128K, v4 UDP=1470, v6 UDP=1450)
  -m, --print_mss          print TCP maximum segment size (MTU - TCP/IP header)
  -o, --output    <filename> output the report or error message to this specified file
  -p, --port      #        server port to listen on/connect to
  -u, --udp                use UDP rather than TCP
      --udp-counters-64bit use 64 bit sequence numbers with UDP
  -w, --window    #[KM]    TCP window size (socket buffer size)
  -z, --realtime           request realtime scheduler
  -B, --bind <host>[:<port>][%<dev>] bind to <host>, ip addr (including multicast address) and optional port and device
  -C, --compatibility      for use with older versions does not sent extra msgs
  -M, --mss       #        set TCP maximum segment size (MTU - 40 bytes)
  -N, --nodelay            set TCP no delay, disabling Nagle's Algorithm
  -S, --tos       #        set the socket's IP_TOS (byte) field

Server specific:
  -s, --server             run in server mode
  -t, --time      #        time in seconds to listen for new connections as well as to receive traffic (default not set)
      --udp-histogram #,#  enable UDP latency histogram(s) with bin width and count, e.g. 1,1000=1(ms),1000(bins)
  -B, --bind <ip>[%<dev>]  bind to multicast address and optional device
  -H, --ssm-host <ip>      set the SSM source, use with -B for (S,G)
  -U, --single_udp         run in single threaded UDP mode
  -D, --daemon             run the server as a daemon
  -V, --ipv6_domain        Enable IPv6 reception by setting the domain and socket to AF_INET6 (Can receive on both IPv4 and IPv6)

Client specific:
  -c, --client    <host>   run in client mode, connecting to <host>
  -d, --dualtest           Do a bidirectional test simultaneously
      --ipg                set the the interpacket gap (milliseconds) for packets within an isochronous frame
      --isochronous <frames-per-second>:<mean>,<stddev> send traffic in bursts (frames - emulate video traffic)
  -n, --num       #[kmgKMG]    number of bytes to transmit (instead of -t)
  -r, --tradeoff           Do a bidirectional test individually
  -t, --time      #        time in seconds to transmit for (default 10 secs)
  -B, --bind [<ip> | <ip:port>] bind ip (and optional port) from which to source traffic
  -F, --fileinput <name>   input the data to be transmitted from a file
  -I, --stdin              input the data to be transmitted from stdin
  -L, --listenport #       port to receive bidirectional tests back on
  -P, --parallel  #        number of parallel client threads to run
  -R, --reverse            reverse the test (client receives, server sends)
  -T, --ttl       #        time-to-live, for multicast (default 1)
  -V, --ipv6_domain        Set the domain to IPv6 (send packets over IPv6)
  -X, --peer-detect        perform server version detection and version exchange
  -Z, --linux-congestion <algo>  set TCP congestion control algorithm (Linux only)

Miscellaneous:
  -x, --reportexclude [CDMSV]   exclude C(connection) D(data) M(multicast) S(settings) V(server) reports
  -y, --reportstyle C      report as a Comma-Separated Values
  -h, --help               print this message and quit
  -v, --version            print version information and quit

[kmgKMG] Indicates options that support a k,m,g,K,M or G suffix
Lowercase format characters are 10^3 based and uppercase are 2^n based
(e.g. 1k = 1000, 1K = 1024, 1m = 1,000,000 and 1M = 1,048,576)

The TCP window size option can be set by the environment variable
TCP_WINDOW_SIZE. Most other options can be set by an environment variable
IPERF_<long option name>, such as IPERF_BANDWIDTH.

Source at <http://sourceforge.net/projects/iperf2/>
Report bugs to <iperf-users@lists.sourceforge.net>
eddy@eddy:~$
```

简单测试

server

```bash
iperf -s -i 1 -p 5001
------------------------------------------------------------
Server listening on TCP port 5001
TCP window size:  128 KByte (default)
------------------------------------------------------------
[  4] local 192.168.159.131 port 5001 connected with 192.168.159.132 port 52464
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0- 1.0 sec   135 MBytes  1.13 Gbits/sec
[  4]  1.0- 2.0 sec   136 MBytes  1.14 Gbits/sec
[  4]  2.0- 3.0 sec  78.8 MBytes   661 Mbits/sec
[  4]  3.0- 4.0 sec  79.4 MBytes   666 Mbits/sec
[  4]  4.0- 5.0 sec   111 MBytes   932 Mbits/sec
[  4]  5.0- 6.0 sec   105 MBytes   877 Mbits/sec
[  4]  6.0- 7.0 sec   121 MBytes  1.02 Gbits/sec
[  4]  7.0- 8.0 sec   122 MBytes  1.03 Gbits/sec
[  4]  8.0- 9.0 sec   127 MBytes  1.06 Gbits/sec
[  4]  9.0-10.0 sec   106 MBytes   891 Mbits/sec
[  4]  0.0-10.0 sec  1.10 GBytes   941 Mbits/sec
```

client

```bash
eddy@eddy:~$ iperf -c 192.168.159.131 -i 1 -p 5001
------------------------------------------------------------
Client connecting to 192.168.159.131, TCP port 5001
TCP window size:  629 KByte (default)
------------------------------------------------------------
[  3] local 192.168.159.132 port 52464 connected with 192.168.159.131 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec   139 MBytes  1.16 Gbits/sec
[  3]  1.0- 2.0 sec   136 MBytes  1.14 Gbits/sec
[  3]  2.0- 3.0 sec  78.2 MBytes   656 Mbits/sec
[  3]  3.0- 4.0 sec  79.8 MBytes   669 Mbits/sec
[  3]  4.0- 5.0 sec   112 MBytes   935 Mbits/sec
[  3]  5.0- 6.0 sec   104 MBytes   871 Mbits/sec
[  3]  6.0- 7.0 sec   122 MBytes  1.02 Gbits/sec
[  3]  7.0- 8.0 sec   121 MBytes  1.02 Gbits/sec
[  3]  8.0- 9.0 sec   127 MBytes  1.07 Gbits/sec
[  3]  9.0-10.0 sec   106 MBytes   886 Mbits/sec
[  3]  0.0-10.0 sec  1.10 GBytes   943 Mbits/sec
eddy@eddy:~$
```

相关参数

```bash
eddy@eddy:~$ iperf -h
Usage: iperf [-s|-c host] [options]
       iperf [-h|--help] [-v|--version]

Client/Server:
  -b, --bandwidth #[kmgKMG | pps]  bandwidth to send at in bits/sec or packets per second
  -e, --enhancedreports    use enhanced reporting giving more tcp/udp and traffic information
  -f, --format    [kmgKMG]   format to report: Kbits, Mbits, KBytes, MBytes
  -i, --interval  #        seconds between periodic bandwidth reports
  -l, --len       #[kmKM]    length of buffer in bytes to read or write (Defaults: TCP=128K, v4 UDP=1470, v6 UDP=1450)
  -m, --print_mss          print TCP maximum segment size (MTU - TCP/IP header)
  -o, --output    <filename> output the report or error message to this specified file
  -p, --port      #        server port to listen on/connect to
  -u, --udp                use UDP rather than TCP
      --udp-counters-64bit use 64 bit sequence numbers with UDP
  -w, --window    #[KM]    TCP window size (socket buffer size)
  -z, --realtime           request realtime scheduler
  -B, --bind <host>[:<port>][%<dev>] bind to <host>, ip addr (including multicast address) and optional port and device
  -C, --compatibility      for use with older versions does not sent extra msgs
  -M, --mss       #        set TCP maximum segment size (MTU - 40 bytes)
  -N, --nodelay            set TCP no delay, disabling Nagle's Algorithm
  -S, --tos       #        set the socket's IP_TOS (byte) field

Server specific:
  -s, --server             run in server mode
  -t, --time      #        time in seconds to listen for new connections as well as to receive traffic (default not set)      --udp-histogram #,#  enable UDP latency histogram(s) with bin width and count, e.g. 1,1000=1(ms),1000(bins)
  -B, --bind <ip>[%<dev>]  bind to multicast address and optional device
  -H, --ssm-host <ip>      set the SSM source, use with -B for (S,G)
  -U, --single_udp         run in single threaded UDP mode
  -D, --daemon             run the server as a daemon
  -V, --ipv6_domain        Enable IPv6 reception by setting the domain and socket to AF_INET6 (Can receive on both IPv4 and IPv6)

Client specific:
  -c, --client    <host>   run in client mode, connecting to <host>
  -d, --dualtest           Do a bidirectional test simultaneously
      --ipg                set the the interpacket gap (milliseconds) for packets within an isochronous frame
      --isochronous <frames-per-second>:<mean>,<stddev> send traffic in bursts (frames - emulate video traffic)
  -n, --num       #[kmgKMG]    number of bytes to transmit (instead of -t)
  -r, --tradeoff           Do a bidirectional test individually
  -t, --time      #        time in seconds to transmit for (default 10 secs)
  -B, --bind [<ip> | <ip:port>] bind ip (and optional port) from which to source traffic
  -F, --fileinput <name>   input the data to be transmitted from a file
  -I, --stdin              input the data to be transmitted from stdin
  -L, --listenport #       port to receive bidirectional tests back on
  -P, --parallel  #        number of parallel client threads to run
  -R, --reverse            reverse the test (client receives, server sends)
  -T, --ttl       #        time-to-live, for multicast (default 1)
  -V, --ipv6_domain        Set the domain to IPv6 (send packets over IPv6)
  -X, --peer-detect        perform server version detection and version exchange
  -Z, --linux-congestion <algo>  set TCP congestion control algorithm (Linux only)

Miscellaneous:
  -x, --reportexclude [CDMSV]   exclude C(connection) D(data) M(multicast) S(settings) V(server) reports
  -y, --reportstyle C      report as a Comma-Separated Values
  -h, --help               print this message and quit
  -v, --version            print version information and quit

[kmgKMG] Indicates options that support a k,m,g,K,M or G suffix
Lowercase format characters are 10^3 based and uppercase are 2^n based
(e.g. 1k = 1000, 1K = 1024, 1m = 1,000,000 and 1M = 1,048,576)

The TCP window size option can be set by the environment variable
TCP_WINDOW_SIZE. Most other options can be set by an environment variable
IPERF_<long option name>, such as IPERF_BANDWIDTH.

Source at <http://sourceforge.net/projects/iperf2/>
Report bugs to <iperf-users@lists.sourceforge.net>
eddy@eddy:~$
```

### TCP

server

```bash
iperf -s -i 1 -p 5002  -l 512
------------------------------------------------------------
Server listening on TCP port 5002
TCP window size:  128 KByte (default)
------------------------------------------------------------
[  4] local 192.168.159.131 port 5002 connected with 192.168.159.132 port 57890
[ ID] Interval       Transfer     Bandwidth
[  4]  0.0- 1.0 sec  39.3 MBytes   330 Mbits/sec
[  4]  1.0- 2.0 sec  35.9 MBytes   301 Mbits/sec
[  4]  2.0- 3.0 sec  49.5 MBytes   416 Mbits/sec
[  4]  3.0- 4.0 sec  38.1 MBytes   320 Mbits/sec
[  4]  4.0- 5.0 sec  20.1 MBytes   168 Mbits/sec
[  4]  5.0- 6.0 sec  20.2 MBytes   170 Mbits/sec
[  4]  6.0- 7.0 sec  19.5 MBytes   164 Mbits/sec
[  4]  7.0- 8.0 sec  19.8 MBytes   166 Mbits/sec
[  4]  8.0- 9.0 sec  20.6 MBytes   173 Mbits/sec
[  4]  0.0-10.0 sec   284 MBytes   238 Mbits/sec
```

client

```bash
eddy@eddy:~$ iperf -c 192.168.159.131 -i 1 -p 5002 -t 10 -l 512
------------------------------------------------------------
Client connecting to 192.168.159.131, TCP port 5002
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[  3] local 192.168.159.132 port 57890 connected with 192.168.159.131 port 5002
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  42.0 MBytes   353 Mbits/sec
[  3]  1.0- 2.0 sec  35.9 MBytes   301 Mbits/sec
[  3]  2.0- 3.0 sec  49.9 MBytes   419 Mbits/sec
[  3]  3.0- 4.0 sec  35.1 MBytes   294 Mbits/sec
[  3]  4.0- 5.0 sec  20.1 MBytes   169 Mbits/sec
[  3]  5.0- 6.0 sec  20.1 MBytes   169 Mbits/sec
[  3]  6.0- 7.0 sec  19.5 MBytes   164 Mbits/sec
[  3]  7.0- 8.0 sec  19.7 MBytes   165 Mbits/sec
[  3]  8.0- 9.0 sec  20.6 MBytes   173 Mbits/sec
[  3]  9.0-10.0 sec  20.8 MBytes   175 Mbits/sec
[  3]  0.0-10.0 sec   284 MBytes   238 Mbits/sec
eddy@eddy:~$
```

### UDP

server

```bash
eddy@eddy:~$ iperf -s -i 1 -p 5002  -l 512 -b 1000M -u
------------------------------------------------------------
Server listening on UDP port 5002
Receiving 512 byte datagrams
UDP buffer size:  208 KByte (default)
------------------------------------------------------------
[  3] local 192.168.159.131 port 5002 connected with 192.168.159.132 port 33698
[ ID] Interval       Transfer     Bandwidth        Jitter   Lost/Total Datagrams
[  3]  0.0- 1.0 sec  11.1 MBytes  92.7 Mbits/sec   0.036 ms  398/23031 (1.7%)
[  3]  1.0- 2.0 sec  10.0 MBytes  83.9 Mbits/sec   0.035 ms    0/20483 (0%)
[  3]  2.0- 3.0 sec  10.5 MBytes  88.3 Mbits/sec   0.035 ms   42/21607 (0.19%)
[  3]  3.0- 4.0 sec  7.31 MBytes  61.3 Mbits/sec   0.035 ms  510/15487 (3.3%)
[  3]  4.0- 5.0 sec  9.28 MBytes  77.8 Mbits/sec   0.043 ms    0/18996 (0%)
[  3]  5.0- 6.0 sec  10.1 MBytes  85.1 Mbits/sec   0.039 ms    0/20783 (0%)
[  3]  6.0- 7.0 sec  12.0 MBytes   101 Mbits/sec   0.036 ms    0/24672 (0%)
[  3]  7.0- 8.0 sec  12.1 MBytes   101 Mbits/sec   0.037 ms    0/24767 (0%)
[  3]  8.0- 9.0 sec  10.5 MBytes  88.2 Mbits/sec   0.037 ms   16/21558 (0.074%)
[  3]  9.0-10.0 sec  12.1 MBytes   102 Mbits/sec   0.169 ms    0/24865 (0%)
[  3]  0.0-10.0 sec   105 MBytes  88.2 Mbits/sec   0.169 ms  966/216249 (0.45%)
eddy@eddy:~$
```

client

```bash
eddy@eddy:~$ iperf -c 192.168.159.131 -i 1 -p 5002 -t 10 -l 512 -b 1000M -u
------------------------------------------------------------
Client connecting to 192.168.159.131, UDP port 5002
Sending 512 byte datagrams, IPG target: 3.91 us (kalman adjust)
UDP buffer size:  208 KByte (default)
------------------------------------------------------------
[  3] local 192.168.159.132 port 33698 connected with 192.168.159.131 port 5002
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  11.2 MBytes  94.2 Mbits/sec
[  3]  1.0- 2.0 sec  10.0 MBytes  83.9 Mbits/sec
[  3]  2.0- 3.0 sec  10.6 MBytes  88.5 Mbits/sec
[  3]  3.0- 4.0 sec  7.56 MBytes  63.4 Mbits/sec
[  3]  4.0- 5.0 sec  9.28 MBytes  77.8 Mbits/sec
[  3]  5.0- 6.0 sec  10.1 MBytes  85.0 Mbits/sec
[  3]  6.0- 7.0 sec  12.0 MBytes   101 Mbits/sec
[  3]  7.0- 8.0 sec  12.1 MBytes   101 Mbits/sec
[  3]  8.0- 9.0 sec  10.5 MBytes  88.3 Mbits/sec
[  3]  9.0-10.0 sec  12.2 MBytes   102 Mbits/sec
[  3]  0.0-10.0 sec   106 MBytes  88.6 Mbits/sec
[  3] Sent 216249 datagrams
[  3] Server Report:
[  3]  0.0-10.0 sec   105 MBytes  88.2 Mbits/sec   0.169 ms  966/216249 (0.45%)
eddy@eddy:~$
```

## iperf3

安装

```bash
sudo apt install iperf3
```

帮助手册

```bash
eddy@eddy:~$ iperf3 -h
Usage: iperf3 [-s|-c host] [options]
       iperf3 [-h|--help] [-v|--version]

Server or Client:
  -p, --port      #         server port to listen on/connect to
  -f, --format   [kmgtKMGT] format to report: Kbits, Mbits, Gbits, Tbits
  -i, --interval  #         seconds between periodic throughput reports
  -F, --file name           xmit/recv the specified file
  -A, --affinity n/n,m      set CPU affinity
  -B, --bind      <host>    bind to the interface associated with the address <host>
  -V, --verbose             more detailed output
  -J, --json                output in JSON format
  --logfile f               send output to a log file
  --forceflush              force flushing output at every interval
  -d, --debug               emit debugging output
  -v, --version             show version information and quit
  -h, --help                show this message and quit
Server specific:
  -s, --server              run in server mode
  -D, --daemon              run the server as a daemon
  -I, --pidfile file        write PID file
  -1, --one-off             handle one client connection then exit
  --rsa-private-key-path    path to the RSA private key used to decrypt
                            authentication credentials
  --authorized-users-path   path to the configuration file containing user
                            credentials
Client specific:
  -c, --client    <host>    run in client mode, connecting to <host>
  --sctp                    use SCTP rather than TCP
  -X, --xbind <name>        bind SCTP association to links
  --nstreams      #         number of SCTP streams
  -u, --udp                 use UDP rather than TCP
  --connect-timeout #       timeout for control connection setup (ms)
  -b, --bitrate #[KMG][/#]  target bitrate in bits/sec (0 for unlimited)
                            (default 1 Mbit/sec for UDP, unlimited for TCP)
                            (optional slash and packet count for burst mode)
  --pacing-timer #[KMG]     set the timing for pacing, in microseconds (default 1000)
  --fq-rate #[KMG]          enable fair-queuing based socket pacing in
                            bits/sec (Linux only)
  -t, --time      #         time in seconds to transmit for (default 10 secs)
  -n, --bytes     #[KMG]    number of bytes to transmit (instead of -t)
  -k, --blockcount #[KMG]   number of blocks (packets) to transmit (instead of -t or -n)
  -l, --length    #[KMG]    length of buffer to read or write
                            (default 128 KB for TCP, dynamic or 1460 for UDP)
  --cport         <port>    bind to a specific client port (TCP and UDP, default: ephemeral port)
  -P, --parallel  #         number of parallel client streams to run
  -R, --reverse             run in reverse mode (server sends, client receives)
  --bidir                   run in bidirectional mode.
                            Client and server send and receive data.
  -w, --window    #[KMG]    set window size / socket buffer size
  -C, --congestion <algo>   set TCP congestion control algorithm (Linux and FreeBSD only)
  -M, --set-mss   #         set TCP/SCTP maximum segment size (MTU - 40 bytes)
  -N, --no-delay            set TCP/SCTP no delay, disabling Nagle's Algorithm
  -4, --version4            only use IPv4
  -6, --version6            only use IPv6
  -S, --tos N               set the IP type of service, 0-255.
                            The usual prefixes for octal and hex can be used,
                            i.e. 52, 064 and 0x34 all specify the same value.
  --dscp N or --dscp val    set the IP dscp value, either 0-63 or symbolic.
                            Numeric values can be specified in decimal,
                            octal and hex (see --tos above).
  -L, --flowlabel N         set the IPv6 flow label (only supported on Linux)
  -Z, --zerocopy            use a 'zero copy' method of sending data
  -O, --omit N              omit the first n seconds
  -T, --title str           prefix every output line with this string
  --extra-data str          data string to include in client and server JSON
  --get-server-output       get results from server
  --udp-counters-64bit      use 64-bit counters in UDP test packets
  --repeating-payload       use repeating pattern in payload, instead of
                            randomized payload (like in iperf2)
  --username                username for authentication
  --rsa-public-key-path     path to the RSA public key used to encrypt
                            authentication credentials

[KMG] indicates options that support a K/M/G suffix for kilo-, mega-, or giga-

iperf3 homepage at: https://software.es.net/iperf/
Report bugs to:     https://github.com/esnet/iperf
eddy@eddy:~$
```

```bash
Server 端参数：

-s, --server，以 Server 模式运行；

-D, --daemon，在后台以守护进程运行；

-I, --pidfile file，指定 pid 文件；

-1, --one-off，只接受 1 次来自 Client 端的测试，然后退出。

Client 端参数

-c, --client ，以 Client 模式运行，并指定 Server 端的地址；

-u, --udp，以 UDP 协议进行测试；

-b, --bandwidth #[KMG][/#]，限制测试带宽。UDP 默认为 1Mbit/秒，TCP 默认无限制；

-t, --time #，以时间为测试结束条件进行测试，默认为 10 秒；

-n, --bytes #[KMG]，以数据传输大小为测试结束条件进行测试；

-k, --blockcount #[KMG]，以传输数据包数量为测试结束条件进行测试；

-l, --len #[KMG]，读写缓冲区的长度，TCP 默认为 128K，UDP 默认为 8K；

--cport ，指定 Client 端运行所使用的 TCP 或 UDP 端口，默认为临时端口；

-P, --parallel #，测试数据流并发数量；

-R, --reverse，反向模式运行（Server 端发送，Client 端接收）；

-w, --window #[KMG]，设置套接字缓冲区大小，TCP 模式下为窗口大小；

-C, --congestion ，设置 TCP 拥塞控制算法（仅支持 Linux 和 FreeBSD ）；

-M, --set-mss #，设置 TCP/SCTP 最大分段长度（MSS，MTU 减 40 字节）；

-N, --no-delay，设置 TCP/SCTP no delay，屏蔽 Nagle 算法；

-4, --version4，仅使用 IPv4；

-6, --version6，仅使用 IPv6；

-S, --tos N，设置 IP 服务类型（TOS，Type Of Service）；

-L, --flowlabel N，设置 IPv6 流标签（仅支持 Linux）；

-Z, --zerocopy，使用 “zero copy”（零拷贝）方法发送数据；

-O, --omit N，忽略前 n 秒的测试；

-T, --title str，设置每行测试结果的前缀；

--get-server-output，从 Server 端获取测试结果；

--udp-counters-64bit，在 UDP 测试包中使用 64 位计数器（防止计数器溢出）。
```



### TCP

server

```bash
iperf3 -s -i 1 -p 1314
```

client

```bash
eddy@eddy:~$ iperf3 -c 192.168.159.131 -p 1314 -f m -i 1 -n 1000M -l 1K
Connecting to host 192.168.159.131, port 1314
[  5] local 192.168.159.132 port 44740 connected to 192.168.159.131 port 1314
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  52.7 MBytes   442 Mbits/sec    0    433 KBytes
[  5]   1.00-2.00   sec   102 MBytes   850 Mbits/sec    0    458 KBytes
[  5]   2.00-3.00   sec  95.5 MBytes   804 Mbits/sec    0    484 KBytes
[  5]   3.00-4.00   sec   103 MBytes   864 Mbits/sec    0    512 KBytes
[  5]   4.00-5.00   sec  99.6 MBytes   835 Mbits/sec    0    512 KBytes
[  5]   5.00-6.00   sec  95.1 MBytes   799 Mbits/sec    0    512 KBytes
[  5]   6.00-7.00   sec   107 MBytes   894 Mbits/sec    0    512 KBytes
[  5]   7.00-8.00   sec   104 MBytes   869 Mbits/sec    0    550 KBytes
[  5]   8.00-9.00   sec  93.2 MBytes   783 Mbits/sec    0    622 KBytes
[  5]   9.00-10.00  sec  78.1 MBytes   655 Mbits/sec    0    622 KBytes
[  5]  10.00-11.00  sec  54.6 MBytes   458 Mbits/sec    1   1.24 MBytes
[  5]  11.00-11.16  sec  16.3 MBytes   846 Mbits/sec    0   1.24 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-11.16  sec  1000 MBytes   752 Mbits/sec    1             sender
[  5]   0.00-11.21  sec   994 MBytes   744 Mbits/sec                  receiver

iperf Done.
```



### UDP

server

```bash
iperf3 -s -i 1 -p 1314
-----------------------------------------------------------
Server listening on 1314
-----------------------------------------------------------
Accepted connection from 192.168.159.131, port 58310
[  5] local 192.168.159.131 port 1314 connected to 192.168.159.131 port 57107
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-1.00   sec   113 MBytes   950 Mbits/sec  0.055 ms  35/3660 (0.96%)  
[  5]   1.00-2.00   sec   119 MBytes   999 Mbits/sec  0.048 ms  2/3813 (0.052%)  
[  5]   2.00-3.00   sec   119 MBytes   998 Mbits/sec  0.072 ms  6/3815 (0.16%)  
[  5]   3.00-4.00   sec   117 MBytes   983 Mbits/sec  0.081 ms  66/3815 (1.7%)  
[  5]   4.00-5.00   sec   111 MBytes   932 Mbits/sec  1.192 ms  174/3731 (4.7%)  
[  5]   5.00-6.00   sec   109 MBytes   916 Mbits/sec  0.126 ms  403/3896 (10%)  
[  5]   6.00-7.00   sec   108 MBytes   903 Mbits/sec  0.460 ms  352/3796 (9.3%)  
[  5]   7.00-8.00   sec   116 MBytes   972 Mbits/sec  0.265 ms  118/3834 (3.1%)  
[  5]   8.00-9.00   sec   115 MBytes   968 Mbits/sec  0.140 ms  127/3814 (3.3%)  
[  5]   9.00-10.00  sec   119 MBytes   998 Mbits/sec  0.038 ms  9/3814 (0.24%)  
[  5]  10.00-10.04  sec  4.94 MBytes  1.02 Gbits/sec  0.027 ms  0/158 (0%)  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.04  sec  1.12 GBytes   962 Mbits/sec  0.027 ms  1292/38146 (3.4%)  receiver
```



client

```bash
iperf3 -u -c 192.168.159.131 -b 1000m -t 10 -p 1314
Connecting to host 192.168.159.131, port 1314
[  5] local 192.168.159.131 port 57107 connected to 192.168.159.131 port 1314
[ ID] Interval           Transfer     Bitrate         Total Datagrams
[  5]   0.00-1.00   sec   119 MBytes   999 Mbits/sec  3814  
[  5]   1.00-2.00   sec   119 MBytes  1.00 Gbits/sec  3814  
[  5]   2.00-3.00   sec   119 MBytes  1000 Mbits/sec  3816  
[  5]   3.00-4.00   sec   119 MBytes  1000 Mbits/sec  3814  
[  5]   4.00-5.00   sec   119 MBytes   999 Mbits/sec  3809  
[  5]   5.00-6.00   sec   119 MBytes  1.00 Gbits/sec  3820  
[  5]   6.00-7.00   sec   119 MBytes   999 Mbits/sec  3813  
[  5]   7.00-8.00   sec   119 MBytes  1.00 Gbits/sec  3816  
[  5]   8.00-9.00   sec   119 MBytes   999 Mbits/sec  3815  
[  5]   9.00-10.00  sec   119 MBytes  1.00 Gbits/sec  3815  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
[  5]   0.00-10.00  sec  1.16 GBytes  1000 Mbits/sec  0.000 ms  0/38146 (0%)  sender
[  5]   0.00-10.04  sec  1.12 GBytes   962 Mbits/sec  0.027 ms  1292/38146 (3.4%)  receiver

iperf Done.
```



# 参考

1. [iperf和iperf3详解 - 宅女士 - 博客园 (cnblogs.com)](https://www.cnblogs.com/carriezhangyan/p/11341829.html)
2. [第 4 章 Netperf 网络测试 (brinnatt.com)](https://www.brinnatt.com/projects/第-4-章-netperf-网络测试/#42、Netperf_语法)

## iperf

命令参数

1. [iperf linux 命令 在线中文手册 (51yip.com)](http://linux.51yip.com/search/iperf)

2. [Iperf参数介绍_cqwei1987的博客-CSDN博客_iperf 参数](https://blog.csdn.net/cqwei1987/article/details/107360426)

3. [iperf 和 iperf3 命令以及参数详解 - cloudwas - 博客园 (cnblogs.com)](https://www.cnblogs.com/cloudwas/p/13084815.html#:~:text= perf是一个网络性能测试工具。 数据包丢失。,Iperf在linux和windows平台均有二进制版本供自由使用。 iperf3是用来测量一个网络最大带宽的工具。 通信协议。 每次测试，它都会报告网络带宽，丢包率和其他参数。)

使用参考

1. [iperf网络测试工具 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1688469)

2. [Iperf使用方法与参数说明_Aiolos2015的博客-CSDN博客_iperf 参数说明](https://blog.csdn.net/lxc19861120/article/details/8494456?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-3-8494456-blog-107360426.pc_relevant_recovery_v2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~Rate-3-8494456-blog-107360426.pc_relevant_recovery_v2&utm_relevant_index=6)

## iperf3

1. [Iperf3网络性能测试工具详解教程 - pycod - 博客园 (cnblogs.com)](https://www.cnblogs.com/xuanbjut/p/14144255.html)

2. [iperf3 测试宽带, 设置tcp和udp包大小 - 韩若明瞳 - 博客园 (cnblogs.com)](https://www.cnblogs.com/han-guang-xue/p/14633679.html)

3. [iperf3使用方法详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/314727150)

4. [iperf3网络测试工具使用方法_小白典的博客-CSDN博客_iperf3使用方法](https://blog.csdn.net/Q0717168/article/details/116944260)

5. [【山外笔记-工具框架】iperf3网络性能测试工具详解教程-云社区-华为云 (huaweicloud.com)](https://bbs.huaweicloud.com/blogs/228741)

