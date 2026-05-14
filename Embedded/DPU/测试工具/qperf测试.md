# qperf测试


qperf 可以用来测试两个节点之间的带宽（bandwidth）和延迟（latency），不仅仅可以用来测试 TCP/IP 协议的性能指标，还可以用来测试 RDMA 传输的指标。使用方法是：一个节点运行 qperf 作为服务端，另一个节点则运行 qperf 作为客户端，与服务端建立连接之后打流，获取带宽和延迟等数据。

为了使用 RoCE 运行 qperf，应该在客户端添加 -cm1 标志。(mellonx)

GitHub源码

[https://github.com/linux-rdma/qperf.git](https://github.com/linux-rdma/qperf.git)



## 安装

```bash
sudo apt install qperf
```

```bash
$ qperf 192.168.2.140 -lca 5 -rca 5 -t 10 -vvu tcp_lat udp_lat tcp_bw udp_bw conf

    qperf是QLogic（千兆逻辑公司）开发的网络性能测试工具。
    192.168.2.140是远程测试节点的IP地址。
    -lca 5和-rca 5分别表示本地和远程CPU亲和性，将测试程序绑定到第5个CPU核心上。
    -t 10表示测试时间为10秒。
    -vvu表示开启详细输出，包括每个测试项的详细性能数据。
    tcp_lat和udp_lat分别表示TCP和UDP的单向延迟测试。
    tcp_bw和udp_bw分别表示TCP和UDP的双向吞吐量测试。
    conf表示输出测试节点的配置信息。
    输出结果显示了针对远程节点的四个不同性能测试的结果，以及测试节点的配置信息：

    tcp_lat：TCP的单向延迟测试结果，latency为1.61us，msg_size为1字节，表示测试发送和接收1字节的数据所需的平均时间。
    udp_lat：UDP的单向延迟测试结果，latency为1.42us，msg_size为1字节，表示测试发送和接收1字节的数据所需的平均时间。
    tcp_bw：TCP的双向吞吐量测试结果，bw为13.3GB/sec，msg_size为64KB，表示测试在10秒内双向传输64KB数据时的平均带宽。
    udp_bw：UDP的双向吞吐量测试结果，send_bw和recv_bw均为10.6GB/sec，msg_size为32KB，表示测试在10秒内发送和接收32KB数据时的平均带宽。
    conf：测试节点的一些配置信息，包括本地节点和远程节点的名称、CPU类型、操作系统、qperf版本等。
```

## 帮助文档

### qperf --help

```bash
$ qperf --help
    qperf：在本地节点上运行 qperf 并充当服务器。
    qperf SERVERNODE [OPTIONS] TESTS：在远程节点上运行 qperf 并充当客户端。SERVERNODE 是服务器节点的名称，TESTS 是测试的名称。
    qperf 可以测量以下网络性能指标：

    带宽：每秒传输的数据量。
    延迟：数据从一个节点传输到另一个节点所需的时间。
    CPU 利用率：在测试期间，每个节点的 CPU 使用率。
    qperf 支持多种测试，包括：

    TCP 流测试：测量 TCP 流的带宽和延迟。
    UDP 流测试：测量 UDP 流的带宽和延迟。
    RDMA 读测试：测量 RDMA 读操作的带宽和延迟。
    RDMA 写测试：测量 RDMA 写操作的带 bandwidth 和延迟。
    qperf 还支持多种选项，用于控制测试参数，如消息大小、测试持续时间等。

    要获取有关 qperf 的更多信息，请使用 --help 选项。以下是使用 --help 选项的示例：

    qperf --help examples：显示一些使用 qperf 的示例。
    qperf --help opts：显示 qperf 的选项的简要说明。
    qperf --help options：显示 qperf 的选项的详细说明。
    qperf --help tests：显示 qperf 支持的测试的简要说明。
    qperf --help TESTNAME：显示 qperf 的测试 TESTNAME 的详细信息。
```



### qperf --help opts

```bash
$ qperf --help opts
    --access_recv OnOff (-ar)           打开/关闭访问接收数据
    -ar1                              开启访问接收数据
    --alt_port Port (-ap)               设置备用路径端口
    --loc_alt_port Port (-lap)        设置本地备用路径端口
    --rem_alt_port Port (-rap)        设置远程备用路径端口
    --cpu_affinity PN (-ca)             设置处理器亲和性
    --loc_cpu_affinity PN (-lca)      设置本地处理器亲和性
    --rem_cpu_affinity PN (-rca)      设置远程处理器亲和性
    --flip OnOff (-f)                   打开/关闭发送方和接收方的互换
    -f1                               打开发送方和接收方的互换
    --help Topic (-h)                   获取有关主题的更多信息
    --host Node (-H)                    标识服务器节点
    --id Device:Port (-i)               设置RDMA设备和端口
    --loc_id Device:Port (-li)        设置本地RDMA设备和端口
    --rem_id Device:Port (-ri)        设置远程RDMA设备和端口
    --listen_port Port (-lp)            设置服务器监听端口
    --loop Var:Init:Last:Incr (-oo)     循环遍历数值
    --msg_size Size (-m)                设置消息大小
    --mtu_size Size (-mt)               设置MTU大小（仅适用于RDMA）
    --no_msgs Count (-n)                发送指定数量的消息
    --cq_poll OnOff                     设置轮询模式开启/关闭
    --loc_cq_poll OnOff (-lcp)        设置本地轮询模式开启/关闭
    --rem_cq_poll OnOff (-rcp)        设置远程轮询模式开启/关闭
    -cp1                              开启轮询模式
    -lcp1                             开启本地轮询模式
    -rcp1                             开启远程轮询模式
    --ip_port Port (-ip)                设置用于测试的TCP端口
    --precision Digits (-e)             设置报告的精度
    --rd_atomic Max (-nr)               设置RDMA读/原子计数
    --loc_rd_atomic Max (-lnr)      设置本地RDMA读/原子计数
    --rem_rd_atomic Max (-rnr)      设置远程RDMA读/原子计数
    --service_level SL (-sl)            设置服务级别
    --service_level SL (-lsl)         设置本地服务级别
    --service_level SL (-rsl)         设置远程服务级别
    --sock_buf_size Size (-sb)          设置套接字缓冲区大小
    --loc_sock_buf_size Size (-lsb)   设置本地套接字缓冲区大小
    --rem_sock_buf_size Size (-rsb)   设置远程套接字缓冲区大小
    --src_path_bits num (-sp)           设置源路径位数
    --loc_src_path_bits num (-lsp)    设置本地源路径位数
    --rem_src_path_bits num (-rsp)    设置远程源路径位数
    --static_rate (-sr)                 设置IB静态速率
    --loc_static_rate (-lsr)          设置本地IB静态速率
    --rem_static_rate (-rsr)          设置远程IB静态速率
    --time Time (-t)                    设置测试持续时间
    --timeout Time (-to)                设置超时时间
    --loc_timeout Time (-lto)         设置本地超时时间
    --rem_timeout Time (-rto)         设置远程超时时间
    --unify_nodes (-un)                 统一节点
    --unify_units (-uu)                 统一单位
    --use_bits_per_sec (-ub)            使用比特/秒而非字节/秒
    --use_cm OnOff (-cm)                使用RDMA连接管理器或不使用
    -cm1                              使用RDMA连接管理器
    --verbose (-v)                      详细模式；打开所有的 -v\[cstu\]
    --verbose_conf (-vc)              显示配置信息
    --verbose_stat (-vs)              显示统计信息
    --verbose_time (-vt继续上面的翻译：

    --verbose_used (-vu)              显示参数信息
    --verbose_more (-vv)              更详细的模式；打开所有的 -v\[CSTU\]
    --verbose_more_conf (-vvc)        显示更多的配置信息
    --verbose_more_stat (-vvs)        显示更多的统计信息
    --verbose_more_time (-vvt)        显示更多的时间信息
    --verbose_more_used (-vvu)        显示更多的参数信息
    --version (-V)                      打印版本信息
    --wait_server Time (-ws)            设置等待服务器的时间
```

### qperf --help options

```bash
$ qperf --help options
    --access_recv OnOff (-ar) ：如果 OnOff 非零，则接收到的数据将被访问。否则，数据将被忽略。默认值为 0。这可以帮助模拟某些应用程序。
    --alt_port Port (-ap) ：设置备用路径端口。这可以启用自动路径故障转移。
    --cpu_affinity PN (-ca) ：将 CPU 亲和性设置为 PN。CPU 从 0 开始按顺序编号。如果 PN 为“任何”，则允许使用任何 CPU，否则 CPU 将限于指定的 CPU。
    --flip OnOff (-f) ：如果非零，则发送方和接收方将扮演相反的角色。
    --help Topic (-h) ：打印有关 Topic 的信息。要查看主题列表，请键入 qperf --help。
    --host Host (-H) ：在当前节点和运行 qperf 的节点 Host 之间运行测试。这也可以作为第一个非选项参数指定。
    --id Device:Port (-i) ：使用 RDMA 设备和端口。
    --listen_port Port (-lp) ：将我们监听的端口设置为 ListenPort。这必须在服务器和客户机机器上设置为相同的端口。默认值为 19765。
    --loop Var:Init:Last:Incr (-oo) ：连续运行测试，按一系列值进行排序。Var 是循环变量；Init 是初始值；Last 是不得超过的值；Incr 是增量。在连接此选项时，设置 --verbose_used (-vu) 选项很有用。
    --msg_size Size (-m) ：将消息大小设置为 Size。默认值因测试而异。假设值以字节为单位指定，但如果值以 kib 或 K、mib 或 M、gib 或 G 结尾，则表示大小是以 kibibyte、mebibyte 或 gibibyte 为单位指定，而如果值以 kb 或 k、mb 或 m、gb 或 g 结尾，则表示大小是以 kilobyte、megabyte 或 gigabyte 为单位指定。
    --mtu_size Size (-mt) ：设置 MTU 大小。仅适用于 RDMA UC/RC 测试。单位以与 --msg_size 选项相同的方式指定。
    --no_msgs N (-n) ：根据发送的消息数而不是时间来设置测试持续时间。
    --cq_poll OnOff (-cp) ：将轮询模式打开或关闭。这仅适用于 RDMA 测试，它确定它们是轮询还是等待完成队列。如果 OnOff 为 0，则它们会等待；否则它们会轮询。
    --ip_port Port (-ip) ：使用端口运行套接字测试。这与 --listen_port 不同，--listen_port 用于同步。这仅适用于套接字测试，它指的是 TCP/UDP/SDP/RDS/SCTP 端口，测试在该端口上运行。
    --precision Digits (-e) ：设置用于报告结果的有效数字数。
    --rd_atomic Max (-nr) ：将 RDMA 读取或原子操作的最大可处理的操作数设置为 Max。这仅适用于 RDMA 读取和原子测试。
    --service_level SL (-sl) ：将 RDMA 服务级别设置为 SL。这仅由 RDMA 测试使用。服务级别必须在 0 到 15 之间。默认服务级别为 0。
    --sock_buf_size Size (-sb) ：设置套接字缓冲区大小。这仅适用于套接字测试。
    --src_path_bits N (-sp) ：设置源路径位。如果 LMC 不为零，则这将导致连接使用具有低位 LMC 位设置为 N 的 LID。
    --loc_src_path_bits N (-lsp)：设置本地源路径位。如果 LMC 不为零，则这将导致连接使用具有低位 LMC 位设置为 N 的 LID。
    --rem_src_path_bits N (-rsp)：设置远程源路径位。
    --static_rate Rate (-sr)：强制 InfiniBand 静态速率。速率可以是以下任意一个：2.5、5、10、20、30、40、60、80、120、1xSDR（2.5 Gbps）、1xDDR（5 Gbps）、1xQDR（10 Gbps）、4xSDR（2.5 Gbps）、4xDDR（5 Gbps）、4xQDR（10 Gbps）、8xSDR（2.5 Gbps）、8xDDR（5 Gbps）和 8xQDR（10 Gbps）。
    --loc_static_rate (-lsr)：强制本地 InfiniBand 静态速率。
    --rem_static_rate (-rsr)：强制远程 InfiniBand 静态速率。
    --time Time (-t) ：将测试持续时间设置为 Time。以秒为单位指定，但如果值以 m、h 或 d 结尾，则表示时间以分钟、小时或天为单位指定。
    --timeout Time (-to) ：将超时设置为 Time。这是用于各种事情的超时，例如交换消息。默认值为 5 秒。
    --unify_nodes (-un) ：统一节点。将它们描述为本地和远程，而不是发送和接收。
    --unify_units (-uu) ：统一结果显示的单位。使用最小公倍数。对脚本很有帮助。
    --use_bits_per_sec (-ub) ：在显示网络速度时使用比特/秒而不是字节/秒。
    --use_cm OnOff (-cm) ：如果 OnOff 非零，则使用 RDMA 连接管理器 (CM)。对于 iWARP 设备，必须使用 CM。默认情况下，会建立连接，而不使用 CM。这仅适用于使用 RC 传输的测试。
    -cm1 ：使用 RDMA 连接管理器。
    --verbose (-v) ：提供更详细的输出。启用 --verbose_conf、--verbose_stat、--verbose_time 和 -verbose_used。
    --verbose_conf (-vc) ：提供配置信息。
    --verbose_stat (-vs) ：提供统计信息。
    --verbose_time (-vt) ：提供计时信息。
    --verbose_used (-vu) ：提供使用的参数信息。
    --verbose_more (-vv) ：提供更详细的输出。启用 --verbose_more_conf、--verbose_more_stat、--verbose_more_time 和 -verbose_more_used。
    --verbose_more_conf (-vvc) ：提供更多配置信息。
    --verbose_more_stat (-vvs) ：提供更多统计信息。
    --verbose_more_time (-vvt) ：提供更多计时信息。
    --verbose_more_used (-vvu) ：提供更多使用的参数信息。
    --version (-V) ：打印 qperf 的当前版本。
    --wait_server Time (-ws) ：如果服务器没有准备好，则继续尝试连接 Timeout 秒，然后放弃。默认值为 5 秒。
```

### man qperf

```bash
QPERF(1) 用户命令 QPERF(1)

姓名
        qperf - 测量 RDMA 和 IP 性能

概要
        性能
        qperf 服务器节点 [选项] 测试

描述
        qperf 测量两个节点之间的带宽和延迟。 它可以通过 TCP/IP 以及 RDMA 传输工作。 在其中一个节点上，qperf 通常不带参数运行
        将其指定为服务器节点。 然后，可以在客户端节点上运行 qperf 来获取带宽、延迟和 CPU 利用率等测量结果。

        在其最基本的形式中，qperf 通过不带参数调用它在一个节点上以服务器模式运行。 在另一个节点上，它使用两个参数运行：服务器节点的名称
        接下来是测试的名称。 测试列表可以在测试部分找到。 还可以指定多种选项。

        通过使用 --help 选项，可以获取有关 qperf 的更多详细信息。 以下是使用 --help 选项的示例：

            qperf --help Examples 使用 qperf 的一些示例
            qperf --help opts 选项摘要
            qperf --help options 选项说明
            qperf --help 测试 测试的简短摘要和描述
            qperf --help TESTNAME 有关测试 TESTNAME 的更多信息

例子
        在这些示例中，我们首先通过不带参数调用 qperf 在服务器模式下名为 myserver 的节点上运行 qperf。 在所有后续示例中，我们在另一个节点上运行 qperf
        并连接到我们假设主机名为 myserver 的服务器。

        要运行 TCP 带宽和延迟测试：
               qperf myserver tcp_bw tcp_lat

        要运行 SDP 带宽测试 10 秒：
               qperf myserver -t 10 sdp_bw

        要运行 UDP 延迟测试然后导致服务器终止：
               qperf myserver udp_lat quit

        要测量 RDMA UD 延迟和带宽：
               qperf myserver ud_lat ud_bw

        要测量 RDMA UC 双向带宽：
               qperf myserver rc_bi_bw

        获取消息大小从 1 到 64K 的 TCP 延迟范围
               qperf myserver -oo msg_size:1:64K:*2 -vu tcp_lat

选项
        -ar, --access_recv 开关
               如果 OnOff 不为零，则一旦接收到数据就进行访问。 否则，数据将被忽略。 默认情况下，OnOff 为 0。这可以帮助模拟某些应用程序。

        -ar1 导致访问接收到的数据。

        -ap, --alt_port 端口
               设置备用路径端口。 这将启用自动路径故障转移。

        -lap, --loc_alt_port 端口
               设置本地备用路径端口。 这将启用自动路径故障转移。

        -rap, --rem_alt_port 端口
               设置远程备用路径端口。 这将启用自动路径故障转移。

        -ca, --cpu_affinity PN
               将 cpu 亲和性设置为 PN。 CPU 从 0 开始按顺序编号。如果 PN 为“any”，则允许任何 cpu，否则 cpu 仅限于指定的一个。

        -lca, --loc_cpu_affinity PN
               将本地处理器关联设置为 PN。

        -rca, --rem_cpu_affinity PN
               将远程处理器关联设置为 PN。

        -f, --flip 开关
               如果非零，则导致发送者和接收者扮演相反的角色。

        -f1 使发送者和接收者扮演相反的角色。

        -h, --help 主题
               打印出有关Topic的信息。 要查看主题列表，请输入 qperf --help

        -H, --host 主机
               在当前节点和节点 Host 上运行的 qperf 之间运行测试。 这也可以指定为第一个非选项参数。

        -i, --id 设备:端口
               使用 RDMA 设备和端口。

        -li, --loc_id 设备:端口
               使用本地 RDMA 设备和端口。

        -ri, --rem_id 设备:端口
               使用远程 RDMA 设备和端口。

        -lp, --listen_port 端口
               将我们监听的端口设置为ListenPort。 必须将其设置为服务器和客户端计算机上的相同端口。 默认值为 19765。

        -oo, --loop Var:Init:Last:Incr
               通过一系列值排序多次运行测试。 Var是循环变量； init为初始值； Last 是不得超过的值，Incr 是
               增量。 与此选项结合设置 --verbose_used (-vu) 选项非常有用。

        -m, --msg_size 大小
               将消息大小设置为大小。 默认值因测试而异。 假设该值以字节为单位指定，但是尾随 kib 或 K、mib 或 M、gib 或 G
               表示大小分别以 kibibytes、mebibytes 或 gibibytes 为单位指定，而尾随的 kb 或 k、mb 或 m、gb 或 g 表示千字节、兆字节
               或千兆字节。

        -mt, --mtu_size 大小
        	    设置 MTU 大小。 仅与 RDMA UC/RC 测试相关。 单位的指定方式与 --msg_size 选项相同。

        -n, --no_msgs N
               通过发送的消息数而不是时间来设置测试持续时间。

        -cp, --cq_poll 开关
               打开或关闭轮询模式。 这仅与 RDMA 测试相关，并确定它们是轮询还是等待完成队列。 如果 OnOff 为 0，则等待； 其他-
               他们明智地进行民意调查。

        -lcp, --loc_cq_poll 开关
               本地打开或关闭轮询模式。

        -rcp, --rem_cq_poll 开关
               远程打开或关闭轮询模式。

        -cp1 打开轮询模式。

        -lcp1 打开本地轮询模式。

        -rcp1 打开远程轮询模式。

        -ip, --ip_port 端口
               使用 Port 运行套接字测试。 这与用于同步的--listen_port 不同。 这仅与套接字测试相关，并指的是
               运行测试的 TCP/UDP/SDP/RDS/SCTP 端口。

        -e, --精确数字
               设置用于报告结果的有效位数。

        -nr, --rd_atomic 最大
               将 RDMA 读取或原子操作可处理的进行中操作的数量设置为 Max。 这仅与 RDMA 读取和原子测试相关。

        -lnr, --loc_rd_atomic 最大
               设置本地读取/原子计数。

        -rnr, --rem_rd_atomic 最大
               设置远程读取/原子计数。

        -sl, --service_level SL
               将 RDMA 服务级别设置为 SL。 这仅由 RDMA 测试使用。 服务级别必须介于 0 到 15 之间。默认服务级别为 0。

        -lsl, --loc_service_level SL
               设置本地服务级别。

        -rsl, --rem_service_level SL
               设置远程服务级别。

        -sb, --sock_buf_size 大小
               设置套接字缓冲区大小。 这仅与套接字测试相关。

        -lsb, --loc_sock_buf_size 大小
               设置本地套接字缓冲区大小。

        -rsb, --rem_sock_buf_size 大小
               设置远程套接字缓冲区大小。

        -sp, --src_path_bits N
               设置源路径位。 如果 LMC 不为零，这将导致连接使用低位 LMC 位设置为 N 的 LID。

        -lsp, --loc_src_path_bits N
               设置本地源路径位。

        -rsp, --rem_src_path_bits N
               设置远程源路径位。

        -sr, --static_rate 速率
               强制 InfiniBand 静态速率。 速率可以是以下之一：2.5、5、10、20、30、40、60、80、120、1xSDR (2.5 Gbps)、1xDDR (5 Gbps)、1xQDR (10 Gbps)、4xSDR (2.5 Gbps)、4xDDR ( 5
               Gbps）、4xQDR（10 Gbps）、8xSDR（2.5 Gbps）、8xDDR（5 Gbps）、8xQDR（10 Gbps）。

        -lsr, --loc_static_rate
               强制本地 InfiniBand 静态速率

        -rsr, --rem_static_rate
               强制远程 InfiniBand 静态速率

        -t, --time 时间
               将测试持续时间设置为时间。 以秒为单位指定，但尾随 m、h 或 d 分别表示时间以分钟、小时或天为单位指定。

        -to, --超时时间
               将超时设置为时间。 这是用于交换消息等各种事情的超时。 默认值为 5 秒。

        -lto, --loc_timeout 时间
               将本地超时设置为“时间”。 这可以在服务器上用于设置最初与每个客户端交换数据时的超时。 然而，一旦我们收到
               客户端的参数，客户端的远程超时会覆盖该参数。

        -rto, --rem_timeout 时间
               将远程超时设置为“时间”。

        -un, --unify_nodes
               统一节点。 用本地和远程而不是发送和接收来描述它们。

        -uu, --unify_units
               统一显示结果的单位。使用最小公分母。 对脚本很有帮助。

        -ub, --use_bits_per_sec
               显示网络速度时使用位/秒而不是字节/秒。

        -cm, --use_cm 开关
               如果 OnOff 不为零，则使用 RDMA 连接管理器 (CM)。 iWARP 设备需要使用 CM。 默认是建立连接而不使用
               厘米。 这仅适用于使用 RC 传输的测试。

        -cm1 使用 RDMA 连接管理器。

        -v, --详细
               提供更详细的输出。 打开 -vc、-vs、-vt 和 -vu。

        -vc, --verbose_conf
               提供有关配置的信息。

        -vs, --verbose_stat
               提供统计信息。

        -vt, --verbose_time
               提供有关时间安排的信息。

        -vu，--verbose_used
               提供有关所使用参数的信息。

        -vv，--verbose_more
               提供更详细的输出。 打开 -vvc、-vvs、-vvt 和 -vvu。

        -vvc, --verbose_more_conf
               有关配置的更多信息。

        -vvs，--verbose_more_stat
               提供有关统计的更多信息。

        -vvt，--verbose_more_time
               提供有关时间安排的更多信息。

        -vvu, --verbose_more_used
               提供有关所使用参数的更多信息。

        -V，--版本
               打印 qperf 的当前版本。

        -ws, --wait_server 时间
               如果服务器尚未准备好，请继续尝试连接 Time 秒，然后再放弃。 默认值为 5 秒。

测试
        conf 显示配置

        quit 导致服务器退出

        rds_bw RDS 流单向带宽

        rds_lat
               RDS 单向延迟

        sctp_bw
               SCTP 流单向带宽

        sctp_lat
               SCTP 单向延迟

        sdp_bw SDP 流单向带宽

        sdp_lat
               SDP单向延迟

        tcp_bw TCP 流单向带宽

        tcp_lat
               TCP 单向延迟

        udp_bw UDP 流单向带宽

        udp_lat
               UDP 单向延迟

        rc_bi_bw
               RC 串流两路带宽

        rc_bw RC 流单向带宽

        rc_lat RC 单向延迟

        uc_bi_bw
               UC 流媒体双向带宽

        uc_bw UC 流单向带宽

        uc_lat UC 单向延迟

        ud_bi_bw
               UD 流媒体双向带宽

        ud_bw UD 流单向带宽

        ud_lat UD 单向延迟

        xrc_bi_bw
               XRC 流式传输双向带宽

        xrc_bw XRC 流单向带宽

        xrc_lat
               XRC 单向延迟

        rc_rdma_read_bw
               RC RDMA 读取流单向带宽

        rc_rdma_read_lat
               RC RDMA 读取单向延迟

        rc_rdma_write_bw
               RC RDMA 写流单向带宽

        rc_rdma_write_lat
               RC RDMA 写入单向延迟

        rc_rdma_write_poll_lat
               RC RDMA 写入单向轮询延迟

        uc_rdma_write_bw
               UC RDMA 写流单向带宽

        uc_rdma_write_lat
               UC RDMA 写入单向延迟

        uc_rdma_write_poll_lat
               UC RDMA 写入单向轮询延迟

        rc_compare_swap_mr
               RC 比较和交换消息速率

        rc_fetch_add_mr
               RC 获取并添加消息传递速率

        ver_rc_compare_swap
               验证 RC 比较和交换

        ver_rc_fetch_add
               验证 RC 获取并添加

作者
        由约翰·乔治撰写。

错误
        如果在没有 RDMA 库的情况下编译 qperf，则所有 RDMA 测试均不可用。 如果在没有 XRC 扩展的情况下编译 qperf，则所有 XRC 测试均不可用。 这
        -f 选项在许多测试中尚未实现。

qperf 2018 年 4 月 QPERF(1)
```

### 常用参数

以下是qperf常用参数说明：

```bash
-t 测试时间，单位为秒，默认值为10秒。
-l 数据长度，单位为字节，默认值为64字节。
-p 测试端口，默认值为端口号为19765。
-i 测试的间隔时间，默认值为1s。
-u 使用UDP协议进行测试。
-v 输出详细的测试结果。可以多次使用增加详细度。
-s 指定服务器地址。
-ca 指定客户端的CPU亲和性（即将进程绑定到某个CPU核心上）。
-rca 指定远程服务器的CPU亲和性。
-lca 指定本地服务器的CPU亲和性。
-fb 指定反向测试的数据流。
-u 指定使用UDP协议进行测试。
-ua 指定UDP测试的数据包大小，默认为1470字节。
```


## 测试

服务端

```bash
qperf
```

客户端

send/receive

```bash
qperf -cm1 192.168.159.131 rc_lat
qperf -cm1 192.168.159.131 rc_bw
```

write/read

```bash
qperf -cm1 192.168.159.131 rc_rdma_write_lat
qperf -cm1 192.168.159.131 rc_rdma_write_bw
```

数据包size： -m

```bash
qperf  -cm1 172.17.31.51 -m 1M rc_bw
```

数据包数量： -n

```bash
qperf  -cm1 172.17.31.51 -m 1M -n 1000 rc_bw
```

循环测试

```bash
qperf 127.0.0.1 -oo msg_size:1:4K:*2 -v tcp_lat
```



# 参考

[qperf - 手册页部分 1： 用户命令 (oracle.com)](https://docs.oracle.com/cd/E56344_01/html/E54075/qperf-1.html)

[qperf 网络性能测试工具_爱吃李子的十豆三的博客-CSDN博客_qperf测试](https://blog.csdn.net/u014516643/article/details/124803638)

[网络测试qperf_code_____monkey的博客-CSDN博客_qperf测试](https://blog.csdn.net/yonghutwo/article/details/123662592?ops_request_misc=%7B%22request%5Fid%22%3A%22166808120516782388022633%22%2C%22scm%22%3A%2220140713.130102334..%22%7D&request_id=166808120516782388022633&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-123662592-null-null.142^v63^control,201^v3^add_ask,213^v2^t3_esquery_v3&utm_term=网络测试qperf&spm=1018.2226.3001.4187)

[Measuring Network Performance in Linux with qperf - OpsDash](https://www.opsdash.com/blog/network-performance-linux.html)

[Measuring Network Performance in Linux with qperf - OpsDash](https://www.opsdash.com/blog/network-performance-linux.html)

[qperf(1): Measure RDMA/IP performance - Linux man page (die.net)](https://linux.die.net/man/1/qperf)

[网络性能测试工具qperf使用_51CTO博客_网络测试工具](https://blog.51cto.com/u_15333820/3464257)

[7.3. 配置 IPoIB 后使用 qperf 测试 RDMA 网络 Red Hat Enterprise Linux 8 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/configuring_infiniband_and_rdma_networks/testing-an-rdma-network-using-qperf-after-ipoib-is-configured_testing-infiniband-networks)

[linux-rdma/qperf (github.com)](https://github.com/linux-rdma/qperf)



