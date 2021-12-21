## 前沿

### 往篇回顾

> 在上一篇中，我们主要分析了新建slub的过程,入口函数是`kmem_cache_create`,主要内容如下：

- 首先搜索现有的slub，看看有没有可以重用的，如果有就给合适的slub再取一个别名，refcount+1，然后返回；
- 如果无法找到合适的可复用slub，则通过`do_kmem_cache_create`新建一个slub；
- 新建slub其实并没有为其从伙伴系统申请内存，全部内容围绕这初始化管理结构体kmem_cache进行，主要动作是如下4个：
  1. 申请管理结构体kmem_cache内存；
  2. 根据传入参数size、flag以及系统全局变量`slub_min_objects`, `slub_max_order`通过碎片最小原则确定内存order；
  3. 初始化kmem_cache;
  4. 通过`init_kmem_cache_nodes`及`init_kmem_cache_cpus`申请内存，初始化后挂到kmem_cache响应成员变量上；

### 本篇主要内容

> 在完成kmem create后，使用`kmem_cache_alloc`申请slub对象的流程，简单来说步骤如下：

## 代码分析

### slab_alloc_node

slub内存申请入口函数: kmem_cache_alloc; 外层函数都比较简单，直接看内存的slab_alloc_node

```c
kmem_cache_alloc()-->slab_alloc()-->slab_alloc_node()
/* 
 * 1. 通过cpu_slab->tid来防止多线程重入导致当前获取的cpu_slab已经不是最新状态
 * 2. 如果cpu_slab->freelist不为空 则尝试从freelist中快速分配
 * 3. 否则进入__slab_alloc慢速分配
 */
static __always_inline void *slab_alloc_node(struct kmem_cache *s,
		gfp_t gfpflags, int node, unsigned long addr)
{
	void **object;
	struct kmem_cache_cpu *c;
	struct page *page;
	unsigned long tid;
	
	s = slab_pre_alloc_hook(s, gfpflags);
	if (!s)
		return NULL;
redo:
	/*
	 * 1. 获取tid
	 * 2. 通过__this_cpu_ptr()获取当前CPU的kmem_cache_cpu结构
	 * tid的作用是防止多线程操作导致获取到的slub不是最新状态
	 */
	do {
		tid = this_cpu_read(s->cpu_slab->tid);
		c = raw_cpu_ptr(s->cpu_slab);
	} while (IS_ENABLED(CONFIG_PREEMPT) &&
		 unlikely(tid != READ_ONCE(c->tid)));

	/*
	 * 上面这段注释说明了tid及barrier()的作用,中文解释见附录
	 * barrier:内存屏障
	 */
	barrier();


	object = c->freelist;
	page = c->page;
	/* 判断当前CPU的slab空闲列表是否为空或者当前slab使用内存页面与管理节点是否不匹配 */
	if (unlikely(!object || !node_match(page, node))) {
		/* 进入慢速分配 */
		object = __slab_alloc(s, gfpflags, node, addr, c);
		stat(s, ALLOC_SLOWPATH);
	} else {
		/* 进入快速分配，获取slub中空闲对象地址 */
		void *next_object = get_freepointer_safe(s, object);

		/* 使用this_cpu_cmpxchg_double()原子指令操作更新freelist,tid
		 * this_cpu_cmpxchg_double是一个内核常用函数，可以单独百度搜索其作用：
		 * 1. 检测 s->cpu_slab->freelist==object && s->cpu_slab->tid==tid
		 * 2. 成功则，更新s->cpu_slab->freelist, s->cpu_slab->tid
		 * 3. 否则，返回失败
		 */
		if (unlikely(!this_cpu_cmpxchg_double(
				s->cpu_slab->freelist, s->cpu_slab->tid,
				object, tid,
				next_object, next_tid(tid)))) {
		/* 获取空闲对象失败，则经note_cmpxchg_failure()记录日志后重回redo标签再次尝试分配 */
			note_cmpxchg_failure("slab_alloc", s, tid);
			goto redo;
		}
		/* 刷新结构体数据 */
		prefetch_freepointer(s, next_object);
		stat(s, ALLOC_FASTPATH);
	}
	/* 如果有__GFP_ZERO，则刷新内存 */
	if (unlikely(gfpflags & __GFP_ZERO) && object)
		memset(object, 0, s->object_size);

	slab_post_alloc_hook(s, gfpflags, object);

	return object;
}
```

### __slab_alloc

```c
/* 
 * 1. new_slab中会依次尝试从cpu_slab->partial链表，kmem_cache_node->partial链表尝试分配
 * 2. 如果上面两个地方都失败了，则只能尝试从伙伴系统中再次申请slab
 */
static void *__slab_alloc(struct kmem_cache *s, gfp_t gfpflags, int node,
			  unsigned long addr, struct kmem_cache_cpu *c)
{
	void *freelist;
	struct page *page;
	unsigned long flags;
	/* 关闭本CPU中断 */
	local_irq_save(flags);
#ifdef CONFIG_PREEMPT
	/*
	 * We may have been preempted and rescheduled on a different
	 * cpu before disabling interrupts. Need to reload cpu area
	 * pointer.
	 */
	c = this_cpu_ptr(s->cpu_slab);
#endif
	/* slab内存的page指针,指向正在使用的slab的页帧, 如果page为空，则重新申请slub */
	page = c->page;
	if (!page)
		goto new_slab;
redo:
	/* 如果节点不匹配就通过deactivate_slab()去激活cpu本地slab? */
	if (unlikely(!node_match(page, node))) {
		int searchnode = node;

		if (node != NUMA_NO_NODE && !node_present_pages(node))
			searchnode = node_to_mem_node(node);

		if (unlikely(!node_match(page, searchnode))) {
			stat(s, ALLOC_NODE_MISMATCH);
			deactivate_slab(s, page, c->freelist);
			c->page = NULL;
			c->freelist = NULL;
			goto new_slab;
		}
	}

	/* 通过pfmemalloc_match()判断当前页面属性是否为pfmemalloc? 如果不是则同样去激活 */
	if (unlikely(!pfmemalloc_match(page, gfpflags))) {
	/* 
	 * 1. 将cpu_slab的freelist全部释放回page->freelist
	 * 2. 根据page(slab)的状态进行不同操作:
	 * 		如果该slab有部分空闲对象，则将page移到kmem_cache_node的partial队列;
	 * 		如果该slab全部空闲，则直接释放该slab;
	 * 		如果该slab全部占用，而且开启了CONFIG_SLUB_DEBUG编译选项，则将page移到full队列;
	 * 3. page的状态也从frozen改变为unfrozen(frozen代表slab在cpu_slub，unfroze代表在partial队列或者full队列)
	 */
		deactivate_slab(s, page, c->freelist);
		c->page = NULL;
		c->freelist = NULL;
		goto new_slab;
	}

	/* must check again c->freelist in case of cpu migration or IRQ */
	freelist = c->freelist;
	if (freelist)
		goto load_freelist;
	/* 通过get_freelist()从页面中获取空闲队列 */
	freelist = get_freelist(s, page);
	/* 获取空闲队列失败，此时则需要创建新的slab，否则更新统计信息进入load_freelist分支取得对象并返回 */
	if (!freelist) {
		c->page = NULL;
		stat(s, DEACTIVATE_BYPASS);
		goto new_slab;
	}
	/* 更新统计信息进入load_freelist分支取得对象并返回 */
	stat(s, ALLOC_REFILL);

load_freelist:
	/*
	 * 把对象从空闲队列中取出，并更新数据信息，然后恢复中断使能，返回对象地址
	 */
	VM_BUG_ON(!c->page->frozen);
	c->freelist = get_freepointer(s, freelist);
	c->tid = next_tid(c->tid);
	local_irq_restore(flags);
	return freelist;

new_slab:
	/* partial不为空则从partial中取出，然后跳转回redo重试分配 */
	if (c->partial) {
		page = c->page = c->partial;
		c->partial = page->next;
		stat(s, CPU_PARTIAL_ALLOC);
		c->freelist = NULL;
		goto redo;
	}
	/* 如果partial为空，意味着当前所有的slab都已经满负荷使用，那么则需使用new_slab_objects()创建新的slab 
	 * 该函数中，首先通过get_partial()获取存在空闲对象的slab并将对象返回；
	 * 继而通过new_slab()创建slab，如果创建好slab后，将空闲对象链表摘下并返回
	*/
	freelist = new_slab_objects(s, gfpflags, node, &c);

	if (unlikely(!freelist)) {
		/* 记录日志后使能中断并返回NULL表示申请失败 */
		slab_out_of_memory(s, gfpflags, node);
		local_irq_restore(flags);
		return NULL;
	}

	page = c->page;
	/* 判断是否未开启调试且页面属性匹配pfmemalloc */
	if (likely(!kmem_cache_debug(s) && pfmemalloc_match(page, gfpflags)))
		goto load_freelist;

	/* 若开启调试并且调试初始化失败，则返回创建新的slab */
	if (kmem_cache_debug(s) &&
			!alloc_debug_processing(s, page, freelist, addr))
		goto new_slab;	/* Slab failed checks. Next slab needed */
	/* deactivate_slab()去激活该page，使能中断并返回 */
	deactivate_slab(s, page, get_freepointer(s, freelist));
	c->page = NULL;
	c->freelist = NULL;
	local_irq_restore(flags);
	return freelist;
}
```

### new_slab

```c
/*
 * 申请新的slub
 *
 */
static struct page *new_slab(struct kmem_cache *s, gfp_t flags, int node)
{
	struct page *page;
	void *start;
	void *p;
	int order;
	int idx;

	if (unlikely(flags & GFP_SLAB_BUG_MASK)) {
		pr_emerg("gfp: %u\n", flags & GFP_SLAB_BUG_MASK);
		BUG();
	}
	/* 
	 * 1. 通过allocate_slab_page从伙伴系统申请内存，如果申请失败则降低要求，以kmem_cache->min中阶数申请
	 * 2. 如果开启了kmemcheck检测功能，则需要申请相同数量的shadow page，并初始化其数据
	 * 3. 分配成功则刷新page->objects数量
	*/
	page = allocate_slab(s,
		flags & (GFP_RECLAIM_MASK | GFP_CONSTRAINT_MASK), node);
	if (!page)
		goto out;

	order = compound_order(page);
	inc_slabs_node(s, page_to_nid(page), page->objects);
	page->slab_cache = s;
	__SetPageSlab(page);
	if (page_is_pfmemalloc(page))
		SetPageSlabPfmemalloc(page);

	start = page_address(page);

	if (unlikely(s->flags & SLAB_POISON))
		memset(start, POISON_INUSE, PAGE_SIZE << order);

	kasan_poison_slab(page);

	for_each_object_idx(p, idx, s, start, page->objects) {
		setup_object(s, page, p);
		if (likely(idx < page->objects))
			set_freepointer(s, p, p + s->size);
		else
			set_freepointer(s, p, NULL);
	}

	page->freelist = start;
	page->inuse = page->objects;
	page->frozen = 1;
out:
	return page;
}
```

## 附录

### barrier、tid起到的作用

- 局部变量tid和object、page在cpu看来没有任何依赖关系;
- 因此编译器和CPU的乱序执行都可能会导致先write object、page再write tid;
- 实际必须先write tid，原因是：

假设先写object、page，此时线程thread1被thread2抢占，而thread2恰好在当前CPU上执行了slab_alloc_node，那么c->tid、c->freelist、c->page将被更新，再切回thread1设置局部变量tid = c->tid，随后执行cmpxchg时必将成功，因为 tid == c->tid，而此时thread1的object和page明显不是最新, tid也就发挥不了应有的作用；

而假设先写tid，此时线程thread1被thread2抢占，而thread2恰好在当前CPU上执行了slab_alloc_node，那么c->tid、c->freelist、c->page将被更新，再切回thread1设置局部变量object和page，随后执行cmpxchg时必将失败，因为 tid != c->tid，然后此时我只需要goto redo即可;

## 参考资料

[slub源码分析](https://www.jeanleo.com/2018/09/07/【linux内存源码分析】slub分配算法（4）/)

[图解slub,slub结构体描述](http://www.wowotech.net/memory_management/426.html)
