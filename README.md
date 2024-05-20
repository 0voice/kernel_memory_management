# 💻 总结整理linux内核的内存管理的资料，包含论文，文章，视频，以及应用程序的内存泄露，内存池相关

<div align=center>
  
<img width="100%" height="100%" src="https://user-images.githubusercontent.com/87457873/146741232-cc01d5bc-595f-4bd7-a285-b0388ff29027.png"/>
  
</div>

<br>

本repo搜集整理全网Linux内核---内存管理模块相关知识。

**所有数据来源于互联网。所谓取之于互联网，用之于互联网。**

如果涉及版权侵犯，请邮件至 wchao_isvip@163.com ，我们将第一时间处理。

如果您对我们的项目表示赞同与支持，欢迎您 [lssues](https://github.com/0voice/kernel_memory_management/issues) 我们，或者邮件 wchao_isvip@163.com 我们，更加欢迎您 [pull requests](https://github.com/0voice/kernel_memory_management/pulls) 加入我们。

感谢您的支持！

This Repo collects and organizes the whole network Linux kernel -- memory management module related knowledge.

**All data comes from the Internet. The so-called take from the Internet, use for the Internet.**

If copyright infringement is involved, please email wchao_isvip@163.com and we will deal with it as soon as possible.

If you agree to our project and support, welcome [lssues](https://github.com/0voice/kernel_memory_management/issues), we, or email wchao_isvip@163.com us, More welcome [pull requests](https://github.com/0voice/kernel_memory_management/pulls) to join us.

Thank you for your support.

<p align="center">
  <a href="https://github.com/0voice/kernel_memory_management#%E8%81%94%E7%B3%BB%E4%B8%93%E6%A0%8F"><img src="https://img.shields.io/badge/微信公众号-green" alt=""></a>
  <a href="https://www.zhihu.com/people/xiao-zhai-nu-linux"><img src="https://img.shields.io/badge/知乎-blue" alt=""></a>
  <a href="https://space.bilibili.com/64514973"><img src="https://img.shields.io/badge/bilibili-red" alt=""></a>
</p>

- 目录
  - [@ 文章](https://github.com/0voice/kernel_memory_management#-%E6%96%87%E7%AB%A0)
  - [@ 视频](https://github.com/0voice/kernel_memory_management#-%E8%A7%86%E9%A2%91)
  - [@ 面试题](https://github.com/0voice/kernel_memory_management#-%E9%9D%A2%E8%AF%95%E9%A2%98)
  - [@ 100篇论文](https://github.com/0voice/kernel_memory_management#-100%E7%AF%87%E8%AE%BA%E6%96%87)
  - [@ 内存池相关](https://github.com/0voice/kernel_memory_management#-%E5%86%85%E5%AD%98%E6%B1%A0%E7%9B%B8%E5%85%B3)
  - [@ 内存泄露](https://github.com/0voice/kernel_memory_management#-%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2)
  - [@ 内存管理工具](https://github.com/0voice/kernel_memory_management#-%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%B7%A5%E5%85%B7)

## 📜 100篇文章

[内存管理（一）：硬件原理 和 分页管理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E7%A1%AC%E4%BB%B6%E5%8E%9F%E7%90%86%20%E5%92%8C%20%E5%88%86%E9%A1%B5%E7%AE%A1%E7%90%86.md)

[内存管理（二）：内存的动态申请和释放](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E5%86%85%E5%AD%98%E7%9A%84%E5%8A%A8%E6%80%81%E7%94%B3%E8%AF%B7%E5%92%8C%E9%87%8A%E6%94%BE.md)

[内存管理（三）：进程的内存消耗和泄漏](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%89%EF%BC%89%EF%BC%9A%E8%BF%9B%E7%A8%8B%E7%9A%84%E5%86%85%E5%AD%98%E6%B6%88%E8%80%97%E5%92%8C%E6%B3%84%E6%BC%8F.md)

[内存管理（四）：内存与IO的交换](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E5%9B%9B%EF%BC%89%EF%BC%9A%E5%86%85%E5%AD%98%E4%B8%8EIO%E7%9A%84%E4%BA%A4%E6%8D%A2.md)

[内存管理（五）：其他工程问题以及调优](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%BA%94%EF%BC%89%EF%BC%9A%E5%85%B6%E4%BB%96%E5%B7%A5%E7%A8%8B%E9%97%AE%E9%A2%98%E4%BB%A5%E5%8F%8A%E8%B0%83%E4%BC%98.md)

---------内存管理系列文章---------

[内存管理系列一：启动简介](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%B8%80%EF%BC%9A%E5%90%AF%E5%8A%A8%E7%AE%80%E4%BB%8B.md)

[内存管理系列二：创建启动阶段的页表](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%BA%8C%EF%BC%9A%E5%88%9B%E5%BB%BA%E5%90%AF%E5%8A%A8%E9%98%B6%E6%AE%B5%E7%9A%84%E9%A1%B5%E8%A1%A8.md)

[内存管理系列三：MMU前CPU初始化及打开MMU](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%B8%89%EF%BC%9AMMU%E5%89%8DCPU%E5%88%9D%E5%A7%8B%E5%8C%96%E5%8F%8A%E6%89%93%E5%BC%80MMU.md)

[内存管理系列四：setup_arch简介(内存管理初始化)](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%9B%9B%EF%BC%9Asetup_arch%E7%AE%80%E4%BB%8B(%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%88%9D%E5%A7%8B%E5%8C%96).md)

[内存管理系列五：alloc_pages从伙伴系统申请空间简易流程](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%BA%94%EF%BC%9Aalloc_pages%E4%BB%8E%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E7%94%B3%E8%AF%B7%E7%A9%BA%E9%97%B4%E7%AE%80%E6%98%93%E6%B5%81%E7%A8%8B.md)

[内存管理系列六：伙伴系统之buffered_rmqueue](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%85%AD%EF%BC%9A%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E4%B9%8Bbuffered_rmqueue.md)

[内存管理系列七：slub初始化](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%B8%83%EF%BC%9Aslub%E5%88%9D%E5%A7%8B%E5%8C%96.md)

[内存管理系列八：slub创建](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%85%AB%EF%BC%9Aslub%E5%88%9B%E5%BB%BA.md)

[内存管理系列九：slub申请内存](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%B9%9D%EF%BC%9Aslub%E7%94%B3%E8%AF%B7%E5%86%85%E5%AD%98.md)

[内存管理系列十：slub回收](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%EF%BC%9Aslub%E5%9B%9E%E6%94%B6.md)

[内存管理系列十一：slub销毁](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E4%B8%80%EF%BC%9Aslub%E9%94%80%E6%AF%81.md)

[内存管理系列十二：vmalloc内存机制](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E4%BA%8C%EF%BC%9Avmalloc%E5%86%85%E5%AD%98%E6%9C%BA%E5%88%B6.md)

[内存管理系列十三：VMA操作](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E4%B8%89%EF%BC%9AVMA%E6%93%8D%E4%BD%9C.md)

[内存管理系列十四：brk](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E5%9B%9B%EF%BC%9Abrk.md)

[内存管理系列十五：do_page_fault缺页中断](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E4%BA%94%EF%BC%9Ado_page_fault%E7%BC%BA%E9%A1%B5%E4%B8%AD%E6%96%AD.md)

[内存管理系列十六：反向映射RMAP](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E5%85%AD%EF%BC%9A%E5%8F%8D%E5%90%91%E6%98%A0%E5%B0%84RMAP.md)

[内存管理系列十七：内存池](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E4%B8%83%EF%BC%9A%E5%86%85%E5%AD%98%E6%B1%A0.md)

[内存管理系列十八：内存回收之LRU链表](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E5%85%AB%EF%BC%9A%E5%86%85%E5%AD%98%E5%9B%9E%E6%94%B6%E4%B9%8BLRU%E9%93%BE%E8%A1%A8.md)

[内存管理系列十九：内存压缩算法](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E5%8D%81%E4%B9%9D%EF%BC%9A%E5%86%85%E5%AD%98%E5%8E%8B%E7%BC%A9%E7%AE%97%E6%B3%95.md)

[内存管理系列二十：内存压缩算法之数据同步](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%BA%8C%E5%8D%81%EF%BC%9A%E5%86%85%E5%AD%98%E5%8E%8B%E7%BC%A9%E7%AE%97%E6%B3%95%E4%B9%8B%E6%95%B0%E6%8D%AE%E5%90%8C%E6%AD%A5.md)

[内存管理系列二十一：内存回收入口](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%BA%8C%E5%8D%81%E4%B8%80%EF%BC%9A%E5%86%85%E5%AD%98%E5%9B%9E%E6%94%B6%E5%85%A5%E5%8F%A3.md)

[内存管理系列二十二：内存回收核心流程](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%B3%BB%E5%88%97%E4%BA%8C%E5%8D%81%E4%BA%8C%EF%BC%9A%E5%86%85%E5%AD%98%E5%9B%9E%E6%94%B6%E6%A0%B8%E5%BF%83%E6%B5%81%E7%A8%8B.md)

----------英文文章鉴赏----------

[Linux: large-memory management histories](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Linux:%20large-memory%20management%20histories.md)

[Looking at kmalloc() and the SLUB Memory Allocator](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Looking%20at%20kmalloc()%20and%20the%20SLUB%20Memory%20Allocator.md)

[Memory Management in OS: Contiguous, Swapping, Fragmentation](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Memory%20Management%20in%20OS:%20Contiguous%2C%20Swapping%2C%20Fragmentation.md)

[Memory Management in Operating System](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Memory%20Management%20in%20Operating%20System.md)

[Operating System - Memory Management](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Operating%20System%20-%20Memory%20Management.md)

[Virtual Memory in OS: What is, Demand Paging, Advantages](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Virtual%20Memory%20in%20OS:%20What%20is%2C%20Demand%20Paging%2C%20Advantages.md)

[Why Do We Need Virtual Memory](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Why%20Do%20We%20Need%20Virtual%20Memory%3F.md)



----------分割线----------

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

[一文让你看懂内存与CPU之间的关系](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E4%B8%80%E6%96%87%E8%AE%A9%E4%BD%A0%E7%9C%8B%E6%87%82%E5%86%85%E5%AD%98%E4%B8%8ECPU%E4%B9%8B%E9%97%B4%E7%9A%84%E5%85%B3%E7%B3%BB.md)

[linux内存管理---详解](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/linux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86---%E8%AF%A6%E8%A7%A3.md)

[一文带你了解，虚拟内存、内存分页、分段、段页式内存管理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E4%B8%80%E6%96%87%E5%B8%A6%E4%BD%A0%E4%BA%86%E8%A7%A3%EF%BC%8C%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E3%80%81%E5%86%85%E5%AD%98%E5%88%86%E9%A1%B5%E3%80%81%E5%88%86%E6%AE%B5%E3%80%81%E6%AE%B5%E9%A1%B5%E5%BC%8F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86.md)

[深入浅出linux内存管理（一）](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAlinux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%B8%80%EF%BC%89.md)

[深入浅出linux内存管理（二）](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAlinux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%EF%BC%88%E4%BA%8C%EF%BC%89.md)

[为什么linux需要虚拟内存](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E4%B8%BA%E4%BB%80%E4%B9%88linux%E9%9C%80%E8%A6%81%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98.md)

[【总结时刻】物理内存空间管理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86-%E7%89%A9%E7%90%86%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4%E7%AE%A1%E7%90%86.md)

[【总结时刻】用户态内存映射](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86-%E7%94%A8%E6%88%B7%E6%80%81%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84.md)

[【总结时刻】内核态内存映射](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86-%E5%86%85%E6%A0%B8%E6%80%81%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84.md)

[虚拟地址空间——MMU](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E8%99%9A%E6%8B%9F%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4%E2%80%94%E2%80%94MMU.md)

[进程的虚拟内存空间](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E8%BF%9B%E7%A8%8B%E7%9A%84%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E7%A9%BA%E9%97%B4.md)


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

[Linux内存问题终极探讨---虚拟内存|内存池|内存泄漏|管理组件](https://www.bilibili.com/video/BV1ML4y1E79a/)


-----西安交通大学内存管理（24讲）提取码1024-----

[背景](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[固定分区分配](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[连续内存分配](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[分页](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[分页硬件和TLB](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[分段管理](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[虚拟内存](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[请求调页](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[页面置换](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[页面置换算法](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[帧分配](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

[颠簸](https://pan.baidu.com/s/12YqOV0Qm4X76PPvkgJDbsw)

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



## 📃 100篇论文

[《ARM的虚拟内存管理技术的研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/ARM%E7%9A%84%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%8A%80%E6%9C%AF%E7%9A%84%E7%A0%94%E7%A9%B6.caj)

[《C语言的内存漏洞分析与研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/C%E8%AF%AD%E8%A8%80%E7%9A%84%E5%86%85%E5%AD%98%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90%E4%B8%8E%E7%A0%94%E7%A9%B6.pdf)

[《FreeRTOS内存管理方案的分析与改进》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/FreeRTOS%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%96%B9%E6%A1%88%E7%9A%84%E5%88%86%E6%9E%90%E4%B8%8E%E6%94%B9%E8%BF%9B.pdf)

[《Linux Memory Management》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%20Memory%20Management.pdf)

[《Linux内存管理分析与研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%88%86%E6%9E%90%E4%B8%8E%E7%A0%94%E7%A9%B6.pdf)

[《Linux内存管理的设计与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《Linux内核中内存池的实现及应用》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%E5%86%85%E6%A0%B8%E4%B8%AD%E5%86%85%E5%AD%98%E6%B1%A0%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8F%8A%E5%BA%94%E7%94%A8.pdf)

[《Linux内核中动态内存检测机制的研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%E5%86%85%E6%A0%B8%E4%B8%AD%E5%8A%A8%E6%80%81%E5%86%85%E5%AD%98%E6%A3%80%E6%B5%8B%E6%9C%BA%E5%88%B6%E7%9A%84%E7%A0%94%E7%A9%B6.pdf)

[《Linux内核伙伴系统分析》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%E5%86%85%E6%A0%B8%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E5%88%86%E6%9E%90.pdf)

[《Linux内核内存池实现研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%E5%86%85%E6%A0%B8%E5%86%85%E5%AD%98%E6%B1%A0%E5%AE%9E%E7%8E%B0%E7%A0%94%E7%A9%B6.pdf)

[《Linux实时内存的研究与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%E5%AE%9E%E6%97%B6%E5%86%85%E5%AD%98%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《Linux操作系统内核分析与研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Linux%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E5%86%85%E6%A0%B8%E5%88%86%E6%9E%90%E4%B8%8E%E7%A0%94%E7%A9%B6.pdf)

[《Memory Management 101： Introduction to Memory Management in Linux》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Memory%20Management%20101%EF%BC%9A%20Introduction%20to%20Memory%20Management%20in%20Linux.pdf)

[《Memory Management in Linux》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Memory%20Management%20in%20Linux.pdf)

[《Memory Management》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Memory%20Management.pdf)

[《NUMA架构内多个节点间访存延时平衡的内存分配策略》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/NUMA%E6%9E%B6%E6%9E%84%E5%86%85%E5%A4%9A%E4%B8%AA%E8%8A%82%E7%82%B9%E9%97%B4%E8%AE%BF%E5%AD%98%E5%BB%B6%E6%97%B6%E5%B9%B3%E8%A1%A1%E7%9A%84%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%AD%96%E7%95%A5.pdf)

[《Nginx Slab算法研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Nginx%20Slab%E7%AE%97%E6%B3%95%E7%A0%94%E7%A9%B6.pdf)

[《TCP_IP协议栈的轻量级多线程实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/TCP_IP%E5%8D%8F%E8%AE%AE%E6%A0%88%E7%9A%84%E8%BD%BB%E9%87%8F%E7%BA%A7%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%AE%9E%E7%8E%B0.caj)

[《VC中利用内存映射文件实现进程间通信的方法》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/VC%E4%B8%AD%E5%88%A9%E7%94%A8%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84%E6%96%87%E4%BB%B6%E5%AE%9E%E7%8E%B0%E8%BF%9B%E7%A8%8B%E9%97%B4%E9%80%9A%E4%BF%A1%E7%9A%84%E6%96%B9%E6%B3%95.pdf)

[《Virtual Memory Management Techniques in 2.6 Kernel and Challenges》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Virtual%20Memory%20Management%20Techniques%20in%202.6%20Kernel%20and%20Challenges.pdf)

[《Visual C 中利用内存映射文件在进程之间共享数据》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/Visual%20C%20%20%E4%B8%AD%E5%88%A9%E7%94%A8%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84%E6%96%87%E4%BB%B6%E5%9C%A8%E8%BF%9B%E7%A8%8B%E4%B9%8B%E9%97%B4%E5%85%B1%E4%BA%AB%E6%95%B0%E6%8D%AE.pdf)

[《Linux Physical Memory Page Allocation》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E3%80%8ALinux%20Physical%20Memory%20Page%20Allocation%E3%80%8B.pdf)

[《一个内存分配器的设计和实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E4%B8%AA%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E5%99%A8%E7%9A%84%E8%AE%BE%E8%AE%A1%E5%92%8C%E5%AE%9E%E7%8E%B0.pdf)

[《一种Linux内存管理机制》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8DLinux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6.pdf)

[《一种TLB结构优化方法》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8DTLB%E7%BB%93%E6%9E%84%E4%BC%98%E5%8C%96%E6%96%B9%E6%B3%95.pdf)

[《一种优化的伙伴系统存储管理算法设计》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8D%E4%BC%98%E5%8C%96%E7%9A%84%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E5%AD%98%E5%82%A8%E7%AE%A1%E7%90%86%E7%AE%97%E6%B3%95%E8%AE%BE%E8%AE%A1.pdf)

[《一种基于虚拟机的动态内存泄露检测方法》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8D%E5%9F%BA%E4%BA%8E%E8%99%9A%E6%8B%9F%E6%9C%BA%E7%9A%84%E5%8A%A8%E6%80%81%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2%E6%A3%80%E6%B5%8B%E6%96%B9%E6%B3%95.pdf)

[《一种提高Linux内存管理实时性的设计方案》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8D%E6%8F%90%E9%AB%98Linux%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%AE%9E%E6%97%B6%E6%80%A7%E7%9A%84%E8%AE%BE%E8%AE%A1%E6%96%B9%E6%A1%88.pdf)

[《一种改进的Linux内存分配机制》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8D%E6%94%B9%E8%BF%9B%E7%9A%84Linux%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E6%9C%BA%E5%88%B6.pdf)

[《一种改进的伙伴系统内存管理方法》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8D%E6%94%B9%E8%BF%9B%E7%9A%84%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%96%B9%E6%B3%95.pdf)

[《一种跨平台内存池的设计与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8D%E8%B7%A8%E5%B9%B3%E5%8F%B0%E5%86%85%E5%AD%98%E6%B1%A0%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《一种高效的池式内存管理器的设计》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%B8%80%E7%A7%8D%E9%AB%98%E6%95%88%E7%9A%84%E6%B1%A0%E5%BC%8F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%99%A8%E7%9A%84%E8%AE%BE%E8%AE%A1.pdf)

[《云计算平台中多虚拟机内存协同优化策略研》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%BA%91%E8%AE%A1%E7%AE%97%E5%B9%B3%E5%8F%B0%E4%B8%AD%E5%A4%9A%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%86%85%E5%AD%98%E5%8D%8F%E5%90%8C%E4%BC%98%E5%8C%96%E7%AD%96%E7%95%A5%E7%A0%94.pdf)

[《云计算平台中多虚拟机内存协同优化策略研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E4%BA%91%E8%AE%A1%E7%AE%97%E5%B9%B3%E5%8F%B0%E4%B8%AD%E5%A4%9A%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%86%85%E5%AD%98%E5%8D%8F%E5%90%8C%E4%BC%98%E5%8C%96%E7%AD%96%E7%95%A5%E7%A0%94%E7%A9%B6.pdf)

[《内存管理机制的高效实现研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6%E7%9A%84%E9%AB%98%E6%95%88%E5%AE%9E%E7%8E%B0%E7%A0%94%E7%A9%B6.pdf)

[《分页存储管理系统中内存有效访问时间的计算》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%88%86%E9%A1%B5%E5%AD%98%E5%82%A8%E7%AE%A1%E7%90%86%E7%B3%BB%E7%BB%9F%E4%B8%AD%E5%86%85%E5%AD%98%E6%9C%89%E6%95%88%E8%AE%BF%E9%97%AE%E6%97%B6%E9%97%B4%E7%9A%84%E8%AE%A1%E7%AE%97.pdf)

[《利用内存映射连续性提高TLB地址覆盖范围的技术评测》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%88%A9%E7%94%A8%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84%E8%BF%9E%E7%BB%AD%E6%80%A7%E6%8F%90%E9%AB%98TLB%E5%9C%B0%E5%9D%80%E8%A6%86%E7%9B%96%E8%8C%83%E5%9B%B4%E7%9A%84%E6%8A%80%E6%9C%AF%E8%AF%84%E6%B5%8B.pdf) 

[《动态内存分配器研究综述》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%8A%A8%E6%80%81%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E5%99%A8%E7%A0%94%E7%A9%B6%E7%BB%BC%E8%BF%B0.pdf)

[《动态存储管理机制的改进及实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%8A%A8%E6%80%81%E5%AD%98%E5%82%A8%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6%E7%9A%84%E6%94%B9%E8%BF%9B%E5%8F%8A%E5%AE%9E%E7%8E%B0.pdf)

[《基于C 的高效内存池的设计与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8EC%20%20%E7%9A%84%E9%AB%98%E6%95%88%E5%86%85%E5%AD%98%E6%B1%A0%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《基于C 自定义内存分配器的实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8EC%20%20%E8%87%AA%E5%AE%9A%E4%B9%89%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E5%99%A8%E7%9A%84%E5%AE%9E%E7%8E%B0.pdf)

[《基于Linux内核的动态内存管理机制的实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8ELinux%E5%86%85%E6%A0%B8%E7%9A%84%E5%8A%A8%E6%80%81%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%9C%BA%E5%88%B6%E7%9A%84%E5%AE%9E%E7%8E%B0.pdf)

[《基于Linux内核页表构建内核隔离空间的研究及实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8ELinux%E5%86%85%E6%A0%B8%E9%A1%B5%E8%A1%A8%E6%9E%84%E5%BB%BA%E5%86%85%E6%A0%B8%E9%9A%94%E7%A6%BB%E7%A9%BA%E9%97%B4%E7%9A%84%E7%A0%94%E7%A9%B6%E5%8F%8A%E5%AE%9E%E7%8E%B0.pdf)

[《基于RDMA和NVM的大数据系统一致性协议研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8ERDMA%E5%92%8CNVM%E7%9A%84%E5%A4%A7%E6%95%B0%E6%8D%AE%E7%B3%BB%E7%BB%9F%E4%B8%80%E8%87%B4%E6%80%A7%E5%8D%8F%E8%AE%AE%E7%A0%94%E7%A9%B6.pdf)

[《基于RDMA高速网络的高性能分布式系统》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8ERDMA%E9%AB%98%E9%80%9F%E7%BD%91%E7%BB%9C%E7%9A%84%E9%AB%98%E6%80%A7%E8%83%BD%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F.pdf)

[《基于RelayFS的内核态内存泄露的检测和跟踪》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8ERelayFS%E7%9A%84%E5%86%85%E6%A0%B8%E6%80%81%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2%E7%9A%84%E6%A3%80%E6%B5%8B%E5%92%8C%E8%B7%9F%E8%B8%AA.pdf)

[《基于linux用户态可自控缓冲区管理设计与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8Elinux%E7%94%A8%E6%88%B7%E6%80%81%E5%8F%AF%E8%87%AA%E6%8E%A7%E7%BC%93%E5%86%B2%E5%8C%BA%E7%AE%A1%E7%90%86%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《基于multimap映射的动态内存分配算法探究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8Emultimap%E6%98%A0%E5%B0%84%E7%9A%84%E5%8A%A8%E6%80%81%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%AE%97%E6%B3%95%E6%8E%A2%E7%A9%B6.pdf)

[《基于云计算虚拟化平台的内存管理研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8E%E4%BA%91%E8%AE%A1%E7%AE%97%E8%99%9A%E6%8B%9F%E5%8C%96%E5%B9%B3%E5%8F%B0%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%A0%94%E7%A9%B6.pdf)

[《基于内存池的空间数据调度算法》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%9F%BA%E4%BA%8E%E5%86%85%E5%AD%98%E6%B1%A0%E7%9A%84%E7%A9%BA%E9%97%B4%E6%95%B0%E6%8D%AE%E8%B0%83%E5%BA%A6%E7%AE%97%E6%B3%95.pdf)

[《多核系统内存管理算法的研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%A4%9A%E6%A0%B8%E7%B3%BB%E7%BB%9F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%AE%97%E6%B3%95%E7%9A%84%E7%A0%94%E7%A9%B6.pdf)

[《实时系统内存管理方案的设计与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%AE%9E%E6%97%B6%E7%B3%BB%E7%BB%9F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%96%B9%E6%A1%88%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《对linux伙伴系统及其反碎片机制的研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%AF%B9linux%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E5%8F%8A%E5%85%B6%E5%8F%8D%E7%A2%8E%E7%89%87%E6%9C%BA%E5%88%B6%E7%9A%84%E7%A0%94%E7%A9%B6.pdf)

[《嵌入式实时系统动态内存分配管理器的设计与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%B5%8C%E5%85%A5%E5%BC%8F%E5%AE%9E%E6%97%B6%E7%B3%BB%E7%BB%9F%E5%8A%A8%E6%80%81%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E7%AE%A1%E7%90%86%E5%99%A8%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《并发数据结构及其在动态内存管理中的应用》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%B9%B6%E5%8F%91%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%8F%8A%E5%85%B6%E5%9C%A8%E5%8A%A8%E6%80%81%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8.pdf)

[《应用协同的进程组内存管理支撑技术》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E5%BA%94%E7%94%A8%E5%8D%8F%E5%90%8C%E7%9A%84%E8%BF%9B%E7%A8%8B%E7%BB%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%94%AF%E6%92%91%E6%8A%80%E6%9C%AF.pdf)

[《支持高性能IPC的内存管理策略研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E6%94%AF%E6%8C%81%E9%AB%98%E6%80%A7%E8%83%BDIPC%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E7%AD%96%E7%95%A5%E7%A0%94%E7%A9%B6.pdf)

[《有效的C 内存泄露检测方法》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E6%9C%89%E6%95%88%E7%9A%84C%20%20%E5%86%85%E5%AD%98%E6%B3%84%E9%9C%B2%E6%A3%80%E6%B5%8B%E6%96%B9%E6%B3%95.pdf)

[《浅析伙伴系统的分配与回收》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E6%B5%85%E6%9E%90%E4%BC%99%E4%BC%B4%E7%B3%BB%E7%BB%9F%E7%9A%84%E5%88%86%E9%85%8D%E4%B8%8E%E5%9B%9E%E6%94%B6.pdf)

[《用户态内存管理关键技术研究》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E7%94%A8%E6%88%B7%E6%80%81%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%85%B3%E9%94%AE%E6%8A%80%E6%9C%AF%E7%A0%94%E7%A9%B6.pdf)

[《申威处理器页表结构Cache的优化研究与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E7%94%B3%E5%A8%81%E5%A4%84%E7%90%86%E5%99%A8%E9%A1%B5%E8%A1%A8%E7%BB%93%E6%9E%84Cache%E7%9A%84%E4%BC%98%E5%8C%96%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《虚拟化系统中的内存管理优化》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E8%99%9A%E6%8B%9F%E5%8C%96%E7%B3%BB%E7%BB%9F%E4%B8%AD%E7%9A%84%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E4%BC%98%E5%8C%96.pdf)

[《面向Linux内核空间的内存分配隔离方法的研究与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E9%9D%A2%E5%90%91Linux%E5%86%85%E6%A0%B8%E7%A9%BA%E9%97%B4%E7%9A%84%E5%86%85%E5%AD%98%E5%88%86%E9%85%8D%E9%9A%94%E7%A6%BB%E6%96%B9%E6%B3%95%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)

[《页面分配器的研究与实现》](https://github.com/0voice/kernel_memory_management/blob/main/%F0%9F%93%81%E8%AE%BA%E6%96%87/%E9%A1%B5%E9%9D%A2%E5%88%86%E9%85%8D%E5%99%A8%E7%9A%84%E7%A0%94%E7%A9%B6%E4%B8%8E%E5%AE%9E%E7%8E%B0.pdf)


## 🌌 内存池相关

   #### 文章
   
   - [18张图揭秘高性能Linux服务器内存池技术是如何实现的](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/18%E5%BC%A0%E5%9B%BE%E6%8F%AD%E7%A7%98%E9%AB%98%E6%80%A7%E8%83%BDLinux%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%86%85%E5%AD%98%E6%B1%A0%E6%8A%80%E6%9C%AF%E6%98%AF%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E7%9A%84.md)
   - [C++ 实现高性能内存池](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/C%2B%2B%20%E5%AE%9E%E7%8E%B0%E9%AB%98%E6%80%A7%E8%83%BD%E5%86%85%E5%AD%98%E6%B1%A0.md)
   - [Nginx 内存池管理](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/Nginx%20%E5%86%85%E5%AD%98%E6%B1%A0%E7%AE%A1%E7%90%86.md)
   - [性能优化:高效内存池的设计与实现](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E3%80%90%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96%E3%80%91%E9%AB%98%E6%95%88%E5%86%85%E5%AD%98%E6%B1%A0%E7%9A%84%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.md)

   #### 框架
   
   - [userpro/MemoryPool](https://github.com/userpro/MemoryPool):一个极简内存池实现
   - [DavidLiRemini/MemoryPool](https://github.com/DavidLiRemini/MemoryPool):简单有效的内存池实现
   - [DGuco/shmqueue](https://github.com/DGuco/shmqueue):基于c++内存池,共享内存和信号量实现高速的进程间通信队列,单进程读单进程写无需加锁，多进程读多进程写用信号量集实现读写锁保证读写安全
   - [ycsoft/pool](https://github.com/ycsoft/pool):基于ANSI C开发的内存池和线程池，性能优异
   - [hansionz/ConcurrentMemoryPool](https://github.com/hansionz/ConcurrentMemoryPool):一个三级缓存的高并发内存池
   - [Fang-create/memory_pool](https://github.com/Fang-create/memory_pool):内存池----仿nginx实现
   - [CandyConfident/HighPerformanceConcurrentServer](https://github.com/CandyConfident/HighPerformanceConcurrentServer):基于C++11、部分C++14/17特性的一个高性能并发httpserver，包括日志、线程池、内存池、定时器、网络io、http、数据库连接等模块。
   - [crspecter/ydx_slab_util](https://github.com/crspecter/ydx_slab_util):实现一个内存池，内存管理机制借鉴memcached，使用一系列链表管理不同大小的内存区块。
   - [jixuduxing/CommLib](https://github.com/jixuduxing/CommLib):linux常用库,使用boost和标准库编写的常用库,包含线程池、内存池、通信、日志、时间处理、定时器
   - [lrsand52m/MemoryPool](https://github.com/lrsand52m/MemoryPool):基于TLS的高并发内存池
   - [tsreaper/my-allocator](一个简单而较为高效的 C++ Allocator，通过内存池实现):一个简单而较为高效的 C++ Allocator，通过内存池实现
   - [lhh0461/simple_mem_pool](https://github.com/lhh0461/simple_mem_pool):简单的C++内存池模块
   - [hardrong/concurrent-memory-pool](https://github.com/hardrong/concurrent-memory-pool):基于TCmalloc实现的内存池
   - [ysluckly/ConcurrentMemoryPool](https://github.com/ysluckly/ConcurrentMemoryPool):基于三级缓存架构的高并发内存池
   - [qixianghui123/memorypool](https://github.com/qixianghui123/memorypool):基于C++实现内存池技术 memorypool
   - [lr-erics/HashIndex](https://github.com/lr-erics/HashIndex):内建内存池的内存索引结构，面向特定场景业务数据，比如在线广告业务数据
   - [1289148370/negix-](https://github.com/1289148370/negix-):移植nginx内存池源码，实现简单的内存池类
   - [xjhahah/MemPool](https://github.com/xjhahah/MemPool):C++项目之内存池技术
   - [besmallw/ngx_palloc](https://github.com/besmallw/ngx_palloc):ngx源码分析——内存池
   - [LumosN/ConcurrentMemoryPool](https://github.com/LumosN/ConcurrentMemoryPool):C++项目 | 高并发内存池
   - [Lotu527/MemoryPool](https://github.com/Lotu527/MemoryPool):基于C++11实现的内存池
   - [YanlinWangWang/Memory-Pool](https://github.com/YanlinWangWang/Memory-Pool):C++实现的多线程内存池
   - [ADreamyj/Cache-Pool](https://github.com/ADreamyj/Cache-Pool):这是一个高并发的内存池项目，其主要解决程序员在申请内存时存在锁竞争以及内存碎片的问题。


## 🍺 内存泄露

- [5 useful tools to detect memory leaks with examples](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/5%20useful%20tools%20to%20detect%20memory%20leaks%20with%20examples.md)
- [内存泄漏的在线排查](https://github.com/0voice/kernel_memory_management/blob/main/%E2%9C%8D%20%E6%96%87%E7%AB%A0/%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E7%9A%84%E5%9C%A8%E7%BA%BF%E6%8E%92%E6%9F%A5.md)

## 🛠 内存管理工具

[Valgrind](https://valgrind.org/)：Valgrind是一个用于构建动态分析工具的工具框架。有一些Valgrind工具可以自动检测许多内存管理和线程错误，并详细分析你的程序。您还可以使用Valgrind来构建新的工具。
Valgrind发行版目前包括7个产品质量的工具:一个内存错误检测器、两个线程错误检测器、一个缓存和分支预测分析器、一个调用图生成缓存和分支预测分析器，以及两个不同的堆分析器。它还包括一个实验性的SimPoint基本块向量生成器。

[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer):Google出品的内存检测工具

#### 内存性能指标

<img width="50%" height="50%" src="https://img-blog.csdnimg.cn/20191020110333604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQwMjM5OTM=,size_16,color_FFFFFF,t_70"/>

#### 指标-工具映射图

<img width="50%" height="50%" src="https://img-blog.csdnimg.cn/20191020110347829.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQwMjM5OTM=,size_16,color_FFFFFF,t_70"/>


#### 工具-指标映射图

<img width="50%" height="50%" src="https://img-blog.csdnimg.cn/2019102011040022.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQwMjM5OTM=,size_16,color_FFFFFF,t_70"/>

#### 分析思路

##### 分析的基本过程

a. 先用 free 和 top，查看系统整体的内存使用情况。

b. 用vmstat及pidstat查看内存变化情况，确定内存问题类型

c. 详细分析，如内存分配分析、缓存/缓冲区分析、具体进程的内存分析

## 联系专栏

#### [Linux内核源码/内存调优/文件系统/进程管理/设备驱动/网络协议栈](https://ke.qq.com/course/4032547?flowToken=1041395)

<br/>
<br/>
<h3 >零领工作</h3> 

---

##### 实时提供，每周发布北京，上海，广州，深圳，杭州，南京，合肥，武汉，长沙，重庆，成都，西安，厦门的c/c++，golang方向的招聘岗位信息。 包含校招，社招，实习岗位， 面经，八股，简历

<img src="https://img.0voice.com/public/0e59910091576beaebe20f303357edf7.jpg" alt="零领工作" style="width:300px;height:300px;">

<br/>
<br/>
