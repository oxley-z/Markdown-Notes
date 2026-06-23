OpenCode 是一个开源的 AI 编程代理（AI coding agent），支持在终端（Terminal）、桌面应用和主流 IDE（如 VS Code）中与 AI 交互完成代码相关任务。

OpenCode 类似于 Claude 的 Code 模式或 Cursor 的 Agent 功能，但完全开源、隐私优先，支持多种大语言模型（LLM），并强调终端体验。
# 安装

```bash
npm install -g opencode-ai
```

# 基础知识

## 关键特性

**两种内置 Agent 模式（在TUI窗口使用TAB按键切换）：**

- **Build 模式**：全权限，可直接编辑文件、执行命令。
- **Plan 模式**：只读规划，默认拒绝编辑，需要确认。

工具集：bash 执行、文件读写、grep 搜索、LSP 诊断等。

上下文感知：自动分析项目结构，生成 AGENTS.md 指南。

分享与协作：一键生成会话分享链接。

## OpenCode CLI

### Web界面

```bash
# 打开默认web界面
opencode web
# 打开以4096为断开的opencode web界面
opencode web --port 4096
# 在网络中访问
opencode web --hostname 0.0.0.0
```

### 启动参数

| 参数         | 简写  | 说明                             |
| ---------- | --- | ------------------------------ |
| --continue | -c  | 继续上次会话                         |
| --session  | -s  | 指定会话ID                         |
| --fork     | ——  | 分叉会话                           |
| --prompt   | ——  | 初始化提示词                         |
| --model    | -m  | 指定模型                           |
| --agent    | ——  | 指定代理                           |
| --port     | ——  | 监听端口                           |
| --hostname | ——  | 主机地址                           |
| --debug    | -d  | 启用调试模式（输出更多日志）                 |
| --cwd      | -c  | 指定当前工作目录（启动时切换到该目录）            |
| --prompt   | ——  | 非交互式模式：直接运行单个提示并输出响应（适合脚本/自动化） |
## OpenCode TUI（终端界面）

OpenCode 提供了一个交互式终端界面（TUI，Terminal User Interface），用于在命令行中与 AI 进行高效协作开发。

TUI 是 OpenCode 的核心使用方式，所有代码分析、修改、执行都通过这个界面完成。

OpenCode TUI 本质是一个可执行命令的 AI 对话终端，它把开发、命令行和 AI 融合在一起。

所有的 TUI 命令使用斜杆 / 唤起。

在 TUI 输入框中输入 / 就会列出联想的命令：

![](image/OpenCode教程/IMG-20260604143811938.png)

#CLI
#### OpenCode TUI 命令

| 命令                           | 描述                  | 快捷键       |
| ---------------------------- | ------------------- | --------- |
| /connect                     | 添加LLM提供商并配置API Keys | ——        |
| /compact                     | 压缩当前会话上下文           | ctrl+x->c |
| /details                     | 切换工具执行详情显示          | ctrl+x->d |
| /editor                      | 打开外部编辑器编写信息         | ctrl+x->e |
| /exit（/quit /q）              | 退出TUI               | ctrl+x->q |
| /export                      | 导出当前对话为 Markdown    | ctrl+x->x |
| /init                        | 创建/更新 AGENTS.md     | ctrl+x->i |
| /models                      | 列出可用模型              | ctrl+x->m |
| /new（/clear）                 | 新建会话                | ctrl+x->n |
| /redo                        | 重做被 /undo 撤销的操作     | ctrl+x->r |
| /sessions（/resume /continue） | 会话列表与切换             | ctrl+x->l |
| /share                       | 分享当前会话              | ctrl+x->s |
| /themes                      | 列出/切换主题             | ctrl+x->t |
| /thinking                    | 显示/隐藏模型推理过程         | ——        |
| /undo                        | 撤销上一条消息（含文件变更）      | ctrl+x->u |
| /unshare                     | 取消分享会话              | ——        |

#TUI
### 快捷键

OpenCode的快捷键实现是以`ctrl+x`为"<mark style="background: #FF5582A6;">前导键</mark>"，后跟具体的命令，类似于vim的快捷键模式，命令一般为对于`/`命令的首字母，参考 [OpenCode TUI 命令](#OpenCode%20TUI%20命令)。

### 使用建议

- 多使用 <mark style="background: #FFB86CA6;">@</mark> 引用文件，提高准确率
- 复杂任务先用计划模式，使用 Tab 键切换模式。
- 小步迭代，不要一次做太复杂。
- 重要操作前确保 Git 已提交。

## OpenCode 规则（AGENTS.md）

在OpenCode中规则是控制AI行为的核心机制。

可以通过提供`AGENT.md`文件，为OpenCode提供自定义指令，让他按照需要的项目规范进行开发。

### AGENTS.md的作用

AGENTS.md相当于给AI提供开发规范，通常用于：

- 定义项目结构；
- 约束代码风格；
- 规范开发流程；
- 指导AI如何执行任务；

上述内容会被加入LLM的上下文中。

### AGENT.md创建及实例

在OpenCode TUI中执行 `/init`

OpenCode会执行以下操作：

- 扫描项目结构；
- 分析代码组织形式；
- 自动生成AGRNTS.md；

```AGENT.md
# 项目说明

这是一个基于xxx的项目，主要完成xxxx

## 项目结构

- doc/ 规范文档
- prj/ 工程目录
  - workspace_0/ 工作空间0
	  - drivers/ 驱动代码
	    - src/ 源码
	    - inc/ 头文件
	  - app/ 应用代码
	  - common/ 公共文件目录
  - workspace_1/ 工作空间1
- temp/ 临时目录
  
  
  
## 代码规范

- 严格使用驼峰命名方式对函数及变量进行命名
- 公共代码放置于prj/common文件夹下
- 对所有的函数必须添加必要注释
  
## 开发约定

- API必须统一做错误处理
- 程序结束必须做资源释放
```

AGENTS.md本质是规范AI的工作过程及产出。

### 兼容性

OpenCode兼容ClaudeCode的规则体系：

- 项目规则：ClAUDE.md
- 全局规则：~/.claude/CLAUDE.md
- 技能目录：~/.claude/skills

可以通过以下选项进行关闭兼容
```
export OPENCODE_DISABLE_CLAUDE_CODE=1
```

### 规则加载优先级

OpenCode启动时按以下顺序加载：

1. 当前目录下查找AGENTS.md；
2. 如果没有，则查找CLAUDE.md；
3. 全局规则~/.config/opencode/AGENTS.md；
4. Claude全局规则~/.claude/CLAUDE.md；

<mark style="background: #FFB86CA6;">注意</mark>

- 当前目录下的AGENTS.md优先级最高；
- 同级只会使用第一个匹配文件。

#AGENTS

# 参考

[https://www.runoob.com/opencode/opencode-tutorial.html](https://www.runoob.com/opencode/opencode-tutorial.html)