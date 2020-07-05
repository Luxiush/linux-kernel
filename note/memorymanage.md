## 内存管理

### 内核空间和用户空间
32位系统, 虚拟地址空间总大小为4G, linux把最高的1G分(0xC0000000~0xFFFFFFFF)配给内核使用(`内核空间`), 剩下的3G留给应用程序(`用户空间`).

每个进程各自私有的用户空间, 对其他进程不可见. 而虚拟内核空间则为所有进程以及内核共享. 对于进程来说这个虚拟地址空间的总大小为4G, 但是实际可用的只有`用户空间`部分.

64位系统, 同32位一样, 内核空间和用户空间在一个在顶部一个在底部. 由于其具有的地址空间相对于现在的市面上常见的物理磁盘(TB级别)来说足够大(`16EB`), 因此内核空间可以直接映射到物理空间, 不需要走`ZONE_HIGHMEM`那套映射机制.[^x64]

[^x64]: https://zhuanlan.zhihu.com/p/66794639


### 内存管理区(Zone)
理想情况下虚拟地址和物理地址可以一一对应(即线性映射), 但是这样内核就最多只能访问1G的物理内存, 如果物理内存大于1G, 那剩下的就访问不到了.

linux的做法是留一部分虚拟地址空间来做动态映射, 根据需要映射到对应的物理内存上, 即`ZONE_HIGHMEM`.

同时内核要和各种设备进行通信, 但是各种设备的地址空间通常很小, 因此还需将最低的一部分空间预留出来. 即`ZONE_DMA`. 这部分空间, 设备能够通过物理地址直接访问, 不经过MMU. [^DMA]

剩下的中间那部分也给个名字, 叫`ZONE_NORMAL`.

`ZONE_MOVABLE`: 一个伪内存区域. 管理可以移动的页面. 用于防止内存碎片化. 位于该区域的页面是可以移动和可以回收的. [^ZONE_MOVABLE]

REF:

<Documentation/admin-guide/mm/concepts.rst>

<include/linux/mmzone.h: enum zone_type>

[^DMA]: DMA(Directly Memory Access)是一种无需CPU的参与就可以让外设和系统内存之间进行双向数据传输的硬件机制.

[^ZONE_MOVABLE]: https://lwn.net/Articles/219589/

```c
/* include/linux/mmzone.h */
struct free_area {
    struct list_head    free_list[MIGRATE_TYPES];
    unsigned long       nr_free;
};

struct zone {
    int node;
    struct pglist_data* zone_pgdat;
    unsigned long       zone_start_pfn;
    struct free_area    free_area[MAX_ORDER];
};
```


### 存储节点(Node)

#### 存储结构

- 一致存储结构(Uniform Memory Architecture): 物理内存是连续的, 每个cpu访问内存所需的时间是相等的.

- 非一致存储结构(Non-Uniform Memory Architecture): 每个cpu都有自己本地的物理内存, 同时也可以通过总线访问其他cpu的物理内存. 显然, 访问自己本地的物理内存比访问其他cpu的物理内存要快.

![https://www.cnblogs.com/xelatex/p/3491301.html](./img/uma_numa.png)

#### 存储节点

linux把访问时间相同的存储空间看做一个`存储节点`(Node).

> On NUMA machines, each NUMA node would have a pg_data_t to describe it's memory layout.

每个存储节点都会有用一个`pg_data_t`来描述其内存布局.

```c
/* include/linux/mmzone.h */
typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];
    // ...
    wait_queue_head_t kswapd_wait;
    struct task_struct* kswapd;
    // ...
    wait_queue_head_t kcompactd_wait;
    struct task_struct* kcompactd;
    // ...
} pg_data_t;


struct pglist_data *node_data[MAX_NUMNODES] __read_mostly;
#define NODE_DATA(nid)  (node_data[nid])

```

#### node的初始化
- pfn: Page Frame Number (页帧号). 把连续的物理内存当成一个page数组, 那么pfn就对应数组的下标. 给定一个物理地址pa, 对应的pfn为`pa >> PAGE_SHIFT`, 即整除一个页面的大小.

```c
/* arch/x86/mm/numa.c */
/* 分配内存, 从memblock中分配内存: x06_numa_init -> numa_init -> numa_register_memblks -> alloc_node_data */
static void __init alloc_node_data(int nid)
{
    nd_pa = memblock_phys_alloc_nid(nd_size, SMP_CACHE_BYTES, nid);
    // ...

    nd = __va(nd_pa);
    node_data[nid] = nd;
    memset(NODE_DATA(nid), 0, sizeof(pg_data_t));
    node_set_online(nid);
}

/* mm/page_alloc.c */
/* 初始化, 初始化节点上的各个管理区(Zone)*/
void __init zone_sizes_init(void)
{
    unsigned long max_zone_pfns[MAX_NR_ZONES];

    memset(max_zone_pfns, 0, sizeof(max_zone_pfns));

#ifdef CONFIG_ZONE_DMA
    max_zone_pfns[ZONE_DMA]        = min(MAX_DMA_PFN, max_low_pfn);
#endif
#ifdef CONFIG_ZONE_DMA32
    max_zone_pfns[ZONE_DMA32]    = min(MAX_DMA32_PFN, max_low_pfn);
#endif
    max_zone_pfns[ZONE_NORMAL]    = max_low_pfn;
#ifdef CONFIG_HIGHMEM
    max_zone_pfns[ZONE_HIGHMEM]    = max_pfn;
#endif

    free_area_init_nodes(max_zone_pfns);
}

void __init free_area_init_nodes(unsigned long *max_zone_pfn)
{
    // ...
    for_each_online_node(nid) {
        pg_data_t* pgdat = NODE_DATA(nid);
        free_area_init_node(nid, NULL,
                find_min_pfn_for_node(nid), NULL);

        // Any memory on that node
        if (pgdat->node_present_pages)
            node_set_state(nid, N_MEMORY);
        check_for_memory(pgdat, nid);
    }
}

void __init free_area_init_node(int nid, unsigned long* zones_size,
                   unsigned long node_start_pfn,
                   unsigned long* zholes_size)
{
    pg_data_t* pgdat = NODE_DATA(nid);
    unsigned long start_pfn = 0;
    unsigned long end_pfn = 0;

    // pg_data_t should be reset to zero when it's allocated
    WARN_ON(pgdat->nr_zones || pgdat->kswapd_classzone_idx);

    pgdat->node_id = nid;
    pgdat->node_start_pfn = node_start_pfn;
    pgdat->per_cpu_nodestats = NULL;
#ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
    get_pfn_range_for_nid(nid, &start_pfn, &end_pfn);
    pr_info("Initmem setup node %d [mem %#018Lx-%#018Lx]\n", nid,
        (u64)start_pfn << PAGE_SHIFT,
        end_pfn ? ((u64)end_pfn << PAGE_SHIFT) - 1 : 0);
#else
    start_pfn = node_start_pfn;
#endif
    calculate_node_totalpages(pgdat, start_pfn, end_pfn,
                  zones_size, zholes_size);

    alloc_node_mem_map(pgdat);
    pgdat_set_deferred_range(pgdat);

    free_area_init_core(pgdat);
}

```

### memblock
早期的内存管理机制, 直接操作物理地址.
负责在内存管理相关的数据结构(node)初始化之前的内存分配.
直接把物理内存当做一个page数组, 管理的时候也是以page为单位.


### 页(Page)
内存管理的最小单位
一个页面大小通常为4K(2^12)

- 零页(`struct boot_params`), 存放内核的启动参数.


#### 页面的申请
伙伴(buddy)算法

```c
/* mm/page_alloc.c */
/*
 * This is the 'heart' of the zoned buddy allocator.
*/
struct page * __alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order, int preferred_nid, nodemask_t *nodemask)
{
    page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
    if (likely(page))
        goto out;

    page = __alloc_pages_slowpath(alloc_mask, order, &ac);

out:
    return page;
}

/*
 get_page_from_freelist
    -> node_relcaim
    -> rmqueue -> __rmqueue
                    -> __rmqueue_smallest
                    -> __rmqueue_fallback

*/

static __always_inline struct page *
__rmqueue(struct zone *zone, unsigned int order, int migratetype,
                        unsigned int alloc_flags)
{
    struct page* page;

retry:
    page = __rmqueue_smallest(zone, order, migratetype);
    if (unlikely(!page)) {
        if (migratetype == MIGRATE_MOVABLE)
            page = __rmqueue_cma_fallback(zone, order);

        if (!page && __rmqueue_fallback(zone, order, migratetype,
                                alloc_flags))
            goto retry; // 当前migratetype的空闲页不够, 尝试从别的migratetype偷一些过来, 然后再试.
    }

    trace_mm_page_alloc_zone_locked(page, order, migratetype);
    return page;
}

/*
5    ++++++++++++++++++++++++++++++++    (+: 当前最小的空闲页)
4                    ****************    (*: 没有分配出去的部分)
3            ********
2        ****
1    xx**                                (x: 分配出去的部分)

2^5 = 2^4 + 2^3 + 2^2 + 2^1 + 2^1

假设需要一个2^1大小的页面, 但是现在空闲的最小页面是2^5, 所以把2^5页面进行拆分, 拆出一个2^1大小的页面, 然后剩下的重新放到frea_area中.
*/
static struct page* __rmqueue_smallest(struct zone *zone, unsigned int order, int migratetype)
{
    // 把zone->free_area扫一遍, 找到能够满足分配请求(1<<order)的最小的空闲页.
    // 从找到的空闲页中拆分出请求的大小(1<<order), 剩下的部分重新记录到free_area中,
    // 如上图所示.
}

static bool __rmqueue_fallback(struct zone *zone, int order, int start_migratetype,
                                unsigned int alloc_flags)
{
    // 从其他的migratetype"偷"一些空闲页面到当前的migratetype
    // fallback的顺序在`falbacks`数组中定义
}

```


#### 页面的释放
```c
/*
buddy的计算(__find_budy_pfn):
buddy_pfn = page_pfn ^ (1 << order)
higher_pfn = page_pfn & ~(1 << order)

higher_pfn = page_pfn & buddy_pfn
*/

/*
free_pages
    -> __free_pages
        -> free_the_page
            -> __free_pages_ok
                -> free_one_page
                    -> __free_one_page
*/
static inline void __free_one_page(struct page* page,
		unsigned long pfn,
		struct zone* zone, unsigned int order,
		int migratetype)
{
	unsigned long combined_pfn;
	unsigned long uninitialized_var(buddy_pfn);
	struct page* buddy;
	unsigned int max_order;

	max_order = min_t(unsigned int, MAX_ORDER, pageblock_order + 1);

	VM_BUG_ON(!zone_is_initialized(zone));
	VM_BUG_ON_PAGE(page->flags & PAGE_FLAGS_CHECK_AT_PREP, page);

	VM_BUG_ON(migratetype == -1);
	if (likely(!is_migrate_isolate(migratetype)))
		__mod_zone_freepage_state(zone, 1 << order, migratetype);

	VM_BUG_ON_PAGE(pfn & ((1 << order) - 1), page);
	VM_BUG_ON_PAGE(bad_range(zone, page), page);

continue_merging:
	while (order < max_order - 1) {
		buddy_pfn = __find_buddy_pfn(pfn, order);
		buddy = page + (buddy_pfn - pfn);

		if (!pfn_valid_within(buddy_pfn))
			goto done_merging;
		if (!page_is_buddy(page, buddy, order))
			goto done_merging;

		// Our buddy is free or it is CONFIG_DEBUG_PAGEALLOC guard page,
		// merge with it and move up one order.
		if (page_is_guard(buddy)) {
			clear_page_guard(zone, buddy, order, migratetype);
		} else {
			list_del(&buddy->lru);
			zone->free_area[order].nr_free--;
			rmv_page_order(buddy);
		}
		combined_pfn = buddy_pfn & pfn;
		page = page + (combined_pfn - pfn);
		pfn = combined_pfn;
		order++;
	}
	if (max_order < MAX_ORDER) {
		// If we are here, it means order is >= pageblock_order.
		// We want to prevent merge between freepages on isolate
		// pageblock and normal pageblock. Without this, pageblock
		// isolation could cause incorrect freepage or CMA accounting.
		//
		// We don't want to hit this code for the more frequent
		// low-order merging.
		if (unlikely(has_isolate_pageblock(zone))) {
			int buddy_mt;

			buddy_pfn = __find_buddy_pfn(pfn, order);
			buddy = page + (buddy_pfn - pfn);
			buddy_mt = get_pageblock_migratetype(buddy);

			if (migratetype != buddy_mt
					&& (is_migrate_isolate(migratetype) ||
						is_migrate_isolate(buddy_mt)))
				goto done_merging;
		}
		max_order++;
		goto continue_merging;
	}

done_merging:
	set_page_order(page, order);

	// If this is not the largest possible page, check if the buddy
	// of the next-highest order is free. If it is, it's possible
	// that pages are being freed that will coalesce soon. In case,
	// that is happening, add the free page to the tail of the list
	// so it's less likely to be used soon and more likely to be merged
	// as a higher order page

	if ((order < MAX_ORDER-2) && pfn_valid_within(buddy_pfn)) {
		struct page * higher_page, * higher_buddy;
		combined_pfn = buddy_pfn & pfn;
		higher_page = page + (combined_pfn - pfn);
		buddy_pfn = __find_buddy_pfn(combined_pfn, order + 1);
		higher_buddy = higher_page + (buddy_pfn - combined_pfn);
		if (pfn_valid_within(buddy_pfn) &&
		    page_is_buddy(higher_page, higher_buddy, order + 1)) {
			list_add_tail(&page->lru,
				&zone->free_area[order].free_list[migratetype]);
			goto out;
		}
	}

	list_add(&page->lru, &zone->free_area[order].free_list[migratetype]);
out:
	zone->free_area[order].nr_free++;
}

```


### 页面的回收 (reclaim)
内存不足的时候回收一些已经分配出去的页面, 把一些页面`换出`

内核线程: kswapd

- watermark 水位, 记录当前内存的剩余量.

high
 当剩余内存在high以上时，系统认为当前内存使用压力不大。
low
 当剩余内存降低到low时，系统就认为内存已经不足了，会触发kswapd内核线程进行内存回收处理
min
 当剩余内存在min以下时，则系统内存压力非常大。一般情况下min以下的内存是不会被分配的，min以下的内存默认是保留给特殊用途使用，属于保留的页框，用于原子的内存请求操作。
比如：当我们在中断上下文申请或者在不允许睡眠的地方申请内存时，可以采用标志GFP_ATOMIC来分配内存，此时才会允许我们使用保留在min水位以下的内存。
<https://blog.csdn.net/rikeyone/article/details/85037249>


```c
/*
__alloc_pages_slowpath
    -> __alloc_pages_direct_reclaim                 // 页面回收, 把页面'换出', 写到磁盘上
        -> __perform_reclaim
            -> try_to_free_pages
                -> do_try_to_free_pages
                    -> shrink_zones
                        -> shrink_node
                            -> shrink_active_list
                                -> mem_cgroup_uncharge_list

    -> __alloc_pages_direct_compact                 // 页面的压缩, 把空闲的页面移到一起
        -> __try_to_compact_pages
            -> compact_zone_order
                -> compact_zone
                    -> isolate_migratepages        // 收集可以迁移的页面
                    -> migrate_pages
                        -> unmap_and_move
                            -> __unmap_and_move
                                -> try_to_unmap     // 释放页表中的映射
                                -> move_to_new_page // 页面迁移, 在page->mapping中会有一个回调函数(ops->migratepage)专门用来迁移页面.
*/
```

### 页面的压缩 (compaction)
防止碎片化
内核线程: kcompactd

把空闲的页面移到一起

页面迁移: Documentation/vm/page_migration.rst

.
