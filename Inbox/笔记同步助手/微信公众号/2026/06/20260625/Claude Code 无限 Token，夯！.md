---
author: Guide
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247555205&idx=1&sn=6bec159553ca29b7211047c6d83c2eeb&chksm=cfb78cf981c758cd59dc2f65e39ce04286cb42fb46481a3a7b3d7eb0dd39cb3ace1713de72f0&mpshare=1&scene=1&srcid=0625aXnQ220WwiUSUvB8cFI3&sharer_shareinfo=e54cbe068bb8c36aae65b96245e6e718&sharer_shareinfo_first=e54cbe068bb8c36aae65b96245e6e718#rd
saved: 2026-06-25 15:45:35
tags:
  - 笔记同步助手
id: 1388ee14-6485-4cd1-a1f0-9f8e1d581240
---

公众号名称：JavaGuide

作者名称：Guide

发布时间：2026-06-21 14:01

大家好，我是小 G。

前段时间简单分享过 Agnes AI 的免费模型，后台和群里问的人还挺多。

这篇文章会集中回答一下常见的问题。并且，我把接入、4K 图片、1M 上下文，以及 GitHub 上已有的 Skill 都整理了一遍。希望对你帮助。

Agnes 目前开放了文本、图片、视频 API。文本模型是 `Agnes-2.0-Flash`，图片模型是 `Agnes-Image-2.1-Flash`，视频模型是 `Agnes-Video-2.0`。

最近的新亮点有两个：**一个是 `Agnes-Image-2.1-Flash` 的 `4K` 图片生成，另一个是 `Agnes-2.0-Flash` 的 `1M Token` 上下文。**

TTS 语音能力也在灰度中，后面如果稳定开放，文本、图片、视频、语音这条内容链路会更完整。

先看最新的调用数据，有点恐怖，也证明免费不是噱头！

![[Inbox/笔记同步助手/微信公众号/2026/06/images/b3d53a73b818f58d5a9281350f506a04_MD5.jpg]]

## 最新一周调用量 3.12T Token

Agnes 全模态模型的周调用量已经到 `3.12T Token`，其中：

-   文本模型 `Agnes-2.0-Flash` 贡献约 `1.9T Token`；
    
-   图片和视频模型 `Agnes-Image-2.1-Flash`、`Agnes-Video-2.0` 合计贡献约 `1.2T Token`。
    

如果只有文本模型免费，开发者可能主要拿它做问答、跑 Agent。图片和视频这一侧也占了不小的量，说明很多人真的在批量试素材，而不是生成一两张图就结束。

这也解释了为什么高峰期慢一点、排队一会儿，都不奇怪。

免费模型进入真实使用后，真正被放大的其实是多轮调用。单次请求慢几秒还好，连续十几轮就会有感觉。

比如 Claude Code 改一个功能，背后可能是读文件、拆任务、写代码、跑命令、看报错、继续修。图片生成也是一样，一张封面经常要先跑 4 到 8 个方向，再挑一个方向重跑高清版。

免费降低的是多试几次的心理成本。

## 先把 Claude Code 接进来

接入链路很简单：

```
Claude Code -> CC Switch -> Agnes API
```

准备好 **Agnes API Key、CC Switch、Claude Code** 就行。

这几个参数后面会用到：

-   API 地址：[https://apihub.agnes-ai.com/v1](https://apihub.agnes-ai.com/v1)
    
-   文本模型：`agnes-2.0-flash`
    
-   图片模型：`agnes-image-2.1-flash`
    
-   视频模型：`agnes-video-v2.0`
    

## 第一步：获取 Agnes API Key

访问 Agnes API Platform：https://platform.agnes-ai.com/，注册登录后进入 API Key 页面，创建并复制一个新的 Key。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/2187192c6a361d2d77b9be1a2b6288c4_MD5.jpg]]

## 第二步：安装 CC Switch

CC Switch 地址：https://github.com/farion1231/cc-switch/releases。

安装完成后打开 CC Switch，顶部选择 `claude-cli`。如果你用的是 Claude Desktop，也可以切到 `Claude(ClaudeCode)-desktop` 标签页，思路一样，都是把 Agnes 配成一个自定义供应商。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/00317b8e3fc59d2c0b4e02f2a6cf5830_MD5.jpg]]

## 第三步：添加 Agnes 供应商并拉取模型

回到 `claude-cli` 页面，点击右上角加号，新增供应商。供应商类型选 `claude`，配置方式选自定义配置，主要填 4 项：

-   API Key：粘贴刚才创建的 Key；
    
-   请求地址：[https://apihub.agnes-ai.com/v1](https://apihub.agnes-ai.com/v1)；
    
-   API 格式：`openai chat completions`；
    
-   模型：`agnes-2.0-flash`。
    

填完以后点一次“获取模型列表”。能拉到 `agnes-2.0-flash`，说明 Key、地址和模型名基本没问题，再按 CC Switch 提示做模型映射。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/379be73b7fc235eb5aa42fcf45a7f47e_MD5.jpg]]

## 第四步：加兼容参数

Claude Code 请求里可能会带一些第三方模型暂时不认识的字段，比如 `thinking`、`context_management`。为了避免请求直接失败，CC Switch 里建议加上这段：

```
{
  "allowed_openai_params": [
    "thinking",
    "context_management"
  ],
  "litellm_settings": {
    "drop_params": true
  }
}
```

![[Inbox/笔记同步助手/微信公众号/2026/06/images/62dc9eeda57925f8d95872463ac193dd_MD5.jpg]]

## 第五步：开启路由并验证

供应商添加完，回到 CC Switch 左上角设置页，进入路由配置，选择本地路由，找到 Claude 相关路由并启用。

开启后，Claude Code 的请求会先走 CC Switch，再由 CC Switch 转到 Agnes API。如果这一步忘了开，前面配置都对，请求也不会走到这条路由。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/0591498f2c1b0736b61a2c480b47d73d_MD5.jpg]]

最后打开 Claude Code。虽然这里显示是 Opus 4.8，但背后实际是 Agnes-2.0-Flash 。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/aa718903f677bb1f18114aec0a740d90_MD5.jpg]]

问一个轻任务验证链路：

```
请用 5 句话解释 Java 中 HashMap 扩容为什么可能影响性能。
```

能正常回复，说明基础链路通了。

如果卡住，先别把配置来回改。按这个顺序看一遍就行：

-   看 CC Switch 日志里有没有请求进来；
    
-   确认 API Key、请求地址、模型名有没有填错；
    
-   高峰期也可能只是请求慢，过几分钟再试。
    

## 4K 图片生成

最直观的升级是图片。

`Agnes-Image-2.1-Flash` 现在支持 `1K`、`2K`、`3K`、`4K`，最高可以生成 `4096x4096` 的 1:1 图片，也支持 `1:1`、`3:4`、`4:3`、`16:9`、`9:16`、`2:3`、`3:2`、`21:9` 等常见比例。

用法也没复杂到哪里去。

以前 1K 的请求大概是这样：

```
{
  "model": "agnes-image-2.1-flash",
  "prompt": "一张 AI Agent 产品落地页首屏图，干净明亮，不要文字，不要水印",
  "size": "1K",
  "ratio": "16:9"
}
```

现在要试 4K，主要就是把 `size` 改成 `4K`：

```
{
  "model": "agnes-image-2.1-flash",
  "prompt": "一张 AI Agent 产品落地页首屏图，干净明亮，不要文字，不要水印",
  "size": "4K",
  "ratio": "16:9"
}
```

返回格式仍然可以走 `url` 或 `b64_json`，这个看你自己的工作流怎么接。

我建议 4K 别只拿来生成“好看大图”。真正能看出差异的，是那些放大后容易露馅的场景。

### 案例一：AI Agent 产品首屏图

产品首屏图很适合测 4K。

它不像纯风景图，只要氛围对就行。画面里通常会有屏幕、设备、人物手部、桌面物品和浅层 UI，一放大就能看出细节是不是糊。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/1bb23520271967f4896a104bf0c74250_MD5.jpg]]

提示词：

```
16:9 横版 4K 产品落地页首屏图。
主题是 AI Agent 正在帮助开发者完成一个代码任务。
画面中央是一台轻薄笔记本电脑，屏幕上是现代代码编辑器、终端窗口和任务进度面板。
桌面干净，有少量便签、机械键盘和咖啡杯，旁边露出开发者正在操作触控板的手。
背景是明亮办公室，浅景深，整体真实摄影风格，干净、有专业感，适合 SaaS 产品官网首屏。
不要文字，不要品牌 Logo，不要水印，不要夸张霓虹，不要复杂 UI 小字。
```

### 案例二：电商产品主图

电商主图很适合测 4K。

因为它会同时考验材质、边缘、反光、背景和主体一致性。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/c99440e442e4825a62ebd62a0f2909ce_MD5.jpg]]

提示词：

```
1:1 4K 电商产品主图。
主体是一只极简风格的智能保温杯，杯身为哑光白色金属材质，顶部有细窄黑色触控屏。
杯子放在浅灰色石材台面上，背景是干净的厨房空间，轻微虚化。
光线从左上方进入，杯身有真实反光和柔和阴影。
画面适合电商详情页首图，不要文字，不要品牌 Logo，不要水印。
```

### 案例三：城市夜景和建筑细节

城市夜景很适合测 4K，因为远处楼体、地面反光、车灯和天空层次都容易露出问题。这里可以把测试点往“远处细节”上压。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/a10c391c36e1bc9cb319a27f558e94e8_MD5.jpg]]

提示词：

```
21:9 超宽 4K 城市夜景。
雨刚停，地面有细腻反光，远处是玻璃幕墙高楼和轻轨轨道。
街道不要太拥挤，重点放在湿润地面、霓虹反射、楼体窗格和远近层次。
画面整体偏冷色，但保留少量暖色路灯。
真实摄影风格，宽画幅电影构图，高细节，不要文字，不要水印。
```

### 案例四：人物半身像

人物图是高风险场景。

脸、手、衣服纹理、头发边缘，任何一个点崩了都很明显。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/ac90faa5622fc13a905234305025fd05_MD5.jpg]]

提示词：

```
3:4 4K 真实摄影风格成年女性半身人像。
人物站在清晨窗边，穿简洁白衬衫，黑色长发自然垂落，侧身看向镜头。
光线从窗户左侧进入，脸部有柔和明暗过渡，背景是浅色室内空间，轻微虚化。
皮肤质感自然，不要夸张磨皮，不要塑料感，不要文字，不要水印。
```

### 案例五：图标和 App 视觉

图标看起来简单，其实很容易测出问题。

小尺寸下要清楚，大尺寸下边缘不能糊，层次还不能太乱。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/33f632f31a35114bf7d94dd4776d20be_MD5.jpg]]

```
1:1 4K 移动端 App 图标。
主题是 AI 笔记应用。
主体是一本文档感更强的打开笔记本，右上角有一个很小的星光符号。
圆角方形底，清爽蓝色和白色为主，轻微玻璃质感。
图形边缘清晰，层次不要太多，适合移动端桌面图标。
不要文字，不要水印，不要复杂纹理。
```

## 1M 上下文

图片升级最直观，但对开发者来说，`1M Token` 上下文也值得单独看一眼。

Agnes-2.0-Flash 支持 1M 上下文后，开发者不用改接口，也不用改模型名。只要请求里的 `messages` 总内容量在 1M Token 范围内，就可以按原方式调用。

它解决的主要是长材料处理问题。以前处理长文档、代码库、会议记录，经常要先切片，再检索，再把 topK 片段塞进上下文。这个流程能用，但切片边界可能丢信息，检索没召回到的内容模型也看不到。1M 上下文至少让一些材料不用被迫拆得那么碎。

![[Inbox/笔记同步助手/微信公众号/2026/06/images/4db87ce68417d23ac33105cd8f8fea34_MD5.jpg]]

上下文窗口（Context Window）= LLM 的工作记忆

窗口大了，也不能什么都往里塞。上下文更像模型的工作记忆，资料放得下只是第一步，后面还要看模型能不能稳定找到关键内容。

1M 上下文更适合这几类任务：

-   长技术手册、合同、论文、会议记录，要求做跨章节问答或风险点检查；
    
-   中等规模代码库，要求理解模块关系、调用链、配置流向和潜在改动影响；
    
-   多份关联文档，要求找出前后矛盾、重复结论或被忽略的约束。
    

我会优先拿代码库任务测。

比如给 Claude Code 一个真实项目，然后让它回答：

```
请帮我看看这个项目的多模型切换模块是如何实现的。

要求：
1. 先列出你读取到的关键文件；
2. 说明请求从入口到模型调用的完整链路；
3. 找出当前实现里最容易出问题的 3 个点；
4. 不要修改代码，只输出分析结果。
```

这比解释一个孤立函数更有参考价值。

真正的项目理解要看文件之间的关系。只给一个函数，模型很容易答得像；给一整个模块，它得知道配置从哪里进来、路由在哪里发生、异常在哪里处理、日志在哪里打。

但我不会让它一上来吞完整仓库。更稳的方式是先看目录结构、文件名和搜索结果，定位目标后再逐步读取文件内容。1M 上下文可以让工作台变大，但资料怎么摆、重点怎么标、证据怎么回查，还是要靠上下文工程来兜住。

1M 上下文适合降低长材料处理的门槛。代码审查、合同风险点、线上故障复盘这类任务，仍然要让模型给出关键文件、证据位置和风险清单，再用人工或脚本核对。

## GitHub 上已经有人把它封进工具链了

我还专门看了一圈 GitHub。

如果一个免费 API 只有官方文档，那还只是供应商自己在讲故事。真正进入开发者圈子，一般会出现两类东西：一类是别人帮你封好的 Skill 和脚本，另一类是第三方工具开始要求适配。

Agnes 现在已经有一些苗头了。

比如这个 Skill：

## https://github.com/Yacey/agnes-ai-generation-skill

![[Inbox/笔记同步助手/微信公众号/2026/06/images/0e1217adb42ee2f2645fa085059c0113_MD5.jpg]]

它把 Agnes 的文本、图片、视频 API 封成 Agent Skill，支持 Codex、Claude Code、OpenClaw、Cursor、Windsurf 等工具。里面提到的能力包括文生图、图生图、文生视频、图生视频、多图视频、关键帧动画，也会把中文图片/视频提示词先翻成英文，减少视频生成不稳定的问题。

还有这个：

**https://github.com/kangarooking/agnes-free-model-skills**

![[Inbox/笔记同步助手/微信公众号/2026/06/images/e32f534fde0ce9f293ff5a2873933f1a_MD5.jpg]]

它把能力拆成 3 个本地 Skill：

-   `agnes-free-text`：调用 `Agnes-2.0-Flash`，支持 Chat Completions、流式输出和工具调用实验；
    
-   `agnes-free-image`：调用 `Agnes Image 2.1 Flash`，支持文生图、图生图和下载返回图片；
    
-   `agnes-free-video`：调用 `Agnes-Video-V2.0`，创建异步视频任务、轮询状态并下载结果。
    

如果你的场景偏视觉工作流，ComfyUI 这边也有人做了节点：

## https://github.com/16nic/comfyui-agnes-ai

![[Inbox/笔记同步助手/微信公众号/2026/06/images/ad3ac3e3224de911079e8703d8674dc2_MD5.jpg]]

这个项目更适合设计和视频用户。它提供了 LLM Chat、图像反推提示词、文生图、图生图、文生视频、图生视频等节点。图片节点已经支持 `1K / 2K / 4K` 画质选择，比例也覆盖了常见横竖屏场景。

我比较在意它 README 里写的错误处理：`502 / 503 / 504 / 524` 会自动重试，连接超时会重试，遇到服务端 OOM 会提示降参数。

这类细节不花哨，但对免费 API 很重要。

免费模型高峰期更容易遇到排队和波动。你如果只是网页里点一次，失败了手动重试就行；一旦接进自动化工作流，就必须考虑重试、超时、降级和错误提示。

另外，Opencode 也有人提了适配 Agnes 的需求：

## https://github.com/anomalyco/opencode/issues/32543

![[Inbox/笔记同步助手/微信公众号/2026/06/images/ea6d1f0b59272c1062c29ad23d50b4fb_MD5.jpg]]

这类 issue 本身不代表官方已经支持，但至少说明有用户想把 Agnes 放进日常编码工具里。

**Agnes 的免费 API 正在从“模型体验”往“工作流插件”走。**

## 视频和 TTS

`Agnes-Video-2.0` 支持 720P / 1080P 输出，也支持首帧生视频、首尾帧生视频、多帧生视频、多镜头内容生成、景别切换、第一视角运镜和光影氛围塑造。对创作者来说，它更适合做短视频草稿、广告素材、剧情分镜、产品展示视频和自动化视频工作流里的片段。

这里用一张《我的小柯基》这款游戏中的图片，很治愈，做了图生视频测试：

![[Inbox/笔记同步助手/微信公众号/2026/06/images/950f90b2a659602a3a48274e7f6fe0dd_MD5.jpg]]

> Agnes 图生视频示例原图

  

第一次是生成失败了，因为图片地址的问题。然后，直接贴上图片地址就没问题了：

![[Inbox/笔记同步助手/微信公众号/2026/06/images/4543031073b33a8d7d23a7290ed537a9_MD5.jpg]]

这类图生视频比纯文生视频更容易判断效果。原图主体已经固定，重点看模型能不能保持狗狗的脸型、毛色和画面风格，同时补出自然的镜头运动、光影变化和环境氛围。

效果如下，还是很不错的：

![[Inbox/笔记同步助手/微信公众号/2026/06/images/387599db044f8ca3835940950f0e7ac4_MD5.jpg]]

> 📹 此处为视频内容（vid: wxv\_4570498302852726785）（上图为封面），未能直接提取，请前往原文查看：[在公众号原文中观看](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247555205&idx=1&sn=6bec159553ca29b7211047c6d83c2eeb&chksm=cfb78cf981c758cd59dc2f65e39ce04286cb42fb46481a3a7b3d7eb0dd39cb3ace1713de72f0&mpshare=1&scene=1&srcid=0625aXnQ220WwiUSUvB8cFI3&sharer_shareinfo=e54cbe068bb8c36aae65b96245e6e718&sharer_shareinfo_first=e54cbe068bb8c36aae65b96245e6e718#rd)

还可以再测一个文生视频场景，重点看多镜头衔接和产品展示感：

```
生成一个 15 秒产品演示视频。  
主题：AI Agent 自动完成一个开发任务。  
0-3 秒：用户输入需求；  
3-6 秒：Agent 读取项目文件和拆分任务；  
6-10 秒：Agent 编写代码、运行测试；  
10-15 秒：任务完成，生成报告。  
风格：现代开发工具界面，节奏干净利落，有轻微科技感，不要夸张特效。
```

![[Inbox/笔记同步助手/微信公众号/2026/06/images/2e1dc351519e1f293ab12df6785159a5_MD5.jpg]]

> 📹 此处为视频内容（vid: wxv\_4557733928098430979）（上图为封面），未能直接提取，请前往原文查看：[在公众号原文中观看](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247555205&idx=1&sn=6bec159553ca29b7211047c6d83c2eeb&chksm=cfb78cf981c758cd59dc2f65e39ce04286cb42fb46481a3a7b3d7eb0dd39cb3ace1713de72f0&mpshare=1&scene=1&srcid=0625aXnQ220WwiUSUvB8cFI3&sharer_shareinfo=e54cbe068bb8c36aae65b96245e6e718&sharer_shareinfo_first=e54cbe068bb8c36aae65b96245e6e718#rd)

TTS 可以放到后续内容链路里看。文本模型写脚本和分镜，图片模型生成关键视觉，视频模型生成片段，TTS 负责旁白或角色台词。

它适合短视频配音、有声内容、AI 助手语音回复、课程/播客旁白、广告素材配音、多语言内容本地化这些场景。

语音能力最终还得看音色自然度、长文本稳定性、停顿控制和批量任务表现，等灰度稳定后再接进完整流程更合适。

## 怎么试更省时间

如果只是想快速判断 Agnes 适不适合自己，不用把所有能力都测一遍。

先从最容易验证的地方开始：把 `Agnes-2.0-Flash` 接进 Claude Code，跑一个轻量代码任务。比如让它写一个 Markdown 图片链接检查工具，看看它会不会扫文件、处理相对路径、保留行号，以及是否遵守“不修改现有 Markdown 文件”这类约束。

接着测 `4K` 图片和 `1M` 上下文。图片不要写“生成一张震撼的科技图”这种空泛提示词，直接选产品首屏图、电商主图、人物半身像、城市建筑、App 图标这类明确场景，放大看边缘、材质、手部、文字和背景细节。长上下文这边，可以把 README、配置文件、关键代码和项目文档交给模型，让它只做分析，不改代码，先看它能不能把模块链路说对。

视频和 TTS 放后面更合适。视频可以先做图生视频、剧情分镜或素材草稿；TTS 等灰度能力稳定后，再接到旁白、课程摘要、广告配音这类场景里。

如果经常生成内容素材，可以把这些步骤封成 Skill：图片生成时固定补上比例、主体、风格和不要项；视频生成时默认拆镜头；异步任务要处理 `task_id`、轮询和重试。后面就能少写临时提示词，让 Agent 按固定流程调用 Agnes。

## 免费 API 适合什么人

Agnes 现在更适合 3 类人。

第一类是开发者。

尤其是经常用 Claude Code、Codex、OpenClaw、Opencode 这类工具的人。Agent 场景太吃 Token，免费 API 能让你更愿意跑项目分析、代码解释、测试生成和低风险改动。

第二类是内容创作者。

4K 图片免费后，封面、海报、商品图、视频关键帧都可以多试几版。以前你可能会纠结“这一版要不要重跑”，现在可以先把方向跑出来，再挑一张高清重做。

第三类是做 AI 应用原型的人。

文本、图片、视频都能走同一套 API，意味着你可以先把产品流程打通。比如一个自动生成短视频脚本、封面和演示片段的小工具，不需要一开始就接很多供应商。

免费不代表不用做工程兜底。高峰期可能慢，视频任务本来就耗时，长上下文请求也可能更重。接进生产流程时，超时、重试、降级、日志、人工复核都要留好。

图片和视频更不用说，生成结果一定要人工挑。尤其是人物、产品、文字、品牌素材这类场景，不能只看第一眼。

## 写在最后

这轮看下来，Agnes 更适合先放在两类场景里试。

一类是开发者工具链，比如 Claude Code 里的项目分析、代码解释、测试思路生成。另一类是内容素材，比如 4K 首屏图、商品图、视频关键帧、图生视频草稿。

免费 API 的价值，会在多跑几轮的时候体现出来。

比如多扫几个文件，多生成几版图，多试两个视频镜头，多做一次失败重试。这些动作单独看都不大，但对日常使用很关键。

当然，接进真实项目还是要留兜底：代码要跑测试，图片和视频要人工挑，长上下文结论要能回到证据位置。把这些前提想清楚，Agnes 至少已经值得接进自己的工具箱里跑一轮。

Agnes AI 文档地址：**https://agnes-ai.com/doc/**

Agnes API Platform： **https://platform.agnes-ai.com/**

![[Inbox/笔记同步助手/微信公众号/2026/06/images/a8def14eda1ec957c2552de209a126f7_MD5.jpg]]

**⭐️推荐阅读**:

> -   [AIGuide：AI 应用开发、AI 编程实战与面试指南](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247555122&idx=1&sn=96278bed8e2b414434398b56785ea2bd&scene=21#wechat_redirect)（对标 JavaGuide，完全开源免费）
>     
> -   [《SpringAI 智能面试平台》（2.0 版本已开源）](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247552320&idx=1&sn=a7e4e5a8d957446e6bb032d78b2fa5fb&scene=21#wechat_redirect)(Star 数量 **2.1k+**)
>     
> -   [AI 应用开发面试指南：大模型、Agent、RAG、MCP、Prompt 工程](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg2OTA0Njk0OA==&action=getalbum&album_id=4412413577266053125&scene=126&sessionid=1777281752800#wechat_redirect)(累计阅读接近 **51w+**)
>     
> -   [AI 编程实战指南：Claude Code、Cursor、Codex、Trae 使用技巧与面试题](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=Mzg2OTA0Njk0OA==&action=getalbum&album_id=3845984209651990529&scene=126&sessionid=1779072612648#wechat_redirect)(累计阅读接近 **85w+**)
>     

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/d388ead8_1782373530239?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg2OTA0Njk0OA%3D%3D%26mid%3D2247555205%26idx%3D1%26sn%3D6bec159553ca29b7211047c6d83c2eeb%26chksm%3Dcfb78cf981c758cd59dc2f65e39ce04286cb42fb46481a3a7b3d7eb0dd39cb3ace1713de72f0%26mpshare%3D1%26scene%3D1%26srcid%3D0625aXnQ220WwiUSUvB8cFI3%26sharer_shareinfo%3De54cbe068bb8c36aae65b96245e6e718%26sharer_shareinfo_first%3De54cbe068bb8c36aae65b96245e6e718%23rd&s=obsidian)