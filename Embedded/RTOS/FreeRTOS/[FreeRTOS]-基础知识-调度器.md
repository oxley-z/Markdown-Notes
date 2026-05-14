# FreeRTOS基础知识-调度器

## 调度器(Scheduler)

## 任务调度

FreeRTOS就是一款支持多任务运行的实时操作系统，具有时间片、抢占式和合作式三种调度方式。

* **合作式调度**，主要用在资源有限的设备上面，现在已经很少使用了。出于这个原因，后面的 FreeRTOS 版本中不会将合作式调度删除掉，但也不会再进行升级了。

* **抢占式调度**，每个任务都有不同的优先级，任务会一直运行直到被高优先级任务抢占或者遇到阻塞式的 API函数，比如 vTaskDelay。

* **时间片调度**，每个任务都有相同的优先级，任务会运行固定的时间片个数或者遇到阻塞式的 API 函数，比如vTaskDelay，才会执行同优先级任务之间的任务切换。

FreeRTOS 默认使用固定优先级的抢占式调度策略，对同等优先级的任务执行时间切片轮询调度，“轮询调度”是指具有相同优先级的任务轮流进入运行状态。

调度器的工作可以分为3个主要部分：

* 选择最高优先级的就绪任务；
* 上下文切换；
* 任务切换后的执行



### [vTaskStartScheduler](https://freertos.org/zh-cn-cmn-s/Documentation/02-Kernel/04-API-references/04-RTOS-kernel-control/03-vTaskStartScheduler)

vTaskStartScheduler() 函数用于启动 FreeRTOS 的任务调度器。一旦调用此函数，调度器将开始运行，并选择最高优先级的就绪任务执行。这是 FreeRTOS 应用程序的一个重要步骤，通常在所有任务和其他 RTOS 资源创建之后调用。

原型如下：

```c
void vTaskStartScheduler( void )
```

vTaskStartScheduler() 函数没有参数，也没有返回值。

调用此函数后，FreeRTOS 将接管系统的主控权，并开始调度任务。通常，在调用 vTaskStartScheduler() 后，程序的执行将不会返回到调用它的点，因为调度器将不断运行，直到发生某些特定的情况（如调用 vTaskEndScheduler() 函数，或者系统进入低功耗模式）。

示例：

```c
void vTaskFunction(void *pvParameters)
{
    for (;;)
    {
        // 任务的主要代码
    }
}
 
void main()
{
    xTaskCreate( vTaskFunction, "vTaskFunction", configMINIMAL_STACK_SIZE, NULL, 0, NULL );
    vTaskStartScheduler();  // 启动调度器
    for(;;);
}
```

函数vTaskStartScheduler()主要做了六件事情：

* **创建空闲任务**，根据是否支持静态内存管理，使用静态方式或者动态方式创建空闲任务。
* 创建定时器服务任务，创建定时器服务任务需要配置启用软件定时器，创建定时器服务任务，同样是根据是否配置支持静态内存管理，使用静态或者动态方式创建定时器服务任务。
* 关闭中断，使用portDISABLE_INTERRUPTS()关闭中断，这种方式只关闭受FreeRTOS管理的中断。关闭中断主要是为了防止SysTick中断在任务调度器开启之前或过程中，产生中断。FreeRTOS会在开始运行第一个任务时，重新打开中断。
* 初始化一些全局变量，并将任务调度器的运行标志设置为已运行。
* 初始化任务运行时间统计功能的时基定时器，任务运行时间统计功能需要一个硬件定时器提供高精度的计数，这个硬件定时器就在这里进行配置，如果配置不启用任务运行时间统计功能的，就无需进行这项硬件定时器的配置。
* 最后就是调用xPortStartScheduler()。

### [vTaskEndScheduler](https://freertos.org/zh-cn-cmn-s/Documentation/02-Kernel/04-API-references/04-RTOS-kernel-control/04-vTaskEndScheduler)

仅用于x86硬件架构中。

停止RTOS内核系统节拍时钟。所有创建的任务自动删除并停止多任务调度。

### [vTaskSuspendAll](https://freertos.org/zh-cn-cmn-s/Documentation/02-Kernel/04-API-references/04-RTOS-kernel-control/05-vTaskSuspendAll)

挂起调度器，会阻止上下文切换，但保留中断使能。当调度器被挂起时，如果一个中断需要请求上下文切换，中断请求将保持等待直到调度器从挂起中恢复。调度器从挂起态恢复需要调用xTaskResumeAll()函数。vTaskSuspendAll()支持嵌套使用，当要恢复调度器运行时，vTaskSuspendAll()函数被调用几次就需要调用xTaskResumeAll()函数几次。

```c
void vTaskSuspendAll( void )
```

 只能在正在执行的任务中调用， 不能在调度器处于初始化状态时（启动调度器之前）调用，不得在调度器挂起时调用其他会引起任务切换的 API，比如 vTaskDelayUntil、vTaskDelay、xQueueSend 等。

### [xTaskResumeAll](https://freertos.org/zh-cn-cmn-s/Documentation/02-Kernel/04-API-references/04-RTOS-kernel-control/06-xTaskResumeAll)

恢复因调用vTaskSuspendAll()函数而挂起的实时内核调度器。xTaskResumeAll()仅恢复调度器，它不会恢复那些被vTaskSuspend()函数挂起的任务。

```c
BaseType_t xTaskResumeAll( void )
```

返回值：

* pdTRUE：表示恢复调度器引起了一次上下文切换。

* pdFALSE：调度器嵌套调用了vTaskSuspendAll()，调度器依然保持挂起状态。

```c
void vTask1( void * pvParameters )
{
    for( ;; )
    {
        /* 任务代码写在这里 */

        /* ... */

        /* 有些时候，某个任务希望可以连续长时间的运行，但这时不能使用taskENTER_CRITICAL ()/taskEXIT_CRITICAL ()的方法，这样会屏蔽掉中断，引起中断丢失，包括系统节拍时钟。可以使用vTaskSuspendAll ()停止RTOS内核调度：*/
        xTaskSuspendAll ();

        /* 执行操作代码放在这里。这样不用进入临界区就可以连续长时间执行了。在这期间，中断仍然会得到响应，RTOS内核系统节拍时钟也会继续保持运作 */

        /* ... */

        /* 操作结束，重新启动RTOS内核 。我们想强制进行一次上下文切换，但是如果恢复调度器的时候已经执行了上下文切换，再执行一次是没有意义的，因此会进行一次判断。*/
        if( !xTaskResumeAll () )
        {
            taskYIELD ();
        }
    }
}
```

## 上下文切换

当需要从一个任务切换到另一个任务时，FreeRTOS会保存当前任务的上下文（寄存器等），并恢复下一个任务的上下文。

### taskYIELD

taskYIELD：用于强制上下文切换的宏。在中断服务程序中的等价版本为 portYIELD_FROM_ISR，这也是个宏，其实现取决于移植层。对于Cortex-M3硬件，这个宏会引起PendSV中断。

taskYIELD() 用于请求切换上下文到另一个任务。但是，如果没有其他任务的优先级高于或等于调用 taskYIELD () 的任务，则 RTOS 调度器将只选择调用 taskYIELD() 的任务，使其再次运行。

## 临界段

临界段代码也叫做临界区，是指那些必须完整运行，不能被打断的代码段，比如有的外设的初始化需要严格的时序，初始化过程中不能被打断。FreeRTOS 在进入临界段代码的时候需要关闭中断，当处理完临界段代码以后再打开中断。FreeRTOS 系统本身就有很多的临界段代码，这些代码都加了临界段代码保护，我们在写自己的用户程序的时候有些地方也需要添加临界段代码保护。

* taskENTER_CRITICAL：用于任务进入临界区的宏，调用taskENTER_CRITICAL会关闭中断，确保在执行期间不会进行任务切换。该临界区是可以嵌套的，即在程序中可以重复地进入临界区，只要后续重复退出相同次数的临界区即可。

* taskEXIT_CRITICAL：用于退出临界区的宏。

```c
#define taskENTER_CRITICAL()		portENTER_CRITICAL()
#define taskEXIT_CRITICAL()			portEXIT_CRITICAL()
```

* taskENTER_CRITICAL_FROM_ISR：用于中断服务程序进入临界区；

* taskENTER_CRITICAL_FROM_ISR：用于中断服务程序退出临界区；

```c
#define taskENTER_CRITICAL_FROM_ISR() portSET_INTERRUPT_MASK_FROM_ISR()
#define taskEXIT_CRITICAL_FROM_ISR( x ) portCLEAR_INTERRUPT_MASK_FROM_ISR( x )
```

<font color=red>注意</font>：中断级别临界段代码保护，是用在中断服务程序中的，而且这个中断的优先级一定要低于configMAX_SYSCALL_INTERRUPT_PRIORITY，因为高于这个优先级的中断服务函数不能调用 FreeRTOS 的 API 函数

参数：

- uxSavedInterruptStatus：taskEXIT_CRITICAL_FROM_ISR() 将 uxSavedInterruptStatus 作为唯一参数。作为 uxSavedInterruptStatus 参数的值 必须是与之匹配的 taskENTER_CRITICAL_FROM_ISR() 调用返回的值。

  taskENTER_CRITICAL_FROM_ISR() 不接受任何参数。

返回值：

taskENTER_CRITICAL_FROM_ISR() 返回调用宏之前的中断掩码状态 。taskENTER_CRITICAL_FROM_ISR() 返回的值 必须作为 uxSavedInterruptStatus 参数用于匹配的 taskEXIT_CRITICAL_FROM_ISR() 调用。

taskEXIT_CRITICAL_FROM_ISR() 不返回任何值。

```c
//定时器 3 中断服务函数
void TIM3_IRQHandler(void)
{
    if(TIM_GetITStatus(TIM3,TIM_IT_Update)==SET) //溢出中断
    {
        status_value=taskENTER_CRITICAL_FROM_ISR(); 
        total_num+=1;
        printf("float_num 的值为: %d\r\n",total_num);
        taskEXIT_CRITICAL_FROM_ISR(status_value);
    }
    TIM_ClearITPendingBit(TIM3,TIM_IT_Update); //清除中断标志位
}
```

<font color=red>注意</font>：

* 确保每次调用 进入临界区的函数必须配对退出临界区的函数，以避免中断状态不正确的情况。
* 若taskENTER_CRITICAL()是用于从非中断中进入临界区，不能在中断服务函数中调用函数taskENTER_CRITICAL()，否则就会通过断言报错。
* 尽量缩短临界区的执行时间，避免长时间禁止任务调度，这样可以减少对系统响应性的影响。
* 中断级进入临界区时不支持嵌套调用，注意保持中断的状态一致性，避免由于多次禁用中断而导致的状态混乱。

# 参考

[FreeRTOS-01-任务相关函数](https://www.cnblogs.com/mrlayfolk/p/15058697.html)

[5. 任务管理](https://doc.embedfire.com/rtos/freertos/i.mx_rt1052/zh/latest/application/task_management.html)

[Freertos学习：05-内核控制](https://www.cnblogs.com/schips/p/rtos-freertos-05.html)

[FreeRTOS-临界区代码保护及调度器挂起与恢复](https://www.cnblogs.com/bokeyuan-dlam/articles/18532001)

