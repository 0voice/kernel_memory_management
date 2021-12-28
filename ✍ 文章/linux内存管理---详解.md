**主要内容：**

- 内存的管理单元
- 获取内存的方法

- 获取高端内存
- 内核内存的分配方式

- 总结

## 内存的管理单元

内存最基本的管理单元是页，同时按照内存地址的划分，大致分为3个区。

### 页

页的大小与体系结构有关，在 x86 结构中一般是 4KB或者8KB。

可以通过 getconf 命令来查看系统的page的大小：



```c
[wangyubin@localhost  ]$ getconf -a | grep -i 'page'
PAGESIZE                           4096
PAGE_SIZE                          4096
_AVPHYS_PAGES                      637406
_PHYS_PAGES                        2012863
```



以上的 PAGESIZE 就是当前机器页大小，即 4KB



页的结构体头文件是： <linux/mm_types.h> 位置：include/linux/mm_types.h



```c
/*
 * 页中包含的成员非常多，还包含了一些联合体
 * 其中有些字段我暂时还不清楚含义，以后再补上。。。
 */
struct page {
    unsigned long flags;    /* 存放页的状态，各种状态参见<linux/page-flags.h> */
    atomic_t _count;        /* 页的引用计数 */
    union {
        atomic_t _mapcount;    /* 已经映射到mms的pte的个数 */
        struct {        /* 用于slab层 */
            u16 inuse;
            u16 objects;
        };
    };
    union {
        struct {
        unsigned long private;        /* 此page作为私有数据时，指向私有数据 */
        struct address_space *mapping;    /* 此page作为页缓存时，指向关联的address_space */
        };
#if USE_SPLIT_PTLOCKS
        spinlock_t ptl;
#endif
        struct kmem_cache *slab;    /* 指向slab层 */
        struct page *first_page;    /* 尾部复合页中的第一个页 */
    };
    union {
        pgoff_t index;        /* Our offset within mapping. */
        void *freelist;        /* SLUB: freelist req. slab lock */
    };
    struct list_head lru;    /* 将页关联起来的链表项 */
#if defined(WANT_PAGE_VIRTUAL)
    void *virtual;            /* 页的虚拟地址 */
#endif /* WANT_PAGE_VIRTUAL */
#ifdef CONFIG_WANT_PAGE_DEBUG_FLAGS
    unsigned long debug_flags;    /* Use atomic bitops on this */
#endif

#ifdef CONFIG_KMEMCHECK
    /*
     * kmemcheck wants to track the status of each byte in a page; this
     * is a pointer to such a status block. NULL if not tracked.
     */
    void *shadow;
#endif
};
```

物理内存的每个页都有一个对应的 page 结构，看似会在管理上浪费很多内存，其实细细算来并没有多少。

比如上面的page结构体，每个字段都算4个字节的话，总共40多个字节。(union结构只算一个字段)

 

那么对于一个页大小 4KB 的 4G内存来说，一个有 4*1024*1024 / 4 = 1048576 个page，

一个page 算40个字节，在管理内存上共消耗内存 40MB左右。

 

如果页的大小是 8KB 的话，消耗的内存只有 20MB 左右。相对于 4GB 来说并不算很多。

### 区

页是内存管理的最小单元，但是并不是所有的页对于内核都一样。

内核将内存按地址的顺序分成了不同的区，有的硬件只能访问有专门的区。

 

内核中分的区定义在头文件 <linux/mmzone.h> 位置：include/linux/mmzone.h

内存区的种类参见 enum zone_type 中的定义。

 

内存区的结构体定义也在 <linux/mmzone.h> 中。

具体参考其中 struct zone 的定义。

 

其实一般主要关注的区只有3个：

| **区**       | **描述**       | **物理内存** |
| ------------ | -------------- | ------------ |
| ZONE_DMA     | DMA使用的页    | <16MB        |
| ZONE_NORMAL  | 正常可寻址的页 | 16～896MB    |
| ZONE_HIGHMEM | 动态映射的页   | >896MB       |

 

某些硬件只能直接访问内存地址，不支持内存映射，对于这些硬件内核会分配 ZONE_DMA 区的内存。

某些硬件的内存寻址范围很广，比虚拟寻址范围还要大的多，那么就会用到 ZONE_HIGHMEM 区的内存，

对于 ZONE_HIGHMEM 区的内存，后面还会讨论。

对于大部分的内存申请，只要用 ZONE_NORMAL 区的内存即可。

## 获取内存的方法

内核中提供了多种获取内存的方法，了解各种方法的特点，可以恰当的将其用于合适的场景。

### 按页获取内存---最原始的方法，用于底层获取内存的方式

以下分配内存的方法参见：<linux/gfp.h>

| **方法**                          | **描述**                                              |
| --------------------------------- | ----------------------------------------------------- |
| alloc_page(gfp_mask)              | 只分配一页，返回指向页结构的指针                      |
| alloc_pages(gfp_mask, order)      | 分配 2^order 个页，返回指向第一页页结构的指针         |
| __get_free_page(gfp_mask)         | 只分配一页，返回指向其逻辑地址的指针                  |
| __get_free_pages(gfp_mask, order) | 分配 2^order 个页，返回指向第一页逻辑地址的指针       |
| get_zeroed_page(gfp_mask)         | 只分配一页，让其内容填充为0，返回指向其逻辑地址的指针 |

 

**alloc方法和 get方法**的区别在于，一个返回的是**内存的物理地址**，一个返回**内存物理地址映射后的逻辑地址**。

如果**无须直接操作物理页结构体的话，一般使用 get 方法**。

 

相应的释放内存的函数如下：也是在 <linux/gfp.h> 中定义的



```c
extern void __free_pages(struct page *page, unsigned int order);
extern void free_pages(unsigned long addr, unsigned int order);
extern void free_hot_page(struct page *page);
```

在请求内存时，参数中有个 gfp_mask 标志，这个标志是控制分配内存时必须遵守的一些规则。

gfp_mask 标志有3类：(所有的 GFP 标志都在 <linux/gfp.h> 中定义)

1. 行为标志 ：控制分配内存时，分配器的一些行为
2. 区标志  ：控制内存分配在那个区(ZONE_DMA, ZONE_NORMAL, ZONE_HIGHMEM 之类)

1. 类型标志 ：由上面2种标志组合而成的一些常用的场景

 

行为标志主要有以下几种：

| **行为标志**  | **描述**                                                     |
| ------------- | ------------------------------------------------------------ |
| __GFP_WAIT    | 分配器可以睡眠                                               |
| __GFP_HIGH    | 分配器可以访问紧急事件缓冲池                                 |
| __GFP_IO      | 分配器可以启动磁盘I/O                                        |
| __GFP_FS      | 分配器可以启动文件系统I/O                                    |
| __GFP_COLD    | 分配器应该使用高速缓存中快要淘汰出去的页                     |
| __GFP_NOWARN  | 分配器将不打印失败警告                                       |
| __GFP_REPEAT  | 分配器在分配失败时重复进行分配，但是这次分配还存在失败的可能 |
| __GFP_NOFALL  | 分配器将无限的重复进行分配。分配不能失败                     |
| __GFP_NORETRY | 分配器在分配失败时不会重新分配                               |
| __GFP_NO_GROW | 由slab层内部使用                                             |
| __GFP_COMP    | 添加混合页元数据，在 hugetlb 的代码内部使用                  |

区标志主要以下3种：

| **区标志**    | **描述**                                        |
| ------------- | ----------------------------------------------- |
| __GFP_DMA     | 从 ZONE_DMA 分配                                |
| __GFP_DMA32   | 只在 ZONE_DMA32 分配 (**注1**)                  |
| __GFP_HIGHMEM | 从 ZONE_HIGHMEM 或者 ZONE_NORMAL 分配 (**注2**) |

**注1：**ZONE_DMA32 和 ZONE_DMA 类似，该区包含的页也可以进行DMA操作。 

​     唯一不同的地方在于，ZONE_DMA32 区的页只能被32位设备访问。 

**注2：**优先从 ZONE_HIGHMEM 分配，如果 ZONE_HIGHMEM 没有多余的页则从 ZONE_NORMAL 分配。

类型标志是编程中最常用的，在使用标志时，应首先看看类型标志中是否有合适的，如果没有，再去自己组合 行为标志和区标志。

| **类型标志** | **实际标志**                                          | **描述**                                                     |
| ------------ | ----------------------------------------------------- | ------------------------------------------------------------ |
| GFP_ATOMIC   | __GFP_HIGH                                            | 这个标志用在中断处理程序，下半部，持有自旋锁以及其他不能睡眠的地方 |
| GFP_NOWAIT   | 0                                                     | 与 GFP_ATOMIC 类似，不同之处在于，调用不会退给紧急内存池。  这就增加了内存分配失败的可能性 |
| GFP_NOIO     | __GFP_WAIT                                            | 这种分配可以阻塞，但不会启动磁盘I/O。  这个标志在不能引发更多磁盘I/O时能阻塞I/O代码，可能会导致递归 |
| GFP_NOFS     | (__GFP_WAIT ｜ __GFP_IO)                              | 这种分配在必要时可能阻塞，也可能启动磁盘I/O，但不会启动文件系统操作。  这个标志在你不能再启动另一个文件系统的操作时，用在文件系统部分的代码中 |
| GFP_KERNEL   | (__GFP_WAIT ｜ __GFP_IO ｜ __GFP_FS )                 | 这是常规的分配方式，可能会阻塞。这个标志在睡眠安全时用在进程上下文代码中。  为了获得调用者所需的内存，内核会尽力而为。这个标志应当为首选标志 |
| GFP_USER     | (__GFP_WAIT ｜ __GFP_IO ｜ __GFP_FS )                 | 这是常规的分配方式，可能会阻塞。用于为用户空间进程分配内存时 |
| GFP_HIGHUSER | (__GFP_WAIT ｜ __GFP_IO ｜ __GFP_FS )｜__GFP_HIGHMEM) | 从 ZONE_HIGHMEM 进行分配，可能会阻塞。用于为用户空间进程分配内存 |
| GFP_DMA      | __GFP_DMA                                             | 从 ZONE_DMA 进行分配。需要获取能供DMA使用的内存的设备驱动程序使用这个标志  通常与以上的某个标志组合在一起使用。 |

以上各种类型标志的使用场景总结：

| **场景**                      | **相应标志**                                                |
| ----------------------------- | ----------------------------------------------------------- |
| 进程上下文，可以睡眠          | 使用 GFP_KERNEL                                             |
| 进程上下文，不可以睡眠        | 使用 GFP_ATOMIC，在睡眠之前或之后以 GFP_KERNEL 执行内存分配 |
| 中断处理程序                  | 使用 GFP_ATOMIC                                             |
| 软中断                        | 使用 GFP_ATOMIC                                             |
| tasklet                       | 使用 GFP_ATOMIC                                             |
| 需要用于DMA的内存，可以睡眠   | 使用 (GFP_DMA｜GFP_KERNEL)                                  |
| 需要用于DMA的内存，不可以睡眠 | 使用 (GFP_DMA｜GFP_ATOMIC)，或者在睡眠之前执行内存分配      |

### 按字节获取内存---用的最多的获取方法

这种内存分配方法是平时使用比较多的，主要有2种分配方法：kmalloc()和vmalloc()

kmalloc的定义在 <linux/slab_def.h> 中



```c
/**
 * @size  - 申请分配的字节数
 * @flags - 上面讨论的各种 gfp_mask
 */
static __always_inline void *kmalloc(size_t size, gfp_t flags)
#+end_src

vmalloc的定义在 mm/vmalloc.c 中
#+begin_src C
/**
 * @size - 申请分配的字节数
 */
void *vmalloc(unsigned long size)
```

kmalloc 和 vmalloc 区别在于：

- kmalloc 分配的内存物理地址是连续的，虚拟地址也是连续的
- vmalloc 分配的内存物理地址是不连续的，虚拟地址是连续的

 

因此在使用中，用的较多的还是 kmalloc，因为kmalloc 的性能较好。

因为kmalloc的物理地址和虚拟地址之间的映射比较简单，只需要将物理地址的第一页和虚拟地址的第一页关联起来即可。

而vmalloc由于物理地址是不连续的，所以要将物理地址的每一页都和虚拟地址关联起来才行。

 

kmalloc 和 vmalloc 所对应的释放内存的方法分别为：



```c
void kfree(const void *)
void vfree(const void *)
```

### slab层获取---效率最高的获取方法

频繁的分配/释放内存必然导致系统性能的下降，所以有必要为频繁分配/释放的对象建立缓存。

而且，如果能为每个处理器建立专用的高速缓存，还可以避免 SMP锁带来的性能损耗。

#### slab层的实现原理

linux中的高速缓存是用所谓 slab 层来实现的，slab层即内核中管理高速缓存的机制。

整个slab层的原理如下：

1. 可以在内存中建立各种对象的高速缓存(比如进程描述相关的结构 task_struct 的高速缓存)
2. 除了针对特定对象的高速缓存以外，也有通用对象的高速缓存

1. 每个高速缓存中包含多个 slab，slab用于管理缓存的对象
2. slab中包含多个缓存的对象，物理上由一页或多个连续的页组成

 

高速缓存->slab->缓存对象之间的关系如下图：

![img](https://cdn.nlark.com/yuque/0/2019/png/149144/1566527734293-27758476-7840-4114-b831-61950111e1ce.png)

#### slab层的应用

slab结构体的定义参见：mm/slab.c



```c
struct slab {
    struct list_head list;   /* 存放缓存对象，这个链表有 满，部分满，空 3种状态  */
    unsigned long colouroff; /* slab 着色的偏移量 */
    void *s_mem;             /* 在 slab 中的第一个对象 */
    unsigned int inuse;         /* slab 中已分配的对象数 */
    kmem_bufctl_t free;      /* 第一个空闲对象(如果有的话) */
    unsigned short nodeid;   /* 应该是在 NUMA 环境下使用 */
};
```

slab层的应用主要有四个方法：

- 高速缓存的创建
- 从高速缓存中分配对象

- 向高速缓存释放对象
- 高速缓存的销毁



```c
/**
 * 创建高速缓存
 * 参见文件： mm/slab.c
 * 这个函数的注释很详细，这里就不多说了。
 */
struct kmem_cache *
kmem_cache_create (const char *name, size_t size, size_t align,
    unsigned long flags, void (*ctor)(void *))

/**
 * 从高速缓存中分配对象也很简单
 * 函数参见文件：mm/slab.c
 * @cachep - 指向高速缓存指针
 * @flags  - 之前讨论的 gfp_mask 标志，只有在高速缓存中所有slab都没有空闲对象时，
 *           需要申请新的空间时，这个标志才会起作用。
 *
 * 分配成功时，返回指向对象的指针
 */
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)

/**
 * 向高速缓存释放对象
 * @cachep - 指向高速缓存指针
 * @objp   - 要释放的对象的指针
 */
void kmem_cache_free(struct kmem_cache *cachep, void *objp)

/**
 * 销毁高速缓存
 * @cachep - 指向高速缓存指针 
 */
void kmem_cache_destroy(struct kmem_cache *cachep)
```

## 获取高端内存

高端内存就是之前提到的 ZONE_HIGHMEM 区的内存。

在x86体系结构中，这个区的内存不能映射到内核地址空间上，也就是没有逻辑地址，

为了使用 ZONE_HIGHMEM 区的内存，内核提供了永久映射和临时映射2种手段：

### 永久映射

永久映射的函数是可以睡眠的，所以只能用在进程上下文中。



```c
/* 将 ZONE_HIGHMEM 区的一个page永久的映射到内核地址空间
 * 返回值即为这个page对应的逻辑地址
 */
static inline void *kmap(struct page *page)

/* 允许永久映射的数量是有限的，所以不需要高端内存时，应该及时的解除映射 */
static inline void kunmap(struct page *page)
```

### 临时映射

临时映射不会阻塞，也禁止了内核抢占，所以可以用在中断上下文和其他不能重新调度的地方。



```c
/**
 * 将 ZONE_HIGHMEM 区的一个page临时映射到内核地址空间
 * 其中的 km_type 表示映射的目的，
 * enum kn_type 的定义参见：<asm/kmap_types.h>
 */
static inline void *kmap_atomic(struct page *page, enum km_type idx)

/* 相应的解除映射是个宏 */
#define kunmap_atomic(addr, idx)    do { pagefault_enable(); } while (0)
```

## 内核内存的分配方式

内核的内存分配和用户空间的内存分配相比有着更多的限制条件，同时也有着更高的性能要求。

下面讨论2个和用户空间不同的内存分配方式。

### 内核栈上的静态分配

用户空间中一般不用担心栈上的内存不足，也不用担心内存的管理问题(比如内存越界之类的)，

即使出了异常也有内核来保证系统的正常运行。

 

而在内核空间则完全不一样，不仅栈空间有限，而且为了管理的效率和尽量减少问题的发生，

内核栈一般都是小而且固定的。

 

在x86体系结构中，内核栈的大小一般就是1页或2页，即 4KB ~ 8KB

内核栈可以在编译内核时通过配置选项将内核栈配置为1页，

配置为1页的好处是分配时比较简单，只有一页，不存在内存碎片的情况，因为一页是本就是分配的最小单位。

当有中断发生时，如果共享内核栈，中断程序和被中断程序共享一个内核栈会可能导致空间不足，

于是，每个进程除了有个内核栈之外，还有一个中断栈，中断栈一般也就1页大小。

 

查看当前系统内核栈大小的方法：



```c
[xxxxx@localhost ~]$ ulimit -a | grep 'stack'
stack size              (kbytes, -s) 8192
```

### 按照CPU分配

与单CPU环境不同，SMP环境下的并行是真正的并行。单CPU环境是宏观并行，微观串行。

真正并行时，会有更多的并发问题。

 

假定有如下场景：



```c
void* p;

if (p == NULL)
{
/* 对 P 进行相应的操作，最终 P 不是NULL了 */
}
else
{
/* P 不是NULL，继续对 P 进行相应的操作 */
}
```

在上述场景下，可能会有以下的执行流程：

1. 刚开始 p == NULL
2. 线程A 执行到 [if (p == NULL)] ，刚进入 if 内的代码时被线程B 抢占 
    由于线程A 还没有执行 if 内的代码，所以 p 仍然是 NULL

1. 线程B 抢占到CPU后开始执行，执行到 [if (p == NULL)]时， 发现 p 是 NULL，执行 if 内的代码
2. 线程B 执行完后，线程A 重新被调度，继续执行 if 的代码 
    其实此时由于线程B 已经执行完，p 已经不是 NULL了，线程A 可能会破坏线程B 已经完成的处理，导致数据不一致

 

在单CPU环境下，上述情况无需加锁，只需在 if 处理之前禁止内核抢占，在 else 处理之后恢复内核抢占即可。

而在SMP环境下，上述情况必须加锁，因为禁止内核抢占只能禁止当前CPU的抢占，其他的CPU仍然调度线程B 来抢占线程A 的执行

 

SMP环境下加锁过多的话，会严重影响并行的效率，如果是自旋锁的话，还会浪费其他CPU的执行时间。

所以内核中才有了按CPU分配数据的接口。

按CPU分配数据之后，每个CPU自己的数据不会被其他CPU访问，虽然浪费了一点内存，但是会使系统更加的简洁高效。

#### 按照CPU分配的优势

按CPU来分配数据主要有2个优点：

1. 最直接的效果就是减少了对数据的锁，提高了系统的性能
2. 由于每个CPU有自己的数据，所以处理器切换时可以大大减少缓存失效的几率 (***注1**)

 

**注1**：如果一个处理器操作某个数据，而这个数据在另一个处理器的缓存中时，那么存放这个数据的那个

处理器必须清理或刷新自己的缓存。持续的缓存失效成为缓存抖动，对系统性能影响很大。

#### 编译时分配

可以在编译时就定义分配给每个CPU的变量，其分配的接口参见：<linux/percpu-defs.h>



```c
/ 给每个CPU声明一个类型为 type，名称为 name 的变量 /
DECLARE_PER_CPU(type, name)
/ 给每个CPU定义一个类型为 type，名称为 name 的变量 /
DEFINE_PER_CPU(type, name)
注意上面两个宏，一个是声明，一个是定义。
其实也就是 DECLARE_PER_CPU 中多了个 extern 的关键字
```

分配好变量后，就可以在代码中使用这个变量 name 了。



```c
DEFINE_PER_CPU(int, name);      / 为每个CPU定义一个 int 类型的name变量 /
get_cpu_var(name)++;            / 当前处理器上的name变量 +1 /
put_cpu_var(name);              / 完成对name的操作后，激活当前处理器的内核抢占 /
```

通过 get_cpu_var 和 put_cpu_var 的代码，我们可以发现其中有禁止和激活内核抢占的函数。

相关代码在 <linux/percpu.h> 中



```c
#define get_cpu_var(var) (*({                
extern int simple_identifier_##var(void);    
preempt_disable();/ 这句就是禁止当前处理器上的内核抢占 /    
&__get_cpu_var(var); }))
#define put_cpu_var(var) preempt_enable()  / 这句就是激活当前处理器上的内核抢占 /
```

#### 运行时分配

除了像上面那样静态的给每个CPU分配数据，还可以以指针的方式在运行时给每个CPU分配数据。

动态分配参见：<linux/percpu.h>



```c
/* 给每个处理器分配一个 size 字节大小的对象，对象的偏移量是 align */
extern void *__alloc_percpu(size_t size, size_t align);
/* 释放所有处理器上已分配的变量 __pdata */
extern void free_percpu(void *__pdata);

/* 还有一个宏，是按对象类型 type 来给每个CPU分配数据的，
 * 其实本质上还是调用了 __alloc_percpu 函数 */
#define alloc_percpu(type)    (type *)__alloc_percpu(sizeof(type), \
                               __alignof__(type))
```

动态分配的一个使用例子如下：



```c
void *percpu_ptr;
unsigned long *foo;

percpu_ptr = alloc_percpu(unsigned long);
if (!percpu_ptr)
    /* 内存分配错误 */

foo = get_cpu_var(percpu_ptr);
/* 操作foo ... */
put_cpu_var(percpu_ptr);
```

## 总结

在众多的内存分配函数中，如何选择合适的内存分配函数很重要，下面总结了一些选择的原则：

| **应用场景**                    | **分配函数选择**                                             |
| ------------------------------- | ------------------------------------------------------------ |
| 如果需要物理上连续的页          | 选择低级页分配器或者 kmalloc 函数                            |
| 如果kmalloc分配是可以睡眠       | 指定 GFP_KERNEL 标志                                         |
| 如果kmalloc分配是不能睡眠       | 指定 GFP_ATOMIC 标志                                         |
| 如果不需要物理上连续的页        | vmalloc 函数 (vmalloc 的性能不如 kmalloc)                    |
| 如果需要高端内存                | alloc_pages 函数获取 page 的地址，在用 kmap 之类的函数进行映射 |
| 如果频繁撤销/创建教导的数据结构 | 建立slab高速缓存                                             |



## 分页内存管理

### 内存管理使用的方法---分页内存管理

将虚拟内存空间和物理内存空间皆划分为大小相同的页面，例如4KB、8KB和16KB等。并将**页作为内存空间的最小分配单位**，一个程序的一个页面(虚拟页面)可以存放在任何一个物理页面中。

一个程序发出的虚拟地址由**虚拟页面号**和**页内偏移值**两部分组成，组成见下： 

![img](https://cdn.nlark.com/yuque/0/2019/png/149144/1566539779520-6aa0041a-9c7d-4644-8bd0-096221ef53d5.png)

### 逻辑地址

逻辑地址（Logic Address）是指由程序产生的与段相关的偏移地址部分。比如，在C程序中，可以使用&操作读取指针变量本身的值，实际上这个值就是逻辑地址。它是相对于你当前进程数据段的地址，和绝对物理地址不相干。只有在intel实模式下，逻辑地址才等同于物理地址。应用程序员仅需要和逻辑地址打交道，分段和分页机制对他们来说是透明的，仅由系统编程人员涉及。编译好的程序的入口地址可以看作是首地址，逻辑地址通常可以认为是编译器为我们分配好的相对于这个首地址的偏移。一个逻辑地址由段标识符和段内偏移量组成。**总之，逻辑地址是相对于应用程序而言的。** 逻辑地址有时也称虚拟地址。

### 线性地址

线性地址（Linear Address）是**逻辑地址到物理地址变换之间的中间层**。程序代码会产生逻辑地址，或说是**段中的偏移地址，加上相应段的基地址就生成了一个线性地址**。如果启用了分页机制，那么线性地址能再经变换以产生一个物理地址。若没有启用分页机制，那么线性地址直接就是物理地址。Intel 80386的线性地址空间容量为4G（2的32次方即32根地址总线寻址）。 需要注意的是，线性地址是抽象的，对于同一平台是大小固定的，比如32位地址总线，线性地址空间为4GB，对于每个进程都是这样。也就是说线性地址空间不是进程共享的，而是进程隔离的，每个进程都有相同大小的线性空间，一个进程对某一线性地址的访问与其他进程对同一线性地址的访问不冲突，得到的值也不尽相同。对于CPU来说，某一时刻只有一个进程在运行，这个进程就认为自己独占4GB的线性空间。当进程切换的时候，线性空间也随之切换。尽管线性空间的大小和内存大小之间没有关系，但是任何线性地址最后还是要转换为物理地址才能被CPU使用去访问物理内存。基于分页机制，4GB的线性地址一部分被映射到物理内存，一部分被映射到磁盘上的交换文件，一部分什么也没有映射。 逻辑地址到线性地址的转换如下图：

![img](https://cdn.nlark.com/yuque/0/2019/png/149144/1566540395214-4aaddde6-85c3-4319-a2a7-8f59daa3e7b9.png)

### 物理地址

物理地址（Physical Address）是指出目前CPU外部地址总线上的寻址物理内存的地址信号，是地址变换的最终结果地址。如果启用了分页机制，那么**线性地址会使用页目录和页表中的项变换成物理地址**。如果没有启用分页机制，那么线性地址就直接成为物理地址了。

### 分段机制

分段提供了**隔绝各个代码段/数据段和堆栈段的机制**，因此多个任务可以运行在同一处理器上而不互相干扰。从逻辑地址（虚拟地址）到线性地址经过了分段处理。分断机制来源于早期的intel 8086处理器，早期的处理器工艺无法在一个处理器上面封装40引脚，8086处理器地址总线宽度为20位，而寄存器的宽度只有16位。20位可寻址空间为1MB，而16位的寄存器无法一次读取20位的地址，这就产生了矛盾。为了解决这一矛盾，使CPU能够寻址1MB空间，这就引入了分段机制。将段寄存器的地址左移4位，加上16位的偏移就形成了一个20位的地址。 **在Linux中没有分段管理，逻辑地址就是线性地址。**

### 分页机制

分页时CPU提供的一种机制，Linux只是根据这种机制的规则，利用它实现了内存管理。分页的基本原理 **把内存划分称为大小固定的若干单元** ，每个单元称之为一页，每页包含4k字节的地址空间。这样每一页的起始地址都是4k字节对齐的。为了能转换成物理地址，我们需要给CPU提供当前任务的线性地址转物理地址的查找表，即页表(page table)。**注意，为了实现每个任务的平坦的虚拟内存，每个任务都有自己的页目录表和页表**。


为了节约页表占用的内存空间，x86将线性地址通过页目录表和页表两级查找转换成物理地址。

32位的线性地址被分成3个部分：

最高10位 Directory **页目录表偏移量**，中间10位 Table是**页表偏移量**，最低12位Offset是物理页内的**字节偏移量**。

页目录表的大小为4k（刚好是一个页的大小），包含1024项，每个项4字节（32位）叫做**页目录项**，页目录项里存储的内容就是页表的物理地址。如果页目录表中的页表尚未分配，则物理地址填0。

页表的大小也是4k，同样包含1024项，每个项4字节叫做**页表项**，页表项的内容为最终物理页的物理内存起始地址。

最低的12位表示物理页内的偏移。

对于一个要转换成物理地址的虚拟地址，CPU首先根据CR3中的值，找到页目录所在的物理页。然后根据虚拟地址的第22位到第31位这10位（最高的10bit)的值作为索引，找到相应的页目录项(PDE,page directory entry),页目录项中有这个虚拟地址所对应页表的物理地址。有了页表的物理地址，根据虚拟地址的第12位到第21位这10位的值作为索引，找到该页表中相应的页表项(PTE,page table entry),页表项中就有这个虚拟地址所对应物理页的物理地址。最后用虚拟地址的最低12位，也就是页内偏移，加上这个物理页的物理地址，就得到了该虚拟地址所对应的物理地址。

![img](https://cdn.nlark.com/yuque/0/2019/png/149144/1566541235948-c35557f4-c4a4-46b4-a80b-f3a8c883c812.png)

下面是一个例子：

![img](https://cdn.nlark.com/yuque/0/2019/png/149144/1566541302151-7ecd7d20-8607-4d78-bfc1-8218f9309542.png)

参考资料：

[逻辑地址、线性地址和物理地址](https://vosamo.github.io/2016/01/VA2PA/)
