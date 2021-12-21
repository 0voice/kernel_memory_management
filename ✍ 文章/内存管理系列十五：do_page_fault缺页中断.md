## 前沿

### 往篇回顾

> 在上一篇中，主要介绍了系统调用brk的相关处理流程；brk根据传入参数brk和mm->brk的比较可以分为两类操作

- 收缩堆内存空间(do_munmap)
  1. 通过find_vma找到start, end两个地址所在的线性地址区间,判断要释放的区间是否与已存在的vma area有重叠;
  2. 有重叠则需要通过__split_vma分割vma area，保留不涉及的地址；分割完成后：
  3. 通过detach_vmas_to_be_unmapped()将要释放的vma area从红黑树，内存双向链表中删除，vmacache_invalidate使vmacache无效
  4. 通过unmap_region()释放vma area的页框
  5. 通过remove_vma_list()刷新管理结构体统计数据
- 扩大堆内存空间(do_brk)
  1. 通过get_unmapped_area()在当前进程的地址空间中查找一个符合len大小的线性区间，且该线性区间必须在addr地址之后
  2. 通过find_vma_links遍历每个vma，确定上一步找到的新区间之前的线性区对象的位置；如果addr位于某个现存vma，则调用do_munmap()删除；删除成功后继续查找；
  3. 对找到的空闲线性区间通过vma_merge尝试与邻近的线性区间进行合并，成功则更新原有vma_area_struct数据并返回即可；
  4. 通过kmem_cache_zalloc()在特定的slab高速缓存vm_area_cachep中为这个线性区分配vm_area_struct结构的描述符，并初始化结构体，更新红黑树，双向链表；
  5. 如果当前vma设置了VM_LOCKED字段，那么通过mlock_vma_pages_range()立即为这个线性区分配物理页框，否则，do_brk()结束；

### linux内存分配函数异同点

| 用户态/内核态 |                API                |             物理地址连续 |                     大小限制                      |                    单位 |                             场景                             |
| :------------ | :-------------------------------: | -----------------------: | :-----------------------------------------------: | ----------------------: | :----------------------------------------------------------: |
| 用户态        |    malloc/calloc/realloc/free     |                   不保证 |                      堆内存                       |                    byte |             calloc初始化为0；realloc改变内存大小             |
| 用户态        |              alloca               |                   不保证 |                      栈内存                       |                    byte |                         向栈申请空间                         |
| 用户态        |            mmap/munmap            |                          |                                                   |                    byte |             将文件利用虚拟内存技术映射到内存中去             |
| 用户态        |             brk,sbrk              |                   不保证 |                      堆内存                       |                    byte |                     虚拟内存到内存的映射                     |
| 内核态        |           vmalloc/vfree           | 虚拟地址连续，物理不确定 | vmalloc区域vmalloc_start~vmalloc_end，比kmalloc慢 |                    byte | 可能睡眠，不能从中断上下文中调用，或其它不允许阻塞情况下调用 |
| 内核态        |  kmalloc/kcalloc/krealloc/kfree   |                 物理连续 |                      64B-4MB                      |       2^order, normal区 | 最大/小值由KMALLOC_MIN_SIZE/KMALLOC_SHIFT_MAX，对应64B/4MB，从/proc/slabinfo中的kmalloc-xxxx中分配，建立在kmem_cache_create基础之上 |
| 内核态        |         kmem_cache_create         |                 物理连续 |                      64B-4MB                      | 字节大小需对齐,normal区 | 便于固定大小数据的频繁分配和释放，分配时从缓存池中获取地址，释放时也不一定真正释放内存。通过slab进行管理 |
| 内核态        | __get_free_page/__get_free_pages  |                 物理连续 |                     0~1024页                      |                normal区 |   __get_free_pages基于alloc_pages，但是限定不能使用HIGHMEM   |
| 内核态        | alloc_page/alloc_pages/free_pages |                 物理连续 |                        4MB                        |                normal区 | CONFIG_FORCE_MAX_ZONEORDER定义了最大页面数2^11，一次能分配到的最大页面数是1024 |

### 本篇主要内容

> 本文主要分析缺页中断流程，关键函数do_page_fault()

## 代码分析

### do_page_fault

```c
/*
 * 入参：regs: 指向保存在堆栈中的寄存器；error_code: 异常的错误码
 */
static int __kprobes do_page_fault(unsigned long addr, unsigned int esr,
				   struct pt_regs *regs)
{
	struct task_struct *tsk;
	struct mm_struct *mm;
	int fault, sig, code;
	unsigned long vm_flags = VM_READ | VM_WRITE | VM_EXEC;
	unsigned int mm_flags = FAULT_FLAG_ALLOW_RETRY | FAULT_FLAG_KILLABLE;

	tsk = current;
	mm  = tsk->mm;

	/* Enable interrupts if they were enabled in the parent context. */
	if (interrupts_enabled(regs))
		local_irq_enable();

	/*
	 * in_atomic判断当前状态是否处于中断上下文或者禁止抢占，如果是跳转到no_context；
	 * 如果当前进程没有mm，说明是一个内核线程，跳转到no_context；
	 */
	if (in_atomic() || !mm)
		goto no_context;
	/* cpsr为当前程序状态寄存器，如果M[0:4]均为0，则处于用户模式，详细见附录 */
	if (user_mode(regs))
		mm_flags |= FAULT_FLAG_USER;

	if (esr & ESR_LNX_EXEC) {
		vm_flags = VM_EXEC;
	} else if ((esr & ESR_ELx_WNR) && !(esr & ESR_ELx_CM)) {
		vm_flags = VM_WRITE;
		mm_flags |= FAULT_FLAG_WRITE;
	}

	/*
	 * As per x86, we may deadlock here. However, since the kernel only
	 * validly references user space from well defined areas of the code,
	 * we can bug out early if this is from code which shouldn't.
	 */
	if (!down_read_trylock(&mm->mmap_sem)) {
	/* 获取锁失败 */
	/* 如果在内核态且在exception table查询不到该地址，则跳转到no_context */
		if (!user_mode(regs) && !search_exception_tables(regs->pc))
			goto no_context;
retry:
		down_read(&mm->mmap_sem);/* 用户态则睡眠等待锁持有者释放锁 */
	} else {
		/*
		 * The above down_read_trylock() might have succeeded in which
		 * case, we'll have missed the might_sleep() from down_read().
		 */
		might_sleep();
#ifdef CONFIG_DEBUG_VM
		if (!user_mode(regs) && !search_exception_tables(regs->pc))
			goto no_context;
#endif
	}

	fault = __do_page_fault(mm, addr, mm_flags, vm_flags, tsk);

	/*
	 * If we need to retry but a fatal signal is pending, handle the
	 * signal first. We do not need to release the mmap_sem because it
	 * would already be released in __lock_page_or_retry in mm/filemap.c.
	 */
	if ((fault & VM_FAULT_RETRY) && fatal_signal_pending(current))
		return 0;

	/*
	 * Major/minor page fault accounting is only done on the initial
	 * attempt. If we go through a retry, it is extremely likely that the
	 * page will be found in page cache at that point.
	 */

	perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS, 1, regs, addr);
	if (mm_flags & FAULT_FLAG_ALLOW_RETRY) {
		if (fault & VM_FAULT_MAJOR) {
			tsk->maj_flt++;
			perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS_MAJ, 1, regs,
				      addr);
		} else {
			tsk->min_flt++;
			perf_sw_event(PERF_COUNT_SW_PAGE_FAULTS_MIN, 1, regs,
				      addr);
		}
		if (fault & VM_FAULT_RETRY) {
			/*
			 * Clear FAULT_FLAG_ALLOW_RETRY to avoid any risk of
			 * starvation.
			 */
			mm_flags &= ~FAULT_FLAG_ALLOW_RETRY;
			goto retry;
		}
	}

	up_read(&mm->mmap_sem);

	/*
	 * 如果没有返回(VM_FAULT_ERROR | VM_FAULT_BADMAP | VM_FAULT_BADACCESS)，则缺页中断处理完成
	 */
	if (likely(!(fault & (VM_FAULT_ERROR | VM_FAULT_BADMAP |
			      VM_FAULT_BADACCESS))))
		return 0;

	/*
	 * If we are in kernel mode at this point, we have no context to
	 * handle this fault with.
	 */
	if (!user_mode(regs))
		goto no_context;

	if (fault & VM_FAULT_OOM) {
		/*
		 * 如果错误类型为OOM，则当前系统无足够内存，触发OOM机制
		 */
		pagefault_out_of_memory();
		return 0;
	}

	if (fault & VM_FAULT_SIGBUS) {
		/*
		 * We had some memory, but were unable to successfully fix up
		 * this page fault.
		 */
		sig = SIGBUS;
		code = BUS_ADRERR;
	} else {
		/*
		 * Something tried to access memory that isn't in our memory
		 * map.
		 */
		sig = SIGSEGV;
		code = fault == VM_FAULT_BADACCESS ?
			SEGV_ACCERR : SEGV_MAPERR;
	}
	/* 给用户进程发送信号，错误内核无法妥善处理 */
	__do_user_fault(tsk, addr, esr, sig, code, regs);
	return 0;

no_context:
	/* 错误发生在内核态；发送Oops错误 */
	__do_kernel_fault(mm, addr, esr, regs);
	return 0;
}
```

### __do_page_fault

```c
/*
 * 1. 通过find_vma查找虚拟地址addr后的最近vma，如果没找到，则方位地址错误，因为它不在所分配的任何一个vma线性区；
 * 2. 如果找到vma，但addr并未落入这个区间，则可能是栈中vma
 * 3. 经检查一切正常后，调用handle_mm_fault分配物理页框
 * VM_FAULT_BADACCESS：严重错误，内核会直接kill该进程
 */
static int __do_page_fault(struct mm_struct *mm, unsigned long addr,
			   unsigned int mm_flags, unsigned long vm_flags,
			   struct task_struct *tsk)
{
	struct vm_area_struct *vma;
	int fault;
	 /* 搜索出现异常的地址前向最近的的vma */
	vma = find_vma(mm, addr);
	fault = VM_FAULT_BADMAP;
	/* 如果vma为NULL，说明addr之后没有vma，所以这个addr是个错误地址 */
	if (unlikely(!vma))
		goto out;
	 /*如果addr之后有vma，但不包含addr，不能断定addr是错误地址，还需检查*/
	if (unlikely(vma->vm_start > addr))
		goto check_stack;

	/*
	 * Ok, we have a good vm_area for this memory access, so we can handle
	 * it.
	 */
good_area:
	/*
	 * Check that the permissions on the VMA allow for the fault which
	 * occurred. If we encountered a write or exec fault, we must have
	 * appropriate permissions, otherwise we allow any permission.
	 */
	if (!(vma->vm_flags & vm_flags)) {
		fault = VM_FAULT_BADACCESS;
		goto out;
	}
	 //分配新页框
	return handle_mm_fault(mm, vma, addr & PAGE_MASK, mm_flags);

check_stack:
	/* 
   	 * addr后面的vma的vm_flags含有VM_GROWSDOWN标志，说明这个vma属于栈的vma
	 * 即addr在栈中，有可能是栈空间不够时再进栈导致的访问错误
	 * 同时检查栈是否还能扩展，如果不能扩展则确认确实是栈溢出导致，即addr确实是栈中地址，不是非法地址
	 * 应进入缺页中断请求
	 */
	if (vma->vm_flags & VM_GROWSDOWN && !expand_stack(vma, addr))
		goto good_area;
out:
	return fault;
}
```

### handle_mm_fault

```c
/*
 * 核心功能封装在__handle_mm_fault中：为引发缺页的进程分配一个物理页框；
 */
int handle_mm_fault(struct mm_struct *mm, struct vm_area_struct *vma,
		    unsigned long address, unsigned int flags)
{
	int ret;

	__set_current_state(TASK_RUNNING);

	count_vm_event(PGFAULT);
	mem_cgroup_count_vm_event(mm, PGFAULT);

	/* do counter updates before entering really critical section. */
	check_sync_rss_stat(current);

	/*
	 * Enable the memcg OOM handling for faults triggered in user
	 * space.  Kernel faults are handled more gracefully.
	 */
	if (flags & FAULT_FLAG_USER)
		mem_cgroup_oom_enable();

	ret = __handle_mm_fault(mm, vma, address, flags);

	if (flags & FAULT_FLAG_USER) {
		mem_cgroup_oom_disable();
                /*
                 * The task may have entered a memcg OOM situation but
                 * if the allocation error was handled gracefully (no
                 * VM_FAULT_OOM), there is no need to kill anything.
                 * Just clean up the OOM state peacefully.
                 */
                if (task_in_memcg_oom(current) && !(ret & VM_FAULT_OOM))
                        mem_cgroup_oom_synchronize(false);
	}

	return ret;
}
```

### brk

```c
/*
 * By the time we get here, we already hold the mm semaphore
 *
 * The mmap_sem may have been released depending on flags and our
 * return value.  See filemap_fault() and __lock_page_or_retry().
 */
static int __handle_mm_fault(struct mm_struct *mm, struct vm_area_struct *vma,
			     unsigned long address, unsigned int flags)
{
	pgd_t *pgd;
	pud_t *pud;
	pmd_t *pmd;
	pte_t *pte;
	/* 如果开启了巨页，则使用hugetlb_fault分配内存 */
	if (unlikely(is_vm_hugetlb_page(vma)))
		return hugetlb_fault(mm, vma, address, flags);
	/* 获取addr对应在当前进程页表的PGD页面目录项 */
	pgd = pgd_offset(mm, address);
	/* 获取对应的PUD表项，如果PUD表项为空，则返回VM_FAULT_OOM错误 */
	pud = pud_alloc(mm, pgd, address);
	if (!pud)
		return VM_FAULT_OOM;
	pmd = pmd_alloc(mm, pud, address);
	if (!pmd)
		return VM_FAULT_OOM;
	/* 以下这段和huge page有关内容并未能理解?
	 * huge page最小页为2M，相当于PTE=PMD
	 */
	if (pmd_none(*pmd) && transparent_hugepage_enabled(vma)) {
		int ret = VM_FAULT_FALLBACK;
		if (!vma->vm_ops)
			ret = do_huge_pmd_anonymous_page(mm, vma, address,
					pmd, flags);
		if (!(ret & VM_FAULT_FALLBACK))
			return ret;
	} else {
		pmd_t orig_pmd = *pmd;
		int ret;

		barrier();
		if (pmd_trans_huge(orig_pmd)) {
			unsigned int dirty = flags & FAULT_FLAG_WRITE;

			/*
			 * If the pmd is splitting, return and retry the
			 * the fault.  Alternative: wait until the split
			 * is done, and goto retry.
			 */
			if (pmd_trans_splitting(orig_pmd))
				return 0;

			if (pmd_protnone(orig_pmd))
				return do_huge_pmd_numa_page(mm, vma, address,
							     orig_pmd, pmd);

			if (dirty && !pmd_write(orig_pmd)) {
				ret = do_huge_pmd_wp_page(mm, vma, address, pmd,
							  orig_pmd);
				if (!(ret & VM_FAULT_FALLBACK))
					return ret;
			} else {
				huge_pmd_set_accessed(mm, vma, address, pmd,
						      orig_pmd, dirty);
				return 0;
			}
		}
	}

	/*
	 * Use __pte_alloc instead of pte_alloc_map, because we can't
	 * run pte_offset_map on the pmd, if an huge pmd could
	 * materialize from under us from a different thread.
	 */
	if (unlikely(pmd_none(*pmd)) &&
	    unlikely(__pte_alloc(mm, vma, pmd, address)))
		return VM_FAULT_OOM;
	/* if an huge pmd materialized from under us just retry later */
	if (unlikely(pmd_trans_huge(*pmd)))
		return 0;
	/*
	 * 获取对应的pte表项，然后跳转到handle_pte_fault中
	 */
	pte = pte_offset_map(pmd, address);

	return handle_pte_fault(mm, vma, address, pte, pmd, flags);
}
```

### handle_pte_fault

```c
/*
 * 根据页表项pte所描述的物理页框是否在物理内存中，分为两大类，由pte_present(*pte)区分
 * 1. 调页请求(物理页不存在)：分配一个页框，根据pte页表项是否为空分为两种情况：
 * 		a. pte为空：pte中尚未写入物理地址，文件映射缺页中断、匿名映射缺页中断
 * 		b. pte不为空，但(PTE_VALID | PTE_PROT_NONE)未置位：相关物理地址已经被交换到外存
 * 2. 写时复制(物理页存在)：被访问的页存在，但该页是只读，内核需要对该页进行写操作；则此时内核将这个已存在的只读页中数据复制到一个新页框中
 */
static int handle_pte_fault(struct mm_struct *mm,
		     struct vm_area_struct *vma, unsigned long address,
		     pte_t *pte, pmd_t *pmd, unsigned int flags)
{
	pte_t entry;
	spinlock_t *ptl;

	/*
	 * pte表项不同处理器可能会大于word size， 此时READ_ONCE和ACCESS_ONCE不保证访问原子性，因此需要内存屏障以保证正确读取了pte表项内容
	 */
	entry = *pte;
	barrier();
	/* 
	 * PTE_PRESENT没有置位，说明pte实际指向的物理地址不存在，可能是调页请求
	 * 有两种情况：
	 * 1. *pte为空；
	 * 2. *pte不为空，但是(PTE_VALID | PTE_PROT_NONE)标志位未置位
	 */
	if (!pte_present(entry)) {

		if (pte_none(entry)) {
		/* 第一种情况：*pte == 0, 说明尚未写入任何物理地址 */
			if (vma->vm_ops)
			/* 如果该vma定义了操作函数集合，说明是文件映射页面缺页中断，将调用do_fault分配物理页 */
				return do_fault(mm, vma, address, pte, pmd,
						flags, entry);
			/* 匿名映射缺页中断，最终会调用alloc_pages从伙伴系统分配页面 */
			return do_anonymous_page(mm, vma, address, pte, pmd,
					flags);
		}
		/* 
		 * 第二种情况：*pte不为空，但是(PTE_VALID | PTE_PROT_NONE)标志位未置位，即物理页不存在
		 * 此刻该物理页已经由主存换出到了外存，则调用do_swap_page()完成页框分配
		 */
		return do_swap_page(mm, vma, address,
					pte, pmd, flags, entry);
	}
/*=============================以下为物理页面存在场景===============================================*/
	if (pte_protnone(entry))
		return do_numa_page(mm, vma, address, entry, pte, pmd);
	/* 到这应该是写时复制触发的缺页中断；即被访问的页面不可写，有两种情况：
	 * 1. 之前给vma映射的是零页(zero-pfn)
	 * 2. 访问fork得到的进程空间(子进程、父进程共享父进程的内存，均为只读页)
	 */
	ptl = pte_lockptr(mm, pmd);
	spin_lock(ptl);
	if (unlikely(!pte_same(*pte, entry)))
		goto unlock;
	/* 写操作时发生的缺页异常 */
	if (flags & FAULT_FLAG_WRITE) {
		if (!pte_write(entry))/* 第二种情况：pte页表项标识不可写，引发COW */
			return do_wp_page(mm, vma, address,
					pte, pmd, ptl, entry);/* 进行写时复制操作 */
		/* 第一种情况: vma映射的是zero-pfn，设置pte_dirty位，标识页内容已被修改 */
		entry = pte_mkdirty(entry);
	}
	entry = pte_mkyoung(entry);/* 设置访问位，通常是_PAGE_ACCESSD */
	if (ptep_set_access_flags(vma, address, pte, entry, flags & FAULT_FLAG_WRITE)) {
		/* pte内容发生变化，需要把新的内容写入pte页表项中，并刷新TLB和cache */
		update_mmu_cache(vma, address, pte);
	} else {
		/*
		 * This is needed only for protection faults but the arch code
		 * is not yet telling us if this is a protection fault or not.
		 * This still avoids useless tlb flushes for .text page faults
		 * with threads.
		 */
		if (flags & FAULT_FLAG_WRITE)
			flush_tlb_fix_spurious_fault(vma, address);
	}
unlock:
	pte_unmap_unlock(pte, ptl);
	return 0;
}
```

### do_fault

```c
/*
 * 文件映射缺页中断处理函数
 */
static int do_fault(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pte_t *page_table, pmd_t *pmd,
		unsigned int flags, pte_t orig_pte)
{
	pgoff_t pgoff = (((address & PAGE_MASK)
			- vma->vm_start) >> PAGE_SHIFT) + vma->vm_pgoff;

	pte_unmap(page_table);
	/* The VMA was not fully populated on mmap() or missing VM_DONTEXPAND */
	if (!vma->vm_ops->fault)
		return VM_FAULT_SIGBUS;
	if (!(flags & FAULT_FLAG_WRITE))
		/*  如果为读中断 */
		return do_read_fault(mm, vma, address, pmd, pgoff, flags,
				orig_pte);
	if (!(vma->vm_flags & VM_SHARED))
	/* 没有定义VM_SHARED, 是一个私有映射且发生了写时复制 */
		return do_cow_fault(mm, vma, address, pmd, pgoff, flags,
				orig_pte);
	/* 共享映射中发生了缺页异常，什么场景？ */
	return do_shared_fault(mm, vma, address, pmd, pgoff, flags, orig_pte);
}
```

### do_anonymous_page

```c
/*
 * 匿名页缺页中断，一般为malloc触发
 * 1. 调用anon_vma_prepare()获取一个anon_vma结构，这个结构可能属于此vma，也可能属于此vma能够合并的前后一个vma
 * 2. 通过伙伴系统分配一个页(在32位上，会优先从高端内存分配)
 * 3. 根据vma默认页表项参数vm_page_prot创建一个页表项，这个页表项用于加入到address对应的页表中
 * 4. 调用page_add_new_anon_rmap()给此page添加一个反向映射
 * 5. 将页表项和页表还有此页进行关联，由于页表已经在调用前分配好页了，只需要将页表项与新匿名页进行关联，然后将设置好的页表项写入address在此页表中的偏移地址即可。
 */
static int do_anonymous_page(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pte_t *page_table, pmd_t *pmd,
		unsigned int flags)
{
	struct mem_cgroup *memcg;
	struct page *page;
	spinlock_t *ptl;
	pte_t entry;

	pte_unmap(page_table);

	/* File mapping without ->vm_ops ? */
	if (vma->vm_flags & VM_SHARED)
		return VM_FAULT_SIGBUS;

	/* 
	 * 1. vma向下增长，且addr==vma->vm_start，则要向下扩展一页保护页
	 * 2. vma向上增长，且addr==vma->end，则要向上扩展一页保护页
	 */
	if (check_stack_guard_page(vma, address) < 0)
		return VM_FAULT_SIGSEGV;

	/* 
	 * 如果不是写操作且允许使用Zero page, 则把zero_pfn的页表条目赋给entry
   	 * 因为这里已经是缺页异常的请求调页的处理，又是读操作，所以肯定是本进程第一次访问这个页
	 * 所以这个页里面是什么内容无所谓，分配个默认全零页就好，进一步推迟物理页的分配，这就会让entry带着zero_pfn跳到标号setpte
	 */
	if (!(flags & FAULT_FLAG_WRITE) && !mm_forbids_zeropage(mm)) {
	/* 创建pte页表项，这个pte会指向内核中一个默认的全是0的页框，并且会有vma->vm_page_prot中的标志，最后会加上PAGE_SPECIAL标志 */
		entry = pte_mkspecial(pfn_pte(my_zero_pfn(address),
						vma->vm_page_prot));
		/* 获取addr的pte table */
		page_table = pte_offset_map_lock(mm, pmd, address, &ptl);
		/* 如果页表项不为空，则说明这页曾经被该进程访问过，可能其他核或线程更改了此页表项 */
		if (!pte_none(*page_table))
			goto unlock;
		goto setpte;
	}
	//=================================下面的处理带FAULT_FLAG_WRITE写标志的缺页中断====================================
    /* 为vma准备反向映射条件 
     * find_mergeable_anon_vma 检查此vma能与前后的vma进行合并吗，如果可以，则使用能够合并的那个vma的anon_vma，如果不能够合并，则申请一个空闲的anon_vma
     * 新建一个 anon_vma_chain
     * 将avc->anon_vma指向获得的vma，avc->vma指向vma，并把avc加入到vma的anon_vma_chain中
     */
	if (unlikely(anon_vma_prepare(vma)))
		goto oom;
		/* 调用伙伴系统分配一个可写的匿名页面. */
	page = alloc_zeroed_user_highpage_movable(vma, address);
	if (!page)/* 分配不成功 */
		goto oom;
	/* 调用内存屏障，设置page->flag中更新标志位PG_uptodate */
	__SetPageUptodate(page);
	/* 更新memcg中的计数，如果超过了memcg中的限制值，则会把这个页释放掉，并返回VM_FAULT_OOM */
	if (mem_cgroup_try_charge(page, mm, GFP_KERNEL, &memcg))
		goto oom_free_page;
	/* 根据vma页参数及pages地址创建pte页表项 */
	entry = mk_pte(page, vma->vm_page_prot);
	
	if (vma->vm_flags & VM_WRITE)
	/* 如果vma区是可写的，则给页表项添加可写标志 */
		entry = pte_mkwrite(pte_mkdirty(entry));
	/* 获取addr对应的pte页表项；由于要修改，所以要上锁，只读是不需要上锁的 */
	page_table = pte_offset_map_lock(mm, pmd, address, &ptl);
	if (!pte_none(*page_table))
		goto release;
	/* 增加mm_struct中匿名页(MM_ANONPAGES)的统计计数 */
	inc_mm_counter_fast(mm, MM_ANONPAGES);
    /* 对这个新页进行反向映射 
     * 主要工作是:
     * 设置此页的_mapcount = 0，说明此页正在使用，但是是非共享的(>0是共享)
     * 统计:
     * 设置page->mapping最低位为1
     * page->mapping指向此vma->anon_vma
     * page->index存放此page在vma中的第几页
     */
	page_add_new_anon_rmap(page, vma, address);
	/* 提交memcg中的统计 */
	mem_cgroup_commit_charge(page, memcg, false);
	/* 把匿名页面添加到LRU链表中，在kswap内核模块会使用LRU链表 */
	lru_cache_add_active_or_unevictable(page, vma);
setpte:
	/* 将上面配置好的页表项写入页表 */
	set_pte_at(mm, address, page_table, entry);

	/* 让mmu更新页表项，应该会清除tlb */
	update_mmu_cache(vma, address, page_table);
unlock:
	pte_unmap_unlock(page_table, ptl);
	return 0;
release:/* 以下为异常处理 */
/* 取消此page在memcg中的计数，这里处理会在mem_cgroup_commit_charge()之前 */
	mem_cgroup_cancel_charge(page, memcg);
	/* 将申请的页释放到每CPU页高速缓存中 */
	page_cache_release(page);
	goto unlock;
oom_free_page:
	page_cache_release(page);
oom:
	return VM_FAULT_OOM;
}
```

### do_swap_page

```c
/*
 * 先由pte得到swap entry,再得到page；再有pte以及pte entry添加到硬件pte页表
 * swap cache: 系统中有多少匿名页曾经被swap-out、现在又被swap-in并且swap-in之后页面中的内容一直没发生变化
 * 1. 为了防止页面在swap in/out时，进程有同步问题；即在swap out操作时，进程访问了换出页面
 * 2. 如果页面数据没有完全写入磁盘，page frame是在swap cache
 * 3. 等数据完全写入磁盘后，且没有进程对page frame进行访问，则swap cache会释放page frame，将其交给buddy system
 * 4. swap cache只存在很短暂时间，page out完成后就删除
 * 5. 曾经被swap out现在又被swap in的匿名页会在swap cache；知道页面中内容发生变化，或者原来用过的交换区空间被回收为止
 */
static int do_swap_page(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pte_t *page_table, pmd_t *pmd,
		unsigned int flags, pte_t orig_pte)
{
	spinlock_t *ptl;
	struct page *page, *swapcache;
	struct mem_cgroup *memcg;
	swp_entry_t entry;
	pte_t pte;
	int locked;
	int exclusive = 0;
	int ret = 0;
	
	if (!pte_unmap_same(mm, pmd, page_table, orig_pte))
		goto out;
	/* 根据pte来获取swap的entry, swap entry和pte有一个对应关系 */
	entry = pte_to_swp_entry(orig_pte);
	if (unlikely(non_swap_entry(entry))) {
		if (is_migration_entry(entry)) {
			migration_entry_wait(mm, pmd, address);
		} else if (is_hwpoison_entry(entry)) {
			ret = VM_FAULT_HWPOISON;
		} else {
			print_bad_pte(vma, address, orig_pte, NULL);
			ret = VM_FAULT_SIGBUS;
		}
		goto out;
	}
	delayacct_set_flag(DELAYACCT_PF_SWAPIN);
	/* 在swapcache里面寻找entry对应的page */
	page = lookup_swap_cache(entry);
	if (!page) {
	/* 如果swapcache里面找不到就在swap area里面找 */
		page = swapin_readahead(entry,
					GFP_HIGHUSER_MOVABLE, vma, address);
		if (!page) {
			/*
			 * Back out if somebody else faulted in this pte
			 * while we released the pte lock.
			 */
			page_table = pte_offset_map_lock(mm, pmd, address, &ptl);
			if (likely(pte_same(*page_table, orig_pte)))
				ret = VM_FAULT_OOM;
			delayacct_clear_flag(DELAYACCT_PF_SWAPIN);
			goto unlock;
		}

		/* Had to read the page from swap area: Major fault */
		ret = VM_FAULT_MAJOR;
		count_vm_event(PGMAJFAULT);/* PGMAJFAULT计数加1 */
		mem_cgroup_count_vm_event(mm, PGMAJFAULT);
	} else if (PageHWPoison(page)) {
		/*
		 * hwpoisoned dirty swapcache pages are kept for killing
		 * owner processes (which may be unknown at hwpoison time)
		 */
		ret = VM_FAULT_HWPOISON;
		delayacct_clear_flag(DELAYACCT_PF_SWAPIN);
		swapcache = page;
		goto out_release;
	}

	swapcache = page;
	/* 加锁 */
	locked = lock_page_or_retry(page, mm, flags);

	delayacct_clear_flag(DELAYACCT_PF_SWAPIN);
	if (!locked) {
		ret |= VM_FAULT_RETRY;
		goto out_release;
	}

	/*
	 * Make sure try_to_free_swap or reuse_swap_page or swapoff did not
	 * release the swapcache from under us.  The page pin, and pte_same
	 * test below, are not enough to exclude that.  Even if it is still
	 * swapcache, we need to check that the page's swap has not changed.
	 */
	if (unlikely(!PageSwapCache(page) || page_private(page) != entry.val))
		goto out_page;

	page = ksm_might_need_to_copy(page, vma, address);
	if (unlikely(!page)) {
		ret = VM_FAULT_OOM;
		page = swapcache;
		goto out_page;
	}

	if (mem_cgroup_try_charge(page, mm, GFP_KERNEL, &memcg)) {
		ret = VM_FAULT_OOM;
		goto out_page;
	}

	/*  获取pte entry.  */
	page_table = pte_offset_map_lock(mm, pmd, address, &ptl);
	if (unlikely(!pte_same(*page_table, orig_pte)))
		goto out_nomap;

	if (unlikely(!PageUptodate(page))) {
		ret = VM_FAULT_SIGBUS;
		goto out_nomap;
	}

	/*
	 * The page isn't present yet, go ahead with the fault.
	 *
	 * Be careful about the sequence of operations here.
	 * To get its accounting right, reuse_swap_page() must be called
	 * while the page is counted on swap but not yet in mapcount i.e.
	 * before page_add_anon_rmap() and swap_free(); try_to_free_swap()
	 * must be called after the swap_free(), or it will never succeed.
	 */
	/* anonpage数加1，匿名页从swap空间交换出来，所以加1 */
	inc_mm_counter_fast(mm, MM_ANONPAGES);
	/* swap page个数减1 */
	dec_mm_counter_fast(mm, MM_SWAPENTS);
	/* 由page和VMA属性创建一个新的pte */
	pte = mk_pte(page, vma->vm_page_prot);
	if ((flags & FAULT_FLAG_WRITE) && reuse_swap_page(page)) {
		pte = maybe_mkwrite(pte_mkdirty(pte), vma);
		flags &= ~FAULT_FLAG_WRITE;
		ret |= VM_FAULT_WRITE;
		exclusive = 1;
	}
	flush_icache_page(vma, page);
	if (pte_swp_soft_dirty(orig_pte))
		pte = pte_mksoft_dirty(pte);
	/* 将新生成的PTE entry添加到硬件页表 */
	set_pte_at(mm, address, page_table, pte);
	/* 根据page是否为swapcache，如果是，则只为page创建rmap，如果不是则创建rmap还要将page添加到LRU链表 */
	if (page == swapcache) {
	
		do_page_add_anon_rmap(page, vma, address, exclusive);
		mem_cgroup_commit_charge(page, memcg, true);
	} else { /* ksm created a completely new copy */
		page_add_new_anon_rmap(page, vma, address);
		mem_cgroup_commit_charge(page, memcg, false);
		lru_cache_add_active_or_unevictable(page, vma);
	}

	swap_free(entry);
	if (vm_swap_full() || (vma->vm_flags & VM_LOCKED) || PageMlocked(page))
		try_to_free_swap(page);
	unlock_page(page);
	if (page != swapcache) {
		/*
		 * Hold the lock to avoid the swap entry to be reused
		 * until we take the PT lock for the pte_same() check
		 * (to avoid false positives from pte_same). For
		 * further safety release the lock after the swap_free
		 * so that the swap count won't change under a
		 * parallel locked swapcache.
		 */
		unlock_page(swapcache);
		page_cache_release(swapcache);
	}

	if (flags & FAULT_FLAG_WRITE) {
		ret |= do_wp_page(mm, vma, address, page_table, pmd, ptl, pte);
		if (ret & VM_FAULT_ERROR)
			ret &= VM_FAULT_ERROR;
		goto out;
	}

	/* No need to invalidate - it was non-present before */
	update_mmu_cache(vma, address, page_table);
unlock:
	pte_unmap_unlock(page_table, ptl);
out:
	return ret;
out_nomap:
	mem_cgroup_cancel_charge(page, memcg);
	pte_unmap_unlock(page_table, ptl);
out_page:
	unlock_page(page);
out_release:
	page_cache_release(page);
	if (page != swapcache) {
		unlock_page(swapcache);
		page_cache_release(swapcache);
	}
	return ret;
}
```

## do_wp_page

```c
/*
 * 写时复制：一般出现在fork后子进程修改内存数据、
 *
 * We should not cow pages in a shared writeable mapping.
 * Just mark the pages writable and/or call ops->pfn_mkwrite.
 */
static int do_wp_page(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pte_t *page_table, pmd_t *pmd,
		spinlock_t *ptl, pte_t orig_pte)
	__releases(ptl)
{
	struct page *old_page;

	old_page = vm_normal_page(vma, address, orig_pte);
	if (!old_page) {
		/*
		 * VM_MIXEDMAP !pfn_valid() case, or VM_SOFTDIRTY clear on a
		 * VM_PFNMAP VMA.
		 *
		 * We should not cow pages in a shared writeable mapping.
		 * Just mark the pages writable and/or call ops->pfn_mkwrite.
		 */
		if ((vma->vm_flags & (VM_WRITE|VM_SHARED)) ==
				     (VM_WRITE|VM_SHARED))
			return wp_pfn_shared(mm, vma, address, page_table, ptl,
					     orig_pte, pmd);

		pte_unmap_unlock(page_table, ptl);
		return wp_page_copy(mm, vma, address, page_table, pmd,
				    orig_pte, old_page);
	}

	/*
	 * Take out anonymous pages first, anonymous shared vmas are
	 * not dirty accountable.
	 */
	if (PageAnon(old_page) && !PageKsm(old_page)) {
		if (!trylock_page(old_page)) {
			page_cache_get(old_page);
			pte_unmap_unlock(page_table, ptl);
			lock_page(old_page);
			page_table = pte_offset_map_lock(mm, pmd, address,
							 &ptl);
			if (!pte_same(*page_table, orig_pte)) {
				unlock_page(old_page);
				pte_unmap_unlock(page_table, ptl);
				page_cache_release(old_page);
				return 0;
			}
			page_cache_release(old_page);
		}
		if (reuse_swap_page(old_page)) {
			/*
			 * The page is all ours.  Move it to our anon_vma so
			 * the rmap code will not search our parent or siblings.
			 * Protected against the rmap code by the page lock.
			 */
			page_move_anon_rmap(old_page, vma, address);
			unlock_page(old_page);
			return wp_page_reuse(mm, vma, address, page_table, ptl,
					     orig_pte, old_page, 0, 0);
		}
		unlock_page(old_page);
	} else if (unlikely((vma->vm_flags & (VM_WRITE|VM_SHARED)) ==
					(VM_WRITE|VM_SHARED))) {
		return wp_page_shared(mm, vma, address, page_table, pmd,
				      ptl, orig_pte, old_page);
	}

	/*
	 * Ok, we need to copy. Oh, well..
	 */
	page_cache_get(old_page);

	pte_unmap_unlock(page_table, ptl);
	return wp_page_copy(mm, vma, address, page_table, pmd,
			    orig_pte, old_page);
}
```

## 附录

### pte 信息

```c
* _PAGE_PRESENT 指定了虚拟内存页是否存在于内存之中，这个之前的pte_present函数里有使用

* _PAGE_ACCESS CPU每次访问内存页时，会自动设置

* _PAGE_DIRTY表示页是否是脏的，即页的内容是否修改过

* _PAGE_FILE与_PAGE_DIRTY相同，但用于不同的上下文，即页不在内存中的时候

* _PAGE_USER，如果设置了_PAGE_USER则允许用户访问该页，否则，只有内核能够访问

* _PAGE_READ、_PAGE_WRITE、_PAGE_EXECUTE制定了普通的用户进程是否允许读取、写入、执行该页中的机器代码

* 对应于这些标志，内核提供了一些函数来查看和设置不同标志的状态

* pte_present 页是否在内存中

* pte_read 从用户空间是否可以读取该页

* pte_write 是否可以写入该页

* pte_exec 该页中的数据是否可以作为二进制代码执行

* pte_dirty 页的内容是否被修改过

* pte_file 该页表项是否属于非线性映射

* pte_young 访问位(_PAGE_ACCESS)是否设置了

* pte_rdpprote 清除该页的读权限

* pte_wrprote 清除该页的写权限

* pte_exprote 清除该页的二进制数据的权限

* pte_mkread 设置读权限

* pte_mkwrite 设置写权限

* pte_mkexec 允许执行页的内容

* pte_mkdirty 将页标记为脏

* pte_mkclean "清除"页，通常是清除_PAGE_DIRTY位

* pte_mkyoung 设置访问位，通常是_PAGE_ACCESSD

* pte_mkold 清除访问位

* install_special_mapping
```

### 文件映射缺页中断处理

#### 读中断处理

```c
/*
 * 对于文件读缺页异常：
 * 1. 首先尝试对异常地址附近的16个页面进行映射，减少后续缺页中断次数
 * 2. 使用vm_ops->fault()函数申请页面
 * 3. 将new page通过do_set_pte加入到进程内存中
 */
static int do_read_fault(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pmd_t *pmd,
		pgoff_t pgoff, unsigned int flags, pte_t orig_pte)
{
	struct page *fault_page;
	spinlock_t *ptl;
	pte_t *pte;
	int ret = 0;

	/*
	 * fault_around_bytes=65536B=16 pages
	 * 如果定义了map_pages()方法; 可以围绕在缺页异常地址周围提前映射尽可能多的页面
	 * 提前建立进程地址空间和page cache的映射关系有利于减少发生缺页中断的次数，从而提高效率
	 */
	if (vma->vm_ops->map_pages && fault_around_bytes >> PAGE_SHIFT > 1) {
		pte = pte_offset_map_lock(mm, pmd, address, &ptl);/* 获取当前pte页表项, 加锁版本 */
		/* 尝试围绕缺页异常addr映射更多的页面，减少缺页异常次数; 使用vm_ops->map_pages()映射页面线性地址 */
		do_fault_around(vma, address, pte, pgoff, flags);
		/* 判断page_table和原来的orig_pte是否一致，什么时候会不同呢？
		 * 如果单线程应该肯定相同；如果多线程，有可能pte又被别的线程映射了？
		 */
		if (!pte_same(*pte, orig_pte))
			goto unlock_out;/* 如果不相同，则认为别的线程已经映射了，直接返回？ */
		pte_unmap_unlock(pte, ptl);
	}
	/* 通过vma->vm_ops->fault()创建一个page cache，完成页面申请
	 * fault函数有不同,可以自己实现；special_mapping_fault
	 */
	ret = __do_fault(vma, address, pgoff, flags, NULL, &fault_page);
	if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
		return ret;

	pte = pte_offset_map_lock(mm, pmd, address, &ptl);
	/* 
	 * 由于内存分配成功后，会更新pte表项
	 * 类似，重读当前缺页异常地址addr对应pte的值和以前读出来的是否一致；
	 * 不一致则说明有别的线程修改了pte，则刚分配的页面无效 */
	if (unlikely(!pte_same(*pte, orig_pte))) {
		pte_unmap_unlock(pte, ptl);
		unlock_page(fault_page);
		page_cache_release(fault_page);
		return ret;
	}
	/* 通过上面操作；已确定New page位置，下一步就是将页与相应进程建立联系；主要两个步骤：
	 * 1. 将页加入到进程页表
	 * 2. 合并到逆向映射数据结构中
	 * 3. 最后通过update_mmu_cache更新处理器MMU缓存
	 */
	do_set_pte(vma, address, fault_page, pte, false, false);
	unlock_page(fault_page);
unlock_out:
	pte_unmap_unlock(pte, ptl);
	return ret;
}
```

#### 私有映射写时复制

```c
/*
 * 私有文件映射的VMA中发生了写时复制：
 * 1. 通过buddy sys申请一个新页面
 * 2. 使用__do_fault通过vma->vm_ops->fault()将文件内容读取到fault_page页面
 * 3. 如果fault_page存在，则将fault_page的内容复制到new_page中
 * 4. 重新获取异常地址对应的页表项
 * 5. 将new_page对应的pte entry设置到硬件页表中
 * 6. 将new_page页面添加到对应的LRU链表
 */
static int do_cow_fault(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pmd_t *pmd,
		pgoff_t pgoff, unsigned int flags, pte_t orig_pte)
{
	struct page *fault_page, *new_page;
	struct mem_cgroup *memcg;
	spinlock_t *ptl;
	pte_t *pte;
	int ret;
	/* 检查VMA是否初始化了RMAP反向映射 */
	if (unlikely(anon_vma_prepare(vma)))
		return VM_FAULT_OOM;
	/* 通过伙伴系统分配一个页面 */
	new_page = alloc_page_vma(GFP_HIGHUSER_MOVABLE, vma, address);
	if (!new_page)
		return VM_FAULT_OOM;

	if (mem_cgroup_try_charge(new_page, mm, GFP_KERNEL, &memcg)) {
		page_cache_release(new_page);
		return VM_FAULT_OOM;
	}
	/* 通过vm_ops->fault()读取文件内容到fault_page页面里 */
	ret = __do_fault(vma, address, pgoff, flags, new_page, &fault_page);
	if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
		goto uncharge_out;
	/* 将fault_page页面的内容复制到刚才新分配的页面new_page中 */
	if (fault_page)
		copy_user_highpage(new_page, fault_page, address, vma);
	__SetPageUptodate(new_page);
	/* 老套路，重新获取addr对应的pte，检查是否被多线程修改；紧接着就要使用do_set_pte修改pte页表了 */
	pte = pte_offset_map_lock(mm, pmd, address, &ptl);
	if (unlikely(!pte_same(*pte, orig_pte))) {
		pte_unmap_unlock(pte, ptl);
		if (fault_page) {
			unlock_page(fault_page);
			page_cache_release(fault_page);
		} else {
			/*
			 * The fault handler has no page to lock, so it holds
			 * i_mmap_lock for read to protect against truncate.
			 */
			i_mmap_unlock_read(vma->vm_file->f_mapping);
		}
		goto uncharge_out;
	}
	do_set_pte(vma, address, new_page, pte, true, true);
	mem_cgroup_commit_charge(new_page, memcg, false);
	/* 将该页加入到LRU缓存活动区域 */
	lru_cache_add_active_or_unevictable(new_page, vma);
	pte_unmap_unlock(pte, ptl);
	if (fault_page) {
		unlock_page(fault_page);
		page_cache_release(fault_page);
	} else {
		/*
		 * The fault handler has no page to lock, so it holds
		 * i_mmap_lock for read to protect against truncate.
		 */
		i_mmap_unlock_read(vma->vm_file->f_mapping);
	}
	return ret;
uncharge_out:
	mem_cgroup_cancel_charge(new_page, memcg);
	page_cache_release(new_page);
	return ret;
}
```

#### 共享映射缺页中断

[do_shared_fault](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/15/ARM64内存管理十五-do_page_fault缺页中断/#共享映射缺页中断)

```c
/*
 * 处理可写的共享映射发生缺页中断的情况
 * 1. 读取文件到fault_page中
 * 2. 使页面变为可写页面(与do_read_page()函数不同之处)
 * 3. 获取fault_page对应的pte
 * 4. 将新生成的PTE entry设置到硬件页表中
 * 5. 将page标记为dirty(与do_read_page()函数不同之处)
 * 6. 通过balance_dirty_pages_ratelimited()来平衡并回写一部分脏页。
 */
static int do_shared_fault(struct mm_struct *mm, struct vm_area_struct *vma,
		unsigned long address, pmd_t *pmd,
		pgoff_t pgoff, unsigned int flags, pte_t orig_pte)
{
	struct page *fault_page;
	struct address_space *mapping;
	spinlock_t *ptl;
	pte_t *pte;
	int dirtied = 0;
	int ret, tmp;
	/* 读取文件到falut_page中 */
	ret = __do_fault(vma, address, pgoff, flags, NULL, &fault_page);
	if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
		return ret;

	/*
	 * Check if the backing address space wants to know that the page is
	 * about to become writable
	 */
	if (vma->vm_ops->page_mkwrite) {
		unlock_page(fault_page);
		/* 使页面变为可写页面(与do_read_page()函数不同之处)
		 *一个页面变成可写，那么进程有可能需要等待这个page内容回写成功(writeback)
		 */
		tmp = do_page_mkwrite(vma, fault_page, address);
		if (unlikely(!tmp ||
				(tmp & (VM_FAULT_ERROR | VM_FAULT_NOPAGE)))) {
			page_cache_release(fault_page);
			return tmp;
		}
	}
	/* 老套路，获取fault_page对应的pte，检查是否改动 */
	pte = pte_offset_map_lock(mm, pmd, address, &ptl);
	if (unlikely(!pte_same(*pte, orig_pte))) {
		pte_unmap_unlock(pte, ptl);
		unlock_page(fault_page);
		page_cache_release(fault_page);
		return ret;
	}
	do_set_pte(vma, address, fault_page, pte, true, false);
	pte_unmap_unlock(pte, ptl);
	/* 将page标记为dirty(与do_read_page()函数不同之处) */
	if (set_page_dirty(fault_page))
		dirtied = 1;
	/*
	 * Take a local copy of the address_space - page.mapping may be zeroed
	 * by truncate after unlock_page().   The address_space itself remains
	 * pinned by vma->vm_file's reference.  We rely on unlock_page()'s
	 * release semantics to prevent the compiler from undoing this copying.
	 */
	mapping = fault_page->mapping;
	unlock_page(fault_page);
	if ((dirtied || vma->vm_ops->page_mkwrite) && mapping) {
		/*
		 * 通过balance_dirty_pages_ratelimited()来平衡并回写一部分脏页?
		 */
		balance_dirty_pages_ratelimited(mapping);
	}

	if (!vma->vm_ops->page_mkwrite)
		file_update_time(vma->vm_file);

	return ret;
}
```

## 参考资料

[内存分配函数总结](https://www.cnblogs.com/arnoldlu/p/8251333.html)

[缺页中断详解](https://blog.csdn.net/u010246947/article/details/10431149)

[COW分析](https://yq.aliyun.com/articles/378695)

[内存回收(匿名页反向映射)](https://www.cnblogs.com/tolimit/p/5398552.html)

[do_swap_page](https://blog.csdn.net/zsj100213/article/details/82433171)

[缺页异常集中情况处理机制简介](https://blog.csdn.net/zsj100213/article/details/82430224)

[dirty cow](https://brucetg.github.io/2018/05/27/DirtyCow（脏牛）漏洞复现/)
