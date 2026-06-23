---
author: 科技前沿萌主
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkwOTYyMzg2MA==&mid=2247493626&idx=1&sn=29e25e9692ba7e06cc0cdeb5abdbfd69&chksm=c0289b7dc653464e4936d21c1e963421eedf637504ecde375a2e4c38f038a29b34376d7eb0d4&mpshare=1&scene=1&srcid=0511jDTQ3gfuQrF14nY84dls&sharer_shareinfo=b472914aaf66c2bdacc68021739a7404&sharer_shareinfo_first=b472914aaf66c2bdacc68021739a7404#rd
saved: 2026-05-11 22:58:22
tags:
  - 笔记同步助手
id: 868d9266-2d0c-473c-b858-55c7a4985c79
---

公众号名称：科技前沿AI

作者名称：科技前沿萌主

发布时间：2026-04-14 23:57

写着代码，突然弹出来一个429。

我深吸一口气。

这已经是今天第三次了。

![[../../../../images/45bd54f3ac4fc17112139bfcca76115a_MD5.jpg]]

用智谱CodingPlan的朋友最近应该懂这种感觉，限流是真的狠，社区里吐槽的帖子一堆，小红书、微博上骂声一片。

你不是不愿意花钱，是花了钱还是被卡着，这才是最让人血压拉满的地方。

然后我看到了一条消息。

美国云平台Modal宣布，免费提供智谱GLM-5.1模型，只限速，不限token用量。

我当时愣了一下，然后打开了那个链接。

先说怎么开通，大概一分钟，真的就一分钟。

打开 https://modal.com 首页，右上角Sign Up

![[../../../../images/9d6f51f9051b2eee01e6389d2f5075f8_MD5.jpg]]

用GitHub或者Google账号登录，

![[../../../../images/c9547b24f4b918ad783a6ee365d6936d_MD5.jpg]]

进来之后去这个地址：

https://modal.com/glm-5-endpoint

![[../../../../images/3e941ddbace884c5c3289c89f9d42db2_MD5.jpg]]

点左侧的「Create token」，输入一个token名称，比如我输入的是`yeoso`，然后弹框里就出来了apiKey。

![[../../../../images/b6c4361dbd69be4a9b59cfd27bca97da_MD5.jpg]]

**这里注意，apiKey只显示一次，一定复制保存好。**

​

把「Example usage」也一起复制下来，里面有关键的baseUrl信息，后面配置要用。

然后说怎么接进来用。

**接进Claude Code**

Modal给的端点是：

https://api.us-west-2.modal.direct/v1/chat/completions

这是OpenAI协议，不能直接塞进Claude Code，需要一个协议转换网关。

还好Modal官方已经做好了，轻量到只有一个Python文件：

https://github.com/modal-projects/modal-jazz/tree/main/frontends/claude

最简单的做法，把这个项目地址和你刚才复制的Example usage文本一起扔给Claude Code，让它帮你把转换服务搭起来。

官方配置文档在这里：

https://code.claude.com/docs/en/llm-gateway

**接进OpenClaw**

Modal也给好了配置demo：

https://github.com/modal-projects/modal-jazz/tree/main/frontends/openclaw

步骤也简单，把`sample.openclaw.json`里的模型密钥内容加到你的`openclaw.json`里，默认存在`～/.openclaw`，然后把`LLM_BACKEND_URL`更新成你的Modal托管LLM的URL，如果设置了`LLM_BACKEND_API_KEY`也一起加上。

可以直接在`openclaw.json`里写，也可以复制`.env.example`到一个`.env`文件里通过环境变量设置。

说到底，Modal这波开放做了一件事，把「国产最强编程模型之一免费用」这件事的门槛，从「抢名额、盯额度、被限流」，降到了「一分钟注册，复制一个apiKey」。

不用抢CodingPlan名额，不用担心额度，不用对着429深呼吸。

GLM-5.1这个水平，对大多数日常编程任务来说完全够用，接起来试试吧。

以上，既然看到这里了，相信是有所共鸣。随手点个赞、在看、转发三连，想第一时间看到新内容，给我个星标⭐即可。感谢陪伴，文字因你而完整，下次见。

---

![[../../../../images/9c5daf0a4f299f69fc25d73d430e4c58_MD5.jpg|cover_image]]

原创 科技前沿萌主 科技前沿AI

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/b060beff_1778511501519?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkwOTYyMzg2MA%3D%3D%26mid%3D2247493626%26idx%3D1%26sn%3D29e25e9692ba7e06cc0cdeb5abdbfd69%26chksm%3Dc0289b7dc653464e4936d21c1e963421eedf637504ecde375a2e4c38f038a29b34376d7eb0d4%26mpshare%3D1%26scene%3D1%26srcid%3D0511jDTQ3gfuQrF14nY84dls%26sharer_shareinfo%3Db472914aaf66c2bdacc68021739a7404%26sharer_shareinfo_first%3Db472914aaf66c2bdacc68021739a7404%23rd&s=obsidian)