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

## 📀 视频

## 📃 论文

## 🌌 内存池相关

## 🍺 内存泄露

## 🛠 内存管理工具
