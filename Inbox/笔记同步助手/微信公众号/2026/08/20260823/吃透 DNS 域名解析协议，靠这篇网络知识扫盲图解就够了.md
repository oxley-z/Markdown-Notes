---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247491970&idx=1&sn=bcfed1cde0897e50decab84c79bc8d12&chksm=c342f4940609086b24b4babb0e304b6052156e849cf9bbf04812984c0d044bfab917b8e91f67&mpshare=1&scene=1&srcid=0823UsA3AvEYzPDFepdnNCrC&sharer_shareinfo=50dfe68aa4a44cb6d9ce74003b09287d&sharer_shareinfo_first=50dfe68aa4a44cb6d9ce74003b09287d#rd
saved: 2026-08-23 00:43:07
tags:
  - 笔记同步助手
id: fc516478-9234-49c4-95b1-8742b6446868
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-08-22 20:14

大家好，我是蟹老板～

日常面试过程中，面试官非常喜欢问的技术点，除了数据结构与算法之外，网络也是必问的。

我们每天敲域名、访问网站、部署项目，看似简单的域名访问，背后藏着整个互联网的寻址逻辑。可以这么说，没有DNS，整个互联网基本瘫痪。

那么问题来了：

> 浏览器究竟是怎么把一个域名变成 IP 地址的？

更加深入一点：

-   • 为什么 DNS 不是只有一台服务器？
    
-   • 根 DNS 服务器到底保存了什么？
    
-   -   `.com` 服务器为什么知道 `example.com` 应该去哪里查询？
-   • DNS 为什么有递归查询和迭代查询？
    
-   • 为什么修改 DNS 记录以后，有的人已经生效，有的人却还没有生效？
    
-   • DNS 为什么大多数时候使用 UDP，但是又必须支持 TCP？
    
-   -   `A`、`AAAA`、`CNAME`、`MX`、`NS` 到底分别是什么？
-   • DNSSEC、DoH、DoT 为什么经常被放在一起讲，但实际上解决的却不是一个问题？
    
-   -   `dig +trace` 为什么能够一步一步从根服务器找到最终的权威 DNS？
-   • 网站能够正常解析出 IP，但是为什么还是打不开？
    

这些问题如果只停留在：

> “DNS 就是把域名转换成 IP 地址。”

那么其实还只是知道了 DNS 最表面的一层作用。

DNS 真正解决的是一个**全球范围、分布式、层次化、可委派、可缓存的名字解析问题**。早期 DNS 标准将它描述为由树形域名空间、资源记录、名称服务器以及解析器共同组成的分布式数据库系统。

## 一、DNS是什么？为什么互联网离不开DNS？

### 1.1 如果没有DNS，上网会是什么样？

假设互联网没有 DNS。

那么你想访问一个网站的时候，也许不能输入：

```
www.example.com
```

而需要直接输入服务器 IP：

```
192.0.2.10
```

访问另外一个网站：

```
198.51.100.20
```

再访问第三个网站：

```
203.0.113.50
```

那么你的日常上网可能就会变成：

```
购物网站：203.0.113.21
公司邮箱：198.51.100.17
搜索网站：192.0.2.38
视频网站：203.0.113.85
```

可以想象一下，如果让一个普通用户背几十个甚至几百个这样的 IP 地址，体验会有多么糟糕。

更麻烦的是：

**IP 地址还可能发生变化。**

假设某个网站原来的服务器地址：

```
192.0.2.10
```

后来因为：

```
服务器迁移
机房切换
负载均衡
CDN调度
故障切换
```

变成：

```
198.51.100.20
```

如果用户记住的是 IP，那么所有人都得重新知道这个地址。

而如果用户记住的是：

```
www.example.com
```

网站管理员只需要修改域名对应的 DNS 数据。

用户仍然访问：

```
www.example.com
```

即可。

这就是名字系统最直接的价值之一：

> **让用户记住一个相对稳定的名字，而把容易随着网络拓扑变化的地址隐藏在名字后面。**

DNS 最初的设计目标之一正是提供一个可扩展的层次化名字系统，从而取代早期集中维护的主机名字表。

![](https://relay-1.bijitongbu.site/p/90411dd3fbbc96ae9772e8f2ddb2e31c.png)

### 1.2 IP地址为什么不适合人类记忆？

IP 地址非常适合计算机。

例如 IPv4：

```
8.8.8.8
```

从协议角度来看非常明确。

IPv6：

```
2001:4860:4860::8888
```

同样能够精确表示一个网络地址。

但是人类更加擅长记住：

```
google.com
example.com
mail.company.com
```

这样的语义化名字。

所以可以把：

```
IP 地址
```

理解成一个人的：

```
身份证号码
```

而：

```
域名
```

更像一个人的：

```
名字
```

现实生活里面，我们通常会说：

> “去找张三。”

而不会说：

> “去找身份证号 110101xxxxxxxxxxxx 的那个人。”

DNS 做的事情，与此非常类似。

![](https://relay-1.bijitongbu.site/p/b3e8302579239e9399b40fb0327c78e3.png)

### 1.3 从“记住IP地址”到“使用域名”

现在我们拥有：

```
www.example.com
```

但是网络层最终需要：

```
IP
```

所以中间必须有人帮我们完成：

```
www.example.com
        ↓
      DNS
        ↓
IP Address
```

于是应用程序只负责问：

> `www.example.com` 在哪里？

DNS 系统负责回答：

> 它对应的是这些网络地址。

然后应用程序才继续进行真正的网络通信。

![](https://relay-1.bijitongbu.site/p/5a2e249a7f471fbc840769e5bc0acabe.png)

### 1.4 DNS本质上解决了什么问题？

经常有人说：

> DNS 就是把域名转换成 IP。

对于入门来说，这句话没有问题。

但是更加准确地说，DNS 是一套：

> **分布式、层次化的命名系统以及查询协议。**

域名不只能关联：

```
IP 地址
```

还能够关联：

```
邮件服务器
权威 DNS 服务器
服务地址
文本信息
证书签发策略
DNSSEC 公钥和签名
```

等大量数据。

这就是为什么 DNS 里面存在：

```
A
AAAA
MX
NS
TXT
SRV
CAA
DNSKEY
RRSIG
```

这么多资源记录类型。

DNS 标准中的核心对象实际上是：

**Resource Record，资源记录。**

查询的本质更接近：

> “请告诉我 `example.com` 这个名字下面，某一种类型的资源记录是什么？”

而不仅仅是：

> “告诉我它的 IP。”

DNS 的标准定义正是围绕树形名称空间与 Resource Record 展开的。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/34acae0599bfd24578936a9c1c5822c9_MD5.jpg]]

### 1.5 DNS在TCP/IP协议栈中的位置

DNS 通常被归类到：

**应用层协议。**

可以简单画成：

```
┌───────────────────────┐
│ HTTP / HTTPS / DNS    │  应用层
├───────────────────────┤
│ TCP / UDP             │  传输层
├───────────────────────┤
│ IP                    │  网络层
├───────────────────────┤
│ Ethernet / Wi-Fi ...  │  链路层
└───────────────────────┘
```

经典 DNS 可以运行在：

```
UDP
TCP
```

之上，传统 DNS 服务使用 53 端口，IANA 同时登记了 TCP/53 与 UDP/53。现代 DNS 还可以运行在 TLS、HTTPS、QUIC 等加密传输之上。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/8a0852a4dbc949185e966077ad288f8d_MD5.jpg]]

## 二、图解：从输入一个域名到找到服务器

### 2.1 浏览器输入 `www.example.com` 后发生了什么？

假设在浏览器输入：

```
https://www.example.com/
```

浏览器首先拿到的是：

```
协议：https
主机：www.example.com
路径：/
```

问题在于：

```
www.example.com
```

还不是一个能够直接交给 IP 层进行互联网路由的地址。

因此需要先完成：

```
www.example.com
↓
DNS解析
↓
目标IP
```

![](https://relay-1.bijitongbu.site/p/66e97e344d3abeb8601ed20ed5c774af.png)

### 2.2 域名是如何变成IP地址的？

最简单的模型：

```
浏览器
   │
   │ “www.example.com 的 A/AAAA 是什么？”
   ↓
DNS解析系统
   │
   │ 返回地址
   ↓
浏览器
```

浏览器最终获得：

```
IPv4 / IPv6 地址
```

然后才能向目标地址建立连接。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/44ee93c87d62f31b7074bf13a6474da7_MD5.jpg]]

### 2.3 DNS解析在整个网络访问过程中的位置

完整流程可以简化为：

```
输入 URL
   ↓
解析 URL
   ↓
DNS 查询
   ↓
得到目标地址
   ↓
建立传输连接
   ↓
TLS（如果使用 HTTPS）
   ↓
HTTP 请求
   ↓
服务器响应
   ↓
浏览器渲染页面
```

这里说“建立传输连接”比简单说“建立 TCP”更加准确。

因为传统：

```
HTTP/1.1
HTTP/2
```

通常运行在 TCP 上。

而：

```
HTTP/3
```

运行在 QUIC 上，QUIC 又基于 UDP。

所以：

> **DNS 通常发生在应用向目标主机建立网络会话之前，但 DNS 完成以后不一定永远都是 TCP。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/19d14ad45dea64d644e31e7a54df9690_MD5.jpg]]

### 2.4 DNS解析完成后，HTTP/TCP才真正开始工作

假设最后 DNS 返回：

```
www.example.com
↓
某个 IP
```

浏览器接下来才知道：

```
目标 IP 是谁
```

之后可以执行类似：

```
DNS
↓
IP
↓
TCP/QUIC
↓
TLS
↓
HTTP
```

当然，如果：

```
DNS 缓存命中
连接已经存在
浏览器进行了预解析
```

实际过程可能被优化。

但是从逻辑依赖来说：

> **如果连目标地址都不知道，就谈不上向目标服务器发送正常的 IP 数据包。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/06efa18c51fdfa6785de55808b35742f_MD5.jpg]]

### 2.5 一张图看懂完整访问过程

```
用户
 │
 │ 输入 https://www.example.com
 ↓
浏览器
 │
 │ 需要知道 www.example.com 在哪里
 ↓
DNS解析
 │
 │ 返回目标 IP
 ↓
浏览器
 │
 │ 向目标 IP 建立 TCP / QUIC 等连接
 ↓
TLS
 │
 │ HTTPS 建立安全会话
 ↓
HTTP
 │
 │ GET /
 ↓
目标服务器 / CDN
 │
 │ 返回 HTML
 ↓
浏览器解析网页
 │
 ├── CSS 域名可能继续 DNS
 ├── JS 域名可能继续 DNS
 ├── 图片域名可能继续 DNS
 └── API 域名可能继续 DNS
```

所以真实网页访问里面，DNS 往往不是只出现一次。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3b975ed6f461e9cb4f26d9a9af54d6fc_MD5.jpg]]

## 三、DNS诞生之前：Hosts文件时代

### 3.1 最早的互联网是怎么通过名字找服务器的？

DNS 出现以前，互联网规模还比较小，可以依靠一个集中维护的主机信息表。

历史上的：

```
HOSTS.TXT
```

记录主机名和地址之间的关系。

RFC 952 描述了当时 Internet Host Table 的标准格式，并指出机器可读取版本以 `HOSTS.TXT` 的形式由 NIC 提供；后来 DNS 标准明确把它描述为被 DNS 取代的主机/地址表。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d63e6762d4350b12362234a03b6270c5_MD5.jpg]]

### 3.2 Hosts文件是什么？

今天 Linux 里面仍然存在：

```
/etc/hosts
```

Windows 里面也有 hosts 文件。

Linux 中可以写：

```
192.0.2.10    test.example.com
```

那么系统名字解析策略允许的情况下：

```
ping test.example.com
```

就可以把：

```
test.example.com
```

解析到：

```
192.0.2.10
```

而不一定需要查询外部 DNS。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7fbc22699b4effd93488923280f12e23_MD5.jpg]]

### 3.3 Hosts文件如何实现“域名 → IP地址”映射？

它本质上非常简单：

```
IP地址        名字
```

例如：

```
127.0.0.1     localhost
192.0.2.10    server-a.example
192.0.2.11    server-b.example
```

可以把它理解成一本本地电话簿：

```
张三 → 138xxxx
李四 → 139xxxx
```

直接查表即可。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f7f5eb2d7986e99805ea7ac3d8cb3216_MD5.jpg]]

### 3.4 Hosts文件的问题

小规模网络没有问题。

如果世界上只有：

```
几十台主机
```

维护一个文件很简单。

但如果变成：

```
1000台
100万台
10亿台
```

问题就出现了：

```
文件越来越大
更新越来越频繁
集中维护压力越来越高
客户端同步困难
修改传播缓慢
名字管理容易冲突
单一管理机构成为瓶颈
```

也就是说：

> **Hosts 文件真正的问题不是“不能保存名字”，而是无法优雅地扩展到全球互联网规模。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1b7569c9b168dc52a6e5721bf7941412_MD5.jpg]]

### 3.5 为什么最终必须诞生DNS？

解决办法就是把：

```
一张巨大的中央表
```

改成：

```
一套分层、分布式管理体系
```

例如：

```
根
↓
.com
↓
example.com
↓
www.example.com
```

根只需要管理下一层。

`.com` 又只需要负责：

```
.com
```

下面的委派。

`example.com` 自己的管理员则管理：

```
www.example.com
mail.example.com
api.example.com
```

这样管理责任就被逐层分散。

这正是 DNS 原始架构的核心设计：一个层次化名称空间被划分成可由不同组织管理的区域。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/60df280f09ba89cd56899af7a79be568_MD5.jpg]]

## 四、域名到底是什么？

### 4.1 域名和IP地址有什么区别？

可以简单理解：

```
域名
=
名字

IP
=
网络地址
```

例如：

```
www.example.com
```

告诉我们：

> “我想访问谁。”

而某个 IP 地址告诉网络：

> “数据包应该往哪里发送。”

二者并不是一回事。

同一个域名完全可以对应：

```
多个IP
```

同一个 IP 也完全可以承载：

```
多个域名
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/b2efee043b48984cc5056d18d48db914_MD5.jpg]]

### 4.2 `www.baidu.com`应该从左往右看还是从右往左看？

从 DNS 层级来看，应该：

**从右往左看。**

例如：

```
www.example.com
```

拆开：

```
www
example
com
.
```

真正的层级：

```
.
└── com
    └── example
        └── www
```

越往右：

```
层级越高
```

越往左：

```
层级越具体
```

DNS 名称空间在标准中就是一棵自根向下的树。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ddf4c77a75a39f04df3825012d5e5441_MD5.jpg]]

### 4.3 根域名 `.`

DNS 的最高层是：

```
.
```

通常我们输入：

```
www.example.com
```

最后那个点会被省略。

完整写法其实可以表示成：

```
www.example.com.
```

最后面的：

```
.
```

代表 DNS 根。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/2e33be4e054c33efb2af6435f5d43361_MD5.jpg]]

### 4.4 顶级域名 `.com`

根下面：

```
com
org
net
cn
uk
...
```

属于顶级域：

**TLD——Top-Level Domain。**

IANA 根区数据库维护顶级域的委派信息，其中既包括 `.com` 等通用顶级域，也包括 `.uk` 等国家和地区代码顶级域。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1b4240032eb99952bef0d16d8b26d7a0_MD5.jpg]]

### 4.5 二级域名 `example.com`

在：

```
.com
```

下面注册：

```
example
```

以后得到：

```
example.com
```

日常语境里经常称为二级域名。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f8645c741ca8493ffaa4e09b40f81303_MD5.jpg]]

### 4.6 子域名 `www.example.com`

在：

```
example.com
```

之下还可以继续创建：

```
www.example.com
mail.example.com
api.example.com
```

甚至：

```
v1.api.example.com
```

DNS 的层级理论上可以继续向下扩展，但协议对单个 label 以及完整名字的编码长度存在限制；经典 DNS 规定单个 label 最长 63 octets，完整编码名字最长 255 octets。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/91934d3fc978e897a1f58821b06d1ea5_MD5.jpg]]

### 4.7 完整限定域名FQDN是什么？

FQDN：

**Fully Qualified Domain Name。**

强调名字是完整的、绝对的。

例如：

```
www.example.com.
```

从：

```
www
```

一直写到：

```
根 .
```

RFC 1035 的 master file 规则指出，以点结尾的域名是 absolute name；不以点结尾的名字在区域文件语境中可能是相对于当前 origin 的 relative name。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7922805667c06169a85cc46712c49ce7_MD5.jpg]]

## 五、图解：DNS的树形层级结构

### 5.1 DNS为什么设计成树形结构？

如果所有人都在同一级别随便创建：

```
mail
www
shop
server
```

很容易冲突。

树形结构则可以逐层授权：

```
.
↓
com
↓
example
↓
www
```

不同组织管理不同的子树。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e25ec7852839ad343931944e0f7f67f5_MD5.jpg]]

### 5.2从根域到主机名

完整结构：

```
.
                         │
             ┌───────────┼────────────┐
             │           │            │
            com         org          cn
             │
        ┌────┴─────┐
        │          │
     example      other
        │
   ┌────┼────┐
   │    │    │
  www  mail api
```

例如：

```
www.example.com.
```

从右往左：

```
.              根
com            顶级域
example.com    下一级域
www.example.com 更具体的名字
```

这里“www”日常常被称作主机名，但是 DNS label 本身并不强制一定代表一台物理主机。

### 5.3 一张图看懂DNS命名空间

```
根域 .
│
├── com
│   │
│   ├── example.com
│   │   ├── www.example.com
│   │   ├── mail.example.com
│   │   └── api.example.com
│   │
│   └── company.com
│
├── org
│
├── net
│
└── cn
```

本质：

> **DNS不是一张扁平表，而是一棵能够逐层授权管理的树。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e7951ac2006dc788e816a36e99594a92_MD5.jpg]]

## 六、DNS服务器为什么不是只有一台？

### 6.1 如果全世界只有一台DNS服务器会发生什么？

假设全世界所有 DNS 请求都发送：

```
同一台机器
```

这台服务器需要回答全球所有：

```
手机
电脑
服务器
IoT
路由器
云服务
```

的名字查询。

显然不可行。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f81779ef3df69fc1f4693c7fcf3b99ff_MD5.jpg]]

### 6.2 单点故障问题

一旦这台 DNS：

```
宕机
断电
网络中断
受到攻击
```

整个互联网名字解析都会受到影响。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/35339a5fb1b72323beec0d09f693ffef_MD5.jpg]]

### 6.3 性能瓶颈问题

全球每秒巨量查询全部涌入：

```
一个节点
```

服务器处理能力和网络带宽都会成为瓶颈。

### 6.4 全球网络延迟问题

如果唯一 DNS 在美国：

```
中国用户
↓
跨洋网络
↓
美国DNS
↓
跨洋返回
```

光网络传播就产生很高延迟。

### 6.5 DNS为什么采用分布式架构？

所以 DNS 采用：

```
分层
+
分布式
+
缓存
+
冗余
```

的体系。

甚至大家常听到的“13 台根服务器”也不能理解成全球只有 13 台物理机器。

截至 2026 年 8 月，根服务器体系配置为 **13 个命名的根服务器标识，由 12 个独立运营组织负责**，但这些标识通过 Anycast 等方式对应世界各地大量实际服务实例；IANA 同样明确指出根服务器是分布于多个国家的大规模服务器网络。

所以：

```
13
```

更准确地说是：

> **13 个根服务器逻辑身份/命名权威入口。**

而不是：

> “地球上真的就摆着 13 台电脑。”

## 七、DNS系统中的四类核心服务器

理解 DNS 解析之前，先认识四个角色。

### 7.1 本地DNS服务器 / DNS递归解析器

你的电脑一般不会自己从根开始把整个 DNS 树走一遍。

通常会把问题交给：

**Recursive Resolver——递归解析器。**

例如：

```
运营商 DNS
公司内部 DNS
路由器后面的解析器
Google Public DNS
Cloudflare 1.1.1.1
阿里公共 DNS
DNSPod Public DNS
```

用户问：

> `www.example.com` 是多少？

递归解析器负责：

> “我替你把最终答案查回来。”

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9d05d2e1b4fd1dc26e260f4d0fbd188f_MD5.jpg]]

### 7.2 根DNS服务器 Root Server

根服务器负责 DNS：

```
根区域 .
```

它通常不会直接告诉你：

```
www.example.com = 某IP
```

而更像告诉递归解析器：

> “你这个名字属于 `.com`，去问负责 `.com` 的服务器。”

根区主要包含对各顶级域的委派信息。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0c7fda7c819e4cd0a87f3716041e2dc6_MD5.jpg]]

### 7.3 顶级域DNS服务器 TLD Server

`.com` TLD DNS 负责：

```
.com
```

这一层相关的委派。

递归解析器问：

> `www.example.com` 去哪里找？

`.com` 服务器会告诉：

> `example.com` 的权威 DNS 在这些服务器上。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/6809aaa85165cbd1b90131b532e8b0f5_MD5.jpg]]

### 7.4 权威DNS服务器 Authoritative DNS Server

最终：

```
example.com
```

对应的权威 DNS 服务器管理该 zone 的数据。

它可能真正保存：

```
www.example.com A ...
mail.example.com MX ...
example.com TXT ...
```

等记录。

它才是：

> **对于自己所管理区域具有权威数据来源意义的服务器。**

DNS 最新术语标准 RFC 9499 对 recursive resolver、authoritative server、zone、delegation 等概念进行了统一定义。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9d4df4ab123fc35dd4ad411f434b9fe2_MD5.jpg]]

### 7.5 四类服务器分别负责什么？

```
客户端
  │
  │ 我想知道 www.example.com
  ▼
递归解析器
  │
  │ 该去哪找 .com？
  ▼
根服务器
  │
  │ 去问 .com TLD
  ▼
.com TLD
  │
  │ example.com 权威服务器在这里
  ▼
example.com 权威DNS
  │
  │ www.example.com 的记录是……
  ▼
递归解析器
  │
  │ 缓存后返回
  ▼
客户端
```

一句话：

```
客户端：
我只想拿最终答案

递归解析器：
我负责帮你跑腿

根：
告诉你顶级域去哪找

TLD：
告诉你目标 zone 的权威服务器去哪找

权威DNS：
真正给出自己管理区域的数据
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/00febb8ba36cfbea4d640383d2fd2f38_MD5.jpg]]

## 八、图解：DNS完整域名解析过程

假设访问：

```
www.example.com
```

这里先按照一个适合理解原理的典型模型讲解。

**真实系统中的本地查询顺序并不是所有操作系统和浏览器都完全一样。** 浏览器可能拥有自己的 DNS cache、DoH 解析逻辑；Linux 还可能受到 NSS、systemd-resolved 等配置影响。因此下面应该理解成“典型逻辑路径”，而不是所有机器永远固定不变的调用顺序。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f5324985608721792a54d288d4a4c2f7_MD5.jpg]]

### 8.1 先在本地寻找答案

浏览器首先可能检查：

```
浏览器自身缓存
```

如果没有，则进入系统名字解析体系。

系统还可能从：

```
DNS客户端缓存
Hosts
本地解析服务
```

等地方获得结果。

都没有答案以后，请求才需要到配置的递归 DNS。

因此简化为：

```
www.example.com
↓
本地有没有现成答案？
├── 有 → 直接返回
└── 没有
     ↓
递归DNS
```

### 8.2 本地DNS询问根DNS服务器

递归解析器：

> 我要找 `www.example.com`。

根：

> 我不知道最终 IP，但是我知道 `.com` 应该去哪里找。

返回：

```
.com 的 NS 委派信息
```

### 8.3 继续询问 `.com`

递归解析器再去找：

```
.com TLD Server
```

问：

> `example.com` 由谁负责？

TLD：

> 这些是 `example.com` 的权威 nameserver。

返回类似：

```
example.com NS ns1...
example.com NS ns2...
```

必要时还可能附带：

```
Glue Record
```

帮助找到 nameserver 地址。

### 8.4 找到权威DNS

递归解析器：

```
根
↓
.com
↓
example.com 权威DNS
```

最终问：

> `www.example.com` 的 A 记录是什么？

权威服务器返回：

```
Answer
```

### 8.5 DNS结果返回给用户

递归服务器拿到答案以后：

```
保存缓存
↓
返回客户端
```

缓存多久由 TTL 等规则决定。

### 8.6 浏览器开始连接目标服务器

DNS 阶段结束以后：

```
域名
↓
IP
```

于是后面才进入：

```
IP路由
↓
TCP / QUIC
↓
TLS
↓
HTTP
```

完整图：

```
客户端
  │
  │ www.example.com?
  ▼
递归 DNS
  │
  ├───────────────→ Root
  │                  │
  │      .com 在这里 │
  │←─────────────────┘
  │
  ├───────────────→ .com TLD
  │                  │
  │ example权威在这里│
  │←─────────────────┘
  │
  ├───────────────→ Authoritative
  │                  │
  │      最终 RR     │
  │←─────────────────┘
  │
  │ 缓存
  ▼
客户端
```

这就是经典 DNS 递归解析的核心路径。

## 九、DNS递归查询和迭代查询

### 9.1 什么是递归查询？

递归查询可以理解成：

> **“你不要告诉我下一步去问谁，我只要最终答案。”**

例如客户端：

```
客户端 → 本地 DNS

www.example.com 是多少？
```

本地 DNS：

> 好，我帮你查到最后。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0cb23389ed31f01ac4d1d78068eb69e2_MD5.jpg]]

### 9.2 什么是迭代查询？

迭代查询更像：

> **“如果你不知道最终答案，就告诉我下一步应该问谁。”**

例如：

```
递归DNS → Root
```

Root：

> 去问 `.com`。

然后：

```
递归DNS → .com
```

TLD：

> 去问 `example.com` 的权威服务器。

不断逼近最终答案。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/cdbd8cb2cdbeaf24d830b900fdac45a0_MD5.jpg]]

### 9.3 客户端和本地DNS之间为什么通常是递归查询？

因为普通客户端不希望：

```
自己维护 root hints
自己处理 delegation
自己缓存大量 NS
自己跟踪整个 DNS 树
```

于是把复杂工作交给：

```
递归解析器
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e4fe1258edd09e37b837139096bf56c6_MD5.jpg]]

### 9.4 本地DNS与其他DNS服务器之间为什么通常是迭代查询？

根和 TLD 没必要：

```
替全球客户端递归跑完整流程
```

否则它们的压力会巨大。

所以它们返回：

```
referral
```

告诉递归服务器：

> 下一步去这里。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f2b733b3b5e866ca7d95430036351bf2_MD5.jpg]]

### 9.5 图解递归查询

```
客户端
   │
   │ “请给我最终答案”
   ▼
递归DNS
   │
   │ 自己完成后面的解析
   │
   ▼
最终答案
   │
   ▼
客户端
```

### 9.6 图解迭代查询

```
解析器 → Root
           │
           └→ 去 .com

解析器 → .com
           │
           └→ 去 example 权威DNS

解析器 → 权威DNS
           │
           └→ 最终答案
```

DNS 标准术语明确区分 recursive query 和 iterative resolution；经典解析算法同样依靠 referral 逐级逼近权威数据。

### 9.7 两种方式有什么区别？

| 对比 | 递归查询 | 迭代查询 |
| :-- | :-- | :-- |
| 目标 | 要最终结果 | 要当前服务器能提供的最好信息 |
| 谁继续查 | 被请求服务器 | 查询方 |
| 常见位置 | Stub → Recursive Resolver | Recursive Resolver → Authoritative hierarchy |
| 返回内容 | 最终答案/失败 | 答案或 referral |

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5d703a401e0686b636c5b07a6ece5f9a_MD5.jpg]]

### 十、DNS为什么查询速度这么快？——DNS缓存

### 10.1 如果每次都从根DNS开始查询会怎么样？

每访问一次：

```
www.example.com
```

都执行：

```
Root
↓
TLD
↓
Authoritative
```

那么：

```
延迟变高
Root压力增大
TLD压力增大
权威DNS压力增大
```

所以 DNS 极度依赖：

**Cache。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a75fc7ab31b9f6127468cd5745850313_MD5.jpg]]

### 10.2 DNS缓存在哪里？

可能包括：

```
浏览器缓存
系统 DNS client/cache
本地递归解析器缓存
上游转发递归解析器缓存
```

需要注意：

> **根服务器、TLD服务器、普通权威服务器的主要角色是提供权威数据，不应该简单统称为“帮客户端层层缓存最终答案的 DNS 服务器”。**

真正承担大量缓存工作的主要是 recursive/caching resolver。

### 10.3 TTL是什么？

TTL：

**Time To Live。**

DNS Resource Record 中存在 TTL。

例如：

```
www.example.com. 300 IN A ...
```

这里：

```
300
```

意味着缓存者通常最多可以按规则把这份记录继续认为有效：

```
300 秒
```

DNS RR 的标准 wire format 本身就包含 TTL 字段。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7c19cc71367d8788c35f9ad9c3e08b18_MD5.jpg]]

### 10.4 TTL过长有什么问题？

假设：

```
TTL = 86400
```

也就是一天。

服务器 IP 在上午 10 点修改。

部分递归 DNS 仍有旧缓存：

```
旧IP
```

那么理论上可能继续使用缓存，直到剩余 TTL 到期。

所以：

```
TTL越长
↓
缓存命中率越高
查询压力越低
↓
但变更传播可能越慢
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/32ca53a57649da094f9db9ad0c929c0b_MD5.jpg]]

### 10.5 TTL过短有什么问题？

例如：

```
TTL = 10
```

意味着缓存很快失效。

于是：

```
查询频率提高
权威 DNS 压力提高
递归查询次数增加
```

因此 TTL 是：

```
缓存效率
vs
变更敏捷性
```

之间的权衡。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0dd1036abb0a26ede8fc6147d9e1368c_MD5.jpg]]

### 10.6 为什么修改DNS记录后不会立即生效？

非常重要：

> **修改权威 DNS 记录，并不会神奇地把全球所有递归服务器已经保存的旧缓存瞬间删除。**

假设旧记录：

```
TTL 3600
```

某解析器：

```
10:00 缓存旧 IP
```

10:05 你修改 DNS。

这个解析器可能仍然继续使用旧缓存，直到它自己的 TTL 生命周期结束。

这就是常说的：

```
DNS propagation
```

中非常重要的一部分。

DNS 对 NXDOMAIN/NODATA 等负面答案也存在负缓存规则；RFC 2308 规定了 negative caching 的处理，而 RFC 9520 又更新了部分解析失败的负缓存要求。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d909b8b16f7f3c3504c05c5951205476_MD5.jpg]]

## 十一、DNS中最重要的资源记录类型

IANA 维护 DNS Resource Record 类型注册表，DNS 远远不止 A 记录。

### 11.1 A记录

```
域名 → IPv4
```

例如：

```
www.example.com. 300 IN A 192.0.2.10
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/954fc366c3ab796e85e7c1596cfa57a6_MD5.jpg]]

### 11.2 AAAA记录

```
域名 → IPv6
```

例如：

```
www.example.com. 300 IN AAAA 2001:db8::10
```

AAAA 是 DNS 支持 IPv6 的标准地址记录。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/32f6f0a426bab1658cb03f1aa3c2524a_MD5.jpg]]

### 11.3 CNAME记录

Canonical Name：

```
一个名字
↓
另一个名字
```

例如：

```
www.example.com. CNAME web.example.net.
```

意思不是：

```
www.example.com = IP
```

而是：

> `www.example.com` 是 `web.example.net` 的别名，继续解析后者。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/67549363804b03b60991350c5f29ff97_MD5.jpg]]

### 11.4 MX记录

Mail Exchange：

```
域名
↓
邮件服务器
```

例如：

```
example.com. MX 10 mail.example.com.
```

数值表示 preference，通常数值较小的服务器具有更高优先级。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/475b77065786338a9c2cb406c493e009_MD5.jpg]]

### 11.5 NS记录

Name Server：

```
这个 zone / delegation 由哪些 DNS nameserver 负责
```

例如：

```
example.com. NS ns1.example.net.
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/79ab3e34c6d524cf7edd9dd58947eb4d_MD5.jpg]]

### 11.6 TXT记录

用于保存文本字符串数据。

非常常见于：

```
SPF
DKIM相关数据
域名所有权验证
各种服务验证
安全策略
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3cda50c0bd3406ceed2d327022e3653f_MD5.jpg]]

### 11.7 PTR记录

PTR 用于把一个特殊反向 DNS 名字：

```
→ 域名
```

最常见就是：

```
IP反查名字
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5c091610acea61e71ec958b32d46e89e_MD5.jpg]]

### 11.8 SOA记录

SOA：

**Start of Authority。**

它描述一个 zone 的关键管理信息，包括：

```
主名称服务器
管理员信息
Serial
Refresh
Retry
Expire
Minimum等字段
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3dbdf1346a0d2d64283f53ae5df00c6a_MD5.jpg]]

### 11.9 SRV记录

SRV 不只是说：

```
域名在哪里
```

而能够描述：

```
某个服务
使用什么协议
端口是多少
目标主机是谁
优先级
权重
```

RFC 2782 专门定义了 SRV 用来定位某个 domain 中特定 protocol/service 的服务器。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/48d37d89317ca2458d1730f6de0bd655_MD5.jpg]]

### 11.10 CAA记录

CAA 用来让域名持有者声明：

> 哪些 CA 被授权为这个域名签发证书。

RFC 8659 是当前 CAA 核心规范。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a3b6d35208b0854cc5494df87414bc90_MD5.jpg]]

### 11.11 常见记录快速对照

```
A       → IPv4
AAAA    → IPv6
CNAME   → 别名目标域名
MX      → 邮件服务器
NS      → 名称服务器
TXT     → 文本数据
PTR     → 反向指向域名
SOA     → Zone权威起始/管理信息
SRV     → 服务位置
CAA     → CA签发策略
```

## 十二、A记录和CNAME记录到底有什么区别？

### 12.1 A记录直接指向IP地址

```
www.example.com
↓
A
↓
192.0.2.10
```

一步得到 IPv4。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/b59c4741279e48eaa4571735ef27db05_MD5.jpg]]

### 12.2 CNAME记录指向另一个域名

```
www.example.com
↓
CNAME
↓
service.provider.example
↓
A / AAAA
↓
最终IP
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1c3fbde3a087d575dacd18bb33531a33_MD5.jpg]]

### 12.3 为什么要设计CNAME？

假设你的网站实际运行在第三方平台：

```
customer.cdn-provider.example
```

你不希望把平台 IP：

```
写死到自己 DNS
```

因为平台可能更换服务器。

于是可以：

```
www.example.com
CNAME
customer.cdn-provider.example
```

以后底层地址变化由服务提供方管理。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e477b48bb01129e02705353b913b6253_MD5.jpg]]

### 12.4 CNAME为什么可能增加一次DNS查询？

因为客户端/递归器拿到：

```
CNAME
```

以后，还需要确保目标名字对应的：

```
A/AAAA
```

已经获得。

不过这并不意味着：

> 每次看到 CNAME 都一定产生一个额外的独立网络 RTT。

因为：

```
目标记录可能已经缓存
响应可能携带相关附加信息
```

![](https://relay-1.bijitongbu.site/p/6f4ca08af17a607cf3b1625687f91ea6.png)

### 12.5 Zone Apex为什么通常不能直接使用CNAME？

经常有人说：

> “根域名不能用 CNAME。”

这里更准确的术语应该是：

**Zone Apex。**

例如：

```
example.com.
```

如果它是这个 zone 的 apex，那么这里必须存在：

```
SOA
NS
```

等数据。

而 CNAME 的核心规则是：

> 一个名字作为 CNAME owner 时，不能同时拥有普通的其他数据。

因此 zone apex 无法按照标准方式把自己简单做成 CNAME。

RFC 1034 已规定 CNAME 节点不能与其他普通数据共存；RFC 2181 进一步澄清相关 CNAME 规则。

一些 DNS 服务商提供：

```
ALIAS
ANAME
CNAME Flattening
```

等能力来实现类似体验，但这些不是把 DNS 标准中的 apex CNAME 规则直接取消了。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/59964148dbdcfe38cabc40bb4deab523_MD5.jpg]]

### 12.6 什么场景应该使用A记录？

当你明确控制目标 IPv4：

```
www.example.com → 192.0.2.10
```

使用 A 非常自然。

### 12.7 什么场景应该使用CNAME？

当目标由：

```
CDN
云平台
SaaS
第三方托管服务
```

管理，而且服务商给你的配置目标是：

```
另一个 DNS name
```

时非常适合。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/708855433994723d9ccd989467bd2ea0_MD5.jpg]]

## 十三、DNS区域、Zone和域名到底是什么关系？

### 13.1 Domain是什么？

Domain 可以从 DNS 名称空间角度理解。

例如：

```
example.com
```

下面理论上包含：

```
www.example.com
mail.example.com
api.example.com
sub.example.com
```

这一整棵子树。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/23e63a64da01764bcd3bddfbecbe6461_MD5.jpg]]

### 13.2 Zone是什么？

Zone 强调的是：

> **一份由特定权威管理边界负责的数据集合。**

DNS 术语标准把 authoritative information 组织成 zone。

### 13.3 Domain和Zone为什么不是完全相同？

假设：

```
example.com
├── www.example.com
├── mail.example.com
└── dev.example.com
```

一开始可能：

```
整个 example.com
```

都属于一个 zone。

后来把：

```
dev.example.com
```

委派给另外一组权威 DNS：

```
example.com zone
├── www
├── mail
└── dev → Delegation
          ↓
       dev.example.com zone
```

这时从名字树来说：

```
dev.example.com
```

仍然位于：

```
example.com
```

这个 domain 子树下面。

但从权威管理边界来说：

```
dev.example.com
```

已经是新的 zone。

![](https://relay-1.bijitongbu.site/p/686e61c1cd3bb1f7c81844c1e98a29fc.png)

### 13.4 什么是DNS区域文件？

传统 BIND 风格 zone file 可能类似：

```
$ORIGIN example.com.

@    3600 IN SOA ns1.example.com. admin.example.com. (
              2026081401
              3600
              600
              86400
              300 )

@    3600 IN NS ns1.example.com.
@    3600 IN NS ns2.example.com.

www  300  IN A 192.0.2.10
mail 300  IN A 192.0.2.20
@    300  IN MX 10 mail.example.com.
```

这是 DNS 权威数据的一种经典表示形式。

![](https://relay-1.bijitongbu.site/p/46325f69b5bd9f436ee12a4dfe354bea.png)

### 13.5 什么是域名委派 Delegation？

假设：

```
example.com
```

由管理员 A 管。

但是：

```
dev.example.com
```

想交给开发部门自己管理。

那么父 zone 可以放：

```
dev.example.com NS ns1.dev-provider.example.
dev.example.com NS ns2.dev-provider.example.
```

于是形成：

**Delegation。**

![](https://relay-1.bijitongbu.site/p/3dc4e3199f93a4053c82e54b859e7cfe.png)

### 13.6 NS记录如何实现管理权下放？

父区：

```
我不继续负责这里的全部权威数据
↓
我告诉查询者下一层应该找谁
```

这就是 DNS 能够分布式管理的核心之一。

### 13.7 图解区域委派

```
example.com Zone
│
├── www.example.com
├── mail.example.com
│
└── dev.example.com
       │
       │ NS Delegation
       ▼
dev.example.com Zone
│
├── git.dev.example.com
└── api.dev.example.com
```

## 十四、DNS报文格式详解

### 14.1 DNS报文整体结构

经典 DNS message：

```
+---------------------+
| Header              |
+---------------------+
| Question            |
+---------------------+
| Answer              |
+---------------------+
| Authority           |
+---------------------+
| Additional          |
+---------------------+
```

RFC 1035 定义了 DNS 消息的基本 wire format。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e34b55f54c33a56502bb6108dd6bce39_MD5.jpg]]

### 14.2 Header

固定头部：

```
12 Bytes
```

包含：

```
Transaction ID
Flags
QDCOUNT
ANCOUNT
NSCOUNT
ARCOUNT
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/edb1aa786ed7468f1b3c3e025dfa75f7_MD5.jpg]]

### 14.3 Question

保存：

```
QNAME
QTYPE
QCLASS
```

例如：

```
QNAME  = www.example.com
QTYPE  = A
QCLASS = IN
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0543f41f68bd1b4f5084e791a6dc6f1b_MD5.jpg]]

### 14.4 Answer

真正回答查询问题的数据。

例如：

```
www.example.com A ...
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/bec3da53bf6b621bf4cffb5c8dee6f4c_MD5.jpg]]

### 14.5 Authority

通常包含：

```
权威相关记录
```

例如 referral 里面的 NS，或者负面回答中的 SOA 等。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9bbc8a6a0d25f1ccfbd537f2cc7ef5b5_MD5.jpg]]

### 14.6 Additional

提供额外有帮助的信息。

例如：

```
NS 指向 ns1.example.com
```

如果服务器还能够提供：

```
ns1.example.com 的地址
```

可能放在 Additional section。

EDNS 的：

```
OPT pseudo-RR
```

同样通过 Additional section 承载。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5f370ac7cfa3f4558d1df3efba5ba574_MD5.jpg]]

## 十五、DNS报文头部字段详解

经典 Header 可以理解为：

```
0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+--+--+--+--+--+--+--+--+--+--+--+
|              ID               |
+--+--+--+--+--+--+--+--+--+--+--+
|QR| Opcode |AA|TC|RD|RA|...| RCODE |
+--+--+--+--+--+--+--+--+--+--+--+
|           QDCOUNT             |
+--------------------------------+
|           ANCOUNT             |
+--------------------------------+
|           NSCOUNT             |
+--------------------------------+
|           ARCOUNT             |
+--------------------------------+
```

RFC 1035 定义了基础 Header；后续标准又从原来的保留位中定义了例如 DNSSEC 使用的 AD/CD 等位，因此现代实现中不要简单认为原始三个 Z bits 仍然全部没有含义。

### 15.1 Transaction ID

16 bit 标识。

例如：

```
0x4a31
```

客户端发 Query：

```
ID = 0x4a31
```

响应通常通过对应 ID 帮助匹配请求。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d95224d65598b9d0de146c3259d48874_MD5.jpg]]

### 15.2 QR

```
0 = Query
1 = Response
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5fb658338022cda708df53652b65679a_MD5.jpg]]

### 15.3 Opcode

表示操作类型。

最常见：

```
0 = QUERY
```

IANA 还注册了：

```
NOTIFY
UPDATE
DSO
```

等 Opcode。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/65af91a7c65f7fa364fc8b40a1c2e125_MD5.jpg]]

### 15.4 AA

Authoritative Answer。

服务器表示：

> 这个回答来自我具有权威性的区域数据。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/73498b217084600094ac88f2cc1262c8_MD5.jpg]]

### 15.5 TC

Truncated。

表示：

```
响应被截断
```

在经典 UDP DNS 中非常重要。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/13c44cdc9c3ccf7bf4bf735a999c1b62_MD5.jpg]]

### 15.6 RD

Recursion Desired。

客户端：

> 我希望服务器进行递归。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0139f45cfdefe1f873cdd55cb0b714ff_MD5.jpg]]

### 15.7 RA

Recursion Available。

服务器：

> 我支持递归能力。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f840c6d50df440df4618a3a9b4939804_MD5.jpg]]

### 15.8 Z

原始 RFC 1035 中预留了一组 Z 位并要求为零；后续 DNSSEC 标准从原保留空间中定义了 AD/CD，现代 Header 因而只剩对应仍然保留的位继续要求保持相应规范值。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/c1519a7581595aa8716e136e756d040b_MD5.jpg]]

### 15.9 RCODE

Response Code。

例如：

```
0 NOERROR
2 SERVFAIL
3 NXDOMAIN
5 REFUSED
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/215eb116a00c8de84ef162b733ed2bef_MD5.jpg]]

### 15.10 四个Count

```
QDCOUNT
Question数量

ANCOUNT
Answer RR数量

NSCOUNT
Authority RR数量

ARCOUNT
Additional RR数量
```

它们都是 DNS 报文 Header 的基础字段。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/2f67b350c01b54c837c6e9acaa2365e7_MD5.jpg]]

## 十六、图解：一个真实DNS查询报文

### 16.1 客户端查询 `www.example.com`

可以使用 Wireshark：

```
dns
```

作为显示过滤器。

也可以 Linux：

```
sudo tcpdump -ni any port 53
```

进一步查看 payload：

```
sudo tcpdump -ni any -s0 -vv -X port 53
```

然后另外一个终端：

```
dig www.example.com A
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1f8e3ef214b1efc3b6c1362f3606d823_MD5.jpg]]

### 16.2 DNS Query报文长什么样？

Wireshark 中通常会看到类似：

```
Domain Name System (query)

Transaction ID: 0x1234

Flags:
    Response: Message is a query
    Recursion desired: Do query recursively

Questions: 1
Answer RRs: 0
Authority RRs: 0

Queries:
    www.example.com
        Type: A
        Class: IN
```

Transaction ID 和实际网络数据每次可能不同。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/68daac4832ede94cb562e9430cf0306b_MD5.jpg]]

### 16.3 Question区域保存了什么？

核心就是：

```
QNAME = www.example.com
QTYPE = A
QCLASS = IN
```

也就是说：

> 我问的是 Internet class 下，这个名字的 IPv4 address record。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1492d7190e2dd3338a6126ee51fc019c_MD5.jpg]]

### 16.4 DNS Response报文长什么样？

典型：

```
Domain Name System (response)

Transaction ID: 0x1234

Flags:
    Response: Message is a response
    Recursion desired
    Recursion available

Questions: 1
Answer RRs: 1
```

然后：

```
Answers
    www.example.com
        Type: A
        TTL: ...
        Address: ...
```

IP 和 TTL 以实际抓包结果为准。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/8bf42a9bab786935b8e3c413660e3645_MD5.jpg]]

### 16.5 Answer区域如何返回IP地址？

Resource Record 通用格式包含：

```
NAME
TYPE
CLASS
TTL
RDLENGTH
RDATA
```

如果：

```
TYPE = A
CLASS = IN
```

那么 RDATA 就承载 IPv4 地址数据。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/61114742f51539693ee950eaa38b209a_MD5.jpg]]

### 16.6 Transaction ID如何对应请求和响应？

Query：

```
ID = 0x1234
```

Response：

```
ID = 0x1234
```

这使客户端能够关联两者。

当然现代 DNS 抗伪造不能只依靠一个 16 bit ID，还会结合源端口随机化、DNSSEC 等机制增强安全性。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/975dc5200ba574a6c46a2608b4f437bf_MD5.jpg]]

### 16.7 Wireshark抓包

推荐实验：

```
dig @8.8.8.8 example.com A
```

Wireshark：

```
udp.port == 53
```

然后逐层展开：

```
Ethernet
↓
IP
↓
UDP
↓
DNS
```

你会发现 DNS 并不神秘。

最终就是：

> **一个拥有固定 Header + 多个变长 section 的网络协议报文。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/97ea37ca94b2087f82b7b6cc498ffa85_MD5.jpg]]

## 十七、DNS为什么通常使用UDP，而不是TCP？

### 17.1 DNS默认使用哪个端口？

经典 DNS：

```
53
```

IANA 同时为：

```
domain / TCP 53
domain / UDP 53
```

登记服务端口。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/cdc4c626236e01f3259d1191c90a7826_MD5.jpg]]

### 17.2 UDP 53

最常见的普通 DNS Query：

```
Client
  │ UDP Query
  ▼
DNS Server
  │ UDP Response
  ▼
Client
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ce3c00de36d3274e9268df5a28c72e9b_MD5.jpg]]

### 17.3 TCP 53

DNS 同样必须认真支持 TCP。

现代标准 RFC 7766 专门强化 DNS-over-TCP 的实现要求，不能再把 TCP 简单理解成“几乎可以不实现的备用功能”。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/925b4737369daca86077f1b3f117baf5_MD5.jpg]]

### 17.4 DNS为什么大量使用UDP？

对于短小的传统查询：

```
UDP
```

具有优势：

```
不需要TCP三次握手
报文开销较低
简单请求/响应效率高
```

因此 classic DNS 中 UDP 非常普遍。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9838c66c9911ed7c0d9f44de491d2c4c_MD5.jpg]]

### 17.5 UDP报文过大怎么办？

经典 RFC 1035 在没有扩展时规定 UDP DNS message 上限：

```
512 octets
```

后来：

```
DNSSEC
IPv6记录
更多Additional数据
```

让 512 bytes 越来越容易不够。

于是出现：

```
EDNS
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/acf644dd0c6b5c47722c64edae4e9dd0_MD5.jpg]]

### 17.6 TC标志位是什么？

如果响应无法完整放进当前允许的 UDP 报文：

```
TC = 1
```

表示：

**Truncated。**

客户端可以转向 TCP 等方式重试。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/20357aa1f896237c04eb9c85f55a632d_MD5.jpg]]

### 17.7 DNS什么时候使用TCP？

常见场景：

```
UDP响应被截断
客户端直接选择TCP
较大的DNS响应
Zone Transfer
某些特殊运维/安全策略
```

需要强调：

> “DNS 只有 UDP 超过 512 字节才使用 TCP”是过度简化的旧式口诀。

现代 DNS 软件需要正确支持 TCP，而且允许直接使用 TCP。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9e3d44f08be938d47683c394ec4c71a1_MD5.jpg]]

### 17.8 DNS区域传送为什么通常使用TCP？

例如：

```
AXFR
```

可能需要传输一个完整 zone 的大量记录。

显然不是一两个短 UDP datagram 能舒服解决的问题。

完整区域传送 AXFR 的现代规范要求基于 TCP 传输。

## 十八、EDNS是什么？为什么现代DNS需要EDNS？

### 18.1 传统DNS UDP报文大小限制

最初经典 DNS：

```
UDP <= 512 bytes
```

在上世纪八十年代非常合理。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/326a32b7a933bfb706dd7caf641a0d0c_MD5.jpg]]

### 18.2 512字节为什么逐渐不够用了？

后来 DNS 增加：

```
IPv6
DNSSEC
更多RR
各种扩展
```

特别是 DNSSEC：

```
DNSKEY
RRSIG
DS
```

明显增加报文大小。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/595ed007b5eab97ec6158bc4ff4ca175_MD5.jpg]]

### 18.3 EDNS的诞生

EDNS：

**Extension Mechanisms for DNS。**

当前核心规范是：

```
RFC 6891
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/89404483a2be3b312b88c31005f07208_MD5.jpg]]

### 18.4 EDNS如何扩大UDP报文大小？

客户端在查询中告诉服务器：

> 我可以接收比经典 512 bytes 更大的 UDP DNS payload。

于是双方不必永远局限于：

```
512
```

注意：

> EDNS 不是简单地把 DNS 头部某个“长度字段”从 512 改大。

它借助一种特殊的：

```
OPT
```

pseudo-resource-record 表达扩展能力。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5c55d32554712f4a7a0210c7d9160d4e_MD5.jpg]]

### 18.5 OPT伪记录是什么？

OPT：

```
不是普通域名数据记录
```

它属于：

```
协议扩展元数据
```

例如告诉服务器：

```
EDNS版本
可接收UDP payload大小
DNSSEC相关标志
EDNS option
```

RFC 6891 明确规定 OPT RR 不应该被缓存。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0a47ad24b3c7884561a80b01743a1a8e_MD5.jpg]]

### 18.6 EDNS和DNSSEC有什么关系？

DNSSEC 经常产生更大的响应。

如果仍严格限制：

```
512 bytes
```

大量响应会频繁截断。

EDNS 为 DNSSEC 等扩展提供了更实际的承载空间。

但仍然要注意：

```
UDP报得越大
```

并不代表越好。

网络路径 MTU、IP 分片以及丢包都必须考虑。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0b5cdedca191f3b82ef864be35592d1d_MD5.jpg]]

## 十九、正向DNS解析和反向DNS解析

### 19.1 什么是正向解析？

我们平常最熟悉：

```
域名
↓
IP
```

例如：

```
www.example.com
↓
A / AAAA
↓
IP
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e9ed16b0d9484206ea1e0e4288f0d78d_MD5.jpg]]

---

### 19.2 域名如何解析为IP地址？

IPv4：

```
A
```

IPv6：

```
AAAA
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/4fb5df99f8591cd8cf90b466ea4c56db_MD5.jpg]]

---

### 19.3 什么是反向解析？

反过来：

```
IP
↓
域名
```

但是 DNS 不会直接问：

```
“192.0.2.10对应什么？”
```

而是把地址转换为特殊 DNS 名字。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0fe22835bafbc7260496956b078141f7_MD5.jpg]]

### 19.4 IPv4和 `in-addr.arpa`

IPv4：

```
192.0.2.10
```

反向 DNS 名称按照字节反转：

```
10.2.0.192.in-addr.arpa.
```

然后查询：

```
PTR
```

记录。

RFC 1035 就定义了 PTR，并以 IN-ADDR.ARPA 作为经典反向解析用途。

IPv6 则使用：

```
ip6.arpa
```

并按照 nibble 反转形成反向名称。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f4126e346d36705e955013b6b28ca325_MD5.jpg]]

### 19.5 反向DNS有什么实际用途？

常见：

```
邮件系统信誉/反垃圾
服务器日志展示
网络排障
资产识别
运维工具
```

需要注意：

```
PTR存在
```

不代表：

```
一定能够证明对端身份可信
```

它本质仍然是一条 DNS 数据。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/89136245592ce689346c68b4324a7641_MD5.jpg]]

## 二十、公共DNS和本地DNS有什么区别？

### 20.1 什么是运营商DNS？

家庭宽带或者移动网络通常通过：

```
DHCP
PPP
网络配置
```

给客户端提供 DNS resolver。

很多时候就是运营商提供的递归 DNS。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/c62d337404085f372817eb42b761db1d_MD5.jpg]]

### 20.2 什么是公共DNS？

公共 DNS：

> 面向互联网用户公开提供递归解析服务的 resolver。

例如：

```
Google Public DNS
Cloudflare 1.1.1.1
AliDNS Public DNS
DNSPod Public DNS
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d660437c02b629891c17f293187ea1fc_MD5.jpg]]

---

### 20.3 `8.8.8.8`是什么？

Google Public DNS 的经典 IPv4 地址包括：

```
8.8.8.8
8.8.4.4
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/58bfe1b91fa729bfcb8cf892a75b43a4_MD5.jpg]]

### 20.4 `1.1.1.1`是什么？

Cloudflare Public DNS 标准 resolver：

```
1.1.1.1
1.0.0.1
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/25a04acff0a3fa4b2f7c6507363de1c4_MD5.jpg]]

### 20.5 国内常见公共DNS是什么？

截至 2026 年，官方仍公开提供例如：

```
阿里公共DNS
223.5.5.5
223.6.6.6
```

阿里云官方文档当前将它们列为免费版公共递归 DNS 服务地址。

DNSPod 官方资料仍列出：

```
119.29.29.29
```

作为公共递归服务地址。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/2dc30d09fccb2e9c643c81488a312ec3_MD5.jpg]]

### 20.6 修改DNS服务器到底改变了什么？

比如从运营商 DNS：

```
→ 1.1.1.1
```

实际上改变的是：

> **客户端首先把递归 DNS 请求交给哪个 resolver。**

不是改变：

```
你的IP
网站服务器
路由器公网地址
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/174db69ee9bdfeb5fdfdca866eda25fd_MD5.jpg]]

### 20.7 DNS服务器会影响上网速度吗？

**可能影响，但不要夸大。**

主要影响：

```
首次解析延迟
缓存命中
解析稳定性
CDN调度结果
```

DNS 查询结束以后，大文件下载速度主要取决于：

```
服务器/CDN
路由
带宽
拥塞
TCP/QUIC等
```

而不是 DNS 一直参与传输。

另外不同 resolver 所在位置不同，也可能使 CDN 权威系统给出不同地址；ECS 等机制还允许递归服务器携带客户端网络前缀信息辅助权威侧调度。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/84ca753469cfa2a1f4c820a4669a054e_MD5.jpg]]

## 二十一、DNS负载均衡是怎么实现的？

### 21.1 一个域名可以对应多个IP地址吗？

当然可以。

例如：

```
www.example.com A 192.0.2.10
www.example.com A 192.0.2.11
www.example.com A 192.0.2.12
```

形成一个 RRset。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/da661f7cad49cc52e446018e70338784_MD5.jpg]]

### 21.2 DNS轮询是什么？

服务器可以返回：

```
多个IP
```

并通过：

```
顺序调整
权重策略
```

等方式帮助分流。

最简单通常称为：

**DNS Round Robin。**

但：

> DNS 返回多个地址，并不意味着客户端一定按照服务器返回顺序机械地每人轮流一个。

客户端选择算法、缓存、中间 resolver 都会影响最终行为。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ce6c24adb3adb75572c2d313e2215cb1_MD5.jpg]]

### 21.3 不同地区为什么可能解析到不同IP？

例如：

```
北京用户
↓
北京边缘节点

广州用户
↓
广州边缘节点
```

权威 DNS 可以根据：

```
请求来源
递归DNS位置
ECS
业务策略
健康状态
```

返回不同结果。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5ca3da02baf57571656a259b933c92ff_MD5.jpg]]

### 21.4 GeoDNS是什么？

GeoDNS 可以理解成：

> 根据查询来源的地理/网络属性返回不同 DNS 结果。

例如：

```
中国大陆 → IP A
欧洲      → IP B
美国      → IP C
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/dc1db3d2551eba3e7fa22ed48565cd9f_MD5.jpg]]

### 21.5 DNS如何配合CDN实现就近访问？

CDN 最重要的目标之一：

```
把用户调度到合适的边缘节点
```

DNS 正是常见的流量调度入口。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/297031f094345e536ba478970d8a8dcf_MD5.jpg]]

### 21.6 DNS负载均衡有什么局限？

DNS 缓存决定了它并不是一个精确到：

```
每一个HTTP请求
```

的实时负载调度器。

例如：

```
TTL = 300
```

recursive resolver 缓存后，未来一段时间可能反复复用结果。

所以 DNS 调度：

```
粒度较粗
受到缓存影响
客户端行为难完全控制
故障切换不能保证瞬时完成
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/43feaafbee85aa097c640435e3ca8851_MD5.jpg]]

## 二十二、图解：DNS和CDN之间是什么关系？

### 22.1 普通DNS解析过程

没有 CDN：

```
www.example.com
↓
A/AAAA
↓
源站IP
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/22667d262cbb42a02fff9b451d7eea67_MD5.jpg]]

### 22.2 接入CDN以后

一种常见方式：

```
www.example.com
↓
CNAME
↓
customer.cdn.example
↓
CDN权威调度
↓
某个Edge IP
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/b725bc4966b39bc800c8efa117fd4ed3_MD5.jpg]]

### 22.3 CNAME如何把请求交给CDN？

你的 DNS：

```
www.example.com
CNAME
customer.cdn.example
```

意味着：

> 真正地址请继续跟随 CDN 管理的名字查找。

于是后面的地址选择权交给 CDN。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a17f9c28d3ab47685d4a7b129d77beff_MD5.jpg]]

### 22.4 CDN如何判断用户位置？

一种常见依据是：

```
递归DNS请求来源地址
```

如果使用：

```
EDNS Client Subnet
```

递归服务器还可能把一定长度的客户端网络前缀交给支持 ECS 的权威系统。RFC 7871 就定义了这个 EDNS 选项。

ECS 可以改善某些地域调度，但也引入：

```
隐私
缓存碎片化
```

等权衡。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/2ba60cfe7fd07b724b9ac9f7e3c34012_MD5.jpg]]

### 22.5 CDN如何选择最近的边缘节点？

“最近”往往并不单纯等于：

```
地图直线距离最近
```

而可能综合：

```
网络拓扑
运营商
地域
节点健康
实时负载
成本
业务策略
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3e880a29f2d7c74530888f3dd88fd0da_MD5.jpg]]

### 22.6 为什么北京和广州可能得到不同IP？

因为 CDN 的 DNS 调度系统可能认为：

```
北京 → Edge A
广州 → Edge B
```

更加合适。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3af938ac5560e058f86bc8326c9215ac_MD5.jpg]]

### 22.7 DNS在CDN调度中的作用

一句话：

> **DNS 经常承担 CDN 的第一层用户流量入口选择工作。**

后面真正的数据传输才发生在：

```
用户
↔
CDN Edge
```

之间。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/cf40ff8e6dbbfaf70543e01c71e56984_MD5.jpg]]

## 二十三、DNS劫持和DNS污染是什么？

### 23.1 什么是DNS劫持？

“DNS 劫持”并不是一个只有唯一严格协议含义的 RFC 字段名称。

实际安全语境里通常指：

> DNS 解析路径或者结果被非预期实体控制、修改或者重定向。

例如：

```
正常应返回 A
↓
实际被改成攻击者IP B
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d3a4cbbf0d2e747a396cdf74e8abe9f2_MD5.jpg]]

### 23.2 什么是DNS污染？

“DNS 污染”通常用于描述：

```
伪造
干扰
注入错误DNS响应
```

使查询者获得错误数据。

这两个词在实际讨论里有重叠，不能认为全世界所有资料都有完全一致的严格边界。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/59efd88308df872557a15ae6882df7ef_MD5.jpg]]

### 23.3 两者有什么区别？

可以用一个工程化理解：

```
DNS劫持：
强调解析控制权被改变/重定向

DNS污染：
强调解析数据受到伪造、注入或干扰
```

但是具体分析问题的时候最好进一步说清楚：

```
Hosts被改？
路由器DNS被改？
运营商resolver返回不同答案？
收到伪造UDP响应？
缓存被投毒？
```

而不要只停留在一个模糊名词。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/08da0021e5aab6ea7b54533c529ba289_MD5.jpg]]

### 23.4 DNS缓存投毒是什么？

Cache Poisoning：

攻击者想办法让 recursive resolver 把：

```
错误DNS数据
```

保存进缓存。

那么后续大量用户都会收到错误答案。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a6023345303251053a4d28e7d7046bcb_MD5.jpg]]

### 23.5 攻击者为什么可以伪造DNS响应？

传统未验证 DNS 缺少数据来源的密码学真实性证明。

尤其经典 UDP DNS：

```
无连接
```

攻击者如果能够成功伪造一份被客户端/解析器接受的响应，就可能误导查询结果。

现代 resolver 会通过：

```
随机事务ID
源端口随机化
更严格的响应匹配
DNSSEC验证
```

等方式提升安全性。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/40289bbe211c5feb48711708b290e57b_MD5.jpg]]

### 23.6 DNS劫持可能造成什么后果？

可能包括：

```
钓鱼网站
广告重定向
恶意软件下载
账号密码窃取
服务不可访问
错误CDN节点
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f4add5b03f97148576b0b8065422f70e_MD5.jpg]]

### 23.7 如何判断自己是否遇到了DNS问题？

可以对比：

```
dig example.com
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com
```

再直接查询权威 DNS。

如果：

```
不同resolver结果明显异常
```

再进一步检查：

```
TTL
CNAME
权威记录
Hosts
DoH
本地缓存
网络设备
```

不能仅凭：

```
不同IP
```

就认定劫持，因为 CDN/GeoDNS 本来就可能合法返回不同地址。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/49ce34db8cb079a1d98112b4220aebf9_MD5.jpg]]

## 二十四、DNSSEC是如何保护DNS安全的？

### 24.1 传统DNS存在什么安全问题？

传统 DNS 最大的问题之一：

> **接收到一条 DNS 数据，并不天然拥有密码学方法证明这条数据确实来自正确的 DNS 权威链，而且没有被篡改。**

DNSSEC 就是为这种真实性与完整性验证设计的。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/de4bcd14dfedb9d28075f628b524cad0_MD5.jpg]]

### 24.2 DNSSEC是什么？

DNSSEC：

**DNS Security Extensions。**

它通过：

```
公钥密码
数字签名
信任链
```

让 validating resolver 能够验证 DNS 数据。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7b0773885929ecc601d910d3785df651_MD5.jpg]]

### 24.3 DNSSEC为什么不是“DNS加密”？

特别重要：

```
DNSSEC
≠
DNS Encryption
```

DNSSEC 的重点：

```
这份数据是真的吗？
有没有被修改？
不存在的记录能否被可信证明？
```

而不是：

```
别人能不能看见我查询了什么？
```

DNSSEC 标准将其目标描述为 DNS 数据的 origin authentication 与 integrity protection。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ae04871cbd43faafd5f4b27049ed127c_MD5.jpg]]

### 24.4 数字签名如何验证DNS记录？

简单理解：

权威 zone 有：

```
私钥
```

对 RRset 生成签名：

```
RRSIG
```

验证方取得：

```
DNSKEY
```

中的公钥。

然后：

```
DNSKEY
↓
验证
↓
RRSIG
↓
证明RRset没有被偷偷修改
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9d8d71992be93bf37d28fa9100450fce_MD5.jpg]]

### 24.5 DNSKEY记录

保存 zone 用于 DNSSEC 验证的公钥信息。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0f85829f659d832e068dcadbdd67f444_MD5.jpg]]

### 24.6 DS记录

DS：

**Delegation Signer。**

关键作用：

```
父 Zone
↓
DS
↓
连接到子 Zone 的 DNSKEY
```

例如：

```
.
↓
com
↓
example.com
```

父级通过 DS 帮助构建对子级 key 的信任。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/34d4e561c0d009adb45401c651ce55bf_MD5.jpg]]

### 24.7 RRSIG记录

保存：

```
某个RRset的数字签名
```

DNSSEC 核心 RR 定义由 RFC 4034 等标准规定。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ab22f1147a6817ccf5dc891d69304f14_MD5.jpg]]

### 24.8 DNSSEC信任链

简化：

```
Root Trust Anchor
       ↓
    .com DS
       ↓
.com DNSKEY
       ↓
example.com DS
       ↓
example.com DNSKEY
       ↓
A / AAAA / MX ...
       +
     RRSIG
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/8c1e34197df058d742802b8b749f2b27_MD5.jpg]]

### 24.9 从根DNS到目标域名如何逐级验证？

Validating resolver 首先拥有可信：

```
Root Trust Anchor
```

然后逐级验证：

```
父级DS
↓
子级DNSKEY
↓
子级签名数据
```

一直走到目标 RRset。

IANA 维护并发布 DNS 根区域的 DNSSEC trust anchor 数据。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3141cd9081a0d209f44cf251067be217_MD5.jpg]]

### 24.10 DNSSEC能够防御什么？

核心是：

```
伪造DNS数据
缓存投毒的一类攻击
中间人偷偷修改DNS答案
```

只要验证链完整，攻击者没有对应私钥，就无法简单伪造一份能够通过验证的签名结果。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ecd8e8f0090e85407a3bc24edb8985f5_MD5.jpg]]

### 24.11 DNSSEC不能解决什么？

DNSSEC 不负责：

```
隐藏你查询什么域名
加密DNS报文
防御所有DDoS
保证网站服务器没有被入侵
保证域名本身不是恶意网站
```

所以：

> **DNSSEC解决“答案可信不可信”，不是解决“查询别人能不能看见”。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/1fe2d748c7d888718d58fa98f3e23c61_MD5.jpg]]

## 二十五、传统DNS为什么会泄露隐私？

### 25.1 普通DNS查询为什么通常是明文的？

经典：

```
UDP/53
TCP/53
```

并没有自动提供 TLS 加密。

所以传统网络路径上的观察者可能看到 DNS query name。

DNS Privacy RFC 对传统 DNS 查询暴露用户访问意图的问题进行了系统分析。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/91fa65c693681fa6b5f61c6ef9383aca_MD5.jpg]]

### 25.2 网络运营商能看到DNS查询吗？

如果使用普通未加密 DNS，并且网络路径经过运营商：

```
查询内容可能被路径上的设备观察
```

即使你改用：

```
8.8.8.8
```

如果仍然是普通：

```
UDP/53
```

也不代表 query payload 自动变成加密。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f5941d4b717a95b9b846f1cf529b07af_MD5.jpg]]

### 25.3 DNS over TLS（DoT）是什么？

DoT：

```
DNS
↓
TLS
↓
TCP
```

使用 TLS 保护客户端和 DNS resolver 之间的 DNS 通信。

RFC 7858 定义 DoT 用 TLS 提供 DNS privacy，并防止网络路径上的窃听和篡改。

专用 DoT 通常使用：

```
853
```

端口。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e467e31dbf25b209f61d68023616bc67_MD5.jpg]]

### 25.4 DNS over HTTPS（DoH）是什么？

DoH：

```
DNS Query
↓
HTTPS
```

RFC 8484 定义：

> 每个 DNS query-response pair 映射为 HTTP exchange。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5bd5a3dc6b948f6c9440135c8f69e4d4_MD5.jpg]]

### 25.5 DoH为什么使用HTTPS？

因为：

```
HTTP
+
TLS
```

提供了成熟的加密传输机制。

DoH 一般运行在 HTTPS 生态中，因此常见端口是：

```
443
```

Cloudflare 官方当前同样说明 DoH 将 DNS 流量封装到 HTTPS 并使用 443。

![](https://relay-1.bijitongbu.site/p/a3c6e9fc3c30e9c9191afe973e51bb48.png)

### 25.6 DoT和DoH有什么区别？

简单对比：

```
DoT：
DNS over TLS
专用加密DNS传输
通常端口853

DoH：
DNS over HTTPS
利用HTTP语义
通常混合在HTTPS/443流量中
```

二者都可以保护：

```
客户端 ↔ 加密DNS解析器
```

之间的传输。

但 resolver 本身仍然需要处理你的 query。

所以：

> **加密 DNS 不是“全世界再也没有任何人知道你查什么”，而是改变和保护某段 DNS 传输链路。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5d2035ec9fd63dcfd6707d29ab820d80_MD5.jpg]]

### 25.7 DNSSEC、DoH、DoT解决的是同一个问题吗？

不是。

这是高频面试题。

```
DNSSEC
↓
答案真实性、完整性

DoT
↓
DNS传输加密（TLS）

DoH
↓
DNS传输加密（HTTPS）
```

可以记：

```
DNSSEC：
“答案是真的假的？”

DoH / DoT：
“路上的人能不能直接看/改我的查询？”
```

它们可以组合使用，而不是互相替代。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/418bba5227ce3d05b66a1c40e0640602_MD5.jpg]]

## 二十六、浏览器访问一个网站时，到底进行了几次DNS查询？

![[Inbox/笔记同步助手/微信公众号/2026/08/images/953671d81eb756b47d8ebb6dcf91f439_MD5.jpg]]

### 26.1 第一次访问网站

没有任何缓存时，浏览器可能需要查询：

```
A
AAAA
```

现代客户端还可能涉及：

```
HTTPS/SVCB
```

等记录。

所以：

> **“访问一个网站一定只产生一次 DNS Query”是不准确的。**

### 26.2 浏览器缓存命中

如果浏览器已有有效结果：

```
网络DNS Query可能是0次
```

### 26.3 操作系统缓存命中

浏览器没有，但系统 resolver/cache 有：

```
仍然可能不产生外部查询
```

### 26.4 本地DNS缓存命中

客户端需要向 recursive resolver 发一条查询。

resolver 发现：

```
缓存中已经有
```

于是：

```
Root：0次
TLD：0次
Authoritative：0次
```

### 26.5 CNAME存在时会发生什么？

例如：

```
www.example.com
↓
CNAME
↓
customer.cdn.example
```

resolver 还需要获得：

```
customer.cdn.example 的地址
```

如果目标没有缓存，就可能产生进一步查询。

### 26.6 页面引用多个域名时会发生什么？

一个网页可能引用：

```
www.example.com
static.examplecdn.com
images.cdn.example
api.example.net
analytics.example.org
fonts.example
```

每一个名字都可能触发独立解析。

所以页面真正的 DNS 行为可能是：

```
HTML主域名
+
图片域名
+
JS域名
+
CSS域名
+
API域名
+
统计域名
```

### 26.7 一张图还原真实网页中的DNS查询过程

```
打开网页
   ↓
www.example.com
   │
   ├── HTML
   │
   ├── static.cdn.example
   │       ↓ DNS
   │
   ├── api.example.net
   │       ↓ DNS
   │
   └── img.examplecdn.com
           ↓ DNS
```

因此答案是：

> **没有固定次数。**

可能：

```
0次
1次
几次
几十次
```

取决于：

```
缓存
A/AAAA/HTTPS查询
CNAME链
页面资源域名数量
浏览器策略
DoH配置
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a7f0d7c8f3157b42b1425f2808048ec2_MD5.jpg]]

## 二十七、DNS解析失败时会发生什么？

### 27.1 DNS服务器没有响应

客户端可能：

```
等待timeout
重试
查询备用DNS
```

最终解析失败。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7773d39c2a300cb640345577480694ec_MD5.jpg]]

### 27.2 域名不存在

权威体系明确表示：

```
这个名字不存在
```

典型：

```
NXDOMAIN
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/7783cd4f175c5b36df5bca07da71e83d_MD5.jpg]]

### 27.3 DNS记录配置错误

例如：

```
A指错IP
CNAME循环
NS委派错误
Glue错误
DNSSEC配置错误
```

都可能产生异常。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/4f94ae64ed50f92aaebe047c965aaadf_MD5.jpg]]

### 27.4 DNS缓存没有更新

权威已经修改：

```
新IP
```

但是递归 resolver 仍持有：

```
旧TTL缓存
```

用户继续得到旧答案。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9dde608210b92f299c1938a156044bd3_MD5.jpg]]

### 27.5 CNAME配置错误

例如：

```
a.example → b.example
b.example → a.example
```

形成：

```
CNAME loop
```

解析无法正常结束。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/8a94143755365c933f94822286bb1588_MD5.jpg]]

### 27.6 DNS服务器故障

权威：

```
宕机
网络不可达
软件异常
```

也可能最终表现：

```
SERVFAIL
timeout
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/b21ffe736c760f033f637c51c7c3d4ce_MD5.jpg]]

### 27.7 网络无法访问DNS服务器

例如：

```
防火墙拦UDP/53
路由故障
VPN策略
DoT/DoH被阻断
```

同样表现成解析失败。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/671d52d6a99d9be50648b03ceb51bb87_MD5.jpg]]

### 27.8 常见DNS错误码是什么意思？

不要把：

```
DNS失败
```

全部理解成：

> “域名不存在。”

至少需要区分：

```
NXDOMAIN
SERVFAIL
REFUSED
timeout
NOERROR但无所需RR
```

## 二十八、DNS常见状态码详解

IANA 当前 DNS RCODE 注册表中的基础错误码包括 NOERROR、FORMERR、SERVFAIL、NXDOMAIN、NOTIMP 和 REFUSED。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ab90417239946ca3a08f878dbbc0f03e_MD5.jpg]]

### 28.1 NOERROR

```
RCODE = 0
```

表示：

**协议查询处理没有错误。**

但注意：

```
NOERROR
```

不等于：

```
Answer一定有你查询的记录
```

名字存在但没有某个 QTYPE：

```
NOERROR
+
空Answer
```

可以形成：

```
NODATA
```

语义。

### 28.2 FORMERR

```
RCODE = 1
```

DNS 报文格式错误。

### 28.3 SERVFAIL

```
RCODE = 2
```

服务器没能成功完成处理。

可能原因：

```
上游不可达
DNSSEC验证失败
权威服务器异常
递归过程中发生错误
```

但仅凭一个 SERVFAIL 不能直接武断确定是哪一种。

### 28.4 NXDOMAIN

```
RCODE = 3
```

表示：

```
Non-Existent Domain
```

目标名字不存在。

### 28.5 NOTIMP

```
RCODE = 4
```

服务器不支持请求的功能。

### 28.6 REFUSED

```
RCODE = 5
```

服务器根据策略拒绝查询。

例如：

```
不允许你使用递归
不允许某类请求
```

### 28.7 NXDOMAIN和SERVFAIL有什么区别？

这是非常重要的区别：

```
NXDOMAIN：
我成功查到了结论：
这个名字不存在

SERVFAIL：
我没办法可靠地完成这次查询
```

可以类比：

```
NXDOMAIN：
“查过档案了，没有这个人。”

SERVFAIL：
“档案系统坏了，我现在查不了。”
```

## 二十九、Linux下如何排查DNS问题？

### 29.1 `ping`能不能用于判断DNS问题？

可以作为线索。

例如：

```
ping 8.8.8.8
```

成功。

但：

```
ping example.com
```

提示：

```
Name or service not known
```

名字解析很值得怀疑。

但是：

> **ping 不是专门的 DNS 诊断工具。**

因为它还涉及：

```
ICMP
网络策略
系统resolver
IPv4/IPv6选择
```

而服务器可能直接禁止 ICMP。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/8476e53157c41f3b3430a57d337ebc6d_MD5.jpg]]

### 29.2 使用 `nslookup`

```
nslookup example.com
```

指定 DNS：

```
nslookup example.com 8.8.8.8
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f5988135472063ee215a4cdb1e660b1f_MD5.jpg]]

### 29.3 使用 `dig`

更加推荐网络/DNS 排障掌握：

```
dig example.com
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/4a3e4d745d9f5a3cb0140e29af1b7e66_MD5.jpg]]

### 29.4 `dig`结果怎么看？

典型：

```
; <<>> DiG <<>> example.com
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: ...
;; flags: qr rd ra
;; QUESTION SECTION:
;example.com.            IN      A

;; ANSWER SECTION:
example.com.      ...    IN      A       ...

;; Query time: ...
;; SERVER: ...
```

先看：

```
status
```

再看：

```
QUESTION
ANSWER
AUTHORITY
ADDITIONAL
```

最后看：

```
SERVER
Query time
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/c38a14a8dc80ce4143faf51da9a285ed_MD5.jpg]]

### 29.5 查询指定DNS服务器

```
dig @8.8.8.8 example.com
```

Cloudflare：

```
dig @1.1.1.1 example.com
```

AliDNS：

```
dig @223.5.5.5 example.com
```

阿里官方排障资料同样使用指定公共 DNS 的 `nslookup` 查询来比较 LocalDNS 与权威解析状态。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0cee81f4a654225e0f57ab2302f5fb28_MD5.jpg]]

### 29.6 A记录

```
dig example.com A
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/95b2099c329adba566552c6ec3f5e244_MD5.jpg]]

### 29.7 CNAME

```
dig www.example.com CNAME
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/6ac2d31110435ec8e146c93c21a40329_MD5.jpg]]

### 29.8 MX

```
dig example.com MX
```

![](https://relay-1.bijitongbu.site/p/d1efb52843c028f527acb1e471fbf49d.png)

### 29.9 NS

```
dig example.com NS
```

![](https://relay-1.bijitongbu.site/p/951319a82e80503cc42f4507b9f14546.png)

### 29.10 `dig +trace`

```
dig +trace example.com
```

它会自己模拟迭代解析思路：

```
Root
↓
TLD
↓
Authoritative
```

特别适合理解：

```
委派到底在哪里断了
```

但是要注意：

> `dig +trace` 是 `dig` 自己执行逐级查询，不等于完全复现你平时使用的 recursive resolver 的缓存和策略。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d951531fa4d7e1def0b2c5a4297cef66_MD5.jpg]]

### 29.11 查看Linux系统当前DNS配置

传统：

```
cat /etc/resolv.conf
```

使用 systemd-resolved 的系统还应该看：

```
resolvectl status
```

查询系统解析路径：

```
resolvectl query example.com
```

另外：

```
getent ahosts example.com
```

常常比：

```
dig
```

更适合验证：

> **应用通过系统 libc/NSS 名字解析路径到底会得到什么。**

![](https://relay-1.bijitongbu.site/p/d4976e2acc109487f27ac6a784eaad22.png)

### 29.12 `/etc/resolv.conf`

典型：

```
nameserver 192.0.2.53
nameserver 1.1.1.1
search example.local
```

不过现代 Linux：

```
/etc/resolv.conf
```

有可能是：

```
systemd-resolved
NetworkManager
DHCP客户端
```

生成或者指向 stub resolver 的符号链接。

不要看到里面：

```
127.0.0.53
```

就立即判断“DNS 配错了”。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5d55480c922c93ed7a1e1a0c154a21a4_MD5.jpg]]

### 29.13 查看 `/etc/hosts`

```
cat /etc/hosts
```

例如：

```
127.0.0.1 localhost
192.0.2.20 test.example.com
```

另外 Linux 上还值得检查：

```
grep '^hosts:' /etc/nsswitch.conf
```

因为：

```
hosts:
```

这一行会影响系统名字服务来源的组合与顺序。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/08e8daa3630e1efa4277d7e6fc25a654_MD5.jpg]]

## 三十、图解：使用dig完整追踪一次DNS解析

执行：

```
dig +trace example.com
```

### 30.1 从根DNS服务器开始

首先会看到根：

```
.    NS    a.root-servers.net.
.    NS    b.root-servers.net.
...
```

本地 iterative query 能够启动，是因为 resolver/tool 拥有 root hints 一类引导信息；IANA 就公开提供根服务器地址和 root hints 数据。

### 30.2 找到顶级域DNS服务器

接着：

```
com. NS ...
```

意思：

> 查 `example.com`，下一层先去 `.com`。

### 30.3 找到权威DNS服务器

`.com` 返回：

```
example.com. NS ...
```

于是：

```
Root
↓
com
↓
example.com NS
```

### 30.4 查询目标域名记录

最后向权威：

```
example.com A?
```

### 30.5 得到最终IP地址

权威服务器给出：

```
ANSWER
```

完整链条：

```
.
↓
com.
↓
example.com.
↓
A / AAAA
```

### 30.6 对照理论理解真实查询结果

以后看到：

```
dig +trace
```

不要只是觉得：

> “输出好多东西。”

而应该立刻分类：

```
这一段是Root referral
这一段是TLD delegation
这一段是权威NS
这一段才是最终Answer
```

当你能这样读以后，DNS 层级结构就不再停留在 PPT 图里面，而是真正能够在网络上被验证。

## 三十一、Windows下如何排查DNS问题？

### 31.1 使用 `nslookup`

```
nslookup example.com
```

指定服务器：

```
nslookup example.com 8.8.8.8
```

Microsoft 将 `nslookup` 定义为用于诊断 DNS 基础设施信息的命令行工具。

### 31.2 查看DNS缓存

```
ipconfig /displaydns
```

可以查看 Windows DNS Client resolver cache。

### 31.3 清理DNS缓存

```
ipconfig /flushdns
```

Microsoft 官方 `ipconfig` 文档说明 `/flushdns` 会刷新并重置 DNS Client resolver cache。

PowerShell 也可以：

```
Clear-DnsClientCache
```

官方说明它与 `ipconfig /flushdns` 对 DNS client cache 的清理作用等价。

### 31.4 修改DNS服务器

可以在 Windows：

```
网络适配器
↓
IPv4 / IPv6
↓
DNS Server
```

修改。

也可以通过 PowerShell 网络命令进行企业自动化配置。

修改后应重新验证：

```
nslookup example.com
```

并确认显示的 resolver 是否符合预期。

### 31.5 如何判断是DNS问题还是网络问题？

顺序可以这样：

```
1. 能否访问默认网关？
2. 能否访问公网IP？
3. 能否查询DNS？
4. 指定其他DNS能否查询？
5. 域名解析出来的IP能否建立连接？
```

### 31.6 为什么能Ping通IP却打不开域名？

典型：

```
IP网络本身是通的
↓
名字解析链路出问题
```

可能：

```
DNS服务器不可达
DNS配置错误
缓存错误
Hosts错误
DoH/安全软件问题
域名本身解析异常
```

Microsoft DNS client troubleshooting 文档也会通过 `nslookup`、DNS cache 等步骤区分名字解析故障。

---

## 三十二、DNS常见故障场景分析

### 32.1 能Ping通 `8.8.8.8`，但是打不开网站？

说明至少有可能：

```
基础公网IP连通存在
```

但是：

```
DNS
代理
TLS
浏览器
防火墙
```

仍可能有问题。

先：

```
nslookup example.com
```

或者：

```
dig example.com
```

确认名字解析。

### 32.2 修改域名解析后为什么迟迟没有生效？

依次检查：

```
权威 DNS 是否已经改对？
↓
NS 委派是否正确？
↓
旧记录 TTL 是多少？
↓
递归DNS是否还有缓存？
↓
浏览器/OS是否还有缓存？
```

特别容易犯的错误：

> 修改以后只看新 TTL。

实际上旧缓存何时过期，取决于：

```
修改之前被缓存的旧记录TTL
```

### 32.3 同一个域名为什么不同电脑解析出的IP不同？

完全可能正常。

原因：

```
不同recursive resolver
CDN
GeoDNS
ECS
缓存时间不同
A/AAAA差异
Split DNS
企业内网DNS
```

所以：

> **不同IP ≠ DNS一定出错。**

### 32.4 为什么换一个DNS服务器网站就能打开？

可能：

```
原resolver缓存异常
原resolver无法访问权威服务器
过滤策略不同
DNSSEC验证结果不同
地域网络不同
解析调度结果不同
```

不能简单归结为：

> “某公共DNS永远比运营商DNS好。”

### 32.5 为什么浏览器能打开网站，`ping`却显示另一个IP？

原因可能非常多：

```
浏览器自己的DNS缓存
浏览器DoH
ping使用系统resolver
A/AAAA选择不同
CDN多IP
缓存时刻不同
浏览器连接复用
```

所以浏览器和 `ping`：

> **不一定走完全相同的名字解析路径。**

### 32.6 为什么删除本机DNS缓存后问题仍然存在？

因为缓存不只在本机：

```
浏览器缓存
↓
系统缓存
↓
递归DNS缓存
↓
其他转发DNS
```

你执行：

```
ipconfig /flushdns
```

或者重启本机，只能影响对应本地缓存。

并不会通知：

```
运营商DNS
8.8.8.8
1.1.1.1
```

把全球缓存全部删除。

### 32.7 为什么域名能解析，但是网站还是打不开？

因为：

> **DNS成功只证明“名字解析阶段成功”。**

后面还有：

```
路由
TCP / QUIC
防火墙
TLS
证书
HTTP
Web服务器
CDN
应用程序
```

例如：

```
dig example.com
```

正常。

但是：

```
curl -v https://example.com
```

在 TLS timeout。

这就已经不属于单纯：

```
DNS找不到IP
```

的问题了。

这也是排障中特别重要的一条：

> **DNS是访问网站的一个环节，不是整个互联网。**

## 三十三、DNS常见面试题

### 33.1 DNS使用TCP还是UDP？

**都使用。**

经典 DNS 服务：

```
UDP/53
TCP/53
```

### 33.2 为什么DNS通常使用UDP？

普通 query/response 较小。

UDP：

```
无需建立TCP连接
协议开销较低
```

适合大量短查询。

### 33.3 DNS什么时候使用TCP？

常见：

```
UDP响应截断
直接选择TCP
较大消息
AXFR等区域传送
```

现代 DNS 实现必须正确支持 TCP。

### 33.4 DNS完整解析过程是什么？

标准回答：

```
客户端
↓
递归resolver
↓
Root
↓
TLD
↓
Authoritative
↓
最终记录
↓
resolver缓存
↓
客户端
```

如果任意层缓存命中：

```
可以提前结束
```

### 33.5 递归查询和迭代查询有什么区别？

```
递归：
你帮我查到最终结果

迭代：
不知道最终答案时告诉我下一步问谁
```

### 33.6 A记录和CNAME有什么区别？

```
A：
名字 → IPv4

CNAME：
名字 → 另外一个名
```

### 33.7 DNS为什么需要缓存？

为了：

```
减少延迟
减少重复查询
减轻Root/TLD/权威压力
提升稳定性
```

### 33.8 TTL是什么意思？

DNS RR：

```
Time To Live
```

决定缓存数据可继续被认为有效的时间范围。

### 33.9 DNS劫持和DNS污染有什么区别？

工程上：

```
劫持：
强调解析控制/结果被重定向

污染：
强调错误或伪造响应被注入
```

但二者不是完全互斥的严格协议字段。

### 33.10 DNSSEC、DoH和DoT分别解决什么问题？

```
DNSSEC
→ 数据真实性和完整性

DoT
→ DNS over TLS
→ 加密传输

DoH
→ DNS over HTTPS
→ 加密传输
```

### 33.11 浏览器输入URL后，DNS在哪一步发生？

典型逻辑：

```
URL解析
↓
确认目标hostname
↓
DNS/name resolution
↓
获得IP
↓
建立目标连接
↓
HTTP
```

缓存命中时外部 DNS 网络查询可以省略。

### 33.12 为什么修改DNS记录后不能立即生效？

因为：

```
旧记录已经被各级缓存
```

，需要等待相应缓存 TTL 到期或者通过特定 resolver 支持的缓存刷新机制处理。

## 三十四、图解：DNS协议完整知识体系总结

学到这里，再来看：

```
DNS
```

已经不能简单理解成：

> “一个域名转 IP 的工具。”

它实际上是一整套全球分布式命名基础设施。

### 34.1 DNS解决了什么问题？

最基础：

```
人类容易记忆的名字
↓
DNS
↓
网络资源信息
```

其中地址解析只是最常见的一种。

### 34.2 DNS域名层级结构

```
.
│
├── com
│    │
│    └── example
│          │
│          ├── www
│          ├── mail
│          └── api
│
├── org
└── cn
```

核心：

```
树形命名
+
逐级管理
```

### 34.3 DNS服务器体系

```
客户端
  ↓
Recursive Resolver
  ↓
Root
  ↓
TLD
  ↓
Authoritative
```

### 34.4 DNS递归与迭代查询

```
客户端 → Resolver
通常希望：
“给我最终答案”

Resolver → DNS hierarchy
通常：
“你不知道就告诉我下一步找谁”
```

### 34.5 DNS缓存机制

```
Browser
↓
OS / Local Resolver
↓
Recursive Resolver
```

缓存通过：

```
TTL
```

显著降低查询成本。

### 34.6 DNS资源记录

```
A       IPv4
AAAA    IPv6
CNAME   Alias
MX      Mail
NS      Name Server
TXT     Text
PTR     Reverse
SOA     Zone metadata
SRV     Service
CAA     CA policy
```

### 34.7 DNS报文格式

```
┌─────────────────┐
│ Header          │
├─────────────────┤
│ Question        │
├─────────────────┤
│ Answer          │
├─────────────────┤
│ Authority       │
├─────────────────┤
│ Additional      │
└─────────────────┘
```

基础 wire format 来源于 DNS 核心规范 RFC 1035。

### 34.8 DNS与UDP/TCP

```
普通短查询
→ UDP非常常见

TCP
→ 同样是DNS标准传输
→ 必须正确支持

UDP不够
→ EDNS扩展能力
→ 或使用TCP等方式
```

### 34.9 DNS与CDN

```
用户
↓
DNS
↓
CDN调度
↓
选择合适Edge
↓
用户访问Edge
```

DNS 从：

```
“名字查地址”
```

进一步承担：

```
流量入口调度
```

的重要作用。

### 34.10 DNS安全机制

可以分成三个问题：

**第一个：答案真实吗？**

```
DNSSEC
```

**第二个：查询传输能否加密？**

```
DoT
DoH
DoQ
```

**第三个：解析器本身是否值得信任？**

这个问题不是：

```
开了DoH
```

就自动消失。

因为你只是把：

```
“谁能看到明文DNS查询”
```

从网络路径中的多个实体，更多地集中到了你选择的加密 DNS provider。

### 34.11 从输入域名到打开网页的完整流程

最后，我们把整篇文章重新压缩成一条链。

用户输入：

```
https://www.example.com
```

第一步：

```
浏览器解析URL
```

得到：

```
scheme = https
host   = www.example.com
```

接下来需要名字解析：

```
本地有没有可用答案？
        │
        ├── 有
        │    ↓
        │ 直接使用
        │
        └── 没有
             ↓
         Recursive DNS
```

递归 DNS 如果自己也没缓存：

```
Recursive Resolver
        │
        ▼
      Root
        │
        │ .com 在哪里
        ▼
     .com TLD
        │
        │ example.com 权威在哪里
        ▼
 Authoritative DNS
        │
        │ A / AAAA / CNAME ...
        ▼
 Recursive Resolver
        │
        │ 缓存TTL
        ▼
      Client
```

客户端最终得到：

```
目标IP
```

然后：

```
DNS解析完成
        ↓
根据IP路由
        ↓
建立TCP / QUIC
        ↓
HTTPS情况下建立安全会话
        ↓
发送HTTP请求
        ↓
服务器 / CDN响应
        ↓
浏览器获得HTML
        ↓
解析CSS / JS / 图片
        ↓
发现新的hostname
        ↓
可能再次进行DNS解析
```

如果配置了 CDN：

```
www.example.com
        ↓
      CNAME
        ↓
    CDN域名
        ↓
 CDN DNS调度
        ↓
合适的Edge IP
```

如果使用 DNSSEC：

```
DNS数据
+
RRSIG
+
DNSKEY
+
DS
+
Trust Chain
        ↓
验证答案真实性与完整性
```

如果使用 DoT / DoH：

```
客户端
        │
        │ 加密DNS传输
        ▼
Encrypted Resolver
```

所以完整 DNS 知识体系最终可以浓缩成：

```
DNS
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
      名称空间            解析体系             数据
        │                  │                  │
        ▼                  ▼                  ▼
   Root/TLD/Domain     Recursive           Resource Record
        │             Iterative               │
        │                  │            A / AAAA / CNAME
        │                  │            MX / NS / TXT
        │                  │            PTR / SOA / SRV
        │                  │            CAA ...
        │                  │
        └──────────┬───────┘
                   │
                 Cache
                   │
                  TTL
                   │
      ┌────────────┼────────────┐
      │            │            │
     UDP          TCP          EDNS
      │            │            │
      └────────────┼────────────┘
                   │
            DNS Security / Privacy
                   │
        ┌──────────┼──────────┐
        │          │          │
      DNSSEC      DoT        DoH
        │          │          │
   真实性/完整性    └────┬─────┘
                         │
                       加密
```

学习 DNS 以后，真正应该建立的并不是一句：

> **“DNS 就是域名解析。”**

而是一套完整的问题分析方式。

以后当你遇到：

```
网站打不开
```

不要第一反应：

> “是不是 DNS 挂了？”

而应该一步一步问：

```
域名能不能解析？
↓
系统到底使用哪个resolver？
↓
查询得到什么RCODE？
↓
A / AAAA / CNAME是否正确？
↓
权威DNS的答案是什么？
↓
递归DNS是否仍然缓存旧数据？
↓
NS委派有没有问题？
↓
DNSSEC验证有没有失败？
↓
解析正常以后，目标IP能不能连接？
↓
TCP/QUIC是否成功？
↓
TLS是否成功？
↓
HTTP是否正常？
```

比如：

```
dig example.com
```

如果已经能够正确拿到：

```
NOERROR
+
正确的 A/AAAA
```

那么 DNS 至少已经完成了自己最核心的那一部分工作。

如果后面的：

```
curl -v https://example.com
```

仍然失败，就应该继续调查：

```
网络
路由
防火墙
TLS
证书
CDN
Web服务器
```

而不是不断：

```
flush DNS
flush DNS
flush DNS
```

这就是理解 DNS 协议最大的价值。

真正掌握 DNS，并不是记住：

```
DNS端口53
DNS使用UDP
A记录是IP
CNAME是别名
```

几个孤立知识点。

而是能够把：

```
名字
↓
树形命名空间
↓
区域委派
↓
递归/迭代解析
↓
缓存
↓
资源记录
↓
DNS报文
↓
UDP/TCP/EDNS
↓
CDN调度
↓
DNSSEC
↓
DoH/DoT
↓
实际网络排障
```

这一整条链路真正连接起来。

当有一天你看到：

```
dig +trace www.example.com
```

能够清楚地知道：

```
为什么第一段是Root
为什么下一段出现TLD
为什么NS意味着委派
为什么Additional里面可能有Glue
为什么最后才出现Answer
为什么TTL决定缓存
为什么NOERROR不一定有A记录
为什么SERVFAIL和NXDOMAIN完全不是一回事
```

这个时候，DNS 对你来说就不再是一个“浏览器背后自动发生的神秘步骤”。

而会变成一个：

> **能够从协议、报文、服务器体系、缓存机制一直分析到真实故障现场的完整网络基础协议。**

往期推荐

[图解TCP/IP协议，看图秒懂](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247483715&idx=1&sn=15b68b15cdfb31d6fe87da06819e2bf3&scene=21#wechat_redirect)

[图解C/C++ 多线程，看图秒懂](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247483715&idx=1&sn=15b68b15cdfb31d6fe87da06819e2bf3&scene=21#wechat_redirect)

[爆肝整理！嵌入式开发必知10种调试手段](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247483715&idx=1&sn=15b68b15cdfb31d6fe87da06819e2bf3&scene=21#wechat_redirect)

[【图解】SSH（安全外壳协议）工作原理](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247483715&idx=1&sn=15b68b15cdfb31d6fe87da06819e2bf3&scene=21#wechat_redirect)

👉 【大厂技术栈路线】[Linux C/C++ 后端开发系统学习路线](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247483715&idx=1&sn=15b68b15cdfb31d6fe87da06819e2bf3&scene=21#wechat_redirect)

👉 【音视频】[音视频流媒体高级开发核心学习路径](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247483783&idx=1&sn=dde455678a3dd744afef1b56b2e39249&scene=21#wechat_redirect)

👉 【Qt进阶】[C++ Qt 桌面&嵌入式式开发一条龙学习攻略](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247483731&idx=1&sn=bb891169764d6d45d70db5e19bb4b0a3&scene=21#wechat_redirect)

👉 【内核底层】[Linux 内核硬核修炼指南](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247484069&idx=1&sn=4c451a9c7b4c0fd83ee0ce651296c568&scene=21#wechat_redirect)

👉 【面试冲刺】[C/C++高频八股面试题1000 题（三）](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247483952&idx=1&sn=27ba763fff161cbcdd4c0f3035a67237&scene=21#wechat_redirect)

👉 【项目实战】[手撕线程池：C++序员的能力试金石](https://mp.weixin.qq.com/s?__biz=MzkzNDk2NTUwOQ==&mid=2247484145&idx=1&sn=379c99cd2faa6f338bf96e7a34f61266&scene=21&poc_token=HOr9emmjF6lYIWU_pEPrGAegzErpRw65H2J7qlbU#wechat_redirect)

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/1310a4dd_1787416970795?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzkzNDk2NTUwOQ%3D%3D%26mid%3D2247491970%26idx%3D1%26sn%3Dbcfed1cde0897e50decab84c79bc8d12%26chksm%3Dc342f4940609086b24b4babb0e304b6052156e849cf9bbf04812984c0d044bfab917b8e91f67%26mpshare%3D1%26scene%3D1%26srcid%3D0823UsA3AvEYzPDFepdnNCrC%26sharer_shareinfo%3D50dfe68aa4a44cb6d9ce74003b09287d%26sharer_shareinfo_first%3D50dfe68aa4a44cb6d9ce74003b09287d%23rd&s=obsidian)