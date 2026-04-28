
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
## 运行claude code

### 快速配置

```powershell
ollama launch claude
```

### 使用模型直接运行

```powershell
ollama launch claude --model kimi-k2.5:cloud
```

# 参考

[Ollama-Claude Code doc](https://docs.ollama.com/integrations/claude-code)
[Claude Code 直连 Ollama / LM Studio：本地、云端开源模型都能跑](https://zhuanlan.zhihu.com/p/2010741899133198392)