---
author: Psyducking
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzUxNjgxMDE4OQ==&mid=2247509075&idx=1&sn=f83f8898e7008577adfdf62567c4f1dc&chksm=f8f3bf1230678934b4f24d54fc0a8cb2f3a652f8ae6c417998a4d84227c5d57aa4a7bf2c08fe&mpshare=1&scene=1&srcid=1109coBgvI0dTBsYvg1nkrwk&sharer_shareinfo=95b6e08f578e13e59c596efe3e071279&sharer_shareinfo_first=95b6e08f578e13e59c596efe3e071279#rd
saved: 2026-11-09 12:05:50
tags:
  - 笔记同步助手
id: 58507673-c9c4-462d-987e-9f2d2123edf2
---

公众号名称：嵌入式专栏

作者名称：Psyducking

发布时间：2026-11-09 11:45

固件如果没有唯一、可追溯、可复现的版本身份是个麻烦。

## 版本号：先约定规则

版本号至少回答三个问题：功能上属于哪个代次、与哪个源码状态对应、什么时候构建的。约定一套规则：

```
主版本.次版本.修订号[-预发布]+构建号
   2    .  1   .   3       -rc1    20261014.a3f2c1d
```

-   -   **语义化部分**（2.1.3）：主版本升表示不兼容的接口变化（比如 Modbus 寄存器地址重新编排），次版本升表示新增功能（向下兼容），修订号升表示修缺陷。
-   -   **预发布标记**（-rc1）：测试阶段的版本，正式发布不带。
-   -   **构建号**（日期 + commit 短哈希）：同一个源码在不同机器、不同时间构建，哈希相同但日期不同，便于区分"代码没变但工具链变了"的情况。

代码里集中在一个头文件，其他地方只引用不硬编码：

```
// version.h —— 由构建脚本生成，不要手工修改、不要提交
#ifndef VERSION_H
#define VERSION_H

#define FW_VERSION_MAJOR   2
#define FW_VERSION_MINOR   1
#define FW_VERSION_PATCH   3

#define FW_VERSION_STRING  "2.1.3"
#define FW_BUILD_TIME      "2026-10-14 19:30:05"
#define FW_BUILD_HASH      "a3f2c1d"
#define FW_BUILD_DIRTY     "0"          // 1 表示有未提交改动，现场固件必须为 0

#endif
```

`FW_BUILD_DIRTY` 这一项很容易被忽略但很关键：本地有未提交改动时构建出的固件，版本号看起来和提交过的一样，行为却可能不同。现场固件必须是 clean 构建，这一位就是给自动化检查用的判据。

## 自动生成版本信息

手工改 `version.h` 一定会漏。用构建脚本从 git 状态生成：

```
#!/bin/sh
# scripts/gen_version.sh —— 每次构建前执行，生成 version.h
OUT="${1:-build/version.h}"
TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "0.0.0")
HASH=$(git rev-parse --short=7 HEAD 2>/dev/null || echo "unknown")
DIRTY=$(git diff --quiet 2>/dev/null && echo 0 || echo 1)
DATE=$(date '+%Y-%m-%d %H:%M:%S')

cat > "$OUT" <<EOF
#ifndef VERSION_H
#define VERSION_H
#define FW_VERSION_STRING  "$TAG"
#define FW_BUILD_TIME      "$DATE"
#define FW_BUILD_HASH      "$HASH"
#define FW_BUILD_DIRTY     "$DIRTY"
#endif
EOF

echo "generated $OUT: $TAG $HASH dirty=$DIRTY"
```

在 CMake 里挂到构建流程的最前面：

```
# 每次构建都重新生成，版本信息永远跟 git 状态一致
add_custom_target(version_h ALL
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/gen
    COMMAND ${PROJECT_SOURCE_DIR}/scripts/gen_version.sh
            ${CMAKE_BINARY_DIR}/gen/version.h
    BYPRODUCTS ${CMAKE_BINARY_DIR}/gen/version.h
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
)

include_directories(${CMAKE_BINARY_DIR}/gen)
add_dependencies(${PROJECT_NAME}.elf version_h)
```

构建产物里带上版本信息，几种输出方式都用上：

```
// 启动时串口打印一行，日志里也能看到
voidboot_banner(void)
{
    logi("fw %s build %s [%s]%s",
         FW_VERSION_STRING, FW_BUILD_TIME, FW_BUILD_HASH,
         FW_BUILD_DIRTY[0] == '1' ? " (DIRTY)" : "");
}

// CLI version 命令的回复，现场一条命令拿到全部身份信息
staticvoidcmd_version(int argc, char **argv)
{
    cli_printf("version: %s\r\n", FW_VERSION_STRING);
    cli_printf("hash:    %s\r\n", FW_BUILD_HASH);
    cli_printf("time:    %s\r\n", FW_BUILD_TIME);
    cli_printf("dirty:   %s\r\n", FW_BUILD_DIRTY);
}

// 上云报文里带上版本，平台侧直接看到每台设备的固件分布
staticvoidmqtt_report_version(void)
{
    cJSON *root = cJSON_CreateObject();
    cJSON_AddStringToObject(root, "fw", FW_VERSION_STRING);
    cJSON_AddStringToObject(root, "hash", FW_BUILD_HASH);
    mqtt_publish("gw/+/version", root);
}
```

`git describe --tags --abbrev=0` 找的是最近的一个 tag，如果当前提交之后还有改动，`git describe`（不带 `--abbrev=0`）会给出 `2.1.3-5-ga3f2c1d` 这种"2.1.3 之后第 5 个提交"的形式，比只给 tag 更精确。脚本里按团队习惯二选一。

## 分支模型

一个五分支的简化模型覆盖嵌入式产品的日常节奏：

![[Inbox/笔记同步助手/微信公众号/2026/11/images/5d910098e2c30f6c2c0feb3a67741743_MD5.svg]]

三条要守住的规矩：

-   -   **`main` 分支的每个提交都对应一个可发布的版本。** 不放半成品，合并进来就必须打 tag。
-   -   **`release/x.y` 分支只做缺陷修复。** 新功能一律回 `dev`，避免发布分支范围蔓延、测试白做。
-   -   **紧急修复走 `hotfix`，从 `main` 拉、修完合回 `main` 和 `dev`。** 只改 `dev` 不发版，线上问题就永远修不掉。

## 提交规范：让历史能读

提交信息写 `fix bug`、`update` 这类信息，三个月后看 log 等于没写。用 Conventional Commits 格式，人和工具都能读：

```
<类型>(<范围>): <主题>

<正文，说明为什么改>

<脚注，关联的 issue>
```

类型控制在六个以内：`feat`（新功能）、`fix`（修缺陷）、`refactor`（重构）、`docs`、`test`、`chore`（构建、依赖）。范围写模块名：`modbus`、`bootloader`、`meter`。

```
fix(modbus): 修正 CRC16 高低字节序与从站不一致的问题

现场抄表偶发校验失败，原因是字节序按小端组装，
与 RTU 规范要求的大端不一致。统一为 spec() 计算顺序。

Refs: #128
```

好处有两个：`git log --oneline --grep="^fix"` 直接列出所有修过的缺陷；配合工具能自动生成 changelog，发布说明不用手工攒。

```
# 生成两个 tag 之间的 changelog
git log --pretty=format:"- %s (%h)" v2.1.2..v2.1.3 > CHANGELOG.draft
```

## 依赖锁定：submodule

HAL 库、协议栈、RTOS 这类第三方代码，复制进仓库会让历史混乱，直接引用 master 又会让构建不可复现。submodule 是标准答案：记录第三方仓库的**固定 commit**。

```
# 添加 HAL 库为一个 submodule，固定到某个 tag
git submodule add -b v1.8.4 https://github.com/STMicroelectronics/STM32CubeF4.git third_party/STM32CubeF4
git submodule update --init --recursive

# 升级依赖是显式动作，改的是那个 commit 指针
cd third_party/STM32CubeF4
git checkout v1.9.0
cd ..
git add third_party/STM32CubeF4
git commit -m "chore(deps): STM32CubeF4 v1.8.4 -> v1.9.0"
```

要守住的纪律：**依赖升级必须是独立提交，不和功能改动混在一起。** 出了问题按提交回滚时，能干净地把"升级 HAL"和"改业务逻辑"分开定位。

对体积小的库（自己写的协议解析、公共组件），也可以直接 vendor 进仓库的 `libs/` 目录——提交历史里能看到全部改动，改起来比 submodule 方便。判断标准：是否会独立于主项目演进。会，用 submodule；不会，vendor。

## 二进制产物与仓库卫生

固件 `.hex`/`.bin`、编译中间文件、Keil/IAR 的工程自动生成文件，一律不进库：

```
# .gitignore
build/
*.o
*.d
*.axf
*.hex
*.bin
*.map
*.lst

# 工具链生成物
.mdk/
*.uvguix.*
DebugConfig/

# 本地临时
*.swp
*.bak
```

固件产物归档到构建服务器或制品库，命名带上完整版本号：`gw_v2.1.3_20261014_a3f2c1d.hex`。需要回溯历史版本时，用版本号在归档里找，比在 git 历史里翻代码再编译更可靠。体积超过 100 MB 的测试数据、波形文件用 git-lfs 或干脆放网盘，链接写进文档。

## CI 门禁：版本可信的最后一道

代码提交上去不等于能构建。一条最小 CI 流水线把这些检查自动化：

![[Inbox/笔记同步助手/微信公众号/2026/11/images/89bfd7036dc9ac529c4762d802c57b7e_MD5.svg]]

三个检查项各有分工：**dirty 检查**保证版本号可信，**静态分析**把 MISRA 规则里的高危项挡在合并前，**单元测试**验证逻辑（PC 端 host 构建那套做法，单元测试一篇里有完整说明）。编译产物带版本号归档，任何时候都能取到可下载的历史固件。

GitHub Actions / GitLab CI / 自建 Jenkins 都能实现，嵌入式项目通常用自建 runner，因为需要交叉编译工具链镜像。

## 版本兼容：Bootloader 与参数区

版本号最后一个容易翻车的地方是**兼容性协商**。升级到新版本后参数区格式变了、Bootloader 不认识新固件的结构，都会导致升级后设备异常。

参数区的版本迁移在前面的 Flash 模拟 EEPROM 一篇里讲过，核心是参数块带版本号，加载时检测到旧版本就执行迁移函数。Bootloader 与 App 之间也要有约定：

```
// boot_if.h —— App 与 Bootloader 共享的接口约定
#define BOOT_IF_MAGIC      0x5A5A5A5Au
#define BOOT_IF_VERSION    3u          // 接口版本，双方必须一致

typedefstruct
{
    uint32_t magic;
    uint32_t if_version;               // 接口结构版本
    uint32_t app_version;              // App 版本号
    uint32_t app_size;                 // App 镜像大小(字节)
    uint32_t app_crc32;                // App 镜像 CRC
    uint32_t build_hash;               // 源码哈希前 4 字节
} boot_if_t;

// Bootloader 侧的校验：接口版本不认识就不启动，保留旧版本
staticboolboot_check_app(const boot_if_t *if_)
{
    if (if_->magic != BOOT_IF_MAGIC)
        returnfalse;
    if (if_->if_version != BOOT_IF_VERSION)
        returnfalse;                  // 接口不兼容，拒绝启动
    if (crc32_calc(APP_BASE, if_->app_size) != if_->app_crc32)
        returnfalse;                  // 镜像损坏
    returntrue;
}
```

规则很简单：**Bootloader 只认自己知道的接口版本**。App 升级引入新的接口字段时，Bootloader 要么升级到兼容新旧两个版本的实现，要么保持旧版只启动旧接口的 App。哑终端式的"升级失败就保留原版本"，比"升级成功但起不来"安全得多——这也是 OTA 防砖那篇里双分区设计的延伸。

\------------ **END**\------------

![[Inbox/笔记同步助手/微信公众号/2026/11/images/eaffcc2363aaf9c1f21cf5bb03338cb9_MD5.jpg|

![]]

**Zephyr 4.5 在RA单片机板卡适配的新功能**







](https://mp.weixin.qq.com/s?__biz=MzI4MDI4MDE5Ng==&mid=2247535459&idx=1&sn=3a0c44b30b6d7cdb0797d6a27e750519&scene=21#wechat_redirect)

![[Inbox/笔记同步助手/微信公众号/2026/11/images/84bedbeeca4e962ef68f5022468eef82_MD5.jpg|

![]]

**分享一个好用的 ESP32 模拟器！**







](https://mp.weixin.qq.com/s?__biz=MzI4MDI4MDE5Ng==&mid=2247535445&idx=1&sn=4b125ad2e117c8c181e8d6417723bd21&scene=21#wechat_redirect)

![[Inbox/笔记同步助手/微信公众号/2026/11/images/78f88699d06dec9fe31dadcbec4ca427_MD5.jpg|

![]]

**嵌入式UI：MCU+LVGL Lottie动画指南**







](https://mp.weixin.qq.com/s?__biz=MzI4MDI4MDE5Ng==&mid=2247535473&idx=1&sn=14bb630eb407bf11c69f255b7282b257&scene=21#wechat_redirect)

  

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/74f70849_1794197148458?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzUxNjgxMDE4OQ%3D%3D%26mid%3D2247509075%26idx%3D1%26sn%3Df83f8898e7008577adfdf62567c4f1dc%26chksm%3Df8f3bf1230678934b4f24d54fc0a8cb2f3a652f8ae6c417998a4d84227c5d57aa4a7bf2c08fe%26mpshare%3D1%26scene%3D1%26srcid%3D1109coBgvI0dTBsYvg1nkrwk%26sharer_shareinfo%3D95b6e08f578e13e59c596efe3e071279%26sharer_shareinfo_first%3D95b6e08f578e13e59c596efe3e071279%23rd&s=obsidian)