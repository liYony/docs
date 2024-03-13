## sys_open

```c
int sys_open(const char *name, int flag, ...)
{
    lwp_get_from_user(kname, (void *)name, len + 1);
    //  open(kname, flag, 0); start
    //      int fd = fd_new(); start
    //          struct dfs_fdtable *fdt dfs_fdtable_get(); start
    struct dfs_fdtable *fdt = &(struct rt_lwp *)rt_thread_self()->lwp->fdt;
    //          struct dfs_fdtable *fdt dfs_fdtable_get(); end
    //          int fd fdt_fd_new(fdt); start
    //              int fd = fd_alloc(fdt, DFS_STDIO_OFFSET);
    int idx = fd_slot_alloc(fdt, startfd);
    struct dfs_fd *fd = (struct dfs_fd *)rt_calloc(1, sizeof(struct dfs_fd));
    fd->ref_count = 1;
    fd->magic = DFS_FD_MAGIC;
    fd->fnode = NULL;
    fdt->fds[idx] = fd;
    int fd = idx;
    //              int fd = fd_alloc(fdt, DFS_STDIO_OFFSET);
    //          int fd fdt_fd_new(fdt); end
    //      int fd = fd_new(); end
    //      struct dfs_fd *d = fd_get(fd); start
    struct dfs_fd *d = fdt_fd_get(fdt, fd) = fdt->fds[fd];;
    //      struct dfs_fd *d = fd_get(fd); end
    dfs_file_open(d, file, flags);
    //  open(kname, flag, 0); end
}
```

## dfs_fnode_find

### struct dfs_fnode

```c
struct dfs_fnode
{
    uint16_t type;               /* Type (regular or socket) */

    char *path;                  /* Name (below mount point) */
    char *fullpath;              /* Full path is hash key */
    int ref_count;               /* Descriptor reference count */
    rt_list_t list;              /* The node of fnode hash table */

    struct dfs_filesystem *fs;
    const struct dfs_file_ops *fops;
    uint32_t flags;              /* self flags, is dir etc.. */

    size_t   size;               /* Size in bytes */
    void *data;                  /* Specific file system data */
};

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
```

