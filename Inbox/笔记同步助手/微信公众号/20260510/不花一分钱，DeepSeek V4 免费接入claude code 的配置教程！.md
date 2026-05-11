---
author: AI小雷
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk2NDQwMDc5NQ==&mid=2247485856&idx=1&sn=e06210398c2be73eb64eb252279c7a0b&chksm=c5f7c25e6bed9bf5744cb927e80df94528100d6f53b3ae920ae4915000991b7d7663c6ebe60b&mpshare=1&scene=1&srcid=0510ulqr0GF0mTpXOyo9ULcX&sharer_shareinfo=d99d8cb222d620f4e5abce7288704ff4&sharer_shareinfo_first=d99d8cb222d620f4e5abce7288704ff4#rd
saved: 2026-05-10 19:07:40
tags:
  - 笔记同步助手
id: 0ce3c4f0-e68a-4cbb-a043-b4b80fb856e3
---

公众号名称：AI智界前沿

作者名称：AI小雷

发布时间：2026-04-26 12:48

![](https://relay-1.bijitongbu.site/p/70700830928012fac75cec7b2954ce2f.png)

大家好，今天是坚持更新的第44天！

最近DeepSeek V4太火了，推特上有不少网友到在拿它跟Claude Opus和GPT-5做对比。

我自己也在官方平台跑了几轮测试，发现写代码逻辑顺，对于复杂问题处理起来得心应手，最关键的是幻觉率对比之前的版本好很多了。

不过好东西往往不便宜，虽然官方API最近搞了限时折扣2.5折，但真拿来跑日常项目或者高频调试，token消耗速度可是很快的

![](https://relay-1.bijitongbu.site/p/e112c7af0289a24fb84c80c7fb8347c2.png)

但是我的想法很简单，就是想先零成本试水，测试下接入claude code后再决定要不要长期投入，不想一上来就掏钱。

我发现，阿里云百炼和魔搭社区其实都在发免费额度，而百炼直接送 100 万token，魔搭每天给2000次调用机会。

更爽的是，这些免费额度可以直接接到Claude Code里，界面还是那个熟悉的界面，但背后跑的是免费的 V4。

![](https://relay-1.bijitongbu.site/p/d621b639e3c5109613f4f2d0e89360cc.png)

一开始我也觉得配置肯定很麻烦，搞不好还会报错，结果我实际操作下来发现真没想象中复杂，就几个简单的步骤，不用写代码，几分钟就能搞定。

配好之后想切哪个模型，仅需点一下就行。

今天我就把整个流程详细写出来，你照着做就行，但是两个平台额度比较有限，跑不了大任务，要想让DeepSeek V4 编程自己的项目可以优先使用官方API的。

### 四步搞定免费接入

首先第一步先把钥匙拿到手。

目前有两个地方能白嫖，一个是阿里云百炼，地址是

https://bailian.console.aliyun.com/cn-beijing#/home

注册登录后在控制台找API-KEY 管理，新建一个，它直接送100万 token，这量级够你折腾很久了。

![](https://relay-1.bijitongbu.site/p/ea1ef7a668b68dbedb8ffef1f76668a0.png)

另一个是魔搭社区，网址 modelscope.cn，这个更直接，每天送 2000 次调用机会。

![](https://relay-1.bijitongbu.site/p/37ebf9bf1e8514da22cba35f7ee6cad9.png)

仅需在首页的访问控制页面那里找到访问令牌即可。

![](https://relay-1.bijitongbu.site/p/54516e72b580d7575cfadb88294de34d.png)

两个平台选一个就行，或者都注册了备用，把生成的 Key 复制下来，找个地方存好，别弄丢了。

其次第二步装个转换器，因为 Claude Code原生不支持直接切第三方模型，得用个叫CC Switch的小工具。

https://github.com/farion1231/cc-switch

打开后找到releases找到自己的系统对应的安装包下载就行了。

装好后打开，界面很简单，点加好，新建供应商，在页面里选 Bailian（对应阿里云）或者ModelScope（对应魔塔）。

![](https://relay-1.bijitongbu.site/p/8a3717716183605994e50b0b72c1d88c.png)

选好后，把刚才复制的Key粘贴进API Key输入框，其他的先不动，然后点保存。

第三步最关键，填模型名字。在CC Switch里找到刚刚保存好的模型配置，比如阿里云百炼模型。

往下拉找到高级选项，可以点击获取模型列表，CC Switch会自动获取所有可调用的AI模型，然后我们选择DeepSeek V4 pro版本即可，可以点击一键设置其他模型统一是DeepSeek V4 pro也是可以的。

![](https://relay-1.bijitongbu.site/p/f3589003918cf559ea80641cb7e8314e.png)

这是需要避坑的是，因为Claude Code的原生逻辑是为Anthropic的Claude模型设计的，当你强行接入OpenA 兼容格式的第三方模型（比如阿里云百炼）时，需要一个请求转换层 / 路由，把 Claude Code 的内部请求转换成标准的 OpenAI格式，再把响应转回来。

如果按照CC Switch原先的阿里云百炼的接口地址的话，在claude code上面调用是失败的，但阿里云百炼已经给出教程了，就是将请求地址改成

https://dashscope.aliyuncs.com/apps/anthropic

然后正常测试就可以使用了。

对于魔搭社区的话，正常配置好api-key后，获取模型列表后，选择DeepSeek V4 Flash模型，应该目前只支持这个模型吧，v4 pro没有显示。

![](https://relay-1.bijitongbu.site/p/fc72ed23836e65668ee51e70b9c265f8.png)

最后一步，点保存配置，这时候你打开终端，输入Claude，界面还是那个熟悉的样子，但背后跑的已经是免费的 DeepSeek V4 了。

试着让它写个 Python 脚本或者解释段代码，你会发现响应速度跟官方没啥区别。

以后想换模型或者切平台，进CC Switch改一下就行，不用反复折腾环境变量，这才是真正的零成本爽用。

比如我我最近比较喜欢了解中国道教文化，于是我输入以下提示词，让claude code（已经接入了DeepSeek V4 Flash）帮我写个展示天罡三十六变和地煞七十二变的网页。

```
请编写一份可独立展示网页代码，按需可以接入外部插件/CDN等。
整体设计采用中式古典古风风格，色调选用米黄、墨黑、朱砂红、石青等传统国风配色，背景辅以淡墨山水、祥云暗纹、竹简纹理，边框使用古风卷云纹样。
文字排版古韵雅致，字体选用古风书法类字体，大量留白，排版简约庄重，贴合道家典籍质感。
页面内容分为两大核心板块：
天罡三十六变
地煞七十二变
要求：
文风使用半文半白古雅文言风格，复刻古代神话、道家典籍的行文语感，用词典雅古朴，拒绝现代大白话；
完整逐条罗列天罡三十六项神通、地煞七十二项神通，每条搭配古文注解、能力释义与功用详解；
开篇增加古籍风序言，补充两者的神话渊源、道法品级区别、正统古典设定背景；
标题、分区采用古风篆刻 / 书法式设计，章节划分清晰；
全程无现代元素、无网红风、无花哨特效，整体氛围庄严肃穆，贴合中国古代仙侠、道教神话设定；
代码整洁，浏览器可直接打开正常显示。
请编写一份可独立展示网页代码，按需可以接入外部插件/CDN等。
整体设计采用中式古典古风风格，色调选用米黄、墨黑、朱砂红、石青等传统国风配色，背景辅以淡墨山水、祥云暗纹、竹简纹理，边框使用古风卷云纹样。
文字排版古韵雅致，字体选用古风书法类字体，大量留白，排版简约庄重，贴合道家典籍质感。
页面内容分为两大核心板块：
天罡三十六变
地煞七十二变
要求：
文风使用半文半白古雅文言风格，复刻古代神话、道家典籍的行文语感，用词典雅古朴，拒绝现代大白话；
完整逐条罗列天罡三十六项神通、地煞七十二项神通，每条搭配古文注解、能力释义与功用详解；
开篇增加古籍风序言，补充两者的神话渊源、道法品级区别、正统古典设定背景；
标题、分区采用古风篆刻 / 书法式设计，章节划分清晰；
全程无现代元素、无网红风、无花哨特效，整体氛围庄严肃穆，贴合中国古代仙侠、道教神话设定；
代码整洁，浏览器可直接打开正常显示。
```

很快claude code会联网搜索后并且完成了，虽然效果挺一般，但总体还是挺不错的。

![](https://relay-1.bijitongbu.site/p/836db535628d20caba297a36c4bc80b2.png)

在同一提示词下，对比DeepSeek V4 pro的生成结果，我个人认为pro还是强些的。

![](https://relay-1.bijitongbu.site/p/d53c642e9eafe995d0cc915c93fa58f7.png)

在Flash模型生成的内容挺可以的。

![](https://relay-1.bijitongbu.site/p/47cf903e8eaa71d07b7ec4e42bf1f120.png)

但我感觉pro模型生成的结果比较棒一些的。

![](https://relay-1.bijitongbu.site/p/bac6ac8c6ce9f6af1ac387c091060dfb.png)

### 写到最后

配置跑通之后，建议你先丢个小任务试试水，如让它帮你梳理项目结构，或者生成一段基础脚本。

跑几次就能摸清它的脾气，哪些场景响应快，哪些需要多给点提示词。

免费额度虽然会消耗，但用来日常辅助开发、查文档完全够用。

也是非常感谢你读到这里，欢迎您关注我的公众号，后续还会分享更多实用的AI工具和用法，帮你把 AI 用在实处。

  

---

![cover_image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/KiapdU6opqFGib7MNyBFMJmJH9VTRwnNcHJUgDtUnj4glcibuK2sDU68P4xxcCyugjQQrFstJQmM7nTLVdOibxOYJIBwFk4BnvKTZiaJlUuoJQics/0?wx_fmt=jpeg)

原创 AI小雷 AI智界前沿

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/db525738_1778411257975?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk2NDQwMDc5NQ%3D%3D%26mid%3D2247485856%26idx%3D1%26sn%3De06210398c2be73eb64eb252279c7a0b%26chksm%3Dc5f7c25e6bed9bf5744cb927e80df94528100d6f53b3ae920ae4915000991b7d7663c6ebe60b%26mpshare%3D1%26scene%3D1%26srcid%3D0510ulqr0GF0mTpXOyo9ULcX%26sharer_shareinfo%3Dd99d8cb222d620f4e5abce7288704ff4%26sharer_shareinfo_first%3Dd99d8cb222d620f4e5abce7288704ff4%23rd&s=obsidian)