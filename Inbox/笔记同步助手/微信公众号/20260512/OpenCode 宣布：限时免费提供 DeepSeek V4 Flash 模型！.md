---
author: 呱呱乐
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzA4NjQ4NzgyOQ==&mid=2447660240&idx=1&sn=6557aa3b4fe36e0fbbefe7c929e865c5&chksm=8a9806a7735bdb6194b841addc06cfe99414304c60cf7ded2bf8e43d94cafbf0c4425e27426d&mpshare=1&scene=1&srcid=0512VrChhSBTYlrvutC8Olw9&sharer_shareinfo=b270cd0f5625ab56dc07064774a3d0f7&sharer_shareinfo_first=b270cd0f5625ab56dc07064774a3d0f7#rd
saved: 2026-05-12 12:01:04
tags:
  - 笔记同步助手
id: e6c0a8c1-8b4e-42d5-99fe-e25bfa3636c3
---

公众号名称：赛博小青蛙

作者名称：呱呱乐

发布时间：2026-05-12 10:00

前两天刚写过 OpenCode Go 的订阅，这两天OpenCode Zen 又搞了个大的——**DeepSeek V4 Flash 直接上了 Zen 的免费模型列表。**

不用订阅，不用付费，注册就能白嫖。

![[Inbox/笔记同步助手/images/722d32cc634c1367693f648aa68575c9_MD5.jpg]]

  

## Zen 上的免费模型一览

OpenCode Zen 有 6 个限时免费模型，官方说法是"限时免费提供，团队正在利用这段时间收集反馈并改进模型"：https://opencode.ai/docs/zh-cn/zen/

​

| 免费模型 | 说明 | 社区评价 |
| :-- | :-- | :-- |
| **DeepSeek V4 Flash Free** | 284B 参数 MoE，1M 上下文 | **Fast and smart. The best.** |
| Big Pickle | OpenCode 的"隐身模型"，0.2M 上下文 | Good - Fast and smart |
| MiniMax M2.5 Free | MiniMax 旗舰的免费版 | 对 OpenCode 支持一般 |
| Ring 2.6 1T Free | 轻量快速 | 目前不太稳定 |
| Nemotron 3 Super Free | NVIDIA 开源模型 | 可用 |

  

![[Inbox/笔记同步助手/images/1551867e77d68d8cb55f1f90e090555d_MD5.jpg]]

  

**DeepSeek V4 Flash Free 是这批免费模型里最能打的。** 1M 上下文窗口，SWE-Bench Verified 79.0，这个分数放在半年前是顶级闭源模型的水平。现在白送，特别适合养虾和养马。

​

```
# BaseURL
https://opencode.ai/zen/v1
```

**注意**：免费期间数据可能会被用于改进模型。NVIDIA 的 Nemotron 还会记录提示词和输出内容。敏感代码别往上扔。

​

## 白嫖和付费，怎么选

简单说：

**先试 Zen 免费版。** 注册就能用 DeepSeek V4 Flash Free，零成本验证它能不能胜任日常工作。如果免费额度够用，就这样，不用花钱。

**不够再上 Go。** Go 的优势是额度大、模型多。12 个模型随便切，Flash 3 万次/5 小时管饱。Zen 免费版限速或限额了，切 Go 继续干。**首月 $5，试错成本较低，而且现在不用抢。**

两条路不冲突。Zen 免费 + Go 订阅组合起来，日常编码基本不用再愁额度。

Zen 注册地址：opencode.ai/auth\[1\]

Go 订阅地址：opencode.ai/zh/go\[2\]

​

#### 引用链接

`[1]` opencode.ai/auth: _https://opencode.ai/auth_  
`[2]` opencode.ai/zh/go: _https://opencode.ai/zh/go_

  

---

![[Inbox/笔记同步助手/images/b08b3ecef7b58d0be7377c0530e4a240_MD5.jpg|cover_image]]

Original 呱呱乐 赛博小青蛙

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/37993c21_1778558463807?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzA4NjQ4NzgyOQ%3D%3D%26mid%3D2447660240%26idx%3D1%26sn%3D6557aa3b4fe36e0fbbefe7c929e865c5%26chksm%3D8a9806a7735bdb6194b841addc06cfe99414304c60cf7ded2bf8e43d94cafbf0c4425e27426d%26mpshare%3D1%26scene%3D1%26srcid%3D0512VrChhSBTYlrvutC8Olw9%26sharer_shareinfo%3Db270cd0f5625ab56dc07064774a3d0f7%26sharer_shareinfo_first%3Db270cd0f5625ab56dc07064774a3d0f7%23rd&s=obsidian)