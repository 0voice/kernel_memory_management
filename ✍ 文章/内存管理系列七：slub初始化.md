## 前沿

### 往篇回顾

在前两篇中，主要介绍了alloc_pages正常情况下，如果从伙伴系统获得内存；简单来说有如下几步：

- 正常情况下，alloc_pages->get_page_from_freelist会使用low阀值遍历zonelist尝试分配, 分两次遍历，首先尝试只从preferred_zone所在node中的zone分配;
  1. 如果某个zone检查水位不足，则会触发起进行内存回收zone_reclaim后，再尝试检查其水位是否符合要求；
- 在get_page_from_freelist中，如果某个zone经过区间、水位校验通过后调用buffered_rmqueue申请内存；
- 在buffered_rmqueue会区分order
  1. order=0，尝试从CPU缓存中分配， 如果CPU缓存中没有空闲内存，则使用rmqueue_bulk从伙伴系统中申请bulk个order=0的空闲内存；
  2. order>0, 尝试从zone的伙伴系统中分配内存__rmqueue(), 从free_list[order]开始找空闲内存(__rmqueue_smallest)，找不到则尝试更高一阶，直到找到为止；
- 如果使用__rmqueue()从free_list中获取空闲页失败，则调用__rmqueue_fallback从migratetype的fallback列表中依次尝试分配；
  1. 为了反碎片，从备用mirgratetype中获取到的内存会首先尝试移动到希望的mirgratype；
  2. 从备用mirgratetype获取内存，是从高阶order=10到低阶进行尝试的；这种机制应该也是为了反碎片；
- 如果以上都没有成功，则会进行第二次get_page_from_freelist, 这次尝试所有zonelist中的zone；
- 如果第二次遍历zonelist也失败，则会触发慢速分配__alloc_pages_slowpath，并修改水位阀值为min；

### slub简介

#### 与slab的比较

- slub来源于slab，slab对于小内存分配管理有不错的表现；但是对于大型的NUMA系统，slab的管理开销会成倍增大，导致异常臃肿；
- slub摒弃了slab的一些管理数据，以及着色概念(为了避免CPU硬件高速缓存频繁换入换出，在一个kmem_cache中的slab的前端空闲区域嵌入空闲偏移);
  - node结点对应的 kmem_cache_node->colour_next * kmem_cache->colour_off 就得到了偏移量(kmem_cache.colour_off * kmem_cache.node[NODE_ID].colour_next);
  - colour_next++，当colour_next等于kmem_cache中的colour时，colour_next置0(kmem_cache.node[NODE_ID].colour_next++);
- slab根据状态划分了3个链表–full,partial和free. slub分配器做了简化，去掉了free链表，对于空闲的slab,slub分配器选择直接将其释放；
- 在NUMA架构上，slub分配器较slab做了简化；

### 本篇主要内容

> 本篇主要分析slub分配器初始化过程，该阶段的主要任务是初始化用于kmalloc的gerneral cache：struct kmem_cache *kmalloc_caches[KMALLOC_SHIFT_HIGH + 1];其中KMALLOC_SHIFT_HIGH=14

## 代码分析

### size_index数组及kmem_caches数据关系

- size_index数组中存储的是gerneral cache的编号；
- 要根据Object size大小来定位合适的gerneral cache，直接用size/8 - 1;
  1. 例如：size=48 ===> 对应size_index的下标=49/8-1=5;
  2. size_index[5] == 6， 对应kmem_cache[6], 其中的slub size 为 64;
  3. 即：如果你要申请一个object size=49的slub，真正给你返回的是一个object size=64的slub；
- linux在Mem_init()函数中完成bootmem到伙伴系统的转移后，随即开始slub初始化

### kmem_cache_init

```c
start_kernel->mm_init()->kmem_cache_init

/*
 * 1. 通过create_boot_cache创建"kmem_cache", "kmem_cache_node" slab,后续结构体struct kmem_cache, struct kmem_cache_node的内存申请都会分别通过这两个slab完成;
 * 2. register_hotmemory_notifier()注册热插拔内存内核通知链回调函数用于热插拔内存处理?
 * 3. 完成"kmem_cache", "kmem_cache_node" slab创建后，要刷新各个node中slab的相应指针;
 * 4. 在create_kmalloc_caches完成 size_index和kmem_caches的映射，并对kmem_cache结构体指针用create_kmalloc_caches创建slab, 进行初始化;
 * 5. 置slab_state=up;
 */ 
void __init kmem_cache_init(void)
{
	//声明静态变量，存储临时kmem_cache管理结构
	static __initdata struct kmem_cache boot_kmem_cache,
		boot_kmem_cache_node;

	if (debug_guardpage_minorder())
		slub_max_order = 0;

	kmem_cache_node = &boot_kmem_cache_node;
	kmem_cache = &boot_kmem_cache;
	/* 
	 * 创建一个slub, size: sizeof(struct kmem_cache_node), name: "kmem_cache_node", 置引用次数refcount=-1
	 * 其中核心函数__kmem_cache_create在下一篇中分析，其关键作用是把kmem_cache结构初始化
	 * 申请slub缓冲区，管理数据放在临时结构体中
	 */
	create_boot_cache(kmem_cache_node, "kmem_cache_node",
		sizeof(struct kmem_cache_node), SLAB_HWCACHE_ALIGN);
	/* 注册热插拔内存内核通知链回调函数用于热插拔内存处理，注册到memory_chain上 */
	register_hotmemory_notifier(&slab_memory_callback_nb);

	/* 将初始化进度改为PARTIAL,表示已经可以分配struct kmem_cache_node */
	slab_state = PARTIAL;

	create_boot_cache(kmem_cache, "kmem_cache",
			offsetof(struct kmem_cache, node) +
				nr_node_ids * sizeof(struct kmem_cache_node *),
		       SLAB_HWCACHE_ALIGN);
	/* 
	 * boot_kmem_cache和boot_kmem_cache_node中的内容拷贝到新申请的对象中,并修正其余node slab中相关指针
	 * 从而完成了struct kmem_cache和struct kmem_cache_node管理结构的bootstrap(自引导)
	 */
	kmem_cache = bootstrap(&boot_kmem_cache);

	kmem_cache_node = bootstrap(&boot_kmem_cache_node);

	/* Now we can use the kmem_cache to allocate kmalloc slabs */
	create_kmalloc_caches(0);

#ifdef CONFIG_SMP
	/* 注册内核通知链回调函数，注册到cpu_chain上 */
	register_cpu_notifier(&slab_notifier);
#endif

	pr_info("SLUB: HWalign=%d, Order=%d-%d, MinObjects=%d, CPUs=%d, Nodes=%d\n",
		cache_line_size(),
		slub_min_order, slub_max_order, slub_min_objects,
		nr_cpu_ids, nr_node_ids);
}
```

### bootstrap

```c
/*
 * 从刚才挂在临时结构的缓冲区中申请kmem_cache的kmem_cache，并将管理数据拷贝到新申请的内存中
 * 将临时kmem_cache向最终kmem_cache迁移，并修正其余node slab中相关指针，使其指向最终kmem_cache 
 */
static struct kmem_cache * __init bootstrap(struct kmem_cache *static_cache)
{
	int node;
	/* 
	 * kmem_cache_zalloc()->kmem_cache_alloc()->slab_alloc(), slab_alloc函数后面会分析
	 * 为create_boot_cache()初始化创建的kmem_cache申请slub空间
	 */
	struct kmem_cache *s = kmem_cache_zalloc(kmem_cache, GFP_NOWAIT);
	struct kmem_cache_node *n;
	/* 将bootstrap()入参的kmem_cache结构数据memcpy()至申请的空间中 */
	memcpy(s, static_cache, kmem_cache->object_size);

	/*
	 * 刷新cpu的slab信息
	 */
	__flush_cpu_slab(s, smp_processor_id());
	/*
	 * 要将新的kmem_cache地址刷新到各个内存管理节点node的slab中
	 * 通过for_each_kmem_cache_node()遍历各个内存管理节点node，获取各个节点的kmem_cache_node,如果不为空：
	 * 遍历其中部分满的slab链表，修正每个slab指向kmem_cache的指针；
	 * 如果开启debug，则对满的slab链表也遍历
	 */
	for_each_kmem_cache_node(s, node, n) {
		struct page *p;

		list_for_each_entry(p, &n->partial, lru)
			p->slab_cache = s;

#ifdef CONFIG_SLUB_DEBUG
		list_for_each_entry(p, &n->full, lru)
			p->slab_cache = s;
#endif
	}
	slab_init_memcg_params(s);
	/* 将kmem_cache添加到全局slab_caches链表中 */
	list_add(&s->list, &slab_caches);
	return s;
}
```

### create_kmalloc_caches

```c
/*
 * 1. 根据KMALLOC_MIN_SIZE大小，对size_index全局数组中数据进行改变
 * 		对于slub分配算法而言，KMALLOC_MIN_SIZE为1 << KMALLOC_SHIFT_LOW，其中KMALLOC_SHIFT_LOW为3，则KMALLOC_MIN_SIZE为8
 * 		因此size_index中数据不会改变
 * 2. 循环调用 create_kmalloc_cache 初始化 kmalloc_caches 结构体数组====>到这，完成了size_index和kmalloc_caches的映射
 * 3. 创建完后，将slab_state置为up
 * 4. 将kmem_cache的name成员进行初始化
 * 5. 如果配置了CONFIG_ZONE_DMA， 则初始化创建kmalloc_dma_caches
 */
void __init create_kmalloc_caches(unsigned long flags)
{
	int i;

	/*  保证kmalloc允许的最小对象大小不能大于256，且该值必须是2的整数幂 */
	BUILD_BUG_ON(KMALLOC_MIN_SIZE > 256 ||
		(KMALLOC_MIN_SIZE & (KMALLOC_MIN_SIZE - 1)));
	/* 对大小在8byte与KMALLOC_MIN_SIZE之间的对象，将其在size_index数组的索引设置为KMALLOC_SHIFT_LOW */
	for (i = 8; i < KMALLOC_MIN_SIZE; i += 8) {
		int elem = size_index_elem(i);

		if (elem >= ARRAY_SIZE(size_index))
			break;
		size_index[elem] = KMALLOC_SHIFT_LOW;
	}
	
	if (KMALLOC_MIN_SIZE >= 64) {
		/* KMALLOC_MIN_SIZE=8,不会进入该分支 */
		for (i = 64 + 8; i <= 96; i += 8)
			size_index[size_index_elem(i)] = 7;

	}

	if (KMALLOC_MIN_SIZE >= 128) {
		 /* KMALLOC_MIN_SIZE=8,不会进入该分支 */
		for (i = 128 + 8; i <= 192; i += 8)
			size_index[size_index_elem(i)] = 8;
	}
	/* 
	 * 循环调用 create_kmalloc_cache 初始化 kmalloc_caches 结构体数组
	 * KMALLOC_SHIFT_HIGH=12,但是size_index最大只到7阶，高阶的怎么处理？
	 * create_kmalloc_cache:
	 * 	1. 通过kmem_cache_zalloc申请一个kmem_cache对象
	 * 	2. 通过create_boot_cache()创建slab
	 * 	3. 将创建的slab添加到slab_caches中
	*/
	for (i = KMALLOC_SHIFT_LOW; i <= KMALLOC_SHIFT_HIGH; i++) {
		if (!kmalloc_caches[i]) {
			kmalloc_caches[i] = create_kmalloc_cache(NULL,
							1 << i, flags);
		}
		if (KMALLOC_MIN_SIZE <= 32 && !kmalloc_caches[1] && i == 6)
			kmalloc_caches[1] = create_kmalloc_cache(NULL, 96, flags);

		if (KMALLOC_MIN_SIZE <= 64 && !kmalloc_caches[2] && i == 7)
			kmalloc_caches[2] = create_kmalloc_cache(NULL, 192, flags);
	}

	/* Kmalloc array is now usable */
	slab_state = UP;
	/* 将kmem_cache的name成员进行初始化 */
	for (i = 0; i <= KMALLOC_SHIFT_HIGH; i++) {
		struct kmem_cache *s = kmalloc_caches[i];
		char *n;

		if (s) {
			n = kasprintf(GFP_NOWAIT, "kmalloc-%d", kmalloc_size(i));

			BUG_ON(!n);
			s->name = n;
		}
	}

#ifdef CONFIG_ZONE_DMA
	for (i = 0; i <= KMALLOC_SHIFT_HIGH; i++) {
		struct kmem_cache *s = kmalloc_caches[i];

		if (s) {
			int size = kmalloc_size(i);
			char *n = kasprintf(GFP_NOWAIT,
				 "dma-kmalloc-%d", size);

			BUG_ON(!n);
			kmalloc_dma_caches[i] = create_kmalloc_cache(n,
				size, SLAB_CACHE_DMA | flags);
		}
	}
#endif
}
#endif /* !CONFIG_SLOB */
```

