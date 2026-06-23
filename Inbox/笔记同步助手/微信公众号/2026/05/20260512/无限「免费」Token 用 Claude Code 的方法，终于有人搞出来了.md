---
author: JackGan
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzMjYzNjI4OQ==&mid=2247485093&idx=1&sn=f86f448d3a36d03e6e69c1c5292a528a&chksm=c338ecaa35b2ca41d8b92c412304d6bb1d86ec3812caae32ea70f56bf13da6ab34d8cb28241b&mpshare=1&scene=1&srcid=05123BtKuG8EXAwDYEO22xES&sharer_shareinfo=038dd5c503846cc8cfa9359f609666ab&sharer_shareinfo_first=038dd5c503846cc8cfa9359f609666ab#rd
saved: 2026-05-12 16:52:20
tags:
  - 笔记同步助手
id: d3153ec9-3cb9-4ca3-be5f-d735e9b8bc0f
---

公众号名称：2077硅基趣谈

作者名称：JackGan

发布时间：2026-04-26 01:27

说句实话。

Claude Code 这个工具，程序员用过就回不去。Terminal 里直接跑代码、PR review、自动调试，一条命令搞定，不用切来切去。

**唯一的问题：要花钱。**

它背后调的是 Anthropic API，按量收费。额度用完就得充值，少则几十美元，多则上百。

但最近有个项目在 GitHub 上悄悄火了——名字就叫 `free-claude-code`。

作者是 **Alishahryar1**。

功能就一个：**让 Claude Code 完全免费跑，不花一分钱。**

​

---

## 方法一：NVIDIA NIM（最推荐）

**免费额度：每分钟 40 次请求，完全够日常用**

Step 1：去 build.nvidia.com/settings/api-keys\[1\] 注册，拿一个免费的 API Key

Step 2：装好 free-claude-code，改两行配置：

​

```
NVIDIA_NIM_API_KEY="nvapi-xxxxx"
MODEL_OPUS=nvidia_nim/z-ai/glm4.7
ENABLE_THINKING=true
```

Step 3：开代理，跑 Claude Code。

**这就完了。**

Claude Code 以为自己还在跟 Anthropic 说话，其实请求全跑到了 NVIDIA NIM——完全免费。

​

---

## 方法二：OpenRouter（几百个免费模型）

**免费模型：DeepSeek R1、Step-3.5-Flash、GPT-OSS-120B……全免费**

Step 1：去 openrouter.ai/keys\[2\] 注册，拿 Key

Step 2：配置：

​

```
OPENROUTER_API_KEY="sk-or-xxxxx"
MODEL_OPUS=open_router/deepseek/deepseek-r1-0528:free
MODEL_SONNET=open_router/stepfun/step-3.5-flash:free
```

Step 3：开代理，直接用。

OpenRouter 的好处是模型多，随时可以换。觉得 DeepSeek R1 不够用？换一个别的试试，反正都免费。

​

---

## 方法三：LM Studio（完全本地，不联网）

**费用：零。流量：零。隐私：最强。**

不想让代码跑到别人服务器上？

LM Studio 在你自己的电脑上跑模型，配好之后 Claude Code 所有请求全在本地处理。

​

```
MODEL_OPUS=lmstudio/unsloth/MiniMax-M2.5-GGUF
MODEL_SONNET=lmstudio/unsloth/Qwen3.5-35B-A3B-GGUF
```

去 lmstudio.ai\[3\] 下载，安装，打开任意一个 GGUF 模型，Claude Code 直接连本地。

**你的代码，永远在你自己的机器上。**

​

---

## 方法四：混着用（高级玩法）

free-claude-code 支持每个模型单独指定不同提供商。

比如这样配置：

​

```
MODEL_OPUS=nvidia_nim/moonshotai/kimi-k2.5     # Opus 级能力用 NIM
MODEL_SONNET=open_router/deepseek/deepseek-r1:free  # Sonnet 级用 OpenRouter 免费额度
MODEL_HAIKU=lmstudio/unsloth/GLM-4.7-Flash-GGUF   # Haiku 级用本地 LM Studio
```

一个项目里同时用三个不同来源，Opus、Sonnet、Haiku 各自跑不同的模型和提供商。

​

---

## Discord / Telegram 远程控制（彩蛋）

这是我觉得最有意思的功能。

配置好 Discord Bot 之后，你在手机上发消息，Claude Code 在你家台式机上跑，然后把结果推回来。

**在地铁上、在排队、在任何地方——手机发一句"帮我 review 这个 PR"，回家直接看结果。**

支持语音消息——你发一条语音，bot 转成文字，当 prompt 发出去，跑完再推回来。

​

---

## 它是怎么工作的？一句话说清楚

free-claude-code 本质是一个**透明代理**。

Claude Code 以为自己在和 `api.anthropic.com` 说话，实际上请求全部发到了你本地的 `localhost:8082`，这个代理接收、翻译、转发。

**Claude Code 完全不知情，你不用改一行代码。**

它还做了几件聪明的事：

-   • **5 类无效请求直接拦截**，本地返回空答案，不浪费额度
-   • **思考令牌自动转换**，让非 Anthropic 模型也能模拟 Claude 的推理能力
-   • **智能路由**，Opus / Sonnet / Haiku 可以各走各的提供商

---

## 有没有坑？

说清楚，有几点：

**VSCode 扩展偶尔弹窗**让你登录 Anthropic 或买额度。这个不是 bug，点一下 authorize 就能过，extension 不知道自己接的是别家模型。

**免费模型的能力和 Claude Opus 有差距**。复杂多文件重构、长上下文推理这种任务，差距还是明显的。日常工具用没问题，别指望它能完全替代旗舰模型在高难度任务上的表现。

**Discord / Telegram 配置有门槛**，新手可能需要看文档。

​

---

## 十分钟搞定，要试试吗？

![[../../../../images/d9505b01086ab60fa3177206fd4fa21c_MD5.jpg]]

说这么多，不如直接动手。

注册 API Key → 装 free-claude-code → 改两行配置 → 跑起来。

十分钟，最多了。

链接在这里：github.com/alishahryar1/free-claude-code\[4\]

工具越来越便宜，门槛越来越低。

**剩下来的问题只有一个——你想用这个能力，做点什么？**

\#AI \#Claude Code \#免费 \#开源 \#编程

​

#### 引用链接

`[1]` build.nvidia.com/settings/api-keys: _https://build.nvidia.com/settings/api-keys_  
`[2]` openrouter.ai/keys: _https://openrouter.ai/keys_  
`[3]` lmstudio.ai: _https://lmstudio.ai_  
`[4]` github.com/alishahryar1/free-claude-code: _https://github.com/alishahryar1/free-claude-code_

---

![[../../../../images/2b93de02aa2ca5e97d28eaaef4878673_MD5.jpg|cover_image]]

原创 JackGan 2077硅基趣谈

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/12c1f37f_1778575939450?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzMjYzNjI4OQ%3D%3D%26mid%3D2247485093%26idx%3D1%26sn%3Df86f448d3a36d03e6e69c1c5292a528a%26chksm%3Dc338ecaa35b2ca41d8b92c412304d6bb1d86ec3812caae32ea70f56bf13da6ab34d8cb28241b%26mpshare%3D1%26scene%3D1%26srcid%3D05123BtKuG8EXAwDYEO22xES%26sharer_shareinfo%3D038dd5c503846cc8cfa9359f609666ab%26sharer_shareinfo_first%3D038dd5c503846cc8cfa9359f609666ab%23rd&s=obsidian)