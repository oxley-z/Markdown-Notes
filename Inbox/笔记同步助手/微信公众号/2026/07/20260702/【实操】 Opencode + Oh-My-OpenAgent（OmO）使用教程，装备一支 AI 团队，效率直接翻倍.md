---
author: extremeFUN
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg5MzI0OTg4OQ==&mid=2247484717&idx=1&sn=2b19b9f5722a1e78cbc2d4ad973a7b4b&chksm=c155b1c345ec0f0b00da46d907c3162a0f6a58168f94843cbca325e614e084dae5552b02a23e&mpshare=1&scene=1&srcid=0702zsjabyR08a7dJi3C7HMr&sharer_shareinfo=179f4c769777e3221b726299401149a0&sharer_shareinfo_first=179f4c769777e3221b726299401149a0#rd
saved: 2026-07-02 09:10:06
tags:
  - 笔记同步助手
id: 2d6c7d9c-4a23-483f-857a-e5146f54ce2f
---

公众号名称：extremeFUN

作者名称：extremeFUN

发布时间：2026-06-03 18:32

> 装上 Oh-My-OpenAgent 后，才是 OpenCode 的正确使用姿势。 只敲一个词：ultrawork ，激活一整支 AI 军团！

## 一、OpenCode 和 OmO 的关系

用一句话讲清楚：**opencode 是引擎，OmO 是改装套件。**

-   **opencode**：一个开源的终端 AI 编程 agent，最大的特点是支持接入 75+ 模型提供商(Anthropic、OpenAI、Gemini、本地 Ollama ......)。它不绑定任何一家模型，你想用谁用谁。
    
-   **Oh-My-OpenAgent（简称 OmO，原来叫 oh-my-opencode）**：opencode 的增强插件。在 OpenCode 上提供完整功能 —— 11 个智能体、54+ 个生命周期钩子、Team Mode、所有 MCP、所有斜杠命令、IntentGate 模式。
    

对于 opencode 的使用这里不多介绍了，可以参考：[【实操】OpenCode 完整使用教程，自带免费模型，号称开源版 Claude Code !](https://mp.weixin.qq.com/s?__biz=Mzg5MzI0OTg4OQ==&mid=2247484698&idx=1&sn=90510c8f2d74651cab59398d8ababd9a&scene=21#wechat_redirect)

  

## 二、 OmO 安装

OmO 有两个版本，可以根据你用的工具选：

| 你想要的 | 命令 | 适用 |
| --- | --- | --- |
| **Ultimate（OpenCode 版）** | `bunx oh-my-openagent install` | 完整版：11 个 agent、54+ 钩子、5 个内置 MCP、Team Mode，全都有 |
| **Light（Codex CLI 版）** | `npx lazycodex-ai install` | 轻量版：塞进 Codex 插件体系的便携组件 |
| **两个都要** | `bunx oh-my-openagent install --platform=both` | 全都装 |

这里是基于 opencode，所以走 Ultimate。安装很简单，一行命令就行：

```
bunx oh-my-openagent install
```

## 三.、AI 团队介绍

OmO 它不止一个 agent，而是由**11 个 agent** 组成的一支专业团队，可自动调度。先看下它们是怎么工作的：

```
用户请求
   ↓
[IntentGate] —— 先判断你「真正」想干什么
   ↓
[Sisyphus] —— 主编排官，规划并派活
   ↓
   ├─→ Prometheus  战略规划
   ├─→ Atlas       待办编排与执行
   ├─→ Oracle      架构咨询
   ├─→ Librarian   文档/代码搜索
   ├─→ Explore     极速代码库 grep
   └─→ 按类别路由的专业 agent
```

下面重点介绍下这些 Agent ：

| 类型 | Agent | 角色定位 | 职责 |
| --- | --- | --- | --- |
| **核心** | Sisyphus（西西弗斯） | 主编排官 | 系统大脑：规划、拆任务、派活、激进并行，带 32k 扩展思考预算，不到完成绝不半途而废 |
| **核心** | Hephaestus（赫菲斯托斯） | 正统工匠 | GPT 原生自主深度 worker，给目标而非菜谱，端到端干完 |
| **核心** | Oracle（神谕） | 顾问（只读） | 架构决策、代码审查、复杂调试，逻辑推理极强 |
| **核心** | Librarian（图书管理员） | 多仓检索（只读） | 文档查询、OSS 实现示例、基于证据的代码库理解 |
| **核心** | Explore（探索者） | 极速 grep（只读） | 快速代码库探索与上下文检索 |
| **核心** | Multimodal-Looker | 视觉分析（只读） | 解析 PDF、图片、图表，提取信息 |
| **规划** | Prometheus（普罗米修斯） | 战略规划师 | 面试模式：迭代提问，写代码前产出详细方案 |
| **规划** | Metis（墨提斯） | 方案顾问 | 规划前分析：识别隐藏意图、模糊点、AI 易翻车处 |
| **规划** | Momus（摩墨斯） | 方案审查者 | 按清晰度、可验证性、完整度三标准校验方案 |
| **编排** | Atlas（阿特拉斯） | 待办指挥者 | 系统化执行 Prometheus 方案，管理 todo、协调工作 |
| **编排** | Sisyphus-Junior | 类别执行者 | 用类别委派时**真正干活**的 agent，按类别自动选模型 |

## 四、配置代理商模型

在 `～/.config/opencode/oh-my-opencode.json` 文件中,可以为不同的 agent 自定义配置代理商的模型，例如：

```
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/master/assets/oh-my-opencode.schema.json",
  "agents": {
    "sisyphus": { "model": "claude-provider/claude-opus-4-7" },
    "hephaestus": { "model": "gpt-provider/gpt-5.5" },
    "oracle": { "model": "gpt-provider/gpt-5.5" }
  }
}
```

## 五、实战

`重点说明下`: 直接跟 Sisyphus 对话是不会触发 `ultrawork` 模式的， 只有对话内容含 ultrawork 或 ulw 关键词时，才会触发 ultrawork 模式。

下面演示下用`/ulw-loop` 命令来触发：

```
/ulw-loop 设计一个科技公司网站，具有科技感，网页有动态效果。
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8fa8ca0b56829c385f46435b65c9c2c1_MD5.jpg]]

Sisyphus 会根据输入的需求自动规划任务并分配给不同的 Agent 并行执行，直到完成交付。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/5a6d3526ede280369dd2c6c34a994c82_MD5.jpg]]

可以看到已经触发 `ultrawork` 模式成功了。在等 `librarian` 返回。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/905a94ea7602e50155327ec59278c746_MD5.jpg]]

等所有任务完成后，来看看效果。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/926cc3eae816d1e5ec6606f18db7dca8_MD5.jpg]]

而且还适配了移动端

![[Inbox/笔记同步助手/微信公众号/2026/07/images/7226ef736054c6dbc924ccaaae77a828_MD5.jpg]]

## 六、总结

简单来说，OpenCode 负责“让 AI 精准执行”，Oh-My-OpenCode 负责“让 AI 智能调度”。一个让你告别闭源的束缚，一个让你拥有一支 AI 团队。项目地址：https://github.com/code-yeongyu/oh-my-openagent

> 如果觉得内容不错，欢迎点赞、分享、在看 三连，想第一时间收到内容推送，可以关注我，加个星标⭐哦～

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/0044c884_1782954604728?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg5MzI0OTg4OQ%3D%3D%26mid%3D2247484717%26idx%3D1%26sn%3D2b19b9f5722a1e78cbc2d4ad973a7b4b%26chksm%3Dc155b1c345ec0f0b00da46d907c3162a0f6a58168f94843cbca325e614e084dae5552b02a23e%26mpshare%3D1%26scene%3D1%26srcid%3D0702zsjabyR08a7dJi3C7HMr%26sharer_shareinfo%3D179f4c769777e3221b726299401149a0%26sharer_shareinfo_first%3D179f4c769777e3221b726299401149a0%23rd&s=obsidian)