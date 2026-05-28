---
author: 仲一
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzg5ODUxNDMxMA==&mid=2247503356&idx=1&sn=7f392e934fe79d94e25be8f4eeeead4e&chksm=c10cda4ea658dda9601ef4dda53ba8ebc25bd05bb2df2b507d824f3515705ae17c561928217a&mpshare=1&scene=1&srcid=0528Cp8NGuC5DbrhnBhmwL8t&sharer_shareinfo=074bf7dc18557d64a261a4be6f1c6c0c&sharer_shareinfo_first=074bf7dc18557d64a261a4be6f1c6c0c#rd
saved: 2026-05-28 13:42:53
tags:
  - 笔记同步助手
id: 45b610d9-6d29-4b62-b4ad-7c2450fbc9a9
---

公众号名称：嵌入式与Linux那些事

作者名称：仲一

发布时间：2026-05-28 12:02

原文链接：[https://zhuanlan.zhihu.com/p/2009605273808024956](https://zhuanlan.zhihu.com/p/2009605273808024956)

点击上方**“嵌入式与Linux那些事”**

选择**“置顶/星标公众号”**

福利干货，第一时间送达

## 前言

我相信每一个学习 C 语言的人都写过 `int main(int argc, char *argv[])`，但你是否真的理解这两个参数背后的内存模型？为什么 `argv[argc]` 一定是 NULL？参数是如何从 shell 传递到你的程序中的？本篇文章将从标准规定、内存布局、系统调用等方面，带你更加深入的了解这个熟悉的陌生人。

文章主要针对 Unix-like 系统，Windows 的`argv`处理略有不同。

## 1\. 一个看似多余却引人深思的判断

我们在使用 C 语言编写程序时，总是免不了用下面的方法遍历命令行参数：

```
for(inti=0;i
```

``但是，我们在这里冒昧的使用了`argc`作为了遍历字符串数组`argv`的边界，这不禁引人深思，`argv`的元素打印完了吗？``

`于是，对于并不确定是否存在的下一个数组元素，也就是一个字符串，我进行了如下判断：`

```
if(argv[argc]==NULL)
{
printf("argv[argc]为NULL\n");
}
```

`我编写的完整测试代码如下，可直接复制粘贴进行验证：`

```
#include

intmain(intargc,char*argv[])
{
printf("接收到的参数数量argc = %d\n",argc);

printf("参数列表如下：\n");
for(inti=0;i
```

`` `这段代码都是一些最基本的 C 语言操作，相信大家都可以看懂，这段代码的运行结果如下：` ``

`` `![](https://relay-1.bijitongbu.site/p/f201f7a90f7b022b8f3341130eae2031.png)` ``

  

``` ``运行结果证明，`argv[argc]` 确实等于 `NULL`。`argv`字符串数组的元素个数比`argc`的值多一个，最后一个元素是`NULL`。`` ```

## `` `2. 标准是怎么规定的` ``

``` ``在查阅资料之后会发现，`argv[argc] == NULL` 其实是 **ISO C 标准** 的强制规定。`` ```

`` `在 C99 标准中明确写道：**argv[argc] shall be a null pointer.**` ``

``` ``那么为什么要这样设计呢？要知道这个`NULL`指针被使用的情形少之又少，以至于不少人可能都不知道它的存在。实际上这恰恰体现了 C 语言设计中的一种哲学，就是**双重保障**。`` ```

``` `` `argc` 告诉了我们数组的长度，方便我们用`for`进行循环。`NULL` 结尾则让 `argv` 变成了一个以空指针结尾的指针数组。这意味着即使我们不知道 `argc` 的值，也可以像处理字符串一样处理参数列表，如下： `` ```

```
char**ptr=argv;
while(*ptr!=NULL)
{
printf("%s\n",*ptr);
ptr++;
}
```

``` ``这种设计让 `argv` 和环境变量（`environ`）的数据结构保持了一致性，下一章我们将深入介绍这方面。`` ```

## `` `3. 栈上的布局` ``

``` ``要真正理解 `argc` 和 `argv`，我们必须看透进程的虚拟内存布局。`` ```

`` `当程序被执行时，操作系统会为新进程分配内存。在进程的**栈（Stack）**的高地址部分，布局通常是这样的（栈由高地址向低地址生长）：` ``

`` `![](https://relay-1.bijitongbu.site/p/f09a8e555a3aa28b03cc2319cbfaf4a5.png)` ``

  

`` `从这张内存布局示意图中，我们可以轻松的看到一些特点：` ``

``` ``真正的字符串，比如 `"-a"` 存在于栈的最低部，而 `argv` 数组里存的只是指向这些字符串的指针。`` ```

``` `` `argv` 数组的上方紧接着就是环境变量数组。这也解释了为什么`argv[argc]==NULL`，因为在栈上，它是`argv`数组后的分隔符，紧接`environ`指针数组。 `` ```

`` `此外，在传统的 Unix 实现中，参数字符串通常是连续存储的，但标准并不保证这一点，现代 Linux 内核通常会把它们紧凑排列，这就形成了上面的这种内存布局。` ``

## `` `4. 追溯根源` ``

`` `在了解了栈上的内存布局之后，可能会有不少人产生疑问：我们只是在运行程序时添加了几个命令行参数，程序运行起来后这些参数就跑到了栈上面，这个操作到底是谁完成的呢？` ``

``` ``事实上，这涉及到 Linux 的进程启动流程。当我们执行 `./app -a xlp` 时，发生了以下操作：`` ```

`` `首先，Shell 会读取你的输入，并按照空格将字符串切分成数组。` ``

``` ``然后，Shell 会先 `fork()` 出一个子进程，然后在子进程中进行系统调用`execve()`替换当前进程映像，函数原型如下：`` ```

```
intexecve(constchar*filename,char*constargv[],char*constenvp[])
```

``` ``这一步过后，参数就从**用户空间**被拷贝到了**内核空间**。`fork + execve` 这套组合拳在 Unix 进程模型中是非常常见的操作。`` ```

`` `内核读取可执行文件，建立新进程的虚拟内存映射，并将参数从内核空间拷贝回新进程的**栈顶**。` ``

``` ``最后，程序入口并不是 `main`，而是 `_start`，是由 glibc 提供的。`_start` 调用 `__libc_start_main`，该函数从栈上获取 `argc` 和 `argv` 的位置，最终调用 `main` 函数。`` ```

``` ``此外，这里我觉得有必要讲一下 Shell 对通配符的处理，如果是执行 `./app *.c`，Shell 会先将 `*.c` 展开成 `a.c b.c`这种形式，然后再传给 `argv`，而程序本身并不知道你输入的是通配符。因此，可以理解为通配符是 Shell 层面的操作，根本轮不到 C 程序来处理它。`` ```

## `` `5. 实现一个简单的进程伪装` ``

``` ``既然我们知道了 `argv` 指向的是可读写的栈内存，那我们能不能修改它？`` ```

`` `答案是肯定的。` ``

``` ``在 Linux 下，`ps` 命令或 `top` 命令查看进程名时，读取的往往就是 `argv[0]` 指向的内存区域。如果我们修改了这段内存，就能改变进程在系统中的显示名称。`` ```

`` `完整代码如下：` ``

```
\#include
\#include
\#include

intmain(intargc,char*argv[])
{
printf("原始进程名:%s\n",argv[0]);

strcpy(argv[0],"xxxxxxxxx66666");

printf("进程名已修改\n");

printf("按任意建退出\n");

getchar();

return0;
}
```

``` ``这段代码中我们使用`getchar`让程序阻塞，运行程序后，我们重开一个终端，执行`ps aux`命令，这是按照进程`id`大小排列的，这个程序我们刚刚启动，我们直接翻到最下面：`` ```

`` `![](https://relay-1.bijitongbu.site/p/df85543636f4dd2f05fcf10f5dfd4ec4.png)` ``

  

``` ``从图中可以看到，进程名确实变了。但是这里要注意一个潜在的 bug ，如果进程名过长可能会导致覆盖掉后面的环境变量内容，大家可以对比上面的栈布局示意图理解一下。如果不小心覆盖了 `environ` 区域，会导致程序内调用 `getenv()` 失效甚至崩溃，因为 `environ` 指针被破坏了。`` ```

``` `` `ps` 命令其实是去读取 `/proc/[pid]/cmdline` 文件（对于 Linux 而言是这样的）。而这个文件映射的正是这块栈内存，修改栈内存的内容自然就修改了 `ps` 的查看结果。 `` ```

`` `这个操作在实际中也是有应用实例的：` ``

``` ``**恶意软件**通过修改 `argv[0]` 把自己伪装成 `syslogd` 或 `kworker` 等系统内核线程，从而欺骗管理员。`` ```

``` ``**正规软件**如 Nginx、Redis 等服务软件，会利用这个特性修改进程名，用来显示当前进程的状态，例如 `nginx: worker process`，从而方便运维排查。`` ```

> `` `原文链接：https://zhuanlan.zhihu.com/p/2009605273808024956；版权归原作者所有，如有侵权，请联系作者删除` ``

`` `end` ``

> `` `往期推荐` ``
> 
> `` `[嵌入式Linux必读经典书籍](http://mp.weixin.qq.com/s?__biz=Mzg5ODUxNDMxMA==&mid=2247487714&idx=1&sn=a7a0821b6105a5970fae7d19dcff6ddb&chksm=c0603c0bf717b51d2a9a2784d8fe2f5f83fa1515f74fd24e69bff962ced8f46e8fb9a983e8d7&scene=21#wechat_redirect)` ``
> 
> `` `[嵌入式学习路线推荐](http://mp.weixin.qq.com/s?__biz=Mzg5ODUxNDMxMA==&mid=2247487792&idx=1&sn=ecbf3a3d4846d1e59b77674c1445fc11&chksm=c0603dd9f717b4cf40e055f9ab066766901050f4d4fc75e5a808d013a9177cb308dd3952f6bb&scene=21#wechat_redirect)` ``
> 
> `` `[一位读者逻辑清晰的提问](https://mp.weixin.qq.com/s?__biz=Mzg5ODUxNDMxMA==&mid=2247493471&idx=1&sn=f25e3ea6f6f6a8e143d798e13b0b2e7d&chksm=c063cbb6f71442a0fc74601e51eb73f7f2f2313381f857804734964eb4a19dc90328836a25d3&scene=21#wechat_redirect)` ``
> 
> `` `[机械转行嵌入式成功上岸](https://mp.weixin.qq.com/s?__biz=Mzg5ODUxNDMxMA==&mid=2247491200&idx=1&sn=961829c53ff44b772562a7d65a91c8be&chksm=c0603269f717bb7f9031046c58c55c7e32f548a161b2a67429fe270bf6be8c7ae9a3355f8011&scene=21#wechat_redirect)` ``
> 
> `` `[一位音视频方向读者秋招上岸的经历](https://mp.weixin.qq.com/s?__biz=Mzg5ODUxNDMxMA==&mid=2247491143&idx=1&sn=d6d9c58b601272e62d9ed0a49c17f4fd&chksm=c06032aef717bbb8ec01029246e110c2fea9035f32a1a7ea13148cd532f7f973889ff8f3cefa&scene=21#wechat_redirect)` ``

`` `![|80](https://relay-1.bijitongbu.site/p/3acacfadb5e8362b0447c11584fe25e2.png)` ``

`` `![|100](https://relay-1.bijitongbu.site/p/8de6f8c577ba84c82a2ba2988414bc34.png)` ``

`` `扫码加我微信   ` ``

`` `进技术交流群` ``

`` `![|80](https://relay-1.bijitongbu.site/p/2e77eb5761cad6f536813ad83b401eda.png)   ` ``

`` `![|19](https://relay-1.bijitongbu.site/p/67c892fbd64d1e854e0923c6bfdcbdf0.png)` ``

`` `分享` ``

`` `![|19](https://relay-1.bijitongbu.site/p/8637ee2eaba668ee33123808595e715a.png)` ``

`` `收藏` ``

`` `![|19](https://relay-1.bijitongbu.site/p/396dd8cebaeb6ebcd9889d994811208f.png)` ``

`` `点赞` ``

`` `![|19](https://relay-1.bijitongbu.site/p/d951641e2050b7e8b94c1a96005896b5.png)` ``

`` `在看` ``

---

![cover_image](https://mmbiz.qpic.cn/sz_mmbiz_jpg/lSRNJGrqOz5Giav4GEEa5gMTVAL3icyLc1087SgOycqyXl4dqKRlXJbgHib1d8zJsgicBqZue0Qb7dzibkUVb0lP7EkuA0IKgsjgrLEAIpIdT4nY/0?wx_fmt=jpeg)

仲一 嵌入式与Linux那些事

`` `   Read more   ` ``

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/daa02eca_1779946971260?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzg5ODUxNDMxMA%3D%3D%26mid%3D2247503356%26idx%3D1%26sn%3D7f392e934fe79d94e25be8f4eeeead4e%26chksm%3Dc10cda4ea658dda9601ef4dda53ba8ebc25bd05bb2df2b507d824f3515705ae17c561928217a%26mpshare%3D1%26scene%3D1%26srcid%3D0528Cp8NGuC5DbrhnBhmwL8t%26sharer_shareinfo%3D074bf7dc18557d64a261a4be6f1c6c0c%26sharer_shareinfo_first%3D074bf7dc18557d64a261a4be6f1c6c0c%23rd&s=obsidian)