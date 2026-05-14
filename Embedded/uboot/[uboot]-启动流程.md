# uboot启动流程

uboot版本：[v2025.07](https://github.com/u-boot/u-boot/releases/tag/v2025.07)

```c
u-boot:启动详细的代码调用流程
u-boot.lds:(arch/arm/cpu/u-boot.lds)
    |-->_start:(arch/arm/lib/vectors.S)
        |-->reset(arch/arm/cpu/armv8/start.S)    
            |-->save_boot_params(arch/arm/cpu/armv8/start.S)	/* 将引导参数保存到内存中 */
                |-->save_boot_params_ret(arch/arm/cpu/armv8/start.S)
                    |-->apply_core_errata(arch/arm/cpu/armv8/start.S)
                    |-->lowlevel_init(arch/arm/cpu/armv8/lowlevel_init.S)	/* 主要完成设置sp指针指向的地址 */
                    |-->acpi_pp_secondary_jump(arch/arm/cpu/armv8/start.S)	/* 让从核在系统启动初期进入一个等待状态 */
                    |-->_main(arch/arm/lib/crt0_64.S)
                        |-->board_init_f_alloc_reserve(common/init/board_init.c)	/* 为u-boot的gd结构体分配空间 */
                        |-->board_init_f_init_reserve(common/init/board_init.c)    	/* 将gd结构体清零 */
            			/* 初始化DDR、定时器、完成代码拷贝(把u-boot的代码拷贝到内存最后面，但是不做拷贝工作，只计算应该拷贝的位置)等等 */
                        |-->board_init_f(common/board_f.c)
                            |-->initcall_run_f(common/board_f.c)    		/* 初始化序列函数 */
                                |-->setup_mon_len(common/board_f.c)			/* 配置gd->mon_len成员 */
                                |-->fdtdec_setup(lib/fdtdec.c)    			/* 配置设备树存储位置gd->fdt_blob */
            					|-->initf_malloc(common/dlmalloc.c)			/* 初始化malloc相关成员gd->malloc_limit等 */
            					|-->log_init(common/log.c);					/* gd中log相关成员初始化gd->default_log_level等 */
                                |-->init_baud_rate(common/board_f.c)        /* 初始化波特率 */
                                |-->serial_init(drivers/serial/serial.c)    /* 初始化串口通信设置 */
                                |-->console_init_f(common/console.c)        /* 初始化控制台 */
                                |-->dram_init()								/* 获取ddr容量信息 */
                                |-->...
                        |-->relocate_code(arch/arm/lib/relocate.S)    			/* 主要完成镜像拷贝和重定位 */
                        |-->spl_relocate_stack_gd(common\spl\spl.c)            	/* 重新定位堆栈，准备执行 board_init_r */
                        /* 完成board_init_f剩余的一些外设初始化工作 */
                        |-->board_init_r(common/board_r.c)							/* 板级初始化 */
                            |-->initcall_run_r(common/board_r.c)					/* 初始化序列函数 */
                                |-->initr_reloc(common/board_r.c)    				/* 设置 gd->flags,标记重定位完成 */
                                |-->initr_caches(common\board_r.c)					/* 使能MMU和I/Dcache */
                                |-->initr_dm(common\board_r.c)						/* 初始化dm框架 */
                                |-->serial_initialize(drivers/serial/serial-uclass.c)/* 初始化串口 */
                                |-->pci_init(drivers\pci\pci-uclass.c)				/* 早期PCI设备初始化 */
                                |-->initr_mmc(common/board_r.c)                     /* 初始化emmc */
                                |-->initr_env(common\board_r.c)						/* 初始化环境 */
                                |-->pci_init(drivers\pci\pci-uclass.c)				/* PCI设备初始化 */
                                |-->console_init_r(common/console.c)                /* 初始化控制台 */
                                |-->kgdb_init(common/kgdb.c)						/* 初始化kgdb调试 */
                                |-->interrupt_init(arch/arm/lib/interrupts.c)       /* 初始化中断 */
                                |-->board_late_init(board/xxxx.c)					/* 平台late初始化 */
                                |-->initr_net(common/board_r.c)                     /* 初始化网络设备 */
                                |-->...
                                |-->run_main_loop(common/board_r.c)					/* 主循环，处理命令 */
                                    |-->main_loop(common/main.c)
                                        |-->bootdelay_process(common/autoboot.c)    /* 读取环境变量bootdelay和bootcmd的内容 */
                                        |-->autoboot_command(common/autoboot.c)     /* 倒计时按下执行，没有操作执行bootcmd的参数 */
                                            |-->abortboot(common/autoboot.c)
                                                |-->printf("Hit any key to stop autoboot: %2d ", bootdelay);
                                                /* 到这里就是我们看到uboot延时3s启动内核的地方 */
                                        |-->cli_loop(common/cli.c)    /* 倒计时按下space键,执行用户输入命令 */
```

# 参考

[uboot专栏](https://blog.csdn.net/silent123go/category_6516448.html)

[1. Uboot启动流程分析——上](https://doc.embedfire.com/lubancat/build_and_deploy/zh/latest/building_image/boot_image_analyse/boot_image_analyse.html#uboot)

[2. Uboot启动流程分析——下](https://doc.embedfire.com/lubancat/build_and_deploy/zh/latest/building_image/boot_image_analyse/boot_image_analyse_down.html#uboot)

[Rockchip_Developer_Guide_UBoot_Nextdev_CN.pdf](https://download.t-firefly.com/product/Board/RK356X/Document/Developer/Rockchip_Developer_Guide_UBoot_Nextdev_CN.pdf)

[u-boot启动流程分析-史上最全最详细](https://zhuanlan.zhihu.com/p/633773454)

[U-boot启动流程与加载内核过程](https://blog.csdn.net/xi_xix_i/article/details/134938566)

[u-boot启动流程分析(2)_板级(board)部分](http://www.wowotech.net/u-boot/boot_flow_2.html)

[随笔分类-Uboot移植适配](https://www.cnblogs.com/liangliangge/category/1603479.html)