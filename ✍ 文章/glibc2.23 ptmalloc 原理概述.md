> Linux用户态下的堆溢出利用即是对ptmalloc2安全机制的绕过，只有深入的了解ptmalloc，才能进行精准的堆溢出攻击

## malloc

### __libc_malloc(传入参数:size)

1.判断 **是否定义hook函数** ，如果存在则调用hook函数，否则跳到2

```c
void *(*hook) (size_t, const void *)
    = atomic_forced_read (__malloc_hook);
  if (__builtin_expect (hook != NULL, 0))
    return (*hook)(bytes, RETURN_ADDRESS (0));

```

2.执行arena_get得到空闲分配区

```c
arena_get (ar_ptr, bytes); 
```

3.调用_int_malloc()获得内存地址

```c
victim = _int_malloc (ar_ptr, bytes); 
```

4.如果分配失败，ptmalloc会尝试再去寻找一个可用的arena并进行内存分配

```c
if (!victim && ar_ptr != NULL)
    {
      LIBC_PROBE (memory_malloc_retry, 1, bytes);
      ar_ptr = arena_get_retry (ar_ptr, bytes);
      victim = _int_malloc (ar_ptr, bytes);
    }

```

5.得到内存地址后解锁分配区

```c
if (ar_ptr != NULL)
    (void) mutex_unlock (&ar_ptr->mutex);
```

6.如果在上面的步骤中获得非mmap得到的chunk，但是chunk对应的分配区与实际的分配区不相符，则程序终止

```c
assert (!victim || chunk_is_mmapped (mem2chunk (victim)) ||
          ar_ptr == arena_for_chunk (mem2chunk (victim)));
```

### _int_malloc

- errout:
  - 打印错误字符串并 return

1.判断 **size是否大于(unsigned long)(-2\*MINSIZE)** MINSIZE=chunk的最小大小 ，如果是则return，否则将size转换为chunk_size

```c
#define checked_request2size(req, sz)               \
  if (REQUEST_OUT_OF_RANGE (req)) {				\
      __set_errno (ENOMEM);						   \
      return 0;								      \
    }									      \
  (sz) = request2size (req);
```

2.判断 **前面是否没有获得空闲分配区** 如果是则跳到3，否则跳到4

```c
if (__glibc_unlikely (av == NULL)) 
```

3.调用sysmalloc尝试获得chunk地址，如果得到则调用alloc_perturb 将清理空间数据，return chunk地址

```c
if (__glibc_unlikely (av == NULL))
    {
      void *p = sysmalloc (nb, av);
      if (p != NULL)
	alloc_perturb (p, bytes);
      return p;
    }

```

4.(从fastbin中分配chunk) 判断 **size是否小于global_max_fast** ,如果是跳到5，否则跳到10

```c
if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ())) 
```

5.根据size获得fastbin索引x，定位当前分配区fastbin

```c
if ((unsigned long) (nb) <= (unsigned long) (get_max_fast ()))
    {
    idx = fastbin_index (nb);
    mfastbinptr *fb = &fastbin (av, idx);
    mchunkptr pp = *fb;

```

6.令victim=fastbin ,判断 **victim是否等于NULL** 如果是则跳到7，否则fastbin =fastbin ->fd

```c
do
{
    victim = pp;
    if (victim == NULL)
    break;
}
while ((pp = catomic_compare_and_exchange_val_acq (fb, victim->fd, victim))
             != victim);

```

7.判断 **victim是否不等于0** 如果是跳到8，否则跳到10

8.判断 **victim_size对应的fastbin索引是否不等于size对应的fastbin索引** ，如果是则goto errout **(errstr = “malloc(): memory corruption (fast)")** 否则跳到9

```c
if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0))
    {
        errstr = "malloc(): memory corruption (fast)";
    errout:
        malloc_printerr (check_action, errstr, chunk2mem (victim), av);
        return NULL;
    }

```

9.将victim chunk指针转换为用户数据区指针，调用alloc_perturb()对用户数据区进行初始化，return

```c
void *p = chunk2mem (victim);
alloc_perturb (p, bytes); 
```

10.(从smallbin中分配chunk) 判断 **size是否在smallbin范围(小于MIN_LARGE_SIZE)内** 如果是则跳到11，否则跳到18

```c
if (in_smallbin_range (nb)) 
```

11.根据size获得smallbin的索引x，定位本分配区的smallbin

```c
idx = smallbin_index (nb);
bin = bin_at (av, idx); 
```

12.令victim等于smallbin ->bk,判断 **smallbin ->bk是否不等于smallbin (即smallbin是否不为空)** 如果是则跳到13，否则跳到18

13.判断 **smallbin ->bk是否等于0** 如果是则执行malloc_consolidate进行初始化，否则跳到14

```c
if (victim == 0) /* initialization check */    malloc_consolidate (av); 
```

14.判断 **victim->bk->fd是否不等于victim** 如果是则goto errout **(errstr = “malloc(): smallbin double linked list corrupted”)** 否则跳到15

```c
bck = victim->bk;
if (__glibc_unlikely (bck->fd != victim))
    {
        errstr = "malloc(): smallbin double linked list corrupted";
        goto errout;
    }
```

15.设置victim的下一个chunk 标志位为pre_chunk_inuse,将smallbin ->bk从双链表中摘除(smallbin ->bk＝smallbin ->bk->bk,smallbin ->bk->fd=smallbin )

```c
set_inuse_bit_at_offset (victim, nb);
bin->bk = bck;
bck->fd = bin;
```

16.如果当前分配区不是main_arena,则设置victim的标志位为NON_MAIN_ARENA

```c
if (av != &main_arena)    
victim->size |= NON_MAIN_ARENA; 
```

17.将victim chunk指针转换为用户数据区指针，调用alloc_perturb()对用户数据区进行初始化，return

```c
void *p = chunk2mem (victim); 
alloc_perturb (p, bytes); 
```

18.根据size获得largebin的索引x

```c
idx = largebin_index (nb); 
```

19.判断**当前分配区是否存在fastbin** (通过flag标志位)如果是则执行malloc_consolidate合并fastbin

```c
if (have_fastchunks (av))    
    malloc_consolidate (av); 
```

20.(处理unsortedbin) 令victim等于unsortedbin->bk, (while循环/20-32/ 直至unsortedbin为空) 判断 **unsortedbin->bk是否不等于unsortedbin** ，如果是跳到21，否则跳到32

```c
while ((victim = unsorted_chunks (av)->bk) != unsorted_chunks (av)) 
```

21.判断 **victim_size是否小于等于2\*SIZE_SZ或者victim_size是否大于system_mem** ,如果是则打印错误 **(“malloc(): memory corruption”)** 并退出，否则跳到22

```c
if (__builtin_expect (victim->size <= 2 * SIZE_SZ, 0)
              || __builtin_expect (victim->size > av->system_mem, 0))
    malloc_printerr (check_action, "malloc(): memory corruption",
    chunk2mem (victim), av);

```

22.判断 **size是否在smallbin范围内并且victim->bk是否等于unsortedbin(即unsorted bin 是否只存在一个chunk)并且unsortedbin的当前chunk是否为last_remainder并且victim_size是否大于(nb+MINSIZE)** 如果是则切割chunk并产生last_remainder,否则跳到25

```c
if (in_smallbin_range (nb) &&
              bck == unsorted_chunks (av) &&
              victim == av->last_remainder &&
              (unsigned long) (size) > (unsigned long) (nb + MINSIZE))
{
    /* split and reattach remainder */
    remainder_size = size - nb;
    remainder = chunk_at_offset (victim, nb);
    unsorted_chunks (av)->bk = unsorted_chunks (av)->fd = remainder;
    av->last_remainder = remainder;
```

23.如果切割后的last_remainder_size在smallbin范围内，则将新的last_remainder的fd_nextsize和bk_nextsize清零

```c
if (!in_smallbin_range (remainder_size))
{
    remainder->fd_nextsize = NULL;
    remainder->bk_nextsize = NULL;
}
```

24.设置victim的pre_inuse和main_arena标志位，设置last_remainder的pre_inuse标志位和last_remainder下一个块的pre_inuse标志位，将victim转换为用户数据区地址，调用alloc_perturb()对用户数据区进行初始化，return

```c
set_head (victim, nb | PREV_INUSE |
                        (av != &main_arena ? NON_MAIN_ARENA : 0));
set_head (remainder, remainder_size | PREV_INUSE);
set_foot (remainder, remainder_size);

check_malloced_chunk (av, victim, nb);
void *p = chunk2mem (victim);
alloc_perturb (p, bytes);
```

25.将victim从unsortedbin中移除

```c
unsorted_chunks (av)->bk = bck; 
bck->fd = unsorted_chunks (av); 
```

26.判断 **victim_size是否等于size(即当前unsortedbin是否与请求等大)** 如果是则跳到27，否则跳到28

27.设置victim下一个块的pre_inuse标志位，设置victim的main_arena标志位，将victim转换为用户数据区地址，调用alloc_perturb()对用户数据区进行初始化，return

28.(将unsortedbin放入对应的smallbin或者largebin) 判断 **victim_size是否在smallbin范围内** 如果是则获得victim在smallbin中的索引y及其在smallbin[y]目标放置位的前一个块和后一个块(放在smallbin的头部)然后跳到31，否则跳到29

```c
if (in_smallbin_range (size))
{
    victim_index = smallbin_index (size);
    bck = bin_at (av, victim_index);
    fwd = bck->fd;
}
```

29.获得victim在largebin中的索引y及其前一个块和后一个块(largebin[y]中chunk按大小排列,具体方法见30)

```c
victim_index = largebin_index (size); 
bck = bin_at (av, victim_index); 
fwd = bck->fd; 
```

30.利用循环和fd_nextsize指针寻找大于victim_size的chunk(如果victim_size小于最小的chunk,则不进入循环)

31.设置分配区的binmap，将victim插入目标放置

```c
mark_bin (av, victim_index); 
victim->bk = bck; 
victim->fd = fwd; 
fwd->bk = victim; 
bck->fd = victim; 
```

32.迭代器＋1，当迭代器小于10000时，跳回20

```c
#define MAX_ITERS       10000 
if (++iters >= MAX_ITERS)    
break; 
```

33.(从largebin中分配chunk) 判断 **size是否不在smallbin范围内** 如果是跳到34，否则跳到40

```c
if (!in_smallbin_range (nb)) 
```

34.定位largebin ,令victim = largebin ->fd,判断 **victim是否不等于&largebin (largebin 是否不为空)并且victim_size是否大于等于size** 如果是则跳到35，否则跳到40

```c
if ((victim = first (bin)) != bin &&
    (unsigned long) (victim->size) >= (unsigned long) (nb))

```

35.利用bk_nextsize找到chunk(同步赋值给victim),使得victim_size大于size

```c
victim = victim->bk_nextsize;
while (((unsigned long) (size = chunksize (victim)) <
        (unsigned long) (nb)))
victim = victim->bk_nextsize;

```

36.如果victim不等于largebin ->bk且victim_size等于victim->fd->size，则令victim=victim->fd

```c
if (victim != last (bin) && victim->size == victim->fd->size)
    victim = victim->fd;

```

37.使用unlink取出victim，切割largebin，判断 **remainder_size是否小于MINSIZE** 如果是则只设置victim的下一个块的标志位pre_inuse和main_arena，否则跳到38

```c
remainder_size = size - nb;
unlink (av, victim, bck, fwd);
if (remainder_size < MINSIZE)
{
    set_inuse_bit_at_offset (victim, size);
    if (av != &main_arena)
    victim->size |= NON_MAIN_ARENA;
}

```

38.将剩余的chunk放入unsortedbin 判断 **unsortedbin->fd->bk是否不等于unsortedbin** ，如果是则goto errout **errstr = “malloc(): corrupted unsorted chunks”**

```c
remainder = chunk_at_offset (victim, nb);
/* We cannot assume the unsorted list is empty and therefore
    have to perform a complete insert here.  */
bck = unsorted_chunks (av);
fwd = bck->fd;
if (__glibc_unlikely (fwd->bk != bck))
{
    errstr = "malloc(): corrupted unsorted chunks";
    goto errout;
}

```

39.设置剩余chunk的标志位及bin链表，将victim转换为用户数据区地址，调用alloc_perturb()对用户数据区进行初始化，return

40.遍历largebin寻找满足要求的空闲块(使用binmap和block缩短检索时间)，如果找到则按照上诉方式分配victim，否则跳到41

41.(从top_chunk中分配chunk) 令victim等于top_chunk,判断 **top_chunk_size是否大于等于(size+MINSIZE)** 如果是则切割top_chunk赋值给victim,将victim转换为用户数据区地址，调用alloc_perturb()对用户数据区进行初始化，return 否则跳到42

```c
victim = av->top;
size = chunksize (victim);

if ((unsigned long) (size) >= (unsigned long) (nb + MINSIZE))
{
    remainder_size = size - nb;
    remainder = chunk_at_offset (victim, nb);
    av->top = remainder;
    set_head (victim, nb | PREV_INUSE |
            (av != &main_arena ? NON_MAIN_ARENA : 0));
    set_head (remainder, remainder_size | PREV_INUSE);

    check_malloced_chunk (av, victim, nb);
    void *p = chunk2mem (victim);
    alloc_perturb (p, bytes);
    return p;
}
```

42.判断 **分配区是否存在fastbin** 如果存在则执行malloc_consolidate()，获得size 在bin中的索引，否则跳到43

```c
else if (have_fastchunks (av))
{
    malloc_consolidate (av);
    /* restore original bin index */
    if (in_smallbin_range (nb))
    idx = smallbin_index (nb);
    else
    idx = largebin_index (nb);
}

```

43.使用sysmalloc分配空间，使用alloc_perturb 清空内存，return

```c
else
{
    void *p = sysmalloc (nb, av);
    if (p != NULL)
    alloc_perturb (p, bytes);
    return p;
}
```

### malloc_consolidate

1.如果max_fast不为0，则清空分配区have_fastbin标志，否则执行malloc_init_state进行初始化

```c
if (get_max_fast () != 0) {
    clear_fastchunks(av);
...
else {
    malloc_init_state(av);
    check_malloc_state(av);
  }

```

2.获得unsortedbin，fastbin[0]，fastbin[MAX]地址

```c
maxfb = &fastbin (av, NFASTBINS - 1);
fb = &fastbin (av, 0); 
```

3.遍历fastbin中的所有chunk，对每个chunk做如下操作

4.获得chunk_size,下一个块的地址与next_chunk_size

```c
size = p->size & ~(PREV_INUSE|NON_MAIN_ARENA); nextchunk = chunk_at_offset(p, size); 
nextsize = chunksize(nextchunk); 
```

5.判断 **上一个块是否空闲** 如果是则合并chunk，并使用unlink将上一个块从bin中释放，否则跳到6

```c
if (!prev_inuse(p)) {
    prevsize = p->prev_size;
    size += prevsize;
    p = chunk_at_offset(p, -((long) prevsize));
    unlink(av, p, bck, fwd);
    }
```

6.判断 **下一个块是否不是是top_chunk** 如果是则跳到7，否则跳到9

7.判断 **下一个块是否空闲** 如果是则合并chunk，并使用unlink将下一个块从bin中释放，否则跳到8

```c
nextinuse = inuse_bit_at_offset(nextchunk, nextsize);
if (!nextinuse) {
    size += nextsize;
    unlink(av, nextchunk, bck, fwd);
}
```

8.将chunk插入unsortedbin头部，return

```c
unsorted_bin->fd = p;
first_unsorted->bk = p;

if (!in_smallbin_range (size)) {
    p->fd_nextsize = NULL;
    p->bk_nextsize = NULL;
}

set_head(p, size | PREV_INUSE);
p->bk = unsorted_bin;
p->fd = first_unsorted;
```

9.将chunk与top_chunk合并，return

```c
else {
    size += nextsize;
    set_head(p, size | PREV_INUSE);
    av->top = p;
}
```

## free

### __libc_free(传入参数:待释放指针)

1.判断 **是否定义hook函数** ，如果存在则调用hook函数，否则跳到2

```c
void (*hook) (void *, const void *)
    = atomic_forced_read (__free_hook);
if (__builtin_expect (hook != NULL, 0))
{
    (*hook)(mem, RETURN_ADDRESS (0));
    return;
}
```

2.判断 **传入指针是否为0** ，如果为0(0为无效指针)则return，否则跳到3

```c
if (mem == 0)                              /* free(0) has no effect */
    return;
```

3.将传入指针转换为chunk指针

4.判断 **chunk是否是由mmap分配得到** ，如果是则跳到5，否则跳到8

5.判断 **是否没有禁用动态阈值选项且chunk大小是否大于mmap分配阈值且chunk大小是否小于mmap默认最大分配阈值** ，如果是则跳到6，否则跳到7

```c
if (chunk_is_mmapped (p))                       /* release mmapped memory. */
{
    /* see if the dynamic brk/mmap threshold needs adjusting */
    if (!mp_.no_dyn_threshold
        && p->size > mp_.mmap_threshold
        && p->size <= DEFAULT_MMAP_THRESHOLD_MAX)

```

6.设置mmap分配阈值等于chunk_size，top_chunk收缩阈值等于chunk_size*2

```c
{
    mp_.mmap_threshold = chunksize (p);
    mp_.trim_threshold = 2 * mp_.mmap_threshold;
    LIBC_PROBE (memory_mallopt_free_dyn_thresholds, 2,
        mp_.mmap_threshold, mp_.trim_threshold);
}

```

7.执行munmap_chunk释放chunk，然后return

```c
munmap_chunk (p);
return; 
```

8.获得chunk的分配区，并执行_int_free函数

```c
ar_ptr = arena_for_chunk (p); 
_int_free (ar_ptr, p, 0); 
```

### _int_free(1.分配区指针，2.chunk指针，3.分配区互斥锁)

- errout:

  判断 **分配区是否被锁** ，如果是则解锁分配区

  打印错误字符串并 return

1.获得chunk_size

```c
size = chunksize (p); 
```

2.判断 **chunk指针是否大于-chunk_size(unsigned)或者chunk指针是否不关于MALOOC_ALIGN_MASK字节对齐** 如果是则goto errout **errstr = “free(): invalid pointer”**

```c
if (__builtin_expect ((uintptr_t) p > (uintptr_t) -size, 0)
      || __builtin_expect (misaligned_chunk (p), 0))
{
    errstr = "free(): invalid pointer";
errout:
    if (!have_lock && locked)
    (void) mutex_unlock (&av->mutex);
    malloc_printerr (check_action, errstr, chunk2mem (p), av);
    return;
}

```

3.判断 **chunk_size是否小于MINSIZE或者chunk_size是否不关于MALOOC_ALIGN_MASK字节对齐** 如果是则goto errout **(errstr = “free(): invalid size”)**

```c
if (__glibc_unlikely (size < MINSIZE || !aligned_OK (size)))
{
    errstr = "free(): invalid size";
    goto errout;
}

```

4.(释放到fastbin) 判断 **chunk_size是否小于等于global_max_fast** 如果是则跳到5，否则跳到10

```c
if ((unsigned long)(size) <= (unsigned long)(get_max_fast ()) 
```

5.分配区加锁，判断 **分配区是否加锁或者next_chunk_size是否小于等于2\*sizeof(INTERNAL_SIZE_T)或者next_chunk_size是否大于等于av->system_mem** ，如果是则goto errout **(errstr = “free(): invalid next size (fast)"**

```c
if (have_lock    || ({ assert (locked == 0);        mutex_lock(&av->mutex);        locked = 1;        chunk_at_offset (p, size)->size <= 2 * SIZE_SZ        || chunksize (chunk_at_offset (p, size)) >= av->system_mem;        })) { errstr = "free(): invalid next size (fast)"; goto errout; }
```

6.解锁分配区，设置分配区flag的has_fastbin比特位为1，根据chunk_size寻找对应fastbins的索引x，根据分配区和索引位置找到chunk对应的&fastbin

```c
set_fastchunks(av);
unsigned int idx = fastbin_index(size);
fb = &fastbin (av, idx);
```

7.判断 **chunk是否等于fastbin** 如果是则goto errout **(errstr = “double free or corruption (fasttop)")**

```c
if (__builtin_expect (old == p, 0))
{
    errstr = "double free or corruption (fasttop)";
    goto errout;
}
```

8.获得fastbin ->size对应的fastbin索引y，将chunk插入fastbin头部

```c
old_idx = fastbin_index(chunksize(old));
p->fd = old2 = old;
```

9.判断 **分配区是否加锁且fastbin 是否不为0且x与y是否不相同** 如果是，则goto errout **(errstr = “invalid fastbin entry (free)"** 否则return

```c
if (have_lock && old != NULL && __builtin_expect (old_idx != idx, 0))
{
    errstr = "invalid fastbin entry (free)";
    goto errout;
}
```

10.(释放到unsortedbin) 判断 **chunk是否不是mmap函数分配的** 如果是则跳到11，否则跳到27

11.分配区加锁

```c
else if (!chunk_is_mmapped(p)) {
if (! have_lock) {
    (void)mutex_lock(&av->mutex);
    locked = 1;
}
```

12.判断 **chunk是否是top_chunk** 如果是则goto errout **(errstr = “double free or corruption (top)"**

```c
nextchunk = chunk_at_offset(p, size);
/* Lightweight tests: check whether the block is already the
    top block.  */
if (__glibc_unlikely (p == av->top))
{
    errstr = "double free or corruption (top)";
    goto errout;
}

```

13.判断 **分配区是否可以分配连续空间且next_chunk指针是否大于(top_chunk指针+top_chunk_size)** 如果是则goto errout **(errstr = “double free or corruption (out)"**

```c
if (__builtin_expect (contiguous (av)
    && (char *) nextchunk
    >= ((char *) av->top + chunksize(av->top)), 0))
{
    errstr = "double free or corruption (out)";
    goto errout;
}

```

14.判断 **next_chunk的pre_inuse标志是否为0** 如果是，则goto errout **(errstr = “double free or corruption (!prev)"**

```c
if (__glibc_unlikely (!prev_inuse(nextchunk)))
{
    errstr = "double free or corruption (!prev)";
    goto errout;
}

```

15.判断 **next_chunk_size是否小于2\*sizeof(size_t)或者next_chunk_size是否大于av->system_mem** 如果是则goto errout **(errstr = “free(): invalid next size (normal)"**

```c
nextsize = chunksize(nextchunk);
    if (__builtin_expect (nextchunk->size <= 2 * SIZE_SZ, 0)
	|| __builtin_expect (nextsize >= av->system_mem, 0))
{
    errstr = "free(): invalid next size (normal)";
    goto errout;
}

```

16.判断 **pre_chunk是否是空闲的** 如果是则使用unlink取出pre_chunk，使chunk等于pre_chunk

```c
if (!prev_inuse(p)) {
    prevsize = p->prev_size;
    size += prevsize;
    p = chunk_at_offset(p, -((long) prevsize));
    unlink(av, p, bck, fwd);
}

```

17.判断 **next_chunk是否等于top_chunk** 如果是，则跳到22

18.判断 **next_chunk是否是空闲的** 如果是则使用unlink取出next_chunk,否则清空next_chunk的pre_inuse

```c
if (!nextinuse) {
unlink(av, nextchunk, bck, fwd);
size += nextsize;
} else
clear_inuse_bit_at_offset(nextchunk, 0);

```

19.获得unsorted_bin地址

20.判断 **unsorted_bin->fd->bk是否不等于unsorted_bin** 如果是，则goto errout **(errstr = “free(): corrupted unsorted chunks”**

```c
bck = unsorted_chunks(av);
fwd = bck->fd;
if (__glibc_unlikely (fwd->bk != bck))
{
    errstr = "free(): corrupted unsorted chunks";
    goto errout;
}

```

21.将chunk插入unsortedbin(bk处),如果chunk_size不在small_bin范围则清空chunk的fd_nextsize和bk_nextsize区域，设置chunk的pre_inuse,size，跳到23

22.跳到此处说明，chunk与top_chunk相邻，top_chunk会合并当前chunk

```c
else {
    size += nextsize;
    set_head(p, size | PREV_INUSE);
    av->top = p;
    check_chunk(av, p);
}
 
```

23.判断 **被释放的最终size是否大于FASTBIN_CONSOLIDATION_THRESHOLD(65536UL)** 如果是则触发malloc_consolidate函数合并fastbin

```c
if ((unsigned long)(size) >= FASTBIN_CONSOLIDATION_THRESHOLD) {
    if (have_fastchunks(av))
    malloc_consolidate(av);
    ...
}

```

24.判断 **分配区是否是main_arena** 如果是则当top_chunk_size大于收缩阈值时，执行systrim函数，收缩top_chunk

```c
if (av == &main_arena) {
#ifndef MORECORE_CANNOT_TRIM
	if ((unsigned long)(chunksize(av->top)) >=
	    (unsigned long)(mp_.trim_threshold))
	  systrim(mp_.top_pad, av);
#endif
}

```

25.如果不是则匹配top_chunk对应的子分配区，当子分配区与当前分配区相同时，执行heap_trim，尝试收缩top_chunk

```c
else {
    /* Always try heap_trim(), even if the top chunk is not
    large, because the corresponding heap might go away.  */
    heap_info *heap = heap_for_ptr(top(av));

    assert(heap->ar_ptr == av);
    heap_trim(heap, mp_.top_pad);
}
```

26.解锁分配区，然后return

```c
(void)mutex_unlock(&av->mutex); 
```

27.(通过munmap释放chunk) 执行munmap_chunk释放chunk
