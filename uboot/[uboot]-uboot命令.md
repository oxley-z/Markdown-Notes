# Uboot命令

## 查询操作

### help&?

进入 uboot 的命令行模式以后输入“help”或者 “?”，然后按下回车即可查看当前 uboot 所支持的命令以及简单的示例。

```bash
?       - alias for 'help'
          打印当前注册的所有命令
base    - print or set address offset
		  打印或设置地址偏移量
bdinfo  - print Board Info structure
		  打印板级信息结构
boot    - boot default, i.e., run 'bootcmd'
		  run 'bootcmd'
bootd   - boot default, i.e., run 'bootcmd'
		  run 'bootcmd'
bootelf - Boot from an ELF image in memory
		  从内存中的ELF镜像启动
bootm   - boot application image from memory
		  从内存中引导应用镜像启动
bootp   - boot image via network using BOOTP/TFTP protocol
		  通过BOOTP/TFTP协议使用网络启动镜像
bootvx  - Boot vxWorks from an ELF image
		  使用ELF镜像启动vxWorks
cipher_test- CIPHER Encrypt And Decrypt Test
		  密码加密和解密测试
cmp     - memory compare
		  内存比较
coninfo - print console devices and information
		  打印控制台设备和信息
cp      - memory copy
		  内存拷贝
crc32   - checksum calculation
		  CRC32校验和计算
dcache  - enable or disable data cache
		  使能/失能数据cache
ddr     - ddr training function
		  内存训练函数
decjpg  - jpgd   - decode jpeg picture.
		  解码jpeg图像
decjpg [format]
dhcp    - boot image via network using DHCP/TFTP protocol
		  使用DHCP/TFTP协议通过网络启动映像
dm      - Driver model low level access
		  驱动程序模型低级别访问
echo    - echo args to console
		  将参数输出到控制台
editenv - edit environment variable
		  修改环境变量
env     - environment handling commands
		  环境处理命令
erase   - erase FLASH memory
		  擦除闪存
exit    - exit script
		  退出脚本
false   - do nothing, unsuccessfully
		  什么也不做,但没有成功
flinfo  - print FLASH memory information
		  打印闪存信息
getinfo - print hardware information
		  打印硬件信息
go      - start application at address 'addr'
		  在地址“addr”启动应用程序
go_a53cpu1- start a53-cpu1 at address 'addr'
		  在地址“addr”启动a53-cpu1
gzwrite - unzip and write memory to block device
		  解压并写入内存以阻止设备
hash    - Calcluate hash.
		  计算哈希散列值
hash_test- hash_test [x]:[0] SHA1; [1] SHA256; [2] HMAC-SHA1; [3] HMAC-SHA256;
help    - print command description/usage
		  打印命令描述/使用
icache  - enable or disable instruction cache
		  使能/失能Icacha
iminfo  - print header information for application image
		  打印应用程序图像的标题信息
imxtract- extract a part of a multi-image
		  提取multi-image的一部分
itest   - return true/false on integer compare
		  对整数比较返回true/false
klad_test- KLAD Test
loadb   - load binary file over serial line (kermit mode)
		  通过串行线加载二进制文件(kermit模式)
loads   - load S-Record file over serial line
		  通过串行线加载S-Record文件
loadx   - load binary file over serial line (xmodem mode)
		  通过串行线加载二进制文件(xmodem模式)
loady   - load binary file over serial line (ymodem mode)
		  通过串行线加载二进制文件(ymodem模式)
loop    - infinite loop on address range
		  地址范围无限循环
md      - memory display
		  内存显示
mii     - MII utility commands
		  MII实用程序命令
mm      - memory modify (auto-incrementing address)
		  内存修改(自动递增地址)
mw      - memory write (fill)
		  内存写(填充)
nfs     - boot image via network using NFS protocol
		  使用NFS协议通过网络引导映像
nm      - memory modify (constant address)
		  内存修改(常量地址)
part    - disk partition related commands
		  与磁盘分区相关的命令
ping    - send ICMP ECHO_REQUEST to network host
		  发送ICMP回显请求到网络主机
printenv- print environment variables
		  打印环境变量
protect - enable or disable FLASH write protection
		  启用或禁用FLASH写保护
pxe     - commands to get and boot from pxe files
		  从pxe文件获取和引导的命令
reset   - Perform RESET of the CPU
		  执行CPU重置
rng_test- RNG Test
rsa_enc_test- RSA PKCS1# V1_5 encrypt/decrypt Test
		  RSA PKCS1# V1 5加密/解密测试
rsa_sign_test- RSA PKCS1# V1_5 Sign Test
		  RSA PKCS1# V1 5符号测试
run     - run commands in an environment variable
		  在环境变量中运行命令
saveenv - save environment variables to persistent storage
		  将环境变量保存到持久存储中
setenv  - set environment variables
		  设置环境变量
setexpr - set environment variable as the result of eval expression
		  将环境变量设置为eval表达式的结果
setvobg - setvobg   - set vo backgroud color.
	- setvobg [dev color]
		  设置vo背景颜色。
sf      - SPI flash sub-system
		  SPI flash子系统
showvar - print local hushshell variables
		  打印本地hushshell变量
sleep   - delay execution for some time
		  延迟执行一段时间
source  - run script from memory
		  从内存中运行脚本
startgx - startgx   - open graphics layer.
	- startgx [layer addr stride x y w h]
	  开放图像层。
startvl - startvl   - open video layer.
	- startvl [layer addr stride x y w h]
	  开放视频层。
startvo - startvo   - open vo device with a certain output interface.
	- startvo [dev intftype sync]
	  打开具有特定输出接口的vo设备。
stopgx  - stopgx   - close graphics layer.
	- stopgx [layer]
	  关闭图像层。
stopvl  - stopvl   - close video layer.
	- stopvl [layer]
	  关闭视频层。
stopvo  - stopvo   - close interface of vo device.
	- stopvo [dev]
	  关闭vo设备接口。
test    - minimal test like /bin/sh
tftpboot- boot image via network using TFTP protocol
true    - do nothing, successfully
unzip   - unzip a memory region
		  解压缩内存区域
usb     - USB sub-system
		  USB子系统
usbboot - boot from USB device
		  USB设备引导
version - print monitor, compiler and linker version
		  打印编译器和链接器版本
```

### bdinfo

bdinfo命令用于查看当前开发板的信息。

### version

version 命令用于查看 uboot 的版本号。

![version](image/uboot%E5%91%BD%E4%BB%A4/96a5b030b658bfffc69ea156a5cb6068.jpg)

## BOOT操作

uboot的本质工作是引导操作系统的启动，对于操作系统的引导命令有如下几个，分别是：boot、bootz、bootm、booti、bootefi。

| 命令    | 描述                                | 内核镜像 |
| ------- | ----------------------------------- | -------- |
| boot    | 读取环境变量 bootcmd 来引导内核镜像 |          |
| bootz   | 引导 ARM 内核镜像                   | zImage   |
| bootm   | 引导 u-boot自定义的内核镜像         | uImage   |
| booti   | 引导 ARM64 内核镜像                 | Image    |
| bootefi | 引导 ARM64 压缩内核镜像             | Image.gz |

### boot&bootd

boot 命令用于读取环境变量bootcmd的值来启动Linux内核。示例如下：

```bash
# setenv bootcmd 'fatload mmc 1:1 80800000 zImage; fatload mmc 1:1 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb; bootz 80800000 - 83000000'
# savenev
# boot
```

### bootz

bootz用于启动zImage镜像文件的内核，其命令格式如下：

```bash
bootz [addr [initrd[:size]] [fdt]]
```

	* addr：kernel Image文件所在的memory地址，必选。
	* [initrd[:size]]：initrd文件在memory中的地址位置和大小size，可以不指定，使用“-”代替即可。
	* fdt：设备树dtb文件在memory中的地址，在ARM64中，必选。

示例：

```bash
tftp 80800000 zImage	#下载系统镜像
tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb	#下载设备树等
bootz 80800000 - 83000000	#启动

fatload mmc 1:1 80800000 zImage
fatload mmc 1:1 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb
bootz 80800000 - 83000000
```

### bootm

bootm用于启动uImage镜像文件的内核，其命令格式如下：

```bash
bootm [addr [initrd[:size]] [fdt]]
```

不使用设备树的情况下启动 Linux 内核的命令示例如下：

```bash
bootm 0x42000000   #从内存地址0x42000000启动内核, 启动前需把内核镜像uImage存放到指定的内存地址
```

### booti

booti 是 bootm 命令的一个子集，可用于执行位于内存中的ARM64内核镜像，其命令格式如下：

```bash
booti addr [initrd[:size]] [fdt]
```

### bootelf

bootelf用于从内存中启动elf镜像。

```bash
bootefi <addr> [<fdt_addr> | -]
```

### bootp

bootp 命令将通过bootp 协议下载映像。Bootp 是bootstrap Protocol 的简称，Bootp 协议将使用TCP/IP 网络协议的UDP 67/68 两个通讯端口，客户机从服务器得到IP 地址、服务器的IP地址、启动映像文件名、网关IP 等信息。

```bash
bootp [loadAddress] [[hostIPaddr:]bootfilename]
```

### bootvx

使用ELF镜像启动vxWorks

```bash
bootvx  [address]
```

### bootvx32

启动32位的VxWorks操作系统镜像

```bash
bootvx32  [address]
```

## 运行命令

### go

go命令用于跳到指定的地址处执行应用，命令格式如下：。

```bash
go addr [arg ...]
```

其中addr未DRAM的首地址，arg部分可进行省略。

### run

run 命令用于运行环境变量中定义的命令，比如可以通过“run bootcmd”来运行 bootcmd 中的启动命令，但是 run 命令最大的作用在于运行我们自定义的环境变量。

### source

source命令用于在内存中执行脚本文件，命令格式如下：

```bash
source [<addr>][:[<image>]|#[<config>]]
```

脚本文件支持两种格式`legacy U-Boot image format`和`Flat Image Tree (FIT) `。两种格式的文件都可以用mkimage工具进行创建。

* addr：脚本文件在内存中的位置，默认为 CONFIG_SYS_LOAD_ADDR；
* image：FIT 文件镜像的名称；
* config：FIT 文件中配置的名称。空格后面的#号开始注释。因此如果未指定*addr* ，则必须使用反斜杠对哈希符号进行转义，或者必须用引号引起来的参数。

如果*image*和*config*都被省略，则使用默认配置，或者如果未定义配置，则使用默认镜像。

## 下载相关

### tftp

tftp命令用于uboot与tftp服务器之间下载上传文件

```bash
tftp  address  filename
```

示例：

```bash
tftp 0xa0000000 kernel.bin	#下载tftp服务器的kernel.bin文件至SDRAM的0xa0000000地址
tftp 0xa0000000 kernel.bin 0x10000000	#把位于SDRAM的0xa0000000地址开始的0x10000000字节的数据上传至tftp服务器的kernel.bin文件
```

### tftpboot

tftpboot命令可以使用TFTP协议通过网络下载文件。按照二进制文件格式下载。另外使用这个命令，必须配置好相关的环境变量。例如serverip和ipaddr。

```bash
tftpboot [loadAddress] [bootfilename]
```

### loadb

loadb命令可以通过串口线下载二进制格式文件。

```bash
loadb [addr [baud]]
```

### loads

loads命令可以通过串口线下载S-Record格式文件。

```bash
loads [off]
```

## 环境变量

环境变量在 flash 中有一份，在 DDR 中也有一份。uboot 开机时，一次性从 flash 中读取全部环境变量到 DDR 中作为环境变量的初始值。使用过程中都是使用 DDR 中的这一份。用户使用 setenv 命令修改的就是 DDR 中的那一份环境变量然后通过saveenv命令就是将DDR中的环境变量重新写入 flash，下一次开机时 DDR 从 flash 中读取到的才能是最新的。

### 环境变量命令

#### printenv

printenv命令用于查看当前的环境变量。

```bash
printenv
```

#### setenv

setenv命令通常用于设置、修改、删除环境变量的值，命令格式如下：

```bash
setenv [-f] name value		#设置环境变量name的值为value
setenv [-f] name			#删除环境变量name的值
```

#### saveenv

saveenv命令用于保存环境变量，环境变量在修改过后必须执行此命令保存，否则重启后环境变量，被修改的环境变量会还原回修改之前的值。

### 常用的环境变量

#### 网络相关

##### ipaddr

uboot下的本地IP地址；

##### serverip

uboot通过tftp指令去tftp服务器下载东西时，tftp服务器的IP地址；

##### gatewayip

uboot本地网关地址；

##### netmask

uboot本地子网掩码；

##### ethaddr

本地网卡的MAC地址。

#### 其他

##### bootdelay

如果CONFIG_AUTOBOOT已激活，则在显示以下行后，有bootdelay秒（默认情况下为2秒）通过按任意键进入控制台，若未按下任意按键，则执行bootcmd命令

```bash
Hit any key to stop autoboot:  2
```

##### baudrate

串口控制台的波特率。

##### bootfile

默认的下载文件名。

##### bootargs

启动内核时传递给Linux内核的启动参数，bootm命令启动内核时会自动将bootargs传给内核。

```bash
bootargs=console=ttySAC2,115200 root=/dev/mmcblk0p2 rw init=/linuxrc rootfstype=ext3
```

root

rootfstype

console

mem

init

initrd, noinitrd

ip

##### bootcmd

在bootdelay倒计时结束以后，执行的命令，一般用来执行启动内核命令。

## 内存操作

内存操作中常见`.b`、`.w`、`.l`和`.q`后缀，含义如下

| 后缀 | 全称          | 数据宽度      | 显示格式     |
| ---- | ------------- | ------------- | ------------ |
| `.b` | **byte**      | 8位（1字节）  | 十六进制字节 |
| `.w` | **word**      | 16位（2字节） | 十六进制字   |
| `.l` | **long word** | 32位（4字节） | 十六进制双字 |
| `.q` | **quad word** | 64位（8字节） | 十六进制四字 |

### md

md命令用于查看内存地址的值

```bash
md [.b, .w, .l, .q] address [# of objects]
```

示例：

```bash
md.b 　　0x40008000   100    #从内存地址0x40008000开始，查看0x100个字节并输出值
md.w    0x40008000   100    #从内存地址0x40008000开始，查看0x100个16位值并输出值
md.l    0x40008000   100    #从内存地址0x40008000开始，查看0x100个32位值并输出值  
md.q    0x40008000   100    #从内存地址0x40008000开始，查看0x100个64位值并输出值  
```

### mw

mw命令用于填充内存地址的值。

```bash
mw [.b, .w, .l, .q] address value [count]
```

示例：

```bash
mw.b 82000000 ff 100000 #从地址0x82000000地址开始填充0x100000(1M字节)个值为0xff的数据
```

### mm

mm命令用于修改内存地址的值。

```bash
mm[.b, .w, .l, .q] address
```

示例：

```bash
mm.b 40008000
40008000: ff ? 0xff
40008001: ff ? 0xee
40008002: ff ? 
40008003: ff ? 
40008004: ff ? quit
```

### nm

nm命令用于修改内存内容的值，修改过程内存地址不变

```bash
nm [.b, .w, .l, .q] address
```

### cp

cp命令用于以[.b,.w,.l]为单位从源地址source的内存复制count个数据到目的地址target的内存中去。

```bash
cp [.b, .w, .l, .q] source target [count]
```

### cmp

cmp命令可以比较2块内存中的内容。.b以字节为单位；.w以字为单位；.l以长字为单位。注意：cmp.b中间不能保留空格，需要连续敲入命令。

```bash
cmp [.b, .w, .l, .q] addr1 addr2 count
```

### mtest

mtest 命令是一个简单的内存读写测试命令，可以用来测试自己开发板上的 DDR，命令格式如下：

```bash
mtest [start [end [pattern [iterations]]]]
```

start是要测试的DRAM 开始地址， end 是结束地址。

示例：

测试 0X80000000~0X80001000这段内存，输入“mtest 80000000 80001000”，结果如下所示：

```bash
# mtest 80000000 80001000
Testing 80000000 ... 80001000:
Pattern FFFFFFFF Writing... Reading...Iteration: 7885
```

上述测试中测试内存范围为0x80000000~0x80001000，已经测试7885次，可通过“Ctrl+C”键结束测试。

## 网络操作

### ping

ping命令用于发送icmp报文到主机，通常用于测试网络是否和目标网络畅通。

```bash
ping pingAddress
```

网络畅通时，显示如下：

```bash
# ping 192.168.1.100
host 192.168.1.100 is alive
```

否则显示：

```bash
ping failed; host 192.168.1.100 is not alive
```

<font color=red>注意</font>：只能在 uboot 中 ping 其他的机器，其他机器不能 ping uboot，因为 uboot 没有对 ping 命令做处理，如果用其他的机器 ping uboot 的话会失败！

### dhcp

dhcp命令用于从路由器获取 IP 地址，前提是得连接到路由器上的，如果开发板是和电脑直连的，那么 dhcp 命令就会失效。直接输入 dhcp 命令即可通过路由器获取到 IP 地址，dhcp命令还可用于使用 DHCP/TFTP 协议通过网络引导操作系统映像。

```bash
dhcp [loadAddress] [[hostIPaddr:]bootfilename]
```

dhcp 与 tftpboot 用法和功能基本一样，最终都是从tftpserver 中下载文件到内存指定位置。不同之处在于：

* dhcp 会先用dhcp 协议申请一个ip地址；
* tftpboot 直接使用 env 中的 ipaddr 作为ip 地址。

示例：

```bash
# dhcp 0x20000000 192.168.1.10:uImage	#将tftp服务器下的uImage文件下载到本地0x20000000地址

# dhcp 40000000 sun8i-h3.dtb
net mac = 02:42:04:8f:d7:97
mac is 02:42:04:8f:d7:97
net mac = 02:42:04:8f:d7:97
BOOTP broadcast 1
DHCP client bound to address 172.16.2.36 (1624 ms)
Using ethernet@1c30000 device
TFTP from server 172.16.1.67; our IP address is 172.16.2.36
Filename ‘sun8i-h3-my-board.dtb’.
Load address: 0x40000000
Loading: ##
449.2 KiB/s
done
Bytes transferred = 18433 (4801 hex)
```

### nfs

```bash
nfs [loadAddress] [[hostIPaddr:]bootfilename]

```

loadAddress 是要保存的 DRAM 地址， [[hostIPaddr:]bootfilename]是要下载的文件地址。

示例：

```bash
nfs 80800000 192.168.0.108:/home/eddy/nfs/zImage
```

命令中的 “ 80800000 ” 表 示 zImage 保 存 地 址 ，“192.168.0.108:/home/eddy/nfs/zImage”表示 zImage 在 192.168.0.108 这个主机的位置。

## NAND Flash操作

### nand info

nand info命令用于查看NAND Flash信息，通常会给出NAND Flash的页大小、OOB域大小、擦除大小等信息。

```bash
# nand info
Device 0: NAND 32MiB 3,3V 8-bit, sector size 16 KiB
```

### nand device

nand device命令用于查看当前NAND设备信息或切换当前的NAND设备。

```bash
nand device [dev]
```

### nand erase

nand erase 命令用于擦除 NAND Flash，NAND Flash 的特性决定了在向 NAND Flash 写数据之前一定要先对要写入的区域进行擦除。

```bash
nand erase off|partition size
nand erase clean [off|partition size]
```

示例：

```bash
# nand erase 0x0 0x20000	#从0地址擦除0x20000个字节

NAND erase: device 0 offset 0x0, size 0x20000
Erasing at 0x0 -- 100% complete.
OK
```

`erase`命令从偏移量"`offset`"开始擦除`“size”`字节。或者分区（partition）可以指定名称，在这种情况下，大小最终会受到限制，需要不超过分区大小（此行为也适用于读取和写入命令）。只能擦除完整的NAND块。

如果指定了“`erase`”而没有偏移量或大小，则整个闪存被擦除。如果“`erase`”指定了分区（partition）但没有分区大小，整个分区被擦除。

如果指定了“`clean`”，则将JFFS2样式的擦除标记写入擦除后的每个块。

此命令不会擦除标记为错误的块。

### nand erase.part

擦除指定的分区

```bash
nand erase.part [clean] partition
```

### nand erase.chip

全片擦除

```bash
nand erase.chip [clean]
```

### nand read

从NAND闪存中的 '`ofs`' 读取 '`size`' 字节到 '`addr`'。标记为 “坏” 的块将被跳过。如果由于发现不可纠正的数据错误而无法读取页，则命令会因错误而停止。

```bash
nand read addr ofs|partition size
```

示例：

```bash
# nand read 86000000 58000 100
NAND read: device 0 offset 0x58000, size 0x100
 256 bytes read: OK
```

### nand read.raw

nand read.raw命令用于在 NAND 闪存中将页面从‘ `ofs`’地址读取到‘`addr`’。这会读取原始页面，因此可以避免 ECC，也可以读取 OOB 区域。

```bash
nand read.raw addr ofs|partition
```

### nand read.oob

nand read.oob命令用于从对应于NAND闪存中的 '`ofs`' 的带外数据区域读取 '`size`' 字节到 '`addr`'。仅限于一个512字节页面或2个256字节页面的16字节数据。此命令不检查坏块或更正错误。

```bash
nand read.oob addr ofs|partition size
```

示例：

```bash
# nand read.oob 86000000 58000 10
NAND read: device 0 offset 0x58000, size 0x10
 16 bytes read: OK
```

### nand write

nand write用于在 NAND flash 中将‘`size`’字节的数据从‘`addr`’写到‘`off`’。

```bash
nand write addr ofs|partition size
```

被标记为坏的块将被跳过。如果由于发现不可纠正的数据错误而无法读取页，则该命令将以错误停止。

### nand write.raw

nand write.raw命令用于在 NAND 闪存中将页面从'`addr`'写到'`ofs`'。这将写入原始页面，因此可以避免 ECC，并且也可以写入 OOB 区域，从而使整个页面按原样写入。

```bash
nand write.raw addr ofs|partition
```

### nand write.trimffs

命令由CONFIG_CMD_NAND_TRIMFFS宏启用，此命令将以与上面描述的“ NAND write”命令相同的方式写入 NAND 闪存，附加检查，确保擦除块末尾所有只包含0xff 数据的页面不会写入 NAND 闪存。

```bash
nand write.trimffs addr ofs|partition size
```

### nand write.oob

nand write.oob命令用于在 NAND Flash 中将'`size`'字节从'`addr`'写入对应于'`off`'的带外数据区域。这仅限于一个512字节页或2个256字节页的16字节数据。不检查坏块。

```bash
nand write.oob addr ofs|partition size
```

示例：

```bash
# nand write.oob 86000000 1cc000 10
NAND write: device 0 offset 0x1cc000, size 0x10
 16 bytes written: OK
```

### nand bad

nand bad用于打印当前设备中所有坏块的列表。

```bash
nand bad
```

### nand dump

nand dump命令用于显示Nand Flash中的数据（16进制）

```bash
nand dump[.oob] off
```

示例：

```bash
# nand dump 1cc000 10
Page 001cc000 dump:
       68 73 71 73 12 05 00 00  87 04 08 78 19 00 40 02
       …………
       df 88 5b 80 bb 11 32 11  6e e7 77 5a 3d b9 c8 bf
OOB:
       ff ff ff ff ff ff ff 45
       ed cc 3a e0 8e 52 f9 ad
```

### nand scrub

nand scrub命令用于彻底擦除整块Nand Flash中的数据，包括OOB。可以擦除软件坏块标志，此操作是不安全的。

```bash
nand scrub [-y] off size | scrub.part partition | scrub.chip
```

### nand markbad

nand markbad命令用于标记Nand的 off 偏移地址处的块为坏块，此操作是不安全的。

```bash
nand markbad off [...]
```

### nand lock

将nand设置为锁定状态或显示锁定的页面。

```bash
nand lock [tight] [status]
```

示例：

```bash
# nand lock			#将NAND芯片设置为锁定状态（所有页面锁定）
# nand lock tight	#设置NAND芯片锁紧状态(软件无法更改锁定)
# nand lock status	#显示所有页面的当前锁定状态
```

### nand unlock

```bash
nand unlock [offset] [size]				#解锁连续区域（不同区域可多次调用）
nand unlock[.allexcept] [offset] [size]	#解锁除指定连续区域外的所有区域
```

## NOR Flash操作

### flinfo

flinfo命令打印全部Flash组的信息，也可以只打印其中某个组。

```bash
flinfo
flinfo N	# 单独打Flash存储器N Block的信息。（在有多块Nor Flash时使用）
```

示例：

```bash
# flinfo 1


Bank # 1: SST: 1x SST39VF1601 (2MB)
Size: 2 MB in 32 Sectors
Sector Start Addresses:
00000000 (RO) 00010000 (RO) 00020000 (RO) 00030000 (RO) 00040000
00050000 00060000 (RO) 00070000 (RO) 00080000 00090000
000A0000 000B0000 000C0000 000D0000 000E0000
000F0000 00100000 00110000 00120000 00130000
00140000 00150000 00160000 00170000 00180000
00190000 001A0000 001B0000 001C0000 001D0000
001E0000 001F0000
```

### 擦除操作



### 写保护操作

## QSPI Flash操作

sf probe

sf read

sf write

sf erase

sf update

sf protect

sf test

## mmc命令

mmc info

mmc rescan

mmc list

mmc dev

mmc read

mmc write

mmc part

## pci命令

在使用pci命令之前，需要确保U-Boot启用了PCI支持。这通常通过在编译U-Boot时启用`CONFIG_CMD_PCI`配置选项来实现。

### pci

pci命令用于显示pci总线bus上PCI设备的短或长列表

```bash
# pci
BusDevFun  VendorId   DeviceId   Device Class       Sub-Class
_____________________________________________________________
00.00.00   0x14e4     0x2711     Bridge device           0x04
01.00.00   0x1106     0x3483     Serial bus controller   0x03
```

### pci probe

探测系统中的 PCI 设备，并打印出所有找到的设备的信息。

```bash
# pci probe
Scanning PCI devices on bus 0
BusDevFun  VendorId   DeviceId   Device Class       Sub-Class
_____________________________________________________________
00.00.00   0x17cd     0xdc16     Bridge device           0x04
00.01.00   0x17cd     0xdc08     Bridge device           0x04
00.02.00   0x17cd     0xdc01     Bridge device           0x04
00.03.00   0x17cd     0xdc16     Bridge device           0x04
00.04.00   0x17cd     0xdc08     Bridge device           0x04
00.05.00   0x17cd     0xdc01     Bridge device           0x04
```

### pci info

显示当前系统中所有 PCI 设备的信息，包括设备地址、设备 ID、厂商 ID 等。

```bash
# pci info
Scanning PCI devices on bus 0
BusDevFun  VendorId   DeviceId   Device Class       Sub-Class
_____________________________________________________________
00.00.00   0x17cd     0xdc16     Bridge device           0x04
00.01.00   0x17cd     0xdc08     Bridge device           0x04
00.02.00   0x17cd     0xdc01     Bridge device           0x04
00.03.00   0x17cd     0xdc16     Bridge device           0x04
00.04.00   0x17cd     0xdc08     Bridge device           0x04
00.05.00   0x17cd     0xdc01     Bridge device           0x04
```

### pci header

pci header命令用于显示PCI设备的头部信息

```bash
# pci header 00.00.00
  vendor ID =                   0x16c3
  device ID =                   0xabcd
  command register ID =         0x0007
  status register =             0x0010
  revision ID =                 0x01
  class code =                  0x06 (Bridge device)
  sub class code =              0x04
  programming interface =       0x00
  cache line =                  0x08
  latency time =                0x00
  header type =                 0x01
  BIST =                        0x00
  base address 0 =              0x18000000
  base address 1 =              0x00000000
  primary bus number =          0x00
  secondary bus number =        0x01
  subordinate bus number =      0x01
  secondary latency timer =     0x00
  IO base =                     0x10
  IO limit =                    0x00
  secondary status =            0x0000
  memory base =                 0x1820
  memory limit =                0x1810
  prefetch memory base =        0xfff0
  prefetch memory limit =       0x0000
  prefetch memory base upper =  0x00000000
  prefetch memory limit upper = 0x00000000
  IO base upper 16 bits =       0x0000
  IO limit upper 16 bits =      0x0000
  expansion ROM base address =  0x18100000
  interrupt line =              0xff
  interrupt pin =               0x01
  bridge control =              0x0000
```

### pci bar

pci bar命令用于显示设备 b.d.f 的BAR地址和大小。

```bash
# pci bar 1.0.0
ID   Base                Size                Width  Type
----------------------------------------------------------
 0   0xfffffffffffff000  0x0000000000001000  64     MEM
```

### pci display

显示PCI配置空间。

pci next

pci modify

### pci device

列出当前系统中所有 PCI 设备的设备地址和设备 ID。

### pci read

从指定的 PCI 设备上读取配置空间数据，需要指定设备地址、偏移地址、数据类型和数据长度。

```bash
# pci read[.b .w .l] <bus.dev.func> <reg> [<data-length>]
```

### pci write

向指定的 PCI 设备上的配置空间写入数据，需要指定设备地址、偏移地址、数据类型和数据长度。

```bash
# pci write[.b .w .l] <bus.dev.func> <reg> <data>
```

### pci enum

重新枚举系统中的所有 PCI 设备，并打印出找到的设备的信息。

### pci regions

<font color=red>**注意**</font>：使用 pci 命令需要先启用 CONFIG_CMD_PCI 配置选项。

## usb命令

### usb start

usb start命令用于加载usb驱动，寻找usb控制器。

### usb reset

usb reset命令用于复位usb控制器

### usb info

usb info命令用于输出扫描到的usb外设。

usb stop

usb tree

usb test

usb storage

usb dev

usb part

usb read addr blk

usb write addr blk

## scsi命令

scsi reset

scsi info

scsi scan

scsi device

scsi part

scsi read

scsi write

## nvme命令

nvme scan

nvme detail

nvme info

nvme device

nvme part

nvme read addr blk

nvme write addr blk

## 其他命令

reset

poweroff

crc32

### dcache

dcache命令用于启用或禁用数据（写通）缓存。

```bash
dcache [on/off]
```

### icache

icache命令用于启用或禁用指令缓存。

```bash
icache [on/off]
```

# 参考

[Use U-Boot — Das U-Boot unknown version documentation](https://docs.u-boot.org/en/latest/usage/index.html) uboot手册。

[U-Boot命令之网络操作命令_setenv ethaddr-CSDN博客](https://blog.csdn.net/weixin_45309916/article/details/109178659)

[u-boot中网络命令 ---- dhcp命令_uboot dhcp传文件-CSDN博客](https://blog.csdn.net/voice_shen/article/details/7533412)

[U-Boot命令之常用命令_uboot go-CSDN博客](https://blog.csdn.net/weixin_45309916/article/details/109190028)

[Uboot 常用指令_ubot stdout-CSDN博客](https://blog.csdn.net/dagefeijiqumeiguo/article/details/51314856)

[uboot的常用命令及用法 - 明明是悟空 - 博客园 (cnblogs.com)](https://www.cnblogs.com/x_wukong/p/9244742.html)

[飞腾uboot命令简单介绍 (qq.com)](https://mp.weixin.qq.com/s/qT-ymrp4e_jR_niAfAP65Q)

[uboot常用命令总结 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/news/1230322)

[U-Boot的常用命令详解 - 怀想天空2013 - 博客园 (cnblogs.com)](https://www.cnblogs.com/cyyljw/p/11126310.html)

[uboot常用命令详解 - 一本书的伤痕 - 博客园 (cnblogs.com)](https://www.cnblogs.com/shangbolei/p/4376115.html)

[U-BOOT中文文档 (2) 配置选项(1)-kaifeng_cu-ChinaUnix博客](http://blog.chinaunix.net/uid-20774803-id-3001423.html)

[U-BOOT中文文档 (2) 配置选项(2)-kaifeng_cu-ChinaUnix博客](http://blog.chinaunix.net/uid-20774803-id-3001424.html)

[U-BOOT中文文档 (2) 配置选项(3)-kaifeng_cu-ChinaUnix博客](http://blog.chinaunix.net/uid-20774803-id-3001425.html)

[Uboot常用命令使用](https://www.cnblogs.com/liangliangge/p/12214305.html)
