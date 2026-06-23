---
author: 狂写教程的椰卷卷
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI1ODQ1NTc3Mg==&mid=2247483814&idx=1&sn=ecbf1035a1728d4b202451cf1ee97a7d&chksm=eb41391f4cc399cad582e43b4c179d0c44e2e387c2c217e9df141d8923b08f7ae145f1299178&mpshare=1&scene=1&srcid=0508rppz8a6irtOlKgfQGxbp&sharer_shareinfo=b12cfd8c0efc3721cdff2f4fafb4526d&sharer_shareinfo_first=b12cfd8c0efc3721cdff2f4fafb4526d#rd
saved: 2026-05-08 19:02:09
tags:
  - 笔记同步助手
id: 40f129aa-46a0-4486-899d-29c40d408f61
---

公众号名称：椰卷卷

作者名称：狂写教程的椰卷卷

发布时间：2026-03-25 21:38

先说结论：Claude Code 不只是"写代码的工具"，而是一个能理解你整个项目的 AI 伙伴，是可以用自然语言进行交互的。但很多人被复杂的命令行界面劝退了——本教程要做的，就是让它变得简单、好用、零门槛。

为什么这套配置值得你花 30 分钟？

### 你可能听说过 Claude Code，但有以下顾虑：

-   ❌ 命令行界面太复杂，看不懂怎么用
    
-   ❌ 配置太麻烦，API Key、端口、配置文件...头大
    
-   ❌ 这是给专业程序员用的吧？我只是想辅助用智能体
    
-   ❌ 试过各种教程，但总是卡在某个步骤就放弃了
    

  

### 这套配置能解决什么？

Claude Code + CC Switch + Trae IDE 的组合，本质上是把强大的 AI 编程助手变成像聊天一样简单：

​

| 你能做什么 | 传统方式 | 这套配置 |
| --- | --- | --- |
| **理解代码** | 自己逐行看文档、查资料 | 选中代码→自然语言→AI 即时解释 |
| **排查错误** | 搜索引擎 + 论坛 + 试错 | AI 读取上下文→精准定位问题 |
| 自动化重复工作 | 手动复制粘贴→耗时耗力 | 告诉 AI 要做什么→它写脚本帮你自动完成 |
| 整理笔记/文献 | 手动摘录→分类→总结 | 丢给 AI→通过skill自动提取重点→生成结构化笔记并存在本地 |

  

### 为什么是这三个工具？

Claude Code CLI：Anthropic 官方出品，让 AI 直接理解你的项目结构，不只是"写代码"

CC Switch：一键存储切换你的大模型 API Key，不用每次都重改 setting.json

Trae IDE：完美支持 Claude Code 插件，界面友好零成本

  

### 适合谁？

-   想用 AI Vibecoding，但被命令行界面劝退的开发者
    
-   学生党/个人开发者，想要高性价比的 AI 智能体/编程方案（本教程支持阿里云等平替 API）
    
-   已经听过 Claude Code，但不知道如何实际使用的观望者
    

  

🕐 时间成本：

-   阅读 + 操作：​
    
    约 30-40 分钟
    
-   一次性配置，长期受益

---

## 📚 教程内容概览

本教程将指导你从零搭建一套稳定的本地 Claude AI 编程环境：

-   ① 前置环境准备（Git + Node.js）
    
-   ② 获取大模型 API Key（支持阿里云等平替方案）
    
-   ③ 安装 Claude Code CLI
    
-   ④ 配置并启动 Claude Code 本地服务
    
-   ⑤ 安装与配置 CC Switch（可视化管理工具）
    
-   ⑥ Trae IDE 插件安装与对接
    
-   ⑦ 全流程功能验证
    
-   ⑧ 常见问题排查
    

⚠️ 重要前置说明：

-   适配系统：​
    
    macOS、Windows 10/11
    
-   费用说明：​
    
    本教程支持阿里云 Coding Plan 等平替方案，成本可控
    

---

## 一、前置环境准备（Git + Node.js）

### 1\. 安装 Git

Git 是版本控制工具，部分依赖安装可能需要用到。

macOS 用户

```
打开终端，输入

git --version

若未安装会自动弹出安装提示，按提示完成安装即可
或从 Git 官网下载安装包手动安装
```

Windows 用户

1.  ```
    从 Git 官网
    下载最新版安装包
    双击安装包，按默认选项一路点击「Next」完成安装
    打开终端（macOS）/CMD（Windows），输入以下命令，能正常输出版本号即为安装成功：
    git --version
    ```
    

### 2\. 安装 Node.js

Node.js 是运行 Claude Code CLI 的必需环境。

**下载安装：**从 Node.js 官网下载 LTS（长期支持）版本，双击安装包按默认选项完成安装。

**验证安装：**打开终端 / CMD，输入以下命令，能正常输出版本号即为安装成功：

​

```
node -v
npm -v
```

  

---

## 二、获取大模型 API Key

这里是没有 Claude 账号的平替办法（现在阿里云、字节方舟、minimax 等大模型厂商都有对应的 coding plan，大家可以按需选择购买，我这里以阿里云 coding plan 为例）

https://help.aliyun.com/zh/model-studio/coding-plan

**先存下复制粘贴你的 API Key 和 Base URL**（Anthropic 这个），后续再用

![[../../../../images/c2632eb88241876901cf2b9dc51ca7bf_MD5.jpg]]

  

---

## 三、安装 Claude Code CLI

Claude Code CLI 是官方提供的本地服务工具，是整套环境的核心。

1.  打开终端（macOS）/CMD（Windows，管理员权限运行），输入以下命令全局安装：
    

```
npm install -g @anthropic-ai/claude-code
```

3.  验证安装成功：
    

```
claude --version
```

---

## 四、安装与配置 CC Switch

CC Switch 是专为 Claude Code 开发的可视化配置工具，无需手动修改配置文件，一键切换 API 密钥、模型、服务地址，还能一键启停本地服务。

### 1\. 下载安装包

访问 CC Switch 官方仓库，进入「Releases」页面，下载对应系统的安装包：

![[../../../../images/d0b6642ebf4060ce2c6a1782d85e9658_MD5.jpg]]

  

### 2\. 解决系统拦截问题（macOS 用户重点看）

macOS 用户首次打开会遇到「Apple 无法验证开发者」的拦截弹窗，按以下步骤放行：

1.  先点击弹窗的「完成」，关闭提示
    
2.  打开 macOS「系统设置」→「隐私与安全性」
    
3.  下滑到「安全性」区域，找到「"CC Switch" 已被阻止使用」的提示，点击「仍要打开」
    
4.  输入你的开机密码确认，再次双击打开软件即可正常启动
    

### 3\. CC Switch 基础配置

在 CC Switch 里编辑供应商：

1.  填写刚刚你存下的：
    

-   API Key
    
-   Base URL
    

![[../../../../images/e745fb7a3fda43651674fdd11c0983fa_MD5.jpg]]

  

2.  这里填写你购买的 coding plan 支持的模型，填写之后保存即可（模型名称一定要填写对，不然会报错）
    

![[../../../../images/2d762cbc67390ba230f6adc3f8e29229_MD5.jpg]]

  

3.  打开终端，直接输入：
    

```
claude --version
```

出现这个界面之后，可以和它互动一下（例如发一个 hi），成功之后会回复给你消息。

![[../../../../images/926750360bb4e8ad9179e7c814930d11_MD5.jpg]]

  

---

## 五、Trae CN IDE 插件安装与对接

### 1\. 安装 TraeCN IDE

从 Trae IDE 官方网站下载最新版安装包，按系统指引完成安装。

### 2\. 安装 Claude Code 官方插件

1.  打开 TraeCN，点击左侧边栏的「扩展」图标
    
2.  在搜索框输入 `Claude Code`，找到 Anthropic 官方发布的插件，点击「安装」
    
3.  安装完成后，重启 Trae IDE，左侧边栏会出现 Claude Code 的图标，即为安装成功
    

![[../../../../images/00079e1134ede7f22939690a655d19d4_MD5.jpg]]

一开始可能会显示这个界面，但是如果你完成了 CC Switch 的配置并成功启动 Claude 之后，它会自动连接你的本地 Claude，如果不成功关闭后重启。

![[../../../../images/dc18c9030f7abe55d8704b7ada4b0c2e_MD5.jpg]]

最后显示这个界面就是成功接上 IDE，就不用再每次打开终端进行命令行交互了。

![[../../../../images/ba4a3fc16f52dba443bd7812c55a33d2_MD5.jpg]]

\[往期推荐\]

就喜欢说点不一样的：  
①[我为什么越听播客越焦虑](https://mp.weixin.qq.com/s?__biz=MzI1ODQ1NTc3Mg==&mid=2247483690&idx=1&sn=e25b1b122306d65004213542d8305ff0&scene=21#wechat_redirect)

### ②现在害怕【AI写作】，或许就像电脑时代害怕【复制粘贴】

努力搬运优质精神食粮（英专生再就业ver.）：  
① [\[论文翻译\] Less is more! AI也要投喂高质量数据](https://mp.weixin.qq.com/s?__biz=MzI1ODQ1NTc3Mg==&mid=2247483698&idx=1&sn=ee0be5ee8f3be702e31b10abc6ea0b03&scene=21#wechat_redirect)

### ②AI时代，如何脱颖而出【译】

J人的自我管理：  
[当我用Obsidian搭了人生管理系统](https://mp.weixin.qq.com/s?__biz=MzI1ODQ1NTc3Mg==&mid=2247483731&idx=1&sn=e0191399cdd45689ec8e551b90f030a8&scene=21#wechat_redirect)

玩点新鲜的AI：

### ①靠和豆包聊我的梦，我写出了人生第一个漫剧故事大纲

  

### ②不是，那个天天挂在老板嘴边的小龙虾到底是什么

  

喜欢就关注我吧～～

  

---

![[../../../../images/8e488017f6a152bd614eaa2e86df7a45_MD5.jpg|cover_image]]

原创 狂写教程的椰卷卷 椰卷卷

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/a63eafbe_1778238128391?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI1ODQ1NTc3Mg%3D%3D%26mid%3D2247483814%26idx%3D1%26sn%3Decbf1035a1728d4b202451cf1ee97a7d%26chksm%3Deb41391f4cc399cad582e43b4c179d0c44e2e387c2c217e9df141d8923b08f7ae145f1299178%26mpshare%3D1%26scene%3D1%26srcid%3D0508rppz8a6irtOlKgfQGxbp%26sharer_shareinfo%3Db12cfd8c0efc3721cdff2f4fafb4526d%26sharer_shareinfo_first%3Db12cfd8c0efc3721cdff2f4fafb4526d%23rd&s=obsidian)