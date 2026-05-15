---
github.io:
csdn: yes
---
# Cache的基本原理


&emsp;&emsp;Cache，中译名高速缓冲存储器，其作用是为了更好的利用局部性原理，以便减少CPU访问主存的次数。简单地说，CPU正在访问的指令和数据及附近的指令和数据，可能会被以后多次访问到。因此，第一次访问这一块区域时，将访问到的这一片区域（根据 Cache 大小决定）复制到 Cache 中，下次需要访问该区域的指令或者数据时，就不用再从主存中取出。
## 基本概念

<font color=red>**Cache 缓存行（Cache line）**</font>
&emsp;&emsp;Cache是由一组称为缓存行（Cache line）的固定大小的数据块组成，其大小是以突发读或者突发写周期的大小为基础的。每个高速缓存行完全是在一个突发读操作周期中进行填充或者下载的。即使处理器只存取一个字节的存储器，高速缓存控制器也启动整个存取器访问周期并请求整个数据块。缓存行第一个字节的地址总是突发周期尺寸的倍数。缓存行的起始位置总是与突发周期的开头保持一致。当从内存中取单元到 Cache 中时，会一次取一个Cache line 大小的内存区域到 Cache 中，然后存进相应的Cache line 中。

<font color=red> **Cache 命中率**</font>
&emsp;&emsp;缓存命中率是影响缓存性能的重要指标之一，表示在处理器与存储器交互过程中，数据从缓存中读取的比率。

<font color=red> **Cache 更新策略(Cache update policy)**</font>
&emsp;&emsp;Cache 更新策略是指当发生 Cache 命中时，写操作应该如何更新数据。Cache更新策略分成两种：写直通和回写。

<font color=red>**写直通（write through）**</font>
&emsp;&emsp;写直通又称写穿，当CPU执行store指令并在cache命中时，我们更新cache中的数据并且更新主存中的数据。cache和主存的数据始终保持一致。

<font color=red>**写回（write back）**</font>
&emsp;&emsp;当CPU执行store指令并在cache命中时，我们只更新cache中的数据。每个cache line中会有一个bit位记录数据是否被修改过，称之为dirty bit。主存中的数据可能是未修改的数据，而修改的数据躺在cache中。cache和主存的数据可能不一致。

<font color=red>**读分配（read allocation）**</font>
&emsp;&emsp;当CPU读数据时，发生cache缺失这种情况下都会分配一个cache line缓存从主存读取的数据默认情况下， cache都支持读分配。

<font color=red> **写分配（write allocation）**</font>
&emsp;&emsp;当CPU写数据发生cache缺失时，才会考虎写分配策略。当我们不支持写分配的情况下，写指令只会更新主存数据，然压就结束了。当支持写分配的时候，我们首先从主存中加载数据到cache line中 (相当于先做个读分配动作)，然后会更新cache line中的数据。

<font color=red> **歧义（ambiguity）**</font>
&emsp;&emsp;歧义是指不同的数据在 Cache 中具有相同的 Tag 和 index。例如：A 进程将物理地址 0x1000 映射到虚拟地址的 0x5000 ，B 进程将虚拟地址 0x1000 映射到物理地址 0x3000 。当 CPU 从 A 进程切换到 B 进程时，在访问虚拟地址 0x1000 时，此时 B进程 的 虚拟地址的 Tag 和 index 与 A 进程在访问虚拟地址 0x1000 时相同，则会出现 Cache 虽然命中，但访问到的数据并不是 B进程需要的数据，此时便出现 Cache歧义。

&emsp;&emsp;歧义问题可以经过操作系统进行避免，当进行进程切换时，对 Cache 进行清除（Flush ），即清除 Cache 中的所有数据，在进程切换后，CPU 读数据时，在进行加载。但是由于进程切换后开始时会有大量的 Cache miss 发生，故会对 Cache 的性能有很大影响。

<font color=red> **别名（alias）**</font>
&emsp;&emsp;当不同的虚拟地址映射相同的物理地址，而这些虚拟地址的 index 不同时就会发生别名问题。

---
## Cache 的一般设计
---
### Cache 的映射方式

#### 直接映射
&emsp;&emsp;内存与 Cache 采用直接映射（direct-mapped）方式关联示意图如下图1所示：
&nbsp;
![直接映射Cache](https://img-blog.csdnimg.cn/913ef3845bc645f3bd6f407f5b29b6f3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)


<center>图1 直接映射</center>

&nbsp;

&emsp;&emsp;如上图1 所示，Cache 与内存采用直接映射方式，其中 Cache 的大小（Cache Size）为 64 Bytes，缓存行（Cache line）的大小为 8 bytes。

&emsp;&emsp;直接映射采用内存与 Cache 多对一的方式，是最简单的地址映射方式，并且其硬件设计简单，成本低，地址变换速度快，而且不涉及**替换算法**问题。但是这种方式不够灵活，Cache 的存储空间并不能得到充分利用，每个主存块只有一个固定位置可存放，容易产生冲突，使 Cache 效率下降，因此只适合大容量 Cache 采用。例如，如果一个程序查询的地址为 0x1000_0000 后查找 0x1000_0040 之后再去查找 0x1000_0000、0x1000_0040，每次查找的地址正好跨过了 Cache 的大小，此时，每次查询 Cache 时，都会出现 Cache 缺失（miss），这会使 Cache 不断的进行替换，这种现象称为 **Cache 颠簸**。最好的方式是将主存 0x1000_0000 与 0x1000_0040 同时复制到 Cache 中，但由于是直接映射的方式，它们都只能复制到 Cache 的第0个 Cache line 中去，即使 Cache 中别的存储空间空着也不能占用，因此这两个块会不断地交替装入Cache中，导致 Cache 命中率降低。

<font color=red>**优点**</font>
* **硬件实现简单，成本低**

 <font color=red>**缺点**</font>
*  **灵活性差**
每个主存块只有一个固定的行可以存放，因此即便cache中有大量空闲空间可用，某个cache块所存储的内容仍可能被替换出去。如果cache容量比较小，则非常容易发生冲突，频繁替换，效率大大降低。
* **直接映射方式一般用于大容量的 Cache 中。**
* **会造成 Cache 的颠簸。**

#### 全相联映射
&emsp;&emsp;在全相连（fully-associative）的方式中，对于一个存储器地址来说，它的数据可以放在任意一个 Cache line 中。全相联映射方式内存与 Cache 关联示意图如下图2所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/5dde174c1b6d427f8db82dc1050f8068.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)
<center>图2 全相联映射</center>
&nbsp;

&emsp;&emsp;如上图2所示，内存中任何一块都可以映射到 Cache 中的任何一个 Cache line 中。

<font color=red>**优点**</font>
* **灵活性好**
Cache中只要有空行，就可以调入所需的主存数据块。

 <font color=red>**缺点**</font>
* **利用效率不高**
因为存在了一个m位的标记位，使cache的行包含了一些对存储无用的信息。
* **速度慢、硬件成本高**
每次访问cache时，需将一个一个遍历并比较标记，才能判断所需主存的字块是否在 Cache 中。


#### 组相联映射
&emsp;&emsp;以两路组相联映射为例
&nbsp;
![组相联映射](https://img-blog.csdnimg.cn/905d8b1e797f462ebf83e6072de914e0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)

<center>图3 组相联映射</center>
&nbsp;

&emsp;&emsp;组相联（set-associative）映射实际上是直接映射和全相联映射的折中方案，存储器中的一个数据不单单只能放在一个 Cache line中，而是可以放在多个 Cache line 中，对于一个组相连结构的 Cache 来说，如果一个数据可以放在 n 个位置，则称这个 Cache 是 n 路组相连的 Cache (n-way set-associative Cache)。

&emsp;&emsp;组相联映射可看成内存与 Cache的 **组** 之间采用直接映射，而 **组内** 采用全相联映射。即在图3中，Cache的组0的 Cache line 0~3可以存储 0x1000_0000、0x1000_0008、0x1000_0010、0x1000_0018 中任意一个地址块，也可以存储 0x1000_0040、0x1000_0048、0x1000_0050、0x1000_0058 中任意一个地址块，如下图4所示：

![组相联映射内部](https://img-blog.csdnimg.cn/eb1fd95eccdd4513942c947d5c6ecd97.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)
<center>图4 组相联映射内部</center>
&nbsp;

<font color=red>**特点**</font>

&emsp;&emsp;组内有一定的灵活性，而且因组内行数较少，比较的硬件电路比全相联方式简单。并且空间利用率比直接映射方式要高。

---
### Cache的地址变换

#### 直接映射方式地址变换
&emsp;&emsp;直接映射方式地址变换如下图5所示：
&nbsp;
![直接映射地址转换](https://img-blog.csdnimg.cn/a78f87a5f7fd4ecea6dcd01aef66ba5d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)

<center>图5 直接映射地址变换</center>
&nbsp;
&emsp;&emsp;直接映射的 Cache 存储器，在处理器访问存储器时，地址被分为三个部分，Tag、index 和 Offset，如上图5所示，首先，通过 index 在 Cache 中找到对应的 Cache line ，由于所有 index 相同的地址都会所引导相同的 Cache line ，这时就需要 Cache line 中 Tag 部分与传来地址的 Tag 部分作比较，如果匹配，则 Cache 命中（hit），不匹配则 Cache 缺失（miss）。接下来通过传来地址的 Offset 部分就可以定位到需要查找的字节。在 Cache line 中还有一个有效位（valid），用来保存当前的 Cache line 是否有有效的数据。

#### 全相联方式映射地址变换
&emsp;&emsp;全相联映射地址变换如下图7所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/279b83f0d8ce4ab990bdcf4cca91aed2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)

<center>图6 全相联映射地址变换</center>
&nbsp;

&emsp;&emsp;全相联映射对于存储器地址来说，它可以存储在任意一个 Cache line 中，即相当于省略了 index 部分，而是在整个 Cache 中对每个 Cache line 的 Tag 部分进行比较，查找到比较结果相等的 Cache line ，这种方式相当于直接使用存储器的内容来寻址。全相连结构的 Cache 有着最大的灵活度，因此它的缺失率是最低的，但是很明显可以看出，由于有大量的内容需要进行比较，它的延迟也是最大的，因此一般这种结构的 Cache 都不会有很大的容量。

#### 组相联映射方式地址变换
&emsp;&emsp;两路组相联映射地址变换如下图7所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/810d29c4c34641b0b961f076358f2676.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBARWRkeV9s,size_1,color_FF0000,t_70,g_se,x_16#pic_center)
<center>图7 2路组相联映射地址变换</center>
&nbsp;

&emsp;&emsp;组相联映射的数据会存放于多个 Cache line 中，在处理器访问存储器时，使用 index 对 Cache line 进行寻址，此时会得到两个 Cache line ，接下来同时对两个 Cache line 的 Tag 部分与传入的地址的 Tag 部分进行比较，找出对应的 Cache line。最终通过有效位（valid）位决定此 Cache line 是否有效，若有效，则根据传入的 Offset 找到对应的数据。



---

🎄
🎨
🎱

---

### 参考资料
《超标量处理器设计-姚永斌》

知乎专栏：[高速缓存与一致性](https://www.zhihu.com/column/cpu-cache)

博客园：[Cache Line操作和Cache相关概念介绍](https://www.cnblogs.com/gujiangtaoFuture/articles/11163844.html )

哔哩哔哩：[计算机组成原理cache组相联映射](https://www.bilibili.com/video/BV1Nh411t7Uh?from=search&seid=16797575022441646681)