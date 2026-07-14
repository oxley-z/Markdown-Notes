---
author: Punch
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247527435&idx=1&sn=dc4b07f53acd071a569ef86bf83fc55e&chksm=f842f23b216d55e94dd0267a8782e80412e5aac8032e906948d4da87e7ebe7f6778cc4d5f1d4&mpshare=1&scene=1&srcid=0714QkgPTDnyJKBBfSKpczwK&sharer_shareinfo=2e2175f650e7b910781640ae1b687bd1&sharer_shareinfo_first=2e2175f650e7b910781640ae1b687bd1#rd
saved: 2026-07-14 10:10:25
tags:
  - 笔记同步助手
id: 020ae413-3771-47cd-880d-dd8c67fcaf50
---

公众号名称：一口Linux

作者名称：Punch

发布时间：2026-07-14 10:00

Linux内核主要包括三种驱动模型：**字符设备驱动**、**块设备驱动**以及**网络设备驱动**。其中，字符设备驱动是Linux驱动开发中最常见、最基础的驱动模型。

本文将从内核源码角度出发，拆解字符设备驱动的机制，涵盖：

-   **字符设备号管理**：内核如何分配和追踪设备号
    
-   **字符设备对象（cdev）**：内核如何抽象和管理字符设备
    
-   **kobj\_map 哈希映射机制**：设备号到 cdev 的快速查找
    
-   **mknod 与 open 系统调用全链路**：从用户态到内核驱动的完整路径
    
-   **共享内存字符设备驱动案例**：简单的字符设备驱动使用方法代码
    

> **📌 提示：**本文基于 Linux 内核 5.x 版本源码进行分析，主要源码文件位于
> 
> `fs/char_dev.c`
> 
> `include/linux/cdev.h`
> 
> `drivers/base/map.c`

---

## **一、字符设备号管理**

本小节的主线是内核如何管理哪些设备号已经分配，哪些设备号可用

### **1.1 dev\_t 设备号概述**

在Linux内核中，每个字符设备都由一个**32 位的设备号（dev\_t）**唯一标识。这个 32 位数值被划分为两部分：

-   **主设备号（Major Number）**：占用高 12 位（bit 12-31），用于标识设备驱动类型
    
-   **次设备号（Minor Number）**：占用低 20 位（bit 0-19），用于区分同一驱动下的不同设备实例
    

内核提供了三个关键宏来操作设备号：

> ```
> #define MINORBITS 20
> #define MINORMASK ((1U << MINORBITS) - 1)
> 
> #define MAJOR(dev) ((unsigned int) ((dev) >> MINORBITS)) /* 提取主设备号 */
> #define MINOR(dev) ((unsigned int) ((dev) & MINORMASK))  /* 提取次设备号 */
> #define MKDEV(ma,mi) (((ma) << MINORBITS) | (mi))        /* 组合生成设备号 */
> ```

  

> **💡 设计思想：**主设备号相当于"设备类型标识"，例如所有I2C设备共享一个主设备号；次设备号则用于区分具体是哪个I2C设备（如 MPU6050 传感器还是 EEPROM 存储器）。这种分层设计使得内核可以在保持设备号空间紧凑的同时，支持大量设备实例

### **1.2 char\_device\_struct 结构体**

内核通过`char_device_struct`结构体来**记录系统中已分配的设备号范围**（主设备号 + 次设备号区间），形成一个资源管理表，防止设备号冲突。内核维护了`chrdevs`哈希表来记录设备号的使用情况：

> /\* 定义在 fs/char\_dev.c \*/  
> \#define CHRDEV\_MAJOR\_HASH\_SIZE 255  
> staticstructchar\_device\_struct{
> structchar\_device\_struct\*next;/\* 将相同哈希值的节点链接成链表 \*/
> unsignedintmajor; /\* 主设备号 \*/
> unsignedintbaseminor; /\* 次设备号起始值 \*/
> intminorct; /\* 次设备号数量 \*/
> charname\[64\]; /\* 设备名称 \*/
> structcdev\*cdev; /\* 内核字符对象（已废弃） \*/
> } \*chrdevs\[CHRDEV\_MAJOR\_HASH\_SIZE\];  

### 🔑 关键设计：内核使用chrdevs维护设备号的分配信息

`chrdevs`数组大小仅为 255，但主设备号范围是 0～511（甚至理论上可以更大）。内核通过取模操作`major % 255`将主设备号映射到哈希桶。具体来说：

-   主设备号 0、255、510... 都落在桶 0
    
-   主设备号 1、256、511... 都落在桶 1
    
-   桶内链表按主设备号从小到大排序
    

这种设计下：在不增加数组规模的前提下，理论上支持无限的主设备号范围，且查找效率不受影响。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/ad7b6c97aa24ff36d8247a6fdce286aa_MD5.jpg]]

### **1.3 \_\_register\_chrdev\_region 函数**

`__register_chrdev_region`是设备号分配的核心函数，负责在`chrdevs`哈希表中**查找或插入**一个空闲的设备号区间，并返回对应的`char_device_struct`指针

> /\* 定义在 fs/char\_dev.c \*/  
> staticstructchar\_device\_struct\*\_\_register\_chrdev\_region(unsignedintmajor,unsignedintbaseminor,intminorct,constchar\*name) {
> structchar\_device\_struct\*cd, \*\*cp;
>  intret\=0;  
> inti;  
> /\* 1. 分配新的 char\_device\_struct 节点 \*/
> cd\=kzalloc(sizeof(structchar\_device\_struct),GFP\_KERNEL);
> if(cd\==NULL)
> returnERR\_PTR(-ENOMEM);
> mutex\_lock(&chrdevs\_lock);
> /\* 2. 如果 major == 0，动态分配主设备号 \*/
> if(major\==0) {
> ret\=find\_dynamic\_major();
> if(ret<0) {
> pr\_err("CHRDEV \\"%s\\" dynamic allocation region is full\\n",name);
> gotoout;
> }
> major\=ret;
> }
> /\* 3. 主设备号范围校验 \*/
> if(major\>=CHRDEV\_MAJOR\_MAX) {
> pr\_err("CHRDEV \\"%s\\" major requested (%u) greater than max (%u)\\n",name,major,CHRDEV\_MAJOR\_MAX\-1);
> ret\= -EINVAL;
> gotoout;
> }
> cd\->major\=major;
> cd\->baseminor\=baseminor;
> cd\->minorct\=minorct;
> strlcpy(cd\->name,name,sizeof(cd\->name));
> /\* 4. 计算哈希桶位置 \*/
> i\=major\_to\_index(major);
> /\* 5. 在链表中找到合适的插入位置（按主设备号、次设备号排序） \*/
> for(cp\= &chrdevs\[i\]; \*cp;cp\= &(\*cp)->next) {
> if((\*cp)->major\>major||
> ((\*cp)->major\==major&&
> ((\*cp)->baseminor\>=baseminor||
> (\*cp)->baseminor\+ (\*cp)->minorct\>baseminor)))
> break;
> }
> /\* 6. 检查次设备号是否冲突（三种重叠检测） \*/
> if(\*cp&& (\*cp)->major\==major) {
> intold\_min\= (\*cp)->baseminor;
>  intold\_max\= (\*cp)->baseminor\+ (\*cp)->minorct\-1;  
> intnew\_min\=baseminor;  
> intnew\_max\=baseminor+minorct\-1;  
> /\* 新范围与已有范围重叠 → 冲突 \*/
> if(new\_max\>=old\_min&&new\_max<=old\_max) {
> ret\= -EBUSY;
> gotoout;
> }
> if(new\_min<=old\_max&&new\_min\>=old\_min) {
> ret\= -EBUSY;
> gotoout;
> }
> /\* 新范围完全覆盖已有范围 \*/
> if(new\_min<old\_min&&new\_max\>old\_max) {
> ret\= -EBUSY;
> gotoout;
> }
> }
> /\* 7. 插入链表 \*/
> cd\->next\= \*cp;
> \*cp\=cd;
> mutex\_unlock(&chrdevs\_lock);
>  return cd;  
> out:
> mutex\_unlock(&chrdevs\_lock);
> kfree(cd);
> returnERR\_PTR(ret);
> }

> **🔑 冲突检测逻辑解析：**内核在插入新设备号时，会检查三种可能的冲突场景：
> 
> 1.  **新范围尾部与已有范围重叠：**`new_max`落在`[old_min, old_max]`区间内
>     
> 2.  **新范围头部与已有范围重叠：**`new_min`落在`[old_min, old_max]`区间内
>     
> 3.  **新范围完全覆盖已有范围：**`new_min < old_min`且`new_max > old_max`
>     
> 
> 只有三种情况都不满足时，才认为设备号范围无冲突，可以安全注册。

### **1.4 find\_dynamic\_major 函数**

当驱动调用`alloc_chrdev_region`（传入 major=0）请求动态分配设备号时，内核会调用`find_dynamic_major`从空闲范围中查找可用的主设备号：

> staticintfind\_dynamic\_major(void){
> inti;
>  structchar\_device\_struct\*cd;  
> /\* 高优先级：在 234～254 范围内查找 \*/
> for(i\=ARRAY\_SIZE(chrdevs) -1;i\>=CHRDEV\_MAJOR\_DYN\_END;i\--) {
> if(chrdevs\[i\] ==NULL)
> returni;
> }
> /\* 低优先级：在 511～384 范围内查找 \*/
> for(i\=CHRDEV\_MAJOR\_DYN\_EXT\_START;i\>=CHRDEV\_MAJOR\_DYN\_EXT\_END;i\--) {
> for(cd\=chrdevs\[major\_to\_index(i)\];cd;cd\=cd\->next) {
> if(cd\->major\==i)
> break;
> }
> if(cd\==NULL)
> returni;
> }
> return\-EBUSY;
> }

## 📌 两级查找策略：

| **优先级** | **查找范围** | **说明** |
| --- | --- | --- |
| 高 | 234 ～ 254 | 传统动态分配区域，"向后兼容"的保留区间 |
| 低 | 511 ～ 384 | 扩展动态分配区域，满足更多设备注册需求 |

相关宏定义：

-   `CHRDEV_MAJOR_MAX 512`
    
-   `CHRDEV_MAJOR_DYN_END 234`
    
-   `CHRDEV_MAJOR_DYN_EXT_START 511`
    
-   `CHRDEV_MAJOR_DYN_EXT_END 384`
    

### **1.5 设备号分配的两大对外接口**

内核提供了两种设备号分配方式，对应不同的使用场景：

| **接口** | **方式** |
| --- | --- |
| `register_chrdev_region()` | **静态分配** |
| `alloc_chrdev_region()` | **动态分配** |

两者最终都调用`__register_chrdev_region()`完成实际注册操作。在Linux系统中，可以通过`cat /proc/devices`查看所有已注册的设备号列表。设备号注销时，无论静态还是动态分配，统一调用`unregister_chrdev_region()`归还资源

---

## **二、字符设备对象（cdev）**

本小节的主线是了解两个结构体的设计，分别是内核字符设备结构体`cdev`，以及管理字符设备的结构体`kobj_map`。

### **2.1 cdev 结构体**

`struct cdev`是内核中表示**字符设备**的核心数据结构。每个字符设备驱动都需要创建一个`cdev`实例，并将其注册到内核中：

> /\* 摘自 include/linux/cdev.h \*/  
> 
> structcdev{
> 
> structkobjectkobj; /\* 内嵌的kobject，用于设备模型管理 \*/
> 
> structmodule\*owner; /\* 指向所属模块，通常为THIS\_MODULE \*/
> 
> conststructfile\_operations\*ops; /\* 设备操作函数集 \*/
> 
> structlist\_headlist; /\* 用于将cdev链接到对应设备号的cdev列表 \*/
> 
> dev\_tdev; /\* 记录该字符设备关联的起始设备号 \*/
> 
> unsignedintcount; /\* 从dev开始连续占用的次设备号数量 \*/
> 
> }\_\_randomize\_layout;  

| **成员变量** | **作用** |
| --- | --- |
| `kobj` | 嵌入的内核对象，使 cdev 能被 sysfs 设备模型管理 |
| `owner` | 指向拥有该设备的模块，用于引用计数管理 |
| `ops` | 指向设备操作函数集（open/read/write/ioctl 等） |
| `list` | 链表节点，用于将使用该 cdev 的 inode 链接起来 |
| `dev` | 起始设备号（主设备号 + 次设备号） |
| `count` | 该 cdev 管理的连续次设备号数量 |

### **2.2 kobj\_map 结构体（哈希表）**

`struct kobj_map`是一个内核内部结构体，定义在`drivers/base/map.c`中，用于**建立设备号（dev\_t）到 struct cdev 的快速查找映射**。

-   `kobj_map`负责将设备号范围映射到对应的`struct cdev`结构体。当内核通过设备号访问字符设备时（如打开设备文件时），会根据设备号在`kobj_map`中查找对应的`struct cdev`，从而获得其操作函数集
    
-   `void *data`指针中保存了`cdev`结构体指针
    
-   `probes`数组是一个哈希桶，每个桶是一个链表，链表节点记录了设备号范围及对应的`data`（通常指向`struct cdev`）和获取`kobject`的函数
    
-   通过`cdev_add`将一个`cdev`添加到系统中时，实际上就是在`kobj_map`中插入一个节点
    

> /\* 摘自 drivers/base/map.c \*/  
> 
> structkobj\_map{
> 
> structprobe{
> 
> structprobe\*next; /\* 链表下一个节点 \*/
> 
> dev\_tdev; /\* 起始设备号 \*/
> 
> unsignedlongrange; /\* 次设备号范围 \*/
> 
> structmodule\*owner; /\* 模块所有者 \*/
> 
> kobj\_probe\_t\*get; /\* 获取kobject的函数（用于查找） \*/
> 
> int(\*lock)(dev\_t,void\*); /\* 锁定函数 \*/
> 
> void\*data; /\* 私有数据，通常指向cdev \*/
> 
> } \*probes\[255\]; /\* 255个桶的哈希表 \*/
> 
> structmutex\*lock; /\* 保护映射表的锁 \*/
> 
> };  

### 🔑 chrdevs 和 cdev\_map —— 两套哈希表的职责分工：

| **哈希表** | **存储内容** | **职责** |
| --- | --- | --- |
| `chrdevs` | 已分配的设备号范围 | 设备号资源管理，防止冲突 |
| `cdev_map` | 设备号到 cdev 映射 | 运行时快速查找设备驱动 |

---

## **三、cdev 注册函数**

本小节的主线是内核如何将用户定义的**`cdev`**结构体注册到内核**`cdev_map`**中

### **3.1 cdev\_init 函数**

`cdev_init`用于初始化一个已经分配的`cdev`结构体，将其与文件操作集关联，并设置内嵌`kobject`的类型：

> voidcdev\_init(structcdev\*cdev,conststructfile\_operations\*fops) {
> 
> memset(cdev,0,sizeof\*cdev); /\* 清零结构体 \*/
> 
> INIT\_LIST\_HEAD(&cdev\->list); /\* 初始化链表头 \*/
> 
> kobject\_init(&cdev\->kobj, &ktype\_cdev\_default); /\* 初始化内嵌kobject \*/
> 
> cdev\->ops\=fops; /\* 绑定文件操作集 \*/
> 
> }  
> EXPORT\_SYMBOL(cdev\_init);  

`EXPORT_SYMBOL`宏将该函数导出到内核符号表，使内核模块（.ko 文件）可以调用它。

### **3.2 cdev\_add 函数**

### **完成设备号，次设备数量等参数记录，并将字符设备注册到内核**

> intcdev\_add(structcdev\*p,dev\_tdev,unsignedcount) {
> interror;
> p\->dev\=dev; /\* 记录起始设备号 \*/
> p\->count\=count; /\* 记录次设备号数量 \*/
> /\* 调用kobj\_map注册，将设备号范围映射到该cdev \*/
> error\=kobj\_map(cdev\_map,dev,count,NULL,exact\_match,exact\_lock,p);
> if(error)
> returnerror;
> /\* 增加模块引用计数，防止模块被卸载 \*/
> kobject\_get(p\->kobj.parent);
> return0;
> }
> EXPORT\_SYMBOL(cdev\_add);  

### **3.3 kobj\_map 函数**

这是将 cdev 插入到全局哈希表`cdev_map`的核心函数，接下来才可以通过设备号查找对应的字符设备，定义在`drivers/base/map.c`中：

> intkobj\_map(structkobj\_map\*domain,dev\_tdev,unsignedlongrange,
> structmodule\*module,kobj\_probe\_t\*probe,
> int(\*lock)(dev\_t,void\*),void\*data) {
> /\* 1. 计算跨越了几个不同的主设备号 \*/
> unsignedn\=MAJOR(dev+range\-1) -MAJOR(dev) +1;
>  unsigned index \= MAJOR(dev);  
> unsigned i;  
> struct probe \*p;  
> if (n \> 255) n \= 255;  
> /\* 安全限制 \*/  
> /\* 2. 分配 n 个连续的 probe 结构体 \*/
> p\=kmalloc\_array(n,sizeof(structprobe),GFP\_KERNEL);
>   
> if (p \== NULL) return\-ENOMEM;  
> /\* 3. 初始化 n 个 probe（每个的 data 都指向同一个 cdev） \*/
> for(i\=0;i<n;i++,p++) {
> p\->owner\=module;
> p\->get\=probe;
> p\->lock\=lock;
> p\->dev\=dev;
> p\->range\=range;
> p\->data\=data;
> }
> /\* 4. 将 probe 节点插入哈希表，按 range 排序 \*/
> mutex\_lock(domain\->lock);
> for(i\=0,p\-=n;i<n;i++,p++,index++) {
> structprobe\*\*s\= &domain\->probes\[index%255\];
> /\* 按 range 大小排序，range 小的在前 \*/
> while(\*s&& (\*s)->range<range)
> s\= &(\*s)->next;
> p\->next\= \*s;
> \*s\=p;
> }
> mutex\_unlock(domain\->lock);
> return0;
> }

> **💡 为什么按 range 排序？**当查找设备号对应的 cdev 时，`kobj_lookup`函数会遍历链表，选择`range`最小且包含该设备号的 probe。这种"最佳匹配"策略确保当多个 cdev 的设备号范围有重叠时，能精确匹配到最具体的那一个。

---

## **四、字符设备号与字符设备对象的关系**

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d719172f3c73563577f7622f93877f78_MD5.jpg]]

-   **`chrdevs`—— 设备号资源管理**  
    记录主设备号的使用区间，防止冲突。在驱动加载时调用`register_chrdev_region`或`alloc_chrdev_region`分配设备号时，进行冲突检测。
    
-   **`cdev_map`—— 运行时查找设备**  
    根据打开设备文件的设备号，快速找到`cdev`结构体，获得`file_operations`。在`cdev_add`函数中调用`kobj_map`函数构建哈希表。
    

---

## **五、次设备号的解析与多实例管理**

本小节用一个非常简单的例子解析为什么需要区分主次设备号，一个主设备号是如何与多个次设备号关联起来的

在实际应用中，**多个功能相同的子设备可以共用同一套驱动程序**：

-   不同子设备功能逻辑一样，操作函数集（`open`、`read`、`write`等）完全相同
    
-   没必要为每个次设备号都分配一个 cdev，只需要一个cdev就能管理多个实例
    
-   用户空间看到三个独立的设备文件（`/dev/device1`对应次设备号`0`，`device2`对应`1`，`device3`对应`2`），但内核中都导向同一个`file_operations`
    

### **5.1 注册 cdev**

> dev\_tdevno\=MKDEV(major,0); /\* 起始次设备号 0 \*/  
> cdev\_init(&my\_cdev, &fops);
> cdev\_add(&my\_cdev,devno,3); /\* 占用次设备号 0, 1, 2 \*/

### **5.2 定义设备私有结构体和数组**

> structmy\_device{
> void\_\_iomem\*regs; /\* 该设备的寄存器映射地址 \*/
> structmutexlock; /\* 互斥锁 \*/
>  /\* ... 其他私有数据 \*/  
> };  
> staticstructmy\_devicedevs\[3\]; /\* 三个实例 \*/  

### **5.3 在 open 中用次设备号绑定私有数据**

> staticintmy\_open(structinode\*inode,structfile\*filp) {
> 
> intminor\=iminor(inode); /\* 获取次设备号 \*/
> 
> if(minor<0||minor\>=3)
> 
> return\-ENODEV;
> 
> structmy\_device\*dev\= &devs\[minor\]; /\* 根据次设备号选择设备 \*/
> 
> filp\->private\_data\=dev; /\* 保存到私有数据中 \*/
> 
> return0;
> 
> }

> **📌 次设备号使用：**这种"一个 cdev → 多个次设备号 → 多个设备实例"的模式是 Linux 驱动开发中的标准做法。通过`iminor(inode)`获取次设备号，再通过`filp->private_data`保存设备私有数据，后续的 read/write/ioctl 操作都能通过`filp->private_data`获取到正确的设备实例

---

## **六、mknod与 open系统调用的完整流程**

本小节介绍从用户态打开一个字符设备驱动的完整流程，关注内核如何通过字符设备号找到对应的字符设备结构体，以及用户定义的操作函数如何被替换

![[Inbox/笔记同步助手/微信公众号/2026/07/images/42dc02a9bff725cfe734372858daaf0b_MD5.jpg]]

### **6.1 系统调用的整体流程**

让我们从用户态的一条命令开始

> mknod /dev/mydevice c 250 0

这个命令触发的过程被拆解为四个关键步骤：

1.  使用`mknod`创建一个字符设备 inode 节点，该节点中保存了默认 open 函数`chrdev_open`。节点定义在`/dev/mydevice`
    
2.  在用户空间使用`open`函数，传入`"/dev/mydevice"`
    
3.  通过一系列系统调用，最后到达`do_dentry_open`函数，该函数会调用`inode`节点对应的默认 open 函数`chrdev_open`
    
4.  `chrdev_open`函数在全局哈希表`cdev_map`中，根据设备号找到用户定义的`f_ops`操作函数，并替换`inode`和`file`中的操作函数。至此，用户自定义的操作函数正式被调用
    

### **6.2 mknod创建 node节点**

在控制台调用`mknod`函数时，经过一系列系统调用，最后会调用`init_special_inode`函数。该函数是 VFS 层处理特殊文件的基石，根据文件类型为`inode`挂载合适的操作函数表，并保存必要的设备号信息。

> voidinit\_special\_inode(structinode\*inode,umode\_tmode,dev\_trdev) {
> 
> inode\->i\_mode\=mode; /\* 保存文件类型和权限 \*/
> 
> if(S\_ISCHR(mode)) {
> 
> /\* 字符设备文件 \*/
> 
> inode\->i\_fop\= &def\_chr\_fops; /\* 默认字符设备操作表 \*/
> 
> inode\->i\_rdev\=rdev; /\* 保存设备号 \*/
> 
> }elseif(S\_ISBLK(mode)) {
> 
> /\* 块设备文件 \*/
> 
> inode\->i\_fop\= &def\_blk\_fops;
> 
> inode\->i\_rdev\=rdev;
> 
> }elseif(S\_ISFIFO(mode))
> 
> /\* 命名管道 \*/
> 
> inode\->i\_fop\= &pipefifo\_fops;
> 
> elseif(S\_ISSOCK(mode)) ;
> 
> /\* 套接字不设置操作表 \*/
> 
> else
> 
> printk(KERN\_DEBUG"bogus i\_mode for inode %s:%lu\\n",inode\->i\_sb\->s\_id,inode\->i\_ino);
> 
> }

> **🔑 理解：**此时 inode 中的`i_fop`指向的是`def_chr_fops`，一个**仅包含默认 open 方法**的操作表。真正的驱动专用操作函数表要等到用户调用`open()`时，才会通过`chrdev_open`函数动态替换。这是一种**"延迟绑定"**的设计模式

### **6.3 def\_chr\_fops —— 字符设备的默认操作函数**

> /\* 来源: fs/char\_dev.c \*/  
> 
> conststructfile\_operationsdef\_chr\_fops\= {
> 
> .open\=chrdev\_open, /\* 只定义了一个 open 方法 \*/
> 
> .llseek\=noop\_llseek, /\* 一个什么都不做的寻址函数 \*/
> 
> };  

### **6.4 chrdev\_open —— 字符设备 open 默认函数**

`chrdev_open`是整个字符设备驱动机制中最关键的桥接函数：

> /\* 来源: fs/char\_dev.c \*/  
> staticintchrdev\_open(structinode\*inode,structfile\*filp) {
> structcdev\*p;
>  struct cdev \*new \= NULL;  
> int ret\=0;  
> /\* 第一步：获取或查找 cdev \*/
> p\=inode\->i\_cdev;
> if(!p) {
> structkobject\*kobj;
> intidx;
> /\* 根据设备号在全局哈希表 cdev\_map 中查找对应的 cdev \*/
> kobj\=kobj\_lookup(cdev\_map,inode\->i\_rdev, &idx);
> if(!kobj)
> return\-ENXIO;
> /\* 通过 kobject 反推出包含它的 cdev \*/
> new\=container\_of(kobj,structcdev,kobj);
> /\* 并发安全：再次检查并设置 inode->i\_cdev（缓存优化） \*/
> p\=inode\->i\_cdev;
> if(!p) {
> inode\->i\_cdev\=p\=new;
> inode\->i\_cindex\=idx;
> list\_add(&inode\->i\_devices, &p\->list);
> new\=NULL;
> }
> }
>   
> /\* 第二步 ★核心★ 替换文件操作表 \*/  
> /\* 将 file 的 f\_op 从 def\_chr\_fops 替换为驱动自己的 fops \*/
> filp\->f\_op\=fops\_get(p\->ops);
> if(!filp\->f\_op)
> return\-ENXIO;
> /\* 第三步：调用驱动自身的 open 函数 \*/
> if(filp\->f\_op\->open) {
> ret\=filp\->f\_op\->open(inode,filp);
> }
> returnret;
> }

### 🔑 chrdev\_open 三步走核心逻辑：

1.  **查找 cdev：**通过`kobj_lookup()`在全局哈希表中查找，用`container_of`反推出 cdev。首次查找后缓存到`inode->i_cdev`
    
2.  **替换 f\_op：**`filp->f_op = fops_get(p->ops)`——最关键的一步，将 file 的文件操作表从`def_chr_fops`替换为驱动自己的`file_operations`
    
3.  **调用驱动 open：**如果驱动实现了自己的 open 方法，则调用它。至此，后续对该文件的 read/write 等操作都会直接调用驱动的对应函数。
    

### **6.5 open 函数的完整调用链**

#### **6.5.1 用户态 open() 到系统调用入口**

用户态的`open()`是 glibc 等 C 库提供的封装函数，它最终会触发系统调用`__NR_open`。在 x86-64 架构上，通过`syscall`指令陷入内核。

#### **6.5.2 内核通用打开流程：do\_sys\_open → do\_filp\_open**

**do\_sys\_open**将用户态的文件路径名拷贝到内核空间，构造`open_how`结构（包含打开标志和模式），然后调用`do_sys_openat2`：

> SYSCALL\_DEFINE3(open,constchar\_\_user\*,filename,
> 
> int,flags,umode\_t,mode) {
> 
> returndo\_sys\_open(AT\_FDCWD,filename,flags,mode);
> 
> }
> 
> longdo\_sys\_open(intdfd,constchar\_\_user\*filename,
> 
> intflags,umode\_tmode) {
> 
> structopen\_howhow\=build\_open\_how(flags,mode);
> 
> returndo\_sys\_openat2(dfd,filename, &how);
> 
> }

**do\_sys\_openat2**执行三个关键动作：

> staticlongdo\_sys\_openat2(intdfd,constchar\_\_user\*filename,
> 
> structopen\_how\*how) {
> 
> structfilename\*tmp\=getname(filename); /\* 拷贝路径到内核空间 \*/
> 
> intfd\=get\_unused\_fd\_flags(how\->flags); /\* 分配空闲fd \*/
> 
> structfile\*f\=do\_filp\_open(dfd,tmp,how); /\* ★路径查找并打开★ \*/
> 
> d\_install(fd,f); /\* 关联 fd 与 file \*/
> 
> returnfd;
> 
> }

#### **6.5.3 路径查找与打开：path\_openat → vfs\_open**

`do_filp_open`最终调`path_openat`（定义在 fs/namei.c），它完成路径的遍历（逐级查找目录项），**找到 inode 节点**，最后调用`vfs_open`来实际打开文件。

#### **6.5.4 do\_dentry\_open —— 核心函数**

`do_dentry_open`负责初始化`struct file`，并根据 inode 类型设置文件操作表，最终调用`open`方法。对于字符设备文件，`inode->i_fop`指向的是`def_chr_fops`。所以`f->f_op->open`实际上就是`chrdev_open`：

> staticintdo\_dentry\_open(structfile\*f,structinode\*inode,
> 
> int(\*open)(structinode\*,structfile\*)) {
> 
> f\->f\_mode\=OPEN\_FMODE(f\->f\_flags) |FMODE\_LSEEK|FMODE\_PREAD|FMODE\_PWRITE;
> 
> f\->f\_inode\=inode;
> 
> f\->f\_mapping\=inode\->i\_mapping;
> 
> /\* ★获取文件操作表：优先使用 inode->i\_fop★ \*/
> 
> f\->f\_op\=fops\_get(inode\->i\_fop);
> 
> if(!f\->f\_op)
> 
> return\-ENXIO;
> 
> if(open)
> 
> error\=open(inode,f);
> 
>   
> /\* 否则调用文件操作表中的 open 方法 \*/  
> 
> /\* 对于字符设备 = chrdev\_open！ \*/
> 
> if(!error&& (f\->f\_mode&FMODE\_OPENED) &&f\->f\_op\->open)
> 
> error\=f\->f\_op\->open(inode,f);
> 
> returnerror;
> 
> }

## 📌 完整调用链路总结：

`用户态 open()`→`sys_open()`→

`do_sys_open()`→`do_sys_openat2()`→

`do_filp_open()`→`path_openat()`→

`vfs_open()`→`do_dentry_open()`→

`chrdev_open()`（替换 fops）→驱动的 open()

---

## **七、字符设备驱动开发中的函数详解**

### **7.1 container\_of 宏**

`container_of`是 Linux 内核中常用的宏之一，它通过一个结构体成员的指针，反向获取包含该成员的结构体的起始地址：

> ```
> /**
> * container_of - 通过结构体成员指针获取父结构体指针
> * @ptr: 成员变量的指针
> * @type: 包含该成员的结构体类型
> * @member: 成员变量在结构体中的名称
> */
> #define container_of(ptr, type, member) ({ \
>     void *__mptr = (void *)(ptr); \
>     BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) && \
>     !__same_type(*(ptr), void), \
>     "pointer type mismatch in container_of"); \
>     ((type *)(__mptr - offsetof(type, member))); })
> 
> /* 简化理解版本： */
> #define container_of(ptr, type, member) \
>     ((type *)((char *)(ptr) - offsetof(type, member)))
> ```

该函数根据结构体成员变量的指针，通过该变量相对于结构体的偏移，得到了该变量对应的结构体的指针。这是一个非常灵活的用法，通过保存某个结构体变量的指针，可以通过该指针反向推出该结构体的指针。

> **💡 container\_of 的应用场景：**在 Linux 内核中，这个宏无处不在。比如在`chrdev_open`中，内核通过`kobj_lookup`拿到了 cdev 内嵌的`kobject`的指针，然后通过`container_of(kobj, struct cdev, kobj)`反推出完整的`cdev`结构体指针。这种"内嵌 + 反推"的设计是内核面向对象编程思想的经典体现

### **7.2 register\_chrdev\_region**

`register_chrdev_region`是 Linux 内核中用于注册字符设备编号范围的函数。该函数为驱动程序预留一段连续的设备号（主设备号 + 起始次设备号），后续将字符设备（通过`cdev_add`）绑定到这些设备号上：

> ```
> /**
> * register_chrdev_region - 注册字符设备编号范围
> * @first: dev_t 类型，指定要注册的起始设备号（使用 MKDEV(major, minor) 宏生成）
> * @count: 需要注册的连续设备号数量（次设备号的范围）
> * @name: 设备名称，会出现在 /proc/devices 文件中
> * 返回值: 成功返回 0
> * 参数无效返回 -EINVAL
> * 设备号被占用返回 -EBUSY
> */
> int register_chrdev_region(dev_t first, unsigned int count, const char *name);
> ```

### **7.3 THIS\_MODULE**

`THIS_MODULE`是一个宏，定义在`<linux/module.h>`头文件中。它本质上是一个指向当前模块的`struct module`结构体的指针：

-   当代码被编译为可加载内核模块（.ko）时，`THIS_MODULE`指向该模块的`struct module`实例
    
-   当代码被静态编译进内核（built-in）时，`THIS_MODULE`通常被定义为`NULL`或一个无实际作用的占位符
    

> staticstructfile\_operationsmy\_fops\= {
> 
> .owner\=THIS\_MODULE, /\* 防止模块在使用中被卸载 \*/
> 
> .open\=my\_open, .read\=my\_read,
> 
> .write\=my\_write,
> 
> .release\=my\_release,
> 
> };
> 
> structcdevmy\_cdev;
> 
> cdev\_init(&my\_cdev, &my\_fops);
> 
> my\_cdev.owner\=THIS\_MODULE;
> 
> cdev\_add(&my\_cdev,devno,count);

内核通过`owner`字段知道哪个模块拥有这个字符设备。当用户空间程序通过系统调用（如`open`）打开设备文件时，内核会自动调用`try_module_get(THIS_MODULE)`增加模块的引用计数；当设备被关闭时，内核会调用`module_put(THIS_MODULE)`减少引用计数。

> **🔑 这样做的目的是确保模块在设备被打开期间不会被卸载。**如果用户正在使用设备，而管理员执行`rmmod`试图卸载驱动，内核会检查模块的引用计数是否为零。若不为零（表示设备正在被使用），则卸载操作会被拒绝，防止因模块代码突然消失而导致系统崩溃。这是一个经典的"资源使用计数"保护模式。

### **7.4 file\_operations 结构体**

`file_operations`是把系统调用和驱动程序关联起来的关键数据结构：

| **函数指针** | **对应系统调用** | **说明** |
| --- | --- | --- |
| `owner` | —— | 指向拥有该结构的模块（THIS\_MODULE） |
| `llseek` | lseek | 修改文件读写位置 |
| `read` | read | 从设备读取数据 |
| `write` | write | 向设备写入数据 |
| `open` | open | 打开设备文件 |
| `release` | close | 关闭设备文件（引用计数归零时） |
| `unlocked_ioctl` | ioctl | 设备控制操作（无 BKL 版本） |
| `mmap` | mmap | 将设备内存映射到用户空间 |
| `poll` | poll/select | 询问设备是否可非阻塞读写 |

---

## **八、案例：使用字符设备驱动实现共享内存**

下面是一个完整的内核模块示例，演示如何使用字符设备驱动实现**共享内存**。这个驱动程序创建两个字符设备（次设备号分别为 0 和 1），每个设备拥有独立的 1024 字节共享内存区域，用户空间可以通过`read/write/ioctl/lseek`操作这些内存。代码参考了书籍Linux设备驱动详解

> \#include <linux/init.h>
> \#include <linux/errno.h>
> \#include <linux/mm.h>
> \#include <linux/sched.h>
> \#include <linux/module.h>
> \#include <linux/ioctl.h>
> \#include <linux/io.h>
> \#include <linux/fs.h>
> \#include <linux/cdev.h>
> \#include <linux/uaccess.h>
> \#include <linux/slab.h>
> \#define GLOBALMEM\_SIZE 1024
> \#define GLOBALMEM\_MAGIC 'M'
> \#define MEM\_CLEAR \_IO(GLOBALMEM\_MAGIC, 0)
>   
> /\* ===== 设备私有数据结构体 ===== \*/  
> structglobalmem\_dev{
> structcdevm\_cdev; /\* 内嵌的字符设备 \*/
> unsignedcharmem\[GLOBALMEM\_SIZE\]; /\* 共享内存区域（1KB） \*/
> };  
> staticint globalmem\_major \= 266;  
> struct globalmem\_dev \*globalmem\_devp;  
> /\* 两个设备实例 \*/  
> /\* ===== open：用户打开设备文件时调用 ===== \*/  
> staticintglobalmem\_open(structinode\*inode,structfile\*filp) {
> structglobalmem\_dev\*dev;
> /\* 通过 inode->i\_cdev (指向内嵌的 m\_cdev) 反推出 globalmem\_dev \*/
> dev\=container\_of(inode\->i\_cdev,structglobalmem\_dev,m\_cdev);
> /\* 保存到 file->private\_data，供后续 read/write 使用 \*/
> filp\->private\_data\=dev;
> return0;
> }
> /\* ===== release：用户关闭设备文件时调用 ===== \*/
> staticintglobalmem\_release(structinode\*inode,structfile\*filp) {
> return0;
> }
>   
> /\* ===== read：用户读取设备数据 ===== \*/  
> staticssize\_tglobalmem\_read(structfile\*filp,char\_\_user\*buf,size\_tcount,loff\_t\*ppos) {
> unsignedlongp\= \*ppos;
>  structglobalmem\_dev\*dev\=filp\->private\_data;  
> if(p\>=GLOBALMEM\_SIZE)
> return0;
> if(count\>GLOBALMEM\_SIZE\-p)
> count\=GLOBALMEM\_SIZE\-p;
> /\* 从内核空间拷贝数据到用户空间 \*/
> copy\_to\_user(buf, (void\*)(dev\->mem+p),count); \*ppos\=p+count;
> returncount;
> }
>   
> /\* ===== write：用户写入设备数据 ===== \*/  
> staticssize\_tglobalmem\_write(structfile\*filp,constchar\_\_user\*buf,size\_tcount,loff\_t\*ppos) {
> unsignedlongp\= \*ppos;
>   
> structglobalmem\_dev\*dev\=filp\->private\_data;  
> if(p\>=GLOBALMEM\_SIZE)
> return0;
> if(count\>GLOBALMEM\_SIZE\-p)
> count\=GLOBALMEM\_SIZE\-p;
> /\* 从用户空间拷贝数据到内核空间 \*/
> copy\_from\_user(dev\->mem+p,buf,count); \*ppos\=p+count;
> returncount;
> }
>   
> /\* ===== llseek：用户调整文件读写位置 ===== \*/  
> staticloff\_tglobalmem\_llseek(structfile\*filp,loff\_toffset,intorig) {
> loff\_tret;
> switch(orig) {
> case0:
> /\* SEEK\_SET：从起始位置移动 \*/
> if(offset<0|| (unsignedint)offset\>GLOBALMEM\_SIZE) {
> ret\= -EINVAL;
> break;
> }
> filp\->f\_pos\= (unsignedint)offset;
> ret\=filp\->f\_pos;
> break;
>   
> case1:  
> /\* SEEK\_CUR：从当前位置移动 \*/
> if((filp\->f\_pos+offset) >GLOBALMEM\_SIZE|| (filp\->f\_pos+offset) <0) {
> ret\= -EINVAL;
> break;
> }
> filp\->f\_pos+=offset;
> ret\=filp\->f\_pos;
>  break;  
> default:
> ret\= -EINVAL;
> }
> returnret;
> }
>   
> /\* ===== unlocked\_ioctl：用户发送设备控制命令 ===== \*/  
> staticlongglobalmem\_ioctl(structfile\*filp,unsignedintcmd,unsignedlongarg) {
> structglobalmem\_dev\*dev\=filp\->private\_data;
> switch(cmd) {
> caseMEM\_CLEAR:
> memset(dev\->mem,0,GLOBALMEM\_SIZE); /\* 清空共享内存 \*/
> break;
> default:
> return\-EINVAL;
> }
> return0;
> }
>   
> /\* ===== 文件操作函数集 ===== \*/  
> staticconststructfile\_operationsglobalmem\_fops\= {
> .owner\=THIS\_MODULE,
> .open\=globalmem\_open,
> .release\=globalmem\_release,
> .llseek\=globalmem\_llseek,
> .read\=globalmem\_read,
> .write\=globalmem\_write,
> .unlocked\_ioctl\=globalmem\_ioctl,
> };
>   
> /\* ===== 模块加载函数 ===== \*/  
> staticint\_\_initglobalmem\_init(void) {  
> /\* 1. 注册设备号：主 266，次 0～1，共 2 个 \*/
> register\_chrdev\_region(MKDEV(globalmem\_major,0),2,"globalmem");
> /\* 2. 为两个设备分配内存 \*/
> globalmem\_devp\=kmalloc(2\*sizeof(structglobalmem\_dev),GFP\_KERNEL);
> memset(globalmem\_devp,0,2\*sizeof(structglobalmem\_dev));
> /\* 3. 初始化并注册次设备号 0 \*/
> cdev\_init(&(globalmem\_devp\[0\].m\_cdev), &globalmem\_fops);
> globalmem\_devp\[0\].m\_cdev.owner\=THIS\_MODULE;
> cdev\_add(&(globalmem\_devp\[0\].m\_cdev),MKDEV(globalmem\_major,0),1);
> /\* 4. 初始化并注册次设备号 1 \*/
> dev\_init(&(globalmem\_devp\[1\].m\_cdev), &globalmem\_fops);
> globalmem\_devp\[1\].m\_cdev.owner\=THIS\_MODULE;
> cdev\_add(&(globalmem\_devp\[1\].m\_cdev),MKDEV(globalmem\_major,1),1);
> return0;
> }
> /\* ===== 模块卸载函数 ===== \*/  
> staticvoid\_\_exitglobalmem\_exit(void) {
> cdev\_del(&(globalmem\_devp\[0\].m\_cdev));
> cdev\_del(&(globalmem\_devp\[1\].m\_cdev));
> kfree(globalmem\_devp);
> dev\_tdevno\=MKDEV(globalmem\_major,0);
> unregister\_chrdev\_region(devno,2);
> }
>   
> MODULE\_AUTHOR("punch");  
> MODULE\_LICENSE("GPL");  
> MODULE\_DESCRIPTION("A simple shared memory character device driver");  
> module\_param(globalmem\_major,int,S\_IRUGO);  
> module\_init(globalmem\_init);  
> module\_exit(globalmem\_exit);  

## 📌 驱动代码关键设计解析：

1.  ### container\_of 的使用：
    
    在`globalmem_open`中，通过`inode->i_cdev`反向推出`globalmem_dev`结构体
    
2.  **private\_data 传递：**在 open 中将设备结构体保存到`filp->private_data`，这是 Linux 驱动中传递上下文的标准模式
    
3.  **copy\_to\_user / copy\_from\_user：**内核和用户空间数据交换必须通过这些安全函数进行
    
4.  ### IOCTL 命令定义：
    
    `_IO(GLOBALMEM_MAGIC, 0)`用于防止命令冲突
    
5.  **多实例管理：**两个次设备号各自对应独立的`globalmem_dev`实例，各拥有独立的 1024 字节共享内存
    

### **8.1 测试驱动**

编译加载驱动后，进行以下步骤来使用：

> \# 1. 加载驱动模块
> 
> ```
> sudo insmod globalmem.ko
> 
> # 2. 查看驱动是否加载成功（应看到主设备号 266 和 "globalmem"）
> cat /proc/devices | grep globalmem
> 
> # 3. 创建设备节点
> sudo mknod /dev/globalmem0 c 266 0
> sudo mknod /dev/globalmem1 c 266 1
> 
> # 4. 设置权限
> sudo chmod 666 /dev/globalmem0 /dev/globalmem1
> 
> # 5. 测试读写
> echo "Hello Linux Driver!" > /dev/globalmem0
> cat /dev/globalmem0
> 
> # 6. 卸载驱动
> sudo rmmod globalmem
> ```

> **💡 小结：**本小节只是为了演示字符设备驱动的简单案例。实际开发中，建议使用`alloc_chrdev_region`动态分配设备号，避免冲突。同时使用`class_create`和`device_create`可以让内核自动在`/dev`下创建设备节点，省去手动`mknod`的步骤。这些都是**Linux 2.6 之后推荐的新字符设备驱动编写方式**

---

## **🎯 总结**

本文从内核源码角度，完整梳理了 Linux 字符设备驱动的核心机制：

1.  **设备号管理（chrdevs）**：通过 255 大小的哈希数组，利用取模操作管理 0～511 乃至更大的主设备号空间，配合链表按顺序组织，提供了高效的设备号冲突检测机制。
    
2.  **cdev 对象与 kobj\_map**：cdev 是字符设备的抽象表示，cdev\_map 是设备号到 cdev 的映射哈希表。两套哈希表各司其职——chrdevs 管理资源分配，cdev\_map 支持运行时查找。
    
3.  **延迟绑定机制**：inode 创建时只挂载默认的`def_chr_fops`（仅含 chrdev\_open），真正驱动专用的 fops 要等到用户 open 时才通过 chrdev\_open 动态替换。
    
4.  **完整调用链路**：`open → sys_open → do_sys_open → do_filp_open → path_openat → do_dentry_open → chrdev_open(替换fops) → 驱动open()`
    
5.  **container\_of 设计模式**：通过内嵌结构体成员指针反推外围结构体，是 Linux 内核"面向对象"编程思想的集中体现。
    
6.  **引用计数保护**：通过`THIS_MODULE`和`owner`字段，防止设备在使用中被意外卸载。
    

---

文章内容为作者过往学习的笔记，接下来会按期更新，预计下一期更新内核结构体kobject解析。如果本文对你有帮助，欢迎**点赞、在看、转发**，让更多 Linux 驱动开发者受益 🚀

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/22f495c6_1783995020783?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUxMjEyNDgyNw%3D%3D%26mid%3D2247527435%26idx%3D1%26sn%3Ddc4b07f53acd071a569ef86bf83fc55e%26chksm%3Df842f23b216d55e94dd0267a8782e80412e5aac8032e906948d4da87e7ebe7f6778cc4d5f1d4%26mpshare%3D1%26scene%3D1%26srcid%3D0714QkgPTDnyJKBBfSKpczwK%26sharer_shareinfo%3D2e2175f650e7b910781640ae1b687bd1%26sharer_shareinfo_first%3D2e2175f650e7b910781640ae1b687bd1%23rd&s=obsidian)