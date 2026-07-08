---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247488000&idx=1&sn=cd97f46c68d6da0c246364377ec2fec9&chksm=c3cedcfdaea8092eedf9829138832cece8c4ca54a4b3beab9c70f7ae5856b84331a14e4c30e9&mpshare=1&scene=1&srcid=07062tkWpkSjbhXej10pQM40&sharer_shareinfo=565ad6ac67424d65d57f81e48ad45309&sharer_shareinfo_first=565ad6ac67424d65d57f81e48ad45309#rd
saved: 2026-07-06 23:02:19
tags:
  - 笔记同步助手
id: 636b58de-d9ee-4d5e-99cf-5f8802372852
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-06-27 22:03

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d5673b61529048fc2392d6b1f5e4601a_MD5.gif]]

大家好，我是蟹老板～

做嵌入式Linux驱动开发这些年来，我被问得最多的问题，不是怎么写**I2C**传感器驱动，也不是怎么调设备树，而是：**Linux的I2C框架为什么要搞这么复杂？整出这么多结构体？**

大伙得明白，Linux I2C 框架真正解决的不是“怎么发一个字节”这么小的问题，而是一个更大的问题。

> 不同 SoC 的 I2C 控制器不一样。

> 不同外设的寄存器协议不一样。

> 不同板子的设备树不一样。

> 用户态还想临时扫一下总线。

> 内核态驱动还想优雅地 **probe/remove**。

这些东西如果全揉在一起，驱动就会变成一坨又一坨平台相关代码。

能跑，但难维护。能交差，但后面的人会骂你。说不定半年后，自己也会骂自己。

这篇文章就我们将重点放在 Linux I2C 框架的分层设计上，看清楚每一层到底负责什么，它们之间怎么串起来，一次 I2C 读写请求又是怎么从外设驱动一路走到硬件控制器的。

## 一、为什么 Linux 需要 I2C 框架

### 1.1 I2C 总线协议的基本特点

**I2C** 的硬件形态很简单，两根线。

一根 **SCL**，负责时钟。

一根 **SDA**，负责数据。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ea766020496632372fe89a09c729b6ce_MD5.jpg]]

通常情况下，一个控制器作为主设备，多个外设挂在同一条总线上。每个外设有自己的地址，主设备通过地址找到对应的设备，再进行读写。

这也是 **I2C** 被大量用在板级低速外设上的原因。线少，成本低，挂设备方便。

比如一块嵌入式板子上，可能是这样的：

```
SoC I2C Controller
        |
        |--- Touch IC       addr = 0x38
        |--- RTC            addr = 0x51
        |--- EEPROM         addr = 0x50
        |--- Temperature IC addr = 0x48
```

从协议角度看，一次常见的 **I2C** 写寄存器大概是这样：

```
START
  Device Address + Write
  Register Address
  Data
STOP
```

一次常见的读寄存器可能是这样：

```
START
  Device Address + Write
  Register Address
RESTART
  Device Address + Read
  Data
STOP
```

看起来是不是很清爽？

问题也恰好出在这里。**I2C** 协议本身很清爽，但 Linux 要面对的不是一个固定芯片，而是一整个生态。

一个传感器可以挂在 NXP 平台上，也可以挂在 Rockchip 平台上，还可以挂在全志、高通、`STM32MP1` 或者别的 SoC 上。外设驱动不应该关心底层控制器寄存器怎么写。否则同一个传感器驱动，每换一个平台就重写一遍，那真的绝了，驱动工程师会原地裂开。

所以 Linux 必须抽象。

它要把“控制器怎么发送数据”和“外设需要发送什么数据”拆开。

这个拆开，就是 Linux I2C 分层设计的起点。

### 1.2 裸机 I2C 驱动的常见痛点

裸机写 **I2C**，很多人都写过。最常见的方式有两种。

一种是直接操作 SoC I2C 控制器寄存器。

另一种是 **GPIO** 模拟 I2C，也就是常说的 `bit-bang`。

裸机项目小的时候，这么写没什么问题。比如一个单片机，只接一个 EEPROM。初始化 I2C，写几个函数完事：

```
i2c_start();
i2c_write_byte(addr);
i2c_write_byte(reg);
i2c_read_byte();
i2c_stop();
```

但项目稍微复杂一点，问题就开始冒出来了。

> 同一个 I2C 控制器上挂了多个设备，谁来管理地址？

> 多个业务模块同时访问 I2C，谁来加锁？

> 一个设备读写失败，错误码怎么往上传？

> 控制器支持中断模式，也支持轮询模式，怎么抽象？

> 某个芯片要先开 `regulator`，再 release reset，再访问 I2C，这些时序放哪里？

> 设备断电后重新上电，驱动怎么恢复状态？

裸机里很多问题靠“约定”和“经验”解决。写的人知道，维护的人不一定知道。板子一多，供应商一换，痛苦就来了。

我以前调过一块板子，触摸屏和摄像头配置芯片挂在同一个 I2C 控制器上。触摸屏驱动里写了一套 I2C 访问函数，摄像头那边又写了一套，`PMIC` 那边还有一套。看上去每个模块都能跑，但只要系统启动顺序稍微变一下，某个设备就偶现 **NACK**。后来查了半天，才发现有个模块在访问失败后没有恢复控制器状态，后面的设备全被坑。

这类问题在裸机里很常见。不是因为大家水平差，而是因为缺一套统一的框架。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/6a886a55a79aebf186d03896ef805227_MD5.jpg]]

### 1.3 Linux 为什么要对 I2C 进行框架化抽象

既然裸机写法问题这么多，Linux内核设计师索性直接做了一套通用I2C框架。核心目的就一个：**解耦**。把硬件控制器、总线、外设、业务逻辑彻底拆开，各司其职。

控制器驱动解决底层发送问题。

外设驱动解决设备业务问题。

核心层负责把两者连接起来。

用户态如果想临时访问，也提供统一入口。

于是 Linux I2C 框架里就有了几个非常关键的对象。

`i2c_adapter` 表示一条 I2C 总线，或者说一个控制器抽象。

`i2c_algorithm` 表示这个控制器具体怎么传输。

`i2c_client` 表示挂在某条 I2C 总线上的一个外设。

`i2c_driver` 表示能驱动某类 I2C 外设的驱动。

`I2C Core` 则负责设备注册、驱动匹配、传输转发、总线管理这些通用事情。

看起来概念很多，但它们的分工其实很朴素。

你可以先记住一句话：

```
adapter 负责总线
client 负责设备
driver 负责驱动
algorithm 负责传输方法
core 负责把它们串起来
```

这句话理解了，Linux I2C 框架就已经入门一半了。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b88ce2c10cd88ececcf58cc7303fd4b5_MD5.jpg]]

## 二、Linux I2C 框架整体分层模型

### 2.1 Linux I2C 框架的整体架构图

先给一张文字版架构图。

```
用户空间
  |
  |  open("/dev/i2c-x")
  |  ioctl(I2C_SLAVE / I2C_RDWR)
  v
i2c-dev 用户态访问层
  |
  v
I2C Core 核心层
  |
  |---- i2c_client
  |---- i2c_driver
  |---- i2c_adapter
  |---- i2c_msg
  |
  v
Adapter 层
  |
  |---- i2c_adapter
  |---- i2c_algorithm
  |---- master_xfer()
  |
  v
SoC I2C 控制器驱动层
  |
  |---- 寄存器
  |---- 中断
  |---- 时钟
  |---- pinctrl
  |
  v
I2C 硬件总线
  |
  |---- EEPROM
  |---- RTC
  |---- Sensor
  |---- Touch IC
```

如果从外设驱动角度看，路径会稍微不一样。

```
I2C 外设驱动
  |
  |  i2c_transfer()
  |  i2c_smbus_read_byte_data()
  v
I2C Core
  |
  v
i2c_adapter
  |
  v
adapter->algo->master_xfer()
  |
  v
SoC I2C 控制器驱动
  |
  v
SCL / SDA 波形
```

一个重点来了。

Linux I2C 里不是所有访问都从用户态开始。大多数正规设备，比如触摸屏、RTC、传感器，都是内核态驱动访问 I2C。用户态的 `/dev/i2c-x` 更多用于调试、测试、板级验证，或者少量简单应用场景。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/e306e5d08939b2de5309bd56c126ac7d_MD5.jpg]]

### 2.2 从硬件控制器到用户空间的分层路径

我们从底往上看。

最底层是硬件 I2C 控制器。它在 SoC 内部，有自己的寄存器。比如控制寄存器、状态寄存器、数据寄存器、中断状态寄存器、时钟分频寄存器。

再上一层是控制器驱动。它知道这些寄存器怎么配置，知道怎么产生 **START**，怎么发送地址，怎么处理 **ACK/NACK**，怎么等待传输完成。

再上一层是 adapter 抽象。Linux 不希望上层关心这个控制器是 Rockchip 的、NXP 的还是 `STM32` 的。所以它把每个控制器抽象成一个 `i2c_adapter`。

然后是 `I2C Core`。它把 `adapter`、`client`、`driver`、`message`、`SMBus` 接口都管理起来。

再上面是外设驱动。外设驱动不直接碰控制器寄存器，只通过 `I2C Core` 提供的 API 发请求。

还有一条分支是用户态访问。`i2c-dev` 会给每个 `adapter` 暴露一个 `/dev/i2c-x` 字符设备。用户程序通过 `ioctl` 设置目标地址，再发起读写。

这套分层的好处是，外设驱动基本可以跨平台。

一个 EEPROM 驱动，不需要知道底层 I2C 控制器是哪个厂家的。只要这个平台的 I2C adapter 正常注册，传输接口正常实现，EEPROM 驱动就能工作。

这就是框架化的意义。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8db8418c44bcd4cfe20384b2146bbfba_MD5.jpg]]

### 2.3 各层之间的职责边界

我们可以这样分：

```
SoC 控制器驱动
  关心：寄存器、中断、时钟、DMA、FIFO、传输状态

Adapter 层
  关心：把控制器能力抽象成 Linux I2C 总线

I2C Core
  关心：设备模型、驱动匹配、传输封装、锁、统一 API

外设驱动
  关心：芯片手册、寄存器表、初始化序列、业务数据

用户态访问层
  关心：临时访问、调试工具、简单用户程序
```

边界清楚了，很多问题就好定位。

比如你驱动 **probe** 不进，大概率先看设备树和匹配机制。

如果 probe 进了，但 `i2c_transfer()` 失败，就看 `adapter`、控制器、中断、`pinctrl`、时钟、电源。

如果 `i2cdetect` 能扫到，但内核驱动不工作，就看外设驱动初始化流程。

要不然你就会陷入一种很痛苦的状态：哪里都怀疑，哪里都改一点，最后板子更玄学了。

### 2.4 I2C Core 在分层模型中的核心作用

`I2C Core` 是这套框架的中枢。它不直接产生 **SCL/SDA** 波形，但没有它，上下层就散了。

`I2C Core` 做的事情包括：

> 管理 `i2c_adapter` 的注册和注销。

> 管理 `i2c_client` 的创建和删除。

> 管理 `i2c_driver` 的注册和匹配。

> 提供 `i2c_transfer()` 这类统一传输接口。

> 封装 `SMBus` 访问接口。

> 处理部分锁和错误返回。

> 把设备模型和 Linux driver model 接起来。

你可以把 `I2C Core` 想成一个调度员。

控制器驱动说：“我这里有一条 I2C 总线，能发数据。”

外设设备说：“我挂在这条总线上，地址是 0x50，compatible 是 xxx。”

外设驱动说：“我能驱动 compatible 为 xxx 的设备。”

`I2C Core` 在中间说：“行，你们匹配上了，我调用 probe。后面外设驱动要传输，就从 adapter 的 `master_xfer` 走。”

它不抢活，但它管流程。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/7ef65b6b669451d6b7bd41c335566468_MD5.jpg]]

## 三、控制器驱动层，SoC I2C 控制器如何接入内核

### 3.1 I2C 控制器驱动负责什么

I2C控制器驱动，也叫适配器驱动，它的活就是**把具体的I2C硬件控制器包装成内核的****`i2c_adapter`**。控制器驱动是最贴近硬件的一层。

具体来说要做这几件事：

-   • 初始化I2C控制器的时钟
    
-   • 配置I2C控制器的引脚（复用功能）
    
-   • 设置I2C总线速率（标准模式100k、快速模式400k等）
    
-   • 实现中断处理函数
    
-   • 实现`i2c_algorithm`里的`master_xfer()`回调
    
-   • 调用`i2c_add_adapter()`把自己注册到系统里
    

控制器驱动不关心上层要读什么数据、写什么数据。它只关心一件事：**上层给我一个**`i2c_msg`**数组，我负责把这些消息转化成硬件时序发出去**。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3f00328c0a8fbc5ff6eb2c27a39033f6_MD5.jpg]]

### 3.2 SoC I2C 控制器与平台驱动模型

Linux内核中，所有片上外设控制器，包括I2C、`SPI`、`UART`、`GPIO`，全部基于平台驱动模型实现。也就是`platform_driver`，这是内核适配片上硬件的标准模型。

为什么要用平台驱动？

因为SoC的控制器是片上固定资源，没有热插拔需求，平台驱动刚好适配这种静态硬件资源。而且平台驱动完美支持设备树匹配，能通过设备树参数灵活配置时钟、速率、引脚、中断号，不用硬编码，适配性极强。

我随便贴一段标准的I2C控制器平台驱动框架代码，大家就能直观感受到结构有多固定。

```
static const struct of_device_id xxx_i2c_of_match[] = {
    { .compatible = "xxx,xxx-i2c" },
    { /* Sentinel */ }
};

static struct platform_driver xxx_i2c_driver = {
    .probe  = xxx_i2c_probe,
    .remove = xxx_i2c_remove,
    .driver = {
        .name = "xxx-i2c",
        .of_match_table = xxx_i2c_of_match,
    },
};
module_platform_driver(xxx_i2c_driver);
```

这段代码是所有I2C控制器驱动的标配。设备树里的`compatible`属性和`of_device_id`匹配之后，内核就会调用**probe**函数，执行控制器初始化、总线注册等全套逻辑。

那为什么设备树`compatible`必须和驱动一致？

这就是平台驱动的匹配规则，一字不差才能触发probe，这也是很多设备注册失败的核心原因之一。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3ff566aa5790771c2b469878943cbc46_MD5.jpg]]

### 3.3 控制器驱动中的寄存器、中断与时钟处理

控制器probe函数里，核心工作就是三件事：拿资源、配硬件、注册总线。最核心的工作是把一个抽象传输请求变成硬件动作。

上层传下来的是 `i2c_msg`。

底层要做的是：

> 设置目标地址。

> 配置读写方向。

> 写 TX FIFO。

> 启动传输。

> 等待中断。

> 读取 RX FIFO。

> 判断 ACK/NACK。

> 处理 STOP。

> 返回传输结果。

比如一个控制器的发送流程可能是这样：

```
写目标地址寄存器
写数据长度寄存器
把数据写入 TX FIFO
设置 START
使能传输
等待中断完成
检查错误状态
清中断
返回结果
```

这里面最容易出问题的是时钟、中断和状态位。

时钟没开，寄存器读写可能全是 0，或者写了没反应。

`pinctrl` 没配，**SCL/SDA** 可能还在 GPIO 模式，或者被 `UART`/`SPI` 占着。

中断没来，传输就卡在等待完成。

状态位没清干净，下一次传输直接异常。

还有一个很常见的问题，总线忙。

有时候设备上一次传输异常，SDA 被外设拉低，控制器看到 bus busy，一直不发起新的传输。这时候软件再怎么重试都不一定有用，得恢复总线，比如切 GPIO 拉 SCL，或者 reset 控制器，甚至重新上电外设。

别笑，这种问题现场很常见。尤其是客户拿一根飞线接了个传感器，还问你为什么偶现死机。你去现场一看，线长得像天线，I2C 波形边沿跟山路一样。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/eca7572167a68831443c0eda092cf3b7_MD5.jpg]]

### 3.4 控制器驱动如何向内核注册 I2C 总线能力

控制器驱动完成硬件初始化后，需要向 `I2C Core` 注册一个 `i2c_adapter`。

核心是填充 `adapter`，并提供 `algorithm`：

```
static const struct i2c_algorithm xxx_i2c_algo = {
    .master_xfer   = xxx_i2c_xfer,
    .functionality = xxx_i2c_func,
};

static int xxx_i2c_register_adapter(struct platform_device *pdev,
                                    struct xxx_i2c_dev *i2c)
{
    struct i2c_adapter *adap = &i2c->adap;

    adap->owner = THIS_MODULE;
    adap->algo = &xxx_i2c_algo;
    adap->dev.parent = &pdev->dev;
    adap->nr = pdev->id;
    strscpy(adap->name, "xxx-i2c-adapter", sizeof(adap->name));

    i2c_set_adapdata(adap, i2c);

    return i2c_add_adapter(adap);
}
```

这里最关键的是：

```
adap->algo = &xxx_i2c_algo;
```

因为上层最终会通过 `adapter` 找到 `algorithm`，再调用 `master_xfer()`。

这就是 I2C 框架里非常经典的一条线：

```
i2c_transfer()
    -> __i2c_transfer()
        -> adap->algo->master_xfer()
            -> 控制器驱动自己的传输函数
```

上层不关心你底层是 FIFO、中断、`DMA` 还是轮询。

你只要实现好 `master_xfer()`，就等于告诉 `I2C Core`：这条总线我能发。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/fd6141aed02c23ea3a35506c0661b5b1_MD5.jpg]]

## 四、Adapter 层，i2c\_adapter 与 i2c\_algorithm

### 4.1 i2c\_adapter 是一条 I2C 总线在内核中的抽象

`i2c_adapter` 是理解 Linux I2C 框架非常重要的结构。

它表示一条 I2C 总线，或者更准确地说，表示一个能发起 I2C 传输的控制器通道。

如果 SoC 有 5 个 I2C 控制器，正常情况下就可能注册出 5 个 `adapter`。

用户空间看到的 `/dev/i2c-0`、`/dev/i2c-1`，本质上也和这些 `adapter` 有关系。

一个 `adapter` 背后可能对应一个真实硬件控制器，也可能是一个模拟 I2C 控制器，比如 GPIO `bit-bang`。上层不用管。只要它是 `adapter`，就可以按 I2C 总线来使用。

这就是抽象带来的爽感。

对外设驱动来说，它只需要知道自己的 `client->adapter` 是哪条总线。至于这条总线底下怎么发，那是 `adapter` 和 `algorithm` 的事。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/a230b7a92140b69e9396665a4faf3e32_MD5.jpg]]

### 4.2 i2c\_algorithm 是控制器传输能力的抽象

`i2c_algorithm` 描述一个 `adapter` 能干什么，以及怎么干。

典型成员包括：

```
struct i2c_algorithm {
    int (*master_xfer)(struct i2c_adapter *adap,
                       struct i2c_msg *msgs,
                       int num);

    int (*smbus_xfer)(struct i2c_adapter *adap,
                      u16 addr,
                      unsigned short flags,
                      char read_write,
                      u8 command,
                      int size,
                      union i2c_smbus_data *data);

    u32 (*functionality)(struct i2c_adapter *adap);
};
```

不同内核版本细节可能会有变化，但核心思想一直在。

> `master_xfer()` 负责标准 I2C 传输。

> `smbus_xfer()` 负责 `SMBus` 风格传输，有些控制器可以硬件支持，有些可以通过 I2C 模拟。

> `functionality()` 用来告诉上层这个 `adapter` 支持哪些能力，比如是否支持 I2C、是否支持 SMBus byte data、word data 等。

外设驱动调用某些 SMBus API 时，`I2C Core` 可能会检查 `adapter` 的能力。如果底层不支持，对应操作就可能失败。

所以你看到一些驱动里会有类似这样的检查：

```
if (!i2c_check_functionality(client->adapter, I2C_FUNC_I2C))
    return -EOPNOTSUPP;
```

这个检查是在确认底层 `adapter` 是否真的支持你要用的传输方式。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b198d1cff6bb60e5def161de6fc7b05c_MD5.jpg]]

### 4.3 master\_xfer 回调与底层传输实现

`master_xfer()` 是控制器驱动的核心战场。

它接收一组 `i2c_msg`，然后完成传输。

为什么是一组？

因为很多 I2C 操作不是单独一个读或者写，而是组合传输。比如先写寄存器地址，再读数据，中间不能释放总线，只能用 `repeated start`。

对应的 `i2c_msg` 可能是两个：

```
struct i2c_msg msgs[2];

msgs[0].addr  = client->addr;
msgs[0].flags = 0;
msgs[0].len   = 1;
msgs[0].buf   = ®

msgs[1].addr  = client->addr;
msgs[1].flags = I2C_M_RD;
msgs[1].len   = len;
msgs[1].buf   = data;

ret = i2c_transfer(client->adapter, msgs, 2);
```

这里 `msgs[0]` 是写寄存器地址。

`msgs[1]` 是读数据。

`i2c_transfer()` 会把这两个 message 作为一组交给底层 `adapter`。底层控制器驱动要保证它们之间的总线时序符合要求。

实际波形通常类似这样：

```
START + addr(W) + reg + RESTART + addr(R) + data + STOP
```

如果控制器驱动没有正确处理 `repeated start`，一些外设就会读失败。

这个问题我真遇到过。某个触摸芯片在一个平台上好好的，换到另一个平台就读不到坐标。驱动完全一样，设备树地址也没错。最后拿逻辑分析仪一看，两个平台的 `repeated start` 时序不一样。那一刻你会发现，软件层看起来一样，底层控制器实现差一点，结果完全不同。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/de1534f3d3ea716902fc75f7bdb43153_MD5.jpg]]

### 4.4 i2c\_add\_adapter() 的注册流程

控制器驱动调用 `i2c_add_adapter()` 之后，`I2C Core` 会把这个 `adapter` 加入系统。

大致会做几件事：

> 给 `adapter` 分配编号。

> 把 `adapter` 注册到设备模型。

> 创建相关 `sysfs` 节点。

> 通知 `I2C Core` 这条总线可用了。

如果启用了 `i2c-dev`，后面还会有对应的用户态访问节点。

注册成功后，你通常能在系统里看到类似信息：

```
ls /sys/class/i2c-adapter/
i2c-0  i2c-1  i2c-2
```

或者：

```
ls /dev/i2c-*
/dev/i2c-0  /dev/i2c-1
```

当然，`/dev/i2c-x` 还依赖 `i2c-dev` 驱动是否启用。

这里有个新手容易误解的点。

看到 `/dev/i2c-1`，只能说明 `adapter` 注册了，不代表某个外设一定存在，也不代表设备驱动一定 probe 成功。

它只说明这条 I2C 总线入口在。

设备在不在，还得扫地址或者看设备树和驱动匹配。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/89a449fb8d1ae46765f90d41dc857939_MD5.jpg]]

### 4.5 多个 I2C Adapter 与硬件总线编号的关系

嵌入式板子上经常有多个 I2C 控制器，比如 I2C0、I2C1、I2C2。

Linux 里的 `adapter` 编号不一定永远等于硬件手册里的编号。很多平台会通过设备树 `alias` 固定编号：

```
aliases {
    i2c0 = &i2c0;
    i2c1 = &i2c1;
    i2c2 = &i2c2;
};
```

如果没有 `alias`，编号可能取决于注册顺序。

这就会带来一个很现实的问题。

你以为传感器在 `/dev/i2c-1`，实际它在 `/dev/i2c-3`。

然后你拿 `i2cdetect -y 1` 扫半天，啥也没有。你开始怀疑硬件，怀疑焊接，怀疑人生。结果最后发现扫错总线。

所以调 I2C 的时候，我一般会先看：

```
ls /sys/class/i2c-adapter/
cat /sys/class/i2c-adapter/i2c-1/name
```

确认这条总线到底对应哪个控制器。别一上来就扫，扫错了也白扫。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/983c65a506d077858a8b85f4863657c8_MD5.jpg]]

## 五、Core 层，I2C Core 如何统一管理设备、驱动和传输

### 5.1 I2C Core 的核心职责

`I2C Core` 是 Linux I2C 框架里最像“中间层”的部分。

它不写外设寄存器。

它也不直接控制 **SCL/SDA**。

它主要负责把 Linux 设备模型和 I2C 总线模型结合起来。

它要处理的问题包括： `adapter` 注册进来后怎么管理。

设备树里的 I2C 子节点怎么变成 `i2c_client`。

外设驱动注册进来后怎么匹配设备。

外设驱动调用传输 API 时怎么转发到底层控制器。

用户态通过 `i2c-dev` 访问时怎么复用同一套传输路径。

还有锁、错误码、`SMBus` 封装等通用机制。

你可以把 `I2C Core` 理解成“胶水层”，但别小看胶水层。一个系统真正稳不稳，很多时候就看胶水层写得是不是规矩。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/fb5b498b4aa1c5b5c9e90a574d023669_MD5.jpg]]

### 5.2 i2c\_client、i2c\_driver、i2c\_adapter 的关系

这三个结构是 I2C 框架的骨架。

> `i2c_adapter` 表示总线。

> `i2c_client` 表示设备。

> `i2c_driver` 表示驱动。

它们之间的关系可以画成这样：

```
i2c_adapter
    |
    |--- i2c_client addr=0x38 compatible="xxx,touch"
    |         |
    |         +--- matched with i2c_driver
    |
    |--- i2c_client addr=0x50 compatible="xxx,eeprom"
              |
              +--- matched with another i2c_driver
```

一个 `adapter` 上可以挂多个 `client`。

一个 `i2c_driver` 可以匹配多个 `client`，只要设备 ID 或 `compatible` 对得上。

一个 `client` 最终绑定一个 `driver`。

这里有点像 `platform` 总线的设备和驱动匹配，只不过 I2C 设备还多了一个地址属性。因为同一条总线上，地址是非常关键的。

比如设备树里这样写：

```
eeprom@50 {
    compatible = "atmel,24c02";
    reg = <0x50>;
};
```

这里的 `reg = <0x50>` 表示 I2C 设备地址。

内核解析这个节点后，会创建一个 `i2c_client`，它的 `addr` 就是 0x50。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3e38108cb022dd891814dba8ad8995a3_MD5.jpg]]

### 5.3 I2C 总线类型 i2c\_bus\_type 的作用

Linux 设备模型里有 bus、device、driver。

I2C 也有自己的 bus type，也就是 `i2c_bus_type`。

这个 bus type 负责 I2C 设备和 I2C 驱动之间的匹配逻辑。

你注册一个 `i2c_driver`，本质上是把一个 driver 注册到 I2C bus。

你创建一个 `i2c_client`，本质上是把一个 device 挂到 I2C bus。

然后设备模型会尝试匹配。

匹配成功后，调用 driver 的 **probe**。

所以 `i2c_driver` 里的 probe 并不是你手动调用的，而是匹配机制自动触发的。

典型外设驱动结构如下：

```
static int xxx_probe(struct i2c_client *client)
{
    struct xxx_data *data;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    i2c_set_clientdata(client, data);

    /* 读取芯片 ID，初始化寄存器 */
    return 0;
}

static void xxx_remove(struct i2c_client *client)
{
    /* 释放资源，关闭设备 */
}

static const struct of_device_id xxx_of_match[] = {
    { .compatible = "vendor,xxx-sensor" },
    { }
};
MODULE_DEVICE_TABLE(of, xxx_of_match);

static const struct i2c_device_id xxx_id[] = {
    { "xxx-sensor", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, xxx_id);

static struct i2c_driver xxx_driver = {
    .driver = {
        .name = "xxx-sensor",
        .of_match_table = xxx_of_match,
    },
    .probe = xxx_probe,
    .remove = xxx_remove,
    .id_table = xxx_id,
};

module_i2c_driver(xxx_driver);
```

不同内核版本里 probe 函数原型可能有过调整，但整体思路不变。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/0182c16c59083a148bece3fbadb228b9_MD5.jpg]]

### 5.4 I2C 设备与驱动的匹配机制

I2C 设备和驱动匹配常见有几种方式。 设备树匹配。 传统 `i2c_device_id` 匹配。 `ACPI` 匹配。

嵌入式 Linux 里最常见的是设备树匹配。

设备树节点：

```
sensor@48 {
    compatible = "vendor,xxx-sensor";
    reg = <0x48>;
};
```

驱动里：

```
static const struct of_device_id xxx_of_match[] = {
    { .compatible = "vendor,xxx-sensor" },
    { }
};
```

只要 `compatible` 一致，`I2C Core` 和设备模型就有机会把它们匹配起来。

但匹配成功是有前提的：

> I2C 控制器节点要 enabled。

> `adapter` 要注册成功。

> 子设备节点要在控制器节点下面。

> 地址要合法。

> 驱动模块要加载。

> `compatible` 要写对。

这些条件少一个，probe 都可能不进。

所以“设备树写了，probe 不进”这种问题，不要只盯着外设驱动。你要顺着链路查。

> 控制器节点 status 是不是 okay？

> `pinctrl` 有没有报错？

> `adapter` 有没有注册？

> `client` 有没有创建？

> driver 有没有加载？

> `compatible` 有没有拼错？

这个排查顺序，比乱改代码靠谱多了。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/7206a3fef76091f4765958dd04ee380e_MD5.jpg]]

### 5.5 i2c\_transfer() 与 i2c\_smbus\_xfer() 的封装意义

Core层给上层驱动提供了两套通用读写接口，覆盖所有I2C设备场景。

一类是标准 I2C transfer。

一类是 `SMBus` 风格 API。

标准 I2C transfer 更灵活：

```
int i2c_transfer(struct i2c_adapter *adap,
                 struct i2c_msg *msgs,
                 int num);
```

你自己组织 `i2c_msg`，适合复杂读写。

SMBus API 更像寄存器访问封装：

```
i2c_smbus_read_byte_data(client, reg);
i2c_smbus_write_byte_data(client, reg, value);
i2c_smbus_read_word_data(client, reg);
```

这类接口写起来舒服，适合很多简单外设。

但别误会。不是所有设备都适合 SMBus API。

有些设备要求特殊时序，比如多字节寄存器地址、连续 burst 读写、特殊 `repeated start`。这种时候用 `i2c_transfer()` 更稳。

我个人的习惯是： 简单 8-bit 寄存器设备，用 SMBus API。 寄存器地址是 16-bit，或者读写时序比较特殊，就直接组 `i2c_msg`。

别为了代码短，牺牲时序可控性。后面调起来会哭。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b3b2dc29bd23918900b46fefcd854f04_MD5.jpg]]

## 六、Client / Driver 层，I2C 外设驱动如何工作

### 6.1 i2c\_client 是一个 I2C 外设在内核中的表示

`i2c_client` 表示一个挂在 I2C `adapter` 上的外设。

它里面最关键的信息包括：

> 设备地址。

> 所属 `adapter`。

> 设备对象 `dev`。

> 设备名。

> 一些标志位。

外设驱动的 probe 拿到的就是 `i2c_client`。

```
static int xxx_probe(struct i2c_client *client)
{
    dev_info(&client->dev, "addr = 0x%x\n", client->addr);
    dev_info(&client->dev, "adapter = %s\n", client->adapter->name);

    return 0;
}
```

通过 `client`，驱动可以知道自己在哪条总线上，目标地址是多少，也可以通过 `client->dev` 使用 `devm` 资源管理、打印日志、获取设备树属性。

比如读取一个 GPIO：

```
data->reset_gpio = devm_gpiod_get_optional(&client->dev,
                                            "reset",
                                            GPIOD_OUT_HIGH);
```

读取 `regulator`：

```
data->vdd = devm_regulator_get(&client->dev, "vdd");
```

读取设备树属性：

```
device_property_read_u32(&client->dev, "vendor,poll-interval", &interval);
```

所以 I2C 外设驱动不是只干 I2C 读写。一个完整外设驱动还要处理电源、复位、中断、输入子系统、字符设备、`IIO`、`hwmon` 等框架。I2C 只是它和芯片通信的通道。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/7c11d54391d2bdf75c9151bcd4817eef_MD5.jpg]]

### 6.2 i2c\_driver 是 I2C 外设驱动的核心结构

`i2c_driver` 描述一个 I2C 外设驱动。

它通常包括：

> driver name。

> `of_match_table`。

> `id_table`。

> probe。

> remove。

可能还有 power management 回调。

一个稍微完整一点的外设驱动结构如下：

```
struct xxx_data {
    struct i2c_client *client;
    struct regulator *vdd;
    struct gpio_desc *reset_gpio;
    int irq;
};

static int xxx_read_reg(struct xxx_data *data, u8 reg)
{
    return i2c_smbus_read_byte_data(data->client, reg);
}

static int xxx_write_reg(struct xxx_data *data, u8 reg, u8 val)
{
    return i2c_smbus_write_byte_data(data->client, reg, val);
}

static int xxx_probe(struct i2c_client *client)
{
    struct xxx_data *data;
    int chip_id;
    int ret;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->client = client;
    i2c_set_clientdata(client, data);

    data->vdd = devm_regulator_get(&client->dev, "vdd");
    if (IS_ERR(data->vdd))
        return PTR_ERR(data->vdd);

    ret = regulator_enable(data->vdd);
    if (ret)
        return ret;

    chip_id = xxx_read_reg(data, 0x00);
    if (chip_id < 0) {
        ret = chip_id;
        goto err_power;
    }

    dev_info(&client->dev, "chip id = 0x%x\n", chip_id);

    ret = xxx_write_reg(data, 0x10, 0x01);
    if (ret)
        goto err_power;

    return 0;

err_power:
    regulator_disable(data->vdd);
    return ret;
}
```

这个例子很普通，但已经能看出外设驱动的基本套路： 拿资源→上电→读 ID→初始化寄存器→注册到对应子系统→失败路径清理。

很多驱动出问题，不是 I2C API 不会用，而是失败路径写得太随便。probe 中间失败，电源没关，reset 状态不对，中断已经申请但资源没处理干净。下一次再加载，系统状态就乱了。

这类 bug 不一定每次出现，最烦。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3fc0e945bec5fc6f05b7bac999754622_MD5.jpg]]

### 6.3 probe/remove 回调的作用

**probe()是驱动和设备的"婚礼"。匹配成功之后，内核调用probe()**，驱动在这里"接管"设备。

一个典型的probe()大概长这样：

```
static int my_sensor_probe(struct i2c_client *client,
                           const struct i2c_device_id *id)
{
    struct my_sensor_data *data;
    
    /* 1. 分配私有数据 */
    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    i2c_set_clientdata(client, data);
    
    /* 2. 检查设备是否活着 */
    if (my_sensor_check_device(client)) {
        dev_err(&client->dev, "device not found\n");
        return -ENODEV;
    }
    
    /* 3. 初始化设备（配置寄存器等） */
    my_sensor_init(client);
    
    /* 4. 注册到其他子系统 */
    data->input_dev = devm_input_allocate_device(&client->dev);
    /* ... 初始化input设备 ... */
    input_register_device(data->input_dev);
    
    return 0;
}
```

**remove()是probe()的反向操作。驱动卸载时，内核调用remove()**，驱动在这里释放资源、注销设备。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8e68d07cc4f691c20511ea7dc15c7714_MD5.jpg]]

### 6.4 驱动如何通过 i2c\_client 访问硬件设备

最常见的寄存器读写可以这样封装：

```
static int xxx_read_u8(struct i2c_client *client, u8 reg, u8 *val)
{
    int ret;

    ret = i2c_smbus_read_byte_data(client, reg);
    if (ret < 0)
        return ret;

    *val = ret & 0xff;
    return 0;
}

static int xxx_write_u8(struct i2c_client *client, u8 reg, u8 val)
{
    return i2c_smbus_write_byte_data(client, reg, val);
}
```

如果设备是 16 位寄存器地址，就可以用 `i2c_transfer()`：

```
static int xxx_read_block(struct i2c_client *client,
                          u16 reg, u8 *buf, int len)
{
    u8 addr_buf[2];
    struct i2c_msg msgs[2];
    int ret;

    addr_buf[0] = reg >> 8;
    addr_buf[1] = reg & 0xff;

    msgs[0].addr = client->addr;
    msgs[0].flags = 0;
    msgs[0].len = 2;
    msgs[0].buf = addr_buf;

    msgs[1].addr = client->addr;
    msgs[1].flags = I2C_M_RD;
    msgs[1].len = len;
    msgs[1].buf = buf;

    ret = i2c_transfer(client->adapter, msgs, 2);
    if (ret == 2)
        return 0;

    if (ret >= 0)
        return -EIO;

    return ret;
}
```

这里有个细节。

`i2c_transfer()` 成功时返回传输完成的 message 数量，不是返回 0。

所以你经常会看到：

```
if (ret == 2)
    return 0;
```

如果你把 `ret < 0` 当失败，`ret == 1` 当成功，那就埋坑了。因为 `ret == 1` 说明只完成了一部分 message，这种情况对组合读写来说通常也是失败。

### 6.5 常见 I2C 外设驱动场景

I2C外设驱动在嵌入式Linux里遍地都是： **传感器**（加速度计、陀螺仪、磁力计、温度传感器等）：通常注册到**IIO**（Industrial I/O）子系统。驱动在probe()里初始化传感器，然后提供`read_raw()`回调供上层读取数据。

**EEPROM**：比如AT24Cxx系列。驱动在**probe()里识别芯片型号，然后注册为一个MTD**设备或一个普通的二进制设备。应用程序可以像读写文件一样读写EEPROM。

**RTC**（实时时钟）：注册到**RTC**子系统。驱动提供`read_time()`和`set_time()`回调。用户用`hwclock`命令就能读写RTC时间。

**触摸屏**：注册到输入子系统。驱动在中断处理函数里读取触摸坐标，然后通过`input_report_abs()`上报给输入子系统。

这些驱动业务不同，但底层的I2C读写逻辑都是一样的——调用`i2c_transfer()`或者`i2c_master_send()`/`recv()`。区别只在于上层注册到了哪个子系统。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/a2026b9f286ae9acd999dabcf88a3a76_MD5.jpg]]

## 七、用户态访问层，/dev/i2c-x 与 i2c-dev

### 7.1 /dev/i2c-x 设备节点是怎么来的

`/dev/i2c-X`设备节点是`i2c-dev`驱动生成的。

`i2c-dev`是I2C子系统里的一个"通用驱动"。它不针对任何具体的I2C设备，而是为每个`i2c_adapter`创建一个字符设备节点。

每个注册的I2C适配器都有一个编号，`i2c-dev`会为它创建对应的`/dev/i2c-X`节点。主设备号是89，次设备号就是适配器的编号。

```
$ ls -l /dev/i2c-*
crw-rw---- 1 root i2c 89, 0 Jan 1 00:00 /dev/i2c-0
crw-rw---- 1 root i2c 89, 1 Jan 1 00:00 /dev/i2c-1
```

所有256个次设备号都预留给I2C了。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/6cdd460fe3a3f24705bff8e8de9b990c_MD5.jpg]]

### 7.2 i2c-dev 驱动在 I2C 框架中的位置

`i2c-dev` 介于用户态和内核Core层之间，属于框架的辅助驱动，不参与底层硬件适配，也不替代外设驱动。

`i2c-dev` 可以理解为 `I2C Core` 上面的一层用户态桥梁。

它不是某个具体外设的驱动。 它也不理解某个芯片的寄存器。 它只是把用户态请求转换成 `I2C Core` 能理解的传输请求。

路径大概是：

```
用户程序
  |
  v
/dev/i2c-x
  |
  v
i2c-dev
  |
  v
I2C Core
  |
  v
i2c_adapter
  |
  v
master_xfer()
```

所以用户态访问和内核态外设驱动，底层最终都可能走到 `adapter` 的传输函数。

这也是为什么 `i2cdetect` 可以用来判断底层总线是否基本可用。

但它不是万能的。 有些设备不喜欢被扫描。 有些地址被内核驱动占用。 有些设备读写需要特殊时序。 有些芯片扫一下可能改变状态。

所以在线上产品里，别随便乱扫 I2C 总线。实验室里可以，量产环境慎重。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/0c716207621b140f6deb1c6b4254c740_MD5.jpg]]

### 7.3 用户态通过 ioctl 访问 I2C 设备

一个简单用户态访问例子：

```
#include 
#include 
#include 
#include 
#include 

int main(void)
{
    int fd;
    int ret;
    unsigned char reg = 0x00;
    unsigned char val;

    fd = open("/dev/i2c-1", O_RDWR);
    if (fd < 0) {
        perror("open");
        return -1;
    }

    ret = ioctl(fd, I2C_SLAVE, 0x50);
    if (ret < 0) {
        perror("ioctl");
        close(fd);
        return -1;
    }

    ret = write(fd, ®, 1);
    if (ret != 1) {
        perror("write");
        close(fd);
        return -1;
    }

    ret = read(fd, &val, 1);
    if (ret != 1) {
        perror("read");
        close(fd);
        return -1;
    }

    printf("val = 0x%02x\n", val);

    close(fd);
    return 0;
}
```

这个例子能说明思路，但实际项目里更推荐使用 `I2C_RDWR` 做组合传输。因为很多寄存器读操作要求 `repeated start`，而简单的 write 再 read 不一定满足设备时序要求。

`I2C_RDWR` 示例：

```
#include 
#include 
#include 
#include 
#include 
#include 

int main(void)
{
    int fd;
    unsigned char reg = 0x00;
    unsigned char val = 0;
    struct i2c_msg msgs[2];
    struct i2c_rdwr_ioctl_data data;

    fd = open("/dev/i2c-1", O_RDWR);
    if (fd < 0) {
        perror("open");
        return -1;
    }

    msgs[0].addr = 0x50;
    msgs[0].flags = 0;
    msgs[0].len = 1;
    msgs[0].buf = ®

    msgs[1].addr = 0x50;
    msgs[1].flags = I2C_M_RD;
    msgs[1].len = 1;
    msgs[1].buf = &val;

    data.msgs = msgs;
    data.nmsgs = 2;

    if (ioctl(fd, I2C_RDWR, &data) < 0) {
        perror("I2C_RDWR");
        close(fd);
        return -1;
    }

    printf("val = 0x%02x\n", val);

    close(fd);
    return 0;
}
```

这个更接近内核态 `i2c_transfer()` 的使用方式。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/04e40473cf85b944ea813375960d8228_MD5.jpg]]

### 7.4 i2c-tools 工具的使用场景

做嵌入式调试，**i2c-tools**是我用得最多的工具，没有之一。排查I2C问题，90%的场景靠这一套工具就能定位问题，不用写代码、不用编译驱动，效率拉满。

**i2cdetect**：扫描I2C总线上的设备。

```
i2cdetect -y 1
```

输出一个地址矩阵，有设备的地方显示地址，没设备的地方显示`--`。这是排查I2C硬件问题的第一招。

**i2cget**：读取设备寄存器的值。

```
i2cget -y 1 0x50 0x10
```

读地址0x50的设备、寄存器0x10的值。

**i2cset**：写设备寄存器。

```
i2cset -y 1 0x50 0x10 0x55
```

**i2cdump**：批量读取设备所有寄存器。

**i2ctransfer**：最灵活的命令，支持复合传输。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/67f2796e15e1549ff0d631d42a34d96f_MD5.jpg]]

### 7.5 用户态访问与内核态驱动访问的区别

用户态通过`/dev/i2c-X`访问和内核态通过`i2c_transfer()`访问，底层走的是同一条路。区别在于：

**内核态驱动**可以享受完整的设备模型支持——probe/remove、电源管理、热插拔。驱动可以注册到其他子系统（输入、IIO、hwmon等），让应用程序通过标准接口访问设备。

**用户态访问**更灵活、更快速。写一个用户态程序访问I2C设备，改完代码重新编译就能跑，不需要编译内核、不需要重启。适合调试和原型验证。

但用户态访问无法利用设备模型（没有probe/remove、没有电源管理）、无法注册到其他子系统（只能自己处理数据）、存在安全风险（任何有权限的用户都能操作I2C总线）。

实际开发中，我一般是先用**i2c-tools**和用户态程序验证硬件和通信没问题，然后再写内核驱动。这样把问题分开排查，效率高很多。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8e286b4974ef31401b0eff34715a53ef_MD5.jpg]]

## 八、一次 I2C 读写请求的完整调用链

### 8.1 内核态 I2C 读写的典型调用方式

前面讲了分层结构，现在我们串起完整调用链，把所有知识点落地。先从最常用的内核态读写开始，这是驱动开发的**核心流程**。

在内核态外设驱动里，常见调用是：

```
ret = i2c_smbus_read_byte_data(client, reg);
```

或者：

```
ret = i2c_transfer(client->adapter, msgs, num);
```

前者更简单，后者更灵活。

不管用哪种方式，最终都要找到 `client` 所属的 `adapter`，然后让 `adapter` 背后的控制器驱动完成传输。

这条链可以简化成：

```
外设驱动
  |
  v
`i2c_smbus_xxx()` / `i2c_transfer()`
  |
  v
I2C Core
  |
  v
client->adapter
  |
  v
adapter->algo->master_xfer()
  |
  v
SoC I2C 控制器驱动
  |
  v
硬件寄存器
  |
  v
SCL/SDA
```

这就是从软件到硬件的**完整路径**。

读懂这条路径，调问题就有方向了。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/47aa8e0369db8083eacf29842c11a210_MD5.jpg]]

### 8.2 `i2c_msg` 如何描述一次 I2C 传输

`i2c_msg` 是标准 I2C 传输的**基本描述单元**，不管是内核态还是用户态传输，最终都会封装成`i2c_msg`消息。

核心结构体如下：

```
struct i2c_msg {
    __u16 addr;  /* 从设备地址 */
    __u16 flags; /* 读写标志、特殊时序标志 */
    __u16 len;   /* 数据长度 */
    __u8 *buf;   /* 数据缓冲区 */
};
```

`flags`是**核心成员**，0代表写操作，`I2C_M_RD`代表读操作。还支持无停止信号、10位地址等特殊时序配置，适配各类非标I2C设备。

一次完整的寄存器读取操作，必须由**两个msg**组成。第一个msg写寄存器地址，第二个msg读寄存器数据。内核支持批量传入多个msg，一次性完成多段时序，**效率极高**。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/281754a4483321e8601cc420b0957409_MD5.jpg]]

### 8.3 `i2c_transfer()` 到 adapter->algo->master\_xfer() 的调用路径

`i2c_transfer()`的调用链大概是这样：

```
`i2c_transfer()`
  └── `__i2c_transfer()`  [17†L9]
        └── adapter->algo->master_xfer(adapter, msgs, num)
```

`__i2c_transfer()`在调用`master_xfer`之前会做一些**准备工作**：加锁（防止多个线程同时访问同一条总线）、检查`adapter`是否支持这次传输、处理10位地址的转换等。

然后就是调用`adapter->algo->master_xfer()`。这个函数指针指向控制器驱动实现的**传输函数**。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/9f77cf65e77552ffe2d4feff04ecc5d1_MD5.jpg]]

### 8.4 控制器驱动如何真正操作硬件完成传输

控制器驱动的`master_xfer()`是真正操作硬件的地方。

它要做的事情：

1.  1.  **准备硬件**：使能时钟、配置传输参数（速率、模式等）
2.  2.  **发起传输**：写控制寄存器，触发START条件+发送设备地址
3.  3.  **等待中断**：用`wait_for_completion()`等待中断处理函数通知传输完成
4.  4.  **中断处理**：在ISR里一个字节一个字节地处理数据收发，处理ACK/NACK，处理STOP条件
5.  5.  **返回结果**：传输完成后，返回成功传输的消息数量或错误码

为了让你感受一下，我贴一段简化版的`master_xfer`伪代码：

```
static int my_i2c_xfer(struct i2c_adapter *adap,
                       struct i2c_msg *msgs, int num)
{
    struct my_i2c *i2c = i2c_get_adapdata(adap);
    int i, ret;
    
    for (i = 0; i < num; i++) {
        /* 设置传输方向 */
        if (msgs[i].flags & I2C_M_RD)
            my_i2c_set_read(i2c);
        else
            my_i2c_set_write(i2c);
        
        /* 设置设备地址 */
        my_i2c_set_addr(i2c, msgs[i].addr);
        
        /* 设置数据缓冲区和长度 */
        my_i2c_set_buf(i2c, msgs[i].buf, msgs[i].len);
        
        /* 触发硬件传输 */
        reinit_completion(&i2c->done);
        my_i2c_start(i2c);
        
        /* 等待中断 */
        ret = wait_for_completion_timeout(&i2c->done, HZ);
        if (ret == 0) {
            ret = -ETIMEDOUT;
            goto out;
        }
        
        if (i2c->error) {
            ret = i2c->error;
            goto out;
        }
    }
    
    return num;
out:
    /* 错误处理：发STOP、复位控制器等 */
    return ret;
}
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ff1e222ebcc59474e16c8b7abe01ebb0_MD5.jpg]]

### 8.5 用户态访问 I2C 时的调用链差异

用户态通过`/dev/i2c-X`访问时，调用链多了几层：

```
用户程序 read()/ioctl()
  └── VFS
        └── i2c-dev.c: `i2cdev_read()` / `i2cdev_ioctl()`  [5†L21-L22]
              └── `i2c_transfer()`  [17†L9]
                    └── `__i2c_transfer()`
                          └── adapter->algo->master_xfer()
                                └── 硬件操作
```

`i2c-dev`从用户空间复制数据到内核空间（或反过来），然后调用`i2c_transfer()`。之后的路和内核态调用**完全一样**。

唯一的区别是：用户态调用要经过系统调用、VFS、权限检查等**额外开销**。但这也带来了好处——用户态程序可以不用写内核代码就能操作I2C设备。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d0ce2507e044f53fb98faaf59eff77e6_MD5.jpg]]

## 九、设备树、匹配机制与注册流程

### 9.1 I2C 控制器节点在设备树中的描述方式

一个 I2C 控制器节点通常描述 SoC 内部的 I2C 控制器资源。

比如：

```
i2c2: i2c@fe5a0000 {
    compatible = "rockchip,rk3399-i2c";
    reg = <0x0 0xfe5a0000 0x0 0x1000>;
    interrupts = ;
    clocks = <&cru SCLK_I2C2>, <&cru PCLK_I2C2>;
    clock-names = "i2c", "pclk";
    pinctrl-names = "default";
    pinctrl-0 = <&i2c2_xfer>;
    #address-cells = <1>;
    #size-cells = <0>;
    status = "okay";
};
```

这里每个字段都可能影响驱动。

> `compatible` 用来匹配控制器驱动。

> `reg` 描述寄存器地址。

> `interrupts` 描述中断。

> `clocks` 描述时钟资源。

> `pinctrl-0` 描述 SCL/SDA 引脚复用。

> `status = "okay"` 表示启用。

> `#address-cells` 和 `#size-cells` 影响子设备地址解析。

如果控制器节点没起来，下面的 I2C 子设备也没戏。因为 `adapter` 都没注册，`client` 从哪来？

### 9.2 I2C 子设备节点如何描述外设地址和 compatible

I2C 外设节点通常写在控制器节点下面：

```
&i2c2 {
    status = "okay";

    temperature@48 {
        compatible = "vendor,temp-sensor";
        reg = <0x48>;
    };

    eeprom@50 {
        compatible = "atmel,24c02";
        reg = <0x50>;
        pagesize = <8>;
    };
};
```

外设节点里的 `reg` 是 I2C 地址。

节点名里的 `@48` 通常也对应地址，方便阅读和设备树规范检查。

`compatible` 用来匹配外设驱动。

如果外设还有中断、电源、复位，也会继续加属性：

```
touch@38 {
    compatible = "vendor,touch-ic";
    reg = <0x38>;
    interrupt-parent = <&gpio1>;
    interrupts = <10 IRQ_TYPE_EDGE_FALLING>;
    reset-gpios = <&gpio2 5 GPIO_ACTIVE_LOW>;
    vdd-supply = <&vcc_3v3>;
};
```

这些属性不会自动让设备工作。它们只是描述硬件连接。驱动要主动读取并使用这些资源。

设备树不是魔法。它只是**硬件说明书**的一种代码化表达。

### 9.3 设备树解析后如何生成 `i2c_client`

当 I2C `adapter` 注册后，I2C Core 会扫描该 `adapter` 下面的设备树子节点。

对于每个可用子节点，内核会创建对应的 `i2c_client`。

这个 `client` 里会保存：

> I2C 地址。

> 所属 `adapter`。

> 设备树节点信息。

> 设备对象。

然后设备模型会尝试匹配对应的 `i2c_driver`。

流程可以画成这样：

```
设备树 i2c 控制器节点
  |
  v
platform_driver probe
  |
  v
注册 `i2c_adapter`
  |
  v
I2C Core 解析子节点
  |
  v
创建 `i2c_client`
  |
  v
匹配 `i2c_driver`
  |
  v
调用外设驱动 probe
```

所以一个外设驱动 probe 的触发，前面其实走了很多步骤。

你只看到 probe 没进，但它可能卡在任何一环。

### 9.4 `of_match_table` 与 `i2c_device_id` 的匹配关系

现代嵌入式 Linux 大多用设备树，所以 `of_match_table` 很常见。

但 `i2c_device_id` 仍然建议保留。

一方面是为了传统非设备树方式。

另一方面模块自动加载时也可能依赖相关信息。

一个常见写法是：

```
static const struct of_device_id xxx_of_match[] = {
    { .compatible = "vendor,xxx" },
    { }
};
MODULE_DEVICE_TABLE(of, xxx_of_match);

static const struct i2c_device_id xxx_id[] = {
    { "xxx", 0 },
    { }
};
MODULE_DEVICE_TABLE(i2c, xxx_id);

static struct i2c_driver xxx_driver = {
    .driver = {
        .name = "xxx",
        .of_match_table = xxx_of_match,
    },
    .probe = xxx_probe,
    .remove = xxx_remove,
    .id_table = xxx_id,
};
```

设备树匹配看 `compatible`。

传统 ID 匹配看 name。

如果你发现手动 insmod 后 probe 不进，除了看 `compatible`，也要看模块 alias 有没有生成，驱动有没有真的注册进 I2C bus。

可以用：

```
modinfo xxx_driver.ko
```

看里面有没有 of alias 或 i2c alias。

这个细节有时候挺救命。

### 9.5 设备没有进入 probe 的**常见原因**

probe 不进是 I2C 驱动**最常遇到**的问题之一。

常见原因不少，比如： 控制器节点没开：

```
status = "disabled";
```

`compatible` 拼错：

```
compatible = "vender,xxx";
```

驱动里是：

```
.compatible = "vendor,xxx";
```

vendor 写成 vender，这种错真有人犯。别问我怎么知道的。

外设节点没放在 I2C 控制器节点下面。

`#address-cells` 或 `#size-cells` 不对。

I2C 控制器驱动没加载，`adapter` 没注册。

外设驱动没编进内核，也没加载模块。

内核版本 probe 原型改了，驱动代码没适配好。

设备地址非法，或者和其他设备冲突。

pinctrl 报错导致控制器 probe 失败。

排查建议按顺序来。

看 `adapter`：

```
ls /sys/class/i2c-adapter/
```

看内核日志：

```
dmesg | grep -i i2c
```

看设备树是否生效：

```
ls /proc/device-tree/
```

看驱动是否加载：

```
lsmod
```

看设备是否出现在 sysfs：

```
find /sys/bus/i2c/devices/ -maxdepth 1 -type l
```

这比直接改驱动靠谱。

## 十、调试方法与常见问题

### 10.1 使用 `dmesg` 查看 I2C 控制器和设备注册信息

调 I2C，第一步看日志。

```
dmesg | grep -i i2c
```

你要关注几类信息。

控制器驱动有没有 probe。

`adapter` 有没有注册。

pinctrl 有没有失败。

clock 有没有获取失败。

外设驱动有没有 probe。

有没有 NACK、timeout、bus busy。

如果你在驱动里加日志，建议带上 `adapter` 名称和设备地址：

```
dev_info(&client->dev, "probe addr=0x%x adapter=%s\n",
         client->addr, client->adapter->name);
```

传输失败时也别只打印 “i2c read failed”。

多打印一点上下文：

```
dev_err(&client->dev, "read reg 0x%02x failed, ret=%d\n",
        reg, ret);
```

未来你会感谢现在的自己。真的。

### 10.2 使用 `i2cdetect`、`i2cget`、`i2cset` 定位通信问题

常见调试流程：

```
i2cdetect -l
```

确认总线。

```
i2cdetect -y 1
```

扫描地址。

```
i2cget -y 1 0x48 0x00
```

读寄存器。

```
i2cset -y 1 0x48 0x10 0x01
```

写寄存器。

但工具使用要小心。

`i2cset` 可能改设备状态。

`i2cdetect` 对某些设备不友好。

如果地址显示 `UU`，通常表示这个地址已经被内核驱动占用。

比如：

```
0 1 2 3 4 5 6 7 8 9 a b c d e f
30: -- -- -- -- -- -- -- -- UU -- -- -- -- -- -- --
```

这不一定是坏事。它说明内核已经有驱动绑定了这个地址。你再从用户态访问，可能会冲突。

线上产品别随便拿工具怼。调试工具很好，但别把调试习惯带到量产脚本里，容易搞事情。

### 10.3 使用逻辑分析仪观察 SCL、SDA 波形

软件调不通的时候，就上逻辑分析仪了。

I2C 是低速总线，逻辑分析仪非常好用，触发抓取一段通信波形，就能看到：

-   • START条件对不对
    
-   • 设备地址发对了没有
    
-   • 从设备有没有回ACK
    
-   • 数据位是否正确
    
-   • STOP条件有没有
    

有时候日志显示 timeout，但真正原因要看波形。

比如：

> SCL 根本没动，可能 pinctrl 或 clock 有问题。

> SCL 有，SDA 没变化，可能引脚复用或上拉有问题。

> 地址发出去了，设备 NACK，可能地址错、电源没开、reset 没释放、芯片没焊好。

> 读到数据全 0xff，可能总线空读、上拉太强、设备没响应。

> 波形边沿很慢，可能上拉电阻过大、线太长、电容太大。

这种问题靠代码看不出来。别硬看。上仪器。

我见过有人调 I2C 调两天，不上逻辑分析仪。最后一接，发现 SCL/SDA 接反。那一瞬间会议室安静了三秒钟。太真实了。

### 10.4 I2C 设备不识别的**常见原因**

设备在`i2cdetect`里扫不到，最常见的原因：

**供电问题**。设备没电、或者电压不对（3.3V的设备接了1.8V）。

**引脚连接错误**。SCL和SDA接反了、或者没接对引脚。

**上拉电阻缺失或阻值不对**。I2C是开漏的，必须要有上拉电阻。阻值太大上升沿太慢，阻值太小功耗太大。通常4.7kΩ是个安全的选择。

**地址错误**。设备数据手册上的地址可能是8位的（含读写位），而`i2cdetect`用的是7位地址。别搞混了。

**总线被占用**。另一个设备把总线拉低了（SCL或SDA一直低电平）。用示波器看一眼两根线的电平。

**时钟太快**。设备支持不了400kHz，但控制器配成了400kHz。改成100kHz试试。

### 10.5 I2C 通信超时、NACK、总线挂死的**排查思路**

**通信超时**：`master_xfer()`返回`-ETIMEDOUT`。

可能原因：设备没响应（检查供电和连接）、时钟配置不对、中断没触发（检查中断号和中断引脚配置）。先用`i2cdetect`确认设备能不能被扫到。

**NACK错误**：设备回了NACK而不是ACK。

可能原因：地址不对、设备忙、命令不支持、写入了只读寄存器。用逻辑分析仪抓一下波形，看看到底是哪一步出了NACK。

**总线挂死**：SCL或SDA一直保持低电平，后续所有通信都失败。

I2C总线挂死通常是因为设备异常将总线拉低。解决办法：

1.  1\. 硬件复位设备（断电重启）
    
2.  2\. 软件复位I2C控制器（有些控制器支持软件复位）
    
3.  3\. 如果控制器支持，发送9个时钟脉冲来解锁总线
    

预防措施：在驱动里加超时处理，超时后主动复位控制器。

## 十一、总结，Linux I2C 分层设计的核心思想

### 11.1 控制器驱动负责“怎么发”

Linux I2C 框架最底层的关键，是控制器驱动。

它负责把一个抽象的 I2C 传输请求，变成真实硬件动作。

怎么配置寄存器。

怎么处理 FIFO。

怎么等中断。

怎么处理 NACK。

怎么发 STOP。

这些都属于“怎么发”。

控制器驱动写得好，上层外设驱动就舒服。控制器驱动有坑，所有挂在这条总线上的设备都会跟着倒霉。

这也是为什么有些板子上“一堆 I2C 设备都不稳定”，最后发现不是外设问题，而是控制器驱动或者 pinctrl、clock 配置问题。

### 11.2 外设驱动负责“发什么”

外设驱动不该关心控制器寄存器。

它关心的是芯片业务。

写哪个寄存器。

读哪个寄存器。

初始化顺序是什么。

中断来了怎么处理。

数据怎么上报给内核子系统。

比如一个温度传感器驱动，它应该关心温度寄存器怎么解析，而不是关心 I2C 控制器怎么产生 START。

一个触摸屏驱动，它应该关心坐标数据怎么读取和上报，而不是关心 SCL 高电平保持多久。

这就是分层。

这也是代码**可移植**的基础。

### 11.3 I2C Core 负责连接控制器、设备和驱动

I2C Core 是中间的组织者。

它把 `adapter` 注册进来。

把设备树子节点变成 `client`。

把 `i2c_driver` 注册进设备模型。

把传输请求转发给底层 `adapter`。

它不直接面对业务，也不直接面对寄存器，但它决定整套框架能不能顺畅工作。

很多时候我们写驱动，只接触到 `i2c_driver` 和 `i2c_client`，容易忽略 I2C Core 的存在。但出问题的时候，你就会发现它很关键。

probe 为什么触发？

`client` 怎么来的？

`adapter` 怎么找？

transfer 怎么到底层？

答案都在 I2C Core 这条线上。

### 11.4 `Adapter`、`Client`、`Driver` 三者关系回顾

最后再把这三个对象拎出来。

`i2c_adapter` 是总线。

`i2c_client` 是设备。

`i2c_driver` 是驱动。

关系如下：

```
一条 I2C 总线
  -> 一个 `i2c_adapter`

总线上的一个外设
  -> 一个 `i2c_client`

能驱动某类外设的代码
  -> 一个 `i2c_driver`
```

通信路径：

```
`i2c_driver`
  -> `i2c_client`
  -> `i2c_adapter`
  -> `i2c_algorithm`
  -> `master_xfer`
  -> 控制器硬件
```

设备注册路径：

```
设备树控制器节点
  -> platform_driver probe
  -> `i2c_add_adapter`
  -> 解析 I2C 子设备节点
  -> 创建 `i2c_client`
  -> 匹配 `i2c_driver`
  -> 调用 probe
```

用户态路径：

```
/dev/i2c-x
  -> i2c-dev
  -> I2C Core
  -> `i2c_adapter`
  -> `master_xfer`
```

这几条线掌握了，Linux I2C 框架就不再是一堆散乱 API。

它是一张网。

### 11.5 Linux I2C 框架的**解耦、复用与可移植性**价值

Linux I2C 框架真正厉害的地方，不是封装了几个读写函数。

它厉害在**解耦**。

SoC 控制器驱动和外设驱动解耦。

设备描述和驱动代码解耦。

用户态调试和内核态正规驱动共用底层传输能力。

不同平台可以复用同一个外设驱动。

不同外设可以复用同一个 `adapter`。

这就是 Linux 驱动框架的味道。

一开始看，结构体多，路径长，感觉麻烦。

写多了以后你会发现，这些麻烦其实是在替你挡更大的麻烦。

如果没有这套框架，每个外设驱动都要自己适配不同 SoC 控制器，自己管总线锁，自己处理设备注册，自己暴露用户态接口。那才是真的乱。

所以学 Linux I2C，不要只停留在“我会调用`i2c_smbus_read_byte_data()`”这个层面。

你要知道 `client` 从哪来。

`adapter` 谁注册。

`driver` 怎么匹配。

transfer 怎么走到底层。

设备树如何参与。

用户态访问为什么能走同一条总线。

这些东西串起来之后，再看 I2C 驱动，你会轻松很多。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/53c3f897_1783350135137?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247488000%26idx%3D1%26sn%3Dcd97f46c68d6da0c246364377ec2fec9%26chksm%3Dc3cedcfdaea8092eedf9829138832cece8c4ca54a4b3beab9c70f7ae5856b84331a14e4c30e9%26mpshare%3D1%26scene%3D1%26srcid%3D07062tkWpkSjbhXej10pQM40%26sharer_shareinfo%3D565ad6ac67424d65d57f81e48ad45309%26sharer_shareinfo_first%3D565ad6ac67424d65d57f81e48ad45309%23rd&s=obsidian)