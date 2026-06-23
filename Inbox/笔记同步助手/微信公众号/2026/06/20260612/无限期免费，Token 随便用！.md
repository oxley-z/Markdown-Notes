---
author: 小 G
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzAxOTcxNTIwNQ==&mid=2457994301&idx=1&sn=f5dd955f69604da512c22b5a7fc1c34f&chksm=8d48a403c5e44968a830698c988100658c2a221dc262bf1400f634e52b1f3bbeca9e313b4650&mpshare=1&scene=1&srcid=0611Wj2CrFqJUwmeR88Vy0NL&sharer_shareinfo=d6ee3a7b38afda4f78223018b952ec89&sharer_shareinfo_first=d6ee3a7b38afda4f78223018b952ec89#rd
saved: 2026-06-12 09:12:37
tags:
  - 笔记同步助手
id: e8eda7ee-3650-4f4c-b1fc-4a7f8ff2cdbb
---

公众号名称：GitHubDaily

作者名称：小 G

发布时间：2026-06-11 17:05

今年 Agent 工具一个接地一个火，越来越多人试图把 AI 接入到日常的工作流中。

但随便挂几个自动 Agent 任务、生成几次图片或视频，Token 的消耗就像流水一样。

一个月 20 美刀的普通订阅根本不够用，而 API 账单更是高到让人玩不起。

最近发现，一家名叫 **Agnes AI** 的模型公司，把这个门槛直接推倒了。

6 月 1 日，Agnes AI 宣布无限期免费开放旗下核心全模态模型 API。

包括文本模型 Agnes-2.0-Flash 、图片模型 Agnes-Image-2.1-Flash，以及视频模型 Agnes-Video-2.0。

其中文本模型排在真实 Agent 评测榜 Claw-Eval第九，图片和视频模型均在 Artificial Analysis 盲评榜的前列。

![[../../../../images/07b00fc4ca5e8dbab452fdc8196e55d3_MD5.jpg]]

该消息一出瞬间爆了，第一周文本模型调用量就超过 1 万亿 Token，图片模型生成超过 200 万张，视频模型生成超过 200 万秒。

我也带着好奇心，将模型接入到 WorkBuddy，上手实测了一波图像和视频生成。

话不多说，直接来看实测结果，大家也可以跟着操作。

### 接入教程

第一步，先获取 Agnes 的 API Key。

访问_https://platform.agnes-ai.com_，注册登录后，在控制台里就创建并复制好 API Key：

![[../../../../images/51a8a684d6178a2beaf7d99b6055db5c_MD5.jpg]]

第二步，打开 WorkBuddy 配置模型。

官方有提供了详细接入 OpenClaw、Hermes、Claude Code 等 Agent 工具的教程。

大家可以自由选择自己熟悉的 Agent 工具，这里演示一下怎么在 WorkBuddy 使用。

打开 WorkBuddy，点击对话框里的「Auto」再点击「配置自定义模型」。

把刚才的 Key 填进去，接口地址填 _https://apihub.agnes-ai.com/v1_，模型名称填和模型名称 `agnes-2.0-flash`，如下图：

![[../../../../images/9af5d9fd8ebbac840092095716058d7b_MD5.jpg]]

填完之后，点击「保存」按钮。Agnes 文本模型就接入到 WorkBuddy 了，后续想用它直接选择它即可。

至于 Agnes 的图像和视频生成模型，则是通过打包成 Skill 进行调用。

第三步，创建图像和视频生成的 Skill。

可以直接在 WorkBuddy 中发送以下提示词，让它帮我们去创建一个图像生成的 Skill：

```
我想使用 Agnes Image 2.1 Flash 模型生图，访问它的 API 文档 https://agnes-ai.com/doc/agnes-image-21-flash，并将它打包成一份 Skill，命名为 agnes-image-gen。
```

至于视频生成的 Skill 也可这样创建，完成之后，点击「技能」，就能看到 agnes-image-gen 和 agnes-image-video。

![[../../../../images/6c78d29690951e4bde226434a8d00e81_MD5.jpg]]

接下来，就可以使用这两个 Skill 来测试 Agnes 图像和视频模型生成的效果。

### 上手实测

我们先来测试一下 Agnes-Image-2.1-Flash 图片模型，对不同风格的图像把控。

先来个赛博朋克科幻风，生成的图片光影和氛围都很到位。

![[../../../../images/c269622ae863cfaab1aa9d41b41d1955_MD5.jpg]]一座城市夜景，高楼林立，霓虹闪烁，雨水反射着光影，赛博朋克风格，整体很有电影感。

接着换个截然不同的风格。王家卫电影风格的人物照，出图的色调和颗粒感，确实有那么点味道。

![[../../../../images/bcaff1d948bd938e9e41f0c85680e565_MD5.jpg]]一位面目沧桑的老人，高品质，照片级真实感，王家卫电影风格，使用柯达 Portra 800 胶卷拍摄，高对比度。

再试一张中国水墨风格的山水画。

留白、墨色的浓淡过渡处理得比较自然，没有那种生硬的 AI 味，还额外加了印章和文字。

![[../../../../images/16232a942973593560cdb1f86c317539_MD5.jpg]]

文生图看完，再来看下模型的图生图。

主要测它的编辑能力，看看免费的模型，在精细活上能做到什么程度，能否保持人物一致性。

先准备一张原图，然后把人物表情改成自然的微微一笑。

这是个局部修改，难点在于改完表情，脸还得是原来那张脸。

![[../../../../images/9671f662c7e4d0eb18959d4543862ba0_MD5.jpg]]左：原图，右：生成图

实测下来，五官、发型、服装保持得还不错，笑得也挺自然的。

接着再试下，将图像生成一张蓝底证件照，成片基本能拿去用，还把人物的发型梳理好。

![[../../../../images/9760c5f684e7c12ef9f69909168a5341_MD5.jpg]]左：原图，右：生成图

从上面的实测来看，Agnes 模型能生成各种不同风格的图像，而且还不错。

图像编辑能力，在人物一致性这块也能较好保持，不会说改着改着就大变样。

总的来说，Agnes-Image-2.1-Flash 图片模型有点东西的，并不是说免费模型就被阉割。

接下来，看看 Agnes-Video-2.0 视频模型的表现。

值得一提，Agnes 生成的视频是原生支持音画同出，不需要我们单独对视频配音，确实香。

第一个场景，多镜头切换。提示词这样：

一场 GT3 赛车比赛，晴天日间，一辆 88 号红色法拉利领跑，远景、中景、特写来回切，要电影质感。

出来的成片效果有点被惊艳到，视频配乐和镜头调度同步得不错，车身号码也保持着一致。

![[../../../../images/10d3d7a1ce70f7581415cfd0b3856551_MD5.jpg]]

> 📹 此处为视频内容（vid: wxv\_4555235625347530754）（上图为封面），未能直接提取，请前往原文查看：[在公众号原文中观看](https://mp.weixin.qq.com/s?__biz=MzAxOTcxNTIwNQ==&mid=2457994301&idx=1&sn=f5dd955f69604da512c22b5a7fc1c34f&chksm=8d48a403c5e44968a830698c988100658c2a221dc262bf1400f634e52b1f3bbeca9e313b4650&mpshare=1&scene=1&srcid=0611Wj2CrFqJUwmeR88Vy0NL&sharer_shareinfo=d6ee3a7b38afda4f78223018b952ec89&sharer_shareinfo_first=d6ee3a7b38afda4f78223018b952ec89#rd)

第二个场景，难度更高的多人物。

一支摇滚乐队在演出，主唱挥手带动观众，背景射灯从暖黄逐渐过渡到冷蓝。

人物多、动作杂，还要顾及灯光变化，这种复杂场景很能看出模型的下限。

![[../../../../images/55829317ad58a75b5903796845e0377d_MD5.jpg]]

> 📹 此处为视频内容（vid: wxv\_4555233822652727296）（上图为封面），未能直接提取，请前往原文查看：[在公众号原文中观看](https://mp.weixin.qq.com/s?__biz=MzAxOTcxNTIwNQ==&mid=2457994301&idx=1&sn=f5dd955f69604da512c22b5a7fc1c34f&chksm=8d48a403c5e44968a830698c988100658c2a221dc262bf1400f634e52b1f3bbeca9e313b4650&mpshare=1&scene=1&srcid=0611Wj2CrFqJUwmeR88Vy0NL&sharer_shareinfo=d6ee3a7b38afda4f78223018b952ec89&sharer_shareinfo_first=d6ee3a7b38afda4f78223018b952ec89#rd)

第三再上难度，图生视频。

找来了一张图片，让模型基于这张图片生成一段在高速路上演追逐大片，这次提示词挺长，可以看下图。

![[../../../../images/e6255ba3d5820d81a74d456c3550c8ed_MD5.jpg]]

生成的视频效果，各种镜头切换、人物的动作相当流畅，人物主体也保持一致性。

而且配乐也是一绝，另外这次直接输出了 10 秒更长的视频。

![[../../../../images/142c738093f2de5d8936386de5ee583c_MD5.jpg]]

> 📹 此处为视频内容（vid: wxv\_4555236402904350722）（上图为封面），未能直接提取，请前往原文查看：[在公众号原文中观看](https://mp.weixin.qq.com/s?__biz=MzAxOTcxNTIwNQ==&mid=2457994301&idx=1&sn=f5dd955f69604da512c22b5a7fc1c34f&chksm=8d48a403c5e44968a830698c988100658c2a221dc262bf1400f634e52b1f3bbeca9e313b4650&mpshare=1&scene=1&srcid=0611Wj2CrFqJUwmeR88Vy0NL&sharer_shareinfo=d6ee3a7b38afda4f78223018b952ec89&sharer_shareinfo_first=d6ee3a7b38afda4f78223018b952ec89#rd)

经过这几轮测试，我心里只想说，Agnes AI 免费开放的模型，表现不输一些闭源模型。

另外，很多团队对模型开放免费之后，后面可能就不会再更新升级了。

但 Agnes AI 团队并不是这样，这周还在对模型进行升级，**已灰度上线，预计本周末更新完**。

升级后，Agnes-2.0-Flash 将原生支持 1M 超长上下文窗口，接入方式不用改。

Agnes-Image-2.1-Flash，也会新增支持 4K 分辨率图片生成，并支持多种宽高比。

### 写在最后

说真的，心底里挺佩服 Agnes AI 团队的，现在 Token 越来越贵，他们还不限量给我们使用。

而且这次免费开放的模型，并不是什么阉割版，Agent、图像、视频生成能力都不错。

后面很多自动化 Agent 场景，或者图像、视频生成的需求都可以交给它来做。

相信无论是对个人，还是独立 AI 团队，用 Agnes 在一定程度上能节省部分 Token 账单。

说再多，也不如花几分钟将 Agnes 模型接入到自己工作流中，跑一遍看看。

文档地址：_https://agnes-ai.com/doc_

今天的分享到此结束，感谢大家抽空阅读，我们下期再见，Respect！

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/d564a1d2_1781226755268?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzAxOTcxNTIwNQ%3D%3D%26mid%3D2457994301%26idx%3D1%26sn%3Df5dd955f69604da512c22b5a7fc1c34f%26chksm%3D8d48a403c5e44968a830698c988100658c2a221dc262bf1400f634e52b1f3bbeca9e313b4650%26mpshare%3D1%26scene%3D1%26srcid%3D0611Wj2CrFqJUwmeR88Vy0NL%26sharer_shareinfo%3Dd6ee3a7b38afda4f78223018b952ec89%26sharer_shareinfo_first%3Dd6ee3a7b38afda4f78223018b952ec89%23rd&s=obsidian)