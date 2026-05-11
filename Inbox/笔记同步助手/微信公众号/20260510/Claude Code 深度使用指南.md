---
author: 何富贵AI
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg4MzE5NTQ3MA==&mid=2247483791&idx=1&sn=6e6ec2c6bb55281459a26f2e717e4875&chksm=ce3bef38068b4a2f6cc80541a20d11e4a5193afb48210250ffeaf42430a11dde1068967921a1&mpshare=1&scene=1&srcid=0510zq3ifBcFyV4VYr0jK5eb&sharer_shareinfo=1184701e42bfd0c08a2850e38ffb6716&sharer_shareinfo_first=1184701e42bfd0c08a2850e38ffb6716#rd
saved: 2026-05-10 10:44:45
tags:
  - 笔记同步助手
id: fe3334f0-b451-43c8-b7dc-59fca69118bf
---

公众号名称：富贵的胡思乱想

作者名称：何富贵AI

发布时间：2026-03-26 17:10

# Claude Code 深度使用指南：Opus Plan、Skill 插件与 CLI 工具全攻略

AI 编程助手已经进化到了一个新阶段。Claude Code 不是一个简单的补全插件，而是一个能在终端里真正理解你项目、帮你写代码、执行命令、管理文件的 AI 编程代理。

本文覆盖安装、多端使用、VS Code 集成、权限配置、高效提示词技巧，以及如何结合 CLI 工具把 Claude Code 的潜力发挥到极致。

---

## 一、一行命令，快速安装

Claude Code 的安装极其简单，根据你的操作系统选择对应命令即可。

**macOS / Linux / WSL：**

​

```
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell：**

​

```
irm https://claude.ai/install.ps1 | iex
```

**Windows CMD：**

​

```
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

> Windows 用户需要提前安装 Git for Windows 作为依赖环境。

安装完成后，进入任意项目目录，输入 `claude` 即可启动。

---

## 二、多端使用：Claude Code 无处不在

Claude Code 不止运行在终端里。它的核心引擎是统一的，你的 `CLAUDE.md` 配置文件、设置和 MCP 服务器在所有端都能共用。

​

| 我想要…… | 最佳方案 |
| --- | --- |
| 从手机或其他设备继续本地会话 | 远程控制（Remote Control） |
| 把 Telegram、Discord、iMessage 或自定义 Webhook 的事件推送到会话 | 频道（Channels） |
| 本地启动任务，移动端继续 | Claude 网页版或 Claude iOS 应用 |
| 定时自动运行 Claude | 云端定时任务或桌面版定时任务 |
| 自动化 PR 审查和 Issue 分类 | GitHub Actions 或 GitLab CI/CD |
| 每次 PR 自动触发代码审查 | GitHub Code Review |
| 把 Slack 里的 bug 报告路由成 Pull Request | Slack 集成 |
| 调试线上 Web 应用 | Chrome 扩展 |
| 为自己的工作流构建自定义 Agent | Agent SDK |

终端、VS Code、JetBrains、桌面应用、网页版——选择你最顺手的入口，底层都是同一个 Claude Code。

---

## 三、在 VS Code 里使用 Claude Code

VS Code 是目前最流行的代码编辑器，Claude Code 与它的配合非常自然。

推荐工作流：

1.  用 VS Code 打开项目，浏览和阅读代码文件
    
2.  打开 VS Code 内置终端（快捷键 ` Ctrl+`` ` 或 ` Cmd+`` `）
    
3.  在终端里直接运行 `claude`，启动 Claude Code
    

这样做的好处是：你可以在左边的编辑器里查看文件，在右边的终端里和 Claude Code 对话，随时检查它的修改结果，形成一个高效的工作闭环。

  

![[Inbox/笔记同步助手/images/f68221c223de2954bf49f7f0ef1f148d_MD5.jpg]]

  

---

## 四、权限设置：完全解锁 AI 写代码

Claude Code 默认每次执行文件操作或运行命令都会请求确认。如果你完全信任它来完成一个任务，这些确认弹窗会打断节奏，浪费时间。

推荐的启动方式：

​

```
ounter(line
claude --dangerously-skip-permissions
```

这个参数的含义是：跳过所有权限确认，让 Claude Code 自由执行写文件、运行命令等操作。

对于构建新项目或完成一个明确的开发任务，这种模式效率最高——完全信任 AI，让它跑完整个流程，你最后审查结果即可。

​

> 注意：在你不完全了解 Claude Code 会做什么的情况下，建议先保留默认确认模式，熟悉后再开启此参数。

---

## 五、高效提示词的七大技巧

给 Claude Code 的指令质量，直接决定了输出质量。以下七个技巧，能让你的提示词从及格变成优秀。

### 1\. 使用 Opus Plan 模式 (隐藏模式)

在 Cluade Code里输入 `/model opusplan` 启动自动opus规划sonnet执行模型，让 Claude Code 用 Opus 模型做架构设计和任务拆解，再用 Sonnet 模型负责具体执行。

这是性价比最高的工作方式——Opus 的战略思考 + Sonnet 的执行速度，既节省 Token，输出质量又高。

  

![[Inbox/笔记同步助手/images/ae02cf75ef7899da728405db3a435094_MD5.jpg]]

  

### 2\. 关注结果，而非任务

不要只描述你想要什么功能，而要描述你想达成什么目标。

差的提示词："给我建一个看板应用。"

好的提示词："目标是创建一个看板应用，让我能够整理已有内容并规划未来内容，跟踪观看量和互动等性能指标，并在多个平台上管理内容。"

描述目标，Claude Code 才能做出真正符合你需求的设计决策。

### 3\. 说清楚"为什么"，比说"做什么"更重要

提供背景和动机，能让 AI 更好地权衡取舍。

比如构建一个落地页，与其说"建一个落地页"，不如说"目的是让访客填写注册表单，其他内容都是次要的。"

这一句话，就能让 Claude Code 在设计和文案上的所有决策都指向正确的方向。

### 4\. 提供参考示例

给 Claude Code 参考素材，输出质量会显著提升：

-   文字描述：最基础，但有效
    
-   截图：比文字好，尤其是 UI 设计
    
-   代码仓库：最佳，提供完整的技术上下文
    

如果你在构建视觉内容，去 Dribbble 找一个你喜欢的设计，截图后直接拖入 Claude Code 聊天窗口。有了视觉参考，输出质量会有质的提升。

### 5\. 让 AI 像专家一样反问你

AI 的强项之一是能帮你进入你不擅长的领域。但在陌生领域里，你往往不知道自己不知道什么。

解决方法是：在提示词里加一句"请像这个领域的专家一样，指出我方案中可能遗漏的问题，并向我提问以完善方案。"

这样 Claude Code 不会只是执行你的指令，而会主动挑战你的假设，帮你发现盲点，最终输出的方案会比你自己想到的更完善。

### 6\. 合理使用 Skill（技能插件）

Skill 本质上是一段文本提示词，告诉 Claude Code 如何在特定场景下用特定方式完成工作。没有魔法，没有复杂性，但效果显著。

Skill 分两类：

-   **性能型 Skill**：教 Claude Code 把某件事做得更好，比如前端设计
    
-   **工作流型 Skill**：把你常用的多步骤流程打包成一个触发词
    

以前端设计为例：没有 Skill，Claude Code 产出的 UI 往往平平无奇。安装官方前端设计 `frontend-design` Skill 后，同样的提示词能产出截然不同的设计质量。

安装方法：在 Claude Code 里输入 `/plugin`，搜索并安装你需要的 Skill，然后重新加载插件即可。

  

![[Inbox/笔记同步助手/images/fa517ab6e10714842ab5d4dc25484990_MD5.jpg]]

  

### 7\. 主动管理上下文窗口

这是很多人会忽视的一个细节，却在悄悄影响 Claude Code 的表现。

Claude Code 有 100 万 Token 的上下文预算，随着使用增加，性能会逐渐下降：

-   前 20 万 Token（约 20%）：黄金区间，输出质量最佳
    
-   超过 20 万 Token 后：性能开始下滑
    
-   接近 100 万 Token 时：Opus 的有效性降至约 78%（Anthropic 官方基准数据）
    

应对方法很简单：**积极使用 `/clear`**。这会重置整个上下文窗口。Claude Code 不会"忘记"你的项目——所有文件都还在，它只是重新读取它需要的内容。

如果当前对话有重要上下文需要延续，先让 Claude Code 输出一个摘要 `/compact`，然后 `/clear`，把摘要粘贴进新会话即可。

输入 `/context` 可以查看当前 Token 用量。建议让 Claude Code 在提示符下方创建一个实时显示上下文用量的状态栏（有个开源项目: `claude-hub` 挺不错，目前我也在用），方便随时监控。当用量接近 20% 时(这是对于max用户而言1M上下文，pro用户的上下文是200k，当接近90%就可以清一次上下文了。)，果断清除，重新开始。

![[Inbox/笔记同步助手/images/9484c407bbb85de8c709fccb07e7ef68_MD5.jpg]]

  

---

## 六、结合 CLI 工具，让 Claude Code 如虎添翼

CLI（命令行界面）工具是让 Claude Code 真正强大起来的关键。这些工具运行在终端里——和 Claude Code 的运行环境相同——因此 Claude Code 可以直接控制它们，几乎没有额外开销。

你可能听说过 MCP。CLI 工具正在逐渐取代它，更高效，Token 成本更低，与 Claude Code 的架构配合更好。

几个实用例子：

-   **Supabase CLI**：Claude Code 可以代替你创建数据库和认证系统
    
-   **Playwright CLI**：Claude Code 可以打开浏览器，自动测试你的 Web 应用
    
-   **GitHub CLI**：用自然语言推送代码、管理仓库
    
-   **Vercel CLI**：不离开终端直接部署站点
    

大多数 CLI 工具包含两个部分：工具本身（你安装）和 Skill（教 Claude Code 如何用好它）。两者通常都能在工具的 GitHub 仓库找到。

以 Playwright 为例：复制 GitHub 仓库 URL，粘贴给 Claude Code，说"按照这个说明安装 Playwright CLI"。三行代码，浏览器自动化能力就到位了。然后你可以直接说"用 Playwright 测试我们的看板应用，写两个测试用例，用有界面的浏览器运行"，Claude Code 会打开真实的浏览器窗口，自动完成所有测试。

---

## 结语

Claude Code 的门槛远比想象中低，但它的上限远比大多数人用到的高。安装只需一行命令，真正的差距在于你如何使用它。

从描述清楚你的目标开始，给它足够的上下文，主动管理好 Token 预算，再配上合适的 Skill 和 CLI 工具——你会发现，AI 不仅仅在"帮你写代码"，而是在真正参与你的开发工作。

---

![[Inbox/笔记同步助手/images/06acdfaa7e2364d23059423e953f946d_MD5.jpg|cover_image]]

原创 何富贵AI 富贵的胡思乱想

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/6bfb19e5_1778381084425?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg4MzE5NTQ3MA%3D%3D%26mid%3D2247483791%26idx%3D1%26sn%3D6e6ec2c6bb55281459a26f2e717e4875%26chksm%3Dce3bef38068b4a2f6cc80541a20d11e4a5193afb48210250ffeaf42430a11dde1068967921a1%26mpshare%3D1%26scene%3D1%26srcid%3D0510zq3ifBcFyV4VYr0jK5eb%26sharer_shareinfo%3D1184701e42bfd0c08a2850e38ffb6716%26sharer_shareinfo_first%3D1184701e42bfd0c08a2850e38ffb6716%23rd&s=obsidian)