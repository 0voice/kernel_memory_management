## 前沿

### 往篇回顾

> 在上两篇中，主要看了slub释放(kmem_cache_free)，销毁(kmem_cache_destroy)的代码流程,入口函数是`kmem_cache_free`,主要内容如下：

待补充！未看的很明白，先往下看别的，回头再来补充！

### vmalloc基本知识

- vmalloc用于分配虚拟地址连续(物理地址不连续)的内存空间，vzmalloc相对于vmalloc多了个0初始化;
- vmalloc/vzmalloc分配的虚拟地址范围在VMALLOC_START/VMALLOC_END之间，属于堆内存;
- linux管理vmalloc分别有两个数据结构: [vm_struct](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/05/ARM64内存管理十二-vmalloc内存机制/#vm_struct), [vm_area_struct](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/05/ARM64内存管理十二-vmalloc内存机制/#vm_area_struct)；前者是内核虚拟地址空间的映射，后者是应用进程虚拟地址空间映射;
- 内核vmalloc区具体地址空间的管理是通过[vmap_area](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/05/ARM64内存管理十二-vmalloc内存机制/#vmap_area)管理的，该结构体记录整个区间的起始和结束;
- vmalloc是内核出于自身的目的使用高端内存页的少数情形之一
- vmalloc在申请内存时逐页分配，确保在物理内存有严重碎片的情况下，vmalloc仍然可以工作

### 本篇主要内容

> 分析vmalloc模块初始化(vmalloc_init())及通过vmalloc申请内存(vmalloc())的主要流程

## vmalloc模块初始化

### vmalloc_init

> vmalloc初始化： start_kernel()->mm_init()->vmalloc_init()

```c
void __init vmalloc_init(void)
{
	struct vmap_area *va;
	struct vm_struct *tmp;
	int i;
	/* 
	 * 遍历每CPU的vmap_block_queue和vfree_deferred变量并进行初始化;
	 * vmap_block_queue是非连续内存块队列管理结构，主要是队列以及对应的保护锁
	 * vfree_deferred是vmalloc的内存延迟释放管理
     */
	for_each_possible_cpu(i) {
	
		struct vmap_block_queue *vbq;
		struct vfree_deferred *p;

		vbq = &per_cpu(vmap_block_queue, i);
		spin_lock_init(&vbq->lock);
		INIT_LIST_HEAD(&vbq->free);
		p = &per_cpu(vfree_deferred, i);
		init_llist_head(&p->list);
		INIT_WORK(&p->wq, free_work);
	}

	/* 将挂接在vmlist链表的各项__insert_vmap_area()输入到非连续内存块的管理中 */
	for (tmp = vmlist; tmp; tmp = tmp->next) {
		va = kzalloc(sizeof(struct vmap_area), GFP_NOWAIT);
		va->flags = VM_VM_AREA;
		va->va_start = (unsigned long)tmp->addr;
		va->va_end = va->va_start + tmp->size;
		va->vm = tmp;
		
		__insert_vmap_area(va);
	}

	vmap_area_pcpu_hole = VMALLOC_END;

	vmap_initialized = true;
}
```

### __insert_vmap_area

```c
/*
 * 将struct vmap_area 塞入全局红黑树中；并塞入全局双向链表vmap_area_list中
 * 1. while循环是对vmap_area_root红黑树进行查找，找到合适的位置后通过rb_link_node()插入红黑树；
 * 2. 在通过rb_insert_color()对红黑树进行平衡调整；
 * 3. 根据插入的红黑树的父子节点关系插入到vmap_area_list链表中；
 */
static void __insert_vmap_area(struct vmap_area *va)
{
	/* 获取红黑树根节点指针 */
	struct rb_node **p = &vmap_area_root.rb_node;
	struct rb_node *parent = NULL;
	struct rb_node *tmp;
	/* 遍历红黑树，找到va合适的插入点 */
	while (*p) {
		struct vmap_area *tmp_va;
		
		parent = *p;
		/* 找到parent节点所属的struct vmap_area结构体指针 */
		tmp_va = rb_entry(parent, struct vmap_area, rb_node);
		/* 找插入点，总的原则就是红黑树的左侧地址小 */
		if (va->va_start < tmp_va->va_end)
			p = &(*p)->rb_left;
		else if (va->va_end > tmp_va->va_start)
			p = &(*p)->rb_right;
		else
			BUG();
	}
	/* 找到合适的插入点后，把节点塞入树中；p为最终位置，parent为p的父节点 */
	rb_link_node(&va->rb_node, parent, p);
	/* 调整颜色，翻转后使其变成标准红黑树 */
	rb_insert_color(&va->rb_node, &vmap_area_root);

	/* 查找插入的内存块管理树的父节点，有则插入到该节点链表的后面位置，否则作为链表头插入到vmap_area_list链表中 */
	tmp = rb_prev(&va->rb_node);
	if (tmp) {
		struct vmap_area *prev;
		prev = rb_entry(tmp, struct vmap_area, rb_node);
		list_add_rcu(&va->list, &prev->list);
	} else
	/* 作为链表头插入到vmap_area_list链表中 */
		list_add_rcu(&va->list, &vmap_area_list);
}
```

## vmalloc内存申请

### __vmalloc_node_range

```c
vmalloc()->__vmalloc_node_flags()->__vmalloc_node()->__vmalloc_node_range()

/* vmalloc具体申请内存过程 
 * 1. 通过__get_vm_area_node()向slub系统申请结构体vm_struct空间
 * 2. 在__vmalloc_area_node()通过查询红黑树、链表找到vmalloc合适的虚拟内存区；再向伙伴系统申请物理空间；再进行映射
 */
void *__vmalloc_node_range(unsigned long size, unsigned long align,
			unsigned long start, unsigned long end, gfp_t gfp_mask,
			pgprot_t prot, unsigned long vm_flags, int node,
			const void *caller)
{
	struct vm_struct *area;
	void *addr;
	unsigned long real_size = size;
	/* 首先对申请内存的大小做对齐后，如果大小为0或者大于总内存，则返回失败 */
	size = PAGE_ALIGN(size);
	if (!size || (size >> PAGE_SHIFT) > totalram_pages)
		goto fail;
	/* 使用kmalloc，在slub中分配vm_struct数据结构体，并返回 */
	area = __get_vm_area_node(size, align, VM_ALLOC | VM_UNINITIALIZED |
				vm_flags, start, end, node, gfp_mask, caller);
	if (!area)
		goto fail;
	/* 根据vm_struct的信息进行物理空间申请，建立页表映射 */
	addr = __vmalloc_area_node(area, gfp_mask, prot, node);
	if (!addr)
		return NULL;

	/*
	 * 标示内存空间已经初始化
	 */
	clear_vm_uninitialized_flag(area);

	/*
	 * 调用kmemleak_alloc()进行内存分配泄漏调测
	 */
	kmemleak_alloc(addr, real_size, 2, gfp_mask);

	return addr;

fail:
	warn_alloc_failed(gfp_mask, 0,
			  "vmalloc: allocation failure: %lu bytes\n",
			  real_size);
	return NULL;
}
```

### __get_vm_area_node

```c
/* 在vmalloc中寻找合适的虚拟内存
 * 1. 通过kmalloc申请管理内存 vm_struct以及 vmap_area
 * 2. 通过alloc_vmap_area从vmalloc区中寻找内存，并插入红黑树中；
 */
static struct vm_struct *__get_vm_area_node(unsigned long size,
		unsigned long align, unsigned long flags, unsigned long start,
		unsigned long end, int node, gfp_t gfp_mask, const void *caller)
{
	struct vmap_area *va;
	struct vm_struct *area;

	BUG_ON(in_interrupt());
	if (flags & VM_IOREMAP)
		align = 1ul << clamp_t(int, fls_long(size),
				       PAGE_SHIFT, IOREMAP_MAX_ORDER);

	size = PAGE_ALIGN(size);
	if (unlikely(!size))
		return NULL;
	/*
	 * 核心函数：kmalloc_node()->__kmalloc_node()申请管理结构体内存
	 * 1. 通过kmalloc_slab()寻找合适的slub申请内存
	 * 2. 通过slab_alloc_node()从slub中分配内存，具体过程可以参考[slub申请内存](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/08/26/ARM64%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%9D-slub%E7%94%B3%E8%AF%B7%E5%86%85%E5%AD%98/)
	 */
	area = kzalloc_node(sizeof(*area), gfp_mask & GFP_RECLAIM_MASK, node);
	if (unlikely(!area))
		return NULL;
	/* 如果没有标记，size要多算一个guard page用于分隔  */
	if (!(flags & VM_NO_GUARD))
		size += PAGE_SIZE;
	/* 申请指定的虚拟地址范围内的未映射空白；即申请不连续的物理内存 */
	va = alloc_vmap_area(size, align, start, end, node, gfp_mask);
	if (IS_ERR(va)) {
		kfree(area);
		return NULL;
	}
	/* 将vmap_area结构体中数据同步到vm_struct结构体中  */
	setup_vmalloc_vm(area, va, flags, caller);

	return area;
}
```

### alloc_vmap_area

```
/*
 * 从vmalloc区取出一段虚拟地址vmalloc : 0xf0800000 - 0xff800000   ( 240 MB)
 * 1. alloc_vmap_area会缓存上次一次搜索的节点到free_vmap_cache, 这样是认为一般虚拟内存是按顺序连续分配的
 * 2. 如果通过缓存没有搜索到就要从全局的vmap_area_root开始搜索: 先红黑树再链表
 * 		a. 在红黑树中根据指定区间寻找合适的区间，找到一个区间正好包含vstart
 * 		b. 在该区间的链表后面寻找合适的空洞进行分配(也有可能在红黑树上找到的就是空洞就不用接着遍历了)
 */
static struct vmap_area *alloc_vmap_area(unsigned long size,
				unsigned long align,
				unsigned long vstart, unsigned long vend,
				int node, gfp_t gfp_mask)
{
	struct vmap_area *va;
	struct rb_node *n;
	unsigned long addr;
	int purged = 0;
	struct vmap_area *first;

	BUG_ON(!size);// 长度为0不行
	BUG_ON(size & ~PAGE_MASK);// 长度不是页面整数倍不行
	BUG_ON(!is_power_of_2(align));// 长度不是2的n次方不行
	/* 先通过kmalloc申请管理结构体 struct vmap_area 内存，过程类似kzalloc_node(),少了一个内存初始化要求 */
	va = kmalloc_node(sizeof(struct vmap_area),
			gfp_mask & GFP_RECLAIM_MASK, node);
	if (unlikely(!va))
		return ERR_PTR(-ENOMEM);

	/*
	 * Only scan the relevant parts containing pointers to other objects
	 * to avoid false negatives.
	 */
	kmemleak_scan_area(&va->rb_node, SIZE_MAX, gfp_mask & GFP_RECLAIM_MASK);

retry:
	spin_lock(&vmap_area_lock);
	/* 
	 * free_vmap_cache记录着最近释放的或最近注册使用的不连续内存页面空间，是用以加快空间的搜索速度
	 * 1. 在红黑树中根据指定区间寻找合适的区间，找到一个区间正好包含vstart
	 * 2. 在该区间的链表后面寻找合适的空洞进行分配(也有可能在红黑书上找到的就是空洞就不用接着遍历了)
	 * 3. 记录本次分配情况
	 */
	if (!free_vmap_cache || /* 缓存为空, 并且在下面条件满足的时候不使用缓存 */
			size < cached_hole_size || /* 至少有一个空洞可以复用, 直接从头开始搜索 */
			vstart < cached_vstart || /* 别缓存的起点小 */
			align < cached_align) { /* 对齐大小比缓存的小 */
nocache:
		cached_hole_size = 0;
		free_vmap_cache = NULL;
	}
	/* record if we encounter less permissive parameters */
	cached_vstart = vstart;
	cached_align = align;

	/* 寻找符合条件的struct vmap_area区域, 如果上次搜索的节点非空，则从上次搜索的节点开始 */
	if (free_vmap_cache) {/* 从上次分配的区域开始向后查找 */
		/* 找到上一次分配节点的struct vmap_area结构体指针 */
		first = rb_entry(free_vmap_cache, struct vmap_area, rb_node);
		/* 上次分配的结束地址向上对齐，得到本次可用的地址 */
		addr = ALIGN(first->va_end, align);
		if (addr < vstart)
		/* 要找的是: va_begin< vstart <va_end; 缓存的末尾对齐之后超出了范围从头开始搜索 */
			goto nocache;
		if (addr + size < addr)
			goto overflow;/* 整数溢出 */

	} else {
		addr = ALIGN(vstart, align);
		if (addr + size < addr)
			goto overflow;/* 整数溢出 */
		/* 第一次搜索，缓存free_vmap_cache为空，从全局vmap_area_root开始搜索 */
		n = vmap_area_root.rb_node;
		first = NULL;
		/* 要找对齐之后的va_start<=vstart<=va_end的节点 */
		while (n) {
			struct vmap_area *tmp;
			tmp = rb_entry(n, struct vmap_area, rb_node);
			if (tmp->va_end >= addr) {
				first = tmp;
				if (tmp->va_start <= addr)
					break; /* 找到要找的区间 */
				n = n->rb_left;/* 否则，说明该节点还位于较高端，我们移动到树的左节点上，查找更小的区域 */
			} else/* 否则说明该节点代表的区域还比较小，我们需要移动到右节点上，查找更大的区域 */
				n = n->rb_right;
		}

		if (!first)
		/* 在first为空的情况下，表示vstart为起始的虚拟地址空间未被使用过，将会直接对该虚拟地址空间进行分配 */
			goto found;
	}

	/* 
	 * 若first不为空，意味着该空间曾经分配过，进入while分支进行处理
	 * 该循环是从first为起点遍历vmap_area_list链表管理的虚拟地址空间链表进行查找
	 * 如果找合适的未使用的虚拟地址空间或者遍历到了链表末尾，都表示找到了该空间 
	 */
	while (addr + size > first->va_start && addr + size <= vend) {
		if (addr + cached_hole_size < first->va_start)
			cached_hole_size = first->va_start - addr;
		addr = ALIGN(first->va_end, align);
		if (addr + size < addr)
			goto overflow;
		/* 如果已经是该区间中链表的最后一个节点，则返回该节点 */
		if (list_is_last(&first->list, &vmap_area_list))
			goto found;
		/* 检查链表的下一个节点是否满足 */
		first = list_entry(first->list.next,
				struct vmap_area, list);
	}

found:
	if (addr + size > vend)
	/* 可用地址加上分配长度超过了分配范围，说明地址空间不足了，退出 */
		goto overflow;
	/* 运行到这里，说明找到合适的地址区间，初始化KVA数据结构 */
	va->va_start = addr;
	va->va_end = addr + size;
	va->flags = 0;
	/* 插入到红黑书和链表当中 */
	__insert_vmap_area(va);
	/* 保存本次分配到的区间信息 */
	free_vmap_cache = &va->rb_node;
	spin_unlock(&vmap_area_lock);

	BUG_ON(va->va_start & (align-1));
	BUG_ON(va->va_start < vstart);
	BUG_ON(va->va_end > vend);

	return va;

overflow:/* 运行到这里，说明地址空间不足 */
	spin_unlock(&vmap_area_lock);
	if (!purged) {/* 如果我们还没有进行KVA地址空间清理(如果幸运的话，我们可以清理出一些可用空间) */
	/* 清理懒模式下没有及时释放的KVA空间，内核中经常用到懒模式，性能方面的原因 */
		purge_vmap_area_lazy();
		/* 设置标志，表示我们已经尝试过清理KVA空间了 */
		purged = 1;
		/* 再试一次，如果还不行再返回错误 */
		goto retry;
	}
	if (printk_ratelimit())
		pr_warn("vmap allocation for size %lu failed: "
			"use vmalloc=<size> to increase size.\n", size);
	kfree(va);
	return ERR_PTR(-EBUSY);
}
```

### __vmalloc_area_node

```c
/* 根据vm_struct的信息进行物理空间申请，建立页表映射 */
static void *__vmalloc_area_node(struct vm_struct *area, gfp_t gfp_mask,
				 pgprot_t prot, int node)
{
	const int order = 0;
	struct page **pages;
	unsigned int nr_pages, array_size, i;
	const gfp_t nested_gfp = (gfp_mask & GFP_RECLAIM_MASK) | __GFP_ZERO;
	const gfp_t alloc_mask = gfp_mask | __GFP_NOWARN;
	/* 计算本次申请的vmalloc共需要多少页内存 */
	nr_pages = get_vm_area_size(area) >> PAGE_SHIFT;
	array_size = (nr_pages * sizeof(struct page *));

	area->nr_pages = nr_pages;
	/* 根据需要的内存页数，申请管理结构体pages内存空间. */
	if (array_size > PAGE_SIZE) {
	/* 如果需要多余1页内存，则通过vmalloc申请. */
		pages = __vmalloc_node(array_size, 1, nested_gfp|__GFP_HIGHMEM,
				PAGE_KERNEL, node, area->caller);
		area->flags |= VM_VPAGES;
	} else {
		/* 否则从slub中申请. */
		pages = kmalloc_node(array_size, nested_gfp, node);
	}
	area->pages = pages;
	if (!area->pages) {
		remove_vm_area(area->addr);
		kfree(area);
		return NULL;
	}

	for (i = 0; i < area->nr_pages; i++) {
		struct page *page;
		/* 从伙伴系统中申请内存. */
		if (node == NUMA_NO_NODE)
			page = alloc_page(alloc_mask);
		else
			page = alloc_pages_node(node, alloc_mask, order);

		if (unlikely(!page)) {
			/* Successfully allocated i pages, free them in __vunmap() */
			area->nr_pages = i;
			goto fail;
		}
		area->pages[i] = page;
		if (gfp_mask & __GFP_WAIT)
			cond_resched();
	}
	/* 
	 * 物理内存申请成功后，进行内存映射.
	 * vmap_page_range_noflush: 建立映射；
	 * set_pte_at(&init_mm, addr, pte, mk_pte(page, prot)): 将页描述符对应的页框和页表项进行关联，映射关系被建立
	 * flush_cache_vmap：刷新缓存；
	 * 详细过程可参考资料
	 */
	if (map_vm_area(area, prot, pages))
		goto fail;
	return area->addr;

fail:
	warn_alloc_failed(gfp_mask, order,
			  "vmalloc: allocation failure, allocated %ld of %ld bytes\n",
			  (area->nr_pages*PAGE_SIZE), area->size);
	vfree(area->addr);
	return NULL;
}
```

## vmalloc内存释放

### vfree

```c
/**
 *	vfree  -  release memory allocated by vmalloc()
 *	@addr:		memory base address
 *
 *	释放通过以下函数申请的内存：vmalloc(), vmalloc_32() or __vmalloc().
 *	如果当前CPU处于中断中，则需要异步释放
 *	最终的释放操作都是由__vunmap()执行
 */
void vfree(const void *addr)
{
	BUG_ON(in_nmi());

	kmemleak_free(addr);

	if (!addr)
		return;
	if (unlikely(in_interrupt())) {
	/* 
	 * 如果当前释放操作在中断中,则将释放的内存空间信息加入到当前CPU的vfree_deferred管理链表中
	 * 继而通过schedule_work()唤醒工作队列，对内存进行一步释放操作
	*/
		struct vfree_deferred *p = this_cpu_ptr(&vfree_deferred);
		if (llist_add((struct llist_node *)addr, &p->list))
			schedule_work(&p->wq);
	} else
		/* 如果不在中断中，则直接释放 */
		__vunmap(addr, 1);
}
```

### __vunmap

```c
/* 
 * 释放vmalloc内存
 * deallocate_pages：0表示由vumap()调用；1表示由vfree()调用
 * 涉及3块内存的释放：
 * 1. vmap_struct管理结构体内存：延迟释放
 * 		在free_vmap_area_noflush()中将vmp_area加入到vmap_purge_list链表
 * 		当加入该链表的数量超过一定值之后，才会通过try_purge_vmap_area_lazy()将vmap_area真正释放掉
 * 2. page管理结构体：vfree(area->pages)、kfree(area->pages)
 * 3. vmalloc真实内存：__free_page(page);
 * 4. vm_struct管理结构体：kfree(area)
 */
static void __vunmap(const void *addr, int deallocate_pages)
{
	struct vm_struct *area;

	if (!addr)
		return;

	if (WARN(!PAGE_ALIGNED(addr), "Trying to vfree() bad address (%p)\n",
			addr))
		return;
	/* 
	 * remove_vm_area()->find_vmap_area()->__find_vmap_area()通过addr在红黑树中找到对应节点对应的struct vmap_area结构体 
	 * 再通过free_unmap_vmap_area()清除指定的虚拟地址
	 */
	area = remove_vm_area(addr);
	if (unlikely(!area)) {
		WARN(1, KERN_ERR "Trying to vfree() nonexistent vm area (%p)\n",
				addr);
		return;
	}

	debug_check_no_locks_freed(addr, area->size);
	debug_check_no_obj_freed(addr, area->size);

	if (deallocate_pages) {
		int i;

		for (i = 0; i < area->nr_pages; i++) {
			struct page *page = area->pages[i];

			BUG_ON(!page);
			__free_page(page);
		}

		if (area->flags & VM_VPAGES)
			vfree(area->pages);
		else
			kfree(area->pages);
	}

	kfree(area);
	return;
}
```

## 附录

### vm_struct

```c
struct vm_struct {
	/* 使得内核可以将vmalloc区域中的所有子区域保存在一个单链表上. */
    struct vm_struct    *next;
	/* 定义了分配的子区域在虚拟地址空间中的起始地址. */
    void            *addr;
	/* size表示该子区域的长度. */
    unsigned long       size;
	/* 存储了与该内存区关联的标志集合, 它只用于指定内存区类型. */
    unsigned long       flags;
	/* 二维指针，指向page指针的数组;每个数组成员都表示一个映射到虚拟地址空间中的物理内存页的page实例. */
    struct page         **pages;
	/* 指定pages中数组项的数目，即涉及的内存页数目. */
    unsigned int        nr_pages;
	/* 仅当用ioremap映射了由物理地址描述的物理内存区域时才需要. */
    phys_addr_t         phys_addr;
	/* The first cache line has the info for VMA tree walking. */
    const void          *caller;
};
```

### vmap_area

> vmap_area表示内核空间的vmalloc区域的一个vmalloc，由rb_node和list进行串联

```c
struct vmap_area {
	unsigned long va_start;
	unsigned long va_end;
	unsigned long flags;
	struct rb_node rb_node;         /* address sorted rbtree */
	struct list_head list;          /* address sorted list */
	struct list_head purge_list;    /* "lazy purge" list */
	struct vm_struct *vm;
	struct rcu_head rcu_head;
};
```

### vm_area_struct

```c
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 * 用户态进程内存对应内核管理结构体；每个vm_area_struct代表进程的一个虚拟地址空间
 */
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */

	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;

	/*
	 * Largest free memory gap in bytes to the left of this VMA.
	 * Either between this VMA and vma->vm_prev, or between one of the
	 * VMAs below us in the VMA rbtree and its ->vm_prev. This helps
	 * get_unmapped_area find a free area of the right size.
	 */
	unsigned long rb_subtree_gap;

	/* Second cache line starts here. */

	struct mm_struct *vm_mm;	/* 指向VMA所属进程的struct mm_struct结构. */
	pgprot_t vm_page_prot;		/* VMA访问权限. */
	unsigned long vm_flags;		/* VMA标志位. */

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap interval tree.
	 */
	struct {
		struct rb_node rb;
		unsigned long rb_subtree_last;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_chain; /* 用于管理RMAP反向映射 */
	struct anon_vma *anon_vma;	/* 用于管理RMAP反向映射 */

	/* VMA操作函数合集，常用于文件映射. */
	const struct vm_operations_struct *vm_ops;

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* 指定文件映射的偏移量，单位是页面；标识文件映射部分的起始位置 */
	struct file * vm_file;		/* 描述一个被映射的文件. 如果非文件映射，则NULL */
	void * vm_private_data;		/* was vm_pte (shared mem) */

#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
};
```

## 参考资料

[vmalloc不连续内存管理1](https://www.jeanleo.com/2018/09/09/【linux内存源码分析】vmalloc不连续内存管理（1）/)

[vmalloc不连续内存管理2](https://www.jeanleo.com/2018/09/09/【linux内存源码分析】vmalloc不连续内存管理（2）/)

[vmalloc不连续内存管理3](https://www.jeanleo.com/2018/09/09/【linux内存源码分析】vmalloc不连续内存管理（3）/)

[vmalloc分配源码详细分析](http://blog.chinaunix.net/uid-25845340-id-3251137.html)

[map_vm_area()源码详细分析](http://edsionte.com/techblog/archives/3559)
