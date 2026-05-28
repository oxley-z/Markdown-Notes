---
author: LLVM二次开发
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzY5MDAxODU4OQ==&mid=2247483926&idx=1&sn=45183f5f318346547f27c373cfd77725&chksm=f29e7fc6db46d7fb1c1ce70bf3af9138435054b696b46cacda8ac17a4a1d145922ba70bd39ee&mpshare=1&scene=1&srcid=05285tgMla2AxJN2HX6qUVR9&sharer_shareinfo=e03371854a9cd8ba81105510123e091d&sharer_shareinfo_first=e03371854a9cd8ba81105510123e091d#rd
saved: 2026-05-28 10:17:19
tags:
  - 笔记同步助手
id: aed8900f-102c-42f1-b752-9ad243e874d7
---

公众号名称：基础编译器

作者名称：LLVM二次开发

发布时间：2026-05-09 11:42

## 目录

-   前置知识：ELF中的Program结构
    

-   ELF的Section和Program表
    

-   .dynamic动态加载Seciton
    

-   Program的加载
    

-   文件加载的地址分布
    

-   ELF加载的地址映射和文件复用
    

-   链接器运行流程
    
-   加载器运行流程
    

  

![[Inbox/笔记同步助手/微信公众号/20260528/images/3bfbfd9da87039a0029195c75672d6f0_MD5.jpg]]

## 链接 和 加载 是完整编译流程的尾部环节，在此之后就只剩 运行 和 问题现场 分析了。理解链接和加载流程，有助于分析解决软件工程中的很多日常问题，同时也是编译器开发的必备基础知识。

## 本文基于LLVM编译器和ELF可执行文件格式体系，为读者梳理链接和加载流程。在正式介绍这两个流程之前，有必要先了解一些ELF的Section和Program结构知识。

  

## ELF中的Programs

在使用了C语言中较多特性的的工程中链接生成的EXE/DSO里，section的数量往往比较庞大，有时达到40个以上。加载时，如果以section为粒度来遍历，功能上可行，但效率上是有很大优化空间的。从加载的角度看，很多具有相同属性（是否可读写、是否可执行、是否需要加载到内存中）的Section是可以合并处理的。ELF标准在设计上将这些相关属性相同的section合并后，映射为若干个Program，从而大幅度减少了加载阶段的迭代次数。

ELF的Section和Program表

在文件内容分布上，一个Program对应若干个Section，如图7\-16所示。

![[Inbox/笔记同步助手/微信公众号/20260528/images/6989f652c2433e005deb020a64d3d75e_MD5.jpg]]

图7\-16ELF文件的Section和Program表

上图中，ELF文件中的Program Header Table体现的是加载过程视角。Section Header Table体现的是链接过程视角，它的区分粒度比加载过程要细致很多。

用llvm-readelf可以查看ELF文件中的Program列表，以及Program与Section之间的映射关系：

```
$ llvm-readelf --program-headers /usr/lib/x86_64-linux-gnu/libc.so.6
. . .
Program Headers:  // Program列表
Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
PHDR         0x000040 0x000000040 0x00000000040 0x000310 0x000310 R   0x8
INTERP       0x1be6a0 0x0001be6a0 0x000001be6a0 0x00001c 0x00001c R   0x10
[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
LOAD         0x000000 0x000000000 0x00000000000 0x0214e8 0x0214e8 R   0x1000
LOAD         0x022000 0x000022000 0x00000022000 0x177624 0x177624 R E 0x1000
LOAD         0x19a000 0x00019a000 0x0000019a000 0x04d2c4 0x04d2c4 R   0x1000
LOAD         0x1e7788 0x0001e8788 0x000001e8788 0x005018 0x008ed8 RW  0x1000
DYNAMIC      0x1eab80 0x0001ebb80 0x000001ebb80 0x0001e0 0x0001e0 RW  0x8
. . .
Section to Segment mapping:  // Section到Program的映射关系
Segment Sections...
00
01     .interp
02     . . . .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_d .gnu.version_r .rela.dyn .rela.plt
03     .plt .plt.got .plt.sec .text __libc_freeres_fn
04     .rodata .stapsdt.base .interp .eh_frame_hdr .eh_frame .gcc_except_table .hash
05     .tdata .init_array . . . .data.rel.ro .dynamic .got .got.plt .data .bss
06     .dynamic
. . .
```

由上面可见，可加载到内存的Program只有4个了，和section视角里动辄40+个section相比，数量大幅下降了，这就是一个提高加载效率的有效设计。

ELF对Program定义了Type类型，用以标识各个Program的加载时用途、是否加载到内存中、是否为线程本地TLS数据等属性。LLVM对Program Type通过一系列枚举值作了定义：

```
// llvm/include/llvm/BinaryFormat/ELF.h
enum {
PT_NULL = 0,            // 无用的Program
PT_LOAD = 1,            // 可加载的Program
PT_DYNAMIC = 2,         // 动态链接信息集合，适用于动态链接场景
PT_INTERP = 3,          // 程序解释器的全路径信息
PT_NOTE = 4,            // 附加信息
PT_SHLIB = 5,           // 保留
PT_PHDR = 6,            // Program Header Table区域
PT_TLS = 7,             // Thread-Local Storage线程本地数据
PT_LOOS = 0x60000000,   // PT_LOOS～PT_HIOS为操作系统保留
PT_HIOS = 0x6fffffff,
PT_LOPROC = 0x70000000, // PT_LOPROC～PT_HIPROC为CPU架构保留
PT_HIPROC = 0x7fffffff,
// . . .
}
```

.dynamic动态加载Seciton

在这里诸多的Program Type中，较关键的是INTERP/LOAD/DYNAMIC这几种，上面libc.so.6中这几种Program对应的Sections和内容、属性说明如表7\-9所示。

表7\-9ELF的几种主要Program

<table style="border-collapse: collapse"><tbody><tr><td data-colwidth="89" width="89" valign="top" style="border: 1px solid \#ddd; padding: 6px 10px"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">Program</span></span></td><td data-colwidth="150" width="150" valign="top" style="border: 1px solid \#ddd; padding: 6px 10px"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">Sections</span></span></td><td data-colwidth="309" width="309" valign="top" style="border: 1px solid \#ddd; padding: 6px 10px"><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">说明</span></span></font></td></tr><tr><td data-colwidth="89" width="89" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><b><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">01 INTERP</span></span></b><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">R</span></span></td><td data-colwidth="150" width="150" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.interp</span></span></td><td data-colwidth="309" width="309" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">程序解释器的全路径标识，本文件中是</span></span></font><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">/lib64/ld-linux-x86-64.so.2</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">。</span></span></font><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">ELF</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">专门为</span></span></font><font face="Times New Roman"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.interp</span></span></font><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">映射了一种</span></span></font><font face="Times New Roman"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">Program</span></span></font><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">类型，便于加载器快速获得程序加载器的路径信息。</span></span></font></td></tr><tr><td data-colwidth="89" width="89" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><b><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">02 LOAD</span></span></b><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">R</span></span></td><td data-colwidth="150" width="150" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.gnu.hash .dynsym .dynstr</span></span><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.gnu.version .gnu.version_d</span></span><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.gnu.version_r</span></span><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.rela.dyn .rela.plt</span></span><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">. . .</span></span></td><td data-colwidth="309" width="309" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">需加载到内存中的只读</span></span></font><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">Sections</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">。</span></span></font><br><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">这里主要是符号表和重定位表系列的</span></span></font><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">sections</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">。</span></span></font></td></tr><tr><td data-colwidth="89" width="89" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><b><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">03 LOAD</span></span></b><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">R+E</span></span></td><td data-colwidth="150" width="150" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.plt .plt.got .plt.sec</span></span><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.text</span></span><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">. . .</span></span></td><td data-colwidth="309" width="309" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">需加载到内存中的可执行汇编指令流。</span></span></font><br><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">包括常规函数和函数链接表</span></span></font><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.plt*</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">等。</span></span></font></td></tr><tr><td data-colwidth="89" width="89" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><b><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">04 LOAD</span></span></b><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">R</span></span></td><td data-colwidth="150" width="150" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.rodata </span></span><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.interp .hash</span></span><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">. . .</span></span></td><td data-colwidth="309" width="309" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">需加载到内存中的只读</span></span></font><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">Sections</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">。</span></span></font></td></tr><tr><td data-colwidth="89" width="89" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><b><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">05 LOAD</span></span></b><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">R+W</span></span></td><td data-colwidth="150" width="150" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.tdata .init_array</span></span><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.dynamic .got .got.plt</span></span><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.data .bss</span></span><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.data.rel.ro </span></span><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">. . .</span></span></td><td data-colwidth="309" width="309" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">需加载到内存的可读写</span></span></font><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">Sections</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">。</span></span></font><br><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">包括全局变量、初始化函数指针表和</span></span></font><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">GOT</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">表等。</span></span></font></td></tr><tr><td data-colwidth="89" width="89" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><b><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">06 DYNAMIC</span></span></b><br><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">R+W</span></span></td><td data-colwidth="150" width="150" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.dynamic</span></span></td><td data-colwidth="309" width="309" valign="center" style="border: 1px solid \#ddd; padding: 6px 10px"><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">动态链接信息表。</span></span></font><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">ELF</span></span><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">专门为</span></span></font><font face="Times New Roman"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">.dynamic</span></span></font><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">映射了一种</span></span></font><font face="Times New Roman"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">Program</span></span></font><font face="宋体"><span style="color: rgba(0, 0, 0, 0.9); font-style: normal; font-weight: normal"><span style="font-size: 15px; color: rgb(0, 0, 0)">类型，便于加载器快速获得动态加载所需的各种信息。</span></span></font></td></tr></tbody></table>

到了这里，结合本表和前面内容，能够归纳出ELF标准中为了提升加载效率所做的设计至少有这几处明显的设计：

-   将具有相同属性的多个Sections映射为一个Program，提升加载数据的颗粒度，减少加载迭代次数
    
-   将Program Header Table的位置设计为紧随ELF Header，使得无论是以页Page、扇区Sector还是磁盘块IO Block为单位来读取ELF文件，ELF Header和Program Header Table都大概率能一次性被读入，降低了加载过程的IO成本
    
-   专门为.interp映射一种Program类型，便于加载器快速获得程序加载器的路径信息。
    
-   专门为.dynamic映射一种Program类型，便于加载器快速获得动态加载所需的各种信息。
    

  

### Program的加载

当我们在Linux命令行下运行一个ELF可执行文件EXE，加载器会递归分析它所依赖的动态链接库，因为一个动态链接库DSO仍会依赖其他的DSO。只有当这个EXE的加载依赖链上的所有DSO都被找到并可以加载，这个EXE才可以进入运行阶段。

一个EXE/DSO所依赖的动态链接库列在.dynamic section中：

```
$ llvm-readelf --dynamic /usr/lib/x86_64-linux-gnu/libfl.so.2 | grep NEEDED
0x0000000000000001 (NEEDED)       Shared library: [libc.so.6]
```

我们以图7\-17中这个EXE加载依赖的例子来看一个可执行文件加载后的典型地址空间分布。

![[Inbox/笔记同步助手/微信公众号/20260528/images/7bad4920b5f279ddf49c9b49a2811fe0_MD5.jpg]]

图7\-17ELF文件加载依赖示意

图中，可执行文件xyza.exe依赖libxyzb.so和libxyzc.so，而libxyzb.so又依赖libxyzd.so。所以，当我们运行xyza.exe，图中的4个ELF文件都要加载到内存。当然，这里所说的“加载到内存”，是指先为文件内容分配好（通常是按页Page）进程地址空间，而文件内容实际拷贝到内存这一动作可以延后到页面实际被引用时，由缺页中断处理函数来处理。

xyza.so等4个文件加载到内存中时，它们会被分配到进程xyza.exe的地址空间中，也即共享同一个地址空间，如图7\-18所示。

![[Inbox/笔记同步助手/微信公众号/20260528/images/00d7b00a908bebe1019f83ccfb499784_MD5.jpg]]

图7\-18ELF文件加载的地址分布示意

这里关于加载地址的设定规则，注意两个要点：

-   ELF加载一个EXE/DSO到内存中，是作整体偏移，即对该文件中的多个Program所作的地址偏移量是相等的。例如图7\-18中，libxyzb.so中各个Program的加载偏移量都是OffsetC。其他几个DSO或EXE也是如此。这样一来，ELF文件中汇编指令、数据之间的相对地址就在加载中保持不变——这是程序运行中相对寻址有效的基本前提。
    
-   DSO文件的加载偏移量（图中的OffsetB、OffsetC和OffsetD）并不要求等于某个特定值，且偏移量在每次加载时可以不同，也可以将内存中实际只加载了一份的DSO映射到不同进程的不同地址。但EXE的加载偏移量必须为0，这是基于ELF对EXE文件类型的设定，即EXE内部定义的符号的地址在链接时即已经全部确定。
    

另外需要主要的是，同一个DSO如果被多个进程引用，那么它的RO LOAD Program在内存中只会有一份物理拷贝，这份物理拷贝会被映射到多个进程的不同虚拟地址。这个机制诠释了DSO的全称Dynamic Shared Object中的Shared，即多进程共享。这个机制的好处显而易见，因为它降低了DSO文件到内存的拷贝总量。

但是，这个共享物理拷贝机制仅限于Program属性为RO时可以使用。如果一个Program X的属性为RW，那么就需要为每个引用该DSO的进程建立一份独享的Program X拷贝；进而如果一个Program Y的属性为RW+TLS，那么就需要为每个引用该DSO的进程中的每个线程Thread建立一份独享的Program Y拷贝。

综上，不同属性的Program在物理地址空间和虚拟地址空间的拷贝及映射可由图7\-19这个例子来更形象的描述。

![[Inbox/笔记同步助手/微信公众号/20260528/images/af46f8d97cec39f870c3d6f05985ba49_MD5.jpg]]

图7\-19ELF加载的地址映射和文件复用

图7\-19中，libxx.so被进程A和B引用，它的诸多Program中的P0/P1/P2三个具有不同属性：

-   P0属性为RO。它在物理地址空间只有1份拷贝，这份拷贝映射到虚拟地址隔离的进程A和B中。
    
-   P1属性为RW。它在物理地址空间有2份拷贝，P1 Pages\_Copy0和P1 Pages\_Copy1，这2份拷贝分别映射到进程A和B中。
    

P2属性为RW+TLS。它在物理地址空间有3份拷贝，P2 Pages\_Copy0、P2 Pages\_Copy1以及P2 Pages\_Copy2，这3份拷贝分别映射到进程A和B的线程Thread A0/A1/B中。

  

## 链接器功能流程

根据从前面几节了解到的ELF文件知识，即便不去看lld等链接器的源码，也是能够推断出链接器的大致功能流程的。链接器的主要工作，就是从输入的若干.o可重定位文件分解出Input Sections，将这些Input Sections重新排列或综合分析，组合或构建出新的Output Sections，然后写进输出的EXE/DSO文件。

当然，链接器对于不同类型Section的处理方式是有很大差异的：

-   对于代码段（.text等）和数据段（.data等）的处理相对简单，几乎就是复制粘贴加上一些排列工作，涉及到的每个section也只是作为一整段buffer来处理。
    
-   而对于符号表的汇总分析和逻辑拆解，则需要将各个.o可重定位文件的.symtab符号表的所有entry进行汇总分析，形成全局符号总表，最后再进行逻辑拆解，形成.symtab/.hash/.gnu.version等等具有不同结构符号表子表。符号表的拆解过程有点像数据库的数据表拆解设计，力求减少耦合，但又不能遗漏加载和分析所需信息。
    
-   重定位是需要另外处理的，且需要在各个Section的大小确定并且分配好地址后进行。
    
-   最后，一些特殊的Section需要基于前面已生成的Sections来确定内容，例如.dynamic和.shstrtab。
    

这样，链接过程中对于各种Section的处理可归纳为图7\-24所示。

![[Inbox/笔记同步助手/微信公众号/20260528/images/c867408e2e25542eb7075f516007e8fd_MD5.jpg]]

图7\-24链接流程中对各个输入输出Section的处理

根据上面列出要做的这些工作，在不细化性能设计的前提下，可以理出链接器的功能流程，如图7\-25所示。

![[Inbox/笔记同步助手/微信公众号/20260528/images/6e38883c029f50674ddeb2c598de7e00_MD5.jpg]]

图7\-25链接器功能流程

链接器的功能流程宏观上并不复杂，但如果真要开发一个链接器，细节就很繁杂了，比如符号总表的数据结构设计、符号和Input/Output Section之间映射的表示、复杂结构Section（如.gnu.hash）的生成、后期生成的Section的大小的计算、各类Section生成顺序的确定、Section和Program之间的映射的确定等等。这些细节的正确性和对性能的影响，都是需要考量的。

  

## 加载器功能流程

上一节，我们根据对ELF文件格式的理解，推断了连接器的主要流程；相应的，加载器的流程也可以推断，并且看起来会更为直接了当。从编译器前端到后端，再到链接器，再到这里的加载器，工具链所要处理的问题体量和复杂度是递减的。LLVM编译器的前端clang和后端llc涉及的代码都在百万行量级，链接器lld的代码在十万行量级，加载器的复杂度最低，一万行以内就能做到比较好的效果。

就ELF文件格式而言，其加载视图的颗粒度是Program，Program的数量一般在10个以内，这比链接器动不动要汇总吸收几百个Input Section然后生成几十个Section要轻松多了。并且，加载器对大部分Program（PT\_LOAD型）的处理，是将其作为一个整体Buffer，也并不需要分析其内部结构。

加载器需要完成的工作主要包括：

-   递归分析所要加载的EXE/DSO所依赖的DSO，形成加载文件集合。
    
-   集合中文件的PT\_LOAD Program的地址分配和加载映射。
    
-   加载时重定位的处理。
    

具体的，加载器的流程如图7\-26所示。

![[Inbox/笔记同步助手/微信公众号/20260528/images/20d1ad2ccb2f3bb63c3e21f14774a902_MD5.jpg]]

图7\-26加载器功能流程

加载器完成上述工作所需要的信息，在ELF文件中有着明显层次化的组织，如图7\-27所示。因此，加载器获取这些信息的流程也是比较固化的：

-   读取ELF Header，获取到Program Header Table的位置和数量、程序执行入口以及本ELF文件的类型这些信息。
    
-   通过ELF Header中的PHT Offset等信息，读取到整个Program Header Table。
    
-   Program Header Table中如果有PT\_INTERP（.interp），就据此收集到.interp指向的程序解释器路径信息。
    
-   Program Header Table中如果有PT\_DYNAMIC（.dynamic），就据此递归收集ELF文件所依赖的DSO，用以形成加载文件集合，同时收集.dynamic表中其他动态链接信息（如.dynsym起始地址等）。
    
-   收集罗列Program Header Table中Type为PT\_LOAD/PT\_TLS的Program，用以形成需要加载的Program集合。
    

![[Inbox/笔记同步助手/微信公众号/20260528/images/1ba88ad0d0f4632458ba4c1f39233857_MD5.jpg]]

图7\-27加载流程视角下的ELF信息组织

---

![[Inbox/笔记同步助手/微信公众号/20260528/images/9d279e5ca5ad76167fe84d5583282024_MD5.jpg|cover_image]]

Original LLVM二次开发 基础编译器

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/5f3a31f5_1779934633707?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzY5MDAxODU4OQ%3D%3D%26mid%3D2247483926%26idx%3D1%26sn%3D45183f5f318346547f27c373cfd77725%26chksm%3Df29e7fc6db46d7fb1c1ce70bf3af9138435054b696b46cacda8ac17a4a1d145922ba70bd39ee%26mpshare%3D1%26scene%3D1%26srcid%3D05285tgMla2AxJN2HX6qUVR9%26sharer_shareinfo%3De03371854a9cd8ba81105510123e091d%26sharer_shareinfo_first%3De03371854a9cd8ba81105510123e091d%23rd&s=obsidian)