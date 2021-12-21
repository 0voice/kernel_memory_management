## 前沿

### 往篇回顾

> 在上一篇中，我们主要分析了申请slub内存的过程,入口函数是`kmem_cache_alloc`,主要内容如下：

- kmem_cache刚刚建立，还没有任何对象可供分配，此时只能从伙伴系统分配一个slab；
- 如果正在使用的slab有free obj，那么就直接分配即可，这种是最简单快捷的；
- 随着正在使用的slab中obj的一个个分配出去，最终会无obj可分配，此时per cpu partial链表中有可用slab用于分配，那么就会从per cpu partial链表中取下一个slab用于分配obj；
- 如果per cpu partial链表也为空，此时发现per node partial链表中有可用slab用于分配，那么就会从per node partial链表中取下一个slab用于分配obj；
- 最后还是不行只能从伙伴系统再申请一个slab；

### 本篇主要内容

> 前面分析了Slub分配算法的缓存区创建及对象分配，现继续分配算法的对象回收，入口函数`kmem_cache_free`

## 代码分析

### kmem_cache_free

```c
void kmem_cache_free(struct kmem_cache *s, void *x)
{
	/* 用于获取回收对象的kmem_cache */
	s = cache_from_obj(s, x);
	if (!s)
		return;
	/* 用于将对象回收 */
	slab_free(s, virt_to_head_page(x), x, _RET_IP_);
	/* 对对象的回收做轨迹跟踪 */
	trace_kmem_cache_free(_RET_IP_, x);
}
```

### slab_free

```c
static __always_inline void slab_free(struct kmem_cache *s,
			struct page *page, void *x, unsigned long addr)
{
	void **object = (void *)x;
	struct kmem_cache_cpu *c;
	unsigned long tid;
	/* 释放处理钩子调用处理，主要是用于去注册kmemleak中的对象？ */
	slab_free_hook(s, x);

redo:
	/*
	 * 如果目前执行代码的CPU，要释放的缓冲区所属的CPU不是同一个(通过tid判断)；
	 * 则不停循环等待抢占；等待切换到同一个CPU，否则this_cpu_cmpxchg_double会失败；
	 */
	do {
		tid = this_cpu_read(s->cpu_slab->tid);
		c = raw_cpu_ptr(s->cpu_slab);
	} while (IS_ENABLED(CONFIG_PREEMPT) &&
		 unlikely(tid != READ_ONCE(c->tid)));

	/* Same with comment on barrier() in slab_alloc_node() */
	barrier();
	/* 如果当前释放的对象与本地CPU的缓存区相匹配，设置该对象尾随的空闲对象指针数据 */
	if (likely(page == c->page)) {
		set_freepointer(s, object, c->freelist);
		/* 快速归还对象？ */
		if (unlikely(!this_cpu_cmpxchg_double(
				s->cpu_slab->freelist, s->cpu_slab->tid,
				c->freelist, tid,
				object, next_tid(tid)))) {
			
			note_cmpxchg_failure("slab_free", s, tid);
			goto redo;
		}
		stat(s, FREE_FASTPATH);
	} else
	/* 否则通过慢速通道释放对象 */
		__slab_free(s, page, x, addr);

}
```

### __slab_free

```c
static void __slab_free(struct kmem_cache *s, struct page *page,
			void *x, unsigned long addr)
{
	void *prior;
	void **object = (void *)x;
	int was_frozen;
	struct page new;
	unsigned long counters;
	struct kmem_cache_node *n = NULL;
	unsigned long uninitialized_var(flags);

	stat(s, FREE_SLOWPATH);
	/* 
	 *判断是否开启调试，如果开启
	 * 1. 通过free_debug_processing进行调试检测
	 * 2. 获取经检验过的合法的kmem_cache_node节点缓冲区管理结构
	*/
	if (kmem_cache_debug(s) &&
		!(n = free_debug_processing(s, page, x, addr, &flags)))
		return;

	do {
		if (unlikely(n)) {
			/* 释放在free_debug_processing中加的锁 */
			spin_unlock_irqrestore(&n->list_lock, flags);
			n = NULL;
		}
		/* 获取缓冲区的信息以及设置对象末尾的空闲对象指针，同时更新缓冲区中对象使用数 */
		prior = page->freelist;
		counters = page->counters;
		set_freepointer(s, object, prior);/* 设置对象末尾空闲对象指针? */
		new.counters = counters;
		was_frozen = new.frozen;
		new.inuse--;
		/* 
		 * 如果缓冲区中被使用的对象为0或者空闲队列为空，且缓冲区未处于冻结态(即缓冲区未处于每CPU对象缓存中):
		 * 该释放的对象是缓冲区中最后一个被使用的对象，对象释放之后的缓冲区是可以被释放回伙伴管理算法的 
		 */
		if ((!new.inuse || !prior) && !was_frozen) {

			if (kmem_cache_has_cpu_partial(s) && !prior) {

				/*
				 * 每CPU存在partial半满队列同时空闲队列不为空
				 * 那么该缓冲区将会设置frozen标识，用于后期将其放置到每CPU的partial队列中
				 */
				new.frozen = 1;

			} else { /* 该缓冲区将会从链表中移出 */
				/* 获取节点缓冲区管理结构 */
				n = get_node(s, page_to_nid(page));
				/*
				 * Speculatively acquire the list_lock.
				 * If the cmpxchg does not succeed then we may
				 * drop the list_lock without any processing.
				 *
				 * Otherwise the list_lock will synchronize with
				 * other processors updating the list of slabs.
				 */
				spin_lock_irqsave(&n->list_lock, flags);

			}
		}

	} while (!cmpxchg_double_slab(s, page,
		prior, counters,
		object, new.counters,
		"__slab_free"));/* 通过cmpxchg_double_slab()将对象释放 */

	if (likely(!n)) {

		/*
		 * 如果刚冻结该缓冲区，则把该缓冲区put_cpu_partial()挂入到每CPU的partial队列中
		 */
		if (new.frozen && !was_frozen) {
			put_cpu_partial(s, page, 1);
			stat(s, CPU_PARTIAL_FREE);
		}
		/*
		 * 该缓冲区本来就是冻结的
		 */
		if (was_frozen)
			stat(s, FREE_FROZEN);
		return;
	}
	/* 
	 * 如果缓冲区无对象被使用，且节点的半满slab缓冲区数量超过了最小临界点，则该页面将需要被释放掉
	 * 跳转至slab_empty执行缓冲区释放操作 
	 */
	if (unlikely(!new.inuse && n->nr_partial >= s->min_partial))
		goto slab_empty;

	/*
	 * 如果本来缓冲区链表都被使用，释放后处于半满状态
	 * 其将从full链表中remove_full()移出，并add_partial()添加至半满partial队列中
	 */
	if (!kmem_cache_has_cpu_partial(s) && unlikely(!prior)) {
		if (kmem_cache_debug(s))
			remove_full(s, n, page);
		add_partial(n, page, DEACTIVATE_TO_TAIL);
		stat(s, FREE_ADD_PARTIAL);
	}
	/* 释放中断锁并恢复中断环境 */
	spin_unlock_irqrestore(&n->list_lock, flags);
	return;

slab_empty:
	if (prior) {
		/* 如果缓冲区非空，则从partial链表中删除 */
		remove_partial(n, page);
		stat(s, FREE_REMOVE_PARTIAL);
	} else {
		/* Slab must be on the full list */
		remove_full(s, n, page);
	}

	spin_unlock_irqrestore(&n->list_lock, flags);
	stat(s, FREE_SLAB);
	discard_slab(s, page);
}
static noinline struct kmem_cache_node *free_debug_processing(
	struct kmem_cache *s, struct page *page, void *object,
	unsigned long addr, unsigned long *flags)
{
	struct kmem_cache_node *n = get_node(s, page_to_nid(page));

	spin_lock_irqsave(&n->list_lock, *flags);
	slab_lock(page);
	/* 检查slab的kmem_cache与page中的slab信息是否匹配，如果不匹配，可能发生了破坏或者数据不符 */
	if (!check_slab(s, page))
		goto fail;
	/* 检查对象地址的合法性，表示地址确切地为某对象的首地址，而非对象的中间位置 */
	if (!check_valid_pointer(s, page, object)) {
		slab_err(s, page, "Invalid object pointer 0x%p", object);
		goto fail;
	}
	/* on_freelist()检测该对象是否已经被释放，避免造成重复释放置 */
	if (on_freelist(s, page, object)) {
		object_err(s, page, object, "Object already free");
		goto fail;
	}
	/* check_object()主要是根据内存标识SLAB_RED_ZONE及SLAB_POISON的设置，对对象空间进行完整性检测 */
	if (!check_object(s, page, object, SLUB_RED_ACTIVE))
		goto out;
	/* 确保用户传入的kmem_cache与页面所属的kmem_cache类型是匹配的，否则将记录错误日志 */
	if (unlikely(s != page->slab_cache)) {
		if (!PageSlab(page)) {
			slab_err(s, page, "Attempt to free object(0x%p) "
				"outside of slab", object);
		} else if (!page->slab_cache) {
			pr_err("SLUB <none>: no slab for object 0x%p.\n",
			       object);
			dump_stack();
		} else
			object_err(s, page, object,
					"page slab pointer corrupt.");
		goto fail;
	}
	/* 如果设置了SLAB_STORE_USER标识，将记录对象释放的track信息 */
	if (s->flags & SLAB_STORE_USER)
		set_track(s, object, TRACK_FREE, addr);
	/* trace()记录对象的轨迹信息，同时还init_object()将重新初始化对象 */
	trace(s, page, object, 0);
	init_object(s, object, SLUB_RED_INACTIVE);
out:
	slab_unlock(page);
	/*
	 * Keep node_lock to preserve integrity
	 * until the object is actually freed
	 */
	return n;

fail:
	slab_unlock(page);
	spin_unlock_irqrestore(&n->list_lock, *flags);
	slab_fix(s, "Object at 0x%p not freed", object);
	return NULL;
}
```

### discard_slab

```c
static void discard_slab(struct kmem_cache *s, struct page *page)
{
	/* 更新统计信息 */
	dec_slabs_node(s, page_to_nid(page), page->objects);
	/* 释放缓冲区 */
	free_slab(s, page);
}
```

### __free_slab

```c
static void __free_slab(struct kmem_cache *s, struct page *page)
{
	/* 获取页面阶数转而获得释放的页面数 */
	int order = compound_order(page);
	int pages = 1 << order;
	/* 对该slab缓冲区进行一次检测，主要是检测是否有内存破坏以记录相关信息 */
	if (kmem_cache_debug(s)) {
		void *p;

		slab_pad_check(s, page);
		for_each_object(p, s, page_address(page),
						page->objects)
			check_object(s, page, p, SLUB_RED_INACTIVE);
	}
	/* 释放影子内存 */
	kmemcheck_free_shadow(page, compound_order(page));
	/* 修改内存页面的状态 */
	mod_zone_page_state(page_zone(page),
		(s->flags & SLAB_RECLAIM_ACCOUNT) ?
		NR_SLAB_RECLAIMABLE : NR_SLAB_UNRECLAIMABLE,
		-pages);
	/* 清除页面的slab信息 */
	__ClearPageSlabPfmemalloc(page);
	__ClearPageSlab(page);

	page_mapcount_reset(page);
	if (current->reclaim_state)
		current->reclaim_state->reclaimed_slab += pages;
	/* 将内存释放回伙伴系统，在伙伴系统内存释放中再分析 */
	__free_pages(page, order);
	/* 释放memcg中的页面处理 */
	memcg_uncharge_slab(s, order);
}
```

## 参考资料

[slub释放](https://www.jeanleo.com/2018/09/07/【linux内存源码分析】slub分配算法（5）/)
