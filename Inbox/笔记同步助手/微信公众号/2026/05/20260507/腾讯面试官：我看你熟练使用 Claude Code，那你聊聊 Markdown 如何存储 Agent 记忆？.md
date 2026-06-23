---
author: 小金
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzIwNDgzMzI3Mg==&mid=2247494926&idx=1&sn=13b79042345b6eca98b5637433a5215d&chksm=96c7aa33e490d64b4d0776e4d3d39b417ef2fd6c83d66ac88311b95477e466969f2ff06d23ac&mpshare=1&scene=1&srcid=0506ErVFA9IP51q37oCWqESK&sharer_shareinfo=8da6ebd6a617d29e4257e8d679588736&sharer_shareinfo_first=8da6ebd6a617d29e4257e8d679588736#rd
saved: 2026-05-07 09:47:33
tags:
  - 笔记同步助手
id: 64d2a26c-8123-4b07-b177-7f68781dd9a5
---

公众号名称：小金AI

作者名称：小金

发布时间：2026-05-06 22:27

原文链接：[https://javaguide.cn/ai/](https://javaguide.cn/ai/)

前面说了这么多向量库、知识图谱、记忆框架，你可能会问：有没有更轻量 Agent 记忆存储方案？

还真有。当你认真审视 Agent 记忆的本质需求时，会发现一个反直觉的答案——**Markdown 文件可能就是最务实的长期记忆载体，性价比拉满了**。

这篇文章是针对[万字详解 Agent 记忆](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247554299&idx=1&sn=d2d57de785fcd709ef2c9665a2ee5c1d&scene=21#wechat_redirect)的补充完善。

  

### 为什么 Markdown 可以作为 Agent 记忆

Markdown 本质上是一种人和 Agent 都能读写的“显式长期记忆”。它不依赖数据库、不需要向量引擎、不用配置检索管道。

核心优势在于**透明、可审查、可版本化、低成本**：

-   • **透明可审计**：随时打开文件，看得到 Agent 记住了什么、写入了什么。没有任何黑盒。
    
-   • **持久化**：文件存活于磁盘，不依赖进程生命周期。进程崩溃、重启、换机器，记忆都在。
    
-   • **版本控制**：记忆可以提交到 Git，回滚、分支、Code Review 随心所欲。
    
-   • **零迁移成本**：标准格式，无供应商锁定。换模型、换框架，只需迁移文件。
    
-   • **成本极低**：本地存储几乎免费，而向量数据库动辄会让成本增加。
    

Manus 把文件系统视为结构化的外部记忆；Claude Code 则把 `CLAUDE.md`和 Auto Memory 明确产品化；OpenClaw 等 Agent 项目/社区实践中，也能看到类似的文件化记忆思路。它们共同说明：在很多 Agent 场景里，文件系统 + Markdown 已经是一个足够务实的长期记忆方案。

### Claude Code 的 CLAUDE.md 机制

Claude Code 的记忆系统采用双轨制：`CLAUDE.md`（人工编写） 和 **Auto Memory（自动积累）**。

![[../../../../images/cf746b6f557ac79620d73a9bfccb9b2a_MD5.jpg]]

#### CLAUDE.md：该写什么、不该写什么

`CLAUDE.md`本质上是给 AI 新人写的 onboarding 文档。写得不好还不如不写——一份臃肿的 `CLAUDE.md`会让真正重要的规则被噪音淹没。

## 该写的东西：

-   • **技术栈和版本信息**：框架版本差异往往是 AI 犯错的源头。你不标注 Spring Boot 版本，它会倾向于生成训练数据中更常见的版本用法。
    
-   • **常用命令**：构建、测试、lint、启动——全部放在代码块里。代码块里的命令 Claude 倾向于照着跑，写在自然语言里的命令它可能根据自己的理解改写。
    
-   • **架构决策和背后的理由**：光写规则不够，写清楚“为什么”能让 Claude 举一反三。比如“不要直接写 SQL，使用 QueryWrapper”——加上“因为 SQL 审计系统依赖 Wrapper 解析来记录操作日志”之后，Claude 在其他需要生成查询的地方也自觉用 Wrapper。
    
-   • **团队约定和项目特有的坑**：提交信息格式、分支命名规范、环境变量依赖。这些 Claude 从代码里读不出来，但一个新入职的工程师一定会问。
    

## 不该写的东西：

-   • 代码风格规则（交给格式化工具）
    
-   • 语言或框架的默认行为（现代 Python 用 f-string 这种事写下来是噪音）
    
-   • 大段参考文档（给链接就行，Claude 需要时会自己去读）
    

> **一句话判断标准**：逐行过一遍 `CLAUDE.md`，每条规则问自己——“如果没有这行，Claude 最近是不是真的犯过这个错”。如果答案是“好像没犯过”，那行就可以删。

#### 怎么写才能让 Claude 真正遵守

**规则要具体可验证**。“注意代码可读性”没法验证；“函数名使用动词开头、单个函数不超过 40 行”可以验证。规则写得越具体，Claude 遵守的概率越高。

**禁令要搭配替代方案**。只说“不要做 X”会让 Claude 在遇到相关场景时卡住。更好的方式是“不要做 X，遇到这种情况应该做 Y”。实战例子：

​

```
# 依赖注入
- 不要使用 @Autowired 字段注入
- 使用构造器注入，配合 Lombok 的 @RequiredArgsConstructor
- 参考示例：UserController.java 中的写法
```

**善用标记词但别滥用**。如果某条规则 Claude 反复违反，加 `IMPORTANT:`或 `YOU MUST:`能引起它的注意。但这招不能经常用——到处标“重要”的文件，等于什么都不重要。

​

> **工程提示**：如果 Claude 反复忽略某条规则，不要急着加感叹号。更大的可能是文件太长了，规则被其他内容稀释了。解决方案是精简文件，不是加强调。

**标题用常规名字**。用 Commands、Structure、Conventions、Testing 这类在 README 里常见的标题。Claude 训练数据里有大量标准结构的 README，它对“这个标题下面通常写什么”有稳定的预期。

​

#### CLAUDE.md 文件的层级结构

| 层级 | 位置 | 作用范围 | 适用场景 |
| :-- | :-- | :-- | :-- |
| **组织级** | 系统目录（如 `/etc/claude-code/CLAUDE.md`） | 所有用户 | 公司编码规范、安全策略，任何设置都无法排除 |
| **用户级** | `～/.claude/CLAUDE.md` | 个人所有项目 | 代码风格偏好、个人工具习惯 |
| **项目级** | `./CLAUDE.md`或 `./.claude/CLAUDE.md` | 团队共享 | 项目架构、编码标准、工作流，提交至 Git |
| **本地级** | `./CLAUDE.local.md` | 个人当前项目 | 沙箱 URL、测试数据偏好，加入 `.gitignore` |

文件加载遵循目录树向上查找规则——从当前工作目录逐级向上，同一目录内 `CLAUDE.local.md`追加在 `CLAUDE.md`之后，越靠近工作目录的规则优先级越高。

## CLAUDE.md 不适合存什么：

-   • 大段日志和完整对话记录
    
-   • 敏感密钥、Token、账号信息
    
-   • 高频变化的运行时数据
    
-   • 可以实时查询的动态信息
    

## 分层管理：项目大了怎么组织

一个人的项目，一份 `CLAUDE.md`够用。项目一大、团队一介入，就需要分层：

​

```
# `CLAUDE.md`（项目根目录）
## Project
Spring Boot 3.2 + MyBatis-Plus + MySQL 8.0 的订单管理服务。

## Commands
- 构建：`mvn clean package`
- 测试：`mvn test`

## Rules
- API 约定：@docs/api-conventions.md
- 数据库规范：@docs/database-rules.md
```

用 `@path/to/file`引用外部文件。**注意**：`@`引用会把整个文件内容嵌入上下文，如果被引用文件很大，每个会话启动时都会烧掉一大块指令预算。大文件改为自然语言引用——"架构细节参见 `docs/architecture.md`"，Claude 需要时会自己读取。

对于更细粒度的控制，可以用 `.claude/rules/`目录组织 path-scoped 规则：

​

```
---
paths:
- "src/main/java/**/controller/**/*.java"
---
# Controller 规范
- 统一使用 Result 包装返回值
- 所有接口必须添加 Swagger 注解
```

这样编辑 Controller 时只加载 Controller 的规则，编辑 Service 时只加载 Service 的规则。

**Auto Memory（自动积累）**：Claude 根据对话自动写入的笔记，包括调试模式、代码习惯、工作流偏好。它存在 `～/.claude/projects//memory/`目录下，MEMORY.md 是入口文件，细节笔记在子文件中。

注意：Auto Memory 需要 Claude Code v2.1.59+，默认开启，可以通过 `/memory`或 `autoMemoryEnabled`配置关闭。

### Markdown 记忆的层级设计

一个完整的 Markdown 记忆体系通常包含多个层级：

-   • **用户级记忆**：存个人偏好和长期习惯，放在 `～/.claude/CLAUDE.md`。比如你偏好用 2-space 缩进、习惯先写测试再写代码、不喜欢用 emoji。
    
-   • **项目级记忆**：存项目规范、技术栈、目录结构，放在仓库根目录的 `CLAUDE.md`。团队成员共享，通过 Git 同步。
    
-   • **子目录级记忆**：存局部模块的专属规则，放在子目录的 `CLAUDE.md`。比如 `backend/`下的 API 设计规范、`docs/`下的写作风格要求。
    
-   • **团队共享记忆**：需要提交到仓库的共同约定。项目级的 `CLAUDE.md`和 `.claude/rules/`目录下可版本化的规则文件。
    
-   • **私有记忆**：不应该提交的个人工作流。`CLAUDE.local.md`这类文件加入 `.gitignore`，只存在本地。
    

### Markdown 记忆和传统长期记忆的区别

![[../../../../images/22b3701098e8676c43cac1425c7b69bd_MD5.jpg]]

Markdown 记忆和传统长期记忆的适用边界

不是所有场景都适合 Markdown，也不是所有场景都适合向量库。关键在于理解各自的适用边界：

​

| 维度 | Markdown 记忆 | 向量库记忆 | RAG 知识库 | 数据库型框架（Mem0等） |
| :-- | :-- | :-- | :-- | :-- |
| **检索精度** | 中等（依赖人工组织） | 高（语义相似度） | 高（语义检索） | 高（混合策略） |
| **调试体验** | **极佳**：直接读写文件 | 中等：需向量查询工具 | 中等：需检索日志 | 复杂：需理解框架逻辑 |
| **部署成本** | **极低**：只需文件读写 | 高：需维护向量服务 | 高：需 RAG pipeline | 高：需框架运行时 |
| **版本控制** | **原生集成**Git | 需额外同步机制 | 需额外同步机制 | 需额外同步机制 |
| **迁移成本** | **零**：复制文件即可 | 高：锁定专有格式 | 高：锁定 pipeline | 极高：绑定框架 |
| **适用场景** | 偏好、约定、踩坑记录 | 多样化记忆检索 | 共享知识查询 | 复杂多源记忆管理 |

Markdown 的局限性也很明确：当你需要从海量非结构化文本中检索特定片段时，人工组织的 Markdown 会成为瓶颈。此时向量库的语义检索能力不可替代。

反过来，当你的记忆需求是“记住这个项目的编码规范”、“记住用户的报告偏好”这类明确、可结构化的信息时，Markdown 的简洁和可维护性完胜复杂系统。

### Markdown 记忆的维护策略

这里以 `CLAUDE.md`为例。

`CLAUDE.md`不是写完就完事的。项目在演进，规则也会过时。

-   • **添加规则要慢**：一条新规则只有在 Claude 确实犯了一个错误、且这条规则能防止同类错误再次发生时，才值得写进去。为还没发生过的事预设规则，往往是在浪费空间。
    
-   • **删规则要果断**：如果某条规则存在很久了，但删掉后 Claude 的行为没有变化，说明这条规则从一开始就没起作用——Claude 本来就会这么做。果断移除，把空间留给真正需要的规则。
    
-   • **错误驱动的持续进化**：每次纠正 Claude 的错误后，追加一句“更新 `CLAUDE.md`，确保下次不再犯”。累积几次同类错误后归纳为一条精炼的规则，避免文件快速膨胀。
    

## 两个预警信号：

-   • **信号一**：Claude 为已经写在文件里的规则道歉（比如“抱歉，我刚才忽略了 XX 规则”）。这说明这条规则的措辞有问题——换个更直接的表述。
    
-   • **信号二**：同一条规则在不同会话中反复被违反。这通常不是措辞问题，而是整份文件太长了，规则被稀释了。解决方案不是改措辞，而是压缩整份文件。
    

## 两个实用的维护习惯：

-   • **对话式审查**：每隔几周，找几个 `CLAUDE.md`里的规则问 Claude：“如果我删掉这条规则，你会改变行为吗？”如果它说不会，那这条规则可能就可以删。
    
-   • **用 `/init`但别直接用**：自动生成的 `CLAUDE.md`是一个合理的起点，但里面可能包含对项目不准确的描述。按原则逐条审查，删掉冗余、补上遗漏。
    

**Git 做版本追踪 + Code Review**：每一次重要记忆更新都 commit，遇到问题可以回滚，code review 可以追溯修改原因。团队共享内容的修改应该走 PR 流程。

## 推荐

## ⭐️文章推荐：

1.  1\. [一个周末，我从零搭出了一个 Claude Code 同款 Agent 系统：Agent Loop、工具调用、Subagent、Skills、记忆、多 Agent 协作、MCP……](https://mp.weixin.qq.com/s?__biz=MzIwNDgzMzI3Mg==&mid=2247494837&idx=1&sn=fdb5f389b266a8ef7fbec68bef85b973&scene=21#wechat_redirect)
    
2.  2\. [终于有好用的 Claude Code 状态栏增强插件了！](https://mp.weixin.qq.com/s?__biz=MzIwNDgzMzI3Mg==&mid=2247494820&idx=1&sn=1165aee9b2cd9a342bace1e0ed2849c5&scene=21#wechat_redirect)
    
3.  3\. [ClaudeCode/Codex 成本爆降 80%的神级开源工具！治好了我的Token焦虑](https://mp.weixin.qq.com/s?__biz=MzIwNDgzMzI3Mg==&mid=2247494855&idx=1&sn=640205a623de3c730e6e7720020a403a&scene=21#wechat_redirect)
    

## 如果文章帮助欢迎点赞&分享！欢迎关注我们👇

---

![[../../../../images/cda0d5eab0be622da10b95c7b9a0282b_MD5.jpg|cover_image]]

Original 小金 小金AI

Read more