---
author: 咔哒
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNjY4NTQ1Mg==&mid=2247484821&idx=1&sn=474b54598e11701ae045c75717883296&chksm=c391c5a8a7c365eaff731e034618c3664472a9118575d2a2ea612f4b6183030fc9c6c6a44b0a&mpshare=1&scene=1&srcid=05097yQIkh9Zsw7LIp23kE0o&sharer_shareinfo=a94ad7ee130b31cb7853f8059d96d23a&sharer_shareinfo_first=a94ad7ee130b31cb7853f8059d96d23a#rd
saved: 2026-05-09 23:31:41
tags:
  - 笔记同步助手
id: 2d42735d-3370-4773-87d6-5c9fd0d625ab
---

公众号名称：二进制茶馆

作者名称：咔哒

发布时间：2026-03-13 21:19

# Claude HUD - 让 Claude Code 状态一目了然

你是否在使用 Claude Code 时遇到过这些问题？

-   写着写着代码，想要查看一下 "context usage"
    
-   不知道 Claude 正在使用哪些工具
    
-   想查看任务进度却无从得知
    
-   想知道会话已经运行了多久
    

**Claude HUD** 就是来解决这些问题的！这是 GitHub 上一个拥有 **4.3k Star** 的热门插件。

---

## 🎯 Claude HUD 是什么？

Claude HUD 是一个 Claude Code 插件，它能在你的终端底部实时显示：

| 显示内容 | 作用 |
| --- | --- |
| 
## 项目路径

 | 知道当前在哪个项目中工作（可配置显示1-3级目录） |
| 

## 上下文健康度

 | 准确知道上下文窗口用了多少，在出问题前及时察觉 |
| 

## 工具活动

 | 实时观看 Claude 读取、编辑、搜索文件的过程 |
| 

## 代理跟踪

 | 查看子代理正在运行什么任务 |
| 

## 待办进度

 | 实时跟踪任务完成情况 |

---

## 👀 它长什么样？

### 默认视图（2行）

```
[Opus | Max] │ my-project git:(main*)
Context █████░░░░░ 45% │ Usage ██░░░░░░░░ 25% (1h 30m / 5h)
```

-   **第1行** — 模型名称、计划名称、项目路径、git 分支
    
-   **第2行** — 上下文进度条（绿→黄→红）和使用时长
    

### 可选视图（启用后）

```
◐ Edit: auth.ts | ✓ Read ×3 | ✓ Grep ×2    ← 工具活动
◐ explore [haiku]: Finding auth code (2m 15s) ← 代理状态
▸ Fix authentication bug (2/5)               ← 待办进度
```

---

## ⚡️ 如何安装？

在 Claude Code 中运行以下命令：

​

### 步骤 1：添加插件市场

```
/plugin marketplace add jarrodwatts/claude-hud
```

### 步骤 2：安装插件

⚠️ **Linux 用户注意**：Linux 上 `/tmp` 是独立的文件系统（tmpfs），可能导致安装失败报错 `EXDEV: cross-device link not permitted`。

**解决方法**：安装前先执行：

```
mkdir -p ～/.cache/tmp && TMPDIR=～/.cache/tmp claude
```

然后在同一个会话中运行安装命令。

安装命令：

```
/plugin install claude-hud
```

### 步骤 3：配置状态栏

```
/claude-hud:setup
```

完成！HUD 会立即显示，**无需重启**。

![](https://relay-1.bijitongbu.site/p/c5c39872fa6b87bc19959fec5d269e6b.png)

---

## 🔧 如何使用？

### 基础配置

运行以下命令进入交互式配置：

```
/claude-hud:configure
```

这个引导流程让你：

-   首次设置：选择预设（Full/Essential/Minimal），然后微调
    
-   随时自定义：开关各项功能、调整 git 显示样式、切换布局
    
-   预览效果：保存前预览 HUD 实际显示效果
    

### 预设选择

| 预设 | 显示内容 |
| --- | --- |
| 
## Full（完整）

 | 全部启用 — 工具、代理、待办、git、使用量、时长 |
| 

## Essential（精简）

 | 活动行 + git 状态，最小化干扰 |
| 

## Minimal（极简）

 | 仅核心 — 模型名称和上下文条 |

### 手动配置

也可以直接编辑配置文件：`～/.claude/plugins/claude-hud/config.json`

**常用配置项**：

| 选项 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `lineLayout` | string | `expanded` | 布局：`expanded`（多行）或 `compact`（单行） |
| `pathLevels` | number | 1 | 项目路径显示的目录层级（1-3） |
| `gitStatus.enabled` | boolean | true | 显示 git 分支 |
| `gitStatus.showDirty` | boolean | true | 显示未提交更改 `*` |
| `display.showDuration` | boolean | false | 显示会话时长 `⏱️ 5m` |
| `display.contextValue` | string | `percent` | 上下文显示格式：`percent`/`tokens`/`remaining` |

---

## 🛠️ 技术原理

Claude HUD 使用 Claude Code 原生的 **statusline API**：

```
Claude Code → stdin JSON → claude-hud → stdout → 显示在终端
↘ transcript JSONL (工具、代理、待办)
```

**特点**：

-   ✅ 使用 Claude Code 原生 token 数据（不是估算的）
    
-   ✅ 解析 transcript 获取工具/代理活动
    
-   ✅ 每 300ms 刷新一次
    
-   ✅ 无需额外窗口，无需 tmux，兼容任何终端
    

---

## 📎 相关链接

-   GitHub 仓库：https://github.com/jarrodwatts/claude-hud\[1\]
    
-   Star 数：4.3k ⭐
    

---

_如果你在使用 Claude Code，这个插件绝对值得一试！特别是对于长时间对话和复杂任务，能帮助你更好地掌握会话状态。_

​

### 引用链接

\[1\]_https://github.com/jarrodwatts/claude-hud_

  

---

![cover_image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/Tibs6tZp0ThknzWLFYR4plcI3JZEr7b31LZX6wS542KhuoIbUSjUJJTEqibAGfKwYTOdhM51MxVcC3pRSCF3P69Y8LicT4uLiaFRn39Qa4qvvH0/0?wx_fmt=jpeg)

咔哒 二进制茶馆

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/19ddf912_1778340699595?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNjY4NTQ1Mg%3D%3D%26mid%3D2247484821%26idx%3D1%26sn%3D474b54598e11701ae045c75717883296%26chksm%3Dc391c5a8a7c365eaff731e034618c3664472a9118575d2a2ea612f4b6183030fc9c6c6a44b0a%26mpshare%3D1%26scene%3D1%26srcid%3D05097yQIkh9Zsw7LIp23kE0o%26sharer_shareinfo%3Da94ad7ee130b31cb7853f8059d96d23a%26sharer_shareinfo_first%3Da94ad7ee130b31cb7853f8059d96d23a%23rd&s=obsidian)