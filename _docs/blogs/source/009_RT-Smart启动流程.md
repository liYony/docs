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

对于ART-PI这个板子，使用的是IMX6ULL芯片，对于DDR内存的起始地址为0x80000000，而且我们是将程序烧写在0x80001000这个位置的，那么就会存在一个问题，程序的链接地址为0xC0001000，但是程序又是烧写在0x80001000这个位置，那么程序的运行不会出错吗？答案是肯定的，会出错；但是如果能保证程序开头的代码仅仅只是一些指令，不涉及到内存访问，也不存在绝对地址访问的话，也还是可以运行的，所以对于RT-Smart来说，前面运行的代码基本上都是与位置无关的；这里还隐藏了一个很严重的问题，就是我要保证MMU能成功初始化，那我肯定要初始化页表啊，而页表又是保存在内存上面的，这里我还必须要访问内存，那怎么办呢？其实解决方法很简单，如前面所说，我们知道内核代码的烧录地址和链接地址，那我们可以得出两个地址的offset(paddr-vaddr)值，那么，我们访问某个变量的时候，如果能将变量的链接地址加上一个offset，是不是就能正确的访问到变量真实存在的物理地址，然后在进行访问，就可以解决这个问题了。

**在启用mmu之前失能的TLB**

```assembly
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
```

**保存虚拟地址的起始值以及物理地址和虚拟地址的偏移值**

```assembly
    ldr r5, =PV_OFFSET
    ldr r9, =KERNEL_VADDR_START
```

**计算程序的大小**

```assembly
	mov r7, #0x100000
    sub r7, #1
    mvn r8, r7
    ldr r6, =__bss_end
    add r6, r7
    and r6, r8    /* r6 end vaddr align up to 1M */
    sub r6, r9    /* r6 is size */
```

首先r6等于_bss_end也就是程序的最后地址，然后在对r6向上1M取整，r9为虚拟地址的起始地址0xC0000000，最终的出来的值r6=r6-r9，r6也就是虚拟地址到程序末尾所占的空间大小，对于这一段内存是需要建立页表映射的。

**建立页表内存映射**

```c
    ldr sp, =svc_stack_n_limit
    add sp, r5    /* use paddr */

    ldr r0, =init_mtbl
    add r0, r5
    mov r1, r6
    mov r2, r5
    bl init_mm_setup
```

因为下面会调用init_mm_setup函数(c函数)，所以必须设置栈，这里的栈是在内存上面，而svc_stack_n_limit在链接的时候地址肯定是大于0xC0001000的，所以需要通过访问0xC0001000+offset来访问到真正的内存。

在内核最开始初始化MMU的时候，我们只需要保证cpu发出的地址能准确访问到内存即可，不需要什么内存管理，所以我们可以直接使用段映射来完成这个功能。

![image-20240314212845474](figures/image-20240314212845474.png)

下图为段映射的示例图：

![image-20240314213227374](figures/image-20240314213227374.png)

要映射4GB的空间，段映射是以2^20=1M空间进行映射的，并且有2^12=4096个。所以我们需要一个大小为4096*4(32bit)字节容量来保存页表。代码如下：

```assembly
.data
.align 14
init_mtbl:
    .space 16*1024
```

然后我们看一下init_mm_setup函数是如何进行映射的。

```c
void init_mm_setup(unsigned int *mtbl, unsigned int size, unsigned int pv_off) {
    unsigned int va;

    for (va = 0; va < 0x1000; va++) {
        unsigned int vaddr = (va << 20);
        if (vaddr >= KERNEL_VADDR_START && vaddr - KERNEL_VADDR_START < size) {
            mtbl[va] = ((va << 20) + pv_off) | NORMAL_MEM;
        } else if (vaddr >= (KERNEL_VADDR_START + pv_off) && vaddr - (KERNEL_VADDR_START + pv_off) < size) {
            mtbl[va] = (va << 20) | NORMAL_MEM;
        } else {
            mtbl[va] = 0;
        }
    }
}
```

这里将映射后的示意图展示一下，方便理解：

![image-20240314231830647](figures/image-20240314231830647.png)

解释一下这里为什么要进行两段映射，首先0xC0000000~0xC0012b14这段内存能保证在初始化MMU后，cpu发出的地址就能准确的访问到实际的物理内存了。

0x80000000~0x80012b14这段内存映射主要是为了保证

**使能mmu**

建立好内存映射就可以使能MMU了。

```assembly
    ldr lr, =after_enable_mmu
    ldr r0, =init_mtbl
    add r0, r5
    b enable_mmu

after_enable_mmu:
```

首先将lr寄存器的值保存为after_enable_mmu标号地址，这里的地址应该是链接地址，然后把init_mtbl页表(已经通过+offset转化为真实的物理地址)作为参数执行enable_mmu函数。

```assembly
.align 2
.global enable_mmu
enable_mmu:
    orr r0, #0x18
    mcr p15, 0, r0, c2, c0, 0    /* ttbr0 */

    mov r0, #(1 << 5)            /* PD1=1 */
    mcr p15, 0, r0, c2, c0, 2    /* ttbcr */

    mov r0, #1
    mcr p15, 0, r0, c3, c0, 0    /* dacr */

    /* invalid tlb before enable mmu */
    mov r0, #0
    mcr p15, 0, r0, c8, c7, 0
    mcr p15, 0, r0, c7, c5, 0    /* iciallu */
    mcr p15, 0, r0, c7, c5, 6    /* bpiall */

    mrc p15, 0, r0, c1, c0, 0
    orr r0, #((1 << 12) | (1 << 11))    /* instruction cache, branch prediction */
    orr r0, #((1 << 2) | (1 << 0))      /* data cache, mmu enable */
    mcr p15, 0, r0, c1, c0, 0
    dsb
    isb
    mov pc, lr
```

里面就是初始化MMU的代码，详细内容这里就不做分析了，最后通过mv pc, lr就可以实现跳转到after_enable_mmu函数了，并且此时已经是使用的链接地址了。后面就进入MMU管理的世界了。

**设置栈清除BSS段**

```c
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
```

**重新初始化页表并使能MMU**

```assembly
    /* initialize the mmu table and enable mmu */
    ldr r0, =platform_mem_desc
    ldr r1, =platform_mem_desc_size
    ldr r1, [r1]
    bl rt_hw_init_mmu_table

    ldr r0, =MMUTable     /* vaddr    */
    add r0, r5            /* to paddr */
    bl  switch_mmu
```

先看一下是如何重新初始化页表的：

```c
struct mem_desc platform_mem_desc[] = {  /* 100ask_imx6ull ddr 512M */
    {KERNEL_VADDR_START, KERNEL_VADDR_START + 0x1FFFFFFF, KERNEL_VADDR_START + PV_OFFSET, NORMAL_MEM}
};

void rt_hw_init_mmu_table(struct mem_desc *mdesc, rt_uint32_t size)
{
    /* set page table */
    for(; size > 0; size--)
    {
        rt_hw_mmu_setmtt(mdesc->vaddr_start, mdesc->vaddr_end,
                mdesc->paddr_start, mdesc->attr);
        mdesc++;
    }
    rt_hw_cpu_dcache_clean((void*)MMUTable, sizeof MMUTable);
}

volatile unsigned long MMUTable[4*1024] __attribute__((aligned(16*1024)));
void rt_hw_mmu_setmtt(rt_uint32_t vaddrStart,
                      rt_uint32_t vaddrEnd,
                      rt_uint32_t paddrStart,
                      rt_uint32_t attr)
{
    volatile rt_uint32_t *pTT;
    volatile int i, nSec;
    pTT  = (rt_uint32_t *)MMUTable + (vaddrStart >> 20);
    nSec = (vaddrEnd >> 20) - (vaddrStart >> 20);
    for(i = 0; i <= nSec; i++)
    {
        *pTT = attr | (((paddrStart >> 20) + i) << 20);
        pTT++;
    }
}
```

主要是建立物理内存在虚拟地址之间的映射关系；示意图如下：

![image-20240314225530594](figures/image-20240314225530594.png)

这里的页表为MMUTable，后面直接切换就行了。
