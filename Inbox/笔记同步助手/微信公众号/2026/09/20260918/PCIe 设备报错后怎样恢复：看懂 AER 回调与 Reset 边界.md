---
author: 底层开发札记
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzYzMzg2NDcwNQ==&mid=2247484937&idx=1&sn=a10b4a3fda72e0a56600039dc68350e6&chksm=f120d0ce0428346d98081b4cd076799d9276ecd1a47e262645df6c9d6c432a07dfbca0d285fa&mpshare=1&scene=1&srcid=0918hp4h7VRFDdaSCMQ2aSqV&sharer_shareinfo=d8c415e368b35447c8d2eadf4ed39396&sharer_shareinfo_first=d8c415e368b35447c8d2eadf4ed39396#rd
saved: 2026-09-18 07:58:26
tags:
  - 笔记同步助手
id: 8f6e39a1-349a-4da6-a132-a1d34b290623
---

公众号名称：底层开发札记

作者名称：底层开发札记

发布时间：2026-09-17 21:15

## PCIe 报错后，设备为什么没有自动恢复

一块 PCIe 采集卡正在 DMA，Root Port 随后报告 Uncorrectable Error。Linux 打印了 AER 日志，驱动也收到了通知，业务却没有回来。

日志到这里并没有给出答案。“AER 已经处理错误”说的是错误被发现和上报，不等于采集卡的 DMA 队列已经重新可用。恢复链涉及三部分：

> Root Port / RCEC 发现并记录错误
> 
> ↓
> 
> AER 服务定位错误源和影响范围
> 
> ↓
> 
> PCI core 协调受影响层级、决定恢复阶段
> 
> ↓
> 
> Function 驱动停止业务并重建设备私有状态

AER 知道哪里出错，却不了解采集卡的 DMA ring、固件状态和在途请求。Function 驱动熟悉设备协议，也不能假定每次 reset 只影响自己。PCI Error Recovery 回调就是两者之间的交接面。

## 一、AER 日志只解决了错误上报

PCIe AER 能把错误分为 Correctable、Uncorrectable Non-Fatal 和 Uncorrectable Fatal。这个严重性会影响后续流程，但它和驱动最终能否恢复不是同一个判断。

Correctable Error 已由链路协议纠正，通常不需要 Function 级 reset。AER 服务主要负责记录、清理状态；在包含该回调成员的目标内核中，驱动还可以选择实现 `cor_error_detected()`，补充设备相关诊断。维护版本较旧时，应先核对对应内核的 `struct pci_error_handlers`。

Non-Fatal 错误进入恢复流程时，通道仍可能可用。驱动可以判断设备能否在不 reset 的情况下恢复，也可以主动请求 reset。

Fatal 错误意味着相关链路或层级不再可信。Linux 会把受影响设备通知为 frozen，并执行上游恢复动作。即使某个驱动认为自己可以原地恢复，也不能单独取消整个层级需要的 reset。

错误入口还可能不是 Native AER。采用 Firmware First 的平台会由固件先处理，再通过 APEI/GHES 报告给操作系统。因此，没有 Linux AER 中断或常见 sysfs 节点，并不能证明链路从未发生错误。

## 二、AER、PCI core 和 Function 驱动各管一段

AER Root Port 服务读取 AER 状态、Error Source ID 和 Header Log，记录错误并触发恢复。

PCI core 确定恢复作用于哪个 Port、Function 或下游子树，遍历受影响设备，合并各驱动返回结果，并在需要时调用平台或 Port 的 reset 操作。

Function 驱动负责停止新请求、收束软件并发、保存诊断信息，并在 PCI 访问恢复后重建设备寄存器、队列、中断和固件状态。

AER 发现错误，PCI core 协调恢复，Function 驱动把业务状态重新接上。清除 AER Status 只是结束本次错误记录，DMA 队列仍要由设备驱动重建。

## 三、驱动先注册恢复能力

支持 PCI 错误恢复的驱动，通过 `struct pci_driver.err_handler` 注册回调。下面的代码只保留接口签名和注册关系：

> static pci\_ers\_result\_t
> 
> demo\_error\_detected(struct pci\_dev \*pdev,
> 
> pci\_channel\_state\_t state);
> 
> static pci\_ers\_result\_t demo\_mmio\_enabled(struct pci\_dev \*pdev);
> 
> static pci\_ers\_result\_t demo\_slot\_reset(struct pci\_dev \*pdev);
> 
> static void demo\_resume(struct pci\_dev \*pdev);
> 
> static const struct pci\_error\_handlers demo\_err\_handlers = {
> 
> .error\_detected = demo\_error\_detected,
> 
> .mmio\_enabled = demo\_mmio\_enabled,
> 
> .slot\_reset = demo\_slot\_reset,
> 
> .resume = demo\_resume,
> 
> };
> 
> static struct pci\_driver demo\_driver = {
> 
> .name = "demo\_pcie",
> 
> .id\_table = demo\_ids,
> 
> .probe = demo\_probe,
> 
> .remove = demo\_remove,
> 
> .err\_handler = &demo\_err\_handlers,
> 
> };

具体函数要按设备协议实现。只要驱动接入这套恢复接口，`error_detected()` 就不能缺少；其他回调是否需要，取决于设备的恢复方式。

当前 mainline 的 `struct pci_error_handlers` 还包含 `reset_prepare()` 和 `reset_done()`，用于 PCI Function reset 前后的设备通知。它们和本文讨论的 AER 阶段回调共用同一个结构，但这并不表示 AER 恢复会接着调用它们。实现时要分别核对错误恢复路径与具体 reset API 的调用链。

注册回调不等于具备恢复能力。如果 `error_detected()` 只是打印日志，没有冻结业务队列、阻止新 DMA 和同步工作线程，后续 reset 就可能与旧请求并发执行。

## 四、通道状态决定还能不能访问硬件

`error_detected()` 的第二个参数是 `pci_channel_state_t`。它描述 PCI I/O 通道在当前恢复阶段是否可信，不是 AER Status 寄存器的简单翻译。

### `pci_channel_io_normal`

通道仍处于正常状态。典型入口是 Non-Fatal AER 恢复。驱动仍应先停止新业务，再根据设备状态判断可以原地恢复，还是需要 reset。

### `pci_channel_io_frozen`

通道已被冻结或隔离，普通 MMIO 可能被阻断。驱动此时应把它当成静默点：停止软件活动、等待可能访问设备的工作退出，但不要为了“试试看还活着吗”反复 `readl()`。

### `pci_channel_io_perm_failure`

恢复框架已经判定设备永久失效。驱动应取消或失败所有待处理 I/O，拒绝新请求，并进入接近 shutdown/remove 的清理路径，不能再返回“需要 reset”继续尝试。

所以，进入 `error_detected()` 后不能先读寄存器再判断状态。normal、frozen 和 permanent failure 对硬件访问的约束不同。

## 五、`error_detected()`：先收住旧业务

这个回调是驱动与 PCI core 的同步点。正在 DMA 的采集卡至少要处理四件事：

**1**阻止用户态或上层继续提交新请求；

**2**标记设备进入 recovery，避免超时线程同时发起第二次 reset；

**3**停止或隔离可能访问 BAR、DMA ring 的工作项；

**4**根据通道状态和设备恢复能力返回意见。

返回值表达的是驱动对下一步的判断，而不是一次普通函数调用的成功或失败：

| 返回值 | 含义 | 后续倾向 |
| --- | --- | --- |
| `PCI_ERS_RESULT_CAN_RECOVER` | 可以在 MMIO 恢复后继续检查 | 进入 `mmio_enabled()` |
| `PCI_ERS_RESULT_NEED_RESET` | 不 reset 无法可靠恢复 | 请求 reset |
| `PCI_ERS_RESULT_RECOVERED` | 当前阶段已恢复 | 等待层级汇总 |
| `PCI_ERS_RESULT_DISCONNECT` | 驱动无法安全恢复 | 走失败/断开路径 |

如果 frozen 后设备寄存器、DMA 上下文和固件状态都需要重建，返回 `CAN_RECOVER` 只会把问题推迟。此时应明确返回 `NEED_RESET`。

## 六、`mmio_enabled()`：只确认设备是否还能救

当受影响层级的 `error_detected()` 返回结果汇总为 `PCI_ERS_RESULT_CAN_RECOVER` 时，PCI core 才进入 MMIO Enable 阶段并调用 `mmio_enabled()`。

这个阶段适合读取少量健康寄存器、确认设备状态机是否仍可用，或者提取在 frozen 阶段不能读取的诊断信息。它不是完整 probe，也不应在确认状态前重新开放业务。

如果设备上下文仍然完整，驱动可以返回 `RECOVERED`。如果检查发现队列状态、固件或寄存器上下文已经丢失，应返回 `NEED_RESET`。

Fatal/frozen 场景需要额外注意：上游恢复框架仍会执行相应的上游 reset，驱动不能用 `CAN_RECOVER` 取消这个动作。需要在 reset 后重建私有状态的驱动，应通过返回值让流程进入 `slot_reset()`，而不是假设 `mmio_enabled()` 之后设备不会再被 reset。

## 七、`slot_reset()`：reset 结束后重建硬件

reset 动作由 PCI core、AER/DPC 服务及平台能力共同协调。Function Level Reset、PM reset、Secondary Bus Reset 和插槽掉电的作用范围不同；其中桥级 reset 可能同时影响一个 Port 下的多个 Function。

驱动的 `slot_reset()` 是 reset 完成后的通知，不是让驱动在回调里随意挑一种 reset。进入该回调时，驱动可以重新访问配置空间、BAR 和 DMA，并开始恢复自己的硬件上下文。

典型工作包括：

恢复 PCI 配置状态和设备使能条件；

重新设置 DMA mask 和总线主控状态；

重新初始化设备寄存器或下载固件；

重建 DMA ring、doorbell 和中断；

清除旧请求与新队列之间可能冲突的标识。

驱动可以复用 probe 中“初始化硬件”的内部函数，但不能重新注册整个 `pci_driver`，也不能重复创建设备节点或用户接口。返回 `RECOVERED` 只表示本 Function 已恢复，还要等待同一层级的其他驱动完成投票。

## 八、`resume()`：硬件确认可用后再开放请求

只有前面阶段汇总为恢复成功，PCI core 才调用 `resume()`。此时驱动才解除 recovery 标记、启动工作线程、开放提交队列并允许用户态重新访问。

如果 `resume()` 才第一次创建 DMA ring 或恢复固件，说明阶段边界已经错位。`slot_reset()` 应完成硬件重建和最小功能确认，`resume()` 只负责让业务重新流动。

恢复硬件也不代表所有旧请求都能继续。块层、网络栈或用户态可能已经把请求判定超时。驱动必须明确在途请求是失败、重试还是重新提交，不能让 reset 前的迟到 completion 完成 reset 后的新请求。

## 九、为什么一个 Function 的返回值会影响整棵子树

PCIe 错误可能影响一个 Port 下的多个设备。Linux `pcie_do_recovery()` 会确定负责恢复的 bridge，再遍历其下游设备，调用回调并合并 `pci_ers_result_t`。

这不是某个 Endpoint 独自决定的流程。在本文核对的 Linux mainline `drivers/pci/pcie/err.c` 中，某个受影响 Function 缺少 `error_detected()`，会被视为没有 AER 恢复驱动，并阻止同一子树继续执行后续恢复回调。某个驱动请求 reset，也可能让其他原本可以原地恢复的 Function 一起经历 reset；维护其他内核版本时应重新核对该合并逻辑。

因此，多 Function 设备或 Switch 下游拓扑必须共同审查：谁共享 reset 域，谁还在 DMA，谁没有注册恢复回调。只看报错 Endpoint 的驱动日志，可能解释不了流程为什么突然进入永久失败。

## 十、Completion Timeout 出现后，从哪里查起

采集卡 I/O 超时，同时看到 Root Port 报 Completion Timeout，并不能立刻把根因锁定在 Completion Timeout。如果链路更早已经掉线，这条错误只是说明某个请求没有等到 Completion。

先把内核日志按时间排开，看看最早出现的是 Surprise Down、Completion Timeout、DPC containment，还是设备自己的 watchdog 超时。顺序一变，排查方向也会变。

随后展开 PCIe 拓扑。`lspci -t` 用来确认 Endpoint 挂在哪个 Port 下，Root Port、Switch Port 和 Endpoint 的 `lspci -vv` 则用来核对能力与状态。Error Source ID、Header Log 和 BDF 对上以后，才知道恢复会扫过哪些 Function。

驱动日志也要落到每一次回调，单独打印一句“正在恢复”没有诊断价值。至少留下这五项：

> BDF → callback → channel\_state → return value → timestamp

如果流程只进入 `error_detected(frozen)` 就转为永久失败，沿同一子树检查是否有 Function 缺少恢复回调，或者某个驱动返回了 `DISCONNECT`。

进入 `slot_reset()` 后，继续核对配置状态、设备状态机、DMA ring、MSI-X、doorbell 和固件状态。回调返回成功，只能说明驱动选择继续；队列是否真正可用，还要看下一次提交和完成。

证据最终应收敛到具体断点，例如：

> AER 将通道标为 frozen，驱动返回 NEED\_RESET；
> 
> 上游 reset 完成后进入 slot\_reset；
> 
> 驱动恢复了 BAR 和 MSI-X，却没有重新创建 DMA ring；
> 
> resume 开放队列后，第一条请求再次超时。

这样的结论可以逐项回查寄存器、回调日志和队列状态，比“AER 没恢复好”更接近真正的修复点。

## 十一、恢复过程中，另外三条路径也可能在动

### 旧中断和工作队列

错误发生后，平台不保证中断已经停止；handler 如果无法安全确认来源，应避免继续访问失效设备。`error_detected()` 返回前还要同步可能触碰 BAR 或 DMA 内存的工作。

### 业务超时触发的 reset

协议驱动自己的 timeout、devlink reload、用户触发 reset 和 AER recovery 必须串行化。两个线程同时释放并重建队列，常会制造比原始 PCIe 错误更难复现的生命周期故障。

### remove

热拔插、手工 unbind 或永久失败可能让设备走向移除。恢复代码必须用清晰的状态机和锁保护私有对象，不能只因为 `struct pci_dev` 指针还存在，就假设硬件仍可访问。

## 十二、什么时候应该停止恢复

如果配置空间持续不可访问、平台 reset 失败、设备固件无法重建、DMA 无法确认停止，或者恢复后立即重复出现同类错误，驱动应让设备离线并向上层返回失败。

“尽量不掉业务”不等于永远返回成功。只要无法证明设备状态、DMA 所有权和软件请求重新一致，继续 I/O 就可能把一个可见故障变成静默数据损坏。安全地失败，也是恢复设计的一部分。

## 事实与来源

通道状态、`pci_error_handlers` 回调和各阶段职责：Linux Kernel Documentation《PCI Error Recovery》。

Correctable、Non-Fatal、Fatal 的 AER 恢复差异：Linux Kernel Documentation《PCI Express Advanced Error Reporting Driver Guide HOWTO》。

子树遍历、返回值合并、reset、`slot_reset()` 与 `resume()` 调度：Linux mainline `drivers/pci/pcie/err.c`。

Root Port AER 状态收集与错误源处理：Linux mainline `drivers/pci/pcie/aer.c`。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/9562093a_1789689504731?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzYzMzg2NDcwNQ%3D%3D%26mid%3D2247484937%26idx%3D1%26sn%3Da10b4a3fda72e0a56600039dc68350e6%26chksm%3Df120d0ce0428346d98081b4cd076799d9276ecd1a47e262645df6c9d6c432a07dfbca0d285fa%26mpshare%3D1%26scene%3D1%26srcid%3D0918hp4h7VRFDdaSCMQ2aSqV%26sharer_shareinfo%3Dd8c415e368b35447c8d2eadf4ed39396%26sharer_shareinfo_first%3Dd8c415e368b35447c8d2eadf4ed39396%23rd&s=obsidian)