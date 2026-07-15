---
author: 修齐识天下
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwODczOTE4Nw==&mid=2247485364&idx=1&sn=a888bb6c2f25d1d9ecfacb58181ee646&chksm=c1aa14f9eac043de11cf3571b3f50642fbfcaba38afe2baf12c736bf9d752bf43c3e9a181d4a&mpshare=1&scene=1&srcid=07144W4rg2pcLWsCTnOT5pUp&sharer_shareinfo=fecaa134f4159bdb31d841064d751bf7&sharer_shareinfo_first=fecaa134f4159bdb31d841064d751bf7#rd
saved: 2026-07-14 15:34:08
tags:
  - 笔记同步助手
id: 3022f693-2ab9-4930-9ac4-0810b7b58d6b
---

公众号名称：修齐识天下

作者名称：修齐识天下

发布时间：2026-07-14 12:08

我们来深入探讨在现代Linux发行版中占据核心地位的初始化系统——**systemd**，特别是它的**启动流程**以及强大的**服务依赖管理**机制。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/18d2ba451c282251bc250e30434a0c65_MD5.jpg]]

## 1\. systemd之前的世界：SysVinit的"线性"困境

要理解systemd的革命性，首先要了解它所取代的传统'**SysVinit'**系统。

**'SysVinit'**是基于**运行级别(****Runlevels****)和****Shell****脚本**的。它的工作方式大致如下：

1.  内核启动完成后，执行第一个用户空间进程/sbin/init。
    
2.  'init'进程读取'/etc/inittab'文件，确定要进入的默认运行级别(例如，runlevel 3是多用户文本模式，runlevel 5是图形界面模式)。
    
3.  'init'进程接着执行对应运行级别目录(如'/etc/rc3.d/')下的所有脚本。
    
4.  这些脚本的命名遵循一种约定，如'S20network'或'K80apache'。
    

-   'S'开头的脚本表示启动(Start)，'K'开头的表示停止(Kill)。
    
-   后面的数字(如'20')决定了脚本的执行顺序。数字小的先执行。
    

**SysVinit****的主要问题**:

-   启动缓慢: 整个启动过程是严格串行的。'S20network'必须等待'S10syslog'执行完毕后才能开始。如果某个服务启动很慢(比如等待网络超时)，它会阻塞后续所有服务的启动。
    
-   依赖管理脆弱: 依赖关系是靠脚本的命名顺序来隐式保证的。如果服务A依赖服务B，你必须确保A的启动脚本数字比B大。这种方式非常脆弱，难以维护，且无法处理复杂的依赖场景(如服务C同时依赖A和B)。
    
-   监控和管理困难: 'init'启动脚本后就“撒手不管”了。它无法可靠地追踪服务的状态(是否崩溃?)，也难以实现服务的自动重启。
    

## 2\. systemd的革命：基于“单元”的并行化与依赖驱动

'systemd'彻底抛弃了'**SysVinit'**的脚本和运行级别模型，引入了一套全新的、基于"单元(Unit)"的声明式管理体系。

**什么是单元(****Unit****)？**一个单元是systemd管理的一个资源。所有东西都被抽象为单元，例如：

-   '\*.service': 一个后台服务(守护进程)。
    
-   '\*.socket': 一个网络套接字(Socket)。
    
-   '\*.target': 一组单元的集合，类似于'SysVinit'的运行级别，但更灵活。
    
-   '\*.mount': 一个文件系统挂载点。
    
-   '\*.path': 一个文件或目录，用于基于路径的触发。
    
-   '\*.timer': 一个定时器，用于定时触发任务。
    

每个单元都有一个对应的配置文件(如'**sshd.service'**)，采用简单的**INI**格式，描述了这个单元的属性和行为。

## systemd的启动流程：

1.  内核启动与控制权交接: 与'SysVinit'一样，内核启动后，将控制权交给第一个用户进程'/sbin/systemd'。
    
2.  目标驱动的启动(Target-Driven):
    

-   'systemd'不再读取'/etc/inittab'，而是启动一个默认的目标单元(Target Unit)，通常是'default.target'。
    
-   'default.target'通常是一个指向'graphical.target'(图形界面)或'multi-user.target'(多用户文本模式)的符号链接。
    

4.  构建依赖关系树:
    

-   假设要启动'graphical.target'。systemd会解析'graphical.target'的单元文件。这个文件会通过'Wants='或'Requires='指令，声明它依赖于其他单元(如'multi-user.target'）。
    
-   'systemd'会递归地解析所有这些依赖关系，在内存中构建一个庞大而精确的依赖关系树。这棵树清晰地描述了所有单元之间的启动顺序和依赖关系。
    

6.  并行化启动(Parallelization):
    

-   这是'systemd'相比'SysVinit'最大的性能优势。一旦依赖关系树构建完成，'systemd'会分析这棵树，找出所有当前没有未满足依赖的单元。
    
-   然后，'systemd'会同时启动所有这些可以并行启动的单元。例如，挂载文件系统、启动日志服务、配置网络等操作，只要它们之间没有直接的依赖关系，就可以同时进行，极大地缩短了系统启动时间。
    

8.  事务性处理(Transactional):
    

-   systemd将启动或停止一组单元的操作视为一个事务(Transaction)。在执行事务之前，它会先验证依赖关系是否能被满足，是否存在冲突。如果存在问题，整个事务就不会被执行，避免了系统进入一个不一致的中间状态。
    

## 示例：从'graphical.target'到'sshd.service'

1.  systemd启动'default.target' -> 'graphical.target'。
    
2.  'graphical.targetWants'= 'multi-user.target'。
    
3.  'multi-user.targetWants'= 'basic.target'和'sshd.service'等。
    
4.  'sshd.serviceWants'= 'network.target'和'syslog.socket'。
    
5.  'network.targetWants'= 'network-pre.target'和具体的网络管理服务(如'NetworkManager.service'）。
    
6.  ...如此递归下去，直到最底层的'sysinit.target'和'local-fs.target'。
    
7.  'systemd'构建完这棵树后，从底层开始，将所有可以并行的单元(如挂载/home和启动'systemd-journald')同时启动。当一个单元启动成功后，依赖于它的上层单元就变成了"可启动"状态，'systemd'会继续将它们加入到启动队列中。
    

## 3\. 强大的服务依赖管理

systemd的依赖管理是**声明式**和**精确**的，通过在单元文件的**\[Unit\]**段中设置不同的指令来实现。

**依赖类型：**

-   'Requires=:'强依赖。如果本单元启动，那么'Requires='中列出的单元也必须启动。如果被依赖的单元启动失败，那么本单元也会被标记为启动失败。
    

-   示例：'docker.service' 'Requires=containerd.service'。没有containerd，docker就无法工作。
    

-   'Wants=:'弱依赖。如果本单元启动，systemd会尝试启动'Wants='中列出的单元。但即使被依赖的单元启动失败，也不会影响本单元的启动。
    

-   示例：'graphical.target' 'Wants=NetworkManager.service'。图形界面希望有网络，但即使网络启动失败，图形界面本身还是可以启动的。这是最常用的依赖类型。
    

-   'BindsTo=':'Requires='的加强版。当被依赖的单元停止或崩溃时，本单元也会被自动停止。
    

-   示例：一个挂载在某个USB设备上的服务，'BindsTo=dev-sdb1.device'。当USB设备被拔掉时，这个服务会自动停止。
    

## 顺序控制：

仅仅声明依赖关系还不够，有时我们还需要控制**启动的顺序**。

-   'After=': “我必须在它之后启动”。这定义了时间上的顺序，但不隐含依赖关系。
    

-   示例：'sshd.service' 'After=network.target'。sshd必须在网络配置完成后才能启动，否则它无法绑定到正确的IP地址。
    

-   'Before=': “我必须在它之前启动”。与'After='相反。
    

**'Wants='****和****'After='****的组合是实现服务依赖启动最经典、最常见的模式****。**

```
# my_app.service
[Unit]
Description=My Application
# 依赖关系：我希望网络服务能启动
Wants=network-online.target
# 顺序关系：我必须在网络真正可用之后再启动
After=network-online.target

[Service]
ExecStart=/usr/bin/my_app

[Install]
WantedBy=multi-user.target
```

这个例子清晰地声明了：'**my\_app.service'**依赖于网络，并且必须在网络就绪后才能启动。

## 4\. Socket激活：更高效的并行化

这是systemd另一个天才的设计。对于网络服务(如**sshd**)，传统的启动方式是：

1.  启动'sshd'进程。
    
2.  'sshd'进程创建并监听(listen)在22号端口上。
    

systemd的方式(Socket Activation):

1.  在系统启动的早期，systemd(而不是sshd进程)就可以代表sshd，提前创建好22号端口的监听套接字(Socket)。这步操作非常快。
    
2.  'sshd.service'本身可以暂时不启动，节省了系统资源。
    
3.  当第一个SSH连接请求到达'22'号端口时，systemd会检测到这个事件。
    
4.  此时，systemd才真正启动'sshd.service'，并将已经建立好的连接(和监听套接字)移交给新启动的sshd进程。
    
5.  sshd进程无需自己创建和监听端口，直接接管连接并开始处理。
    

**Socket****激活的好处**:

极致的并行化: 服务之间可以不再有启动顺序的依赖。即使服务A依赖服务B，只要A通过Socket与B通信，systemd就可以先为B创建好Socket，然后同时启动A和B。A在启动时就可以向B的Socket发送请求，即使B的进程还没完全初始化好，操作系统内核也会为它缓存这些请求。

按需启动(On-demand): 服务只在第一次被请求时才启动，节省了空闲时的系统资源。

## 总结

systemd通过引入**单元(****Unit****)的概念，将复杂的系统资源和服务管理，转变为一种声明式的、基于依赖关系**的模型。

-   启动流程: 它通过解析单元之间的依赖关系，构建出一棵精确的依赖树，并基于这棵树实现了最大程度的并行化启动，彻底解决了'SysVinit'的串行瓶颈。
    
-   依赖管理: 它提供了'Requires=', 'Wants=', 'After=', 'Before='等一系列精确的指令，让开发者能够清晰地声明服务之间的强弱依赖和启动顺序，使得系统行为变得高度可预测和可维护。
    
-   高级特性: 像Socket激活这样的设计，进一步解耦了服务之间的启动依赖，将并行化和资源利用效率推向了极致。
    

虽然systemd因其复杂性和"侵入性"而备受争议，但它所带来的在启动速度、系统管理能力和依赖控制方面的巨大进步，是其成为现代Linux发行版事实标准的根本原因。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/3c8903ba_1784014445333?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwODczOTE4Nw%3D%3D%26mid%3D2247485364%26idx%3D1%26sn%3Da888bb6c2f25d1d9ecfacb58181ee646%26chksm%3Dc1aa14f9eac043de11cf3571b3f50642fbfcaba38afe2baf12c736bf9d752bf43c3e9a181d4a%26mpshare%3D1%26scene%3D1%26srcid%3D07144W4rg2pcLWsCTnOT5pUp%26sharer_shareinfo%3Dfecaa134f4159bdb31d841064d751bf7%26sharer_shareinfo_first%3Dfecaa134f4159bdb31d841064d751bf7%23rd&s=obsidian)