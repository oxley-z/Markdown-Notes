---
author: LongWay
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUzMjk3OTYwOA==&mid=2247484537&idx=1&sn=62deb9943f05f6f3ad4e6afd4622f22a&chksm=fb9d6ca850a185d731af0531ad82484f1962dbe14c947d8b77ba5d9353518462b990a301e4e2&mpshare=1&scene=1&srcid=0722rz6bXqAa2xZaa7a9Rpqi&sharer_shareinfo=8e6057a42caca29e5647b72f78ff7a36&sharer_shareinfo_first=8e6057a42caca29e5647b72f78ff7a36#rd
saved: 2026-07-22 07:54:30
tags:
  - 笔记同步助手
id: cef0cb10-4e22-4afa-b701-e24dd3f7cab9
---

公众号名称：学嵌入式的长路

作者名称：LongWay

发布时间：2026-07-22 07:23

> LONGWAY EMBEDDED

\`#define\` 远不止定义常量 · Beyond Constants

宏的高级写法 ·

从 do{}while(0) 到 ## 拼接  

used宏do{}while(0)X-Macro可变参数宏

> 01
> do{}while(0)：宏的最佳外套

> 02
> ## 与 #：字符串化与拼接魔法

> 03
> 可变参数宏：灵活的日志系统

> 04
> X-Macro：数据驱动的代码生成

> 05
> 终极综合：注册表风格的设备驱动框架

C 语言的宏（#define）是嵌入式开发中最被低估的利器。很多人只知道用宏定义常量，却不知道宏可以写出媲美模板的代码生成能力。本文带你深入宏的四个高阶技巧，每个都配嵌入式实战场景。

01

### do{}while(0)：宏的最佳外套

do{}while(0): The Safe Wrapper for Macros

问题：普通宏的陷阱

你写过这样的宏吗？

> GitHub Dark · C

\#defineLED\_ON()HAL\_GPIO\_WritePin(LED\_GPIO\_Port,LED\_Pin,SET)

\#defineLED\_OFF()HAL\_GPIO\_WritePin(LED\_GPIO\_Port,LED\_Pin,RESET)

单条语句没问题，但换成多条：

> GitHub Dark · C

\#defineSAFE\_PRINT(msg)\\

if(uart\_busy)\\

printf("busy\\n");\\

printf(msg)

调用 if (cond) SAFE\_PRINT("hello"); 时，展开后 else 会错误地挂到内部的 if 上——经典的「悬空 else」bug。

解决方案：do{}while(0)

> GitHub Dark · C

\#defineSAFE\_PRINT(msg)\\

do{\\

if(uart\_busy)\\

printf("busy\\n");\\

else\\

printf(msg);\\

}while(0)

**为什么是 \`do{}while(0)\` 而不是 \`{}\`？**

> GitHub Dark · C

// ❌ 大括号方案：

\#defineBAD\_MACRO(){func1();func2();}

if(cond)

BAD\_MACRO();

else

other();// 编译错误：else 前多了一个分号！

// ✅ do{}while(0) 方案：

\#defineGOOD\_MACRO()do{func1();func2();}while(0)

if(cond)

GOOD\_MACRO();

else

other();// ✅ 完美编译

do{}while(0)末尾的分号是语句的自然结束符，而 {} 末尾加 ; 就会在 if 后面多出一条空语句，导致 else 悬空。

实战：断言宏

> GitHub Dark · C

\#defineASSERT(cond,msg)\\

do{\\

if(!(cond)){\\

printf("ASSERT: %s in %s:%d\\n",\\

msg,\_\_FILE\_\_,\_\_LINE\_\_);\\

hard\_fault\_handler();\\

}\\

}while(0)

voidapp\_task(void){

uint32\_t\*p\=get\_buffer();

ASSERT(p!\=NULL,"buffer pointer is NULL");

\*p\=0x55;// 安全访问

}

02

### ## 与 #：字符串化与拼接魔法

\## and \#: Stringification and Token Pasting Magic

\#：参数变成字符串

> GitHub Dark · C

\#defineTO\_STRING(x)#x

printf("%s\\n",TO\_STRING(12345));// 输出 "12345"

printf("%s\\n",TO\_STRING(GPIOA));// 输出 "GPIOA"

**实战：注册表打印**

> GitHub Dark · C

\#defineREG\_DUMP(reg)\\

do{\\

printf("%s = 0x%08lX\\n",\\

\#reg,(unsignedlong)(reg));\\

}while(0)

// 用法

REG\_DUMP(USART1\-\>SR);// 输出: USART1->SR = 0x000000C0

REG\_DUMP(TIM2\-\>CNT);// 输出: TIM2->CNT = 0x00000452

##：拼接标识符

> GitHub Dark · C

\#defineMAKE\_REG(name,num)name##num

// 展开为：GPIOA、GPIOB

MAKE\_REG(GPIO,A)

MAKE\_REG(GPIO,B)

**实战：批量定义寄存器访问宏**

> GitHub Dark · C

// 批量生成引脚控制宏

\#definePIN\_FUNC(port,num)\\

do{\\

MAKE\_REG(GPIO,port)\-\>BSRR\=(1<<num);\\

}while(0)

// 用法

PIN\_FUNC(A,5);// GPIOA->BSRR = (1 << 5); → PA5 置高

PIN\_FUNC(B,3);// GPIOB->BSRR = (1 << 3); → PB3 置高

实战：自动生成中断向量表名称

> GitHub Dark · C

\#defineIRQ\_HANDLER(irq\_num)\\

voidMAKE\_REG(USART,irq\_num)##\_IRQHandler(void)

// 展开为：void USART1\_IRQHandler(void)

IRQ\_HANDLER(1){

// USART1 中断处理

uint8\_tdata\=USART1\-\>DR;

}

03

### 可变参数宏：灵活的日志系统

Variadic Macros: Flexible Logging System

基本用法

> GitHub Dark · C

\#defineLOG(fmt,...)\\

printf("\[LOG\] "fmt"\\n",\_\_VA\_ARGS\_\_)

// 用法

LOG("temp = %d°C",temperature);// 输出: \[LOG\] temp = 28°C

问题：零参数的兼容性

> GitHub Dark · C

LOG("system init OK");// 展开为 printf("\[LOG\] system init OK\\n", );

// 多了一个逗号 → 编译错误

解决方案 C99：##\_\_VA\_ARGS\_\_（GCC 扩展）

> GitHub Dark · C

\#defineLOG(fmt,...)\\

printf("\[LOG\] "fmt"\\n",##\_\_VA\_ARGS\_\_)

// ^^ 双 ## 表示：如果 \_\_VA\_ARGS\_\_ 为空，删除前面的逗号

LOG("system init OK");// ✅ printf("\[LOG\] system init OK\\n");

LOG("temp = %d°C",temperature);// ✅ printf("\[LOG\] temp = %d°C\\n", 28);

实战：带等级的日志系统

> GitHub Dark · C

\#defineLOG\_LEVEL\_NONE0

\#defineLOG\_LEVEL\_ERR1

\#defineLOG\_LEVEL\_WARN2

\#defineLOG\_LEVEL\_INFO3

\#ifndefLOG\_LEVEL

\#defineLOG\_LEVELLOG\_LEVEL\_INFO

\#endif

\#defineLOG\_ERR(fmt,...)\\

do{\\

if(LOG\_LEVEL\>\=LOG\_LEVEL\_ERR)\\

printf("\[ERR\] "fmt"\\n",##\_\_VA\_ARGS\_\_);\\

}while(0)

\#defineLOG\_WARN(fmt,...)\\

do{\\

if(LOG\_LEVEL\>\=LOG\_LEVEL\_WARN)\\

printf("\[WARN\] "fmt"\\n",##\_\_VA\_ARGS\_\_);\\

}while(0)

\#defineLOG\_INFO(fmt,...)\\

do{\\

if(LOG\_LEVEL\>\=LOG\_LEVEL\_INFO)\\

printf("\[INFO\] "fmt"\\n",##\_\_VA\_ARGS\_\_);\\

}while(0)

// ⚡ 编译时就能裁剪！空宏无运行时开销

voidtest\_log(void){

LOG\_INFO("ADC value: %d",adc\_val);

LOG\_WARN("buffer 80%% full (%d/%d)",used,total);

LOG\_ERR("I2C timeout on bus %d",bus\_id);

}

> TECH NOTE
> 编译时，如果 LOG\_LEVEL 设为 LOG\_LEVEL\_WARN，所有 LOG\_INFO() 的 if 条件恒假，优化器会直接删除——**零运行时开销**。

04

### X-Macro：数据驱动的代码生成

X-Macro: Data-Driven Code Generation

X-Macro 是最被低估的宏技，它用同一个「数据表」同时生成枚举、字符串表、处理函数，保证三者始终同步。

实战：GPIO 功能表

先定义一个数据驱动表：

> GitHub Dark · C

// === gpio\_pins.def ===

X(GPIO\_PIN\_LED,'A',5)// PA5 - LED

X(GPIO\_PIN\_BTN,'B',3)// PB3 - Button

X(GPIO\_PIN\_UART\_TX,'A',9)// PA9 - USART1 TX

X(GPIO\_PIN\_UART\_RX,'A',10)// PA10 - USART1 RX

\#undefX

然后，用不同的 \#define X 生成不同内容：

> GitHub Dark · C

// === gpio\_enum.h — 生成枚举 ===

typedefenum{

\#defineX(name,port,pin)name,

\#include"gpio\_pins.def"

GPIO\_PIN\_COUNT

}gpio\_pin\_t;

// 展开为：

// typedef enum {

// GPIO\_PIN\_LED,

// GPIO\_PIN\_BTN,

// GPIO\_PIN\_UART\_TX,

// GPIO\_PIN\_UART\_RX,

// GPIO\_PIN\_COUNT

// } gpio\_pin\_t;

> GitHub Dark · C

// === gpio\_names.h — 生成字符串表 ===

staticconstchar\*gpio\_pin\_names\[\]\={

\#defineX(name,port,pin)#name,

\#include"gpio\_pins.def"

};

// 展开为：

// static const char \*gpio\_pin\_names\[\] = {

// "GPIO\_PIN\_LED",

// "GPIO\_PIN\_BTN",

// "GPIO\_PIN\_UART\_TX",

// "GPIO\_PIN\_UART\_RX",

// };

> GitHub Dark · C

// === gpio\_init.c — 生成初始化代码 ===

staticvoidgpio\_init\_all(void){

\#defineX(name,port,pin)\\

do{\\

GPIO\_InitTypeDefinit\={0};\\

init.Pin\=MAKE\_REG(GPIO\_PIN\_,pin);\\

init.Mode\=GPIO\_MODE\_OUTPUT\_PP;\\

init.Speed\=GPIO\_SPEED\_FREQ\_LOW;\\

MAKE\_REG(HAL\_GPIO\_Init,)(MAKE\_REG(GPIO,port),&init);\\

}while(0);

\#include"gpio\_pins.def"

}

一次性宏（不用头文件）

如果不想分离文件，用宏参数传递：

> GitHub Dark · C

\#defineGPIO\_PIN\_TABLE\\

X(GPIO\_PIN\_LED,A,5)\\

X(GPIO\_PIN\_BTN,B,3)\\

X(GPIO\_PIN\_UART\_TX,A,9)\\

X(GPIO\_PIN\_UART\_RX,A,10)

// 生成枚举

typedefenum{

\#defineX(name,port,pin)name,

GPIO\_PIN\_TABLE

GPIO\_PIN\_COUNT

}gpio\_pin\_t;

\#undefX

X-Macro 的真正价值

普通做法是手写枚举、手写字符串表、手写初始化函数——三份信息靠人工同步，改一个引脚要改三个地方。X-Macro 把数据集中在**一个地方**，所有代码都从这个数据推导生成，**不可能出现不一致**。

这在管理几十上百个引脚的项目中，节省的时间是量级的。

EOF

### 终极综合：注册表风格的设备驱动框架

Final: Registry-Style Device Driver Framework

把前面所有技巧结合：

> GitHub Dark · C

// 1. do{}while(0) 确保宏安全

// 2. ## 拼接寄存器名

// 3. 可变参数做日志

// 4. X-Macro 做设备表

\#defineDEVICE\_TABLE\\

X(DEV\_USART1,USART1,115200,'A',9)\\

X(DEV\_USART2,USART2,9600,'D',5)\\

X(DEV\_I2C1,I2C1,400000,'B',6)\\

X(DEV\_SPI1,SPI1,8000000,'A',5)

// 生成设备枚举

typedefenum{

\#defineX(name,reg,baud,port,pin)name,

DEVICE\_TABLE

DEV\_COUNT

}device\_id\_t;

\#undefX

// 生成初始化统一入口

\#defineDEV\_INIT(name,reg,baud,port,pin)\\

staticvoidMAKE\_REG(name,\_init)(void){\\

LOG\_INFO("Init %s @ %d baud",#name,baud);\\

/\*实际初始化...\*/\\

LED\_ON();\\

}

DEVICE\_TABLE

\#undefX

// 统一调用

voidall\_devices\_init(void){

\#defineX(name,reg,baud,port,pin)MAKE\_REG(name,\_init)();

DEVICE\_TABLE

\#undefX

}

> TECH NOTE
> 🏷️ 宏技巧 C语言 嵌入式编程 代码生成 do-while-0 X-Macro 可变参数宏 预处理器

> 我是 **学嵌入式的长路**，用代码和思考陪你走通嵌入式的每一公里。
> 喜欢这类嵌入式技术复盘，可以点个在看；想看某个方向，也欢迎在评论区告诉我。

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/cbb3eff5_1784678068822?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUzMjk3OTYwOA%3D%3D%26mid%3D2247484537%26idx%3D1%26sn%3D62deb9943f05f6f3ad4e6afd4622f22a%26chksm%3Dfb9d6ca850a185d731af0531ad82484f1962dbe14c947d8b77ba5d9353518462b990a301e4e2%26mpshare%3D1%26scene%3D1%26srcid%3D0722rz6bXqAa2xZaa7a9Rpqi%26sharer_shareinfo%3D8e6057a42caca29e5647b72f78ff7a36%26sharer_shareinfo_first%3D8e6057a42caca29e5647b72f78ff7a36%23rd&s=obsidian)