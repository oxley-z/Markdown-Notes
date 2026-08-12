---
author: LongWay
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUzMjk3OTYwOA==&mid=2247484636&idx=1&sn=2c7ded340f279e08958f14bf88da101c&chksm=fb1912a52eb1f01f9696f79d5910a1704b3f9fbc5271cb094c8258e24c4ab67fd067968987d6&mpshare=1&scene=1&srcid=0812IffxtKwxxHuFV64TQ7Ts&sharer_shareinfo=d5c1660368391acf9224d621a5e55416&sharer_shareinfo_first=d5c1660368391acf9224d621a5e55416#rd
saved: 2026-08-12 09:43:35
tags:
  - 笔记同步助手
id: 5258e723-4498-4584-ba8b-5f6df0b745ce
---

公众号名称：学嵌入式的长路

作者名称：LongWay

发布时间：2026-08-12 07:13

> LONGWAY EMBEDDED

启动文件只是编译器自动生成的一段汇编？

startup\_xxx.s 深度解析从 Reset 到 main

**startup.s**Reset\_Handler**向量表**.data**.bss**SystemInit

> 01
> 中断向量表：芯片的“通讯录”

> 02
> Reset\_Handler：一切从这里开始

> 03
> 弱定义：每个异常/中断都有“保底处理”

> 04
> 完整的开机时间线

> 05
> 常见陷阱与调试技巧

> 06
> 不同架构的启动对比

> 07
> 动手实验：改编启动文件

嵌入式开发者写了几百个 `main()`，那么有人会问：**MCU 上电后第一条指令执行的是什么？全局变量为什么能在 main 之前就赋好初值？** 答案全藏在那个不起眼的 `startup_xxx.s` 启动文件里。

启动文件是芯片厂商提供的一段汇编代码，负责把 CPU 从复位状态带到 C 语言 `main()` 环境。它不是“可选配置”——没有它，你的程序根本跑不起来。本文将带你逐段解析启动文件的每个环节，并画出一条完整的开机时间线。

01

### 中断向量表：芯片的“通讯录”

CHAPTER 01

启动文件的第一个“作品”就是中断向量表。以 ARM Cortex-M 为例，向量表放在 Flash 的首地址（0x08000000 或 0x00000000），格式固定：

> ARM ASSEMBLY

; startup\_stm32f407xx.s（简化）  
.section .isr\_vector, "a", %progbits  
.typeg\_pfnVectors, %object  
.sizeg\_pfnVectors, .-g\_pfnVectors  
g\_pfnVectors:  
.word\_estack; 0: 栈顶地址（SP 初始值）  
.wordReset\_Handler; 1: 复位向量（PC 初始值）  
.wordNMI\_Handler; 2: NMI 异常  
.wordHardFault\_Handler; 3: HardFault  
.wordMemManage\_Handler; 4: MemManage  
.wordBusFault\_Handler; 5: BusFault  
.wordUsageFault\_Handler; 6: UsageFault  
.word0, 0, 0, 0; 7-10: 保留  
.wordSVC\_Handler; 11: SVCall  
.wordDebugMon\_Handler; 12: DebugMon  
.word0; 13: 保留  
.wordPendSV\_Handler; 14: PendSV  
.wordSysTick\_Handler; 15: SysTick  
; 以下为外设中断（编号从 16 开始）  
.wordWWDG\_IRQHandler; Window WatchDog  
.wordPVD\_IRQHandler; PVD through EXTI  
; ... 每个外设一个表项

**Cortex-M 的独特之处：** 复位后硬件自动从地址 0x00000000 加载 SP，从 0x00000004 加载 PC，跳转到 `Reset_Handler`。这不同于传统 ARM（PC 从 0 开始执行指令），完全是硬件设计的魔法。

> 向量表的位置由 `__Vectors` 标号定义，链接脚本控制它放哪个 Section。如果你做 IAP（应用内编程），需要在 0x08000000 + N 偏移处放第二张向量表，并修改 SCB->VTOR 寄存器。

02

### Reset\_Handler：一切从这里开始

CHAPTER 02

`Reset_Handler` 是系统上电/复位后第一个执行的代码。它的工作分四步：

## 2.1 搬运 .data 段

C 语言中写了 `uint32_t counter = 100;` —— 这个 100 存在哪里？

• **编译时：**`100` 这个值放在 Flash 的 `LOADADDR(.data)` 区域（也叫 **LMA**，Load Memory Address）

• **运行时：**`counter` 变量在 SRAM 的 `.data` 段（也叫 **VMA**，Virtual Memory Address）

启动文件必须把 100 从 Flash 拷贝到 SRAM：

> ARM ASSEMBLY

Reset\_Handler:  
; 1. 复制 .data 段：Flash → SRAM  
ldrr0, =\_sidata; Flash 中 .data 的源地址（LMA）  
ldrr1, =\_sdata; SRAM 中 .data 的目标地址（VMA）  
ldrr2, =\_edata; SRAM 中 .data 的结束地址  
subsr2, r2, r1; 计算 .data 段大小  
ble .L\_loop\_data; 如果大小为 0，跳过  
.L\_copy\_data:  
ldrbr4, \[r0\], **#1 ; 从 Flash 读 1 字节**  
strbr4, \[r1\], **#1 ; 写入 SRAM**  
subsr2, r2, **#1 ; 计数减 1**  
bne .L\_copy\_data; 循环直到拷贝完  
.L\_loop\_data:

## 2.2 清零 .bss 段

未初始化或初始化为 0 的全局变量（`static int flag;`、`uint8_t buffer[1024];`）放在 `.bss` 段。SRAM 上电后内容是随机的，必须清零：

> ARM ASSEMBLY

; 2. 清零 .bss 段  
ldrr1, =\_sbss; .bss 起始地址  
ldrr2, =\_ebss; .bss 结束地址  
movsr3, **#0**  
subsr2, r2, r1  
ble .L\_loop\_bss  
.L\_zero\_bss:  
strbr3, \[r1\], **#1**  
subsr2, r2, **#1**  
bne .L\_zero\_bss  
.L\_loop\_bss:

> **⚠️ 字节搬运 vs 字搬运**：上面的示例按字节搬运，效率低。实际芯片的启动文件会用 `ldmia`/`stmia` 一次搬运 4 字（16 字节），速度提升 4 倍。但为了可读性，这里用字节方式展示。

## 2.3 调用 SystemInit()

C 环境还没准备好？不，现在可以调 C 函数了——SP 已设、RAM 已初始化：

> ARM ASSEMBLY

; 3. 系统时钟初始化  
blSystemInit

`SystemInit()` 是 CMSIS 标准定义的函数，由芯片厂商实现。它完成：

• 切换 HSI/HSE 振荡器

• 配置 PLL 锁相环倍频

• 设置 AHB/APB1/APB2 分频系数

• 配置 Flash 等待周期（适配高主频）

以 STM32F407 为例，`SystemInit()` 将 HSE 8MHz → PLL 倍频 168MHz（主频），并确保 APB1 ≤ 42MHz、APB2 ≤ 84MHz。

## 2.4 \_\_libc\_init\_array() 与全局构造函数

C++ 和 GNU C 中，你可以在全局作用域写构造函数（或 `__attribute__((constructor))`）：

> C

// 这个函数在 main() 之前自动执行！  
\_\_attribute\_\_((constructor))  
**void** early\_init(**void**) {  
init\_hardware\_early();  
}

`__libc_init_array()` 遍历 `.init_array` 段，逐一调用所有构造函数指针：

> ARM ASSEMBLY

; 4. 调用全局构造函数  
bl\_\_libc\_init\_array

对于纯 C 项目，`__libc_init_array` 也可能不存在——这时链接脚本里根本没有 `.init_array` 段。

## 2.5 跳转到 main()

最后：

> ARM ASSEMBLY

; 5. 进入主程序  
blmain  
; main 返回后执行（嵌入式场景基本不会返回）  
blexit

如果 `main()` 意外返回，`exit()` 会调用注册的 `atexit` 函数，然后死循环。

03

### 弱定义：每个异常/中断都有“保底处理”

CHAPTER 03

启动文件末尾通常会为所有中断提供一个**弱定义（weak）**的默认处理函数：

> ARM ASSEMBLY

.weakHardFault\_Handler  
.typeHardFault\_Handler, %function  
HardFault\_Handler:  
b . ; 死循环

`weak` 属性意味着：如果用户在 C 代码中写了同名的 `HardFault_Handler()`，链接器优先选用户的；如果没写，就链接这个默认的死循环版本。这种设计让初学者不用为每个中断写 handler，也能保证程序不会跑飞。

04

### 完整的开机时间线

CHAPTER 04

从按下复位到 `main()` 第一条语句，完整的流程：

> TEXT

时间 事件 说明  
─── ──── ────  
T+0CPU 复位释放 硬件从 0x0000 加载 SP, PC  
T+1Reset\_Handler 开始 硬件跳入启动文件的汇编入口  
T+2 关闭全局中断 确保初始化不受中断干扰  
T+3 搬运 .data 从 Flash 拷贝到 SRAM  
T+4 清零 .bss 初始化全局/静态零值变量  
T+5 调用 SystemInit() 配置时钟树（HSE→PLL→168MHz）  
T+6 调用 \_\_libc\_init\_array() 执行全局构造函数  
T+7 跳转 main() C 世界正式开始  
├─ 硬件初始化 HAL\_Init(), MX\_GPIO\_Init() 等  
├─ 外设配置 USART, SPI, I2C, Timer...  
└─ **while**(1){...} 主循环

总耗时：在 168MHz STM32F4 上约 **200-500 μs**（取决于 .data/.bss 大小和 Flash 读取速度）。

05

### 常见陷阱与调试技巧

CHAPTER 05

## 5.1 栈溢出在 main 之前

如果 `_estack` 指向的地址超过实际 SRAM 范围，或者在 .data 拷贝前就调用了函数（压栈），程序会在 `SystemInit()` 里面 HardFault。**症状：** 调试器看到 `Reset_Handler` 执行后就进异常。

**排查法：** 检查链接脚本中 `_estack` 计算是否正确（通常是 SRAM 起始 + 大小）。

## 5.2 .data 段复制方向反了

常见笔误：把 `_sidata`（Flash 源）和 `_sdata`（RAM 目标）搞反。结果从 SRAM 向 Flash 拷贝，读出来全是垃圾。**症状：** 全局变量的初值全是乱码。

## 5.3 VTOR 没改导致 IAP 失败

做了 Bootloader + App 分区，应用区放在 0x08010000，但向量表还在 0x08000000。中断触发时 CPU 读到的还是 Bootloader 的向量表。

**解决：** 在 `SystemInit()` 或 `Reset_Handler` 早期加入：

> C

SCB\->VTOR = 0x08010000; // 重定向向量表到 App 分区

## 5.4 .bss 清零过大导致延时可见

大数组（如 LCD 显存 `uint16_t lcd_buffer[320*240];`）在 .bss 段。400μs 的启动延迟在某些场景（如快速重连外设）不可接受。一个做法是把大数组放进独立的 `.noinit` 段，由程序员手动初始化。

06

### 不同架构的启动对比

CHAPTER 06

| 架构 | 向量表位置 | 启动方式 | 关键差异 |
| --- | --- | --- | --- |
| Cortex-M (STM32) | Flash 首地址 | 硬件加载 SP+PC | 无需汇编跳转，向量表包含栈顶 |
| Cortex-A (i.MX6ULL) | 固定地址 | ROM Code 加载 SPL | 多级 boot，u-boot SPL 负责 DDR 初始化 |
| RISC-V (GD32V) | 可配置 mtvec | 从 0 取第一条指令 | 无统一标准，ECLIC/CLIC 不同 |
| ESP32 (Xtensa) | ROM + Flash | 一级 Bootloader 加载 | 两级 boot + flash cache 初始化 |

07

### 动手实验：改编启动文件

CHAPTER 07

克隆一份官方启动文件，自己动手改一下，理解会更深刻：

## 1\. 在 .data 拷贝前后插入 GPIO 翻转——用示波器量拷贝耗时

## 2\. 强行删掉 \_\_libc\_init\_array 调用，看全局构造函数是否还执行

## 3\. 手动修改 \_estack 为非法地址，观察芯片行为

🏷️ 标签：**#启动文件****#startup.s****#Reset\_Handler****#中断向量表****#.data段搬运**#.bss清零**#SystemInit****#\_\_libc\_init\_array****#VTOR****#链接脚本****#Cortex\-M****#ARM汇编****#嵌入式底层**

> 我是 **学嵌入式的长路**，用代码和思考陪你走通嵌入式的每一公里。
> 喜欢这类嵌入式技术复盘，可以点个在看；想看某个方向，也欢迎在评论区告诉我。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/171ce069_1786499013597?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUzMjk3OTYwOA%3D%3D%26mid%3D2247484636%26idx%3D1%26sn%3D2c7ded340f279e08958f14bf88da101c%26chksm%3Dfb1912a52eb1f01f9696f79d5910a1704b3f9fbc5271cb094c8258e24c4ab67fd067968987d6%26mpshare%3D1%26scene%3D1%26srcid%3D0812IffxtKwxxHuFV64TQ7Ts%26sharer_shareinfo%3Dd5c1660368391acf9224d621a5e55416%26sharer_shareinfo_first%3Dd5c1660368391acf9224d621a5e55416%23rd&s=obsidian)