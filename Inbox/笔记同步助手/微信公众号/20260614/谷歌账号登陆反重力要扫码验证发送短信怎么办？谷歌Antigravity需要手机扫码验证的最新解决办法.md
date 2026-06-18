---
author: MyLabs
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg4ODAyNzM0Ng==&mid=2247487231&idx=1&sn=3bdf4ecef4504010b5bcf9faebe7cd0a&chksm=ce024d47789be38af4b9d44f485548626358bd7c92722c8c3ecbded9da75964dc1111ffd7950&mpshare=1&scene=1&srcid=0614fysYnLvhX1NAuxYWAg7M&sharer_shareinfo=9a9e6eee0277c16ba2e0e268c00d364c&sharer_shareinfo_first=9a9e6eee0277c16ba2e0e268c00d364c#rd
saved: 2026-06-14 15:31:45
tags:
  - 笔记同步助手
id: bda3b627-f6ba-4100-915e-642e0d208637
---

公众号名称：Mac的实验室

作者名称：MyLabs

发布时间：2026-06-14 12:57

**最近部分使用Google Antigravity的朋友反映，他们在使用谷歌Gemini反重力时弹出手机号验证，需要手机扫描二维码发送短信才能验证，但是发送短信总是失败，卡在这一步上无法进行下去了，这种情况怎么办？**

![[Inbox/笔记同步助手/images/b0d303225620d968c93d5c27b12efcc9_MD5.jpg]]

  

反重力验证页面提示：验证您的信息后才能继续，需要使用您的手机号扫码，打开相机应用扫码上方二维码，然后点按显示的链接继续操作。

按照页面说明去操作后，发现Google为了安全起见，需要验证设备或电话号码，然后跳转到需要手机号发送短信才可以：

![[Inbox/笔记同步助手/images/213beed26f0049e30d56a399a1d39762_MD5.jpg]]

![[Inbox/笔记同步助手/images/428e6c5a0472e38aecd3fee7bf48c0d5_MD5.jpg]]

  

这就跟注册时一样无解了，安卓/iPhone开魔法后扫码之后跳转短信，要求给 244444 开头的号码发送短信，直接发提示发送失败，手动 +1 之后发送成功但 Google 这边没有反应。

![[Inbox/笔记同步助手/images/d69cde29625eea3c259ba5d35d8acaa6_MD5.jpg]]

  

论坛这边已经有人吐槽这件事了，刚买的token额度都还没用就遇上风控验证了。

![[Inbox/笔记同步助手/images/08899522b660ba8efe29fdedda8c5d0e_MD5.jpg]]

![[Inbox/笔记同步助手/images/ecd6f9a83ada52a48b1fcc44ce3609c2_MD5.jpg]]

  

### **这里顺手简单说明一下谷歌Antigravity究竟是什么？**

  

### **Antigravity（反重力）是谷歌推出的全新 **AI 智能体驱动的软件开发平台，也就是IDE编辑器，专门为写代码开发使用的工具平台。****

  

---

  

### **为什么使用上面这两个工具平台会出现扫码发送短信验证这种情况？**

  

根据收集的网友反馈，通常是由于**更换网络环境频繁**触发了谷歌的风控防范措施引起的，前几个月的时候Google就已经在做这道验证措施了，一开始是可以让用户选择通过验证手机号来解决的：

![[Inbox/笔记同步助手/images/50ab93812d1f43ee4ec291215ac92fb3_MD5.jpg]]

![[Inbox/笔记同步助手/images/c9345ed9a2efee86fcb6f9507057d6e5_MD5.jpg]]

  

后面使用反重力薅羊毛的用户实在太多了，懂的自然懂，谷歌大善人演都不演了，直接把风控措施拉到最高值，砍掉了短信验证的选项，只提供扫码发短信验证的方式了。

只要用户ip切换得太频繁，就会弹出要求扫码验证。

年初就有爆出谷歌在考虑改革了，原因是谷歌认为短信验证不安全——

对！！短信验证它都信不过了。

系统强制用户用手机去扫码发短信验证。

  

![[Inbox/笔记同步助手/images/d13a6769e99fcf3c2b890035195fed6b_MD5.jpg]]

  

### **怎么解决扫码发送短信失败无法验证的问题？**

  

![[Inbox/笔记同步助手/images/298c06eb077ee1b8b7aead0d4c1e0999_MD5.jpg]]

**方法一：保持手机和电脑是同一美国节点，用同一个美国ip，保持一致性，然后用手机Chrome浏览器扫码发送短信验证。**

使用86号码发送短信有一定的成功率。但成功的案例不多！可以尝试一下。

**方法二：86手机号发送不了短信，改使用国外手机号去发送短信验证。**

鉴于现在无法使用接码平台的方式接收验证码验证了，只能使用插有国外sim卡的方式去扫码发送短信。

**方法三：由大神摸索出来的方法，独一无二。使用Google Pixel、三星手机这些具备Google 原生系统的手机去扫码发送短信，记得用干净的美国ip环境，直接就能验证成功！**

为什么不能用国产手机设备？国产手机的安卓系统都不是原生的，都是经过厂商改造和阉割了，毕竟为了合规和功能性使用习惯等等原因，导致这些手机安卓系统并没有完整的谷歌框架。

而谷歌最新的风控系统会检测发送端是否具备完整的谷歌框架，并且只接受视为受信任的移动设备端验证。

那没有以上那些型号的手机设备怎么办？可以试试安卓模拟器，模拟原生安卓系统。大神就是用这种方式跑出来的脚本，直接一次性通过谷歌的反重力验证了！觉得麻烦不想这样的同学可以在公众号：**Mac的实验室** 后台回复 **反重力**获取帮助。

验证成功后它会出现下图的提示：

![[Inbox/笔记同步助手/images/e5a37435683a845620e35f086b0e6227_MD5.jpg]]

  

然后返回电脑端，就会出现unlock的解锁记载中的提示：正在解锁访问权限

![[Inbox/笔记同步助手/images/ee0a31e784f8f006dcc0a97bf7e01e90_MD5.jpg]]

  

出现这个提示就代表验证成功了，然后页面会自动跳转到反重力使用界面。之后就可以放心使用Antigravity了！觉得麻烦不想折腾的同学可以在公众号：Mac的实验室 后台回复 反重力 获取求助哦！

![[Inbox/笔记同步助手/images/a58d2684533c7ec8c282fd8bcb2b86b2_MD5.jpg]]

  

谷歌Gemini的反重力验证方法就分享到这里，希望对你有所启发，后面如果有新方法会继续更新，欢迎关注！有问题也欢迎留言反馈哦！

![[Inbox/笔记同步助手/images/6721c7d4de8dca44abfaf64ebe01f62c_MD5.jpg]]

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/0e682f85_1781422303324?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg4ODAyNzM0Ng%3D%3D%26mid%3D2247487231%26idx%3D1%26sn%3D3bdf4ecef4504010b5bcf9faebe7cd0a%26chksm%3Dce024d47789be38af4b9d44f485548626358bd7c92722c8c3ecbded9da75964dc1111ffd7950%26mpshare%3D1%26scene%3D1%26srcid%3D0614fysYnLvhX1NAuxYWAg7M%26sharer_shareinfo%3D9a9e6eee0277c16ba2e0e268c00d364c%26sharer_shareinfo_first%3D9a9e6eee0277c16ba2e0e268c00d364c%23rd&s=obsidian)