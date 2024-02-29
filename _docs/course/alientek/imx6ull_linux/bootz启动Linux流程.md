# bootz启动Linux内核

```c
typedef struct bootm_headers {
	/*
	 * Legacy os image header, if it is a multi component image
	 * then boot_get_ramdisk() and get_fdt() will attempt to get
	 * data from second and third component accordingly.
	 */
	image_header_t	*legacy_hdr_os;		/* image header pointer */
	image_header_t	legacy_hdr_os_copy;	/* header copy */
	ulong		legacy_hdr_valid;

#if defined(CONFIG_FIT)
	const char	*fit_uname_cfg;	/* configuration node unit name */

	void		*fit_hdr_os;	/* os FIT image header */
	const char	*fit_uname_os;	/* os subimage node unit name */
	int		fit_noffset_os;	/* os subimage node offset */

	void		*fit_hdr_rd;	/* init ramdisk FIT image header */
	const char	*fit_uname_rd;	/* init ramdisk subimage node unit name */
	int		fit_noffset_rd;	/* init ramdisk subimage node offset */

	void		*fit_hdr_fdt;	/* FDT blob FIT image header */
	const char	*fit_uname_fdt;	/* FDT blob subimage node unit name */
	int		fit_noffset_fdt;/* FDT blob subimage node offset */

	void		*fit_hdr_setup;	/* x86 setup FIT image header */
	const char	*fit_uname_setup; /* x86 setup subimage node name */
	int		fit_noffset_setup;/* x86 setup subimage node offset */
#endif

#ifndef USE_HOSTCC
	image_info_t	os;		        /* os image info */
	ulong		ep;		            /* entry point of OS */

ulong		rd_start, rd_end;       /* ramdisk start/end */

	char		*ft_addr;	        /* flat dev tree address */
	ulong		ft_len;		        /* length of flat device tree */

	ulong		initrd_start;
	ulong		initrd_end;
	ulong		cmdline_start;
	ulong		cmdline_end;
	bd_t		*kbd;
#endif

	int		verify;		/* getenv("verify")[0] != 'n' */

    // 下面宏定义表示BOOT的不同阶段
#define	BOOTM_STATE_START	(0x00000001)
#define	BOOTM_STATE_FINDOS	(0x00000002)
#define	BOOTM_STATE_FINDOTHER	(0x00000004)
#define	BOOTM_STATE_LOADOS	(0x00000008)
#define	BOOTM_STATE_RAMDISK	(0x00000010)
#define	BOOTM_STATE_FDT		(0x00000020)
#define	BOOTM_STATE_OS_CMDLINE	(0x00000040)
#define	BOOTM_STATE_OS_BD_T	(0x00000080)
#define	BOOTM_STATE_OS_PREP	(0x00000100)
#define	BOOTM_STATE_OS_FAKE_GO	(0x00000200)	/* 'Almost' run the OS */
#define	BOOTM_STATE_OS_GO	(0x00000400)
	int		state;

#ifdef CONFIG_LMB
	struct lmb	lmb;		/* for memory mgmt */
#endif
} bootm_headers_t;

extern bootm_headers_t images;
```

```c
typedef struct image_info {
	ulong		start, end;		        /* start/end of blob */
	ulong		image_start, image_len; /* start of image within blob, len of image */
	ulong		load;			        /* load addr for the image */
	uint8_t		comp, type, os;		    /* compression, type of image, os type */
	uint8_t		arch;			        /* CPU architecture */
} image_info_t;
```

```c
int do_bootz(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
	int ret;

	/* Consume 'bootz' */
	argc--; argv++;

    // 主要用于初始化images的相关成员变量。
	if (bootz_start(cmdtp, flag, argc, argv, &images))
		return 1;

	/*
	 * We are doing the BOOTM_STATE_LOADOS state ourselves, so must
	 * disable interrupts ourselves
	 */
    // 关闭中断
	bootm_disable_interrupts();

	images.os.os = IH_OS_LINUX; // 表示我们要启动的镜像类型为Linux
    // 来执行不同的BOOT阶段，包括：
    //      - BOOTM_STATE_OS_PREP
    //      - BOOTM_STATE_OS_FAKE_GO
    //      - BOOTM_STATE_OS_GO
	ret = do_bootm_states(cmdtp, flag, argc, argv,
			      BOOTM_STATE_OS_PREP | BOOTM_STATE_OS_FAKE_GO |
			      BOOTM_STATE_OS_GO,
			      &images, 1);

	return ret;
}
```

```c
static int bootz_start(cmd_tbl_t *cmdtp, int flag, int argc,
			char * const argv[], bootm_headers_t *images)
{
	int ret;
	ulong zi_start, zi_end;

    // 执行BOOT的BOOTM_STATE_START阶段
	ret = do_bootm_states(cmdtp, flag, argc, argv, BOOTM_STATE_START,
			      images, 1);

	/* Setup Linux kernel zImage entry point */
	if (!argc) {
		images->ep = load_addr;
		debug("*  kernel: default image load address = 0x%08lx\n",
				load_addr);
	} else {
        // 设置images的ep成员变量，也就是系统镜像的入口点，使用bootz命令启动系统的时候就会设置系统在DRAM中的存储位置。
        // 这个存储位置就是系统镜像的入口点，也就是说images->ep=0x80800000。
		images->ep = simple_strtoul(argv[0], NULL, 16);
		debug("*  kernel: cmdline image address = 0x%08lx\n",
			images->ep);
	}

    // 此函数会判断之前的系统镜像是否为Linux镜像文件，并且会打印处镜像的相关信息。
	ret = bootz_setup(images->ep, &zi_start, &zi_end);
	if (ret != 0)
		return 1;

	lmb_reserve(&images->lmb, images->ep, zi_end - zi_start);

	/*
	 * Handle the BOOTM_STATE_FINDOTHER state ourselves as we do not
	 * have a header that provide this informaiton.
	 */
    // 查找ramdisk和设备树(dtb)文件，但是我们没有用到ramdisk，因此此函数在这里仅仅用于查找设备树(dtb)文件。
	if (bootm_find_images(flag, argc, argv))
		return 1;

#ifdef CONFIG_SECURE_BOOT
	extern uint32_t authenticate_image(
			uint32_t ddr_start, uint32_t image_size);
	if (authenticate_image(images->ep, zi_end - zi_start) == 0) {
		printf("Authenticate zImage Fail, Please check\n");
		return 1;
	}
#endif
	return 0;
}
```

```c
#define	LINUX_ARM_ZIMAGE_MAGIC	0x016f2818

int bootz_setup(ulong image, ulong *start, ulong *end)
{
	struct zimage_header *zi;

    // 获取image头部的magic，用于校验Linux镜像
	zi = (struct zimage_header *)map_sysmem(image, 0);
	if (zi->zi_magic != LINUX_ARM_ZIMAGE_MAGIC) {
		puts("Bad Linux ARM zImage magic!\n");
		return 1;
	}

	*start = zi->zi_start;
	*end = zi->zi_end;

	printf("Kernel image @ %#08lx [ %#08lx - %#08lx ]\n", image, *start,
	      *end);

	return 0;
}
```

```c
int bootm_find_images(int flag, int argc, char * const argv[])
{
	int ret;

	/* find ramdisk */
	ret = boot_get_ramdisk(argc, argv, &images, IH_INITRD_ARCH,
			       &images.rd_start, &images.rd_end);
	if (ret) {
		puts("Ramdisk image is corrupt or invalid\n");
		return 1;
	}

#if defined(CONFIG_OF_LIBFDT)
	/* find flattened device tree */
	ret = boot_get_fdt(flag, argc, argv, IH_ARCH_DEFAULT, &images,
			   &images.ft_addr, &images.ft_len);
	if (ret) {
		puts("Could not find a valid device tree\n");
		return 1;
	}
	set_working_fdt_addr((ulong)images.ft_addr);
#endif

#if defined(CONFIG_FIT)
	/* find all of the loadables */
	ret = boot_get_loadable(argc, argv, &images, IH_ARCH_DEFAULT,
			       NULL, NULL);
	if (ret) {
		printf("Loadable(s) is corrupt or invalid\n");
		return 1;
	}
#endif

	return 0;
}
```