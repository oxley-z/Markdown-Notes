---
author: 坚强粑粑
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI5MTExNjE0OQ==&mid=2648442756&idx=1&sn=9c080a083d3c226410fd6886499d6052&chksm=f51c87ab54d97c6f14b4babd59f04e726461bb840778582db42ee032c8a546f32962ae12c21b&mpshare=1&scene=1&srcid=0507lKrYyauAO0W06SUUI39O&sharer_shareinfo=7bfa012aaa9d3496def00e15b66eb77e&sharer_shareinfo_first=7bfa012aaa9d3496def00e15b66eb77e#rd
saved: 2026-05-07 12:00:40
tags:
  - 笔记同步助手
id: 1fa84a8b-02b2-41e9-a54b-d79e69d2c836
---

公众号名称：坚强粑粑

作者名称：坚强粑粑

发布时间：2026-04-30 17:45

> 整理时间：2026-04-29  
> 来源：官方文档、社区实践、安全审计报告  
> 分类：AI / ClaudeCode

---

## 一、CLAUDE.md 配置原则

### 核心原则：保持简短

-   **控制在 60 行以内**，硬上限 300 行
    
-   LLM 能可靠遵循约 150-200 条指令，Claude Code 系统提示已占用约 50 条
    
-   **只放 Claude 可能忽略的信息**：构建命令、测试命令、分支命名规范、项目特定架构决策
    
-   **能从代码推断的内容不要写进去**
    
-   规则太多？拆分到 `.claude/rules/`目录下按需加载
    
-   **关键规则用标签包裹**防止被忽略
    

### 示例：好的 CLAUDE.md 结构

```
## 工作流
- 每次代码变更后运行 `npm test`
- 每个任务创建新分支，绝不直接提交到 main
- 使用 Conventional Commits（feat:, fix:, refactor:, docs:）
- 每次提交前运行 `eslint . --fix`
- 完成后通过 `gh pr create` 创建 PR

## 技术栈
- Node.js 18+, Express 4.x, PostgreSQL 16
- 测试：Jest + React Testing Library
- 认证：JWT + bcrypt
```

## 二、工作流最佳实践

### 1\. 复杂任务用 Plan Mode

-   按 `Shift+Tab`两次进入计划模式
    
-   Claude 只研究和规划，不写代码
    
-   确认计划后再切换回正常模式执行
    
-   **官方推荐流程**：`探索 → 规划 → 实现 → 提交`
    

### 2\. 让 Claude 先采访你

-   给出简单需求描述，让 Claude 用 `AskUserQuestion`工具采访你
    
-   它能发现你忽略的边缘情况
    
-   **采访后开新会话执行**（采访对话会污染上下文）
    

### 3\. 分阶段工作流

-   理解代码库 → 修改
    
-   先规划 → 再实现
    
-   生成 → 验证
    
-   **不要把所有步骤压缩到一个大提示词里**
    

### 4\. 小任务别用复杂工作流

-   **3-5 分钟能完成的事，直接用原生 Claude Code**
    
-   复杂工作流（Superpowers、Spec Kit 等）适用于多文件、多步骤的大任务
    
-   重命名变量这种小事，一句话就行
    

## 三、调试与纠错

### 1\. 粘贴 bug，说"fix"

-   把错误信息粘贴给 Claude，说一个字："fix"
    
-   **不要指导怎么修**，不要猜测原因，不要指定解决方案
    
-   Claude 的调试能力比想象中强，管得越多越容易带偏
    
-   直接让 Claude 修的成功率 80%+
    

### 2\. 两次失败 = /clear

-   同一个问题修正超过两次，`/clear`重新开始
    
-   上下文污染会降低性能
    
-   官方建议：修正超过两次就重启
    

### 3\. 走偏了？Esc Esc 回滚

-   按两次 `Esc`（或 `/rewind`）直接回滚到上一个检查点
    
-   在同一上下文中纠正偏差往往更糟
    
-   同一个问题偏差两次？`/clear`重启
    

### 4\. 要求重写平庸方案

-   当 Claude 给出能工作但不优雅的解决方案时，不要修补
    
-   说："知道你现在知道的一切，抛弃这个，实现优雅的解决方案"
    
-   重写版本通常比修补版本好得多
    

## 四、上下文管理

### 1\. 50% 时手动压缩

-   上下文使用超过 60-70% 时，性能明显下降
    
-   \*\*在 50% 时手动执行 `/compact`\*\*，不要等自动压缩
    
-   用 `/statusline`实时监控使用情况
    
-   或使用 `/clear`开新会话
    

### 2\. /compact 可指定压缩策略

```
# 聚焦 API 变更压缩
/compact focusing on API changes

# 保留测试相关历史
/compact keep test-related history

# 保留错误解决历史
/compact keep error resolution
```

### 3\. Checkpoints（检查点）

-   每次 Claude 操作自动创建
    
-   **可独立回滚对话或代码**
    
-   跨会话持久化
    
-   **不是 git 的替代品**
    

## 五、Subagents（子智能体）

### 1\. 在提示词中加"use subagents"

-   Claude 会自动拆分任务给多个子智能体并行处理
    
-   适合代码审查、大规模重构
    

### 2\. 专用子智能体 > 通用 mega-agent

-   创建**功能特定**的子智能体（如"前端组件智能体"）
    
-   而不是通用的（如"QA 智能体"）
    
-   功能越具体，上下文越精准，效果越好
    

### 3\. 子智能体有独立上下文窗口

-   研究、验证、审查隔离在独立上下文中
    
-   **防止污染和偏见**
    
-   不污染主上下文
    

### 4\. 并行子智能体示例

```
# 代码审查：9 个子智能体，每个关注不同质量维度
claude-flow orchestrate "审查用户认证模块" --agents 9 --parallel

# 跨文件重命名：3 个子智能体并行
claude-flow orchestrate "重命名所有文件中的 User 为 Account" --agents 3 --parallel
```

## 六、Skills（技能）管理

### 1\. 技能应该是文件夹结构

```
skills/
  api-design/
    SKILL.md          # 主文件：核心规则和索引
    references/       # 语料库、参考资料
    scripts/          # 辅助脚本
    examples/         # 示例代码
```

-   主文件只包含核心规则和索引
    
-   语料库、检查表放在 `references/`
    
-   **渐进式披露**：Claude 只在需要时读取子目录内容
    

### 2\. 添加 Gotchas（坑点记录）部分

**这是长期最有价值的技术**：每次 Claude 犯错时记录失败模式，长期积累成为**信噪比最高的内容**。

#### Gotchas 结构示例

```
# SKILL.md

## 核心规则
...

## Gotchas（坑点记录）

### 2026-04-15: API 分页参数遗漏
- **问题**：生成 API 时忘记添加分页参数
- **表现**：返回所有数据导致性能问题
- **修复**：在 SKILL.md 中添加分页规则
- **预防**：检查清单中增加"是否包含分页"

### 2026-04-20: AI 写作模式
- **问题**：生成的技术文档有 AI 写作痕迹
- **表现**：过度使用"此外"、"值得注意的是"等
- **修复**：添加 humanizer 技能到工作流
- **预防**：生成后运行 humanizer 检查

### 2026-04-25: 测试覆盖不足
- **问题**：只测试 happy path，忽略边缘情况
- **表现**：测试通过但实际运行报错
- **修复**：添加边界值测试规则
- **预防**：检查清单中增加"边缘情况覆盖"
```

#### Gotchas 维护原则

1.  **每次犯错必记录**：不要等，立即记录
    
2.  **包含四个要素**：问题描述、表现形式、修复方法、预防措施
    
3.  **定期回顾**：每周回顾一次，识别重复出现的模式
    
4.  **转化为规则**：如果某个坑点出现 3 次以上，转化为正式规则
    
5.  **归档已解决的**：超过 30 天未出现的问题，移到归档区
    

### 3\. 技能可基于上下文自动触发

-   不需要显式调用
    
-   YAML frontmatter 控制自动应用条件
    
-   危险操作（如部署）用 `disable-model-invocation: true`防止误触发
    

## 七、Superpowers 使用详解

### 1\. 什么是 Superpowers？

Superpowers 是由 Jesse Vincent 和 Prime Radiant 团队开发的 Claude Code 插件（14 万+ GitHub stars），解决**工程纪律**问题。

**核心功能**：

-   强制结构化工作流：头脑风暴 → 分支隔离 → 详细计划 → 执行
    
-   TDD（测试驱动开发）
    
-   代码审查
    
-   系统调试
    
-   验证完成
    

### 2\. 安装与配置

```
# 在 Claude Code 会话中安装
/plugin install superpowers@claude-plugins-official

# 下次启动时看到"You have Superpowers"即表示成功
```

### 3\. 技能激活方式

| 技能 | 何时激活 | 触发方式 |
| --- | --- | --- |
| brainstorming | 创建功能或组件前 | 单独使用时自动 |
| writing-plans | 需求需要多步分解时 | 单独使用时自动 |
| test-driven-development | 实现功能或修复 bug 前 | 需在 CLAUDE.md 中显式配置 |
| systematic-debugging | 遇到 bug、测试失败、意外行为时 | 需在 CLAUDE.md 中显式配置 |
| code-reviewer | 完成主要实现步骤后 | 需在 CLAUDE.md 中显式配置 |
| dispatching-parallel-agents | 多个独立任务可并行时 | 当 2+ 任务无依赖时自动 |
| verification-before-completion | 声称工作完成前 | 需在 CLAUDE.md 中显式配置 |

### 4\. CLAUDE.md 配置示例

```
## Superpowers 工作流规则

### 新功能开发
- 使用 /opsx:propose 开始（路由到 OpenSpec）
- 跳过 brainstorming/writing-plans（避免重复）

### 编码纪律
- 使用 /opsx:apply 时，始终遵循 TDD：先写失败的测试，再实现代码
- 遇到 bug 时，使用 systematic-debugging 技能
- 完成主要实现后，自动触发 code-reviewer

### 验证规则
- 声称工作完成前，必须通过 verification-before-completion
- 所有测试必须通过，无跳过测试
```

### 5\. 四步强制序列

1.  **头脑风暴**：解决重大架构决策（比写代码便宜）
    
2.  **分支隔离**：每个功能在独立分支上开发
    
3.  **详细计划**：编写可审查的计划文档
    
4.  **执行**：按计划实施，每步都有检查点
    

### 6\. 实际效果

-   计划覆盖 17 个文件，精确到每个文件的职责
    
-   实施产生 26 次提交，每次对应计划任务
    
-   计划可被 spec-compliance 子智能体审查
    
-   发现遗漏的基准测试等问题
    

## 八、Spec Kit（OpenSpec）使用详解

### 1\. 什么是 OpenSpec？

OpenSpec 是 Fission AI 开发的开源框架，解决**需求不匹配**问题。将一句话需求扩展为四个结构化文档。

**核心功能**：

-   proposal.md：为什么、范围、**不在范围内什么**（防止 AI 添加未请求的功能）
    
-   specs/：使用 GIVEN/WHEN/THEN 场景的行为规范
    
-   design.md：技术决策及推理
    
-   tasks.md：实现清单，每个任务 2-5 分钟可完成
    

### 2\. 安装与配置

```
# 需要 Node.js 20.19.0+
npm install -g @fission-ai/openspec@latest

cd your-project
openspec init  # 选择 Claude Code

# 创建 openspec/ 目录，包含 specs/、changes/archive/、AGENTS.md
```

### 3\. 与 Claude Code 集成

```
// .claude/settings.json
{
  "mcpServers": {
    "openspec": {
      "command": "npx",
      "args": ["-y", "@fission-ai/openspec-mcp"]
    }
  },
  "permissions": {
    "allow": ["Bash:openspec:*", "Bash:npm:*", "Bash:git:*"]
  }
}
```

### 4\. 工作流程

```
# 会话 1：需求 → 规范
> /opsx:propose 用户认证 API，Express + MongoDB + JWT

# 生成：
# - openspec/changes/YYYY-MM-DD--proposal.md
# - openspec/changes/YYYY-MM-DD--specs/
# - openspec/changes/YYYY-MM-DD--design.md
# - openspec/changes/YYYY-MM-DD--tasks.md

# 会话 2：规范 → 实现
> /opsx:apply

# 会话 3：独立验证
> /opsx:archive  # 归档当前迭代
```

### 5\. Delta/Archive 机制

-   **Delta**：每次迭代保留决策历史
    
-   **Archive**：归档已完成的迭代，保留审计追踪
    
-   解决设计决策在迭代中丢失的问题
    

### 6\. 五大常见陷阱

| 陷阱 | 表现 | 预防 |
| --- | --- | --- |
| 规范写成伪代码 | 描述实现而非行为 | 使用 GIVEN/WHEN/THEN |
| 过度详细规范 | 限制 AI 创造性 | 描述"什么"，不描述"怎么做" |
| 每次功能后不归档 | 历史混乱 | 每个功能完成后执行 archive |
| 与 Superpowers 冲突 | 两个规划系统重复 | 在 CLAUDE.md 中路由到一个 |
| 忽略 out-of-scope | AI 添加未请求功能 | 明确定义范围边界 |

## 九、权限与安全

### 1\. Hooks vs CLAUDE.md

| 需求 | 推荐 | 原因 |
| --- | --- | --- |
| 文件保存后自动 lint | Hook | 每次必须执行 |
| 阻止写入敏感文件 | Hook | 安全不能妥协 |
| 代码规范遵循 | CLAUDE.md | 需要情境判断 |
| API 命名规则 | CLAUDE.md | 存在例外模式 |

### 2\. Allowlist 减少审批疲劳

```
{
  "permissions": {
    "allow": [
      "Bash(npm run lint:*)",
      "Bash(npm run test:*)",
      "Bash(git status)",
      "Read",
      "Glob",
      "Grep"
    ]
  }
}
```

### 3\. deny 比 Hooks 更安全

```
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Bash(curl:*)"
    ]
  }
}
```

-   权限评估顺序：`deny → ask → allow`
    
-   设为 `deny`后文件对 Claude"不可见"
    

### 4\. --dangerously-skip-permissions 正确使用

| 适用场景 | 不适用场景 |
| --- | --- |
| Lint 修复自动化 | 联网环境 |
| 样板代码生成 | 包含敏感数据的环境 |
| **封闭工作流** | 通用开发工作 |

**重要**：应在**无互联网的隔离环境**中使用。企业可设置 `disableBypassPermissionsMode: true`全局禁用。

​

## 十、规格与实现分离

### 推荐流程

1.  **Session 1**：通过采访创建规格
    
2.  **Session 2**：基于规格实现
    
3.  **Session 3**：独立验证
    

### 为什么分离？

-   采访和规格讨论污染上下文
    
-   新会话有干净的上下文窗口
    
-   规格文档作为实现依据
    
-   验证会话独立于实现偏见
    

## 十一、Claude Code 常见陷阱（Gotchas）

### DoltHub 团队总结的 8 大陷阱

| # | 陷阱 | 表现 | 缓解方法 |
| --- | --- | --- | --- |
| 1 | 过早放弃 | "已实现大部分功能，但 XX 不工作" | 拆分任务为更小单元 |
| 2 | 上下文压缩后变笨 | 忘记之前纠正的错误 | 手动 /compact，必要时 /clear |
| 3 | 初始测试质量差 | 测试看起来对但实际失败 | TDD 模式，仔细审查测试 |
| 4 | 修改测试而非代码 | 降低测试标准匹配错误代码 | 严格审查测试变更 |
| 5 | 忘记编译 | 测试失败因为未编译 | 在 CLAUDE.md 中明确编译步骤 |
| 6 | 工作目录混乱 | 留下测试脚本、构建产物 | git status 检查，手动清理 |
| 7 | Git 操作危险 | 错误的变更合并到 PR | 人工执行 Git 操作 |
| 8 | 重写但不删除旧代码 | 新旧代码共存 | 审查 diff，确认删除 |

### 陷阱 1：过早放弃

**表现**：

```
我已实现大部分功能。功能在 XX 情况下工作正常。
但 YY 情况下不工作。代码已充分测试，这是好的开始。
```

**缓解**：

-   拆分任务为更小、更隔离的单元
    
-   即使人类认为可以分组，Claude Code 也需要分离
    
-   示例：两个相似表 → 分两个 PR，每个 10 分钟完成
    

### 陷阱 2：上下文压缩后变笨

**表现**：

-   不知道之前看的文件
    
-   重复之前纠正的错误
    
-   性能明显下降
    

**缓解**：

-   50% 时手动 `/compact`
    
-   指定压缩策略（保留什么）
    
-   必要时 `/clear`\+ `git reset --hard`
    

### 陷阱 3 & 4：测试问题

**表现**：

-   生成看起来对但失败的测试
    
-   修改测试匹配错误代码
    
-   降低测试标准
    

**缓解**：

-   TDD 模式：先写测试
    
-   仔细审查生成的测试
    
-   严格审查测试变更（比代码变更更严格）
    

### 陷阱 5：忘记编译

**表现**：

-   测试循环失败因为未编译
    
-   依赖变更后忘记重新编译
    

**缓解**：

-   CLAUDE.md 中明确编译步骤
    
-   测试前强制编译
    
-   注意：编译语言 vs 解释语言混合时特别容易出错
    

### 陷阱 6 & 7：工作目录和 Git

**表现**：

-   留下测试脚本、数据库文件
    
-   Git 操作错误导致 PR 混乱
    

**缓解**：

-   每次完成后 `git status`检查
    
-   **人工执行 Git 操作**（分支、提交、推送）
    
-   Claude Code 只修改文件，不操作 Git
    

### 陷阱 8：重写但不删除

**表现**：

-   创建新文件但不删除旧文件
    
-   新旧代码共存导致混淆
    

**缓解**：

-   审查 diff 确认删除
    
-   明确指示"删除旧实现"
    
-   检查文件列表确认清理
    

## 十二、工具组合策略

### Claude Code + OpenSpec + Superpowers 三重栈

**解决三个核心问题**：

| 问题 | 工具 | 解决方式 |
| --- | --- | --- |
| AI 构建的不是你想要的 | OpenSpec | 需求 → 结构化规范 |
| AI 跳过工程纪律 | Superpowers | 强制 TDD、审查、验证 |
| 设计决策在迭代中丢失 | OpenSpec | Delta/Archive 机制 |

### 分工明确

```
OpenSpec 负责：思考 WHAT（构建什么、为什么）
Superpowers 负责：确保 HOW（如何构建好）
Claude Code 负责：执行（编辑文件、运行测试、处理 Git）
```

### 配置协作

```
# CLAUDE.md 中的路由规则

## 规划阶段
- 任何新功能：从 /opsx:propose 开始
- 跳过 brainstorming/writing-plans（避免重复）

## 编码阶段
- 使用 /opsx:apply 时：始终遵循 TDD
- 遇到 bug：使用 systematic-debugging
- 完成实现：触发 code-reviewer
- 声称完成：通过 verification-before-completion
```

## 十三、核心原则总结

### 上下文是宝贵资源

-   保持简洁、及时压缩、污染就重置
    
-   50% 时手动 `/compact`
    
-   两次失败 = `/clear`
    

### 系统约束 > 提示词约束

-   用 Hooks 和权限配置代替"希望 Claude 记住"
    
-   `deny`比 Hooks 更安全
    
-   关键规则用标签包裹
    

### 分而治之

-   子智能体、分阶段工作流、规格与实现分离
    
-   专用子智能体 > 通用 mega-agent
    
-   小任务简单处理，复杂任务才需要完整工作流
    

### 不要过度工程

-   3-5 分钟能完成的事，直接用原生 Claude Code
    
-   复杂工作流适用于多文件、多步骤的大任务
    
-   重命名变量这种小事，一句话就行
    

### 持续改进

-   每次犯错必记录 Gotchas
    
-   定期回顾，识别重复模式
    
-   将高频问题转化为正式规则
    

## 参考资料

-   Claude Code 官方文档：https://docs.anthropic.com/claude-code
    
-   Superpowers 插件：https://github.com/obra/superpowers
    
-   OpenSpec 框架：https://openspec.dev/
    
-   10 个必备最佳实践：https://discuss.huggingface.co/t/10-essential-claude-code-best-practices-you-need-to-know/174731
    
-   Claude Code Gotchas（DoltHub）：https://www.dolthub.com/blog/2025-06-30-claude-code-gotchas/
    
-   Superpowers 插件详解：https://www.builder.io/blog/claude-code-superpowers-plugin
    
-   OpenSpec + Superpowers 三重栈：https://www.heyuan110.com/posts/ai/2026-04-09-claude-code-openspec-superpowers/
    

\#Claudecode

  

---

![[Inbox/笔记同步助手/images/9d6264ce6bea77606fcf1609a5cd7f30_MD5.jpg|cover_image]]

原创 坚强粑粑 坚强粑粑

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/de5eb50f_1778126439491?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI5MTExNjE0OQ%3D%3D%26mid%3D2648442756%26idx%3D1%26sn%3D9c080a083d3c226410fd6886499d6052%26chksm%3Df51c87ab54d97c6f14b4babd59f04e726461bb840778582db42ee032c8a546f32962ae12c21b%26mpshare%3D1%26scene%3D1%26srcid%3D0507lKrYyauAO0W06SUUI39O%26sharer_shareinfo%3D7bfa012aaa9d3496def00e15b66eb77e%26sharer_shareinfo_first%3D7bfa012aaa9d3496def00e15b66eb77e%23rd&s=obsidian)