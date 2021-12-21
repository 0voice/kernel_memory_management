## 前沿

### 往篇回顾

> 上一篇中，我主要学习了缺页中断在内核中的处理过程；低估了该部分的复杂度，尝试使用一篇来记录主要内容，到后面发现实在太长了，影响了学习的心情，到后面do_wp_page(写时复制)以及do_swap_page(访问内存在swap区)都法仔细看了；后面回过头来肯定要再学习。

- 在入口函数do_page_fault中，判断了cpu当前状态、获取锁等操作后，调用__do_page_fault做进一步处理
- 在__do_page_fault中判断请求中断addr是否合法(通过find_vma(addr)的结果)，如果合法则调用handle_mm_fault->__handle_mm_fault进一步处理
- __handle_mm_fault增加了对透明巨页的支持，在非巨页模式下则调用handle_pte_fault做最终的中断处理，分为以下几种不同情况

### 缺页中断不同情况处理总结

| PTE_PRESENT | pte entry是否为空 | vma->vm_ops |    brk处理函数    |              场景               |                             备注                             |
| :---------- | :---------------: | ----------: | :---------------: | :-----------------------------: | :----------------------------------------------------------: |
| 未置位      |        空         |      已定义 |     do_fault      |        文件映射缺页中断         | 根据WRITE、SHARED两个标志位状态分为读brk,私有映射COW,共享映射COW |
| 未置位      |        空         |      未定义 | do_anonymous_page |        由malloc引发的brk        | 根据WRITE标志，如果仅读，则只需返回zero page；否则申请物理页面 |
| 置位        |       非空        |          NA |   do_swap_page    |        访问页面在swap区         |     如果不在swap cache中，则需要swap in；需要进一步学习      |
| 置位        |       非空        |          NA |    do_wp_page     | 由fork后访问引起的缺页中断，COW | 如果WRITE、SHARED同时置位，则不需要分配页面；需要进一步学习  |

### 反向映射RMAP

#### 基本概念

- 从物理页面到虚拟地址空间VMA的反向映射，即针对page，通过RMAP可以找出所有使用该page的VMA
- 一个物理页面可以同时被多个进程的虚拟地址内存映射(fork, KSM)，但一个虚拟页面同时只能有一个物理页面与之映射

#### 实现基础

- 通过[struct anon_vma](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/23/ARM64内存管理十六-反向映射RMAP/#anon_vma), [struct anon_vma_chain](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/23/ARM64内存管理十六-反向映射RMAP/#anon_vma_chain)和[sturct vm_area_struct](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/23/ARM64内存管理十六-反向映射RMAP/#vm_area_struct)建立了联系，通过物理页面反向查找到VMA
- anon_vma是per_process的；对于同一块匿名线性映射区(vm_area_struct)，如果被不同几个进程使用，则每个进程中都有一个anon_vma与之对应
- anon_vma_chain则是把使用了该匿名线性映射区(vm_area_struct)的进程都串起来了

#### 用途

- 如果页面要被回收(kswapd)、迁移(swap)，就必须要找出哪些进程在使用这个页面，然后断开这些虚拟地址到物理页面的映射
- 匿名页面断开映射操作在rmap_walk_anon中进行

## 反向映射代码流程

反向映射是跟随匿名线性区物理页面实际分配而创建、更新的；大概在下面两种情况：

- malloc触发匿名线性区缺页异常, 分配新页面过程中同时建立该页面的rmap
- fork后更新父进程的所有rmap信息
- 栈空间扩张，申请的新页需要建立页面的rmap

### malloc触发匿名线性区缺页异常

主要处理在`do_anonymous_page`中，该函数大概流程已经在前一篇中分析了；现在着重分析和反向映射相关的部分，主要分为两步：

- 通过anon_vma_prepare函数确保vma拥有av,avc，这两个结构体是RMAP的根基
- 通过page_add_new_anon_rmap函数为新分配的page建立RMAP

#### anon_vma_prepare

do_anonymous_page->anon_vma_prepare: 在确定要给vma分配物理页面后；首先调用`anon_vma_prepare`创建该vma的av及avc

```c
/* 
 * 目标：创建该vma的av及avc；当然如果vma->anon_vma!=NULL,则什么都不做
 * 1. 从slub中申请一个avc管理结构体
 * 2. 尝试检查该vma是否可以前后合并，可以则复用已有的ac，否则从slub中新申请一个av
 * 3. 使用得到的av,avc初始化vma中成员变量；并用anon_vma_chain_link初始化avc
 */
int anon_vma_prepare(struct vm_area_struct *vma)
{
	struct anon_vma *anon_vma = vma->anon_vma;
	struct anon_vma_chain *avc;

	might_sleep();
	if (unlikely(!anon_vma)) {
	/* 如vma->anon_vma==NULL */
		struct mm_struct *mm = vma->vm_mm;
		struct anon_vma *allocated;
		/* 从slab中分配一个avc管理结构体 */
		avc = anon_vma_chain_alloc(GFP_KERNEL);
		if (!avc)
			goto out_enomem;
        /* 
		 * 检查vma能否与其前/后vma进行合并，如果可以，则返回能够合并的那个vma的anon_vma，如果不可以，返回NULL
         * 1. 主要检查vma前后的vma是否连在一起(vma->vm_end == 前/后vma->vm_start)
         * 2. vma->vm_policy和前/后vma->vm_policy
         * 3. 是否都为文件映射，除了(VM_READ|VM_WRITE|VM_EXEC|VM_SOFTDIRTY)其他标志位是否相同，如果为文件映射，前/后vma映射的文件位置是否正好等于vma映射的文件 + vma的长度
         */
		anon_vma = find_mergeable_anon_vma(vma);
		allocated = NULL;
		if (!anon_vma) {
		/* 不可以合并，则为该vma申请一个av管理结构体，并初始化数据
		 * 1. refcount=1;
		 * 2. anon_vma->root=self; anon_vma->parent=self;
		 * 3. degree = 1;
		 */
			anon_vma = anon_vma_alloc();
			if (unlikely(!anon_vma))
				goto out_enomem_free_avc;
			allocated = anon_vma;
		}
		/* 获取av锁 */
		anon_vma_lock_write(anon_vma);
		/* page_table_lock to protect against threads */
		spin_lock(&mm->page_table_lock);
		if (likely(!vma->anon_vma)) {
			/* 将刚申请的或合并的av赋给目标vma */
			vma->anon_vma = anon_vma;
			/* 初始化刚申请的avc管理结构体：
			 * 1. 将avc->same_vma挂入vma->anon_vma_chain链表
			 * 2. 将avc插入anon_vma红黑树
			 */
			anon_vma_chain_link(vma, avc, anon_vma);
			
			anon_vma->degree++;
			/* 置空后释放 */
			allocated = NULL;
			avc = NULL;
		}
		spin_unlock(&mm->page_table_lock);
		anon_vma_unlock_write(anon_vma);

		if (unlikely(allocated))
			put_anon_vma(allocated);
		if (unlikely(avc))
			anon_vma_chain_free(avc);
	}
	/* 如果vma->anon_vma!=NULL,则什么都不做 */
	return 0;

 out_enomem_free_avc:
	anon_vma_chain_free(avc);
 out_enomem:
	return -ENOMEM;
}
```

#### page_add_new_anon_rmap

do_anonymous_page->page_add_new_anon_rmap

```c
/* 
 * 通过anon_vma_prepare，确保了中断的vma一定有相应的av,avc；如此就有了反向映射基本框架
 * 目标：为vma中的page添加反向映射
 * page: 目标页; vma: 访问此页的vma; address: 线性地址
 * 
 */
void page_add_new_anon_rmap(struct page *page,
	struct vm_area_struct *vma, unsigned long address)
{
	 /* 地址必须处于vma中 */
	VM_BUG_ON_VMA(address < vma->vm_start || address >= vma->vm_end, vma);
	SetPageSwapBacked(page);
	/* 设置此页的_mapcount = 0，说明此页正在使用，但是是非共享的(>0是共享) */
	atomic_set(&page->_mapcount, 0); /* increment count (starts at -1) */
	if (PageTransHuge(page))/* 如果是透明大页 */
		__inc_zone_page_state(page, NR_ANON_TRANSPARENT_HUGEPAGES);
	__mod_zone_page_state(page_zone(page), NR_ANON_PAGES,
			hpage_nr_pages(page));
	/* 进行反向映射
     * 设置page->mapping最低位为1,表示该页为匿名页
     * page->mapping指向此vma->anon_vma
     * page->index存放此page在vma中的虚拟页框号，计算方法：page->index = ((address - vma->vm_start) >> PAGE_SHIFT) + vma->vm_pgoff;
     */
	__page_set_anon_rmap(page, vma, address, 1);
}
```

### fork子进程后触发COW,为新进程建立RMAP根基

fork过程中，子进程会拷贝父进程的vma(除标记VM_DONTCOPY)和这些vma对应的页表项，即使vma是非共享的；要确保在fork后能通过RMAP找到父子进程对应的vma，主要的处理过程在`dup_mm()`中

#### dup_mm

```c
static int dup_mmap(struct mm_struct *mm, struct mm_struct *oldmm)
{
	struct vm_area_struct *mpnt, *tmp, *prev, **pprev;
	struct rb_node **rb_link, *rb_parent;
	int retval;
	unsigned long charge;
	/* 获取每CPU的dup_mmap_sem这个读写信号量的读锁 */
	uprobe_start_dup_mmap();
	/* 获取父进程的mm->mmap_sem的写锁 */
	down_write(&oldmm->mmap_sem);
	flush_cache_dup_mm(oldmm);
	/* 如果父进程的mm_struct的flags设置了MMF_HAS_UPROBES，
	 *　则子进程的mm_struct的flags设置MMF_HAS_UPROBES和MMF_RECALC_UPROBES 
	 */
	uprobe_dup_mmap(oldmm, mm);
	/* 获取子进程的mmap_sem，后面SINGLE_DEPTH_NESTING意思需要查查 */
	down_write_nested(&mm->mmap_sem, SINGLE_DEPTH_NESTING);

	/* No ordering required: file already has been exposed. */
	RCU_INIT_POINTER(mm->exe_file, get_mm_exe_file(oldmm));
	/* 复制父进程进程相关数据到子进程的mm */
	mm->total_vm = oldmm->total_vm;
	mm->shared_vm = oldmm->shared_vm;
	mm->exec_vm = oldmm->exec_vm;
	mm->stack_vm = oldmm->stack_vm;
	/* 子进程vma红黑树的根结点，保存到rb_link中 */
	rb_link = &mm->mm_rb.rb_node;
	rb_parent = NULL;
	/* 获取指向线性区对象的链表头，链表是经过排序的，按线性地址升序排列，
	 * 但是mmap并不是list_head结构，是一个struct vm_area_struct指针，
	 * 这里由于子进程的mm是刚创建的，mm->map为空，而pprev是一个指向指针的指针 */
	pprev = &mm->mmap;
	retval = ksm_fork(mm, oldmm);
	if (retval)
		goto out;
	retval = khugepaged_fork(mm, oldmm);
	if (retval)
		goto out;

	prev = NULL;
	/* 遍历父进程所有vma，通过mm->mmap链表遍历 */
	for (mpnt = oldmm->mmap; mpnt; mpnt = mpnt->vm_next) {
		struct file *file;

		if (mpnt->vm_flags & VM_DONTCOPY) {
		/* 父进程中标记了VM_DONTCOPY的vma不复制，要在子进程中将该段删掉 */
			vm_stat_account(mm, mpnt->vm_flags, mpnt->vm_file,
							-vma_pages(mpnt));
			continue;
		}
		charge = 0;
		/* 此vma要求需要检查是否有足够的空闲内存用于映射 */
		if (mpnt->vm_flags & VM_ACCOUNT) {
		/* 此vma的页数量, (mpnt->vm_end - mpnt->vm_start) >> PAGE_SHIFT */
			unsigned long len = vma_pages(mpnt);
			/* 安全检查，是否有足够的内存 */
			if (security_vm_enough_memory_mm(oldmm, len)) /* sic */
				goto fail_nomem;
			charge = len;
		}
		/* 分配一个vma结构体用于子进程该vma的管理结构体 */
		tmp = kmem_cache_alloc(vm_area_cachep, GFP_KERNEL);
		if (!tmp)
			goto fail_nomem;
		*tmp = *mpnt;
		/* 初始化子进程新的vma的anon_vma_chain为空 */
		INIT_LIST_HEAD(&tmp->anon_vma_chain);
		/* 视情况复制父进程vma的权限，非vm_flags */
		retval = vma_dup_policy(mpnt, tmp);
		if (retval)
			goto fail_nomem_policy;
		/* 将子进程新的vma的vm_mm指向子进程的mm */
		tmp->vm_mm = mm;
		/* 关键函数：对父子进程的anon_vma和anon_vma_chain进行处理 
         * 如果父进程的此vma没有anon_vma，直接返回
		 * 1. vma用于映射文件应该没有anon_vma
		 * 2. 该vma中没有实际分配page
         */
		if (anon_vma_fork(tmp, mpnt))
			goto fail_nomem_anon_vma_fork;
		tmp->vm_flags &= ~VM_LOCKED;
		tmp->vm_next = tmp->vm_prev = NULL;
		/* 获取vma所映射的文件，如果是匿名映射区，则为空 */
		file = tmp->vm_file;
		if (file) {
			/* 共享文件映射场景，暂不考虑 */
			struct inode *inode = file_inode(file);
			struct address_space *mapping = file->f_mapping;

			get_file(file);
			if (tmp->vm_flags & VM_DENYWRITE)
				atomic_dec(&inode->i_writecount);
			i_mmap_lock_write(mapping);
			if (tmp->vm_flags & VM_SHARED)
				atomic_inc(&mapping->i_mmap_writable);
			flush_dcache_mmap_lock(mapping);
			/* insert tmp into the share list, just after mpnt */
			vma_interval_tree_insert_after(tmp, mpnt,
					&mapping->i_mmap);
			flush_dcache_mmap_unlock(mapping);
			i_mmap_unlock_write(mapping);
		}

		/* 此vma用于映射hugetlbfs中的大页的情况 */
		if (is_vm_hugetlb_page(tmp))
			reset_vma_resv_huge_pages(tmp);

        /* 
		 * 用于构建子进程的vma链表
		 * pprev是指向子进程的mm->mmap(用于vma排序存放的链表) 
         * 第一次循环时将子进程的mm->mmap指向tmp
         * 后面的循环将前一个vma的vm_next指向当前mva
         */
		*pprev = tmp;
		pprev = &tmp->vm_next;
		tmp->vm_prev = prev;
		prev = tmp;
		/* 将vma加入到mm的mm_rb这个vma红黑树中 */
		__vma_link_rb(mm, tmp, rb_link, rb_parent);
		rb_link = &tmp->vm_rb.rb_right;
		rb_parent = &tmp->vm_rb;
		/* 子进程mm的线性区个数++ */
		mm->map_count++;
		/* 做页表的复制
         * 将父进程的vma对应的开始地址到结束地址这段地址的页表复制到子进程中
         * 如果这段vma有可能会进行写时复制(vma可写，并且不是共享的VM_SHARED)，那就会对子进程和父进程的页表项都设置为映射的页是只读的(vma中权限是可写)
         */
		retval = copy_page_range(mm, oldmm, mpnt);

		if (tmp->vm_ops && tmp->vm_ops->open)
			tmp->vm_ops->open(tmp);

		if (retval)
			goto out;
	}
	/* 与体系架构相关的dup_mmap处理 */
	arch_dup_mmap(oldmm, mm);
	retval = 0;
out:
	/* 释放子进程mm->mmap_sem这个读写信号量的写锁 */
	up_write(&mm->mmap_sem);
	/* 刷新父进程的页表tlb */
	flush_tlb_mm(oldmm);
	/* 释放父进程mm->mmap_sem这个读写信号量的写锁 */
	up_write(&oldmm->mmap_sem);
	uprobe_end_dup_mmap();
	return retval;
fail_nomem_anon_vma_fork:
	mpol_put(vma_policy(tmp));
fail_nomem_policy:
	kmem_cache_free(vm_area_cachep, tmp);
fail_nomem:
	retval = -ENOMEM;
	vm_unacct_memory(charge);
	goto out;
}
```

#### anon_vma_fork

```c
/* 
 * 完成对父进程单个vma的RMAP处理
 * 1. 遍历父进程该pvma的anon_vma_chain，复制后avc->vma链接到子进程vma的avc链中;avc->av指向父avc->av
 * 2. 申请子进程对应该vma的av、avc；并初始化成员变量
 */
int anon_vma_fork(struct vm_area_struct *vma, struct vm_area_struct *pvma)
{
	struct anon_vma_chain *avc;
	struct anon_vma *anon_vma;
	int error;

	/* 
	 * 如果父进程的此vma没有anon_vma，直接返回
	 * 1. vma用于映射文件应该没有anon_vma
	 * 2. 该vma中没有实际分配page
	 */
	if (!pvma->anon_vma)
		return 0;

	/* Drop inherited anon_vma, we'll reuse existing or allocate new. */
	vma->anon_vma = NULL;

    /* 
     * 1. 遍历父进程的vma的anon_vma_chain链表，对每个结点新建一个anon_vma_chain
     * 2. 设置新的avc->vma指向子进程的vma
     * 3. 设置新的avc->anon_vma指向父进程anon_vma_chain结点指向的anon_vma(这个anon_vma不一定属于父进程)
     * 4. 将新的avc->same_vma加入到子进程的anon_vma_chain链表中
     * 5. 将新的avc->rb加入到父进程anon_vma_chain结点指向的anon_vma
     */
	error = anon_vma_clone(vma, pvma);
	if (error)
		return error;

	/* An existing anon_vma has been reused, all done then. */
	if (vma->anon_vma)
		return 0;

	/* 申请子进程对于该vma独有的av. */
	anon_vma = anon_vma_alloc();
	if (!anon_vma)
		goto out_error;
	/* 申请子进程对于该vma独有的avc. */
	avc = anon_vma_chain_alloc(GFP_KERNEL);
	if (!avc)
		goto out_error_free_anon_vma;

	/* 将新的anon_vma的root指向父进程的anon_vma的root */
	anon_vma->root = pvma->anon_vma->root;
	anon_vma->parent = pvma->anon_vma;
	
	/* 对父进程与子进程的anon_vma共同的root的refcount进行+1 */
	get_anon_vma(anon_vma->root);
	/* Mark this anon_vma as the one where our new (COWed) pages go. */
	vma->anon_vma = anon_vma;
	/* 对这个新的anon_vma上锁 */
	anon_vma_lock_write(anon_vma);
	/* 新的avc的vma指向子进程的vma
     * 新的avc的anon_vma指向子进程vma的anon_vma
     * 新的avc的same_vma加入到子进程vma的anon_vma_chain链表的头部
     * 新的avc的rb加入到子进程vma的anon_vma的红黑树中
     */
	anon_vma_chain_link(vma, avc, anon_vma);
	anon_vma->parent->degree++;
	/* 对这个anon_vma解锁 */
	anon_vma_unlock_write(anon_vma);

	return 0;

 out_error_free_anon_vma:
	put_anon_vma(anon_vma);
 out_error:
	unlink_anon_vmas(vma);
	return -ENOMEM;
}
```

#### anon_vma_clone

```c
/* 
 * 对vma->avc中所有avc，申请新的acv管理结构体，并初始化成员数据
 * 新的avc的vma指向子进程的vma
 * 新的avc的anon_vma指向子进程vma的anon_vma
 * 新的avc的same_vma加入到子进程vma的anon_vma_chain链表的头部
 * 新的avc的rb加入到子进程vma的anon_vma的红黑树中
 */
int anon_vma_clone(struct vm_area_struct *dst, struct vm_area_struct *src)
{
	struct anon_vma_chain *avc, *pavc;
	struct anon_vma *root = NULL;
	/* 遍历父进程的每个anon_vma_chain链表中的结点，保存在pavc中 */
	list_for_each_entry_reverse(pavc, &src->anon_vma_chain, same_vma) {
		struct anon_vma *anon_vma;
		/* 快速分配一个新的avc结构 */
		avc = anon_vma_chain_alloc(GFP_NOWAIT | __GFP_NOWARN);
		/* 如果分配失败 */
		if (unlikely(!avc)) {
			unlock_anon_vma_root(root);
			root = NULL;
			/* 再次分配，一定要分配成功 */
			avc = anon_vma_chain_alloc(GFP_KERNEL);
			if (!avc)
				goto enomem_failure;
		}
		/* 获取父结点的pavc指向的anon_vma */
		anon_vma = pavc->anon_vma;
		/* 对anon_vma的root上锁
         * 如果root != anon_vma->root，则对root上锁，并返回anon_vma->root
         * 第一次循环，root = NULL
         */
		root = lock_anon_vma_root(root, anon_vma);
		/* 
         * 设置新的avc->vma指向子进程的vma
         * 设置新的avc->anon_vma指向父进程anon_vma_chain结点指向的anon_vma(这个anon_vma不一定属于父进程)
         * 将新的avc->same_vma加入到子进程的anon_vma_chain链表头部
         * 将新的avc->rb加入到父进程anon_vma_chain结点指向的anon_vma
         */
		anon_vma_chain_link(dst, avc, anon_vma);

		/*
		 * 如果pav->degree<2则可以复用av，否则要重新申请
		 */
		if (!dst->anon_vma && anon_vma != src->anon_vma &&
				anon_vma->degree < 2)
			dst->anon_vma = anon_vma;
	}
	if (dst->anon_vma)
		dst->anon_vma->degree++;
	unlock_anon_vma_root(root);
	return 0;

 enomem_failure:
	/*
	 * dst->anon_vma is dropped here otherwise its degree can be incorrectly
	 * decremented in unlink_anon_vmas().
	 * We can safely do this because callers of anon_vma_clone() don't care
	 * about dst->anon_vma if anon_vma_clone() failed.
	 */
	dst->anon_vma = NULL;
	unlink_anon_vmas(dst);
	return -ENOMEM;
}
```

### 栈空间扩张，申请的新页需要建立页面的rmap

一个进程的栈对应一个vma，如果系统所有栈大小足够使用，则所有进程栈的vma->vm_start,vm_end都相同；空间不够，则调用`expand_downwards()`向下扩张

#### expand_downwards

```c
/* 
 * 
 */
int expand_downwards(struct vm_area_struct *vma,
				   unsigned long address)
{
	int error;

	/* 如果栈已经在使用中，是栈空间不足导致需要向下增长的，那么这里基本不做任何工作，因为栈的vma已经有了anon_vma */
	if (unlikely(anon_vma_prepare(vma)))
		return -ENOMEM;
	/* 获取目标address线性地址对应的页的起始地址 */
	address &= PAGE_MASK;
	/* 检查此地址是否合理 */
	error = security_mmap_addr(address);
	if (error)
		return error;
	/* 对anon_vma->root->rwsem上锁 */
	vma_lock_anon_vma(vma);

	/*
	 * vma->vm_start/vm_end cannot change under us because the caller
	 * is required to hold the mmap_sem in read mode.  We need the
	 * anon_vma lock to serialize against concurrent expand_stacks.
	 */

	/* 因为是向下扩展，address一定会小于vma->vm_start，因为外面一层函数判断过了才会调用到此函数 */
	if (address < vma->vm_start) {
		unsigned long size, grow;
		/* address到进程栈结束的大小范围，也就是栈扩大后最小的大小 */
		size = vma->vm_end - address;
		/* 需要扩张的大小(以页为单位) */
		grow = (vma->vm_start - address) >> PAGE_SHIFT;

		error = -ENOMEM;
		if (grow <= vma->vm_pgoff) {
			/* 做检查，主要是空间大小及rlimit限制检查 */
			error = acct_stack_growth(vma, size, grow);
			if (!error) {
				/* 对该进程页表上锁 */
				spin_lock(&vma->vm_mm->page_table_lock);
				anon_vma_interval_tree_pre_update_vma(vma);
				vma->vm_start = address;
				vma->vm_pgoff -= grow;
				anon_vma_interval_tree_post_update_vma(vma);
				vma_gap_update(vma);
				spin_unlock(&vma->vm_mm->page_table_lock);

				perf_event_mmap(vma);
			}
		}
	}
	vma_unlock_anon_vma(vma);
	khugepaged_enter_vma_merge(vma, vma->vm_flags);
	validate_mm(vma->vm_mm);
	return error;
}
```

## 附录

### 映射搜索

- 父进程中已经映射了的vma匿名页，通过其指向的av中保存的avc红黑树遍历各个映射了此页的vma；
- 父进程中没有映射的vma匿名页，如果子进程中访问了该页，则在do_anonymous_page()函数中会被映射，RMAP时只能访问到子进程avc；
- 父进程中没有映射的vma匿名页，fork后访问了该页，则在do_anonymous_page()函数中会被映射，RMAP时能访问到子进程avc，这时需要page->index来判断子进程avc是否涉及；

### anon_vma_chain

```c
struct anon_vma_chain {
    /* 此结构所属的vma */
    struct vm_area_struct *vma;
    /* 此结构加入的红黑树所属的anon_vma */
    struct anon_vma *anon_vma;
    /* 用于加入到所属vma的anon_vma_chain链表中；记录了该chain属于哪些vma */
    struct list_head same_vma;  
    /* 用于加入到其他进程或者本进程vma的anon_vma的红黑树中 */
    struct rb_node rb;           
    unsigned long rb_subtree_last;
#ifdef CONFIG_DEBUG_VM_RB
    unsigned long cached_vma_start, cached_vma_last;
#endif
};
```

### anon_vma

```c
/* 匿名线性区描述符，每个匿名vma都会有一个这个结构 */
struct anon_vma {
    /* 指向此anon_vma所属的root */
    struct anon_vma *root;        /* Root of this anon_vma tree */
    /* 读写信号量 */
    struct rw_semaphore rwsem;    /* W: modification, R: walking the list */

    /* 红黑树中结点数量，初始化时为1，也就是只有本结点，当加入root的anon_vma的红黑树时，此值不变 */
    atomic_t refcount;


    /* 红黑树的根，用于存放引用了此anon_vma所属线性区中的页的所有avc */
    struct rb_root rb_root;    /* Interval tree of private "related" vmas */
};
```

### vm_area_struct

```c
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */

	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

	/* 整个链表会按地址大小递增排序 */
	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;

 /* 此vma的子树中最大的空闲内存块大小(bytes) */
	unsigned long rb_subtree_gap;

	/* Second cache line starts here. */

	struct mm_struct *vm_mm;	/* 指向所属的内存描述符 */
	pgprot_t vm_page_prot;		/* 页表项标志的初值，当增加一个页时，内核根据这个字段的值设置相应页表项中的标志 */
	unsigned long vm_flags;		/* 线性区标志读写可执行权限会复制到页表项中，由分页单元去检查这几个权限 */

/* 链接到反向映射所使用的数据结构，用于文件映射的线性区，主要用于文件页的反向映射 */
	struct {
		struct rb_node rb;
		unsigned long rb_subtree_last;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 * MAP_PRIVATE
	 * 拥有MAP_PRIVATE标识的vma可能存在于 i_mmap,anon_vma list中；
	 * 拥有MAP_SHARED vma只可能存在于i_mmap tree；
	 * 匿名的MAP_PRIVATE、堆和栈的vma都会存在于这个链表中
	 * 如果anon_vma为空，那么anon_vma_chain一定为空
	 * 作用：指向匿名线性区链表头的指针，将此mm_struct中所有匿名线性区链接起来；
	 */
	struct list_head anon_vma_chain; /* Serialized by mmap_sem &
					  * page_table_lock */
	struct anon_vma *anon_vma;	/* 指向anon_vma数据结构的指针，对于匿名线性区，此为重要结构 */

	/* 指向线性区操作的方法，特殊的线性区会设置，默认会为空 */
	const struct vm_operations_struct *vm_ops;

	/* 文件映射：在映射文件中的偏移量
	 * 匿名线性区：为0或vma开始地址对应的虚拟页框号，用于栈区反向映射计算
	 */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units, *not* PAGE_CACHE_SIZE */
	/* 文件映射：建立shmem共享内存中返回的struct file
	 * 匿名线性区：NULL或一个匿名文件
	 */				   
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* 指向内存区的私有映射 */

#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
};
```

### anon_vma

```c
/* 每个匿名线性区描述符都有一个 */
struct anon_vma {
	struct anon_vma *root;		/* 指向此anon_vma所属的root */
	struct rw_semaphore rwsem;	/* W: modification, R: walking the list */
	/* 红黑树中节点数量，初始化为1，即只有本结点；当加入root的anon_vma的红黑树时，此值不变 */
	atomic_t refcount;

	/*
	 * Count of child anon_vmas and VMAs which points to this anon_vma.
	 *
	 * This counter is used for making decision about reusing anon_vma
	 * instead of forking new one. See comments in function anon_vma_clone.
	 */
	unsigned degree;

	struct anon_vma *parent;	/* Parent of this anon_vma */

	/* rb root，用于存放引用了此anon_vma所属线性区中的页的其它线性区，用于匿名反向映射 */
	struct rb_root rb_root;	/* Interval tree of private "related" vmas */
};
```

### mm_struct

```c
/* 
 * 1. 内存描述符，除了内核线程(使用被调度出去进程的mm_struct)、轻量级进程外(父进程mm_struct)，每个进程都有一个该结构体 
 * 2. 所有内存描述符放在一个双向链表中，链表中第一个元素是init_mm
 */
struct mm_struct {
	/* 指向线性区对象的链表头, 链表是经过排序的，按线性地址升序排列，里面包括匿名映射线性区和文件映射线性区 */
	struct vm_area_struct *mmap;		/* list of VMAs */
	struct rb_root mm_rb;
	u32 vmacache_seqnum;                   /* per-thread vmacache */
#ifdef CONFIG_MMU
    /* 在进程地址空间中找一个可以使用的线性地址空间，查找一个空闲的地址区间
     * len: 指定区间的长度
     * 返回新区间的起始地址
     */
	unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
#endif
	/* 标识第一个分配的匿名线性区或文件内存映射的线性地址 */
	unsigned long mmap_base;		/* base of mmap area */
	unsigned long mmap_legacy_base;         /* base of mmap area in bottom-up allocations */
	unsigned long task_size;		/* size of task vm space */
	/* 所有vma中最大的结束地址 */
	unsigned long highest_vm_end;		/* highest vma end address */
	pgd_t * pgd;	/* 指向页全局目录 */
	/* 次数计数器，存放了共享此mm_struct的轻量级进程个数，但所有mm_users在mm_count的计算中只算1 */
	atomic_t mm_users;			/* How many users with user space? */
	/* 主使用计数器，当mm_count递减时，系统会检查是否为0，为0则解除这个mm_struct */
	atomic_t mm_count;			/* How many references to "struct mm_struct" (users count as 1) */
	atomic_long_t nr_ptes;			/* 页表数 */
#if CONFIG_PGTABLE_LEVELS > 2
	atomic_long_t nr_pmds;			/* PMD page table pages */
#endif
	/* 线性区的个数，默认最多是65535个，系统管理员可以通过写/proc/sys/vm/max_map_count文件修改这个值 */
	int map_count;				/* number of VMAs */
	/* 线性区的自旋锁和页表的自旋锁 */
	spinlock_t page_table_lock;		/* Protects page tables and some counters */
	struct rw_semaphore mmap_sem;	/* 线性区的读写信号量，当需要对某个线性区进行操作时，会获取 */
	/* 用于链入双向链表中 */
	struct list_head mmlist;		/* List of maybe swapped mm's.	These are globally strung
						 * together off init_mm.mmlist, and are protected
						 * by mmlist_lock
						 */

	/* 进程所拥有的最大页框数 */
	unsigned long hiwater_rss;	/* High-watermark of RSS usage */
	/* 进程线性区中的最大页数 */
	unsigned long hiwater_vm;	/* High-water virtual memory usage */

	unsigned long total_vm;		/* 进程地址空间的大小(页框数) */
	unsigned long locked_vm;	/* 锁住而不能换出的页的数量 */
	unsigned long pinned_vm;	/* Refcount permanently increased */
	unsigned long shared_vm;	/* 共享文件内存映射中的页数量 */
	unsigned long exec_vm;		/* 可执行内存映射中的页数量 */
	unsigned long stack_vm;		/* 用户态堆栈的页数量 */
	unsigned long def_flags;
	unsigned long start_code, end_code, start_data, end_data;/* 可执行代码、已初始化数据相关位置 */
	unsigned long start_brk, brk, start_stack;	/* 堆栈相关位置 */
	unsigned long arg_start, arg_end, env_start, env_end;/* 命令行参数、环境变量相关位置 */

	unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

	/*
	 * Special counters, in some configurations protected by the
	 * page_table_lock, in other configurations by being atomic.
	 */
	struct mm_rss_stat rss_stat;

	struct linux_binfmt *binfmt;

	cpumask_var_t cpu_vm_mask_var;

	/* Architecture-specific MM context */
	mm_context_t context;

	unsigned long flags; /* Must use atomic bitops to access the bits */

	struct core_state *core_state; /* coredumping support */
#ifdef CONFIG_AIO
	spinlock_t			ioctx_lock;
	struct kioctx_table __rcu	*ioctx_table;
#endif
#ifdef CONFIG_MEMCG
	/*
	 * "owner" points to a task that is regarded as the canonical
	 * user/owner of this mm. All of the following must be true in
	 * order for it to be changed:
	 *
	 * current == mm->owner
	 * current->mm != mm
	 * new_owner->mm == mm
	 * new_owner->alloc_lock is held
	 */
	struct task_struct __rcu *owner;
#endif

	/* store ref to file /proc/<pid>/exe symlink points to */
	struct file __rcu *exe_file;	/* 代码段中映射的可执行文件的file */
#ifdef CONFIG_MMU_NOTIFIER
	struct mmu_notifier_mm *mmu_notifier_mm;
#endif
#if defined(CONFIG_TRANSPARENT_HUGEPAGE) && !USE_SPLIT_PMD_PTLOCKS
	pgtable_t pmd_huge_pte; /* protected by page_table_lock */
#endif
#ifdef CONFIG_CPUMASK_OFFSTACK
	struct cpumask cpumask_allocation;
#endif
#ifdef CONFIG_NUMA_BALANCING
	/*
	 * numa_next_scan is the next time that the PTEs will be marked
	 * pte_numa. NUMA hinting faults will gather statistics and migrate
	 * pages to new nodes if necessary.
	 */
	unsigned long numa_next_scan;

	/* Restart point for scanning and setting pte_numa */
	unsigned long numa_scan_offset;

	/* numa_scan_seq prevents two threads setting pte_numa */
	int numa_scan_seq;
#endif
#if defined(CONFIG_NUMA_BALANCING) || defined(CONFIG_COMPACTION)
	/*
	 * An operation with batched TLB flushing is going on. Anything that
	 * can move process memory needs to flush the TLB when moving a
	 * PROT_NONE or PROT_NUMA mapped page.
	 */
	bool tlb_flush_pending;
#endif
	struct uprobes_state uprobes_state;
#ifdef CONFIG_X86_INTEL_MPX
	/* address of the bounds directory */
	void __user *bd_addr;
#endif
};
```

## 参考资料

[蜗窝科技-逆向映射演进](http://www.wowotech.net/memory_management/reverse_mapping.html)

[tolimit-逆向页反向映射](https://www.cnblogs.com/tolimit/p/5398552.html)
