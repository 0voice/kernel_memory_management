## What is Virtual Memory?

**Virtual Memory** is a storage mechanism which offers user an illusion of having a very big main memory. It is done by treating a part of secondary memory as the main memory. In Virtual memory, the user can store processes with a bigger size than the available main memory.

Therefore, instead of loading one long process in the main memory, the OS loads the various parts of more than one process in the main memory. Virtual memory is mostly implemented with demand paging and demand segmentation.

In this Operating system tutorial, you will learn:

- [What is Virtual Memory?](https://www.guru99.com/virtual-memory-in-operating-system.html#1)
- [How Virtual Memory Works?](https://www.guru99.com/virtual-memory-in-operating-system.html#2)
- [What is Demand Paging?](https://www.guru99.com/virtual-memory-in-operating-system.html#3)
- [Types of Page replacement methods](https://www.guru99.com/virtual-memory-in-operating-system.html#4)
- [FIFO Page Replacement](https://www.guru99.com/virtual-memory-in-operating-system.html#5)
- [Optimal Algorithm](https://www.guru99.com/virtual-memory-in-operating-system.html#6)
- [LRU Page Replacement](https://www.guru99.com/virtual-memory-in-operating-system.html#7)
- [Advantages of Virtual Memory](https://www.guru99.com/virtual-memory-in-operating-system.html#8)
- [Disadvantages of Virtual Memory](https://www.guru99.com/virtual-memory-in-operating-system.html#9)

## Why Need Virtual Memory?

Here, are reasons for using virtual memory:

- Whenever your computer doesn’t have space in the physical memory it writes what it needs to remember to the hard disk in a swap file as virtual memory.
- If a computer running Windows needs more memory/RAM, then installed in the system, it uses a small portion of the hard drive for this purpose.

## How Virtual Memory Works?

In the modern world, virtual memory has become quite common these days. It is used whenever some pages require to be loaded in the main memory for the execution, and the memory is not available for those many pages.

So, in that case, instead of preventing pages from entering in the main memory, the OS searches for the RAM space that are minimum used in the recent times or that are not referenced into the secondary memory to make the space for the new pages in the main memory.

Let’s understand virtual memory management with the help of one example.

### For example:

Let’s assume that an OS requires 300 MB of memory to store all the running programs. However, there’s currently only 50 MB of available physical memory stored on the RAM.

- The OS will then set up 250 MB of virtual memory and use a program called the Virtual Memory Manager(VMM) to manage that 250 MB.
- So, in this case, the VMM will create a file on the hard disk that is 250 MB in size to store extra memory that is required.
- The OS will now proceed to address memory as it considers 300 MB of real memory stored in the RAM, even if only 50 MB space is available.
- It is the job of the VMM to manage 300 MB memory even if just 50 MB of real memory space is available.

## What is Demand Paging?

![img](https://www.guru99.com/images/1/011720_0611_VirtualMemo1.png)

A demand paging mechanism is very much similar to a paging system with swapping where processes stored in the secondary memory and pages are loaded only on demand, not in advance.

So, when a context switch occurs, the OS never copy any of the old program’s pages from the disk or any of the new program’s pages into the main memory. Instead, it will start executing the new program after loading the first page and fetches the program’s pages, which are referenced.

During the program execution, if the program references a page that may not be available in the main memory because it was swapped, then the processor considers it as an invalid memory reference. That’s because the page fault and transfers send control back from the program to the OS, which demands to store page back into the memory.

## Types of Page Replacement Methods

Here, are some important Page replacement methods

- FIFO
- Optimal Algorithm
- LRU Page Replacement

## FIFO Page Replacement

FIFO (First-in-first-out) is a simple implementation method. In this method, memory selects the page for a replacement that has been in the virtual address of the memory for the longest time.

### Features:

- Whenever a new page loaded, the page recently comes in the memory is removed. So, it is easy to decide which page requires to be removed as its identification number is always at the FIFO stack.
- The oldest page in the main memory is one that should be selected for replacement first.

## Optimal Algorithm

The optimal page replacement method selects that page for a replacement for which the time to the next reference is the longest.

### Features:

- Optimal algorithm results in the fewest number of page faults. This algorithm is difficult to implement.
- An optimal page-replacement algorithm method has the lowest page-fault rate of all algorithms. This algorithm exists and which should be called MIN or OPT.
- Replace the page which unlike to use for a longer period of time. It only uses the time when a page needs to be used.

## LRU Page Replacement

The full form of LRU is the Least Recently Used page. This method helps OS to find page usage over a short period of time. This algorithm should be implemented by associating a counter with an even- page.

### How does it work?

- Page, which has not been used for the longest time in the main memory, is the one that will be selected for replacement.
- Easy to implement, keep a list, replace pages by looking back into time.

### Features:

- The LRU replacement method has the highest count. This counter is also called aging registers, which specify their age and how much their associated pages should also be referenced.
- The page which hasn’t been used for the longest time in the main memory is the one that should be selected for replacement.
- It also keeps a list and replaces pages by looking back into time.

### Fault rate

Fault rate is a frequency with which a designed system or component fails. It is expressed in failures per unit of time. It is denoted by the Greek letter ? (lambda).

## Advantages of Virtual Memory

Here, are pros/benefits of using Virtual Memory:

- Virtual memory helps to gain speed when only a particular segment of the program is required for the execution of the program.
- It is very helpful in implementing a multiprogramming environment.
- It allows you to run more applications at once.
- It helps you to fit many large programs into smaller programs.
- Common data or code may be shared between memory.
- Process may become even larger than all of the physical memory.
- Data / code should be read from disk whenever required.
- The code can be placed anywhere in physical memory without requiring relocation.
- More processes should be maintained in the main memory, which increases the effective use of CPU.
- Each page is stored on a disk until it is required after that, it will be removed.
- It allows more applications to be run at the same time.
- There is no specific limit on the degree of multiprogramming.
- Large programs should be written, as virtual address space available is more compared to physical memory.

## Disadvantages of Virtual Memory

Here, are drawbacks/cons of using virtual memory:

- Applications may run slower if the system is using virtual memory.
- Likely takes more time to switch between applications.
- Offers lesser hard drive space for your use.
- It reduces system stability.
- It allows larger applications to run in systems that don’t offer enough physical RAM alone to run them.
- It doesn’t offer the same performance as RAM.
- It negatively affects the overall performance of a system.
- Occupy the storage space, which may be used otherwise for long term data storage.

## Summary:

- Virtual Memory is a storage mechanism which offers user an illusion of having a very big main memory.
- Virtual memory is needed whenever your computer doesn’t have space in the physical memory
- A demand paging mechanism is very much similar to a paging system with swapping where processes stored in the secondary memory and pages are loaded only on demand, not in advance.
- Important Page replacement methods are 1) FIFO 2) Optimal Algorithm 3) LRU Page Replacement.
- In FIFO (First-in-first-out) method, memory selects the page for a replacement that has been in the virtual address of the memory for the longest time.
- The optimal page replacement method selects that page for a replacement for which the time to the next reference is the longest.
- LRU method helps OS to find page usage over a short period of time.
- Virtual memory helps to gain speed when only a particular segment of the program is required for the execution of the program.
- Applications may run slower if the system is using virtual memory.
