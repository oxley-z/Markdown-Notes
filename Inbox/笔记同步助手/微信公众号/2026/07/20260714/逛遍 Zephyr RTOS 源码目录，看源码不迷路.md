---
author: AI与嵌入式OS系统
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzY5NzM0Mjg4NA==&mid=2247483750&idx=1&sn=92635dd3d64c4f1d17dd8643cb7b7757&chksm=f5020b2d09241f302028edf1ab2479b9f7d7edb99160a6dcf7bddba53499a806651076c65e03&mpshare=1&scene=1&srcid=0714YCx42txkl5XbAarteQI0&sharer_shareinfo=5748c75b84b034281a96799324a72154&sharer_shareinfo_first=5748c75b84b034281a96799324a72154#rd
saved: 2026-07-14 22:03:51
tags:
  - 笔记同步助手
id: 00ea4a2d-c080-4b8c-b964-e60febc2c09a
---

公众号名称：AI与嵌入式OS系统应用

作者名称：AI与嵌入式OS系统

发布时间：2026-07-12 21:09

搞嵌入式的朋友多少听过 Zephyr。但第一次把它 clone 下来，面对上百个文件夹、几千个源文件，大概率会懵一会儿——这玩意儿从哪儿看起？

这篇文章我用大白话把 Zephyr 的每个文件夹翻一遍，说清楚它们分别是干嘛的、谁跟谁有关系、写代码的时候该去哪儿找。读完应该对 Zephyr 的整体架构心里有数了。

---

## 顶层的"大院子"：west 工作区

![[Inbox/笔记同步助手/微信公众号/2026/07/images/cd13d5ddcd5e24579eb01c90a6a69769_MD5.jpg]]

你 clone 下来的 `zephyr/` 根目录，不只是一个 git 仓库，而是一个 west 工作区。

什么是 west？它是 Zephyr 官方的多仓库管理工具，类似 Google 的 repo。Zephyr 项目太大，不可能全塞进一个 git 仓库，所以核心代码放在 `zephyr` 仓库，各厂家的 HAL、第三方库、bootloader 拆成几十个独立 git 仓库，用 west 统一拉取管理。

顶层目录长这样：

```
zephyr/                  ← clone 下来的根目录（west 工作区）
├── .west/               ← west 的配置目录
├── zephyr/              ← Zephyr 核心代码仓库（主体）
├── bootloader/          ← Bootloader 仓库（MCUboot）
├── modules/             ← 外部依赖模块仓库
├── tools/               ← 开发工具仓库
├── libgcc_s_fix/        ← 工具链补丁
└── .vscode/             ← VS Code 配置
```

逐个说。

### `.west/` —— west 的"身份证"

目录不大但重要。里面存放 west 工作区的元数据：工作区叫什么、用哪个版本的 manifest、各模块映射到哪个路径。

看到某个文件夹里有 `.west/`，就知道这是一个 west 工作区。类似于大院的物业管理处，记录整个工作区的信息。

### `zephyr/` —— 核心代码库

Zephyr RTOS 的主体代码，也是今天要重点逛的地方。后面展开讲。

### `bootloader/` —— 启动加载器

通常是 MCUboot，一个专为嵌入式设备设计的安全启动加载器。

干啥用的？设备上电后总得先跑一小段代码，把主程序加载起来、验证固件有没有被篡改、检查要不要升级——这就是 bootloader 的活。

MCUboot 支持安全启动（密码学签名验证固件完整性）、OTA 升级（A/B 分区，失败可回滚）、镜像加密。

`bootloader/` 就是设备开机第一段运行的代码，负责加载和验证主程序。

> **【配图建议 2】**：MCUboot 启动流程图——上电 → Bootloader 验签 → 跳转主程序 / 升级分区切换

### `modules/` —— 外部依赖模块

Zephyr 本身只保留核心代码，跟芯片厂商相关的 HAL 库、第三方开源库都拆到这里了。

几个例子：

-   `modules/hal_nordic/` —— Nordic 芯片的 HAL（nRF 系列）
    
-   `modules/hal_st/` —— ST 芯片的 HAL（STM32 系列）
    
-   `modules/hal_espressif/` —— 乐鑫的 HAL（ESP32 系列）
    
-   `modules/hal_nxp/` —— NXP 芯片的 HAL
    
-   `modules/crypto/` —— 加密库（如 mbedTLS）
    
-   `modules/lib/` —— 其他第三方库
    

这些模块都是 west 根据 `zephyr/west.yml` 里的清单自动下载的，不用手动管理。算是"外援仓库"，Zephyr 核心代码需要但自己不带的东西都在这里。

### `tools/` —— 开发工具

辅助开发的工具，比如 `native_simulator/`（本机模拟器），不用真硬件就能在 PC 上跑 Zephyr，方便开发和测试。

### `libgcc_s_fix/` —— 工具链补丁

针对 `libgcc_s` 库的修复。`libgcc_s` 是 GCC 编译器的运行时支持库，某些工具链版本有 bug，这个目录提供补丁文件。相当于"工具链创可贴"。

### `.vscode/` —— 编辑器配置

VS Code 的项目配置文件，包含 IntelliSense 路径、调试配置等。用 VS Code 开发 Zephyr 的话，这个目录能让体验丝滑不少。

---

## 深入核心：`zephyr/` 目录全景

现在进入主角——`zephyr/` 目录。这里有 20 多个文件夹和一堆配置文件，先给一张"地图"：

```
zephyr/
├── arch/        ← CPU 架构相关
├── boards/      ← 开发板配置
├── cmake/       ← 构建系统
├── doc/         ← 文档
├── drivers/     ← 设备驱动
├── dts/         ← 设备树
├── include/     ← 公共头文件
├── kernel/      ← 内核核心
├── lib/         ← 通用库
├── misc/        ← 杂项
├── modules/     ← 模块集成
├── samples/     ← 示例代码
├── scripts/     ← 脚本工具
├── share/       ← 共享文件
├── snippets/    ← 配置片段
├── soc/         ← SoC 芯片相关
├── subsys/      ← 子系统
├── tests/       ← 测试代码
├── CMakeLists.txt   ← 构建入口
├── Kconfig.zephyr   ← 配置入口
└── west.yml         ← 仓库清单
```

接下来按"从下到上"（从硬件到应用）的顺序逛，这样更符合理解逻辑。

---

## 最底层：跟 CPU 和芯片打交道的目录

### `arch/` —— CPU 架构适配层

这个文件夹解决的问题是：不同 CPU 的"脾气"不一样，怎么让 Zephyr 在各种 CPU 上都能跑？

ARM Cortex-M、Intel x86、RISC-V，指令集不一样、中断处理方式不一样、寄存器不一样、内存管理单元（MMU）也不一样。Zephyr 要支持所有这些 CPU，就得为每种 CPU 写一套底层适配代码。这就是 `arch/` 干的事。

打开 `arch/` 目录，会看到这些子目录：

| 子目录 | 对应的 CPU 架构 | 常见芯片 |
| --- | --- | --- |
| `arm/` | 32 位 ARM（Cortex-M0/M3/M4/M7/M33，Cortex-R） | STM32、nRF52、i.MX RT |
| `arm64/` | 64 位 ARM（Cortex-A） | 树莓派、服务器级 ARM |
| `x86/` | Intel/AMD x86 | PC、Atom 处理器 |
| `riscv/` | RISC-V | 国产 RISC-V 芯片、SiFive |
| `xtensa/` | Xtensa（可配置处理器） | ESP32 系列 |
| `arc/` | ARC（Synopsys） | 一些 IoT 芯片 |
| `mips/` | MIPS | 部分网络设备 |
| `nios2/` | Altera Nios II | FPGA 软核 |
| `sparc/` | SPARC | 航空航天领域 |
| `posix/` | POSIX（不是真 CPU，是在 PC 上模拟） | 开发测试用 |
| `common/` | 多架构共享的代码 | — |

每个架构目录里通常包含：

-   上下文切换代码（往往是汇编写的，因为要直接操作寄存器）
    
-   中断和异常处理
    
-   启动代码（CPU 上电后最先跑的代码）
    
-   内存管理（MMU/MPU 配置）
    
-   系统调用（用户态和内核态的切换）
    

打个比方，Zephyr 内核像个"翻译官"，说的"官方语言"是统一的 API（比如 `k_sleep()`、`k_sem_give()`）。但跟不同的 CPU 沟通需要不同的"方言翻译"——`arch/` 就是各种方言的翻译本。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8140c2e84ff5b0ccfb7f192523f48853_MD5.jpg]]

### `soc/` —— 片上系统（SoC）适配层

`arch/` 解决了同一种 CPU 架构的共性问题，但同一个架构下，不同厂家的芯片差异很大。比如：

-   STM32F4 和 nRF52832 都是 ARM Cortex-M4，但时钟系统、Flash 大小、外设布局完全不同
    
-   ESP32-C3 和 ESP32-S3 都是 RISC-V/Xtensa，但管脚和外设有差异
    

这些芯片级别的差异适配，就在 `soc/` 目录里。

按厂家分子目录：

```
soc/
├── nordic/       ← Nordic 的 nRF 系列
│   ├── nrf52/    ← nRF52832、nRF52840 等
│   ├── nrf53/    ← nRF5340（双核）
│   ├── nrf91/    ← nRF9160（带 LTE）
│   └── ...
├── st/           ← ST 的 STM32 系列
├── nxp/          ← NXP 的 i.MX、Kinetis 等
├── espressif/    ← 乐鑫 ESP32
│   ├── esp32/    ← ESP32 原版
│   ├── esp32c3/  ← ESP32-C3（RISC-V）
│   ├── esp32s3/  ← ESP32-S3
│   └── ...
├── renesas/      ← 瑞萨
├── microchip/    ← Microchip
├── ti/           ← 德州仪器
├── gd/           ← 兆易创新（GD32）
├── intel/        ← Intel
└── ...
```

每个芯片目录里通常包含：

-   时钟初始化代码：配置 PLL、设置系统时钟频率
    
-   引脚复用配置：哪些引脚做 GPIO、哪些做 UART
    
-   中断控制器配置：设置中断优先级
    
-   内存映射：Flash 和 RAM 的地址范围
    
-   启动汇编代码：芯片上电后的第一步初始化
    

`arch/` 和 `soc/` 的区别：`arch/` 管 CPU 架构层面的事（ARM 还是 RISC-V），`soc/` 管具体芯片层面的事（STM32F4 还是 nRF52832）。

### `boards/` —— 开发板配置

有了 CPU 架构适配（`arch/`）和芯片适配（`soc/`），还不够。因为同一颗芯片可以装在不同的板子上：

-   同样是 nRF52840，可以做成 Nordic 官方 DK 开发板，也可以做成 USB Dongle
    
-   同样是 STM32F4，可以做成 ST 官方 Nucleo 板，也可以做成各种第三方开发板
    

不同的板子，外设接法不同、LED 和按键的引脚不同、有没有以太网/显示屏也不同。这些"板子级别"的差异配置，就在 `boards/` 目录里。

按厂商分子目录（有 100 多个厂商）：

```
boards/
├── nordic/
│   ├── nrf52840dk/       ← nRF52840 DK 开发板
│   ├── nrf52840dongle/   ← nRF52840 USB Dongle
│   ├── nrf5340dk/        ← nRF5340 DK
│   ├── thingy52/         ← Nordic Thingy:52 IoT 原型平台
│   └── ...
├── st/
│   ├── nucleo_f401re/    ← ST Nucleo F401RE
│   ├── disco_l475_iot1/  ← ST B-L475E-IOT01A IoT Discovery
│   └── ...
├── raspberrypi/
│   └── pico/             ← 树莓派 Pico
├── espressif/
│   └── esp32/            ← ESP32 DevKit
├── arduino/              ← Arduino 各种板
├── beagle/               ← BeagleBone
├── adafruit/             ← Adafruit 的板子
└── ...（100 多个厂商目录）
```

每块板子的目录里通常包含：

-   `board.yml`：板子的基本信息
    
-   `<board>.dts`：板子级设备树，描述这块板子上有什么外设、怎么接的
    
-   `<board>_defconfig`：默认配置（Kconfig 语法）
    
-   `<board>.yaml`：板子的测试配置
    
-   有时还有 `board.c` / `board.h`：板子特定的初始化代码
    

特别说一下 `boards/shields/`。这个目录放的是"扩展板"（Shield）的配置，类似 Arduino 的扩展盾板——在主板上插一个扩展板，上面可能有显示屏、传感器、WiFi 模块。Zephyr 把这些扩展板也做了配置支持，方便即插即用。

---

## 硬件描述层：`dts/` —— 设备树

### 为什么需要设备树？

传统嵌入式开发中，硬件信息（比如 UART0 的基地址是 0x40002000、中断号是 12、接的引脚是 P0.6 和 P0.8）通常写死在 C 代码或头文件里。问题是换一块板子就得改代码，容易出错，代码和硬件信息混在一起也很乱。

Zephyr 借鉴 Linux 的做法，用设备树（Device Tree）来描述硬件。设备树是一种数据结构，把硬件信息从代码里抽离出来，用专门的文件描述。编译时，设备树被编译成 C 头文件，代码里引用这些头文件就能拿到硬件信息。

### `dts/` 目录结构

```
dts/
├── arm/          ← ARM 架构的设备树
├── arm64/        ← ARM64 架构
├── riscv/        ← RISC-V
├── x86/          ← x86
├── xtensa/       ← Xtensa（ESP32）
├── arc/          ← ARC
├── posix/        ← POSIX 模拟
├── sparc/        ← SPARC
├── nios2/        ← Nios II
├── common/       ← 通用设备树片段
└── bindings/     ← 设备树绑定规范（YAML 格式）
```

前几个按架构分的目录，存放各芯片/开发板的设备树源文件（`.dts` 和 `.dtsi` 文件），跟 `boards/` 和 `soc/` 里的设备树配合使用。

### 重点说下 `dts/bindings/`

`bindings/` 是设备树绑定规范，用 YAML 格式写。它定义了：设备树里的每个节点应该有哪些属性、每个属性是什么类型、什么含义。

举个例子，一个 GPIO 控制器的 binding 可能这样写：

```
compatible: "vendor,gpio"
properties:
reg:
    required:true
    type:array
    description:寄存器基地址和大小
interrupts:
    required:true
    type:array
ngpios:
    type:int
    default:32
```

这样，当设备树里写了一个 `compatible = "vendor,gpio"` 的节点，编译器就知道去检查它有没有 `reg` 和 `interrupts` 属性，类型对不对。

`dts/` 是"硬件的说明书"，`bindings/` 是"说明书的语法规范"。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/00bce51ff8b94004e74f8ef32bfe7058_MD5.jpg]]

---

## 驱动层：`drivers/` —— 设备驱动大全

### 这是 Zephyr 里"最热闹"的目录

`drivers/` 是整个 Zephyr 中子目录最多的地方，因为嵌入式世界里的硬件外设种类实在太多。Zephyr 把所有驱动按外设类型分到不同的子目录里。

挑常见的介绍一下：

### 常见驱动目录一览

| 目录 | 驱动的硬件 | 通俗解释 |
| --- | --- | --- |
| `gpio/` | GPIO 控制器 | 控制芯片引脚的高低电平，比如点亮 LED、读按键 |
| `serial/` | 串口（UART） | 串口通信，最常见的调试输出方式 |
| `spi/` | SPI 总线 | SPI 通信协议，连 Flash、传感器、显示屏 |
| `i2c/` | I2C 总线 | I2C 通信协议，连传感器、EEPROM |
| `i2s/` | I2S 总线 | 数字音频传输，连麦克风、音频芯片 |
| `i3c/` | I3C 总线 | I2C 的升级版，更快更强 |
| `adc/` | ADC（模数转换） | 把模拟电压读成数字值，比如读电池电压 |
| `dac/` | DAC（数模转换） | 把数字值转成模拟电压输出 |
| `pwm/` | PWM 控制器 | 输出脉宽调制波形，控制电机、LED 亮度 |
| `flash/` | Flash 存储 | 读写芯片内/外 Flash，存储固件和数据 |
| `ethernet/` | 以太网控制器 | 有线网络通信 |
| `wifi/` | WiFi 模块 | 无线网络 |
| `bluetooth/` | 蓝牙控制器 | 蓝牙物理层（HCI 层） |
| `usb/` | USB 控制器 | USB 设备/主机/OTG |
| `can/` | CAN 总线控制器 | 车载 CAN 总线通信 |
| `sensor/` | 各类传感器 | 温度、加速度、陀螺仪、压力、光照…… |
| `display/` | 显示屏 | LCD、OLED 驱动 |
| `dma/` | DMA 控制器 | 直接内存访问，减轻 CPU 搬运数据的负担 |
| `clock_control/` | 时钟控制器 | 配置芯片内部时钟 |
| `interrupt_controller/` | 中断控制器 | 管理所有中断源 |
| `timer/` | 硬件定时器 | 定时和计时 |
| `watchdog/` | 看门狗 | 系统挂了自动重启 |
| `rtc/` | 实时时钟 | 掉电也能走时的时钟 |
| `crypto/` | 加密加速器 | 硬件 AES、SHA 等加密加速 |
| `entropy/` | 随机数发生器 | 硬件随机数源 |
| `audio/` | 音频设备 | 音频编解码器 |
| `video/` | 视频设备 | 摄像头 |
| `led/` | LED 控制器 | LED 驱动（不只是 GPIO 那种简单控制） |
| `led_strip/` | LED 灯带 | WS2812 之类可编程灯带 |
| `mfd/` | 多功能设备 | 一个芯片集成多种功能的 |
| `regulator/` | 稳压器 | 电压调节 |
| `charger/` | 充电器芯片 | 电池充电管理 |
| `fuel_gauge/` | 电量计 | 测量电池电量 |
| `sdhc/` | SD 卡控制器 | 读写 SD 卡 |
| `disk/` | 磁盘 | 磁盘设备 |
| `eeprom/` | EEPROM | 电可擦除可编程只读存储器 |
| `gnss/` | GNSS（GPS） | 卫星定位 |
| `modem/` | 调制解调器 | 蜂窝网络模块 |
| `ieee802154/` | IEEE 802.15.4 | 低速无线通信（Thread、6LoWPAN 用） |
| `lora/` | LoRa | 远距离低功耗无线通信 |
| `ps2/` | PS/2 接口 | 老式键盘鼠标接口 |
| `kscan/` | 按键扫描 | 矩阵键盘 |
| `input/` | 输入设备 | 触摸屏、按键等输入 |
| `peci/` | PECI | Intel 平台环境控制接口 |
| `ptp_clock/` | PTP 时钟 | 精确时间协议 |
| `reset/` | 复位控制器 | 硬件复位 |
| `hwinfo/` | 硬件信息 | 读取芯片 ID、版本等 |
| `hwspinlock/` | 硬件自旋锁 | 多核间同步 |
| `ipm/` | 核间通信 | 多核处理器核间消息 |
| `mailbox/` | 邮箱 | 核间通信 |
| `mbox/` | Mailbox | 核间通信 |
| `mdio/` | MDIO 总线 | 以太网 PHY 管理 |
| `memc/` | 内存控制器 | 外部 SDRAM 等 |
| `mipi_dbi/` | MIPI DBI | 显示接口 |
| `mipi_dsi/` | MIPI DSI | 显示接口 |
| `mspi/` | MSPI | 串行外设 |
| `pcie/` | PCIe | 高速外设接口 |
| `pm_cpu_ops/` | CPU 电源管理 | 低功耗模式 |
| `power_domain/` | 电源域 | 电源域管理 |
| `retained_mem/` | 保留内存 | 复位后不丢失的内存 |
| `smbus/` | SMBus | 系统管理总线 |
| `syscon/` | 系统控制器 | 杂项系统控制 |
| `tee/` | TEE | 可信执行环境 |
| `usb_c/` | USB-C | USB-C 接口管理 |
| `w1/` | 1-Wire | 单总线协议 |
| `xen/` | Xen | Xen 虚拟化 |
| `virtualization/` | 虚拟化 | 虚拟化支持 |
| `edac/` | EDAC | 错误检测和纠正 |
| `cache/` | 缓存 | CPU 缓存管理 |
| `mm/` | 内存管理 | 内存映射 |
| `bbram/` | 电池供电 RAM | 电池备份 RAM |
| `counter/` | 计数器 | 计数器 |
| `console/` | 控制台 | 控制台输出 |
| `coredump/` | 核心转储 | 系统崩溃时导出内存 |
| `dp/` | DisplayPort | 显示接口 |
| `espi/` | eSPI | 增强型串行外设接口 |
| `fpga/` | FPGA | FPGA 管理 |
| `auxdisplay/` | 辅助显示 | 字符 LCD 等 |
| `dai/` | 数字音频接口 | 音频 |
| `sip_svc/` | SiP 服务 | 系统级服务 |
| `misc/` | 杂项 | 不好分类的 |

### 每个驱动目录里有什么？

以 `drivers/gpio/` 为例，会看到一堆 `.c` 文件，每个对应一款芯片的 GPIO 驱动：

```
drivers/gpio/
├── gpio_dw.c           ← DesignWare GPIO
├── gpio_esp32.c        ← ESP32 GPIO
├── gpio_gecko.c        ← Silicon Labs Gecko GPIO
├── gpio_gd32.c         ← GD32 GPIO
├── gpio_imx.c          ← NXP i.MX GPIO
├── gpio_intel.c        ← Intel GPIO
├── gpio_nrfx.c         ← Nordic nRF GPIO
├── gpio_stm32.c        ← STM32 GPIO
├── gpio_handlers.c     ← 系统调用处理
├── gpio_hogs.c         ← GPIO 预配置
├── CMakeLists.txt      ← 构建规则
└── Kconfig             ← 配置选项
```

Zephyr 的驱动遵循统一的 API（比如 GPIO 操作都用 `gpio_pin_set()`、`gpio_pin_get()`），不管底层是 STM32 还是 nRF，上层调用的接口是一样的。这就是驱动模型抽象的好处。

### 统一 API 怎么映射到不同芯片？

这个问题是理解 Zephyr 驱动模型的关键。很多人看了 `gpio_pin_set()` 这个函数，会好奇：STM32 和 nRF 的寄存器完全不一样，同一个函数名怎么能操作两块不同的芯片？

答案是一套叫"驱动 API 表"的机制。我一步步拆开讲。

#### 第一步：定义"虚函数表"

在 `include/zephyr/drivers/gpio.h` 第 792 行，Zephyr 定义了一个结构体 `gpio_driver_api`：

```
__subsystem struct gpio_driver_api {
    int (*pin_configure)(const struct device *port, gpio_pin_t pin,
                         gpio_flags_t flags);
    int (*port_get_raw)(const struct device *port,
                        gpio_port_value_t *value);
    int (*port_set_masked_raw)(const struct device *port,
                               gpio_port_pins_t mask,
                               gpio_port_value_t value);
    int (*port_set_bits_raw)(const struct device *port,
                             gpio_port_pins_t pins);
    int (*port_clear_bits_raw)(const struct device *port,
                               gpio_port_pins_t pins);
    int (*port_toggle_bits)(const struct device *port,
                            gpio_port_pins_t pins);
    int (*pin_interrupt_configure)(const struct device *port,
                                   gpio_pin_t pin,
                                   enum gpio_int_mode, enum gpio_int_trig);
    int (*manage_callback)(const struct device *port,
                           struct gpio_callback *cb, boolset);
    uint32_t (*get_pending_int)(const struct device *dev);
    /* ... 还有一些可选的 ... */
};
```

这个结构体里全是函数指针，没有一个实现。本质上它是一份"合同"：你想当 GPIO 驱动，就得提供这些函数，签名得跟我一样。

如果用过 C++ 或 Java，这其实就是面向对象里的"接口"（interface）或"纯虚基类"。C 语言没有 class 和 virtual，Zephyr 用结构体里放函数指针来模拟。

#### 第二步：每个芯片驱动填这张表

STM32 的驱动在 `drivers/gpio/gpio_stm32.c` 第 695 行这样填表：

```
static const struct gpio_driver_api gpio_stm32_driver = {
    .pin_configure          = gpio_stm32_config,
    .port_get_raw           = gpio_stm32_port_get_raw,
    .port_set_masked_raw    = gpio_stm32_port_set_masked_raw,
    .port_set_bits_raw      = gpio_stm32_port_set_bits_raw,
    .port_clear_bits_raw    = gpio_stm32_port_clear_bits_raw,
    .port_toggle_bits       = gpio_stm32_port_toggle_bits,
    .pin_interrupt_configure = gpio_stm32_pin_interrupt_configure,
    .manage_callback        = gpio_stm32_manage_callback,
};
```

Nordic nRF 的驱动在 `drivers/gpio/gpio_nrfx.c` 第 418 行这样填表：

```
static const struct gpio_driver_api gpio_nrfx_drv_api_funcs = {
    .pin_configure          = gpio_nrfx_pin_configure,
    .port_get_raw           = gpio_nrfx_port_get_raw,
    .port_set_masked_raw    = gpio_nrfx_port_set_masked_raw,
    .port_set_bits_raw      = gpio_nrfx_port_set_bits_raw,
    .port_clear_bits_raw    = gpio_nrfx_port_clear_bits_raw,
    .port_toggle_bits       = gpio_nrfx_port_toggle_bits,
    .pin_interrupt_configure = gpio_nrfx_pin_interrupt_configure,
    .manage_callback        = gpio_nrfx_manage_callback,
};
```

结构体的字段名一模一样，但赋值的函数名不一样。`gpio_stm32_port_set_bits_raw` 和 `gpio_nrfx_port_set_bits_raw` 是两个完全不同的函数，各自去操作自家芯片的寄存器。

看一眼 STM32 的实现（`gpio_stm32.c` 第 443 行）：

```
static int gpio_stm32_port_set_bits_raw(const struct device *dev,
                                        gpio_port_pins_t pins)
{
    const struct gpio_stm32_config *cfg = dev->config;
    GPIO_TypeDef *gpio = (GPIO_TypeDef *)cfg->base;

    WRITE_REG(gpio->BSRR, pins);   // 直接写 STM32 的 BSRR 寄存器
    return 0;
}
```

Nordic 的实现（`gpio_nrfx.c`，查 `gpio_nrfx_port_set_bits_raw`）则是另一套寄存器操作。两边都在干同一件事——把指定引脚拉高——但走的是各自芯片的路径。

#### 第三步：把这张表挂到设备实例上

光有表还不够，得让上层能找到它。

Zephyr 里每个硬件外设都是一个 `struct device` 实例（定义在 `include/zephyr/device.h` 第 403 行）：

```
struct device {
    const char *name;
    const void *config;     // 配置数据（基地址、时钟等）
    const void *api;        // ← 这就是 API 表的指针
    struct device_state *state;
    void *data;             // 私有数据
    /* ... */
};
```

关键是那个 `api` 字段。STM32 驱动在 `gpio_stm32.c` 第 781 行用 `DEVICE_DT_DEFINE` 宏把设备实例创建出来：

```
DEVICE_DT_DEFINE(__node,
                 gpio_stm32_init,       // 初始化函数
                 PM_DEVICE_DT_GET(__node),
                 &gpio_stm32_data_##__suffix,
                 &gpio_stm32_cfg_##__suffix,
                 PRE_KERNEL_1,
                 CONFIG_GPIO_INIT_PRIORITY,
                 &gpio_stm32_driver)    // ← 把 API 表传进去
```

最后一个参数 `&gpio_stm32_driver` 就是前面填的那张表。宏展开后，这个指针被赋给了 `struct device` 的 `api` 字段。

Nordic 驱动里也有类似的写法（`gpio_nrfx.c` 第 468 行的 `DEVICE_DT_INST_DEFINE`），传的是 `&gpio_nrfx_drv_api_funcs`。

到这里，每个 GPIO 端口设备实例都带着一张属于自己的 API 表。STM32 的 GPIOA 设备挂的是 STM32 的表，nRF 的 P0 设备挂的是 nRF 的表。

#### 第四步：上层 API 通过 `port->api` 找到对应的实现

现在看 `gpio_pin_set()` 怎么工作的。它在 `include/zephyr/drivers/gpio.h` 第 1616 行：

```
static inline int gpio_pin_set(const struct device *port, gpio_pin_t pin,
                               int value)
{
    /* ...省略 ACTIVE_LOW 逻辑判断... */
    return gpio_pin_set_raw(port, pin, value);
}
```

它调了 `gpio_pin_set_raw`，在 1576 行：

```
static inline int gpio_pin_set_raw(const struct device *port, gpio_pin_t pin,
                                   int value)
{
    int ret;

    if (value != 0) {
        ret = gpio_port_set_bits_raw(port, (gpio_port_pins_t)BIT(pin));
    } else {
        ret = gpio_port_clear_bits_raw(port, (gpio_port_pins_t)BIT(pin));
    }

    return ret;
}
```

继续往下追。`gpio_port_set_bits_raw` 在 1348 行（这是一个 `__syscall`，带用户态时走系统调用，不带时走 `z_impl_` 版本）：

```
static inline int z_impl_gpio_port_set_bits_raw(const struct device *port,
                                                gpio_port_pins_t pins)
{
    const struct gpio_driver_api *api =
        (const struct gpio_driver_api *)port->api;   // ← 取出 API 表

    return api->port_set_bits_raw(port, pins);       // ← 调表里的函数
}
```

关键就两行：

1.  把 `port->api` 取出来，强转成 `gpio_driver_api *`
    
2.  调 `api->port_set_bits_raw`
    

由于 `port` 是运行时传入的参数，你传 STM32 的设备指针，`port->api` 就是 STM32 填的那张表，`api->port_set_bits_raw` 实际指向 `gpio_stm32_port_set_bits_raw`；你传 nRF 的设备指针，就指向 `gpio_nrfx_port_set_bits_raw`。

同一个 `gpio_pin_set()` 调用，运行时根据传入的 `port` 不同，自动找到对应芯片的实现。这就是 C 语言版的"多态"。

#### 第五步：设备实例哪来的？

你可能还会问：代码里 `gpio_pin_set(port, pin, value)` 的那个 `port` 是从哪儿来的？

最常见的方式是从设备树取。在 `gpio.h` 里有个 `gpio_dt_spec` 结构体，用宏 `GPIO_DT_SPEC_GET` 从设备树节点初始化：

```
const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_NODELABEL(led0), gpios);
```

`DT_NODELABEL(led0)` 找到设备树里 `led0` 这个节点，`gpios` 属性里写了"这个 LED 接在哪个 GPIO 控制器的哪个引脚上"。编译时 Zephyr 把它展开成对应的 `struct device *` 指针和引脚号。运行时 `led.port` 就是那个 GPIO 控制器的设备实例，`led.pin` 就是引脚号。

然后你就可以：

```
gpio_pin_set(led.port, led.pin, 1);
```

或者用更简洁的 DT 版本：

```
gpio_pin_set_dt(&led, 1);
```

#### 整条调用链一览

把整条链串起来看：

```
应用代码
  gpio_pin_set_dt(&led, 1)             ← 你写的
    ↓
  gpio_pin_set(led.port, led.pin, 1)   ← inline 函数，gpio.h
    ↓
  gpio_pin_set_raw(port, pin, 1)       ← inline 函数，gpio.h
    ↓
  gpio_port_set_bits_raw(port, BIT(pin)) ← __syscall，gpio.h
    ↓
  z_impl_gpio_port_set_bits_raw(port, pins) ← inline 实现，gpio.h
    ↓
  api = port->api                       ← 取出那张 API 表
  api->port_set_bits_raw(port, pins)    ← 函数指针调用
    ↓
  根据传入的 port 不同，走到不同的实现：
    ├─ STM32: gpio_stm32_port_set_bits_raw() → WRITE_REG(gpio->BSRR, pins)
    └─ nRF:   gpio_nrfx_port_set_bits_raw()  → nrf_gpio_port_out_set()
```

#### 这套设计为什么这么搞

好处很明显：

1.  应用代码跟硬件解耦。你写 `gpio_pin_set()` 不用关心是 STM32 还是 nRF，换芯片不用改应用代码。
    
2.  加新芯片只改一个文件。比如要支持某个新厂家的 GPIO，只要在 `drivers/gpio/` 下加一个 `gpio_xxx.c`，实现那几个函数，填一张 `gpio_driver_api` 表就行。上层的 `gpio_pin_set()` 等代码一行都不用动。
    
3.  多个设备实例可以共用同一套实现。STM32 有 GPIOA、GPIOB、GPIOC……它们用同一份 `gpio_stm32_driver` 表，区别只在 `config` 里存的基地址不同。
    

这套"接口 + 函数指针表"的模式在 Zephyr 里到处都是，不只是 GPIO。UART、SPI、I2C、ADC……每种外设都有一张自己的 `xxx_driver_api` 表。理解了 GPIO 这一套，其他外设的驱动模型是同样的套路。

---

## 内核核心：`kernel/` —— 操作系统的"发动机"

### 这才是 RTOS 的灵魂

`kernel/` 目录是 Zephyr 最核心的代码，定义了操作系统的所有基本功能。看看里面都有啥：

| 文件 | 功能 | 通俗解释 |
| --- | --- | --- |
| `sched.c` | 调度器 | 决定哪个线程先跑、跑多久 |
| `thread.c` | 线程管理 | 创建、删除、挂起、恢复线程 |
| `init.c` | 系统初始化 | 系统启动时的初始化流程 |
| `mutex.c` | 互斥锁 | 保护共享资源，防止多线程冲突 |
| `sem.c` | 信号量 | 线程间同步和资源计数 |
| `queue.c` | 消息队列 | 线程间传递消息 |
| `msg_q.c` | 消息队列 | 消息队列（另一种实现） |
| `mailbox.c` | 邮箱 | 线程间传递消息（带类型） |
| `pipe.c` | 管道 | 线程间传递数据流 |
| `mempool.c` | 内存池 | 分配和释放固定大小内存块 |
| `kheap.c` | 堆内存 | 动态内存分配（类似 malloc） |
| `mem_slab.c` | 内存块分配 | 分配固定大小内存块 |
| `mem_domain.c` | 内存域 | 内存隔离（用户空间） |
| `mmu.c` | MMU 管理 | 内存保护单元 |
| `timer.c` | 软件定时器 | 定时执行任务 |
| `timeout.c` | 超时管理 | 各种超时机制的基础 |
| `timeslicing.c` | 时间片调度 | 多线程轮流执行 |
| `idle.c` | 空闲线程 | 没事干时跑的线程，可进入低功耗 |
| `work.c` | 工作队列 | 中断里不方便做的事，丢到工作队列里做 |
| `poll.c` | 轮询 | 等待多个事件 |
| `events.c` | 事件 | 事件机制 |
| `condvar.c` | 条件变量 | 条件等待 |
| `futex.c` | Futex | 快速用户态互斥 |
| `smp.c` | 多核支持 | 多核处理器支持 |
| `spinlock_validate.c` | 自旋锁验证 | 自旋锁调试 |
| `userspace.c` | 用户空间 | 用户态和内核态隔离 |
| `userspace_handler.c` | 用户空间处理 | 用户态系统调用处理 |
| `fatal.c` | 错误处理 | 系统崩溃时的处理 |
| `device.c` | 设备管理 | 设备实例管理 |
| `errno.c` | 错误码 | errno 机制 |
| `version.c` | 版本信息 | 系统版本号 |
| `banner.c` | 启动横幅 | 开机打印的 Zephyr 横幅 |
| `stack.c` | 栈管理 | 栈信息 |
| `cpu_mask.c` | CPU 掩码 | 多核 CPU 亲和性 |
| `ipi.c` | 核间中断 | 多核通信 |
| `irq_offload.c` | IRQ 卸载 | 在中断上下文执行函数 |
| `paging/` | 内存换页 | 内存分页机制 |
| `include/` | 内核内部头文件 | 内核私有头文件 |

### 几个核心概念解释

**调度器（scheduler）**：Zephyr 是个实时操作系统，"实时"的关键就是调度器。它要保证高优先级的线程能立刻抢占低优先级的线程。支持抢占式调度（高优先级线程可以打断低优先级线程）、时间片轮转（同优先级的线程轮流跑）、协作式调度（线程主动让出 CPU）。

**线程（thread）**：Zephyr 的基本执行单元。每个线程有自己的栈、优先级和入口函数。用 `k_thread_create()` 创建线程。

**同步原语**：多线程环境下，线程之间需要协调。Zephyr 提供了信号量（计数器，表示"有多少资源可用"）、互斥锁（同一时间只允许一个线程访问某资源）、事件（通知"某件事发生了"）。

**工作队列（workqueue）**：中断处理函数里不能做太耗时的操作（因为中断会打断正常流程）。如果在中断里需要做复杂处理，就把工作"丢"到工作队列里，由一个专门的内核线程来执行。

`kernel/` 就是 Zephyr 的操作系统内核，管着线程、调度、同步、内存这些最核心的事。

---

## 公共头文件：`include/` —— API 的"门面"

### 存放所有的公共 API 声明

写 Zephyr 应用时，`#include <zephyr/kernel.h>`、`#include <zephyr/drivers/gpio.h>` 这些引用的文件就在这个目录里。

```
include/zephyr/
├── kernel.h           ← 内核 API（线程、信号量、互斥锁等）
├── device.h           ← 设备模型
├── devicetree.h       ← 设备树宏
├── drivers/           ← 驱动 API 头文件
├── bluetooth/         ← 蓝牙 API
├── net/               ← 网络 API
├── fs/                ← 文件系统 API
├── logging/           ← 日志 API
├── shell/             ← Shell API
├── sys/               ← 系统工具
├── toolchain.h        ← 工具链相关
├── types.h            ← 基本类型定义
├── irq.h              ← 中断 API
├── fatal.h            ← 错误处理
├── init.h             ← 初始化级别定义
├── linker/            ← 链接脚本相关
├── posix/             ← POSIX 兼容 API
├── random/            ← 随机数 API
├── debug/             ← 调试 API
├── storage/           ← 存储 API
├── pm/                ← 电源管理 API
├── ...（还有很多）
```

### 为什么单独拿出来说？

因为这个目录是写代码时最常打交道的。想用什么功能，就 `#include` 对应的头文件。

比如操作 GPIO：

```
#include <zephyr/drivers/gpio.h>
```

用蓝牙：

```
#include <zephyr/bluetooth/bluetooth.h>
```

`include/` 是 Zephyr 的"API 菜单"，写应用时来这里点菜。

---

## 通用库：`lib/` —— 工具箱

### 不属于内核，也不属于驱动的通用代码

`lib/` 目录放着各种工具性质的库，供内核和应用使用。跟 `kernel/` 不同，这些库不是操作系统的核心功能，而是辅助性的。

```
lib/
├── libc/          ← C 标准库的精简实现
├── posix/         ← POSIX 兼容层
├── crc/           ← CRC 校验
├── hash/          ← 哈希算法
├── heap/          ← 堆内存管理
├── mem_blocks/    ← 内存块管理
├── os/            ← OS 抽象层
├── utils/         ← 通用工具函数
├── cpp/           ← C++ 支持
├── acpi/          ← ACPI（x86 平台）
├── runtime/       ← 运行时支持
├── open-amp/      ← OpenAMP（多核通信框架）
└── smf/           ← 状态机框架
```

### 几个重要的库

\*\*`libc/`\*\*：Zephyr 自带了一个极简的 C 标准库实现（叫 minimal libc），提供 `memcpy`、`strlen`、`printf` 这些基本函数。需要更完整的功能，也可以配置成用 newlibc（GNU 的 libc）或 picolibc。

\*\*`posix/`\*\*：POSIX 兼容层，提供 `pthread`、`mqueue`、`semaphore` 这些 POSIX 标准接口。好处是原本在 Linux 上写的程序，移植到 Zephyr 上更容易了，因为 API 是兼容的。

\*\*`crc/`\*\*：各种 CRC 校验算法。嵌入式里 CRC 用得很多，比如通信校验、Flash 校验。

\*\*`hash/`\*\*：哈希算法，用来做快速查找。

\*\*`heap/`\*\*：堆内存管理。Zephyr 支持多种堆实现，包括一个优化过的 `sys_heap`。

\*\*`cpp/`\*\*：C++ 语言支持。Zephyr 不光能用 C，还能用 C++ 写应用。

\*\*`smf/`\*\*：状态机框架（State Machine Framework）。很多嵌入式系统都是状态机驱动的，Zephyr 提供了这个框架方便写状态机。

\*\*`open-amp/`\*\*：OpenAMP 是一个多核通信框架。比如 nRF5340 是双核芯片，两个核之间怎么通信？就是用 OpenAMP。

---

## 子系统：`subsys/` —— 高级功能模块

### 这是 Zephyr 最"丰富"的目录

`subsys/` 是 Zephyr 的功能子系统集合。它建立在 kernel 和 drivers 之上，提供较高级的功能。如果说 `kernel/` 是发动机、`drivers/` 是轮子，那 `subsys/` 就是车上的空调、导航、音响。

```
subsys/
├── bluetooth/     ← 蓝牙协议栈
├── net/           ← 网络协议栈
├── fs/            ← 文件系统
├── shell/         ← 命令行 Shell
├── logging/       ← 日志系统
├── usb/           ← USB 子系统
├── debug/         ← 调试子系统
├── ipc/           ← 进程间通信
├── dfu/           ← 固件升级
├── pm/            ← 电源管理
├── canbus/        ← CAN 总线
├── mgmt/          ← 设备管理
├── ...（还有很多）
```

### 重点子系统介绍

#### `bluetooth/` —— 蓝牙协议栈

Zephyr 内置了一个完整的蓝牙协议栈，通过了蓝牙 SIG 认证。支持 BLE（低功耗蓝牙，各种 Profile 如心率、血压、位置）、蓝牙 mesh（网状网络）、蓝牙 5.0+ 特性（长距离、高速度、广播扩展）、多种角色（Central、Peripheral、Broadcaster、Observer）。

这是 Zephyr 最强力的功能之一，很多 IoT 设备选择 Zephyr 就是因为它的蓝牙支持好。

#### `net/` —— 网络协议栈

完整的 TCP/IP 网络协议栈。基础协议有 TCP、UDP、IP、ICMP、IPv4、IPv6；应用协议有 HTTP、HTTPS、MQTT、CoAP、SNTP、DNS、Websocket；底层支持 Ethernet、WiFi、IEEE 802.15.4、6LoWPAN、Thread；安全方面有 TLS/DTLS 加密通信。

#### `fs/` —— 文件系统

支持多种文件系统：FAT（最通用，SD 卡常用）、LittleFS（专为 Flash 设计的轻量文件系统）、FFS（Nordic 的 Flash 文件系统）、FUSE（用户态文件系统）。

#### `shell/` —— 命令行 Shell

Zephyr 内置了一个功能强大的串口 Shell，可以通过串口终端输入命令来操作设备。支持命令自动补全、历史记录、管道、多种后端（串口、RTT、Telnet）。

#### `logging/` —— 日志系统

提供分级日志（ERROR、WARNING、INFO、DEBUG），支持多种后端（串口、RTT、网络）、日志过滤、运行时控制、十六进制数据打印。

#### `usb/` —— USB 子系统

USB 设备栈，支持 USB Device 模式（让设备被 PC 识别为 U 盘、串口、HID 等）、USB Host 模式（让设备能接 U 盘、键盘等）、USB OTG、USB-C。

#### `pm/` —— 电源管理

嵌入式设备通常很在意功耗。Zephyr 的电源管理支持多种低功耗状态（睡眠、深度睡眠、关机）、自动进入低功耗、唤醒源配置、每个设备的电源管理。

#### `dfu/` —— 固件升级

支持 OTA（Over-The-Air）升级，包括 MCUmgr 协议、镜像管理、A/B 分区切换。

#### `mgmt/` —— 设备管理

MCUmgr 是 Zephyr 的远程设备管理协议，支持远程文件传输、远程 Shell、固件升级、设备信息查询。

#### `ipc/` —— 进程间通信

进程间/核间通信，支持多核处理器核间消息传递。

#### `canbus/` —— CAN 总线

CAN 总线协议栈，支持 CANopen 和 ISOTP。

#### `debug/` —— 调试

包括 GDB Stub、性能分析、内存分析。

#### 其他子系统

| 子系统 | 功能 |
| --- | --- |
| `audio/` | 音频处理 |
| `display/` | 显示子系统 |
| `input/` | 输入子系统 |
| `fb/` | 帧缓冲 |
| `emul/` | 设备模拟（测试用） |
| `dsp/` | 数字信号处理 |
| `rtio/` | 实时 I/O 操作 |
| `zbus/` | 消息总线 |
| `lorawan/` | LoRaWAN 协议 |
| `modbus/` | Modbus 协议 |
| `jwt/` | JSON Web Token |
| `stats/` | 统计信息 |
| `settings/` | 设置持久化 |
| `tracing/` | 跟踪 |
| `timing/` | 时间测量 |
| `task_wdt/` | 任务级看门狗 |
| `sd/` | SD 卡 |
| `portability/` | 可移植性 |
| `random/` | 随机数 |
| `retention/` | 数据保持 |
| `sensing/` | 传感器框架 |
| `bindesc/` | 二进制描述 |
| `demand_paging/` | 按需分页 |
| `dap/` | Debug Access Port |
| `testsuite/` | 测试套件 |
| `llext/` | 动态加载扩展 |
| `sip_svc/` | SiP 服务 |

---

## 构建系统：`cmake/` + `scripts/`

### `cmake/` —— CMake 构建模块

Zephyr 用 CMake 作为构建系统。`cmake/` 目录存放构建过程中用到的各种 CMake 脚本和模块。

```
cmake/
├── compiler/       ← 编译器配置（GCC、Clang、IAR 等）
├── toolchain/      ← 工具链配置
├── linker/         ← 链接脚本
├── linker_script/  ← 链接脚本模板
├── flash/          ← 烧录配置
├── bintools/       ← 二进制工具（objcopy、objdump 等）
├── modules/        ← 外部模块集成
├── ide/            ← IDE 项目生成
├── sca/            ← 静态代码分析
├── emu/            ← 模拟器
├── app/            ← 应用构建
├── kobj.cmake      ← 内核对象配置
├── gen_version_h.cmake ← 版本头文件生成
└── ...
```

几个重要的子目录：

-   `compiler/`：Zephyr 支持多种编译器（GCC、Clang、IAR、ARM Compiler 等），每种编译器的选项和特性不同，这里做适配
    
-   `toolchain/`：工具链配置，Zephyr 有自己的 SDK（Zephyr SDK），也支持用厂家提供的工具链
    
-   `linker/`：链接脚本，定义程序在 Flash 和 RAM 中的布局
    
-   `flash/`：烧录配置，支持各种烧录器（J-Link、ST-Link、OpenOCD 等）
    

### `scripts/` —— Python 脚本工具集

Zephyr 的构建和开发流程中大量使用 Python 脚本。`scripts/` 就是这些脚本的"大本营"。

```
scripts/
├── twister/        ← 自动化测试框架（Zephyr 的"pytest"）
├── kconfig/        ← Kconfig 配置系统
├── dts/            ← 设备树编译
├── west_commands/  ← west 的扩展命令
├── tracing/        ← 跟踪工具
├── ci/             ← CI/CD 脚本
├── release/        ← 发布脚本
├── checkpatch.pl   ← 代码风格检查
├── pylib/          ← Python 库
├── coccinelle/     ← 代码模式检查
├── tests/          ← 脚本自身的测试
├── utils/          ← 工具脚本
├── build/          ← 构建辅助
├── logging/        ← 日志相关脚本
├── net/            ← 网络工具
├── coredump/       ← 核心转储解析
├── footprint/      ← 代码大小分析
├── native_simulator/ ← 本机模拟器
├── generate_usb_vif/ ← USB VIF 生成
├── list_boards.py  ← 列出所有支持的板子
├── list_hardware.py ← 列出所有支持的硬件
├── list_shields.py ← 列出所有扩展板
├── get_maintainer.py ← 找出某文件的责任人
└── ...
```

重点说说 `twister/`。Twister 是 Zephyr 的自动化测试框架，名字来源于"龙卷风"（tornado 的兄弟）。它能在真实硬件上跑测试、在 QEMU 模拟器上跑测试、在本机（native\_posix）上跑测试、并行跑多个测试、生成测试报告。

Zephyr 每次提交代码，CI 都会用 twister 跑几千个测试用例，保证代码质量。

---

## Kconfig 配置系统

### 什么是 Kconfig？

Zephyr 支持的硬件和功能太多了，一个具体的固件不可能全用上。比如做一个简单的温度计，不需要蓝牙、不需要 WiFi、不需要文件系统。怎么"裁剪"掉不需要的部分？

答案就是 Kconfig。它是一套配置系统，让你通过勾选的方式选择需要哪些功能。用过 Linux 内核的 `make menuconfig` 的话，Zephyr 的 `west build -t menuconfig` 是一模一样的体验。

### Kconfig 文件分布

几乎每个目录里都有 `Kconfig` 文件：

-   `Kconfig.zephyr` —— 顶层 Kconfig，所有配置的入口
    
-   `kernel/Kconfig` —— 内核相关配置
    
-   `drivers/Kconfig` —— 驱动相关配置
    
-   `subsys/Kconfig` —— 子系统相关配置
    
-   各子目录里还有更细分的 `Kconfig.*` 文件
    

### Kconfig 怎么用？

构建时打开配置菜单：

```
west build -b <board> -t menuconfig
```

会出来一个蓝色的文字菜单，用方向键浏览，空格键勾选，`?` 查看帮助。选完保存退出，配置就生效了。

配置会被保存到一个 `.config` 文件里，也可以保存为 `<board>.conf` 文件（通常放在项目的 `prj.conf` 里）。

Kconfig 相当于 Zephyr 的"功能开关面板"，想用什么功能就开什么，不用就关，让固件大小可控。

---

## 学习利器：`samples/` —— 示例代码

### 学 Zephyr 最好的方式

光看文档和源码学得慢。最好的学习方式是看示例、跑示例、改示例。Zephyr 官方提供了大量的示例代码，就在 `samples/` 目录里。

```
samples/
├── hello_world/          ← 最简单的 Hello World
├── basic/                ← 基础示例
├── synchronization/      ← 线程同步
├── philosophers/         ← 哲学家就餐（经典并发示例）
├── kernel/               ← 内核功能示例
├── drivers/              ← 驱动使用示例
├── bluetooth/            ← 蓝牙示例（几十个）
├── net/                  ← 网络示例
├── subsys/               ← 子系统示例
├── sensor/               ← 传感器示例
├── boards/               ← 板级特定示例
├── application_development/ ← 应用开发示例
├── arch/                 ← 架构相关示例
├── compression/          ← 压缩
├── cpp/                  ← C++ 示例
├── modules/              ← 模块使用示例
├── posix/                ← POSIX 示例
├── shields/              ← 扩展板示例
├── sysbuild/             ← 系统构建示例
├── tfm_integration/      ← TF-M 安全集成示例
├── userspace/            ← 用户空间示例
└── fuel_gauge/           ← 电量计示例
```

### 从 `hello_world` 开始

每个示例的结构都很标准，以 `hello_world` 为例：

```
samples/hello_world/
├── CMakeLists.txt   ← 构建文件
├── prj.conf         ← 项目配置（Kconfig）
├── README.rst       ← 说明文档
├── sample.yaml      ← 测试配置（给 twister 用）
└── src/
    └── main.c       ← 主程序源码
```

`main.c` 里就是：

```
#include <zephyr/kernel.h>

int main(void)
{
    printk("Hello World! %s\n", CONFIG_BOARD);
    return 0;
}
```

就这么简单。编译和运行：

```
west build -b qemu_x86 samples/hello_world
west build -t run
```

### 几个值得关注的示例

-   `philosophers/`：经典哲学家就餐问题，演示多线程和同步
    
-   `synchronization/`：线程同步的基础用法
    
-   `bluetooth/beacon/`：做一个蓝牙 Beacon
    
-   `bluetooth/central_hr/`：做一个蓝牙中心设备，连接心率传感器
    
-   `net/sockets/`：网络 socket 编程示例
    
-   `subsys/shell/`：Shell 使用示例
    
-   `subsys/logging/`：日志使用示例
    

---

## 测试：`tests/` —— 质量保证

### Zephyr 对测试的重视

Zephyr 是个严肃的项目，有严格的测试要求。`tests/` 目录存放所有测试代码，由 twister 框架自动运行。

```
tests/
├── kernel/          ← 内核测试（调度、线程、同步等）
├── drivers/         ← 驱动测试
├── bluetooth/       ← 蓝牙测试
├── net/             ← 网络测试
├── lib/             ← 库测试
├── subsys/          ← 子系统测试
├── arch/            ← 架构测试
├── benchmarks/      ← 性能基准测试
├── bsim/            ← 仿真测试（babbleSim）
├── unit/            ← 单元测试（不需要硬件）
├── integration/     ← 集成测试
├── posix/           ← POSIX 兼容性测试
├── kconfig/         ← Kconfig 测试
├── cmake/           ← CMake 测试
├── boot/            ← 启动测试
├── misc/            ← 杂项测试
├── modules/         ← 模块测试
├── application_development/ ← 应用开发测试
├── robot/           ← Robot Framework 测试
└── test_config.yaml ← 测试配置
```

### 几个特别的测试目录

`benchmarks/`：性能基准测试，测量 Zephyr 各项操作的性能，比如线程切换耗时、中断响应延迟。

`bsim/`：babbleSim 仿真测试。蓝牙这种东西，在真实硬件上测试很麻烦，babbleSim 可以在 PC 上模拟多个蓝牙设备互相通信，测试协议栈的正确性。

`unit/`：单元测试，不需要真硬件，在 PC 上直接跑，测试单个函数或模块的逻辑。

`robot/`：Robot Framework 测试，用 Python 写的自动化测试，可以测端到端的场景。

---

## 文档：`doc/` —— 官方说明书

### 想查文档？在这里

Zephyr 的官方文档用 Sphinx + reStructuredText 写的，源文件就在 `doc/` 目录。可以在 Zephyr 官网 看到编译后的 HTML 版本。

```
doc/
├── index.rst          ← 文档首页
├── introduction/      ← 介绍
├── develop/           ← 开发指南
├── kernel/            ← 内核文档
├── hardware/          ← 硬件支持
├── services/          ← 子系统服务
├── connectivity/      ← 连接性（蓝牙/网络）
├── security/          ← 安全
├── safety/            ← 功能安全
├── contribute/        ← 贡献指南
├── releases/          ← 版本发布说明
├── project/           ← 项目信息
├── templates/         ← 模板
├── _static/           ← 静态资源
├── _templates/        ← Sphinx 模板
├── _scripts/         ← 文档构建脚本
├── _extensions/       ← Sphinx 扩展
├── _doxygen/          ← Doxygen 配置（API 文档）
├── images/            ← 图片资源
├── conf.py            ← Sphinx 配置
├── build/             ← 文档构建输出
└── ...
```

### 几个实用的文档章节

-   `introduction/`：新手从这里开始
    
-   `develop/`：开发指南，怎么建项目、怎么编译、怎么调试
    
-   `kernel/`：内核原理详解，讲线程、调度、同步
    
-   `hardware/`：硬件支持，哪些芯片和板子被支持
    
-   `services/`：各子系统的用法
    
-   `connectivity/`：蓝牙和网络的用法
    
-   `security/`：安全相关，包括 TLS、安全启动、TF-M
    
-   `safety/`：功能安全，IEC 61508、ISO 26262 这些安全标准
    

---

## 其他目录和文件

### `snippets/` —— 配置片段

Snippets 是可复用的配置片段，包含设备树片段和 Kconfig 片段。构建时引入这些片段，可以快速开启某些功能。

```
snippets/
├── cdc-acm-console/    ← USB CDC ACM 串口控制台
├── nordic-flpr/        ← Nordic FLPR 核心配置
├── nordic-flpr-xip/    ← FLPR 核心从 Flash 运行
├── nordic-ppr/         ← Nordic PPR 核心配置
├── nordic-ppr-xip/     ← PPR 核心从 Flash 运行
├── nus-console/        ← Nordic UART Service 控制台
├── ram-console/        ← RAM 控制台
├── rtt-console/        ← RTT 控制台
└── xen_dom0/           ← Xen Domain 0
```

比如想用 RTT（Real-Time Transfer）做控制台输出（比串口快得多，调试利器），引入 `rtt-console` snippet 就行。

### `share/` —— 共享文件

存放 CMake 包导出文件，让外部项目能通过 `find_package(Zephyr)` 引用 Zephyr。包含：

-   `zephyr-package/` —— Zephyr CMake 包
    
-   `sysbuild/` —— sysbuild（多镜像构建）CMake 包
    
-   `zephyrunittest-package/` —— 单元测试 CMake 包
    

### `misc/` —— 杂项

放一些不好分类的东西，`generated/` 子目录存放构建过程中生成的文件。

### `modules/`（zephyr 目录内的）

跟顶层的 `modules/` 配合，用于把外部模块的 CMake 集成到 Zephyr 构建系统中。

### `submanifests/` —— 子清单

west 的子清单目录，支持额外的仓库管理，比较少用。

### `build/` —— 构建输出

编译时产生的中间文件和最终固件都在这里，不是源码，可以随时删除重建。

### 重要的顶层文件

| 文件 | 作用 |
| --- | --- |
| `CMakeLists.txt` | 顶层 CMake 构建文件，整个构建的入口 |
| `Kconfig`/ `Kconfig.zephyr` | Kconfig 配置系统的顶层文件 |
| `west.yml` | west 的清单文件，定义了要拉取哪些外部仓库 |
| `VERSION` | Zephyr 版本号 |
| `SDK_VERSION` | 推荐的 Zephyr SDK 版本 |
| `version.h.in` | 版本头文件模板 |
| `LICENSE` | Apache 2.0 许可证 |
| `MAINTAINERS.yml` | 维护者列表（每个模块由谁负责） |
| `CODEOWNERS` | GitHub 代码所有者（PR 通知谁 review） |
| `CODE_OF_CONDUCT.md` | 社区行为准则 |
| `CONTRIBUTING.rst` | 贡献指南 |
| `README.rst` | 项目说明 |
| `.clang-format` | C 代码格式化规则 |
| `.editorconfig` | 编辑器配置 |
| `.checkpatch.conf` | 代码风格检查配置 |
| `.gitlint` | Git commit message 格式规则 |
| `.yamllint` | YAML 文件格式检查 |
| `.gitattributes`/ `.gitignore` | Git 配置 |
| `.mailmap` | 邮件地址映射（处理同一人多个邮箱的情况） |
| `zephyr-env.sh`/ `zephyr-env.cmd` | 环境变量设置脚本（Linux/Windows） |
| `.codecov.yml` | 代码覆盖率配置 |
| `.github/` | GitHub Actions CI 配置、Issue 模板 |

---

## 它们之间的关系——一张图看懂

看了这么多，可能有点晕。理一下 Zephyr 各目录之间的依赖关系：![[Inbox/笔记同步助手/微信公众号/2026/07/images/2598f05056033efb7ffc69fb197b7ad2_MD5.jpg]]

横向支撑：

```
cmake/ + scripts/      ← 构建系统（编译、链接、烧录）
Kconfig                ← 配置系统（裁剪功能）
boards/                ← 板子配置（针对具体硬件）
doc/                   ← 文档
tests/                 ← 测试
```

---

## 实战：怎么用这些目录

光知道目录是啥还不够，来点实用的——实际开发时会怎么跟这些目录打交道？

### 开发流程

1.  选板子：去 `boards/` 找你用的开发板，确认 Zephyr 支持它
    
2.  建项目：参考 `samples/` 创建自己的项目，写 `CMakeLists.txt`、`prj.conf` 和 `main.c`
    
3.  查 API：去 `include/` 找需要的头文件，看 API 怎么用
    
4.  配置功能：用 `menuconfig`（基于 `Kconfig`）开启需要的功能
    
5.  编译：`west build -b <board>`（用 `cmake/` 和 `scripts/`）
    
6.  烧录：`west flash`（用 `cmake/flash/` 配置）
    
7.  调试：用 `shell/` 和 `logging/` 子系统输出调试信息
    
8.  测试：参考 `tests/` 写测试，用 `twister` 跑
    

### 常见场景对应目录

| 我想做的事 | 该看哪个目录 |
| --- | --- |
| 我的板子 Zephyr 支持吗 | `boards/` |
| 这个芯片怎么初始化 | `soc/` |
| 这个外设怎么用 | `drivers/`\+ `include/zephyr/drivers/` |
| 内核 API 怎么用 | `include/zephyr/kernel.h`\+ `samples/kernel/` |
| 怎么用蓝牙 | `subsys/bluetooth/`\+ `samples/bluetooth/` |
| 怎么联网 | `subsys/net/`\+ `samples/net/` |
| 怎么写文件 | `subsys/fs/` |
| 怎么加日志 | `subsys/logging/` |
| 怎么加串口命令 | `subsys/shell/` |
| 怎么做低功耗 | `subsys/pm/` |
| 怎么做 OTA 升级 | `subsys/dfu/`\+ `subsys/mgmt/` |
| 怎么定义硬件信息 | `dts/`\+ `boards/` |
| 怎么配置编译选项 | `Kconfig`\+ `prj.conf` |
| 怎么看官方文档 | `doc/` |
| 怎么学别人怎么写 | `samples/` |
| 怎么写测试 | `tests/` |

---

## 给新手的建议

### 学习路径

1.  先跑 hello\_world：`samples/hello_world/`，先让 Zephyr 跑起来
    
2.  看内核文档：`doc/kernel/`，理解线程、调度、同步这些基本概念
    
3.  跑几个示例：`samples/basic/`、`samples/synchronization/`、`samples/philosophers/`
    
4.  试试外设：GPIO、UART、I2C，看 `samples/drivers/`
    
5.  试试子系统：Shell、Logging，看 `samples/subsys/`
    
6.  根据需要深入：需要蓝牙就看 `subsys/bluetooth/`，需要网络就看 `subsys/net/`
    

### 别被目录吓到

Zephyr 代码量很大，目录很多，但你不需要全部了解。大部分时候只需要：

-   知道 API 在 `include/` 里
    
-   知道配置用 `Kconfig`
    
-   知道硬件描述在 `dts/` 和 `boards/`
    
-   知道示例在 `samples/`
    

其他目录，用到的时候再查就行。

### 善用工具

-   `west boards`：列出所有支持的板子
    
-   `west build -t menuconfig`：配置功能
    
-   `west build -t guiconfig`：图形化配置
    
-   `scripts/get_maintainer.py`：找某个文件的责任人
    
-   `scripts/list_boards.py`：列出板子信息
    

---

## 写在最后

Zephyr 的目录结构借鉴了 Linux 内核的很多实践：设备树、Kconfig、CMake 构建、模块化设计。这些设计让 Zephyr 既能做到极简（裁剪到几十 KB），又能做到功能丰富（蓝牙、网络、文件系统全都有）。

理解了目录结构，就掌握了 Zephyr 的"地图"。以后不管想找什么功能，都知道该去哪里看。

如果对某个具体目录还想深入了解，欢迎在评论区留言，可以再展开细说。

---

> 全文完
> 
> 如果觉得有帮助，欢迎点赞、在看、转发。
> 
> 更多嵌入式技术干货，请关注本公众号，下期见。

---

_本文基于 Zephyr 源码目录结构撰写，Zephyr 在持续演进，部分目录可能有调整，请以最新源码为准。_

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/4636f012_1784037823494?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzY5NzM0Mjg4NA%3D%3D%26mid%3D2247483750%26idx%3D1%26sn%3D92635dd3d64c4f1d17dd8643cb7b7757%26chksm%3Df5020b2d09241f302028edf1ab2479b9f7d7edb99160a6dcf7bddba53499a806651076c65e03%26mpshare%3D1%26scene%3D1%26srcid%3D0714YCx42txkl5XbAarteQI0%26sharer_shareinfo%3D5748c75b84b034281a96799324a72154%26sharer_shareinfo_first%3D5748c75b84b034281a96799324a72154%23rd&s=obsidian)