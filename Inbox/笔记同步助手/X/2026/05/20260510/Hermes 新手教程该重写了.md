---
author: 老爸的AI联萌
source: X
url: https://x.com/i/status/2053054035738108006
saved: 2026-05-10 12:39:55
tags:
  - 笔记同步助手
id: 527c2471-1b34-437a-af3e-9a31120cdb2a
---

![[../../../../images/b44d4b225e56ed06ef96ae39263b70a6_MD5.jpg|cover]]

œ我本来不想写 Hermes 新手教程。

一个已经出来这么久的工具，再写“从零开始”，听起来有点像蹭旧热度。可我最近翻了几篇旧教程，又对着 v0.13.0 重新走了一遍，反倒觉得这篇必须写。

Hermes 这种工具，最怕教程过期。

过期教程比没有教程更麻烦，因为它大部分还对。安装命令可能还能跑，provider 也能配，CLI 也能聊两句，于是人很容易以为问题出在自己身上。可新版已经把不少东西推进到生产化一侧，durable multi-agent Kanban、/goal、checkpoints v2、Gateway 自动恢复会话、no\_agent cron、Google Chat、providers plugin、Curator 维护技能库，连安全默认项都比早期严了不少。

麻烦也就藏在这里。

旧教程往往把重点放在“跑起来”，新版 Hermes 真正需要的是“养起来”。前者是安装、选模型、发第一条消息；后者还要管记忆、技能、消息入口、任务调度、profile 边界和自动化。顺序一乱，就会变成最典型的 Agent 新手灾难，工具看起来很强，人坐在屏幕前查半天 token、allowlist、base URL 和 profile 状态。

尤其是那些一上来就讲“7x24 小时 Agent 军团”的文章。看着当然兴奋，自动接单、自动写稿、自动发布、多平台调度、飞书中台、任务看板、长期记忆，一口气全摆出来，像是明天就能把一家公司塞进电脑里。

我不太建议第一天这么干。一个 bot token 填错，一个 profile 切错，一个本地模型 endpoint 写错，半天就过去了。

所以这篇还是新手教程，但不是“复制这行命令就行”的那种。

我想写的是一个更笨、也更稳的顺序，从零开始把一个 Hermes 养活，先让它稳定说话、能用工具、能恢复会话、知道什么该记、什么不该记。这个地基站住了，再去接 Gateway、Memory、Skills、Profile、Cron、Kanban。

先让一个 Hermes 稳定工作。

## Hermes 到底是什么

先别把 Hermes Agent 和 Hermes 模型混在一起。名字相近，东西不是一类。

在我看来，Hermes 更像一个 Agent runtime。模型只是脑子，Hermes 管的是脑子之外那些麻烦事，工具、会话、记忆、Skills、自动化任务、聊天入口、多 Agent profile 和任务看板。

普通聊天工具像临时喊来的助手，问完就散。Hermes 更像给这个助手安排了一张桌子，旁边有笔记本、有 SOP、有消息入口，也有一块任务板。临时问一句概念解释，用不上这么重的家伙；整理日报、追踪资料、维护项目、保存偏好、定时执行低风险任务，才是它舒服的位置。

![[../../../../images/8590fd42fd3cfad3bb641167d294f1b0_MD5.jpg]]

## 第一天不要做三件

新手最容易被“高级玩法”带跑。我也不例外，看到多平台 Gateway、多 profile、Kanban 调度，手就痒。可这类东西第一天全打开，排错面会膨胀得很快。

Telegram、Discord、Slack、WhatsApp、飞书都能接，问题是任何一个 bot token、allowlist、home channel、平台权限没配好，Hermes 看起来就像坏了。多 Agent 也一样，profile 很强，每个 profile 却都有自己的配置、密钥、记忆、sessions、skills、cron、gateway 状态；一口气创建 commander、coder、reviewer、researcher，排错面会直接翻倍。

本地模型也先放一放。本地模型当然好玩，可第一次上手时，base URL、模型名、OpenAI-compatible 接口、上下文长度、工具调用稳定性会一起涌上来。先用一个稳定云端 provider 跑通，心里有了基准，再接 Ollama 或 vLLM，会少很多冤枉路。

我现在会按这个顺序走，

\`\`\` 安装 Hermes → 选择模型 provider → 跑通 CLI / TUI 会话 → 确认 sessions 能恢复 → 配 SOUL / Memory → 用 Skills → 接 Gateway → 建 Profile → 上 Cron / Kanban / Docker \`\`\`

![[../../../../images/c0edcb042ab0b4ca4986c8bcece61131_MD5.jpg]]

先把版本跑出

安装本身不复杂，官方脚本就这一行，

curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

这行适合 macOS、Linux 和 WSL2。

如果你用的是 Windows，现在不用先绕去 WSL2 了。官方已经放出 Windows Native Beta，可以在 PowerShell 里直接装：

powershell -NoProfile -ExecutionPolicy Bypass -Command "iwr https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1 -UseB | iex"

这里要把话说完整一点。Windows 原生支持是真的，不是我脑补；但它现在仍然是 beta。CLI、TUI、Gateway、Profile、Skills 这些主线能力可以按 Windows 原生路径走，Dashboard 里的内嵌 terminal 仍然更偏 WSL2 场景。你只是想在 Windows 上把 Hermes 跑起来，可以直接走原生安装；你要做重度开发、长期自动化和复杂本地工具链，WSL2 仍然是更稳的备选。

安装完成后，重载 shell。Windows PowerShell 里则直接重开一个窗口，再跑后面的命令。

\`\`\` source ～/.bashrc # 或者 source ～/.zshrc \`\`\`

我会立刻检查版本，

\`\`\` hermes --version \`\`\`

现在的最新版输出大概是这样，

\`\`\` Hermes Agent v0.13.0 (2026.5.7) Up to date \`\`\`

这一步别省。Hermes 改得太勤，很多“教程失效”未必是作者写错了，常常只是命令长了新枝。

![[../../../../images/f2ca4d456a43a556b5541b88d4f97f6f_MD5.jpg]]

准备认真用的话，doctor 也别往后拖，

\`\`\` hermes doctor \`\`\`

它不神秘，就是把环境、配置和依赖扫一遍。比自己盯着报错猜半天强。

我还会顺手跑一次，

\`\`\` hermes --help \`\`\`

跑它不是为了背命令。先看一眼入口，心里会稳很多。model、gateway、cron、kanban、skills、curator、memory、profile、dashboard、logs 都在这一屏里，Hermes 现在的重心，其实也写在这一串命令里了。

![[../../../../images/2e79ef64493d8cec942033b9a696536f_MD5.jpg]]

第二步，先把模型接上

Hermes 不是模型，它必须接一个 provider。

最稳的入口是，

\`\`\` hermes model \`\`\`

官方支持的 provider 很多，Nous Portal、OpenRouter、OpenAI Codex、Anthropic、Kimi、Qwen、DeepSeek、Hugging Face、AWS Bedrock、GitHub Copilot、Vercel AI Gateway、自定义 OpenAI-compatible endpoint 等。

第一次用，不要追求最花哨。先跑通最要紧。

我会按这个顺序选，

\`\`\` 已有稳定 API Key → 先用它 不知道选什么 → OpenRouter 或 Nous Portal 想用国内模型 → Kimi / Qwen 先跑简单任务 想用本地模型 → 等 CLI 跑通以后再接 Ollama / vLLM \`\`\`

还有一个硬条件容易被忽略，Hermes 要求模型至少 64K context。Agent 要带工具、历史、记忆、任务状态，小上下文模型很容易半路失忆。

模型配完后，先别接 Telegram。先在终端里确认它能回答。

\`\`\` hermes --tui \`\`\`

或者经典 CLI，

\`\`\` hermes \`\`\`

第一条测试不要问哲学，也不要让它写大项目。问一个能验证工具的任务，

\`\`\` 请检查当前目录，告诉我这里最像主项目入口的 5 个文件，并说明你的判断依据。 \`\`\`

成功标准很简单，

-   它知道当前 provider 和模型。
-   它能正常回复。
-   它能在需要时调用文件或终端工具。
-   你追问第二轮，它没有断掉。

这一步过了，才算 Hermes 有生命体征。

第三步，确认会话能恢复

Agent 工作台跟普通聊天最大的区别之一，是能接着干。

跑完一个简单会话后，退出，再试，

\`\`\` hermes --continue \`\`\`

或者，

\`\`\` hermes -c \`\`\`

能回到刚才的上下文，说明 sessions 基础正常。后面的长期任务、多 Agent、Gateway、Cron，都会用到它。

如果恢复不了，先查，

\`\`\` hermes sessions list \`\`\`

很多人以为是记忆坏了，其实只是切了 profile，或者 session 没保存。

第四步，搞清楚 Hermes 的几类配置文件

Hermes 的配置不难，难的是别把东西放错地方。

～/.hermes/.env 放密钥。API Key、bot token、平台 token 这种东西放这里。不要发给别人，不要贴到公开仓库。

～/.hermes/config.yaml 放普通配置。比如默认模型、terminal backend、gateway、工具开关。

～/.hermes/SOUL.md 放稳定人格和硬规则。比如回答前先验证，危险命令要提醒，不要编造 API，做完任务要说明怎么检查。

～/.hermes/memories/MEMORY.md 放 Agent 的工作笔记。比如这台机器的环境、项目习惯、踩过的坑。

～/.hermes/memories/USER.md 放用户偏好。比如默认中文、结论先行、少写长篇废话。

项目根目录的 AGENTS.md 放项目级规则。比如这个项目怎么测试、不能动哪些目录、发布前要跑什么命令。

这里最容易犯的错，是把所有东西都塞进 MEMORY.md。我以前也这么干过，结果很快就乱了。固定规则应该放 SOUL.md，项目规范放 AGENTS.md，会变化的环境事实和工作笔记才放 MEMORY.md，用户偏好放 USER.md。

新手第一版 SOUL.md 不用写得像公司制度，几行就够，

\`\`\` # Hermes 行为规则

不确定的命令、路径、配置，先查再答。 三步以上任务先列计划。 高危操作先说明风险，不要直接执行。 完成任务后告诉我怎么验证。 临时猜测不要写进长期记忆。 \`\`\`

别一开始写两千字人设。写太满，后面自己都不知道哪条规则在起作用，Hermes 也未必知道。

第五步，Memory 和 Skill 不是一回事

Hermes 有意思的地方，不在于它会“记住我”这句话，而在于它能把记忆和流程分开。

Memory 记事实和偏好。

比如，

\`\`\` 以后整理 AI 工具更新时，保留英文产品名、原始链接和 Markdown 格式。 \`\`\`

这适合进 Memory。

Skill 记流程和 SOP。

比如，

\`\`\` 每次生成 AI 工具日报时，先抓 AI hot，再看 X 热门推文，再按 agent/coding/video/image/infra 分类，最后给出 5 个可写选题。 \`\`\`

这适合沉淀成 Skill。

我会这样区分，

\`\`\` Memory = 它该知道什么 Skill = 某类任务该怎么做 \`\`\`

![[../../../../images/33b3bf13dc5ffc3ff108ee18c81b88ec_MD5.jpg]]

训练 Hermes 时，天天喊“记住我”用处有限。一个任务做顺了，直接让它沉淀成 Skill，效果更稳定，

\`\`\` 我们刚才这个流程以后会反复用。请把它保存成一个 Skill，包含触发条件、执行步骤、注意事项和验证方法。 \`\`\`

写完别急着信，检查一下，

\`\`\` hermes skills list \`\`\`

外部技能也能搜，

\`\`\` hermes skills search github hermes skills inspect <skill-name> hermes skills install <skill-name> \`\`\`

最新版里还有 Curator。这个东西我挺喜欢，它管的是 agent-created skills，重复的合并，过时的归档，质量差的清掉一点。

\`\`\` hermes curator status hermes curator run \`\`\`

它不是来乱删技能的。按当前 help 的说明，bundled 和 hub-installed skills 不会被它自动处理；归档也能恢复。把它当成技能库保洁就行，别当成技能库刽子手。

第六步，接 Gateway，让 Hermes 出现在聊天软件里

CLI 跑通以后，再接 Gateway。

\`\`\` hermes gateway setup hermes gateway run \`\`\`

长期运行可以用，

\`\`\` hermes gateway install hermes gateway start hermes gateway status \`\`\`

如果只是本地试，gateway run 更直观。它在前台跑，报错能直接看到。等一切稳定，再装成后台服务。

接平台时先选一个。

个人用，Telegram 或 Discord 通常最快。团队协作，飞书会更顺手。具体有哪些平台，以当前版本的 hermes gateway setup 菜单为准，不要拿旧教程硬套。

Gateway 成功不看服务有没有启动，要看聊天软件里能不能给 Hermes 发消息，并收到回复。

第一条消息可以很简单，

\`\`\` 请回复一句话，说明你已经通过当前聊天平台接入 Hermes。 \`\`\`

再发第二条，

\`\`\` 请记住，以后我从这个聊天入口让你整理资料时，默认输出 Markdown，并保留原始链接。 \`\`\`

第二天换一个说法让它做同类任务，看它有没有沿用偏好。测到这一步，才算测到了 Gateway + Memory 的组合能力。

![[../../../../images/3d1462418a8beef952eecc52e7713c56_MD5.jpg]]

第七步，Profile 才是多 Agent 的正确起点

很多人说多 Agent，其实说的是多个 profile。

Profile 是独立 Hermes home。每个 profile 都有自己的 config.yaml、.env、SOUL.md、memories、sessions、skills、cron jobs、state database。

创建一个写代码助手，

\`\`\` hermes profile create coder --clone coder setup coder chat \`\`\`

创建一个审查助手，

\`\`\` hermes profile create reviewer --clone reviewer setup reviewer chat \`\`\`

\--clone 的意思是复制当前配置、密钥和 SOUL，但给新 profile 一套新的 sessions 和 memory。它适合创建相似但职责不同的 agent。

如果要完整复制，包括记忆、会话、skills、cron，可以用，

\`\`\` hermes profile create backup --clone-all \`\`\`

这里必须讲一个坑，Profile 不是 sandbox。

Profile 只是状态隔离。它隔离的是配置、记忆、会话、技能，不隔离文件权限。默认 local terminal backend 下，它仍会以当前用户身份访问文件系统。

如果只想让它在某个项目目录工作，要设置 terminal.cwd。如果要限制命令执行环境，就得用 Docker、SSH、Modal、Singularity 这类 backend。

这句话要记住，

\`\`\` Profile 管身份和状态，Sandbox 管权限和边界。 \`\`\`

不要把两者混了。

第八步，Docker 到底解决什么问题

Docker 和 Hermes 有两种关系。

第一种，把 Hermes 本体跑在 Docker 里。适合部署长期在线的服务，也能少污染宿主机环境。

第二种，Hermes 还在宿主机跑，但它执行终端命令时进 Docker 沙箱。适合不想让 Agent 直接碰宿主机的场景。

这两个不是一回事。

把 Hermes 本体跑在 Docker 里，最小形态大概是，

\`\`\` mkdir -p ～/.hermes

docker run -it --rm \\ -v ～/.hermes:/opt/data \\ nousresearch/hermes-agent setup \`\`\`

后台 gateway，

\`\`\` docker run -d \\ --name hermes \\ --restart unless-stopped \\ -v ～/.hermes:/opt/data \\ -p 8642:8642 \\ nousresearch/hermes-agent gateway run \`\`\`

核心是这个挂载，

\`\`\` ～/.hermes:/opt/data \`\`\`

因为 /opt/data 是容器里的 Hermes 状态目录。.env、config、sessions、memories、skills、cron、logs 都在这里。没挂载，容器重建后，很多东西就没了。

只是想让 Hermes 命令执行更安全，可以先用，

\`\`\` hermes config set terminal.backend docker \`\`\`

这时 Hermes 本体还在宿主机，进 Docker 的只是 terminal tool。

第九步，Kanban 是最新版最值得关注的新能力

只做个人聊天，Kanban 不是第一天要开的东西。

但要让多个 Hermes profiles 像一个小团队一样协作，Kanban 就很关键。

最新版里可以直接看，

\`\`\` hermes kanban --help \`\`\`

它的描述很清楚，这是一个 durable SQLite-backed task board。任务可以被原子 claim，可以设置依赖，可以由 named profile 在隔离 workspace 中执行。

最新版继续把这件事往前推，增加了 heartbeat、reclaim、zombie detection、retry budget、incomplete exit auto-block 等能力。翻译成人话就是，以前多 Agent 很像“派出去就听天由命”，现在更像有任务板、有心跳、有失败恢复、有交接记录。

![[../../../../images/dbcb4a786d08dfe6b7fcca7a1a7ddb1a_MD5.jpg]]

入门只需要知道这几个命令，

\`\`\` hermes kanban init hermes kanban boards list hermes kanban create --help hermes kanban list hermes kanban show <task-id> hermes kanban dispatch --help \`\`\`

不要第一天就让它自动接管大项目。更适合的练习是，

\`\`\` 创建一个小任务板，AI 日报生产。 任务 A，收集 10 条 AI hot。 任务 B，筛出 5 个可写选题。 任务 C，给每个选题写标题和读者痛点。 让 researcher profile 做 A，让 editor profile 做 B 和 C。 \`\`\`

这类任务失败成本低，结构清楚，适合测试 Kanban。

第十步，Cron 适合做固定任务，不适合许愿

Hermes 有 cron，

\`\`\` hermes cron list hermes cron add hermes cron status hermes cron run <job> \`\`\`

最新版还加入了 no\_agent cron，某些 watchdog 类任务只跑脚本，不必每次都启动 Agent。

我给 cron 的第一批任务通常很保守。早上出一份 AI 工具简报，晚上备份一个目录，每周审计一次 Skills，或者每天看一个网页有没有更新。这些活儿出错了也好处理。

自动发布文章、自动删文件、自动操作账号，我不建议第一天就交给它。自动化不是越猛越好，能复查、能回滚、能解释，才能长期放着跑。

排错顺序

Hermes 出问题时，最怕凭感觉乱改 .env 和 config.yaml。我一般先跑这一串，

\`\`\` hermes doctor hermes model hermes setup hermes sessions list hermes --continue hermes gateway status hermes logs errors \`\`\`

hermes: command not found 多半是 shell 没重载，先 source ～/.bashrc 或 source ～/.zshrc。模型不回，先查 provider、API Key、模型名、base URL。本地模型奇怪，重点看 endpoint 是否 OpenAI-compatible、context 够不够、模型名和实际加载的是否一致。Gateway 启动了却收不到消息，别急着重装，token、allowlist、pairing、home channel、平台权限逐项看。记忆没生效，先确认是不是同一个 profile，再看 session 是否恢复、MEMORY.md / USER.md 有没有写入。Profile 能访问外部文件也不奇怪，profile 管状态，sandbox 才管边界。Docker 重启后配置丢了，大概率是 ～/.hermes 没挂到 /opt/data。Skills 乱了，先看 hermes skills list 和 hermes skills audit，别上来手删一堆。

一个适合第一周的练习，AI 日报助手

我觉得第一周最适合拿 AI 日报练手。它够真实，又不危险；做坏了最多是一份简报难看，不会把项目删掉。

第一天，在 CLI 里直接丢这个任务，

\`\`\` 请帮我生成一份今天的 AI 工具更新简报。 要求， 1. 每条保留产品名、原始链接、更新时间 2. 按 agent / coding / video / image / infra 分类 3. 输出 Markdown 4. 末尾写一下哪些偏好值得下次继续保留 \`\`\`

它做完以后，不要只看内容，要纠正格式。比如，

\`\`\` 以后这类简报不要写空泛评价。每条只保留事实、链接和一句“值得看的原因”。 请把这个偏好写入你的记忆。 \`\`\`

第二天换一个要求，

\`\`\` 继续昨天的 AI 工具更新简报。 但这次只保留最值得写成公众号文章的 5 条。 每条给出，标题方向、读者痛点、实操交付物 \`\`\`

连续做三天，差不多就能沉淀成 Skill，

\`\`\` 我们已经连续做了三次 AI 工具更新简报。 请把稳定流程保存成一个 Skill，名字叫 ai-daily-brief。 要求包含触发条件、信息来源、筛选规则、输出格式和验证方法。 \`\`\`

这比第一天喊“帮我打造一个 AI 帝国”实在得多。小任务安全、低成本、可复查，Hermes 也能在这个过程中慢慢记住偏好，沉淀流程，之后再接聊天软件、定时任务、Profile、Kanban，心里会有底。

## 收一下

我看重 Hermes，主要是因为“连续性”。很多 AI 工具并不笨，只是太容易断。今天刚教会一个格式，明天换个窗口又忘；今天修过一个坑，后天又绕回来犯一次。

Hermes 把 memory、skills、sessions、gateway、profiles、kanban 放在一起，正是冲着这个断裂去的。

但它也不是免维护机器。模型要选，密钥要管，工具要配，日志要查，权限要审，Skills 要清理，任务边界也要设计。它会成长，可它长成什么样，很大程度上取决于人怎么喂任务、怎么给反馈、允许它保存什么流程。

我的建议仍然很土。先跑通一个会话，再养成一个流程，再组织一支队伍。

第一天做到三件事就够了，

\`\`\` hermes model hermes --tui hermes doctor \`\`\`

能说话，能用工具，能恢复，能排错。

这个地基打好，Hermes 才有资格变成长期 AI 工作台。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/06dafd39_1778387994647?u=https%3A%2F%2Fx-v2.clipfx.app%2F2053054035738108006)