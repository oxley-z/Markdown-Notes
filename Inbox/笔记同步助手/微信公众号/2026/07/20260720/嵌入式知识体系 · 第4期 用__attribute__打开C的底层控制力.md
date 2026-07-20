---
author: LongWay
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUzMjk3OTYwOA==&mid=2247484534&idx=1&sn=f73fb932e8797cc78de776968ab4d64b&chksm=fbafbae2e48dc46d1de8ea9091f243a9e0e5956dfdd09b0a8ca4230d9d87493c7985da43fa5b&mpshare=1&scene=1&srcid=07208XoOgfLkPI7CUCHLSZP1&sharer_shareinfo=c91a17a8c16cb481a365132970a4110e&sharer_shareinfo_first=c91a17a8c16cb481a365132970a4110e#rd
saved: 2026-07-20 07:56:58
tags:
  - 笔记同步助手
id: 0660aba3-4acd-4cd5-8a49-4ca5d6fa7f02
---

公众号名称：学嵌入式的长路

作者名称：LongWay

发布时间：2026-07-20 07:11

> LONGWAY EMBEDDED

标准 C 不够用？

用 \_\_attribute\_\_

打开 C 的底层控制力

\_\_attribute\_\_GCC Extensionsectionweakpacked/aligned

> 01
> section：把变量/函数放到指定内存段

> 02
> aligned / packed：精细控制对齐

> 03
> weak：弱符号实现默认函数与重写

> 04
> constructor / destructor：main 前后的自动执行

> 05
> 其他高频 \_\_attribute\_\_

> 06
> 综合实战：构建模块化初始化框架

> 07
> 注意事项与最佳实践

> 08
> 总结

假设你在调试一个诡异的启动问题——程序还没进 main() 就卡死了。你用 JTAG 连上去，发现 PC 停在某个函数里。但代码里没人调过它。

又或者，你希望在 HAL 库中提供一个**默认的 GPIO 初始化函数**，用户如果没写自己的版本，就自动用默认的——但 C 语言没有“弱符号”的概念。

再或者，你想把一段初始化代码放到某个特殊内存段，比如 DTCM 或紧耦合内存（TCM），在链接脚本里精细控制它的位置。

这些需求，**标准 C 语言**一个都满足不了。但 GCC 的 \_\_attribute\_\_ 全部能搞定。

\_\_attribute\_\_ 是 GCC 提供的编译器扩展机制，用 \_\_attribute\_\_((属性名)) 的语法给变量、函数、类型添加**编译期元信息**。在嵌入式开发中，它几乎是不可或缺的瑞士军刀。

01

### section：把变量/函数放到指定内存段

Section: Place Symbols in Custom Memory Sections

链接脚本把代码分成 .text（代码）、.data（初始化的全局变量）、.bss（未初始化的全局变量）等段。默认情况下，所有函数都进 .text，所有全局变量都根据初始化情况进 .data 或 .bss。

\_\_attribute\_\_((section("name"))) 让你**自定义段名**，把特定变量或函数放到指定的内存区域。

场景 1：把数据放到特定 RAM 区域

某些 MCU 有片内 SRAM 和紧耦合内存（TCM），TCM 访问延迟更低。把关键变量放到 TCM：

> GitHub Dark · C

// 将中断频繁访问的变量放到 TCM

\#defineITCM\_RAM\_\_attribute\_\_((section(".itcm\_ram")))

ITCM\_RAMvolatileuint32\_tsystem\_tick;

ITCM\_RAMstructcan\_rx\_bufrx\_buffers\[16\];

然后在链接脚本中定义 .itcm\_ram 段映射到 TCM 地址范围。**不需要修改链接脚本就无法匹配**——但如果链接脚本不包含这个段，链接会报错，确保你意识到需要适配。

场景 2：构建初始化函数表

在 RTOS 或框架代码中，经常需要让各个模块的初始化函数自动注册到一个表里，然后统一遍历调用：

> GitHub Dark · C

// 定义一个初始化函数指针类型

typedefvoid(\*init\_fn\_t)(void);

// 定义一个段标记

\#defineINIT\_PRIO(n)\_\_attribute\_\_((section(".init\_call."#n)))

// 各模块注册初始化函数

voidspi\_init(void)INIT\_PRIO(1);

voidi2c\_init(void)INIT\_PRIO(2);

voiduart\_init(void)INIT\_PRIO(3);

// 链接器会把这些函数指针按优先级顺序排列在 .init\_call 段

// 然后统一遍历：

externinit\_fn\_t\_\_init\_call\_start\[\];

externinit\_fn\_t\_\_init\_call\_end\[\];

voidrun\_init\_chain(void){

for(init\_fn\_t\*fn\=\_\_init\_call\_start;fn<\_\_init\_call\_end;fn++){

(\*fn)();

}

}

这种模式在 FreeRTOS 的 CreateThread() 中、Linux 内核的 \_\_initcall 中广泛使用。核心思路是：**用编译期的段分配 + 链接脚本的排序，替代运行时的全局数组维护。**

场景 3：把版本信息固化到固定地址

把固件版本字符串放到固定的 Flash 地址，上位机可以通过读取该地址来判断固件版本：

> GitHub Dark · C

\_\_attribute\_\_((section(".fw\_version")))

constcharfirmware\_version\[\]\="v2.1.0-build20260714";

02

### aligned / packed：精细控制对齐

aligned / packed: Fine-Grained Alignment Control

aligned(n)：提升对齐要求

之前讲到内存对齐时，关注的是自然对齐的**最低要求**。\_\_attribute\_\_((aligned(n))) 可以**提升**对齐等级，适合以下场景：

> GitHub Dark · C

// 让缓冲区按 32 字节对齐（DMA 优化、Cache Line 对齐）

uint8\_tdma\_buffer\[1024\]\_\_attribute\_\_((aligned(32)));

// 让结构体整体按 16 字节对齐

struct\_\_attribute\_\_((aligned(16)))matrix{

floatdata\[4\]\[4\];

};

Cortex-M7 的 Cache Line 通常是 32 字节，DMA 缓冲区如果跨越 Cache Line 边界，会引发一致性问题。用 aligned(32) 确保缓冲区首地址对齐，是解决这类问题的最简单手段。

packed：取消填充

\_\_attribute\_\_((packed)) 在第 \#3 篇已经详细讲过，这里只做总结：

> GitHub Dark · C

struct\_\_attribute\_\_((packed))can\_frame{

uint16\_tid;// offset 0

uint8\_tdlc;// offset 2

uint8\_tdata\[8\];// offset 3

};// sizeof = 11（而不是正常对齐的 12）

适用于**协议解析**和**存储格式**，但要注意非对齐访问的性能代价。

> TECH NOTE
> **对比：**aligned(n)是往上加的（要求更严格），packed 是往下去掉的（要求更宽松）。两者可以组合：

> GitHub Dark · C

struct\_\_attribute\_\_((packed,aligned(4))){

uint8\_ta;

uint32\_tb;

};// 内部不填充，但整体按 4 字节对齐

03

### weak：弱符号实现默认函数与重写

weak: Default Implementations and Overrides

这是嵌入式开发中**最有用**的 \_\_attribute\_\_ 之一。

问题

在标准 C 中，如果定义了两个相同名称的函数，链接器会报 “multiple definition” 错误。但在很多框架中，我们希望：**提供一个默认实现，允许用户在自己的代码中覆盖它。**

这就是弱符号（weak symbol）的用武之地。

语法

> GitHub Dark · C

// 弱符号定义：提供一个默认实现

\_\_attribute\_\_((weak))voidHAL\_GPIO\_Init(void){

// 默认初始化：全部设为推挽输出

GPIOA\-\>CRL\=0x33333333;

GPIOA\-\>CRH\=0x33333333;

}

// 用户在自己的代码中定义一个同名函数（强符号）

voidHAL\_GPIO\_Init(void){

// 自定义初始化

RCC\-\>APB2ENR|\=RCC\_APB2ENR\_IOPAEN;

GPIOA\-\>CRL\=0x44444444;// 开漏输出

}

// 链接器优先选择强符号。如果用户定义了，弱符号被忽略。

嵌入式中 weak 的经典用法

## HAL 库的中断回调：

> GitHub Dark · C

// HAL 库提供的弱符号回调

\_\_attribute\_\_((weak))voidHAL\_UART\_RxCpltCallback(UART\_HandleTypeDef\*huart){

// 什么都不做——用户可以自行实现

}

// 用户代码中实现

voidHAL\_UART\_RxCpltCallback(UART\_HandleTypeDef\*huart){

// 处理接收完成事件

rx\_buffer\[rx\_index++\]\=huart\-\>Instance\-\>DR;

}

STM32 HAL 库中几乎所有回调函数都是 weak 的。用户只需要定义同名的函数，链接器自动忽略弱符号版本——**不需要修改库代码，不需要条件编译宏**，比 \#ifdef 方案优雅得多。

## RTOS 的默认钩子函数：

> GitHub Dark · C

\_\_attribute\_\_((weak))voidvApplicationStackOverflowHook(TaskHandle\_txTask,char\*pcTaskName){

// 默认死循环——至少让你知道栈溢出了

for(;;);

}

\_\_attribute\_\_((weak))voidvApplicationIdleHook(void){

// 默认什么都不做

}

weak 的工作原理

> CHECKLIST

## 1.编译器给弱符号打上特殊标记

## 2.链接器在解析符号时，强符号优先于弱符号

## 3.如果同一个符号有多个弱定义，链接器随机选取一个（所以 .o 文件的链接顺序会影响结果）

## 4.如果弱符号没有被替换，它正常工作；如果被替换，弱符号被丢弃

> PITFALL
> ⚠️ **注意：** weak 必须用在**函数定义**或**变量定义**上（分配存储空间），不能用在声明（extern）上。extern \_\_attribute\_\_((weak)) 只是声明一个可能存在也可能不存在的符号——如果没人提供就会链接失败。

04

### constructor / destructor：main 前后的自动执行

constructor / destructor: Hooks Around main

这两个属性让函数在 main() 之前或 exit() 之后自动执行。

基本用法

> GitHub Dark · C

\_\_attribute\_\_((constructor))voidbefore\_main(void){

// 在 main() 之前自动执行

init\_hardware();

init\_heap();

}

\_\_attribute\_\_((destructor))voidafter\_main(void){

// 在 exit() 或 main() 返回后自动执行

cleanup\_resources();

}

执行顺序控制

可以给 constructor 指定优先级（数值越小越靠前）：

> GitHub Dark · C

\_\_attribute\_\_((constructor(101)))voidearly\_init(void){

// 最先执行

}

\_\_attribute\_\_((constructor(102)))voidmid\_init(void){

// 第二个执行

}

\_\_attribute\_\_((constructor(103)))voidlate\_init(void){

// 第三个执行

}

优先级范围一般用 **101～65535**，0～100 保留给编译器内部使用。

嵌入式中不要过度依赖 constructor

在资源受限的 MCU 上，constructor 的使用场景有限，原因有三：

> CHECKLIST

1.**依赖 C 运行时初始化**：constructor 由 \_\_libc\_init\_array() 遍历段中的函数指针来调用。如果程序不用 C 标准库（比如裸机 \-nostdlib），这段代码可能不存在

2.**启动顺序不可控**：在不同 .c 文件中定义的 constructor 函数，执行顺序取决于链接器排列的 .init\_array 段顺序，不易预测

## 3.替代方案更可靠：在启动文件（startup\_xxx.s）或系统初始化函数中显式调用的方案更可控

> TECH NOTE
> **推荐做法：** 在 main() 入口处显式调用 SystemInit() + \_\_libc\_init\_array() + 各模块初始化函数。constructor 更适合**PC 端测试代码**或**不需要极端可靠性的场景**。

05

### 其他高频 \_\_attribute\_\_

Other High-Frequency \_\_attribute\_\_ Extensions

used：防止编译器优化掉“没用”的变量

编译器会在 \-Os 优化下移除没有任何引用的变量。但如果你希望变量**因为链接脚本的段定义而被链接器保留**，需要用 used 标记：

> GitHub Dark · C

// 基于 section 的初始化表：这个函数没人直接调用，但不能被优化掉

staticvoid\_\_attribute\_\_((used,section(".init\_call.1")))spi\_pins\_init(void){

// ...

}

always\_inline：强制内联

inline 关键字只是**建议**编译器内联。\_\_attribute\_\_((always\_inline))强制内联：

> GitHub Dark · C

staticinline\_\_attribute\_\_((always\_inline))uint32\_tread\_reg(volatileuint32\_t\*addr){

return\*addr;

}

在 MCU 上，高频调用的关键路径（如中断服务程序中的位操作）适合强制内联，避免函数调用开销。但要注意**代码膨胀**——内联后的代码会被复制到每个调用点。

naked：裸函数（无编译器生成的序言/尾声）

中断服务程序在某些编译器上要求用 naked 来避免编译器干扰栈帧：

> GitHub Dark · C

\_\_attribute\_\_((naked))voidSysTick\_Handler(void){

\_\_asmvolatile(

"push {r4-r11, lr}\\n\\t"

"bl systick\_isr\\n\\t"

"pop {r4-r11, pc}\\n\\t"

);

}

> PITFALL
> ⚠️ naked 函数中**不能有局部变量和函数调用**（除了内联汇编中的调用），因为编译器不会帮你管理栈帧。

unused：消除“未使用”警告

> GitHub Dark · C

staticvoid\_\_attribute\_\_((unused))debug\_print(constchar\*msg){

// 暂时不用，但不删

}

适合那些在调试阶段保留、但在发布版本中可能不用到的函数。

06

### 综合实战：构建模块化初始化框架

Practical Build: A Modular Init Framework

下面组合 section + used + constructor 实现一个**松耦合的模块注册系统**，它可以替代手动维护全局初始化列表。

> GitHub Dark · C

/\*module\_init.h\*/

\#ifndefMODULE\_INIT\_H

\#defineMODULE\_INIT\_H

typedefvoid(\*module\_init\_fn\_t)(void);

// 模块注册宏：按优先级把初始化函数放入 .module\_init 段

\#defineMODULE\_INIT(priority)\\

staticvoidmodule\_init\_##\_\_LINE\_\_(void);\\

staticmodule\_init\_fn\_t\_\_attribute\_\_((used,section(".module\_init."#priority)))\\

\_\_module\_init\_ptr\_##\_\_LINE\_\_\=module\_init\_##\_\_LINE\_\_;\\

staticvoidmodule\_init\_##\_\_LINE\_\_(void)

/\*main.c\*/

\#include"module\_init.h"

// 模块 A：优先级 1（最先）

MODULE\_INIT(1){

// 初始化 GPIO、时钟等基础外设

RCC\-\>APB2ENR\=0xFFFFFFFF;

}

// 模块 B：优先级 2

MODULE\_INIT(2){

// 初始化 UART

USART1\-\>CR1\=USART\_CR1\_UE;

}

// 模块 C：优先级 3

MODULE\_INIT(3){

// 初始化 I2C

I2C1\-\>CR1\=I2C\_CR1\_PE;

}

// 在 main() 中统一执行

externmodule\_init\_fn\_t\_\_module\_init\_start\[\];

externmodule\_init\_fn\_t\_\_module\_init\_end\[\];

intmain(void){

for(module\_init\_fn\_t\*fn\=\_\_module\_init\_start;fn<\_\_module\_init\_end;fn++){

(\*fn)();

}

// ... 主循环

}

链接脚本中需要添加：

> Linker Script

SECTIONS{

.module\_init:{

\_\_module\_init\_start\=.;

KEEP(\*(.module\_init.\*))

\_\_module\_init\_end\=.;

}\>FLASH

}

## 这样做的优势：

> CHECKLIST

✓✅ **松耦合**：每个模块只需要包含头文件，不需要手动注册

✓✅ **自动排序**：优先级数字决定执行顺序

✓✅ **零运行时开销**：初始化的遍历是线性，没有链表管理开销

✓✅ **编译时检查**：函数签名不匹配会在编译时报错

07

### 注意事项与最佳实践

Notes and Best Practices

## 1\. 可移植性

\_\_attribute\_\_ 是 GCC 扩展，**不是 C 标准的一部分**。如果要跨编译器（ARMCC、IAR、Clang），需要用宏抽象：

> GitHub Dark · C

\#ifdef\_\_GNUC\_\_

\#defineWEAK\_\_attribute\_\_((weak))

\#definePACKED\_\_attribute\_\_((packed))

\#defineSECTION(x)\_\_attribute\_\_((section(x)))

\#else

\#ifdef\_\_ICCARM\_\_

\#defineWEAK\_\_weak

\#definePACKED\_\_packed

\#defineSECTION(x)@x// IAR 语法

\#endif

\#endif

## 2\. 多用宏封装

直接写 \_\_attribute\_\_ 会降低代码可读性。建议封装成有意义的名字：

> GitHub Dark · C

// 好的做法

\#defineRAM\_FUNC\_\_attribute\_\_((section(".ram\_functions"),long\_call))

\#defineIN\_ITCM\_\_attribute\_\_((section(".itcm\_code")))

\#defineALIGN\_32\_\_attribute\_\_((aligned(32)))

\#defineWEAK\_\_attribute\_\_((weak))

\#defineUSED\_\_attribute\_\_((used))

\#definePACKED\_\_attribute\_\_((packed))

## 3\. 理解链接脚本

section 属性的能力上限取决于链接脚本。**不了解链接脚本，section 属性只能发挥一半功力。** 建议在写 section 属性前，至少读过一遍 MCU 的 .ld 文件。

## 4\. 推荐还是谨慎

> ATTRIBUTE MATRIX

属性

推荐指数

说明

packed

⭐⭐⭐

协议解析必备，注意非对齐代价

aligned

⭐⭐⭐

DMA/Cache 场景必用

weak

⭐⭐⭐⭐⭐

HAL 库回调、框架扩展，最实用的属性

section

⭐⭐⭐⭐

高端用法，构建初始化链的利器

constructor

⭐⭐

裸机 MCU 上慎用，PC 调试可用

used

⭐⭐⭐

配合 section 属性使用

always\_inline

⭐⭐⭐

关键路径优化，注意代码膨胀

naked

⭐⭐

高级用户专用，新手容易踩坑

EOF

### 总结

Summary

\_\_attribute\_\_是 C 语言通往底层控制能力的钥匙：

> CHECKLIST

✓**\`section\`** 让你操控变量的物理位置，上对链接脚本，下接分散加载

✓**\`packed\` / \`aligned\`** 让你精细掌控内存布局

✓**\`weak\`** 是框架设计的利器，是 HAL 库可扩展性的基石

✓**\`constructor\` / \`destructor\`**提供了执行时序的钩子

✓**\`used\` / \`always\_inline\` / \`naked\`**等小工具在特定场景下不可或缺

这些能力都不是 C 标准提供的，但 GCC 经过几十年的打磨，这些扩展已经成了事实标准——ARMCC、IAR 的对应语法几乎都是等价的。掌握它们，你的 C 代码就有了“编译期元编程”的能力，许多复杂的设计模式变得举重若轻。

> NEXT ISSUECOMING UP

> 下期预告
> 顺理成章——宏的进阶技巧里，## 拼接和 X-Macro 的魔法能让代码生成自动化，继续深化“用编译器帮你写代码”的主题。

> 我是 **学嵌入式的长路**，用代码和思考陪你走通嵌入式的每一公里。
> 喜欢这类嵌入式技术复盘，可以点个在看；想看某个方向，也欢迎在评论区告诉我。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/d372cd8b_1784505416327?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUzMjk3OTYwOA%3D%3D%26mid%3D2247484534%26idx%3D1%26sn%3Df73fb932e8797cc78de776968ab4d64b%26chksm%3Dfbafbae2e48dc46d1de8ea9091f243a9e0e5956dfdd09b0a8ca4230d9d87493c7985da43fa5b%26mpshare%3D1%26scene%3D1%26srcid%3D07208XoOgfLkPI7CUCHLSZP1%26sharer_shareinfo%3Dc91a17a8c16cb481a365132970a4110e%26sharer_shareinfo_first%3Dc91a17a8c16cb481a365132970a4110e%23rd&s=obsidian)