---
author: LLVM二次开发
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzY5MDAxODU4OQ==&mid=2247484018&idx=1&sn=34c0866bfc09becf79e67ae5520cb5ef&chksm=f23a5fa3625f9f0fbfc2850ef54a5e54ad789e4028e2e82e48bb50b13b31fa372db5208a9217&mpshare=1&scene=1&srcid=0528HQhtsaXWFGuKH4lzEmYZ&sharer_shareinfo=e72947d53733c6b811e8dd4b0603d272&sharer_shareinfo_first=e72947d53733c6b811e8dd4b0603d272#rd
saved: 2026-05-28 09:58:33
tags:
  - 笔记同步助手
id: edfad330-8648-4159-8811-35eb1ff4d3ad
---

公众号名称：基础编译器

作者名称：LLVM二次开发

发布时间：2026-05-25 16:46

![[images/86e9a4fcd458a30fcde5a146920ac24d_MD5.jpg]]

目录

-   为什么要理解ELF可执行文件格式
    
-   ELF简介
    

-   ELF文件的内容和分类
    

-   ELF中的常见Sections
    

-   ELF文件的结构
    

-   ELF Header
    

-   Section表
    

-   Program表
    

-   LLVM对ELF结构的表达
    

-   ELF文件在编译过程中的位置
    

-   编译阶段
    

-   链接阶段
    

-   加载阶段
    

-   编译器视角对ELF的需求
    

-   链接需求、加载需求、运行需求
    
-   其他需求
    

为什么要理解ELF可执行文件格式

在程序构建运行（编译、链接、加载、运行）整个过程的后半阶段，软件程序以便于机器解读和处理的二进制（可执行或可链接）文件的形态存在。常见的可执行文件格式包括ELF、COFF、WebAssembly、MachO这几种，在LLVM的后端LLC以及链接器LLD中都有支持。

![[images/aabf390f21e8568cdf4fbc6c4eb177b9_MD5.jpg]]

ELF等可执行文件在格式设计上的出发点，是为了便于让机器而不是人类来解读处理，以此来实现高效的链接和加载过程\--这是这些可执行可链接文件格式的存在意义。这就和前面利于人类解读但不便于机器解读的程序源代码形态形成了鲜明对比。在链接之前的编译过程，是一个将利于人类阅读的源码转换为利于机器解读和进一步加工处理的二进制形态的过程。

诸多可执行文件格式中，应用最为广泛的两种格式是ELF和COFF。ELF格式主要用于Unix系环境下，COFF主要用于Windows环境下。目前，ELF格式是包括Ubuntu、Fedora等在内的诸多Linux发型版系统中的标准可执行文件格式，在服务器、桌面系统以及嵌入式系统中都很常见。因为ELF格式不和特定架构（x86/ARM等）硬绑定，也不限定运行环境的大小端、地址位宽，其在x86之外的很多通用处理器中使用广泛。此外，如果要为一个非Unix系环境制定可执行文件格式时，事实上也是可以直接套用ELF文件格式的。

ELF的格式标准来自GNU对类Unix环境下ABI（Application Binary Interface）的定义，其中还包含了Unix环境下软件安装包等规范的说明：

```
https://elinux.org/Executable_and_Linkable_Format_(ELF)
```

即便是对于不从事编译器开发的读者，对ELF文件结构和链接加载过程的理解，也是有价值的能力。当开发者对于ELF文件理解逐渐形成框架，程序编译、链接、加载这些过程对其而言也会逐渐从黑盒变成灰盒乃至于白盒状态。在对ELF形成上述理解的基础上，开发者日常遇到和ELF有关的问题，就能迅速知道该问题对应ELF中哪一个具体知识模块，从而缩小问题范围，也就能更流畅的对其分析解决。

  

## ELF简介

这一节我们看ELF的几个基本方面：

-   一个典型的ELF文件中，包含程序运行所需的哪些要素？
    
-   ELF文件能够以哪些标准进行内部分类？在各个分类方法下，又具体分为哪几类？
    
-   ELF文件格式存在于软件开发运行过程的哪些子过程？它在各个子过程的角色是什么？
    
-   需求方面，从链接、加载、性能的角度看，我们希望ELF应当具备哪些能力？
    

  

### ELF文件的内容和分类

ELF文件由C/C++源码经编译或链接而来，它的主要内容就是源码中函数和数据的二进制形式。C源码的函数经过编译生成了某个架构的汇编指令流，一般被放在ELF文件的.text section；而源码中的变量数据经过编译，一般根据其具体的初始化状态分配到.data或.bss section。看一个简单的例子，例中我们将ed01.c和ed02.c这2个C源文件编译并链接生成DSO可执行文件：

```
// ed01.c
extern int val0;
extern int val1;
extern int val2;
extern void func5 ();
extern void func6 ();
extern void func9 (char *);
void func2 () {
val0 = 0; val1 = 2; val2 = 7;
}
void func3 () {
func5 (); func6 ();
}
void func7 () {
func6 ();
}
void func8 () {
func9 ("hello\n");
}
// ed02.c
int val0 = 9;
int val1;
static int val4 = 7;
void func5 () {
val4 = 9;
}
```

用LLVM工具链中的clang和lld将源码编译链接成DSO动态链接库文件：

```
$ clang --target=sparcv9 -c -fPIC -O0 ed01.c
$ clang --target=sparcv9 -c -fPIC -O0 ed02.c
$ ld.lld --shared ed01.o ed02.o -o libed.so
```

用ls和file可以查看所生成DSO文件的体积以及ELF大致分类信息：

```
$ ls libed.so -ll
-rwxrwxr-x 1 sw sw 3896 Nov 18 01:29 libed.so
$ file libed.so
libed.so: ELF 64-bit MSB shared object, SPARC V9, total store ordering, version 1 (SYSV), dynamically linked, not stripped
```

file命令打出的ELF 64-bit MSB shared object, SPARC V9字符串部分，反映了这个ELF文件的基本分类要素：

-   “ELF 64”表明，它是一个ELF64位格式文件。
    
-   “MSB”表明，它的数据部分按照MSB大端模式排列，而不是LSB小端模式。
    
-   “shared object”表明，它是一个动态链接库文件，而不是可执行文件EXE或可重定位文件REL。
    
-   “SPARC V9”表明，它的目标硬件架构是SparcV9。
    

GCC工具链中的readelf和LLVM工具链中的兼容工具llvm-readelf，都可以细致的查看ELF信息：

```
$ llvm-readelf -a libed.so
ELF Header:
Magic:   7f 45 4c 46 02 02 01 00 00 00 00 00 00 00 00 00
Class:                             ELF64
Data:                              2's complement, big endian
Version:                           1 (current)
OS/ABI:                            UNIX - System V
ABI Version:                       0
Type:                              DYN (Shared object file)
Machine:                           Sparc v9
Version:                           0x1
. . .
Section Headers:
[Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
[ 0]                   NULL            00000000000 000000 000000 00      0   0  0
[ 1] .dynsym           DYNSYM          00000000238 000238 000108 18   A  4   1  8
[ 2] .gnu.hash         GNU_HASH        00000000340 000340 000040 00   A  1   0  8
[ 3] .hash             HASH            00000000380 000380 000060 04   A  1   0  4
[ 4] .dynstr           STRTAB          000000003e0 0003e0 00003a 00   A  0   0  1
[ 5] .rel.dyn          REL             00000000420 000420 000050 10   A  1   0  8
[ 6] .rel.plt          REL             00000000470 000470 000030 10  AI  1  13  8
[ 7] .rodata           PROGBITS        000000004a0 0004a0 000007 01 AMS  0   0  1
[ 8] .text             PROGBITS        000001004a8 0004a8 0000e0 00  AX  0   0  4
[ 9] .plt              PROGBITS        00000200590 000590 0000e0 00 WAX  0   0 16
[10] .dynamic          DYNAMIC         00000300670 000670 0000f0 10  WA  4   0  8
[11] .got              PROGBITS        00000300760 000760 000028 00  WA  0   0  8
[12] .data             PROGBITS        00000400788 000788 000008 00  WA  0   0  4
[13] .got.plt          PROGBITS        00000400790 000790 000030 00  WA  0   0  8
[14] .bss              NOBITS          000004007c0 0007c0 000004 00  WA  0   0  4
[15] .comment          PROGBITS        00000000000 0007c0 00007d 01  MS  0   0  1
[16] .symtab           SYMTAB          00000000000 000840 000180 18     18   6  8
[17] .shstrtab         STRTAB          00000000000 0009c0 00008b 00      0   0  1
[18] .strtab           STRTAB          00000000000 000a4b 00006c 00      0   0  1
. . .
```

readelf -a解析ELF动态链接库文件DSO的完整输出，一般包括全局ELF表头ELF Header、Section表、加载用的Program表、动态链接信息表.dynamic、动态链接重定位表.rel.dyn、函数重定位表.rel.plt、动态链接符号表.dynsym、全文符号表.symtab这几个重要部分。这里先初步了解全局表头ELF Header和Section表部分，其他部分下节介绍。

ELF Header部分的Magic是ELF魔数标识和基本类别信息，就是这文件开头的16个字节让file工作对其作出了快速识别。Class、Data、Type、Machine这4个字段是ELF标准内部对文件进行分门别类的几个不同维度：

-   Class字段，是对处理器位宽的分类，支持32位处理器和64位处理器。
    
-   Data字段，是对数据编码方式的分类，支持大端模式和小端模式。
    
-   Type字段，是对不同编译阶段产物的分类，支持可重定位文件Relocatable、动态链接库文件DSO、可执行文件EXE、以及运行阶段产生的可调试文件Core。
    
-   Machine字段，是对目标硬件架构的分类，支持x86、ARM等多种处理器架构。
    

Section Headers以列表形式给出ELF文件中所有Section的摘要信息。ELF文件的主体部分，就是一个个不同逻辑功能的Section。以上面打印为例：

  

-   .dynsym是本文件中用于动态链接的符号表。
    
-   .gnu.hash和.hash是用于以符号名为键快速定位到某个符号的哈希表。
    
-   .dynstr用于存放.dynsym动态链接符号表中各个符号的名字字符串。
    
-   .rel.dyn是动态链接符号的重定位信息表。
    
-   .rel.plt是参与动态链接的函数符号的信息表。
    
-   .rodata是只读数据段，常用与存放只读字符串。
    
-   .text用于存放源文件中函数源码编译生成的汇编指令流。
    
-   .plt是函数连接表，一般是运行阶段调用访问外部函数的跳板代码段。
    
-   .dynamic集中存放动态链接的各种关键信息，其设计出发点是加速加载动作。
    
-   .got是全局偏移量表，用于存放访问外部变量符号的加载后地址。
    
-   .data存放具有初值的全局变量。
    
-   .got.plt是全局函数偏移量表，用于存放访问外部函数符号的加载后地址。
    
-   .bss存放不具有初值的全局变量。
    
-   .comment保存着版本控制信息。
    
-   .symtab是涵盖本文件中所有符号的符号表，是.dynsym的一个超集superset。
    
-   .shstrtab是本ELF文件中所有section的名字字符串存放表。
    
-   .strtab用于存放.dynsym动态链接符号表中各个符号的名字字符串。
    

从上面各个section的职能分布来看来看，ELF文件的目的就是支撑动态链接、程序加载和运行这些软件过程。各个section的结构和section间的关联会在后续展开介绍。

  

### ELF文件的结构

ELF文件的主体，由内容相互重叠的Section序列和Program序列构成，如图7\-1的ELF整体结构图所示。

![[images/a67fd8ac13a3f25d0b0e9806efc2526c_MD5.jpg]]

图7\-1ELF文件结构

为了索引到这些互相重叠的Sections和Programs，同时提供出ELF文件的一些基本属性，ELF文件在结构上还包含ELF Header文件头、Program Header Table和Section Header Table三个部分。ELF文件这5个组成部分的功能分别是：

-   ELF Header，即整个文件的文件头。除了上节介绍的一些分类字段，ELF Header中还有两个Offset字段，分别是Program Header Table和Section Header Table这两部分在文件中的偏移量，以便于链接器、加载器和readelf等工具快速的找到它们。
    
-   Section Header Table，是Section摘要和索引表，其每个表项是相应Section在文件中的位置信息和摘要。
    
-   Program Header Table，是Program摘要和索引表，其每个表项是相应Program在文件中的位置信息和摘要。
    
-   Sections，是按照逻辑功能划分和生成的内容块，用于连接过程和视角。
    
-   Programs，是对Section内容块更粗粒度的划分，用于加载过程和视角。
    

这样一来，ELF文件在顶层可视作一个树状结构，ELF Header指向Program Header Table和Section Header Table，而Program Header Table和Section Header Table中的各entry又分别指向各个Program和各个Section。

ELF文件格式的一个特色，是其中Programs和Sections在内容区域在文件中是重叠的，一般若干个Sections归属到一个Program，而有的Sections则不归属到任何Program。这可以看ELF格式的Linking View和Execution View，如图7\-2所示。

![[images/d171d74882a8f2c84c92359bee1a13bc_MD5.jpg]]

图7\-2ELF文件结构的链接视图和加载执行视图

Section和Program这两种不同视角看到的组件，承担了ELF不同功能。

ELF的Section划分，是按照逻辑功能来的，这样的划分能有效服务于编译链接加载等过程所需的内容查找和全局信息综合。ELF中主要的那些section，一般能被划分到加载器指示、符号表、重定位、指令机器码、全局变量这几个功能大类。同时，Section之间是有着各种关联和引用或指向关系的，例如，符号表section .dynsym中各个符号的名字就是它引用的字符串section .dynstr来实际提供。再比如，ELF文件中完整的符号表服务是由.gnu.hash、.dynsym、.dynstr、.gnu.version这几个section叠加起来完成的。

而Program的划分，则是特定服务于加载过程的。它将若干RWX属性相同的section组织成地址相接的一整块内容，减少了加载器需要分析的总信息量和加载次数。显然，加载块数的减少也有利于降低分页系统中的页碎片。

这里有一点值得注意：在顶层的内容布局上，ELF格式让Programs Header Table紧跟着 ELF Header，却将Section Header Table安排在了文件的末尾。这样的设计并非是随意的，而是为了最大程度提升加载这一过程的效率。在Linux系统中，文件从硬盘到内存的读取，一般是以大小为4096字节的Page页为最小单位。这样一来，加载器读取的第一个页当中，就有很大可能包含了完整的ELF Header加上完整的Program Header，等于是一个Page的读取就承载了两级索引内容。如果Programs表被排在文件其他地方，那么读取ELF Header后还得要根据里面Programs表的偏移量再去硬盘读一个Page页，加载效率就低了。

  

#### ELF Header

ELF Header位于ELF文件的开头，其定义结构是固定的，总体分为ELF32和ELF64两种。ELF32用于32位地址空间环境，ELF64则用于64位环境。我们借助LLVM中的定义来查看ELF Header的结构：

```
// llvm/include/llvm/BinaryFormat/ELF.h
struct Elf32_Ehdr
struct Elf64_Ehdr
```

其中64位ELF Header定义为：

```
struct Elf64_Ehdr {
unsigned char e_ident[EI_NIDENT];  // ELF Header的前16字节，它的成员都是一字节一位域，不用区分大小端
Elf64_Half e_type;   // ELF文件类型，区分REL/DSO/EXE
Elf64_Half e_machine;    // 处理器架构，常见的有X86_64/AArch64等
Elf64_Word e_version;    // ELF版本，目前固定位1，即EV_CURRENT
Elf64_Addr e_entry;    // 程序执行入口的地址
Elf64_Off e_phoff;    // Program Header Table在文件中的位置
Elf64_Off e_shoff;    // Section Header Table在文件中的位置
Elf64_Word e_flags;    // 处理器架构特定的一些特性flag
Elf64_Half e_ehsize;    // ELF Header的大小，以字节计
Elf64_Half e_phentsize;    // Program Header entry大小，以字节计
Elf64_Half e_phnum;    // Program Header entry的数量
Elf64_Half e_shentsize;    // Section Header entry大小，以字节计
Elf64_Half e_shnum;    // Section Header entry的数量
Elf64_Half e_shstrndx;    // Section名字所引用字符串section的编号
// . . .
};
```

结合前面llvm-readelf –a查看到的Header信息，可以看出ELF Header中定义了文件类型（EXE/DSO/REL）、机器架构、版本、执行入口地址、Program和Section表在文件中的位置、Program/Section Header Entry的大小、Program和Section各自的数量。综合来看，ELF Header的存在，是为了索引到Program表和Section表，同时提供处理器架构、数据区域大小端这些文件元信息。

至于EI\_CLASS位宽（ELF64/ELF32）、EI\_DATA大小端格式、EI\_VERSION ELF格式版本、EI\_OSABI适配系统等这些最基础的分类信息，在ELF Header的开头（也就是文件的开头）16个字节中，即Elf64\_Ehdr结构体的e\_ident\[EI\_NIDENT\]成员：

```
enum {
EI_MAG0 = 0,       // ELF文件类型的MAGIC Number，等于固定值0x7f
EI_MAG1 = 1,       // ELF文件类型的MAGIC Number，等于固定值’E’
EI_MAG2 = 2,       // ELF文件类型的MAGIC Number，等于固定值’L’
EI_MAG3 = 3,       // ELF文件类型的MAGIC Number，等于固定值’F’
EI_CLASS = 4,      // 区分ELF32和ELF64
EI_DATA = 5,       // 区分LSB和MSB，即大小端
EI_VERSION = 6,    // 目前固定为1，即枚举值EI_CURRENT
EI_OSABI = 7,      // ABI适用的操作系统平台，常见的是UNIX System V
EI_ABIVERSION = 8,  // ABI版本
EI_PAD = 9,        // padding bytes的开始位置
EI_NIDENT = 16     // e_ident长度，值固定为16
};
```

e\_ident部分是ELF Header中最基础的信息，它们以字节为粒度进行定义，从而避免了ELF64与ELF32的差异、大小端差异造成的解析困扰。EI\_CLASS、EI\_DATA等这些位域的枚举取值完整集合，读者可以用cscope或source insight等代码分析工具作具体解析。

  

#### Section表与Program表

ELF Header所指向的Section Header Table和Program Header Table，都是连续的若干表项，每个表项的结构也是固定的，如下图7\-3所示。

![[images/d171d74882a8f2c84c92359bee1a13bc_MD5.jpg]]

图7\-3ELF文件的Section Header和Program Header

Section Header的表项结构，参看llvm代码：

```
// llvm/include/llvm/BinaryFormat/ELF.h
struct Elf32_Shdr;
struct Elf64_Shdr;
```

其中，64位的Section表项定义：

```
struct Elf64_Shdr {
Elf64_Word sh_name;    // Section的名字
Elf64_Word sh_type;    // Section的类型，可能是符号表、字符串表、重定位表、程序、数据等类型
Elf64_Xword sh_flags;   // Section的属性标识
Elf64_Addr sh_addr;   // Section的地址
Elf64_Off sh_offset;   // Section在文件中的位置
Elf64_Xword sh_size;   // Section的大小，以字节数计
Elf64_Word sh_link;   // 本Section需要引用的另一Section的序号
Elf64_Word sh_info;   // 某些Section会用到的额外信息
Elf64_Xword sh_addralign;   // Section的起始地址对齐要求
Elf64_Xword sh_entsize;   // 该Section中一个条目的大小，以字节数计
};
```

Section表项的大部分字段含义上都比较单纯，要么是地址，要么是空间大小。但其中有3个字段的内涵较为丰富：

-   sh\_type标识该Section的类型，这是一个Section区别于其他类型Section的根本属性。
    
-   sh\_flags字段包含该Section的若干属性，比如这个Section在运行阶段是否可写、运行阶段是否占用内存、是否引用其他Section、是否包含可执行的机器指令等等。 sh\_flags可包含的完整属性值，参看ELF.h中对SHF\_WRITE等枚举值的定义。
    
-   sh\_link字段标识的是该Section所引用的其他Section。例如，.dynsym引用.dynstr，因为.dynsym符号表中各个符号的名字字符串存放在.dynstr中。
    

再看Program Header Table中的表项结构，参看llvm代码：

```
// llvm/include/llvm/BinaryFormat/ELF.h
struct Elf32_Phdr
struct Elf64_Phdr
```

以64位的Program Header Table表项为例：

```
// Program header for ELF64.
struct Elf64_Phdr {
Elf64_Word p_type;    // 类型，可以为加载LOAD、动态链接信息DYNAMIC等
Elf64_Word p_flags;   // 属性比特位标识
Elf64_Off p_offset;   // 在文件中的位置
Elf64_Addr p_vaddr;   // 该segment加载时的虚拟地址
Elf64_Addr p_paddr;   // 该segment加载时的物理地址
Elf64_Xword p_filesz; // 该segment在文件中的内容大小，以字节计
Elf64_Xword p_memsz;  // 该segment加载时在需占据的内存大小，以字节计
Elf64_Xword p_align;  // 该segment的起始地址对齐要求
};
```

从结构体看，Program Header表项比Section Header表现更单纯，并且字段数量比Section Header少很多。这瑟吉欧因为Program表所服务的程序加载这一阶段，其流程比Section表所服务的链接阶段要更简单。Program表项中大部分字段的含义也很直白，也要么表示地址值，要么表示空间大小。我们看看字段含义相对丰富一些的p\_type和p\_flags字段：

-   p\_type是这个Program的类型，该字段的常见枚举值中，PT\_LOAD表示该Program应当被直接加载到内存，PT\_DYNAMIC表示该Program集中提供动态链接各项信息，而PT\_INTERP表示该Program中包含本ELF文件预期的加载器信息。PT\_LOAD等合法枚举值也定义在ELF.h中。
    
-   p\_flags字段标识该Program是否可读、可写、可执行，以及目标系统平台和硬件平台的自定义的属性。
    

  

### ELF文件在编译过程中的位置

ELF文件格式在软件构建运行过程中的存在，涵盖编译、链接、加载、运行等阶段，参看图7\-4，包括：

-   编译阶段。C源码经过gcc/clang这些编译器的编译，生成了ELF Relocatable文件，即平时大家看到的.o文件，又称为Object目标文件。ELF Relocatable的特点是此时文件中的全局变量和函数都没有被分配地址。
    
-   链接阶段。ELF Relocatable文件经由链接器（GNU的ld或LLVM的lld）链接，生成可执行文件Executable（后面简称EXE）或动态链接库文件Dynamic Shared Object（后面简称DSO）。
    
-   加载阶段。加载器将EXE或DSO加载进内存，形成程序的进程内存镜像。
    
-   运行阶段。在该阶段，如果出现错误，系统可以保留程序运行现场，包括栈、寄存器、其他要素，生成ELF Core文件，供调试分析。
    

![[images/aabf390f21e8568cdf4fbc6c4eb177b9_MD5.jpg]]

图7\-4编译流程中ELF形态的位置

可以看出来，ELF文件格式这一形态，纵向承担了实现高级编程语言运行到物理机器上这一工程长跑的最后几棒，其重要性并不亚于各类高级语言本身。事实上，ELF文件格式是SYSTEM V ABI系统标准的最主要内容，它是可执行文件和操作系统间的关键接口。

从上图可以看出，ELF在软件过程中的存在子形态，包括Relocatable可重定位文件、DSO动态链接库文件以及EXE可执行文件、以及Core核心文件。对于编译器和链接器而言，我们主要关注的是Relocatable、DSO以及EXE这三种ELF文件。它们三者的关键区别在于这几点：

-   Relocatable文件不可加载，需要链接生成DSO或者EXE之后才可加载运行。
    
-   DSO和EXE这两种文件都可以由系统的加载器直接加载运行。这里有个冷知识，DSO动态链接库也是可加载运行的，只要在它的ELF Header中标注清楚入口地址就行。
    
-   DSO和EXE的区别在于，EXE中各个程序部分（Program或Section）的地址是固定的，其加载起始地址不可调整；而DSO的加载只要求其中各个程序部分的相对地址不变，但其整体的加载起始地址可以是任意的——即我们常说的位置无关代码，PIC Position Independent Code概念。
    

  

### 我们对ELF的需求

不妨以这样的视角来考察ELF：假设ELF标准还未存在，但整个软件开发过程中和ELF直接连接的那些实体和概念都先一步已存在，应用场景对ELF文件格式的需求也呼之欲出。那么，在此情况下:

-   What。我们希望ELF具备那些能力？包括功能和性能方面。
    
-   How。对于这些需求，已然成为现有标准的ELF是如何应对的？
    
-   Where。对这些需求的回应，具体体现在ELF整体标准的哪一个细节中？
    
-   What If。同样的问题，在未涉猎ELF的情况下，由你来设计解决方案，又会打算怎么做？
    

这样的思考方式不仅针对ELF。在学习某项技术的初期，提出这样一些引导性问题，能够牵引自己的思维主动向前走，从而在学习过程中和自己既有的技术体系进行互动比对。这样的系列问题，能使学习者兼以一个批判比较的思维角度来看待现有方案设计，也能将这些新知识点融入自己既有的知识框架中。

回到对ELF的需求，对一种可执行文件格式的基本需求应包括功能和性能两方面：

-   从功能角度看，ELF纵向涵盖了程序构建执行过程的最后几个子过程，那么对每个子过程中的形态转换和信息传递应当有足够的信息支撑。
    
-   从性能角度看，我们希望它在设计上对编译、链接、加载、运行的高效执行也都能起到支撑
    

展开来看，见图7\-5所示。

![[images/86e9a4fcd458a30fcde5a146920ac24d_MD5.jpg]]

图7\-5软件工程对可执行文件格式的需求

从上图看，要定义一种实用的可执行文件格式，要考虑的点还是很多的。​软件过程上，这个格式要涵盖编译、链接、加载、运行等多个子过程；信息组织上，要支持文件内核跨文件的符号分析，要为各个符号定义足够的信息来支撑链接和加载等子过程；还要支持跨架构，不能和特定硬件架构绑定；性能上，更是要处处为链接和加载性能作考虑。​由此可见，一个能满足上述需求的实用可执行文件格式，其内部设计应当会有丰富的特性和精巧的构思。​接下来的几个小节，就通过典型例子和ELF内部关键结构的介绍，来展示ELF的这些丰富特性和精巧构思。

---

![[images/3af9b61d96b8267627e38ee9d6a5df6f_MD5.jpg|cover_image]]

原创 LLVM二次开发 基础编译器

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/7188f62f_1779933510604?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzY5MDAxODU4OQ%3D%3D%26mid%3D2247484018%26idx%3D1%26sn%3D34c0866bfc09becf79e67ae5520cb5ef%26chksm%3Df23a5fa3625f9f0fbfc2850ef54a5e54ad789e4028e2e82e48bb50b13b31fa372db5208a9217%26mpshare%3D1%26scene%3D1%26srcid%3D0528HQhtsaXWFGuKH4lzEmYZ%26sharer_shareinfo%3De72947d53733c6b811e8dd4b0603d272%26sharer_shareinfo_first%3De72947d53733c6b811e8dd4b0603d272%23rd&s=obsidian)