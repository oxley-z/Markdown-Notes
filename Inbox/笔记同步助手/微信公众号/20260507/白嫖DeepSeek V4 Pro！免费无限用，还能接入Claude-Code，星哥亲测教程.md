---
author: 星哥说事
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzU0NDc0MTAyMg==&mid=2247491002&idx=1&sn=390c5279cfd8e374a91c8c9719b714d1&chksm=fa0a7ee50a67ea762e7cf73b0c86da75e0269e7fbcd8b731d08b910566d1fc1b37d237f03b10&mpshare=1&scene=1&srcid=0507MmAX8oj5ePV0tY8gBE7j&sharer_shareinfo=ad79bd8bd25e96dc0dda87a34a25e59c&sharer_shareinfo_first=ad79bd8bd25e96dc0dda87a34a25e59c#rd
saved: 2026-05-07 12:01:19
tags:
  - 笔记同步助手
id: 23f3ef84-ed2b-4879-94fe-9118c7c353f9
---

公众号名称：星哥玩云

作者名称：星哥说事

发布时间：2026-04-27 16:26

# 白嫖DeepSeek V4 Pro！免费无限用，还能接入Claude-Code，星哥亲测教程

最近星哥在逛圈的时候发现个超实用的 “羊毛”，必须分享给各位朋友 ——DeepSeek V4 Pro 居然能免费薅了！而且不是那种限时限次的试用，注册就能用，还支持 API 调用，能直接接入 Claude Code 等工具，相当于白嫖一个强力大模型，不管是日常轻量任务还是复杂的代码、写作需求，都能搞定。

不仅有DeepSeek V4 还有GLM4.6和4.7。

先放个结论：按星哥的步骤配置完，你就能在本地工具里直接调用 DeepSeek V4 Pro 的免费额度，不用充值，不用复杂操作，5 步就能搞定！

![[Inbox/笔记同步助手/images/9dae66989380314edacb66872f5418d4_MD5.jpg]]

免费的模型：

![[Inbox/笔记同步助手/images/a78a2d33e5d534f3facd764403550037_MD5.jpg]]

## 一、准备工作

想薅这个羊毛，提前准备好这两个工具 / 账号就行，都是易获取的：

1.  1\. ZenMux 账号：直接去官网（zenmux.ai）注册，这是对接 DeepSeek V4 Pro 免费版的关键平台。
    
2.  2\. cc switch（非必须）：一款本地 API 转发 / 切换工具，核心作用是把第三方 API 接入本地使用环境，还支持模型映射、转发，玩 API 的朋友应该不陌生；
    

如果没有安装CC的，可以参考之前星哥的文章： [安装Claude Code和cc-switch](https://mp.weixin.qq.com/s?__biz=MzU0NDc0MTAyMg==&mid=2247490717&idx=1&sn=2c81844a471d08b438f20eb36c45f608&scene=21#wechat_redirect)

cc switch是非必须的，如果你有其他的大模型软件比如Cherry Studio，也可以用这个来连接使用deepseek v4 pro。

​

## 二、流程

整个操作逻辑特别清晰，就是 “注册→建 Key→限模型→配工具→测试”，一步都不绕，星哥给大家拆解得明明白白：

### 第一步：下载安装 cc switch

先把 cc switch 下载好（大家可以用自己常用的渠道搜，或者找靠谱的资源站），安装完成后先别着急配置，等后面拿到 API Key 再操作。

### 第二步：注册 ZenMux 账号

打开 ZenMux 官网，点击注册按钮，星哥建议直接用 Google 登录，几秒就能完成，不用填一堆繁琐信息，省时间。

![[Inbox/笔记同步助手/images/68195d79f4649cfd035137e1712eedf7_MD5.jpg]]

### 第三步：创建专属 API Key

登录 ZenMux 后，点击右上角的头像，找到 “PAYG API” 页面进入，然后点击 “创建新的 API Key”，生成后先复制保存好，后面要用到。

![[Inbox/笔记同步助手/images/2f6931ef0c6a6aa1f1fdb6185f71d92c_MD5.jpg]]

如图：点击创建API密钥

![[Inbox/笔记同步助手/images/0b59904b0961a59672b41002b16266cd_MD5.jpg]]

### 第四步：限制模型范围（关键！）

这一步千万别漏，不然可能调用不到免费模型。创建 API Key 时，会看到 “Model Restriction（限制模型范围）” 的选项，在这里填入两个免费模型：

​

> deepseek/deepseek-v4-flash-freedeepseek
> 
> deepseek-v4-pro-free
> 
> GLM4.6V Flash
> 
> GLM4.7 Flash

星哥提醒下：flash-free 版本速度更快，适合查资料、简单问答这类轻量任务；pro-free 版本能力更强，做复杂推理、写代码、写长文选它准没错。填好后点击确认就行。

![[Inbox/笔记同步助手/images/199e983e19f980652562cfd4247bb991_MD5.jpg]]

![[Inbox/笔记同步助手/images/1ed696cf25e3488086aef32fd61c6d68_MD5.jpg]]

把密钥复制出来

![[Inbox/笔记同步助手/images/45d32ce60f549589f1d303b27bc87820_MD5.jpg]]

## 三、配置 Cherry Studio

星哥喜欢用Cherry Studio来测试密钥是否正常使用

### 1.添加供应商

![[Inbox/笔记同步助手/images/2a622f53ec3f95e390f6160970dfa070_MD5.jpg]]

### 2.填写密钥和api地址

密钥：刚才申请到的密钥

api地址:看截图

![[Inbox/笔记同步助手/images/e63818358d6be9d0769cc07447ca3f31_MD5.jpg]]

### 3.添加模型

点击获取模型列表，搜索free

![[Inbox/笔记同步助手/images/2f1fa7fb38e72a64dd60cf775196f32e_MD5.jpg]]

### 4.测试对话

它的回答：我是DeepSeek最新版本的模型，但并不是DeepSeek-v4。

![[Inbox/笔记同步助手/images/fc8936e17d97fc3e8c8ab5d6e5851698_MD5.jpg]]

## 四、配置 cc switch

打开安装好的 cc switch，新建一个自定义配置，按下面的信息填：

API Key：粘贴刚才在 ZenMux 创建的那个 Key；

模型映射：同样填入 “deepseek/deepseek-v4-flash-free 或者 deepseek/deepseek-v4-pro-free”，和前面限制的模型对应上。

### 1.添加配置

进入CC-Switch，点击加号

![[Inbox/笔记同步助手/images/4c49ddf9e7e7ace1045c402ac38becac_MD5.jpg]]

### 2.编辑供应商

如图必填

​

```
api key 申请到的
请求地址：zenmux点ai/api/anthropic   【 把点换成.  ，前面加上https://】
模型名称全部填 deepseek/deepseek-v4-pro-free
```

![[Inbox/笔记同步助手/images/7c1ca0d8548d93025b55c7ee25a90c3a_MD5.jpg]]

### 3.验证是否配置成功

配置完别着急用，先在本地工具里发起一个简单的请求，如果能正常返回结果，说明已经成功接入 DeepSeek V4 Pro 免费版了！

星哥亲测，只要步骤没出错，基本一次就能成。

![[Inbox/笔记同步助手/images/23f114f48aa4ff98474e192ff1290e8a_MD5.jpg]]

## 五、总结

虽然是免费羊毛，但用的时候注意这几点，能避免踩坑：

1.  1\. 关于稳定性：毕竟是免费模型，高峰时段可能会响应变慢、排队甚至短暂不可用，属于正常情况。星哥建议遇到这种情况，要么切换到 flash-free 模型，要么稍等 10 分钟再试；
    
2.  2\. 关于模型能力：pro-free 版本虽然够用，但和付费版比，在上下文长度、推理深度上略有限制。如果做长文本处理、高强度开发，记得把内容分段，效果会更好；
    
3.  3\. 关于安全：API Key 千万别明文上传到公开仓库，也别随便分享给别人，避免被滥用，导致账号出问题。
    

总的来说，这个 DeepSeek V4 Pro 免费版的羊毛性价比超高，不管是个人日常用，还是开发者测试接口，都能省不少钱。

星哥亲测整个流程下来，前后不到 10 分钟，配置好后就能一直用免费额度，感兴趣的朋友赶紧动手试试！

  

---

![[Inbox/笔记同步助手/images/52ecb44f74dfc6674b67c9fbf62c023b_MD5.jpg|cover_image]]

Original 星哥说事 星哥玩云

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/bd3a3b62_1778126478809?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU0NDc0MTAyMg%3D%3D%26mid%3D2247491002%26idx%3D1%26sn%3D390c5279cfd8e374a91c8c9719b714d1%26chksm%3Dfa0a7ee50a67ea762e7cf73b0c86da75e0269e7fbcd8b731d08b910566d1fc1b37d237f03b10%26mpshare%3D1%26scene%3D1%26srcid%3D0507MmAX8oj5ePV0tY8gBE7j%26sharer_shareinfo%3Dad79bd8bd25e96dc0dda87a34a25e59c%26sharer_shareinfo_first%3Dad79bd8bd25e96dc0dda87a34a25e59c%23rd&s=obsidian)