---
author: LabHub
source: 微信公众号
url: https://mp.weixin.qq.com/s?__biz=MzI5NzQxNzU0Ng==&mid=2247489385&idx=1&sn=cea0003ef634eacc9237f8f4dd0b6d15&chksm=ed1b0d15a76d08206c8bad8107bffc3e2a37948ac29029c4c819b0a973e367b532caaacae9fa&mpshare=1&scene=1&srcid=0717G2kCq2d2VEuXdzJZ7WZM&sharer_shareinfo=28458ffbd1406ebcb648e70f9851e140&sharer_shareinfo_first=28458ffbd1406ebcb648e70f9851e140#rd
saved: 2026-07-17 13:35:20
tags:
  - 笔记同步助手
id: be62abea-2652-4a3c-9036-a0e474a54c97
---

公众号名称：LabHub

作者名称：LabHub

发布时间：2026-07-17 09:21

原文链接：[https://github.com/fungos/cr](https://github.com/fungos/cr)

```
#define CR_HOST
#include "cr.h"
int main(int argc, char *argv[]) {
    cr_plugin ctx;
    cr_plugin_open(ctx, "game.dll");
    while (!cr_plugin_update(ctx)) { /* host loop */ }
    cr_plugin_close(ctx);
    return 0;
}
```

嵌入式开发的调试循环有多折磨人，写过的人都知道：改一行代码 → 编译 → 烧录到 Flash → 重启开发板 → 等启动 → 操作到刚才的界面 → 看效果。这个循环短则 30 秒，长则几分钟。一天下来，等编译烧录的时间可能比写代码的时间还长。

cr 做的事情是把"烧录+重启"替换成"热加载"。你的主程序变成一个轻量 Host ，真正的业务逻辑编译成动态库（.so/.dll/.dylib ）。 Host 监控动态库文件的修改时间戳，检测到更新就自动卸载旧版本、加载新版本——整个过程你的开发板不用重启，外设状态通过 `CR_STATE` 宏自动迁移。

截至 2026 年 7 月， cr 在 GitHub 上有 1.8K Star ， MIT 协议。只有**一个头文件** `cr.h`，零外部依赖，从 2017 年维护至今。

## 三个 API 就够用

cr 的公开 API 只有三个函数，加上一个宏用于标注需要跨重载保持的状态：

```
cr_plugin_open()   — 加载插件
cr_plugin_update() — 每帧调用，自动检测重载
cr_plugin_close()  — 清理
CR_STATE           — 标注需要保存/恢复的全局变量
```

Host 端（你的 main 函数所在的可执行文件）极其精简。它只负责窗口创建、输入轮询、调用 `cr_plugin_update()`。业务逻辑全部放在 Guest （动态库）里。 Guest 暴露一个 `cr_main` 入口函数，接收操作类型参数：

```
CR_EXPORT int cr_main(struct cr_plugin *ctx, enum cr_op operation) {
    switch (operation) {
        case CR_LOAD:   return on_load(ctx);    // 重载后恢复状态
        case CR_UNLOAD: return on_unload(ctx);  // 重载前保存状态
        case CR_CLOSE:  /* 程序退出 */ break;
    }
    return on_update(ctx);  // CR_STEP — 每帧调用
}
```

`CR_LOAD` 和 `CR_UNLOAD` 这两个回调是你保存/恢复状态的时机。举个例子——你的游戏里有个玩家的位置坐标，热重载后如果这个坐标丢了，玩家会瞬移回原点。解决方法：

```
static float CR_STATE player_x = 0.0f;
static float CR_STATE player_y = 0.0f;
```

`CR_STATE` 宏把这些变量放进一个特殊的 `.state` 数据段。 cr 在重载前自动备份这段内存，重载后自动恢复到新实例中。不需要你写序列化代码，不需要保存到文件。

## 崩溃保护：写崩了也不会丢进度

cr 内置了一套崩溃回滚机制。如果你新编译的动态库一加载就崩溃（比如空指针解引用）， cr 会：

1.  捕获信号（`SIGSEGV` / `SIGILL` / `SIGBUS` / `SIGABRT`）
2.  自动回滚到上一个能正常运行的版本
3.  在 `ctx.failure` 中记录崩溃原因

你甚至可以在 `CR_UNLOAD` 里保存一份快照——cr 保证每次重载前都会调用 `CR_UNLOAD`，这样即使新版本崩了，你的状态也安全地保存在旧版本的内存里，回滚后正常恢复。

这种"热重载+崩溃保护"的组合，意味着你可以在开发过程中大胆改代码——写崩了不会丢状态，不会丢调试上下文。相比传统的"烧录→崩了→加日志→再烧录"的循环，效率提升非常明显。

## 在哪些平台上能用

cr 在三个主要平台都经过了测试： Linux （.so ）、 macOS （.dylib ）、 Windows （.dll ）。 Windows 上还有一个额外的贴心功能：自动处理 PDB 文件锁定问题——Visual Studio 调试时会锁住 PDB 文件导致编译器无法覆盖， cr 会在加载 DLL 时自动把 PDB 复制一份重命名，让编译器能继续写入原始文件。

作者 Danny Grein 在 2017 年的博客里写了造这个轮子的原因：当时他做游戏开发，每次改一行 UI 代码就要等 2-3 分钟重新链接整个项目。他先是手动写了个 `dlopen`/`dlsym` 的 PoC ，后来慢慢完善成了现在的 cr 。

## 多插件支持：不止一个动态库

cr 支持同时加载多个插件——一个 Host 可以管理多个 `.so`/`.dll`。比如你可以把渲染逻辑放在一个插件里，物理引擎放在另一个插件里，各自独立热重载。

Windows 下多个插件同时运行没有问题。 Linux 和 macOS 下多个插件共存时，崩溃处理可能会有一些边界情况——这是因为信号处理器是进程级的，多个插件共享同一个信号处理函数时， cr 需要区分"是哪个插件崩了"。

## 局限：什么场景不适合用 cr

**跨平台二进制分发**。 热重载依赖动态库（`.so`/`.dll`），这意味着你的部署目标必须能编译动态库。如果你的产品最终要跑在 MCU 裸机上（无 OS 、无动态链接器）， cr 只能在开发阶段用——最终固件需要静态链接， cr 帮不上忙。

**C++ 重度使用场景**。 cr 的核心是 C 语言级的重载。它对 C 代码工作得很好，但如果你的代码大量使用 C++ 的虚函数表、模板实例化、静态构造函数，跨重载的行为可能会不稳定。 C++ 编译器在链接时的重排可能导致 vtable 位置变化， cr 的状态迁移机制处理不了这种情况。如果你主要写 C++ 且需要稳定的热重载，可以考虑 RuntimeCompiledCPlusPlus (RCCPP) 这个替代方案。

**静态变量的地址稳定性**。 `CR_STATE` 宏能自动迁移全局变量的值，但不能保证变量的地址在重载前后不变。如果你的代码里存在指向静态变量的指针，重载后这些指针会变成悬空指针。 cr 提供了四种安全模式（`CR_SAFEST` / `CR_SAFE` / `CR_UNSAFE` / `CR_DISABLE`）来控制状态验证的严格程度，默认是 `CR_UNSAFE`——只检查大小是否匹配，不检查地址。

cr 的 README 里有一句话总结得很好：对于增量、局部化的 C 代码改动，它几乎不会出问题。一旦改动的范围变大、或者涉及 C++ 特性，就需要更小心。

## 一个文件能做的事

cr 不是第一个热重载方案，但它是门槛最低的一个。不需要配置、不需要构建系统集成、不需要学习新的 DSL 。你的 `main.c` 加 10 行 Host 代码，剩余的代码按正常方式编译成动态库——就完成了。

下次你在嵌入式 Linux 开发板上调试应用的时候，可以考虑加一行 `#include "cr.h"`。改代码后只需要 `make && scp` 把 `.so` 传到板子上，程序自动检测并热加载。从"等 3 分钟"变成"等 3 秒"。

---

内容效果不满意？[点此反馈](https://feedback.notebooksyncer.com/feedback/d2fda6de_1784266518020?u=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzI5NzQxNzU0Ng%3D%3D%26mid%3D2247489385%26idx%3D1%26sn%3Dcea0003ef634eacc9237f8f4dd0b6d15%26chksm%3Ded1b0d15a76d08206c8bad8107bffc3e2a37948ac29029c4c819b0a973e367b532caaacae9fa%26mpshare%3D1%26scene%3D1%26srcid%3D0717G2kCq2d2VEuXdzJZ7WZM%26sharer_shareinfo%3D28458ffbd1406ebcb648e70f9851e140%26sharer_shareinfo_first%3D28458ffbd1406ebcb648e70f9851e140%23rd&s=obsidian)