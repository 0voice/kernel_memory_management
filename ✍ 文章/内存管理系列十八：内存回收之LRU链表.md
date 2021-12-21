## 前沿

### 往篇回顾

> 在上一篇中，主要分析了内存池的实现、运行机制；内存池实际是一个最后的内存资源储存池，本身还是依赖于现有的伙伴系统或者slub系统

- 主要涉及`mempool_create`, `mempool_destroy`, `mempool_alloc`, `mempool_free`这几个函数；
- 重点在内存申请的时候，首先从自定义的mempool->alloc尝试分配，失败后再从内存池中获取；
- 如果内存池也分配失败，则会自行加入mempool->wait队列中，并将本进程自动睡眠等待有空闲内存或者时间截止被唤醒；

### 内存回收简述

- 当linux系统内存压力就大时，就会对系统的每个压力大的zone(最新的linux动向逐渐转向按node回收)进程内存回收，内存回收主要是针对匿名页和文件页进行的
- 内存回收主要有两个方面: 释放页；将页回写到swap区，再释放内存页
  1. 进程堆栈，数据段使用的匿名页：存放swap区
  2. 进程代码段映射的可执行文件的文件页：直接释放
  3. 打开文件进行读写使用的文件页：如果页中数据与文件数据不一致，则回写到磁盘对应的文件页，如果一致则释放
  4. 进行文件映射mmap共享内存时使用的页：如果页中数据与文件数据不一致，则进行回写到磁盘对应文件中，如果一致，则直接释放
  5. 进行匿名mmap共享内存时使用的页：存放到swap分区中
  6. 进行shmem共享内存时使用的页：存放到swap分区中

### lru链表分类

- LRU_INACTIVE_ANON：称为非活动匿名页lru链表，此链表中保存的是此zone中所有最近没被访问过的并且可以存放到swap分区的页描述符，在此链表中的页描述符的PG_active标志为0
- LRU_ACTIVE_ANON：称为活动匿名页lru链表，此链表中保存的是此zone中所有最近被访问过的并且可以存放到swap分区的页描述符，此链表中的页描述符的PG_active标志为1
- LRU_INACTIVE_FILE：称为非活动文件页lru链表，此链表中保存的是此zone中所有最近没被访问过的文件页的页描述符，此链表中的页描述符的PG_active标志为0
- LRU_ACTIVE_FILE：称为活动文件页lru链表，此链表中保存的是此zone中所有最近被访问过的文件页的页描述符，此链表中的页描述符的PG_active标志为1
- LRU_UNEVICTABLE：此链表中保存的是此zone中所有禁止换出的页的描述符, 一般是被mlock锁定

### lru缓存

- 在多核环境下，在同时需要对lru链表进行修改时，锁的竞争会非常频繁，因此内核提供了一个[lru缓存](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/30/ARM64内存管理十八-内存回收之LRU链表/#pagevec)机制，用以减少锁的竞争频率
- [lru缓存](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/30/ARM64内存管理十八-内存回收之LRU链表/#pagevec)是按照要对lru操作的类型纬度定义的，对lru链表的操作主要有以下几种：
  1. 将不处于lru链表的新页放入到lru链表中, 对应`static DEFINE_PER_CPU(struct pagevec, lru_add_pvec)`
  2. 将非活动lru链表中的页移动到非活动lru链表尾部(活动页不需要这样做，后面说明), 对应`static DEFINE_PER_CPU(struct pagevec, lru_rotate_pvecs)`
  3. 将处于活动lru链表尾部的页移动到非活动lru链表, 对应`static DEFINE_PER_CPU(struct pagevec, lru_deactivate_pvecs)`
  4. 将处于非活动lru链表的页移动到活动lru链表头, 对应`static DEFINE_PER_CPU(struct pagevec, activate_page_pvecs)`
  5. 将页从lru链表中移除(不需要依赖lru)
- 由于4中lru缓冲链表都是cpu级别的；而目前lru的5种链表都是zone级别的，因此在一个页加入lru缓存前，必须设置好页的属性，才能配合lru缓存进行工作

### 本篇主要内容

> 对于整个内存回收来说，lru链表是关键中的关键，实际上整个内存回收，做的事情就是处理LRU(Least Recently Used)链表的收缩，所以这篇文章就先说说系统的lru链表

## LRU基本操作代码分析

### 新页加入lru链表

#### __lru_cache_add

> 当需要将一个新页加入到lru链表中，必须先加入到当前CPU的`lru_add_pvec`缓存中

```c
/* 加入到lru_add_pvec缓存中 */
static void __lru_cache_add(struct page *page)
{
	/* 获取此CPU的lru缓存 */
	struct pagevec *pvec = &get_cpu_var(lru_add_pvec);
	/* page->_count++ 
     * 在页从lru缓存移动到lru链表时，这些页的page->_count会--
     */
	page_cache_get(page);
	/* 检查LRU缓存是否已满 */
	if (!pagevec_space(pvec))
	/* 满则将此lru缓存中的页放到lru链表中，核心转移函数：__pagevec_lru_add_fn */
		__pagevec_lru_add(pvec);
	/* 将page加入到此cpu的lru缓存中,注意
	 * 加入pagevec实际上只是将pagevec中的pages数组中的某个指针指向此页
	 * 如果此页原本属于lru链表，那么现在实际还是在原来的lru链表中 
	 */
	pagevec_add(pvec, page);
	put_cpu_var(lru_add_pvec);
}
```

#### __lru_cache_add

```c
/* 将lru_add缓存中的页加入到lru链表中 */
static void __pagevec_lru_add_fn(struct page *page, struct lruvec *lruvec,
				 void *arg)
{
	/* 判断此页是否是page cache页(映射文件的页) */
	int file = page_is_file_cache(page);
	/* 是否是活跃的页, 判断page的PG_active标志
     * 1. 置位，则将此页加入到活动lru链表中
     * 2. 没置位，则加入到非活动lru链表中
     */
	int active = PageActive(page);
	/* 获取page所在的lru链表，里面会检测是映射页还是文件页，并且检查PG_active，最后能得出该page应该放到哪个lru链表中 
	 * 1. PG_unevictable置位，则加入到LRU_UNEVICTABLE链表中
	 * 2. 如果PG_swapbacked置位，则加入到匿名页lru链表，否则加入到文件页lru链表
     * 3. PG_active置位，则加入到活动lru链表，否则加入到非活动lru链表
     */
	enum lru_list lru = page_lru(page);

	VM_BUG_ON_PAGE(PageLRU(page), page);

	SetPageLRU(page);
	/* 
	* 将page加入到对应lru链表头部中：
	* 1. 获取页的数量，如果支持透明大叶，会是多个页
	* 2. 通过mem_cgroup_update_lru_size更新lruvec中lru类型的链表的page num
	* 3. 加入对应lru链表头部，更新统计
	*/
	add_page_to_lru_list(page, lruvec, lru);
	/* 更新lruvec中的reclaim_stat */
	update_page_reclaim_stat(lruvec, file, active);
	trace_mm_lru_insertion(page, lru);
}
```

### 将处于非活动链表中的页移动到非活动链表尾部

> 当一个脏页需要回收时，系统首先会将页异步回写到swap或磁盘对应文件中，再通过`rotate_reclaimable_page`将页移动到非活动lru链表尾部

#### rotate_reclaimable_page

```c
void rotate_reclaimable_page(struct page *page)
{
	 /* 此页加入到非活动lru链表尾部的条件 */
	if (!PageLocked(page) && !PageDirty(page) && !PageActive(page) &&
	    !PageUnevictable(page) && PageLRU(page)) {
		struct pagevec *pvec;
		unsigned long flags;
        /* page->_count++，因为这里会加入到lru_rotate_pvecs这个lru缓存中 
         * lru缓存中的页移动到lru时，会对移动的页page->_count--
         */
		page_cache_get(page);
		local_irq_save(flags);/* 禁止中断 */
		/* 获取当前CPU的lru_rotate_pvecs缓存 */
		pvec = this_cpu_ptr(&lru_rotate_pvecs);
		if (!pagevec_add(pvec, page))
		/* lru_rotate_pvecs缓存已满，将当前缓存中的页加入到非活动lru链表尾部
		 * 转移核心函数：pagevec_move_tail_fn 
		 */
			pagevec_move_tail(pvec);
		local_irq_restore(flags);/* 重新开启中断 */
	}
}
```

#### pagevec_move_tail_fn

```c
/* 
 * 将lru缓存pvec中的页移动到非活动lru链表尾部操作的回调函数
 * 这些页原本就属于非活动lru链表
 */
static void pagevec_move_tail_fn(struct page *page, struct lruvec *lruvec,
				 void *arg)
{
	int *pgmoved = arg;
	
	if (PageLRU(page) && !PageActive(page) && !PageUnevictable(page)) {
	/* 获取页应该放入匿名页lru链表还是文件页lru链表，通过页的PG_swapbacked标志判断 */
		enum lru_list lru = page_lru_base_type(page);
		/* 加入到对应的非活动lru链表尾部 */
		list_move_tail(&page->lru, &lruvec->lists[lru]);
		(*pgmoved)++;
	}
}
```

### 将活动lru链表中的页加入到非活动lru链表中

> 文件系统主动将一些没有被进程映射的页进行释放时，会讲一些活动lru链表的页移动到非活动lru链表中；内存回收过程中并不会使用这种方式；

#### deactivate_file_page

```c
void deactivate_file_page(struct page *page)
{
	/* 如果页被锁在内存中禁止换出，则跳出 */
	if (PageUnevictable(page))
		return;
        /*page->count==1 说明此页没有进程映射*/
	if (likely(get_page_unless_zero(page))) {
    /* 获取本cpu的deactivate缓存链表 */
		struct pagevec *pvec = &get_cpu_var(lru_deactivate_file_pvecs);
/* 将page加入deactivate链表后如果链表满了，则触发lru_deactivate_file_fn函数，将lru缓存中页放到lru链表中 */
		if (!pagevec_add(pvec, page))
			pagevec_lru_move_fn(pvec, lru_deactivate_file_fn, NULL);
		put_cpu_var(lru_deactivate_file_pvecs);
	}
}
```

#### lru_deactivate_file_fn

```c
static void lru_deactivate_file_fn(struct page *page, struct lruvec *lruvec,
			      void *arg)
{
	int lru, file;
	bool active;
/*此页不在lru中，不处理此页*/
	if (!PageLRU(page))
		return;
/*此页被锁定，则不处理此页*/
	if (PageUnevictable(page))
		return;

	/* 有进程映射了此页，不处理 */
	if (page_mapped(page))
		return;
/*获取页的活动标志 */
	active = PageActive(page);
    /* 根据页的PG_swapbacked判断此页是否需要依赖swap分区 */
	file = page_is_file_cache(page);
    /* 获取此页需要加入匿名页或者文件页lru链表，也是通过PG_swapbacked标志判断 */
	lru = page_lru_base_type(page);
    /* 从活动lru链表中删除 */
	del_page_from_lru_list(page, lruvec, lru + active);
    /* 清除PG_active和PG_referenced */
	ClearPageActive(page);
	ClearPageReferenced(page);
    /* 加到非活动页lru链表头部 */
	add_page_to_lru_list(page, lruvec, lru);
    /* 如果此页当前正在回写或者是脏页 */
	if (PageWriteback(page) || PageDirty(page)) {
		/*
		 * PG_reclaim could be raced with end_page_writeback
		 * It can make readahead confusing.  But race window
		 * is _really_ small and  it's non-critical problem.
		 */
        /* 则设置此页需要回收 */
		SetPageReclaim(page);
	} else {
       /* 如果此页是干净的，并且非活动的，则将此页移动到非活动lru链表尾部
         * 因为此页回收起来更简单，不用回写
         */
		list_move_tail(&page->lru, &lruvec->lists[lru]);
		__count_vm_event(PGROTATED);
	}

	if (active)
		__count_vm_event(PGDEACTIVATE);
	update_page_reclaim_stat(lruvec, file, 0);
}
```

### 将非活动lru链表页加入到活动lru链表

使用场景：

- 非活动页被标记为活动页后，要加入到活动lru链表中

#### activate_page

```c
void activate_page(struct page *page)
{
/* 该页要在lru链表中，非活动页，没有被锁定 */
	if (PageLRU(page) && !PageActive(page) && !PageUnevictable(page)) {
		struct pagevec *pvec = &get_cpu_var(activate_page_pvecs);

		page_cache_get(page);
		if (!pagevec_add(pvec, page))
			pagevec_lru_move_fn(pvec, __activate_page, NULL);
		put_cpu_var(activate_page_pvecs);
	}
}
```

#### __activate_page

```c
static void __activate_page(struct page *page, struct lruvec *lruvec,
			    void *arg)
{
	if (PageLRU(page) && !PageActive(page) && !PageUnevictable(page)) {
        /*是否为文件页*/
		int file = page_is_file_cache(page);
        /*获取lru类型*/
		int lru = page_lru_base_type(page);
        /*将此页从lru链表中移除*/
		del_page_from_lru_list(page, lruvec, lru);
        /*设置page的PG_active标志，说明此页已经在lru链表中*/
		SetPageActive(page);
        /*获取lru最终所属链表*/
		lru += LRU_ACTIVE;
        /*将此页加入到活动页lru链表头*/
		add_page_to_lru_list(page, lruvec, lru);
		trace_mm_lru_activate(page);

		__count_vm_event(PGACTIVATE);
        /* 更新lruvec中zone_reclaim_stat->recent_scanned[file]++和zone_reclaim_stat->recent_rotated[file]++ */
		update_page_reclaim_stat(lruvec, file, 1);
	}
}
```

## 附录

#### pagevec

```c
struct pagevec {
    /* 当前数量 */
    unsigned long nr;
    unsigned long cold;
    /* 指针数组，每一项都可以指向一个页描述符，默认大小是14 */
    struct page *pages[PAGEVEC_SIZE];
};
```

#### lruvec

\``` c++ /* lru链表描述符，主要有5个双向链表 */ struct lruvec { /* 5个lru双向链表头 */ struct list_head lists[NR_LRU_LISTS]; struct zone_reclaim_stat reclaim_stat; #ifdef CONFIG_MEMCG /* 所属zone */ struct zone *zone; #endif };

## 参考资料

[tolimit-lru链表](https://www.cnblogs.com/tolimit/p/5447448.html)
