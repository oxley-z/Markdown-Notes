---
author: 九皋山人者
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzIxNTUxNDA5NQ==&mid=2247486594&idx=1&sn=91c09f29223465c32b37f4090dfa46b4&chksm=96a5f02cfa2d581628db873dfd49f2059013bebfbb33eb70add5ff747d502de58f8fd879f454&mpshare=1&scene=1&srcid=0509ZmMXAGTGEUs9kv6AoidX&sharer_shareinfo=8af01de77b1c25f84c3f0d8091f8cb4d&sharer_shareinfo_first=8af01de77b1c25f84c3f0d8091f8cb4d#rd
saved: 2026-05-09 23:34:32
tags:
  - 笔记同步助手
id: f62bd7f7-1ec0-40d9-b004-79e25ea5e4c9
---

公众号名称：AI 方寸山

作者名称：九皋山人者

发布时间：2026-03-21 00:04

![[../../../../images/9ecbaa9c4f94abbad8d459c5b93d21ab_MD5.jpg]]

> 原文：50 Claude Code Tips and Best Practices For Daily Use  
> 作者：Vishwas@CodevolutionWeb

> Claude Code 的使用技巧 50 条，看了就能学会。主要偏向于小的设置和操作，认真看了一遍，自己达标 95%，非常实用。故在此转译分享给使用 Claude Code 的你。

你用 Claude Code 已经够久了，知道它确实能打。现在你想要的是，把所有能榨出来的优势都榨干。我整理了 50 条 Claude Code 的最佳实践和使用技巧，不管你才上手一周，还是已经深度用了几个月，都能派上用场。内容来源包括 Anthropic 官方文档、Claude Code 的构建者 Boris Cherny、社区经验，以及我自己过去一年里的日常实战。

​

## 1\. 设置 `cc` 别名

这是我每次启动 Claude Code 会话的方式。把下面这行加进你的 **`～/.zshrc`**（或 **`～/.bashrc`**）：

```
alias cc='claude --dangerously-skip-permissions'
```

执行 **`source ～/.zshrc`** 让它生效。之后你输入 **`cc`** 就行，不用每次敲 **`claude`**，也能跳过所有权限确认。这个参数名故意取得很吓人。只有在你彻底明白 Claude Code 会对你的代码库做什么、能做什么之后，再用它。我在关于自定义 Claude Code 的文章里还讲了更多类似的别名。

​

## 2\. 在前面加 `!`，直接内联运行 bash 命令

输入 **`!git status`** 或 **`!npm test`**，命令会立刻执行。命令本身和输出结果都会进入上下文，所以 Claude 能看到结果并继续处理。这比先让 Claude 帮你运行命令更快。

​

## 3\. 按 `Esc` 停止 Claude，按两次 `Esc` 回滚任何东西

按一次 `Esc`，Claude 会立刻停止当前动作，但不会丢失上下文。你可以马上改方向。

按两次 `Esc`（或者用 **`/rewind`**）会打开一个可滚动菜单，里面列出 Claude 创建过的所有检查点。你可以恢复代码、恢复对话，或者两者一起恢复。直接说 “Undo that” 也可以。总共有四种恢复方式：恢复代码和对话、只恢复对话、只恢复代码，或者从某个检查点开始往后做摘要。

这意味着，你可以放心尝试一个自己只有 40% 把握的方案。成了最好，不成就回滚，基本零损伤。唯一要注意的是：检查点只跟踪文件编辑。通过 bash 命令产生的变更，比如数据库迁移或数据库操作，是不会被记录进去的。

如果你想接着上次的进度继续，**`claude --continue`** 会恢复最近一次对话，**`claude --resume`** 会打开一个会话选择器。

​

## 4\. 给 Claude 一个能自查的反馈回路

要让 Claude 能发现自己的错误，就得给它一个反馈回路。把测试命令、linter 检查，或者预期输出写进提示词里。

```
把 auth 中间件从 session token 重构成 JWT。
改完后运行现有测试套件。
如果有失败，先修复，再算完成。
```

Claude 会运行测试、看到失败，再自己修好，不需要你中途介入。Boris Cherny 说，光这一点就能把质量提升 2 到 3 倍。对于 UI 改动，你可以配上 Playwright MCP server，让 Claude 打开浏览器、和页面交互，并验证 UI 是否符合预期。这个反馈回路能抓到很多单元测试漏掉的问题。

​

## 5\. 给你的语言装一个代码智能插件

LSP 插件会在 Claude 每次编辑文件后自动给出诊断信息：类型错误、未使用的导入、缺失的返回类型。Claude 能在你还没反应过来之前就看到这些问题并修掉。这是你能装的影响最大的一类插件。

选一个适合自己的，执行安装命令：

```
/plugin install typescript-lsp@claude-plugins-official
/plugin install pyright-lsp@claude-plugins-official
/plugin install rust-analyzer-lsp@claude-plugins-official
/plugin install gopls-lsp@claude-plugins-official
```

C#、Java、Kotlin、Swift、PHP、Lua 和 C/C++ 也都有对应插件。运行 `/plugin`，进入 Discover 标签页，可以浏览完整列表。你还需要在系统里安装对应的语言服务器二进制；如果缺失，插件会提示你。

​

## 6\. 用 `gh` CLI，并顺手教会 Claude 使用任何 CLI 工具

**`gh` CLI** 可以直接处理 PR、Issue 和评论，不需要额外接一个 MCP server。CLI 工具比 MCP server 更省上下文，因为它们不会把整套工具 schema 塞进你的上下文窗口里。`jq`、`curl` 以及其他标准 CLI 工具也是一样。

对于 Claude 还不认识的工具，你可以这么说：“先用 `sentry-cli --help` 学一下它怎么用，再用它找出线上最近的一条错误。” Claude 会读帮助输出，自己摸清语法，然后执行命令。哪怕是很小众的内部 CLI，它也能上手。

​

## 7\. 复杂推理时加上 `ultrathink`

这是一个关键词，会把推理 effort 提到 high，并在 Opus 4.6 上触发自适应推理。Claude 会根据问题复杂度动态分配思考资源。适合用在架构决策、棘手调试、多步骤推理，或者任何你希望 Claude 先认真想清楚再动手的任务上。

你也可以用 **`/effort`** 永久设置推理强度。对于简单任务，低一点的 effort 会更快也更便宜。effort 要和问题匹配，变量重命名这种事，没必要烧太多思考 token。

​

## 8\. 用 skills 按需扩展知识

Skill 本质上是 Markdown 文件，用来按需扩展 Claude 的知识。它和 **`CLAUDE.md`** 不一样：`CLAUDE.md` 每次会话都会加载，而 skill 只有在当前任务相关时才会加载。这样你的上下文会更干净。

你可以在 **`.claude/skills/`** 里创建 skill，也可以安装自带现成 skill 的插件（运行 **`/plugin`** 可以浏览）。适合放进 skill 的，是那些 Claude 有时需要、有时不需要的专业领域知识，比如 API 约定、部署流程、编码模式。

​

## 9\. 用手机控制 Claude Code

运行 **`claude remote-control`** 启动一个会话，然后通过 **`claude.ai/code`** 或 iOS/Android 上的 Claude App 连接进去。会话实际上跑在你的本地机器上，手机或浏览器只是一个远程窗口。你可以在外面发消息、批准工具调用、查看进度。

如果你已经用了第 1 条里的 **`cc`** 别名，Claude 本身就拥有完整权限，不需要每一步都人工批准。这样远程控制会更顺手：任务发出去，人走开，等 Claude 做完或者真遇到意外时，再用手机回来看看。

​

## 10\. 把上下文窗口扩展到 100 万 token

Sonnet 4.6 和 Opus 4.6 都支持 100 万 token 的上下文窗口。在 Max、Team 和 Enterprise 套餐里，Opus 会自动升级到 100 万上下文。你也可以在会话中途切模型：**`/model opus[1m]`** 或 **`/model sonnet[1m]`**。

如果你担心大上下文下的质量表现，可以先从 50 万开始，再逐步往上加。上下文越大，触发 compact 之前能塞进去的内容越多，但不同任务下，回答质量会有波动。你可以用 **`CLAUDE_CODE_AUTO_COMPACT_WINDOW`** 控制何时触发 compact，用 **`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`** 设置阈值百分比。把适合你工作流的甜蜜点试出来。

​

## 11\. 不确定怎么做时，用 Plan Mode

Plan Mode 适合多文件修改、陌生代码、架构类决策。它确实有开销，前面要多花几分钟，但能避免 Claude 信心满满地在错误方向上折腾 20 分钟。

小而清晰的任务就别用了。如果一个 diff 用一句话就能说清，直接干就行。你也可以随时用 **`Shift+Tab`** 在 Normal、Auto-Accept 和 Plan 这几种权限模式之间切换，不用退出当前会话。

​

## 12\. 不相关的任务之间，先跑一次 `/clear`

一个干净会话加一个明确提示词，永远胜过一个已经搅成一锅粥、持续了三小时的会话。任务换了？先 **`/clear`**。

我知道这看起来像在丢掉前面的积累，但重新开始通常效果更好。长会话会慢慢退化，因为前面堆积的上下文会淹没当前指令。花 5 秒跑个 **`/clear`**，再写一个聚焦的起始提示，通常能帮你省下 30 分钟的低效拉扯。

​

## 13\. 别替 Claude 解释 bug，直接把原始数据贴给它

用文字描述 bug 很慢。你会看到 Claude 先猜、再改、再猜、再改，来回折腾。

直接把错误日志、CI 输出或者 Slack 讨论串贴给它，然后说一句“修”。Claude 会读分布式系统日志，顺着 trace 去找哪里出的问题。你自己的解释，往往是在增加抽象层，反而把定位根因最关键的细节给丢了。把原始数据给它，然后让开。

这对 CI 也一样有效。把 CI 输出贴上，再说一句“去把失败的 CI 测试修掉”，这是我见过最稳定的模式之一。你也可以贴一个 PR URL 或 PR 编号，让 Claude 去看失败的 checks 并修复。如果第 6 条里的 `gh` CLI 已经装好，后面的事它自己会接上。

你也可以直接从终端把输出管道给 Claude：

```
cat error.log | claude "解释这个错误并给出修复建议"
npm test 2>&1 | claude "修复失败的测试"
```

## 14\. 用 `/btw` 处理快速插问

**`/btw`** 会弹出一个覆盖层，让你提一个快速问题，而且不会把它写进当前对话历史。我常用它来问和当前会话相关的小澄清，比如：“你为什么选这个方案？”或者“另一个方案的代价是什么？”答案会显示在一个可关闭的浮层里，主上下文仍然保持干净，Claude 也能继续工作。

​

## 15\. 用 `--worktree` 创建隔离的并行分支

**`claude --worktree feature-auth`** 会创建一个隔离的工作副本，并新建对应分支。Claude 会帮你把 git worktree 的创建和清理都处理掉。

Claude Code 团队把这称作生产力上的巨大突破之一。你可以一口气拉起 3 到 5 个 worktree，每个 worktree 里跑一个独立的 Claude 会话。我自己通常开 2 到 3 个。每个 worktree 都有自己的会话、自己的分支、自己的文件系统状态。

本地 worktree 的上限，其实就是你机器的上限。多个 dev server、build 和 Claude 会话一起抢 CPU，很快就会吃满。Builder.io 的做法是把每个 agent 放进独立的云容器里，并给一个浏览器预览，这样你的本地机器就能留给那些真的需要你亲自上手的工作。

​

## 16\. 用 `Ctrl+S` 暂存你的提示词

你正写到一半一大段提示词，突然发现自己得先问个小问题。按 **`Ctrl+S`** 可以把这段草稿暂存起来。你先提完那个小问题并提交，原来的草稿会自动恢复。

​

## 17\. 用 `Ctrl+B` 把长任务扔到后台

当 Claude 启动一个耗时很长的 bash 命令，比如跑测试、执行构建、跑迁移，按 **`Ctrl+B`** 就能把它送到后台。Claude 会在后台进程继续运行时接着工作，你也可以继续聊天。等进程结束，结果会自动回来。

​

## 18\. 加一条实时状态栏

状态栏本质上是一个 shell 脚本，会在 Claude 每轮交互后运行一次。它会在终端底部显示实时信息，比如当前目录、git 分支，以及用颜色标出来的上下文占用情况。

最快的配置方式是在 Claude Code 里直接运行 **`/statusline`**。它会问你想显示什么，然后帮你生成脚本。我在自定义 Claude Code 的文章里还写了完整的配置过程和可直接复制的脚本。

​

## 19\. 用 subagents 保持主上下文干净

比如你可以说：“用 subagents 去弄清楚支付流程里失败交易是怎么处理的。” 这会拉起一个独立的 Claude 实例，拥有自己的上下文窗口。它会去读相关文件、理解代码库，然后回传一份简洁总结。

这样你的主会话能保持干净，留出更多空间用来真正构建东西。一次深入调查，很容易在你还没写任何代码前，就先吃掉半个上下文窗口。Subagent 的价值，就是把这部分成本隔离出去。内置类型包括 Explore（Haiku，适合快速搜文件）和 Plan（只读分析）。如果想看更完整的能力，可以去看关于 subagents 和 agent teams 的指南。

​

## 20\. 用 agent teams 做多会话协同

这还是实验功能，但很强。先在设置或环境变量里加上 **`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`**。然后你就可以对 Claude 说：“创建一个 3 人的 agent team，并行重构这些模块。” Team lead 会把工作拆给各个 teammate，每个人都有自己的上下文窗口和共享任务列表，teammate 之间还能互相发消息协作。

一开始建议 3 到 5 个 teammate，每人 5 到 6 个任务。不要把会修改同一个文件的任务分给不同人。两个 teammate 同时改一个文件，结果通常就是互相覆盖。建议先从研究类和评审类任务开始，比如 PR review、bug 调查，然后再试并行实现。

​

## 21\. 用指令引导 compaction

当上下文发生 compact（自动触发，或你手动执行 **`/compact`**）时，你要明确告诉 Claude 应该保留什么，比如：“`/compact focus on the API changes and the list of modified files.`” 你也可以在 **`CLAUDE.md`** 里加长期指令：“compact 时保留完整的已修改文件列表和当前测试状态。”

​

## 22\. 用 `/loop` 做周期性检查

**`/loop 5m check if the deploy succeeded and report back`** 会在后台安排一个周期性提示词，只要会话还开着，它就会按周期触发。时间间隔可选，默认 10 分钟，支持 **`s`**、**`m`**、**`h`** 和 **`d`** 这些单位。你也可以循环执行其他命令，比如 **`/loop 20m /review-pr 1234`**。这些任务是会话级的，3 天后会自动过期，所以不会因为忘了关就一直跑。`/loop` 很适合用来盯部署、看 CI 流水线，或者轮询某个外部服务，让你能把注意力放在别的事情上。

​

## 23\. 用语音输入写出信息更丰富的提示词

运行 **`/voice`** 开启按住说话，然后按住 **`Space`** 开始口述。你的语音会被实时转写进提示词里，而且你可以在同一条消息里混用语音和键盘输入。口述提示词的一个天然优势是：你往往会讲出更多背景、约束和目标，不会因为嫌打字麻烦而省略关键信息。这个功能需要 **Claude.ai** 账号，而不是 API key。你还可以在 **`～/.claude/keybindings.json`** 里把按住说话的按键改成 **`meta+k`** 这类组合键，省掉按键识别预热时间。

​

## 24\. 同一个问题纠正两次还没好，就重新开一局

如果你和 Claude 已经在同一个问题上反复纠正了两轮，结果还是没修好，那么上下文里现在已经堆满了失败方案，这些东西正在主动拖累下一次尝试。直接 **`/clear`**，然后用一个更好的起始提示，把你刚才学到的东西写进去。一个上下文干净、提示更锋利的新会话，几乎总是比一个拖着大量死路历史的长会话更有效。

​

## 25\. 直接告诉 Claude 应该看哪些文件

用 **`@`** 可以直接引用文件：比如 **`@src/auth/middleware.ts`** 里有 session 处理逻辑。`@` 前缀会自动解析成文件路径，所以 Claude 一开始就知道该看哪里。

Claude 当然也能自己 grep、自己搜代码库，但它仍然需要先收敛候选范围，再识别哪个文件才是对的。每多一次搜索，都会多消耗 token 和上下文。你一开始就把正确文件指给它看，等于把整个搜索成本都省掉了。

​

## 26\. 用模糊提示探索陌生代码

“你会怎么改进这个文件？” 是一个非常好的探索型提示。不是每个提示词都必须精确到不能再精确。有时候你想让 Claude 以一个新鲜视角去读现有代码，模糊一点的问题，反而能给它更多空间，让它提出一些你自己不会想到的观察。

我在刚接手陌生仓库时经常这么用。Claude 往往能指出我第一次读时看不出来的模式、不一致之处，和一些值得改进的地方。

​

## 27\. 用 `Ctrl+G` 直接编辑计划

当 Claude 给出一个计划后，按 **`Ctrl+G`** 会在你的文本编辑器里打开这份计划，你可以直接改。加限制、删步骤、改方向，都可以在 Claude 动手写一行代码之前完成。这个功能特别适合“整体方向差不多对，但我想微调几步”的情况，不用重新把整套思路解释一遍。

​

## 28\. 运行 `/init`，然后把结果砍掉一半

**`CLAUDE.md`** 是项目根目录下的一个 Markdown 文件，用来给 Claude 提供持续性的指令，比如构建命令、编码规范、架构决策、仓库约定。Claude 每次会话开始时都会读取它。**`/init`** 会根据你的项目结构自动生成一个起步版本，把构建命令、测试脚本和目录结构都扫出来。

但它生成的结果通常偏臃肿。如果你都说不清某一行为什么要放在那里，那就删掉。去掉噪音，补上真正缺的东西。关于怎么组织这种文件，我在另一篇写 `CLAUDE.md` 的指南里讲得更细。

​

## 29\. 判断 `CLAUDE.md` 每一行值不值得保留的标准

对于 **`CLAUDE.md`** 里的每一行，都问自己一句：如果没有这行，Claude 会不会犯错？如果 Claude 本来就能自己做对，那这行就是噪音。每多一行不必要的指令，真正重要的那些指令就会被稀释一点。大致上，指令预算在 150 到 200 条左右，再多，遵循度就开始下降，而系统提示本身已经占掉了大约 50 条。

​

## 30\. Claude 犯错之后，就让它“更新你的 `CLAUDE.md`，保证别再犯”

当 Claude 出错时，你可以直接说：“更新 **`CLAUDE.md`**，确保这件事以后别再发生。” Claude 会自己把这条规则写进去。到下一个会话，它就会自动遵守。

随着时间推移，你的 **`CLAUDE.md`** 会变成一个由真实错误不断塑形出来的活文档。为了避免它无限膨胀，可以用 **`@imports`**（第 32 条）引用单独的文件，比如 **`@docs/solutions.md`**，把模式和修复经验放进去。这样你的 **`CLAUDE.md`** 能保持精简，而 Claude 又能在需要时按需读取细节。

​

## 31\. 把只在特定场景生效的规则放进 `.claude/rules/`

你可以把 Markdown 规则文件放进 **`.claude/rules/`**，按主题组织说明。默认情况下，每个规则文件都会在会话开始时加载。如果只想让某条规则在 Claude 处理某些文件时才加载，可以给它加上 **`paths`** frontmatter。

​

## 32\. 用 `@imports` 让 `CLAUDE.md` 保持精简

你可以用 **`@docs/git-instructions.md`** 这样的形式引用文档。也可以引用 **`@README.md`**、**`@package.json`**，甚至 **`@～/.claude/my-project-instructions.md`**。

Claude 会在需要时再读取这些文件。你可以把 **`@imports`** 理解成一句话：“如果你需要更多上下文，这里有。” 好处是，它不会把这些额外内容一股脑塞进每次会话都要读的主文件里。

​

## 33\. 用 `/permissions` 把安全命令加入白名单

别再为了 **`npm run lint`** 这种命令点第一百次“批准”了。**`/permissions`** 可以把你信任的命令加入白名单，这样你能保持工作流不断档。名单之外的命令仍然会照常提示你确认。

​

## 34\. 想让 Claude 放开手脚工作时，用 `/sandbox`

运行 **`/sandbox`** 可以开启操作系统级隔离。写操作会被限制在项目目录里，网络请求也会被限制到你批准过的域名。它在 macOS 上用 Seatbelt，在 Linux 上用 bubblewrap，所以限制会作用到 Claude 拉起的每一个子进程。在 auto-allow 模式下，沙箱内的命令可以不经过权限提示直接执行，相当于给了你接近完全自治的体验，但还带着护栏。

如果你想让 Claude 做一些无人值守的工作，比如夜间迁移、实验性重构，建议直接在 Docker 容器里跑。容器能提供完整隔离、方便回滚，也能让你更放心地让 Claude 连跑几个小时。

​

## 35\. 为重复任务创建自定义 subagent

这和第 19 条里临时拉起 subagent 不一样。自定义 subagent 是预先配置好的 agent，保存在 **`.claude/agents/`** 里。比如，一个用 Opus 且只有只读工具的 security-reviewer，或者一个为了快而使用 Haiku 的 quick-search agent。

你可以用 **`/agents`** 浏览和创建它们。对于需要自己文件系统副本的 agent，还可以设置 **`isolation: worktree`**。

​

## 36\. 给你的技术栈挑对 MCP server

最值得优先上的 MCP server 有这几个：**Playwright**，适合浏览器测试和 UI 验证；**PostgreSQL/MySQL**，适合直接查询 schema；**Slack**，适合读取 bug 报告和讨论上下文；**Figma**，适合设计到代码的工作流。

Claude Code 支持动态工具加载，所以只有在 Claude 真要用的时候，这些 server 的定义才会被加载进来。如果你想看更全面的列表，可以去看 2026 年最值得用的 MCP server 指南。

​

## 37\. 设置你的输出风格

运行 **`/config`**，选择你偏好的输出风格。内置选项包括 Explanatory（详细、分步骤）、Concise（简短、行动导向）和 Technical（精确、术语友好）。

你也可以把自定义输出风格做成文件，放到 **`～/.claude/output-styles/`** 里。

​

## 38\. 建议放进 `CLAUDE.md`，硬性要求用 hooks

**`CLAUDE.md`** 本质上是建议性的。Claude 大约会遵循其中 80% 的内容。Hook 则是确定性的，命中率接近 100%。如果某件事必须每次都发生、不能有例外，比如格式化、lint、安全检查，那就把它做成 hook。若只是希望 Claude 在决策时参考，那写进 **`CLAUDE.md`** 就够了。

​

## 39\. 用 PostToolUse hook 自动格式化

每次 Claude 编辑文件后，你的 formatter 都应该自动跑起来。你可以在 `.claude/settings.json` 里加一个 PostToolUse hook，让它在 Claude 编辑或写入文件后，自动对目标文件运行 Prettier（或者你自己的 formatter）。

**`|| true`** 的作用，是避免 hook 失败反过来卡住 Claude。你还可以把其他工具串进去，比如再加一个 **`npx eslint --fix`** 作为第二个 hook。

如果你正好也用编辑器打开着同一批文件，Claude 工作期间可以考虑先关掉 format-on-save。有开发者反馈，编辑器保存动作可能会让 prompt cache 失效，迫使 Claude 重新读取文件。格式化这件事，让 hook 来做更稳。

​

## 40\. 用 PreToolUse hook 拦截破坏性命令

你可以在 Bash 工具上挂一个 PreToolUse hook，拦截 **`rm -rf`**、**`drop table`**、**`truncate`** 这种模式。这样 Claude 连尝试都不会尝试。这个 hook 会在 Claude 实际执行工具之前触发，所以能在破坏发生之前就把命令拦下来。

把它加进项目的 **`.claude/settings.json`** 里即可。你可以交互式用 **`/hooks`** 配，也可以直接对 Claude 说：“加一个 PreToolUse hook，拦截 `rm -rf`、`drop table` 和 `truncate` 命令。”

​

## 41\. 用 hook 在 compaction 后保住关键信息

长会话里一旦发生 context compaction，Claude 很容易把你到底在做什么给忘掉。一个带 **`compact`** matcher 的 Notification hook，可以在每次 compact 发生后，自动把关键上下文重新注入回来。

你可以直接对 Claude 说：“设置一个 Notification hook，在 compact 后提醒你当前任务、已修改文件和各种限制条件。” Claude 会把这个 hook 配进设置里。适合重新注入的内容包括：当前任务描述、已修改文件列表、以及那些硬性约束，比如“不要改 migration 文件”。

这在多小时持续作业的长会话里尤其有价值，因为那时候你往往已经深陷某个功能，不允许 Claude 轻易丢线。

​

## 42\. 认证、支付和数据变更，一律人工复核

Claude 很会写代码。但这几类决策必须有人亲自看：认证流程、支付逻辑、数据变更、破坏性数据库操作。不管其他地方看起来多顺，这些地方都要人工复核。一个权限范围配错的认证逻辑、一个支付 webhook 配错、一次悄无声息删列的迁移，代价都可能是用户、收入，或者信任。自动化测试并不能覆盖这里面的所有风险。

​

## 43\. 用 `/branch` 试另一条路，不丢当前进度

**`/branch`**（或者 **`/fork`**）会在当前节点复制出一条对话分支。你可以在分支里尝试那个冒险的重构方案。成了，就留着；不成，你原来的对话完全不受影响。它和第 3 条里的 rewind 不同，rewind 是回到过去，而这里是两条路径都保留下来。

​

## 44\. 当你自己也说不清需求时，让 Claude 反过来采访你

你知道自己想做什么，但又觉得手头的信息还不足以让 Claude 高质量实现。这时候，可以让 Claude 先向你提问。

```
我想做 [简要描述]。请使用 AskUserQuestion 工具详细采访我，
重点问技术实现、边界情况、顾虑和权衡。
不要问显而易见的问题。
一直问到我们把关键点都覆盖完为止，
然后把完整规格写到 SPEC.md。
```

等规格文档定稿后，再开一个新会话来执行。这样上下文是干净的，规格也完整了。

​

## 45\. 让一个 Claude 写，另一个 Claude 审

第一个 Claude 负责实现功能，第二个 Claude 站在全新上下文里，以 staff engineer 的视角来 review。因为 reviewer 不知道实现时走过哪些捷径，它会把每一个捷径都重新拎出来质疑一遍。

这个思路也适用于 TDD。A 会话写测试，B 会话写代码去通过这些测试。

​

## 46\. 用对话式方式审 PR

不要只让 Claude 给你做一次性 PR review，虽然你当然也可以这么做。更好的做法是：在会话里打开这个 PR，然后围绕它展开对话。“这个 PR 里风险最大的改动是什么？”“如果它并发执行，会坏在哪里？”“这里的错误处理和仓库其他部分一致吗？”

对话式审查能抓到更多问题，因为你可以一路深挖那些真正重要的地方。一次性 review 往往更容易抓出样式小问题，却漏掉架构层面的风险。

​

## 47\. 给会话命名并用颜色区分

**`/rename auth-refactor`** 会给提示条加一个标签，帮助你知道当前是哪一个会话。**`/color red`** 或 **`/color blue`** 可以设置提示条颜色。可选颜色有：red、blue、green、yellow、purple、orange、pink、cyan。当你同时跑着 2 到 3 个并行会话时，花 5 秒钟命名和上色，可以大幅降低你把消息发错终端的概率。

​

## 48\. Claude 做完时放个提示音

加一个 Stop hook，让 Claude 每次完成响应时都播放系统声音。这样你可以把任务丢出去，切去做别的事，等听到一声提示，再回来继续。

​

## 49\. 用 `claude -p` 做批处理扇出

你可以遍历一组文件，用非交互模式逐个喂给 Claude。`--allowedTools` 可以限定它在每个文件上允许做什么。再配合 `&` 并行运行，就能把吞吐量拉满。

这非常适合批量转换文件格式、统一更新导入路径，或者执行那种“每个文件彼此独立”的重复性迁移任务。

​

## 50\. 自定义转圈动画里的动词

Claude 思考时，终端里会显示一个转圈动画，配上像 “Flibbertigibbeting...” 和 “Flummoxing...” 这样的动词。你可以把这些词全换成你自己的风格。直接对 Claude 说：

把我用户设置里的 spinner verbs 替换成这些：负责任地胡思乱想、假装在思考、自信地瞎猜、甩锅给上下文窗口

你甚至不用自己列清单。你只要告诉 Claude 你想要什么风格，比如：“把 spinner verbs 换成哈利波特咒语风格。” Claude 会自己生成整套词表。这只是个小细节，但能让等待过程稍微更有意思一点。

​

## 收尾

你不需要一口气用上这 50 条。只挑一条，正好对应你上一次使用里最烦的那个问题，明天试一下就行。真正能落地的一条技巧，比你收藏了五十条却一条没用，价值大得多。

我会持续写 Claude Code 相关内容。你也可以看看我写的其他 Claude Code 指南。

\#Claude \#ClaudeCode \#Agent \#技巧 \#最佳实践 \#实践

---

![[../../../../images/231365f9acf3a8b7129065a8c3bd995a_MD5.jpg|cover_image]]

Original 九皋山人者 AI 方寸山

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/84dfb6ef_1778340870513?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIxNTUxNDA5NQ%3D%3D%26mid%3D2247486594%26idx%3D1%26sn%3D91c09f29223465c32b37f4090dfa46b4%26chksm%3D96a5f02cfa2d581628db873dfd49f2059013bebfbb33eb70add5ff747d502de58f8fd879f454%26mpshare%3D1%26scene%3D1%26srcid%3D0509ZmMXAGTGEUs9kv6AoidX%26sharer_shareinfo%3D8af01de77b1c25f84c3f0d8091f8cb4d%26sharer_shareinfo_first%3D8af01de77b1c25f84c3f0d8091f8cb4d%23rd&s=obsidian)