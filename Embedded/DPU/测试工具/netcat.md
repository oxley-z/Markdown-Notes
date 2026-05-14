# netcat发送文字

---

netcat可用于测试两台主机之间TCP建立链接的时间。

安装

```bash
sudo apt install netcat
```



传输文字

server

```bash
 ✘ eddy@eddy  ~/test/testcon  nc -l -p 5412
00
00


00
00
```

client

```bash
eddy@eddy:~/test/testcon$ nc 192.168.159.131 5412
00
00


00
00

```



# 参考

1. [netcat使用方法简介 | Jack Huang's Blog (huangwang.github.io)](https://huangwang.github.io/2019/09/22/netcat使用方法简介/)

2. [Linux 下用 netcat 测量 tcp 连接建立时间 – 肥叉烧 feichashao.com](https://feichashao.com/nc_tcp_time/#more-1966)

3. [黑客工具之Netcat详细使用教程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/397058259)





🎆

🎄

🥌