---
author: 相声1
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI5MDExODA5NA==&mid=2648352572&idx=1&sn=4078ed1269ac620840963c8946cc4b03&chksm=f596ba2a481ecf96092541b74aa98c4053b22bd59bbac2006c01dad3d10709db1e08090c3feb&mpshare=1&scene=1&srcid=0512HALOU84l965reY2qEzsR&sharer_shareinfo=a2c3104da83d95055bfeb905a01ae377&sharer_shareinfo_first=a2c3104da83d95055bfeb905a01ae377#rd
saved: 2026-05-12 21:27:34
tags:
  - 笔记同步助手
id: fb7a42d5-3b85-4cd4-b1d1-ff6a4e7c4820
---

公众号名称：相声的技术备忘录

作者名称：相声1

发布时间：2026-05-12 11:30

最近折腾AI编程，有一个感受越来越强烈：术业有专攻。

无论是写软文、做策划，还是硬核的代码生成，DeepSeek（尤其是今年4月刚更新的版本V4）的表现都让我眼前一亮。最关键的是，它的网页版至今还是完全免费的。

但当我想把这份免费算力接入到OpenClaw（龙虾）、Claude Code 或者 VS Code里辅助开发时，问题就出现了——Deepseek的官方API不仅要钱，还有各种限流。

难道就没有办法让这些编程工具，直接调用那个“免费”的Deepseek网页版吗？

还真有！

经过一番挖掘，我发现了一款名为ds2api的神器（目前最新版 v4.6.1），官网链接是https://github.com/CJackHwang/ds2api，目前已经有4.1K的Star了。

![[Inbox/笔记同步助手/images/4c4e8bf5694a2351700daad27a27ec3e_MD5.jpg]]

它能把DeepSeek的网页对话能力，无缝转换成标准的OpenAI/Claude兼容接口。

也就是说，它把DeepSeek网页版变成了你的私有API服务器。

无需再购买Token，即可让各类编程工具免费调用这个国内最强大的ai。

DeepSeek网页版有多个模型，具体区别如下：

DeepSeek V4 Flash：快速推理模型(默认)

DeepSeekV4 Pro：高性能专家模型

DeepSeek V4 Vision：多模态视觉模型

上述3种模型还分别对应着NoThinking(关闭思考)和Search(联网搜索)。

通过DS2API，你可以访问以上所有的模型，基本上可以说，DeepSeek网页版能用什么，DS2API就能提供什么。

下面我就以Windows环境为例，手把手教大家搭建。

这一路我踩了不少坑，文中都会帮大家一一避开。

  

---

  

## 1.下载ds2api

下载链接为：

https://github.com/CJackHwang/ds2api/releases/latest

选择“ds2api\_v4.6.1\_windows\_amd64.zip”

![[Inbox/笔记同步助手/images/0c137c97fb5b59048dbdfbeaa95f9036_MD5.jpg]]

  

---

  

## 2.调试ds2api

将下载的ds2api\_v4.6.1\_windows\_amd64.zip，解压缩到某个文件夹，注意不要使用中文目录名。

将“config.example.json”这个文件名改为“config.json”，然后双击运行“ds2api.exe”。

坑1​：一定要先将“config.example.json”改为“config.json”，不然双击“ds2api.exe”后黑窗口一闪而过，程序并没有运行成功。

![[Inbox/笔记同步助手/images/0afc9c68af6b3f25666e9844a5ef46b9_MD5.jpg]]

程序启动后会弹出一个黑窗口，里面写的有默认密码，登录地址等等。

![[Inbox/笔记同步助手/images/0cfc03800957ec2b0f71db261bab64d4_MD5.jpg]]

在浏览器中打开http://192.168.56.1:5001/，或者是http

![[Inbox/笔记同步助手/images/75fca82405aaadce23026ead536fc47b_MD5.jpg]]

点击“管理面板”，输入默认密钥admin，即可进入管理员设置界面

![[Inbox/笔记同步助手/images/427957cc3b76f1737248fbe7273f965a_MD5.jpg]]

进入设置界面后，点击“添加账号”

![[Inbox/笔记同步助手/images/cc2b615ddf300dee8e4a62ce71fbff39_MD5.jpg]]

如下图所示，名称可以不写，但是最下面的手机号和密码是必须填写的（手机号那里的可选是错误的，应该是必填）。

![[Inbox/笔记同步助手/images/c8e870913ef88e363a6b6f91d433598e_MD5.jpg]]

密码对应的是Deepseek网页的登录密码。

  

---

  

坑2：不知道这个Deepseek的密码是哪里来的。

以前使用网页版的Deepseek时，都是使用手机验证码登录的，基本上没用过密码，后来在登录界面上找了一下，发现还真有个密码登录的选项。

点击下面的“密码登录”按钮

![[Inbox/笔记同步助手/images/d8c43c6f8c48c0fff0c1b5dae125c7cb_MD5.jpg]]

再点击“忘记密码”，就可以使用手机号和验证码来重新设置密码了。

这个密码要记住，后续有用。

![[Inbox/笔记同步助手/images/26870dceaa03883c11a2963c3e0ad1f4_MD5.jpg]]

另外，如果是新手机号首次使用，要点击“立即注册”，在注册界面可以设置密码。

![[Inbox/笔记同步助手/images/ecb1b8cfdafb371b53efeef4e015c1bd_MD5.jpg]]

在ds2api的界面上添加多个手机号后，下一步就是添加API密钥了。

点击“添加密钥”

![[Inbox/笔记同步助手/images/77a93d9d8d38b2222f1878e7c36bb48c_MD5.jpg]]

在弹出的窗口中，点击“生成”，其他内容可以不填写。这个密钥一定要复制出来，后续不会再显示。

如果实在没记住，可以在config.json这个文件里找。

![[Inbox/笔记同步助手/images/58836aa1d630cdd448c490183bd13eab_MD5.jpg]]

另外，程序自带的3个账号要删除掉，那个只是示例，用不了。

  

---

  

## 3.测试API

添加好DeepSeek账号和API密钥以后，可以点击左边的“API测试”，试一下这个模型能否使用，正常来说是没问题的。

![[Inbox/笔记同步助手/images/86dfe08a4d753703a4ffe3e200829727_MD5.jpg]]

至此为止，这个程序就设置完了，剩下的就是如何调用了。

  

---

  

## 4.Chatbox调用

点击“设置”，打开设置界面，点击“添加”，名称可以随便写，我写的是d1，API模式选择OPENAI API兼容，API密钥填写刚才记录的密钥，API主机填写http://192.168.56.1:5001/v1，或者写http://127.0.0.1:5001/v1

![[Inbox/笔记同步助手/images/05bf3a26c58d67987a1ba6368e17a125_MD5.jpg]]

然后点击“获取”按钮，会自动列出所有可用的模型，建议选择Deepseek-V4-Pro。

![[Inbox/笔记同步助手/images/f833622ccc966997a5fceae0d27e3a89_MD5.jpg]]

设置好以后，在对话框就可以看到刚才设置的模型了。

![[Inbox/笔记同步助手/images/15ef24851801f9bd70d59b539c0c70c4_MD5.jpg]]

可以问一下这个模型是什么。

![[Inbox/笔记同步助手/images/90973ecc1822dc0db5c69380283c3adc_MD5.jpg]]

  

---

  

## 5.Trae调用

Trae一定要升级到最新版本，不然好像是没法自定义模型。

点击右上角的设置图标，然后选择“模型”，点击“添加模型”

![[Inbox/笔记同步助手/images/0cd93122ca34fd6f84ea1cbc17db99f1_MD5.jpg]]

在弹出的对话框中选择“自定义配置”，相关内容如下图所示：

![[Inbox/笔记同步助手/images/ff0cb696745871e81bfde0ebb535989f_MD5.jpg]]

然后点击“高级配置”，在模型展示名称中，输入一个特殊的名字，用于区分软件内置的Deepseek模型，我是在模型前面加了一个a-。

![[Inbox/笔记同步助手/images/9a48c9fa9b40f306aa3d76b8baf99c69_MD5.jpg]]

添加完模型以后，在对话框里选择模型的地方，取消勾选Auto Mode，下滑到最下方，就可以看到自己添加的模型了。

![[Inbox/笔记同步助手/images/3fe25b759ac9837b0bd2bb26fad9ed5b_MD5.jpg]]

测试了一下，比Trae自带的模型好用多了，既聪明，还不用排队。

  

---

  

## 6.Claude Code调用

这里，我使用了cc switch这个软件，这是一个开源桌面工具，可以统一管理多个 AI 编程 CLI 的供应商配置，实现一键切换。

点击cc switch右上角的+号

![[Inbox/笔记同步助手/images/e68873546a78f21cc446feeaaae28047_MD5.jpg]]

使用第一个，自定义配置。

![[Inbox/笔记同步助手/images/93c924a63ba69e5a084842a383dbbda7_MD5.jpg]]

填写供应商名称、API KEY，请求地址，这3项就可以了

![[Inbox/笔记同步助手/images/d0687a27577c22abb5d7ae737927ff85_MD5.jpg]]

回到主界面，找到刚才添加的d1，点击启用，再点击命令行图标，即可启动了。

![[Inbox/笔记同步助手/images/6abce1138cff8c0a4c2b14570fd0d36c_MD5.jpg]]

坑3​：一定要多添加几个手机号，不然AI运行的会比较慢。

另外，不要使用自己常用的手机号，可能会污染你的Deepseek对话列表，也可能会被Deepseek封号，毕竟这不是正常的调用。

我自己添加了3个手机号，使用Trae写了一个测试程序，发现3个号码都被调用了，而且是被同时调用的。如下图所示：

![[Inbox/笔记同步助手/images/bce526275a30b20bc0757b1f3190f083_MD5.jpg]]

  

---

![[Inbox/笔记同步助手/images/dfce54b981580c450215871f5c6a1f3c_MD5.jpg|cover_image]]

Original 相声1 相声的技术备忘录

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/84834ca5_1778592453549?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI5MDExODA5NA%3D%3D%26mid%3D2648352572%26idx%3D1%26sn%3D4078ed1269ac620840963c8946cc4b03%26chksm%3Df596ba2a481ecf96092541b74aa98c4053b22bd59bbac2006c01dad3d10709db1e08090c3feb%26mpshare%3D1%26scene%3D1%26srcid%3D0512HALOU84l965reY2qEzsR%26sharer_shareinfo%3Da2c3104da83d95055bfeb905a01ae377%26sharer_shareinfo_first%3Da2c3104da83d95055bfeb905a01ae377%23rd&s=obsidian)