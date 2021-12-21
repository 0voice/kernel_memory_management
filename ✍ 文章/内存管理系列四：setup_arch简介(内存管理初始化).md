linux内存管理主要分3个阶段：

- MMU未打开，还在汇编时代时；
- fixmap, memblock时代，此时伙伴系统还未成形，一直到mm_init()函数中mem_ini()将空闲内存加载到zone中；
- 伙伴系统建立

## start_kernel

`start_kernel`中和内存管理相关的系统初始化函数主要如下：

—->setup_arch: 体系机构的设置函数，还负责初始化自举分配器

—->setup_per_cpu_areas: 定义per-cpu变量,为各个cpu分别创建一份这些变量副本

—->build_all_zonelists: 建立节点(node)和内存域(zone)的数据结构

—->mem_init: 停用bootmem分配器并迁移到实际的内存管理函数

—->kmem_cache_init: 初始化内核内部用于小块内存区的分配器

—->setup_per_cpu_pageset: 为各个cpu的zone的pageset数组的第一个数组元素分配内存

## setup_arch

setup_arch

—->machine_specific_memory_setup: 创建一个列表，包括系统占据的内存区和空闲内存区

—->parse_early_param: 解析dtb树命令行

—->setup_memory: 确定每个节点可用的物理内存也数目，初始化bootmem分配器，分配各种内存区

—->paging_init: 初始化内核页表并启动内存分页

```
---->pagetable_init: 确保直接映射到内核地址空间的物理内存被初始化
```

—->zone_size_init:初始化系统中所有节点的pgdat_t实例

```
---->add_active_range: 对可用的物理内存建立一个相对简单的列表

---->free_area_init_nodes: 建立完备的内核数据结构
```

## 本文主要内容

> 本篇主要介绍memblock建立过程及分页机制化，主要有如下几个步骤

- setup_machine_fdt: 解析dtb，收集内存信息及bootargs
  1. hoosen node。该节点有一个bootargs属性，该属性定义了内核的启动参数，而在启动参数中，可能包括了mem=nn[KMG]这样的参数项。initrd-start和initrd-end参数定义了initial ramdisk image的物理地址范围。
  2. memory node。这个节点主要定义了系统中的物理内存布局。主要的布局信息是通过reg属性来定义的，该属性定义了若干的起始地址和size条目。
  3. DTB header中的memreserve域。对于dts而言，这个域是定义在root node之外的一行字符串，例如：/memreserve/ 0x05e00000 0x00100000。
  4. reserved-memory node。这个节点及其子节点定义了系统中保留的内存地址区域。保留内存有两种(1. 静态定义，用reg属性定义的address和size; 2. 动态定义，通过size属性定义了保留内存区域的长度，或者通过alignment属性定义对齐属性)
- early_fixmap_init: 对保留的fixmap区域创建映射
- early_ioremap_init: 初始化early_ioremap机制
- arm64_memblock_init: 初始化memblock机制
- paging_init: 初始化内核页表，内存节点，内存域及页帧page, 此函数功能较为复杂
- request_standard_resources：将memblock.memory挂载到iomem_resource资源树下
- early_ioremap_reset: 结束early_ioremap机制
- unflatten_device_tree: dtb转换为device_node tree
- 根据device node tree初始化CPU，psci

## 关键函数分析

### setup_arch

```
void __init setup_arch(char **cmdline_p)
{
	setup_processor();
/*
__fdt_pointer: fdt phy, 在__mmap_switched中被赋值
setup_machine_fdt: 校验fdt合法性，并通过下面3个回调函数扫描fdt中预留的memory及传入内核参数
1. 扫描device tree函数：of_scan_flat_dt
2. 3个回调函数：
early_init_dt_scan_chosen： 解析chosen信息，其中一般包括initrd、bootargs，保留命令行参数到boot_command_line全局变量中
early_init_dt_scan_root: 从fdt中初始化{size, address}结构信息，保存到dt_root_addr_cells,dt_root_size_cells中
early_init_dt_scan_memory：扫描fdt中memory区域，寻找device_type="memory"的节点，并通过以下函数将该区域添加到memblock管理系统
	early_init_dt_add_memory_arch
		memblock_add：将region添加到memblock.memory中
*/
	setup_machine_fdt(__fdt_pointer); 

	init_mm.start_code = (unsigned long) _text;
	init_mm.end_code   = (unsigned long) _etext;
	init_mm.end_data   = (unsigned long) _edata;
	init_mm.brk	   = (unsigned long) _end;
	/* 将fdt中解析的boot_command_line参数传出setup_arch函数*/
	*cmdline_p = boot_command_line;
	/* 
	对保留的fixmap区域创建映射，并对fixmap校验是否在同一pmd内，否则报错
	pgd_offset_k,pud_offset,pmd_offset:获取addr对应在p*d表中entry的地址
	pud_populate(struct mm_struct *mm, pud_t *pud, pmd_t *pmd): 使用pmd指针指向的内存填充pud，然后设置内存屏障，并isb(inner shareable)
	*/
	early_fixmap_init();
	
	/* 
	* 将fixmap区域各个区域的地址存入slot_virt全局静态数组中
	* 第一块区域的首地址为：FIXADDR_TOP(0xfefe0000),以后每一块往低地址区域扩展
	*/
	early_ioremap_init();
	/* 
	解析boot_command_line, 并设置done标志位
	*/
	parse_early_param();

	/*
	 *  Unmask asynchronous aborts after bringing up possible earlycon.
	 * (Report possible System Errors once we can report this occurred)
	 */
	local_async_enable();
	/*
	 * 1. 使用回调函数fdt_find_uefi_params获取fdt中关于uefi的信息，所有要扫描的参数包含在dt_param这个全局数组中
	 *         例如：system table, memmap address, memmap size, memmap desc. size, memmap desc. version
	 * 2. 将解析出的内存区域加入memblock.reserve区域
	 * 3. 调用uefi_init(), reserve_regions()等函数进行uefi模式初始化
	 * 4. 初始化完成后，将前面reserve的区域unreserve
	 * 
	 */
	efi_init();
	/*
	 * 1. 初始化memblock, 详情见该函数进一步分析
	 */
	arm64_memblock_init();

	/* Parse the ACPI tables for possible boot-time configuration */
	acpi_boot_table_init();
	/*
	 * 该函数较复杂，内容较多，是buddy系统的基础，目前没有完全理解，详细分析可参考资料
	 * 1. map_mem：初始化内核页表
	 * 2. bootmem_init：初始化内存节点，内存域及页帧page
	 */
	paging_init();
	/*
	 * 将memblock.memory挂载到iomem_resource资源树下, 资源树是一颗倒挂的树
	 * request_resource：将设备实体登记注册到总线空间链
	 * 在遍历memblock.memory过程中，会检查kernel_code,kernel_data是否属于某region，如果是则挂载到该region下
	 * 
	 */
	request_standard_resources();
	/*
	 * early_ioremap 功能到此就结束了
	 * 在buddy系统还未建立前，要进行寄存器访问基本职能使用early_ioremap 功能
	 */
	early_ioremap_reset();

	if (acpi_disabled) {
		/*
		 * 将dtb转换为device_node tree，根节点为全局变量 of_allnodes
		 * 后续内核会遍历tree来初始化描述的各个device
		 */
		unflatten_device_tree();
		psci_dt_init();
		cpu_read_bootcpu_ops();
#ifdef CONFIG_SMP
		of_smp_init_cpus();
#endif
	} else {
		psci_acpi_init();
		acpi_init_cpus();
	}

#ifdef CONFIG_SMP
	smp_build_mpidr_hash();
#endif

#ifdef CONFIG_VT
#if defined(CONFIG_VGA_CONSOLE)
	conswitchp = &vga_con;
#elif defined(CONFIG_DUMMY_CONSOLE)
	conswitchp = &dummy_con;
#endif
#endif
	if (boot_args[1] || boot_args[2] || boot_args[3]) {
		pr_err("WARNING: x1-x3 nonzero in violation of boot protocol:\n"
			"\tx1: %016llx\n\tx2: %016llx\n\tx3: %016llx\n"
			"This indicates a broken bootloader or old kernel\n",
			boot_args[1], boot_args[2], boot_args[3]);
	}
}
```

### arm64_memblock_init

> setup_arch->arm64_memblock_init

```
void __init arm64_memblock_init(void)
{
	/*
	将memblock.reserve保留区域大小强制限定，因为总内存大小有限
	并将最后超标的内存用memblock_remove_range进行截断
	*/
	memblock_enforce_memory_limit(memory_limit);

	/*
	 * Register the kernel text, kernel data, initrd, and initial
	 * pagetables with memblock.
	 */
	memblock_reserve(__pa(_text), _end - _text);
#ifdef CONFIG_BLK_DEV_INITRD
	if (initrd_start)
		memblock_reserve(__virt_to_phys(initrd_start), initrd_end - initrd_start);
#endif
	/*
	1. 将dtb区域加入memblock.reserve, initial_boot_params=fdt
	2. 通过fdt_header.off_mem_rsvmap指针，寻找/memreserve/ fields，并加入memblock.reserve
	3. 使用回调__fdt_scan_reserved_mem, 找出所有"reserved-memory",分析后加入memblock.reserve，并分配内存空间
	*/
	early_init_fdt_scan_reserved_mem();

	/* 4GB maximum for 32-bit only capable devices */
	if (IS_ENABLED(CONFIG_ZONE_DMA))
		arm64_dma_phys_limit = max_zone_dma_phys();
	else
		arm64_dma_phys_limit = PHYS_MASK + 1;
	/* CMA区域或CMA上下文, 保留连续的内存空间给Global CMA area使用，稍后返回给伙伴系统从而可以被用作正常申请, 关于CMA具体可参考：https://www.cnblogs.com/newjiang/p/9592797.html */
	dma_contiguous_reserve(arm64_dma_phys_limit);
	/* 设置memblock_can_resize标志，目前暂未确定用途？*/
	memblock_allow_resize();
	/* 如果打开了memblock_debug开关，则会打印memblock结构体中目前保存的memory、reserve/* 
	memblock_dump_all();
}
```

### early_init_fdt_scan_reserved_mem

```
/**
 * early_init_fdt_scan_reserved_mem() - create reserved memory regions
 *
 * This function grabs memory from early allocator for device exclusive use
 * defined in device tree structures. It should be called by arch specific code
 * once the early allocator (i.e. memblock) has been fully activated.
 * 将dtb中定义的需要reserve的memory加入memblock.reserve，主要涉及/memreserve/、reserved-memory 以及dtb本身所占的内存空间
 * 其中/memreserve/、reserved-memory的区别参考：https://blog.csdn.net/prike/article/details/79524117
 */
void __init early_init_fdt_scan_reserved_mem(void)
{
	int n;
	u64 base, size;

	if (!initial_boot_params)
		return;

	/* Reserve the dtb region, 将dtb区域加入memblock.reserve, initial_boot_params=fdt */
	early_init_dt_reserve_memory_arch(__pa(initial_boot_params),
					  fdt_totalsize(initial_boot_params),
					  0);

	/* Process header /memreserve/ fields */
	for (n = 0; ; n++) {
		/* 通过fdt_header.off_mem_rsvmap指针，找出/memreserve/ fields的base & size */
		fdt_get_mem_rsv(initial_boot_params, n, &base, &size);
		if (!size)
			break;
			/*将/memreserve/ fields 加入memblock.reserve， */
		early_init_dt_reserve_memory_arch(base, size, 0);
	}
	/* 将/reserved-memory/ fields 加入memblock.reserve， 并将其保存到全局数组reserved_mem， 目前并未确定有什么作用*/
	of_scan_flat_dt(__fdt_scan_reserved_mem, NULL);
	/* 将/reserved-memory/ fields 分配内存空间 */
	fdt_init_reserved_mem();
}
```

> 在看paging_init()函数前，我们先看下目前的内存状态

### 目前所有的内存分为了2部分

- OS已经收集到的内存分布信息(来自dtb解析)，保存在memblock中，这部分又分为3个小部分
  1. 系统内存占据的空间，信息保存在memblock.memory中
  2. 已经使用或者保留使用的，信息保存在memblock.reserve中
  3. dtb中reserved-memory，但是有No-map属性，这种内存不属于OS管辖
- OS还未收集到的内存部分，暂未管辖(这部分稍后会被加载到伙伴系统中)

在目前状态下，OS还无法正常使用它们，因为memblock中定义的都是物理地址；而目前仅有两段内存是已经mapping过(kernel image, fdt),其余段都还是黑暗状态，接下来就要给第一部分内存做mapping

### paging_init

```
/*
 * paging_init() sets up the page tables, initialises the zone memory
 * maps and sets up the zero page.
 */
void __init paging_init(void)
{
	void *zero_page;
	/*
	* 对memblock.memory建立对应的mapping
	* 其中要注意的是在mapping过程中，如果某region的任意translation table不存在的话，都需要进行页表内存的分配；但目前buddy还没有ready，无法动态分配
	* 而memblock分配的内存都还没有创建地址映射，一旦通过memblock_alloc分配物理内存，则会产生panic；因此需要设定memblock上限
	* 
	* 这一块对于section map情况下代码还是有疑问？ 
	*
	* 在map_mem之后，所有之前通过__create_page_tables创建的描述符都被覆盖，对于具体create_mapping细节，见参考资料2
	*/
	map_mem();
	fixup_executable();

	/* allocate the zero page. */
	zero_page = early_alloc(PAGE_SIZE);
	/* 稀疏内存管理模型，为Buddy系统做基础，详细分析见后 */
	bootmem_init();

	empty_zero_page = virt_to_page(zero_page);

	/*
	 * TTBR0 is only used for the identity mapping at this stage. Make it
	 * point to zero page to avoid speculatively fetching new entries.
	 */
	cpu_set_reserved_ttbr0();
	flush_tlb_all();
	cpu_set_default_tcr_t0sz();
}
```

### buddy系统初始化

#### 到目前为止，内核完成了如下工作

- memblock已经通过arm64_memblock_init完成了初始化, 至此系统中的内存可以通过memblock分配了
- paging_init完成了分页机制的初始化, 至此内核已经布局了一套完整的虚拟内存空间

#### 稀疏内存管理将整个物理地址空间划分为section

- 对于ARM64，一般支持48bit物理地址(256T)，section为1G物理块，可以划分为256K个seciton；
- 每个在位的section在软件上抽象为一个struct mem_section结构体；
- 对于每个section又可以分为若干Pageblock，每个pageblock的状态由4bit来描述

#### bootmem_init

```
/* 初始化内存数据结构(内存节点，内存域，页帧page)，不再依赖于特定体系结构 */
void __init bootmem_init(void)
{
	unsigned long min, max;
	/* 获取最大，最小页号 */
	min = PFN_UP(memblock_start_of_DRAM());
	max = PFN_DOWN(memblock_end_of_DRAM());

	early_memtest(min << PAGE_SHIFT, max << PAGE_SHIFT);

	/*
	 * Sparsemem tries to allocate bootmem in memory_present(), so must be
	 * done after the fixed reservations.
	 */
	/* 遍历所有的memory region，每个memory region换分成1G大小的section，并设置section在位 */
	arm64_memory_present();
	/*
	 * 重点，见下详细分析
	 */
	sparse_init();
	/*
	 * 初始化zone的核心，暂未完全看明白，详细可见参考资料
	 */
	zone_sizes_init(min, max);

	high_memory = __va((max << PAGE_SHIFT) - 1) + 1;
	max_pfn = max_low_pfn = max;
}
```

#### sparse_init

```
/*
 * Allocate the accumulated non-linear sections, allocate a mem_map
 * for each and record the physical to section mapping.
 */
void __init sparse_init(void)
{
	unsigned long pnum;
	struct page *map;
	unsigned long *usemap;
	unsigned long **usemap_map;
	int size;
#ifdef CONFIG_SPARSEMEM_ALLOC_MEM_MAP_TOGETHER
	int size2;
	struct page **map_map;
#endif

	/* see include/linux/mmzone.h 'struct mem_section' definition */
	BUILD_BUG_ON(!is_power_of_2(sizeof(struct mem_section)));

	/* Setup pageblock_order for HUGETLB_PAGE_SIZE_VARIABLE */
	set_pageblock_order();

	/*
	 * map is using big page (aka 2M in x86 64 bit)
	 * usemap is less one page (aka 24 bytes)
	 * so alloc 2M (with 2M align) and 24 bytes in turn will
	 * make next 2M slip to one more 2M later.
	 * then in big system, the memory will have a lot of holes...
	 * here try to allocate 2M pages continuously.
	 *
	 * powerpc need to call sparse_init_one_section right after each
	 * sparse_early_mem_map_alloc, so allocate usemap_map at first.
	 */
	size = sizeof(unsigned long *) * NR_MEM_SECTIONS;
	/* 为每个section分配一个内存,用于保存bitmap内存地址 */
	usemap_map = memblock_virt_alloc(size, 0);
	if (!usemap_map)
		panic("can not allocate usemap_map\n");
		/*分配section的pageblock bitmap位图内存 */
	alloc_usemap_and_memmap(sparse_early_usemaps_alloc_node,
							(void *)usemap_map);

	for (pnum = 0; pnum < NR_MEM_SECTIONS; pnum++) {
		if (!present_section_nr(pnum))//遍历每一个section，如果不在位则continue
			continue;

		usemap = usemap_map[pnum];
		if (!usemap)
			continue;
		/* 
		* 每个section中，有1<<(30-12)个page，因此需要1<<(30-12)个struct page结构体
		* 需要(1<<(30-12))*sizeof(struct page) = 16M.也就是1G物理内存，需要4K个物理页面来存struct page
		* 
		*/
		map = sparse_early_mem_map_alloc(pnum);

		if (!map)
			continue;
		/* 
		* mem_section.section_mem_map为该section对应的struct page指针，其中bit0表示section是否在位、bit1表示是否从section到page有map过
		* pageblock_flags为该section所有pageblock的MIGRATE_TYPES属性的内存地址，每pageblock用4bit存放类型
		*/
		sparse_init_one_section(__nr_to_section(pnum), pnum, map,
								usemap);
	}

	vmemmap_populate_print_last();

	memblock_free_early(__pa(usemap_map), size);
}
```

### build_all_zonelists

> build_all_zonelists->build_all_zonelists_init->__build_all_zonelists

```
static int __build_all_zonelists(void *data)
{
	int nid;
	int cpu;
	pg_data_t *self = data;

#ifdef CONFIG_NUMA
	memset(node_load, 0, sizeof(node_load));
#endif

	if (self && !node_online(self->node_id)) {
		build_zonelists(self);
		build_zonelist_cache(self);
	}

	for_each_online_node(nid) {
		pg_data_t *pgdat = NODE_DATA(nid);

		build_zonelists(pgdat);
		build_zonelist_cache(pgdat);
	}

	/*
	 * Initialize the boot_pagesets that are going to be used
	 * for bootstrapping processors. The real pagesets for
	 * each zone will be allocated later when the per cpu
	 * allocator is available.
	 *
	 * boot_pagesets are used also for bootstrapping offline
	 * cpus if the system is already booted because the pagesets
	 * are needed to initialize allocators on a specific cpu too.
	 * F.e. the percpu allocator needs the page allocator which
	 * needs the percpu allocator in order to allocate its pagesets
	 * (a chicken-egg dilemma).
	 */
	for_each_possible_cpu(cpu) {
		setup_pageset(&per_cpu(boot_pageset, cpu), 0);

#ifdef CONFIG_HAVE_MEMORYLESS_NODES
		/*
		 * We now know the "local memory node" for each node--
		 * i.e., the node of the first zone in the generic zonelist.
		 * Set up numa_mem percpu variable for on-line cpus.  During
		 * boot, only the boot cpu should be on-line;  we'll init the
		 * secondary cpus' numa_mem as they come on-line.  During
		 * node/memory hotplug, we'll fixup all on-line cpus.
		 */
		if (cpu_online(cpu))
			set_cpu_numa_mem(cpu, local_memory_node(cpu_to_node(cpu)));
#endif
	}

	return 0;
}
```

\###

```
static void build_zonelists(pg_data_t *pgdat)
{
	int node, local_node;
	enum zone_type j;
	struct zonelist *zonelist;

	local_node = pgdat->node_id;

	zonelist = &pgdat->node_zonelists[0];
	j = build_zonelists_node(pgdat, zonelist, 0);

	/*
	 * Now we build the zonelist so that it contains the zones
	 * of all the other nodes.
	 * We don't want to pressure a particular node, so when
	 * building the zones for node N, we make sure that the
	 * zones coming right after the local ones are those from
	 * node N+1 (modulo N)
	 */
	for (node = local_node + 1; node < MAX_NUMNODES; node++) {
		if (!node_online(node))
			continue;
		j = build_zonelists_node(NODE_DATA(node), zonelist, j);
	}
	for (node = 0; node < local_node; node++) {
		if (!node_online(node))
			continue;
		j = build_zonelists_node(NODE_DATA(node), zonelist, j);
	}

	zonelist->_zonerefs[j].zone = NULL;
	zonelist->_zonerefs[j].zone_idx = 0;
}
```

参考资料：

1. http://www.wowotech.net/memory_management/memory-layout.html
2. http://www.wowotech.net/memory_management/mem_init_3.html

[memblock结构体介绍](https://blog.csdn.net/zhoutaopower/article/details/88243238)

[bootmem_init详细分析](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/02-memory/03-initialize/04-bootmem_init)

[bootmem_init详细分析2](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/kernel/02-memory/03-initialize/04-bootmem_init)
