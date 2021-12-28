- c 内存管理 
  - Glibc 的 ptmalloc 
    - unsorted bin 
    - small bin 
    - large bin 
    - Fast Bins 
    - mmaped chunk 
  - Google 的 tcmalloc 
    - small object 的分配 
    - large object 的分配 
    - span 
    - 对象释放和重分配 
    - central free list 
    - GC 

## c 内存管理

有了上述的基础理论知识，我们现在继续讨论 malloc / free 这一组函数。

既然 malloc 是对系统调用的封装，那么每次 malloc 都会经过系统调用去申请内存吗？

一次系统调用的过程就是，保存当前进程用户态的调用栈，然后切换到内核态，调用完毕再切换回用户态，每一次系统调用需要保存调用栈，恢复调用栈。如果有大量的系统调用，将耗费大量的CPU资源在内核态和用户态的切换上，明显降低CPU的利用率。

![img](https://cdn.nlark.com/yuque/0/2020/png/1328939/1590835369128-3ec385a8-8ff1-4359-8e60-9a239328acc9.png)

为了降低系统调用的频率，我们可以选择将申请的内存缓存起来，每次调用多申请一些内存，释放也不立刻还给操作系统，而是等待下次申请再次复用。

针对这种情况，衍生出了很多的内存管理方案，包括linux 自带的glibc 中的 ptmalloc ， Google 的 tcmalloc ，FaceBook 的 jemalloc。

### Glibc 的 ptmalloc

ptmalloc 的内存分配器中在最早的版本时一个进程只有一个主分配区，在多线程的情况下，锁的争用非常激烈，严重影响了 malloc 的分配效率，于是ptmalloc根据系统对分配区的争用情况动态增加非主分配区的数量，分配区的数量一旦增加，就不会再减少。1个进程的内存分配区域包括 1个主分配区+N个非主分配区。

主分配区可以访问进程的heap区域和 mmap 映射区域，也就是主分配区可以使用 brk 和 mmap 申请内存。

而非主分配区只能访问的 mmap映射区域，非主分配区每次使用mmap()向操作系统“批发”HEAP_MAX_SIZE（32位系统上默认为1MB，64位系统默认为64MB）大小的虚拟内存，当用户向非主分配区请求分配内存时再切割成小块“零售”出去。

当某一线程需要调用malloc()分配内存空间时，该线程先查看线程私有变量中是否已经存在一个分配区，如果存在，尝试对该分配区加锁，如果加锁成功，使用该分配区分配内存，如果失败，该线程搜索循环链表试图获得一个没有加锁的分配区。如果所有的分配区都已经加锁，那么malloc()会开辟一个新的分配区，把该分配区加入到全局分配区循环链表并加锁，然后使用该分配区进行分配内存操作。在释放操作中，线程同样试图获得待释放内存块所在分配区的锁，如果该分配区正在被别的线程使用，则需要等待直到其他线程释放该分配区的互斥锁之后才可以进行释放操作。

申请小块内存时会产生很多内存碎片，ptmalloc在整理时也需要对分配区做加锁操作。每个加锁操作大概需要5～10个cpu指令，而且程序线程很多的情况下，锁等待的时间就会延长，导致malloc性能下降。一次加锁操作需要消耗100ns左右，正是锁的缘故，导致ptmalloc在多线程竞争情况下性能远远落后于tcmalloc。最新版的ptmalloc对锁进行了优化，加入了PER_THREAD和ATOMIC_FASTBINS优化，但默认编译不会启用该优化，这两个对锁的优化应该能够提升多线程内存的分配的效率。

ptmalloc 管理内存不是以页为单位的，而是 chunk。chunk 以双向链表的形式组织。

![img](https://cdn.nlark.com/yuque/0/2020/png/1328939/1590835369231-b74d8e9a-f32c-41b7-b1d6-99889d62922a.png)

一个 chunk 有3个标志位，A | M | P

- P : 前一个块是否可用，0 表示前一个块空闲可用，此时 prev_size 才有效。通过 prev_size， 程序可以计算出前一个块的起始地址。
- M ： 表示内存是从 mmap 申请的还是从 heap 申请的

- A ：表示该chunk属于主分配区或者非主分配区

![img](https://cdn.nlark.com/yuque/0/2020/png/1328939/1590835369269-9a8ab968-5f0a-4a59-8c8d-59f62027feda.png)

#### unsorted bin

数组中的第一个为unsorted bin。Unsorted bin可以看作是small bins和large bins的cache，Unsorted bin 中的空闲chunk不排序，分配时，如果在unsorted bin中没有合适的chunk，就会把unsorted bin中的所有chunk分别加入到所属的bin中，然后再在bin中分配合适的chunk。 如果被用户释放的chunk大于max_fast，或者fast bins中的空闲chunk合并后，这些chunk首先会被放到unsorted bin队列中。

#### small bin

数组中从2开始编号的前64个bin称为 small bins，同一个small bin中的chunk具有相同的大小。两个相邻的small bin中的chunk大小相差8bytes。small bins中的chunk按照最近使用顺序进行排列，最后释放的chunk被链接到链表的头部，而申请chunk是从链表尾部开始，也就是最近不使用的优先分配，这样每一个chunk 都有相同的机会被ptmalloc选中。

#### large bin

Small bins后面的bin被称作 large bins。large bins中的每一个bin分别包含了一个给定范围内的chunk，其中的chunk按大小序排列。相同大小的chunk同样按照最近使用顺序排列。ptmalloc使用“smallest-first，best-fit”原则在空闲large bins中查找合适的chunk。

#### Fast Bins

一般的情况是，程序在运行时会经常需要申请和释放一些较小的内存空间。当分配器合并了相邻的几个小的chunk之后，也许马上就会有另一个小块内存的请求，这样分配器又需要从大的空闲内存中切分出一块，这样无疑是比较低效的，故而，ptmalloc中在分配过程中引入了fast bins，不大于max_fast （默认值为64B）的chunk被释放后，首先会被放到 fast bins 中，fast bins中的chunk并不改变它的使用标志P。这样也就无法将它们合并，当需要给用户分配的chunk小于或等于max_fast时，ptmalloc首先会在fast bins中查找相应的空闲块，然后才会去查找bins中的空闲chunk。在某个特定的时候，ptmalloc会遍历fast bins中的chunk，将相邻的空闲chunk进行合并，并将合并后的chunk加入unsorted bin中，然后再将usorted bin里的chunk加入bins中。

fast bins 相当于 small bins 的一个缓冲区。

#### mmaped chunk

当需要分配的chunk足够大，而且fast bins和bins都不能满足要求，甚至top chunk本身也不能满足分配需求时，ptmalloc会使用mmap来直接使用内存映射来将页映射到进程空间。这样分配的chunk在被free时将直接解除映射，于是就将内存归还给了操作系统，再次对这样的内存区的引用将导致segmentation fault错误。这样的chunk也不会包含在任何bin中。

从分配内存的大小来看：

unsorted bin  是 bins 的缓冲区

fast Bins 是 small bins 的缓冲区

small bins < large bins < top chunk < mmaped chunk

small bins 和 large bins  的分界线是 64B

large bins 和 top chunk  的分界线是 512B

top chunk 和 mmaped chunk  的分界线是 128KB

分配算法概述，以32系统为例，64位系统类似。

- 小于等于64字节：用pool算法，最近不使用的优先分配。也就是从 small bins 中分配。
- 64到512字节之间：从 large bins 分配，在最佳匹配算法分配和pool算法分配中取一种合适的。也就是

- 大于等于512字节：用最佳匹配算法分配。
- 大于等于mmap分配阈值（默认值128KB）：根据设置的mmap的分配策略进行分配，如果没有开启mmap分配阈值的动态调整机制，大于等于128KB就直接调用mmap

下面简单总结下 malloc 的过程。

一个进程启动之后，

1. 第一次分配内存时，一般情况下只存在一个主分配区，但也有可能从父进程那里继承来了多个非主分配区，在这里主要讨论主分配区的情况。brk值等于start_brk，所以实际上heap大小为0，top chunk大小也是0。这时，如果不增加heap大小，就不能满足任何分配要求。所以，若用户的请求的内存大小小于mmap分配阈值，则ptmalloc会初始heap。然后在heap中分配空间给用户，以后的分配就基于这个heap进行。若第一次用户的请求就大于mmap分配阈值，则ptmalloc直接使用mmap()分配一块内存给用户，而heap也就没有被初始化，直到用户第一次请求小于mmap分配阈值的内存分配。
2. 非第一次分配，ptmalloc

1. 1. 首先会查找fast bins，
   2. 如果不能找到匹配的 chunk，则查找small bins。

1. 1. 若还是不行，合并fast bins，把chunk加入unsorted bin，在unsorted bin中查找，
   2. 若还是不行，把unsorted bin中的chunk全加入large bins中，并查找large bins。

1. 1. 在fast bins和small bins中的查找都需要精确匹配，而在large bins中查找时，则遵循“smallest-first，best-fit”的原则，不需要精确匹配。
   2. 若以上方法都失败了，则ptmalloc会考虑使用top chunk。

1. 1. 若top chunk也不能满足分配要求。而且所需chunk大小大于mmap分配阈值，则使用mmap进行分配。否则增加heap，增大top chunk。以满足分配要求。

free 的过程：

ptmalloc的内存收缩是从top chunk开始，如果与top chunk相邻的那个chunk在我们NoSql的内存池中没有释放，top chunk以下的空闲内存都无法返回给系统，即使这些空闲内存有几十个G也不行。

### Google 的 tcmalloc

google 的 tcmalloc 比 ptmalloc 的设计就简单多了。

为了优化多线程对锁的争用，整个内存分配区分为 thread cache 和 central heap

同时将内存对象分为大对象和小对象，以32K为界。

![img](https://cdn.nlark.com/yuque/0/2020/png/1328939/1590835369301-d5868103-c049-4ae1-a748-ca023bbdca6d.png)

#### small object 的分配

每个线程都有私有的thread cache ，当申请小块内存的时候，会优先分配 thread cache  中的内存，thread cache 的内存以数组+链表的形式组织起来。

如下图每一个class 表示一个定长大小的内存块。8字节，16字节，32字节，跟 ptmalloc 里的 small bins 差不多。

![img](https://cdn.nlark.com/yuque/0/2020/png/1328939/1590835369332-62cae195-75e4-4da9-a8f1-061b85cfcfec.png)

小内存（<=32K）的分配过程如下：

1. 先将需要申请的内存适配到合适的 class 中，比如 7字节的内存申请，将从 class=8字节的链表上申请。
2. 如果链表不为空，返回链条头部的内存块。

1. 如果链表空，从 central heap 中批发的一块内存，然后切成当前 class 大小的小内存，放到这个class对应的链表中，然后返回最近使用的一块内存，也就是链表头。
2. 如果 central heap 也是空的，从central page allocator. 那里分配一块内存，然后切成当前 class 的大小，放到 central heap 里，再按照步骤3，给 thread local 分配内存。

#### large object 的分配

central heap 中大内存的组织形式以 page 为单位，1page=4K，类似上面的 small bins 和 class 的形式，也是数组+链表。只是链表的元素变成了 page* K 大小的页。数组的大小是 256， 每个 entry 是 kPages大小的链表入口。K<=255。

![img](https://cdn.nlark.com/yuque/0/2020/png/1328939/1590835369365-6de54c77-bb0d-47a5-855d-233cb395f7b0.png)

大内存（>32K）的分配过程如下：

大于 32 K的内存分配，最终都会适配成 4K 的倍数来分配，比如需要5K，就会分配8K。

1. 先将需要申请的内存适配到合适的 class 中，先找到适配的链表。
2. 如果链表不为空，返回链表头。

1. 如果适配的pages 所在的链表为空，继续遍历下一个链表，比如8K的链表为空，此时遍历16K的链表去找。
2. 如果找到了，比如16K的，此时剩下的内存就会插到合适的链表中，对于5K来说，需要返回对方8K，然后将剩余的8K插入8K的链表中。

1. 如果最终 255K 的链表也没有内存，就要从操作系统申请内存了， 用 sbrk,brk,mmap 之类的方式。

#### span

tcmalloc 的heap管理是以 page 为单位的，为了管理这些 page，引入了 span 的概念。span 是一组 **连续的** page，类似于操作系统底层用多级页表来管理的方式一样。上文中的 kPage 的数组，kPage 就是大小不同的 span。span 可以被分配也可以被释放。

span被释放时，就会变成多个page，插入到上述的 central heap 的链表中。

span被分配时，要么是一块大内存分配给了应用层，要么是被用于切成小块的pages 插入到 central Heap 中。

由于 span 是一组 **连续的** page，因此span 和 page 的关联通过一个 central array 来记录。给所有的 page 编号，central array 就是整个pages 数组，如下图，span  a 和 2个page 关联，span b 和一个page 关联，span c  和 5 个page 关联。

![img](https://cdn.nlark.com/yuque/0/2020/png/1328939/1590835369399-545ee475-ef5c-451d-a153-e87fdf92450f.png)

一个32位的系统有 2^20 个 4K的 pages。因此这个 central array 的大小是 4MB 。还记得操作系统中用于虚拟地址到物理地址映射的页表的大小吗？如果不使用多级页表，页表大小也是 4K，这里的 central array 与页表异曲同工。

当然在 64 位系统上，会使用 3 层的基数树来管理 pages，此时的虚拟内存空间已经达到 256 TB。

#### 对象释放和重分配

一个对象释放的过程：

1. 先计算 page 的编号，这个应该可以根据地址值算出。
2. 然后根据 central array 找到这个 page 对应的 span, span 中包含了该对象的大小信息，是 small object 还是 large object。

1. 如果是 small object , span 会有对应的 size-class 信息，那就把它插入到 thread cache 的 size-class 中。当然 thread cache 的大小上限是 2MB，如果超过这个值，会启动GC机制，将无用的 object 移到 central free list 中。
2. 如果是 large object ，span 中会有这个 object 所包含的 pages numbers 范围，比如 [p,q]，此时会在 central array  的 p-1 and q+1 范围内也找依照有没有一样是 free 的pages ,有的话，全部合并，归还到 central heap 中。

#### central free list

上文提到了的 small object 会还到 central free list 中。

thread cache 中的 每一个 size-class 都会对应一个 central free list。这个 central free list 是所有 thread cache 共用的。每个 central free list 保存着一组 spans，每个 span 下有一个链表，保存着 free objects。

当 object 需要重新分配的时候，可以直接从 central free list 的某个span上分配一个 free object.。如果所有span 的链表都是空的，就从 central Heap 分配了。

当一个 object 被释放到 central free list 的时候，会添加到对应的 span的list 中。如果链表长度跟 span 里的small object 数一样，这个 span 就会完全被释放并放回到 central heap 中。

#### GC

thread cache 达到 2MB 时会触发GC。

GC时会遍历所有的 central free list, 并把一些 objects 从 central free list 移到 central list。

被移动的 objects 数取决于一个参数 L (per-list low-water-mark)。

L 记录了从上次GC开始，central free lists 的最小长度。L 可以作为历史记录来预测将来可能使用的 list 长度。需要移到 central free list 的 object 数 =  L/2 。

这个算法有一个很好的特性，当一个线程停止的时候，所占用的内存是某个值的时候，所有这个大小的 objects 都会被移到 central free list，给其他的线程复用.
