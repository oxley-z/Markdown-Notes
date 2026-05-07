---
author: Brook Li
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI1ODc5MDY2OA==&mid=2247484585&idx=1&sn=657a8f1c1d5369b7620c40e52d3bf077&chksm=eb88f123214665d7bba9239780d052da6508fd82e3ae78cf277f8cadfc7d3b61a44be692a758&mpshare=1&scene=1&srcid=0507hgYtFrLpOmFgOCJJvtsc&sharer_shareinfo=0ad28a2526e0f76ffa262b4dfe431804&sharer_shareinfo_first=0ad28a2526e0f76ffa262b4dfe431804#rd
saved: 2026-05-07 12:01:05
tags:
  - 笔记同步助手
id: 111baab9-fa82-451f-9845-b3c5c5c95228
---

公众号名称：Brook的知识分享平台

作者名称：Brook Li

发布时间：2026-04-10 23:25

近期在网上看到了Karpathy提出的LLM Wiki知识库构建范式，这是一个快速构建小型知识库的方法。与传统 RAG 不同，该方法主张通过 LLM 将原始素材“编译”为结构化的 Markdown 文档。作者实测下来效果不错，所以分享一下。

Karpathy Wiki GitHub原文地址：

https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f#file-llm-wiki-md

这个Markdown文件说明了构建LLM Wiki知识库的原理和步骤，可以理解为一个指南，并非一个具体的工具或者Skill，我们可以使用这个指南让我们的AI工具生成具体的行动清单或者创建Skill，接下来将以构建一个胜任素质模型研究的LLM Wiki进行举例。

**运行环境说明**

环境的安装本文不做介绍，简单列出：

<table style="border-collapse: collapse"><tbody><tr><td data-colwidth="135" style="border: 1px solid \#ddd; padding: 6px 10px"><div style="color: rgb(0, 0, 0)"><span>AI工具：</span></div></td><td data-colwidth="287" style="border: 1px solid \#ddd; padding: 6px 10px"><div style="color: rgb(0, 0, 0)"><span>Claude Code</span></div></td></tr><tr><td data-colwidth="135" style="border: 1px solid \#ddd; padding: 6px 10px"><div style="color: rgb(0, 0, 0)"><span>AI模型：</span></div></td><td data-colwidth="287" style="border: 1px solid \#ddd; padding: 6px 10px"><span>MiniMax-M2.7</span></td></tr><tr><td data-colwidth="135" style="border: 1px solid \#ddd; padding: 6px 10px"><div style="color: rgb(0, 0, 0)"><span>其他工具：</span></div></td><td data-colwidth="287" style="border: 1px solid \#ddd; padding: 6px 10px"><span>CC Switch(用于配置模型的API)</span></td></tr></tbody></table>

  

**自动创建知识库**

在根目录新建了一个readme.md，内容如下，其中横线上方的是我添加的提示词，\# LLM Wiki及以下的内容是llm-wiki.md的原文：

```
请参考以下内容，在当前文件夹下构建一个关于胜任素质模型的wiki，
请创建好目录和以后的行动指南，确保所有动作可以按规则进行。
所有关于规则的文件使用中文。
----------
# LLM Wiki
...llm-wiki.md原文...
```

在Claude Code中，使用/init初始化命令，创建CLAUDE.md文件：

![[Inbox/笔记同步助手/images/43d3bc7863351a4be4f1e0e6881859b1_MD5.jpg]]

自动生成的CLAUDE.md文件内容如下：

````
# CLAUDE.md
This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.
## 项目概述
这是一个基于 LLM 的个人知识库（胜任素质模型 wiki）。LLM 负责维护和更新所有 wiki 页面，用户负责提供来源、引导分析和提问。
## 目录结构
```
kb2_胜任素质模型/
├── README.md              # LLM Wiki 模式说明
├── CLAUDE.md              # 本文件 - Wiki 维护规则
├── raw/                   # 原始来源（不可修改）
│   └── sources/           # 源文档（文章、论文等）
└── wiki/                  # LLM 生成的 wiki 页面
├── index.md           # 内容索引（按分类组织）
├── log.md             # 操作日志（按时间顺序）
└── ...                # 其他 wiki 页面
```
## 核心原则
1. **raw/ 目录不可修改** - 原始来源是信任根源，LLM 只读取不修改
2. **wiki/ 目录由 LLM 完全拥有** - LLM 创建、更新和维护所有 wiki 页面
3. **知识是累积的** - 每次添加新来源时，LLM 整合到现有 wiki，不重复劳动
## 工作流程
### 摄入（Ingest）
当用户提供新来源时：
1. 读取源文档
2. 与用户讨论关键要点
3. 在 wiki 中创建/更新相关页面
4. 更新 index.md
5. 在 log.md 记录本次摄入
### 查询（Query）
当用户提问时：
1. 先读取 index.md 找到相关页面
2. 阅读相关页面，综合答案
3. 如答案有价值，将其归档为 wiki 新页面
### 清理（Lint）
定期健康检查：
- 检查页面间矛盾
- 标记过时内容
- 找出孤立页面（无 inbound 链接）
- 补充缺失的交叉引用
- 识别需要进一步研究的内容缺口
## index.md 格式
```markdown
# 索引
## 胜任素质
- [页面标题](wiki/页面标题.md) - 一句话描述
## 概念
...
## 来源
...
```
## log.md 格式
每条记录以日期开头，便于 grep 解析：
```markdown
## [2026-04-10] ingest | 来源标题
- 讨论了 xxx 关键点
- 更新了 yyy 页面
## [2026-04-10] query | 用户问题
- 综合了 zzz 页面的信息
```
## 页面frontmatter规范
Wiki 页面应包含 frontmatter：
```yaml
---
title: 页面标题
tags: [标签1, 标签2]
created: 2026-04-10
source: 来源名称
---
```
## 工具建议
- **Obsidian** - 推荐作为 wiki 浏览工具，支持图形视图、Dataview 插件
- **qmd** - 本地 markdown 搜索引擎，支持 CLI 和 MCP server（可选）
- **Obsidian Web Clipper** - 浏览器扩展，将网页转为 markdown
- **Marp** - markdown 幻灯片格式
## 通用操作命令
由于这是纯 markdown 文件项目，无构建/测试命令。核心操作是：
- 摄入新来源：读取 raw/sources/ 中的文件并整合到 wiki
- 查询：搜索 wiki/ 目录中的相关页面
- 清理：运行 lint 工作流程检查 wiki 健康度
## 扩展
如需添加搜索能力，可以使用项目根目录的 qmd MCP server。
````

将项目的文件夹按要求创建，我的输入都是PDF文档，没有图片，将输入文件放入/raw/sources目录下，在命令行输入：执行摄入操作

![[Inbox/笔记同步助手/images/a661910847257dcedf3b88b1d4fbea0d_MD5.jpg]]

根据提示，处理所有文档，在输入框回复Claude Code：

![[Inbox/笔记同步助手/images/8ca012a1b455e4fccc9edfecf0eb507a_MD5.jpg]]

可以看到，在我们没有任何信息输入的情况下，自动为我们建立了一个知识库：

![[Inbox/笔记同步助手/images/78f9d5bec7394cfa595620d2cdfb161d_MD5.jpg]]

**知识库预览**

AI生成的知识库都是Markdown格式，我推荐使用Obsidian预览，它可以很好的编辑和查看Markdown格式的文件，打开根目录即可。可以看到，AI生成了4类文件，分别是：

唯一的一个Index索引文件：

![[Inbox/笔记同步助手/images/626cba4619d238ff751db9655603861a_MD5.jpg]]

log文件：

![[Inbox/笔记同步助手/images/fbf8b911e044721a0886de4bcd1bf4b6_MD5.jpg]]

关键知识概览文件：

![[Inbox/笔记同步助手/images/ff3d3e5f91179826507345806965a796_MD5.jpg]]

每篇文档的知识提取文件：

![[Inbox/笔记同步助手/images/35fc0820cdc0af135c6820e112c44647_MD5.jpg]]

可以看到，我们创建了一个具有层级结构的Markdown格式的知识库，索引-知识总结-原始知识文件，三层结构。

而且Markdown格式非常易于编辑和读取，也非常容易被AI所消费。例如使用其他AI Agent执行任务时，可以指定这个整理好的知识库作为输入，让AI Agent读取Index一步步的查找所需要的知识。

**知识库优化**

默认生成的知识库，未必包含了全部我们关注的知识主题，我们可以建立不同的知识主题，总结所需要的知识。我们先来优化一下目录结构，我的提示词是：\[整理一下wiki的结构，将index和log文件保留在wiki的根目录，再wiki下建立两个新文件夹，分别是 知识主题 和知识来源，将35个来源分别总结的文件移动到知识来源文件夹，将总结性的主题知识移动到知识主题文件夹。\]

![[Inbox/笔记同步助手/images/e9bfb7a71b06ef802bee064caf76dfcf_MD5.jpg]]

可以看到，在log中AI记录了这一次的整理，并且给出了很好的建议：

![[Inbox/笔记同步助手/images/59a8170eb04f3fc1e85718264d08d9dc_MD5.jpg]]

我现在想做一些专题知识整理，并且对已有的知识增加我感兴趣的内容，提示词如下：\[增加胜任素质要素的主题，着重分析这些研究中，提取到了哪些胜任素质要素及其分类。可以通过表格或者任何易于理解的方式进行展示。也可以再更新一下所有的知识来源，提取raw原始文档中知识要素相关的内容。\]

![[Inbox/笔记同步助手/images/23049299cfac97d4e3bc1f12d52e7e19_MD5.jpg]]

![[Inbox/笔记同步助手/images/b78386817d71e8253ad9427d2d4b6623_MD5.jpg]]

整理后的Markdown概览如下：

![[Inbox/笔记同步助手/images/6db2dc38a739e7327a3d1390d1acf489_MD5.jpg]]

![[Inbox/笔记同步助手/images/b122edfaee9064cbf41054a6dbfcedb7_MD5.jpg]]

![[Inbox/笔记同步助手/images/56575e570e51039527808c0862ca7439_MD5.jpg]]

**总结**

可以看到，通过这种方式，非常快速地搭建了一个小型专题知识库，并且能够有效地作为其他AI的输入。

AI的能力越来越强，我们未来可能关注的是如何有效地使用、引导、管控、审计AI的工作过程和成果。在本案例的方法中，可以有效激活AI的能力，人只需要做决策，输入少量内容，最大化利用AI的能力帮忙完成任务。

感谢大家阅读。

本文章仅代表作者个人看法。

**欢迎点击以下名片关注**

  

---

![[Inbox/笔记同步助手/images/f19d625398ebd96522176272893d5d33_MD5.jpg|cover_image]]

原创 Brook Li Brook的知识分享平台

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/86daacf3_1778126464347?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI1ODc5MDY2OA%3D%3D%26mid%3D2247484585%26idx%3D1%26sn%3D657a8f1c1d5369b7620c40e52d3bf077%26chksm%3Deb88f123214665d7bba9239780d052da6508fd82e3ae78cf277f8cadfc7d3b61a44be692a758%26mpshare%3D1%26scene%3D1%26srcid%3D0507hgYtFrLpOmFgOCJJvtsc%26sharer_shareinfo%3D0ad28a2526e0f76ffa262b4dfe431804%26sharer_shareinfo_first%3D0ad28a2526e0f76ffa262b4dfe431804%23rd&s=obsidian)