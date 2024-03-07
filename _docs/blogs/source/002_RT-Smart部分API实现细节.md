## 1 lwp_map_user

相关宏定义：

```c
#define ARCH_SECTION_SHIFT  20
#define ARCH_SECTION_SIZE   (1 << ARCH_SECTION_SHIFT)	// 0x00100000 = 1M
#define ARCH_SECTION_MASK   (ARCH_SECTION_SIZE - 1)		// 0x000FFFFF
#define ARCH_PAGE_SHIFT     12
#define ARCH_PAGE_SIZE      (1 << ARCH_PAGE_SHIFT)		// 0x00001000 = 4K
#define ARCH_PAGE_MASK      (ARCH_PAGE_SIZE - 1)		// 0x00000FFF
#define ARCH_PAGE_TBL_SHIFT 10
#define ARCH_PAGE_TBL_SIZE  (1 << ARCH_PAGE_TBL_SHIFT)	// 0x00000400 = 1K
#define ARCH_PAGE_TBL_MASK  (ARCH_PAGE_TBL_SIZE - 1)	// 0x000003FF
```

```c
// map_va = 0xBFD00234, map_size = 0x00100000
void *lwp_map_user(struct rt_lwp *lwp, void *map_va, size_t map_size, int text)
{
    void *ret = RT_NULL;
    size_t offset = 0;

    if (!map_size)
    {
        return 0;
    }

    // 保证代码页对齐
    offset = (size_t)map_va & ARCH_PAGE_MASK;			// offset   = 0xBFD00234 & 0x00000FFF = 0x00000234
    map_size += (offset + ARCH_PAGE_SIZE - 1);			// map_size = 0x00100000 + 0x00000234 + 0x00000FFF = 0x00101233
    map_size &= ~ARCH_PAGE_MASK;						// map_size = 0x00101233 & 0xFFFFF000 = 0x00101000
    map_va = (void *)((size_t)map_va & ~ARCH_PAGE_MASK);// map_va   = 0xBFD00234 & 0xFFFFF000 = 0xBFD00000
    // map_va   = 0xBFD00234 ---> 0xBFD00000
    // map_size = 0x00100000 ---> 0x00101000

    rt_mm_lock();
    ret = _lwp_map_user(lwp, map_va, map_size, text);
    rt_mm_unlock();
    if (ret)
    {
        ret = (void *)((char *)ret + offset);
    }
    return ret;
}
```

### 1.1 _lwp_map_user

```c
// map_va = 0xBFD00000, map_size = 0x00101000
static void *_lwp_map_user(struct rt_lwp *lwp, void *map_va, size_t map_size, int text)
{
    void *va = RT_NULL;
    int ret = 0;
    rt_mmu_info *m_info = &lwp->mmu_info;
    int area_type;

    va = rt_hw_mmu_map_auto(m_info, map_va, map_size, MMU_MAP_U_RWCB);
    if (!va)
    {
        return 0;
    }

    area_type = text ? MM_AREA_TYPE_TEXT : MM_AREA_TYPE_DATA;
    ret = lwp_map_area_insert(&lwp->map_area, (size_t)va, map_size, area_type);
    if (ret != 0)
    {
        unmap_range(lwp, va, map_size, 1);
        return 0;
    }
    return va;
}
```

#### 1.1.1 rt_hw_mmu_map_auto

```c
void *rt_hw_mmu_map_auto(rt_mmu_info *mmu_info, void *v_addr, size_t size, size_t attr)
{
    void *ret;

    rt_mm_lock();
    ret = _rt_hw_mmu_map_auto(mmu_info, v_addr, size, attr);
    rt_mm_unlock();
    return ret;
}
// v_addr = 0xBFD00000, size = 0x00101000
void *_rt_hw_mmu_map_auto(rt_mmu_info *mmu_info, void *v_addr, size_t size, size_t attr)
{
    size_t vaddr;
    size_t offset;
    int pages;
    int ret;

    if (!size)
    {
        return 0;
    }
    offset = (size_t)v_addr & ARCH_PAGE_MASK;	// offset = 0xBFD00000 & 0x00000FFF = 0
    size += (offset + ARCH_PAGE_SIZE - 1);		// size = 0x00101000 + 0 + 0x00000FFF = 0x00101FFF;
    pages = (size >> ARCH_PAGE_SHIFT);			// pages = 0x00101FFF >> 12 = 0x101 = 257
    if (v_addr)
    {
        vaddr = (size_t)v_addr;					// vaddr = 0xBFD00000 
        vaddr &= ~ARCH_PAGE_MASK;				// vaddr = 0xBFD00000 & 0xFFF00000 = 0xBFD00000
        if (check_vaddr(mmu_info, (void*)vaddr, pages) != 0)
        {
            return 0;
        }
    }
    else
    {
        vaddr = find_vaddr(mmu_info, pages);
    }
    if (vaddr) {
        rt_enter_critical();
        ret = __rt_hw_mmu_map_auto(mmu_info, (void*)vaddr, pages, attr);
        if (ret == 0)
        {
            rt_hw_cpu_tlb_invalidate();
            rt_exit_critical();
            return (void*)((char*)vaddr + offset);
        }
        rt_exit_critical();
    }
    return 0;
}
```

##### check_vaddr

```c
// va = 0xBFD00000, pages = 0x101 = 257
static int check_vaddr(rt_mmu_info *mmu_info, void *va, int pages)
{
    size_t loop_va = (size_t)va & ~ARCH_PAGE_MASK; // loop_va = 0xBFD00000 & 0xFFFFF000 = 0xBFD00000
    size_t l1_off, l2_off;
    size_t *mmu_l1, *mmu_l2;

    if (!pages)
    {
        return -1;
    }

    if (!mmu_info)
    {
        return -1;
    }
	// 判断分配的内存是否在虚拟地址空间内
    l1_off = ((size_t)va >> ARCH_SECTION_SHIFT);					// l1_off = 0xBFD
    if (l1_off < mmu_info->vstart || l1_off > mmu_info->vend)		// l1_off < 0x001UL || l1_off > 0xC00UL
    {
        return -1;
    }
    l1_off += ((pages << ARCH_PAGE_SHIFT) >> ARCH_SECTION_SHIFT);	// l1_off = 0xBFD + ((0x101 << 12) >> 20) = 0xBFE
    if (l1_off < mmu_info->vstart || l1_off > mmu_info->vend + 1) 	// l1_off < 0x001UL || l1_off > 0xC00UL
    {
        return -1;
    }

    while (pages--)                                                 // pages = 257
    {
        l1_off = (loop_va >> ARCH_SECTION_SHIFT);					// l1_off = 0xBFD00000 >> 20 = 0xBFD
        l2_off = ((loop_va & ARCH_SECTION_MASK) >> ARCH_PAGE_SHIFT);// (0xBFD00000 & 0x000FFFFF) >> 12 = 0
        mmu_l1 =  (size_t*)mmu_info->vtable + l1_off;               // mmu_info->vtable地址 + 0xBFD

        if (*mmu_l1 & ARCH_MMU_USED_MASK)
        {
            mmu_l2 = (size_t *)((*mmu_l1 & ~ARCH_PAGE_TBL_MASK) - mmu_info->pv_off); // 
            if (*(mmu_l2 + l2_off) & ARCH_MMU_USED_MASK)
            {
                return -1;
            }
        }
        loop_va += ARCH_PAGE_SIZE;
    }
    return 0;
}
```

```c
// v_addr = 0xBFD00000, npages = 0x101 = 257
static int __rt_hw_mmu_map_auto(rt_mmu_info *mmu_info, void* v_addr, size_t npages, size_t attr)
{
    size_t loop_va = (size_t)v_addr & ~ARCH_PAGE_MASK;              // loop_va = 0xBFD00000 & 0xFFFFF000 = 0xBFD00000
    size_t loop_pa;
    size_t l1_off, l2_off;
    size_t *mmu_l1, *mmu_l2;

    if (!mmu_info)
    {
        return -1;
    }

    while (npages--)
    {
        loop_pa = (size_t)rt_pages_alloc(0);                            // 分配物理页面4K
        if (!loop_pa)
            goto err;

        l1_off = (loop_va >> ARCH_SECTION_SHIFT);                       // l1_off = 0xBFD00000 >> 20 = 0xBFD
        l2_off = ((loop_va & ARCH_SECTION_MASK) >> ARCH_PAGE_SHIFT);    // (0xBFD00000 & 0x000FFFFF) >> 12 = 0
        mmu_l1 =  (size_t*)mmu_info->vtable + l1_off;

        if (*mmu_l1 & ARCH_MMU_USED_MASK)
        {
            mmu_l2 = (size_t *)((*mmu_l1 & ~ARCH_PAGE_TBL_MASK) - mmu_info->pv_off);
            rt_page_ref_inc(mmu_l2, 0);
        }
        else
        {
            //mmu_l2 = (size_t*)rt_malloc_align(ARCH_PAGE_TBL_SIZE * 2, ARCH_PAGE_TBL_SIZE);
            mmu_l2 = (size_t*)rt_pages_alloc(0);
            if (mmu_l2)
            {
                rt_memset(mmu_l2, 0, ARCH_PAGE_TBL_SIZE * 2);
                /* cache maintain */
                rt_hw_cpu_dcache_clean(mmu_l2, ARCH_PAGE_TBL_SIZE);

                *mmu_l1 = (((size_t)mmu_l2 + mmu_info->pv_off) | 0x1);
                /* cache maintain */
                rt_hw_cpu_dcache_clean(mmu_l1, 4);
            }
            else
                goto err;
        }

        loop_pa += mmu_info->pv_off;
        *(mmu_l2 + l2_off) = (loop_pa | attr);
        /* cache maintain */
        rt_hw_cpu_dcache_clean(mmu_l2 + l2_off, 4);

        loop_va += ARCH_PAGE_SIZE;
    }
    return 0;
err:
    {
        /* error, unmap and quit */
        int i;
        void *va, *pa;

        va = (void*)((size_t)v_addr & ~ARCH_PAGE_MASK);
        for (i = 0; i < npages; i++)
        {
            pa = rt_hw_mmu_v2p(mmu_info, va);
            pa = (void*)((char*)pa - mmu_info->pv_off);
            rt_pages_free(pa, 0);
            va = (void*)((char*)va + ARCH_PAGE_SIZE);
        }

        __rt_hw_mmu_unmap(mmu_info, v_addr, npages);
        return -1;
    }
}
```