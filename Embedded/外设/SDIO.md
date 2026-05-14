## SD卡容量描述

1. 标准容量卡（SDSC）：最大容量为2GB。

2. 高容量卡（SDHC）：容量大小为2~32GB的卡。

3. 扩展容量卡（SDXC）：容量大小为32GB~2TB的卡。

## SD卡寄存器
&emsp;&emsp;SD卡共有8个寄存器，用于设定或表示SD卡信息。这些寄存器只能通过对应的命令访问，对SD卡进行控制操作需要使用对应命令来操作，SDIO共定义了64个命令，每个命令都有特殊的意义，可以实现一些特定的功能，SD卡接收到命令后，根据相应命令对相关寄存器进行修改，程序控制中只需要发送组合命令就能实现SD卡的控制和读写操作。

<center>SD卡寄存器</center>

| 寄存器名称 | 宽度 | 描述 |
| -- | -- | -- |
| CID | 128 | 卡识别号（Card identification number）：用于识别卡的个体号码（唯一的）。 |
| RCA | 16 | 相对地址（Relative card address）：卡的本地系统地址，初始化时，动态的由卡建议，主机核准。 |
| DSR | 16 | 驱动级寄存器（Driver Stage Register）：配置卡的输出驱动。 |
| CSD | 128 | 卡的特定数据（Card Specific Data）：卡的操作条件信息。 |
| SCR | 64 | SD配置寄存器（SD Configuration Register）：有关SD卡的特殊功能信息。 |
| OCR | 32 | 操作条件寄存器（Operation Conditions Register）。 |
| SSR | 512 | SD状态（SD Status）：SD卡专有特征信息。 |
| CSR | 32 | 卡状态（Card Status）：卡状态信息。 |



## SDIO命令

### SD命令类型
* 无响应广播命令（bc）：发送到所有卡，不返回任务响应；
* 带响应广播命令（bcr）：发送到所有卡，同时接收来自所有卡响应；
* 寻址命令（ac）：发送到选定卡，DAT线无数据传输；
* 寻址数据传输命令（adtc）：发送到选定卡，DAT线有数据传输。



<center>基本命令（class 0）</center>

| SDIO命令 | 缩写 | 描述 |
|--|--|--|
| CMD0 | GO_IDLE_STATE | 重置所有卡至Idle状态 |
| CMD1 | 保留 | 保留 |
| CMD2 | ALL_SEND_CID | 要求所有卡发送CID号 |
| CMD3 | SEND_RELATIVE_ADDR | 要求所有卡发布一个新的相对地址 |
| CMD4 | SET_DSR | 对所有的卡DSR寄存器进行编程 |
| CMD5 | 保留 | 保留 |
| CMD6 | 保留 | 保留 |
| CMD7 | SELECT/DESELECT_CARD | 根据获取的指定的RCA，选中SD卡。如果在选中一个卡的状态下，有选中其他的卡，那么之前的卡会自动取消选中，如果发送地址0，则表示取消选中所有卡。当RCA等于0时，主机可以执行以下操作的其中一个：1、使用其他RCA号码取消选择卡。2、重新发送CMD3以将其RCA编号更改为除0以外的其他值，然后使用RCA=0的CMD7选择卡。 |
| CMD8 | SEND_IF_COND | 发送SD存储卡接口信息，包括主机供电信息并询问是否支持该电压。保留位设置为0 |
| CMD9 | SEND_CSD | SD卡从CMD线发送CSD相关数据给Host |
| CMD10 | SEND_CID | SD卡从CMD线发送CID相关数据给Host |
| CMD11 | VOLATILE_SWITCH | 切换总线电平至1.8V |
| CMD12 | STOP_TRANSMISSION | 强制卡停止传输 |
| CMD13 | SEND_STATUS | 被选中的卡返回状态 |
| CMD14 | 保留 | 保留 |
| CMD15 | GO_INACTIVE_STATE | 使选中的卡进入Inactive(非活动)状态，当Host明确想要停用卡时发送此命令，保留位设置为0。 |

<center>面向块的读命令（class 2）</center>

| CMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| CMD16 | SET_BLOCKLEN | 对于标准容量SD存储卡，此命令设置所以后续块命令（读取、写入、锁定）的块长度（以字节为单位）。默认块长度固定为512Bytes，仅当CSD中允许部分块读取操作时，设置长度才对内存访问命令有效。对于SDHC和SDXC卡，块长度由CMD16设置 |
| CMD17 | READ_SINGLE_BLOCK | 对于标准SD卡，此命令用于读取SD卡的一个块的数据，读取的大小由SET_BLOCKLEN命令指定。对于SDHC和SDXC卡，块大小固定为512Bytes。 |
| CMD18 | READ_MULTIPLE_BLOCK | 连续读取SD卡多数据块的数据，直到主机发送CMD12指令。该指令带一个参数，表示要读取的数据块首地址，块长度由CMD16命令设置。 |
| CMD19 | SEND_TUNING_BLOCK | 在调整模式下发送64bytes的特定数据块，用于SDR50和SDR104模式最佳采样点的检测。 |
| CMD20 | SPEED_CLASS_CONTROL | 速度等级控制命令。 |
| CMD21 | 保留 | 保留 |
| CMD22 | 保留 | 保留 |
| CMD23 | SET_BLOCK_COUNT | 指定CMD18和CMD25的块个数 |

<center>面向块的写命令（class 4）</center>

| CMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| CMD16 | SET_BLOCKLEN | 见class2 |
| CMD20 | SPEED_CLASS_CONTROL | 速度等级控制命令 |
| CMD23 | SET_BLOCK_COUNT | 指定CMD18和CMD25的块个数 |
| CMD24 | WRITE_BLOCK | 写入单块数据，对于标准SD卡（SDSC）块长度取决于CMD16，对于SDHC和SDXC卡，块长度固定为512Bytes |
| CMD25 | WRITE_MULTIPLE_BLOCK | 连续写如多个数据块，直到出现STOP_TRANSMISSION命令，块长度与WRITE_BLOCK一致 |
| CMD26 | 保留 | 保留 |
| CMD27 | PROGRAM_CSD | 写CSD寄存器 |

<center> 面向块的写保护命令（class 6） </center>

| CMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| CMD28 | SET_WRITE_PROT | 如果SD卡支持写保护功能，此命令设置寻址组的写保护位，SDHC和SDXC不支持此命令。 |
| CMD29 | CLR_WRITE_PROT | 如果SD卡支持写保护功能，此命令清除寻址组的写保护位，SDHC和SDXC不支持此命令。 |
| CMD30 | SEND_WRITE_PROT | 如果SD卡支持写保护功能，此命令用于获取SD卡的写保护位的状态，SDHC和SDXC不支持此命令。 |

<center>擦除命令（class 5）</center>

| CMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| CMD32 | ERASE_WR_BLK_START | 设置要擦除的第一个写入块的地址。 |
| CMD33 | ERASE_WR_BLK_END | 设置要连续擦除的最后一个块的地址。 |
| CMD38 | ERASE | 擦除所有先前选择的写入块。 |
| CMD39 | 保留 | 保留 |
| CMD41 | 保留 | 保留 |

<center>锁定卡（class 7）</center>

| CMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| CMD16 | SET_BLOCKLEN | 见class2 |
| CMD40 | 为安全规范保留 | 为安全规范保留 |
| CMD42 | LOCK_UNLOCK | 用于设置/重置密码或锁定/解锁卡，数据块的大小由SET_BLOCK_LEN设置，参数和锁卡数据中的保留位应设置为0。 |
| CMD43-49 CMD51 | 保留 | 保留 |

<center>特定于应用程序的命令（class 8）</center>

| CMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| CMD55 | APP_CMD | 向卡发送下一个命令时应用程序特定命令而不是标准命令。 |
| CMD56 | GEN_CMD | 用于将数据块传输到卡或从卡获取数据块以用于通用/应用特定命令。对于SDSC卡块长度由SET_BLOCK_LEN设置。对于SDHC和SDXC卡，块长度固定为512Bytes。主机设置RD/WR=1用于从卡读取数据，设置为0用于向卡写入数据。 |
| CMD58-59 CMD60-63 | 保留 | 保留 |

<center>I/O模式命令（class 9）</center>

| CMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| CMD52-54 | SDIO指令 | SDIO指令 |

<center>SD卡应用程序特定命令</center>

| ACMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| ACMD1-5 | 保留 | 保留 |
| ACMD6 | SET_BUS_WIDTH | 定义数据位宽（00=1位，10=4位总线）用于数据转移。允许的数据总线宽度在SCR寄存器中给出。 |
| ACMD7-12 | 保留 | 保留 |
| ACMD13 | SD_STATUS | 发送SD卡状态。 |
| ACMD14-16 | 为安全规范保留 | 为安全规范保留 |
| ACMD17 | 保留 | 保留 |
| ACMD18 | 保留，用于SD安全应用程序 | 保留，用于SD安全应用程序 |
| ACMD19-21 | 保留 | 保留 |
| ACMD22 | SEND_NUM_WR_BLOCKS | 发送已写入（无错误）的数据块个数，相应为32bit+CRC数据块，如果WRITE_BL_PAARTIAL = '0'，则ACMD22的长度始终为512Bytes，如果WRITE_BL_PAARTIAL = '1'，则ACMD22的长度是执行写命令时使用的块长度。 |
| ACMD23 | SET_WR_BLK_ERASE_COUNT | 设置写入前要预擦除的写入块数（用来加速多数据块操作） '1' = default）一个块[^1]。 |
| ACMD24 | 保留 | 保留 |
| ACMD25 | 为SD卡安全应用预留 | 为SD卡安全应用预留 |
| ACMD26 | 为SD卡安全应用预留 | 为SD卡安全应用预留 |
| ACMD27-28 | 为安全应用预留 | 为安全应用预留 |
| ACMD29 | 保留 | 保留 |
| ACMD30-35 | 为安全规范保留 | 为安全规范保留 |
| ACMD36-37 | 保留 | 保留 |
| ACMD38 | 为SD卡安全应用预留 | 为SD卡安全应用预留 |
| ACMD39-40 | 保留 | 保留 |
| ACMD41 | SD_SEND_OP_COND | 发送主机容量支持信息（HCS）并要求被访问的卡在CMD线路的响应中发送其状态寄存器（OCR）的响应内容。当卡收到SEND_IF_COND时HCS有效。 |
| ACMD42 | SET_CLR_CARD_DETECT | 连接[1]/断开[0]卡上的CD/DAT(pin 1)的50Ω上拉电阻。 |
| ACMD43 ACMD49 |  | 为SD卡安全应用预留 |
| ACMD51 | SEND_SCR | 读取SD卡配置寄存器CSR |
| ACMD52-54 | 为安全规范保留 | 为安全规范保留 |
| ACMD55 |  | 相当于CMD55 |
| ACMD56-59 | 为安全规范保留 | 为安全规范保留 |

[^1]:无论是否使用预擦除 (ACMD23) 功能，都应使用命令 STOP_TRAN(CMD12) 来停止 Write Multiple Block 中的传输。

<center>开关功能命令（class10）</center>

| CMD命令 | 缩写 | 描述 |
| -- | -- | -- |
| CMD6 | SWITCH_FUNC | 检查可切换功能（模式0）和开关卡功能（模式1）。 |
| CMD34-37 CMD50 CMD57 | 为CMD6设置的每个命令系统保留 | 为CMD6设置的每个命令系统保留 |



