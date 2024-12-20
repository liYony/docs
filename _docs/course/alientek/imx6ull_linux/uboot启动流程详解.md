# uboot启动流程详解

## _start

```c
_start:

#ifdef CONFIG_SYS_DV_NOR_BOOT_CFG
	.word	CONFIG_SYS_DV_NOR_BOOT_CFG
#endif

	b	reset
	ldr	pc, _undefined_instruction
	ldr	pc, _software_interrupt
	ldr	pc, _prefetch_abort
	ldr	pc, _data_abort
	ldr	pc, _not_used
	ldr	pc, _irq
	ldr	pc, _fiq
```

## reset

```c
reset:
	/* Allow the board to save important registers */
	b	save_boot_params
```

## save_boot_params

```c
ENTRY(save_boot_params)
	b	save_boot_params_ret		@ back to my caller
ENDPROC(save_boot_params)
```

## save_boot_params_ret

```c
save_boot_params_ret:
	/*
	 * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
	 * except if in HYP mode already
	 */
    // 读取寄存器 cpsr中的值，并保存到 r0寄存器中。
	mrs	r0, cpsr
    // 将寄存器r0 中的值与 0X1F 进行与运算，结果保存到rl 寄存器中，目的就是提取 cpsr 的 bit0~bit4 这5位，这5位为 M4 M3 M2 M1 M0，M[4:0]这五位用来设置处理器的工作模式。
	and	r1, r0, #0x1f		@ mask mode bits
    // 判断 r1 寄存器的值是否等于 0X1A(0b11010)，也就是判断当前处理器模式是否处于 Hyp 模式。
	teq	r1, #0x1a		@ test for HYP mode
    // 如果 rl 和 0X1A 不相等，也就是 CPU 不处于 Hyp 模式的话就将 r0 寄存器的bito~5进行清零，其实就是清除模式位。
	bicne	r0, r0, #0x1f		@ clear all mode bits
    // 如果处理器不处于 Hyp 模式的话就将 r0 的寄存器的值与 Ox13 进行或运算,0x13=0b10011，也就是设置处理器进入 SVC 模式。
	orrne	r0, r0, #0x13		@ set SVC mode
    // r0 寄存器的值再与 0xC0 进行或运算，那么 r0 寄存器此时的值就是 0xD3，cpSr的I为和 F 位分别控制IRQ 和 FIQ这两个中断的开关，设置为1就关闭了 FIQ 和IRQ!
	orr	r0, r0, #0xc0		@ disable FIQ and IRQ
    // 将r0寄存器写回到 cpsr寄存器中。
	msr	cpsr,r0

/*
 * Setup vector:
 * (OMAP4 spl TEXT_BASE is not 32 byte aligned.
 * Continue to use ROM code vector only in OMAP4 spl)
 */
#if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
	/* Set V=0 in CP15 SCTLR register - for VBAR to point to vector */
    // 设置SCTLR寄存器第13bit(CR_V=1<<13)的值V=0：
    //     - V=0：向量表的基地址为0x00000000，软件可以重定位向量表。
    //     - V=1：向量表的基地址为0xFFFF0000，软件不能重定位向量表。
	mrc	p15, 0, r0, c1, c0, 0	@ Read CP15 SCTLR Register
	bic	r0, #CR_V		@ V = 0
	mcr	p15, 0, r0, c1, c0, 0	@ Write CP15 SCTLR Register

	/* Set vector address in CP15 VBAR register */
    // 设置r0寄存器的值为_start，也就是uboot的入口地址(0x87800000)。该地址也是向量表的起始地址。
	ldr	r0, =_start
    // 将r0寄存器的值写入到CP15的c12寄存器，也就是VBAR寄存器，相当于重新设置了向量表的基地址。
	mcr	p15, 0, r0, c12, c0, 0	@Set VBAR
#endif

	/* the mask ROM code should have PLL and others stable */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
	bl	cpu_init_cp15
	bl	cpu_init_crit
#endif

	bl	_main
```

## cpu_init_cp15

```c
ENTRY(cpu_init_cp15)
	/*
	 * Invalidate L1 I/D
	 */
	mov r0, #0          @ set up for MCR
	mcr p15, 0, r0, c8, c7, 0   @ invalidate TLBs　　/*禁止从TLB中取地址描述符,也就是禁止虚拟地址到物理地址的转换，因为刚开始操作的都是物理寄存器！*/
	mcr p15, 0, r0, c7, c5, 0   @ invalidate icache /*关闭指令cache*/
	mcr p15, 0, r0, c7, c5, 6   @ invalidate BP array　/*关闭分支预测*/
	mcr     p15, 0, r0, c7, c10, 4  @ DSB　/*多核cpu之间进行数据同步*/
	mcr     p15, 0, r0, c7, c5, 4   @ ISB　/*进行指令同步，放弃流水线中已经取到的指令，重新取指令*/	
	/*
	 * disable MMU stuff and caches
	 */
	/*******************************************************
	*1、为什么要关闭mmu？
	*因为MMU是把虚拟地址转化为物理地址得作用
	*而我们现在是要设置控制寄存器，而控制寄存器本来就是实地址（物理地址），
	*再使能MMU，不就是多此一举了吗？
	********************************************************/	
	/******************************************************************
	*2、为什么要关闭cache？
	*catch和MMU是通过CP15管理的，刚上电的时候，CPU还不能管理他们。
	*所以上电的时候MMU必须关闭，指令cache可关闭，可不关闭，但数据cache一定要关闭
	*否则可能导致刚开始的代码里面，去取数据的时候，从catch里面取，
	*而这时候RAM中数据还没有cache过来，导致数据预取异常
	*******************************************************************/
    mrc p15, 0, r0, c1, c0, 0
    bic r0, r0, #0x00002000 @ clear bits 13 (--V-)　/*设置成正常异常模式，即异常向量表的基地址为0x00000000*/
    bic r0, r0, #0x00000007 @ clear bits 2:0 (-CAM) /*关闭指令cache,关闭指令对齐检测，关闭mmu*/
    orr r0, r0, #0x00000002 @ set bit 1 (--A-) Align /*使能对齐检测*/
    orr r0, r0, #0x00000800 @ set bit 11 (Z---) BTB /*使能分支预测*/
#ifdef CONFIG_SYS_ICACHE_OFF
    bic r0, r0, #0x00001000 @ clear bit 12 (I) I-cache
#else
    orr r0, r0, #0x00001000 @ set bit 12 (I) I-cache　/*时能指令cache*/
#endif
    mcr p15, 0, r0, c1, c0, 0

#ifdef CONFIG_ARM_ERRATA_716044
    mrc p15, 0, r0, c1, c0, 0   @ read system control register
    orr r0, r0, #1 << 11    @ set bit #11
    mcr p15, 0, r0, c1, c0, 0   @ write system control register
#endif

#if (defined(CONFIG_ARM_ERRATA_742230) || defined(CONFIG_ARM_ERRATA_794072))
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 4     @ set bit #4
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

#ifdef CONFIG_ARM_ERRATA_743622
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 6     @ set bit #6
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

#ifdef CONFIG_ARM_ERRATA_751472
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 11    @ set bit #11
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif
#ifdef CONFIG_ARM_ERRATA_761320
    mrc p15, 0, r0, c15, c0, 1  @ read diagnostic register
    orr r0, r0, #1 << 21    @ set bit #21
    mcr p15, 0, r0, c15, c0, 1  @ write diagnostic register
#endif

    mov pc, lr          @ back to my caller　/*程序返回*/
ENDPROC(cpu_init_cp15)
```

## cpu_init_crit

```c
ENTRY(cpu_init_crit)
	/*
	 * Jump to board specific initialization...
	 * The Mask ROM will have already initialized
	 * basic memory. Go here to bump up clock rate and handle
	 * wake up conditions.
	 */
	b	lowlevel_init		@ go setup pll,mux,memory
ENDPROC(cpu_init_crit)
```

## lowlevel_init

```c
ENTRY(lowlevel_init)
	/*
	 * Setup a temporary stack. Global data is not available yet.
	 */
    // 根据include/configs/mx6ullevk.h相关宏定义得出CONFIG_SYS_INIT_SP_ADDR = 0x00900000 + 0X1FF00 = 0X0091FF00，所以sp指向0X0091FF00，属于内部RAM
    /*
                                    |-------------------|<-- 0x00900000
                                    |                   |
                                    |                   |
                                    |                   |
                                    |                   |
                                    |                   |
                                    |                   |
      CONFIG_SYS_INIT_SP_ADDR/sp -->|-------------------|<-- 0x0091FF00
                                    |                   |
                                    |-------------------|<-- 0x0091FFFF
    */
	ldr	sp, =CONFIG_SYS_INIT_SP_ADDR
	bic	sp, sp, #7 /* 8-byte alignment for ABI compliance */
#ifdef CONFIG_SPL_DM
	mov	r9, #0
#else
	/*
	 * Set up global data for boards that still need it. This will be
	 * removed soon.
	 */
#ifdef CONFIG_SPL_BUILD
	ldr	r9, =gdata
#else
    // sp指针减去GD_SIZE(248)，然后做8字节对齐
    /*
                                |-------------------|<-- 0x00900000
                                |                   |
                                |                   |
                                |                   |
                                |                   |
                          sp -->|-------------------|<-- 0x0091FE08
                                |                   |
     CONFIG_SYS_INIT_SP_ADDR -->|-------------------|<-- 0x0091FF00
                                |                   |
                                |-------------------|<-- 0x0091FFFF
    */
	sub	sp, sp, #GD_SIZE
	bic	sp, sp, #7
	mov	r9, sp
#endif
#endif
	/*
	 * Save the old lr(passed in ip) and the current lr to stack
	 */
	push	{ip, lr}

	/*
	 * Call the very early init function. This should do only the
	 * absolute bare minimum to get started. It should not:
	 *
	 * - set up DRAM
	 * - use global_data
	 * - clear BSS
	 * - try to start a console
	 *
	 * For boards with SPL this should be empty since SPL can do all of
	 * this init in the SPL board_init_f() function which is called
	 * immediately after this.
	 */
	bl	s_init
    // 出栈的时候pc=lr
	pop	{ip, pc}
ENDPROC(lowlevel_init)
```

## s_init

```c
void s_init(void)
{
	struct anatop_regs *anatop = (struct anatop_regs *)ANATOP_BASE_ADDR;
	struct mxc_ccm_reg *ccm = (struct mxc_ccm_reg *)CCM_BASE_ADDR;
	u32 mask480;
	u32 mask528;
	u32 reg, periph1, periph2;

	if (is_cpu_type(MXC_CPU_MX6SX) || is_cpu_type(MXC_CPU_MX6UL) ||
	    is_cpu_type(MXC_CPU_MX6ULL) || is_cpu_type(MXC_CPU_MX6SLL))
		return;

	/* Due to hardware limitation, on MX6Q we need to gate/ungate all PFDs
	 * to make sure PFD is working right, otherwise, PFDs may
	 * not output clock after reset, MX6DL and MX6SL have added 396M pfd
	 * workaround in ROM code, as bus clock need it
	 */

	mask480 = ANATOP_PFD_CLKGATE_MASK(0) |
		ANATOP_PFD_CLKGATE_MASK(1) |
		ANATOP_PFD_CLKGATE_MASK(2) |
		ANATOP_PFD_CLKGATE_MASK(3);
	mask528 = ANATOP_PFD_CLKGATE_MASK(1) |
		ANATOP_PFD_CLKGATE_MASK(3);

	reg = readl(&ccm->cbcmr);
	periph2 = ((reg & MXC_CCM_CBCMR_PRE_PERIPH2_CLK_SEL_MASK)
		>> MXC_CCM_CBCMR_PRE_PERIPH2_CLK_SEL_OFFSET);
	periph1 = ((reg & MXC_CCM_CBCMR_PRE_PERIPH_CLK_SEL_MASK)
		>> MXC_CCM_CBCMR_PRE_PERIPH_CLK_SEL_OFFSET);

	/* Checking if PLL2 PFD0 or PLL2 PFD2 is using for periph clock */
	if ((periph2 != 0x2) && (periph1 != 0x2))
		mask528 |= ANATOP_PFD_CLKGATE_MASK(0);

	if ((periph2 != 0x1) && (periph1 != 0x1) &&
		(periph2 != 0x3) && (periph1 != 0x3))
		mask528 |= ANATOP_PFD_CLKGATE_MASK(2);

	writel(mask480, &anatop->pfd_480_set);
	writel(mask528, &anatop->pfd_528_set);
	writel(mask480, &anatop->pfd_480_clr);
	writel(mask528, &anatop->pfd_528_clr);
}
```

## _main

```c
ENTRY(_main)

/*
 * Set up initial C runtime environment and call board_init_f(0).
 */

#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
	ldr	sp, =(CONFIG_SPL_STACK)
#else
    // sp指向0x0091FF00
	ldr	sp, =(CONFIG_SYS_INIT_SP_ADDR)
#endif
#if defined(CONFIG_CPU_V7M)	/* v7M forbids using SP as BIC destination */
	mov	r3, sp
	bic	r3, r3, #7
	mov	sp, r3
#else
	bic	sp, sp, #7	/* 8-byte alignment for ABI compliance */
#endif
    // 调用board_init_f_alloc_reserve函数，并且参数为r0(sp)
    // board_init_f_alloc_reserve函数主要为了留出早期的malloc内存区域和gd内存区域
    // 执行完的结果如下：
    /*
                                    |-------------------|<-- 0x00900000
                                    |                   |
                                    |                   |
                                    |                   |
                             top -->|-------------------|<-- 0x0091FA00
                                    |  gd 256B(248+8)   |
                                    |-------------------|<-- 0x0091FB00
                                    |  early malloc     |
                                    |     0x400B        |
      CONFIG_SYS_INIT_SP_ADDR/sp -->|-------------------|<-- 0x0091FF00
                                    |      256B         |
                                    |-------------------|<-- 0x0091FFFF
    */
	mov	r0, sp
	bl	board_init_f_alloc_reserve
    // board_init_f_alloc_reserve函数的返回值为r0(top)，将sp指向r0，也就是说sp=0x0091FA00
	mov	sp, r0
	/* set up gd here, outside any C code */
    // 将r0寄存器的值写入到r9寄存器，因为r9寄存器保存着全局变量gd的地址。
    // #define   DECLARE_GLOBAL_DATA_PTR      register volatile gd_t *gd asm("r9")
    // uboot中定义了一个指向gd_t的指针gd，gd存放在r9里面的。
	mov	r9, r0
    // board_init_f_init_reserve主要用于初始化gd，也就是清零内存；
    // 另外设置了gd->malloc_base为 gd基地址+gd大小做16字节对齐=0x0091FA00+256(248+8)=0x0091FB00。
    /*
                                    |-------------------|<-- 0x00900000
                                    |                   |
                                    |                   |
                                    |                   |
                              sp -->|-------------------|<-- 0x0091FA00
                                    |  gd 256B(248+8)   |
                 gd->malloc_base -->|-------------------|<-- 0x0091FB00
                                    |  early malloc     |
                                    |     0x400B        |
         CONFIG_SYS_INIT_SP_ADDR -->|-------------------|<-- 0x0091FF00
                                    |      256B         |
                                    |-------------------|<-- 0x0091FFFF
    */
	bl	board_init_f_init_reserve

	mov	r0, #0
    /*
        |-----------------------------------------|<--- 0xA0000000
        |          reserve_round_4k=0             |
        |-----------------------------------------|<--- 0xA0000000
        |          reserve_mmu=0x4000             |
        |              (64KB alignment)           |
        |-----------------------------------------|<--- 0x9FFF0000
        |                                         |
        |          reserve_uboot=0xA8EF4          |
        |               (4KB alignment)           |
        |                                         |
        |-----------------------------------------|<--- 0x9FF47000 ----- gd->relocaddr
        |                                         |
        |                                         |
        |        reserve_malloc=16MB+8KB          |
        |                                         |
        |                                         |
        |-----------------------------------------|<--- 0x9EF45000
        |            reserve_board=80B            |
        |               bd(bd_t)                  |
        |-----------------------------------------|<--- 0x9EF44FB0 ----- gd->bd
        |      reserve_global_data=248B(bd)       |
        |               gd(gd_t)                  |
        |-----------------------------------------|<--- 0x9EF44EB8 ----- gd->new_gd
        |            reserve_stacks               |
        |-----------------------------------------|<--- 0x9EF44E90 ----- gd->start_addr_sp
        |                                         |
        |                                         |
        |                                         |
        |                                         |
        |                                         |
        |                                         |
        |-----------------------------------------|<--- 0x80000000
    */
	bl	board_init_f

#if ! defined(CONFIG_SPL_BUILD)

/*
 * Set up intermediate environment (new sp and gd) and call
 * relocate_code(addr_moni). Trick here is that we'll return
 * 'here' but relocated.
 */
    // 重新设置环境(sp和gd)，在上面的board_init_f会初始化gd的所有成员变量，其中gd->start_addr_sp=0x9EF44E90，该地址为DDR中的地址。
    // 说明新的sp和gd将会存放在DDR中去。而不是内部的RAM了。
	ldr	sp, [r9, #GD_START_ADDR_SP]	/* sp = gd->start_addr_sp */
#if defined(CONFIG_CPU_V7M)	/* v7M forbids using SP as BIC destination */
	mov	r3, sp
	bic	r3, r3, #7
	mov	sp, r3
#else
	bic	sp, sp, #7	/* 8-byte alignment for ABI compliance */
#endif
    // 首先将gd->bd的值赋给r9，然后由于新的gd在bd的下面，所以再用r9减去gd的大小就是新的gd的位置，获取到新的gd的位置然后赋值给r9
    /*
        |-----------------------------------------|<--- 0x9EF45000
        |            reserve_board=80B            |
        |               bd(bd_t)                  |
        |-----------------------------------------|<--- 0x9EF44FB0 ----- gd->bd
        |      reserve_global_data=248B(bd)       |
        |               gd(gd_t)                  |
        |-----------------------------------------|<--- 0x9EF44EB8 ----- gd->new_gd
    */
	ldr	r9, [r9, #GD_BD]		/* r9 = gd->bd */
	sub	r9, r9, #GD_SIZE		/* new GD is below bd */

    // 设置lr寄存器为here，这样后面执行其他函数返回的时候就返回到了下面的here标号处
	adr	lr, here
	ldr	r0, [r9, #GD_RELOC_OFF]		/* r0 = gd->reloc_off */
    // lr寄存器的值加上r0的值然后重新赋值给lr寄存器。
    // 因为接下来要重定位代码，也就是把代码拷贝到新的地方去(现在的uboot存放的起始地址为0x87800000，下面要将uboot拷贝到DDR最后的地址空间去，
    // 把0x87800000开始的内存空出来，其中就包括here，因此lr中的here要使用重定位后的地址。)
	add	lr, lr, r0
#if defined(CONFIG_CPU_V7M)
	orr	lr, #1				/* As required by Thumb-only */
#endif
    // r0等于uboot需要重定位的地址。这里为0x9FF47000，并且作为relocate_code的参数然后执行relocate_code函数开始重定位代码。
	ldr	r0, [r9, #GD_RELOCADDR]		/* r0 = gd->relocaddr */
	b	relocate_code
here:
/*
 * now relocate vectors
 */
    // 做中断向量表的重定位
	bl	relocate_vectors

/* Set up final (full) environment */

	bl	c_runtime_cpu_setup	/* we still call old routine here */
#endif
#if !defined(CONFIG_SPL_BUILD) || defined(CONFIG_SPL_FRAMEWORK)
# ifdef CONFIG_SPL_BUILD
	/* Use a DRAM stack for the rest of SPL, if requested */
	bl	spl_relocate_stack_gd
	cmp	r0, #0
	movne	sp, r0
	movne	r9, r0
#endif
    // 清除bss段
	ldr	r0, =__bss_start	/* this is auto-relocated! */

#ifdef CONFIG_USE_ARCH_MEMSET
	ldr	r3, =__bss_end		/* this is auto-relocated! */
	mov	r1, #0x00000000		/* prepare zero to clear BSS */

	subs	r2, r3, r0		/* r2 = memset len */
	bl	memset
#else
	ldr	r1, =__bss_end		/* this is auto-relocated! */
	mov	r2, #0x00000000		/* prepare zero to clear BSS */

clbss_l:cmp	r0, r1			/* while not at end of BSS */
#if defined(CONFIG_CPU_V7M)
	itt	lo
#endif
	strlo	r2, [r0]		/* clear 32-bit BSS word */
	addlo	r0, r0, #4		/* move to next */
	blo	clbss_l
#endif

#if ! defined(CONFIG_SPL_BUILD)
	bl coloured_LED_init
	bl red_led_on
#endif
    // 下面调用board_init_r函数，并且传入两个参数
    //      - r0：r0=r9，也就是说第一个参数为gd的地址。
    //      - r1：r1=gd->relocaddr，uboot重定位后的地址。
	/* call board_init_r(gd_t *id, ulong dest_addr) */
	mov     r0, r9                  /* gd_t */
	ldr	r1, [r9, #GD_RELOCADDR]	/* dest_addr */
	/* call board_init_r */
#if defined(CONFIG_SYS_THUMB_BUILD)
	ldr	lr, =board_init_r	/* this is auto-relocated! */
	bx	lr
#else
	ldr	pc, =board_init_r	/* this is auto-relocated! */
#endif
	/* we should not return here. */
#endif

ENDPROC(_main)
```

## board_init_f

```c
void board_init_f(ulong boot_flags)
{
// #ifdef CONFIG_SYS_GENERIC_GLOBAL_DATA
// 	/*
// 	 * For some archtectures, global data is initialized and used before
// 	 * calling this function. The data should be preserved. For others,
// 	 * CONFIG_SYS_GENERIC_GLOBAL_DATA should be defined and use the stack
// 	 * here to host global data until relocation.
// 	 */
// 	gd_t data;

// 	gd = &data;

// 	/*
// 	 * Clear global data before it is accessed at debug print
// 	 * in initcall_run_list. Otherwise the debug print probably
// 	 * get the wrong vaule of gd->have_console.
// 	 */
// 	zero_global_data();
// #endif

	gd->flags = boot_flags;
	gd->have_console = 0;
    // 运行初始化序列init_sequence_f里面的一系列函数，init_sequence_f里面包含了一系列的初始化函数(定义在common/borad_f.c)
	if (initcall_run_list(init_sequence_f))
		hang();

#if !defined(CONFIG_ARM) && !defined(CONFIG_SANDBOX) && \
		!defined(CONFIG_EFI_APP)
	/* NOTREACHED - jump_to_copy() does not return */
	hang();
#endif
}
```

## initcall_run_list

```c
int initcall_run_list(const init_fnc_t init_sequence[])
{
	const init_fnc_t *init_fnc_ptr;

	for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
		unsigned long reloc_ofs = 0;
		int ret;

		if (gd->flags & GD_FLG_RELOC)
			reloc_ofs = gd->reloc_off;
#ifdef CONFIG_EFI_APP
		reloc_ofs = (unsigned long)image_base;
#endif
		debug("initcall: %p", (char *)*init_fnc_ptr - reloc_ofs);
		if (gd->flags & GD_FLG_RELOC)
			debug(" (relocated to %p)\n", (char *)*init_fnc_ptr);
		else
			debug("\n");
		ret = (*init_fnc_ptr)();
		if (ret) {
			printf("initcall sequence %p failed at call %p (err=%d)\n",
			       init_sequence,
			       (char *)*init_fnc_ptr - reloc_ofs, ret);
			return -1;
		}
	}
	return 0;
}
```

## init_sequence_f[]

```c
static init_fnc_t init_sequence_f[] = {
    // 设置gd的mon_len成员变量，此处为__bss_end-_start，也就是整个代码的长度。0X878A8E74-0x87800000=0XA8E74
	setup_mon_len,
    // 初始化gd中跟malloc有关的成员变量，比如malloc_limit，此函数会设置gd->malloc_limit=CONFIG_SYS_MALLOC_F_LEN=0X400；malloc_limit表示malloc内存池大小。
	initf_malloc,
	initf_console_record,
	arch_cpu_init,		/* basic arch cpu dependent setup */
	initf_dm,
	arch_cpu_init_dm,
	mark_bootstage,		/* need timer, go after init dm */
    // 板子早期的一些初始化，I.MX6ULL用来初始化串口的IO配置
	board_early_init_f,
    // 初始化定时器，Cortex-A7内核有一个定时器，通过这个定时器给uboot提供时间。
	timer_init,		/* initialize timer */
    // 对于I.MX6ULL来说是设置VDDSOC电压。
	board_postclk_init,
    // 用于获取一些时钟，I.MX6ULL获取的是sdhc_clk时钟，也就是sd卡外设的时钟。
	get_clocks,
    // env_init函数是和环境变量有关的，设置gd的成员变量env_addr，也就是环境变量保存的地址。
	env_init,		/* initialize environment */
    // 用于初始化波特率，根据环境变量baundrate来初始化gd->baundrate。
	init_baud_rate,		/* initialze baudrate settings */
    // 初始化串口
	serial_init,		/* serial communications setup */
    // 设置gd->have_console=1，表示有个控制台，此函数也将前面暂存在缓存区中的数据通过控制台打印出来。
	console_init_f,		/* stage 1 init of console */
	display_options,	/* say that we are here */
	display_text_info,	/* show debugging info if required */
	print_cpuinfo,		/* display cpu info (and speed) */
	show_board_info,

	INIT_FUNC_WATCHDOG_INIT
	INIT_FUNC_WATCHDOG_RESET
	init_func_i2c,
    // 输出字符串“DRAM:”
	announce_dram_init,
    // 初始化DRAM，对于I.MX6ULL，只是设置gd->ram_size的值，也就是512MB
	dram_init,		/* configure available RAM banks */
    // 完成一些测试，初始化gd->post_init_f_time
	post_init_f,
	INIT_FUNC_WATCHDOG_RESET
    // testdram，测试DRAM，空函数。
	testdram,
	INIT_FUNC_WATCHDOG_RESET
	INIT_FUNC_WATCHDOG_RESET
	/*
	 * Now that we have DRAM mapped and working, we can
	 * relocate the code and continue running from DRAM.
	 *
	 * Reserve memory at end of RAM for (top down in that order):
	 *  - area that won't get touched by U-Boot and Linux (optional)
	 *  - kernel log buffer
	 *  - protected RAM
	 *  - LCD framebuffer
	 *  - monitor code
	 *  - board info struct
	 */
    // 设置目的地址，设置如下三个参数的值：
    //      - gd->ram_size=0X20000000   //ram大小为 0X20000000=512MB
    //      - gd->ram_top=0XA0000000    //ram最高地址为 0X80000000+0X20000000=0XA0000000
    //      - gd->relocaddr=0XA0000000  //重定位后最高地址为 0XA0000000
	setup_dest_addr,
    // 对gd->relocaddr作4k对齐
    reserve_round_4k,
    // 留出MMU的TLB的位置，分配MMU的TLB表内存以后会对gd->relocaddr做64K字节对齐。设置下面值：
    //      - gd->arch.tlb_size= 0X4000         //MMU的 TLB表大小
    //      - gd->arch.tlb_addr=0X9FFF0000      //MMU的 TLB表起始地址， ，64KB对齐以后
    //      - gd->relocaddr=0X9FFF0000          //relocaddr地址
    reserve_mmu,
    // 留出跟踪调试的内存。
    reserve_trace,
    // 留出重定位后的uboot所占用的内存区域，uboot所占大小由gd->mon_len所指定，留出uboot的空间还要对gd->relocaddr做4K字节对齐。并且重新设置gd->start_addr_sp。
    //      - gd->mon_len = 0XA8EF4 
    //      - gd->start_addr_sp = 0X9FF47000 
    //      - gd->relocaddr = 0X9FF47000
    reserve_uboot,
    // 留出malloc区域，调整gd->start_addr_sp位置，malloc区域由TOTAL_MALLOC_LEN定义：
    // CONFIG_SYS_MALLOC_LEN为16MB=0X1000000;
    // CONFIG_ENV_SIZE=8KB=0X2000
    //      - TOTAL_MALLOC_LEN=CONFIG_SYS_MALLOC_LEN+CONFIG_ENV_SIZE=0X1002000
    //      - gd->start_addr_sp=0X9EF45000 //0X9FF47000-16MB-8KB=0X9EF45000
    reserve_malloc,
    // 留出板子bd所占的内存区，bd是结构体bd_t，大小为80字节。
    //      - gd->start_addr_sp=0X9EF44FB0 
    //      - gd->bd=0X9EF44FB0
    reserve_board,
    setup_machine,
    // 留出gd_t的内存区域，gd_t结构体大小为248字节。
    //      - gd->start_addr_sp=0X9EF44EB8      //0X9EF44FB0-248=0X9EF44EB8 
    //      - gd->new_gd=0X9EF44EB8
    reserve_global_data,
    // 留出设备树的内存区域，I.MX6ULL没用到
    reserve_fdt,
    reserve_arch,
    // 留出栈空间，先对gd->start_addr_sp减去16，然后在做16字节对齐。如果使能IRQ的话还要留出IRQ相应的内存。
    //      - gd->start_addr_sp=0X9EF44E90
    reserve_stacks,
    // 设置dram信息。就是设置gd->bd->bi_dram[0].start和gd->bd->bi_dram[0].size。后面会传递给linux内核，告诉linux DRAM的起始地址和大小。
    //      - gd->bd->bi_dram[0].start=0x80000000
    //      - gd->bd->bi_dram[0].size=0x20000000
    setup_dram_config,
    show_dram_config,
    // 显示新的sp
    //      - gd->start_addr_sp=0X9EF44E90
    display_new_sp,
    INIT_FUNC_WATCHDOG_RESET
    // 重定位设备树
    reloc_fdt,
    // 设置gd的一些其他成员变量，供后面重定位使用，并且将以前的gd拷贝到gd->new处。
    //      - Relocation offsetis:18747000
    //      - Relocating to 9ff47000, new gd at 9ef44eb8，sp at 9ef44e90
    setup_reloc,
    NULL,
};
```

## relocate_code

```c
ENTRY(relocate_code)
    // __image_copy_start = 0X87800000
	ldr	r1, =__image_copy_start	/* r1 <- SRC &__image_copy_start */
	subs	r4, r0, r1		/* r4 <- relocation offset */
	beq	relocate_done		/* offset = 0, skip relocation */
    // __image_copy_end = 0x8785dd54。
	ldr	r2, =__image_copy_end	/* r2 <- SRC &__image_copy_end */

    // 循环拷贝
copy_loop:
	ldmia	r1!, {r10-r11}		/* copy from source address [r1]    */
	stmia	r0!, {r10-r11}		/* copy to   target address [r0]    */
	cmp	r1, r2			/* until source end address [r2]    */
	blo	copy_loop

	/*
	 * fix .rel.dyn relocations
	 */
	ldr	r2, =__rel_dyn_start	/* r2 <- SRC &__rel_dyn_start */
	ldr	r3, =__rel_dyn_end	/* r3 <- SRC &__rel_dyn_end */
fixloop:
	/* r0为Label，r1为Flag */
	ldmia	r2!, {r0-r1}		/* (r0,r1) <- (SRC location,fixup) */
	/* 取r1中的低8位 */
	and	r1, r1, #0xff
	/* 判断r1是否等于0x17，不等于则执行fixnext函数 */
	cmp	r1, #23			/* relative fixup? */
	bne	fixnext

	/* relative fix: increase location by offset */
	/* r0 = Label + offset */
	add	r0, r0, r4
	/* 读取拷贝代码后Label+offset地址处的值，该值也就是拷贝代码前的值 */
	ldr	r1, [r0]
	/* Label+offset地址处的值 = (Label+offset地址处的值) + offset */
	add	r1, r1, r4
	/* 将Label+offset地址处的值加上offset的值重新写入Label+offset地址 */
	str	r1, [r0]
fixnext:
	/* .rel.dyn段是否处理完成 */
	cmp	r2, r3
	blo	fixloop

relocate_done:

#ifdef __XSCALE__
	/*
	 * On xscale, icache must be invalidated and write buffers drained,
	 * even with cache disabled - 4.2.7 of xscale core developer's manual
	 */
	mcr	p15, 0, r0, c7, c7, 0	/* invalidate icache */
	mcr	p15, 0, r0, c7, c10, 4	/* drain write buffer */
#endif

	/* ARMv4- don't know bx lr but the assembler fails to see that */

#ifdef __ARM_ARCH_4__
	mov	pc, lr
#else
	bx	lr
#endif

ENDPROC(relocate_code)
```

## relocate_vectors

```c
ENTRY(relocate_vectors)

#ifdef CONFIG_CPU_V7M
	/*
	 * On ARMv7-M we only have to write the new vector address
	 * to VTOR register.
	 */
	ldr	r0, [r9, #GD_RELOCADDR]	/* r0 = gd->relocaddr */
	ldr	r1, =V7M_SCB_BASE
	str	r0, [r1, V7M_SCB_VTOR]
#else
#ifdef CONFIG_HAS_VBAR
	/*
	 * If the ARM processor has the security extensions,
	 * use VBAR to relocate the exception vectors.
	 */
	// 很简单，就是修改CP15的 VBAR寄存器，也就是将新的向量表首地址写入到寄存器VBAR中，设置向量表偏移。
	ldr	r0, [r9, #GD_RELOCADDR]	/* r0 = gd->relocaddr */
	mcr     p15, 0, r0, c12, c0, 0  /* Set VBAR */
#else
	/*
	 * Copy the relocated exception vectors to the
	 * correct address
	 * CP15 c1 V bit gives us the location of the vectors:
	 * 0x00000000 or 0xFFFF0000.
	 */
	ldr	r0, [r9, #GD_RELOCADDR]	/* r0 = gd->relocaddr */
	mrc	p15, 0, r2, c1, c0, 0	/* V bit (bit[13]) in CP15 c1 */
	ands	r2, r2, #(1 << 13)
	ldreq	r1, =0x00000000		/* If V=0 */
	ldrne	r1, =0xFFFF0000		/* If V=1 */
	ldmia	r0!, {r2-r8,r10}
	stmia	r1!, {r2-r8,r10}
	ldmia	r0!, {r2-r8,r10}
	stmia	r1!, {r2-r8,r10}
#endif
#endif
	bx	lr

ENDPROC(relocate_vectors)
```

## board_init_r

```c
void board_init_r(gd_t *new_gd, ulong dest_addr)
{
#ifdef CONFIG_NEEDS_MANUAL_RELOC
	int i;
#endif

#ifdef CONFIG_AVR32
	mmu_init_r(dest_addr);
#endif

#if !defined(CONFIG_X86) && !defined(CONFIG_ARM) && !defined(CONFIG_ARM64)
	gd = new_gd;
#endif

#ifdef CONFIG_NEEDS_MANUAL_RELOC
	for (i = 0; i < ARRAY_SIZE(init_sequence_r); i++)
		init_sequence_r[i] += gd->reloc_off;
#endif

	if (initcall_run_list(init_sequence_r))
		hang();

	/* NOTREACHED - run_main_loop() does not return */
	hang();
}
```

## init_sequence_r[]

```c
init_fnc_t init_sequence_r[] = {
    // 初始化和调试跟踪相关的内容
    initr_trace,
    // 用于设置gd->flags，标记重定位完成
    initr_reloc,
    // 初始化cache，使能cache
    initr_caches,
    // 初始化重定位后gd的一些成员变量
    initr_reloc_global_data,
    initr_barrier,
    // 初始化malloc
    initr_malloc,
    // 初始化控制台相关内容
    initr_console_record,
    // 启动状态重定位
    bootstage_relocate,
    initr_bootstage,
    // 板级初始化，包括74xx芯片，I2C，FEC，USB和QSPI等
    board_init, /* Setup chipselects */
    // stdio初始化
    stdio_init_tables,
    // 初始化串口
    initr_serial,
    // 与调试有关，通知已经在RAM中运行
    initr_announce,
    INIT_FUNC_WATCHDOG_RESET
    INIT_FUNC_WATCHDOG_RESET
    INIT_FUNC_WATCHDOG_RESET
    // 初始化电源芯片
    power_init_board,
    initr_flash,
    INIT_FUNC_WATCHDOG_RESET
    // 初始化nand和emmc
    initr_nand,
    initr_mmc,
    // 初始化环境变量
    initr_env,
    INIT_FUNC_WATCHDOG_RESET
    // 初始化其他核
    initr_secondary_cpu,
    INIT_FUNC_WATCHDOG_RESET
	// 各种输入输出设备初始化，如LCD。I.MX6ULL使用drv_video_init函数初始化LCD。
    stdio_add_devices,
	// 初始化跳转表
    initr_jumptable,
	// 控制台初始化，初始化成功后会调用stdio_print_curennt_devices函数来打印当前的控制台设备。
    console_init_r, /* fully init console as a device */
    INIT_FUNC_WATCHDOG_RESET
	// 初始化中断系统
    interrupt_init,
	// 使能中断
    initr_enable_interrupts,
	// 初始化网络地址，也就是获取MAC地址，读取环境变量<ethaddr>的值
    initr_ethaddr,
	// 板子后续初始化，函数定义在mx6ull_alientek_emmc.c里面，
    board_late_init,
    INIT_FUNC_WATCHDOG_RESET
    INIT_FUNC_WATCHDOG_RESET
    INIT_FUNC_WATCHDOG_RESET
	// 初始化网络设备，调用过程如下：
	// initr_net->eth_initialize->board_eth_init()
    initr_net,
    INIT_FUNC_WATCHDOG_RESET
	// 主循环，处理命令。
    run_main_loop,
};
```

## run_main_loop

```c
static int run_main_loop(void)
{
#ifdef CONFIG_SANDBOX
	sandbox_main_loop_init();
#endif
	/* main_loop() can return to retry autoboot, if so just run it again */
	for (;;)
		main_loop();
	return 0;
}
```

## main_loop

```c
void main_loop(void)
{
	const char *s;
    // 打印启动进度
	bootstage_mark_name(BOOTSTAGE_ID_MAIN_LOOP, "main_loop");

#ifndef CONFIG_SYS_GENERIC_BOARD
	puts("Warning: Your board does not use generic board. Please read\n");
	puts("doc/README.generic-board and take action. Boards not\n");
	puts("upgraded by the late 2014 may break or be removed.\n");
#endif

#ifdef CONFIG_VERSION_VARIABLE
	setenv("ver", version_string);  /* set version variable */
#endif /* CONFIG_VERSION_VARIABLE */
    // 跟命令行初始化相关，初始化hush shell相关的变量
	cli_init();
    // 获取环境变量preboot的内容，该内容是一些预启动命令，一般不适用该环境变量。
	run_preboot_environment_command();

#if defined(CONFIG_UPDATE_TFTP)
	update_tftp(0UL, NULL, NULL);
#endif /* CONFIG_UPDATE_TFTP */
    // 读取环境变量boot_delay和bootcmd的内容，然后将bootdelay的值赋值给全局变量stored_bootdelay，返回值为环境变量bootcmd的值。
	s = bootdelay_process();
	if (cli_process_fdt(&s))
		cli_secure_boot_cmd(s);

    // 检查倒计时是否结束？倒计时结束之前有没有被打断？代码主要判断如下(精简代码)：
    //      - stored_bootdelay != -1        // stored_bootdelay不等于1，即环境变量boot_delay
    //      - s                             // s不为空，即环境变量bootcmd
    //      - !abortboot(stored_bootdelay)  // 函数返回值为0
    //          abortboot->abortboot_normal(bootdelay)
    //              abortboot_normal函数主要完成以下功能：
    //                  - 默认返回值设置为0(表示默认会执行run_command_list函数)。
    //                  - 输出字符串"Hit any key to stop autoboot :"
    //                  - 判断是否有按键按下
    //                      - 有按键按下：设置返回值为1，设置bootdelay=0，跳出倒计时循环。
    //                      - 无按键按下：正常执行完循环，返回值不变还是为0
    // 如果三个条件同时成立就会执行函数run_command_list函数(autoboot_command函数内)。
    //      - 主要执行bootcmd环境变量中的值。
    //
    // 综上，如果autoboot_command函数没被打断，那么执行run_command_list函数，
    // 如果被打断了，autoboot_command函数退出，往下执行cli_loop函数。
	autoboot_command(s);

	cli_loop();
}
```

## cli_loop

```c
void cli_loop(void)
{
#ifdef CONFIG_SYS_HUSH_PARSER
    // 进入
	parse_file_outer();
	/* This point is never reached */
	for (;;);
#else
	cli_simple_loop();
#endif /*CONFIG_SYS_HUSH_PARSER*/
}
```

## parse_file_outer

```c
#ifndef __U_BOOT__
static int parse_file_outer(FILE *f)
#else
int parse_file_outer(void)
#endif
{
	int rcode;
	struct in_str input;
#ifndef __U_BOOT__
	setup_file_in_str(&input, f);
#else
    // 初始化input的成员变量
	setup_file_in_str(&input);
#endif
    // hush shell的解释器，负责接收命令行输入，然后解析相应的命令。
	rcode = parse_stream_outer(&input, FLAG_PARSE_SEMICOLON);
	return rcode;
}
```

## parse_stream_outer

```c
static int parse_stream_outer(struct in_str *inp, int flag)
{

	struct p_context ctx;
	o_string temp=NULL_O_STRING;
	int rcode;
#ifdef __U_BOOT__
	int code = 1;
#endif
	// 下面的do-while就是处理输入命令的。
	do {
		ctx.type = flag;
		initialize_context(&ctx);
		update_ifs_map();
		if (!(flag & FLAG_PARSE_SEMICOLON) || (flag & FLAG_REPARSING)) mapset((uchar *)";$&|", 0);
		inp->promptmode=1;

		// 解析命令！！
		rcode = parse_stream(&temp, &ctx, inp,
				     flag & FLAG_CONT_ON_NEWLINE ? -1 : '\n');
#ifdef __U_BOOT__
		if (rcode == 1) flag_repeat = 0;
#endif
		if (rcode != 1 && ctx.old_flag != 0) {
			syntax();
#ifdef __U_BOOT__
			flag_repeat = 0;
#endif
		}
		if (rcode != 1 && ctx.old_flag == 0) {
			done_word(&temp, &ctx);
			done_pipe(&ctx,PIPE_SEQ);
#ifndef __U_BOOT__
            // 处理解析到的命令函数的调用过程如下：
            // run_list->run_list_real->run_pipe_real->cmd_process
            // 最终是调用cmd_process函数来实现处理命令的。
			run_list(ctx.list_head);
#else
			code = run_list(ctx.list_head);
			if (code == -2) {	/* exit */
				b_free(&temp);
				code = 0;
				/* XXX hackish way to not allow exit from main loop */
				if (inp->peek == file_peek) {
					printf("exit not allowed from main input shell.\n");
					continue;
				}
				break;
			}
			if (code == -1)
			    flag_repeat = 0;
#endif
		} else {
			if (ctx.old_flag != 0) {
				free(ctx.stack);
				b_reset(&temp);
			}
#ifdef __U_BOOT__
			if (inp->__promptme == 0) printf("<INTERRUPT>\n");
			inp->__promptme = 1;
#endif
			temp.nonnull = 0;
			temp.quote = 0;
			inp->p = NULL;
			free_pipe_list(ctx.list_head,0);
		}
		b_free(&temp);
	/* loop on syntax errors, return on EOF */
	} while (rcode != -1 && !(flag & FLAG_EXIT_FROM_LOOP) &&
		(inp->peek != static_peek || b_peek(inp)));
#ifndef __U_BOOT__
	return 0;
#else
	return (code != 0) ? 1 : 0;
#endif /* __U_BOOT__ */
}
```

## cmd_process

```c
enum command_ret_t cmd_process(int flag, int argc, char * const argv[],
			       int *repeatable, ulong *ticks)
{
	enum command_ret_t rc = CMD_RET_SUCCESS;
	cmd_tbl_t *cmdtp;

	/* Look up command in command table */
    // 在命令表中查找对指定的命令。
	cmdtp = find_cmd(argv[0]);
	if (cmdtp == NULL) {
		printf("Unknown command '%s' - try 'help'\n", argv[0]);
		return 1;
	}

	/* found - check max args */
	if (argc > cmdtp->maxargs)
		rc = CMD_RET_USAGE;

#if defined(CONFIG_CMD_BOOTD)
	/* avoid "bootd" recursion */
	else if (cmdtp->cmd == do_bootd) {
		if (flag & CMD_FLAG_BOOTD) {
			puts("'bootd' recursion detected\n");
			rc = CMD_RET_FAILURE;
		} else {
			flag |= CMD_FLAG_BOOTD;
		}
	}
#endif

	/* If OK so far, then do the command */
	if (!rc) {
		if (ticks)
			*ticks = get_timer(0);
        
        // 执行具体的命令。
        // 调用cmdtp->(*cmd)(struct cmd_tbl_s *, int, int, char * const []);成员。
		rc = cmd_call(cmdtp, flag, argc, argv);
		if (ticks)
			*ticks = get_timer(*ticks);
		*repeatable &= cmdtp->repeatable;
	}
	if (rc == CMD_RET_USAGE)
		rc = cmd_usage(cmdtp);
	return rc;
}
```
