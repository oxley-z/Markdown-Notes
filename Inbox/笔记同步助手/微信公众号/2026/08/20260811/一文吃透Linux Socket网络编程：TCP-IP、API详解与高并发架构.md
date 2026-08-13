---
author: Debug 蟹老板
source: 微信公众号
url: https://mp.weixin.qq.com/s/WCEgAKosxUmUEdpQuc5RcQ
saved: 2026-08-11 22:04:40
tags:
  - 笔记同步助手
id: f3ca4b59-753c-4cac-b25a-558db1c2fce1
---

公众号名称：Linux教程

作者名称：Debug 蟹老板

发布时间：2026-08-11 20:27

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d5673b61529048fc2392d6b1f5e4601a_MD5.gif]]

大家好，我是蟹老板～

在 Linux 网络编程之中，Socket 乃是一个极为重要的概念。

我们平时访问网页、连接数据库、使用 SSH 登录服务器、调用 HTTP 接口、运行 Redis、Nginx、MQTT、WebSocket 以及各种 RPC 框架的时候，底层几乎都离不开 Socket。

可是对于刚刚开始学习网络编程的人来说，Socket 往往会给人一种比较割裂的感觉。

服务端为什么需要依次调用：

```
socket();
bind();
listen();
accept();
recv();
send();
```

客户端为什么通常只有：

```
socket();
connect();
send();
recv();
```

`accept()` 为什么又返回一个新的文件描述符？

一次 `send()` 为什么不能保证对端一次 `recv()` 就能够完整读取？

程序明明只是调用了一次 `send()`，数据为什么最终能够从网卡发送到另外一台计算机？

服务器连接数量越来越多以后，为什么又会出现 `select`、`poll` 和 `epoll`？

这些问题如果只停留在 API 记忆层面，是比较难真正串联起来的。

**因此，理解 Linux Socket 最重要的并不是记住几十个函数，而是建立一条完整的数据流：**

```
应用程序
    ↓
Socket API
    ↓
文件描述符
    ↓
内核 Socket
    ↓
TCP/UDP
    ↓
IP
    ↓
网卡驱动
    ↓
物理网络
    ↓
对端网卡
    ↓
对端协议栈
    ↓
对端 Socket
    ↓
对端应用程序
```

**只要把这一条链路真正理解清楚，Socket 编程里面很多看起来零散的问题，其实都能够顺理成章地推导出来。**

## 一、网络通信，在 Linux 中到底是怎么发生的？

### 1.1 两台计算机为什么能够互相“说话”

假设现在存在两台计算机：

```
计算机 A：192.168.1.10
计算机 B：192.168.1.20
```

A 想给 B 发送一句：

```
Hello
```

从应用程序角度来看，可能只是：

```
send(fd, "Hello", 5, 0);
```

但是计算机并不真正理解“Hello”这个业务含义。

字符串在内存里面最终是：

```
'H' = 0x48
'e' = 0x65
'l' = 0x6C
'l' = 0x6C
'o' = 0x6F
```

这些数据需要经过网络协议栈不断封装。

例如 TCP 通信可以高度简化成为：

```
应用数据
   ↓
TCP Segment
   ↓
IP Packet
   ↓
Ethernet Frame
   ↓
网卡发送
```

到达对端以后则反过来：

```
Ethernet Frame
   ↓
IP Packet
   ↓
TCP Segment
   ↓
Socket接收队列
   ↓
应用程序
```

因此网络通信本质上就是：

> **把一台计算机进程内存中的数据，经过协议以及物理网络，可靠或者尽力地转移到另外一台计算机的进程中。**

![](https://relay-1.bijitongbu.site/p/b92e39884a7300cd2a7de1a4b8610021.png)

### 1.2 从应用程序到网卡：数据经历了什么

例如：

```
send(sockfd, buf, len, 0);
```

一个非常粗略的发送过程可以表示成为：

```
用户空间 buf
      ↓
send()
      ↓
系统调用进入内核
      ↓
Socket发送缓冲区
      ↓
TCP协议处理
      ↓
IP协议处理
      ↓
邻居/路由处理
      ↓
网络设备发送队列
      ↓
网卡驱动
      ↓
DMA
      ↓
网卡
      ↓
网络
```

需要特别注意：

```
send();
```

成功返回，并不等价于：

> 数据已经到达对端应用程序。

很多情况下，它只是表示：

> **内核已经接受了这些数据，并将其放入发送路径中。**

**真正什么时候进入网卡、什么时候抵达对端、什么时候被对端应用程序读取，是后续发生的事情。**

![](https://relay-1.bijitongbu.site/p/5aabcf20b9396df0b9d2ffd1253a0935.png)

### 1.3 Socket 是什么：为什么它是网络编程的入口

Socket 可以理解成为：

> **Linux 内核提供给应用程序使用网络协议栈的一种通信端点。**

应用程序并不需要自己手动构造：

```
TCP Header
IP Header
Ethernet Header
```

也不需要自己直接控制网卡发送每一个数据包。

应用只需要：

```
socket();
connect();
send();
recv();
```

**内核负责把这些抽象操作转换成为真正的网络协议操作。**

因此 Socket 如同应用程序和内核网络协议栈之间的一扇门。

![](https://relay-1.bijitongbu.site/p/b19faa5f122e351e9c58d1b9ceddabe7.png)

### 1.4 本文主线：从一次 TCP 通信推导 Linux 网络编程

全文会始终围绕下面这一条主线展开：

```
服务端创建Socket
      ↓
bind
      ↓
listen
      ↓
客户端connect
      ↓
TCP三次握手
      ↓
服务端accept
      ↓
客户端send
      ↓
内核TCP/IP
      ↓
网络
      ↓
服务端recv
      ↓
服务端send
      ↓
客户端recv
      ↓
close
```

然后在这个基础之上继续解决真实工程问题：

```
一条连接
   ↓
多条连接
   ↓
阻塞问题
   ↓
非阻塞
   ↓
IO多路复用
   ↓
epoll
   ↓
消息协议
   ↓
异常处理
   ↓
高并发服务器
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/458fd62c36a8557dd890cab927414f9b_MD5.jpg]]

## 二、Socket 基础：网络通信如何被抽象成“文件”

### 2.1 Linux“一切皆文件”与 Socket

Linux 中经常说：

> **一切皆文件。**

这个说法并不是说所有对象真的都存放在磁盘文件里面，而是说很多不同类型的资源，都尽量使用统一的文件操作模型进行管理。

例如：

```
普通文件
字符设备
管道
Socket
eventfd
timerfd
```

都可以通过文件描述符访问。

因此：

```
int fd = socket(AF_INET, SOCK_STREAM, 0);
```

返回的也是一个整数文件描述符。

后续：

```
read(fd, ...);
write(fd, ...);
close(fd);
```

和普通文件的操作形式非常相似。

但是 Socket 文件描述符背后对应的并不是磁盘 inode 数据，而是网络协议相关的内核对象。

![](https://relay-1.bijitongbu.site/p/47397d1d280d6cafb1b833efc81d1c22.png)

### 2.2 Socket、文件描述符与内核对象

应用程序执行：

```
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
```

以后，会得到：

```
sockfd = 3
```

**这个 3 并不是 Socket 本身。**

**它只是当前进程文件描述符表中的一个索引。**

关系可以高度简化成为：

```
用户空间
sockfd = 3
    ↓
进程文件描述符表
    ↓
struct file
    ↓
struct socket
    ↓
struct sock
    ↓
TCP/UDP协议控制结构
```

因此：

```
close(sockfd);
```

**本质上是在释放当前进程对这个 Socket 文件对象的引用。**

对于 TCP 来说，协议层还可能继续在内核里面维护 FIN、重传以及 TIME\_WAIT 等状态。

![](https://relay-1.bijitongbu.site/p/d7700d9198ff4c9cb90133e5a9ffd324.png)

### 2.3 IP 地址、端口与 Socket 地址

假设服务器：

```
IP   = 192.168.1.20
Port = 8080
```

IP 地址解决的是：

> **网络中要找到哪一台主机。**

端口解决的是：

> **到达主机以后，要交给哪个网络程序。**

例如同一台机器上可能同时运行：

```
22    SSH
80    HTTP
443   HTTPS
3306  MySQL
6379  Redis
```

因此：

```
IP + Port
```

**共同定位某一个网络服务端点。**

**一条 TCP 连接通常由四元组区分：**

```
源IP
源端口
目标IP
目标端口
```

例如：

```
192.168.1.10:52314
        ↓
192.168.1.20:8080
```

![](https://relay-1.bijitongbu.site/p/95d939d76900cd928d0df3baafa0c8d2.png)

### 2.4 IPv4、IPv6 与 `sockaddr` 地址结构

IPv4 常使用：

```
struct sockaddr_in {
    sa_family_t    sin_family;
    in_port_t      sin_port;
    struct in_addr sin_addr;
};
```

例如：

```
struct sockaddr_in addr;

memset(&addr, 0, sizeof(addr));

addr.sin_family = AF_INET;
addr.sin_port = htons(8080);

inet_pton(AF_INET,
          "192.168.1.20",
          &addr.sin_addr);
```

IPv6 对应：

```
struct sockaddr_in6 {
    sa_family_t     sin6_family;
    in_port_t       sin6_port;
    uint32_t        sin6_flowinfo;
    struct in6_addr sin6_addr;
    uint32_t        sin6_scope_id;
};
```

但是：

```
bind();
connect();
accept();
```

为了能够兼容不同协议族，参数通常使用通用的：

```
struct sockaddr *
```

因此调用时经常需要转换：

```
bind(fd,
     (struct sockaddr *)&addr,
     sizeof(addr));
```

如果程序需要同时处理 IPv4、IPv6 或其他地址族，可以使用：

```
struct sockaddr_storage;
```

保存足够大的通用地址结构。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9fe647c0797227de9d05edc3622fb92e_MD5.jpg]]

### 2.5 TCP Socket 与 UDP Socket 的本质区别

创建 TCP Socket：

```
socket(AF_INET, SOCK_STREAM, 0);
```

这里：

```
SOCK_STREAM
```

**意味着面向字节流的 Socket。**

典型底层协议是 TCP。

UDP：

```
socket(AF_INET, SOCK_DGRAM, 0);
```

其中：

```
SOCK_DGRAM
```

**代表数据报。**

它们最根本的区别可以先简单理解成为：

**TCP**

```
先建立连接
再连续传输字节流
提供可靠、有序传输
```

**UDP**

```
无需建立TCP式连接
直接发送一份份独立Datagram
不保证可靠、有序、必达
```

**这会导致二者的编程模型存在明显区别。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ed9ef8b9ffae8e5023b01f29fddee20b_MD5.jpg]]

## 三、TCP通信原理：连接建立之后才能传数据

### 3.1 TCP 为什么是面向连接的

**TCP 所谓的“连接”，并不是两台机器之间真的拉出一根独占物理线。**

**TCP 连接实际上是通信双方内核分别维护的一组状态。**

例如需要记录：

```
源IP
目标IP
源端口
目标端口
发送序列号
接收序列号
发送窗口
接收窗口
重传状态
拥塞窗口
TCP状态
```

**因此 TCP 在正常发送数据以前，需要先建立双方的协议状态。**

**这就是三次握手存在的重要原因。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/ee9de40c28b5d8d1ef90dacfd6760033_MD5.jpg]]

### 3.2 三次握手到底建立了什么

客户端发送：

```
SYN
SEQ = X
```

服务端返回：

```
SYN + ACK
SEQ = Y
ACK = X + 1
```

客户端再返回：

```
ACK
ACK = Y + 1
```

完成以后，双方进入：

```
ESTABLISHED
```

这个过程至少完成：

-   • 双向通信能力确认；
    
-   • 初始序列号同步；
    
-   • MSS 协商；
    
-   • Window Scale 协商；
    
-   • SACK 能力协商；
    
-   • TCP Timestamp 等选项协商。
    

**因此三次握手建立的并不只是一个“连接标志”，而是一套 TCP 传输所需要的上下文状态。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/6e202ddcfa72a9c00e47fadbf0fbd8f3_MD5.jpg]]

### 3.3 TCP 全双工通信模型

**TCP 是全双工的。**

也就是说，一条连接里面同时存在：

```
Client → Server
```

和：

```
Server → Client
```

两个独立方向。

因此连接建立以后：

```
send();
recv();
```

双方都可以执行。

例如：

```
Client                  Server

 send("hello")
     ─────────────────→

                 send("world")
     ←─────────────────
```

两个方向甚至可以同时发生。

这也是为什么 TCP 关闭连接通常不是单纯一次“断开”，而需要分别关闭两个发送方向。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/fc5cbdc46335664081fb090761e0ca5f_MD5.jpg]]

### 3.4 TCP 字节流意味着什么

**这是 Socket 编程里面最重要的知识点之一。**

TCP 提供的是：

> **可靠、有序的字节流。**

**它不保留应用程序每次 `send()` 的消息边界。**

假设客户端：

```
send(fd, "ABC", 3, 0);
send(fd, "DEF", 3, 0);
```

服务端并不一定：

```
recv(fd, buf, 3, 0);
```

刚好读到：

```
ABC
```

下一次刚好：

```
DEF
```

实际可能是：

```
第一次 recv：
AB

第二次 recv：
CDEF
```

也可能：

```
第一次 recv：
ABCDEF
```

因此：

```
send次数
```

和：

```
recv次数
```

**没有一一对应关系。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5ecbb9aba961fad43c51e72a9605e24e_MD5.jpg]]

### 3.5 四次挥手与连接释放

假设客户端主动关闭。

典型流程：

```
Client                     Server

FIN
──────────────────────────→

                         ACK
←──────────────────────────

                         FIN
←──────────────────────────

ACK
──────────────────────────→
```

为什么通常需要四次？

**因为 TCP 是全双工。**

客户端发送 FIN 只表示：

> **Client 已经没有数据再发送。**

并不表示 Server 也立即发送完了。

服务端可以先：

```
ACK Client的FIN
```

继续发送剩余数据。

等服务端自己也完成发送，再发 FIN。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/456b9f43a028d7cc6735434574834c10_MD5.jpg]]

### 3.6 TIME\_WAIT、CLOSE\_WAIT 为什么会出现

主动关闭的一方通常可能经历：

```
ESTABLISHED
   ↓
FIN_WAIT_1
   ↓
FIN_WAIT_2
   ↓
TIME_WAIT
   ↓
CLOSED
```

被动关闭的一方：

```
ESTABLISHED
   ↓
CLOSE_WAIT
   ↓
LAST_ACK
   ↓
CLOSED
```

**TIME\_WAIT**

主要作用包括：

-   • 保证最后一个 ACK 丢失后还能响应对方重发的 FIN；
    
-   • 避免旧连接延迟报文污染后续同四元组连接。
    

**因此看到 TIME\_WAIT 并不等于程序泄漏。**

**CLOSE\_WAIT**

表示：

> **对端已经关闭发送方向，本地也收到了 FIN，但是本地应用程序还没有把这个 Socket 正常关闭。**

大量 CLOSE\_WAIT 更值得检查应用程序是否：

```
忘记close
线程阻塞
连接对象泄漏
错误路径没有释放Socket
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/c5d194aa5491c5e8befa833026aa5046_MD5.jpg]]

# 四、TCP服务器：从 socket() 到 accept()

### 4.1 `socket()`：创建通信端点

TCP 服务端第一步：

```
int listen_fd;

listen_fd = socket(AF_INET,
                   SOCK_STREAM,
                   0);

if (listen_fd < 0) {
    perror("socket");
    return -1;
}
```

这里创建的是一个 TCP Socket。

但是刚刚创建的时候：

```
它还没有端口
没有进入监听状态
也没有连接任何客户端
```

可以把它理解为：

> **先创建了一个通信对象，但是还没有说明它要扮演什么角色。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/5887ec5e56cdbe6e4d35ebe4b15446a4_MD5.jpg]]

### 4.2 `bind()`：给服务器绑定 IP 和端口

服务器通常需要固定端口，否则客户端不知道应该连接哪里。

例如：

```
struct sockaddr_in addr;

memset(&addr, 0, sizeof(addr));

addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
addr.sin_addr.s_addr = htonl(INADDR_ANY);

if (bind(listen_fd,
         (struct sockaddr *)&addr,
         sizeof(addr)) < 0) {
    perror("bind");
    return -1;
}
```

其中：

```
INADDR_ANY
```

表示绑定本机所有合适的 IPv4 本地地址。

如果服务器只希望监听：

```
127.0.0.1
```

则只接受本机连接。

如果绑定：

```
192.168.1.20
```

则主要监听该本地地址。

因此 `bind()` 的核心意义是：

> **为 Socket 确定本地地址身份。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/3e879ca5efb6b92830ae82c68d6e7b3f_MD5.jpg]]

### 4.3 `listen()`：从普通 Socket 变成监听 Socket

调用：

```
listen(listen_fd, 128);
```

以后，这个 Socket 被用于被动等待连接。

可以理解为从：

```
普通TCP Socket
```

变成：

```
Listening Socket
```

它不再用于直接和某一个具体客户端交换业务数据，而是负责：

> **接收新的连接请求。**

此时客户端可以向：

```
Server IP:8080
```

发起 TCP 三次握手。

需要注意，Linux 内核中的监听连接管理比简单的“一个 backlog 队列”更加复杂。

从逻辑上可以理解为存在：

```
尚未完成握手的连接请求
```

和：

```
已经完成握手、等待accept的连接
```

两类状态。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/91ea8eb4a09635b641b5c91185b72adf_MD5.jpg]]

### 4.4 `accept()`：为什么会返回一个新的 Socket

服务器：

```
int client_fd;

client_fd = accept(listen_fd,
                   NULL,
                   NULL);
```

很多初学者最容易疑惑：

> 已经有一个 listen\_fd 了，为什么 accept 还要返回 client\_fd？

因为它们承担的职责完全不同。

`listen_fd`：

```
负责继续监听8080端口
```

`client_fd`：

```
专门代表某一个已经建立的TCP连接
```

例如：

```
listen_fd
    │
    ├── client_fd_1
    │      192.168.1.10:50001
    │          ↕
    │      192.168.1.20:8080
    │
    ├── client_fd_2
    │      192.168.1.11:50002
    │          ↕
    │      192.168.1.20:8080
    │
    └── client_fd_3
```

**服务器必须保留 `listen_fd`，继续接受后续新客户端。**

**每一个 `accept()` 返回的连接 Socket，则用于服务对应客户端。**

![](https://relay-1.bijitongbu.site/p/e18577cf4258d3b29e6a209904ab64c8.png)

### 4.5 `recv()/read()`：接收客户端数据

```
char buf[1024];

ssize_t n = recv(client_fd,
                 buf,
                 sizeof(buf),
                 0);
```

**返回值极为重要。**

**`n > 0`**

成功读取：

```
n Byte
```

**`n == 0`**

对端已经进行了正常关闭，当前 TCP 字节流到达 EOF。

通常需要：

```
close(client_fd);
```

**`n < 0`**

发生错误。

需要检查：

```
errno
```

例如：

```
EINTR
EAGAIN
ECONNRESET
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/4f42b308f7fc685e75a555a77a0b1c5c_MD5.jpg]]

### 4.6 `send()/write()`：向客户端发送数据

例如 Echo Server：

```
send(client_fd,
     buf,
     n,
     0);
```

但是必须注意：

```
send();
```

**的返回值表示实际被接受的字节数。**

不能永远假定：

```
send(fd, buf, 1000, 0);
```

一定返回：

```
1000
```

**尤其是非阻塞 Socket、大量数据发送或者被信号打断时，需要正确处理 Partial Write。**

通常会编写：

```
ssize_t send_all(int fd,
                 const void *buf,
                 size_t len)
{
    const char *p = buf;
    size_t total = 0;

    while (total < len) {
        ssize_t n = send(fd,
                         p + total,
                         len - total,
                         MSG_NOSIGNAL);

        if (n > 0) {
            total += n;
            continue;
        }

        if (n < 0 && errno == EINTR)
            continue;

        return -1;
    }

    return (ssize_t)total;
}
```

![](https://relay-1.bijitongbu.site/p/64f1b570be07e06b7e400ae8f6a60240.png)

### 4.7 `close()`：关闭连接

```
close(client_fd);
```

表示应用程序释放这个文件描述符。

对于 TCP Socket，内核会根据连接状态进行后续协议关闭处理。

但是：

```
close();
```

**返回并不代表四次挥手已经在网络上全部完成。**

内核可能继续维护：

```
FIN发送
FIN重传
ACK
TIME_WAIT
```

等状态。

![](https://relay-1.bijitongbu.site/p/d3761eea37b7b86ad448367bd3fa38dc.png)

### 4.8 TCP服务器完整程序执行流程

最基础 TCP Server：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define SERVER_PORT 8080

int main(void)
{
    int listen_fd;
    int client_fd;

    struct sockaddr_in server_addr;
    char buf[1024];

    listen_fd = socket(AF_INET,
                       SOCK_STREAM,
                       0);

    if (listen_fd < 0) {
        perror("socket");
        return 1;
    }

    memset(&server_addr,
           0,
           sizeof(server_addr));

    server_addr.sin_family = AF_INET;
    server_addr.sin_port =
        htons(SERVER_PORT);

    server_addr.sin_addr.s_addr =
        htonl(INADDR_ANY);

    if (bind(listen_fd,
             (struct sockaddr *)&server_addr,
             sizeof(server_addr)) < 0) {
        perror("bind");
        close(listen_fd);
        return 1;
    }

    if (listen(listen_fd, 128) < 0) {
        perror("listen");
        close(listen_fd);
        return 1;
    }

    printf("server listening on %d\n",
           SERVER_PORT);

    for (;;) {

        client_fd = accept(listen_fd,
                           NULL,
                           NULL);

        if (client_fd < 0) {
            perror("accept");
            continue;
        }

        for (;;) {
            ssize_t n;

            n = recv(client_fd,
                     buf,
                     sizeof(buf),
                     0);

            if (n > 0) {
                send(client_fd,
                     buf,
                     n,
                     MSG_NOSIGNAL);
            } else {
                break;
            }
        }

        close(client_fd);
    }

    close(listen_fd);
    return 0;
}
```

**这个程序能够工作，但是它存在一个明显问题：**

> **同一时刻只能认真服务一个客户端。**

这会推动我们后面进入多进程、多线程以及 epoll。

![](https://relay-1.bijitongbu.site/p/b90a6106d3a8052ba24d67a58bb6513f.png)

## 五、TCP客户端：connect() 背后发生了什么

### 5.1 客户端为什么通常不需要主动 bind

服务端需要固定端口：

```
8080
```

因为所有客户端都需要知道到哪里连接。

客户端通常不需要固定本地端口。

例如：

```
socket();
connect();
```

在 `connect()` 过程中，Linux 可以自动选择：

```
本地源IP
临时源端口
```

形成：

```
192.168.1.10:52731
        ↓
192.168.1.20:8080
```

其中：

```
52731
```

**通常由内核从临时端口范围中选择。**

客户端当然也可以主动 `bind()`，例如需要：

-   • 指定源 IP；
    
-   • 指定本地端口；
    
-   • 多网卡场景；
    
-   • 特定协议设计。
    

但一般普通客户端不需要。

![](https://relay-1.bijitongbu.site/p/684239413b824f174354b2334f8126aa.png)

### 5.2 `socket()` 创建客户端 Socket

```
int fd;

fd = socket(AF_INET,
            SOCK_STREAM,
            0);
```

此时只是创建 TCP Socket。

真正确定对端是在：

```
connect();
```

阶段。

![](https://relay-1.bijitongbu.site/p/629601449495558311904a4d62cbc3cb.png)

### 5.3 `connect()` 与 TCP 三次握手

例如：

```
struct sockaddr_in server;

server.sin_family = AF_INET;
server.sin_port = htons(8080);

inet_pton(AF_INET,
          "192.168.1.20",
          &server.sin_addr);

connect(fd,
        (struct sockaddr *)&server,
        sizeof(server));
```

**阻塞 Socket 中，`connect()` 通常会等待连接建立成功或者失败。**

底层发生：

```
客户端选择本地IP和临时端口
        ↓
发送SYN
        ↓
等待SYN-ACK
        ↓
发送ACK
        ↓
进入ESTABLISHED
        ↓
connect返回
```

如果目标主机对应端口没有程序监听，可能收到 RST，然后：

```
connect();
```

返回：

```
ECONNREFUSED
```

如果网络路径直接丢弃 SYN，则可能经历 SYN 重试，最终连接超时。

![](https://relay-1.bijitongbu.site/p/e40637289e7b0aa61e51cc8fd7c6b4eb.png)

### 5.4 客户端的数据发送与接收

连接建立以后：

```
send(fd,
     "hello",
     5,
     0);
```

然后：

```
recv(fd,
     buf,
     sizeof(buf),
     0);
```

需要注意，TCP 并没有规定：

```
客户端必须先send
服务器才能send
```

**只要连接已经 ESTABLISHED，双方都可以根据应用协议决定收发时机。**

![](https://relay-1.bijitongbu.site/p/a1536787a760bcfec8698d353e7b2f5d.png)

### 5.5 客户端主动断开连接

客户端调用：

```
close(fd);
```

通常会成为主动关闭方。

因此客户端可能进入：

```
FIN_WAIT_1
FIN_WAIT_2
TIME_WAIT
```

**如果一个客户端程序每秒大量建立和主动关闭短连接，就可能看到许多 TIME\_WAIT。**

这并不一定是服务器问题。

![](https://relay-1.bijitongbu.site/p/12e3405e64385a5f043771c933913f65.png)

### 5.6 TCP客户端完整程序执行流程

```
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

int main(void)
{
    int fd;
    struct sockaddr_in server;

    char send_buf[] = "hello";
    char recv_buf[1024];

    fd = socket(AF_INET,
                SOCK_STREAM,
                0);

    if (fd < 0) {
        perror("socket");
        return 1;
    }

    memset(&server, 0, sizeof(server));

    server.sin_family = AF_INET;
    server.sin_port = htons(8080);

    if (inet_pton(AF_INET,
                  "127.0.0.1",
                  &server.sin_addr) != 1) {
        perror("inet_pton");
        close(fd);
        return 1;
    }

    if (connect(fd,
                (struct sockaddr *)&server,
                sizeof(server)) < 0) {
        perror("connect");
        close(fd);
        return 1;
    }

    if (send(fd,
             send_buf,
             strlen(send_buf),
             MSG_NOSIGNAL) < 0) {
        perror("send");
        close(fd);
        return 1;
    }

    ssize_t n = recv(fd,
                     recv_buf,
                     sizeof(recv_buf) - 1,
                     0);

    if (n > 0) {
        recv_buf[n] = '\0';
        printf("recv: %s\n", recv_buf);
    }

    close(fd);
    return 0;
}
```

![](https://relay-1.bijitongbu.site/p/cae17802530914ef7ed5016b9e31ad12.png)

## 六、一次TCP通信全过程：把客户端和服务器串起来

### 6.1 服务端启动并监听端口

服务端：

```
socket
 ↓
bind :8080
 ↓
listen
```

内核中建立监听状态。

此时执行：

```
ss -lnt
```

可能看到：

```
LISTEN
0.0.0.0:8080
```

![](https://relay-1.bijitongbu.site/p/a73bf6671befdd154fc51da07d29b9d2.png)

### 6.2 客户端发起连接

客户端：

```
connect();
```

导致内核发出：

```
SYN
```

服务端监听 Socket 收到以后返回：

```
SYN-ACK
```

客户端返回：

```
ACK
```

三次握手完成。

![](https://relay-1.bijitongbu.site/p/9317fe01cf22b0e3016a09f28a30dbcb.png)

### 6.3 accept 如何建立新的已连接 Socket

握手完成以后，服务端内核已经拥有一个已建立连接对象。

它进入等待应用取走的连接队列。

服务端调用：

```
accept();
```

从中取出一个连接并创建新的文件描述符：

```
listen_fd
     │
     └── accept
           ↓
        client_fd
```

因此严格来说：

> **`accept()` 并不是三次握手的发起者。**

**在正常情况下，三次握手主要由内核协议栈完成。**

**`accept()` 是应用程序从内核取得一条已经准备好的连接。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/fa6df5d256be1c9d67d8805b89550533_MD5.jpg]]

### 6.4 数据如何从用户态进入内核发送缓冲区

客户端：

```
send(fd, buf, len, 0);
```

数据大致经过：

```
用户态buf
   ↓
系统调用
   ↓
内核Socket
   ↓
Socket发送缓存
   ↓
TCP发送队列
```

**如果发送缓冲区有空间，`send()` 可以较快返回。**

**如果发送缓存已满，阻塞 Socket 可能等待空间。**

非阻塞 Socket 则可能返回：

```
EAGAIN
EWOULDBLOCK
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9eaa333ced54fc215fb4a28df8352e0d_MD5.jpg]]

### 6.5 数据如何经过 TCP/IP 协议栈和网卡

之后 TCP 会综合考虑：

```
MSS
发送窗口
接收窗口
拥塞窗口
Nagle
重传状态
```

决定什么时候发送多少数据。

然后：

```
TCP
 ↓
IP
 ↓
路由
 ↓
邻居解析
 ↓
Qdisc
 ↓
网卡驱动
 ↓
NIC TX Ring
 ↓
DMA
 ↓
网卡
```

网卡最终把数据转换成为物理信号发送出去。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9626ff40db7aa0e597cc1fe0d35dc160_MD5.jpg]]

### 6.6 对端如何接收并交给应用程序

服务端网卡接收到帧：

```
网卡RX
  ↓
DMA写入内存
  ↓
驱动/NAPI
  ↓
Ethernet
  ↓
IP
  ↓
TCP
  ↓
查找对应Socket
  ↓
接收队列
  ↓
唤醒阻塞recv
  ↓
用户程序得到数据
```

因此：

```
recv();
```

**不是直接从网卡读数据。**

它读取的是：

> **当前已经被 Linux TCP 协议栈接收并放入该 Socket 接收队列的数据。**

![](https://relay-1.bijitongbu.site/p/3daff10b728bf0a3409a15c853ef3b63.png)

### 6.7 从连接建立到连接关闭的完整时序图

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a5fa8cd7fb02e56971d6e69319562271_MD5.jpg]]

## 七、UDP编程：没有连接也能通信

### 7.1 UDP 为什么不需要 listen 和 accept

UDP 没有 TCP 那种连接状态。

服务端不需要等待：

```
SYN
SYN-ACK
ACK
```

因此也不需要：

```
listen();
accept();
```

一个 UDP Socket 绑定端口以后，就可以接收来自多个对端的数据报。

```
Client A ─── Datagram ───┐
                         │
Client B ─── Datagram ───┼→ UDP Server Socket
                         │
Client C ─── Datagram ───┘
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/c1501e76b441e6e699b65033e6ecb806_MD5.jpg]]

### 7.2 `sendto()` 与 `recvfrom()`

UDP 发送时需要知道：

> **这一个 Datagram 发给谁。**

因此：

```
sendto(fd,
       buf,
       len,
       0,
       (struct sockaddr *)&peer,
       sizeof(peer));
```

接收：

```
recvfrom(fd,
         buf,
         sizeof(buf),
         0,
         (struct sockaddr *)&peer,
         &peer_len);
```

可以同时得到：

```
数据
发送者地址
```

![](https://relay-1.bijitongbu.site/p/fcf078a8b136e6c4769ee108166a93e5.png)

### 7.3 UDP服务端编程流程

典型流程：

```
socket(SOCK_DGRAM)
      ↓
bind
      ↓
recvfrom
      ↓
sendto
```

例如：

```
int fd = socket(AF_INET,
                SOCK_DGRAM,
                0);

bind(fd,
     (struct sockaddr *)&addr,
     sizeof(addr));

for (;;) {
    struct sockaddr_in peer;
    socklen_t peer_len = sizeof(peer);

    ssize_t n = recvfrom(fd,
                         buf,
                         sizeof(buf),
                         0,
                         (struct sockaddr *)&peer,
                         &peer_len);

    if (n > 0) {
        sendto(fd,
               buf,
               n,
               0,
               (struct sockaddr *)&peer,
               peer_len);
    }
}
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0d88c62f0484bbe4c88af5dd8ece5b18_MD5.jpg]]

### 7.4 UDP客户端编程流程

客户端可以直接：

```
socket
 ↓
sendto
 ↓
recvfrom
```

普通 UDP 通信甚至可以不调用 `connect()`。

不过 UDP Socket 也允许调用：

```
connect();
```

这里不会建立 TCP 三次握手，而是给 Socket 设置默认对端，使应用可以使用：

```
send();
recv();
```

并让内核过滤其他对端的数据报。

![](https://relay-1.bijitongbu.site/p/8150c1b7b48957c87fdfc1e1317ddb89.png)

### 7.5 UDP数据报与TCP字节流的区别

TCP：

```
send("ABC")
send("DEF")

接收端看到的是连续字节流：
ABCDEF
```

UDP：

```
sendto("ABC")
sendto("DEF")
```

会形成两个独立 Datagram。

接收端：

```
第一个recvfrom
→ ABC

第二个recvfrom
→ DEF
```

**UDP 保留数据报边界。**

但是如果接收缓冲区太小：

```
recvfrom(fd, buf, 2, ...);
```

去接收：

```
ABC
```

剩余部分一般不会像 TCP 一样留给下一次读取。

**因此 UDP 应用必须预留足够的 Datagram 接收空间。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/919f876f523b73fadc95034ecd6ca1e0_MD5.jpg]]

### 7.6 TCP和UDP到底应该怎么选

TCP 更适合：

-   • 文件传输；
    
-   • HTTP；
    
-   • SSH；
    
-   • 数据库；
    
-   • 需要可靠有序传输；
    
-   • 业务天然是连续字节流。
    

UDP 更适合：

-   • 实时音视频；
    
-   • DNS；
    
-   • 局域网发现；
    
-   • 遥测；
    
-   • 自定义可靠协议；
    
-   • 能够容忍少量丢包；
    
-   • 需要保留报文边界。
    

不能简单认为：

```
TCP慢
UDP快
```

真正应该判断的是：

> **业务需要什么语义？**

如果最终需要在 UDP 上自行实现：

```
序号
ACK
重传
拥塞控制
重排序
```

系统复杂度也会迅速增加。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/702a1f9dd257a4e95b18046dd3045c5f_MD5.jpg]]

## 八、Socket数据收发：为什么一次 send 不等于一次 recv

### 8.1 发送缓冲区与接收缓冲区

每一个 TCP Socket 都存在发送和接收相关缓存。

可以高度抽象成为：

```
应用程序
    │
 send()
    ↓
┌───────────────┐
│ Socket Send   │
│ Buffer        │
└───────────────┘
        ↓
       TCP
        ↓
      网络
        ↓
       TCP
        ↓
┌───────────────┐
│ Socket Receive│
│ Buffer        │
└───────────────┘
        ↓
      recv()
        ↓
    应用程序
```

这两个 Buffer 把：

```
应用程序处理速度
```

和：

```
网络发送接收速度
```

进行了一定程度的解耦。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a0483ea0d3f5709bd1e15b8f08159e01_MD5.jpg]]

### 8.2 `send()` 到底把数据发送到了哪里

例如：

```
ret = send(fd,
           data,
           4096,
           0);
```

如果返回：

```
4096
```

更加准确的理解是：

> **Linux Socket 层已经接受了这 4096 Byte，后续由内核负责发送。**

它不代表：

```
对端recv已经读到4096 Byte
```

甚至也不严格代表：

```
网卡已经把4096 Byte全部发出
```

这就是为什么应用层协议如果需要知道：

> 对端业务真的处理成功了吗？

**必须自己设计应用层 ACK。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/18964b8f19a38c75789095ddfe301536_MD5.jpg]]

### 8.3 `recv()` 为什么可能只读到部分数据

假设对端发送：

```
1000 Byte
```

你的：

```
recv(fd,
     buf,
     1000,
     0);
```

可能只返回：

```
200
```

为什么？

因为此时内核接收队列里面可能只有 200 Byte 可以立即交付。

剩余数据：

-   • 还在网络中；
    
-   • 还在网卡；
    
-   • 正在协议栈处理；
    
-   • 还没有到达。
    

TCP 保证的是：

```
字节顺序
```

不是：

```
一次recv把你想要的所有数据全读完
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e2de000c23a98a23a6ef26f60345083d_MD5.jpg]]

### 8.4 TCP为什么没有“消息边界”

假设：

```
send(fd, "ABC", 3, 0);
send(fd, "123", 3, 0);
```

TCP 内核关注的是连续序列空间：

```
A B C 1 2 3
```

它不会记录：

```
前三字节来自send1
后三字节来自send2
```

因此 TCP 没有应用消息边界。

**这是协议设计者必须自己解决的问题。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/2a13e0678ad95085a9692a32a80b7e8f_MD5.jpg]]

### 8.5 粘包和拆包到底是什么

所谓“粘包”：

发送端：

```
Packet A
Packet B
```

接收端一次：

```
recv();
```

读到了：

```
Packet A + Packet B
```

所谓“拆包”或者“半包”：

发送端一个 Packet：

```
ABCDEFGH
```

接收：

```
recv1 → ABC
recv2 → DEFGH
```

**这些并不代表 TCP 错误。**

它们只是：

> **TCP 字节流与应用消息之间缺少边界定义。**

![](https://relay-1.bijitongbu.site/p/61ca8aa680bfaec14bcadd5cc81cec19.png)

### 8.6 固定长度、分隔符与Length+Body协议设计

**固定长度**

规定：

```
每帧64 Byte
```

优点简单。

缺点浪费空间，并且不适合数据长度变化大的业务。

**分隔符**

例如文本协议：

```
hello\r\n
world\r\n
```

适合：

-   • 命令行；
    
-   • 文本协议。
    

但是 Payload 中出现分隔符时需要转义或者编码。

**Length + Body**

更加通用：

```
+--------+--------+
| Length | Body   |
+--------+--------+
| 4 Byte | N Byte |
+--------+--------+
```

例如：

```
00 00 00 05
48 65 6C 6C 6F
```

接收端先读取 4 Byte Length：

```
5
```

然后等待 5 Byte Body。

**这是很多二进制协议常见的设计。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/940674134efe54165d2d2995a72ed963_MD5.jpg]]

### 8.7 如何正确处理半包和多包

推荐使用接收 Buffer 加解析循环。

例如：

```
Socket recv
    ↓
追加到应用层buffer
    ↓
buffer中是否有完整Header？
    ↓
解析Length
    ↓
是否有完整Packet？
    ├── 否 → 等下一次recv
    └── 是
         ↓
      取出Packet
         ↓
      继续检查buffer中是否还有下一包
```

伪代码：

```
while (1) {

    n = recv(fd,
             temp,
             sizeof(temp),
             0);

    if (n <= 0)
        break;

    buffer_append(&conn->rxbuf,
                  temp,
                  n);

    while (1) {

        if (buffer_size(&conn->rxbuf) < HEADER_SIZE)
            break;

        uint32_t body_len =
            parse_length(&conn->rxbuf);

        size_t packet_len =
            HEADER_SIZE + body_len;

        if (buffer_size(&conn->rxbuf) < packet_len)
            break;

        handle_packet(buffer_data(&conn->rxbuf),
                      packet_len);

        buffer_consume(&conn->rxbuf,
                       packet_len);
    }
}
```

**这才是 TCP 应用层正确解析方式。**

![](https://relay-1.bijitongbu.site/p/9d4a34498ae421afdfe19d22bff7adc5.png)

## 九、阻塞与非阻塞：程序为什么会“卡住”

### 9.1 什么是阻塞 Socket

**默认创建的 Socket 通常工作在阻塞模式。**

所谓阻塞，是指：

> **当前操作暂时无法完成时，调用线程进入睡眠等待。**

例如：

```
recv(fd, buf, 1024, 0);
```

当前没有任何数据。

那么线程可能暂停执行，直到：

```
有数据到达
对端关闭
发生错误
收到信号
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/541305df6c7a0a4e919b99e3a963cfc4_MD5.jpg]]

### 9.2 accept、recv、send 分别会在哪里阻塞

**`accept()`**

当：

```
没有已经完成并等待应用取走的新连接
```

时阻塞。

**`recv()`**

当：

```
当前没有可读取数据
连接也没有关闭
```

时阻塞。

**`send()`**

**很多人认为 `send()` 永远不会阻塞，这是错误的。**

如果：

```
Socket发送缓冲区已经很满
```

而网络又暂时无法继续发送数据，阻塞模式的 `send()` 可能等待缓存空间。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/546aabe8dff6a75a832c0769e2ec21bd_MD5.jpg]]

### 9.3 `fcntl()` 设置非阻塞模式

获取原 Flag：

```
int flags = fcntl(fd,
                  F_GETFL,
                  0);
```

设置：

```
fcntl(fd,
      F_SETFL,
      flags | O_NONBLOCK);
```

可以封装：

```
int set_nonblocking(int fd)
{
    int flags;

    flags = fcntl(fd,
                  F_GETFL,
                  0);

    if (flags < 0)
        return -1;

    return fcntl(fd,
                 F_SETFL,
                 flags | O_NONBLOCK);
}
```

![](https://relay-1.bijitongbu.site/p/a34342faac7a786b07d4ce11e5f55264.png)

### 9.4 `EAGAIN/EWOULDBLOCK` 到底意味着什么

非阻塞模式调用：

```
recv();
```

当前没有数据时，不会让线程睡眠。

而是立即返回：

```
-1
```

并设置：

```
errno = EAGAIN
```

或者：

```
EWOULDBLOCK
```

它的含义不是：

> Socket 坏了。

而是：

> **现在暂时做不了，请以后再试。**

同样，非阻塞 `send()` 如果发送缓存暂时没有空间，也可能返回 `EAGAIN`。

![](https://relay-1.bijitongbu.site/p/d5ab3545eda126bdb1eb16074d961abb.png)

### 9.5 非阻塞为什么不等于异步

这是非常重要的区别。

非阻塞：

```
操作不能立即完成
→ 函数直接返回
→ 你以后自己再尝试
```

它仍然是：

```
你的线程主动调用read/send
```

异步则更强调：

```
提交操作
    ↓
系统在后续完成
    ↓
完成时通知应用
```

因此：

```
O_NONBLOCK
```

**并不等价于真正意义上的异步 IO。**

`epoll` 本身也主要是一种：

```
事件就绪通知机制
```

它告诉你：

> **现在某个 fd 很可能可以读或者写了。**

**真正的数据传输仍然通常由应用调用 `recv()`、`send()` 完成。**

![](https://relay-1.bijitongbu.site/p/89a7775841a10e9bc97478816be7aa4a.png)

### 9.6 阻塞与非阻塞应该怎么选择

少量连接、程序简单：

```
阻塞IO
```

往往更加容易实现。

例如：

-   • 小工具；
    
-   • 命令行客户端；
    
-   • 低并发控制程序。
    

大量连接、事件驱动：

```
非阻塞 + epoll
```

通常更加合适。

关键不在于：

```
非阻塞一定高级
```

而是：

> **并发模型和业务规模是否需要。**

![](https://relay-1.bijitongbu.site/p/01f9cd12788051b4b5dae61601ac3150.png)

## 十、多客户端并发：一个服务器如何同时服务万人

### 10.1 单线程阻塞模型的问题

最简单服务器：

```
accept Client A
     ↓
recv A
     ↓
处理A
     ↓
send A
     ↓
继续recv A
```

如果 A 一直不发送数据：

```
recv(A);
```

会阻塞。

那么：

```
Client B
Client C
Client D
```

即使已经连接，也得不到及时处理。

**因此一个阻塞线程很难直接同时管理大量独立连接。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d36afbf263d678bd12908c195aa14f9e_MD5.jpg]]

### 10.2 多进程并发模型

一种经典做法：

```
Main Process
     │
     ├─ accept Client A → fork Process A
     ├─ accept Client B → fork Process B
     └─ accept Client C → fork Process C
```

优点：

-   • 隔离性强；
    
-   • 一个子进程崩溃不一定影响其他连接；
    
-   • 编程逻辑简单。
    

缺点：

-   • 进程创建成本；
    
-   • 内存开销；
    
-   • 上下文切换；
    
-   • 进程间共享数据麻烦。
    

适合连接数不是极大的传统服务器模型。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/a3d4169d4e2e153bf39183c7cec63c05_MD5.jpg]]

### 10.3 多线程并发模型

```
Main Thread
    │
    ├── Thread A → Client A
    ├── Thread B → Client B
    └── Thread C → Client C
```

优点：

-   • 共享地址空间；
    
-   • 通信方便；
    
-   • 每个客户端逻辑直观。
    

缺点：

-   • 线程栈占用；
    
-   • 大量上下文切换；
    
-   • 锁竞争；
    
-   • 线程数量不能无限增加。
    

**一万个连接就创建一万个线程，通常并不是理想方案。**

![](https://relay-1.bijitongbu.site/p/b408b982ddcde29e1f04438964b125f3.png)

### 10.4 线程池模型

更加合理的设计：

```
Accept Thread
     ↓
Task Queue
     ↓
┌──────────────┐
│ Worker Thread│
│ Worker Thread│
│ Worker Thread│
└──────────────┘
```

线程数量固定。

任务动态分配给工作线程。

避免：

```
每个请求都创建和销毁线程
```

**但是如果每一个工作线程仍然在一个慢 Socket 上长期阻塞，线程池容量同样可能被占满。**

![](https://relay-1.bijitongbu.site/p/04596d1725b0e007030a31f4c836faf5.png)

### 10.5 IO多路复用模型

于是出现另一种思路：

> **不给每一个连接单独创建一个线程，而是让一个线程同时等待很多 fd。**

例如：

```
fd 10
fd 11
fd 12
fd 13
...
```

交给：

```
select
poll
epoll
```

等待。

当某几个真正有事件时：

```
fd 11 readable
fd 13 readable
```

应用只处理它们。

**这就是 IO 多路复用。**

![](https://relay-1.bijitongbu.site/p/a089bd1dfdb8af018a6a8d12c05920f7.png)

### 10.6 Reactor事件驱动模型

Reactor 的基本思想：

```
事件源
   ↓
事件多路复用器
   ↓
事件循环
   ↓
Handler
```

例如：

```
epoll_wait()
      ↓
有新连接？
      ↓
accept

某连接可读？
      ↓
recv + parse

某连接可写？
      ↓
flush send buffer
```

**一个事件循环能够管理大量连接。**

这也是高性能网络程序中极其常见的结构。

![](https://relay-1.bijitongbu.site/p/dfb421b869975693108292f88188ad22.png)

### 10.7 不同并发模型的性能与适用场景

![](https://relay-1.bijitongbu.site/p/00aeea8d343df3145eeaddf608303c19.png)

  

## 十一、select、poll与epoll：Linux高并发的核心

### 11.1 IO多路复用到底“复用”了什么

所谓“复用”，并不是：

> 多个连接共用同一个 Socket。

而是：

> **一个线程使用同一个等待机制，同时观察多个 fd 的 IO 状态。**

例如：

```
Thread
  │
  └── epoll_wait
          │
          ├── fd3
          ├── fd7
          ├── fd11
          ├── fd25
          └── fd1000
```

哪个 fd 就绪，就处理哪个。

![](https://relay-1.bijitongbu.site/p/041632261f65317475f5d40ba47e8758.png)

### 11.2 `select()` 的工作原理

基本接口：

```
select(maxfd + 1,
       &readfds,
       &writefds,
       &exceptfds,
       &timeout);
```

应用准备：

```
fd_set readfds;

FD_ZERO(&readfds);
FD_SET(listen_fd, &readfds);
FD_SET(client1, &readfds);
FD_SET(client2, &readfds);
```

调用 `select()`。

返回以后，通过：

```
FD_ISSET(fd, &readfds)
```

判断哪个 fd 就绪。

![](https://relay-1.bijitongbu.site/p/f817e485bf000c0309f57d47c609565a.png)

### 11.3 select 为什么有 FD 数量限制

`select()` 使用固定大小：

```
fd_set
```

表示 fd 集合。

在很多 Linux 用户空间环境中：

```
FD_SETSIZE
```

典型值是：

```
1024
```

如果文件描述符数值超过这个范围，就不能直接安全地放入默认 `fd_set`。

除此之外，`select()` 每次调用：

-   • 都需要重新准备 fd\_set；
    
-   • 内核需要检查 fd；
    
-   • 用户空间返回后还需要遍历集合查找就绪项。
    

**大量 fd 时效率会逐渐下降。**

![](https://relay-1.bijitongbu.site/p/1e224a8896f7518b3575d74b97334ddd.png)

### 11.4 `poll()` 做了哪些改进

`poll()` 使用数组：

```
struct pollfd {
    int fd;
    short events;
    short revents;
};
```

调用：

```
poll(fds,
     nfds,
     timeout);
```

**它不再依赖固定大小的 `fd_set`，因此不受 `FD_SETSIZE` 这种固定集合表示的直接限制。**

但是：

```
大量fd
```

时，应用和内核仍然需要处理整个 `pollfd` 数组。

**即使只有少量连接发生事件，也需要在很多项中寻找。**

![](https://relay-1.bijitongbu.site/p/2f4d1c1d5130e4dd34191b9cd0e4fcc8.png)

### 11.5 `epoll_create/epoll_ctl/epoll_wait`

创建 epoll：

```
int epfd = epoll_create1(EPOLL_CLOEXEC);
```

添加 fd：

```
struct epoll_event ev;

ev.events = EPOLLIN;
ev.data.fd = listen_fd;

epoll_ctl(epfd,
          EPOLL_CTL_ADD,
          listen_fd,
          &ev);
```

等待：

```
struct epoll_event events[128];

int n = epoll_wait(epfd,
                   events,
                   128,
                   -1);
```

返回以后：

```
for (int i = 0; i < n; i++) {
    int fd = events[i].data.fd;

    ...
}
```

修改：

```
epoll_ctl(epfd,
          EPOLL_CTL_MOD,
          fd,
          &ev);
```

删除：

```
epoll_ctl(epfd,
          EPOLL_CTL_DEL,
          fd,
          NULL);
```

![](https://relay-1.bijitongbu.site/p/4971c692cb85c448ec98c3e833916c99.png)

### 11.6 epoll 为什么适合高并发

epoll 的重要设计特点之一是：

> **关注的 fd 集合长期保存在内核中。**

**应用不需要像 `select()` 那样每次重新提交整个 fd 集合。**

同时 epoll 会维护：

```
Interest List
Ready List
```

当 fd 状态发生变化并满足事件条件时，相关就绪信息进入 epoll 的就绪管理结构。

应用调用：

```
epoll_wait();
```

主要获得的是：

```
当前已经就绪的事件
```

而不是每次从头检查全部连接。

因此在：

```
连接数量很多
真正活跃连接只占少数
```

的典型高并发服务器场景中，epoll 很有优势。

需要注意：

> **epoll 并不是说所有操作都是绝对 O(1)，也不是只要换成 epoll 程序就一定快。**

真正性能仍然受到：

-   • 业务逻辑；
    
-   • 锁；
    
-   • 内存分配；
    
-   • 数据复制；
    
-   • 系统调用数量；
    
-   • 网络协议；
    
-   • CPU；
    
-   • NUMA；
    
-   • fd 活跃比例。
    

等因素影响。

![](https://relay-1.bijitongbu.site/p/c2b8c789f125a72e43e6bd7456e941e1.png)

### 11.7 LT水平触发与ET边缘触发

**LT：Level Triggered**

只要 fd 当前仍然满足条件：

```
还有数据没读完
```

下一次 `epoll_wait()` 仍然可以继续报告可读。

例如：

```
Socket有1000Byte
应用只读100Byte
```

剩余：

```
900Byte
```

仍然可读。

下一次 epoll 会继续通知。

LT 更容易编程，也是 epoll 默认模式。

**ET：Edge Triggered**

更加关注：

> **状态从“不就绪”变成“就绪”的边沿。**

假设数据到达触发一次事件。

应用如果只读取一部分，然后停止，剩余数据可能不会因为“它一直都已经是可读状态”而再次产生同样的边沿提醒。

**所以 ET 下必须一次尽量把数据读干净。**

![](https://relay-1.bijitongbu.site/p/d5252bbc759904087cd804c21d57ddf3.png)

### 11.8 ET模式为什么必须配合非阻塞IO

ET 常见读取模式：

```
for (;;) {
    ssize_t n = recv(fd,
                     buf,
                     sizeof(buf),
                     0);

    if (n > 0) {
        handle(buf, n);
        continue;
    }

    if (n < 0 &&
        (errno == EAGAIN ||
         errno == EWOULDBLOCK)) {
        break;
    }

    ...
}
```

也就是：

> **一直读到 EAGAIN。**

如果 fd 是阻塞模式：

```
当前所有数据读完
```

以后再调用一次 `recv()`，线程可能直接阻塞在那里。

那么整个事件循环就卡死了。

因此 ET 模式通常必须和：

```
O_NONBLOCK
```

结合。

![](https://relay-1.bijitongbu.site/p/22a3a5daec0d4b6197428c152897a8d0.png)

### 11.9 select、poll、epoll完整对比

![](https://relay-1.bijitongbu.site/p/e6a4859d9ddf6effd7211dfd7c420617.png)

  

## 十二、Socket常用选项：很多网络问题其实藏在这里

### 12.1 `setsockopt()` 到底在设置什么

Socket 本身拥有大量可配置属性。

接口：

```
setsockopt(fd,
           level,
           optname,
           &value,
           sizeof(value));
```

例如：

```
SOL_SOCKET
```

表示 Socket 通用层。

```
IPPROTO_TCP
```

表示 TCP 协议层。

因此不同 Option 实际对应不同层次。

![](https://relay-1.bijitongbu.site/p/5f5ef701ce9b58ace779c9f540eccae4.png)

### 12.2 `SO_REUSEADDR`：端口为什么无法立即重新绑定

典型设置：

```
int on = 1;

setsockopt(fd,
           SOL_SOCKET,
           SO_REUSEADDR,
           &on,
           sizeof(on));
```

它允许在 Linux 规定的地址复用条件下更加灵活地重新绑定本地地址，常用于服务器重启等场景。

但必须注意：

> **`SO_REUSEADDR` 并不等于“无条件忽略所有端口占用”。**

例如另一个正常运行的进程已经以冲突方式绑定同一个地址和端口，不能认为设置它以后就必然能够抢占。

此外：

```
TIME_WAIT
```

和：

```
listen端口不能bind
```

之间的关系也没有网上某些简单说法那么直接。

更加正确的理解是：

> **`SO_REUSEADDR` 是本地地址绑定策略的一部分，而不是一个“强行清除TIME\_WAIT”的开关。**

![](https://relay-1.bijitongbu.site/p/e5cc0a02ebb2af55433b7d11d1521dfd.png)

### 12.3 `SO_KEEPALIVE`：如何检测失效连接

TCP 连接可能出现：

```
网线被拔掉
路由器断电
对端系统突然崩溃
```

但是本地如果一直没有发送任何数据，可能无法立即知道。

开启：

```
int on = 1;

setsockopt(fd,
           SOL_SOCKET,
           SO_KEEPALIVE,
           &on,
           sizeof(on));
```

允许 TCP 在连接长时间空闲以后发送 Keepalive Probe。

Linux 还可以调整：

```
TCP_KEEPIDLE
TCP_KEEPINTVL
TCP_KEEPCNT
```

不过默认 Keepalive 时间通常比较长。

因此对于：

```
秒级故障检测
```

很多应用会自己设计心跳，而不是完全依赖系统 TCP Keepalive。

![](https://relay-1.bijitongbu.site/p/50b85a11ec83c31eaa0e81979afd3a73.png)

### 12.4 `SO_RCVBUF/SO_SNDBUF`：调整收发缓冲区

发送 Buffer：

```
int size = 1024 * 1024;

setsockopt(fd,
           SOL_SOCKET,
           SO_SNDBUF,
           &size,
           sizeof(size));
```

接收：

```
setsockopt(fd,
           SOL_SOCKET,
           SO_RCVBUF,
           &size,
           sizeof(size));
```

缓冲区大小会影响：

-   • 可暂存数据量；
    
-   • 高 BDP 链路吞吐；
    
-   • 内存占用；
    
-   • 应用短时处理抖动。
    

但是不能简单认为：

```
缓冲区越大性能越好
```

过大的缓存可能：

-   • 占用大量内存；
    
-   • 增加排队；
    
-   • 隐藏应用消费能力不足问题。
    

**Linux TCP 还具备自动缓冲区调优机制，因此真实项目应该结合测量结果进行调整。**

![](https://relay-1.bijitongbu.site/p/4acb641d3307840bdf2cd6b8e272095c.png)

### 12.5 `TCP_NODELAY` 与 Nagle 算法

Nagle 算法主要用于减少大量小 TCP Segment。

如果存在尚未确认的小数据，新的小写入可能暂时等待合并。

实时交互业务可能希望关闭 Nagle：

```
int on = 1;

setsockopt(fd,
           IPPROTO_TCP,
           TCP_NODELAY,
           &on,
           sizeof(on));
```

典型适用：

-   • 游戏；
    
-   • RPC；
    
-   • 远程终端；
    
-   • 低延迟交互。
    

**但是开启 `TCP_NODELAY` 并不能解决应用自己一字节一字节低效调用 `send()` 的全部问题。**

**合理批量发送仍然非常重要。**

![](https://relay-1.bijitongbu.site/p/4ff3a25400c4b33d4078f6bd3357e12c.png)

### 12.6 `SO_RCVTIMEO/SO_SNDTIMEO` 超时控制

设置接收超时：

```
struct timeval tv;

tv.tv_sec = 5;
tv.tv_usec = 0;

setsockopt(fd,
           SOL_SOCKET,
           SO_RCVTIMEO,
           &tv,
           sizeof(tv));
```

阻塞接收在超时后通常返回错误，并可能表现为：

```
EAGAIN
EWOULDBLOCK
```

发送超时：

```
SO_SNDTIMEO
```

主要影响阻塞型发送操作等待的时间。

需要注意：

> **Socket IO 超时并不等价于完整业务请求超时。**

例如业务规定：

```
一个请求总共最多允许3秒
```

则更加可靠的方式是使用统一 Deadline，而不是每一次 `recv()` 都重新获得 3 秒。

![](https://relay-1.bijitongbu.site/p/ef0d9ac2f8eeb9b6c475543579fca1aa.png)

### 12.7 `shutdown()` 和 `close()` 有什么区别

`close(fd)`：

> **释放应用对整个 Socket 文件描述符的引用。**

`shutdown()` 可以只关闭某个通信方向：

```
shutdown(fd, SHUT_RD);
shutdown(fd, SHUT_WR);
shutdown(fd, SHUT_RDWR);
```

例如：

```
shutdown(fd, SHUT_WR);
```

表示：

> **本地以后不再发送数据，但是仍然允许接收。**

内核通常会在 TCP 发送方向产生 FIN。

这种方式称为：

```
Half-Close
```

例如客户端：

```
上传数据完成
   ↓
shutdown(SHUT_WR)
   ↓
告诉服务端“请求体已经结束”
   ↓
继续recv服务端最终响应
```

![](https://relay-1.bijitongbu.site/p/2713c09e52bfc31dc3d4b83a3f6c592c.png)

## 十三、网络字节序：为什么网络通信还需要转换大小端

### 13.1 大端与小端是什么

假设：

```
uint32_t value = 0x12345678;
```

四个 Byte：

```
12 34 56 78
```

**大端**

低地址保存高位 Byte：

```
低地址
  ↓
12 34 56 78
```

**小端**

低地址保存低位 Byte：

```
低地址
  ↓
78 56 34 12
```

不同 CPU 架构可能采用不同字节序。

![](https://relay-1.bijitongbu.site/p/96e822a5e45da2a483126960a672f064.png)

### 13.2 为什么网络统一采用网络字节序

如果通信双方 CPU 字节序不同：

```
机器A：小端
机器B：大端
```

直接传递多字节整数，就可能得到完全不同的值。

**因此 TCP/IP 协议规定多字节协议字段使用统一：**

```
Network Byte Order
```

**传统网络字节序为大端。**

![](https://relay-1.bijitongbu.site/p/d272c13385ebe141865d665d80102b57.png)

### 13.3 `htons/htonl`

Host To Network Short：

```
uint16_t port = htons(8080);
```

Host To Network Long：

```
uint32_t value = htonl(0x12345678);
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/4dc30381d3007be0e0d2ec9f04964187_MD5.jpg]]

### 13.4 `ntohs/ntohl`

Network To Host Short：

```
uint16_t host_port =
    ntohs(network_port);
```

Network To Host Long：

```
uint32_t host_value =
    ntohl(network_value);
```

这也是为什么 Socket 地址中：

```
addr.sin_port = htons(8080);
```

不能直接：

```
addr.sin_port = 8080;
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/9078d99fbf994498f3999631c63855fa_MD5.jpg]]

### 13.5 IP地址转换：`inet_pton/inet_ntop`

字符串：

```
192.168.1.100
```

转换为网络地址结构：

```
struct in_addr addr;

inet_pton(AF_INET,
          "192.168.1.100",
          &addr);
```

反过来：

```
char str[INET_ADDRSTRLEN];

inet_ntop(AF_INET,
          &addr,
          str,
          sizeof(str));
```

对于 IPv6：

```
INET6_ADDRSTRLEN
```

以及：

```
AF_INET6
```

**现代程序应该优先使用：**

```
inet_pton
inet_ntop
```

**而不是依赖一些只适合 IPv4 或语义容易混淆的老接口。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/e45abd44ffa21233f75e3d7833eaf792_MD5.jpg]]

## 十四、Socket异常处理：真实项目中最容易踩的坑

### 14.1 recv 返回 0、-1、正数分别意味着什么

对于 TCP：

```
ssize_t n = recv(fd,
                 buf,
                 sizeof(buf),
                 0);
```

**`n > 0`**

成功读取：

```
n Byte
```

**`n == 0`**

对端已经进行了正常的发送方向关闭，数据流到达 EOF。

应该进入连接关闭处理。

**`n == -1`**

检查：

```
errno
```

**不能看到 `-1` 就一律关闭连接。**

例如：

```
EINTR
```

可以重新调用。

```
EAGAIN
```

**在非阻塞模式下只是当前没有更多数据。**

![[Inbox/笔记同步助手/微信公众号/2026/08/images/37ad6f40f57c271cf395caebe5f14719_MD5.jpg]]

### 14.2 send 为什么可能发送不完整

例如：

```
ssize_t n = send(fd,
                 buf,
                 10000,
                 0);
```

可能：

```
n = 4096
```

剩余：

```
5904 Byte
```

**需要应用后续继续发送。**

非阻塞服务器通常为每条连接维护：

```
Output Buffer
```

例如：

```
Application Message
       ↓
append send buffer
       ↓
Socket可写
       ↓
send as much as possible
       ↓
如果还有剩余
注册EPOLLOUT
       ↓
下一次继续发送
```

而不是在事件循环里面死等全部数据发送完成。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/8f5bdd418fc5c7cbe6afb6f6ecf3576f_MD5.jpg]]

### 14.3 `SIGPIPE` 为什么会导致程序退出

假设对端已经关闭连接。

本地继续：

```
send(fd,
     buf,
     len,
     0);
```

某些情况下内核会产生：

```
SIGPIPE
```

默认处理动作可能导致进程退出。

**服务器程序必须正确处理。**

常见方式：

```
send(fd,
     buf,
     len,
     MSG_NOSIGNAL);
```

或者进程级忽略：

```
signal(SIGPIPE, SIG_IGN);
```

前者通常粒度更加明确。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/16afaff5c333e51706fb10e7ea57f4b8_MD5.jpg]]

### 14.4 Connection Reset by Peer 是怎么产生的

常见：

```
ECONNRESET
```

**意味着 TCP 收到了连接复位 RST，或者连接被协议栈判定为异常复位。**

可能场景：

-   • 对端进程异常终止；
    
-   • 对端强制关闭；
    
-   • 对端在不合适状态接收到数据；
    
-   • 网络中间设备复位连接；
    
-   • 对端 Socket 存在未处理数据并以异常方式结束。
    

因此：

```
Connection reset by peer
```

不是普通 FIN 关闭。

它代表连接异常终止。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/b54ec6f9d37e3081bacaca986750e903_MD5.jpg]]

### 14.5 Broken Pipe 是怎么产生的

如果本地继续向一个已经无法正常发送的连接写数据，可能：

```
send()
→ EPIPE
```

错误文字常表现：

```
Broken pipe
```

同时还可能产生 SIGPIPE。

因此真实服务器发送失败处理必须同时考虑：

```
返回值
errno
SIGPIPE
```

![[Inbox/笔记同步助手/微信公众号/2026/08/images/bfae09d8ba58ebf087c953027510bdaa_MD5.jpg]]

### 14.6 Connection Refused 又意味着什么

客户端：

```
connect();
```

如果目标主机可达，但是目标 TCP 端口没有监听服务，主机通常可能直接返回：

```
RST
```

客户端得到：

```
ECONNREFUSED
```

这和：

```
连接超时
```

不同。

**Refused**

通常说明：

```
目标主机能回应
但该端口不存在接受连接的服务
```

**Timeout**

可能说明：

-   • 防火墙丢包；
    
-   • 网络不通；
    
-   • 路由问题；
    
-   • 对端无响应。
    

![[Inbox/笔记同步助手/微信公众号/2026/08/images/947a0fc468432756bbc6373fc8933af3_MD5.jpg]]

### 14.7 EINTR、EAGAIN 等常见错误处理

**EINTR**

系统调用被信号中断。

例如：

```
if (n < 0 && errno == EINTR)
    continue;
```

**EAGAIN / EWOULDBLOCK**

非阻塞 IO：

```
现在暂时无法完成
以后再试
```

不是连接异常。

**ECONNRESET**

对端复位连接。

通常需要关闭。

**EPIPE**

连接发送方向已不可用。

**ETIMEDOUT**

操作或者 TCP 连接超时。

**ENOTCONN**

Socket 当前未处于可以执行该操作的连接状态。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/0fbda0481222f5f52ff70f7a56543a9a_MD5.jpg]]

### 14.8 网络断开后如何安全释放资源

事件驱动服务器关闭连接时，不应只：

```
close(fd);
```

而忘记其他资源。

通常需要：

```
从epoll移除
    ↓
停止定时器
    ↓
取消业务请求
    ↓
释放接收Buffer
    ↓
释放发送Buffer
    ↓
删除连接表记录
    ↓
close fd
```

特别要防止：

> **fd 已经被关闭并重新分配给其他连接，但是旧异步任务仍然拿着原来的 fd 数字进行操作。**

工程上通常需要使用：

```
Connection对象
Generation
Reference Count
Unique Connection ID
```

等方式避免生命周期错误。

![[Inbox/笔记同步助手/微信公众号/2026/08/images/f62421faf756554a9c2537309d7415c1_MD5.jpg]]

## 十五、网络调试：Socket程序出问题应该怎么查

### 15.1 `ping`：先判断网络是否可达

```
ping 192.168.1.20
```

可以帮助判断：

-   • IP 基础连通性；
    
-   • RTT；
    
-   • 是否明显丢包。
    

但是：

> **ping 通不代表 TCP 端口一定正常。**

因为 Ping 使用 ICMP，而应用可能使用 TCP。

同样：

> **ping 不通也不一定代表应用 TCP 一定不通。**

某些防火墙会专门禁 ICMP。

### 15.2 `ss/netstat`：查看端口与TCP连接状态

查看监听：

```
ss -lntp
```

查看全部 TCP：

```
ss -antp
```

查看指定端口：

```
ss -antp | grep ':8080'
```

查看状态：

```
ss -ant state established
ss -ant state time-wait
ss -ant state close-wait
```

例如服务端声称：

```
我已经监听8080
```

第一步就应该验证：

```
ss -lntp | grep 8080
```

而不是立即抓包。

### 15.3 `lsof`：到底哪个进程占用了端口

```
lsof -iTCP:8080 -sTCP:LISTEN
```

可以看到：

```
COMMAND
PID
USER
FD
```

也可以：

```
lsof -i :8080
```

快速查找占用。

### 15.4 `nc`：快速测试TCP/UDP服务器

监听 TCP：

```
nc -l 8080
```

连接：

```
nc 127.0.0.1 8080
```

**如果自己的客户端连接 `nc` 成功，而自己的服务器失败，就可以快速缩小问题范围。**

UDP：

```
nc -u 192.168.1.20 9000
```

### 15.5 `curl`：测试应用层协议

HTTP：

```
curl -v http://127.0.0.1:8080/
```

HTTPS：

```
curl -vk https://example.com/
```

`-v` 可以观察：

-   • DNS；
    
-   • 连接；
    
-   • HTTP Header；
    
-   • 重定向；
    
-   • TLS 部分信息。
    

**如果 TCP 已经正常，但 HTTP 返回错误，就应该把排查重点从 Socket 层转到应用协议层。**

### 15.6 `tcpdump`：从数据包定位问题

抓取 8080：

```
tcpdump -i eth0 \
        -nn \
        -s 0 \
        port 8080
```

保存：

```
tcpdump -i eth0 \
        -nn \
        -s 0 \
        -w socket.pcap \
        port 8080
```

只看 TCP SYN：

```
tcpdump -i eth0 \
        -nn \
        'tcp[tcpflags] & tcp-syn != 0'
```

抓包可以回答：

```
SYN到底发出去了吗？
服务端收到SYN了吗？
SYN-ACK返回了吗？
谁先发FIN？
有没有RST？
有没有重传？
```

### 15.7 Wireshark：观察三次握手与数据传输

Wireshark 可以更加直观地分析：

```
SYN
SYN-ACK
ACK
TCP Payload
Window
Retransmission
Duplicate ACK
FIN
RST
```

**Follow TCP Stream 可以把一条连接的应用层字节流重新组合起来。**

但需要注意：

> **抓包工具看到的是抓包点观察到的数据，不一定等于整个网络的绝对事实。**

例如：

-   • 网卡 TSO；
    
-   • GRO；
    
-   • 抓包丢包；
    
-   • 镜像端口丢帧；
    

都可能影响观察结果。

### 15.8 从“程序异常”到“抓包定位”的完整排查流程

![[Inbox/笔记同步助手/微信公众号/2026/08/images/d20d8cc0b20178023d399de42ecf1a8d_MD5.jpg]]

## 十六、实战：从零实现一个Linux TCP服务器

### 16.1 实现最简单的Echo Server

核心：

```
accept
 ↓
recv
 ↓
send原数据
```

也就是：

```
Client：hello
     ↓
Server
     ↓
Client：hello
```

**这是测试 Socket 最经典的程序。**

### 16.2 支持多个客户端连接

最初可以使用线程：

```
void *client_thread(void *arg)
{
    int fd = *(int *)arg;

    for (;;) {
        char buf[1024];

        ssize_t n = recv(fd,
                         buf,
                         sizeof(buf),
                         0);

        if (n <= 0)
            break;

        send(fd,
             buf,
             n,
             MSG_NOSIGNAL);
    }

    close(fd);
    return NULL;
}
```

每次 `accept()` 后创建线程。

**这种方法适合理解并发，但不适合作为极高连接数服务器最终结构。**

### 16.3 改造成非阻塞Socket

监听 Socket：

```
set_nonblocking(listen_fd);
```

客户端 Socket 同样：

```
set_nonblocking(client_fd);
```

然后所有 IO 都必须正确处理：

```
EAGAIN
EWOULDBLOCK
```

例如：

```
for (;;) {

    ssize_t n = recv(fd,
                     buf,
                     sizeof(buf),
                     0);

    if (n > 0) {
        ...
        continue;
    }

    if (n == 0) {
        close_connection(fd);
        break;
    }

    if (errno == EINTR)
        continue;

    if (errno == EAGAIN ||
        errno == EWOULDBLOCK)
        break;

    close_connection(fd);
    break;
}
```

### 16.4 使用epoll管理客户端

事件循环：

```
epoll_wait
   ↓
listen_fd readable？
   ├─是→accept所有可接受连接
   │
   └─client_fd readable？
          ↓
       recv直到EAGAIN
          ↓
       解析消息
```

监听 Socket 在 ET 下同样应该不断：

```
accept();
```

直到：

```
EAGAIN
```

**因为一次通知可能对应多个已经排队的新连接。**

### 16.5 实现消息协议解决粘包问题

定义协议：

```
+------------+-------------+
| Length     | Payload     |
+------------+-------------+
| uint32_t   | N Byte      |
+------------+-------------+
```

**Length 使用网络字节序。**

发送：

```
uint32_t len_net =
    htonl(payload_len);

send_buffer_append(conn,
                   &len_net,
                   sizeof(len_net));

send_buffer_append(conn,
                   payload,
                   payload_len);
```

接收时：

```
Buffer < 4 Byte
→ 等待

Buffer >= 4 Byte
→ 解析Length

Buffer < 4 + Length
→ 等待

Buffer >= 完整Packet
→ 取出并处理
```

必须限制：

```
MAX_PACKET_SIZE
```

防止恶意客户端声明：

```
Length = 4GB
```

**导致服务器分配巨量内存。**

### 16.6 增加连接异常与断线处理

需要处理：

```
recv == 0
ECONNRESET
EPIPE
EPOLLERR
EPOLLHUP
EPOLLRDHUP
```

例如注册：

```
ev.events =
    EPOLLIN |
    EPOLLRDHUP |
    EPOLLET;
```

**收到关闭事件时，仍应根据程序设计把当前能读取的数据处理完整，然后释放连接。**

### 16.7 完成一个可运行的epoll TCP Server

下面给出一个精简但可以体现核心结构的示例：

```
#define _GNU_SOURCE

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <fcntl.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>

#define PORT        8080
#define MAX_EVENTS  128
#define BUF_SIZE    4096

static int set_nonblocking(int fd)
{
    int flags = fcntl(fd,
                      F_GETFL,
                      0);

    if (flags < 0)
        return -1;

    return fcntl(fd,
                 F_SETFL,
                 flags | O_NONBLOCK);
}

static void close_client(int epfd,
                         int fd)
{
    epoll_ctl(epfd,
              EPOLL_CTL_DEL,
              fd,
              NULL);

    close(fd);
}

int main(void)
{
    int listen_fd;
    int epfd;

    struct sockaddr_in addr;
    struct epoll_event ev;
    struct epoll_event events[MAX_EVENTS];

    listen_fd = socket(AF_INET,
                       SOCK_STREAM | SOCK_NONBLOCK,
                       0);

    if (listen_fd < 0) {
        perror("socket");
        return 1;
    }

    int reuse = 1;

    setsockopt(listen_fd,
               SOL_SOCKET,
               SO_REUSEADDR,
               &reuse,
               sizeof(reuse));

    memset(&addr,
           0,
           sizeof(addr));

    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr =
        htonl(INADDR_ANY);

    if (bind(listen_fd,
             (struct sockaddr *)&addr,
             sizeof(addr)) < 0) {
        perror("bind");
        return 1;
    }

    if (listen(listen_fd, 128) < 0) {
        perror("listen");
        return 1;
    }

    epfd = epoll_create1(EPOLL_CLOEXEC);

    if (epfd < 0) {
        perror("epoll_create1");
        return 1;
    }

    memset(&ev,
           0,
           sizeof(ev));

    ev.events =
        EPOLLIN |
        EPOLLET;

    ev.data.fd = listen_fd;

    if (epoll_ctl(epfd,
                  EPOLL_CTL_ADD,
                  listen_fd,
                  &ev) < 0) {
        perror("epoll_ctl listen");
        return 1;
    }

    printf("server listening on %d\n",
           PORT);

    for (;;) {

        int n = epoll_wait(epfd,
                           events,
                           MAX_EVENTS,
                           -1);

        if (n < 0) {
            if (errno == EINTR)
                continue;

            perror("epoll_wait");
            break;
        }

        for (int i = 0; i < n; i++) {

            int fd =
                events[i].data.fd;

            uint32_t e =
                events[i].events;

            if (fd == listen_fd) {

                for (;;) {

                    struct sockaddr_in peer;
                    socklen_t peer_len =
                        sizeof(peer);

                    int client_fd =
                        accept4(
                            listen_fd,
                            (struct sockaddr *)&peer,
                            &peer_len,
                            SOCK_NONBLOCK |
                            SOCK_CLOEXEC);

                    if (client_fd < 0) {

                        if (errno == EINTR)
                            continue;

                        if (errno == EAGAIN ||
                            errno == EWOULDBLOCK)
                            break;

                        perror("accept4");
                        break;
                    }

                    struct epoll_event cev;

                    memset(&cev,
                           0,
                           sizeof(cev));

                    cev.events =
                        EPOLLIN |
                        EPOLLRDHUP |
                        EPOLLET;

                    cev.data.fd =
                        client_fd;

                    if (epoll_ctl(
                            epfd,
                            EPOLL_CTL_ADD,
                            client_fd,
                            &cev) < 0) {

                        perror("epoll_ctl client");
                        close(client_fd);
                    }
                }

                continue;
            }

            if (e &
                (EPOLLERR |
                 EPOLLHUP)) {

                close_client(epfd, fd);
                continue;
            }

            if (e & EPOLLIN) {

                int closed = 0;

                for (;;) {

                    char buf[BUF_SIZE];

                    ssize_t r =
                        recv(fd,
                             buf,
                             sizeof(buf),
                             0);

                    if (r > 0) {

                        size_t sent = 0;

                        while (sent <
                               (size_t)r) {

                            ssize_t s =
                                send(
                                    fd,
                                    buf + sent,
                                    r - sent,
                                    MSG_NOSIGNAL);

                            if (s > 0) {
                                sent += s;
                                continue;
                            }

                            if (s < 0 &&
                                errno == EINTR)
                                continue;

                            if (s < 0 &&
                                (errno == EAGAIN ||
                                 errno ==
                                 EWOULDBLOCK)) {

                                /*
                                 * 真正生产实现这里不能
                                 * 丢弃剩余数据，应放入
                                 * Connection发送Buffer，
                                 * 并注册EPOLLOUT。
                                 */
                                break;
                            }

                            closed = 1;
                            break;
                        }

                        if (closed)
                            break;

                        continue;
                    }

                    if (r == 0) {
                        closed = 1;
                        break;
                    }

                    if (errno == EINTR)
                        continue;

                    if (errno == EAGAIN ||
                        errno == EWOULDBLOCK)
                        break;

                    closed = 1;
                    break;
                }

                if (closed) {
                    close_client(epfd,
                                 fd);
                    continue;
                }
            }

            if (e & EPOLLRDHUP) {
                close_client(epfd,
                             fd);
            }
        }
    }

    close(epfd);
    close(listen_fd);

    return 0;
}
```

需要特别说明：

这个示例主要展示：

```
non-blocking
accept4
epoll
ET
recv直到EAGAIN
异常关闭
```

真正生产级服务器还必须进一步设计：

```
Connection对象
输入Buffer
输出Buffer
EPOLLOUT
协议解析
流控
超时管理
连接限额
业务线程池
内存池
日志
TLS
安全限制
```

因此：

> **会写一个 epoll Echo Server，只是进入高性能网络编程的起点，而不是终点。**

## 十七、从Socket继续往上：HTTP、WebSocket与RPC

### 17.1 HTTP底层为什么仍然离不开Socket

HTTP 是应用层协议。

经典 HTTP/1.1 和 HTTP/2 通常运行在 TCP 之上。

因此底层仍然需要：

```
socket
 ↓
connect
 ↓
TCP
 ↓
发送HTTP数据
```

HTTPS 则是：

```
HTTP
 ↓
TLS
 ↓
TCP
 ↓
Socket
```

**所以框架虽然把 Socket API 隐藏起来了，底层网络通信并没有消失。**

### 17.2 一个HTTP请求在Socket层发生了什么

例如浏览器访问：

```
http://example.com/
```

可以高度简化成为：

```
DNS解析
   ↓
得到Server IP
   ↓
socket()
   ↓
connect(Server:80)
   ↓
TCP三次握手
   ↓
send HTTP Request
   ↓
GET / HTTP/1.1
Host: example.com
...
   ↓
服务器recv
   ↓
HTTP解析
   ↓
服务器send Response
   ↓
客户端recv
```

**HTTP Keep-Alive 允许多个 HTTP 请求复用同一条 TCP 连接。**

这能够减少：

```
重复三次握手
TCP慢启动
连接创建和销毁
```

的成本。

### 17.3 WebSocket长连接建立在什么基础上

**WebSocket 并不是绕过 TCP 的特殊连接。**

典型流程：

```
TCP连接
   ↓
HTTP Upgrade握手
   ↓
协议升级成功
   ↓
双方在同一条TCP连接上
持续传输WebSocket Frame
```

因此 WebSocket 同样面对：

```
TCP字节流
连接断开
Keepalive
缓冲区
事件驱动
```

等问题。

只是 WebSocket 自己定义了：

```
Frame
Opcode
Payload Length
Mask
Ping/Pong
```

**等更高层协议。**

### 17.4 RPC框架如何封装Socket通信

一个 RPC 框架可以把：

```
get_user(123);
```

变成：

```
Method ID
Request ID
Length
Serialized Payload
```

例如：

```
+--------+--------+---------+---------+
|Length  |Req ID  |Method   |Payload  |
+--------+--------+---------+---------+
```

然后：

```
序列化
 ↓
Socket发送
 ↓
Server解析
 ↓
执行函数
 ↓
生成Response
 ↓
Socket返回
```

**因此 RPC 做的事情本质上是：**

> **在 Socket 字节流之上建立更加高级的请求响应语义。**

### 17.5 Redis、Nginx为什么大量使用事件驱动网络模型

这类服务器可能同时维护大量：

```
Keep-Alive连接
空闲连接
长连接
```

如果：

```
一个连接 = 一个线程
```

线程数量会非常庞大。

事件驱动模型则可以：

```
少量事件线程
       ↓
管理大量非阻塞Socket
       ↓
只有真正活跃的连接才消耗处理时间
```

因此：

```
epoll
Reactor
Event Loop
```

**成为高性能网络程序中的重要基础。**

不过实际高性能程序还会继续考虑：

-   • 多 Event Loop；
    
-   • 多核；
    
-   • SO\_REUSEPORT；
    
-   • Worker；
    
-   • 零拷贝；
    
-   • sendfile；
    
-   • io\_uring；
    
-   • TLS Offload；
    
-   • NUMA；
    
-   • 内存管理。
    

### 17.6 从Socket到高性能网络框架的技术演进

可以把学习路线理解成为：

```
阻塞Socket
   ↓
多进程/多线程
   ↓
非阻塞Socket
   ↓
select/poll
   ↓
epoll
   ↓
Reactor
   ↓
Event Loop
   ↓
网络框架
```

框架最终把复杂细节封装成为：

```
on_connect()
on_message()
on_write_complete()
on_close()
```

**但是理解框架之前掌握 Socket，仍然非常重要。**

因为当出现：

```
连接泄漏
半包
发送阻塞
大量TIME_WAIT
CLOSE_WAIT
EPOLLOUT持续触发
CPU空转
Connection Reset
```

**这些问题的时候，最终仍然必须回到底层 Socket 和 TCP 行为分析。**

## 十八、完整的数据流思维

### 18.1 Socket API只是表象，核心是理解数据流

学习 Socket 最容易陷入的误区，就是把下面这些函数当成独立知识点：

```
socket
bind
listen
accept
connect
send
recv
close
```

实际上它们共同描述的是一条通信链路的生命周期。

服务端：

```
socket
 ↓
bind
 ↓
listen
 ↓
accept
 ↓
recv/send
 ↓
close
```

客户端：

```
socket
 ↓
connect
 ↓
send/recv
 ↓
close
```

API 只是表象。

它们背后真正运行的是：

```
Socket对象
TCP状态机
协议缓冲区
网络协议栈
网卡
```

### 18.2 从应用层、内核协议栈到网卡建立整体认知

一次最普通的：

```
send(fd, data, len, 0);
```

背后可以展开成为：

```
用户程序
   ↓
send()
   ↓
系统调用
   ↓
Socket发送缓存
   ↓
TCP字节流
   ↓
TCP分段
   ↓
IP封装
   ↓
路由
   ↓
邻居子系统
   ↓
Qdisc
   ↓
网卡驱动
   ↓
DMA
   ↓
NIC
   ↓
物理网络
```

对端：

```
NIC
 ↓
DMA
 ↓
网卡驱动
 ↓
NAPI
 ↓
IP
 ↓
TCP
 ↓
Socket接收队列
 ↓
recv()
 ↓
应用程序
```

**当这条链真正建立以后，很多问题就不再神秘。**

例如：

```
send成功但对端没收到
```

并不矛盾。

因为 `send()` 只是整条链路中的前面一步。

### 18.3 Linux网络编程核心知识图谱

可以把 Socket 网络编程整理成为下面这张知识图谱：

```
Linux Socket
                         │
        ┌────────────────┼────────────────┐
        │                │                │
      地址              传输             IO模型
        │                │                │
   IP / Port         TCP / UDP       Blocking
 sockaddr                │          Nonblocking
 IPv4/IPv6               │                │
                         │          select/poll/epoll
                         │
              ┌──────────┼──────────┐
              │          │          │
            建连       收发数据     关闭
              │          │          │
          Handshake   Buffer      FIN/RST
          accept      Stream      TIME_WAIT
          connect     Packet      CLOSE_WAIT
                         │
                    应用层协议
                         │
           Header / Length / CRC / ID
                         │
                      并发模型
                         │
            Process / Thread / Reactor
                         │
                      工程调试
                         │
           ss / tcpdump / Wireshark
```

**真正需要掌握的不是某一个函数，而是这些知识之间的关系。**

### 18.4 下一步：从Socket进入Linux高性能网络编程

当最基础的 Socket 模型已经掌握以后，下一步就可以继续研究：

```
epoll内部机制
Reactor与Proactor
io_uring
sendfile
splice
零拷贝
TCP性能调优
SO_REUSEPORT
多核网络服务器
RSS/RPS/RFS
NAPI
XDP
AF_XDP
DPDK
TLS
HTTP/2
HTTP/3
QUIC
```

**但是这些更高级的技术依然没有脱离本文最开始的主线：**

```
应用产生数据
   ↓
交给内核
   ↓
协议栈组织数据
   ↓
网卡发送
   ↓
对端协议栈接收
   ↓
交给对端应用
```

**所以，真正理解 Linux Socket 网络编程，不应该停留在：**

> 我会调用 `socket()`、`bind()`、`listen()` 和 `accept()`。

而应该能够回答：

```
为什么服务端需要两个Socket？
connect到底在等待什么？
send的数据先去了哪里？
为什么一次recv拿不到完整消息？
为什么非阻塞会返回EAGAIN？
epoll到底在解决什么问题？
为什么ET模式必须一直读到EAGAIN？
TCP关闭以后为什么还有TIME_WAIT？
Connection Reset和正常FIN有什么区别？
程序出问题以后应该从应用、Socket还是网络哪一层查？
```

**当这些问题能够被连接成为一套完整的数据流模型以后，就已经从“会写 Socket API”，逐渐进入了“真正理解 Linux 网络程序如何运行”的阶段。**

**而这套思维，也是后续学习 Nginx、Redis、高性能 RPC、Linux TCP 内核、epoll、io\_uring、DPDK 以及现代网络框架最重要的基础。**

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/74095d83_1786457076222?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FWCEgAKosxUmUEdpQuc5RcQ&s=obsidian)