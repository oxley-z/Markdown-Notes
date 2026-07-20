---
author: LabHub
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI5NzQxNzU0Ng==&mid=2247489706&idx=1&sn=44ea279b2298b469d186ba1edcafc43b&chksm=edf14444ad1069e6d49b99b81434134627461515b8e95406e31f6895d724834a12ef60257b0c&mpshare=1&scene=1&srcid=0719ZF0JD5KE6d6bQowNgYMJ&sharer_shareinfo=44e3f842788c407b0853b9ae4247fa78&sharer_shareinfo_first=44e3f842788c407b0853b9ae4247fa78#rd
saved: 2026-07-19 12:54:14
tags:
  - 笔记同步助手
id: 3a39cf55-04d1-4b75-8c20-6c15a19d564e
---

公众号名称：LabHub

作者名称：LabHub

发布时间：2026-07-18 21:55

原文链接：[https://github.com/rxi/log.c](https://github.com/rxi/log.c)

给 C 项目加日志，只需要两个文件——log.h\[1\] + log.c\[2\]，加起来约 200 行 C ，零依赖。

rxi/log.c\[3\]， 2k Star 。 6 级日志（ TRACE/DEBUG/INFO/WARN/ERROR/FATAL ），可选彩色终端输出，最多 32 个回调——同时写文件和终端、发网络、接自定义后端，全走同一个 `log_info()` 调用。

```
#define log_trace(...) log_log(LOG_TRACE, __FILE__, __LINE__, __VA_ARGS__)
#define log_debug(...) log_log(LOG_DEBUG, __FILE__, __LINE__, __VA_ARGS__)
#define log_info(...)  log_log(LOG_INFO,  __FILE__, __LINE__, __VA_ARGS__)
#define log_warn(...)  log_log(LOG_WARN,  __FILE__, __LINE__, __VA_ARGS__)
#define log_error(...) log_log(LOG_ERROR, __FILE__, __LINE__, __VA_ARGS__)
#define log_fatal(...) log_log(LOG_FATAL, __FILE__, __LINE__, __VA_ARGS__)
```

6 个宏，每个自动填 `__FILE__` 和 `__LINE__`。用起来就是 `log_info("value: %d", x)`——跟 `printf` 一样的格式。

## 一份日志，多个输出

log.c 默认输出到终端。加一个文件输出只需要一行：

```
FILE *fp = fopen("app.log", "a");
log_add_fp(fp, LOG_TRACE);
```

之后每次 `log_info()` 同时写终端和文件。不需要 `fprintf` 遍代码。

它的回调系统支持最多 32 个输出。从 源码\[4\] 可以看到回调结构：

```
typedef struct {
  log_LogFn fn;     // 回调函数——拿到 log_Event，自己决定怎么输出
  void *udata;      // 用户数据——透传给回调
  int level;        // 这个回调的最小日志级别
} Callback;

static struct {
  Callback callbacks[32];
} L;
```

每个回调有自己的级别阈值。 TRACE 级别的回调收所有日志， ERROR 级别的只收错误和致命。加一个 UDP 网络日志回调：写一个 `udp_callback(log_Event *ev)` 函数里调 `sendto()`，然后 `log_add_callback(udp_callback, &sockfd, LOG_WARN)`——警告以上自动发网络。

## 线程安全：一个锁搞定

多线程环境里 `log_info()` 同时被多个线程调用——log.c 通过可选锁处理：

```
static void lock(void)   { if (L.lock) { L.lock(true, L.udata); } }
static void unlock(void) { if (L.lock) { L.lock(false, L.udata); } }

void log_log(int level, const char *file, int line, const char *fmt, ...) {
    lock();
    // ... 格式化 + 分发到所有回调 ...
    unlock();
}
```

不上锁也没关系——单线程场景不需要额外开销。需要线程安全时，`log_set_lock(my_mutex_lock, &my_mutex)` 设一个锁函数就行。 freertos 环境里传 `xSemaphoreTake` / `xSemaphoreGive` 也能用。

## 嵌入式也能用

log.c 只依赖 `stdio.h`、`stdarg.h`、`stdbool.h`、`time.h`——都是标准库。 ARM Cortex-M 上开 `-nostdlib` 需要自己补 `fprintf` 和 `vfprintf`，或者写自定义回调绕过：`log_add_callback(uart_callback, NULL, LOG_INFO)` 把日志直接打到 UART 。

默认的时间戳依赖 `time()` 和 `localtime()`。嵌入式环境没有 RTC 可以设 `ev->time = NULL` 跳过时间戳，或者用 HAL 的 tick 自己填。

## 局限

**不支持格式化字符串以外的类型**。 不像 spdlog\[5\] 能直接 `log_info("vector: {}", vec)`。就是 `printf` 格式，够用但朴素。

**没有日志轮转**。 文件输出不会自动切分。长时间运行需要外部 `logrotate` 或自己写回调。

**没有异步写入**。 `log_log` 是同步的——格式化 + 写回调全在调用线程上。高频日志场景考虑加一个 ring buffer + 独立写线程。

---

200 行源码，读一遍就知道一个生产级日志库的骨架是什么。log.c\[6\] 在 GitHub ，直接看。

---

参考链接

\[1\] log.h: https://github.com/rxi/log.c/blob/master/src/log.h

\[2\] log.c: https://github.com/rxi/log.c/blob/master/src/log.c

\[3\] rxi/log.c: https://github.com/rxi/log.c

\[4\] 源码: https://github.com/rxi/log.c/blob/master/src/log.c

\[5\] spdlog: https://github.com/gabime/spdlog

\[6\] log.c: https://github.com/rxi/log.c

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/85419bf3_1784436853412?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI5NzQxNzU0Ng%3D%3D%26mid%3D2247489706%26idx%3D1%26sn%3D44ea279b2298b469d186ba1edcafc43b%26chksm%3Dedf14444ad1069e6d49b99b81434134627461515b8e95406e31f6895d724834a12ef60257b0c%26mpshare%3D1%26scene%3D1%26srcid%3D0719ZF0JD5KE6d6bQowNgYMJ%26sharer_shareinfo%3D44e3f842788c407b0853b9ae4247fa78%26sharer_shareinfo_first%3D44e3f842788c407b0853b9ae4247fa78%23rd&s=obsidian)