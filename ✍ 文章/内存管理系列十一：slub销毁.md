## 前沿

### 往篇回顾

> 在上一篇中，主要看了slub释放的代码流程,入口函数是`kmem_cache_free`,主要内容如下：

待补充！

## 代码分析

### kmem_cache_destroy

```c
void kmem_cache_destroy(struct kmem_cache *s)
{
	struct kmem_cache *c, *c2;
	LIST_HEAD(release);
	bool need_rcu_barrier = false;
	bool busy = false;

	BUG_ON(!is_root_cache(s));

	get_online_cpus();
	get_online_mems();

	mutex_lock(&slab_mutex);

	s->refcount--;
	if (s->refcount)
		goto out_unlock;/* 如果引用不为0,则不释放该slub */
	/* 引用==0,释放slub,关键函数为do_kmem_cache_shutdown */
	for_each_memcg_cache_safe(c, c2, s) {
		if (do_kmem_cache_shutdown(c, &release, &need_rcu_barrier))
			busy = true;
	}

	if (!busy)
		do_kmem_cache_shutdown(s, &release, &need_rcu_barrier);

out_unlock:
	mutex_unlock(&slab_mutex);

	put_online_mems();
	put_online_cpus();

	do_kmem_cache_release(&release, need_rcu_barrier);
}
```

### do_kmem_cache_shutdown

```c
static int do_kmem_cache_shutdown(struct kmem_cache *s,
		struct list_head *release, bool *need_rcu_barrier)
{
	/* 销毁slub核心函数：__kmem_cache_shutdown->kmem_cache_close */
	if (__kmem_cache_shutdown(s) != 0) {
		printk(KERN_ERR "kmem_cache_destroy %s: "
		       "Slab cache still has objects\n", s->name);
		dump_stack();
		return -EBUSY;
	}

	if (s->flags & SLAB_DESTROY_BY_RCU)
		*need_rcu_barrier = true;

#ifdef CONFIG_MEMCG_KMEM
	if (!is_root_cache(s))
		list_del(&s->memcg_params.list);
#endif
	/* 成功后，将slub从slab_cache中删除 */
	list_move(&s->list, release);
	return 0;
}
```

### kmem_cache_close

```c
/*
 * Release all resources used by a slab cache.
 */
static inline int kmem_cache_close(struct kmem_cache *s)
{
	int node;
	struct kmem_cache_node *n;
	/* 释放本地CPU的缓存区，即kmem_cache_cpu管理的缓存区空间 */
	flush_all(s);
	/* 遍历各节点，get_node()获取节点下的kmem_cache_node管理结构 */
	for_each_kmem_cache_node(s, node, n) {
	/* 释放各个node中的半满队列 */
		free_partial(s, n);
		/* 如果nr_partial非0，则说明有异常；销毁失败 */
		if (n->nr_partial || slabs_node(s, node))
			return 1;
	}
	/* 将kmem_cache的每CPU缓存管理kmem_cache_cpu通过free_percpu()归还给系统 */
	free_percpu(s->cpu_slab);
	/* 将每node管理结构体kmem_cache_node归还系统 */
	free_kmem_cache_nodes(s);
	return 0;
}
```

### flush_all

```c
/* 
 * on_each_cpu_cond任务是遍历所有cpu，执行作为入参传入的函数has_cpu_slab 
 * 判断各个cpu上的资源是否存在，存在则通过flush_cpu_slab对该cpu上的资源进行释放
 */
static void flush_all(struct kmem_cache *s)
{
	on_each_cpu_cond(has_cpu_slab, flush_cpu_slab, s, 1, GFP_ATOMIC);
}
```

### on_each_cpu_cond

```c
/* 
 * cond_func： 钩子函数，用于根据调用者传入的CPU信息参数来判断是否需要打断该CPU以执行入参func的操作
 * 		has_cpu_slab:判断cpu是否有缓冲区，有则返回true；即需要CPU被打断去执行本地缓存释放动作
 * smp_call_func_t func: 任务函数
 * info: 入参
 * gfp_flags：申请cpumask空间的标识
 */
void on_each_cpu_cond(bool (*cond_func)(int cpu, void *info),
			smp_call_func_t func, void *info, bool wait,
			gfp_t gfp_flags)
{
	cpumask_var_t cpus;
	int cpu, ret;

	might_sleep_if(gfp_flags & __GFP_WAIT);
	/* 申请cpumask空间 */
	if (likely(zalloc_cpumask_var(&cpus, (gfp_flags|__GFP_NOWARN)))) {
		/* 禁止抢占内核 */
		preempt_disable();
		/* 遍历各个CPU，根据cond_func()判断是否需要对该CPU进行打断处理，如果需要则cpumask_set_cpu()对该CPU进行标志 */
		for_each_online_cpu(cpu)
			if (cond_func(cpu, info))
				cpumask_set_cpu(cpu, cpus);
		/* 打断各个标志位对应的CPU去执行func()的操作 */
		on_each_cpu_mask(cpus, func, info, wait);
		/* 完了将会恢复抢占，释放cpumask空间 */
		preempt_enable();
		free_cpumask_var(cpus);
	} else {
		/*
		 * zalloc_cpumask_var()申请不到空间，将会逐个处理器进行打断再进行处理，其最终功能和作用与申请到空间的情况都是一致的
		 */
		preempt_disable();
		for_each_online_cpu(cpu)
			if (cond_func(cpu, info)) {
				ret = smp_call_function_single(cpu, func,
								info, wait);
				WARN_ON_ONCE(ret);
			}
		preempt_enable();
	}
}
```

### __flush_cpu_slab

flush_cpu_slab()->__flush_cpu_slab()

```c
/* 将本地CPU的缓存区进行释放 */
static inline void __flush_cpu_slab(struct kmem_cache *s, int cpu)
{
	/* 首先获取本地CPU的kmem_cache_cpu管理结构 */
	struct kmem_cache_cpu *c = per_cpu_ptr(s->cpu_slab, cpu);
	
	if (likely(c)) {
	/* 
	 * 如果本地CPU存在缓存区占用, 则通过flush_slab()释放本地缓存区 
	 * 其主要通过deactivate_slab()去激活本地缓存区；即释放
	 */
		if (c->page)
			flush_slab(s, c);
	/* 本地CPU半满缓存列表进行释放 */
		unfreeze_partials(s, c);
	}
}
```

### deactivate_slab

```c
/*
 * Remove the cpu slab
 */
static void deactivate_slab(struct kmem_cache *s, struct page *page,
				void *freelist)
{
	enum slab_modes { M_NONE, M_PARTIAL, M_FULL, M_FREE };
	struct kmem_cache_node *n = get_node(s, page_to_nid(page));
	int lock = 0;
	enum slab_modes l = M_NONE, m = M_NONE;
	void *nextfree;
	int tail = DEACTIVATE_TO_HEAD;
	struct page new;
	struct page old;
	/* 如果不为空，则该CPU的slub对象被其它CPU释放了, 将会更新统计同时设置tail标识为DEACTIVATE_TO_TAIL */
	if (page->freelist) {
		stat(s, DEACTIVATE_REMOTE_FREES);
		tail = DEACTIVATE_TO_TAIL;
	}
	/* 如果为空，意味着该缓存区的对象已经全部分配到了CPU的kmem_cache_cpu中freelist链表中 */
	/*
	 * 1. 通过while循环遍历CPU上的freelist链表get_greepointer()获取空闲对象
	 * 2. 通过内部的do-while循环，借用__cmpxchg_double_slab()比较交换将对象以插入缓存区页面的freelist空闲链表头的方式归还回去
	 * 3. 目标：当页面还处于冻结状态，将会释放每CPU的所有可用对象回到缓冲区的空闲列表中
	 */
	while (freelist && (nextfree = get_freepointer(s, freelist))) {
		void *prior;
		unsigned long counters;

		do {
			prior = page->freelist;
			counters = page->counters;
			set_freepointer(s, freelist, prior);
			new.counters = counters;
			new.inuse--;
			VM_BUG_ON(!new.frozen);

		} while (!__cmpxchg_double_slab(s, page,
			prior, counters,
			freelist, new.counters,
			"drain percpu freelist"));

		freelist = nextfree;
	}

	/*
	 * Stage two: Ensure that the page is unfrozen while the
	 * list presence reflects the actual number of objects
	 * during unfreeze.
	 *
	 * We setup the list membership and then perform a cmpxchg
	 * with the count. If there is a mismatch then the page
	 * is not unfrozen but the page is on the wrong list.
	 *
	 * Then we restart the process which may have to remove
	 * the page from the list that we just put it on again
	 * because the number of objects in the slab may have
	 * changed.
	 */
redo:
	/* 将缓冲区的freeelist以及counters信息存到临时old结构中以备后用 */
	old.freelist = page->freelist;
	old.counters = page->counters;
	VM_BUG_ON(!old.frozen);

	/* Determine target state of the slab */
	new.counters = old.counters;
	if (freelist) {
	/* 把前面步骤一未被归还的那个对象归还到缓冲区中，同时更新new信息，此时new.freelist持有该缓存区的所有空闲对象 */
		new.inuse--;
		set_freepointer(s, freelist, old.freelist);
		new.freelist = freelist;
	} else
		new.freelist = old.freelist;
	/* 将临时缓冲区状态设置为非冻结 */
	new.frozen = 0;
	
	if (!new.inuse && n->nr_partial >= s->min_partial)
	/* 该slab缓存区中无对象被使用(!new.inuse)，且部分满slab个数大于最小值，意味着该缓存区需要被销毁 */
		m = M_FREE;
	else if (new.freelist) {
	/* freelist不为空，仅使用了部分对象，则标识m为M_PARTIAL */
		m = M_PARTIAL;
		if (!lock) {
			lock = 1;
			/*
			 * Taking the spinlock removes the possiblity
			 * that acquire_slab() will see a slab page that
			 * is frozen
			 */
			spin_lock(&n->list_lock);
		}
	} else {
	/* freelist为空，仅使用了部分对象，则标识m为M_PARTIAL */
		m = M_FULL;
		if (kmem_cache_debug(s) && !lock) {
			lock = 1;
			/*
			 * This also ensures that the scanning of full
			 * slabs from diagnostic functions will not see
			 * any frozen slabs.
			 */
			spin_lock(&n->list_lock);
		}
	}
	/* 
	 * 判断上一次的缓存区状态l与接下来的操作状态m是否一致，不一致则意味着需要发生变更
	 * 其将会先判断l的状态为M_PARTIAL或M_FULL，继而采取对应的remove_partial()或remove_full()链表摘除操作 
	 */
	if (l != m) {

		if (l == M_PARTIAL)

			remove_partial(n, page);

		else if (l == M_FULL)

			remove_full(s, n, page);

		if (m == M_PARTIAL) {

			add_partial(n, page, tail);
			stat(s, tail);

		} else if (m == M_FULL) {

			stat(s, DEACTIVATE_FULL);
			add_full(s, n, page);

		}
	}

	l = m;
	/* 
	 * 判断自redo到此，缓存区是否发生过对象操作变更，如果没发生过的话，将会把new暂存的空闲对象挂载到缓存区中以及更新counters
	 * 否则将会跳转回redo标签重新执行前面的操作
	 */
	if (!__cmpxchg_double_slab(s, page,
				old.freelist, old.counters,
				new.freelist, new.counters,
				"unfreezing slab"))
		goto redo;

	if (lock)
		spin_unlock(&n->list_lock);

	if (m == M_FREE) {
		stat(s, DEACTIVATE_EMPTY);
		discard_slab(s, page);
		stat(s, FREE_SLAB);
	}
}
```

## 参考资料

[slub释放](https://www.jeanleo.com/2018/09/08/【linux内存源码分析】slub分配算法（6）/)
