# FreeRTOS配置选项

## configUSE_PREEMPTION

如果 [configUSE_PREEMPTION](https://freertos.org/Documentation/02-Kernel/03-Supported-devices/02-Customization/#configuse_preemption) 为 0，则表示抢占已关闭， 而且只有当运行状态的任务进入“阻塞”或“挂起”状态， 或运行状态任务调用 taskYIELD()， 或中断服务程序 (ISR) 手动请求上下文切换时，才会发生上下文切换。

## configUSE_TIME_SLICING

如果 [configUSE_TIME_SLICING](https://freertos.org/Documentation/02-Kernel/03-Supported-devices/02-Customization/#configuse_time_slicing) 为 0，则表示时间切片已关闭， 因此调度器不会在每个 tick 中断上在同等优先级的任务之间切换。



