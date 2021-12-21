# 💻 总结整理linux内核的内存管理的资料，包含论文，文章，视频，以及应用程序的内存泄露，内存池相关

![内存管理](https://user-images.githubusercontent.com/87457873/146741232-cc01d5bc-595f-4bd7-a285-b0388ff29027.png)



## 📜 文章



[内存管理（一）：硬件原理 和 分页管理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E7%A1%AC%E4%BB%B6%E5%8E%9F%E7%90%86%20%E5%92%8C%20%E5%88%86%E9%A1%B5%E7%AE%A1%E7%90%86.md)

[内存管理（二）：内存的动态申请和释放](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E5%86%85%E5%AD%98%E7%9A%84%E5%8A%A8%E6%80%81%E7%94%B3%E8%AF%B7%E5%92%8C%E9%87%8A%E6%94%BE.md)

[内存管理（三）：进程的内存消耗和泄漏](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E8%BF%9B%E7%A8%8B%E7%9A%84%E5%86%85%E5%AD%98%E6%B6%88%E8%80%97%E5%92%8C%E6%B3%84%E6%BC%8F.md)

[内存管理（四）：内存与IO的交换](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%9A%E5%86%85%E5%AD%98%E4%B8%8EIO%E7%9A%84%E4%BA%A4%E6%8D%A2.md)

[内存管理（五）：其他工程问题以及调优](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%9A%E5%85%B6%E4%BB%96%E5%B7%A5%E7%A8%8B%E9%97%AE%E9%A2%98%E4%BB%A5%E5%8F%8A%E8%B0%83%E4%BC%98.md)

[Linux 内核(5.4.81)—内存管理模块源码分析](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/linux%20%E5%86%85%E6%A0%B8(5.4.81)%E2%80%94%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%A8%A1%E5%9D%97%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md)

[glibc2.23 ptmalloc 原理概述](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/glibc2.23%20ptmalloc%20%E5%8E%9F%E7%90%86%E6%A6%82%E8%BF%B0.md)

[多核心Linux内核路径优化的不二法门之-slab与伙伴系统](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%A4%9A%E6%A0%B8%E5%BF%83Linux%E5%86%85%E6%A0%B8%E8%B7%AF%E5%BE%84%E4%BC%98%E5%8C%96%E7%9A%84%E4%B8%8D%E4%BA%8C%E6%B3%95%E9%97%A8%E4%B9%8B-slab%E4%B8%8E%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F.md)

[尽情阅读，技术进阶，详解mmap原理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%B0%BD%E6%83%85%E9%98%85%E8%AF%BB%EF%BC%8C%E6%8A%80%E6%9C%AF%E8%BF%9B%E9%98%B6%EF%BC%8C%E8%AF%A6%E8%A7%A3mmap%E5%8E%9F%E7%90%86.md)

[浅谈Linux内存管理机制](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E6%B5%85%E8%B0%88Linux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6.md)

[Linux中的内存管理机制](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Linux%E4%B8%AD%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6.md)

[C++中内存管理之new、delete](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/C%2B%2B%E4%B8%AD%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8Bnew%E3%80%81delete.md)

[malloc和free的实现原理解析](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/malloc%E5%92%8Cfree%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90.md)

[常用寄存器总结](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%B8%B8%E7%94%A8%E5%AF%84%E5%AD%98%E5%99%A8%E6%80%BB%E7%BB%93.md)

[内存碎片之外部碎片与内部碎片](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%A2%8E%E7%89%87%E4%B9%8B%E5%A4%96%E9%83%A8%E7%A2%8E%E7%89%87%E4%B8%8E%E5%86%85%E9%83%A8%E7%A2%8E%E7%89%87.md)

[Linux虚拟内存管理，MMU机制，原来如此](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Linux%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%8CMMU%E6%9C%BA%E5%88%B6%EF%BC%8C%E5%8E%9F%E6%9D%A5%E5%A6%82%E6%AD%A4.md)

[一文了解，Linux内存管理，malloc、free 实现原理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E4%B8%80%E6%96%87%E4%BA%86%E8%A7%A3%EF%BC%8CLinux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%8Cmalloc%E3%80%81free%20%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md)

[内存管理之内存映射](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8B%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84.md)

[内存管理之分页](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8B%E5%88%86%E9%A1%B5.md)

[内存管理之内核空间和用户空间](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8B%E5%86%85%E6%A0%B8%E7%A9%BA%E9%97%B4%E5%92%8C%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4.md)

[Linux 内存占用分析的几个方法，你知道几个？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Linux%20%E5%86%85%E5%AD%98%E5%8D%A0%E7%94%A8%E5%88%86%E6%9E%90%E7%9A%84%E5%87%A0%E4%B8%AA%E6%96%B9%E6%B3%95%EF%BC%8C%E4%BD%A0%E7%9F%A5%E9%81%93%E5%87%A0%E4%B8%AA%EF%BC%9F.md)

[深入理解 Linux 内存子系统](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20Linux%20%E5%86%85%E5%AD%98%E5%AD%90%E7%B3%BB%E7%BB%9F.md)

[深入理解 glibc malloc：内存分配器实现原理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20glibc%20malloc%EF%BC%9A%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E5%99%A8%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md)

[图解 Linux 内存性能优化核心思想](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%9B%BE%E8%A7%A3%20Linux%20%E5%86%85%E5%AD%98%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E6%A0%B8%E5%BF%83%E6%80%9D%E6%83%B3.md)


## 📀 视频

[Linux内核源码/内存调优/文件系统/进程管理/设备驱动/网络协议栈](https://ke.qq.com/course/4032547?tuin=c47fb40b&taid=12394910548068387)

[内存管理 ---Slab | 内存映射 | kmalloc | vmalloc | 内核源码 | MM | brk](https://www.bilibili.com/video/BV1fb4y1S7UY/)

[90分钟了解 Linux内存架构--- numa的优势 | slab的实现 | vmalloc的原理](https://www.bilibili.com/video/BV1C64y1v7WW/)

[内存分配与回收机制---伙伴算法|slab分析|内存映射|进程虚拟空间|请求调页|写时复制](https://www.bilibili.com/video/BV1mi4y1A7Gr/)

[3种内存泄漏的解决方案--hook|malloc函数|free函数|避免内存泄漏](https://www.bilibili.com/video/BV11K4y1R7xz/)

[剖析Linux内核MMU机制---页表原理|高速缓存|TLB工作原理|内存映射|不连续页原理](https://www.bilibili.com/video/BV1xy4y1W7a6/)

[虚拟内存空间之VMA实战操作](https://www.bilibili.com/video/BV1i44y1r7yL/)

[Linux内核内存管理(一)---内存映射|空间管理|ARM32/64页表|slab分配器|malloc](https://www.bilibili.com/video/BV1EQ4y1d76X/)

[Linux内核内存管理(二)---malloc|mmap|反向映射|缺页中断处理|回收页面|KSM实现|内存漏洞|匿名页面](https://www.bilibili.com/video/BV1Qy4y1g7mY/)

[Linux内核内存管理(三)---Slab机制架构|物理页面|管理区|分配/释放页面](https://www.bilibili.com/video/BV1NU4y1j7HR/)

[Linux内核之内存页回收---LRU及反向映射？如何异步回收、直接回收？以及回收slab缓存](https://www.bilibili.com/video/BV1Df4y1b7BM/)

[Linux内核内存管理专题训练营（一）---伙伴系统|slab分配器|vmalloc()|malloc()|TLB|虚拟内存|缺页机制](https://www.bilibili.com/video/BV1Z64y1671b/)

[Linux内核内存管理专题训练营（二）---伙伴系统|slab分配器|vmalloc()|malloc()|TLB|虚拟内存|缺页机制](https://www.bilibili.com/video/BV1354y1G7E9/)

[Linux内核精讲之内存管理---物理内存组织|内核引导|内存映射](https://www.bilibili.com/video/BV1MK4y1M7LR/)

[Linux物理内存页面分配---kmalloc|slab/slub|页框分配机制](https://www.bilibili.com/video/BV1QM4y1K75H/)

## ❓ 面试题

- [59问：内存管理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md)
  - [如何知道计算机内存布局？内存空间有多少？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#1%E5%A6%82%E4%BD%95%E7%9F%A5%E9%81%93%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4%E6%9C%89%E5%A4%9A%E5%B0%91)
  - [何时去探明内存布局？由谁去探明呢？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#2%E4%BD%95%E6%97%B6%E5%8E%BB%E6%8E%A2%E6%98%8E%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80%E7%94%B1%E8%B0%81%E5%8E%BB%E6%8E%A2%E6%98%8E%E5%91%A2)
  - [kernel会加载到何处呢？由什么决定它的位置？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#3kernel%E4%BC%9A%E5%8A%A0%E8%BD%BD%E5%88%B0%E4%BD%95%E5%A4%84%E5%91%A2%E7%94%B1%E4%BB%80%E4%B9%88%E5%86%B3%E5%AE%9A%E5%AE%83%E7%9A%84%E4%BD%8D%E7%BD%AE)
  - [kernel映像如何隐匿自己的位置？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#4kernel%E6%98%A0%E5%83%8F%E5%A6%82%E4%BD%95%E9%9A%90%E5%8C%BF%E8%87%AA%E5%B7%B1%E7%9A%84%E4%BD%8D%E7%BD%AE)
  - [探知的e820表如何处理？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#5%E6%8E%A2%E7%9F%A5%E7%9A%84e820%E8%A1%A8%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86)
  - [内存是连续的吗？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#6%E5%86%85%E5%AD%98%E6%98%AF%E8%BF%9E%E7%BB%AD%E7%9A%84%E5%90%97)
  - [处理完毕的e820表如何管理？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#7%E5%A4%84%E7%90%86%E5%AE%8C%E6%AF%95%E7%9A%84e820%E8%A1%A8%E5%A6%82%E4%BD%95%E7%AE%A1%E7%90%86)
  - [启动之时内存如何映射的？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#8%E5%90%AF%E5%8A%A8%E4%B9%8B%E6%97%B6%E5%86%85%E5%AD%98%E5%A6%82%E4%BD%95%E6%98%A0%E5%B0%84%E7%9A%84)
  - [保护模式是怎样的？相比实模式有何特点？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#9%E4%BF%9D%E6%8A%A4%E6%A8%A1%E5%BC%8F%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84%E7%9B%B8%E6%AF%94%E5%AE%9E%E6%A8%A1%E5%BC%8F%E6%9C%89%E4%BD%95%E7%89%B9%E7%82%B9)
  - [页保护模式是怎样的？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#10%E9%A1%B5%E4%BF%9D%E6%8A%A4%E6%A8%A1%E5%BC%8F%E6%98%AF%E6%80%8E%E6%A0%B7%E7%9A%84)
  - [页面映射有何作用？都有什么好处？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#11%E9%A1%B5%E9%9D%A2%E6%98%A0%E5%B0%84%E6%9C%89%E4%BD%95%E4%BD%9C%E7%94%A8%E9%83%BD%E6%9C%89%E4%BB%80%E4%B9%88%E5%A5%BD%E5%A4%84)
  - [x86支持的映射模式都有哪些形式？如何分级的？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#12x86%E6%94%AF%E6%8C%81%E7%9A%84%E6%98%A0%E5%B0%84%E6%A8%A1%E5%BC%8F%E9%83%BD%E6%9C%89%E5%93%AA%E4%BA%9B%E5%BD%A2%E5%BC%8F%E5%A6%82%E4%BD%95%E5%88%86%E7%BA%A7%E7%9A%84)
  - [内核如何处理多样式的页映射？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#13%E5%86%85%E6%A0%B8%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86%E5%A4%9A%E6%A0%B7%E5%BC%8F%E7%9A%84%E9%A1%B5%E6%98%A0%E5%B0%84)
  - [面对NUMA等复杂内存环境如何处理？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#14%E9%9D%A2%E5%AF%B9numa%E7%AD%89%E5%A4%8D%E6%9D%82%E5%86%85%E5%AD%98%E7%8E%AF%E5%A2%83%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86)
  - [内核页表如何建立？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#15%E5%86%85%E6%A0%B8%E9%A1%B5%E8%A1%A8%E5%A6%82%E4%BD%95%E5%BB%BA%E7%AB%8B)
  - [内核态进程虚拟地址与物理内存的映射关系？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#16%E5%86%85%E6%A0%B8%E6%80%81%E8%BF%9B%E7%A8%8B%E8%99%9A%E6%8B%9F%E5%9C%B0%E5%9D%80%E4%B8%8E%E7%89%A9%E7%90%86%E5%86%85%E5%AD%98%E7%9A%84%E6%98%A0%E5%B0%84%E5%85%B3%E7%B3%BB)
  - [用户态进程虚拟内存与物理内存的关系如何？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#17%E7%94%A8%E6%88%B7%E6%80%81%E8%BF%9B%E7%A8%8B%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E4%B8%8E%E7%89%A9%E7%90%86%E5%86%85%E5%AD%98%E7%9A%84%E5%85%B3%E7%B3%BB%E5%A6%82%E4%BD%95)
  - [内存管理框架如何构造？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#18%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%A1%86%E6%9E%B6%E5%A6%82%E4%BD%95%E6%9E%84%E9%80%A0)
  - [Kernel内存空间如何划分？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#19kernel%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4%E5%A6%82%E4%BD%95%E5%88%92%E5%88%86)
  - [64位地址空间如何划分？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#2064%E4%BD%8D%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4%E5%A6%82%E4%BD%95%E5%88%92%E5%88%86)
  - [内存分配空间如何实现不可预测性？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#21%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%A9%BA%E9%97%B4%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E4%B8%8D%E5%8F%AF%E9%A2%84%E6%B5%8B%E6%80%A7)
  - [物理内存是如何管理的？怎么分配的？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#22%E7%89%A9%E7%90%86%E5%86%85%E5%AD%98%E6%98%AF%E5%A6%82%E4%BD%95%E7%AE%A1%E7%90%86%E7%9A%84%E6%80%8E%E4%B9%88%E5%88%86%E9%85%8D%E7%9A%84)
  - [Buddy管理算法所处的位置？在什么地方体现？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#23buddy%E7%AE%A1%E7%90%86%E7%AE%97%E6%B3%95%E6%89%80%E5%A4%84%E7%9A%84%E4%BD%8D%E7%BD%AE%E5%9C%A8%E4%BB%80%E4%B9%88%E5%9C%B0%E6%96%B9%E4%BD%93%E7%8E%B0)
  - [内存碎片化了怎么办？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#24%E5%86%85%E5%AD%98%E7%A2%8E%E7%89%87%E5%8C%96%E4%BA%86%E6%80%8E%E4%B9%88%E5%8A%9E)
  - [如何为驱动应用预留大块连续内存？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#25%E5%A6%82%E4%BD%95%E4%B8%BA%E9%A9%B1%E5%8A%A8%E5%BA%94%E7%94%A8%E9%A2%84%E7%95%99%E5%A4%A7%E5%9D%97%E8%BF%9E%E7%BB%AD%E5%86%85%E5%AD%98)
  - [LRU如何运作？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#26lru%E5%A6%82%E4%BD%95%E8%BF%90%E4%BD%9C)
  - [内存回收是如何运作的？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#27%E5%86%85%E5%AD%98%E5%9B%9E%E6%94%B6%E6%98%AF%E5%A6%82%E4%BD%95%E8%BF%90%E4%BD%9C%E7%9A%84)
  - [相同的内存浪费内存空间了？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#28%E7%9B%B8%E5%90%8C%E7%9A%84%E5%86%85%E5%AD%98%E6%B5%AA%E8%B4%B9%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4%E4%BA%86)
  - [页面空间监测手段有什么？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#29%E9%A1%B5%E9%9D%A2%E7%A9%BA%E9%97%B4%E7%9B%91%E6%B5%8B%E6%89%8B%E6%AE%B5%E6%9C%89%E4%BB%80%E4%B9%88)
  - [如何降低页面分配的可预测性？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#30%E5%A6%82%E4%BD%95%E9%99%8D%E4%BD%8E%E9%A1%B5%E9%9D%A2%E5%88%86%E9%85%8D%E7%9A%84%E5%8F%AF%E9%A2%84%E6%B5%8B%E6%80%A7)
  - [如何防范内存泄密？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#31%E5%A6%82%E4%BD%95%E9%98%B2%E8%8C%83%E5%86%85%E5%AD%98%E6%B3%84%E5%AF%86)
  - [如何查看Buddy管理算法下的内存类型信息？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#32%E5%A6%82%E4%BD%95%E6%9F%A5%E7%9C%8Bbuddy%E7%AE%A1%E7%90%86%E7%AE%97%E6%B3%95%E4%B8%8B%E7%9A%84%E5%86%85%E5%AD%98%E7%B1%BB%E5%9E%8B%E4%BF%A1%E6%81%AF)
  - [小块内存空间如何分配管理？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#33%E5%B0%8F%E5%9D%97%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4%E5%A6%82%E4%BD%95%E5%88%86%E9%85%8D%E7%AE%A1%E7%90%86)
  - [SLUB如何管理内存的？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#34slub%E5%A6%82%E4%BD%95%E7%AE%A1%E7%90%86%E5%86%85%E5%AD%98%E7%9A%84)
  - [如何查看slab信息？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#35%E5%A6%82%E4%BD%95%E6%9F%A5%E7%9C%8Bslab%E4%BF%A1%E6%81%AF)
  - [如何防范slab空闲链表的攻击？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#36%E5%A6%82%E4%BD%95%E9%98%B2%E8%8C%83slab%E7%A9%BA%E9%97%B2%E9%93%BE%E8%A1%A8%E7%9A%84%E6%94%BB%E5%87%BB)
  - [SLUB分配如何防止被预判？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#37slub%E5%88%86%E9%85%8D%E5%A6%82%E4%BD%95%E9%98%B2%E6%AD%A2%E8%A2%AB%E9%A2%84%E5%88%A4)
  - [kmalloc和kfree如何实现的？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#38kmalloc%E5%92%8Ckfree%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E7%9A%84)
  - [kernel的内存泄漏如何定位？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#39kernel%E7%9A%84%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E5%A6%82%E4%BD%95%E5%AE%9A%E4%BD%8D)
  - [kernel有内存检测机制吗？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#40kernel%E6%9C%89%E5%86%85%E5%AD%98%E6%A3%80%E6%B5%8B%E6%9C%BA%E5%88%B6%E5%90%97)
  - [支离破碎的内存如何得到大块连续内存？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#41%E6%94%AF%E7%A6%BB%E7%A0%B4%E7%A2%8E%E7%9A%84%E5%86%85%E5%AD%98%E5%A6%82%E4%BD%95%E5%BE%97%E5%88%B0%E5%A4%A7%E5%9D%97%E8%BF%9E%E7%BB%AD%E5%86%85%E5%AD%98)
  - [如何查看vmalloc信息？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#42%E5%A6%82%E4%BD%95%E6%9F%A5%E7%9C%8Bvmalloc%E4%BF%A1%E6%81%AF)
  - [Percpu内存空间如何管理的？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#43percpu%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4%E5%A6%82%E4%BD%95%E7%AE%A1%E7%90%86%E7%9A%84)
  - [从proc接口还可以看到什么？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#44%E4%BB%8Eproc%E6%8E%A5%E5%8F%A3%E8%BF%98%E5%8F%AF%E4%BB%A5%E7%9C%8B%E5%88%B0%E4%BB%80%E4%B9%88)
  - [容器的内存如何管理？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#45%E5%AE%B9%E5%99%A8%E7%9A%84%E5%86%85%E5%AD%98%E5%A6%82%E4%BD%95%E7%AE%A1%E7%90%86)
  - [内核如何防范信息外泄？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#46%E5%86%85%E6%A0%B8%E5%A6%82%E4%BD%95%E9%98%B2%E8%8C%83%E4%BF%A1%E6%81%AF%E5%A4%96%E6%B3%84)
  - [物理内存页面耗尽了如何处理？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#47%E7%89%A9%E7%90%86%E5%86%85%E5%AD%98%E9%A1%B5%E9%9D%A2%E8%80%97%E5%B0%BD%E4%BA%86%E5%A6%82%E4%BD%95%E5%A4%84%E7%90%86)
  - [内核代码段如何进行自我防护？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#48%E5%86%85%E6%A0%B8%E4%BB%A3%E7%A0%81%E6%AE%B5%E5%A6%82%E4%BD%95%E8%BF%9B%E8%A1%8C%E8%87%AA%E6%88%91%E9%98%B2%E6%8A%A4)
  - [内核代码段如何防护注入？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#49%E5%86%85%E6%A0%B8%E4%BB%A3%E7%A0%81%E6%AE%B5%E5%A6%82%E4%BD%95%E9%98%B2%E6%8A%A4%E6%B3%A8%E5%85%A5)
  - [kernel程序空间能否再压榨？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#50kernel%E7%A8%8B%E5%BA%8F%E7%A9%BA%E9%97%B4%E8%83%BD%E5%90%A6%E5%86%8D%E5%8E%8B%E6%A6%A8)
  - [面向用户态程序，内核提供了哪些内存分配接口？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#51%E9%9D%A2%E5%90%91%E7%94%A8%E6%88%B7%E6%80%81%E7%A8%8B%E5%BA%8F%E5%86%85%E6%A0%B8%E6%8F%90%E4%BE%9B%E4%BA%86%E5%93%AA%E4%BA%9B%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E6%8E%A5%E5%8F%A3)
  - [brk接口实现了什么？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#52brk%E6%8E%A5%E5%8F%A3%E5%AE%9E%E7%8E%B0%E4%BA%86%E4%BB%80%E4%B9%88)
  - [mmap接口实现了什么？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#53mmap%E6%8E%A5%E5%8F%A3%E5%AE%9E%E7%8E%B0%E4%BA%86%E4%BB%80%E4%B9%88)
  - [用户态内存如何管理？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#54%E7%94%A8%E6%88%B7%E6%80%81%E5%86%85%E5%AD%98%E5%A6%82%E4%BD%95%E7%AE%A1%E7%90%86)
  - [glibc对brk和mmap如何使用？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#55glibc%E5%AF%B9brk%E5%92%8Cmmap%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8)
  - [如何查看进程内存映射信息？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#56%E5%A6%82%E4%BD%95%E6%9F%A5%E7%9C%8B%E8%BF%9B%E7%A8%8B%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84%E4%BF%A1%E6%81%AF)
  - [如何查看进程内存占用实际情况？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#57%E5%A6%82%E4%BD%95%E6%9F%A5%E7%9C%8B%E8%BF%9B%E7%A8%8B%E5%86%85%E5%AD%98%E5%8D%A0%E7%94%A8%E5%AE%9E%E9%99%85%E6%83%85%E5%86%B5)
  - [如何查看进程内存片段映射详情？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#58%E5%A6%82%E4%BD%95%E6%9F%A5%E7%9C%8B%E8%BF%9B%E7%A8%8B%E5%86%85%E5%AD%98%E7%89%87%E6%AE%B5%E6%98%A0%E5%B0%84%E8%AF%A6%E6%83%85)
  - [如何查看进程内存映射汇总信息？](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/59%E9%97%AE%EF%BC%9A%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md#59%E5%A6%82%E4%BD%95%E6%9F%A5%E7%9C%8B%E8%BF%9B%E7%A8%8B%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84%E6%B1%87%E6%80%BB%E4%BF%A1%E6%81%AF)



## 📃 论文

## 🌌 内存池相关

## 🍺 内存泄露

## 🛠 内存管理工具
