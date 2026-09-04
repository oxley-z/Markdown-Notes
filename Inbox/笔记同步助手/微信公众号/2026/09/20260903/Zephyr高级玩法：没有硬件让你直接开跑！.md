---
author: Crist Xu
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU0MzQzMzc5Nw==&mid=2247494506&idx=1&sn=eea24490abacaf683c92b49825583ca1&chksm=fa6c53f0d78569fe209602d7dd7681bb2c1258f172aa3071a7ff89f84b272eb6aeb789b4b94d&mpshare=1&scene=1&srcid=090350RlEUwV09nFpo6IGOpb&sharer_shareinfo=9851bd4eda22e83a785e2a4607c9d961&sharer_shareinfo_first=9851bd4eda22e83a785e2a4607c9d961#rd
saved: 2026-09-03 00:03:42
tags:
  - 笔记同步助手
id: 0e6ff3b1-a0c7-4835-94c7-e67fde92a797
---

公众号名称：痞子衡嵌入式

作者名称：Crist Xu

发布时间：2026-09-02 21:03

在接触Zephyr之后，相信很多开发者做的第一件事，就是先找一块官方支持的开发板，把Demo跑起来、把环境搭起来，然后再开始功能开发和调试。

不过问题来了：**如果手边没有开发板，还能不能进行Zephyr开发？**

对于传统嵌入式开发来说，这似乎是一个“不成立”的问题。毕竟没有硬件，程序跑在哪里？驱动怎么验证？功能又该如何调试？

实际上，Zephyr早就为开发者提供了两套“脱离真实硬件”的运行方案——**QEMU**和**native\_sim**。它们都能够让应用程序在没有开发板的情况下运行和调试，极大地降低了开发门槛，也让持续集成（CI）、自动化测试以及早期软件开发成为可能。

但是，很多初学者都会把这两者混为一谈，认为它们只是不同的“模拟器”。事实上，两者在设计目标、实现机制以及适用场景上有着本质区别：

-   **QEMU侧重于硬件仿真，尽可能模拟目标平台的运行环境；**
-   **native\_sim则更像是在主机上运行的 Zephyr 仿真环境，重点验证软件逻辑而非硬件行为。**

如果不了解两者的能力边界，很容易出现这样的情况：明明程序在native\_sim上运行正常，上板后却问题不断；或者花费大量时间在QEMU中调试，却只是为了验证一段与硬件毫无关系的业务逻辑。

因此，选择正确的平台不仅关系到开发效率，更直接影响测试结论的可靠性。

本文将从**本质差异、能力矩阵、构建与调试、典型应用场景、常见误区以及选型建议**六个方面，系统梳理QEMU与native\_sim的区别，帮助大家在没有开发板的情况下，依然能够高效开展 Zephyr 开发与调试工作。

![[Inbox/笔记同步助手/微信公众号/2026/09/images/6c82f8bdf9d4b931081202dacff8917c_MD5.jpg]]

## 一、本质差异：它们在“模拟什么”

-   QEMU：提供一个**虚拟硬件平台**。它模拟的是具体CPU架构与部分外设（如 Cortex‑M3的NVIC、SysTick、UART等），Zephyr以“固件”的身份在虚拟板上启动。换言之，QEMU更接近“真实MCU的执行环境”。
    

-   **native\_sim**（旧称native\_posix）：提供一个**主机进程级的仿真环境**。Zephyr以“Linux/WSL进程”的形式运行，线程由主机的pthread驱动，时钟用主机定时器，控制台直连宿主标准输出。它不模拟MCU外设与寄存器，仅复刻Zephyr的内核行为与系统抽象。

一句话概括：  
**QEMU像虚拟开发板，native\_sim像在PC上跑的Zephyr模型。**

## 二、能力矩阵：到底能测什么、不能测什么

-   **QEMU能测的：**
    

-   CPU指令与异常语义（以架构为准）：中断、HardFault、栈帧、返回路径
    
-   核心时序：SysTick、时基中断、调度器切换边界
-   少量“被实现”的外设：常见为UART、定时器；不同虚拟机型能力不同
-   设备模型与驱动绑定流程（probe/init/ISR 路径）

-   **QEMU**不能测的：
    

-   未被QEMU实现的外设与SoC特性（大量MCU专用外设并不存在）
-   电气特性、真实引脚、DMA时序、板级布线带来的时延与抖动

-   **native\_sim能测的：**
    

-   业务与协议栈逻辑（TCP/UDP、应用状态机）
-   Zephyr IPC与内核原语（信号量、消息队列、工作队列）
    
-   单元测试与回归测试（ztest/twister），速度极快，适合CI
-   与主机的网络TAP/TUN交互（便于快速做端到端联调）

-   **native\_sim不能测的：**
    

-   MCU级寄存器/中断精确定时
    
-   外设驱动的真实工作路径（GPIO/I2C/SPI/PWM/ADC等）
    

## 三、构建与运行：命令对照

**<u>QEMU</u><u>（以</u> <u>Cortex‑M3</u><u>为例）</u>**

```
#
构建并运行官方样例
west build -b qemu_cortex_m3 samples/hello_world -p always
west build -t run
```

**<u>native\_sim</u>**

```
#
构建并运行
west build -b native_sim samples/hello_world -p always
./build/zephyr/zephyr.exe
```

注意：Windows原生环境下，native\_sim依赖主机的C预处理器（cpp）参与设备树预处理；更推荐在WSL中构建，或用MSYS2提供cpp。

## 四、调试方式：两者都能用GDB，但细节不同

**<u>QEMU</u><u>调试（两种主流姿势）</u>**

```
Shell
# A.
让 west 起好gdbserver
west build -b qemu_cortex_m3 /path/to/app -p always
west -t debugserver
arm-zephyr-eabi-gdb build/zephyr/zephyr.elf
(gdb) target remote :1234
(gdb) break main; continue
```

\# B. 手动起QEMU再连

```
qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic \
-kernel build/zephyr/zephyr.elf -s -S
arm-zephyr-eabi-gdb build/zephyr/zephyr.elf
(gdb) target remote :1234
```

<u>native\_sim</u> **<u>调试</u>**

**Shell**

```
#
直接用主机 GDB，体验更接近“普通 Linux 进程”
gdb ./build/zephyr/zephyr.exe
(gdb) break main
(gdb) run
```

补充：两者都建议在prj.conf打开以下选项以便定位崩溃与栈溢出：

```
CONFIG_PRINTK=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3
CONFIG_EXCEPTION_DEBUG=y
CONFIG_INIT_STACKS=y
CONFIG_STACK_SENTINEL=y
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_ISR_STACK_SIZE=2048
```

## 五、典型场景选择：什么时候应该用什么

-   **写应用与协议栈、做ztest/CI → 选**native\_sim**  
    理由：快、稳定、调试体验好；与主机网络配合顺滑，适合高频迭代**

-   **验证中断、时序、驱动入口、异常路径 → **选**QEMU**  
    理由：更贴近 MCU 行为，能看到栈帧与硬Fault，捕捉真实的异常场景****
    

-   **先native\_sim做逻辑正确性，再QEMU做系统语义校验，最后上真板做外设收尾**
    
    → 工程上最稳的三段式路线
    

## 六、常见误区与避坑

### 1\. 把native\_sim当作“虚拟 MCU”

这是误解。它不模拟MCU寄存器/外设，只适合逻辑与协议层验证。

**2\. 在QEMU上验证所有驱动**

不现实。除非QEMU为你的目标外设提供了模型，否则只能验证通用路径，无法代表实际硬件。

**3\. HardFault/Lockup 就是工具链问题**

很多时候是访问了未实现外设、函数指针非法、栈溢出或异常重入。优先扩大栈、打开异常日志、在z\_arm\_hard\_fault 断点观察 pc/lr/xPSR，再addr2line回源码行定位。

**4\. Windows原生编译native\_sim失败**

缺cpp导致设备树预处理报错是常见现象。建议WSL构建，或通过MSYS2安装并加入PATH。

## 七、小结

如果把开发过程比作盖房子，那么：

-   **QEMU像是按照真实建筑图纸搭建的样板房；**
-   **native\_sim则像是在电脑里进行结构计算和方案验证。**

QEMU与native\_sim看起来都属于“没有开发板也能运行Zephyr”的方案，但本质上解决的是两类不同的问题：

-   **QEMU**
    
    关注的是**硬件仿真**，适合验证驱动、启动流程、中断、设备树等与硬件强相关的功能。
    
-   **native\_sim**
    
    关注的是**软件逻辑验证**，适合快速开发、单元测试、协议栈验证以及CI 自动化测试。
    

对于大多数Zephyr开发者来说，最佳实践往往不是二选一，而是**组合使用**：

先利用native\_sim快速完成逻辑开发和自动化测试，再通过QEMU验证与硬件相关的行为，最后在真实开发板上进行最终确认。

这样既能获得最高的开发效率，又能够保证最终运行结果与真实硬件保持一致。

毕竟，无论模拟器做得多么逼真，它都不能完全替代真实硬件；但在开发早期，善用QEMU和native\_sim，往往能让我们少等待开发板、少踩硬件坑，把更多时间放在功能实现本身上。

**选择合适的工具，比盲目追求“真机调试”更重要。**

接下来小编会连续给大家带来几篇关于Qemu/NativeSim实战的讲解文章，敬请期待！

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/4372c8ea_1788365019215?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU0MzQzMzc5Nw%3D%3D%26mid%3D2247494506%26idx%3D1%26sn%3Deea24490abacaf683c92b49825583ca1%26chksm%3Dfa6c53f0d78569fe209602d7dd7681bb2c1258f172aa3071a7ff89f84b272eb6aeb789b4b94d%26mpshare%3D1%26scene%3D1%26srcid%3D090350RlEUwV09nFpo6IGOpb%26sharer_shareinfo%3D9851bd4eda22e83a785e2a4607c9d961%26sharer_shareinfo_first%3D9851bd4eda22e83a785e2a4607c9d961%23rd&s=obsidian)