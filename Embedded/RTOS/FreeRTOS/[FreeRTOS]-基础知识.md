# FreeRTOS基础知识

FreeRTOS 内核是一个实时操作系统，支持各种架构。它的基础非常适合构建嵌入式微控制器应用程序。它提供了以下功能：

- 多任务计划程序。
- 多个内存分配选项（包括创建完全静态分配的系统的功能）。
- 任务间协调基元，包括任务通知、消息队列、多种信号灯类型以及流和消息缓冲区。
- 支持多核微控制器上的对称多处理 (SMP)。

FreeRTOS 内核设计为小型、简单且易于使用。典型的 RTOS 内核二进制映像大小为 4000 到 9000 字节。

## 基础概念

* **任务（Task）**：FreeRTOS 中的任务类似于标准操作系统中的线程。每个任务都有自己的独立执行路径，并且可以被调度器管理。
* **调度器（Scheduler）**：调度器是FreeRTOS系统已经实现的任务管理组件，用于分配 CPU 的使用权，负责管理任务的执行顺序。FreeRTOS 提供了优先级调度和时间片轮转调度两种机制。
* **队列（Queue）**：队列用于任务之间的数据传递。它支持多任务同时读写，并且可以根据优先级进行任务间的通信和同步。
* **信号量（Semaphore）**：信号量用于任务同步和资源管理。二值信号量和计数信号量是最常用的两种类型。
* **互斥量（Mutex）**：互斥量用于保护共享资源，防止多个任务同时访问导致数据不一致的问题。
* **定时器（Timer）**：定时器用于在特定时间间隔触发事件。FreeRTOS 提供软件定时器，用户可以灵活配置。
* **内存管理**：FreeRTOS 提供多种内存管理方案，包括静态分配、动态分配以及堆内存分配策略。

# 参考

[FreeRTOS中文官网](https://www.freertos.org/zh-cn-cmn-s)

[RTOS 基础知识](https://www.freertos.org/zh-cn-cmn-s/Documentation/01-FreeRTOS-quick-start/01-Beginners-guide/01-RTOS-fundamentals)

[FreeRTOS学习笔记＞基础知识](https://blog.csdn.net/qq_56044767/article/details/141532579)

[FreeRTOS 调度（单核、AMP 和 SMP）](https://freertos.org/zh-cn-cmn-s/Documentation/02-Kernel/02-Kernel-features/01-Tasks-and-co-routines/04-Task-scheduling)

[FreeRTOS调度器启动过程分析](https://blog.csdn.net/qq_43460068/article/details/134723487)
