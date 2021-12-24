## 1. Overview

In this tutorial, we’ll learn about the virtual memory concept in operating systems (OS). **We’ll examine the problems that form the main motivation for the creation of virtual memory.** Finally, we’ll explain the purpose of using this feature in an OS.

## 2. Motivation

Computers are designed to be able to execute many programs, each operating on different amounts of data. Operating systems are expected to exploit better computer resources, guaranteeing fast and efficient processing.

Three main problems cause the processing to be slow in terms of time and consumed memory. Since we store data in bytes in the disk, the CPU must load the needed data into RAM when executing programs.

While being executed, an OS allows a program to use a certain range of addresses from RAM. Suppose this space is ![64](https://www.baeldung.com/wp-content/ql-cache/quicklatex.com-de25eb06760758d2c59eb63fca270413_l3.svg) bits, which means an expected RAM size of ![2^{64} = 16EB](https://www.baeldung.com/wp-content/ql-cache/quicklatex.com-326ee749f48c17de326592fce2b48e6b_l3.svg) (Exabyte) is required here.

Now let’s assume that the OS already reserved a portion of it (![6 GB](https://www.baeldung.com/wp-content/ql-cache/quicklatex.com-397a6edf1fe3a65657f65ce4800c3c86_l3.svg)), but the computer has less memory than the required RAM. **Trying to use addresses that are out of range will crash the computer.**

Furthermore, it’s not feasible for an OS to have the capacity of such a large RAM: ![img](https://www.baeldung.com/wp-content/uploads/sites/4/2021/05/notenough.png)

When executing multiple programs simultaneously, an OS will assign each one of them a continuous partition of RAM, allowing them to be processed at the same time. Now let’s assume that two programs finished their execution. If the space freed up by the two programs is not [continuous](https://en.wikipedia.org/wiki/Continuous_memory), and not enough for other programs to run, the RAM will have holes in different places. **This results in** **[memory fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing)):**

![img](https://www.baeldung.com/wp-content/uploads/sites/4/2021/05/conti.png)

Since many programs are executed simultaneously, more than one program can access the same case of memory. **To change their value, these programs can collide with each other, corrupt the memory, or crash the system.** The figure below shows how data can be easily corrupted when accessed by multiple programs: ![img](https://www.baeldung.com/wp-content/uploads/sites/4/2021/05/security.png)

## 3. Introduction to Virtual Memory

**Virtual Memory is a technique aiming to solve memory’s physical shortages by using the secondary memory so that an OS considers it as a part of the main memory.** Virtual memory is temporary memory. The size of the virtual memory storage depends on the addressing scheme used by an OS and the available secondary memory.

Virtual memory maps program addresses into RAM addresses. If no more space is available, these addresses will be mapped into the disk:

![img](https://www.baeldung.com/wp-content/uploads/sites/4/2021/05/Capture.png)

**The main advantage of virtual memory is that an OS can load programs larger than its physical memory.** It makes an impression to the users that the computer has unlimited memory. It also provides [memory protection](https://en.wikipedia.org/wiki/Memory_protection).

In order to realize the mapping operations, virtual memory needs to use [page tables and translations](https://www.baeldung.com/cs/virtual-memory). Page Tables are a data structure that stores page tables known as page table entry (PTE). Page tables’ goal is to map virtual addresses to physical addresses. It is a contiguous block and the smallest unit of virtual memory.

Creating memory pages consists of partitioning them into [equal-sized frames](https://en.wikipedia.org/wiki/Page_(computer_memory)). Each frame refers to each physical address frame number, a frame offset, or an absolute address.

The translation is the process of transforming a virtual address to a physical address using page tables: ![img](https://www.baeldung.com/wp-content/uploads/sites/4/2021/05/pava.png)

**To make the process of reading data faster, virtual memory can use [cache memory](https://en.wikipedia.org/wiki/CPU_cache).** Since physical caches don’t guarantee a fast process, we use virtual caches. The CPU is directly connected to the cache, and it looks up the virtual addresses in it while avoiding translation. Translation will occur only if an address isn’t found. Eventually, each program has its own virtual cache:

![img](https://www.baeldung.com/wp-content/uploads/sites/4/2021/05/cache-1.png)

## 4. Why Do We Need Virtual Memory?

Virtual memory plays an important role in a modern-day OS. This section will discuss some of the crucial reasons why an OS needs to use virtual memory.

### 4.1. Memory Space Problem

**Virtual memory makes it easy to share code; therefore, we don’t have to keep several copies of the same code.** With virtual memory, different virtual addresses can map to the same location in the physical memory. Consequently, we don’t have to store multiple copies of the same code in the main memory.

RAM is very costly. As a result, the use of a very large RAM is not a feasible solution to alleviate any need for storage allocation. **Assigning a virtual memory for each program and mapping addresses to the disk eliminates space problems.**

In the case of a binary file, when we load them in an OS, each function reserves a fixed address in the main memory. **If virtual memory doesn’t exist, we can’t load more than one program in the main memory.** This means that without virtual memory, we can only run one program at a time. This is because each program might have to use different functions that may point to the same addresses in RAM.

### 4.2. Data Security

**Another important issue is data security.** In general, a program can guess another program’s physical address and gain access to sensitive and secret data.

If an OS uses the virtual memory technique, even if some programs have to access the same address, they all have different mappings. This ensures that they will access different addresses in the RAM or the disk. This is how virtual memory enables data security.

**It also provides position independence to the data.** We can store data at any position in the main memory.

### 4.3. Memory Fragmentation and Errors

**Virtual memory facilitates each program with its own mapping.** The data space won’t have to be continuous, and each program can store data wherever it wants.

**It also facilitates debugging and provides options for checking various features, like unallocated memory and null pointers.**

Let’s assume that two or more applications are running in an OS at the same time. All of these applications use the direct addresses of RAM. If one of the applications results in an error, like [memory error](https://en.wikipedia.org/wiki/Out_of_memory), while running, this could destroy and take down the other applications.

Some physical devices, like video RAM, might reserve some memory addresses prior, depending on the hardware used in the computer. If the OS starts loading programs without knowing the reserved addresses, the OS may read and write to the reserved addresses. This can result in physically breaking the plugged-in devices. Virtual memory helps the OS to avoid these issues.

## 5. Conclusion

In this article, we discussed virtual memory in detail. We started our discussion by describing some of the motivations for using virtual memory in OS. Then we gave a general overview of the virtual memory concept. Finally, we explained why we need virtual memory in OS.
