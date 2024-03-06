# RT-Smart应用程序(.elf)文件加载

[RT-Thread-RT-Smart ELF 应用程序加载运行过程分析RT-Thread问答社区 - RT-Thread](https://club.rt-thread.org/ask/article/8fd18929073be592.html)

以下面hello.elf为例：

```shell
msh /root_demo/bin>hello.elf
```

## 1 msh线程处理

首先RT-Thread的MSH首先会想办法得到"hello.elf"字符串，然后执行msh_exec函数，传入参数即跟得到的"hello.elf"字符串相关：

```c
void finsh_thread_entry(void *parameter)
{
    ...
    while(1)
    {
        ... // 获取
        msh_exec()
    }
}
```

## 2 msh_exec

其中msh_exec函数主要是为了解析"hello.elf"字符串：

```c
int msh_exec(char *cmd, rt_size_t length)
{
    ...
#ifdef RT_USING_LWP
    /* exec from msh_exec , debug = 0*/
    /* _msh_exec_lwp return is pid , <= 0 means failed */
    if (_msh_exec_lwp(0, cmd, length) > 0)
    {
        return 0;
    }
#endif
    ...
}
```

## 3 _msh_exec_lwp

```c
int _msh_exec_lwp(int debug, char *cmd, rt_size_t length)
{
    int argc;
    int cmd0_size = 0;
    char *argv[FINSH_ARG_MAX];
    char *pg_name;
    int ret;
    rt_kprintf("%s(%s<%d>)\r\n", __func__, cmd, length);
    /* find the size of first command */
    while ((cmd[cmd0_size] != ' ' && cmd[cmd0_size] != '\t') && cmd0_size < length)
        cmd0_size ++;
    if (cmd0_size == 0)
        return -1;

    /* split arguments */
    rt_memset(argv, 0x00, sizeof(argv));
    argc = msh_split(cmd, length, argv);
    if (argc == 0)
        return -1;

    /* try to find program in working directory */
    pg_name = _msh_exec_search_path("", argv[0]);
    if (pg_name)
    {
        goto found_program;
    }

    /* only check these paths when the first argument doesn't contain path 
       seperator */
    if (strstr(argv[0], "/"))
    {
        return -1;
    }

    /* try to find program in /bin */
    pg_name = _msh_exec_search_path("/bin", argv[0]);
    if (pg_name)
    {
        goto found_program;
    }

    /* try to find program in dirs registered to env path */
    pg_name = _msh_exec_search_env(argv[0]);
    if (pg_name)
    {
        goto found_program;
    }

    /* not found in anywhere */
    return -1;

    /* found program */
found_program:
    ret = exec(pg_name, debug, argc, argv);
    rt_free(pg_name);

    return ret;
}
```

主要流程如下：

- 解析出可执行文件和相关参数。
- 在当前路径下查找是否有可执行文件(可包含路径)。
  - 能找到：
    - 直接执行exec函数。
  - 不能找到：
    - 在"/bin"目录下查找可执行文件(不能包含路径"/")。
    - 在环境变量PATH里面查找可执行文件(不能包含路径"/")。
    - 直接执行exec函数。

## 4 exec

exec主要是调用lwp_execve函数：

```c
pid_t exec(char *filename, int debug, int argc, char **argv)
{
    return lwp_execve(filename, debug, argc, argv, __environ);
}
```

其中有几个参数介绍一下：

- filename：程序的名字。
- debug：运行在msh_exec 的时候，debug = 0。
- argc：参数个数。
- argv：参数内容。
- __environ：环境变量。

```c
pid_t lwp_execve(char *filename, int debug, int argc, char **argv, char **envp)
{
    int result;
    rt_base_t level;
    struct rt_lwp *lwp;
    char *thread_name;
    char *argv_last = argv[argc - 1];
    int bg = 0;
    struct process_aux *aux;
    int tid = 0;
    rt_kprintf("%s\r\n", filename);
    if (filename == RT_NULL)
    {
        return -RT_ERROR;
    }
    // 分配lwp内存并且初始化(重要的是进程ID pid)
    lwp = lwp_new();

    ...
    // 获取线程ID tid
    if ((tid = lwp_tid_get()) == 0)
    {
        lwp_ref_dec(lwp);
        return -ENOMEM;
    }
#ifdef RT_USING_USERSPACE
    // 用户空间初始化，主要实现功能如下：
    // 1. 分配4M的应用程序mmu_table
    // 2. 复制系统初始化创建的MMU_Table到应用程序的mmu_table
    // 3. 清楚应用程序空间的mmu_table，也就是前三个page 3M。
    // 4. 初始化lwp->mmu_info:
    //    a. lwp->mmu_info->vtable = mmu_table;
    //    b. lwp->mmu_info->vstart = USER_VADDR_START = 0x00100000UL;
    //    c. lwp->mmu_info->vend = USER_VADDR_TOP = 0xC0000000UL;
    //    d. lwp->mmu_info->pv_off = PV_OFFSET;
    if (lwp_user_space_init(lwp) != 0)
    {
        lwp_tid_put(tid);
        lwp_ref_dec(lwp);
        return -ENOMEM;
    }
#endif
    
    // 判断是否是后台运行，如果是，参数应该减少一个，并且设置bg=1
    if (argv_last[0] == '&' && argv_last[1] == '\0')
    {
        argc--;
        bg = 1;
    }
    
    // 这个函数为给定的轻量级进程复制了参数和环境变量，并将它们组织成一个特定的内存布局，这通常用于在进程创建时传递给新的进程。
    // 这样的布局允许新的进程在不需要重新分配内存的情况下直接访问这些参数和环境变量。
    // 保存在lwp->args里面。
    if ((aux = lwp_argscopy(lwp, argc, argv, envp)) == RT_NULL)
    {
        lwp_tid_put(tid);
        lwp_ref_dec(lwp);
        return -ENOMEM;
    }

    result = lwp_load(filename, lwp, RT_NULL, 0, aux);
#ifdef ARCH_MM_MMU
    if (result == 1)
    {
        /* dynmaic */
        lwp_unmap_user(lwp, (void *)(USER_VADDR_TOP - ARCH_PAGE_SIZE));
        result = load_ldso(lwp, filename, argv, envp);
    }
#endif /* ARCH_MM_MMU */
    if (result == RT_EOK)
    {
        rt_thread_t thread = RT_NULL;
        rt_uint32_t priority = 25, tick = 200;

        lwp_copy_stdio_fdt(lwp);

        /* obtain the base name */
        thread_name = strrchr(filename, '/');
        thread_name = thread_name ? thread_name + 1 : filename;
#ifndef ARCH_MM_MMU
        struct lwp_app_head *app_head = lwp->text_entry;
        if (app_head->priority)
        {
            priority = app_head->priority;
        }
        if (app_head->tick)
        {
            tick = app_head->tick;
        }
#endif /* not defined ARCH_MM_MMU */
        thread = rt_thread_create(thread_name, lwp_thread_entry, RT_NULL,
                LWP_TASK_STACK_SIZE, priority, tick);
        if (thread != RT_NULL)
        {
            struct rt_lwp *self_lwp;

            thread->tid = tid;
            lwp_tid_set_thread(tid, thread);
            LOG_D("lwp kernel => (0x%08x, 0x%08x)\n", (rt_uint32_t)thread->stack_addr,
                    (rt_uint32_t)thread->stack_addr + thread->stack_size);
            level = rt_hw_interrupt_disable();
            self_lwp = lwp_self();
            if (self_lwp)
            {
                //lwp->tgroup_leader = &thread; //add thread group leader for lwp
                lwp->__pgrp = tid;
                lwp->session = self_lwp->session;
                /* lwp add to children link */
                lwp->sibling = self_lwp->first_child;
                self_lwp->first_child = lwp;
                lwp->parent = self_lwp;
            }
            else
            {
                //lwp->tgroup_leader = &thread; //add thread group leader for lwp
                lwp->__pgrp = tid;
            }
            if (!bg)
            {
                if (lwp->session == -1)
                {
                    struct tty_struct *tty = RT_NULL;
                    tty = (struct tty_struct *)console_tty_get();
                    lwp->tty = tty;
                    lwp->tty->pgrp = lwp->__pgrp;
                    lwp->tty->session = lwp->session;
                    lwp->tty->foreground = lwp;
                    tcgetattr(1, &stdin_termios);
                    old_stdin_termios = stdin_termios;
                    stdin_termios.c_lflag |= ICANON | ECHO | ECHOCTL;
                    tcsetattr(1, 0, &stdin_termios);
                }
                else
                {
                    if (self_lwp != RT_NULL)
                    {
                        lwp->tty = self_lwp->tty;
                        lwp->tty->pgrp = lwp->__pgrp;
                        lwp->tty->session = lwp->session;
                        lwp->tty->foreground = lwp;
                    }
                    else
                    {
                        lwp->tty = RT_NULL;
                    }

                }
            }
            thread->lwp = lwp;
#ifndef ARCH_MM_MMU
            struct lwp_app_head *app_head = (struct lwp_app_head*)lwp->text_entry;
            thread->user_stack = app_head->stack_offset ?
                              (void *)(app_head->stack_offset -
                                       app_head->data_offset +
                                       (uint32_t)lwp->data_entry) : RT_NULL;
            thread->user_stack_size = app_head->stack_size;
            /* init data area */
            rt_memset(lwp->data_entry, 0, lwp->data_size);
            /* init user stack */
            rt_memset(thread->user_stack, '#', thread->user_stack_size);
#endif /* not defined ARCH_MM_MMU */
            rt_list_insert_after(&lwp->t_grp, &thread->sibling);

            if (debug && rt_dbg_ops)
            {
                lwp->debug = debug;
                rt_thread_control(thread, RT_THREAD_CTRL_BIND_CPU, (void*)0);
            }
            rt_hw_interrupt_enable(level);

            rt_thread_startup(thread);
            return lwp_to_pid(lwp);
        }
    }

    lwp_tid_put(tid);
    lwp_ref_dec(lwp);

    return -RT_ERROR;
}
```

### 4.1 lwp_user_space_init

```c
#define USER_VADDR_TOP    0xC0000000UL
#define USER_HEAP_VEND    0xB0000000UL
#define USER_HEAP_VADDR   0x80000000UL
#define USER_STACK_VSTART 0x70000000UL
#define USER_STACK_VEND   USER_HEAP_VADDR
#define LDSO_LOAD_VADDR   0x60000000UL
#define USER_VADDR_START  0x00100000UL
#define USER_LOAD_VADDR   USER_VADDR_START

int lwp_user_space_init(struct rt_lwp *lwp)
{
    return arch_user_space_init(lwp);
}

int arch_user_space_init(struct rt_lwp *lwp)
{
    size_t *mmu_table;

    mmu_table = (size_t*)rt_pages_alloc(2);
    if (!mmu_table)
    {
        return -1;
    }

    lwp->end_heap = USER_HEAP_VADDR;
    rt_memcpy(mmu_table + (KERNEL_VADDR_START >> ARCH_SECTION_SHIFT), MMUTable + (KERNEL_VADDR_START >> ARCH_SECTION_SHIFT), ARCH_PAGE_SIZE);
    rt_memset(mmu_table, 0, 3 * ARCH_PAGE_SIZE);
    rt_hw_cpu_dcache_ops(RT_HW_CACHE_FLUSH, mmu_table, 4 * ARCH_PAGE_SIZE);
    rt_hw_mmu_map_init(&lwp->mmu_info, (void*)USER_VADDR_START, USER_VADDR_TOP - USER_VADDR_START, mmu_table, PV_OFFSET);

    return 0;
}
```

### 4.2 lwp_argscopy

```c
static struct process_aux *lwp_argscopy(struct rt_lwp *lwp, int argc, char **argv, char **envp)
{
    int size = sizeof(int) * 5; /* store argc, argv, envp, aux, NULL */
    struct process_aux *aux;

    int *args;
    char *str;
    char **new_argve;
    int i;
    int len;

    // 代码首先计算了存储所有参数、环境变量和辅助结构体process_aux所需的内存大小。
    // 这包括每个字符串的长度、指向这些字符串的整数指针以及process_aux结构体本身的大小。
    for (i = 0; i < argc; i++)
    {
        size += (rt_strlen(argv[i]) + 1);
    }
    size += (sizeof(int) * argc);

    i = 0;
    if (envp)
    {
        while (envp[i] != 0)
        {
            size += (rt_strlen(envp[i]) + 1);
            size += sizeof(int);
            i++;
        }
    }

    /* for aux */
    size += sizeof(struct process_aux);

    args = (int *)rt_malloc(size);
    if (args == RT_NULL)
    {
        return RT_NULL;
    }

    /* argc, argv[], 0, envp[], 0 */
    str = (char *)((size_t)args + (argc + 2 + i + 1 + AUX_ARRAY_ITEMS_NR * 2 + 1) * sizeof(int));


    new_argve = (char **)&args[1];
    args[0] = argc;

    for (i = 0; i < argc; i++)
    {
        len = rt_strlen(argv[i]) + 1;
        new_argve[i] = str;
        rt_memcpy(str, argv[i], len);
        str += len;
    }
    new_argve[i] = 0;
    i++;

    new_argve[i] = 0;
    if (envp)
    {
        int j;
        for (j = 0; envp[j] != 0; j++)
        {
            len = rt_strlen(envp[j]) + 1;
            new_argve[i] = str;
            rt_memcpy(str, envp[j], len);
            str += len;
            i++;
        }
        new_argve[i] = 0;
    }
    /* aux */
    aux = (struct process_aux *)(new_argve + i);
    aux->item[0].key = AT_EXECFN;
    aux->item[0].value = (uint32_t)(size_t)new_argve[0];
    i += AUX_ARRAY_ITEMS_NR * 2;
    new_argve[i] = 0;

    lwp->args = args;

    return aux;
}
```

### 4.3 lwp_load

```c
RT_WEAK int lwp_load(const char *filename, struct rt_lwp *lwp, uint8_t *load_addr, size_t addr_size, struct process_aux *aux)
{
    uint8_t *ptr;
    int ret = -1;
    int len;
    int fd = -1;

    /* check file name */
    RT_ASSERT(filename != RT_NULL);
    /* check lwp control block */
    RT_ASSERT(lwp != RT_NULL);

    /* copy file name to process name */
    rt_strncpy(lwp->cmd, filename, RT_NAME_MAX);

    if (load_addr != RT_NULL)
    {
        lwp->lwp_type = LWP_TYPE_FIX_ADDR;
        ptr = load_addr;
    }
    else
    {
        lwp->lwp_type = LWP_TYPE_DYN_ADDR;
        ptr = RT_NULL;
    }

    fd = open(filename, O_BINARY | O_RDONLY, 0);
    if (fd < 0)
    {
        LOG_E("ERROR: Can't open elf file %s!", filename);
        goto out;
    }
    len = lseek(fd, 0, SEEK_END);
    if (len < 0)
    {
        LOG_E("ERROR: File %s size error!", filename);
        goto out;
    }

    lseek(fd, 0, SEEK_SET);

    ret = load_elf(fd, len, lwp, ptr, aux);
    if ((ret != RT_EOK) && (ret != 1))
    {
        LOG_E("lwp load ret = %d", ret);
    }

out:
    if (fd > 0)
    {
        close(fd);
    }
    return ret;
}
```

#### 4.3.1 load_elf

```c
```

