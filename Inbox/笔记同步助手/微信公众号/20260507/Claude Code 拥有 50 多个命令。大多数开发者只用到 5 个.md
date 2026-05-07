---
author: dev
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI0NDQ0ODU3MA==&mid=2247547119&idx=1&sn=617f61705a175d3f54eaa5d4d1fae643&chksm=e8319e4002048ae231abbeb739d72c51231928e036f797ff7c3b5264ce64d1770482ee70e185&mpshare=1&scene=1&srcid=0501YhM5BlSLi9U90wJFq9RS&sharer_shareinfo=06854d6dc8f76d05ac71fbfee2ecd325&sharer_shareinfo_first=06854d6dc8f76d05ac71fbfee2ecd325#rd
saved: 2026-05-07 09:42:02
tags:
  - 笔记同步助手
id: fe0d8d11-ad53-4e04-acc0-f546044313bd
---

公众号名称：大迁世界

作者名称：dev

发布时间：2026-04-09 19:00

说句扎心的话：**Claude Code 拥有超过 50 个指令，但绝大多数开发者只会在那儿干巴巴地敲其中的 3 到 5 个。**

剩下的指令就那么冷冰冰地躺在 `/help` 文档里吃灰。它们原本能让你的生产力原地起飞 10 倍，前提是——你得知道它们的存在。然而，根据 2026 年 3 月的最新数据，那些精通 15 个以上指令的硬核开发者，交付速度比普通人快 3 到 4 倍。尽管如此，很多人依然只是把它当成“终端里的 ChatGPT”来糟蹋。

![[Inbox/笔记同步助手/images/98bd6c405b202f539289c686eadf4a3b_MD5.jpg]]

### 指令的三重境界：看透工具的底层逻辑

在深挖具体指令前，你必须理解 Claude Code 的操作逻辑。它不仅仅是一个对话框，而是一套立体的操作系统。换句话说，它分为三类：

-   **第一类：CLI 指令（启动配置）：** 在终端开启 Claude 时注入的初速度。
    
-   **第二类：斜杠指令（会话核心）：** 在交互过程中输入的 `/` 招式。
    
-   **第三类：快捷键（神经反射）：** 真正的高手从来不浪费时间打完整个单词。
    

### 第一部分：保命级核心指令（每日必用 Top 10）

这 10 个指令是你的职业底座。不管你是刚入行的新手还是老油条，先把它练成肌肉记忆。

#### 1\. `/init` —— 赋予项目“灵魂”

它的核心意义在于创建一个 `CLAUDE.md` 文件。这是项目的“长期记忆”。与其一遍遍跟 AI 解释你的代码规范，不如一次性写进这里。

```
# CLAUDE.md

## Authentication
- Use JWT tokens, not sessions
- Store in httpOnly cookies

## Testing
- Write tests for all API endpoints
- Use Jest, not Mocha

## Error Handling
- Return structured errors: { error: string, code: number }
```

#### 2\. `/compact` —— 内存大清理

当上下文占用过高时，它能精准总结核心决策。2026 年 2 月更新后，这个操作现在是**秒级完成**。建议在占用率达到 70-80% 时主动执行。

#### 3\. `/clear` —— 彻底重启

切换完全不同的任务时，用它清空历史。不过要注意，这也会擦除该目录下的指令历史。

#### 4\. `/model` —— 大脑切换

在 Sonnet（平衡）、Opus（巅峰）和 Haiku（极速）之间横跳。

```
/model sonnet          # 切换到 Sonnet 4.6 (日常利器)
/model opus            # 切换到 Opus 4.6 (架构杀手)
/model haiku           # 切换到 Haiku 4.5 (体力活专家)
```

#### 5\. `/cost` —— 实时算账

别等月底看到账单才心惊肉跳。实时监控你的 Token 消耗，毕竟 Opus 虽好，烧起钱来也挺疼。

```
Session cost: $2.47
Input tokens: 48,392
Output tokens: 12,847
Model: claude-sonnet-4-20250514
```

#### 6\. `/context` —— 进度条预警

显示当前上下文百分比。如果你发现 Claude 开始“装傻”或记性变差，赶紧看看这里。

```
Context usage: 67% (134,400 / 200,000 tokens)
```

#### 7\. `/diff` —— 拒绝开盲盒

查看 Claude 刚才到底改了哪行代码。提交前的代码评审，全靠它了。

```
/diff              # Show all changes
/diff src/auth.ts  # Show changes to specific file
```

#### 8\. `/help` —— 实时说明书

指令会随版本更新而变，这里是你唯一的权威信源。

#### 9\. `/memory` —— 现场改规矩

无需离开会话，直接编辑 `CLAUDE.md`。此外，你可以使用快捷语法：

```
# Use async/await for all database queries
```

以 `#` 开头的笔记会直接追加到记忆文件中。

#### 10\. `/resume` —— 续上那段缘

加载并继续之前的对话。你可以对 Claude 说：“帮我找找去年 12 月那个会话”，它真的能帮你翻出来。

​

### 第二部分：高阶进阶指令（拉开差距的杀手锏）

#### 11\. `/btw` —— 不打断的插嘴

这是 2026 年 3 月最出圈的神技。在 Claude 埋头重构时，你可以突然插个题外话，问完它会自动回到刚才的任务。

#### 12\. `/fast` —— 极速模式

开启 API 优化设置。同样的 Opus 4.6，开启后响应速度像打了鸡血一样快。

#### 13\. `/plan` —— 谋定而后动

进入只读规划模式。Claude 会先给你看方案，你点个头，它才敢动手。这种“三思而后行”的逻辑能预防 90% 的事故。

#### 14\. `/todos` —— 永不消失的任务书

一个能跨会话存在的清单。即便你关掉 Claude，未完成的工作依然在那儿盯着你。

#### 15\. `/simplify` —— 2026 全新代码评审

取代了过时的 `/review`。它会调用三个并行代理，从安全性、性能、规范性三个维度对你的代码进行全方位的“降维打击”。

​

### 第三部分：CLI 启动标志（真正高手的暗号）

这些标志控制着 Claude 启动时的初始状态。

-   \*\*`claude --print "..."`\*\*：一闪电战。执行单个查询后立即退出，非常适合脚本编写。
    
    ```
    result=$(claude --print "Generate a random UUID")
    echo $result
    ```
    
-   \*\*`claude -c`\*\*：一键接续上次在该目录下的事业。
    
-   \*\*`--agents`\*\*：启动时预设子代理。
    
    ```
    claude --agents '{
      "test-writer": {
        "role": "Write comprehensive Jest tests",
        "model": "claude-sonnet-4"
      }
    }'
    ```
    
-   \*\*`--dangerously-skip-permissions`\*\*：⚠️ 仅在受信任的容器环境（如 Docker/CI）中使用。它会跳过所有审批，开启全自动狂飙模式。
    

### 第四部分：快捷键（你的效率倍增器）

-   \*\*`Shift + Tab`\*\*：在正常、自动接受、规划模式之间一键循环。
    
-   \*\*`Esc Esc`\*\*：瞬间呼出回滚菜单。你可以选择只回滚代码而保留对话，或者反之。
    
-   \*\*`! + 命令`\*\*：在会话中直接执行 Bash 指令。例如 `! git status`。
    
-   \*\*`@ + 路径`\*\*：文件路径自动补全。
    
-   \*\*`Ctrl + T`\*\*：开关任务列表。
    

### 第五部分：那些被藏起来的“禁术”

-   \*\*`/vim`\*\*：给输入框加上 Vim 键位。是的，你甚至可以在 Prompt 里用 `h/j/k/l` 导航和编辑。
    
-   \*\*`/remote-control`\*\*：在手机上控制你电脑里的 Claude。哪怕在下班路上，也能远程操控它修个 Bug。
    
-   \*\*`/usage-report`\*\*：生成月度分析报告，详细拆解你的时间都花在哪了，Token 都烧在哪了。
    

### 第六部分：实战全自动化工作流

#### 场景：长达一天的复杂重构

1.  **开启规划模式**`claude` -> `Shift+Tab` (进入 Plan 模式)
    
2.  **描述重构**“将 Auth 模块从 session 模式改为 JWT，并使用 bcrypt 加密密码。”
    
3.  **监控并压缩**`/context` 查看占用。当达到 70% 时执行：`/compact retain auth patterns and migration strategy`
    
4.  **评审与提交**`/diff` 查看改动。`/simplify` 进行最终质量检查。`! git add .` -> `! git commit -m "feat: jwt migration"`
    
5.  **导出归档**`/export` 将这套神操作存为团队的知识资产。
    

### 最后

Claude Code 的 50 多个指令，你不需要一天全学会。然而，如果你能每周强迫自己加一个新招式，一个月后你就能把同龄人远远甩在身后。

**最后：**

**[精通 React 面试：从零到中高级(针对面试回答)](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NDQ0ODU3MA==&action=getalbum&album_id=4438329314299920385&from_itemidx=1&from_msgid=2247546427&sessionid=#wechat_redirect)**

**[CSS终极指南](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NDQ0ODU3MA==&action=getalbum&album_id=4274694215210369041&from_itemidx=1&from_msgid=2247538974&sessionid=#wechat_redirect)**

**[Vue 设计模式实战指南](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NDQ0ODU3MA==&action=getalbum&album_id=4198498164833402888&from_itemidx=1&from_msgid=2247535710#wechat_redirect)**

[20个前端开发者必备的响应式布局](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NDQ0ODU3MA==&action=getalbum&album_id=4143266676156547085&from_itemidx=1&from_msgid=2247533575&sessionid=#wechat_redirect)

**[深入React:从基础到最佳实践完整攻略](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NDQ0ODU3MA==&action=getalbum&album_id=4062982567140671496&from_itemidx=1&from_msgid=2247531462#wechat_redirect)**

**[python 技巧精讲](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NDQ0ODU3MA==&action=getalbum&album_id=3973477600504201220&from_itemidx=1&from_msgid=2247529168&sessionid=#wechat_redirect)**

**[React Hook 深入浅出](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI0NDQ0ODU3MA==&action=getalbum&album_id=3660558698339991553&from_itemidx=1&from_msgid=2247523919&scene=173&subscene=91&sessionid=1728003498&enterid=1728004916&count=3&nolastread=1#wechat_redirect)**

**[CSS技巧与案例详解](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&__biz=MzI0NDQ0ODU3MA==&scene=2&album_id=3545535769677725703&count=3&uin=&key=&devicetype=iMac+MacBookPro18%2C3+OSX+OSX+14.3+build\(23D56\)&version=13080812&lang=zh_CN&nettype=WIFI&ascene=2&fontScale=100)**

**[vue2与vue3技巧合集](https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&__biz=MzI0NDQ0ODU3MA==&scene=1&album_id=2509459125236416515&count=3#wechat_redirect)**

  

---

![[Inbox/笔记同步助手/images/1f008eadc671d6d144671a0fcc6a6970_MD5.jpg|cover_image]]

原创 dev 大迁世界