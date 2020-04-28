---
title: kernel_virtual_addr_map
date: 2019-07-12 10:51:28
tags:
    - memory
    - kernel
categories:
    - memory
---

## 1. User space
mmap, munmap - map or unmap files or devices into memory. 
将文件或者设备与内存映射起来。

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);


#include <unistd.h>
int getpagesize(void);

```
<!--more-->

## 2. Kernel space
### 2.1. 静态映射
start_kernel() ->  
&nbsp;&nbsp;setup_arch() ->  
&nbsp;&nbsp;&nbsp;&nbsp; paging_init() ->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;devicemaps_init(const struct machine_desc *mdesc) ->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;mdesc->map_io()

涉及到的结构体__struct machine_desc__
```c
struct machine_desc {
	unsigned int		nr;		/* architecture number	*/
	const char		*name;		/* architecture 

	unsigned int		nr_irqs;	/* number of IRQs */

#ifdef CONFIG_ZONE_DMA
	phys_addr_t		dma_zone_size;	/* size of DMA-able area */
#endif
	void			(*init_meminfo)(void);
	void			(*reserve)(void);/* reserve mem blocks	*/
	void			(*map_io)(void);/* IO mapping function	*/
	void			(*init_early)(void);
	void			(*init_irq)(void);
	void			(*init_time)(void);
	void			(*init_machine)(void);
	void			(*init_late)(void);
#ifdef CONFIG_MULTI_IRQ_HANDLER
	void			(*handle_irq)(struct pt_regs *);
#endif
	void			(*restart)(enum reboot_mode, const char *);
};

struct map_desc {
	unsigned long virtual;
	unsigned long pfn;
	unsigned long length;
	unsigned int type;
};
```
例如我们在arch/arm/mach-s3c24xx/mach-smdk2440.c
smdk2440_map_io() ->  
&nbsp;&nbsp;iotable_init(mach_desc, size)

```c
static struct map_desc smdk2440_iodesc[] __initdata = {
	{
		.virtual	= (u32)S3C24XX_VA_ISA_WORD,
		.pfn		= __phys_to_pfn(S3C2410_CS2),
		.length		= 0x10000,
		.type		= MT_DEVICE,
	}, 
};

MACHINE_START(S3C2440, "SMDK2440")
	/* Maintainer: Ben Dooks <ben-linux@fluff.org> */
	.atag_offset	= 0x100,

	.init_irq	= s3c2440_init_irq,
	.map_io		= smdk2440_map_io,
	.init_machine	= smdk2440_machine_init,
	.init_time	= smdk2440_init_time,
MACHINE_END
```
在这一步定义了物理与虚拟地址之间的映射关系，一旦编译完成就不会修改，<font color=red> 静态编译常用于不容易变动的物理地址与虚拟地址之间的映射，例如寄存器的映射。</font>

### 2.2. 页表创建
Linux下的页表映射分为两种，一是Linux自身的页表映射，另一种是ARM32 MMU硬件的映射。这样是为了更大的灵活性，可以映射Linux bits 到 硬件tables 上，例如有YOUNG, DIRTY bits.

参见 linux/arch/arm/include/asm/pagetable-2level.h 注释：
```c
/*
 * Hardware-wise, we have a two level page table structure, where the first
 * level has 4096 entries, and the second level has 256 entries.  Each entry
 * is one 32-bit word.  Most of the bits in the second level entry are used
 * by hardware, and there aren't any "accessed" and "dirty" bits.
 *
 * Linux on the other hand has a three level page table structure, which can
 * be wrapped to fit a two level page table structure easily - using the PGD
 * and PTE only.  However, Linux also expects one "PTE" table per page, and
 * at least a "dirty" bit.
 *
 * Therefore, we tweak the implementation slightly - we tell Linux that we
 * have 2048 entries in the first level, each of which is 8 bytes (iow, two
 * hardware pointers to the second level.)  The second level contains two
 * hardware PTE tables arranged contiguously, preceded by Linux versions
 * which contain the state information Linux needs.  We, therefore, end up
 * with 512 entries in the "PTE" level.
 *
 * This leads to the page tables having the following layout:
 *
 *    pgd             pte
 * |        |
 * +--------+
 * |        |       +------------+ +0
 * +- - - - +       | Linux pt 0 |
 * |        |       +------------+ +1024
 * +--------+ +0    | Linux pt 1 |
 * |        |-----> +------------+ +2048
 * +- - - - + +4    |  h/w pt 0  |
 * |        |-----> +------------+ +3072
 * +--------+ +8    |  h/w pt 1  |
 * |        |       +------------+ +4096
 *
 * See L_PTE_xxx below for definitions of bits in the "Linux pt", and
 * PTE_xxx for definitions of bits appearing in the "h/w pt".
```

32bit的Linux采用三级映射：PGD-->PMD-->PTE，64bit的Linux采用四级映射：PGD-->PUD-->PMD-->PTE，多了个PUD

>PGD - Page Global Directory
PUD - Page Upper Directory
PMD - Page Middle Directory
PTE - Page Table Entry。

在ARM32 Linux采用两层映射，省略了PMD，除非在定义了CONFIG_ARM_LPAE才会使用3级映射。

![页表转换过程](https://raw.githubusercontent.com/JShell07/images/master/kernel_virtual_addr_map/page_to_addr.png)

![1 level page table](https://raw.githubusercontent.com/JShell07/images/master/kernel_virtual_addr_map/1level_page_table.png)

![2 level page table](https://raw.githubusercontent.com/JShell07/images/master/kernel_virtual_addr_map/2level_page_table.png)

更多可以参看<<ARM ® Architecture Reference Manual -- ARMv7-A and ARMv7-R edition>> (ARM DDI 0406C.c (ID051414)) B3 Virtual Memory System Architecture (VMSA)

iotable_init()->  
&nbsp;&nbsp;create_mapping()->  
&nbsp;&nbsp;&nbsp;&nbsp;alloc_init_pud()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;alloc_init_pmd()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;alloc_init_pte()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;set_pte_ext->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;cpu_set_pte_ext()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;cpu_v7_set_pte_ext() （linux/arm/arm/mm/proc-v7-2level.S）

```c
void __init create_mapping(struct map_desc *md)
{
	...
	pgd = pgd_offset_k(addr);
	end = addr + length;
	do {
		unsigned long next = pgd_addr_end(addr, end);

		alloc_init_pud(pgd, addr, next, phys, type);

		phys += next - addr;
		addr = next;
	} while (pgd++, addr != end);	
}

static void __init alloc_init_pte(pmd_t *pmd, unsigned long addr,
				  unsigned long end, unsigned long pfn,
				  const struct mem_type *type)
{
	pte_t *pte = early_pte_alloc(pmd, addr, type->prot_l1);
	do {
		set_pte_ext(pte, pfn_pte(pfn, __pgprot(type->prot_pte)), 0);
		pfn++;
	} while (pte++, addr += PAGE_SIZE, addr != end);
}
```

set_pte_ext()函数，根据配置情况最终指向的是cpu_v7_set_pte_ext()
```c
/*
 *	cpu_v7_set_pte_ext(ptep, pte)
 *
 *	Set a level 2 translation table entry.
 *
 *	- ptep  - pointer to level 2 translation table entry
 *		  (hardware version is stored at +2048 bytes)
 *	- pte   - PTE value to store
 *	- ext	- value for extended PTE bits
 */
ENTRY(cpu_v7_set_pte_ext)
#ifdef CONFIG_MMU
	str	r1, [r0]			@ linux version

	bic	r3, r1, #0x000003f0
	bic	r3, r3, #PTE_TYPE_MASK
	orr	r3, r3, r2
	orr	r3, r3, #PTE_EXT_AP0 | 2

	tst	r1, #1 << 4
	orrne	r3, r3, #PTE_EXT_TEX(1)

	eor	r1, r1, #L_PTE_DIRTY
	tst	r1, #L_PTE_RDONLY | L_PTE_DIRTY
	orrne	r3, r3, #PTE_EXT_APX

	tst	r1, #L_PTE_USER
	orrne	r3, r3, #PTE_EXT_AP1

	tst	r1, #L_PTE_XN
	orrne	r3, r3, #PTE_EXT_XN

	tst	r1, #L_PTE_YOUNG
	tstne	r1, #L_PTE_VALID
	eorne	r1, r1, #L_PTE_NONE
	tstne	r1, #L_PTE_NONE
	moveq	r3, #0

 ARM(	str	r3, [r0, #2048]! )
 THUMB(	add	r0, r0, #2048 )
 THUMB(	str	r3, [r0] )
	ALT_SMP(W(nop))
	ALT_UP (mcr	p15, 0, r0, c7, c10, 1)		@ flush_pte
#endif
	bx	lr
ENDPROC(cpu_v7_set_pte_ext)
```

### 2.3. 动态映射
#### 2.3.1. virtual memory data struct
linux/include/linux/mm_types.h

```c
struct mm_struct {
	struct vm_area_struct *mmap;		/* list of VMAs */
	
#ifdef CONFIG_MMU
	unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
#endif
	unsigned long mmap_base;		/* base of mmap area */
	
	unsigned long task_size;		/* size of task vm space */
	unsigned long highest_vm_end;		/* highest vma end address */
	pgd_t * pgd;
	
	atomic_long_t nr_ptes;			/* Page table pages */

	unsigned long start_code, end_code, start_data, end_data;
	unsigned long start_brk, brk, start_stack;
	unsigned long arg_start, arg_end, env_start, env_end;

	/* Architecture-specific MM context */
	mm_context_t context;
	...
}

struct vm_area_struct {
	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */
					   
	struct vm_area_struct *vm_next, *vm_prev;

	/* Second cache line starts here. */
	struct mm_struct *vm_mm;	/* The address space we belong to. */
	pgprot_t vm_page_prot;		/* Access permissions of this VMA. */
	unsigned long vm_flags;		/* Flags, see mm.h. */

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units, *not* PAGE_CACHE_SIZE */
};
```
我们在kernel 中想要访问当前进程的mm_struct，可以使用<font color=red>current</font>
```c
	struct mm_struct *mm = current->mm;
```

另外，我们可以访问 /proc/<pid>/maps 得到某一进程的内存区域。（/proc/self 始终指向正在运行的进程）

![](https://raw.githubusercontent.com/JShell07/images/master/kernel_virtual_addr_map/proc_self_maps.png)

字段 | 含义
:-: | :-
00400000-0040b000 | vma->vm_start ~ vma->vm_end
r-xp | vma->vm_flags
00000000 | vma->vm_pgoff
08:01 | 主从设备号
1171031 | 设备节点inode 值
bin/cat | 设备节点名字

#### 2.3.2. memory mapping
在使用high mem addr时，我们需要如下函数：
```c
void *kmap(struct page * page)
void kunmap(struct page *page)

void *kmap_atomic(struct page * page)
void kumap_atomic(struct page *page)
```
如果* page 是low mem addr 直接返回，反之，是high mem addr kmap()在内核专用的空间创建特殊的映射。kmap() 可能会睡眠，而kmap_atomic() 会进行原子操作，不允许sleep.

kernel 映射函数 __remap_pfn_range()__

__注意:__
__<font color=red> 如果使用的high mem， vmalloc() 分配的空间，我们只能PAGE_SIZE 的进行映射，他们本身逻辑连续，而物理地址非连续。</font>__

```c
int remap_pfn_range(struct vm_area_struct *vma, unsigned long addr,
		    unsigned long pfn, unsigned long size, pgprot_t prot)
{
	pgd_t *pgd;
	unsigned long next;
	unsigned long end = addr + PAGE_ALIGN(size);
	struct mm_struct *mm = vma->vm_mm;
	int err;

	pfn -= addr >> PAGE_SHIFT;
	pgd = pgd_offset(mm, addr);
	flush_cache_range(vma, addr, end);
	do {
		next = pgd_addr_end(addr, end);
		err = remap_pud_range(mm, pgd, addr, next,
				pfn + (addr >> PAGE_SHIFT), prot);
		if (err)
			break;
	} while (pgd++, addr = next, addr != end);

	return err;
}
```
调用关系如下：
remap_pfn_range()->  
&nbsp;&nbsp;remap_pud_range()->  
&nbsp;&nbsp;&nbsp;&nbsp;remap_pmd_range()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;remap_pte_range()

```c
/*
 * maps a range of physical memory into the requested pages. the old
 * mappings are removed. any references to nonexistent pages results
 * in null mappings (currently treated as "copy-on-access")
 */
static int remap_pte_range(struct mm_struct *mm, pmd_t *pmd,
			unsigned long addr, unsigned long end,
			unsigned long pfn, pgprot_t prot)
{
	pte_t *pte;
	spinlock_t *ptl;

	pte = pte_alloc_map_lock(mm, pmd, addr, &ptl);

	do {
		BUG_ON(!pte_none(*pte));
		set_pte_at(mm, addr, pte, pte_mkspecial(pfn_pte(pfn, prot)));
		pfn++;
	} while (pte++, addr += PAGE_SIZE, addr != end);

	pte_unmap_unlock(pte - 1, ptl);
	return 0;
}

static inline void set_pte_at(struct mm_struct *mm, unsigned long addr,
			      pte_t *ptep, pte_t pteval)
{
	unsigned long ext = 0;

	if (addr < TASK_SIZE && pte_valid_user(pteval)) {
		if (!pte_special(pteval))
			__sync_icache_dcache(pteval);
		ext |= PTE_EXT_NG;
	}

	set_pte_ext(ptep, pteval, ext);
}
```
可以看到与上面的静态映射其实也是差不多的，最终也会调用到set_pte_ext() -> cpu_v7_set_pte_ext() 函数。

__<font color=red>因此，映射的本质都是重新建立页表2-level 或 3-level，并刷新页表</font>__

#### 2.3.3. io memory mapping
ioremap将一个IO地址空间映射到内核的虚拟地址空间上去，便于访问。[ioremap 百度百科](https://baike.baidu.com/item/ioremap/994207)

```c
#define ioremap(cookie,size)		__arm_ioremap((cookie), (size), MT_DEVICE)
#define ioremap_nocache(cookie,size)	__arm_ioremap((cookie), (size), MT_DEVICE)
#define ioremap_cache(cookie,size)	__arm_ioremap((cookie), (size), MT_DEVICE_CACHED)
#define ioremap_wc(cookie,size)		__arm_ioremap((cookie), (size), MT_DEVICE_WC)
#define iounmap				__arm_iounmap
```
&nbsp;&nbsp;ioremap()->  
&nbsp;&nbsp;&nbsp;&nbsp;__arm_ioremap()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__arm_ioremap_caller()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__arm_ioremap_pfn_caller()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__ioremap_page_range() ->__  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ioremap_page_range()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ioremap_pud_range()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ioremap_pmd_range()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ioremap_pte_range()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;set_pte_at()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;set_pte_ext()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;cpu_set_pte_ext()->  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;cpu_v7_set_pte_ext() （linux/arm/arm/mm/proc-v7-2level.S）

可以看见，大致与上面的静态映射，动态映射是相同的， 只是不同点在于__arm_ioremap_pfn_caller（） 函数中调用get_vm_area_caller（）进行vm_area 空间的申请。

```c
void __iomem * __arm_ioremap_pfn_caller(unsigned long pfn,
	unsigned long offset, size_t size, unsigned int mtype, void *caller)
{
	const struct mem_type *type;
	int err;
	unsigned long addr;
	struct vm_struct *area;
	phys_addr_t paddr = __pfn_to_phys(pfn);

#ifndef CONFIG_ARM_LPAE
	/*
	 * High mappings must be supersection aligned
	 */
	if (pfn >= 0x100000 && (paddr & ~SUPERSECTION_MASK))
		return NULL;
#endif

	type = get_mem_type(mtype);
	if (!type)
		return NULL;

	size = PAGE_ALIGN(offset + size);

	/*
	 * Try to reuse one of the static mapping whenever possible.
	 */
	if (size && !(sizeof(phys_addr_t) == 4 && pfn >= 0x100000)) {
		struct static_vm *svm;

		svm = find_static_vm_paddr(paddr, size, mtype);
		if (svm) {
			addr = (unsigned long)svm->vm.addr;
			addr += paddr - svm->vm.phys_addr;
			return (void __iomem *) (offset + addr);
		}
	}

	area = get_vm_area_caller(size, VM_IOREMAP, caller);
 
 	addr = (unsigned long)area->addr;
	area->phys_addr = paddr;

	err = ioremap_page_range(addr, addr + size, paddr,
				 __pgprot(type->prot_pte));

	flush_cache_vmap(addr, addr + size);
	return (void __iomem *) (offset + addr);
}

```

get_vm_area_caller（）函数可以看见是从 VMALLOC_START ~ VMALLOC_END区域内分配一个空间。 

__注:__
由于我们的ioremap（）是将IO 连续的物理地址映射成Kernel 能访问的虚拟地址， 所以本身物理地址的连续性是能保证的。vmalloc.c 中的核心函数get_vm_area_caller（）详细分析可见最后的参看资料。
```c
struct vm_struct *get_vm_area_caller(unsigned long size, unsigned long flags,
				const void *caller)
{
	return __get_vm_area_node(size, 1, flags, VMALLOC_START, VMALLOC_END,
				  NUMA_NO_NODE, GFP_KERNEL, caller);
}
```

```c
static int ioremap_pte_range(pmd_t *pmd, unsigned long addr,
		unsigned long end, phys_addr_t phys_addr, pgprot_t prot)
{
	pte_t *pte;
	u64 pfn;

	pfn = phys_addr >> PAGE_SHIFT;
	pte = pte_alloc_kernel(pmd, addr);
	if (!pte)
		return -ENOMEM;
	do {
		BUG_ON(!pte_none(*pte));
		set_pte_at(&init_mm, addr, pte, pfn_pte(pfn, prot));
		pfn++;
	} while (pte++, addr += PAGE_SIZE, addr != end);
	return 0;
}
```

io_remap_page_range() 如果没有定义，则该函数是等效于**remap_pfn_range()**
```c
#ifndef io_remap_pfn_range
#define io_remap_pfn_range remap_pfn_range
#endif
```

#### 2.3.4. dma memory mapping
参见之前文章 -> [DMA memory mapping](https://jshell07.github.io/2019/06/28/kernel-dma-mem/)

## 参看资料
__basic__
[虚拟地址映射机制--动态、静态](https://www.cnblogs.com/embeded-linux/p/11108930.html)

[Linux的mmap内存映射机制解析](https://blog.csdn.net/zqixiao_09/article/details/51088478)

[ARM MMU页表框架](https://blog.csdn.net/vichie2008/article/details/48274967)

[Linux内存管理 (2)页表的映射过程](https://www.cnblogs.com/arnoldlu/p/8087022.html)

__memory mapping__
[内存映射函数remap_pfn_range学习——示例分析（1）](https://www.cnblogs.com/pengdonglin137/p/8149859.html)

[内存映射函数remap_pfn_range学习——示例分析（2）](https://www.cnblogs.com/pengdonglin137/p/8150462.html)

[内存映射函数remap_pfn_range学习——代码分析（3）](https://www.cnblogs.com/pengdonglin137/p/8150981.html)

__ioremap__
[Linux 字符设备驱动开发基础（五）—— ioremap() 函数解析](https://blog.csdn.net/zqixiao_09/article/details/50859505)

__vmalloc__
[Linux高端内存映射(下)](https://blog.csdn.net/vanbreaker/article/details/7591844)

[高端内存映射之vmalloc分配内存中不连续的页--Linux内存管理(十九)](https://cloud.tencent.com/developer/article/1381130)