---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247486702&idx=1&sn=7177aa889ddfaf8ff2da30aea48f037f&chksm=c32fd4a1910f4e99a84e40e0e2c181dccaa1052322d6313ead276c2db8db9c23d3920c5d990f&mpshare=1&scene=1&srcid=0604i087W8nO1KJopsnyxBmH&sharer_shareinfo=cefac1dc62ce8c3845761b7ba8ab08b3&sharer_shareinfo_first=cefac1dc62ce8c3845761b7ba8ab08b3#rd
saved: 2026-06-04 08:33:57
tags:
  - 笔记同步助手
id: 5be4b440-0827-41a4-bec7-26eec0f35435
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-06-03 20:21

![[Inbox/笔记同步助手/微信公众号/20260604/images/d5673b61529048fc2392d6b1f5e4601a_MD5.gif]]

大家好，我是蟹老板～

前几天后台有个的兄弟留言说：“哥，Linux驱动到底是在写啥？不就是`read`、`write`、`ioctl`那一套吗？”

兄弟，你那是应用层接口，那是皮，Linux设备驱动模型的精髓，在于皮囊底下的骨架和经络，你如果只盯着`file_operations`看，那你永远只能写写简单的字符设备，一旦碰到复杂的总线、热插拔、电源管理，立马凉凉。

Linux驱动模型其实就干了一件事：**让设备和驱动能够自动找到彼此。**

![[Inbox/笔记同步助手/微信公众号/20260604/images/889f842a66f836340c64b038e91208c6_MD5.jpg]]

## 一、 驱动模型底层基石：内核核心对象体系

Linux内核是用C语言写的，但面向对象的思想却无处不在。为了把系统里的各种总线、设备、驱动给统一管起来，内核整出了三个最底层的核心对象：kobject、kset 和 ktype。

### 1.1 kobject：内核最小基础对象

![[Inbox/笔记同步助手/微信公众号/20260604/images/d5e320388698b79f684050c60f21cbd3_MD5.jpg]]

内核代码里有个说法：**一切皆 `kobject`**，任何一个内核对象要能被统一管理，都需要一个身份证——这个身份证就是`struct kobject`。

```
struct kobject {
    const char *name;                 // 名字，对应sysfs下的一个目录
    struct list_head entry;           // 链表节点，用于组成双向链表
    struct kobject *parent;           // 父对象指针，体现出层级关系
    struct kset *kset;                // 当前kobject所属的集合
    struct kobj_type *ktype;          // 当前kobject的类型信息
    struct kernfs_node *sd;           // sysfs文件系统的目录项
    struct kref kref;                 // 引用计数
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};
```

别看这一堆字段，重点就三个：**name**（名字）、**parent**（父子关系）、**kref**（引用计数）、**ktype**（类型）。

`kref`就是内核里的"智能指针"原型，说白了就是一个计数器——有人用，计数就加1；用完就减1，减到0的时候调用`release`方法释放对象。内核用这个机制管理生命周期，防止用着用着对象被人删了。

`parent`指针构建起了 kobject 之间的**父子层级关系**，这种层级关系在内核中形成了一种树状结构。USB控制器下面挂USB设备，USB设备下面又挂USB接口的功能单元，sysfs里就是一层一层目录套目录。

但是这里有个坑：**kobject本身并不单独使用**。

Greg Kroah-Hartman在他那篇经典文章里说过这么一句话：\*\*内核代码很少创建独立的kobject。相反，kobject被嵌入到其他结构体中，用于控制对更大的、特定领域对象的访问。\*\*用面向对象的说法就是——kobject是所有设备对象的"基类"，你写一个具体的设备驱动，定义的结构体里第一件事就是塞一个kobject进去，比如UIO驱动的`struct uio_map`：

```
struct uio_map {
    struct kobject kobj;
    struct uio_mem *mem;
};
```

那给定一个指向kobject的指针，怎么拿到外层的`uio_map`？总不能让每个人都去算偏移量吧。

内核提供了`container_of`宏来解决这个问题：

```
struct uio_map *u_map = container_of(kp, struct uio_map, kobj);
```

这个宏在驱动开发的代码里到处可见，是C语言实现"继承"和"多态"的核心。

### 1.2 kset：对象集合容器

![[Inbox/笔记同步助手/微信公众号/20260604/images/1cbc3e3e34e753624753f175e6204989_MD5.jpg]]

既然有了 `kobject` 这种单个的对象，那如果有一堆同类型的对象凑在一起，总得有个容器来装吧？那谁管一堆对象的集合？

这时候 `kset` 就粉墨登场了。

```
struct kset {
    struct list_head list;        // 包含在kset内的所有kobject构成的双向链表
    spinlock_t list_lock;         // 链表锁
    struct kobject kobj;          // kset自己的kobject，所以kset本身也是一个kobject
    const struct kset_uevent_ops *uevent_ops;  // 热插拔事件操作
};
```

kset是个容器，里面有个双向链表`list`，把所有属于它的kobject串起来。同时kset自己还嵌了一个kobject，所以kset本身在sysfs里也表现为一个目录。

比如说，你在`/sys/bus/`下看到的所有目录——`i2c`、`spi`、`pci`、`platform`——每个目录对应一个kset。每个总线是一个kset，总线下面挂的设备和驱动各自又是kobject，全都挂在总线kset的链表里。

sysfs的目录结构很大程度上就是根据kset来组织的。

### 1.3 ktype：对象属性操作模板

![[Inbox/笔记同步助手/微信公众号/20260604/images/692cb74f7a3c3f5d21221a69773e514d_MD5.jpg]]

刚学驱动那会儿，我经常把 `kset` 和 `ktype` 搞混，心想这俩货不都是分类的意思吗？

其实差别大了去了，`kset` 关注的是**组织结构**，管的是“把哪些对象放在一起”；而 `ktype` 关注的是**行为和属性**，管的是“这群对象应该怎么放屁、怎么拉屎”。

ktype长这样：

```
struct kobj_type {
    void (*release)(struct kobject *kobj);          // 释放函数
    const struct sysfs_ops *sysfs_ops;              // sysfs操作函数
    struct attribute **default_attrs;               // 默认属性列表
};
```

每个嵌入了kobject的结构体都需要一个对应的ktype。ktype控制着kobject在创建和销毁时应该做什么——怎么释放、sysfs里的属性文件该怎么读写。

说白了，kobject定义了"有什么"，ktype定义了"怎么操作"。

### 1.4 三大对象的层级关系

![[Inbox/笔记同步助手/微信公众号/20260604/images/c6028e0e91de90e63cc6303980da00fc_MD5.jpg]]

总结一下吧：**kobject+kset+ktype=整个Linux驱动模型的底层骨架**。

-   • **kobject**给每个内核对象一个"身份证"
    
-   • **kset**把这些"身份证"按类别放到不同的"抽屉"里
    
-   • **ktype**规定了每个抽屉里的文件的读写规则
    

这个底层骨架就是sysfs的"活水源"。sysfs下的每个目录对应一个kobject，每个属性文件通过ktype里的`sysfs_ops`来读写。

device、driver、bus、class，都建立在这套体系之上。

## 二、驱动模型三大核心支柱：总线、设备、驱动

驱动模型的灵魂就一句话：**总线负责连接，设备挂在总线，驱动匹配设备。**

![[Inbox/笔记同步助手/微信公众号/20260604/images/cd2fd50a27ebd32dec50463d4b48ad6f_MD5.jpg]]

把这个逻辑刻在脑子里。它解释了驱动的整个生命周期：从注册开始，经过匹配，执行probe，最后正确运行。

当年我第一次听到这个概念的时候，心里这不脱裤子放屁吗？我直接写个驱动把硬件操作写死，不也能跑得飞起？

直到后来我们组做了一款产品，同一个视频解码芯片，今天要挂在 I2C 总线上读写配置，明天因为方案变动，要改挂到 SPI 总线上。要是按照老一套的写法，那几天我别想睡觉了，得把整个驱动重构一遍。但如果用了这套模型，我只需要改一下设备端的总线归属，驱动层的核心逻辑连一个字都不用动！

### 2.1 总线：设备与驱动的桥梁

![[Inbox/笔记同步助手/微信公众号/20260604/images/28e952965209d80132a0188d1d6269ff_MD5.jpg]]

在内核里，总线是用 `struct bus_type` 结构体来表示的，官方文档里是这么说的：总线是处理器和一个或多个设备之间的通道。

站在设备模型的角度，**所有设备都通过总线连接，即使是内部的、虚拟的platform总线也是如此**。

`bus_type`结构体里有些关键字段，我们来看一眼：

```
struct bus_type {
    const char *name;                     // 总线名称，如"platform"、"i2c"
    const char *dev_name;
    struct device *dev_root;
    const struct attribute_group **bus_groups;
    const struct attribute_group **dev_groups;
    const struct attribute_group **drv_groups;
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);
    // ... 还有PM电源管理相关函数
};
```

```
};

struct subsys_private *p;                // 驱动核心私有数据
struct lock_class_key lock_key;
```

重点说说这个`match`函数指针——**每当总线上有新设备或新驱动加入，总线的match函数就会被调用，判断驱动和是否支持该设备**。匹配成功，则事情还没完，后续还会调用`probe`完成真正的初始化。

内核里预定义了很多总线类型：**platform\_bus\_type**、**i2c\_bus\_type**、**spi\_bus\_type**、**pci\_bus\_type**……

我刚开始做嵌入式驱动的时候，一直不理解为什么GPIO控制器、UART控制器都要注册成platform设备，明明这些硬件不是真的通过物理总线连接的。后来才明白——**platform总线是Linux内核定义的一条虚拟总线**，专门用来管理那些没办法归类到I2C、SPI、USB等实体总线的设备。

ARM SoC里的片内外设几乎全挂在这条虚拟总线上。

### 2.2 设备：硬件实体的内核抽象

![[Inbox/笔记同步助手/微信公众号/20260604/images/d1a476033d0b9d805871a531312dc4ce_MD5.jpg]]

接下来看设备，设备就是**硬件资源的内核抽象，告诉系统"我是谁，我有什么"**。

不管你的硬件是一个高大上的PCIe显卡，还是一个几毛钱的LED小灯，在内核眼里，它都被抽象成了 `struct device`。

`struct device`这个结构体定义特别大，我挑几个核心的说：

-   • **name**：设备名称，用于匹配驱动
    
-   • **bus**：指向该设备所属的总线（必须明确）
    
-   • **parent**：父设备指针，体现了硬件拓扑关系
    
-   • **platform\_data**：设备相关的硬件特定数据（寄存器地址、中断号、DMA通道等）
    
-   • **of\_node**：如果使用了设备树，这个字段指向对应的设备树节点
    

注册设备的调用流程是这样的：

```
设备注册调用device_add()
  -> bus_add_device()   // 把设备添加到总线的设备列表里
  -> bus_probe_device() // 检查已有驱动能否匹配
  -> 如果已经注册的同总线驱动，match如果成功，触发probe
```

**设备和驱动的匹配逻辑**：

设备注册时，它的所属总线b会遍历drivers链表，对每个驱动调用`bus.match`；如果先前注册了驱动，逻辑同样，设备注册时也会尝试已经注册的驱动。

### 2.3 驱动：硬件操作逻辑载体

![[Inbox/笔记同步助手/微信公众号/20260604/images/eaff53bb76701ab6c627ac4072783f54_MD5.jpg]]

驱动是啥？驱动就是\*\*"负责操作和控制具体硬件的代码模板"\*\*，告诉系统"我能操作哪些设备，怎么操作"。

`struct device_driver` 就是那个负责干苦力的工人，核心字段：

-   • **name**：驱动名称
    
-   • **bus**：这个驱动属于哪个总线
    
-   • **probe/remove/shutdown/suspend/resume**：核心回调函数
    
-   • **of\_match\_table**：设备树匹配表（驱动开发中太重要了）
    
-   • **owner**：所属内核模块
    

注册驱动的流程和注册设备互为镜像：

```
驱动注册调用driver_register()
  -> bus_add_driver()   // 把驱动添加到总线的驱动链表
  -> driver_attach()    // 遍历总线上所有设备，调用bus.match进行匹配
  -> 匹配成功就调用driver.probe（对应platform_driver里的probe）
```

平台总线上的驱动注册尤其要注意设置`drv->driver.bus = &platform_bus_type`，内核代码里`__platform_driver_register`强制把驱动的总线类型指向`platform_bus_type`。这意味着所有platform驱动都被自动归到platform总线的麾下。

### 2.4 三者联动核心关系

![[Inbox/笔记同步助手/微信公众号/20260604/images/c98d49a512654e67e01429297376cf42_MD5.jpg]]

很多面试官喜欢问驱动模型核心思想是什么？

直接回答：**总线管理设备和驱动，匹配成功后触发probe完成初始化**，基本不会错。

## 三、核心工作机制：设备与驱动的匹配全过程

这里说驱动开发最核心的逻辑：**设备和驱动怎么自动绑定？谁调用probe？**

很多刚接触驱动的小伙伴天天都在纳闷：我代码里明明就写了一个 `probe` 函数，我也没在别的地方调它啊，它怎么自己就莫名其妙地跑起来了？这就涉及到了内核里最核心的匹配绑定机制。

### 3.1 内核匹配的两大核心场景

![[Inbox/笔记同步助手/微信公众号/20260604/images/d550152f4f0c81caf2c133350f028d87_MD5.jpg]]

在实际开发中，设备和驱动谁先注册，完全取决于系统的加载顺序。内核必须做到无论谁先来，最后都能正确相遇。

场景一：**先注册设备，后注册驱动**

这种情况在现代Linux系统里挺常见的。系统启动的时候，内核首先去解析设备树或者板级文件，把硬件设备一个接一个地注册到对应的总线上。这时候，由于对应的驱动模块还没加载（可能还在外设的 ko 文件里躺着呢），这些设备就只能静静地在总线的设备链表里当单身狗。过了会儿，用户态通过 `insmod` 或者 `modprobe` 把驱动模块给加载进来了。驱动一注册，内核就会带着这个驱动去遍历总线上的那条设备链表，挨个比对，最后把属于它的设备给领走。

场景二：**先注册驱动，后注册设备**

这种场景经常发生在热插拔设备上。比如你写了一个USB摄像头的驱动，系统一开机就把驱动给加载好了。这时候由于你还没插摄像头，总线的设备链表里啥也没有。等到哪天你突然把摄像头往USB接口上一怼，USB总线控制器捕获到了电信号的变化，立马在内核里动态创建了一个 `struct device` 并注册。新设备一进来，内核就拿着它去遍历已经存在的驱动链表，正好发现了那个嗷嗷待哺的摄像头驱动，当场一拍即合。

### 3.2 四大匹配规则优先级（面试高频，我能证明 ）

![[Inbox/笔记同步助手/微信公众号/20260604/images/ea2cfeb1c91117f5ad14e3f7d539d0e0_MD5.jpg]]

如果你去面试Linux驱动岗位，十个面试官有九个会问：**platform总线如何完成设备和驱动的匹配？** 标准答案就是这四大规则：

## 1\. 设备树匹配（最高优先级）

当内核开启设备树支持时，总线会调用`of_driver_match_device()`，对比设备节点的`compatible`属性和驱动`of_match_table`中定义的兼容性列表。

```
static const struct of_device_id my_of_match[] = {
    { .compatible = "lckfb,mychardev" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_driver = {
    .driver = {
        .name = "mychardev",
        .of_match_table = my_of_match,
    },
    .probe = my_probe,
};
```

设备树里这么写：

```
mychardev@0 {
    compatible = "lckfb,mychardev";
    reg = <0x12340000 0x1000>;
    interrupts = <56>;
};
```

只要compatible对上了，内核就知道这个驱动支持这个设备节点。

## 2\. 设备ID表匹配

非设备树的场景（比如ACPI系统或者老式板级文件），使用`platform_device_id`表进行匹配。

```
static const struct platform_device_id my_id_table[] = {
    { "mychardev-v1", (kernel_ulong_t) &my_device_data_v1 },
    { "mychardev-v2", (kernel_ulong_t) &my_device_data_v2 },
    { }
};
MODULE_DEVICE_TABLE(platform, my_id_table);
```

## 3\. 名称字符串匹配

这是最原始、最直接的一种匹配方式。如果设备树匹配和ID表匹配都没命中，内核会退而求其次，比较设备的`name`字段和驱动的`driver.name`字段是否相同。

驱动里设置：

```
static struct platform_driver my_driver = {
    .driver = {
        .name = "mychardev",   // 驱动名称
    },
};
```

设备的名称也要是`mychardev`才能匹配上。

不过这种匹配方式太死板，灵活性差，平台驱动开发中我基本不推荐用，除非有特殊限制。

## 4\. 自定义match匹配（最低优先级）

总线可以注册自己的`match`函数，完全自定义匹配逻辑。比如说，pci总线就有自己复杂的匹配规则，不按常理出牌。不过一般业务驱动开发不会用到这一层，总线框架自己玩的东西。

### 3.3 probe函数触发完整流程

![[Inbox/笔记同步助手/微信公众号/20260604/images/3ab4c15f6f9878c9462e6b2d820c1dfd_MD5.jpg]]

一旦匹配成功，内核接下来要干的事情就顺理成章了，总线会调用内核的统一接口，顺藤摸瓜一路回调到你写在驱动里的 `probe` 函数。

拿platform总线来举例。前面提到，`__platform_driver_register`会把驱动的`probe`回调绑定到总线的`probe`层，最终调用链是这样的：

注册时调用`driver_register()` → `bus_add_driver()` → `driver_attach()` → `__driver_attach()` → 调用`driver_match_device()`进行匹配判断 → 如果匹配，调用`driver_probe_device()` → 最终调用`really_probe()` → 调用总线或驱动的`probe`函数。

针对platform总线，platform\_drv\_probe里做了几件重要的事情：

1.  1\. 将通用`struct device`指针转换为platform专有的`struct platform_driver`和`struct platform_device`
    
2.  2\. 设置设备节点的默认时钟属性
    
3.  3\. 将设备附加到电源域
    
4.  4\. 最后调用开发者实现的`drv->probe(dev)`
    

所以，你写在platform\_driver结构体里的probe函数，确实是在匹配成功后由内核自动调用的。**你不用自己调用，系统会帮你调用，条件是把设备和驱动注册到同一条总线上，match返回成功**。

probe里一般要做什么？

-   • 从`platform_device`中提取硬件资源（寄存器地址、IRQ号等）
    
-   • 申请字符设备号和注册cdev（字符设备驱动场景下）
    
-   • ioremap映射寄存器地址
    
-   • 申请中断并注册中断处理函数
    
-   • 初始化硬件状态
    
-   • 最后通常是创建设备类class和自动设备节点，这样用户空间才能通过设备节点访问硬件
    

### 3.4 remove函数解绑机制

![[Inbox/笔记同步助手/微信公众号/20260604/images/80e7086ac8108d1721f5c764ed2424c1_MD5.jpg]]

有probe就必须有remove，不然驱动卸载时内存就泄露得不成样子。remove函数在驱动卸载时被内核调用，职责是回收probe里占用的所有资源：

-   • 释放内存映射（iounmap）
    
-   • 释放中断（free\_irq）
    
-   • 注销cdev
    
-   • 释放设备号
    
-   • 删除自动创建设备节点
    

这个流程和 `probe` 刚好完全相反。在 `probe` 里你拿了多少好处，在 `remove` 里你就得一五一十全部吐出来。

对于platform驱动，新版本的Linux内核推荐使用`remove_new`（返回void），因为原来的`remove`返回int但返回值基本被忽略，新版本语义更清晰。

## 四、进阶核心：设备类、sysfs与udev用户态交互机制

Linux设备模型除了搞定内核内部的驱动匹配，还要解决一个问题：**用户空间的应用程序怎么访问硬件？**

答案藏在/sys和/dev这两个目录里。

### 4.1 设备类：设备分类管理体系

![[Inbox/笔记同步助手/微信公众号/20260604/images/aa39113d58e5702e3c502b8903cb1b9a_MD5.jpg]]

`struct class`是一个更高层次的抽象，它按功能对设备进行"分类"。注意，class和设备映射到具体总线的逻辑是独立的，这是Linux设备模型的一大特点——分类和物理连接是解耦的。

这个结构体的主要作用就是在/sys/class下创建设备的分类视图。

在字符设备驱动里，我们经常会这么写：

```
static struct class *my_class;
my_class = class_create(THIS_MODULE, "my_device_class");
device_create(my_class, NULL, devno, NULL, "mydevice");
```

这样做的结果是：

-   • 在/sys/class/my\_device\_class下面会生成一个软链接（或子目录）指向实际的设备路径
    
-   • 同时，`/dev/mydevice`会自动生成（配合udev机制）
    

class\_create和device\_create把设备模型的底层逻辑和用户空间连接起来了。

### 4.2 sysfs文件系统：内核设备可视化

![[Inbox/笔记同步助手/微信公众号/20260604/images/18a4a18fa3206f9a979b52bdc0dd2e8a_MD5.jpg]]

sysfs是一个虚拟文件系统，默认挂载在`/sys`目录。它把内核的数据结构以文件和目录的形式暴露给用户空间，让你像操作普通文件一样读取和修改内核参数。

sysfs的目录结构有固定的三个核心入口：**/sys/bus**、**/sys/devices**、**/sys/class**。

-   • **/sys/devices**是设备树的"原始副本"，按照物理连接层次展示所有设备（一个很大的树结构）
    
-   • **/sys/bus**以总线的维度展示设备和驱动：每个总线目录下包含`devices`和`drivers`两个子目录，分别存放注册到该总线的设备和驱动
    
-   • **/sys/class**按设备功能分类，方便用户态程序按用途查找设备
    

sysfs里那些属性文件是怎么来的呢？每个kobject有自己的`ktype`，ktype里定义了`sysfs_ops`和一组默认属性。当你调用`device_add`注册设备时，内核会通过`sysfs_create_dir`为kobject创建sysfs目录，并调用`sysfs_create_file`等函数创建属性文件。

通过sysfs调试设备参数实在是太方便了。举个例子，我要看一个platform设备的中断号是多少，直接`cat /sys/devices/platform/xxx/uevent`就能看到硬件信息（虽然最终用户态udev也能看到）。更常用的做法是：**echo 0 > /sys/module/xxx/parameters/debug** 来开关驱动debug日志。没有sysfs，这种运行时配置就难搞多了。

### 4.3 udev机制：自动创建设备节点

![[Inbox/笔记同步助手/微信公众号/20260604/images/8ad4d55d1545224ab57d30616f80b8a9_MD5.jpg]]

很多刚入门的开发者觉得很神奇——为什么驱动模块加载后，/dev目录下会自动出现我想要的设备节点？这背后是udev在发力。

系统启动阶段，内核通过devtmpfs在/dev下生成基础设备文件。然后udevd守护进程（用户空间）监听内核通过netlink发送的uevent事件，解析内核传递的环境变量，匹配/etc/udev/rules.d/下的规则，决定如何创建/删除设备节点、如何设置权限和属主、甚至加载/卸载驱动模块。

具体工作流程分两步：

1.  1\. 驱动程序通过class\_create和device\_create在/sys/class下创建分类标识和设备信息
    
2.  2\. udevd捕获到内核的uevent事件后，在/dev下动态生成对应的设备节点
    

这对用户空间应用程序来说，设备节点是确定的稳定入口，无需关心设备什么时候插拔。我U盘插上去瞬间/dev/sdb1就出现了，这是udev干的活。

**做个冷热插拔的小实验**：

当你注册一个设备节点时，驱动代码里调用了`device_create`，内核会通过kobject\_uevent函数向用户空间发送"add"事件，udevd抓到事件后运行处理程序创建设备节点。

反过来，移除驱动时发送"remove"事件，udev删除节点。

在嵌入式设备开发中，很多人用`mdev`（一个轻量化的udev实现，常见于Busybox系统）来处理设备节点的动态创建。它在资源受限的环境中非常实用，功能虽比完整udev少，但核心设备节点自动创建的功能一点也不含糊。

## 五、实战：模型如何驱动真实硬件

### 5.1 案例1：platform总线 + 字符设备驱动开发流程

结合前面的理论，我们从头撸一个完整的platform字符设备驱动代码框架。

这个例子比较经典，可能和你平时写的有些出入，但核心逻辑是一样的：**platform\_driver负责匹配设备，cdev负责字符设备操作，sysfs和udev负责与用户空间的交互**。

驱动部分框架如下：

```
#include 
#include 
#include 
#include 
#include 
#include 

#define DEV_NAME "mychardev"

static int major;
staticstruct class *my_class;

static int my_open(struct inode *inode, struct file *file)
{
    printk(KERN_INFO "my device opened\n");
    return 0;
}

static ssize_t my_read(struct file *file, char __user *buf, size_t len, loff_t *off)
{
    printk(KERN_INFO "my device read\n");
    return 0;
}

static ssize_t my_write(struct file *file, const char __user *buf, size_t len, loff_t *off)
{
    printk(KERN_INFO "my device written\n");
    return len;
}

static conststruct file_operations my_fops = {
    .owner = THIS_MODULE,
    .open = my_open,
    .read = my_read,
    .write = my_write,
};

static int my_probe(struct platform_device *pdev)
{
    printk(KERN_INFO "my_probe called!\n");
    
    // 动态申请设备号并注册cdev
    major = register_chrdev(0, DEV_NAME, &my_fops);
    if (major < 0) {
        printk(KERN_ERR "register_chrdev failed\n");
        return major;
    }
    
    // 自动创建设备节点需要class
    my_class = class_create(THIS_MODULE, DEV_NAME);
    if (IS_ERR(my_class)) {
        unregister_chrdev(major, DEV_NAME);
        return PTR_ERR(my_class);
    }
    device_create(my_class, NULL, MKDEV(major, 0), NULL, DEV_NAME);
    
    return 0;
}

static int my_remove(struct platform_device *pdev)
{
    printk(KERN_INFO "my_remove called!\n");
    device_destroy(my_class, MKDEV(major, 0));
    class_destroy(my_class);
    unregister_chrdev(major, DEV_NAME);
    return 0;
}

static conststruct of_device_id my_of_match[] = {
    { .compatible = "lckfb,mychardev" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, my_of_match);

staticstruct platform_driver my_driver = {
    .probe = my_probe,
    .remove_new = my_remove,
    .driver = {
        .name = "mychardev",
        .of_match_table = my_of_match,
    },
};

module_platform_driver(my_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Platform Char Device Demo");
```

编译加载后：

```
insmod mydriver.ko
cat /proc/devices            # 看到mychardev对应主设备号
ls /dev/mychardev            # 设备节点已生成
echo "hello" > /dev/mychardev    # 触发write回调
cat /dev/mychardev           # 触发read回调
rmmod mydriver               # 触发remove，设备节点被删除
```

### 5.2 案例2：设备树（Device Tree）的应用

随着设备树的普及，我建议**不要再把硬件信息写到platform\_device结构体里初始化**。维护一个c文件描述I2C/SPI/GPIO的寄存器地址和中断号，会导致内核代码和硬件信息紧耦合，换一块板子就得重新编译内核。设备树就是来救场的——**把板级硬件信息从内核代码抽离出来，放到dts文件中**。

设备树匹配驱动的核心就是`compatible`属性：**驱动中的`of_match_table`和设备树节点的`compatible`能对应上，内核就会匹配并调用probe**。

写DTS文件大致分成两步。

首先，编写dts文件（或dtsi片段）：

```
/ {
    my_device: mychardev@0 {
        compatible = "lckfb,mychardev";
        reg = <0x12340000 0x1000>;
        interrupts = <56>;
        status = "okay";
    };
};
```

然后驱动端声明`of_match_table`，告诉内核我的驱动支持哪些compatible字符串。

内核在启动过程中处理DTB（设备树二进制文件），把它解析成`device_node`树，然后为每个节点创建`platform_device`、`i2c_client`等设备实例，资源（内存地址、IRQ号）也随之传递。驱动中的probe被调用时，从`platform_device`中提取硬件资源并初始化硬件。

### 5.3 调试与工具

做底层驱动开发，写代码往往只占两成时间，剩下八成时间全在调试 Bug。

内核是个小黑盒，要是出了问题，我们怎么知道设备模型里到底哪儿掉链子了？

### （1）sysfs节点监控

-   • `ls /sys/bus/platform/devices/`：查看所有平台设备
    
-   • `ls /sys/bus/platform/drivers/`：查看所有平台驱动
    
-   • `cat /sys/devices/platform/xxx/uevent`：查看设备的硬件资源信息
    
-   • `echo 0 > /sys/module/xxx/parameters/debug`：动态开关驱动调试（前提是模块有可调参数）
    
-   • `tree /sys/class/xxx`：查看设备分类视图，快速定位某个设备是否注册成功
    

### （2）uevent事件的捕获与分析

直接运行`udevadm monitor --property`，可以看到内核发出的完整uevent事件（包括action、devpath、major/minor等关键信息），这对调试设备节点没自动生成的问题非常非常有用。

### （3）内核日志与调试技巧

-   • `dmesg | tail`：实时查看probe/remove日志
    
-   • `echo 8 > /proc/sys/kernel/printk`：提高内核日志输出级别
    
-   • 动态调试：在内核中配置CONFIG\_DYNAMIC\_DEBUG，然后`echo "file mydriver.c +p" > /sys/kernel/debug/dynamic_debug/control`来动态控制打印
    
-   • `insmod`时加上`dyndbg=+p`参数可以启用模块的调试打印
    

就拿uevent监控来说，曾经有一个项目里deamon进程总是拿不到设备节点，我通过`udevadm monitor`发现内核根本没往用户空间发add事件。最后排查到驱动中的`class_create`调晚了一步，device\_add先完成，所以sysfs下设备类目录没生成，udev自然没法工作——这能看出内核和用户空间的交互时机有多敏感。

驱动模型的核心概念：

![[Inbox/笔记同步助手/微信公众号/20260604/images/a6d37c56ee7f675c65a36e0bdc2a2d21_MD5.jpg]]

调试驱动的时候，这三个命令保你平安：

```
udevadm monitor --property   # 看内核发什么事件
tree /sys/class/              # 看设备类是否注册好
dmesg -w                      # 看probe报什么错误
```

如果一个设备匹配了但probe没跑起来，多半是中断号、内存地址、DMA通道那部分资源解析出问题了，去`/sys/devices/platform/<设备>/uevent`里cat一下看资源是不是空的。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/a80ac463_1780533235266?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247486702%26idx%3D1%26sn%3D7177aa889ddfaf8ff2da30aea48f037f%26chksm%3Dc32fd4a1910f4e99a84e40e0e2c181dccaa1052322d6313ead276c2db8db9c23d3920c5d990f%26mpshare%3D1%26scene%3D1%26srcid%3D0604i087W8nO1KJopsnyxBmH%26sharer_shareinfo%3Dcefac1dc62ce8c3845761b7ba8ab08b3%26sharer_shareinfo_first%3Dcefac1dc62ce8c3845761b7ba8ab08b3%23rd&s=obsidian)