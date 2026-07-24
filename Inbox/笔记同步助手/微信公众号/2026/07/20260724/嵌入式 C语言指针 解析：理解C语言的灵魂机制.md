---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247488199&idx=1&sn=6a586df484b9674c1228b7fa197169d8&chksm=c303b91c4c4b5e27baf15b2a8a82f7b55b2dbb00735ba362c4e104b85476ed3fd31a75460efd&mpshare=1&scene=1&srcid=07240NFmzmVql5FbicKcihgq&sharer_shareinfo=be1c4258bd94e0b4dc84f0bce39ebaaf&sharer_shareinfo_first=be1c4258bd94e0b4dc84f0bce39ebaaf#rd
saved: 2026-07-24 15:57:14
tags:
  - 笔记同步助手
id: 7afde028-0d24-4342-a38c-7a49a548df2b
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-07-03 20:23

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d5673b61529048fc2392d6b1f5e4601a_MD5.gif]]

大家好，我是蟹老板～

“C语言的灵魂是指针。”这话听得耳朵都起茧子了吧？但说真的，很多人觉得指针难，是因为把指针当成了“变量”。你得把指针当成“门牌号”！掌握了指针，你才算真正摸懂了C语言的底层逻辑。

## 一、指针的本质

### 1.1 内存编址与寻址基础

咱们先从最底层说起。

计算机的内存就是一大块存储空间，由无数个存储单元组成。每个存储单元可以存1个bit——就是0或者1。但1个bit太少了，啥也干不了。所以前辈们规定：**8个bit组成1个byte（字节），字节是内存寻址的最小单元**。

什么叫“寻址的最小单元”？就是说你给每个字节编一个号，这个编号就是地址。就像小区里的门牌号：301、302、403……有了门牌号，你就能精准找到一家人。内存也一样，有了地址，CPU就能精准找到每个字节。

那问题来了：**地址本身占多大空间？**

这取决于你的系统是多少位的。32位系统用32个0/1来表示一个地址，所以一个地址占4个字节。64位系统用64个0/1，地址占8个字节。门牌号的位数越多，能编的门就越多。32位能编址的范围是2^32=4GB，64位就是2^64，大到根本用不完。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/bf73a35b67f77071ccfd8dacccdc0dca_MD5.jpg]]

### 1.2 指针即地址——变量与内存的关系

好，现在我们知道内存有地址了。那变量是啥？

你在代码里写`int a = 999;`，编译器会干三件事：

1.  1\. 在内存里找一块空闲空间——int是4字节，就找4个连续的字节
    
2.  2\. 把999转成二进制（补码），填进去
    
3.  3\. 把这块空间的**起始地址**和变量名`a`绑定起来
    

所以，**变量名是给人看的，地址是给机器看的**。

那指针变量呢？它也是一个变量，但它存的东西比较特殊——**它存的是别的变量的地址**。

打个比方：普通变量`a`是一个钱包，里面装着999块钱。指针变量`p`是一张小纸条，上面写着“钱包a放在301房间”。你拿着小纸条，就能找到钱包，然后掏钱。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8716a7238028d33686a971e8ffea3451_MD5.jpg]]

### 1.3 指针的声明与解引用

声明指针的语法是`类型 *变量名`：

```
int a = 999;
int *p = &a;    // p是一个指针，指向a的地址
```

这里的`*`表示“这是一个指针变量”。`&a`是取地址操作符，拿到a的地址。

那怎么通过指针拿到a的值呢？用解引用操作符`*`：

```
printf("%d\n", *p);    // 输出999
*p = 100;              // 把a的值改成100
```

**注意**：声明里的`*`和运算里的`*`不是一回事。声明里的`*`是说“这是个指针”，运算里的`*`是说“取出这个地址里存的东西”。别搞混了。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8d2d40c1ab36a4555ef8a307b00a6820_MD5.jpg]]

### 1.4 指针的大小与类型意义

来看一道经典面试题：一个`char *` 的指针和一个`double *` 的指针，哪个占用的内存空间大？

很多人脱口而出：肯定是`double *` 大啊，`double` 都占8字节了。

直接零分。

不管是`char`、`int` 还是什么复杂的结构体，只要是在32位系统里，任何类型的指针变量本身**都只占4个字节**！因为它们装的都是一个32位的内存地址。在64位系统里，所有指针都占8个字节。

既然大家都一样大，那为什么声明指针的时候还要分`char *`、`int *` 呢？直接统一叫`void *` 或者`pointer` 不行吗？

不行。指针的类型不是决定它自己有多大，而是决定了两件事：

1.  1.  **步长**：当指针做加减法的时候，往前或往后跳过几个字节。
2.  2.  **了解读方式**：当解引用的时候，CPU到底是从这个起始地址开始一次性读1个字节、2个字节还是4个字节？

如果是`char *p`，`*p` 就只读1个字节；如果是`int *p`，`*p` 就会一次性读4个字节，并且按照整型的格式去解析这4个字节。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/1701768535234b9abb027ee97862830f_MD5.jpg]]

## 二、指针的核心操作与进阶语法

看懂原理只是入门，会用指针操作，才算真正掌握。

### 2.1 指针的算术运算

指针的加减法，**不是地址数值的加减，是元素个数的加减。**

```
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;        // p指向arr[0]

p = p + 1;           // p现在指向arr[1]
printf("%d\n", *p);  // 输出20
```

`p+1`不是地址加1，而是**地址加`sizeof(指向的类型)`**。`int*`加1跳4字节，`char*`加1跳1字节，`double*`加1跳8字节。

这个设计太聪明了。你想啊，如果`p+1`就是地址加1，那`int* p`指向一个int（4字节），`p+1`指向了int中间的位置——这还怎么玩？**指针算术是以“元素”为单位的，不是以“字节”为单位的**。

指针还能相减：

```
int arr[5] = {10, 20, 30, 40, 50};
int *p = &arr[0];
int *q = &arr[4];
ptrdiff_t diff = q - p;    // diff = 4，不是16
```

两个指针相减得到的是**元素的个数**，不是字节数。类型是`ptrdiff_t`，在`<stddef.h>`里定义。

指针比较也很直接——比较地址的大小。`p < q`就是问“p指向的地址是否比q小”。

有个边界情况要注意：你可以让指针指向数组最后一个元素的下一个位置，**但不能解引用它**。这是未定义行为。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/112d3fbce0395d24f300662218c35dd2_MD5.jpg]]

### 2.2 指针与数组

数组和指针，几乎是绑定的。

数组名在绝大多数情况下**等价于指向首元素的常量指针**。

为什么说是常量指针？因为数组的内存地址是固定的，不能被修改。所以arr++、arr=arr+1这种写法，全部会直接报错。

```
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;    // arr就是&arr[0]

// 下面两种写法完全等价
printf("%d\n", arr[2]);    // 输出30
printf("%d\n", *(p + 2));  // 输出30
```

数组下标运算`[]`本质上就是指针运算的语法糖。`arr[i]`等价于`*(arr + i)`。反过来，`p[i]`也等价于`*(p + i)`——**指针也可以用下标**。

多维数组呢？`int arr[3][4]`——`arr`是一个指向“含4个int的数组”的指针。`arr+1`跳的不是4个字节，而是**一整行**（16个字节）。这一点新手特别容易懵。

```
int arr[3][4];
int (*p)[4] = arr;    // p指向第一行
p++;                  // 现在p指向第二行
```

`(*p)[4]`这个写法就是“数组指针”——指向数组的指针。后面第三章会细讲。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f4902d2a9dde1f54b3e9ca43c517b1f4_MD5.jpg]]

### 2.3 指针与字符串

嵌入式开发里，字符串用得极多，日志打印、协议报文、设备ID，全是字符串操作。

C语言里字符串有两种玩法。

## 第一种：字符数组

```
char str1[] = "hello";
```

`str1`是一个数组，在栈上分配了6个字节（5个字符+1个'\\0'），内容可修改。

## 第二种：字符指针

```
const char *str2 = "hello";
```

`str2`是一个指针，指向**字符串常量区**的"hello"。这个字符串是只读的，**不能修改**。

```
str1[0] = 'H';   // OK
str2[0] = 'H';   // 段错误！字符串常量不能改
```

所以声明指向字符串常量的指针时，**一定要加`const`**。不加也能编译，但运行时修改就崩了。

另外注意：`char *p = "hello"`和`char p[] = "hello"`的区别——前者p指向只读区，后者p在栈上。面试常考。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ad710f54c0acee87e287771d83bbba32_MD5.jpg]]

### 2.4 多级指针

一级指针存变量地址，二级指针存**一级指针的地址**。

就这么简单。所有多级指针，都是层层嵌套的地址存储。

```
int a = 10;
int *p = &a;      // 一级指针
int **pp = &p;    // 二级指针
```

`pp`存的是`p`的地址。`*pp`取到`p`的值（即a的地址），`**pp`取到a的值。

二级指针最典型的应用场景是**函数里修改指针本身**：

```
void alloc_memory(int **ptr, int size) {
    *ptr = (int*)malloc(size * sizeof(int));
}

int *p = NULL;
alloc_memory(&p, 10);    // 传指针的地址
// 现在p指向了堆上分配的内存
```

如果传的是`int *p`，函数里修改的是p的副本，外面p还是NULL。**想修改指针本身，就得传指针的地址**——也就是二级指针。

多级指针在动态内存分配、二维数组动态分配里用得很多。但别滥用——三级以上指针可读性极差，除非你写的是极其底层的库。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3ed1638d7a66033feccf96ee106038af_MD5.jpg]]

## 三、复杂指针声明

看到这里，基础指针大家应该都懂了。但真正劝退大多数人的，是复杂指针声明——数组指针、指针数组、函数指针、返回指针的函数......

### 3.1 指针数组与数组指针

这一对概念，是经典恶心人的概念组合：指针数组、数组指针，也是面试必考题。

**指针数组**：`int *p[5]`——**p是一个数组，有5个元素，每个元素是`int*`类型**。

```
int a = 1, b = 2, c = 3;
int *p[3] = {&a, &b, &c};
printf("%d\n", *p[1]);    // 输出2
```

**数组指针**：`int (*p)[5]`——**p是一个指针，指向一个包含5个int的数组**。

```
int arr[5] = {1, 2, 3, 4, 5};
int (*p)[5] = &arr;        // p指向整个数组
printf("%d\n", (*p)[2]);   // 输出3
```

怎么记？**看运算符优先级**。`[]`优先级比`*`高，所以`int *p[5]`先结合`[]`——p是数组。`int (*p)[5]`用括号让`*`先结合——p是指针。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/bb2b306ef1c504729bc29b0c3b6073a4_MD5.jpg]]

### 3.2 函数指针

如果说普通指针是指向内存数据的入口，那函数指针就是指向代码段函数的入口。

嵌入式开发中，函数指针的地位极高。

函数指针——指向函数的指针。函数的入口地址也可以存到指针里。

声明方式：

```
int (*func_ptr)(int, int);    // func_ptr是一个指针，指向一个返回int、接收两个int参数的函数
```

赋值和使用：

```
int add(int a, int b) { return a + b; }

int (*p)(int, int) = add;    // 函数名就是地址
int result = p(3, 5);        // 调用，result = 8
```

**嵌入式里函数指针干啥用？** 太多了。

**回调机制**：你写一个驱动，让上层应用注册一个回调函数。驱动在特定事件发生时调用这个函数。上层代码和底层驱动解耦。

**状态机**：不用满屏的`switch-case`。每个状态对应一个处理函数，用函数指针数组做状态跳转表。

```
typedef void (*state_handler_t)(void);

state_handler_t state_table[] = {
    state_idle,
    state_running,
    state_error
};

void run_state_machine(int state) {
    state_table[state]();    // 直接调用对应状态的处理函数
}
```

代码量减少，可读性暴增。我做过一个通信协议的状态机，原来用switch写了200多行，改成函数指针表之后不到80行，还更容易加新状态。

**中断向量表**：本质上就是一个函数指针数组。每个中断号对应一个处理函数的地址——这玩意是芯片启动时固件写死的。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/fa697670bec349a5d64f9d8697987bde_MD5.jpg]]

### 3.3 返回指针的函数

这种函数叫“指针函数”——返回类型是指针。

```
int* find_max(int arr[], int size) {
    int *max = &arr[0];
    for (int i = 1; i < size; i++) {
        if (arr[i] > *max) max = &arr[i];
    }
    return max;
}
```

嵌入式里大量工具函数都是这种格式，比如字符串查找、数据缓冲区获取、硬件信息读取函数，都会返回对应的内存指针。

**陷阱**：**不要返回局部变量的地址**。

```
int* get_value(void) {
    int a = 10;
    return &a;    // 大错特错！a在函数返回后就被销毁了
}
```

返回的指针指向了一块已经被释放的栈内存——这就是**悬空指针**。你用它就是未定义行为，可能偶尔对，可能随机崩。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f70ee0315bd08d2277978afdceb0ae92_MD5.jpg]]

### 3.4 复杂声明的阅读法则

遇到特别复杂的声明怎么办？

记住一个万能法则：右左法则。

**规则**：从最里层的标识符开始，先往右看，再往左看。遇到括号就掉头。重复直到读完。

举个例子：

```
int (*p[5])(int, int);
```

从`p`开始——往右看，是`[5]`，所以p是一个包含5个元素的数组。遇到`]`，掉头往左看——是`*`，所以数组里的元素是指针。再往左看，跳出括号，是`(int, int)`——指针指向的函数接收两个int参数。再往左看，是`int`——函数返回int。

结论：**p是一个数组，包含5个函数指针，每个指针指向一个返回int、接收两个int参数的函数**。

右左法则能解一切复杂声明。但说实话，**真写代码的时候别搞这么复杂**。太复杂的声明用`typedef`拆开：

```
typedef int (*operation_t)(int, int);
operation_t p[5];
```

阅读体验好十倍。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d63c206beb7c1cbfc2c688bff39da98f_MD5.jpg]]

## 四、指针与类型修饰符

普通软件开发可能很少纠结const、volatile这些修饰符，但嵌入式开发者必须吃透。这些修饰符是**保障硬件代码稳定运行的核心关键**。

### 4.1 const与指针

`const` 这个关键字，意思是指数不能变（只读）。跟指针混在一起时，位置决定一切。

```
const int *p1; // 常量指针：p1指向的内容不能变，但p1自己可以变，指向别的地方去
int * const p2; // 指针常量：p2自己这个指针变量不能变，但它指向的内容可以变
const int * const p3; // 常量指针常量：统统不能变
```

记不住？再教你个土办法：**以`\*` 为分界线。**

-   • 如果`const` 在`*` 的左边（比如`const int *` 或者`int const *`），修饰的就是指针指向的内容。
    
-   • 如果`const` 在`*` 的右边（比如`* const`），修饰的就是指针变量本身。
    

在嵌入式开发里，如果你的函数参数是一个大结构体的指针，为了防止在函数里面不小心把结构体的内容改了，一定要加上`const`：

```
void print_sensor_data(const sensor_data_t *data) {
    // data->value = 100; // 如果你敢这么写，编译器直接报错，安全隐患提前消灭
}
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/5a827feccac2ea95544712846b5b67cb_MD5.jpg]]

### 4.2 volatile与指针

`volatile` 这个关键字，。它的核心作用只有一个：**禁止编译器优化，每次都从真实内存读取数据**。

编译器很聪明，他看到你连续读一个变量，中间没有修改过，他就会觉得：这变量没变啊，为了好性能，我直接把这变量读到 CPU 的寄存器里，后面每次都从寄存器拿，不跑去内存拿了。

但在嵌入式里，如果这个指针指向的是一个硬件外设的寄存器，比如串口的数据接收寄存器（DR）：

```
uint32_t *uart_dr = (uint32_t *)0x40013804;

while (*uart_dr == 0) {
    // 等待串口接收数据
}
```

如果编译器开了`-O2` 优化，它一看循环里没有修改`*uart_dr` 的代码，它就会把这条读指令优化掉，变成死循环。但实际上，这个寄存器的值是会随着外部硬件信号的输入而改变的！

所以必须加上`volatile`：

```
volatile uint32_t *uart_dr = (volatile uint32_t *)0x40013804;
```

这告诉编译器：老大，这个地址对应的内存是个疯子，它的值随时会变。你每次用它的时候，必须老老实实去物理内存/寄存器里重新读，千万别自作聪明做缓存。

哪些情况需要`volatile`呢？

-   -   **硬件寄存器**：寄存器的值可能被硬件改变
-   -   **中断服务程序修改的变量**：主循环和中断都会访问
-   -   **RTOS中多任务共享的变量**

![[Inbox/笔记同步助手/微信公众号/2026/07/images/0860f477e19ec8075d875167acbb2c25_MD5.jpg]]

### 4.3 const volatile的组合使用

`const` 和`volatile` 一个只读一个可变，这俩能一起用吗？

当然能。

**只读但硬件可改的寄存器**。比如ADC转换完成标志位——程序只能读，不能写；但硬件会在转换完成时自动修改它。

```
const volatile uint32_t *adc_done = (const volatile uint32_t*)0x40012400;
while (*adc_done == 0) {
    // 等待ADC转换完成
}
```

`const`告诉编译器“程序不能改这个值”，`volatile`告诉编译器“这个值可能被硬件改”——两者完全不冲突。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/0711022f81efca584e92ecf0714d5721_MD5.jpg]]

### 4.4 restrict关键字（C99）

`restrict`是C99引入的。它告诉编译器：**这个指针是访问它所指向数据的唯一方式**。

```
void copy_data(int *restrict dest, const int *restrict src, size_t n) {
    for (size_t i = 0; i < n; i++) {
        dest[i] = src[i];
    }
}
```

有了`restrict`，编译器可以放心优化，因为知道dest和src不会重叠。**嵌入式里用得不多**，因为大部分嵌入式编译器对C99支持不完整。知道有这么个东西就行。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/eb189a628d5e0f6238ca5f596dee9196_MD5.jpg]]

## 五、指针与内存管理

嵌入式开发的核心难点，归根结底就是内存管理。指针用错，内存直接崩，设备直接死机重启。

### 5.1 栈、堆、静态存储区与指针

C程序的内存布局大致分这几块：

**栈（Stack）** ：局部变量、函数参数。编译器自动分配释放。速度快，空间小。**函数返回后栈上的数据就没了**——前面说的“不要返回局部变量地址”就是因为这个。

**堆（Heap）** ：动态分配的内存。手动分配（malloc）和释放（free）。空间大，但速度慢，还有碎片问题。

**静态存储区**：全局变量、static变量。程序启动时分配，程序结束时释放。

**常量区**：字符串常量等。只读。

指针可以指向以上任何一个区域。**但不同区域的生命周期不同**——这是指针使用中最大的坑。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ba7f98b4f96a18489591f3399d27b26d_MD5.jpg]]

### 5.2 动态内存分配与指针

在资源有限的嵌入式系统（比如只有 64KB RAM 的单片机）里，我非常不建议大规模使用`malloc` 和`free`。

```
int *p = (int*)malloc(10 * sizeof(int));
if (p == NULL) {
    // 内存分配失败
}
// 使用p...
free(p);
p = NULL;    // 释放后置NULL
```

为什么**嵌入式里动态内存要慎用**？原因有三：

1.  1.  **内存碎片**：反复malloc/free会产生碎片，最后大块内存分配不出来
2.  2.  **不确定性**：malloc的耗时不确定，实时系统里是大忌
3.  3.  **忘了free**：内存泄漏，嵌入式设备可能几个月不重启，慢慢耗尽

很多嵌入式项目**禁止使用动态内存**。所有内存静态分配，或者用内存池。

如果你非要用，每次`malloc` 之后请务必判断返回值是否为`NULL`！

![[Inbox/笔记同步助手/微信公众号/2026/07/images/210b2381dc8008f52698f88a23a74211_MD5.jpg]]

### 5.3 内存池与指针

既然标准的`malloc` 这么危险，那嵌入式里怎么做动态内存管理？——**内存池（Memory Pool）。**

我们可以提前定义好几个固定大小的对象数组，把它们当成池子。

```
#define BLOCK_SIZE  32
#define BLOCK_COUNT 10

uint8_t pool_storage[BLOCK_COUNT][BLOCK_SIZE];
uint8_t *free_blocks[BLOCK_COUNT];
int top = -1;

// 初始化时，把每行的首地址塞进空闲链表/栈里
void pool_init(void) {
    for(int i=0; i<BLOCK_COUNT; i++) {
        free_blocks[++top] = pool_storage[i];
    }
}
```

指针在内存池里扮演什么角色？**索引**——用指针去管理这些固定大小的 Block。申请和释放都只是普通的指针出栈入栈操作，时间复杂度是$O(1)$，而且绝对没有内存碎片。RTOS 内核（比如 FreeRTOS, RT-Thread）里的内存管理很多都是这个思想。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/55aeaf2c845998a9c359a00289e2255e_MD5.jpg]]

### 5.4 void指针

`void *` 叫无类型指针，或者通用指针。它能接收任意类型的指针，不需要强转。

```
void *generic_ptr;
int n = 10;
generic_ptr = &n; // 没问题，不需要写 (void *)&n
```

但是，`void *` 的特性也带来了约束：

-   -   **不能直接进行算术运算**：因为编译器不知道它的步长是多少（不过有些标准如 GNU C 允许把`void *` 当`char *` 算，但为了可移植性千万别这么写）。
-   -   **不能直接解引用**：必须先强转成具体类型的指针。

这玩意儿最大的用处是设计通用接口。看看 C 标准库的`memcpy`：

```
void *memcpy(void *dest, const void *src, size_t n);
```

`memset`：

```
void *memset(void *s, int c, size_t n);
```

不管你传的是结构体、字符数组还是整型数组，它都能通吃。内部实现时，它会强转成`char *`，然后一个字节一个字节地复制。这就是 C 语言里的泛型编程基础。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/043710650383a5012a8997720668d31a_MD5.jpg]]

## 六、指针在嵌入式硬件编程中的应用

前面所有知识点，最终都是为了服务硬件编程。嵌入式和普通C语言开发的本质区别，就是**直接通过指针操作物理内存、硬件寄存器**。

### 6.1 指针操作绝对地址

在写桌面软件的时候，你敢直接读写一个绝对地址，操作系统（Windows/Linux）反手就是一个不准访问保护，程序直接崩溃。

但在嵌入式里，你可以直接读写**绝对地址**。

```
// 向地址0x40021000写入0x01
*(volatile uint32_t*)0x40021000 = 0x01;

// 读取地址0x40021000的值
uint32_t val = *(volatile uint32_t*)0x40021000;
```

语法拆解：`(volatile uint32_t*)0x40021000`把一个整数强制转成指针。然后`*`解引用。

这就是嵌入式里操作寄存器的最基本方式。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f0c5b6847b06676b1c8da9858b62791a_MD5.jpg]]

### 6.2 硬件寄存器映射

实际项目中不会像上面那样裸写地址。一般用宏或者结构体。

**宏方式**：

```
#define PERIPH_BASE   0x40000000
#define GPIOA_BASE    (PERIPH_BASE + 0x20000)

#define GPIOA_MODER   (*(volatile uint32_t*)(GPIOA_BASE + 0x00))
#define GPIOA_ODR     (*(volatile uint32_t*)(GPIOA_BASE + 0x14))
```

**结构体方式**（CMSIS标准做法）：

```
typedef struct {
    volatile uint32_t MODER;
    volatile uint32_t OTYPER;
    volatile uint32_t OSPEEDR;
    volatile uint32_t PUPDR;
    volatile uint32_t IDR;
    volatile uint32_t ODR;
    // ...
} GPIO_TypeDef;

#define GPIOA ((GPIO_TypeDef*)0x40020000)

GPIOA->MODER = 0xFFFF;    // 用起来像操作对象
GPIOA->ODR |= (1 << 5);   // 设置PA5为高电平
```

结构体方式更清晰，编译器生成的代码和直接指针操作一样高效。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f110bac516a2601dd249a38de0ab87b2_MD5.jpg]]

### 6.3 内存对齐与指针

**对齐**是处理器对数据存放地址的要求。比如4字节的int必须放在地址是4的倍数的位置。

不对齐会怎样？有些处理器直接**硬件报错**，有些能访问但**性能暴跌**。

结构体里暗藏玄机：

```
struct example {
    char c;    // 1字节
    int i;     // 4字节——但偏移量必须是4的倍数，所以c后面补了3个字节
};
// sizeof(struct example) = 8，不是5
```

结构体成员之间可能有**填充字节**。用指针操作结构体时，**不要假设成员是连续存放的**。

指针类型转换也有对齐风险：

```
uint8_t buffer[10];
uint32_t *p = (uint32_t*)&buffer[1];    // buffer[1]的地址可能不是4的倍数
*p = 0x12345678;    // 未对齐访问！
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d54bcbaacedd0e35cd900874cbff4c44_MD5.jpg]]

### 6.4 字节序（大端/小端）与指针

**字节序**是多字节数据在内存里的存放顺序。

**小端**：低字节在低地址。x86、ARM默认小端。

**大端**：高字节在低地址。网络协议、某些嵌入式平台用大端。

```
uint32_t val = 0x12345678;
uint8_t *p = (uint8_t*)&val;

// 小端：p[0]=0x78, p[1]=0x56, p[2]=0x34, p[3]=0x12
// 大端：p[0]=0x12, p[1]=0x34, p[2]=0x56, p[3]=0x78
```

用指针做类型转换就能揭示字节序。这在**网络协议解析**里特别重要——收到的数据包可能是大端，你的处理器可能是小端，得用指针转换。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/35736c76b023482c7ef07a21f5ac2f6e_MD5.jpg]]

### 6.5 DMA缓冲区与指针

DMA（直接内存访问）是嵌入式里的重要外设。DMA的缓冲区指针有几个注意事项：

1.  1.  **地址对齐**：DMA通常要求缓冲区地址对齐到缓存行大小
2.  2.  **缓存一致性**：DMA和CPU可能共享缓存，需要维护一致性
3.  3.  **volatile**：DMA修改的缓冲区要用volatile修饰

```
// 典型DMA配置
#define DMA_BUFFER_SIZE 1024
__attribute__((aligned(32))) uint8_t dma_buffer[DMA_BUFFER_SIZE];

// DMA描述符中的指针
typedef struct {
    volatile uint32_t SRC_ADDR;    // 源地址指针
    volatile uint32_t DST_ADDR;    // 目标地址指针
    volatile uint32_t CTRL;        // 控制字段
} DMA_Desc_t;

DMA_Desc_t *desc = (DMA_Desc_t*)0x40026000;
desc->SRC_ADDR = (uint32_t)memory_source;
desc->DST_ADDR = (uint32_t)dma_buffer;
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/62ed6565ed745d543dc10e42808534d3_MD5.jpg]]

## 七、指针的安全性与防御性编程

指针是一把双刃剑。高效、灵活，但极其危险。

### 7.1 野指针与悬空指针

**野指针**：指针变量里存了一个**非法地址**。

三种典型成因：

```
int *p;          // 1. 未初始化——p里是随机值
p = &arr[10];    // 2. 越界访问——arr只有10个元素，arr[10]是第11个
free(p);         // 3. 释放后继续用——p指向的内存已归还
```

野指针的危害极大。它指向的内存可能是：

-   • 系统关键数据——改了系统就崩
    
-   • 其他变量的内存——数据被莫名篡改，bug极难定位
    
-   • 受保护的内存——直接段错误
    

**悬空指针**是野指针的一种——指向的内存曾经合法，但已经被释放了。

```
int *p = (int*)malloc(sizeof(int));
free(p);
*p = 10;    // 悬空指针！内存已释放
```

区别：野指针是“从来没合法过”，悬空指针是“曾经合法但现在不合法了”。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/27c634df720db5f652d4429d5c9e834c_MD5.jpg]]

### 7.2 段错误与内存越界

在嵌入式 Linux 上，如果你指针越界访问了不属于你的内存空间，内核会立刻毫不留情地抛出一个`Segmentation fault`（段错误），然后把你的进程杀掉。这时候你还能拿个 core dump 文件调一下。

但在裸机或者轻量级 RTOS（比如 FreeRTOS）里，**没有内存保护单元（MPU）的话，根本没有段错误这种概念！**

你指针越界了，写了别人的全局变量，系统不会报错。它会继续跑，直到那个被你踩坏的变量在半小时后被另一个模块使用时，才会引发匪夷所思的逻辑错误。这种 Bug 查起来，真的会让人抓掉大把头发。

数组越界就是典型的指针越界：

```
int arr[10];
arr[10] = 100;    // 越界！arr[10]是第11个元素
```

数组名退化成指针后，越界访问编译器**不检查**。你自己得负责。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/051a0ab1a01af0f7adc2515f8336daea_MD5.jpg]]

### 7.3 指针的安全使用规范

几条铁律：

## 1\. 指针必须初始化

```
int *p = NULL;    // 好习惯
```

## 2\. 释放后置NULL

```
free(p);
p = NULL;    // 防止悬空指针
```

## 3\. 使用前检查

```
if (p != NULL) {
    *p = 10;
}
```

## 4\. 不要返回局部变量的地址

## 5\. 小心边界——数组访问别越界

![[Inbox/笔记同步助手/微信公众号/2026/07/images/7972d3d8cfff75fafe1d79065ea3c4f1_MD5.jpg]]

### 7.4 常见指针错误的调试技巧

**GDB调试**：`print p`看指针值，`x/10xb p`看内存内容。

**Address Sanitizer**：GCC/clang的`-fsanitize=address`，能检测内存越界和use-after-free。

**静态分析**：用`-Wall -Wextra -Werror`，很多指针问题编译器能警告。

**打印日志**：在关键指针操作前后打印地址和值。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f4dfbab3d07d1ab0de18fa339bcab8d4_MD5.jpg]]

## 八、嵌入式指针实战案例

### 8.1 GPIO寄存器操作实战

完整例子——配置STM32的PA5为输出，然后翻转电平：

```
// 基地址定义
#define PERIPH_BASE    0x40000000UL
#define AHB1PERIPH_BASE (PERIPH_BASE + 0x00020000UL)
#define GPIOA_BASE      (AHB1PERIPH_BASE + 0x0000UL)

// 寄存器结构体
typedef struct {
    volatile uint32_t MODER;    // 0x00
    volatile uint32_t OTYPER;   // 0x04
    volatile uint32_t OSPEEDR;  // 0x08
    volatile uint32_t PUPDR;    // 0x0C
    volatile uint32_t IDR;      // 0x10
    volatile uint32_t ODR;      // 0x14
} GPIO_TypeDef;

// 映射到地址
#define GPIOA ((GPIO_TypeDef*)GPIOA_BASE)

// 使能GPIOA时钟（RCC）
#define RCC_BASE        0x40023800UL
#define RCC_AHB1ENR     (*(volatile uint32_t*)(RCC_BASE + 0x30))

void gpio_init(void) {
    // 使能GPIOA时钟
    RCC_AHB1ENR |= (1 << 0);
    
    // 配置PA5为输出（MODER[11:10] = 01）
    GPIOA->MODER &= ～(3 << 10);
    GPIOA->MODER |= (1 << 10);
    
    // 推挽输出
    GPIOA->OTYPER &= ～(1 << 5);
    
    // 高速
    GPIOA->OSPEEDR |= (3 << 10);
}

void gpio_toggle(void) {
    GPIOA->ODR ^= (1 << 5);    // 翻转PA5
}
```

每一行都是在用指针操作硬件。`GPIOA`是一个指针，`->`访问结构体成员，最终编译成对绝对地址的读写。

### 8.2 中断向量表与函数指针

Cortex-M的中断向量表示例：

```
// 中断处理函数声明
void Reset_Handler(void);
void NMI_Handler(void);
void HardFault_Handler(void);
void SysTick_Handler(void);
void USART1_IRQHandler(void);

// 向量表——函数指针数组
__attribute__((section(".isr_vector")))
void (* const g_pfnVectors[])(void) = {
    (void(*)())0x20000000,      // 栈顶地址
    Reset_Handler,              // 复位
    NMI_Handler,                // NMI
    HardFault_Handler,          // HardFault
    // ...
    SysTick_Handler,            // SysTick
    // ...
    USART1_IRQHandler,          // USART1
};
```

**芯片上电后从向量表的第二个条目（Reset\_Handler）开始执行**。中断发生时，硬件根据中断号从向量表里取出对应的函数指针，跳转执行。

这就是函数指针最底层的应用。

### 8.3 RTOS任务栈与指针

RTOS里每个任务有自己的栈。任务切换时，需要保存和恢复栈指针。

```
typedef struct {
    uint32_t *stack_top;      // 栈顶指针
    uint32_t *stack_base;     // 栈底指针
    uint32_t stack_size;      // 栈大小
    void (*task_entry)(void*);// 任务入口函数指针
    void *task_param;         // 任务参数
    uint32_t *current_sp;     // 当前栈指针
} Task_TCB_t;

Task_TCB_t task1;
uint32_t task1_stack[1024];

void task1_entry(void *param) {
    while (1) {
        // 任务代码
    }
}

void init_task(void) {
    task1.stack_base = task1_stack;
    task1.stack_top = task1_stack + 1024;
    task1.task_entry = task1_entry;
    task1.current_sp = task1.stack_top;
}
```

任务栈本质上就是一块连续内存，用指针来管理。栈溢出检查就是看指针有没有超出边界。

### 8.4 通信协议解析中的指针应用

解析数据包，指针是最高效的方式：

```
// 协议头定义
typedef struct __attribute__((packed)) {
    uint8_t  sync;          // 同步字节
    uint8_t  len;           // 数据长度
    uint16_t cmd;           // 命令字
    uint32_t seq;           // 序列号
} ProtocolHeader_t;

// 数据包缓冲区
uint8_t rx_buffer[256];

void parse_packet(uint8_t *buffer, uint32_t size) {
    if (size < sizeof(ProtocolHeader_t)) return;
    
    ProtocolHeader_t *header = (ProtocolHeader_t*)buffer;
    
    // 检查同步字节
    if (header->sync != 0xAA) return;
    
    // 注意字节序！cmd和seq可能是网络字节序（大端）
    uint16_t cmd = ntohs(header->cmd);    // 大小端转换
    uint32_t seq = ntohl(header->seq);
    
    // 数据部分在header后面
    uint8_t *data = buffer + sizeof(ProtocolHeader_t);
    uint32_t data_len = header->len;
    
    // 处理数据...
}
```

用指针偏移跳过协议头，直接定位到数据区。这就是为什么嵌入式协议栈里指针无处不在。

## 九、嵌入式指针面试题

**Q1：指针和数组有什么区别？**

数组名在大多数情况下退化为指向首元素的指针。但：

-   -   `sizeof(arr)`是数组总大小，`sizeof(p)`是指针大小（4或8）
-   -   `&arr`是数组的地址（类型是数组指针），`&p`是指针的地址（类型是二级指针）

**Q2：`char *p`和`char p[]`有什么区别？**

`char *p`是指针，指向字符串常量区（只读）。`char p[]`是数组，在栈上分配（可读写）。

```
char *p = "hello";    // 不能修改p[0]
char p[] = "hello";   // 可以修改p[0]
```

**Q3：常量指针和指针常量有什么区别？**

`const int *p`：指向的内容不可改。`int * const p`：指针本身不可改。记忆：const在_左边修饰内容，在_右边修饰指针。

**Q4：函数指针和指针函数有什么区别？**

函数指针：`int (*p)(int)`——指向函数的指针。  
指针函数：`int* func(int)`——返回指针的函数。

**Q5：什么是野指针？怎么避免？**

野指针是指向非法地址的指针。避免方法：初始化、不越界、释放后置NULL、不返回局部变量地址。

**Q6：`void*`指针有什么特点？**

可以指向任何类型。不能直接解引用。不能做算术运算。常用于泛型编程。

**Q7：嵌入式里为什么用`volatile`？**

防止编译器优化，确保每次都真实读写。硬件寄存器、中断共享变量、RTOS共享变量必须用。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/2e62603a_1784879830429?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247488199%26idx%3D1%26sn%3D6a586df484b9674c1228b7fa197169d8%26chksm%3Dc303b91c4c4b5e27baf15b2a8a82f7b55b2dbb00735ba362c4e104b85476ed3fd31a75460efd%26mpshare%3D1%26scene%3D1%26srcid%3D07240NFmzmVql5FbicKcihgq%26sharer_shareinfo%3Dbe1c4258bd94e0b4dc84f0bce39ebaaf%26sharer_shareinfo_first%3Dbe1c4258bd94e0b4dc84f0bce39ebaaf%23rd&s=obsidian)