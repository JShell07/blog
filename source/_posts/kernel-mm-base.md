---
title: kernel_mm_base
date: 2020-03-04 17:05:37
tags: mm
categories: memory
---

## 1. 术语
__UMA（Uniform memory access）__  
系统中的所有的processor共享一个统一的，一致的物理内存空间。无论从哪一个processor发起访问，对内存地址的访问时间都是一样的。

__NUMA（Non-uniform memory access）__  
对某个内存地址的访问是和该memory与processor之间的相对位置有关的。

<!--more-->

## 2. memory model
### 2.1. Flat memory model
如果cpu在访问物理内存的时候，其地址空间是连续的。PFN（page frame number）和mem_map数组index的关系是线性的（有一个固定偏移，如果内存对应的物理地址等于0，那么PFN就是数组index）. 映射与物理地址连续对应，两者可能差于固定偏移。

### 2.2. Discontinuous Memory Mode
如果cpu在访问物理内存的时候，其地址空间有一些空洞，是不连续的。

iscontiguous memory本质上是flat memory内存模型的扩展。在每一个成片的大块内存中node， 其内部模型是flat memory.

![Discontinuous Memory Mode](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/base/Discontiguous%20Memory%20Model.gif)

### 2.3. Sparse Memory Mode
连续的地址空间按照SECTION（例如1G）被分成了一段一段的，其中每一section都是hotplug的。内存地址空间可以被切分的更细，支持更离散的Discontiguous memory。

sparse memory多了一个section的概念，让转换变成了PFN<--->Section<--->page。 

![Sparse Memory Mode](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/base/Sparse%20Memory%20Model.gif)

在include/asm-generic/memory_model.h 可见
```
/*
 * supports 3 memory models.
 */
#if defined(CONFIG_FLATMEM)

#define __pfn_to_page(pfn)      (mem_map + ((pfn) - ARCH_PFN_OFFSET))
#define __page_to_pfn(page)     ((unsigned long)((page) - mem_map) + \
                                 ARCH_PFN_OFFSET)

#elif defined(CONFIG_DISCONTIGMEM)
#define __pfn_to_page(pfn)                      \
({      unsigned long __pfn = (pfn);            \
        unsigned long __nid = arch_pfn_to_nid(__pfn);  \
        NODE_DATA(__nid)->node_mem_map + arch_local_page_offset(__pfn, __nid);\
})

#define __page_to_pfn(pg)                                               \
({      const struct page *__pg = (pg);                                 \
        struct pglist_data *__pgdat = NODE_DATA(page_to_nid(__pg));     \
        (unsigned long)(__pg - __pgdat->node_mem_map) +                 \
         __pgdat->node_start_pfn;                                       \
})

#elif defined(CONFIG_SPARSEMEM_VMEMMAP)
/* memmap is virtually contiguous.  */
#define __pfn_to_page(pfn)      (vmemmap + (pfn))
#define __page_to_pfn(page)     (unsigned long)((page) - vmemmap)

#elif defined(CONFIG_SPARSEMEM)
/*
 * Note: section's mem_map is encoded to reflect its start_pfn.
 * section[i].section_mem_map == mem_map's address - start_pfn;
 */
#define __page_to_pfn(pg)                                       \
({      const struct page *__pg = (pg);                         \
        int __sec = page_to_section(__pg);                      \
        (unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec))); \
})

#define __pfn_to_page(pfn)                              \
({      unsigned long __pfn = (pfn);                    \
        struct mem_section *__sec = __pfn_to_section(__pfn);    \
        __section_mem_map_addr(__sec) + __pfn;          \
})
#endif /* CONFIG_FLATMEM/DISCONTIGMEM/SPARSEMEM */

#define page_to_pfn __page_to_pfn
#define pfn_to_page __pfn_to_page
```

## 3. 内存需求

向内核线程/用户进程提供服务 
- 应用程序访问远大于物理内存虚拟地址空间（Virtual Address Space） 
- 每个进程独立运行空间，可提供内存保护（memory protection） 
- 提供内存映射（Memory Mapping）机制，以便把物理内存、I/O空间、Kernel Image、文件等对象映射到相应进程的地址空间中，方便进程的访问 
- 提供公平、高效的物理内存分配（Physical Memory Allocation）算法
- 提供进程间内存共享的方法（以虚拟内存的形式），也称作Shared Virtual Memory 

更高级需求
内存的热拔插（memory hotplug）
- 内存的size超过了虚拟地址可寻址的空间怎么办（high memory）
- 超大页（hugetlbpage）的支持
- 利用磁盘作为交换页以扩大可用内存（各种swap机制和算法）
- 在NUMA系统中通过移动物理页面位置的方法提升内存的访问效率（Page migration）
- 内存泄漏的检查
- 内存碎片的整理
- 内存不足时的处理（oom kill机制） 

fixme
![linux mem structure](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/base/mem_structure.gif)

__cpu 访问内存方式__  
cpu <---> MMU <---> Memory

__Device 访问内存方式__  
Device <---> CPU <---> MEM  
Device <---> DMA <---> MEM  
Device <---> IOMMU <---> MEM  


## Reference
[Linux kernel内存管理的基本概念](http://www.wowotech.net/memory_management/concept.html)

[Linux内存模型](http://www.wowotech.net/memory_management/memory_model.html)

[ARM64架构下地址翻译相关的宏定义](http://www.wowotech.net/memory_management/arm64-memory-addressing.html)
