> 上一篇中，详细分析了MMU打开前，为MMU打开后能正常启动linux kernel进行了3块区域的section map; 本篇聚焦为打开MMU而进行的CPU初始化, 主要内容：

- cache和TLB处理
- memory attributes lookup table的创建
- SCTLR_EL1、TCR_EL1的设定(详细见参考资料)

## 首先介绍icache, dcache

### icache: 指令cache(instruction cache)

- 由cp15协处理器中控制寄存器1的第12位控制，一般在MMU开启之后被使用
- icache一般有512个entry，每个16 bytes；如果miss cache, 则从内存中读取指令，且触发`8-word linefill`, 将该指令所在区域8 word写进某个entry

### dcache: 数据cache

- ARM dcache架构由cache存储器和写缓冲器（write-buffer）组成，其中写缓冲器是CACHE按照FIFO原则向主存写的缓冲处理器
- 一般来说CACHEABILITY和BUFFERABILITY都是可以配置的，所以，一块存储区域可以配置成下面4种方式：NCNB CNB NCB CB(例: ip map都采用NCNB)
- DCaches使用的是虚拟地址，它的大小是16KB,它被分成512行（entry）, 每行8个字（8 words,32Bits）。每行有两个修改标志位（dirty bits），第一个标志位标识前4个字，第二个标志位标识后4个字，同时每行中还有一个TAG 地址（标签地址）和一个valid bit

## memory type

<ARMv8采用了weakly-order内存模型,即处理器实际对内存访问（load and store）的执行序列和program order不一定保持严格的一致，处理器可以对内存访问进行reorder；例如，对于写操作，processor可能会合并两个写请求

因此，有必要对内存进行分类，以便CPU可以对不同类型做一些性能优化

- normal memory: 常用内存，访问无side effect，processor可以进行reorder, repeat或者merge以及分支预测
- IO memory: 外设IO内存，例如设备的FIFO队列，地址固定不变；process不能进行Speculative data accesses

## memory attribute

- gathering(G): 对多个memory访问是否可以合并
- ordering(R)：对内存访问指令是否可以重排
- Early Write Acknowledgement(E): PE访问memory是有问有答的,为了加快写速度，系统的中间环节可能会设定一些write buffer；例如：以便确定完成一次erite transaction. nE表示写操作的ack必须来自最终的目的而不是中间write buffer.

## 代码分析

#### __cpu_setup

```assembly
/*
 *	__cpu_setup
 *
 *	Initialise the processor for turning the MMU on.  Return in x0 the
 *	value of the SCTLR_EL1 register.
 *  老版本为vmalle1is(inner sharebility),这条指令的作用范围是所有PES，由于__cpu_setup函数会在每个cpu core上执行，因此vmalle1会多次执行，不合理
 */
ENTRY(__cpu_setup)
	tlbi	vmalle1				// Invalidate local TLB, vmalle1: vm all e1,表示该操作仅适用于EL1
	dsb	nsh                     // memory barrier，保证在执行打开MMU时，已经完成；nsh表示none inner shareable

	mov	x0, #3 << 20
	msr	cpacr_el1, x0			// Enable FP/ASIMD,CPACR:architectural feature access control register, 用来控制trace，浮点运算单元及SIMD单元；
	mov	x0, #1 << 12			// Reset mdscr_el1 and disable
	msr	mdscr_el1, x0			// access to the DCC from EL0, MDSCR： monitor debug system control register, 用来控制debug系统
	isb					// Unmask debug exceptions now,
	enable_dbg				// since this is per-cpu
	reset_pmuserenr_el0 x0			// Disable PMU access from EL0
	.................
	msr	tcr_el1, x10           //到这里x10已经准备好了TCR寄存器的值
ENDPROC(__cpu_setup)
```

#### __enable_mmu

```assembly
/*
 * Enable the MMU.
 *
 *  x0  = SCTLR_EL1 value for turning on the MMU.
 *  x27 = *virtual* address to jump to upon completion
 *
 * other registers depend on the function called upon completion
 * 传入4个参数：
 * 1. X0寄存器，保存了打开MMU时设定的SCTLR_EL1值(在__cpu_setup函数中设定)
 * 2. x25, 保存idmap_pg_dir值
 * 3. x26：保存swapper_pg_dir值
 * 4. x27: 执行完毕函数后，要跳到哪里取执行(__mmap_switched)
 */
__enable_mmu:
	ldr	x5, =vectors
	msr	vbar_el1, x5            // Vector Base Address Register (EL1), 将异常向量表存入寄存器中; 如果一个exception最终送达EL1,则CPU会跳转到该寄存器来查询向量表
	msr	ttbr0_el1, x25			// load TTBR0，当系统运行后，在进程切换时，会修改TTBR0的值，切换到真实的进程地址空间上去
	msr	ttbr1_el1, x26			// load TTBR1，用于kernel space
	isb
	msr	sctlr_el1, x0           //打开MMU
	isb
	br	x27
ENDPROC(__enable_mmu)
```

#### __mmap_switched

```assembly
/*
 * The following fragment of code is executed with the MMU enabled.
 * 用户空间的进程当陷入内核态的时候，stack切换到内核栈，实际上就是该进程的thread info内存段（4K或者8K）的顶部
 */
	.set	initial_sp, init_thread_union + THREAD_START_SP
__mmap_switched:
	adr_l	x6, __bss_start
	adr_l	x7, __bss_stop

1:	cmp	x6, x7
	b.hs	2f
	str	xzr, [x6], #8			// Clear BSS
	b	1b
2:
	adr_l	sp, initial_sp, x4          // adr_l将符号地址转变为运行时地址,即sp=initial_sp
	str_l	x21, __fdt_pointer, x5		// Save FDT pointer
	str_l	x24, memstart_addr, x6		// Save PHYS_OFFSET
	mov	x29, #0
	b	start_kernel
ENDPROC(__mmap_switched)
```

## 参考资料

http://www.wowotech.net/armv8a_arch/__cpu_setup.html

http://www.wowotech.net/armv8a_arch/turn-on-mmu.html
