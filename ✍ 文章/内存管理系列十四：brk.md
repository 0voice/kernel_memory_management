## 前沿

### 往篇回顾

> 在前3篇中，首先2篇介绍了内核vmalloc初始化，申请内存以及释放；后一篇介绍了VMA相关的操作；

- 每个进程都有自己的VMA系统，对应管理该进程的内存分布情况，由结构体mm_struct管理;
- 为了解决缓解扫描所有VMA带来的性能问题，在内核3.16版本以后对vma模块进行了优化，新增缓存机制;
- VMA是进程级别的，vma_cache是线程级别的，为解决两者间同步，引入了vmacache_seqnum；
- 调用find_vma函数首先会查找当前线程的vma cache，并校验`current->vmacache_seqnum`值是否与`mm_strcut->vmacache_seqnum`;不一致则cache无效；
- 如果在vma cache中找不到，则会遍历红黑树寻找，找到后更新vma cache；
- 而insert_vm_struct则通过find_vma_links()找出新增vma要插入的位置；然后通过vma_link()执行插入动作；

### brk功能简述

- 从Linux来看，用户态进程分配内存有两种方式，分别由两个系统调用完成: brk和mmap，两种方式分配的都是虚拟内存，访问发生缺页中断才会分配物理内存；
  1. brk分配是将堆内存最高地址指针往高地址推；
  2. mmap是在进程虚拟地址空间中找一块空闲的虚拟内存；

### 本篇主要内容

> 分析系统调用brk、mmap过程

## 代码分析

### brk

```c
/*
 * brk系统调用
 * 在该函数中，首先通过传入参数brk和mm->brk对比，判断是需要申请虚拟内存还是释放
 * brk < mm->brk: 通过do_munmap()释放内存
 * 否则先通过find_vma_intersection()检查扩大以后的堆是否与已经存在的某个虚拟内存重合
 * 		重合则直接退出
 * 		否则，调用do_brk()扩大堆
 * 如果进程内存有VM_LOCKED，则需要立即分配物理空间并完成映射
 */
SYSCALL_DEFINE1(brk, unsigned long, brk)
{
	unsigned long retval;
	unsigned long newbrk, oldbrk;
	struct mm_struct *mm = current->mm;
	unsigned long min_brk;
	bool populate;

	down_write(&mm->mmap_sem);

#ifdef CONFIG_COMPAT_BRK
	/*
	 * 确定堆空间的起始地址min_brk
	 */
	if (current->brk_randomized)
		min_brk = mm->start_brk;
	else
		min_brk = mm->end_data;
#else
	min_brk = mm->start_brk;
#endif
	if (brk < min_brk)
		goto out;

	/*
	 * Check against rlimit here. If this check is done later after the test
	 * of oldbrk with newbrk then it can escape the test and let the data
	 * segment grow beyond its set limit the in case where the limit is
	 * not page aligned -Ram Gupta
	 * 如果RLIMIT_DATA不是RLIM_INFINITY，需要保证数据段加上brk区域不超过RLIMIT_DATA
	 */
	if (check_data_rlimit(rlimit(RLIMIT_DATA), brk, mm->start_brk,
			      mm->end_data, mm->start_data))
		goto out;

	newbrk = PAGE_ALIGN(brk);
	oldbrk = PAGE_ALIGN(mm->brk);
	if (oldbrk == newbrk)
		goto set_brk;

	/* 堆空间地址向上生长，如果brk<=mm->brk，则是释放内存操作. */
	if (brk <= mm->brk) {
	/* 返回0表示释放内存成功.
	 * do_munmap: 释放从newbrk开始，len=oldbrk-newbrk的虚拟内存；相当于释放newbrk->oldbrk
	*/
		if (!do_munmap(mm, newbrk, oldbrk-newbrk))
			goto set_brk;
		goto out;
	}

	/* Check against existing mmap mappings. */
	/* 查找一块VMA包含start_addr的VMA，使得这个vma完全包含(oldbrk, newbrk+PAGE_SIZE)，如果找到了则完成 */
	if (find_vma_intersection(mm, oldbrk, newbrk+PAGE_SIZE))
		goto out;

	/* 申请虚拟地址空间. 将addr位置向后继续申请len字节长度, 扩展堆内存的长度. */
	if (do_brk(oldbrk, newbrk-oldbrk) != oldbrk)
		goto out;

set_brk:
	mm->brk = brk;/* 更新brk. */
	/* 当指定VM_LOCK标志位时，表示需要马上为这块进程虚拟地址空间分配物理页面并建立映射关系 */
	populate = newbrk > oldbrk && (mm->def_flags & VM_LOCKED) != 0;
	up_write(&mm->mmap_sem);
	if (populate)
	/* mm_populate调用__mm_populate来分配页面，同时ignore_erros */
		mm_populate(oldbrk, newbrk - oldbrk);
	return brk;

out:
	retval = mm->brk;
	up_write(&mm->mmap_sem);
	return retval;
}
```

### do_munmap

```c
/*
 * 收缩堆内存
 * 通过find_vma找到start, end两个地址所在的线性地址区间,判断要释放的区间是否与已存在的vma area有重叠
 * 有重叠则需要通过__split_vma分割vma area，保留不涉及的地址；
 * 分割完成后：
 * 1. 通过detach_vmas_to_be_unmapped()将要释放的vma area从红黑树，内存双向链表中删除，vmacache_invalidate使vmacache无效
 * 2. 通过unmap_region()释放vma area的页框
 * 3. 通过remove_vma_list()刷新管理结构体统计数据
 */
int do_munmap(struct mm_struct *mm, unsigned long start, size_t len)
{
	unsigned long end;
	struct vm_area_struct *vma, *prev, *last;

	if ((start & ~PAGE_MASK) || start > TASK_SIZE || len > TASK_SIZE-start)
		return -EINVAL;

	len = PAGE_ALIGN(len);
	if (len == 0)
		return -EINVAL;

	/* 找到其实地址落在哪个VMA中 */
	vma = find_vma(mm, start);
	if (!vma)
		return 0;/* 如果没有这样的线性区，也没有与线性地址区间重叠的线性区，该线性地址区间上没有线性区 */
	prev = vma->vm_prev;
	/* we have  start < vma->vm_end  */

	/* if it doesn't overlap, we have nothing.. */
	end = start + len;
	if (vma->vm_start >= end)
		return 0;

	/*
	 * If we need to split any vma, do it now to save pain later.
	 *
	 * Note: mremap's move_vma VM_ACCOUNT handling assumes a partially
	 * unmapped vm_area_struct will remain in use: so lower split_vma
	 * places tmp vma above, and higher split_vma places tmp vma below.
	 */
	if (start > vma->vm_start) {
		int error;

		/*
		 * 要拆分vma，势必多吃一个VMA
		 */
		if (end < vma->vm_end && mm->map_count >= sysctl_max_map_count)
			return -ENOMEM;
		/* 由于start > vma->vm_start ，所以要释放的空间和vm_start 之间有一段gap。这里就是分离这段gap */
		error = __split_vma(mm, vma, start, 0);
		if (error)
			return error;
		prev = vma;
	}

	/* 找到要释放空间结束地址的vm_area_struct */
	last = find_vma(mm, end);
	
	if (last && end > last->vm_start) {
	/* 说明要释放的空间和last的vm_end 之间有gap，分割后仅保留(last, vm_end)之间的VM */
		int error = __split_vma(mm, last, end, 1);
		if (error)
			return error;
	}
	 /* 更新vma的值，使它指向线性地址区间的第一个线性区。
	  * 如果prev为NULL，即没有上述线性区，就从mm->mmap获得第一个线性区的地址：
	  */
	vma = prev ? prev->vm_next : mm->mmap;

	/*
	 * unlock any mlock()ed ranges before detaching vmas
	 */
	 /* 如果这段要释放的空间是lock的，则要unlock掉 */
	if (mm->locked_vm) {
		struct vm_area_struct *tmp = vma;
		while (tmp && tmp->vm_start < end) {
			if (tmp->vm_flags & VM_LOCKED) {
				mm->locked_vm -= vma_pages(tmp);
				munlock_vma_pages_all(tmp);
			}
			tmp = tmp->vm_next;
		}
	}

	/*
	 * Remove the vma's, and unmap the actual pages
	 */
	detach_vmas_to_be_unmapped(mm, vma, prev, end);
	/* 清除与线性地址区间对应的页表项并释放相应的页框（稍后讨论）：*/
	unmap_region(mm, vma, prev, start, end);

	arch_unmap(mm, vma, start, end);

	/* 更新一些统计信息，完成vma释放 */
	remove_vma_list(mm, vma);

	return 0;
}
```

### __split_vma

```c
/*
 * 把线性地址区间交叉的线性区划分为两个较小的区，一个在线性地址区间外部，一个在区间内部
 * mm: 内存描述符指针；vma: 线性区描述符指针；addr：交叉点；new_blow: 在区间起始处还是结束处标志
 */
static int __split_vma(struct mm_struct *mm, struct vm_area_struct *vma,
	      unsigned long addr, int new_below)
{
	struct vm_area_struct *new;
	int err = -ENOMEM;

	if (is_vm_hugetlb_page(vma) && (addr &
					~(huge_page_mask(hstate_vma(vma)))))
		return -EINVAL;
	/* vm_area_cachep: SLAB cache for vm_area_struct structures; 获得一个新的线性区描述符vm_area_struct */
	new = kmem_cache_alloc(vm_area_cachep, GFP_KERNEL);
	if (!new)
		goto out_err;

	/* 用参数vma描述符的字段初始化新描述符字段 */
	*new = *vma;

	INIT_LIST_HEAD(&new->anon_vma_chain);
	/*
	 * 1：结束地址在vma线性区内部，必须把新线性区放在vma线性区的前面；
	 * 2：
	 */
	if (new_below)
		new->vm_end = addr;
	else {
		new->vm_start = addr;
		new->vm_pgoff += ((addr - vma->vm_start) >> PAGE_SHIFT);
	}
	/* 复制mempolicy */
	err = vma_dup_policy(vma, new);
	if (err)
		goto out_free_vma;

	err = anon_vma_clone(new, vma);
	if (err)
		goto out_free_mpol;

	if (new->vm_file)
		get_file(new->vm_file);
	/* 如果定义了新线性区的open方法，函数就执行它? */
	if (new->vm_ops && new->vm_ops->open)
		new->vm_ops->open(new);
	/* 根据new_below的值调用vma_adjust函数把新线性区描述符链接到线性区链表mm->mmap和rb树mm->mm_rb。
	 * 此外，vma_adjust函数还要根据线性区vma的最新大小对rb树进行调整。 */
	if (new_below)
		err = vma_adjust(vma, addr, vma->vm_end, vma->vm_pgoff +
			((addr - new->vm_start) >> PAGE_SHIFT), new);
	else
		err = vma_adjust(vma, vma->vm_start, addr, vma->vm_pgoff, new);

	/* Success. */
	if (!err)
		return 0;

	/* Clean everything up if vma_adjust failed. */
	if (new->vm_ops && new->vm_ops->close)
		new->vm_ops->close(new);
	if (new->vm_file)
		fput(new->vm_file);
	unlink_anon_vmas(new);
 out_free_mpol:
	mpol_put(vma_policy(new));
 out_free_vma:
	kmem_cache_free(vm_area_cachep, new);
 out_err:
	return err;
}
```

### unmap_region

```c
/*
 * 遍历线性区链表并释放它们的页框
 * vma: 指向第一个被删除线性区描述符的指针vma；start,end: 界定被删除线性地址区间的范围
 */
static void unmap_region(struct mm_struct *mm,
		struct vm_area_struct *vma, struct vm_area_struct *prev,
		unsigned long start, unsigned long end)
{
	struct vm_area_struct *next = prev ? prev->vm_next : mm->mmap;
	struct mmu_gather tlb;

	lru_add_drain();
	/*
	 * 初始化每CPU变量mmu_gathers；该值依赖于CPU体系结构，通常存放成功更新进程页表项所需要的所有信息
	 * vma: 指向第一个被删除线性区描述符的指针vma；start,end: 界定被删除线性地址区间的范围
	 */
	tlb_gather_mmu(&tlb, mm, start, end);
	update_hiwater_rss(mm);
	/* 扫描vma所有页表项，释放相应的页 */
	unmap_vmas(&tlb, vma, start, end);
	/* 回收在上一步已经清空的进程页表 */
	free_pgtables(&tlb, vma, prev ? prev->vm_end : FIRST_USER_ADDRESS,
				 next ? next->vm_start : USER_PGTABLES_CEILING);
	/* 
	 * a.调用flush_tlb_mm()刷新TLB
	 * b.释放页框回伙伴系统 */
	tlb_finish_mmu(&tlb, start, end);
}
```

### do_brk

```c
/*
 *  this is really a simplified "do_mmap".  it only handles anonymous maps.
 *  该函数只处理匿名映射，与文件无关；在函数完成后，进程空间中有一个匿名vm area能映射到[addr, addr len)
 *  do_brk首先判断虚拟地址空间是否足够，然后查找VMA插入点，并判断是否能够进行VMA合并。如果找不到VMA插入点，则新建一个VMA，并更新到mm->mmap中
 */
static unsigned long do_brk(unsigned long addr, unsigned long len)
{
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *vma, *prev;
	unsigned long flags;
	struct rb_node **rb_link, *rb_parent;
	pgoff_t pgoff = addr >> PAGE_SHIFT;
	int error;

	len = PAGE_ALIGN(len);
	if (!len)
		return addr;

	flags = VM_DATA_DEFAULT_FLAGS | VM_ACCOUNT | mm->def_flags;
	/* 首先判断虚拟地址空间是否还有足够剩余，然后在当前进程的用户空间获得一个未映射区间的起始地址 */
	error = get_unmapped_area(NULL, addr, len, 0, MAP_FIXED);
	/* 获得的起始地址肯定是按页对齐的，否则异常 */
	if (error & ~PAGE_MASK)
		return error;
	/* 判断当前的物理内存是否处于锁定状态，如果是则保持和原来状态的一致 */
	error = mlock_future_check(mm, mm->def_flags, len);
	if (error)
		return error;

	/*
	 * mm->mmap_sem is required to protect against another thread
	 * changing the mappings in case we sleep.
	 */
	verify_mm_writelocked(mm);

	/*
	 * 查找是不是已经存在某个vma覆盖了地址addr，如果是则该存在的vma可能只是覆盖了部分[addr, addr+len),也可能覆盖了全部
	 * 调用do_munmap把覆盖的部分清除
	 * 已有的vma可能被切分，与[addr, addr+len)区域有重叠关系的部分会被清除，其余部分保留
	 */
	while (find_vma_links(mm, addr, addr + len, &prev, &rb_link,
			      &rb_parent)) {
		if (do_munmap(mm, addr, len))
			return -ENOMEM;
	}

	/* Check against address space limits *after* clearing old maps... */
	if (!may_expand_vm(mm, len >> PAGE_SHIFT))
		return -ENOMEM;

	if (mm->map_count > sysctl_max_map_count)
		return -ENOMEM;

	if (security_vm_enough_memory_mm(mm, len >> PAGE_SHIFT))
		return -ENOMEM;

	/* 尝试将[addr, addr+len)与前后的vma area合并；该函数在上一篇中有详细分析 */
	vma = vma_merge(mm, prev, addr, addr + len, flags,
					NULL, NULL, pgoff, NULL);
	/* 如果vma合并成功，则只需要维护原有的vma_area_struct数据即可 */
	if (vma)
		goto out;

	/*
	 * 如果无法合并，则新建一个vma管理结构体
	 */
	vma = kmem_cache_zalloc(vm_area_cachep, GFP_KERNEL);
	if (!vma) {
		vm_unacct_memory(len >> PAGE_SHIFT);
		return -ENOMEM;
	}
	/* 初始化管理结构体，并将其连接到红黑树，双向链表中 */
	INIT_LIST_HEAD(&vma->anon_vma_chain);
	vma->vm_mm = mm;
	vma->vm_start = addr;
	vma->vm_end = addr + len;
	vma->vm_pgoff = pgoff;
	vma->vm_flags = flags;
	vma->vm_page_prot = vm_get_page_prot(flags);
	vma_link(mm, vma, prev, rb_link, rb_parent);
out:
	perf_event_mmap(vma);
	mm->total_vm += len >> PAGE_SHIFT;
	if (flags & VM_LOCKED)
		mm->locked_vm += (len >> PAGE_SHIFT);
	vma->vm_flags |= VM_SOFTDIRTY;
	return addr;
}
```

### get_unmapped_area

```c
unsigned long
get_unmapped_area(struct file *file, unsigned long addr, unsigned long len,
		unsigned long pgoff, unsigned long flags)
{
	unsigned long (*get_area)(struct file *, unsigned long,
				  unsigned long, unsigned long, unsigned long);
	/* 针对特定平台的检查，目前arm64中arch_mmap_check 是一个空函数 */
	unsigned long error = arch_mmap_check(addr, len, flags);
	if (error)
		return error;

	/* 申请虚拟空间的地址不能超过最大值。这里可以知道虚拟空间size 的最大值就是TASK_SIZE */
	if (len > TASK_SIZE)
		return -ENOMEM;
	/* 指向当前进程的unmap 空间的分配函数 */
	get_area = current->mm->get_unmapped_area;
	/* file 不为空的话，则unmap 空间的分配函数执行file中指定的函数 */
	if (file && file->f_op->get_unmapped_area)
		get_area = file->f_op->get_unmapped_area;
	/* 使用前面已经指定的分配函数来在未映射的虚拟空间中映射的空间中申请 */
	addr = get_area(file, addr, len, pgoff, flags);
	if (IS_ERR_VALUE(addr))
		return addr;

	if (addr > TASK_SIZE - len)
		return -ENOMEM;
	if (addr & ~PAGE_MASK)
		return -EINVAL;
	/* 目前arm64 空函数 */
	addr = arch_rebalance_pgtables(addr, len);
	error = security_mmap_addr(addr);/* 安全检查 */
	return error ? error : addr;
}
```

## 参考资料

[do_munmap释放线性地址区间](https://blog.csdn.net/yunsongice/article/details/5637554)

[do_brk分析](https://blog.51cto.com/2ndmoon/1283728)

[vma与虚拟内存区域有关的操作](http://edsionte.com/techblog/archives/3564)

[vma合并虚拟内存区域](http://edsionte.com/techblog/archives/3586)

[malloc()之后，内核发生了什么?](https://edsionte.com/techblog/archives/4174)

[Arnold Lu malloc实例](https://www.cnblogs.com/arnoldlu/p/8329283.html)
