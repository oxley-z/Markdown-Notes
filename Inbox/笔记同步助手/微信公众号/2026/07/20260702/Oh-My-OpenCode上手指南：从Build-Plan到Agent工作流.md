---
author: 你的老W
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzYyMTkyMjU4MQ==&mid=2247483768&idx=1&sn=5f61f2c49e32a31162872f5b331bf64c&chksm=fe9478778eab6f1f09cda983477dd85a1889f2086547fb0b513f31abb061c4755003d7baeadd&mpshare=1&scene=1&srcid=0702ycEqlMaxrUWTz7w9s7Dv&sharer_shareinfo=3f8a1196708fdacf99d1c56183c7581e&sharer_shareinfo_first=3f8a1196708fdacf99d1c56183c7581e#rd
saved: 2026-07-02 09:00:56
tags:
  - 笔记同步助手
id: 22b285a8-8279-40c2-a9a7-b6955eae0b70
---

公众号名称：吾AI生活

作者名称：你的老W

发布时间：2026-05-14 09:00

今年重度使用OpenCode，后端接的自己几个Coding Plan，经常看到有人推荐增强插件 OMO，装上用了几个月，确实很爽。

顺便说一句，这个项目已经改名为 Oh-My-OpenAgent，不过很多人（比如我）还是习惯叫 Oh-My-OpenCode 或简写 OMO。

最初装完后，发现原来的两种模式 `Build` 和 `Plan` 没有了，README 里的介绍还是比较简单，但是包含了多个 Agent 和 Category，初用会有一些迷惑。于是进一步去了解了它的功能、设计和使用方式，折腾完才理清楚。

OpenCode 原生其实很好理解，核心就是两个模式：`Build` 和 `Plan`。`Build` 负责干活，能读文件、改代码、跑命令；`Plan` 只读分析模式，适合先讨论方案，不直接动代码。

这套设计很清楚，没什么理解成本。

但装上 OMO 之后，OpenCode 里会多出一套新的 Agent 系统：Sisyphus、Hephaestus、Prometheus、Atlas，还有一堆不会直接出现在 Tab 里的专家 Agent。同时 OMO 自身的配置文件里还有一套复杂的 `agents` 和 `categories` 模型配置。

先简单说明一点：

> OMO 是把 OpenCode 的工作方式扩展成了一套 Agent 工作流

原生 OpenCode：

> ```
> 提需求
> ↓
> 模型直接处理
> ↓
> 输出结果
> ```

装了 OMO 之后：

> ```
> 提需求
> ↓
> Sisyphus 判断任务
> ↓
> 决定直接处理、调用专家 Agent，或者按 Category 派发子任务
> ↓
> 必要时进入 ultrawork 长任务流程
> ↓
> 持续规划、执行、验证
> ```

OMO 让 OpenCode 具备任务编排能力，这本身也是 Harness 的一种践行。

### #日常使用选择

如果只是刚开始用 OMO，不需要一上来就研究所有 Agent。

我觉得先按这几种场景用就够了。

普通任务，直接用 Sisyphus。

> ```
> 帮我看下这个模块有没有明显问题
> ```

复杂任务，加 `ulw`。

> ```
> ulw 帮我重构这个认证模块，保留现有行为，补齐测试
> ```

只想先规划，不想立刻动代码，Tab 切 Prometheus。

> ```
> 帮我规划一下这个模块的迁移方案，先不要改文件
> ```

已经有 Prometheus 的计划，要开始执行，用 `/start-work`。

> ```
> /start-work
> ```

需要专家判断时，给 Sisyphus 一个明确提示。

> ```
> Ask @oracle to review this architecture
> Ask @librarian to check official docs
> Ask @explore to find similar implementations
> ```

这几种基本覆盖了日常使用。

小任务别硬开 ultrawork，也没必要每次都手动点名 Agent。OMO 的价值不是把所有任务都复杂化，而是遇到长任务、复杂改造、模糊问题时，它能多一层拆分、委派和检查。

### #OMO 的几个 Agent

装完 OMO 后，最直观的变化就是 Tab 切换里，原先的 `Build` / `Plan` 没有了，出现了几个新 Agent。

常见的是这四个：

| Agent | 可以怎么理解 |
| --- | --- |
| `Sisyphus` | 默认主控，负责判断意图、拆任务、委派 |
| `Hephaestus` | 深度执行者，偏复杂实现和端到端处理 |
| `Prometheus` | 规划师，适合先访谈需求、生成计划 |
| `Atlas` | 计划执行追踪者，负责读取计划、跟踪 Todo、验证完成度 |

> `Hephaestus` 有个限制：它要求必须使用 GPT-5.5，如果手上没有GPT-5.5模型可用，可以 disable 这个 Agent。

日常使用里，最常接触的其实是 `Sisyphus`。

它可以理解成 Build 的增强版，但又不完全等于 Build。原生 Build 基本就是单模型直接干活，而 Sisyphus 即使没有开启 ultrawork，也有一定编排意识：简单任务直接处理，复杂一点的任务可能会委派给别的 Agent。

`Prometheus` 更接近 Plan 的增强版。它不是只给一个大概方案，而是会更强调需求澄清、任务拆分、执行顺序和验收标准。对于复杂改造，比直接让模型上手改代码要稳一些。

`Atlas` 不是一般意义上的执行 Agent。更准确地说，它是计划执行追踪器。Prometheus 先生成计划，后面通过 `/start-work` 让 Atlas 去读取这个计划，再跟踪 Todo 和验证结果。

### #专家 Agent 作用

OMO 里不止 Tab 里那几个 Agent。

还有一些隐藏在后面的专家 Agent，比如：

| Agent | 作用 |
| --- | --- |
| `Oracle` | 架构、调试、复杂问题咨询，只读 |
| `Librarian` | 查文档、查代码、找参考实现 |
| `Explore` | 快速搜索代码库上下文 |
| `Metis` | 规划顾问，找模糊点和风险 |
| `Momus` | 规划审查员，检查计划是否足够清楚 |
| `Multimodal-Looker` | 看图片、PDF、图表 |
| `Sisyphus-Junior` | Category 路径下生成的临时执行器 |

这些不是日常 Tab 切过去用的，更像 Sisyphus 背后的“专家池”。可以通过类似 `@oracle`、`@librarian` 这样的方式给 Sisyphus 一个路由建议，让它去调用对应专家。

比如：

> ```
> Ask @oracle to review this design
> Ask @librarian how this library is usually used
> Ask @explore to find related code paths
> ```

这里的 `@oracle` 不是绕过 Sisyphus 直接跟 Oracle 对话，而是告诉 Sisyphus：这个任务更适合交给 Oracle。

这个设计挺关键。OMO 表面上是多 Agent，实际使用时还是有一个主控来判断该找谁，而不是让用户自己手动管理所有 Agent。

### #Agent 和 Category 的关系/区别

OMO 最容易看混的地方，是配置里同时有 `agents` 和 `categories`。

看起来它们都能配置模型，很容易误以为 Agent 会套用 Category 的模型配置。但实际不是这么回事，在 OMO 的 `task` 委派里，`category` 和 `subagent_type` 是两条互斥路径。

可以简单理解成：

-   `Agent`：让指定专家干活
    
-   `Category`：按任务类型选择一套模型策略，然后生成一个通用执行器
    

如果走 Agent 路径，比如 `subagent_type="oracle"`，实际执行者就是 Oracle，它用 Oracle 自己的模型和工具权限。

如果走 Category 路径，比如 `category="deep"`，实际执行者不是某个专家，而是 `Sisyphus-Junior` 这种临时执行器，模型来自 `categories.deep` 的配置。

这也是我觉得 OMO 设计里最需要理解的一点。

Agent 解决的是“谁来干”。

Category 解决的是“用什么意图/方式去干”。

它们不是叠加关系，而是委派时二选一。

### #Category 作用

Category 看起来有点抽象，但它的作用很实用：按任务类型决定该用哪套模型策略。

比如可以把任务分成这些类别：

| Category | 适合什么 |
| --- | --- |
| `quick` | 小修小补、单文件修改 |
| `deep` | 深度问题解决 |
| `ultrabrain` | 复杂推理、架构判断 |
| `visual-engineering` | 前端、UI、动画 |
| `writing` | 文档、技术写作 |

这样 Sisyphus 派发任务时，不需要直接说“调用某某模型”，而是说“这是一个 deep 任务”或“这是一个 quick 任务”。背后由配置决定具体用哪个模型。

日常用下来，大部分普通子任务其实会走 Category 路径；只有确实需要特定能力时，才会走 Oracle、Librarian、Explore 这种指定 Agent 路径。

一般情况下，主控 Sisyphus 会自动根据任务决定归类到哪个 Category 下。

### #ultrawork 开启策略

OMO 里容易被误解的另一个词是 `ultrawork`，也就是常说的 `ulw`。

它不是装完后默认一直开启。

正常情况下，直接跟 Sisyphus 对话，它只是一个更有编排意识的默认 Agent。只有当消息里包含 `ultrawork` 或 `ulw` 关键词时，才会触发 ultrawork 模式。

比如：

> ```
> ulw 把这个项目从 REST 迁移到 GraphQL
> ```

触发之后，它会进入一套更激进的工作协议：

-   先强制收集上下文
    
-   复杂任务必须先规划
    
-   能委派就委派
    
-   通过循环机制持续推进，直到认为任务完成
    

这里也有一个常见误解：ultrawork 不是一条固定流水线。

不是：

> ```
> ulw → Prometheus → Atlas → 执行
> ```

更准确的是：

> ```
> ulw 触发
> ↓
> Sisyphus 进入 ultrawork 协议
> ↓
> 按任务情况收集上下文、规划、委派
> ↓
> 每个子任务独立判断走 Category 还是 Agent
> ↓
> 循环检查是否完成
> ```

Prometheus 不是每次必经，Atlas 也不是 ultrawork 的默认执行者。Prometheus 更适合“先生成计划”，Atlas 更适合“根据已有计划执行”。ultrawork 则是一套长任务推进协议。

### #最后

整体而言，OMO 比较适合以下使用者：

-   OpenCode 重度用户
    
-   手上模型比较多的
    
-   并发执行，节省时间
    
-   希望 Agent 能主动规划、拆分、验证
    

如果只是偶尔让它解释代码、改几行配置，原生 Build / Plan 已经够了。OMO 会带来额外的复杂度和Token消耗，不一定合适。

Over……

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/958b3f24_1782954055656?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzYyMTkyMjU4MQ%3D%3D%26mid%3D2247483768%26idx%3D1%26sn%3D5f61f2c49e32a31162872f5b331bf64c%26chksm%3Dfe9478778eab6f1f09cda983477dd85a1889f2086547fb0b513f31abb061c4755003d7baeadd%26mpshare%3D1%26scene%3D1%26srcid%3D0702ycEqlMaxrUWTz7w9s7Dv%26sharer_shareinfo%3D3f8a1196708fdacf99d1c56183c7581e%26sharer_shareinfo_first%3D3f8a1196708fdacf99d1c56183c7581e%23rd&s=obsidian)