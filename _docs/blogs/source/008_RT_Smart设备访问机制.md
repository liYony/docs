# RT-Smart 设备访问机制

## 1 示例程序

### 1.1 用户程序

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

/* Defines the event in which the key is pressed */
#define EVENT_KEY_PRESS   10
#define KEY0_DEVICE_PATH  "/dev/key0"

int main(void)
{
    int fd;
    int value;

    /* Open the device, default by blocking mode */
    fd = open(KEY0_DEVICE_PATH, O_RDONLY);
    if (fd < 0)
    {
        printf("open device failed\n");
        return 0;
    }

    printf("Please press the USER button\n");
    /* read data */
    if(read(fd, &value, 1) == 1)
    {
        if (value == EVENT_KEY_PRESS)
            printf("key press\n");
    }

    close(fd);

    printf("The program will exit\n");

    return 0;
}
```

### 1.2 驱动程序

```c
#include <rtthread.h>
#include <rtdevice.h>
#include <board.h>
#include <poll.h>
#include <dfs_file.h>

#define GET_PIN(PORTx, PIN) (32 * (PORTx - 1) + (PIN & 31))
#define USER_KEY GET_PIN(7, 13) // PG13

static uint8_t is_init;
static uint8_t key_state;
static uint8_t key_state_old;
static rt_device_t device;

void irq_callback()
{
    /* enter interrupt */
    key_state = rt_pin_read(USER_KEY);
    rt_interrupt_enter();
    rt_wqueue_wakeup(&(device->wait_queue), (void *)POLLIN);
    /* leave interrupt */
    rt_interrupt_leave();
}

static void drv_key_init(void)
{
    key_state = 1;
    key_state_old = 1;
    rt_pin_mode(USER_KEY, PIN_MODE_INPUT);
    rt_pin_attach_irq(USER_KEY, PIN_IRQ_MODE_RISING_FALLING, irq_callback, RT_NULL);
    rt_pin_irq_enable(USER_KEY, PIN_IRQ_ENABLE);
}

/* Open the key device, and initialize the hardware the first time you open it */
static int drv_key_open(struct dfs_fd *fd)
{
    if (!is_init)
    {
        is_init = 1;
        /* Initialize the hardware */
        drv_key_init();
    }
    /* Increase reference count */
    device->ref_count ++;
    return 0;
}
/* Close the key device, and reset the hardware when the device is no longer in use */
static int drv_key_close(struct dfs_fd *fd)
{
    /* Reduced reference count */
    device->ref_count --;
    /* Reset the hardware when the device is no longer in use */
    if (device->ref_count == 0)
    {
        /* ... */
        is_init = 0;
    }
    return 0;
}

/* Read key state */
static int drv_key_read(struct dfs_fd *fd, void *buf, size_t count)
{
    *(int *)buf = !rt_pin_read(USER_KEY);
    return 1;
}

/* Use poll to check the state of the key */
static int drv_key_poll(struct dfs_fd *fd, struct rt_pollreq *req)
{
    int mask = 0;
    int flags = 0;

    /* only support POLLIN */
    flags = fd->flags & O_ACCMODE;
    if (flags == O_RDONLY || flags == O_RDWR)
    {
        /* Add to wait queue, suspend the current thread */
        rt_poll_add(&(device->wait_queue), req);
        /* If the key is pressed, mark a POLLIN event */
        if (key_state != key_state_old)
        {
            key_state_old = key_state;
            mask |= POLLIN;
        }
    }
    return mask;
}

/*
 * Realize the fops variables.
 */
static struct dfs_file_ops drv_key_fops =
{
    drv_key_open,
    drv_key_close,
    RT_NULL,
    drv_key_read,
    RT_NULL,
    RT_NULL,
    RT_NULL,
    RT_NULL,
    drv_key_poll,
};

/*
 * Key device initialization function.
 */
static int rt_hw_key_init(void)
{
    rt_err_t ret;

    /* 1. Allocates memory for device structures, Use the calloc function */
    device = rt_calloc(1, sizeof(struct rt_device));
    if (device == RT_NULL)
    {
        return -1;
    }

    /* 2. Set to miscellaneous device */
    device->type = RT_Device_Class_Miscellaneous;

    /* 3. register a key device */
    ret = rt_device_register(device, "key0", RT_DEVICE_FLAG_RDONLY);
    if (ret != RT_EOK)
    {
        rt_free(device);
        return ret;
    }
    /* 4. set fops */
    device->fops = &drv_key_fops;

    return ret;
}
/* Using below macro to export this function, the function will be called automatically after kernel startup */
INIT_DEVICE_EXPORT(rt_hw_key_init);
```

## 2 系统调用

RT-Thread Smart 的用户态是固定地址方式运行，当需要系统服务时通过系统调用的方式陷入到内核中。

### 2.1 系统调用号

在应用层我们看到的是open()，read()，write()等由C库封装好的接口，这些接口分别对应了RT-Smart内核中sys_open()，sys_read()，sys_write()等函数。对于应用层的open，read，write等系统调用，在ArmV7架构下，都是通过触发软中断来进入内核态的。至于C库函数是如何调用软中断这里先不管，这些不同的C库函数只是中断调用号不相同而已。至于RT-Smart的中断号描述，可以参考`userapps/sdk/rt-thread/include/sys/syscall_no.h`文件：

```c
NRSYS(exit)            /* 01 */
NRSYS(read)
NRSYS(write)
NRSYS(lseek)
NRSYS(open)            /* 05 */
NRSYS(close)
NRSYS(ioctl)
NRSYS(fstat)
....
```

### 2.2 中断向量表

```assembly
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
    ldr pc, _vector_irq
    ldr pc, _vector_fiq

...

_vector_swi:
    .word vector_swi
```

### 2.3 软中断

当C库函数执行swi指令后会自动的跳到内核设置好的中断向量表_vector_swi处执行中断处理，也就是vector_swi函数。

```assembly
/*
 * void SVC_Handler(void);
 */
.global vector_swi
.type vector_swi, % function
vector_swi:
    push {lr}
    mrs lr, spsr
    push {r4, r5, lr}

    cpsie i

    push {r0 - r3, r12}

    bl rt_thread_self
    bl lwp_user_setting_save

    and r0, r7, #0xf000
    cmp r0, #0xe000
    beq lwp_signal_quit

    cmp r0, #0xf000
    beq ret_from_user
    and r0, r7, #0xff
    bl lwp_get_sys_api
    cmp r0, #0           /* r0 = api */
    mov lr, r0

    pop {r0 - r3, r12}
    beq svc_exit
    blx lr

svc_exit:
    cpsid i
    pop {r4, r5, lr}
    msr spsr_cxsf, lr
    pop {lr}
```

简单分析这个汇编函数：

24~25行：可以看出中断调用号是保存在r7寄存器里面的。

26~31行：判断lwp_get_sys_api函数根据中断调用号得出的系统调用函数是否为空，不为空则跳转到该函数执行。lwp_get_sys_api的实现如下：

```c
const static void* func_table[] =
{
    (void *)sys_exit,            /* 01 */
    (void *)sys_read,
    (void *)sys_write,
    (void *)sys_lseek,
    (void *)sys_open,            /* 05 */
    (void *)sys_close,
    ....
}

const void *lwp_get_sys_api(rt_uint32_t number)
{
    const void *func = (const void *)sys_notimpl;

    if (number == 0xff)
    {
        func = (void *)sys_log;
    }
    else
    {
        number -= 1;
        if (number < sizeof(func_table) / sizeof(func_table[0]))
        {
            func = func_table[number];
        }
    }

    return func;
}
```

这样就完成了用户空间到内核空间的切换。使用户态的open函数最终调用到内核的sys_open函数，用户态的write函数最终调用到内核的sys_write函数，以此类推。

## 3 sys_open函数实现

```c
int sys_open(const char *name, int flag, ...)
{
    int ret = -1;
    rt_size_t len = 0;
    char *kname = RT_NULL;

    if (!lwp_user_accessable((void *)name, 1))
    {
        return -EFAULT;
    }

    len = rt_strlen(name);
    if (!len)
    {
        return -EINVAL;
    }

    kname = (char *)kmem_get(len + 1);
    if (!kname)
    {
        return -ENOMEM;
    }

    lwp_get_from_user(kname, (void *)name, len + 1);
    ret = open(kname, flag, 0);

    kmem_put(kname);
    return (ret < 0 ? GET_ERRNO() : ret);
}
```

07~16行：做一系列检查，比如检查name是否在用户空间，检查name的长度是否合法等。

18~27行：从用户空间拷贝name到内核空间，并调用open函数打开文件；最后释放name在内核的空间并将打开文件的返回值返回给用户空间。

### 3.1 open函数实现

open函数的实现如下：

```c
int open(const char *file, int flags, ...)
{
    int fd, result;
    struct dfs_fd *d;

    /* allocate a fd */
    fd = fd_new();
    if (fd < 0)
    {
        rt_set_errno(-ENOMEM);

        return -1;
    }
    d = fd_get(fd);

    result = dfs_file_open(d, file, flags);
    if (result < 0)
    {
        /* release the ref-count of fd */
        fd_release(fd);

        rt_set_errno(result);

        return -1;
    }

    return fd;
}
```

函数首先调用fd_new函数分配一个fd，然后调用dfs_file_open函数打开文件。fd_new函数这里就不看代码了，它的主要实现逻辑是：从进程的fdtable中查找一个可用的fd，如果不能查找到可用的fd，将会扩展fdtable，然后再返回可以的fd；成功返回fd后也就可以分配一个文件描述信息(struct dfs_fd)，并且将分配的文件描述信息添加到进程的fdtable中。这样就将fd和文件描述信息(struct dfs_fd)关联起来了。

### 3.2 dfs_file_open详解

```c
int dfs_file_open(struct dfs_fd *fd, const char *path, int flags)
{
    struct dfs_filesystem *fs;
    char *fullpath;
    int result;
    struct dfs_fnode *fnode = NULL;
    rt_list_t *hash_head;

    /* parameter check */
    if (fd == NULL)
        return -EINVAL;

    /* make sure we have an absolute path */
    fullpath = dfs_normalize_path(NULL, path);
    if (fullpath == NULL)
    {
        return -ENOMEM;
    }

    LOG_D("open file:%s", fullpath);

    dfs_fm_lock();
    /* fnode find */
    fnode = dfs_fnode_find(fullpath, &hash_head);
    if (fnode)
    {
        fnode->ref_count++;
        fd->pos   = 0;
        fd->fnode = fnode;
        dfs_fm_unlock();
        rt_free(fullpath); /* release path */
    }
    else
    {
        /* find filesystem */
        fs = dfs_filesystem_lookup(fullpath);
        if (fs == NULL)
        {
            dfs_fm_unlock();
            rt_free(fullpath); /* release path */
            return -ENOENT;
        }

        fnode = rt_calloc(1, sizeof(struct dfs_fnode));
        if (!fnode)
        {
            dfs_fm_unlock();
            rt_free(fullpath); /* release path */
            return -ENOMEM;
        }
        fnode->ref_count = 1;

        LOG_D("open in filesystem:%s", fs->ops->name);
        fnode->fs    = fs;             /* set file system */
        fnode->fops  = fs->ops->fops;  /* set file ops */

        /* initialize the fd item */
        fnode->type  = FT_REGULAR;
        fnode->flags = 0;

        if (!(fs->ops->flags & DFS_FS_FLAG_FULLPATH))
        {
            if (dfs_subdir(fs->path, fullpath) == NULL)
                fnode->path = rt_strdup("/");
            else
                fnode->path = rt_strdup(dfs_subdir(fs->path, fullpath));
            LOG_D("Actual file path: %s", fnode->path);
        }
        else
        {
            fnode->path = fullpath;
        }
        fnode->fullpath = fullpath;

        /* specific file system open routine */
        if (fnode->fops->open == NULL)
        {
            dfs_fm_unlock();
            /* clear fd */
            if (fnode->path != fnode->fullpath)
            {
                rt_free(fnode->fullpath);
            }
            rt_free(fnode->path);
            rt_free(fnode);

            return -ENOSYS;
        }

        fd->pos   = 0;
        fd->fnode = fnode;

        /* insert fnode to hash */
        rt_list_insert_after(hash_head, &fnode->list);
    }

    fd->flags = flags;

    if ((result = fnode->fops->open(fd)) < 0)
    {
        fnode->ref_count--;
        if (fnode->ref_count == 0)
        {
            /* remove from hash */
            rt_list_remove(&fnode->list);
            /* clear fd */
            if (fnode->path != fnode->fullpath)
            {
                rt_free(fnode->fullpath);
            }
            rt_free(fnode->path);
            fd->fnode = NULL;
            rt_free(fnode);
        }

        dfs_fm_unlock();
        LOG_D("%s open failed", fullpath);

        return result;
    }

    fd->flags |= DFS_F_OPEN;
    if (flags & O_DIRECTORY)
    {
        fd->fnode->type = FT_DIRECTORY;
        fd->flags |= DFS_F_DIRECTORY;
    }
    dfs_fm_unlock();

    LOG_D("open successful");
    return 0;
}
```

024行：对于一个从没打开过的设备，dfs_fnode_find的调用返回结果肯定是NULL。所以会执行else分支。

036行：根据路径的名字来判断是什么文件系统，根据filesystem_table[DFS_FILESYSTEMS_MAX]里面保存的信息来查找(自己的调试信息)：

| index | device  | dev_id(rt_device_find(device)) | name  | path |
| ----- | ------- | ------------------------------ | ----- | ---- |
| 0     | virtual | 0                              | devfs | /dev |
| 1     | sd0     | 0xC0148E74                     | elm   | /    |
| 2     | -       | 0                              | null  | null |
| 3     | -       | 0                              | null  | null |

所以根据打开的设备名字/dev/key0，可以判断出来文件系统为devfs，并且将相关信息保存在struct dfs_filesystem *fs成员里面：

```c
struct dfs_file_ops
{
    int (*open)     (struct dfs_fd *fd);
    int (*close)    (struct dfs_fd *fd);
    int (*ioctl)    (struct dfs_fd *fd, int cmd, void *args);
    int (*read)     (struct dfs_fd *fd, void *buf, size_t count);
    int (*write)    (struct dfs_fd *fd, const void *buf, size_t count);
    int (*flush)    (struct dfs_fd *fd);
    int (*lseek)    (struct dfs_fd *fd, off_t offset);
    int (*getdents) (struct dfs_fd *fd, struct dirent *dirp, uint32_t count);

    int (*poll)     (struct dfs_fd *fd, struct rt_pollreq *req);
};

struct dfs_filesystem_ops
{
    char *name;
    uint32_t flags;      /* flags for file system operations */

    /* operations for file */
    const struct dfs_file_ops *fops;

    /* mount and unmount file system */
    int (*mount)    (struct dfs_filesystem *fs, unsigned long rwflag, const void *data);
    int (*unmount)  (struct dfs_filesystem *fs);

    /* make a file system */
    int (*mkfs)     (rt_device_t dev_id, const char *fs_name);
    int (*statfs)   (struct dfs_filesystem *fs, struct statfs *buf);

    int (*unlink)   (struct dfs_filesystem *fs, const char *pathname);
    int (*stat)     (struct dfs_filesystem *fs, const char *filename, struct stat *buf);
    int (*rename)   (struct dfs_filesystem *fs, const char *oldpath, const char *newpath);
};

struct dfs_filesystem
{
    rt_device_t dev_id;     /* Attached device */

    char *path;             /* File system mount point */
    const struct dfs_filesystem_ops *ops; /* Operations for file system type */

    void *data;             /* Specific file system data */
};
```

至于fs成员里面的成员变量可查看devfs虚拟文件系统的初始化部分代码：

```c
int dfs_init(void)
{
    ...
    {
        extern int devfs_init(void);

        /* if enable devfs, initialize and mount it as soon as possible */
        devfs_init();

        dfs_mount(NULL, "/dev", "devfs", 0, 0);
    }
    ...
}
```

其中devfs_init函数主要是为了注册一个devfs虚拟文件系统。详细代码如下：

```c
static const struct dfs_file_ops _device_fops =
{
    dfs_device_fs_open,
    dfs_device_fs_close,
    dfs_device_fs_ioctl,
    dfs_device_fs_read,
    dfs_device_fs_write,
    RT_NULL,                    /* flush */
    RT_NULL,                    /* lseek */
    dfs_device_fs_getdents,
    dfs_device_fs_poll,
};

static const struct dfs_filesystem_ops _device_fs =
{
    "devfs",
    DFS_FS_FLAG_DEFAULT,
    &_device_fops,

    dfs_device_fs_mount,
    RT_NULL,
    RT_NULL,
    RT_NULL,

    RT_NULL,
    dfs_device_fs_stat,
    RT_NULL,
};

int devfs_init(void)
{
    /* register rom file system */
    dfs_register(&_device_fs);

    return 0;
}
```

对于dfs_mount函数代码比较多，这里精简后的代码如下：

```c
int dfs_mount(const char   *device_name,
              const char   *path,
              const char   *filesystemtype,
              unsigned long rwflag,
              const void   *data)
{
    const struct dfs_filesystem_ops **ops;
    struct dfs_filesystem *iter;
    struct dfs_filesystem *fs = NULL;
    char *fullpath = NULL;
    rt_device_t dev_id;

    /* device_name = NULL */
    dev_id = NULL;

    /* ops = &_device_fs */
    for (ops = &filesystem_operation_table[0];
            ops < &filesystem_operation_table[DFS_FILESYSTEM_TYPES_MAX]; ops++)
        if ((*ops != NULL) && (strncmp((*ops)->name, filesystemtype, strlen((*ops)->name)) == 0))
            break;

    /* make full path for special file */
    fullpath = dfs_normalize_path(NULL, path);

    /* Check if the path exists or not, raw APIs call, fixme */
    if ((strcmp(fullpath, "/") != 0) && (strcmp(fullpath, "/dev") != 0))
    {
        struct dfs_fd fd;

        fd_init(&fd);
        if (dfs_file_open(&fd, fullpath, O_RDONLY | O_DIRECTORY) < 0)
        {
            rt_free(fullpath);
            rt_set_errno(-ENOTDIR);

            return -1;
        }
        dfs_file_close(&fd);
    }

    /* check whether the file system mounted or not  in the filesystem table
     * if it is unmounted yet, find out an empty entry */

    for (iter = &filesystem_table[0];
            iter < &filesystem_table[DFS_FILESYSTEMS_MAX]; iter++)
    {
        /* check if it is an empty filesystem table entry? if it is, save fs */
        if (iter->ops == NULL)
            (fs == NULL) ? (fs = iter) : 0;
        /* check if the PATH is mounted */
        else if (strcmp(iter->path, path) == 0)
        {
            rt_set_errno(-EINVAL);
            goto err1;
        }
    }

    /* register file system */
    fs->path   = fullpath;
    fs->ops    = *ops;
    fs->dev_id = dev_id;
    /* For UFS, record the real filesystem name */
    fs->data = (void *) filesystemtype;

    /* call mount of this filesystem */
    (*ops)->mount(fs, rwflag, data);
    return 0;
}
```

通过上面的代码我们大致描述一下dfs_filesystem_lookup获取到的fs成员信息：

```c
fs = &filesystem_table[0];
fs->dev_id = 0;
fs->path = "/dev";
fs->ops = &_device_fs;
fs->data = (void *)"devfs";
fs->ops->name = "devfs";
fs->ops->flags = DFS_FS_FLAG_DEFAULT;
fs->ops->fops = &_device_fops;
fs->ops->mount = dfs_device_fs_mount;
fs->ops->stat = dfs_device_fs_stat;
fs->ops->fops->open = dfs_device_fs_open;
fs->ops->fops->close = dfs_device_fs_close;
fs->ops->fops->ioctl = dfs_device_fs_ioctl;
fs->ops->fops->read = dfs_device_fs_read;
fs->ops->fops->write = dfs_device_fs_write;
fs->ops->fops->getdents = dfs_device_fs_getdents;
fs->ops->fops->poll = dfs_device_fs_poll;
```

044行：为struct dfs_fnode *fnode分配内存。

051~073行：主要是在初始化fnode的内容：

```c
fnode->ref_count = 1;
fnode->fs    = fs = &filesystem_table[0];
fnode->fops  = fs->ops->fops = &_device_fops;
fnode->type  = FT_REGULAR;
fnode->flags = 0;
fnode->path = "/key0";
fnode->fullpath = "/dev/key0";
```

090~091行：初始化struct dfs_fd *fd的成员变量。

```c
struct dfs_fd
{
    uint16_t magic;              /* file descriptor magic number */
    uint32_t flags;              /* Descriptor flags */
    int ref_count;               /* Descriptor reference count */
    off_t    pos;                /* Current file position */
    struct dfs_fnode *fnode;     /* file node struct */
    void *data;                  /* Specific fd data */
};
```

主要初始化内容：

```c
fd->pos   = 0;
fd->fnode = fnode;
```

根据这部分代码可以发现，如果直接通过dfs_fnode_find查找fnode成功也主要是初始化上面的这两个值。

后面的代码就是设置fd->flags，然后执行fnode->fops->open(fd)函数了；如果成功执行就最后在给fd->flags添加一个已经打开的标志。

### 3.3 dfs_device_fs_open详解

前面已经提到将fnode->fops设置为了&_device_fops。

```c
static const struct dfs_file_ops _device_fops =
{
    dfs_device_fs_open,
    dfs_device_fs_close,
    dfs_device_fs_ioctl,
    dfs_device_fs_read,
    dfs_device_fs_write,
    RT_NULL,                    /* flush */
    RT_NULL,                    /* lseek */
    dfs_device_fs_getdents,
    dfs_device_fs_poll,
};
```

当sys_open系统调用运行的时候，最终会调用到fnode->fops->open函数(对于devfs文件系统来说就是dfs_device_fs_open函数)。也正是这个变量让我们用户态的程序最终能调用驱动程序的代码。

```c
int dfs_device_fs_open(struct dfs_fd *file)
{
    rt_err_t result;
    rt_device_t device;
    rt_base_t level;

    RT_ASSERT(file->fnode->ref_count > 0);
    if (file->fnode->ref_count > 1)
    {
        file->pos = 0;
        return 0;
    }
    /* open root directory */
    if ((file->fnode->path[0] == '/') && (file->fnode->path[1] == '\0') &&
        (file->flags & O_DIRECTORY))
    {
        struct rt_object *object;
        struct rt_list_node *node;
        struct rt_object_information *information;
        struct device_dirent *root_dirent;
        rt_uint32_t count = 0;

        /* disable interrupt */
        level = rt_hw_interrupt_disable();

        /* traverse device object */
        information = rt_object_get_information(RT_Object_Class_Device);
        RT_ASSERT(information != RT_NULL);
        for (node = information->object_list.next; node != &(information->object_list); node = node->next)
        {
            count ++;
        }

        root_dirent = (struct device_dirent *)rt_malloc(sizeof(struct device_dirent) +
                      count * sizeof(rt_device_t));
        if (root_dirent != RT_NULL)
        {
            root_dirent->devices = (rt_device_t *)(root_dirent + 1);
            root_dirent->read_index = 0;
            root_dirent->device_count = count;
            count = 0;
            /* get all device node */
            for (node = information->object_list.next; node != &(information->object_list); node = node->next)
            {
                object = rt_list_entry(node, struct rt_object, list);
                root_dirent->devices[count] = (rt_device_t)object;
                count ++;
            }
        }
        rt_hw_interrupt_enable(level);

        /* set data */
        file->fnode->data = root_dirent;

        return RT_EOK;
    }

    device = rt_device_find(&file->fnode->path[1]);
    if (device == RT_NULL)
    {
        return -ENODEV;
    }

#ifdef RT_USING_POSIX
    if (device->fops)
    {
        /* use device fops */
        file->fnode->fops = device->fops;
        file->fnode->data = (void *)device;

        /* use fops */
        if (file->fnode->fops->open)
        {
            result = file->fnode->fops->open(file);
            if (result == RT_EOK || result == -RT_ENOSYS)
            {
                file->fnode->type = FT_DEVICE;
                return 0;
            }
        }
    }
    else
#endif
    {
        result = rt_device_open(device, RT_DEVICE_OFLAG_RDWR);
        if (result == RT_EOK || result == -RT_ENOSYS)
        {
            file->fnode->data = device;
            file->fnode->type = FT_DEVICE;
            return RT_EOK;
        }
    }

    file->fnode->data = RT_NULL;
    /* open device failed. */
    return -EIO;
}
```

13~56行的意思就是用户打开的文件为/dev的时候，需要将所有RT_Object_Class_Device的device信息保存在root_dirent变量里面。然后再将fd->fnode->data赋值为root_dirent。

![image-20240314001134767](figures/image-20240314001134767.png)

这个主要是用于ls这样的命令，比如执行`ls /dev`命令的时候，会有如下的代码：

```c
void ls(const char *pathname)
{
    if (dfs_file_open(&fd, path, O_DIRECTORY) == 0)
    {
            ...
            memset(&dirent, 0, sizeof(struct dirent));
            length = dfs_file_getdents(&fd, &dirent, sizeof(struct dirent));
            ...
    }
}
```

这里dfs_file_open传入的参数path的值也就是/dev，所以目的就是在fd->fnode->data把所有的device数据保存下来，然后可以调用dfs_file_getdents函数去得到每一个device的信息，这样就可以实现ls /dev命令的打印，如：

```shell
msh />ls /dev
Directory /dev:
urandom             0
random              0
zero                0
null                0
ptmx                0
wdt2                0
wdt1                0
winusb              0
usbd                0
gt911               0
emmc1               0
emmc0               0
emmc                0
sd1                 0
sd0                 0
sd                  0
rtc                 0
pwm1                0
lcd                 0
i2c4                0
i2c3                0
e1                  0
console             0
uart0               0
pin                 0
```

实际应用层上不会去打开/dev这个目录，而是具体去打开某一个设备，比如设备/dev/key0。

58行：根据前面的分析，如果时/dev/key0(devfs文件系统中的文件)这样的设备，会将fnode->path设置为/key0(dfs_file_open会将dev文件前缀给去掉)，这里通过rt_device_find(&file->fnode->path[1])函数，就可以完美的对接到rtthread的设备管理框架上面去了，相当于在打开key0这个设备。那么我们就可以通过rt_device_find来查找到key0这个设备。

65~82行：这部分就很简单了，因为我们已经查找到key0这个设备并且保存在局部变量device里面，那我们是不是可以直接调用device->fops->open来调用key0驱动程序的底层open函数(drv_key_open)呢？当然，这样做肯定是没有问题的，只是有个小小的不足，假如系统调用sys_read触发的时候，最终会调用到dfs_device_fs_read，那我们岂不是还需要在dfs_device_fs_read函数中在rt_device_find一下key0这个设备，然后再执行device->fops->read()；显然，这样做的话，效率会比较低。那rtthread的做法是怎么样的呢？

68~69行：这里实现了一个比较粗暴的做法，原先file->fnode->fops = &_device_fops，现在file->fnode->fops = device->fops = &drv_key_fops，这样，在调用file->fnode->fops->open的时候，实际上就是调用了key0驱动程序的底层open函数(drv_key_open)。而当系统调用sys_read触发的时候，最终也将不会调用到dfs_device_fs_read，因为file->fnode->fops = device->fops = &drv_key_fops，而是直接调用到key0驱动程序的底层read函数(drv_key_read)。这样做就省去了一部分中间过程，使得访问底层设备的效率大大提升了。然后在将device保存在file->fnode->data里面。


72~79行：这里就是安全的去调用底层key0驱动程序中的open函数(drv_key_open)了。如果成功打开后，将file->fnode->type设置为FT_DEVICE。

总结一下：

首先每个进程都有一个fdtable成员，里面是所有文件描述信息，每一个成员的索引作为fd(文件描述符)，用户可以依靠这个fd从fdtable中查找到对应的文件描述信息(struct dfs_fd)。在用户进程调用open函数打开/dev/key0这个设备的时候，会通过系统调用最终调用到sys_open函数，sys_open函数会为这个/dev/key0这个设备在fdtable里面分配一个struct dfs_fd结构体的对象，并且将分配成功的对象记录在fdtable->fds中，方便用户使用fd来查找已经分配的struct dfs_fd结构体对象。然后就可以调用dfs_file_open函数来完成打开设备的操作了。详细过程不分析了，这里记录一下进程的一些重要信息。假设打开的/dev/key0设备分配到的fd为3(因为0、1、2文件描述符已经被系统的标准输入、标准输出、标准错误输出占用了)。

```c
struct dfs_fd *fd = (struct dfs_fd *)rt_calloc(1, sizeof(struct dfs_fd));
lwp->fdtable->fds[3] = fd;
lwp->fdtable->fds[3]->fnode = fnode;
lwp->fdtable->fds[3]->fnode->fs = &filesystem_table[0];
lwp->fdtable->fds[3]->fnode->fops  = &drv_key_fops;
```

## 4 sys_read函数实现

```c
ssize_t sys_read(int fd, void *buf, size_t nbyte)
{
    void *kmem = RT_NULL;
    ssize_t ret = -1;

    if (!nbyte)
    {
        return -EINVAL;
    }

    if (!lwp_user_accessable((void *)buf, nbyte))
    {
        return -EFAULT;
    }

    kmem = kmem_get(nbyte);
    if (!kmem)
    {
        return -ENOMEM;
    }

    ret = read(fd, kmem, nbyte);
    if (ret > 0)
    {
        lwp_put_to_user(buf, kmem, ret);
    }

    kmem_put(kmem);
    return (ret < 0 ? GET_ERRNO() : ret);
}
```

这个函数首先分配一块nbyte大小的内存空间，然后调用read函数来读取设备的数据，如果读取成功，则将读取到的数据拷贝到用户空间。核心实现是read()函数和lwp_put_to_user()函数。

### 4.1 read函数实现

函数源码如下：

```c
int read(int fd, void *buf, size_t len)
{
    int result;
    struct dfs_fd *d;

    /* get the fd */
    d = fd_get(fd);
    if (d == NULL)
    {
        rt_set_errno(-EBADF);

        return -1;
    }

    result = dfs_file_read(d, buf, len);
    if (result < 0)
    {
        rt_set_errno(result);

        return -1;
    }

    return result;
}
```

这个函数首先根据fd获取文件描述信息(struct dfs_fd)，然后调用dfs_file_read()函数来读取设备的数据。

### 4.2 dfs_file_read函数分析

```c
int dfs_file_read(struct dfs_fd *fd, void *buf, size_t len)
{
    int result = 0;

    if (fd == NULL)
    {
        return -EINVAL;
    }

    if (fd->fnode->fops->read == NULL)
    {
        return -ENOSYS;
    }

    if ((result = fd->fnode->fops->read(fd, buf, len)) < 0)
    {
        fd->flags |= DFS_F_EOF;
    }

    return result;
}
```

这个函数就不会有dfs_file_open那么复杂了，再dfs_file_open函数中已经建立了fd对应的fnode的基本信息，并且已经将fnode->fops指向了对应的驱动文件操作函数，所以这里直接调用fops->read()函数即可。就能直接调用到drv_key_fops->read()函数了。也就是底层的drv_key_read函数。

