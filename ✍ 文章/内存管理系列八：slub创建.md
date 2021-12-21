## 前沿

### 往篇回顾

在上一篇中，我们正式进入slub系统的学习，主要分析了slub系统初始化的过程，关键函数是`start_kernel()->mm_init()->kmem_cache_init()`，在该函数中依次完成了以下动作：

- 完成slub管理框架内存申请，即两个关键slub：”kmem_cache”, “kmem_cache_node”;
- 使用`bootstrap`函数刷新各个内存node节点中struct page->slab_cache指针；
- 在`create_kmalloc_caches`中，完成size_index和kmem_caches映射，并使用`create_kmalloc_cache`初始化kmem_caches结构体数组；

### 本篇主要内容

> 本篇主要分析slub分配器申请过程

## 代码分析

### kmem_cache_create

```c
/*
 * kmem_cache_create - Create a cache.
 * @name: 用于/proc/slabinfo文件中显示此高速缓冲的字符串
 * @size: 要创建的slub中每个对象的大小
 * @align: 对象对齐偏移量
 * @flags: 对应slab的标志
 * @ctor: 构建对象构造函数
 * 
 * 函数成功时返回指向cache的指针，失败时返回NULL.
 * 中断中不允许调用，但是调用过程中可以被中断.
 * 当针对cache的新的页框分配成功时运行ctor构造函数.
 *
 */
struct kmem_cache *
kmem_cache_create(const char *name, size_t size, size_t align,
		  unsigned long flags, void (*ctor)(void *))
{
	struct kmem_cache *s;
	const char *cache_name;
	int err;

	get_online_cpus();
	get_online_mems();
	memcg_get_cache_ids();

	mutex_lock(&slab_mutex);

	err = kmem_cache_sanity_check(name, size);
	if (err) {
		s = NULL;	/* suppress uninit var warning */
		goto out_unlock;
	}

	flags &= CACHE_CREATE_MASK;
	/* 
	 * 调用find_mergeable查找是否有能复用的kmem_cache,全局变量slab_unmerge被置位时不允许复用：
	 * list_for_each_entry_reverse会遍历slab_cache全局链表，检查以下几项
	 * 1. 两者的size要足够接近，差距小于sizeof(void *)；且现有的要大些，当然新创建的size要经过L1对齐
	 * 2. flags标志
	 * 3. 两个kemem_cache对齐要一致
	 *
	 * 如果找到合适的复用kmem_cache，则kmem_cache->refcount++，refcount反映kmem_cache的复用次数
     * 最后调用sysfs_slab_alias在sysfs中添加别名
	 */
	s = __kmem_cache_alias(name, size, align, flags, ctor);
	if (s)
		goto out_unlock;
	/* 
	 * 使用GFP_KERNEL模式申请内存，将形参name的内容copy到这段新申请的内存中
	 * 如果name本身在rdata数据区，则不复制
	 */
	cache_name = kstrdup_const(name, GFP_KERNEL);
	if (!cache_name) {
		err = -ENOMEM;
		goto out_unlock;
	}
	/* 
	 * 将主要参数配置到slab描述符，然后将得到的描述符加入slab_caches全局链表中，其中：
	 * 1. kmem_cache_zalloc：使用GFP_KERNEL, 为kmem_cache结构体申请内存
     * 2. 初始化kmem_cache结构体数据
	 * 3. init_memcg_params初始化kmem_cache的memcg_cache_params参数，用于表示该slub是属于哪个用户的？
	 * 4. __kmem_cache_create： 创建slub描述符的核心：对齐操作、计算需要的页面、对象数目、对slab着色等，后面会进一步分析
	 * 
	 */
	s = do_kmem_cache_create(cache_name, size, size,
				 calculate_alignment(flags, align, size),
				 flags, ctor, NULL, NULL);
	if (IS_ERR(s)) {
		err = PTR_ERR(s);
		kfree_const(cache_name);
	}

out_unlock:
	mutex_unlock(&slab_mutex);

	memcg_put_cache_ids();
	put_online_mems();
	put_online_cpus();

	if (err) {
		if (flags & SLAB_PANIC)
			panic("kmem_cache_create: Failed to create slab '%s'. Error %d\n",
				name, err);
		else {
			printk(KERN_WARNING "kmem_cache_create(%s) failed with error %d",
				name, err);
			dump_stack();
		}
		return NULL;
	}
	return s;
}
```

### __kmem_cache_create

```c
/*
 * 初始化slub结构
 */
int __kmem_cache_create(struct kmem_cache *s, unsigned long flags)
{
	int err;

	err = kmem_cache_open(s, flags);
	if (err)
		return err;

	/* Mutex is not taken during early boot */
	if (slab_state <= UP)
		return 0;

	memcg_propagate_slab_attrs(s);
    /* 将kmem_cache添加到sysfs */
	err = sysfs_slab_add(s);
	if (err)
    /*如果出错则销毁slub*/
		kmem_cache_close(s);

	return err;
}
```

### kmem_cache_open

```c
static int kmem_cache_open(struct kmem_cache *s, unsigned long flags)
{
/*根据slub_debug标志看是否要更新flags*/
	s->flags = kmem_cache_flags(s->size, flags, s->name, s->ctor);
	s->reserved = 0;

	if (need_reserve_slab_rcu && (s->flags & SLAB_DESTROY_BY_RCU))
		s->reserved = sizeof(struct rcu_head);
	/* 
	 * 计算kmem_cache中相关参数，并初始化s， 主要是slub要申请的内存阶数
	 * 1. 根据编译选项确定是否需要一些debug空间，用以确定最终每个object的size
	 * 2. 通过calculate_order计算实际需要申请内存order阶数
	 * 3. 初始化s中相关成员变量
	 */
	if (!calculate_sizes(s, -1))
		goto error;
	if (disable_higher_order_debug) {
		/*
		 * Disable debugging flags that store metadata if the min slab
		 * order increased.
		 */
		if (get_order(s->size) > get_order(s->object_size)) {
			s->flags &= ~DEBUG_METADATA_FLAGS;
			s->offset = 0;
			if (!calculate_sizes(s, -1))
				goto error;
		}
	}

#if defined(CONFIG_HAVE_CMPXCHG_DOUBLE) && \
    defined(CONFIG_HAVE_ALIGNED_STRUCT_PAGE)
	if (system_has_cmpxchg_double() && (s->flags & SLAB_DEBUG_FLAGS) == 0)
		/* Enable fast mode */
		s->flags |= __CMPXCHG_DOUBLE;
#endif

	/* 设置partial链表的最小值；避免当obj size过大时，对页面分配器造成冲击 */
	set_min_partial(s, ilog2(s->size) / 2);

	/*
	 * 初始化cpu_partial，该变量含义见附录
	 */
	if (!kmem_cache_has_cpu_partial(s))
		s->cpu_partial = 0;
	else if (s->size >= PAGE_SIZE)
		s->cpu_partial = 2;
	else if (s->size >= 1024)
		s->cpu_partial = 6;
	else if (s->size >= 256)
		s->cpu_partial = 13;
	else
		s->cpu_partial = 30;

#ifdef CONFIG_NUMA
	s->remote_node_defrag_ratio = 1000;
#endif
	if (!init_kmem_cache_nodes(s))
		goto error;
	/* 
	 * 1. 通过__alloc_percpu()为每个CPU申请空间
	 * 2. 通过init_kmem_cache_cpus()将申请空间初始化至每个CPU上
   	 */
	if (alloc_kmem_cache_cpus(s))
		return 0;
	/* 走到这则说明cpu_slab初始化失败了 */
	free_kmem_cache_nodes(s);
error:
	if (flags & SLAB_PANIC)
		panic("Cannot create slab %s size=%lu realsize=%u "
			"order=%u offset=%u flags=%lx\n",
			s->name, (unsigned long)s->size, s->size,
			oo_order(s->oo), s->offset, flags);
	return -EINVAL;
}
```

### calculate_order

```c
/* 主要是计算slab所需页面的阶数；然后使用计算出的数据初始化kmem_cache */
static inline int calculate_order(int size, int reserved)
{
	int order;
	int min_objects;
	int fraction;
	int max_objects;

	/*
	 * First we reduce the acceptable waste in a slab. Then
	 * we reduce the minimum objects required in a slab.
	 */
	/* 确定最小objects数量 */
	min_objects = slub_min_objects;
	if (!min_objects)
		min_objects = 4 * (fls(nr_cpu_ids) + 1);
	/* 计算最大objects数量 */
	max_objects = order_objects(slub_max_order, size, reserved);
	min_objects = min(min_objects, max_objects);
	/* 
	 * 接着通过两个while循环，分别对min_objects及fraction进行调整，通过slab_order()计算找出最佳的阶数，其中fraction用来表示slab内存未使用率的指标(碎片因子)，
	 * 目标：不断调整单个slab的对象数以及降低碎片指标，由此找到一个最佳值
	 * 要满足：碎片部分乘以fraction小于slab大小 
	 * (slab_size - reserved) % size <= slab_size / fraction
	 */
	while (min_objects > 1) {
		fraction = 16;
		while (fraction >= 4) {
			order = slab_order(size, min_objects,
					slub_max_order, fraction, reserved);
			if (order <= slub_max_order)
				return order;
			fraction /= 2;
		}
		min_objects--;
	}

	/*
	 * 如果obj个数及碎片因子都调整到最低了仍得不到最佳阶值时
	 * 尝试一个slab仅放入单个对象，由此计算出的order不大于slub_max_order，则将该值返回
	 */
	order = slab_order(size, 1, slub_max_order, 1, reserved);
	if (order <= slub_max_order)
		return order;

	/*
	 * 将阶数值调整至最大值MAX_ORDER,再尝试
	 */
	order = slab_order(size, 1, MAX_ORDER, 1, reserved);
	if (order < MAX_ORDER)
		return order;
	return -ENOSYS;
}
```

### slab_order

```c
/* 
 * 寻找合适的阶数，给slab分配内存： 
 * 1. 如果使用最低阶数slub_min_order，就能产生超过MAX_OBJS_PER_PAGE个对象，则简单
 * 2. 循环递增阶数：从min_order及min_objects需要的内存大小开始(min_objects是上层函数while循环的，每次递减)
 * 		满足：(slab_size - reserved) % size <= slab_size / fraction 即可(为减少内存碎片)
 */
static inline int slab_order(int size, int min_objects,
				int max_order, int fract_leftover, int reserved)
{
	int order;
	int rem;
	int min_order = slub_min_order;
	/*
	 * MAX_OBJS_PER_PAGE(32767): 内存页面存储对象个数使用的objects是u15的长度
	 * 所以如果order_objects()以min_order换算内存大小剔除reserved后，对象个数大于MAX_OBJS_PER_PAGE，则改为MAX_OBJS_PER_PAGE进行求阶 
	 */
	if (order_objects(min_order, size, reserved) > MAX_OBJS_PER_PAGE)
		return get_order(size * MAX_OBJS_PER_PAGE) - 1;
	/* 通过for循环，调整阶数以期找到一个能够容纳该大小最少对象数量及其保留空间的并且内存的使用率满足条件的阶数 */
	for (order = max(min_order,
				fls(min_objects * size - 1) - PAGE_SHIFT);
			order <= max_order; order++) {

		unsigned long slab_size = PAGE_SIZE << order;

		if (slab_size < min_objects * size + reserved)
			continue;

		rem = (slab_size - reserved) % size;

		if (rem <= slab_size / fract_leftover)
			break;

	}

	return order;
}
```

### init_kmem_cache_nodes

```c
/* 
 * 通过for_each_node_state遍历每个node节点
 * 1. 向kmem_cache_node全局管理控制slub为所遍历的节点申请一个kmem_cache_node结构空间对象
 * 2. 并将kmem_cache的s内的成员node初始化 
 */
static int init_kmem_cache_nodes(struct kmem_cache *s)
{
	int node;
	for_each_node_state(node, N_NORMAL_MEMORY) {
		struct kmem_cache_node *n;
	 /* 
	 * slab_state如果是DOWN状态，表示slub分配器还没有初始化完毕
	 * 1. kmem_cache_node结构空间对象的cache还没建立，暂时无法进行对象分配
	 * 2. 此时将会通过early_kmem_cache_node_alloc()进行kmem_cache_node对象的slab进行创建
	 */
		if (slab_state == DOWN) {
			early_kmem_cache_node_alloc(node);
			continue;
		}
		n = kmem_cache_alloc_node(kmem_cache_node,
						GFP_KERNEL, node);
		/* 申请失败则释放s */
		if (!n) {
			free_kmem_cache_nodes(s);
			return 0;
		}
		/* 将申请的node加入到s的成员变量node中，并初始化结构体 */
		s->node[node] = n;
		init_kmem_cache_node(n);
	}
	return 1;
}
```

## 附录

### kmem_cache结构体

```c
/*
 * Slab cache management.
 */
struct kmem_cache {
	/* 一个per cpu变量，对于每个cpu来说，相当于一个本地内存缓存池。当分配内存的时候优先从本地cpu分配内存以保证cache的命中率。 */
	struct kmem_cache_cpu __percpu *cpu_slab;
	/* Used for retriving partial slabs etc */
	unsigned long flags;
	/* 每个node节点中部分空slab缓冲区数量不能低于这个值 */
	unsigned long min_partial;
	int size;		/* 每个对象的实际大小(包括后面的对象指针meta data)，是各种地址对齐后的结果 */
	int object_size;	/* 单纯对象大小 */
	int offset;		/* offset就是存储下个object地址数据相对于这个object首地址的偏移 */
	/* per cpu partial中所有slab的free obj数量最大值，超过该值就会将所有slab转移到kmem_cache_node的partial链表 */
	int cpu_partial;	
	/* 
	 * 低16位代表一个slab中所有object的数量(oo & ((1 << 16) - 1))
	 * 高16位代表一个slab管理的page数量((2^((oo>>16) &  ((1 << 16) - 1))) pages)
	 * order与object数量的关系非常简单：((PAGE_SIZE << order) - reserved) / size
	 */
	struct kmem_cache_order_objects oo;

	/* 在slab缓冲区压缩时使用，kmeme_cache中所有slab缓冲区的objects数量都不会大于max中的值 */
	struct kmem_cache_order_objects max;
	/* 当按照oo大小分配内存, 出现内存不足使用min方式分配。min只需要可以容纳一个object即可 */
	struct kmem_cache_order_objects min;
	gfp_t allocflags;	/* 从伙伴系统分配内存掩码 */
	int refcount;		/* slub重用计数器，slub destory时会判断 */
	void (*ctor)(void *);
	int inuse;		/* 空闲元数据偏移量 */
	int align;		/* 对齐标准 */
	int reserved;		/* Reserved bytes at the end of slabs */
	const char *name;	/* Name (only for display!) */
	struct list_head list;	/* 所有的 kmem_cache 结构都会链入这个链表，链表头是 slab_caches */
#ifdef CONFIG_SYSFS
	struct kobject kobj;	 /* 用于sysfs文件系统，在/sys中会有个slub的专用目录 */
#endif
#ifdef CONFIG_MEMCG_KMEM
	struct memcg_cache_params memcg_params;
	int max_attr_size; /* 用于NUMA架构，该值越小，越倾向于在本结点分配对象 */
#ifdef CONFIG_SYSFS
	struct kset *memcg_kset;
#endif
#endif

#ifdef CONFIG_NUMA
	/*
	 * Defragmentation by allocating from a remote node.
	 */
	int remote_node_defrag_ratio;
#endif
/* 此高速缓存的SLAB链表，每个NUMA结点有一个，有可能该高速缓存有些SLAB处于其他结点上 */
	struct kmem_cache_node *node[MAX_NUMNODES];
};
```

### kmem_cache_cpu结构体

```c
struct kmem_cache_cpu { 
    void **freelist;    /* 指向下一个可用的object */
    unsigned long tid;  /* 全局唯一，检查是否有并发,递增,没分配一次+1 */
    struct page *page;  /* slab内存的page指针,指向正在使用的slab的页帧 */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	/* 本地slab partial链表, 放到CPU的部分空slab链表中的slab会被冻结 */
    struct page *partial;   
#endif
}; 
```

### kmem_cache_node结构体

```c
/* SLAB链表结构 */
struct kmem_cache_node {
    /* 锁 */
    spinlock_t list_lock;

/* SLAB用 */
#ifdef CONFIG_SLAB
    /* 只使用了部分对象的SLAB描述符的双向循环链表 */
    struct list_head slabs_partial;    /* partial list first, better asm code */
    /* 不包含空闲对象的SLAB描述符的双向循环链表 */
    struct list_head slabs_full;
    /* 只包含空闲对象的SLAB描述符的双向循环链表 */
    struct list_head slabs_free;
    /* 高速缓存中空闲对象个数(包括slabs_partial链表中和slabs_free链表中所有的空闲对象) */
    unsigned long free_objects;
    /* 高速缓存中空闲对象的上限 */
    unsigned int free_limit;
    /* 下一个被分配的SLAB使用的颜色 */
    unsigned int colour_next;    /* Per-node cache coloring */
    /* 指向这个结点上所有CPU共享的一个本地高速缓存 */
    struct array_cache *shared;    /* shared per node */
    struct alien_cache **alien;    /* on other nodes */
    /* 两次缓存收缩时的间隔，降低次数，提高性能 */
    unsigned long next_reap;    
    /* 0:收缩  1:获取一个对象 */
    int free_touched;        /* updated without locking */
#endif

/* SLUB用 */
#ifdef CONFIG_SLUB
	/* slab节点中slab的数量 */
    unsigned long nr_partial;
	/* 只使用了部分对象的SLAB描述符的双向循环链表 */
    struct list_head partial;
#ifdef CONFIG_SLUB_DEBUG
    atomic_long_t nr_slabs;
    atomic_long_t total_objects;
    struct list_head full;
#endif
#endif

};
```

## 参考资料

[slub debug](http://www.wowotech.net/memory_management/427.html)

[slub create代码解析](http://blog.chinaunix.net/uid-25940216-id-3202940.html)

[图解slub,slub结构体描述](http://www.wowotech.net/memory_management/426.html)

[slub 分配器描述](https://www.cnblogs.com/tolimit/p/4566189.html#)

[slub及slab区别](https://blog.csdn.net/wh8_2011/article/details/52287557)
