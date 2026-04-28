Ollama是一个开源的、轻量级的本地大语言模型运行和管理平台，它允许用户在自己的设备上轻松运行、创建和分享各种大语言模型，当然也可以使用API在本地调用云端模型使用。

[Ollama官网(https://ollama.com/)](https://ollama.com/)

# 安装Ollama

在`Windows PowerShell`中运行：

```powershell
irm https://ollama.com/install.ps1 | iex
```

# 配置云端模型

## 登录ollama

```powershell
ollama signal
```

这里需要注意，需要把上面中的 `https://ollama.com/connect?name=.....`链接复制到浏览器打开，然后登录你的账号，确认授权：

![](image/Ollama安装使用/IMG-20260427104346964.png)

## 创建API-Keys

在以下地址创建ollama key：[https://ollama.com/settings/keys](https://ollama.com/settings/keys)

## Ollma运行云端模型

```powershell
> ollama
Ollama 0.21.2

▸ Chat with a model (gpt-oss:120b-cloud)
    Start an interactive chat with a model

  Launch OpenClaw (install)
    Personal AI with 100+ skills

  Launch Claude Code
    Anthropic's coding tool with subagents

  Launch OpenCode (not installed)
    Anomaly's open-source coding agent

  More...
    Show additional integrations


↑/↓ navigate • enter launch • → configure • esc quit
```

选择 `Chat with a model (gpt-oss:120b-cloud)`

```powershell
Connecting to 'gpt-oss:120b-cloud' on 'ollama.com' ⚡
>>> Send a message (/? for help)
```

之后即可使用。

## 使用claude code调用Ollama云端模型

### 快速配置

```powershell
ollama launch claude
```

### 使用模型直接运行

```powershell
ollama launch claude --model kimi-k2.5:cloud
```

## 可用的云端模型

ollama远程可用模型查询 [https://ollama.com/search?c=cloud](https://ollama.com/search?c=cloud)

| 模型名称                         | 说明          |
| :--------------------------- | :---------- |
| minimax-m2:cloud             | Text        |
| minimax-m2.1:cloud           | Text        |
| minimax-m2.5:cloud           | Text        |
| minimax-m2.7:cloud           | Text        |
| qwen3.5:cloud                | Text, Image |
| qwen3.5:397b-cloud           | Text, Image |
| qwen3-coder-next:cloud       | Text        |
| qwen3-next:80b-cloud         | Text        |
| glm-4.6:cloud                | Text        |
| ministral-3:3b-cloud         | Text, Image |
| ministral-3:8b-cloud         | Text, Image |
| ministral-3:14b-cloud        | Text, Image |
| devstral-small-2:24b-cloud   | Text, Image |
| devstral-2:123b-cloud        | Text        |
| nemotron-3-super:cloud       | Text        |
| nemotron-3-nano:30b-cloud    | Text        |
| gemini-3-flash-preview:cloud | Text, Image |
| deepseek-v3.2:cloud          | Text        |
| kimi-k2-thinking:cloud       | Text        |
| gpt-oss:20b-cloud            | Text        |
| gpt-oss:120b-cloud           | text        |

# 参考

[Ollama-Claude Code doc](https://docs.ollama.com/integrations/claude-code)
[Claude Code 直连 Ollama / LM Studio：本地、云端开源模型都能跑](https://zhuanlan.zhihu.com/p/2010741899133198392)