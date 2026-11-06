# RDP 远程桌面缩放问题修复记录

> 记录时间：2026-11-06
> 目标机器：`DESKTOP-8BMSR8V`（Windows 11 专业版 Build 26300）
> 连接方式：手机端 Microsoft Remote Desktop
> 最终方案：切断客户端 DPI 覆盖，缩放保持 100%

---

## 一、问题现象

手机端 RDP 客户端**关闭「自定义分辨率」**后，远程会话显示异常：另一台 Windows 10 电脑在同样操作下能保持高分辨率，而这台机器上的界面元素比例明显不对。

## 二、诊断过程

### 2.1 两台机器配置对比

对两台机器的 `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp` 做了完整只读导出对比：

| 配置项 | 本机（Win11 26300） | 对照机（Win10 19045） |
|---|---|---|
| `DesktopWidth` | 未设置 | 未设置 |
| `DesktopHeight` | 未设置 | 未设置 |
| `MaxXResolution` | 未设置 | 未设置 |
| `MaxYResolution` | 未设置 | 未设置 |
| `MaxMonitors` | 未设置 | 未设置 |
| `ColorDepth` / `MinEncryptionLevel` | 5 / 2 | 5 / 2 |
| `WdName` | Microsoft RDP 8.0 | Microsoft RDP 8.0 |
| 组策略限制 | 无 | 无 |
| 当时会话分辨率 | 2560 × 1380 | 2560 × 1340 |

**结论：两台机器的 RDP 服务端分辨率配置完全一致，全部走系统默认。**
原先怀疑的 `DesktopWidth` / `DesktopHeight` / `MaxXResolution` / `MaxYResolution` 这条路径，与本次问题无关。

### 2.2 实际分辨率并不低

关闭「自定义分辨率」并重连后，实测分辨率为 **2560 × 1340**，与对照机完全一致，属正常 2K 级别。

说明：**分辨率由客户端决定，服务端注册表没有参与。**

### 2.3 真正的问题：缩放被强制为 225%

实测发现实际缩放为 **225%**：

```
物理分辨率   2560 × 1340
AppliedDPI   216  →  225%
逻辑分辨率   1138 × 595
```

2560 × 1340 除以 2.25 后，逻辑空间只剩 1138 × 595，比 1280 × 720 还窄，窗口严重拥挤。

### 2.4 根本原因：RDP 的 DPI 重定向

在远程桌面内手动调整缩放**完全不生效**，表现为：

```
LogPixels   = 144   → 150%   ← 设置已写入
AppliedDPI  = 216   → 225%   ← 实际生效值纹丝不动
```

这是 RDP 8.0 之后的协议行为：**客户端连接时会把自身的缩放比例（deviceScaleFactor）发送给远程主机，主机强制采用**。手机端计算出的 DPI 为 225%，每次连接都会把远程会话重置回 225%，覆盖掉服务端的所有设置。

---

## 三、已执行的改动

### 3.1 核心开关：让 RDP 忽略客户端缩放

注册表路径：

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations
```

| 值名称 | 类型 | 数据 | 状态 |
|---|---|---|---|
| `IgnoreClientDesktopScaleFactor` | DWORD | `1` | **已写入，已生效** |

生效验证：改动并注销重登后，`AppliedDPI` 从 `216`（225%）变为 `96`（100%），证明客户端的强制覆盖已被切断。

### 3.2 最终状态

```
物理分辨率       2560 × 1340
AppliedDPI       96  →  100%
逻辑分辨率       2560 × 1340
IgnoreClientDesktopScaleFactor  = 1   （已生效，保留）
LogPixels                       = 168 （诊断阶段写入的残留值，Win11 下不生效，不影响当前显示）
Win8DpiScaling                  = 0
```

**最终决定：缩放保持 100%，不再调整。** 在此分辨率下元素清晰、布局完整，符合使用习惯。

`LogPixels = 168` 是排查过程中写入的测试值。Windows 11 已改用 per-monitor DPI，该值不再决定系统缩放，因此保留与否都不影响显示。若希望注册表保持整洁，可按下文 5.2 清理。

---

## 四、脚本与命令

### 4.1 一键修复脚本

工作区文件：`Fix-RDP-Scaling.ps1`

以管理员身份运行，完成开关写入。注意该脚本同时包含设置 `LogPixels` 的语句，如仅需复现本次最终状态，手动执行下方命令即可。

### 4.2 手动命令（复现最终状态）

**第一步：写入开关（需管理员权限）**

```powershell
New-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations' -Name 'IgnoreClientDesktopScaleFactor' -Value 1 -PropertyType DWord -Force
```

**第二步：注销并重新登录**

```powershell
shutdown /l
```

> 注意：必须是「注销」，不是「断开连接」。断开重连不会重新加载用户配置。

**验证命令：**

```powershell
(Get-ItemProperty 'HKCU:\Control Panel\Desktop\WindowMetrics' -Name AppliedDPI).AppliedDPI
```

返回 `96` 表示缩放为 100%，客户端覆盖已被切断。

---

## 五、恢复方法

### 5.1 恢复默认缩放行为（跟随客户端）

删除开关即可：

```powershell
Remove-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations' -Name 'IgnoreClientDesktopScaleFactor' -Force
```

删除后注销重登，远程会话将重新跟随客户端发送的缩放比例（225%）。

### 5.2 清理诊断残留值（可选）

`LogPixels` 当前为诊断阶段写入的 `168`，不影响显示。如需还原：

```powershell
Set-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'LogPixels' -Value 96 -Type DWord
Set-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'Win8DpiScaling' -Value 0 -Type DWord
```

### 5.3 完全还原检查清单

| 项目 | 原始状态 | 当前状态 | 恢复方式 |
|---|---|---|---|
| `IgnoreClientDesktopScaleFactor` | 不存在 | `1` | `Remove-ItemProperty` |
| `LogPixels` | `96` | `168` | 设回 `96` |
| `Win8DpiScaling` | `0` | `0` | 无需处理 |
| `RDP-Tcp` 下各分辨率键 | 均未设置 | 均未设置 | 从未改动，无需恢复 |

> 本次改动**未触碰** `DesktopWidth`、`DesktopHeight`、`MaxXResolution`、`MaxYResolution`、`MaxMonitors` 等任何分辨率相关键值。

---

## 六、附：诊断脚本

工作区文件：`RDP-Diag-ReadOnly.ps1`

纯只读脚本，用于采集 RDP 配置信息，不修改任何设置。输出到同目录 `RDP-Diag-Result.txt`，便于多台机器对比。

---

## 七、结论摘要

1. **分辨率从来不是问题**：关闭自定义分辨率后为 2560 × 1340，与正常机器一致。
2. **真正的问题是 RDP 的 DPI 重定向**：客户端把 225% 强推给主机，导致服务端缩放设置失效，逻辑空间被压缩到 1138 × 595。
3. **解决方案**：`IgnoreClientDesktopScaleFactor = 1` 切断覆盖源，已生效。
4. **最终状态**：缩放 100%，逻辑空间 2560 × 1340，显示效果符合预期，不再调整。
