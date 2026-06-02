---
author: 小金
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzIwNDgzMzI3Mg==&mid=2247495102&idx=1&sn=0e9d3035743b6111ce0780b5c15d0851&chksm=96be4b399d4350f68198ea3462af70331d1bb14c56bd3fd7dcdcc18e1c1123b00fdbfcb5001f&mpshare=1&scene=1&srcid=0601vr9bDFWy0rGwjDZZGyZV&sharer_shareinfo=d9dfd3587d83efa8015562e05caeb690&sharer_shareinfo_first=d9dfd3587d83efa8015562e05caeb690#rd
saved: 2026-06-01 23:47:38
tags:
  - 笔记同步助手
id: 097979e1-2492-4ed2-a3c0-96df332a43be
---

公众号名称：小金AI

作者名称：小金

发布时间：2026-06-01 16:39

Claude Code 好用，这个不用多说，大家都知道。

A 社的一些操作该骂就骂，但 Claude Code 还是该用就用。

但真要把它当主力工具来跑，账单也挺有存在感。尤其是那种让 Agent 反复读代码、改代码、跑测试的任务，一晚上下来，心跳很容易跟着 API 用量一起跳。

而且，Claude 对国内用户非常不友好，订阅容易封号，很多时候只能被迫用第三方模型。

这两天小 G 刷到一个开源项目，一下子就被吸引住了：**Free Claude Code**。

![[Inbox/笔记同步助手/微信公众号/20260601/images/db49b3744f370f370e7aa1a12f5639c0_MD5.jpg]]

它不是 Anthropic 官方出的免费版 Claude Code，也不是破解 Claude Code。

更准确地说，**它是一个本地代理，把 Claude Code 发出去的 Anthropic Messages API 请求，转到其他模型服务上。**

先叠个甲。

这篇不是教大家白嫖官方 Claude，也不是说它能 100% 替代 Claude Sonnet / Opus 的效果。

它解决的问题很简单：**让 Claude Code 这个客户端继续用，但模型流量可以走 NVIDIA NIM、OpenRouter、Gemini、DeepSeek、Kimi、Ollama、LM Studio 这些后端。**

![[Inbox/笔记同步助手/微信公众号/20260601/images/e4063df41e39bc6e99f26b790b339555_MD5.jpg]]

你熟悉的 Claude Code 工作流还在，后面的模型可以自己换。

## 它的原理是什么

Free Claude Code 的核心是一个本地 FastAPI 服务。

Claude Code 原本会把请求发给 Anthropic。这个项目在本机起一个代理，提供 `/v1/messages`、`/v1/messages/count_tokens`、`/v1/models`这类 Anthropic 兼容接口，然后再根据你的配置，把请求转给不同 provider。

你可以把它理解成一个中间翻译层：

![[Inbox/笔记同步助手/微信公众号/20260601/images/68c6d343cd8f628da02b9d36ccdf6916_MD5.jpg]]

Claude Code 继续说 Anthropic 的协议，后端模型可以是 OpenAI-compatible，也可以是 Anthropic Messages 风格，还可以是本地 Ollama、llama.cpp、LM Studio。

Claude Code 本身最强的地方，不只是模型，而是整套开发体验：读项目、改文件、调用工具、生成计划、跑命令、接 IDE。

Free Claude Code 想保留这套体验，然后把模型选择权拿回来一点。

README 里目前列了 **17 个 provider backend**：

NVIDIA NIM、OpenRouter、Google AI Studio、DeepSeek、Mistral、Mistral Codestral、OpenCode Zen、OpenCode Go、Wafer、Kimi、Cerebras、Groq、Fireworks AI、Z.ai、LM Studio、llama.cpp、Ollama。

![[Inbox/笔记同步助手/微信公众号/20260601/images/7a40592a92cb0471b52b76829d785816_MD5.jpg]]

这里面有云端 API，也有本地模型。

如果你想省钱，可以接带免费额度的 provider；如果你更在意隐私，可以接本地 Ollama 或 LM Studio；如果你只是想把 Opus、Sonnet、Haiku 这些请求拆开走不同模型，它也支持按模型层级路由。

比如 README 里提到，可以分别设置：

```
MODEL_OPUS=
MODEL_SONNET=
MODEL_HAIKU=
MODEL="nvidia_nim/nvidia/nemotron-3-super-120b-a12b"
```

空着的 tier 会继承 `MODEL`。

这块对重度用户挺实用。不是所有任务都需要最贵模型，改文案、补测试、扫 lint、解释报错，很多时候用便宜模型就够了。

## 安装方式很粗暴

macOS / Linux：

```
curl -fsSL "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.sh?raw=1" | sh
```

Windows PowerShell：

```
irm "https://github.com/Alishahryar1/free-claude-code/blob/main/scripts/install.ps1?raw=1" | iex
```

装完后启动代理：

```
fcc-server
```

服务起来后，终端里会打印本地 Admin UI 地址，默认类似：

```
http://127.0.0.1:8082/admin
```

在这个页面里填 provider key，选模型，点 Validate，再 Apply。

最后用这个命令启动 Claude Code：

```
fcc-claude
```

`fcc-claude`会读取当前代理端口和本地 auth token，帮你设置 Claude Code 需要的环境变量，然后再启动真正的 `claude`命令。

这里有个细节别跳过：安装脚本是直接从 GitHub 拉下来执行，最好先打开 `scripts/install.sh`或 `scripts/install.ps1`看一眼。这类工具会改本机命令和配置，养成先看的习惯没坏处。

## Admin UI 比手改配置舒服

早期这类代理工具很容易变成 `.env`地狱。

Free Claude Code 现在把常用配置放进了本地 Admin UI。README 里明确说，普通用户优先在 `/admin`里改设置，不建议手工编辑托管配置文件。

它支持的 key 也比较多，比如：

```
NVIDIA_NIM_API_KEY
OPENROUTER_API_KEY
GEMINI_API_KEY
DEEPSEEK_API_KEY
KIMI_API_KEY
GROQ_API_KEY
OLLAMA_BASE_URL
LM_STUDIO_BASE_URL
```

例如这是 DeepSeek API Key 创建地址：https://platform.deepseek.com/ 。

![[Inbox/笔记同步助手/微信公众号/20260601/images/db7109dcf257cc5d26afa62f2c91dac2_MD5.jpg]]

DeepSeek 创建 API Key

本地模型不需要 API key，但你要先把对应服务跑起来。

比如 Ollama：

```
ollama pull llama3.1
ollama serve
```

然后模型名按项目约定写成：

```
ollama/llama3.1
```

LM Studio 默认地址是：

```
http://localhost:1234/v1
```

llama.cpp 默认地址是：

```
http://localhost:8080/v1
```

这类本地方案听起来很香，但小 G 也得说一句：

**Claude Code 对上下文、工具调用、代码推理能力要求不低。真不建议本地部署模型，一般的配置很难达到满血状态，比如直接用现成的 API。**

## 不止终端，还能接 VS Code 和 JetBrains

![[Inbox/笔记同步助手/微信公众号/20260601/images/7577d65bafdc99b5bca8c18326631e79_MD5.jpg]]

这个项目不只服务终端。

README 里写了 VS Code Extension 的配置方式，核心就是给 Claude Code 扩展加环境变量：

```
"claudeCode.environmentVariables": [
  { "name": "ANTHROPIC_BASE_URL", "value": "http://localhost:8082" },
  { "name": "ANTHROPIC_AUTH_TOKEN", "value": "freecc" },
  { "name": "CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY", "value": "1" },
  { "name": "CLAUDE_CODE_AUTO_COMPACT_WINDOW", "value": "190000" }
]
```

JetBrains 也能配，不过要改 Claude ACP 的安装配置文件。

这个功能对 IDE 用户很关键。很多人日常开发已经在 VS Code / JetBrains 里了，不想为了 Agent 单独切终端。能把同一个代理接进 IDE，使用成本会低很多。

项目还支持 Claude Code 的 `/model`选择器，不过 README 里也写了前提：需要开启 Gateway model discovery。

![[Inbox/笔记同步助手/微信公众号/20260601/images/70ca2954ba00d2d86900205cfe4ee693_MD5.jpg]]

![[Inbox/笔记同步助手/微信公众号/20260601/images/abd58ea23ad4c7d94329629d10110e7a_MD5.jpg]]

## 还能接 Discord、Telegram 和语音

![[Inbox/笔记同步助手/微信公众号/20260601/images/d8c3438757088fda9e612aed2a779b55_MD5.jpg]]

Free Claude Code 另一个比较野的功能，是把 Claude Code 会话接到 Discord 或 Telegram。

![[Inbox/笔记同步助手/微信公众号/20260601/images/03786323058dbfe5122de71bde342d11_MD5.jpg]]

你可以配置 bot token、允许的频道或用户 ID，再限制 `ALLOWED_DIR`，让远程消息触发 Claude Code 会话。

它还支持语音笔记转文字。后端可以走本地 Whisper，也可以走 NVIDIA NIM 的语音转写。

比如在外面突然想到一个修复思路，直接给 Telegram bot 发一句话，让它在指定目录里开一个任务。回到电脑前，再看 diff 和测试结果。

不过这块也更敏感。

远程 bot、代码执行、允许目录、API key，全都放在一起，权限一定要收住。尤其是 `ALLOWED_DIR`这种配置，别图省事直接给整个 home 目录。能给项目目录，就只给项目目录。

## 适合谁用

小 G 觉得它最适合三类人。

第一类，是 Claude Code 重度用户。

你已经习惯了 Claude Code 的交互，但又想把一部分低风险任务切到便宜模型或免费额度上。比如解释代码、补单测、生成 README、跑小修小补。

第二类，是喜欢折腾多模型路由的人。

同一个 Claude Code 前端，后面接不同 provider。Opus 类请求走一个模型，Sonnet 类请求走另一个模型，普通 fallback 再走一个更便宜的模型。这样可以按任务成本做拆分。

第三类，是想在本地模型上试 Agent 工作流的人。

Ollama、LM Studio、llama.cpp 都支持。虽然效果要看模型和上下文长度，但至少给了一个低成本试验入口。

不太适合谁？

如果你只想要官方 Claude Code 原汁原味的稳定体验，那就没必要折腾。代理层越多，变量越多。provider 的限流、模型兼容、工具调用格式、流式输出，都可能成为问题。

另外，仓库虽然热度很高，协议也是 MIT，但目前 GitHub 没有发布 release，也没有 tag。`pyproject.toml`里版本写的是 `2.0.0`，要求 Python `>=3.14.0`。这意味着你最好把它当成一个更新很快的开发者工具，而不是一个已经完全定型的企业级产品。

## 最后

Free Claude Code 真正戳人的地方，不是名字里的 free。

而是它抓住了一个很实际的需求：大家喜欢 Claude Code 的开发体验，但不想所有任务都被绑定在同一个模型和同一套计费上。

它把 Claude Code 变成一个更开放的前端。

后面你接免费额度、接便宜模型、接本地模型，都可以自己选。

当然，别把它想成魔法。模型能力不够，Agent 该犯错还是会犯错；provider 不稳定，任务该中断还是会中断；本地模型上下文不够，读大项目照样吃力。

但如果你本来就在用 Claude Code，又想试试多模型路由、降低一部分成本，或者把本地模型塞进 Agent 工作流里，这个项目值得翻一翻。

GitHub 项目地址：

**https://github.com/Alishahryar1/free-claude-code**

> ![[Inbox/笔记同步助手/微信公众号/20260601/images/a8def14eda1ec957c2552de209a126f7_MD5.jpg]]

> **⭐️推荐阅读**:
> 
> -   [《SpringAI 智能面试平台》（2.0 版本已开源）](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247552320&idx=1&sn=a7e4e5a8d957446e6bb032d78b2fa5fb&scene=21#wechat_redirect)(Star 数量 **2.1k+**)
>     
> -   [AI 应用开发面试指南：大模型、Agent、RAG、MCP、Prompt 工程](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg2OTA0Njk0OA==&action=getalbum&album_id=4412413577266053125&scene=126&sessionid=1777281752800#wechat_redirect)(累计阅读接近 **50w+**)
>     
> -   [AI 编程实战指南：Claude Code、Cursor、Codex、Trae 使用技巧与面试题](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg2OTA0Njk0OA==&action=getalbum&album_id=3845984209651990529&scene=126&sessionid=1779072612648#wechat_redirect)(累计阅读接近 **70w+**)
>     

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/917f29fb_1780328856695?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzIwNDgzMzI3Mg%3D%3D%26mid%3D2247495102%26idx%3D1%26sn%3D0e9d3035743b6111ce0780b5c15d0851%26chksm%3D96be4b399d4350f68198ea3462af70331d1bb14c56bd3fd7dcdcc18e1c1123b00fdbfcb5001f%26mpshare%3D1%26scene%3D1%26srcid%3D0601vr9bDFWy0rGwjDZZGyZV%26sharer_shareinfo%3Dd9dfd3587d83efa8015562e05caeb690%26sharer_shareinfo_first%3Dd9dfd3587d83efa8015562e05caeb690%23rd&s=obsidian)