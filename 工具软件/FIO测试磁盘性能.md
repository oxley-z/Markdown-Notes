```bash
### IOPS（I/O ops/sec）
单位：每秒完成的 IO 请求数。与 block size 有直接关系（同样 IOPS、较大 block size => 更高吞吐量）。

### 带宽（BW）
单位：KB/s、MB/s（或 MiB/s）。BW ≈ IOPS × block_size（相同 block_size 时成立）。

### 延迟（latency）
fio 报告通常包含：slat（submit latency）、clat（completion latency）和 lat（总延迟 = slat + clat）。单位通常是 usec（微秒）或 ms。

### 队列深度（iodepth）
每个 job 在内核/设备前可以同时挂起的未完成 IO 数。更高 iodepth 可以提高并行性与吞吐，但也可能增加延迟。

### numjobs / threads
同时运行的并发 job 数（每个 job 又有 iodepth）。总并发 = numjobs × iodepth（近似）。

### 随机 vs 顺序（rand vs seq）
随机 I/O 对 SSD/HDD 的表现差异很大；数据库类负载通常更关注 4K 随机读/写和尾延迟。

### O_DIRECT / direct=1
绕过内核 page cache（测真实设备性能），通常测试物理设备时应启用 direct=1。

### 预热（preconditioning）
对 SSD 做随机写测试前，要先“预热”设备直到性能稳定（否则会测到缓存/擦写放大等非稳态表现）。
```

# 常用操作

## 顺序写

注意：此处需要改为文件，如果直接使用已分区的磁盘，将导致磁盘分区及数据丢失

```bash
sudo ./fio -filename=/home/your_mount_point/testfile -direct=1 -iodepth 1 -thread -rw=write -ioengine=psync -bs=16k -size=20G -numjobs=8 -runtime=100 -group_reporting -name=mytest
```

## 顺序读

```bash
sudo ./fio -filename=/home/your_mount_point/testfile -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=16k -size=20G -numjobs=8 -runtime=100 -group_reporting -name=mytest
```

## 随机读

```bash
sudo ./fio -filename=/home/your_mount_point/testfile -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=16k -size=20G -numjobs=8 -runtime=100 -group_reporting -name=mytest
```

## 随机写

```bash
sudo ./fio -filename=/home/your_mount_point/testfile -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=16k -size=20G -numjobs=8 -runtime=100 -group_reporting -name=mytest
```

## 混合随机读写

```bash
sudo ./fio -filename=/home/your_mount_point/testfile -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=20G -numjobs=8 -runtime=100 -group_reporting -name=mytest
```

## 随机读时延

```bash
sudo ./fio -filename=/home/your_mount_point/testfile -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=1M -numjobs=1 -runtime=100 -group_reporting -name=mytest
```

## 随机写时延

```bash
sudo ./fio -filename=/home/your_mount_point/testfile -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=4k -size=1M -numjobs=1 -runtime=100 -group_reporting -name=mytest
```





# 主要参数

`--name=<name>`：测试名称（job的名字）

`--filename=<path>`：文件或设备路径（例如 `/mnt/testfile` 或 `/dev/nvme0n1`）

>  可以是具体的文件位置，也可以是设备文件
>
>  注意，如果写裸设备（如 `/dev/nvme0n1`）会抹掉数据，请注意系统和数据安全，测试原始磁盘性能时一定要确认设备是未挂载使用的。
>
>  ioengine=libaio时不能对目录并发IO

`--size=1G`：文件大小（或每 job 的目标大小）

> 如何合理的设置size的大小？？？
>
> 如果你只想测设备极限性能,size 可以设得较小，比如 1G 或 10G，要注意小数据量可能命中缓存；如果是评估稳定性能（更真实）推荐size至少大于设备缓存，甚至大于内存容量；如果是长时间压力测试（带 --time_based）一般使用time_based 和runtime=60（或更久），再设一个比较大的size（比如设备容量的 50% 或固定 100G/500G）
>
> **数据库（OLTP，小块随机 IO）**
>
> - `--size` 建议 ≥ 内存大小（比如 2× 内存），避免命中页缓存。
>
> **日志/队列（顺序写）**
>
> - `--size` 可以相对小，但最好覆盖设备写缓存（几 GB 以上）。
>
> **大数据/顺序读写**
>
> - `--size` 建议几十 GB 以上，模拟大文件吞吐。
>
> **虚拟化/分布式存储**
>
> - `--size` 建议设置为设备容量的 30%~50%，更接近生产环境。

`--ioengine=libaio|io_uring|sync|psync`：I/O 后端。

> fio 的核心参数之一，决定了 fio 发起 I/O 请求的方式。不同引擎对应着不同的 **I/O 调用模型**（fio 在执行 I/O 时，需要调用操作系统/库函数来“发请求”；不同的 `ioengine` 就是不同的 I/O 实现方式，比如 同步/异步、系统调用/内核接口、用户态/内核态；）

| 特性         | psync                                | libaio                                  |
| :----------- | :----------------------------------- | :-------------------------------------- |
| **I/O 模型** | 同步阻塞 (`pread`/`pwrite`)          | 异步非阻塞 (Linux Native AIO)           |
| **并发方式** | 靠多线程 (`numjobs`) 堆并发          | 靠单线程内多队列 (`iodepth`) 实现高并发 |
| **性能上限** | 较低，受限于线程切换开销             | 高，CPU 开销低，更适合高 IOPS 场景      |
| **典型用途** | 模拟传统同步应用（如部分数据库配置） | 测试硬件极限，或模拟高并发异步应用      |
| **必要参数** | `numjobs` 设高                       | `iodepth` 设高，且**必须** `direct=1`   |
| **如何选择** | **模拟真实同步负载**                 | **追求极限性能**                        |

`--direct=1`：使用 O_DIRECT（绕过 page cache）。

`--rw=read|write|randread|randwrite|randrw|readwrite`：I/O 模式。

`--bs=4k|1M`：每个 I/O 的大小（block size）。

> **小块（1K ~ 8K）**
>
> - 单次 I/O 数据少，IOPS 高，但吞吐低；
> - 延迟对业务影响更明显；
> - 适合模拟数据库（OLTP）这类小块随机访问。
>
> **中等块（16K ~ 64K）**
>
> - IOPS 和吞吐兼顾；
> - 很多日志型/中等负载业务会落在这里。
>
> **大块（128K ~ 1M 甚至更大）**
>
> - 单次 I/O 数据大，IOPS 低，但吞吐高；
> - 适合模拟顺序扫描、大文件拷贝、大数据分析。

`--iodepth=32`：队列深度（对 libaio、io_uring 有效）。

> 每个job任务向IO引擎提交IO请求，内核会把请求放入队列，最大可挂起32个请求，使队列维持32的“飞行深度”，配置建议参考如下，具体的设置要结合自己的业务场景模式进行，重点在于你的场景中更多的关注那些指标。
>
> **SSD（SATA SSD）**
>
> - 控制器并行度有限，一般 `iodepth=32` 就够，超过 64 提升不大。
>
> **NVMe SSD**
>
> - 支持很高队列深度（通常 64k），但 fio 实测常用范围是 `1 ~ 256`；
> - 推荐测试点：`iodepth=1, 4, 16, 32, 64, 128`，看性能和延迟曲线。
>
> **HDD（机械盘）**
>
> - 物理寻道瓶颈明显，高队列深度提升有限；
> - `iodepth=1~4` 就能看到真实表现，再大意义不大。
>
> **企业存储/RAID/分布式存储**
>
> - 建议扫描从小到大（1, 4, 16, 32, 64, 128, 256），找到性能饱和点。

`--numjobs=4`：并发 job 数。

`--time_based` / `--runtime=60`：以时间为基准运行（比如跑 60 秒）。

> 默认情况下，fio 运行时是“写完多少数据”就结束，例如你设定 `--size=1G`，那么每个 job 会写/读 1GB 后退出。加上 `--time_based`，fio 会忽略 `--size`（除非 `--size` 比运行时间还小），只按时间运行，直到 `--runtime` 到期才停止，也会配合--ramp_time=10使用，意味着前10 秒不记录结果（给设备/缓存“预热”），之后才算正式数据

`--group_reporting`：合并 job 报告，便于查看总吞吐/IOPS。

`--rwmixread=70`：读写混合时设置读占比（适用于 `randrw`）。

`--output=xxx` / `--output-format=json`：把结果导出为文件或 JSON，便于自动分析。

`--buffered=1/0`：是否使用文件系统缓存（通常用 `direct=1` 替代）。

> 通常使用--direct=1，否则你可能只在测内存缓存，得到假高的吞吐

`--fallocate=1`：先用 fallocate 预分配文件，避免测试时文件系统分配干扰。

# 输出解读

```bash
randread: (groupid=0, jobs=4): err= 0: pid=12345: Thu Sep  5 10:00:00 2025
  read: IOPS=45300, BW=176.95MiB/s (185.55MB/s)(10.00GiB/60001msec)
    slat (usec): min=2, max=52, avg=3.56, stdev=0.76
    clat (usec): min=10, max=2000, avg=150.12, stdev=50.34
     lat (usec): min=12, max=2002, avg=153.68, stdev=50.36
    clat percentiles (usec):
     |  1.00th=[  40],  5.00th=[  60], 10.00th=[  80], 50.00th=[ 140],
     | 90.00th=[ 200], 95.00th=[ 300], 99.00th=[ 500], 99.90th=[1500]
  cpu          : usr=5.00%, sys=10.00%, ctx=12345, majf=0, minf=100
```

> `IOPS=45300`：整个组（4 个 job 合计）的每秒 IO 数。
>
> `BW=176.95MiB/s (185.55MB/s)`：吞吐量。这里同时给出二进制 MiB/s（除以 1024*1024）和十进制 MB/s（除以 1,000,000）。
> 计算示例（以便理解）：`IOPS × block_size` → `45300 × 4096 bytes = 185,548,800 bytes/s`
> 然后：`185,548,800 / 1,048,576 = 176.953125 MiB/s`（这是上面 176.95MiB/s 的由来）。
>
> `10.00GiB/60001msec`：测试期间传输了 10GiB，用时 ~60.001 秒。
>
> `slat (usec)`：submit latency（提交延迟），即 fio 线程把请求递交给内核所花的时间。通常很小（微秒级）。
>
> `clat (usec)`：completion latency（完成延迟），即从提交到 IO 完成在设备/内核层面的耗时，反映磁盘/控制器延迟与排队延迟。
>
> `lat (usec)`：总延迟 = slat + clat。
>
> `clat percentiles`：延迟百分位分布，非常重要！例如 `99.90th = 1500 usec` 意味着 99.9% 的 IO 在 1.5ms 内完成，尾部 0.1% 的 IO 超过 1.5ms。对于延迟敏感服务，这些尾部很关键（不能只看平均）。
>
> `cpu`：fio 运行期间的 CPU 使用情况（usr/sys/context switches/major/minor faults），用于判断是否 CPU 成为瓶颈。



# 参考

[博客园-磁盘性能测试工具FIO-使用教程](https://www.cnblogs.com/nanxi-xz/p/19078589)

[阿里云-测试块存储性能](https://help.aliyun.com/zh/ecs/user-guide/test-the-performance-of-block-storage-devices#section-f8v-673-2po)