---
author: LongWay
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUzMjk3OTYwOA==&mid=2247484583&idx=1&sn=1c8fe77540f0208872cdb454d4503d69&chksm=fb9b699cd9849d932de55446b3cbb35290775beadf6cc0c91d2310c61c8833c0f7a565c9140a&mpshare=1&scene=1&srcid=0806pTYcWn2mvu0Zs7Pd51SU&sharer_shareinfo=e634427b24beb365072c92728276f71e&sharer_shareinfo_first=e634427b24beb365072c92728276f71e#rd
saved: 2026-08-06 23:40:05
tags:
  - 笔记同步助手
id: 1f2ffd9f-60ee-431f-aba3-d6f0940efcb4
---

公众号名称：学嵌入式的长路

作者名称：LongWay

发布时间：2026-08-06 22:49

> LONGWAY EMBEDDED

深入理解 · ARM 汇编基础

ARM 汇编基础

技术要点

saved\_pcLDR/STR/ADD/SUB/B/BLXPCprintf

> 01
> ARM vs Thumb vs Thumb-2 指令集演进

> 02
> AAPCS 调用约定的寄存器分工

> 03
> 栈帧结构与函数序言/尾声

> 04
> 异常入口的汇编处理

> 05
> 小结

很多嵌入式工程师对汇编的态度是：「这辈子不会手写，但反汇编一定要能看懂。」

这个态度很务实——实际工作中，你很少需要从头写汇编文件，但分析 crash dump、优化临界区、理解启动代码、debug 反汇编窗口——这些场景几乎每周都遇到。如果看不懂 ARM 汇编，你就像在迷雾中做调试。

这篇从指令集演进讲到调用约定，目标是：**下次看反汇编或 crash dump 时，你能认出每条指令在做什么，以及栈上的数据是怎么组织的。**

01

### ARM vs Thumb vs Thumb-2 指令集演进

Technical Notes

ARM 指令集经历了几次大的架构变更，理解这些变更是看懂反汇编的前提：

> ATTRIBUTE MATRIX

指令集

指令宽度

出现于

特点

ARM

32-bit

ARMv4+

功能齐全，但代码密度低

Thumb

16-bit

ARMv4T

压缩指令，密度高，功能受限

Thumb-2

16/32-bit 混编

ARMv6T2+

兼顾密度和功能，Cortex-M 标配

**关键转折点：Cortex-M 系列只支持 Thumb/Thumb-2，不支持 ARM 指令集。** 如果你看到 LDR R0, =0x40020000 这样的指令以为自己不认识——别紧张，它只是一个被汇编器转成 PC 相对加载的伪指令。

看懂 Thumb-2 的几个高频指令比全部背下来更实用：

> GitHub Dark · C

@数据传输类

LDRR0,\[R1,#4\]@从R1+4加载32位到R0

STRR0,\[R1\]@将R0写入R1指向的地址

LDRBR0,\[R1\]@加载1个字节（零扩展）

STRBR0,\[R1\]@存储1个字节

LDMIAR0!,{R1\-R4}@从R0加载多个寄存器，R0自增

STMIAR0!,{R1\-R4}@存储多个寄存器到R0指向的内存

@算术逻辑类

ADDR0,R1,R2@R0\=R1+R2

SUBR0,R1,#1@R0\=R1\-1

MOVR0,#0xFF@R0\=0xFF

ANDR0,R1,R2@R0\=R1&R2

ORRR0,R1,R2@R0\=R1|R2

LSLR0,R1,#2@R0\=R1<<2(逻辑左移)

LSRR0,R1,#2@R0\=R1\>\>2(逻辑右移)

@分支跳转类

Blabel@无条件跳转（±2KB范围）

BLfunc@带链接跳转（用于函数调用）

BXLR@返回（跳转到LR中的地址）

CBZR0,label@如果R0\=\=0则跳转

CBNZR0,label@如果R0!\=0则跳转

一个快速识别技巧：**看指令的第一个数字**——16-bit Thumb 指令的操作码通常短于 32-bit Thumb-2 指令。Cortex-M 反汇编时，指令地址是 2 的倍数（Thumb 地址的 bit\[0\] 固定为 1），但指令本身可能是 2 字节或 4 字节。

02

### AAPCS 调用约定的寄存器分工

Technical Notes

AAPCS（ARM Architecture Procedure Call Standard）定义了函数调用时寄存器的职责分工。这是理解反汇编的基石：

> ATTRIBUTE MATRIX

寄存器

别名

功能

调用者保存？

R0-R3

a1-a4

参数传递 / 返回值

✅ 调用者保存

R4-R11

v1-v8

局部变量

❌ 被调用者保存

R12

IP

内部调用暂存（intra-procedure-call scratch）

✅ 不保存

R13

SP

栈指针

永远不参与普通传参

R14

LR

链接寄存器（保存返回地址）

被调用者保存

R15

PC

程序计数器

—

**规则速记：**

> CHECKLIST

✓**前 4 个参数**放在 R0～R3。第 5 个参数开始压栈。

✓**返回值**在 R0（32 位）或 R0+R1（64 位）。

✓**R4～R11**由被调用函数负责保存。如果一个函数要用 R4，必须先在函数入口处压栈，退出时恢复。

✓**R12 (IP)**临时寄存器，函数调用过程中可能会被链接器生成的 veneer 代码覆盖——不要在函数调用之间期望它的值不变。

看一个实际的函数调用反汇编：

> GitHub Dark · C

// C 代码

intadd\_four(inta,intb,intc,intd,inte){

returna+b+c+d+e;

}

> GitHub Dark · C

@反汇编之后的ARM代码（示意）

add\_four:

@R0\=a,R1\=b,R2\=c,R3\=d,(e在栈上)

ADDR0,R0,R1@R0\=a+b

ADDR0,R0,R2@R0+\=c

ADDR0,R0,R3@R0+\=d

LDRR3,\[SP,#0\]@从栈取第五个参数e

ADDR0,R0,R3@R0+\=e

BXLR@返回，返回值在R0

03

### 栈帧结构与函数序言/尾声

Technical Notes

当一个函数使用超过 4 个局部变量、或者需要调用其他函数（需要使用 LR），编译器会在栈上开辟一块区域——**栈帧**。

> GitHub Dark · C

@典型的Cortex\-M函数序言(prologue)

my\_function:

PUSH{R4\-R7,LR}@保存被调用者保存的寄存器+LR

SUBSP,SP,#32@为局部变量分配32字节

@...函数体...

@典型的函数尾声(epilogue)

ADDSP,SP,#32@回收局部变量空间

POP{R4\-R7,PC}@恢复寄存器并返回

@注意：POP{...,PC}等价于恢复LR后再BXLR

@Cortex\-M硬件直接支持POP到PC并完成返回

**分析 crash dump 时的关键思维框架：**

> GitHub Dark · C

当前SP→\[局部变量空间\]←SP+偏移

\[R7(备份)\]

\[R6(备份)\]

\[R5(备份)\]

\[R4(备份)\]

\[LR(返回地址)\]←函数返回时POP到PC

上层SP→...

当发生 HardFault 时，MSP/PSP 指向的栈帧中存储了发生异常前的 R0-R3、R12、LR、PC、xPSR。**查看 \`PC\` 的备份值（通常叫 \`saved\_pc\`）**，就到了发生异常的指令地址，再对照反汇编清单定位具体代码行。

04

### 异常入口的汇编处理

Technical Notes

Cortex-M 的异常处理是硬件自动完成的——这比传统 ARM7/ARM9 手动压栈省很多事。但启动代码和中断处理函数仍然需要一些汇编包装。

**硬件自动保存的寄存器（入异常时）**：xPSR, PC, LR, R12, R3, R2, R1, R0——共 8 个 32 位寄存器，32 字节。CPU 自动压入当前栈（MSP 或 PSP）。

> GitHub Dark · C

@Cortex\-M的PendSV处理——RTOS上下文切换的核心

@典型的PendSV\_Handler实现

PendSV\_Handler:

@关闭中断，防止保存时被打断

CPSIDI

@检查是否发生了嵌套

MRSR0,PSP@获取当前PSP

CBZR0,skip\_save@PSP\=0表示第一次切换

@保存当前任务的寄存器

STMDBR0!,{R4\-R11}@保存R4～R11到任务栈

@更新任务的栈指针

LDRR1,\=current\_tcb

STRR0,\[R1\]@TCB\-\>sp\=R0

skip\_save:

@切换到下一个任务

LDRR1,\=next\_tcb

LDRR2,\[R1\]

LDRR0,\[R2\]@新任务的栈指针

@恢复新任务的寄存器

LDMIAR0!,{R4\-R11}@弹出R4～R11

MSRPSP,R0@更新PSP

@恢复中断

CPSIEI

@异常返回——硬件自动恢复R0\-R3,R12,LR,PC,xPSR

BXLR

这段汇编是 FreeRTOS、RT-Thread、uCOS 等 RTOS 上下文切换的核心。看懂它需要三样东西：**理解 AAPCS 的寄存器分工、知道硬件自动压栈的 8 个寄存器、熟悉多寄存器加载/存储指令（LDMIA/STMDB）。**

如果你能完整解释 STMDB R0!, {R4-R11} 和 LDMIA R0!, {R4-R11} 做了什么，以及为什么不需要保存 R0-R3——说明你已经掌握了 ARM 汇编在嵌入式领域的核心应用场景。

EOF

### 小结

Technical Notes

ARM 汇编不必畏惧。把它当成一门有规律的符号语言来理解：**R0-R3 传参数、R4-R11 存变量、R13 管栈、R14 记返回地址、R15 指下一条指令。** Thumb-2 混编指令看似复杂，但反汇编中 80% 的场景只是 LDR/STR/ADD/SUB/B/BLX 这几种。下次 HardFault 找不出原因时，与其在 C 代码里加九个 printf，不如打开 .lst 文件找那条 saved\_pc 对应的指令——答案通常就在那里。

> TECH NOTE
> 🏷️ ARM汇编 Thumb Thumb\-2 AAPCS 栈帧 Cortex-M 异常处理

> 我是 **学嵌入式的长路**，用代码和思考陪你走通嵌入式的每一公里。
> 喜欢这类嵌入式技术复盘，可以点个在看；想看某个方向，也欢迎在评论区告诉我。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/15834c57_1786030803480?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUzMjk3OTYwOA%3D%3D%26mid%3D2247484583%26idx%3D1%26sn%3D1c8fe77540f0208872cdb454d4503d69%26chksm%3Dfb9b699cd9849d932de55446b3cbb35290775beadf6cc0c91d2310c61c8833c0f7a565c9140a%26mpshare%3D1%26scene%3D1%26srcid%3D0806pTYcWn2mvu0Zs7Pd51SU%26sharer_shareinfo%3De634427b24beb365072c92728276f71e%26sharer_shareinfo_first%3De634427b24beb365072c92728276f71e%23rd&s=obsidian)