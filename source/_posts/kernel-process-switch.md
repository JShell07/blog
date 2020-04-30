---
title: kernel_process_switch
date: 2020-04-23 15:21:29
tags:
    - process
categories:
    - schedule
---

任务的调度可以分为两步：
1. 使用某种算法选出下一个将要运行的任务
2. swtich 到该任务并执行

本篇文章主要讲任务是怎样切换。

<!--more-->

## 1. 调度时机
调度可以分为主动或者被动。 主动态的有：
- IO R/W 时发生的阻塞
- user space 进程主动使用sleep()或exit() , 或者kernel space 驱动等使用schedule() 进行状态切换
- 进程在等待signal, mutex 资源时



被动态触发调度的有：
- tick clock, 进程运行时间片用完
- fork() 新进程时，可能引起被动调度
- 从中断、异常及系统调用状态结束返回时，ret_from_sys_call() 判断调度标志是否需要转换，这样考虑是为了效率，在内核态就将需要状态转换
- 若支持preempt，则从抢占模式退出时

## 2. flow

入口都是从`schedule()`。 在前期判断当前运行的task 是否需要将block data 进行flush 操作。关闭抢占，进行调度。
![schedule_flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_process/schedule_flow.png)

```c
/* linux-4.9.198 src */
static inline void sched_submit_work(struct task_struct *tsk)
{
	if (!tsk->state || tsk_is_pi_blocked(tsk))
		return;
	
    /*
	 * If we are going to sleep and we have plugged IO queued,
	 * make sure to submit it to avoid deadlocks.
	 */
	if (blk_needs_flush_plug(tsk))
		blk_schedule_flush_plug(tsk);
}

asmlinkage __visible void __sched schedule(void)
{
	struct task_struct *tsk = current;

	sched_submit_work(tsk);
	do {
		preempt_disable();
		__schedule(false);
		sched_preempt_enable_no_resched();
	} while (need_resched());
}
```

linux 使用`struct rq` 保存local runqueue，使用`smp_processor_id()` 读取当前运行CPU ID， 并进一步得到local rq。
之后<font color=red>关闭local cpu irq </font> 避免local 中断影响，将当前task 从rq 队列中remove， 通过合适的算法从rq 中pick next task， 最后使用context_switch 切换到next task 上运行。


```c
static void __sched notrace __schedule(bool preempt)
{
	struct task_struct *prev, *next;
	unsigned long *switch_count;
	struct pin_cookie cookie;
	struct rq *rq;
	int cpu;

	cpu = smp_processor_id();
	rq = cpu_rq(cpu);
	prev = rq->curr;

	local_irq_disable();
	rcu_note_context_switch();

	switch_count = &prev->nivcsw;
	if (!preempt && prev->state) {
        deactivate_task(rq, prev, DEQUEUE_SLEEP);
        prev->on_rq = 0;

		switch_count = &prev->nvcsw;
	}
    
	next = pick_next_task(rq, prev, cookie);
	clear_tsk_need_resched(prev);
	clear_preempt_need_resched();
	rq->clock_skip_update = 0;

	if (likely(prev != next)) {
		rq->nr_switches++;
		rq->curr = next;
		++*switch_count;

		rq = context_switch(rq, prev, next, cookie); /* unlocks the rq */
	} 
}
```
### 2.1. pick next task
在另一篇中讲解进程调度算法。

[kernel_process_management](https://jshell07.github.io/blog/2020/04/23/kernel-process-scheduler/)

`pick_next_task()`会按照sched_class 优先级进行pick_next_task callback， sched_class 优先级如下：
```c
sched_class_hightest(stop_sched_class) -> 
dl_sched_class -> 
rt_sched_class ->
fair_sched_class ->
idle_sched_class ->
NULL
```

```c
/* linux-4.9.198/kernel/sched/core.c */
#define sched_class_highest (&stop_sched_class)
#define for_each_class(class) \
   for (class = sched_class_highest; class; class = class->next)

static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct pin_cookie cookie)
{
	const struct sched_class *class = &fair_sched_class;
	struct task_struct *p;

	/*
	 * Optimization: we know that if all tasks are in
	 * the fair class we can call that function directly:
	 */
	if (likely(prev->sched_class == class &&
		   rq->nr_running == rq->cfs.h_nr_running)) {
		p = fair_sched_class.pick_next_task(rq, prev, cookie);
		if (unlikely(p == RETRY_TASK))
			goto again;

		/* assumes fair_sched_class->next == idle_sched_class */
		if (unlikely(!p))
			p = idle_sched_class.pick_next_task(rq, prev, cookie);

		return p;
	}

again:
	for_each_class(class) {
		p = class->pick_next_task(rq, prev, cookie);
		if (p) {
			if (unlikely(p == RETRY_TASK))
				goto again;
			return p;
		}
	}

	BUG(); /* the idle class will always have a runnable task */
}
```

### 2.2. context_switch

![context_switch flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_process/context_switch_flow.png)

```c
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
	       struct task_struct *next, struct pin_cookie cookie)
{
	struct mm_struct *mm, *oldmm;
	mm = next->mm;
	oldmm = prev->active_mm;

	if (!mm) {
		next->active_mm = oldmm;
		atomic_inc(&oldmm->mm_count);
		enter_lazy_tlb(oldmm, next);
	} else
		switch_mm_irqs_off(oldmm, mm, next);

	if (!prev->mm) {
		prev->active_mm = NULL;
		rq->prev_mm = oldmm;
	}

	switch_to(prev, next, prev);
	barrier();

	return finish_task_switch(prev);
}
```

在此之前，我们需要有一个认知，进程切换的情况可能是：
- 普通进程 -> 普通进程
- 普通进程 -> 内核进程/thread
- 内核进程 -> 普通进程
- 内核进程 -> 内核进程

ps: 普通进程 指user space 进程。

![process_thread_mm.png](https://raw.githubusercontent.com/JShell07/images/master/kernel_process/process_thread_mm.png)

以32 bit ARM 为例， 普通进程之间0~3G 独立虚拟空间, 3~4G 共享内核地址空间。因此，当next 为内核进程时，我们可以使用pre 的`active_mm` 空间，内核进程100% 不会使用到0~3G 的虚拟空间，他只需要的是3~4G 的内核空间的地址转换表。

那主要分为两大类：next 是否为kernel thread。若是，next 借助pre->active_mm（减少分配mm, 映射上等时间的浪费）， 反之，则需要进行switch_mm。 当然，kernel 也考虑到了如果上次也为kernel thread 情况如下， context_switch 中就有了：
```c
	if (!prev->mm) {
		prev->active_mm = NULL;
		rq->prev_mm = oldmm;
	}
```

那pre 应该将active_mm 的引用数减少，以便回收。在之后的`finish_task_switch()` 就有`mmdrop()`的尝试回收：
```c
static struct rq *finish_task_switch(struct task_struct *prev)
	__releases(rq->lock)
{
   	struct mm_struct *mm = rq->prev_mm;
    rq->prev_mm = NULL;
    if (mm)
		mmdrop(mm);
    ...
}
static inline void mmdrop(struct mm_struct *mm)
{
	if (unlikely(atomic_dec_and_test(&mm->mm_count)))
		__mmdrop(mm);
}
```

#### 2.2.1. switch_mm/switch_mm_irqs_off
`switch_mm` 把虚拟内存从一个进程映射切换到新进程中， 

```c
static inline void
switch_mm(struct mm_struct *prev, struct mm_struct *next,
	  struct task_struct *tsk)
{
#ifdef CONFIG_MMU
	unsigned int cpu = smp_processor_id();

	/*
	 * __sync_icache_dcache doesn't broadcast the I-cache invalidation,
	 * so check for possible thread migration and invalidate the I-cache
	 * if we're new to this CPU.
	 */
	if (cache_ops_need_broadcast() &&
	    !cpumask_empty(mm_cpumask(next)) &&
	    !cpumask_test_cpu(cpu, mm_cpumask(next)))
		__flush_icache_all();

	if (!cpumask_test_and_set_cpu(cpu, mm_cpumask(next)) || prev != next) {
		check_and_switch_context(next, tsk);
		if (cache_is_vivt())
			cpumask_clear_cpu(cpu, mm_cpumask(prev));
	}
#endif
}
```

在`check_and_switch_context()` 函数中有考虑是否CPU 是否支持TLB ASID (address space ID)。CPU中往往设计了TLB和Cache。Cache为了更快的访问main memory中的数据和指令，而TLB是为了更快的进行地址翻译而将部分的页表内容缓存到了Translation lookasid buffer中，避免了从main memory访问页表的过程。 进程A，B 在kernel space 可以共享，但是各自的虚拟空间user space 则不同，TLB 势必要进行更新，那怎么才能既保证正确性有保证效率呢？ARM 中引入了TLB，ASID， VMID 来支持区别application, virtual machine 的情况。switch_mm 时flush 掉pre->active_mm, 通过global 标志来区别kernel space or user space, 若为global entry 则无需flush。

但是ASID 能记录的ID 数目是要受限于HW 的bits。在arm32 上是8 bit, 意味着能记录256 个ID。当系统中各个cpu的TLB中的asid合起来不大于256个的时候，系统正常运行，一旦超过256的上限后，我们将全部TLB flush掉，并重新分配ASID，每达到256上限，都需要flush tlb并重新分配HW ASID。具体分配ASID代码如下：

```c
/* linux-4.9.198/arch/arm/mm/context.c */
/*
 * On ARMv6, we have the following structure in the Context ID:
 *
 * 31                         7          0
 * +-------------------------+-----------+
 * |      process ID         |   ASID    |
 * +-------------------------+-----------+
 * |              context ID             |
 * +-------------------------------------+
 *
 * The ASID is used to tag entries in the CPU caches and TLBs.
 * The context ID is used by debuggers and trace logic, and
 * should be unique within all running processes.
 *
 * In big endian operation, the two 32 bit words are swapped if accessed
 * by non-64-bit operations.
 */
#define ASID_BITS	8
#define ASID_MASK	((~0ULL) << ASID_BITS)
#define ASID(mm)	((unsigned int)((mm)->context.id.counter & ~ASID_MASK))
#define ASID_FIRST_VERSION	(1ULL << ASID_BITS)
#define NUM_USER_ASIDS		ASID_FIRST_VERSION

static atomic64_t asid_generation = ATOMIC64_INIT(ASID_FIRST_VERSION);

static u64 new_context(struct mm_struct *mm, unsigned int cpu)
{
	static u32 cur_idx = 1;
	u64 asid = atomic64_read(&mm->context.id);
	u64 generation = atomic64_read(&asid_generation);

	if (asid != 0) {
		u64 newasid = generation | (asid & ~ASID_MASK);

		/*
		 * If our current ASID was active during a rollover, we
		 * can continue to use it and this was just a false alarm.
		 */
		if (check_update_reserved_asid(asid, newasid))
			return newasid;

		/*
		 * We had a valid ASID in a previous life, so try to re-use
		 * it if possible.,
		 */
		asid &= ~ASID_MASK;
		if (!__test_and_set_bit(asid, asid_map))
			return newasid;
	}

	/*
	 * Allocate a free ASID. If we can't find one, take a note of the
	 * currently active ASIDs and mark the TLBs as requiring flushes.
	 * We always count from ASID #1, as we reserve ASID #0 to switch
	 * via TTBR0 and to avoid speculative page table walks from hitting
	 * in any partial walk caches, which could be populated from
	 * overlapping level-1 descriptors used to map both the module
	 * area and the userspace stack.
	 */
	asid = find_next_zero_bit(asid_map, NUM_USER_ASIDS, cur_idx);
	if (asid == NUM_USER_ASIDS) {
		generation = atomic64_add_return(ASID_FIRST_VERSION,
						 &asid_generation);
		flush_context(cpu);
		asid = find_next_zero_bit(asid_map, NUM_USER_ASIDS, 1);
	}

	__set_bit(asid, asid_map);
	cur_idx = asid;
	cpumask_clear(mm_cpumask(mm));
	return asid | generation;
}
```

上面函数考虑了如下几种情况：
1. 若(asid = atomic64_read(&mm->context.id)) != 0，说明此mm 已经分配过software asid（generation＋hw asid）了，因初始值为0。那么new context不过就是将software asid中的旧的generation更新为当前的generation而已。 
2. 若asid == 0， 则需要分配一个新的HW ID
   - 若能找到空闲HW ID则返回software asid(当前generation + 新分配hw asid)
   - 若不能找到，多个CPU 上的old generation 需要被flush 掉, 标记tlb_flush_pending。

有了上面的认知后，在看下面的代码就比较明了。最终，会调用到`cpu_switch_mm()`
```c
void check_and_switch_context(struct mm_struct *mm, struct task_struct *tsk)
{
	unsigned long flags;
	unsigned int cpu = smp_processor_id();
	u64 asid;

	asid = atomic64_read(&mm->context.id);
	if (!((asid ^ atomic64_read(&asid_generation)) >> ASID_BITS)
	    && atomic64_xchg(&per_cpu(active_asids, cpu), asid))
		goto switch_mm_fastpath;

	/* Check that our ASID belongs to the current generation. */
	asid = atomic64_read(&mm->context.id);
	if ((asid ^ atomic64_read(&asid_generation)) >> ASID_BITS) {
		asid = new_context(mm, cpu);
		atomic64_set(&mm->context.id, asid);
	}

	if (cpumask_test_and_clear_cpu(cpu, &tlb_flush_pending)) {
		local_flush_bp_all();
		local_flush_tlb_all();
	}

	atomic64_set(&per_cpu(active_asids, cpu), asid);
	cpumask_set_cpu(cpu, mm_cpumask(mm));

switch_mm_fastpath:
	cpu_switch_mm(mm->pgd, mm);
}
```

汇编级的代码主要做了设置context ID register， translation table base0 register.
```c
/*
 *	cpu_v7_switch_mm(pgd_phys, tsk)
 *	Set the translation table base pointer to be pgd_phys
 *	- pgd_phys - physical address of new TTB
 *
 *	It is assumed that:
 *	- we are not using split page tables
 *
 *	Note that we always need to flush BTAC/BTB if IBE is set
 *	even on Cortex-A8 revisions not affected by 430973.
 *	If IBE is not set, the flush BTAC/BTB won't do anything.
 */
ENTRY(cpu_v7_switch_mm)
#ifdef CONFIG_MMU
	mmid	r1, r1				@ get mm->context.id
	ALT_SMP(orr	r0, r0, #TTB_FLAGS_SMP)
	ALT_UP(orr	r0, r0, #TTB_FLAGS_UP)
#ifdef CONFIG_PID_IN_CONTEXTIDR
	mrc	p15, 0, r2, c13, c0, 1		@ read current context ID
	lsr	r2, r2, #8			@ extract the PID
	bfi	r1, r2, #8, #24			@ insert into new context ID
#endif
	mcr	p15, 0, r1, c13, c0, 1		@ set context ID
	isb
	mcr	p15, 0, r0, c2, c0, 0		@ set TTB 0
	isb
#endif
	bx	lr
ENDPROC(cpu_v7_switch_mm)
```

![context_id_register](https://raw.githubusercontent.com/JShell07/images/master/kernel_process/Context_ID_Register.png)

#### 2.2.2. switch_to
保存prev 通用寄存器和堆栈等， 恢复next 以便切换后运行。

```c
#define switch_to(prev,next,last)					\
do {									\
	last = __switch_to(prev,task_thread_info(prev), task_thread_info(next));	\
} while (0)


ENTRY(__switch_to)
 UNWIND(.fnstart	)
 UNWIND(.cantunwind	)
	add	ip, r1, #TI_CPU_SAVE
 ARM(	stmia	ip!, {r4 - sl, fp, sp, lr} )	@ Store most regs on stack
 THUMB(	stmia	ip!, {r4 - sl, fp}	   )	@ Store most regs on stack
 THUMB(	str	sp, [ip], #4		   )
 THUMB(	str	lr, [ip], #4		   )
	ldr	r4, [r2, #TI_TP_VALUE]
	ldr	r5, [r2, #TI_TP_VALUE + 4]
#ifdef CONFIG_CPU_USE_DOMAINS
	mrc	p15, 0, r6, c3, c0, 0		@ Get domain register
	str	r6, [r1, #TI_CPU_DOMAIN]	@ Save old domain register
	ldr	r6, [r2, #TI_CPU_DOMAIN]
#endif
#ifdef CONFIG_CPU_USE_DOMAINS
	mcr	p15, 0, r6, c3, c0, 0		@ Set domain register
#endif
	add	r4, r2, #TI_CPU_SAVE
 THUMB(	mov	ip, r4			   )
 ARM(	ldmia	r4, {r4 - sl, fp, sp, pc}  )	@ Load all regs saved previously
 THUMB(	ldmia	ip!, {r4 - sl, fp}	   )	@ Load all regs saved previously
 THUMB(	ldr	sp, [ip], #4		   )
 THUMB(	ldr	pc, [ip]		   )
 UNWIND(.fnend		)
ENDPROC(__switch_to)
```

先从prev `thread_info offset #TI_CPU_SAVE` 的 `struct cpu_context_save`， 并将当前r4 ~ r10, r12 ~ r15 进行保存。接着从next 同样的offset `struct cpu_context_save` 恢复寄存器。

```c
struct thread_info {
	unsigned long		flags;		/* low level flags */
	int			preempt_count;	/* 0 => preemptable, <0 => bug */
	mm_segment_t		addr_limit;	/* address limit */
	struct task_struct	*task;		/* main task structure */
	__u32			cpu;		/* cpu */
	__u32			cpu_domain;	/* cpu domain */
	struct cpu_context_save	cpu_context;	/* cpu context */
	__u32			syscall;	/* syscall number */
	__u8			used_cp[16];	/* thread used copro */
	unsigned long		tp_value[2];	/* TLS registers */
	union fp_state		fpstate __attribute__((aligned(8)));
	union vfp_state		vfpstate;
};

DEFINE(TI_CPU_SAVE,		offsetof(struct thread_info, cpu_context));
DEFINE(TI_TP_VALUE,		offsetof(struct thread_info, tp_value));
DEFINE(TI_CPU_DOMAIN,		offsetof(struct thread_info, cpu_domain));

struct cpu_context_save {
	__u32	r4;
	__u32	r5;
	__u32	r6;
	__u32	r7;
	__u32	r8;
	__u32	r9;
	__u32	sl; /* r10, stack limit register, r11, ip - intra-procedure-call scratch register */
	__u32	fp; /* r12, frame pointer register */
	__u32	sp; /* stack pointer register */
	__u32	pc;
	__u32	extra[2];		/* Xscale 'acc' register, etc */
};
```


## Reference
[进程切换分析(1)：基本框架](http://www.wowotech.net/process_management/context-switch-arch.html)
[进程切换分析（2）：TLB处理](http://www.wowotech.net/process_management/context-switch-tlb.html)
[进程切换分析（3）：同步处理](http://www.wowotech.net/process_management/scheudle-sync.html)
[Linux内核浅析-进程调度时机和过程](https://zhuanlan.zhihu.com/p/75760406?from_voters_page=true)