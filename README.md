# 💻 总结整理linux内核的内存管理的资料，包含论文，文章，视频，以及应用程序的内存泄露，内存池相关

![内存管理](https://user-images.githubusercontent.com/87457873/146741232-cc01d5bc-595f-4bd7-a285-b0388ff29027.png)



## 📜 文章

[内存管理（一）：硬件原理 和 分页管理.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E7%A1%AC%E4%BB%B6%E5%8E%9F%E7%90%86%20%E5%92%8C%20%E5%88%86%E9%A1%B5%E7%AE%A1%E7%90%86.md)

[内存管理（二）：内存的动态申请和释放.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E5%86%85%E5%AD%98%E7%9A%84%E5%8A%A8%E6%80%81%E7%94%B3%E8%AF%B7%E5%92%8C%E9%87%8A%E6%94%BE.md)

[内存管理（三）：进程的内存消耗和泄漏.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E8%BF%9B%E7%A8%8B%E7%9A%84%E5%86%85%E5%AD%98%E6%B6%88%E8%80%97%E5%92%8C%E6%B3%84%E6%BC%8F.md)

[内存管理（四）：内存与IO的交换.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%9A%E5%86%85%E5%AD%98%E4%B8%8EIO%E7%9A%84%E4%BA%A4%E6%8D%A2.md)

[内存管理（五）：其他工程问题以及调优.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%9A%E5%85%B6%E4%BB%96%E5%B7%A5%E7%A8%8B%E9%97%AE%E9%A2%98%E4%BB%A5%E5%8F%8A%E8%B0%83%E4%BC%98.md)

[多核心Linux内核路径优化的不二法门之-slab与伙伴系统.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%A4%9A%E6%A0%B8%E5%BF%83Linux%E5%86%85%E6%A0%B8%E8%B7%AF%E5%BE%84%E4%BC%98%E5%8C%96%E7%9A%84%E4%B8%8D%E4%BA%8C%E6%B3%95%E9%97%A8%E4%B9%8B-slab%E4%B8%8E%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F.md)

[尽情阅读，技术进阶，详解mmap原理.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%B0%BD%E6%83%85%E9%98%85%E8%AF%BB%EF%BC%8C%E6%8A%80%E6%9C%AF%E8%BF%9B%E9%98%B6%EF%BC%8C%E8%AF%A6%E8%A7%A3mmap%E5%8E%9F%E7%90%86.md)

[浅谈Linux内存管理机制.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E6%B5%85%E8%B0%88Linux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6.md)

[Linux中的内存管理机制.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Linux%E4%B8%AD%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6.md)

[Linux虚拟内存管理，MMU机制，原来如此.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Linux%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%8CMMU%E6%9C%BA%E5%88%B6%EF%BC%8C%E5%8E%9F%E6%9D%A5%E5%A6%82%E6%AD%A4.md)

[一文了解，Linux内存管理，malloc、free 实现原理.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E4%B8%80%E6%96%87%E4%BA%86%E8%A7%A3%EF%BC%8CLinux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%8Cmalloc%E3%80%81free%20%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.md)

[内存管理之内存映射.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8B%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84.md)

[内存管理之分页.md](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8B%E5%88%86%E9%A1%B5.md)

[内存管理之内核空间和用户空间](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B9%8B%E5%86%85%E6%A0%B8%E7%A9%BA%E9%97%B4%E5%92%8C%E7%94%A8%E6%88%B7%E7%A9%BA%E9%97%B4.md)


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




## 📃 论文

## 🌌 内存池相关

## 🍺 内存泄露

## 🛠 内存管理工具
