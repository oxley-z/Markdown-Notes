---
github.io:
csdn: yes
---

# Cache 组织形式（VIVT、VIPT、PIPT）

# 1. 虚拟高速缓存 VIVT（Virtually-Indexed Virtually-Tagged）

&emsp;&emsp;虚拟高速缓存以虚拟地址作为查找对象，即虚拟地址做 index，虚拟地址做 Tag，在查找 Cache line 过程中不借助物理地址，在 ARM 初期的 ARM4、ARM5 等处理器的 Cache 主要采用这种方式。

![vivt](https://img-blog.csdnimg.cn/d98ad40d82e945578dda2b6674110776.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)
<center>图1 VIVT 数据查找</center>
&nbsp;

&emsp;&emsp;如上图 1 所示，虚拟高速缓存（VIVT）以虚拟地址（VA）作为查找对象，当CPU发出虚拟地址时，Cache 根据送来的虚拟地址中的 index 部分在 Cache中查找对应的 Cache line，将查找的 Cache line 对应的 Tag 部分与虚拟地址（VA）的 Tag 部分进行比较，若 Tag 部分相同，则命中。之后根据Cache line中的有效位（valid）确定 Cache line 中的数据是否有效，有效则根据虚拟地址中的 Offset 部分找到对应的数据，将其返回给 CPU。若在比较 Tag 过程中发现不对应，则发生 Cache miss，此时需要将虚拟地址通过 MMU 转为物理地址后根据物理地址找到对应的主存数据存放地址，将数据返回给 CPU 和 Cache。


### <font color=red>**优点**</font>

&emsp;&emsp;CPU在进行读取或写入操作时，不需要每次都将虚拟地址通过 MMU 转为物理地址，这大大节省了 CPU 等待的时间。同时由于不需要进行虚拟地址转物理地址，所以硬件设计也相对更加简单。

### <font color=red>**缺点**</font>
&emsp;&emsp;当然 VIVT 也有它的不足，在进行数据查找过程中，由于使用虚拟地址作为 Tag，所以引入两个问题。 <font color=red>歧义(ambiguity) [^ambiguity]</font> 和 <font color=red>别名(alias) [^alias]</font>。为了保证系统的正确工作，可以使用操作系统来避免出现歧义和别名两个问题。

---

## 2. 物理标记的虚拟高速缓存 VIPT（Virtually-Indexed Physically-Tagged）
&emsp;&emsp;在使用虚拟存储器的系统中，物理标记的虚拟高速缓存VIPT 使用虚拟地址做 index ，物理地址做 Tag。在利用虚拟地址索引 Cache 的同时利用 TLB/MMU 将虚拟地址转换为物理地址。然后将转换后的物理地址与虚拟地址索引到的 Cache line 中的 Tag 作比较，如果匹配则命中。这种方式要比 VIVT 实现复杂，在进行进程切换时，不在需要对 Cache 进行 invalidate 等操作(因为匹配过程中需要借物理地址)。但是这种方法仍然存在 Cache 别名（Alias）的问题（即两个不同的虚拟地址映射到同一物理地址，且位于不同的 Cache line）。

![VIPT](https://img-blog.csdnimg.cn/acfa698dcb604038a186cb505d946221.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)
<center>图2 VIPT 数据查找</center>
&nbsp;

&emsp;&emsp;如上图 2 所示，物理标记的虚拟高速缓存（VIVT）以物理地址部分位作为 Tag ，使用虚拟地址 index 进行 Cache line 的索引。当 CPU 发出虚拟地址（VA）时，根据虚拟地址的 index 进行 Cache line 的查找，同时，内存管理单元（MMU）将虚拟地址（VA）转为物理地址（PA），当地址转换完成，同时 Cache 控制器也对 Cache line 查找完成，此时将查找到的 Cache line 对应的 Tag 和物理地址 Tag 域进行比较，以判断 Cache 是否命中。

---

## 3. 物理高速缓存 PIPT（Physically-Indexed，Physically-Tagged）
&emsp;&emsp;在使用虚拟存储器的系统中，物理高速缓存 PIPT 中的 Tag 和 index 均为物理地址，而 CPU 发出的是虚拟地址，这便需要通过 TLB/MMU 查询内存中的页表，先将 CPU 发出的虚拟地址转换为物理地址，再进行 Cache 的缓存查找，这种先将 CPU 发出的虚拟地址转为物理地址，之后将物理地址作为访问 Cache 的 Tag 和 index 的方式称为 PIPT（Physically-Indexed，Physically-Tagged）。ARM 的 Cortex-A 系列高性能处理器大多采用 PIPT 方式，PIPT 的方式在芯片的设计要比 VIPT 复杂很多，而且需要等待 TLB/MMU 将虚拟地址转换为物理地址后，才能进行 Cache line 寻找操作，所以相对的速度相较于 VIPT 慢。

&emsp;&emsp;由于寻址 TLB 的过程也需要消耗一定的时间，对处理器的周期时间会造成一定的影响，所以一些处理器将访问 TLB 的过程单独作为一个流水线，用以减少访问 TLB 对处理器周期时间的影响。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c17fd366acc648a7b0c7176f423fb163.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)
<center>图3 PIPT 数据查找</center>
&nbsp;

&emsp;&emsp;如上图 3 所示，物理高速缓存 PIPT使用物理地址做 Tag 和 index 。首先CPU 发出的虚拟地址经过 MMU 转换成物理地址，之后将物理地址发往 Cache 控制器，通过物理地址中的 index 部分对 Cache line 进行查找，查找到对应 Cache line 后将物理地址的 Tag 部分与 Cache line 中的 Tag 部分进行比较，如果相同，则命中，之后根据有效位（valid）进一步确定是否直接返回给 CPU，如果 Tag 部分不相同，则将内存中对应地址的数据取出返回给 CPU 和 Cache。

---
# 参考资料
---
《超标量处理器设计》

知乎专栏：[高速缓存一致性](https://www.zhihu.com/column/cpu-cache)

CSDN：[Cache的相关知识](https://blog.csdn.net/m0_47799526/article/details/111186953)

---

♑
🧨
⏺



[^ambiguity]:歧义：是指不同的数据在 Cache 中具有相同的 Tag 和 index。参见 [Cache的基本原理](https://blog.csdn.net/qq_41596356/article/details/122360914)
[^alias]:别名：当不同的虚拟地址映射相同的物理地址，而这些虚拟地址的 index 不同时会出现 Cache alias。参见 [Cache的基本原理](https://blog.csdn.net/qq_41596356/article/details/122360914)