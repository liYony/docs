## 入口函数

要知道入口函数是什么，我们首先得知道程序的入口地址，以及入口地址上放了什么东西。这些东西我们可以从链接脚本中得到：

```
SECTIONS
{
    . = 0xc0001000;
    . = ALIGN(4096);
    .text :
    {
        KEEP(*(.text.entrypoint))       /* The entry point */
        *(.vectors)
        *(.text)                        /* remaining code */
        *(.text.*)                      /* remaining code */
        ...
    }
    ...
}
```

从该脚本的内容可以知道，程序的入口地址是0xc0001000，依次放的内容是.text.entrypoint、.vectors、.text、.text.*、...段。程序中并没有发现有.text.entrypoint段，但是.vectors段中有内容，内容如下：

```assembly
.section .vectors, "ax"
.code 32

.globl system_vectors
system_vectors:
#ifdef RT_USING_USERSPACE
    b _reset
#else
    ldr pc, _vector_reset
#endif
    ldr pc, _vector_undef
    ldr pc, _vector_swi
    ldr pc, _vector_pabt
    ldr pc, _vector_dabt
    ldr pc, _vector_resv
    ...
```

所以猜测应该是将.vectors段的内容放到了程序的入口地址上，也就是说b _reset这条指令应该在程序的入口地址0xc0000000上。具体是不是我们可以通过反汇编来验证一下，在工程目录执行scons --dump即可获取到反汇编后的代码rtt.asm，如下：

```assembly
.globl system_vectors
system_vectors:
#ifdef RT_USING_USERSPACE
    b _reset
c0001000:       ea00e08d        b       c003923c <_reset>
#else
    ldr pc, _vector_reset
#endif
    ldr pc, _vector_undef
c0001004:       e59ff018        ldr     pc, [pc, #24]   ; c0001024 <_vector_undef>
    ldr pc, _vector_swi
c0001008:       e59ff018        ldr     pc, [pc, #24]   ; c0001028 <_vector_swi>
    ldr pc, _vector_pabt
c000100c:       e59ff018        ldr     pc, [pc, #24]   ; c000102c <_vector_pabt>
    ldr pc, _vector_dabt
c0001010:       e59ff018        ldr     pc, [pc, #24]   ; c0001030 <_vector_dabt>
......
```

我们可以看到程序的入口地址确实在0xc0000000上，所以猜测是正确的。

## 程序的启动

在程序的入口地址0xc0000000处执行的指令是b _reset，所以程序的启动是从_reset函数开始的，下面对_reset函数进行简化并调整部分顺序(不影响功能)，代码如下：

```assembly
.data
.align 14
init_mtbl:
    .space 16*1024

.text
/* reset entry */
.globl _reset
_reset:
    /* invalid tlb before enable mmu */
    mrc p15, 0, r0, c1, c0, 0
    bic r0, #1
    mcr p15, 0, r0, c1, c0, 0
    dsb
    isb
    mov r0, #0
    mcr p15, 0, r0, c8, c7, 0
    mcr p15, 0, r0, c7, c5, 0    /* iciallu */
    mcr p15, 0, r0, c7, c5, 6    /* bpiall */
    dsb
    isb

    ldr r5, =PV_OFFSET
    ldr r9, =KERNEL_VADDR_START

    mov r7, #0x100000
    sub r7, #1
    mvn r8, r7
    ldr r6, =__bss_end
    add r6, r7
    and r6, r8    /* r6 end vaddr align up to 1M */
    sub r6, r9    /* r6 is size */

    ldr sp, =svc_stack_n_limit
    add sp, r5    /* use paddr */

    ldr r0, =init_mtbl
    add r0, r5
    mov r1, r6
    mov r2, r5
    bl init_mm_setup

    ldr lr, =after_enable_mmu
    ldr r0, =init_mtbl
    add r0, r5
    b enable_mmu

after_enable_mmu:

    /* disable the data alignment check */
    mrc p15, 0, r1, c1, c0, 0
    bic r1, #(1<<1)
    mcr p15, 0, r1, c1, c0, 0

    /* setup stack */
    bl  stack_setup

    /* clear .bss */
    mov r0,#0                   /* get a zero                       */
    ldr r1,=__bss_start         /* bss start                        */
    ldr r2,=__bss_end           /* bss end                          */

bss_loop:
    cmp r1,r2                   /* check if data to clear           */
    strlo r0,[r1],#4            /* clear 4 bytes                    */
    blo bss_loop                /* loop until done                  */

    /* initialize the mmu table and enable mmu */
    ldr r0, =platform_mem_desc
    ldr r1, =platform_mem_desc_size
    ldr r1, [r1]
    bl rt_hw_init_mmu_table

    ldr r0, =MMUTable     /* vaddr    */
    add r0, r5            /* to paddr */
    bl  switch_mmu

    /* call C++ constructors of global objects */
    ldr     r0, =__ctors_start__
    ldr     r1, =__ctors_end__

ctor_loop:
    cmp     r0, r1
    beq     ctor_end
    ldr     r2, [r0], #4
    stmfd   sp!, {r0-r1}
    mov     lr, pc
    bx      r2
    ldmfd   sp!, {r0-r1}
    b       ctor_loop
ctor_end:

    /* start RT-Thread Kernel */
    ldr     pc, _rtthread_startup
_rtthread_startup:
    .word rtthread_startup
```

这段代码里面还是有几个重要知识点的，这里将代码拆出来分别讲解。

首先，对于ART-PI这个板子，使用的是IMX6ULL芯片，内存的起始地址为0x80000000，

