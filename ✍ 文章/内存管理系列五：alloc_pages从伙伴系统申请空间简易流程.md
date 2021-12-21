## 前沿

### 往篇回顾

根据前面几篇介绍，目前为止已经走过了如下几个阶段：

- 在MMU打开前的kernel space、fdt mapping.
- 打开MMU后，通过解析dtb，初始化了memblock, 进入fixmap, memblock时代.
- 之后在经过page_init()->bootmem_init函数初始化zone，zonelist，以及struct page结构体初始化等，buddy sys初具模型.
- 在接下来的mm_init()->mem_init()函数，会把目前空闲的mem通过__free_pages(page, order)添加到buddy sys.

### buddy system基本概念：

- 伙伴系统是基于struct zone，即每个zone都有自己的伙伴系统，zone->free_area用于描述本zone的伙伴系统状态.
- free_area中对不同的类型的MIGRATE_TYPES有区分管理，具体分类见附录
- free_area维护有最高2^10(MAX_ORDER=10)的连续页框链表.

### 伙伴系统的页框分配方式主要有两种：

- 快速分配(get_page_from_freelist)：
  1. 根据zonelist优先级顺序，以zone的WATER_LOW阀值从相应zone的伙伴系统中分配连续页框.
- 慢速分配(__alloc_pages_slowpath)：(如果不允许__GFP_WAIT，则不能进行慢速分配)
  1. 在快速分配失败后，根据zonelist优先级，以zone的WATER_MIN阀值从相应的zone的伙伴系统中分配连续页框.
  2. 失败，则唤醒kswapd内核线程进行页框回收、页框的异步压缩、轻同步压缩、OOM等使系统获得更多空闲页框；并在这些操作的过程中不时调用快速分配尝试获取内存.

### 本篇主要内容

在mem_init()完成buddy sys初始化后就可以正常申请内存了，接下来看看在伙伴系统的框架内，如何申请、释放内存；以及更进一步页面回收、内存规整、直接回收内存等.

> 本篇中主要聚焦快速分配，慢速分配中不同的动作再以后进一步分析.

## 代码分析

### alloc_pages_nodemask: heart of buddy

```c
/*
 * 1. gfp_mask: 上层要求分配内存时使用的标志
 * 2. order： 要分配的阶数-1
 * 3. zonelist： 合适的zone列表
 * 4. nodemask: node掩码
 */
struct page *
__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
			struct zonelist *zonelist, nodemask_t *nodemask)
{
	/* 用于指向第一个合适的zone */
	struct zoneref *preferred_zoneref;
	struct page *page = NULL;
	unsigned int cpuset_mems_cookie;
	/* 分配标识，默认WMARK_LOW来判断是否进入慢速分配 */
	int alloc_flags = ALLOC_WMARK_LOW|ALLOC_CPUSET|ALLOC_FAIR;
	gfp_t alloc_mask; /* The gfp_t that was actually used for allocation */
	struct alloc_context ac = {
		.high_zoneidx = gfp_zone(gfp_mask),
		.nodemask = nodemask,
		 /* 从gfp_mask中获取选定的页框类型，当中只会检查__GFP_MOVABLE和__GFP_RECLAIMABLE */
		.migratetype = gfpflags_to_migratetype(gfp_mask),
	};

	gfp_mask &= gfp_allowed_mask;

	lockdep_trace_alloc(gfp_mask);

	might_sleep_if(gfp_mask & __GFP_WAIT);

	if (should_fail_alloc_page(gfp_mask, order))
		return NULL;

	/*
	 * Check the zones suitable for the gfp_mask contain at least one
	 * valid zone. It's possible to have an empty zonelist as a result
	 * of __GFP_THISNODE and a memoryless node
	 */
	if (unlikely(!zonelist->_zonerefs->zone))
		return NULL;
	/* 如果使能了CONFIG_CMA，则允许从CMA区域分配 */
	if (IS_ENABLED(CONFIG_CMA) && ac.migratetype == MIGRATE_MOVABLE)
		alloc_flags |= ALLOC_CMA;

retry_cpuset:
	/* 获取读锁 */
	cpuset_mems_cookie = read_mems_allowed_begin();

	/* We set it here, as __alloc_pages_slowpath might have changed it */
	ac.zonelist = zonelist;
	/* The preferred zone is used for statistics later */
	/* 
	 * 获取链表中第一个zone，每一次retry_cpuset就是尝试在下一个zone进行分配
     */
	preferred_zoneref = first_zones_zonelist(ac.zonelist, ac.high_zoneidx,
				ac.nodemask ? : &cpuset_current_mems_allowed,
				&ac.preferred_zone);
	if (!ac.preferred_zone)
		goto out;
	ac.classzone_idx = zonelist_zone_idx(preferred_zoneref);

	/* 它限制只在分配到当前进程的各个CPU所关联的结点分配内存 */
	alloc_mask = gfp_mask|__GFP_HARDWALL;
	/* 第一次尝试分配页框(快速分配)
     * 快速分配时以low阀值为标准
     * 遍历zonelist，尝试获取2的order次方个连续的页框 
     * 在遍历zone时，如果此zone当前空闲内存减去需要申请的内存之后，空闲内存是低于low阀值，那么此zone会进行快速内存回收
	 * 具体分配流程见下
     */
	page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
	if (unlikely(!page)) {
		/*
		 * Runtime PM, block IO and its error handling path
		 * can deadlock because I/O on the device might not
		 * complete.
		 * 快速分配没有成功，则下面要进行慢速分配；其中就有可能触发页面压缩、页面回收，过程中涉及页面换出
		 * 如果当前分配task_struct->flags标识有PF_MEMALLOC_NOIO标识，则分配页框过程中不允许有页面换出动作
		 * 则在alloc_mask中需要把__GFP_IO和__GFP_FS清除
		 */
		alloc_mask = memalloc_noio_flags(gfp_mask);
        /* 如果之前没有分配成功，这里尝试进入慢速分配，主要流程如下： 
         * 1. 慢速分配首先会唤醒所有符合条件zone的kswapd线程进行内存回收
		 * 2. 再次尝试快速分配；疑问：唤醒回收线程后，不需要等待？
         * 3. 如果标记了忽略阀值，则从保留的内存里回收(__alloc_pages_high_priority)
         * 4. 然后进行内存压缩(__alloc_pages_direct_compact), 其中会尝试再次快速分配
         * 5. 最后再尝试直接内存回收，触发OOM(out_of_memory),前提是所有zones的ZONE_OOM_LOCKED锁均未被占用
		 * 6. 如果判断需要retry，则再次尝试1,2,3,4
         */
		page = __alloc_pages_slowpath(alloc_mask, order, &ac);
	}
	/*  接下去两个动作是为实现kmemchek动态内存检测工具，具体可以查阅资料 */
	if (kmemcheck_enabled && page)
		kmemcheck_pagealloc_alloc(page, order, gfp_mask);

	trace_mm_page_alloc(page, order, alloc_mask, ac.migratetype);

out:
	/*
	 * When updating a task's mems_allowed, it is possible to race with
	 * parallel threads in such a way that an allocation can fail while
	 * the mask is being updated. If a page allocation is about to fail,
	 * check if the cpuset changed during allocation and if so, retry.
	 */
	 /* 如果都没有分配成功，这里会不停尝试重新分配，获取zonelist的下一个zone */
	if (unlikely(!page && read_mems_allowed_retry(cpuset_mems_cookie)))
		goto retry_cpuset;

	return page;
}
EXPORT_SYMBOL(__alloc_pages_nodemask);
```

### get_page_from_freelist

```c
/*
 * get_page_from_freelist goes through the zonelist trying to allocate
 * a page.
 * 在zonelist中遍历所有zone，获取指定连续的页框数：
 * 1. 在遍历zone时，如果zone当前空闲内存-申请的内存<LOW,则该zone会触发内存回收
 * 2. 第一轮循环只会尝试从preferred_zone中获取页框
 * 3. 第二轮会遍历整个zonelist的zone
 */
static struct page *
get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
						const struct alloc_context *ac)
{
	struct zonelist *zonelist = ac->zonelist;
	struct zoneref *z;
	struct page *page = NULL;
	struct zone *zone;
	nodemask_t *allowednodes = NULL;/* zonelist_cache approximation */
	int zlc_active = 0;		/* set if using zonelist_cache */
	int did_zlc_setup = 0;		/* just call zlc_setup() one time */
	/* 是否考虑脏页过多的判断值，如果脏页过多，则不在此zone进行分配 */
	bool consider_zone_dirty = (alloc_flags & ALLOC_WMARK_LOW) &&
				(gfp_mask & __GFP_WRITE);
	int nr_fair_skipped = 0;
	bool zonelist_rescan;

zonelist_scan:
	zonelist_rescan = false;

	/*
	 * Scan zonelist, looking for a zone with enough free.
	 * See also __cpuset_node_allowed() comment in kernel/cpuset.c.
	 */
	for_each_zone_zonelist_nodemask(zone, z, zonelist, ac->high_zoneidx,
								ac->nodemask) {
		unsigned long mark;
	/*
	 * 检查是否有必要在该zone尝试分配
	 * 1. check zonelist_cache的full标志位
	 */
		if (IS_ENABLED(CONFIG_NUMA) && zlc_active &&
			!zlc_zone_worth_trying(zonelist, z, allowednodes))
				continue;
	/*
	* 检查此zone是否属于该CPU所允许分配的zone
	* ALLOC_CPUSET: 仅允许在该CPU管辖的zone中分配
	*/
		if (cpusets_enabled() &&
			(alloc_flags & ALLOC_CPUSET) &&
			!cpuset_zone_allowed(zone, gfp_mask))
				continue;
		/*
		 * ALLOC_FAIR: 只从preferred_zone所在node中的zone分配
		 * 该机制保证第一轮循环只会从preferred_zone所在node分配，第二轮会去除该标志
		 */
		if (alloc_flags & ALLOC_FAIR) {
		/* 判断zone和preferred_zone是否属于同一个node，不属于则跳出循环，因为后面的页不会属于此node了 */
			if (!zone_local(ac->preferred_zone, zone))
				break;
			if (test_bit(ZONE_FAIR_DEPLETED, &zone->flags)) {
				nr_fair_skipped++;
				continue;
			}
		}
		/* 如果gfp_mask中允许进行脏页回写，那么如果此zone在内存中有过多的脏页，则跳过此zone，不对此zone进行处理
         * 这里大概意思是脏页过多，kswapd会将这些脏页进行回写，这里就不将这些脏页进行回写了，会增加整个zone的压力
         */
		if (consider_zone_dirty && !zone_dirty_ok(zone))
			continue;
		/* 选择分配阀值有alloc_min alloc_low alloc_high三种，要求分配后空闲页框数量不能少于阀值，默认是alloc_low */
		mark = zone->watermark[alloc_flags & ALLOC_WMARK_MASK];
        /* 根据阀值查看zone中是否有足够的空闲页框，空闲内存数量保存在 zone->vm_stat[NR_FREE_PAGES]
		 * 算法: 分配后剩余的页框数量 > 阀值 + zone reserve，高于则返回true
         * 三个阀值的大小关系是min < low < high
         * high一般用于判断zone是否平衡
         * 快速分配时，用的阀值是low
         * 慢速分配中，用的阀值是min
         * 在准备oom进程时，用的阀值是high
         * 分配后剩余的页框数量 <= 阀值 + zone reserve: 进行快速内存回收
         */		
		if (!zone_watermark_ok(zone, order, mark,
				       ac->classzone_idx, alloc_flags)) {
			int ret;

			/* Checked here to keep the fast path fast */
			BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
			/* 如果分配标志中有 ALLOC_NO_WATERMARKS标志，代表无视阀值，直接分配 */
			if (alloc_flags & ALLOC_NO_WATERMARKS)
				goto try_this_zone;

			if (IS_ENABLED(CONFIG_NUMA) &&
					!did_zlc_setup && nr_online_nodes > 1) {
				/* NUMA系统中如果使用了zlc(zonelist_cache)，则取出此zonelist允许的node列表 */
				allowednodes = zlc_setup(zonelist, alloc_flags);
				zlc_active = 1;
				did_zlc_setup = 1;
			}
            /* 
             * 判断是否对此zone进行内存回收，
			 * 1. 开启了内存回收，对此zone进行内存回收
			 * 2. 当内存回收未开启, 只会对距离比较近的zone进行回收
             * 		zone_allows_reclaim(): 判断zone所在node是否与preferred_zone所在node的距离 < RECLAIM_DISTANCE(30或10)
             */
			if (zone_reclaim_mode == 0 ||
			    !zone_allows_reclaim(ac->preferred_zone, zone))
				goto this_zone_full;

			/*
			 * As we may have just activated ZLC, check if the first
			 * eligible zone has failed zone_reclaim recently.
			 */
			if (IS_ENABLED(CONFIG_NUMA) && zlc_active &&
				!zlc_zone_worth_trying(zonelist, z, allowednodes))
				continue;
            /*
             * 内存回收： 回收到了2^order数量的页框时，才会返回真，即使回收了，没达到这个数量，也返回假；
			 * 内存回收原理另分析
             */
			ret = zone_reclaim(zone, gfp_mask, order);
			switch (ret) {
			case ZONE_RECLAIM_NOSCAN:
				/* did not scan */
				continue;
			case ZONE_RECLAIM_FULL:
				/* scanned but unreclaimable */
				continue;
			default:
				/*
				* 回收到了一些或者回收到了2^order个页框 
				* 检查是否满足水位要求，同理
				*/
				if (zone_watermark_ok(zone, order, mark,
						ac->classzone_idx, alloc_flags))
					goto try_this_zone;

				/*
				 * 无法回收到足够内存：
				 * 1. 如果是按WMARK_MIN进行分配
				 * 2. 已经回收到一些内存
				 * ===》调到this_zone_full===》标记zone
				 */
				if (((alloc_flags & ALLOC_WMARK_MASK) == ALLOC_WMARK_MIN) ||
				    ret == ZONE_RECLAIM_SOME)
					goto this_zone_full;

				continue;
			}
		}

try_this_zone:
        /* 
		 * 尝试从这个zone获取连续页框：
         * 只有当此zone中空闲页框数量 - 本次需要分配的数量 > 此zone的low阀值，这样才能执行到这
         * 如果本意从preferred_zone分配内存，但是preferred_zone没有足够内存，到了此zone进行分配，那么分配的页数量会统计到此zone的NR_ALLOC_BATCH
         */
		page = buffered_rmqueue(ac->preferred_zone, zone, order,
						gfp_mask, ac->migratetype);
		if (page) {
		/*
		* page出炉前做一些准备：
		1. kasan数据准备
		*/
			if (prep_new_page(page, order, gfp_mask, alloc_flags))
				goto try_this_zone;
			return page;
		}
this_zone_full:
		if (IS_ENABLED(CONFIG_NUMA) && zlc_active)
		/* 在zonelist的zonelist_cache中标记此node为满状态 */
			zlc_mark_zone_full(zonelist, z);
	}

    /* 
	 * 如果第一次ALLOC_FAIR分配没有能够分配到内存，第二次尝试非ALLOC_FAIR分配 
     * 第二次会遍历zonelist中其他node上的zone
     */
	if (alloc_flags & ALLOC_FAIR) {
		alloc_flags &= ~ALLOC_FAIR;
		/* nr_fair_skipped不为0，说明此node有些zone的batch页已经用尽，这里要增加一些给它 */
		if (nr_fair_skipped) {
			zonelist_rescan = true;
			/* 
			 * 重新设置除目标zone之外，node中在此目标zone前面的zone的batch页数量大小 
             * 设置为: batch页数量 = high阀值 - 低阀值 - 当前batch数量
             */
			reset_alloc_batches(ac->preferred_zone);
		}
		if (nr_online_nodes > 1)
			zonelist_rescan = true;
	}

	if (unlikely(IS_ENABLED(CONFIG_NUMA) && zlc_active)) {
		/* 禁止zonelist_cache，zonelist_cache用于快速扫描的，它标记着哪个node有空闲内存哪个node没有，扫描时就跳过某些node */
		zlc_active = 0;
		zonelist_rescan = true;
	}
	/* 跳回去，尝试再次扫描一遍zonelist，这里最多只会进行一次再次扫描，因为第二次就不会把 zonelist_rescan 设置为true了 */
	if (zonelist_rescan)
		goto zonelist_scan;

	return NULL;
}
```

## 附录

### zonelist type

```c
/* 这几个链表主要用于反内存碎片 */
enum {
    MIGRATE_UNMOVABLE,         /* 页框内容不可移动,在内存中位置必须固定，无法移动到其他地方，核心内核分配的大部分页面都属于这一类。 */
    MIGRATE_RECLAIMABLE,         /* 页框内容可回收,不能直接移动，但是可以回收，因为还可以从某些源重建页面，比如映射文件的数据属于这种类别，kswapd会按照一定的规则，周期性的回收这类页面。 */
    MIGRATE_MOVABLE,             /* 页框内容可移动，属于用户空间应用程序的页属于此类页面，它们是通过页表映射的，因此我们只需要更新页表项，并把数据复制到新位置就可以了
                                 * 当然要注意，一个页面可能被多个进程共享，对应着多个页表项。 
                                 */
    MIGRATE_PCPTYPES,             /* 用来表示每CPU页框高速缓存的数据结构中的链表的迁移类型数目 */
    MIGRATE_RESERVE = MIGRATE_PCPTYPES,     
#ifdef CONFIG_CMA
    MIGRATE_CMA,                   /* 预留一段的内存给驱动使用，但当驱动不用的时候，伙伴系统可以分配给用户进程用作匿名内存或者页缓存。而当驱动需要使用时，就将进程占用的内存通过回收或者迁移的方式将之前占用的预留内存腾出来，供驱动使用。 */
#endif   
#ifdef CONFIG_MEMORY_ISOLATION
    MIGRATE_ISOLATE,            /* 不能从这个链表分配页框，因为这个链表专门用于NUMA结点移动物理内存页，将物理内存页内容移动到使用这个页最频繁的CPU */
#endif
    MIGRATE_TYPES
};
```

### zone_modify

```c
#define __GFP_DMA   ((__force gfp_t)0x01u)   
#define __GFP_HIGHMEM   ((__force gfp_t)0x02u)   
#define __GFP_DMA32 ((__force gfp_t)0x04u)  
#define __GFP_MOVABLE ((__force gfp_t)0x08u)
```

### zone_action

```c
#define __GFP_WAIT  ((__force gfp_t)0x10u)  //表示分配内存的请求可以中断。也就是说，调度器在该请求期间可随意选择另一个过程执行，或者该请求可以被另一个更重要的事件中断。   
#define __GFP_HIGH  ((__force gfp_t)0x20u)  //如果请求非常重要，则设置__GFP_HIGH，即内核急切的需要内存时。在分配内存失败可能给内核带来严重得后果时，一般会设置该标志   
#define __GFP_IO    ((__force gfp_t)0x40u)  //在查找空闲内存期间内核可以进行I/O操作。这意味着如果内核在内存分配期间换出页，那么仅当设置该标志时，才能将选择的页写入磁盘。   
#define __GFP_FS    ((__force gfp_t)0x80u)  //允许内核执行VFS操作   
#define __GFP_COLD  ((__force gfp_t)0x100u) //如果需要分配不在CPU高速缓存中的“冷”页时，则设置__GFP_COLD。   
#define __GFP_NOWARN    ((__force gfp_t)0x200u) //在分配失败时禁止内核故障警告。   
#define __GFP_REPEAT    ((__force gfp_t)0x400u) //在分配失败后自动重试，但在尝试若干次之后会停止。   
#define __GFP_NOFAIL    ((__force gfp_t)0x800u) //在分配失败后一直重试，直至成功。   
#define __GFP_NORETRY   ((__force gfp_t)0x1000u)//不重试，可能失败   
#define __GFP_COMP  ((__force gfp_t)0x4000u)//增加复合页元数据   
#define __GFP_ZERO  ((__force gfp_t)0x8000u)//在分配成功时，将返回填充字节0的页。   
#define __GFP_NOMEMALLOC ((__force gfp_t)0x10000u) //不适用紧急分配链表   
#define __GFP_HARDWALL   ((__force gfp_t)0x20000u) //只在NUMA系统上有意义。它限制只在分配到当前进程的各个CPU所关联的结点分配内存。如果进程允许在所有的CPU上运行（默认情况下），该标志是没有意义的。只有进程可以运行的CPU受限时，该标志才有意义。   
#define __GFP_THISNODE  ((__force gfp_t)0x40000u)//页只在NUMA系统上有意义，如果设置该比特位，则内存分配失败的情况下不允许使用其他结点作为备用，需要保证在当前结点或者明确指定的结点上成功分配内存。   
#define __GFP_RECLAIMABLE ((__force gfp_t)0x80000u) //将分配的内存标记为可回收   
#define __GFP_MOVABLE   ((__force gfp_t)0x100000u)  //将分配的内存标记为可移动   
  
#define __GFP_BITS_SHIFT 21 /* Room for 21 __GFP_FOO bits */   
#define __GFP_BITS_MASK ((__force gfp_t)((1 << __GFP_BITS_SHIFT) - 1))   
  
/* This equals 0, but use constants in case they ever change */  
#define GFP_NOWAIT  (GFP_ATOMIC & ~__GFP_HIGH)  
```

### fallbacks

```c
static int fallbacks[MIGRATE_TYPES][4] = {
    [MIGRATE_UNMOVABLE]   = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE,     MIGRATE_RESERVE },
    [MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE,   MIGRATE_MOVABLE,     MIGRATE_RESERVE },
#ifdef CONFIG_CMA
    [MIGRATE_MOVABLE]     = { MIGRATE_CMA,         MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE, MIGRATE_RESERVE },
    [MIGRATE_CMA]         = { MIGRATE_RESERVE }, /* Never used */
#else
    [MIGRATE_MOVABLE]     = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE,   MIGRATE_RESERVE },
#endif
    [MIGRATE_RESERVE]     = { MIGRATE_RESERVE }, /* Never used */
#ifdef CONFIG_MEMORY_ISOLATION
    [MIGRATE_ISOLATE]     = { MIGRATE_RESERVE }, /* Never used */
#endif
};
```

### 关键结构体简介

```c
typedef struct pglist_data {
	/* 本node管辖的zones：ZONE_HIGHMEM, ZONE_NORMAL, ZONE_DMA.*/
	struct zone node_zones[MAX_NR_ZONES];
	/* zone节点备用链表, UMA只有一条，NUMA有两条，在start_kernel->build_all_zonelists->...->build_zonelists中初始化 */
	struct zonelist node_zonelists[MAX_ZONELISTS];
	/* zone数量 */
	int nr_zones;
#ifdef CONFIG_FLAT_NODE_MEM_MAP	/* means !SPARSEMEM */
	/* 指向mem_map属于本节点的第一个页表描述符, struct page数组的第一个page，代表了节点的所有物理帧*/
	struct page *node_mem_map;
#ifdef CONFIG_PAGE_EXTENSION
	struct page_ext *node_page_ext;
#endif
#endif
#ifndef CONFIG_NO_BOOTMEM
    /*
     *指向自举内存分配器数据结构的实例；自举内存分配器(boot memory allocator):
     * 在系统启动期间，内存管理子系统初始化之前，内核也需要使用内存
     * 以及用于初始化内存管理子系统的内存；
     */
	struct bootmem_data *bdata;
#endif
#ifdef CONFIG_MEMORY_HOTPLUG
	spinlock_t node_size_lock;
#endif
	/* 节点第一个页面帧逻辑编号，所有页帧是依次编号的，每个页帧的号码都是全局唯一的。在UMA中总是0 */
	unsigned long node_start_pfn;
	unsigned long node_present_pages; /* 本节点占页帧数目 */
	/* node_spanned_pages=node_end_pfn-node_start_pfn */
	unsigned long node_spanned_pages; /* total size of physical page
					     range, including holes */
	/* 这是从0开始的节点号（NID） */
	int node_id;
	/* 交换守护进程的等待队列 */
	wait_queue_head_t kswapd_wait;
	/* 直接内存回收中的等待队列 */
	wait_queue_head_t pfmemalloc_wait;
	/* 指向该节点的kswapd守护进程，该进程用于释放页面 */
	struct task_struct *kswapd;	/* Protected by
					   mem_hotplug_begin/end() */
	/* 用于页交换子系统的实现，用来定义需要释放的区域的长度 */
	int kswapd_max_order;
	/* ZONE_HIGHMEM, ZONE_NORMAL或 ZONE_DMA */
	enum zone_type classzone_idx;
#ifdef CONFIG_NUMA_BALANCING
	/* Lock serializing the migrate rate limiting window */
	spinlock_t numabalancing_migrate_lock;

	/* Rate limiting time interval */
	unsigned long numabalancing_migrate_next_window;

	/* Number of pages migrated during the rate limiting time interval */
	unsigned long numabalancing_migrate_nr_pages;
#endif
} pg_data_t;
struct zonelist {
	/* 当前node, 每个zone的备份列表. 在当前节点的zone中无可用内存时，会向这些备用节点进行分配 */
	struct zonelist_cache *zlcache_ptr;		     // NULL or &zlcache
	/* 在当前节点中所有zone列表 */
	struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
#ifdef CONFIG_NUMA
	/* 用于优化，通过位图指示相应的zone是否有内存可用 */
	struct zonelist_cache zlcache;			     // optional ...
#endif
}
/* 描述内存节点的备用zone列表描述符，用于性能优化，在高版本内核中，由于加入了直接回收机制，被逐渐的弱化了 */
struct zonelist_cache {
	unsigned short z_to_n[MAX_ZONES_PER_ZONELIST];		/* zone->nid */
	DECLARE_BITMAP(fullzones, MAX_ZONES_PER_ZONELIST);	/* zone full? */
	/* 
	* 最后检查时间，在zlc_setup()会被更新
	* zlc_setup(): 内存分配是跳过不在cpuset中的，以及最近被标识为full的
	*/
	unsigned long last_full_zap;		/* when last zap'd (jiffies) */
}
```

参考资料：

[伙伴系统](https://www.cnblogs.com/tolimit/p/4610974.html)

[节点初始化](https://blog.csdn.net/wyy4045/article/details/81708895)
