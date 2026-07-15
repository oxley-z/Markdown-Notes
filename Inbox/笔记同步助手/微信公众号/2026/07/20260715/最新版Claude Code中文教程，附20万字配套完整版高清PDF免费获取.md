---
author: 武哥
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzAwMjk5Mjk3Mw==&mid=2247502709&idx=1&sn=386404fb93c1e7d4109617ccdd1e9fff&chksm=9b7c3f8fd2ebf169168132586579056e438504992ac2f44dd964d1c6706057d571e9fce4ed9d&mpshare=1&scene=1&srcid=0715cBVLmkoToXhNnKUxZ41j&sharer_shareinfo=8deb6636ced279a3257130561e8907d4&sharer_shareinfo_first=8deb6636ced279a3257130561e8907d4#rd
saved: 2026-07-15 12:17:29
tags:
  - 笔记同步助手
id: 45f449b3-cf35-4dd1-aca6-a2a9a812670c
---

公众号名称：武哥聊编程

作者名称：武哥

发布时间：2026-07-15 11:54

原文链接：[https://claude.aigcbaba.com/](https://claude.aigcbaba.com/)

最近我一直在整理一套面向中文用户的 Claude Code 教程，网站已经上线。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d2c3d01d920dc8dd12e7bc6480c62df9_MD5.jpg]]

在线阅读：

https://claude.aigcbaba.com

配套20万字PDF高清完整版，本公众号私信回复：cc中文教程，即可获取。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/741d1a49c6922784b3eb47a3f8cf71cc_MD5.jpg]]

这个网站的名字很直接，叫Claude Code 中文喂饭教程。为什么叫“喂饭教程”？因为我发现很多人不是不想学 AI 编程，也不是不想用 Claude Code，而是卡在第一步：

-   不知道 Claude Code 是什么
    

-   不知道 CLI、IDE 扩展、桌面版、Web 该用哪个
    

-   不知道国内大模型怎么配置
    

-   不知道怎么让 Claude Code 读项目、改代码、检查结果
    

-   更不知道怎么把 AI 编程用成一套稳定的工作流
    

很多教程讲得太抽象，一上来就是概念、参数、配置文件。但真正的新手需要的是：

> 我现在应该点哪里？下一步该做什么？这一步做完怎么验证？出错了先查哪里？
> 
> ![[Inbox/笔记同步助手/微信公众号/2026/07/images/f2821280fdb7e4b2991ce57828f6d945_MD5.gif||45]]

所以我做了这个网站。

> 01

> 这个网站适合谁？

![[Inbox/笔记同步助手/微信公众号/2026/07/images/77376a53d7f3afc3b8d13c195e26bd0d_MD5.jpg||60]]

如果你是下面几类人，可以重点看一下：

刚开始接触 Claude Code 的朋友

你可能听过 AI 编程、vibe coding、coding agent，但还没有真正把工具跑起来。这个网站会从“Claude Code 是什么”“该选哪个入口”“怎么安装和登录”开始讲，不默认你已经懂一堆英文文档。

想用国内大模型接入 Claude Code 的朋友

很多国内用户会遇到模型访问、API Key、Base URL、provider 配置这些问题。网站里单独整理了用 cc switch 切换器接入 DeepSeek、通义千问、Kimi、硅基流动、智谱 GLM、豆包/火山方舟等国内模型的配置思路。

这里我也专门补了一点实操内容：如果你没有 Claude 官方账号，或者网络环境不方便，想通过国内大模型使用 Claude Code，教程里会带你用 cc switch 把官方 Claude、国内主力网关、备用网关做成多个配置档，需要时一键切换。不只是 DeepSeek，像通义千问、Kimi、硅基流动、智谱 GLM、豆包/火山方舟这类国内模型，也都可以按同一套思路去配置。

cc switch 是免费工具，可以直接到官网下载：https://ccswitch.io/zh/

如果你在配置过程中卡住，也欢迎关注我的公众号武哥聊编程，我会持续更新 Claude Code 的中文教程和踩坑记录。

已经会写代码，但想把 Claude Code 用到真实项目里的人

Claude Code 不只是“生成一段代码”。真正有价值的是让它读项目、限制范围、修改代码、运行检查、解释 diff、完成验收。这个网站会围绕真实工程流程来讲，而不是只给你几句提示词。

想把 AI 编程能力用于求职、作品集和长期成长的人

我也专门加了 AI 求职相关内容：怎么用项目证明 Claude Code 能力，怎么写进简历，面试时怎么讲清楚自己和 AI 的协作流程。

当然，如果你是完全零基础，还没有学过编程，也没有做过完整项目，我更建议你先去主站 AIGC 编程网补基础：https://aigcbaba.com。AIGC 编程网更适合从计算机小白到实战开发的学习路线，重点是通过项目补齐前端、后端、接口、数据库、部署这些基础能力。因为 Claude Code 不是“零基础直接变程序员”的捷径。更合理的路径是：开发基础 -> 实战项目 -> AI 实战能力 -> Claude Code 提效。

开发能力是入场券，Claude Code 是加速器。没有入场券，加速器帮不上忙。

> 02

> 这个网站主要讲什么？

![[Inbox/笔记同步助手/微信公众号/2026/07/images/77376a53d7f3afc3b8d13c195e26bd0d_MD5.jpg||60]]

目前网站内容主要分成几块。

1\. 快速开始

适合第一次接触 Claude Code 的人，从入门教程、国内模型配置、常见问题、适合人群开始。

2\. 入口地图

Claude Code 有 Terminal CLI、VS Code / JetBrains 扩展、桌面版、Web / mobile，不同入口适合不同场景。很多人第一步就选错了工具，所以我把这些入口的使用场景拆开讲：改本地项目、跑检查、看 Git 优先用 CLI；只想解释当前文件优先 IDE；远程跟进任务再用 Web 或 mobile。

3\. 安装与配置

包括 Windows PowerShell / WinGet / npm 安装、CLI 登录、桌面版配置、IDE 扩展，以及国内大模型配置。目前站内已经有基于 cc switch 的 DeepSeek、通义千问、Kimi、硅基流动、智谱 GLM、豆包/火山方舟等配置思路，后续也会继续更新。

4\. 使用实战

这部分是我最想强调的：不要只把 Claude Code 当成聊天工具，而是把它用进真实项目流程。

比如：

-   第一次让 Claude Code 阅读项目
    

-   让它先读项目、进 Plan Mode 再动手
    

-   如何用权限模式限制它只改指定范围
    

-   如何让它自己做必要检查、跑测试
    

-   如何审查 diff、完成验收
    

-   Git 提交前怎么检查
    

-   一次 Claude Code 任务的完整闭环
    

5\. 进阶能力

包括 CLAUDE.md 项目记忆、权限与设置、MCP、Skills、Hooks、Subagents 等内容。这些不是一开始就必须学，但当你想把 Claude Code 用成长期稳定流程时，它们会非常有价值。

6\. 排障手册

登录失败、模型接口 401/404/超时、权限卡住、上下文丢失、过度修改、Git 安全（unsafe repository）等常见问题，我会逐步整理成中文排障指南。

> 03

> 我为什么要做这个网站？

![[Inbox/笔记同步助手/微信公众号/2026/07/images/77376a53d7f3afc3b8d13c195e26bd0d_MD5.jpg||60]]

一句话：

> 我希望更多中文用户，不只是“尝鲜 AI 编程”，而是真的把 Claude Code 用成能交付结果的工程协作者。
> 
> ![[Inbox/笔记同步助手/微信公众号/2026/07/images/f2821280fdb7e4b2991ce57828f6d945_MD5.gif||45]]

现在很多人对 AI 编程有两个极端误解。一种是过度期待，觉得有了 AI 就不用学编程了。另一种是完全排斥，觉得 AI 写代码不可靠，所以不值得用。

我自己的看法更中间一点：Claude Code 不是替你负责的程序员。它更像一个很强的协作助手。你目标讲得越清楚，范围控得越好，验收做得越认真，它的价值就越大。真正重要的不是“让 AI 随便生成代码”，而是建立一套可控流程。

这才是我认为更专业、更高效的 vibe coding。

> 04

> 这个网站会持续更新

![[Inbox/笔记同步助手/微信公众号/2026/07/images/77376a53d7f3afc3b8d13c195e26bd0d_MD5.jpg||60]]

目前网站已经有一批基础内容，后面我会继续补充：

-   更多真实项目实战案例
    

-   更多国内大模型配置经验
    

-   Claude Code CLI、IDE、桌面版、Web 的组合用法
    

-   企业项目里如何使用 Claude Code
    

-   求职作品集和面试表达
    

-   业务扩展实战内容
    

如果你正在学 AI 编程，或者想系统了解 Claude Code，可以先收藏这个网站：

https://claude.aigcbaba.com

如果你还没有开发基础，想先补项目实战，可以去 AIGC 编程网：

https://aigcbaba.com

希望这套教程能帮你少踩坑，真正把 Claude Code 用起来。

如果觉得对你有用，转发给身边的小伙伴～

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/fbe163eb_1784089047680?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzAwMjk5Mjk3Mw%3D%3D%26mid%3D2247502709%26idx%3D1%26sn%3D386404fb93c1e7d4109617ccdd1e9fff%26chksm%3D9b7c3f8fd2ebf169168132586579056e438504992ac2f44dd964d1c6706057d571e9fce4ed9d%26mpshare%3D1%26scene%3D1%26srcid%3D0715cBVLmkoToXhNnKUxZ41j%26sharer_shareinfo%3D8deb6636ced279a3257130561e8907d4%26sharer_shareinfo_first%3D8deb6636ced279a3257130561e8907d4%23rd&s=obsidian)