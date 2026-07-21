---
author: v3n0m
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk5MDMwNzU5NQ==&mid=2247484556&idx=1&sn=b2188b7d9dd4c8c0a7bb6ff5277e3ded&chksm=c422bdc8f5842957c702a4bc56675728b6c780ce49097c4de78bf7555cde66d5fdde2fff91bd&mpshare=1&scene=1&srcid=0720AceA7vYQJzESlQBDE2Ui&sharer_shareinfo=930036109ad0f1b5f21b3efece45fbe0&sharer_shareinfo_first=930036109ad0f1b5f21b3efece45fbe0#rd
saved: 2026-07-20 21:37:44
tags:
  - 笔记同步助手
id: 37f3c673-e252-4c05-bcf7-286ad111b5f1
---

公众号名称：v3n0m

作者名称：v3n0m

发布时间：2026-07-17 23:55

![[Inbox/笔记同步助手/微信公众号/2026/07/images/f8738e342e8e408fe42cd3fc95c798ae_MD5.jpg]]

近期不少粉丝在后台留言，纷纷安利商汤SenseNova大模型API，据说有免费的ds4可以用，那还等什么，搞起来。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/b2ad82a4363423a15eba587eb79986ce_MD5.jpg]]

打开官网找到TokenPlan的位置果然是免费的，但是有使用限制。

## 一、三大免费模型核心能力一览

| 模型名称 | 免费额度 | 核心优势 |
| --- | --- | --- |
| sensenova-6.7-flash-lite | 每 5h 1500 次 | 图文一体理解，办公长文档、票据、截图分析首选，最大输出 64K tokens |
| sensenova-u1-fast | 每 5h 1500 次 | 支持 11 种主流画幅，最高 2752×1536 高清图，批量制作数据海报、流程图 |
| deepseek-v4-flash | 每 5h 500 次 | 支持多级思考模式，复杂算法、长文本梳理、智能体开发专用 |

### 二、关键接口基础信息

OpenAI兼容接口（绝大多数开发工具）

```
BaseURL：https://token.sensenova.cn/v1
对话接口：POST/v1/chat/completions
生图接口：POST/v1/images/generations
```

Anthropic兼容接口（Claude系列工具）

```
BaseURL：https://token.sensenova.cn
对话接口：POST/v1/messages
统一鉴权规则所有请求Header携带密钥：Authorization:Bearer$你的API_KEY
```

  

## 三、3 分钟快速接入教程

### Step1 注册开通免费额度

访问官网并且手机号注册，无需企业资质，个人直接使用。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/21e65b581b5ee49c8dd58824838bc49d_MD5.jpg]]

### Step2 创建专属 API Key

控制台左侧「API Keys」新增密钥，仅创建时可见，妥善保存，支持随时注销轮换。

![[Inbox/笔记同步助手/微信公众号/2026/07/images/8ddf38c965c3c3810c46aef53e5ce549_MD5.jpg]]

### Step3 标准 cURL 调用示例（多模态图文对话）

```
curl https://token.sensenova.cn/v1/chat/completions ^-H "Authorization: Bearer 你的key" ^-H "Content-Type: application/json" ^-d "{\"model\":\"deepseek-v4-flash\",\"messages\":[{\"role\":\"system\",\"content\":\"You are a helpful assistant.\"},{\"role\":\"user\",\"content\":\"你好!\"}],\"stream\":false}"
curl https://token.sensenova.cn/v1/chat/completions ^
-H "Authorization: Bearer 你的key" ^
-H "Content-Type: application/json" ^
-d "{\"model\":\"deepseek-v4-flash\",\"messages\":[{\"role\":\"system\",\"content\":\"You are a helpful assistant.\"},{\"role\":\"user\",\"content\":\"你好!\"}],\"stream\":false}"
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/d3d1421a216dfa1ce4094be790cd3944_MD5.jpg]]

返回如上图内容就说明你的key已经生效了。

## 四、配置Hermes agent

老样子我还是用本机的Hermes做演示。

找到config.yaml修改配置。

```
model:  default: deepseek-v4-flash  provider: custom  base_url: https://token.sensenova.cn/v1  api_key: 你的key
model:
default: deepseek-v4-flash
provider: custom
base_url: https://token.sensenova.cn/v1
api_key: 你的key
```

![[Inbox/笔记同步助手/微信公众号/2026/07/images/4d9c7e27c19c5294a9149857b4b7b342_MD5.jpg]]

如上图所示配置成功，当然你也可以选择hermes的命令模式配置，这个官方文档里有我就不写了。

## 五、其他

官方文档里还是挺全的，兼容**OpenAI** 和**Anthropic**协议目前市面上主流的Cursor、OpenCode、TRAE、OpenClaw、Hermes Agen、tClaude Code等都有详细的配置文档，大家可以自行选择适合自己的。

## 最后

目前从使用的角度来说速度还可以，一般任务和测试完全够用，毕竟是免费的还要啥自行车。

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/a6f42159_1784554662418?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk5MDMwNzU5NQ%3D%3D%26mid%3D2247484556%26idx%3D1%26sn%3Db2188b7d9dd4c8c0a7bb6ff5277e3ded%26chksm%3Dc422bdc8f5842957c702a4bc56675728b6c780ce49097c4de78bf7555cde66d5fdde2fff91bd%26mpshare%3D1%26scene%3D1%26srcid%3D0720AceA7vYQJzESlQBDE2Ui%26sharer_shareinfo%3D930036109ad0f1b5f21b3efece45fbe0%26sharer_shareinfo_first%3D930036109ad0f1b5f21b3efece45fbe0%23rd&s=obsidian)