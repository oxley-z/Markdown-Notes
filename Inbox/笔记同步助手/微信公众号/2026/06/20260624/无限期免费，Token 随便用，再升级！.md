---
author: 小 G
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzAxOTcxNTIwNQ==&mid=2457994435&idx=1&sn=e377d2843b72057612132cdd90e3ebfb&chksm=8d3f3c698a05354c139567fd8df047f49ab69413f64d42bc5c3cd131f634cb230c764662ece3&mpshare=1&scene=1&srcid=0624HzcZvEmGUmNAdntkGnuW&sharer_shareinfo=b183b17439e65ff0aad6ec95ba3f56e5&sharer_shareinfo_first=b183b17439e65ff0aad6ec95ba3f56e5#rd
saved: 2026-06-24 17:12:10
tags:
  - 笔记同步助手
id: fd259a59-8cd6-425d-aabf-5626620a6812
---

公众号名称：GitHubDaily

作者名称：小 G

发布时间：2026-06-24 17:05

不久前，跟大家分享过 Agnes AI，它把旗下三款全模态模型的 API 无限期免费开放。

分别是 Agnes-2.0-Flash、Agnes-Image-2.1-Flash、Agnes-Video-2.0，文本、图片、视频模型全覆盖。

没想到那篇文章发出去之后爆了，但使用人一多，就会出现各种问题，Agnes 也留意到。

于是在 GitHub 上创建一个叫 **Agnes-AI** 项目，进行收集并跟踪修复，往后大家遇到问题都可以在 Issues 上看看。

GitHub：_https://github.com/AgnesAI-Labs/Agnes-AI_

![](https://relay-1.bijitongbu.site/p/13c8f72a29ac260456a048a67a4f0e0a.png)

回头算了下， Agnes 已经免费开放第三周了，今天看到其公布最新一则统计数据，非常亮眼。

全模态模型单周总 Token 调用量再创新高，达到了 **4.11 万亿**，在 OpenRouter 上仅次于 DeepSeek V4 Flash。

其中文本模型贡献约 2.67 万亿，图片和视频合起来约 1.44 万亿，图片一周生成 567 万多张，视频 237 万多秒。

![](https://relay-1.bijitongbu.site/p/5b73056aaacda42feb8c6b694d5a7a30.png)

更关键的是，它不光没有限制大家调用，还接着对模型能力进行升级，并且照样可以免费使用。

目前 Agnes-2.0-Flash 已支持 1M 超长上下文，代码、长文档可以一股脑喂进去；Agnes-Image-2.1-Flash 也支持 4K 超高清图片生成，最高能出 4096×4096 尺寸；还有一个 Agnes TTS 能力也正在灰度即将上线。

然后上次文章发完，后台不少同学在问怎么在 Claude Code 上使用 Agnes？

今天抽空跟大家说说，也顺便测试一下模型 4K 超高清图生成的效果。

### 接入 Claude Code

如果还没有 API Key，可以先到 Agnes 后台上 _https://platform.agnes-ai.com_创建。

上次不少同学说看到一个有效余额，这个其实不用管的，通过 API Key 调用模型均是免费。

![](https://relay-1.bijitongbu.site/p/28f7bc4ed45d910b1d8de6c83aba151d.png)

打开 cc-switch 给 Claude Code 配置模型，选「自定义配置」，填上供应商名称。

再把 API Key 粘贴进去，请求地址填这个：https://apihub.agnes-ai.com/v1

![](https://relay-1.bijitongbu.site/p/bf068282d624e96a108bc99f997fa4d0.png)

继续点击「高级选项」，这里需要注意，API 格式选「OpenAI Chat Completions」。

而模型映射全部填 「agnes-2.0-flash」，还可以把 1M 勾上，声明支持百万上下文长度。

![](https://relay-1.bijitongbu.site/p/6f286334e98bb53620707012c6b10e15.png)

为了避免请求不兼容，建议在下面配置 JSON 里增加下面两个参数，直接复制粘贴过去即可。

```
{
"allowed_openai_params": [
"thinking",
"context_management"
],
"litellm_settings": {
"drop_params": true
}
}
```

![](https://relay-1.bijitongbu.site/p/0987e90e6cbed6586c0e6c07164af461.png)

点击「保存」，回到主界面「启用」，再把本地路由开关打开，如果没看到这个开关可到设置里开启。

![](https://relay-1.bijitongbu.site/p/81c6fe2ea75f77d0a086b164a0e4b21d.png)

接下来，打开 Claude Code 就可以使用了。但到这里只能使用 Agnes-2.0-Flash 文本模型。

对于 Agnes-Image-2.1-Flash 图片模型和 Agnes-Video-2.0 视频模型，需要通过 Skill。

现在很多开发者，开始围绕 Agnes 做相关开源项目了，在 GitHub 上一搜就能看到。

![](https://relay-1.bijitongbu.site/p/27e35b4484b5a55cd124da91e67dee40.png)

其中 agnes-ai-generation-skill，把 Agnes 的文本、图片、视频 API 封装成了一个 Skill。

正好可以装一个使用，只需要 Claude Code 发送一句话让 AI 自己去安装就行：

```
帮我安装这个 Skill 在当前项目。https://github.com/Yacey/agnes-ai-generation-skill
```

![](https://relay-1.bijitongbu.site/p/42e8acf6593b892627583732e7d0992e.png)

### 实测 4K 生成效果

装好 Skill 之后，就可以在 Claude Code 上使用 Agnes 生成图片。

先来个简单点的，让它生成一张 4K 超高清产品海报图，提示词如下：

![](https://relay-1.bijitongbu.site/p/3559eeabc8a3be833783dea2439f7fee.png)

稍等一会，就一张 3848 × 2160 尺寸的超高清产品大图就生成了，整体效果还不错。

![](https://relay-1.bijitongbu.site/p/a21394fe26c1bb6f45f64be08fc53ddc.png)

看起来挺高级的，而且名称渲染也正确，但下面描述的文字有点模糊不清晰。

于是我再试下，让它生成一张手机拍摄的照片，一篇手写内容，这次还指定了宽高比。

![](https://relay-1.bijitongbu.site/p/a391c1bca0878c78f765367167482e5f.png)

这次字体全部正确，指定的尺寸也对上，放大还能清楚看到纸张的破旧纹理。

![](https://relay-1.bijitongbu.site/p/68f55297b53a66b571239af7b1a2aacf.png)

继续上难度，Agnes 模型支持 4K 图片生成，对老旧照片修复的场景来说是非常实用。

于是我找来了一张破损极其严重的老照片，模糊到我们肉眼都很难看清全部信息。

![](https://relay-1.bijitongbu.site/p/d91285315ed933e06f5ed1fddb7cb3c5.png)

直接将图片发送给 Agnes-Image-2.1-Flash 做 4K 高清修复。

这张图的确是有点难到 Agnes 了，第一次生成的图片没有看到婴儿，然后再试了一次。

![](https://relay-1.bijitongbu.site/p/7ba69c0fde753bb8ad52a79f05579d98.png)

结果修出来的效果，我觉得已达到修复原图的九成以上，人物、姿态、背景基本都正确。

大家可以看下，左边是第一次生成效果，右边是第二次同样的提示词生成效果。

![](https://relay-1.bijitongbu.site/p/b0ad983f7f4e35a3725e298daa18d019.png)

所以有时候，在使用 Agnes 模型生成图片，如果觉得不满意可以多抽下卡。

当然也得说点实在话。在实测 4K 图片生成的时候，API 接口请求挺慢的，需要给点耐心。

可能现在真的太多人使用了，为了保证稳定，Agnes 不得不做一些限制。

4K 图片生成一分钟只能请求一次，另外 1M 上下文也一样，在高峰期时会压到 512K。

这些限制可以理解，毕竟免费肯定是很多人都在薅，但对大家的使用影响并不大。

另外 Agnes 也非常用心了，全模态模型免费开放使用后，的确大家都遇到了各种问题。

官方还特意进行了收集，并把已知问题和高频报错都放到项目 GitHub Issues 上。

比如接入 Codex 请求 API 报 400、图片生成出现 422、视频字幕模糊等问题。

还放出了修复进度看板，大家的反馈、Bug、功能排期实时同步，处理到哪一步全部公开透明。

![](https://relay-1.bijitongbu.site/p/50e42f39d2e078f72081169c005bc804.png)

### 写在最后

说实话，这两年免费的 AI 产品见过不少，能做到像 Agnes 这样的，确实不多。

它没把免费当噱头，这半个多月还给模型加上 1M 上下文、4K 图生成功能，依然免费开放。

或许真如他们官网所说，Agnes 想做的，是让世界级的 AI 属于每一个人。

如果还没有尝试过 Agnes，不妨接入看看，反正是免费的不用白不用。

文档地址：_https://agnes-ai.com/doc_

今天的分享到此结束，感谢大家抽空阅读，我们下期再见，Respect！

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/00f470d2_1782292328686?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzAxOTcxNTIwNQ%3D%3D%26mid%3D2457994435%26idx%3D1%26sn%3De377d2843b72057612132cdd90e3ebfb%26chksm%3D8d3f3c698a05354c139567fd8df047f49ab69413f64d42bc5c3cd131f634cb230c764662ece3%26mpshare%3D1%26scene%3D1%26srcid%3D0624HzcZvEmGUmNAdntkGnuW%26sharer_shareinfo%3Db183b17439e65ff0aad6ec95ba3f56e5%26sharer_shareinfo_first%3Db183b17439e65ff0aad6ec95ba3f56e5%23rd&s=obsidian)