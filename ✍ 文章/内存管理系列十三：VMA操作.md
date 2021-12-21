## 前沿

### 往篇回顾

> 在上两篇中，主要分析了vmalloc模块初始化、vmalloc申请内存、vmalloc内存释放的过程

- 在kernel_start()->mm_init()函数中会调用vmalloc_init()进行模块初始化；主要有个任务：
  1. 初始化各个cpu中vmalloc的管理结构vmap_block_queue、vfree_deferred，目前并未看到作用是什么；
  2. 将挂在全局链表`vm_struct *vmlist`中的vmalloc区通过__insert_vmap_area()插入`vmap_area_root`红黑树及链表中;
- 通过vmalloc申请非连续物理内存主要有如下几步：
  1. 首先通过__get_vm_area_node()向slub系统为管理结构体申请内存;
  2. 在alloc_vmap_area()通过查询红黑树、链表找到vmalloc合适的虚拟内存区，并将该区域插入红黑树、链表中;
  3. 通过__vmalloc_area_node()为area分配管理page数组，并向伙伴系统申请物理空间，并使用map_vm_area()将物理地址和虚拟地址进行映射；
- 由于所有进程共享内核地址空间，内核中所有vm_struct放在一起，通过vm_struct->next链成表；
- vmalloc区的具体地址映射信息抽成一个结构vmap_area，该结构在内核通过红黑树(vmap_area_root)和双向链表(vmap_area_list)管理;

### VMA简介

- 进程的虚拟内存空间(管理结构体struct mm_struct)按功能分为如下几部分: stack, mmap, heap, bbs segment, data segment, text segment; 均由VMA管理, 每块虚拟内存对应一个管理结构体[vm_area_struct](https://pzh2386034.github.io/Black-Jack/linux-memory/2019/09/05/ARM64内存管理十三-VMA操作/#vm_area_struct);
- 类似内核vmalloc，VMA也是有一颗红黑树，一个链表来管理；
- 当使用malloc()->brk()向系统请求一些内存时，内核仅仅更新堆的VMA，真正分配物理页帧在发生缺页中断时，系统调用do_page_fault()；

### 本篇主要内容

> vmalloc是内核的虚拟内存申请工具；而对于用户态的虚拟内存管理，则主要依赖VMA模块

## 代码分析

### find_vma

```c
/* Look up the first VMA which satisfies  addr < vm_end,  NULL if none.
 * struct mm_struct是描述进程内存管理的核心数据结构
 * 首先尝试从vma cache中查找，再从进程红黑树中查找
 * 1. 快速查找：首先从4槽位 per-thread VMA 缓存hash表中进行查找；
 * 2. 慢速查找：从进程红黑树中查找到合适VMA后，通过vmacache_update()更新到vmacache
 * 当释放VMA时，必须使所有线程的VMA cache无效，否则后续VMA查找将可能指向空指针
 */
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
	struct rb_node *rb_node;
	struct vm_area_struct *vma;

	/* 
	 * 首先从vma cache中查找，如果addr在某个vma地址范围内，则直接返回该vma
	 * 其中有一个关键函数vmacache_valid，关联一个严重的内存提权漏洞CVE-2018-17182，见附录资料
	 * vmacache_find：主要为了解决释放VMA后，为使高速缓存无效，避免遍历所有线程的VMA，防止出现性能问题
	 */
	vma = vmacache_find(mm, addr);
	if (likely(vma))
		return vma;

	rb_node = mm->mm_rb.rb_node;
	vma = NULL;

	while (rb_node) {
		struct vm_area_struct *tmp;

		tmp = rb_entry(rb_node, struct vm_area_struct, vm_rb);

		if (tmp->vm_end > addr) {
			vma = tmp;
			if (tmp->vm_start <= addr)
				break;
			rb_node = rb_node->rb_left;
		} else
			rb_node = rb_node->rb_right;
	}

	if (vma)/* 将新找到的vma更新到vma cache中 */
		vmacache_update(addr, vma);
	return vma;
}
```

### insert_vm_struct

```c
/* 
 * 在线性区对象链表和内存描述符的红黑树中插入一个vm
 * mm: 指定进程内存描述符；vma:要插入的vm_area_struct对象地址
 * 1. 利用find_vma_links()寻找出将要插入的节点位置,及前驱，父节点
 * 2. 利用__vma_link_list()和__vma_link_rb()将结点分别插入链表和红黑树中，vma_link()是其前端函数
 * 3. 若线性区用于文件映射，那么利用__vma_link_file()处理，暂不讨论
 * 4. 线性区计数+1
 */
int insert_vm_struct(struct mm_struct *mm, struct vm_area_struct *vma)
{
	struct vm_area_struct *prev;
	struct rb_node **rb_link, *rb_parent;

	/*
	 * The vm_pgoff of a purely anonymous vma should be irrelevant
	 * until its first write fault, when page's anon_vma and index
	 * are set.  But now set the vm_pgoff it will almost certainly
	 * end up with (unless mremap moves it elsewhere before that
	 * first wfault), so /proc/pid/maps tells a consistent story.
	 *
	 * By setting it to reflect the virtual start address of the
	 * vma, merges and splits can happen in a seamless way, just
	 * using the existing file pgoff checks and manipulations.
	 * Similarly in do_mmap_pgoff and in do_brk.
	 */
	if (!vma->vm_file) {
		BUG_ON(vma->anon_vma);
		vma->vm_pgoff = vma->vm_start >> PAGE_SHIFT;
	}
	/* 找到vma插入点 */
	if (find_vma_links(mm, vma->vm_start, vma->vm_end,
			   &prev, &rb_link, &rb_parent))
		return -ENOMEM;
	if ((vma->vm_flags & VM_ACCOUNT) &&
	     security_vm_enough_memory_mm(mm, vma_pages(vma)))
		return -ENOMEM;
	/* 实际执行vma插入动作 */
	vma_link(mm, vma, prev, rb_link, rb_parent);
	return 0;
}
```

### insert_vm_struct

```c
/* 
 * 当一个新建的VMA区域加入进程时，内核会试图将这个新区域与已存在的区域进行合并，vma合并条件：
 * 1. 新区域之前的prve区域终于地址是否与新区域地址重合
 * 2. 新区域结束地址是否与其之后的next区域起始地址重合
 * 3. 然后再检查要合并的区域是否有相同的标志
 * 4. 如果合并区域均映射了磁盘文件，则还要检查其映射文件是否相同，以及文件内的偏移量是否连续
 */
struct vm_area_struct *vma_merge(struct mm_struct *mm,
			struct vm_area_struct *prev, unsigned long addr,
			unsigned long end, unsigned long vm_flags,
			struct anon_vma *anon_vma, struct file *file,
			pgoff_t pgoff, struct mempolicy *policy)
{
	pgoff_t pglen = (end - addr) >> PAGE_SHIFT;
	struct vm_area_struct *area, *next;
	int err;

	/*  VM_SPECIAL标志指定了该区域不能和其他区域合并，因此立即返回NULL */
	if (vm_flags & VM_SPECIAL)
		return NULL;

	if (prev)
		next = prev->vm_next;
	else
		next = mm->mmap;
	area = next;
	if (next && next->vm_end == end)		/* cases 6, 7, 8 */
		next = next->vm_next;

	/*
	 * Can it merge with the predecessor? prev->vm_end == addr
	 * can_vma_merge_after 判断两者的标志和映射文件等是否相同
	 */
	if (prev && prev->vm_end == addr &&
			mpol_equal(vma_policy(prev), policy) &&
			can_vma_merge_after(prev, vm_flags,
						anon_vma, file, pgoff)) {
		/*
		 * OK, it can.  Can we now merge in the successor as well? end == next->vm_start
		 */
		if (next && end == next->vm_start &&
				mpol_equal(policy, vma_policy(next)) &&
				can_vma_merge_before(next, vm_flags,
					anon_vma, file, pgoff+pglen) &&
				is_mergeable_anon_vma(prev->anon_vma,
						      next->anon_vma, NULL)) {
							/* cases 1, 6 */
			err = vma_adjust(prev, prev->vm_start,
				next->vm_end, prev->vm_pgoff, NULL);
		} else					/* cases 2, 5, 7 */
			err = vma_adjust(prev, prev->vm_start,
				end, prev->vm_pgoff, NULL);
		if (err)
			return NULL;
		khugepaged_enter_vma_merge(prev, vm_flags);
		return prev;
	}

	/*
	 * Can this new request be merged in front of next?
	 */
	if (next && end == next->vm_start &&
			mpol_equal(policy, vma_policy(next)) &&
			can_vma_merge_before(next, vm_flags,
					anon_vma, file, pgoff+pglen)) {
		if (prev && addr < prev->vm_end)	/* case 4 */
			err = vma_adjust(prev, prev->vm_start,
				addr, prev->vm_pgoff, NULL);
		else					/* cases 3, 8 */
			err = vma_adjust(area, addr, next->vm_end,
				next->vm_pgoff - pglen, NULL);
		if (err)
			return NULL;
		khugepaged_enter_vma_merge(area, vm_flags);
		return area;
	}

	return NULL;
}
```

## 附录

### vm_area_struct

```c
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
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
	unsigned long vm_pgoff;		/* 指定文件映射的偏移量，单位是页面 */
	struct file * vm_file;		/* 描述一个被映射的文件. */
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

[VMA,进程虚拟空间简介](https://linux.cn/article-9393-1.html)

[vma_merge函数分析](http://edsionte.com/techblog/archives/3586)

[CVE-2018-17182 VMA-UFA 提权漏洞原理](https://www.360zhijia.com/anquan/423092.html)

[CVE-2018-17182 VMA-UFA 提权漏洞](https://www.anquanke.com/post/id/161632)

[CVE-2018-17182 VMA-UFA 提权漏洞利用code](https://github.com/jas502n/CVE-2018-17182)

[find_vma导致系能问题](http://www.trueeyu.com/2015/04/15/findvma/)
