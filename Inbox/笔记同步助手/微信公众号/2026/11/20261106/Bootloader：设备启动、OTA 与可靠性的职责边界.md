---
author: code闲录
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247527943&idx=1&sn=84a7dd4d47cbc9dc0c7b8d2b77b47c8e&chksm=f803374b2321a337383ae79492e06458fe7390a9a36ba479fa4adbb74abf901a5aa08edc664d&mpshare=1&scene=1&srcid=1106a7mV7S9WojBjGJEyPNLX&sharer_shareinfo=88a35e259e10a275327b657e701e5669&sharer_shareinfo_first=88a35e259e10a275327b657e701e5669#rd
saved: 2026-11-06 12:07:02
tags:
  - 笔记同步助手
id: d2f59a5f-0f48-42b9-a346-864c6590d60d
---

公众号名称：一口Linux

作者名称：code闲录

发布时间：2026-11-06 11:59

# Bootloader：设备启动、OTA 与可靠性的职责边界

> 目标：梳理嵌入式设备从复位到应用启动时，Bootloader 需要做什么、不该做什么，以及它如何与 OTA、回滚和 Recovery 配合。
> 
> 关联：说明固件的获取、写入与发布；本文聚焦启动阶段的最终裁决。ESP-IDF / ESP32-S3 用于说明具体落地方式。

## 1\. 先用一句话理解 Bootloader

**Bootloader 是复位后、应用之前执行的最小启动程序：它建立最小运行环境，验证并选择可信镜像，然后跳转；没有可安全启动的应用时，回退或进入受控恢复路径。**

```
上电 / 复位
    │
    ▼
ROM Bootloader（芯片固化的第一阶段）
    │
    ▼
二级 Bootloader（产品可配置的启动策略）
    ├─ 最小硬件初始化
    ├─ 读取分区表与启动记录
    ├─ 选择并验证候选镜像
    ├─ 装载 / 映射镜像，跳转给应用
    └─ 失败时回滚或进入 Recovery
```

一句话记忆：**应用负责把新固件准备好；Bootloader 负责决定下次能否、该启动谁。**

![[Inbox/笔记同步助手/微信公众号/2026/11/images/82482a8d5f989c89b2ed3b05156f184e_MD5.jpg]]

## 2\. 职责边界：Bootloader、应用与 Recovery 怎么分工

Bootloader 处在最早期可信链中，代码应尽量小、状态应尽量少。它是“启动裁决者”，不是缩小版应用。

| 事情                            | 推荐责任方              | 原因                                      |
| ----------------------------- | ------------------ | --------------------------------------- |
| 时钟、RAM、Cache/MMU、Flash 等早期初始化 | Bootloader         | 应用尚未运行，必须先具备读镜像和跳转条件。                   |
| 解析分区表、读取启动控制记录                | Bootloader         | 这是选择 factory、A/B 槽、test 或 Recovery 的依据。 |
| 镜像头、边界、兼容性、摘要/签名检查            | Bootloader         | 不能把“是否允许执行”交给可能损坏的应用。                   |
| 下载固件、断点续传、MQTT/HTTPS、复杂包解析    | 应用或 Recovery       | 协议栈和重试逻辑复杂，会扩大最早期可信代码的缺陷面。              |
| 擦写非活动 OTA 槽、写入安装进度            | 应用或 Recovery       | 健康应用最适合完成正常升级；单槽修复由 Recovery 接管。        |
| 设置候选槽、上报升级状态                  | 应用写入，Bootloader 读取 | 应用知道何时写完；Bootloader 只信任已持久化的启动记录。       |
| 健康检查与新版本确认                    | 应用执行，Bootloader 裁决 | 应用最了解外设、网络、配置迁移和业务是否真的可用。               |
| 串口 / USB 本地救援                 | Recovery 优先        | 复杂救援能力不宜塞进难以修复的 Bootloader。             |

> 关键点：HTTPS、下载摘要和应用层验签都很重要，但不能替代启动时验证。应用可能没有机会运行，且 Flash 可能在下载完成后损坏或被替换。

## 3\. Bootloader 的启动决策

### 3.1 建立最小运行环境

Bootloader 只初始化“安全读 Flash、校验镜像、跳转”所需的部分。常见项包括：

-   初始化栈、必要 RAM、时钟和 Flash 时序；
    
-   初始化 Cache/MMU 或 XIP 映射，使 Flash 中的代码和数据可访问；
    
-   设置看门狗、读取复位原因，避免早期异常长期卡死；
    
-   读取芯片型号、修订版本、eFuse 或安全配置；
    
-   输出最小串口日志或错误码，方便定位失败阶段。
    

此阶段不能假设网络、文件系统、RTOS 或普通驱动可用。关键初始化失败时，应复位、记录错误或进入恢复路径，不能继续跳转。

### 3.2 读取分区与启动状态

Bootloader 不应凭固定地址猜测应用位置，而应读取受校验的分区表或等价布局描述，得到：

-   factory、test、A/B OTA 槽、Recovery、暂存区的位置和容量；
    
-   OTA 启动控制区，例如 ESP-IDF 的 `otadata`；
    
-   已确认槽、候选槽、试启动次数、最近失败原因；
    
-   恢复请求、工厂 GPIO 或诊断模式等受控条件。
    

启动控制记录也是可靠性关键点。常见设计为双副本或追加式记录，并带 CRC、序列号和提交标志；更新时先写新副本，验证完成后再切换，确保掉电后至少能读到一份有效记录。

### 3.3 A/B 试启动、确认与回滚

可靠 OTA 的关键不是“写入成功”，而是“新镜像在真实设备上已确认可用”。典型闭环如下：

```
旧应用：下载并校验包 → 写非活动槽 → 写 pending 状态 → 重启
Bootloader：验证 pending 镜像 → 持久化 tries → 启动候选槽
新应用：完成关键自检
   ├─ 成功：写 confirmed 状态，成为新的回退基线
   └─ 失败 / 超时 / 复位：未确认；下次由 Bootloader 回退
```

候选镜像的启动规则必须确定、可解释，并且在断电后仍成立：

```
有 pending（候选）槽吗？
├─ 有：镜像有效且 tries 未耗尽？
│  ├─ 是：先持久化递增 tries，再启动 pending 槽
│  └─ 否：清除 pending，回到最后确认槽
└─ 无：启动最后确认槽

最后确认槽也无效？
└─ 尝试其他允许启动的镜像；都无效则进入 Recovery / 外部救援
```

**必须先持久化试启动次数，再跳转。** 否则候选镜像刚启动就掉电或复位时，次数可能永远不增加，设备会无限重试坏版本。应用也不能“进了 `main` 就确认”，健康检查应覆盖配置可读、关键外设、必要网络和核心业务。

![[Inbox/笔记同步助手/微信公众号/2026/11/images/9e8626d081fb080667a588ea6965f77b_MD5.jpg]]

## 4\. 验证什么：可启动、可信与可回退

| 检查 | 回答的问题 | 不能替代什么 |
| --- | --- | --- |
| 分区表 CRC / MD5 | 布局是否意外损坏 | 固件来自可信发布者。 |
| 镜像头、段地址、长度、芯片兼容性 | 镜像能否在本硬件上装载 | 镜像未被恶意替换。 |
| CRC、校验和或 SHA-256 | 传输或存储是否损坏 | 发布者身份认证。 |
| 数字签名 / Secure Boot | 是否由被授权密钥签发 | Flash 内容保密。 |
| anti-rollback 安全版本 | 是否回退到已知有漏洞的版本 | 正常功能兼容性验证。 |

验证通过后，Bootloader 按芯片镜像格式复制需驻留 RAM 的段、配置可从 Flash 执行的映射、设置 CPU 状态和入口参数，再跳转到应用入口。跳转不是普通函数调用：Bootloader 不应期待应用返回。

安全与可靠性必须一起考虑：

-   ROM 或不可修改的根信任验证下一阶段，二级 Bootloader 再验证应用，形成逐级验证；
    
-   发布私钥只留在受控签名系统；设备保存公钥、摘要或硬件根信任，不保存私钥；
    
-   Flash Encryption 解决离线读取 Flash 的机密性；Secure Boot / 签名解决是否允许执行的真实性与完整性，两者不能互相替代；
    
-   anti-rollback 以最低安全版本为启动门槛。eFuse 等一次性配置必须先在隔离样机演练密钥、升级和救援流程。
    

## 5\. 没有有效应用时：Recovery 与外部救援

当所有候选镜像都无效时，Bootloader 的职责是**停止启动不可信代码，并转入仍能修复设备的路径**：

-   启动受保护的 Recovery 分区；
    
-   进入芯片 ROM 下载模式或受控串口 / USB 救援；
    
-   输出复位原因、失败阶段、槽位和校验结果等可定位日志；
    
-   在授权的工厂或维护条件下选择 test 镜像或恢复数据分区。
    

Recovery 负责复杂工作，例如读取暂存包、验签、覆盖写单主应用区、写后复核及本地/联网下载。Bootloader 只负责“何时进入、能否启动 Recovery”。

三种架构的侧重点不同：

| 架构 | Bootloader 的主要任务 | 故障后的结果 |
| --- | --- | --- |
| 单分区原地覆盖 | 验证主应用；无效时进入 Recovery 或外部救援 | 旧应用已被覆盖，不能直接回滚。 |
| A/B 双槽位 | 读启动记录、验候选槽、限制试启动、确认后切换基线、失败回滚 | 直接启动最后确认的旧槽，适合无人值守设备。 |
| Recovery + 暂存区 | 识别主应用无效或恢复请求，验证并启动 Recovery | Recovery 重新安装可信包，不等于保留旧版本。 |

无论使用哪种架构，**Bootloader 自身、分区表和启动控制区都必须受写保护或安全启动链保护**，不能由普通应用任意覆盖。

![[Inbox/笔记同步助手/微信公众号/2026/11/images/a3c7cf364f4a091fe291c8a4fa41e636_MD5.jpg]]

## 6\. ESP-IDF / ESP32-S3 落地要点

ESP-IDF 中，ROM Bootloader 先加载 `bootloader.bin`；二级 Bootloader 读取分区表和双份 `otadata`，选择 factory 或 OTA app 分区并加载应用。应用使用 `esp_ota_begin()`、`esp_ota_write()`、`esp_ota_end()` 写入非运行槽，用 `esp_ota_set_boot_partition()` 请求下次试启动。启用回滚后，新应用应在自检成功时调用 `esp_ota_mark_app_valid_cancel_rollback()`；确认失败时调用 `esp_ota_mark_app_invalid_rollback_and_reboot()`。

当前项目采用 8 MiB Flash、双 OTA 槽和 LittleFS：

| 分区 | 偏移 | 大小 | 作用 |
| --- | --- | --- | --- |
| Bootloader | `0x000000` | 32 KiB | ESP-IDF 二级 Bootloader。 |
| Partition Table | `0x008000` | 4 KiB | 分区表及 MD5。 |
| otadata | `0x010000` | 8 KiB | 双份 OTA 启动选择记录。 |
| ota\_0 | `0x020000` | 2 MiB | 应用槽 A。 |
| ota\_1 | `0x220000` | 2 MiB | 应用槽 B。 |
| littlefs | `0x600000` | 2 MiB | UI 资源、相册与本地 OTA 固件包。 |

项目已启用 `CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y`。升级代码使用 `esp_ota_get_next_update_partition(NULL)` 选择未运行槽，完成 `esp_ota_end()` 后以 `esp_image_verify()` 复核；只有所有步骤成功才调用 `esp_ota_set_boot_partition()`。新镜像启动后，应在 Wi-Fi、关键外设、文件系统和核心服务初始化完成后，再调用确认逻辑。

工程约束：OTA 镜像不能超过单槽的 2 MiB，也不能擦写当前运行槽、Bootloader 或分区表；修改 `partitions.csv` 后，需完整烧录分区表、应用与 `otadata`，不能只覆盖应用镜像。

## 7\. 启动决策伪代码与验证清单

```
void boot_main(void)
{
    boot_record_t rec = load_latest_valid_record(); // 双副本 + CRC + 序列号

    if (recovery_requested(rec)) {
        boot_recovery_if_valid();
    }

    if (rec.pending_slot != SLOT_NONE) {
        if (!image_bootable_and_trusted(rec.pending_slot)) {
            clear_pending_with_reason(rec.pending_slot, IMAGE_INVALID);
        } else if (rec.try_count < MAX_TRIES) {
            persist_increment_try_count(); // 必须在跳转前持久化
            boot_jump(rec.pending_slot);
        } else {
            mark_rollback(rec.pending_slot, TRIES_EXHAUSTED);
        }
    }

    if (image_bootable_and_trusted(rec.confirmed_slot)) {
        boot_jump(rec.confirmed_slot);
    }

    boot_recovery_if_valid();
    enter_external_rescue_or_reset();
}
```

上线前逐项确认：

-   ⬜ Bootloader 不承担下载和复杂安装逻辑，应用、Recovery 与它的边界明确。
    
-   ⬜ 分区表与架构匹配，候选槽、启动控制区、Recovery/暂存区（如需要）齐全。
    
-   ⬜ 候选镜像均完成格式、边界、兼容性和完整性验证；量产设备还验证签名与安全版本。
    
-   ⬜ pending、confirmed、试启动次数和失败原因可掉电恢复；tries 在跳转前持久化。
    
-   ⬜ 新应用只在关键自检后确认；确认前复位、镜像损坏和 tries 耗尽均能回滚。
    
-   ⬜ 所有镜像无效时，有明确的 Recovery 或外部救援路径。
    
-   ⬜ 已覆盖下载中断、写入掉电、控制记录损坏、签名失败、回滚和 Recovery 重装测试。
    

最简面试回答：**Bootloader 负责最小硬件初始化、读取分区和启动状态、验证并选择可信镜像、装载跳转，以及在候选版本未确认或无有效应用时回滚/进入 Recovery。OTA 下载和写新镜像由应用或 Recovery 完成；Bootloader 是启动阶段的最终裁决者。**

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/60f5d2b6_1793938019449?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUxMjEyNDgyNw%3D%3D%26mid%3D2247527943%26idx%3D1%26sn%3D84a7dd4d47cbc9dc0c7b8d2b77b47c8e%26chksm%3Df803374b2321a337383ae79492e06458fe7390a9a36ba479fa4adbb74abf901a5aa08edc664d%26mpshare%3D1%26scene%3D1%26srcid%3D1106a7mV7S9WojBjGJEyPNLX%26sharer_shareinfo%3D88a35e259e10a275327b657e701e5669%26sharer_shareinfo_first%3D88a35e259e10a275327b657e701e5669%23rd&s=obsidian)