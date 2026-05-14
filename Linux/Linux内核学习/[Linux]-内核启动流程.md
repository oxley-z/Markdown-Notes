# ARM64平台Linux内核启动流程

ARM64  QEMU仿真平台的构建、Linux源码的编译、QEMU的启动，可查看[[QEMU]-搭建arm64_linux_kernel环境](../../Embedded/QEMU/[QEMU]-搭建arm64_linux_kernel环境.md)

内核版本：linux-5.10 [The Linux Kernel Archives](https://www.kernel.org/)

## kernel启动第一阶段

kernel启动的第一阶段主要为汇编代码，用于**初始化CPU、初始化kernel执行环境，配置C语言执行环境配置页表、配置MMU、跳转至C阶段**；

### kernel第一行代码的入口

在 [[QEMU]-搭建arm64_linux_kernel环境](../../Embedded/QEMU/[QEMU]-搭建arm64_linux_kernel环境.md) 完成kernel源码的编译后，会生成以下几个文件，分别是：

* `Image`(arch/arm64/boot)：Image就是我们的kernel image；
* `vmlinux`(./)：这是未压缩的、可链接格式（ELF） 的内核可执行文件。可供gdb调试使用的内核文件，禁止编辑，vmlinux.lds文件是在编译时，由vmlinux.lds.S文件对链接器ld的输出进行排序后生成；vmlinux.lds.S是用来对输出文件中的段进行排序，并定义相关的符号名；
* `System.map`(./)：这是一个文本文件，包含了内核符号（函数名、变量名）与其在运行时内存地址的映射关系列表。它是在链接 `vmlinux` 时由链接器生成的。
* `.config`(./)：这是内核编译的**核心配置文件**。它记录了所有配置选项（`CONFIG_XXX=y/m/n`）的设置，决定了哪些功能被编译进内核 (`y`)、编译为模块 (`m`)、还是排除 (`n`)。它由 `make menuconfig`, `make xconfig`, `make defconfig` 等命令生成或修改。
* `modules.builtin`(./)：这是一个文本文件，列出了所有直接编译进内核映像（`vmlinux`） 的模块（即配置为 `=y` 的模块）的路径。
* `modules.builtin.modinfo`(./)：类似于 `modules.builtin`，但包含了编译进内核的模块的元信息（`modinfo` 可以查询的信息）。

> [[内核]编译Linux内核后生成文件及作用](https://zhuanlan.zhihu.com/p/1911116928740750432)

通过`readelf`工具查看vmlinux文件可得如下内容：

```bash
$ readelf -h ./vmlinux
ELF 头：
  Magic：   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              DYN (共享目标文件)
  系统架构:                          AArch64
  版本:                              0x1
  入口点地址：               0xffff800010000000
  程序头起点：          64 (bytes into file)
  Start of section headers:          450798920 (bytes into file)
  标志：             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         3
  Size of section headers:           64 (bytes)
  Number of section headers:         38
  Section header string table index: 37
```

可以看到程序入口点位于地址`0xffff800010000000`，这个入口地址在代码中的哪个位置可以通过反汇编vmlinux文件查看

```bash
$ aarch64-linux-gnu-objdump -dxh vmlinux > vmlinux.s
```

```assembly
vmlinux：     文件格式 elf64-littleaarch64
vmlinux
体系结构：aarch64， 标志 0x00000150：
HAS_SYMS, DYNAMIC, D_PAGED
起始地址 0xffff800010000000

程序头：
    LOAD off    0x0000000000010000 vaddr 0xffff800010000000 paddr 0xffff800010000000 align 2**16
         filesz 0x0000000001deb200 memsz 0x0000000001e6cc94 flags rwx
    NOTE off    0x00000000015695e8 vaddr 0xffff8000115595e8 paddr 0xffff8000115595e8 align 2**2
         filesz 0x000000000000003c memsz 0x000000000000003c flags r--
   STACK off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**4
         filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
私有标志 = 0：

节：
Idx Name          Size      VMA               LMA               File off  Algn
  0 .head.text    00010000  ffff800010000000  ffff800010000000  00010000  2**16
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .text         00dd21b8  ffff800010010000  ffff800010010000  00020000  2**11
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .got.plt      00000018  ffff800010de21b8  ffff800010de21b8  00df21b8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
...
SYMBOL TABLE:
ffff800010000000 l    d  .head.text     0000000000000000 .head.text
ffff800010010000 l    d  .text  0000000000000000 .text
ffff800010de21b8 l    d  .got.plt       0000000000000000 .got.plt
ffff800010df0000 l    d  .rodata        0000000000000000 .rodata
ffff8000114f4b20 l    d  .pci_fixup     0000000000000000 .pci_fixup

...
Disassembly of section .head.text:

ffff800010000000 <_text>:
ffff800010000000:       91005a4d        add     x13, x18, #0x16
ffff800010000004:       14557fff        b       ffff800011560000 <primary_entry>
        ...
ffff800010000010:       01e80000        .word   0x01e80000
ffff800010000014:       00000000        .word   0x00000000
ffff800010000018:       0000000a        .word   0x0000000a
```

可以看到，Linux的第一条指令为`add     x13, x18, #0x16`，对应符号表的`.head.text`段，`.head.text`在include/linux/init.h文件中被定义__HEAD宏表示：

```c
/* For assembly routines */
#define __HEAD		.section	".head.text","ax"
#define __INIT		.section	".init.text","ax"
#define __FINIT		.previous
```

通过搜索源码可知，内核入口点位于`arch/arm64/kernel/head.S`中

![__HEAD](image/[Linux]-内核启动流程/__HEAD.png#pic_center)

```assembly
[arch/arm64/kernel/head.s]
...
/*
 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * This code is mostly position independent so you call this at
 * __pa(PAGE_OFFSET).
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
	__HEAD
_head:
	/*
	 * DO NOT MODIFY. Image header expected by Linux boot-loaders.
	 */
#ifdef CONFIG_EFI
	/*
	 * This add instruction has no meaningful effect except that
	 * its opcode forms the magic "MZ" signature required by UEFI.
	 */
	add	x13, x18, #0x16
	b	primary_entry
#else
	b	primary_entry			// branch to kernel start, magic
	.long	0				// reserved
#endif
	.quad	0				// Image load offset from start of RAM, little-endian
	le64sym	_kernel_size_le			// Effective size of kernel image, little-endian
	le64sym	_kernel_flags_le		// Informative flags, little-endian
...
```

在__HEAD中第一个执行的指令为`add x13, x18, #0x16`，与反汇编查看的第一条语句一致，之后转到`primary_entry`函数。

查看`head.s`文件的头部注释信息可知内核启动的必要条件：**关闭 MMU、关闭D-cache、x0传递给FDT blob的物理地址**。

<font color=red>注意</font>：数据高速缓存一定要关闭，因为在内核启动过程中取数据时会先访问高速缓存，而可能高速缓存中缓存了以前u-boot的一些数据，这些数据对于内核来说是错误的。 而指令高速缓存可以打开，是因为U-boot和内核代码是不重叠的，不会存在指令高速缓存有冲突。

```c
/*
 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * This code is mostly position independent so you call this at
 * __pa(PAGE_OFFSET).
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
```

### primary_entry函数

启动过程中的汇编阶段，是从arch/arm64/kernel/head.S文件开始，其内部只包含一个主要函数`primary_entry` ，经过层层调用，head.s汇编代码的的调用如下，最后执行`start_kernel`函数，内核启动进入C语言阶段。

```assembly
__HEAD
	primary_entry
		preserve_boot_args
		el2_setup
		set_cpu_boot_mode_flag
		__create_page_tables
		__cpu_setup
		__primary_switch
			__enable_mmu
			__relocate_kernel
			__primary_switched
				__pi_memset
				kaslr_early_init
				start_kernel
```

`primary_entry`函数的整体调用如下：

```assembly
[arch/arm64/kernel/head.s]
	__INIT

	/*
	 * The following callee saved general purpose registers are used on the
	 * primary lowlevel boot path:
	 *
	 *  Register   Scope                      Purpose
	 *  x21        primary_entry() .. start_kernel()        FDT pointer passed at boot in x0
	 *  x23        primary_entry() .. start_kernel()        physical misalignment/KASLR offset
	 *  x28        __create_page_tables()                   callee preserved temp register
	 *  x19/x20    __primary_switch()                       callee preserved temp registers
	 *  x24        __primary_switch() .. relocate_kernel()  current RELR displacement
	 */
SYM_CODE_START(primary_entry)
	bl	preserve_boot_args
	bl	el2_setup			// Drop to EL1, w0=cpu_boot_mode
	adrp	x23, __PHYS_OFFSET
	and	x23, x23, MIN_KIMG_ALIGN - 1	// KASLR offset, defaults to 0
	bl	set_cpu_boot_mode_flag
	bl	__create_page_tables
	/*
	 * The following calls CPU setup code, see arch/arm64/mm/proc.S for
	 * details.
	 * On return, the CPU will be ready for the MMU to be turned on and
	 * the TCR will have been set.
	 */
	bl	__cpu_setup			// initialise processor
	b	__primary_switch
SYM_CODE_END(primary_entry)
```

#### preserve_boot_args

保存从bootloader传递到kernel的x0~x3参数。

```assembly
[arch/arm64/kernel/head.s]
/*
 * Preserve the arguments passed by the bootloader in x0 .. x3
 */
SYM_CODE_START_LOCAL(preserve_boot_args)
	mov	x21, x0				// x21=FDT 将dtb的地址暂存在x21寄存器，释放出x0使用

	adr_l	x0, boot_args			// x0保存boot_args变量的地址
	stp	x21, x1, [x0]			// 将x0、x1的值保存到boot_args[0]、boot_args[1]
	stp	x2, x3, [x0, #16]		// 将x2、x3的值保存到boot_args[2]、boot_args[3]

	dmb	sy				// needed before dc ivac with
						// MMU off

	mov	x1, #0x20			// 4 x 8 bytes
	b	__inval_dcache_area		// tail call
SYM_CODE_END(preserve_boot_args)
```

#### el2_setup

程序执行至此，CPU处于哪一个exception level呢？根据ARM64 boot protocol，CPU要么处于EL2（推荐）或者non-secure EL1。如果在EL1，情形有些类似过去arm处理器的感觉，处于EL2稍微复杂一些，需要对virtualisation extensions进行基本的设定，然后将cpu退回到EL1，此处暂未分析。

```assembly
[arch/arm64/kernel/head.s]
/*
 * If we're fortunate enough to boot at EL2, ensure that the world is
 * sane before dropping to EL1.
 *
 * Returns either BOOT_CPU_MODE_EL1 or BOOT_CPU_MODE_EL2 in w0 if
 * booted in EL1 or EL2 respectively.
 */
SYM_FUNC_START(el2_setup)
	msr	SPsel, #1			// We want to use SP_EL{1,2}
	mrs	x0, CurrentEL
	cmp	x0, #CurrentEL_EL2
	b.eq	1f
	mov_q	x0, (SCTLR_EL1_RES1 | ENDIAN_SET_EL1)
	msr	sctlr_el1, x0
	mov	w0, #BOOT_CPU_MODE_EL1		// This cpu booted in EL1
	isb
	ret

1:	mov_q	x0, (SCTLR_EL2_RES1 | ENDIAN_SET_EL2)
	msr	sctlr_el2, x0

#ifdef CONFIG_ARM64_VHE
	/*
	 * Check for VHE being present. For the rest of the EL2 setup,
	 * x2 being non-zero indicates that we do have VHE, and that the
	 * kernel is intended to run at EL2.
	 */
	mrs	x2, id_aa64mmfr1_el1
	ubfx	x2, x2, #ID_AA64MMFR1_VHE_SHIFT, #4
#else
	mov	x2, xzr
#endif

	/* Hyp configuration. */
	mov_q	x0, HCR_HOST_NVHE_FLAGS
	cbz	x2, set_hcr
	mov_q	x0, HCR_HOST_VHE_FLAGS
set_hcr:
	msr	hcr_el2, x0
	isb

	/*
	 * Allow Non-secure EL1 and EL0 to access physical timer and counter.
	 * This is not necessary for VHE, since the host kernel runs in EL2,
	 * and EL0 accesses are configured in the later stage of boot process.
	 * Note that when HCR_EL2.E2H == 1, CNTHCTL_EL2 has the same bit layout
	 * as CNTKCTL_EL1, and CNTKCTL_EL1 accessing instructions are redefined
	 * to access CNTHCTL_EL2. This allows the kernel designed to run at EL1
	 * to transparently mess with the EL0 bits via CNTKCTL_EL1 access in
	 * EL2.
	 */
	cbnz	x2, 1f
	mrs	x0, cnthctl_el2
	orr	x0, x0, #3			// Enable EL1 physical timers
	msr	cnthctl_el2, x0
1:
	msr	cntvoff_el2, xzr		// Clear virtual offset

#ifdef CONFIG_ARM_GIC_V3
	/* GICv3 system register access */
	mrs	x0, id_aa64pfr0_el1
	ubfx	x0, x0, #ID_AA64PFR0_GIC_SHIFT, #4
	cbz	x0, 3f

	mrs_s	x0, SYS_ICC_SRE_EL2
	orr	x0, x0, #ICC_SRE_EL2_SRE	// Set ICC_SRE_EL2.SRE==1
	orr	x0, x0, #ICC_SRE_EL2_ENABLE	// Set ICC_SRE_EL2.Enable==1
	msr_s	SYS_ICC_SRE_EL2, x0
	isb					// Make sure SRE is now set
	mrs_s	x0, SYS_ICC_SRE_EL2		// Read SRE back,
	tbz	x0, #0, 3f			// and check that it sticks
	msr_s	SYS_ICH_HCR_EL2, xzr		// Reset ICC_HCR_EL2 to defaults

3:
#endif

	/* Populate ID registers. */
	mrs	x0, midr_el1
	mrs	x1, mpidr_el1
	msr	vpidr_el2, x0
	msr	vmpidr_el2, x1

#ifdef CONFIG_COMPAT
	msr	hstr_el2, xzr			// Disable CP15 traps to EL2
#endif

	/* EL2 debug */
	mrs	x1, id_aa64dfr0_el1
	sbfx	x0, x1, #ID_AA64DFR0_PMUVER_SHIFT, #4
	cmp	x0, #1
	b.lt	4f				// Skip if no PMU present
	mrs	x0, pmcr_el0			// Disable debug access traps
	ubfx	x0, x0, #11, #5			// to EL2 and allow access to
4:
	csel	x3, xzr, x0, lt			// all PMU counters from EL1

	/* Statistical profiling */
	ubfx	x0, x1, #ID_AA64DFR0_PMSVER_SHIFT, #4
	cbz	x0, 7f				// Skip if SPE not present
	cbnz	x2, 6f				// VHE?
	mrs_s	x4, SYS_PMBIDR_EL1		// If SPE available at EL2,
	and	x4, x4, #(1 << SYS_PMBIDR_EL1_P_SHIFT)
	cbnz	x4, 5f				// then permit sampling of physical
	mov	x4, #(1 << SYS_PMSCR_EL2_PCT_SHIFT | \
		      1 << SYS_PMSCR_EL2_PA_SHIFT)
	msr_s	SYS_PMSCR_EL2, x4		// addresses and physical counter
5:
	mov	x1, #(MDCR_EL2_E2PB_MASK << MDCR_EL2_E2PB_SHIFT)
	orr	x3, x3, x1			// If we don't have VHE, then
	b	7f				// use EL1&0 translation.
6:						// For VHE, use EL2 translation
	orr	x3, x3, #MDCR_EL2_TPMS		// and disable access from EL1
7:
	msr	mdcr_el2, x3			// Configure debug traps

	/* LORegions */
	mrs	x1, id_aa64mmfr1_el1
	ubfx	x0, x1, #ID_AA64MMFR1_LOR_SHIFT, 4
	cbz	x0, 1f
	msr_s	SYS_LORC_EL1, xzr
1:

	/* Stage-2 translation */
	msr	vttbr_el2, xzr

	cbz	x2, install_el2_stub

	mov	w0, #BOOT_CPU_MODE_EL2		// This CPU booted in EL2
	isb
	ret

SYM_INNER_LABEL(install_el2_stub, SYM_L_LOCAL)
	/*
	 * When VHE is not in use, early init of EL2 and EL1 needs to be
	 * done here.
	 * When VHE _is_ in use, EL1 will not be used in the host and
	 * requires no configuration, and all non-hyp-specific EL2 setup
	 * will be done via the _EL1 system register aliases in __cpu_setup.
	 */
	mov_q	x0, (SCTLR_EL1_RES1 | ENDIAN_SET_EL1)
	msr	sctlr_el1, x0

	/* Coprocessor traps. */
	mov	x0, #0x33ff
	msr	cptr_el2, x0			// Disable copro. traps to EL2

	/* SVE register access */
	mrs	x1, id_aa64pfr0_el1
	ubfx	x1, x1, #ID_AA64PFR0_SVE_SHIFT, #4
	cbz	x1, 7f

	bic	x0, x0, #CPTR_EL2_TZ		// Also disable SVE traps
	msr	cptr_el2, x0			// Disable copro. traps to EL2
	isb
	mov	x1, #ZCR_ELx_LEN_MASK		// SVE: Enable full vector
	msr_s	SYS_ZCR_EL2, x1			// length for EL1.

	/* Hypervisor stub */
7:	adr_l	x0, __hyp_stub_vectors
	msr	vbar_el2, x0

	/* spsr */
	mov	x0, #(PSR_F_BIT | PSR_I_BIT | PSR_A_BIT | PSR_D_BIT |\
		      PSR_MODE_EL1h)
	msr	spsr_el2, x0
	msr	elr_el2, lr
	mov	w0, #BOOT_CPU_MODE_EL2		// This CPU booted in EL2
	eret
SYM_FUNC_END(el2_setup)
```

#### set_cpu_boot_mode_flag

在进入这个函数的时候，有一个前提条件：w20寄存器保存了cpu启动时候的Eexception level ，具体代码如下：

```assembly
[arch/arm64/kernel/head.s]
/*
 * Sets the __boot_cpu_mode flag depending on the CPU boot mode passed
 * in w0. See arch/arm64/include/asm/virt.h for more info.
 */
SYM_FUNC_START_LOCAL(set_cpu_boot_mode_flag)
	adr_l	x1, __boot_cpu_mode
	cmp	w0, #BOOT_CPU_MODE_EL2
	b.ne	1f
	add	x1, x1, #4
1:	str	w0, [x1]			// This CPU has booted in EL1
	dmb	sy
	dc	ivac, x1			// Invalidate potentially stale cache line
	ret
SYM_FUNC_END(set_cpu_boot_mode_flag)
```

由于系统启动之后，需要了解CPU启动时候的Exception level，因此需要一个全局变量 \__boot_cpu_mode来保存启动时的CPU mode，__boot_cpu_mode定义入下：

```assembly
[arch/arm64/kernel/head.s]
/*
 * We need to find out the CPU boot mode long after boot, so we need to
 * store it in a writable variable.
 *
 * This is not in .bss, because we set it sufficiently early that the boot-time
 * zeroing of .bss would clobber it.
 */
SYM_DATA_START(__boot_cpu_mode)
	.long	BOOT_CPU_MODE_EL2
	.long	BOOT_CPU_MODE_EL1
SYM_DATA_END(__boot_cpu_mode)
```

#### __create_page_tables

建立页表初始化的过程；

```assembly
[arch/arm64/kernel/head.s]
/*
 * Setup the initial page tables. We only setup the barest amount which is
 * required to get the kernel running. The following sections are required:
 *   - identity mapping to enable the MMU (low address, TTBR0)
 *   - first few MB of the kernel linear mapping to jump to once the MMU has
 *     been enabled
 */
SYM_FUNC_START_LOCAL(__create_page_tables)
...
SYM_FUNC_END(__create_page_tables)
```

#### __cpu_setup

__cpu_setup函数位于`arch/arm64/mm/proc.s`路径，主要用于初始化CPU，内容包括：

* cache和TLB的处理；
* Memory、attributes lookup table的创建；
* SCTLR_EL1、TCR_EL1的设定；

```assembly
[arch/arm64/mm/proc.s]
/*
 *	__cpu_setup
 *
 *	Initialise the processor for turning the MMU on.
 *
 * Output:
 *	Return in x0 the value of the SCTLR_EL1 register.
 */
	.pushsection ".idmap.text", "awx"
SYM_FUNC_START(__cpu_setup)
	tlbi	vmalle1				// Invalidate local TLB
	dsb	nsh

	mov	x1, #3 << 20
	msr	cpacr_el1, x1			// Enable FP/ASIMD
	mov	x1, #1 << 12			// Reset mdscr_el1 and disable
	msr	mdscr_el1, x1			// access to the DCC from EL0
	isb					// Unmask debug exceptions now,
	enable_dbg				// since this is per-cpu
	reset_pmuserenr_el0 x1			// Disable PMU access from EL0
	reset_amuserenr_el0 x1			// Disable AMU access from EL0

	/*
	 * Memory region attributes
	 */
	mov_q	x5, MAIR_EL1_SET
#ifdef CONFIG_ARM64_MTE
	/*
	 * Update MAIR_EL1, GCR_EL1 and TFSR*_EL1 if MTE is supported
	 * (ID_AA64PFR1_EL1[11:8] > 1).
	 */
	mrs	x10, ID_AA64PFR1_EL1
	ubfx	x10, x10, #ID_AA64PFR1_MTE_SHIFT, #4
	cmp	x10, #ID_AA64PFR1_MTE
	b.lt	1f

	/* Normal Tagged memory type at the corresponding MAIR index */
	mov	x10, #MAIR_ATTR_NORMAL_TAGGED
	bfi	x5, x10, #(8 *  MT_NORMAL_TAGGED), #8

	/* initialize GCR_EL1: all non-zero tags excluded by default */
	mov	x10, #(SYS_GCR_EL1_RRND | SYS_GCR_EL1_EXCL_MASK)
	msr_s	SYS_GCR_EL1, x10

	/* clear any pending tag check faults in TFSR*_EL1 */
	msr_s	SYS_TFSR_EL1, xzr
	msr_s	SYS_TFSRE0_EL1, xzr
1:
#endif
	msr	mair_el1, x5
	/*
	 * Set/prepare TCR and TTBR. We use 512GB (39-bit) address range for
	 * both user and kernel.
	 */
	mov_q	x10, TCR_TxSZ(VA_BITS) | TCR_CACHE_FLAGS | TCR_SMP_FLAGS | \
			TCR_TG_FLAGS | TCR_KASLR_FLAGS | TCR_ASID16 | \
			TCR_TBI0 | TCR_A1 | TCR_KASAN_FLAGS
	tcr_clear_errata_bits x10, x9, x5

#ifdef CONFIG_ARM64_VA_BITS_52
	ldr_l		x9, vabits_actual
	sub		x9, xzr, x9
	add		x9, x9, #64
	tcr_set_t1sz	x10, x9
#else
	ldr_l		x9, idmap_t0sz
#endif
	tcr_set_t0sz	x10, x9

	/*
	 * Set the IPS bits in TCR_EL1.
	 */
	tcr_compute_pa_size x10, #TCR_IPS_SHIFT, x5, x6
#ifdef CONFIG_ARM64_HW_AFDBM
	/*
	 * Enable hardware update of the Access Flags bit.
	 * Hardware dirty bit management is enabled later,
	 * via capabilities.
	 */
	mrs	x9, ID_AA64MMFR1_EL1
	and	x9, x9, #0xf
	cbz	x9, 1f
	orr	x10, x10, #TCR_HA		// hardware Access flag update
1:
#endif	/* CONFIG_ARM64_HW_AFDBM */
	msr	tcr_el1, x10
	/*
	 * Prepare SCTLR
	 */
	mov_q	x0, SCTLR_EL1_SET
	ret					// return to head.S
SYM_FUNC_END(__cpu_setup)
```

> [慢慢欣赏arm64内核启动19 primary_entry之__cpu_setup代码第一部分](https://blog.csdn.net/shipinsky/article/details/141809684)
>
> [__cpu_setup注释](https://blog.csdn.net/haotianmai/article/details/129890593?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-8-129890593-blog-142514018.235^v43^pc_blog_bottom_relevance_base7&spm=1001.2101.3001.4242.5&utm_relevant_index=9)
>
> [ARM64启动汇编和内存初始化(中) --- (二)](https://www.cnblogs.com/jianhua1992/p/16842866.html)

#### __primary_switch

主要工作是为打开MMU做准备。

```assembly
[arch/arm64/kernel/head.s]
SYM_FUNC_START_LOCAL(__primary_switch)
#ifdef CONFIG_RANDOMIZE_BASE
	mov	x19, x0				// preserve new SCTLR_EL1 value
	mrs	x20, sctlr_el1			// preserve old SCTLR_EL1 value
#endif

	adrp	x1, init_pg_dir
	bl	__enable_mmu
#ifdef CONFIG_RELOCATABLE
#ifdef CONFIG_RELR
	mov	x24, #0				// no RELR displacement yet
#endif
	bl	__relocate_kernel
#ifdef CONFIG_RANDOMIZE_BASE
	ldr	x8, =__primary_switched
	adrp	x0, __PHYS_OFFSET
	blr	x8

	/*
	 * If we return here, we have a KASLR displacement in x23 which we need
	 * to take into account by discarding the current kernel mapping and
	 * creating a new one.
	 */
	pre_disable_mmu_workaround
	msr	sctlr_el1, x20			// disable the MMU
	isb
	bl	__create_page_tables		// recreate kernel mapping

	tlbi	vmalle1				// Remove any stale TLB entries
	dsb	nsh

	msr	sctlr_el1, x19			// re-enable the MMU
	isb
	ic	iallu				// flush instructions fetched
	dsb	nsh				// via old mapping
	isb

	bl	__relocate_kernel
#endif
#endif
	ldr	x8, =__primary_switched
	adrp	x0, __PHYS_OFFSET
	br	x8
SYM_FUNC_END(__primary_switch)
```

在函数中通过 \__enable_mmu 函数来开启MMU，并调用__primary_switched函数；

##### __primary_switched

在__primary_switched函数中，进行一些C环境的准备，并在最后，调用执行start_kernel函数，内核的启动进入到C语言环境阶段；

```assembly
[arch/arm64/kernel/head.s]
/*
 * The following fragment of code is executed with the MMU enabled.
 *
 *   x0 = __PHYS_OFFSET
 */
SYM_FUNC_START_LOCAL(__primary_switched)
	adrp	x4, init_thread_union
	add	sp, x4, #THREAD_SIZE
	adr_l	x5, init_task
	msr	sp_el0, x5			// Save thread_info

#ifdef CONFIG_ARM64_PTR_AUTH
	__ptrauth_keys_init_cpu	x5, x6, x7, x8
#endif

	adr_l	x8, vectors			// load VBAR_EL1 with virtual
	msr	vbar_el1, x8			// vector table address
	isb

	stp	xzr, x30, [sp, #-16]!
	mov	x29, sp

#ifdef CONFIG_SHADOW_CALL_STACK
	adr_l	scs_sp, init_shadow_call_stack	// Set shadow call stack
#endif

	str_l	x21, __fdt_pointer, x5		// Save FDT pointer

	ldr_l	x4, kimage_vaddr		// Save the offset between
	sub	x4, x4, x0			// the kernel virtual and
	str_l	x4, kimage_voffset, x5		// physical mappings

	// Clear BSS
	adr_l	x0, __bss_start
	mov	x1, xzr
	adr_l	x2, __bss_stop
	sub	x2, x2, x0
	bl	__pi_memset
	dsb	ishst				// Make zero page visible to PTW

#ifdef CONFIG_KASAN
	bl	kasan_early_init
#endif
#ifdef CONFIG_RANDOMIZE_BASE
	tst	x23, ~(MIN_KIMG_ALIGN - 1)	// already running randomized?
	b.ne	0f
	mov	x0, x21				// pass FDT address in x0
	bl	kaslr_early_init		// parse FDT for KASLR options
	cbz	x0, 0f				// KASLR disabled? just proceed
	orr	x23, x23, x0			// record KASLR offset
	ldp	x29, x30, [sp], #16		// we must enable KASLR, return
	ret					// to __primary_switch()
0:
#endif
	add	sp, sp, #16
	mov	x29, #0
	mov	x30, #0
	b	start_kernel
SYM_FUNC_END(__primary_switched)
```

## kernel启动第二阶段

kernel启动的第二阶段是C语言阶段，从函数`start_kernel`开始，start_kernel()函数是所有Linux平台进入系统内核初始化后的入口函数；主要完成剩余的与硬件平台相关的初始化工作，这些初始化操作，有的是公共的，有的是需要配置才会执行的；内核工作需要的模块的初始化依次被调用，如：内存管理、调度系统、异常处理等；

### start_kernel

start_kernel函数是内核启动的核心参数，启动过程中的初始化工作基本都在此完成。

```c
[init/main.c]
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
	char *command_line;
	char *after_dashes;

	set_task_stack_end_magic(&init_task);
	smp_setup_processor_id();
	debug_objects_early_init();

	cgroup_init_early();

	local_irq_disable();
	early_boot_irqs_disabled = true;

	/*
	 * Interrupts are still disabled. Do necessary setups, then
	 * enable them.
	 */
	boot_cpu_init();
	page_address_init();
	pr_notice("%s", linux_banner);
	early_security_init();
	setup_arch(&command_line);
	setup_boot_config(command_line);
	setup_command_line(command_line);
	setup_nr_cpu_ids();
	setup_per_cpu_areas();
	smp_prepare_boot_cpu();	/* arch-specific boot-cpu hooks */
	boot_cpu_hotplug_init();

	build_all_zonelists(NULL);
	page_alloc_init();

	pr_notice("Kernel command line: %s\n", saved_command_line);
	/* parameters may set static keys */
	jump_label_init();
	parse_early_param();
	after_dashes = parse_args("Booting kernel",
				  static_command_line, __start___param,
				  __stop___param - __start___param,
				  -1, -1, NULL, &unknown_bootoption);
	if (!IS_ERR_OR_NULL(after_dashes))
		parse_args("Setting init args", after_dashes, NULL, 0, -1, -1,
			   NULL, set_init_arg);
	if (extra_init_args)
		parse_args("Setting extra init args", extra_init_args,
			   NULL, 0, -1, -1, NULL, set_init_arg);

	/*
	 * These use large bootmem allocations and must precede
	 * kmem_cache_init()
	 */
	setup_log_buf(0);
	vfs_caches_init_early();
	sort_main_extable();
	trap_init();
	mm_init();

	ftrace_init();

	/* trace_printk can be enabled here */
	early_trace_init();

	/*
	 * Set up the scheduler prior starting any interrupts (such as the
	 * timer interrupt). Full topology setup happens at smp_init()
	 * time - but meanwhile we still have a functioning scheduler.
	 */
	sched_init();
	/*
	 * Disable preemption - early bootup scheduling is extremely
	 * fragile until we cpu_idle() for the first time.
	 */
	preempt_disable();
	if (WARN(!irqs_disabled(),
		 "Interrupts were enabled *very* early, fixing it\n"))
		local_irq_disable();
	radix_tree_init();

	/*
	 * Set up housekeeping before setting up workqueues to allow the unbound
	 * workqueue to take non-housekeeping into account.
	 */
	housekeeping_init();

	/*
	 * Allow workqueue creation and work item queueing/cancelling
	 * early.  Work item execution depends on kthreads and starts after
	 * workqueue_init().
	 */
	workqueue_init_early();

	rcu_init();

	/* Trace events are available after this */
	trace_init();

	if (initcall_debug)
		initcall_debug_enable();

	context_tracking_init();
	/* init some links before init_ISA_irqs() */
	early_irq_init();
	init_IRQ();
	tick_init();
	rcu_init_nohz();
	init_timers();
	hrtimers_init();
	softirq_init();
	timekeeping_init();

	/*
	 * For best initial stack canary entropy, prepare it after:
	 * - setup_arch() for any UEFI RNG entropy and boot cmdline access
	 * - timekeeping_init() for ktime entropy used in rand_initialize()
	 * - rand_initialize() to get any arch-specific entropy like RDRAND
	 * - add_latent_entropy() to get any latent entropy
	 * - adding command line entropy
	 */
	rand_initialize();
	add_latent_entropy();
	add_device_randomness(command_line, strlen(command_line));
	boot_init_stack_canary();

	time_init();
	perf_event_init();
	profile_init();
	call_function_init();
	WARN(!irqs_disabled(), "Interrupts were enabled early\n");

	early_boot_irqs_disabled = false;
	local_irq_enable();

	kmem_cache_init_late();

	/*
	 * HACK ALERT! This is early. We're enabling the console before
	 * we've done PCI setups etc, and console_init() must be aware of
	 * this. But we do want output early, in case something goes wrong.
	 */
	console_init();
	if (panic_later)
		panic("Too many boot %s vars at `%s'", panic_later,
		      panic_param);

	lockdep_init();

	/*
	 * Need to run this when irqs are enabled, because it wants
	 * to self-test [hard/soft]-irqs on/off lock inversion bugs
	 * too:
	 */
	locking_selftest();

	/*
	 * This needs to be called before any devices perform DMA
	 * operations that might use the SWIOTLB bounce buffers. It will
	 * mark the bounce buffers as decrypted so that their usage will
	 * not cause "plain-text" data to be decrypted when accessed.
	 */
	mem_encrypt_init();

#ifdef CONFIG_BLK_DEV_INITRD
	if (initrd_start && !initrd_below_start_ok &&
	    page_to_pfn(virt_to_page((void *)initrd_start)) < min_low_pfn) {
		pr_crit("initrd overwritten (0x%08lx < 0x%08lx) - disabling it.\n",
		    page_to_pfn(virt_to_page((void *)initrd_start)),
		    min_low_pfn);
		initrd_start = 0;
	}
#endif
	setup_per_cpu_pageset();
	numa_policy_init();
	acpi_early_init();
	if (late_time_init)
		late_time_init();
	sched_clock_init();
	calibrate_delay();
	pid_idr_init();
	anon_vma_init();
#ifdef CONFIG_X86
	if (efi_enabled(EFI_RUNTIME_SERVICES))
		efi_enter_virtual_mode();
#endif
	thread_stack_cache_init();
	cred_init();
	fork_init();
	proc_caches_init();
	uts_ns_init();
	buffer_init();
	key_init();
	security_init();
	dbg_late_init();
	vfs_caches_init();
	pagecache_init();
	signals_init();
	seq_file_init();
	proc_root_init();
	nsfs_init();
	cpuset_init();
	cgroup_init();
	taskstats_init_early();
	delayacct_init();

	poking_init();
	check_bugs();

	acpi_subsystem_init();
	arch_post_acpi_subsys_init();
	sfi_init_late();
	kcsan_init();

	/* Do the rest non-__init'ed, we're now alive */
	arch_call_rest_init();

	prevent_tail_call_optimization();
}
```

#### 函数属性及调用约束

* **asmlinkage**：保证函数参数从寄存器传递，符合汇编调用约定；
* **__visible**：防止编译器优化掉符号，使调试和链接器可见；
* **__init**：防止编译器优化掉符号，使调试和链接器可见；
* **no_sanitize_address** ：关闭 AddressSanitizer 检测。

#### 变量

```c
	char *command_line;
	char *after_dashes;
```

* **command_line**：变量存储了从 bootloader（如 GRUB、U-Boot 等）传递给 Linux 内核的启动参数。
* **after_dashes**：是一个字符指针，用于指向内核命令行中 `--` 之后的部分。这个分隔符用于将**内核参数**和**init 进程参数**分开。

#### 初始化内核基本结构

```c
	set_task_stack_end_magic(&init_task);
	smp_setup_processor_id();
	debug_objects_early_init();

	cgroup_init_early();
```

* set_task_stack_end_magic：初始化 init_task 栈边界标记，用于栈溢出检测；
* smp_setup_processor_id：获取当前 CPU 的物理 ID；
* debug_objects_early_init：初始化内核调试对象系统，依赖于配置项CONFIG_DEBUG_OBJECTS；
* cgroup_init_early：cgroup 控制组数据结构初始化， 用于控制 Linux 系统资源。

#### 关闭中断，保证安全初始化

```c
	local_irq_disable();
	early_boot_irqs_disabled = true;

	/*
	 * Interrupts are still disabled. Do necessary setups, then
	 * enable them.
	 */
```

* local_irq_disable禁用本地 CPU 中断，防止在硬件和内核数据结构尚未初始化时被中断打断。

* early_boot_irqs_disabled设置标记，方便后续检查。

#### CPU初始化及早期架构相关配置

```c
	boot_cpu_init();
	page_address_init();
	pr_notice("%s", linux_banner);
	early_security_init();
	setup_arch(&command_line);
	setup_boot_config(command_line);
	setup_command_line(command_line);
	setup_nr_cpu_ids();
	setup_per_cpu_areas();
	smp_prepare_boot_cpu();	/* arch-specific boot-cpu hooks */
	boot_cpu_hotplug_init();
```

* boot_cpu_init：设置当前引导系统的CPU在物理上存在，在逻辑上可以使用，初始化 boot CPU 的特权寄存器和 CPU 状态等；
* page_address_init：初始化高端内存的映射表，设置内存也管理相关数据结构；
* pr_notice：打印 Linux 版本号、编译时间等信息；
* early_security_init：早期安全模块初始化；
* setup_arch：内核架构进行初始化，每个体系都有自己的setup_arch()函数，是由顶层Makefile中的ARCH变量定义的，其中参数command_line是从bootloader中传下来的；
* setup_boot_config：读取启动参数和硬件配置；
* setup_command_line：保存内核命令行参数，以便后面使用；
* setup_nr_cpu_ids：如果只是 SMP(多核 CPU)的话，此函数用于获取CPU 核心数量， CPU 数量保存在变量nr_cpu_ids 中；
* setup_per_cpu_areas：设置SMP体系每个CPU使用的内存空间，同时拷贝初始化段里数据；
* smp_prepare_boot_cpu：为SMP系统里引导CPU进行准备工作；
* boot_cpu_hotplug_init：NUMA 节点和 CPU 热插拔早期初始化。

> [Boot CPU 初始化和早期架构设置](https://blog.csdn.net/qq_38145502/article/details/151014473?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-151014473-blog-116866285.235%5Ev43%5Epc_blog_bottom_relevance_base7&spm=1001.2101.3001.4242.2&utm_relevant_index=3#t6)

#### 内存节点初始化相关配置

```c
	build_all_zonelists(NULL);
	page_alloc_init();
```

* build_all_zonelists：在当前处理的结点和系统中其它节点的内存域之间建立一种等级次序（系统内存页区(zone)链表），之后将根据这种次序来分配内存；
* page_alloc_init：初始化所有内存管理节点列表，以便后面进行内存管理初始化。

> [kernel启动流程-start_kernel的执行_3.build_all_zonelists](https://blog.csdn.net/jasonactions/article/details/113725545)
>
> [深入探讨start_kernel()函数的实现原理和相关技术](https://www.lxlinux.net/14038.html)

#### 解析启动命令行参数

```c
	pr_notice("Kernel command line: %s\n", saved_command_line);
	/* parameters may set static keys */
	jump_label_init();
	parse_early_param();
	after_dashes = parse_args("Booting kernel",
				  static_command_line, __start___param,
				  __stop___param - __start___param,
				  -1, -1, NULL, &unknown_bootoption);
	if (!IS_ERR_OR_NULL(after_dashes))
		parse_args("Setting init args", after_dashes, NULL, 0, -1, -1,
			   NULL, set_init_arg);
	if (extra_init_args)
		parse_args("Setting extra init args", extra_init_args,
			   NULL, 0, -1, -1, NULL, set_init_arg);
```

* jump_label_init：内核中跳转标签（Jump Label）机制的初始化函数，用于优化频繁检查的运行时条件分支的性能特性；
* parse_early_param：解析内核启动参数中早期使用的参数（__setup() 宏注册的参数）；
* parse_args：解析处理剩余的参数，若参数不匹配，可以通过 unknown 函数处理未知参数；
* set_init_arg：设置 init 进程的启动参数；
* unknown_bootoption：记录无法识别的参数。

#### 各类内存缓冲、异常及调试跟踪设施初始化

```c
	/*
	 * These use large bootmem allocations and must precede
	 * kmem_cache_init()
	 */
	setup_log_buf(0);
	vfs_caches_init_early();
	sort_main_extable();
	trap_init();
	mm_init();

	ftrace_init();

	/* trace_printk can be enabled here */
	early_trace_init();
```

* setup_log_buf：初始化内核日志缓冲区；
* vfs_caches_init_early：虚拟文件系统早期初始化，包括包括dcache和inode的hash表初始化工作；
* sort_main_extable：对内核内部的异常表进行堆排序（也就是\__start\_\__ex_table和\__stop___ex_table之中的所有元素进行），以便加速访问；

* trap_init：内核陷阱异常进程初始化；
* mm_init：内存管理初始化；

* ftrace_init：调试和跟踪设施初始化。

* early_trace_init：使能trace_printk，用于调试跟踪；

#### 调度器与内核数据结构初始化

```c
	/*
	 * Set up the scheduler prior starting any interrupts (such as the
	 * timer interrupt). Full topology setup happens at smp_init()
	 * time - but meanwhile we still have a functioning scheduler.
	 */
	sched_init();
	/*
	 * Disable preemption - early bootup scheduling is extremely
	 * fragile until we cpu_idle() for the first time.
	 */
	preempt_disable();
	if (WARN(!irqs_disabled(),
		 "Interrupts were enabled *very* early, fixing it\n"))
		local_irq_disable();
	radix_tree_init();

	/*
	 * Set up housekeeping before setting up workqueues to allow the unbound
	 * workqueue to take non-housekeeping into account.
	 */
	housekeeping_init();

	/*
	 * Allow workqueue creation and work item queueing/cancelling
	 * early.  Work item execution depends on kthreads and starts after
	 * workqueue_init().
	 */
	workqueue_init_early();

	rcu_init();

```

* sched_init：初始化进程调度器数据结构并创建运行队列；
* preempt_disable：调用进程调度，并禁止内核抢占；
* radix_tree_init：内核radix树数据结构初始化；
* housekeeping_init：初始化内核维护结构；
* workqueue_init_early：允许早期工作队列创建和调度；
* rcu_init：初始化RCU（Read Copy Update(读-拷贝修改)），包括 RCU、kvfree RCU 等，RCU主要提供在读取数据机会比较多，但更新比较的少的场合，这样减少读取数据锁的性能低下的问题；

#### 调试追踪初始化

```c
	/* Trace events are available after this */
	trace_init();

	if (initcall_debug)
		initcall_debug_enable();
```

* trace_init：初始化跟踪事件系统，ftrace的作用是帮助开发人员了解Linux内核的运行时行为，以便于进行故障调试或者性能分析，要配置 CONFIG_TRACING ，否则是空函数。
* initcall_debug_enable：用于启用这些初始化调用的详细调试信息输出；

* context_tracking_init：上下文追踪初始化，要配置 CONFIG_CONTEXT_TRACKING_FORCE，否则是空函数；

#### 中断及时钟相关初始化

```c
	/* init some links before init_ISA_irqs() */
	early_irq_init();
	init_IRQ();
	tick_init();
	rcu_init_nohz();
	init_timers();
	hrtimers_init();
	softirq_init();
	timekeeping_init();
	...
	time_init();
```

* early_irq_init：前期中断初始化，主要初始化 struct irq_desc 这个类型的一个数组，另外还进行了架构相关的前期中断初始化；
* init_IRQ：架构相关的中断初始化；
* tick_init：初始化时钟滴答控制器；
* rcu_init_nohz：内核中用于初始化 RCU（Read-Copy-Update）无滴答模式的函数。需要配置CONFIG_NO_HZ；
* init_timers：主要初始化引导CPU的时钟相关的数据结构，注册时钟的回调函数，当时钟到达时可以回调时钟处理函数，最后初始化时钟软件中断处理 TIMER_SOFTIQR；
* hrtimers_init：初始化高精度的定时器，并设置回调函数；
* softirq_init：初始化软件中断，软件中断与硬件中断区别就是中断发生时，软件中断是使用线程来监视中断信号，而硬件中断是使用CPU硬件来监视中断；
* timekeeping_init：初始化系统时钟计时，初始化内核里与时钟计时相关的变量；
* time_init：初始化系统时钟。

#### 内核随机数和防护机制初始化

```c
	/*
	 * For best initial stack canary entropy, prepare it after:
	 * - setup_arch() for any UEFI RNG entropy and boot cmdline access
	 * - timekeeping_init() for ktime entropy used in rand_initialize()
	 * - rand_initialize() to get any arch-specific entropy like RDRAND
	 * - add_latent_entropy() to get any latent entropy
	 * - adding command line entropy
	 */
	rand_initialize();
	add_latent_entropy();
	add_device_randomness(command_line, strlen(command_line));
	boot_init_stack_canary();
```

* rand_initialize：内核中随机数生成器子系统的初始化；
* add_latent_entropy：内核初始化添加潜在熵，潜在熵（Latent Entropy）是一种编译时和启动时的熵收集技术，它利用内核代码布局、内存地址随机化等编译时和链接时的不确定性作为熵源；
* add_device_randomness：用于收集设备相关的数据作为熵源，这些数据虽然不是完全随机的，但包含一定的不可预测性，可以增强随机数池的熵；
* boot_init_stack_canary：初始化堆栈保护的canary值，用来防止栈溢出。

#### 性能与调试设施

```c
	perf_event_init();
	profile_init();
	call_function_init();
```

* perf_event_init()：性能计数器初始化；
* profile_init()：系统调用和性能分析初始化；
* call_function_init()：异步函数调用初始化；

#### 启用中断

```c
	WARN(!irqs_disabled(), "Interrupts were enabled early\n");

	early_boot_irqs_disabled = false;
	local_irq_enable();
```

* local_irq_enable：启用本地 CPU 中断，允许内核正常调度和中断处理。

#### 内存管理与控制组初始化

```c
	kmem_cache_init_late();

	/*
	 * HACK ALERT! This is early. We're enabling the console before
	 * we've done PCI setups etc, and console_init() must be aware of
	 * this. But we do want output early, in case something goes wrong.
	 */
	console_init();
	if (panic_later)
		panic("Too many boot %s vars at `%s'", panic_later,
		      panic_param);

	lockdep_init();

	/*
	 * Need to run this when irqs are enabled, because it wants
	 * to self-test [hard/soft]-irqs on/off lock inversion bugs
	 * too:
	 */
	locking_selftest();

	/*
	 * This needs to be called before any devices perform DMA
	 * operations that might use the SWIOTLB bounce buffers. It will
	 * mark the bounce buffers as decrypted so that their usage will
	 * not cause "plain-text" data to be decrypted when accessed.
	 */
	mem_encrypt_init();

#ifdef CONFIG_BLK_DEV_INITRD
	if (initrd_start && !initrd_below_start_ok &&
	    page_to_pfn(virt_to_page((void *)initrd_start)) < min_low_pfn) {
		pr_crit("initrd overwritten (0x%08lx < 0x%08lx) - disabling it.\n",
		    page_to_pfn(virt_to_page((void *)initrd_start)),
		    min_low_pfn);
		initrd_start = 0;
	}
#endif
	setup_per_cpu_pageset();
	numa_policy_init();
```

* kmem_cache_init_late：slab内存分配器初始化；
* console_init：console_init()函数执行控制台的初始化操作；在console_init()函数执行之前的printk打印信息，需要在console_init()函数执行之后才能打印出来；因为在console_inie()函数之前，printk的打印信息都保存在一个缓存区中，等到console_init()函数执行之后，控制台被初始化完成，就可以将缓冲区中的内容打印出来；
* lockdep_init：初始化锁的状态跟踪模块。由于内核大量使用锁来进行多进程、多处理器的同步操作，那么死锁就会在代码不合理时出现，这时要知道那个锁造成的，真是比较困难的。遇到这种情况，就需要想办法知道那个锁造成的，因此就需要跟踪锁的使用状态，以便发现出错时，把锁的状态打印出来；
* locking_selftest：测试锁的API是否使用正常，进行自我测试。比如测试自旋锁、读写锁、一般信号量和读写信号量；
* mem_encrypt_init：内核中内存加密子系统初始化，在支持内存加密的平台上，初始化内存加密相关的设置，确保内核在运行时可以使用内存加密功能来保护数据。
* setup_per_cpu_pageset：创建每个CPU的高速缓存集合数组；
* numa_policy_init：numa 策略初始化，numa是NonUniform Memory AccessAchitecture的缩写，主要用来提高多个CPU访问内存的速度。

#### ACPI前期及时钟后期初始化

* acpi_early_init：ACPI前期初始化；
* late_time_init：时钟相关后期的初始化；
* sched_clock_init：内核中调度器时钟系统的初始化；
* calibrate_delay：用于校准 BogoMIPS 值和计算 `loops_per_jiffy`。这个函数在内核启动过程中确定处理器的速度，为后续的延迟函数提供基准；

#### 控制组、命名空间、进程等初始化

```c
	pid_idr_init();
	anon_vma_init();
#ifdef CONFIG_X86
	if (efi_enabled(EFI_RUNTIME_SERVICES))
		efi_enter_virtual_mode();
#endif
	thread_stack_cache_init();
	cred_init();
	fork_init();
	proc_caches_init();
	uts_ns_init();
	buffer_init();
	key_init();
	security_init();
	dbg_late_init();
	vfs_caches_init();
	pagecache_init();
	signals_init();
	seq_file_init();
	proc_root_init();
	nsfs_init();
	cpuset_init();
	cgroup_init();
	taskstats_init_early();
	delayacct_init();

	poking_init();
	check_bugs();
```

* pid_idr_init：内核中进程 ID（PID）管理子系统的初始化函数。负责初始化用于高效管理进程 ID 的 IDR（Integer ID Radix Tree）机制；
* anon_vma_init：匿名虚拟内存区域（Anonymous Virtual Memory Area）管理机制初始化，分配一个anon_vma_cachep作为anon_vma的slab缓存，这个技术是PFRA（页框回收算法）技术中的组成部分，用于快速的定位指向同一页框的所有页表项；
* efi_enter_virtual_mode：初始化EFI并进入虚拟模式，EFI是ExtensibleFirmware Interface的缩写，就是INTEL公司新开发的BIOS接口；
* thread_stack_cache_init：内核线程栈缓存管理系统的初始化；
* cred_init：凭证子系统系统初始化，用于表示任务的安全属性；
* fork_init：根据当前物理内存计算出来可以创建进程（线程）的最大数量，并进行进程环境初始化，为task_struct分配空间
* proc_caches_init：进程缓存初始化；
* uts_ns_init：内核 UTS（UNIX Timesharing System）命名空间初始化，用于隔离系统标识符，如节点名（hostname）、域名（domainname）等；
* buffer_init：文件系统的缓存区初始化，并计算最大可以使用的文件缓存；
* key_init：初始化内核安全键管理列表和结构，内核秘钥管理系统；
* security_init：初始化内核安全管理框架，以便提供文件\登陆等权限；
* dbg_late_init：内核调试系统后期初始化；
* vfs_caches_init：虚拟文件系统进行缓存初始化，提高虚拟文件系统的访问速度；
* pagecache_init：页缓存初始化；
* signals_init：内核中信号处理子系统的初始化；
* seq_file_init：内核序列文件（sequential file）接口的初始化；
* proc_root_init：初始化系统进程文件系统，主要提供内核与用户进行交互的平台，方便用户实时查看进程的信息。
* nsfs_init：为命名空间文件系统（nsfs）进行初始化注册，它并不直接创建任何命名空间，而是为后续通过 `clone` 或 `unshare` 系统调用创建的命名空间提供一个统一的、用于展示和管理的虚拟文件系统接口。
* cpuset_init：初始化CPUSET。CPUSET主要为控制组提供CPU和内存节点的管理的结构；
* cgroup_init：初始化进程控制组，主要用来为进程和其子程提供性能控制。比如限定这组进程的CPU使用率为20％；
* taskstats_init_early：初始化任务状态相关的缓存、队列和信号量。任务状态主要向用户提供任务的状态信息；
* delayacct_init：任务延迟机制初始化，初始化每个任务延时计数。当一个任务等待CPU或者IO同步的时候，都需要计算等待时间；
* poking_init：初始化一个临时的内存区域（称为 "poking area"），这个区域用于安全地动态修改正在运行的内核代码；
* check_bugs：检查CPU配置、FPU等是否非法使用不具备的功能，在ARM架构下check_writebuffer_bugs 测试写缓存一致性；

ACPI及平台相关初始化

```c
	acpi_subsystem_init();
	arch_post_acpi_subsys_init();
	sfi_init_late();
	kcsan_init();
```

* acpi_subsystem_init：acpi子系统初始化；
* arch_post_acpi_subsys_init：架构相关的后 ACPI 初始化；
* sfi_init_late：处理来自 SFI（Simple Firmware Interface）表的系统配置信息。SFI 是英特尔为低功耗、移动设备（如早期的 Intel MID 平台）设计的一种轻量级固件接口；
* kcsan_init：初始化内核并发性检测工具 KCSAN（Kernel Concurrency Sanitizer）。KCSAN 是一个用于在运行时检测内核中的数据竞争（data race）的动态分析工具。

#### 核心子系统的初始化

```c
	/* Do the rest non-__init'ed, we're now alive */
	arch_call_rest_init();
```

* arch_call_rest_init：作为一个架构特定的跳转点，用于调用 `rest_init` 函数。在 `start_kernel` 完成所有核心子系统的初始化后，它通过这个接口将控制权最终交给 `rest_init` 函数。`rest_init` 会创建第一个用户态进程（init进程）并启动内核的空闲任务，标志着内核启动过程的完成。

### arch_call_rest_init

待完善

> [kernel启动流程-start_kernel的执行_7.arch_call_rest_init](https://blog.csdn.net/jasonactions/article/details/114007646)

# 参考

Documentation/arm64/booting.rst

[Linux内核启动流程-基于ARM64](https://mshrimp.github.io/2020/04/19/Linux%E5%86%85%E6%A0%B8%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B-%E5%9F%BA%E4%BA%8EARM64/)

[ARM64 linux 启动](https://www.cnblogs.com/memoryart/p/arm64_linux_boot.html)

[ARM64的启动过程之（一）：内核第一个脚印](http://www.wowotech.net/armv8a_arch/arm64_initialize_1.html)

[3.9. kernel 启动流程](http://www.pedestrian.com.cn/kernel/kernel_start/index.html#kernel)

[内核入口 - start_kernel](https://docs.openatom.club/initialization/linux-initialization-4)

[Linux kernel arm64 启动流程](https://blog.csdn.net/qq_38145502/article/details/151014473?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-151014473-blog-116866285.235^v43^pc_blog_bottom_relevance_base7&spm=1001.2101.3001.4242.2&utm_relevant_index=3)

[Linux 内核启动分析Part 1](https://github.com/buckrudy/Blog/issues/1)
