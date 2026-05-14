# perf 性能检测工具

`perf` 是 Linux 下的一款性能分析工具，能够进行函数级与指令级的热点查找。

## perf --help

```bash
perf --help

 usage: perf [--version] [--help] [OPTIONS] COMMAND [ARGS]

 The most commonly used perf commands are:
   annotate        解析perf record生成的perf.data文件，显示被注释的代码。
   archive         根据数据文件记录的build-id，将所有被采样到的elf文件打包。利用此压缩包，可以再任何机器上分析数据文件中记录的采样数据。
   bench           perf中内置的benchmark，目前包括两套针对调度器和内存管理子系统的benchmark。
   buildid-cache   管理perf的buildid缓存，每个elf文件都有一个独一无二的buildid。buildid被perf用来关联性能数据与elf文件。
   buildid-list    列出数据文件中记录的所有buildid。
   c2c             Shared Data C2C/HITM Analyzer.
   config          Get and set variables in a configuration file.
   daemon          Run record sessions on background
   data            Data file related processing
   diff            对比两个数据文件的差异。能够给出每个符号（函数）在热点分析上的具体差异。
   evlist          列出数据文件perf.data中所有性能事件。
   ftrace          simple wrapper for kernel's ftrace functionality
   inject          该工具读取perf record工具记录的事件流，并将其定向到标准输出。在被分析代码中的任何一点，都可以向事件流中注入其它事件。
   iostat          Show I/O performance metrics
   kallsyms        Searches running kernel for symbols
   kmem            针对内核内存（slab）子系统进行追踪测量的工具
   kvm             用来追踪测试运行在KVM虚拟机上的Guest OS。
   list            列出当前系统支持的所有性能事件。包括硬件性能事件、软件性能事件以及检查点。
   lock            分析内核中的锁信息，包括锁的争用情况，等待延迟等。
   mem             内存存取情况
   record          收集采样信息，并将其记录在数据文件中。随后可通过其它工具对数据文件进行分析。
   report          读取perf record创建的数据文件，并给出热点分析结果。
   sched           针对调度器子系统的分析工具。
   script          执行perl或python写的功能扩展脚本、生成脚本框架、读取数据文件中的数据信息等。
   stat            执行某个命令，收集特定进程的性能概况，包括CPI、Cache丢失率等。
   test            perf对当前软硬件平台进行健全性测试，可用此工具测试当前的软硬件平台是否能支持perf的所有功能。
   timechart       针对测试期间系统行为进行可视化的工具
   top             类似于linux的top命令，对系统性能进行实时分析。
   version         display the version of perf binary
   probe           用于定义动态检查点。
   trace           关于syscall的工具。

 See 'perf help COMMAND' for more information on a specific command.
```



## Ubuntu20.04安装Perf工具

```bash
sudo apt install linux-tools-common
sudo apt install linux-tools-5.13.0-40-generic
sudo apt install linux-cloud-tools-5.13.0-40-generic
```

## 记录一段时间内系统/进程的性能事件

```bash
perf record
```

### 常用参数

```bash
 -e：选择性能事件
 -p：待分析进程的id
 -t：待分析线程的id
 -a：分析整个系统的性能,即所有CPU采样
 -C：只采集指定CPU数据
 -c：事件的采样周期
 -o：指定输出文件，默认为perf.data
 -A：以append的方式写输出文件
 -f：以OverWrite的方式写输出文件
 -g：记录函数间的调用关系
```

### 例子

```bash
perf record -a -e cycles -o cycle.perf -g sleep 10
[ perf record: Woken up 18 times to write data ]
[ perf record: Captured and wrote 4.953 MB cycle.perf (~216405 samples) ]
```

## 读取生成的数据文件，并显示分析数据

```bash
perf report
```

### 常用参数

```bash
-i：输入的数据文件
-v：显示每个符号的地址
-d <dos>：只显示指定dos的符号
-C：只显示指定comm的信息（Comm. 触发事件的进程名）
-S：只考虑指定符号
-U：只显示已解析的符号
-g[type,min,order]：显示调用关系，具体等同于perf top命令中的-g
-c：只显示指定cpu采样信息
-M：以指定汇编指令风格显示
–source：以汇编和source的形式进行显示
-p<regex>：用指定正则表达式过滤调用函数
```

### 例子

```bash
perf report -i cycle.perf | more
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 34K of event 'cpu-clock'
# Event count (approx.): 8508750000
#
# Children      Self  Command          Shared Object               Symbol
# ........  ........  ...............  ..........................  ...............................................
#
    99.20%     0.00%  swapper          [kernel.kallsyms]           [k] 0xffffffff90c00107
            |
            ---0xffffffff90c00107
               |
               |--49.81%--0xffffffff92e94731
               |          0xffffffff92e9462d
               |          0xffffffff92e95c3e
               |          0xffffffff92e953fb
               |          0xffffffff91963c73
               |          0xffffffff90d09be0
               |          |
               |           --49.81%--0xffffffff90d099a4
               |                     0xffffffff90d096b3
               |                     0xffffffff9164cbde
               |                     0xffffffff9164c607
               |                     |
               |                      --49.79%--0xffffffff919731eb
               |                                |
               |                                 --49.78%--0xffffffff91972a5b
               |
                --49.39%--0xffffffff90c7d81a
                          0xffffffff90d09be0
                          |
                           --49.39%--0xffffffff90d099a4
                                     0xffffffff90d096b3
                                     0xffffffff9164cbde
                                     |
                                      --49.38%--0xffffffff9164c607
                                                |
                                                 --49.37%--0xffffffff919731eb
                                                           0xffffffff91972a5b

    99.20%     0.00%  swapper          [kernel.kallsyms]           [k] 0xffffffff90d09be0
            |
            ---0xffffffff90d09be0
               |
                --99.19%--0xffffffff90d099a4
                          0xffffffff90d096b3
                          0xffffffff9164cbde
                          |
                           --99.19%--0xffffffff9164c607
                                     |
                                      --99.16%--0xffffffff919731eb
                                                |
                                                 --99.15%--0xffffffff91972a5b

--More--
```



## 实时显示系统/进程的性能统计信息

```bash
perf top
```

### 常用参数

```bash
-e：指定性能事件
-a：显示在所有CPU上的性能统计信息
-C：显示在指定CPU上的性能统计信息
-p：指定进程PID
-t：指定线程TID
-K：隐藏内核统计信息
-U：隐藏用户空间的统计信息
-s：指定待解析的符号信息
‘‐G’ or‘‐‐call‐graph’ <output_type,min_percent,call_order>
graph: 使用调用树，将每条调用路径进一步折叠。这种显示方式更加直观。
每条调用路径的采样率为绝对值。也就是该条路径占整个采样域的比率。
fractal
默认选项。类似与 graph，但是每条路径前的采样率为相对值。
flat
不折叠各条调用
选项 call_order 用以设定调用图谱的显示顺序，该选项有 2个取值，分别是
callee 与caller。
将该选项设为callee 时，perf按照被调用的顺序显示调用图谱，上层函数被下层函数所调用。
该选项被设为caller 时，按照调用顺序显示调用图谱，即上层函数调用了下层函数路径，也不显示每条调用路径的采样率
注： Perf top需要root权限
```

### 例子

```bash
sudo perf top -g
```

```c
Samples: 1K of event 'cpu-clock:pppH', 4000 Hz, Event count (approx.): 251003285 lost: 0/0 drop: 0/0
  Children      Self  Shared Object            Symbol
+   42.35%     0.15%  perf                     [.] __dso__load_kallsyms
+   36.54%     0.10%  [kernel]                 [k] vfs_read
+   35.50%     0.05%  libpthread-2.31.so       [.] __libc_read
+   35.50%     0.10%  [kernel]                 [k] ksys_read
+   35.15%     0.05%  [kernel]                 [k] proc_reg_read
+   35.10%     0.10%  [kernel]                 [k] seq_read
+   34.48%     0.72%  [kernel]                 [k] seq_read_iter
+   26.92%     0.41%  [kernel]                 [k] s_show
+   26.62%     0.46%  [kernel]                 [k] seq_printf
+   26.10%     0.41%  [kernel]                 [k] seq_vprintf
+   22.84%     5.11%  [kernel]                 [k] vsnprintf
+   22.52%     0.28%  [kernel]                 [k] do_user_addr_fault
+   21.86%     0.19%  [kernel]                 [k] __handle_mm_fault
+   20.94%     0.36%  libc-2.31.so             [.] __stpncpy_sse2_unaligned
+   20.60%     0.29%  [kernel]                 [k] get_page_from_freelist
+   20.60%     0.05%  [kernel]                 [k] __alloc_pages
+   19.69%    19.69%  [kernel]                 [k] clear_page_orig
+    6.98%     3.36%  [kernel]                 [k] __softirqentry_text_start
+    6.73%     0.80%  [kernel]                 [k] cpuidle_enter_state
+    6.54%     0.26%  [kernel]                 [k] s_next
+    6.50%     6.40%  [kernel]                 [k] __lock_text_start
+    6.44%     0.56%  [kernel]                 [k] update_iter
     6.33%     6.33%  [kernel]                 [k] format_decode
+    6.23%     0.66%  [kernel]                 [k] pointer
     5.72%     5.72%  [kernel]                 [k] number
+    5.36%     0.72%  [kernel]                 [k] string
     4.65%     4.44%  [kernel]                 [k] string_nocheck
For a higher level overview, try: perf top --sort comm,dso
```

`perf output` 监控接口在多个列中显示数据：

- `Children` 列

  表示该符号名下调用函数性能事件在所有采样中的比例

- `Overhead` 列

  显示函数或库占用的 CPU 百分比。

- `Shared Object `列

  显示使用函数的程序或库的名称。

- `Symbol` 列

  显示功能名称或符号。在内核空间中执行的功能由 `[k]` 标识，在用户空间中执行的功能由 `[.]` 标识。

## 用于分析指定程序的性能概况

```bash
perf stat
```

### 常用参数

```bash
-e：选择性能事件
-i：禁止子任务继承父任务的性能计数器。
-r：重复执行 n 次目标程序，并给出性能指标在n 次执行中的变化范围。
-n：仅输出目标程序的执行时间，而不开启任何性能计数器。
-a：指定全部cpu
-C：指定某个cpu
-A：将给出每个处理器上相应的信息。
-p：指定待分析的进程id
-t：指定待分析的线程id
```

### 例子

```bash
# perf stat
sudo perf stat ib_send_bw -d rxe0 -R -s 2M 192.168.159.131
---------------------------------------------------------------------------------------
                    Send BW Test
 Dual-port       : OFF          Device         : rxe0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 TX depth        : 128
 CQ Moderation   : 100
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 2
 Max inline data : 0[B]
 rdma_cm QPs     : ON
 Data ex. method : rdma_cm
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0017 PSN 0xa67937
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:159:131
 remote address: LID 0000 QPN 0x0018 PSN 0xf8296a
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:159:131
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 2097152    1000             228.77             205.87             0.000103
---------------------------------------------------------------------------------------

 Performance counter stats for 'ib_send_bw -d rxe0 -R -s 2M 192.168.159.131':

         10,027.98 msec task-clock                #    0.993 CPUs utilized
               143      context-switches          #   14.260 /sec
                 1      cpu-migrations            #    0.100 /sec
             1,260      page-faults               #  125.648 /sec
   <not supported>      cycles
   <not supported>      instructions
   <not supported>      branches
   <not supported>      branch-misses

      10.100792733 seconds time elapsed

       0.274651000 seconds user
       9.542177000 seconds sys


```

输出解释如下：

`task-clock`：任务真正占用的处理器时间，单位为ms

`CPUs utilized` = task-clock/time elapsed (CPU的占用率)

`context-swichees`：程序在运行过程中的上下文切换次数

`CPU-migrations`：程序在运行过程中发生的处理器迁移次数。Linux为了维持多个处理器的负载均衡在特定条件下会将某个任务从一个CPU迁移到另一个CPU。CPU迁移和上下文切换：发生上下文切换不一定会发生CPU迁移，而发生CPU迁移时肯定会发生上下文切换。发生上下文切换有可能只是把上下文从当前CPU换出，下一次调度器还是将进程安排在这个CPU上执行。

`page-faults`:缺失异常的次数。当应用程序请求的页面尚未建立，请求的页面不在内存中，或者请求的页面虽然在内存中，但物理地址和虚拟地址的映射关系尚未建立时，都会触发一次却也异常。另外TLB不命中，页面访问权限不匹配等情况也会触发缺页异常。

`cycles`：消耗的处理器周期数。

`stalled-cycles-frontend`：指令读取或解码的步骤，未能按理想状态发挥并行左右，发生停滞的时钟周期。

`stalled-cycles-backend`：指令执行步骤，发生停滞的时钟周期。

`instructions`：执行了多少条指令。IPC为平均每个cpu cycle执行了多少条指令。

`branches`：遇到的分支指令数。

`branch-misses` ：是预测错误的分支指令数。

## 相关示例

```bash
perf top -e cycles:k     #显示内核和模块中，消耗最多CPU周期的函数

perf top -e kmem:kmem_cache_alloc     #显示分配高速缓存最多的函数

perf top -g         #得到调用关系图

perf top -e cycles         #指定性能事件

perf top -p 23015,32476         #查看这两个进程的cpu cycles使用情况

perf top -s comm,pid,symbol         #显示调用symbol的进程名和进程号

perf top --comms nginx,top         #仅显示属于指定进程的符号

perf top --symbols kfree         #仅显示指定的符号

perf stat -r 10 ls > /dev/null         #执行10次程序，给出标准偏差与期望的比值

perf stat -v ls > /dev/null         #显示更详细的信息

perf stat -n ls > /dev/null         #只显示任务执行时间，不显示性能计数器

perf stat -a -A ls > /dev/null         #单独给出每个CPU上的信息

perf stat -C 0						#统计CPU 0的信息

perf stat -e syscalls:sys_enter ls          #ls命令执行了多少次系统调用

perf record -p `pgrep -d ',' nginx`      #记录nginx进程的性能数据

perf record ls -g    #记录执行ls时的性能数据

perf record -e syscalls:sys_enter ls      #记录执行ls时的系统调用，可以知道哪些系统调用最频繁

perf lock record ls      #记录

perf lock report      #报告

Name：内核锁的名字。
aquired：该锁被直接获得的次数，因为没有其它内核路径占用该锁，此时不用等待。
contended：该锁等待后获得的次数，此时被其它内核路径占用，需要等待。
total wait：为了获得该锁，总共的等待时间。
max wait：为了获得该锁，最大的等待时间。
min wait：为了获得该锁，最小的等待时间。
perf kmem record ls      #记录

perf kmem stat --caller --alloc -l 20      #报告

Callsite：内核代码中调用kmalloc和kfree的地方。
Total_alloc/Per：总共分配的内存大小，平均每次分配的内存大小。
Total_req/Per：总共请求的内存大小，平均每次请求的内存大小。
Hit：调用的次数。
Ping-pong：kmalloc和kfree不被同一个CPU执行时的次数，这会导致cache效率降低。
Frag：碎片所占的百分比，碎片 = 分配的内存 - 请求的内存，这部分是浪费的。
有使用--alloc选项，还会看到Alloc Ptr，即所分配内存的地址。

perf sched record sleep 10    

perf report latency --sort max 
TASK：进程名和pid。
Runtime：实际的运行时间。
Switches：进程切换的次数。
Average delay：平均的调度延迟。
Maximum delay：最大的调度延迟。
Maximum delay at：最大调度延迟发生的时刻。

perf probe --line schedule    #前面有行号的可以探测，没有行号的就不行了

perf report latency --sort max    #在schedule函数的12处增加一个探测点
```



# 参考

[Tutorial - Perf Wiki (kernel.org)](https://perf.wiki.kernel.org/index.php/Tutorial)

[绿色记忆:利用perf剖析Linux应用程序 (gmem.cc)](https://blog.gmem.cc/perf)

[第 18 章 perf 入门 Red Hat Enterprise Linux 8 | Red Hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/getting-started-with-perf_monitoring-and-managing-system-status-and-performance)

[Linux perf命令详解及常用参数解析 - 寒冰宇若 - 博客园 (cnblogs.com)](https://www.cnblogs.com/wx170119/p/11855263.html)

[系统级性能分析工具perf的介绍与使用 - ArnoldLu - 博客园 (cnblogs.com)](https://www.cnblogs.com/arnoldlu/p/6241297.html)

[系統級性能分析工具 — Perf | Jason note (jasonblog.github.io)](https://jasonblog.github.io/note/linux_tools/xi_tong_ji_xing_neng_fen_xi_gong_ju__perf.html)

[Linux 性能诊断 perf使用指南-阿里云开发者社区 (aliyun.com)](https://developer.aliyun.com/article/65255#slide-21)

[Linux Perf 性能分析工具及火焰图浅析 – 滴滴云博客 (didiyun.com)](https://blog.didiyun.com/index.php/2019/01/02/linux-perf/)

[性能优化工具：perf – duanple](http://duanple.com/?p=1142)

[perf性能分析 | Hexo (melonshell.github.io)](https://melonshell.github.io/2019/10/09/tool1_perf/)

[Linux性能分析工具Perf简介 - 老王系统屋 - SegmentFault 思否](https://segmentfault.com/a/1190000021465563)

