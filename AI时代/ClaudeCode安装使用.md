# 安装claude

在Windows Powershell中执行：

```bash
irm https://claude.ai/install.ps1 | iex
```

安装完成后的claude在`C:\Users\asus\.local\bin`目录下存在`claude.exe`则说明安装成功；

# 配置环境变量

将`claude.exe`的文件位置加入到环境变量中：

![](image/ClaudeCode安装使用/IMG-20260423171623732.png)

之后确定并退出命令行，重新打开命令行输入

```bash
PS D:\> claude --version
2.1.114 (Claude Code)
PS D:\> 
```

环境变量配置完成。

# 绕过校验

安装并配置好环境变量后，初次使用 Claude Code ，可能会强制要求登录 Anthropic 账户，需要通过修改配置，来绕过校验，在C盘用户目录下（C:\Users\\%USERNAME%\\.claude.json）找到`claude.json`，加入以下代码:

```json
"hasCompletedOnboarding": true,
```

保存文件，然后在新终端中重新运行 `claude`。

注意：最后需要加上英文版的`,`。

再次执行`claude`即可正常启动：

![](image/ClaudeCode安装使用/IMG-20260423171623710.png)

# 切换国产大模型
## 阿里云（百炼平台）

进入`阿里云百炼`：
官网：[https://bailian.console.aliyun.com/](https://bailian.console.aliyun.com/)

### API-Key创建

首先需要在在阿里云百炼中 [创建API-Key](https://bailian.console.aliyun.com/cn-beijing?spm=a2c4g.11186623.0.0.7c6c5ec6Q6vx5O&tab=model#/api-key&userCode=okjhlpr5)；

### 环境变量配置

在 Windows 中，可以通过 CMD 或 PowerShell 将阿里云百炼提供的 Base URL 和 [API Key](https://help.aliyun.com/zh/model-studio/get-api-key) 设置为环境变量。

#### CMD

```cmd
# 用百炼 API Key 替换 YOUR_DASHSCOPE_API_KEY
setx ANTHROPIC_API_KEY "YOUR_DASHSCOPE_API_KEY"
setx ANTHROPIC_BASE_URL "https://dashscope.aliyuncs.com/apps/anthropic"
setx ANTHROPIC_MODEL "qwen3.6-plus"
```

查看参数是否设置成功

```cmd
echo %ANTHROPIC_API_KEY%
echo %ANTHROPIC_BASE_URL%
echo %ANTHROPIC_MODEL%
```

#### PowerShell

```powershell
# 用百炼 API Key 替换 YOUR_DASHSCOPE_API_KEY
[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "YOUR_DASHSCOPE_API_KEY", [EnvironmentVariableTarget]::User)
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://dashscope.aliyuncs.com/apps/anthropic", [EnvironmentVariableTarget]::User)
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "qwen3.6-plus", [EnvironmentVariableTarget]::User)
```

配置完成后重启PowerShell。

查看参数是否设置成功：

```powershell
echo $env:ANTHROPIC_API_KEY
echo $env:ANTHROPIC_BASE_URL
echo $env:ANTHROPIC_MODEL 
```
### 模型切换

#### 方式1（进入claude后切换）

进入claude后输入
```plaintext
/model [模型名称]
```

#### 方式2（命令行直接切换进入）

```plaintext
claude --model qwen3.6-plus
```

具体的模型名称可查看阿里云百炼中的模型名称。

### 参考

[阿里云百炼Claude Code接入参考](https://help.aliyun.com/zh/model-studio/claude-code?spm=a2c4g.11186623.help-menu-2400256.d_0_4_2.9c8a6fd1PDWZcP)

## 腾讯云（Tencent Cloud）

### API-Key创建

首先需要在在 [腾讯混元API管理](https://hunyuan.cloud.tencent.com/#/app/apiKeyManage) 中创建API-Key；

### 环境变量配置

#### PowerShell

```PowerShell
# 用腾讯混元 API Key 替换 YOUR_DASHSCOPE_API_KEY

[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "YOUR_DASHSCOPE_API_KEY", [EnvironmentVariableTarget]::User)

[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.hunyuan.cloud.tencent.com/anthropic", [EnvironmentVariableTarget]::User)

[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "hunyuan-2.0-thinking-20251109", [EnvironmentVariableTarget]::User)
```

配置完成后重启`PowerShell`即可使用。
### token用量查询

[腾讯云-资源包管理](https://console.cloud.tencent.com/hunyuan/packages)

### 参考

[混元 Anthropic API 兼容接口相关调用示例](https://cloud.tencent.com/document/product/1729/127293)

## 国家超算平台

#### API-Key创建

首先需要在在 [国家超算平台API管理]([https://hunyuan.cloud.tencent.com/#/app/apiKeyManage](https://www.scnet.cn/ui/console/index.html#/llm/apikeys)) 中创建API-Key；

### 环境变量配置

#### PowerShell

```PowerShell
# 用国家超算平台 API Key 替换 YOUR_DASHSCOPE_API_KEY

[Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "YOUR_DASHSCOPE_API_KEY", [EnvironmentVariableTarget]::User)

[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL", "https://api.scnet.cn/api/llm/anthropic", [EnvironmentVariableTarget]::User)

[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL", "MiniMax-M2.5", [EnvironmentVariableTarget]::User)
```

配置完成后重启`PowerShell`即可使用。

### 参考

[国家超算平台第三方工具接入](https://www.scnet.cn/ac/openapi/doc/2.0/moduleapi/tutorial/callbytools_claudecode.html)

# 卸载claude code

## 步骤1 删除npm包

```bash
npm uninstall -g @anthropic-ai/claude-code  
npm uninstall -g https://gaccode.com/claudecode/install
```

## 步骤2 删除配置文件

```bash
# 删除用户配置和状态
rm -rf ~/.claude
rm ~/.claude.json

# 删除特定用于项目的配置（在您的项目目录行）
rm -rf .claude
rm -f .mcp.json

# 删除用户设置和状态
Remove-Item -Path "$env:USERPROFILE\.claude" -Recurse -Force
Remove-Item -Path "$env:USERPROFILE\.claude.json" -Force

# 删除特定于项目的设置（从您的项目目录运行）
Remove-Item -Path ".claude" -Recurse -Force
Remove-Item -Path ".mcp.json" -Force
```


# 免费Token

[2026 大模型 API 免费额度汇总20260317](https://cloud.tencent.com/developer/article/2626756?cps_key=1d358d18a7a17b4a6df8d67a62fd3d3d)