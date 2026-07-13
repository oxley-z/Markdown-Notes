---
author: 比特酱
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=Mzk3NTk5MTI0Mg==&mid=2247485188&idx=1&sn=8fcfc2676f255ccf70507021a03bd18d&chksm=c514272b0b8b066e26ea3ddc8bf0e92a9c41be078e71b081345730d091b011bce705e8ba7d0b&mpshare=1&scene=1&srcid=0712qIOFXFfxzwHTybc0L1pI&sharer_shareinfo=f792c8d2882e3aeb178afbe53da98e2b&sharer_shareinfo_first=f792c8d2882e3aeb178afbe53da98e2b#rd
saved: 2026-07-12 20:04:43
tags:
  - 笔记同步助手
id: 168fd983-d40f-47ab-b046-e3e566ffaf49
---

公众号名称：比特酱

作者名称：比特酱

发布时间：2026-06-24 12:53

## 一、一些概念

U-Boot新版LOG(CONFIG\_LOG=y)、老版打印体系(CONFIG\_LOG=n /CONFIG\_LOGLEVEL):

### 1.老版(无CONFIG\_LOG，靠CONFIG\_LOGLEVEL)

\--无独立日志子系统，只是对 printf/debug/warn/error做简单全局过滤;

\--没有统一日志缓存、后端分发、分类管理;

\--全局单一等级控制，所有驱动、核心共用一套阈值。

### 2.新版(CONFIG\_LOG=y 完整日志框架)

\--独立子系统：统一日志缓冲区、日志分类、多输出后端、运行时动态控制;

\--每条日志携带等级、模块分类 ID，双重过滤;

\--提供完整log\_xxx() 分级打印接口，代码编译后真实生效;

\--支持串口、环形缓存、文件、网络syslog多路径同时输出。

## 二、常见日志配置项说明

### 1.基本日志功能开(总开关)

CONFIG\_LOG:核心开关，启用后才能支持U-Boot的日志系统(如log\_debug、log\_info等函数)。

### 2.日志输出级别

日志级别从低到高(详细程度递增)通常包括：LOGL\_EMERG、LOGL\_ALERT、LOGL\_CRIT、LOGL\_ERR、LOGL\_WARNING、LOGL\_NOTICE、LOGL\_INFO、LOGL\_DEBUG、CONFIG\_LOG\_DEFAULT\_LEVEL:设置默认日志输出级别(例如选择LOG\_LEVEL\_INFO则只输出INFO及更高级别的日志，忽略DEBUG信息);CONFIG\_LOG\_MAX\_LEVEL：设置允许的最高日志级别(限制可输出的最详细日志等级);

### 3.日志输出目标

控制日志输出到哪里(例如串口、网口)。

## 三、源码及配置文件级分析

log.h

```
在include/log.h中定义
#if CONFIG_IS_ENABLED(LOG)  //当CONFIG_LOG=n(新版日志关闭)

/* Emit a log record if the level is less that the maximum */
#define log(_cat, _level, _fmt, _args...) ({ \
int _l = _level; \
if (_LOG_DEBUG != 0 || _l <= _LOG_MAX_LEVEL) \
_log((enum log_category_t)(_cat), \
(enum log_level_t)(_l | _LOG_DEBUG), __FILE__, \
__LINE__, _log_func, \
pr_fmt(_fmt), ##_args); \
})

/* Emit a dump if the level is less that the maximum */
#define log_buffer(_cat, _level, _addr, _data, _width, _count, _linelen)  ({ \
int _l = _level; \
if (_LOG_DEBUG != 0 || _l <= _LOG_MAX_LEVEL) \
_log_buffer((enum log_category_t)(_cat), \
(enum log_level_t)(_l | _LOG_DEBUG), __FILE__, \
__LINE__, _log_func, _addr, _data, \
_width, _count, _linelen); \
})

#else //旧版日志
/* Note: _LOG_DEBUG != 0 avoids a warning with clang */
#define log(_cat, _level, _fmt, _args...) ({ \
int _l = _level; \
if (_LOG_DEBUG != 0 || _l <= LOGL_INFO || \
(_DEBUG && _l == LOGL_DEBUG)) \
printf(_fmt, ##_args); \
})

#define log_buffer(_cat, _level, _addr, _data, _width, _count, _linelen)  ({ \
int _l = _level; \
if (_LOG_DEBUG != 0 || _l <= LOGL_INFO || \
(_DEBUG && _l == LOGL_DEBUG)) \
print_buffer(_addr, _data, _width, _count, _linelen); \
})
#endif

/*
3个入参:
1).LOG_CATEGORY：当前文件/模块所属日志分类ID,是每个驱动文件单独定义的宏（核心用于按模块过滤日志）;
2).LOGL_XXX：日志等级常量;
3).##_fmt：可变参数打印字符串。
*/
#define log_emer(_fmt...)	log(LOG_CATEGORY, LOGL_EMERG, ##_fmt)
#define log_alert(_fmt...)	log(LOG_CATEGORY, LOGL_ALERT, ##_fmt)
#define log_crit(_fmt...)	log(LOG_CATEGORY, LOGL_CRIT, ##_fmt)
#define log_err(_fmt...)	log(LOG_CATEGORY, LOGL_ERR, ##_fmt)
#define log_warning(_fmt...)	log(LOG_CATEGORY, LOGL_WARNING, ##_fmt)
#define log_notice(_fmt...)	log(LOG_CATEGORY, LOGL_NOTICE, ##_fmt)
#define log_info(_fmt...)	log(LOG_CATEGORY, LOGL_INFO, ##_fmt)
#define log_debug(_fmt...)	log(LOG_CATEGORY, LOGL_DEBUG, ##_fmt)
#define log_content(_fmt...)	log(LOG_CATEGORY, LOGL_DEBUG_CONTENT, ##_fmt)
#define log_io(_fmt...)		log(LOG_CATEGORY, LOGL_DEBUG_IO, ##_fmt)
#define log_cont(_fmt...)	log(LOGC_CONT, LOGL_CONT, ##_fmt)

//日志级别在include/log.h中定义
enum log_level_t {
/** @LOGL_EMERG: U-Boot is unstable */
LOGL_EMERG = 0,
/** @LOGL_ALERT: Action must be taken immediately */
LOGL_ALERT,
/** @LOGL_CRIT: Critical conditions */
LOGL_CRIT,
/** @LOGL_ERR: Error that prevents something from working */
LOGL_ERR,
/** @LOGL_WARNING: Warning may prevent optimal operation */
LOGL_WARNING,
/** @LOGL_NOTICE: Normal but significant condition, printf() */
LOGL_NOTICE,
/** @LOGL_INFO: General information message */
LOGL_INFO,
/** @LOGL_DEBUG: Basic debug-level message */
LOGL_DEBUG,
/** @LOGL_DEBUG_CONTENT: Debug message showing full message content */
LOGL_DEBUG_CONTENT,
/** @LOGL_DEBUG_IO: Debug message showing hardware I/O access */
LOGL_DEBUG_IO,

/** @LOGL_COUNT: Total number of valid log levels */
LOGL_COUNT,
/** @LOGL_NONE: Used to indicate that there is no valid log level */
LOGL_NONE,

/** @LOGL_LEVEL_MASK: Mask for valid log levels */
LOGL_LEVEL_MASK = 0xf,
/** @LOGL_FORCE_DEBUG: Mask to force output due to LOG_DEBUG */
LOGL_FORCE_DEBUG = 0x10,

/** @LOGL_FIRST: The first, most-important log level */
LOGL_FIRST = LOGL_EMERG,
/** @LOGL_MAX: The last, least-important log level */
LOGL_MAX = LOGL_DEBUG_IO,
/** @LOGL_CONT: Use same log level as in previous call */
LOGL_CONT = -1,
};
```

.config

```
U-Boot的日志输出由配置项控制，需要在.config文件中开启CONFIG_LONG(启动U-Boot日志总开关)、
CONFIG_LOG_LEVEL设置默认日志级别(控制输出粒度),U-Boot的默认日志级别由配置项CONFIG_LOGLEVEL控制，默认值通常为LOGL_INFO(信息级)或
LOGL_WARNING(警告级),这两个级别都高于LOGL_DEBUG(调试级)。根据日志系统的规则,仅输出级别高于或等于
当前设置级别的日志，因此LOGL_DEBUG日志默认不会被打印。
例如：
#
# Debug commands 包含log命令
CONFIG_CMD_LOG=y

#
# Logging
#
CONFIG_LOG=y
CONFIG_LOG_MAX_LEVEL=7
CONFIG_LOG_DEFAULT_LEVEL=6
CONFIG_LOG_CONSOLE=y
CONFIG_LOGF_FILE=y
CONFIG_LOGF_LINE=y
CONFIG_LOGF_FUNC=y
CONFIG_LOGF_FUNC_PAD=20
# CONFIG_LOG_SYSLOG is not set
# CONFIG_SPL_LOG is not set
# CONFIG_LOG_ERROR_RETURN is not set
```

## 四、测试

### 1.参考QEMU ZYNQ7000 U-BOOT下自定义命令

### 2.流程:

![[Inbox/笔记同步助手/微信公众号/2026/07/images/3a0a426f4f1fb26a3cab0647ddb99ff2_MD5.jpg]]

## 五、U-Boot log命令

### 1.log level: 查看当前全局等级;

### 2.log level 7: 设置全局等级为7;

### 3.log categories: 列出所有日志分类;

4.log drivers: 查看日志输出后端。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/dc098045_1783857881499?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzk3NTk5MTI0Mg%3D%3D%26mid%3D2247485188%26idx%3D1%26sn%3D8fcfc2676f255ccf70507021a03bd18d%26chksm%3Dc514272b0b8b066e26ea3ddc8bf0e92a9c41be078e71b081345730d091b011bce705e8ba7d0b%26mpshare%3D1%26scene%3D1%26srcid%3D0712qIOFXFfxzwHTybc0L1pI%26sharer_shareinfo%3Df792c8d2882e3aeb178afbe53da98e2b%26sharer_shareinfo_first%3Df792c8d2882e3aeb178afbe53da98e2b%23rd&s=obsidian)