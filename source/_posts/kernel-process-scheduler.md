---
title: kernel_process_scheduler
date: 2020-04-23 14:57:19
tags:
    - process
categories:
    - schedule
---

OS 中进程的调度会涉及两个主要部分：
- 调度算法(pick_next_task)
- 进程切换(context_switch)，即，参看之前文章[kernel_process_switch](https://jshell07.github.io/blog/2020/04/23/kernel-process-switch/)

调度器(scheduler)是一个OS 的核心部分。不同的任务有不同的需求，任务可以分类为：
1. 普通进程
   - 交互式(interactive )进程
   - 批处理(Batch)进程，处理IO 操作
2. 实时进程

<!--more-->

## 1. 用户空间API
从用户空间来看，进程优先级就是nice value和scheduling priority。对应到内核，有静态优先级、realtime优先级、归一化优先级和动态优先级等概念。

调度策略主要三种类型：
- 普通类型， normal priority (静态性nice 值设定)
- RT 类型， real time priority (rt priority， 决定在何时采用什么策略)
- deadline 类型 (优先级在三类中最高，在一定时间周期类，CPU 资源优先释放给其使用deadline 时间)
  
### 1.1. 常见POSIX API
`nice()` 改变进程的优先级，可以增加（负数）或者减少（正数）优先级。getpriority/setpriority 可以理解成nice() 增强版。
```c
#include <unistd.h>
int nice(int inc);

#include <sys/time.h>
#include <sys/resource.h>
int getpriority(int which, int who);
int setpriority(int which, int who, int prio);
```

### 1.2. RealTime 实时性POSIX API
调度器在运作的时候往往设定一组规则来决定何时，选择哪一个进程进入执行状态，执行多长的时间。那些“规则”就是调度策略。 

>进程优先级有两个范围，一个是nice value，用前两个小节的API来set或者get。另外一个优先级是rt priority，完全碾压nice value这种优先级.

```c
#include <sched.h>

int sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);
int sched_getscheduler(pid_t pid);

int sched_setparam(pid_t pid, const struct sched_param *param);
int sched_getparam(pid_t pid, struct sched_param *param);
```

### 1.3. 统一性的GNU/linux API 
该类主要是用于deadline 类型进程。
下面是GNU/linux 操作系统具有，而POSIX 没有的统一API， 可以完成上面API 的功能(在Ubuntu 上man 并不能找到该函数)。
```c
#include <sched.h>

int sched_setattr(pid_t pid, const struct sched_attr *attr, unsigned int flags);
int sched_getattr(pid_t pid, const struct sched_attr *attr, unsigned int size, unsigned int flags);
```

## 2. 调度器
linux 的调度器伴随着需求在逐步改善。但是万丈高楼平地起，调度器的发展如下表所示；

scheduler | linux version | remarks
:-: | :- | :-
O(n) | 2.4.x | 遍历所有进程，选取优先级最高的，不被打扰的执行时间片，时间复杂度与N 个进程有关, 优先级越高，分配到的运行时间片越多
O(1) | <2.6.23 | 依照不同优先级排列成不同的队列，Runqueue中有两个优先级队列（struct prio_array）分别用来管理active（即时间片还有剩余）和expired（时间片耗尽）的进程。active队列的task一个个的耗尽其时间片，挂入到expired队列。当active队列的task为空的时候，切换active和expired队列，开始一轮新的调度过程。 
CFS | >2.6.23 | Completely Fair Scheduler，完全公平调度策略。引入virtual time 记录每个进程运行时间，选取运行时间片最少的进程执行。同时，将优先级作为时间因子，高优先级会获取更多的时间片。

O(1) 优先级队列示意图如下：
![O(1) runqueue ]()

从Linux 2.6.23开始，Linux引入scheduling class的概念，将调度器模块化，更有利于扩展性。
目前linux 支持的调度器就有：
- RT scheduler
- Deadline scheduler
- CFS scheduler
- Idle scheduler
  
```c
struct sched_class {
    const struct sched_class *next;
    void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
    void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
    void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);
    struct task_struct * (*pick_next_task)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
    /* ... */
}; 
```

sched_class | remarks | policy
:-: | :- | :-
dl_sched_class | deadline调度器 | SCHED_DEADLINE
rt_sched_class | 实时调度器 | SCHED_FIFO、SCHED_RR
fair_sched_class | 完全公平调度器 | SCHED_NORMAL、SCHED_BATCH
idle_sched_class | idle task | SCHED_IDLE 

我们再分析kernel schedule() 函数时，在`context_switch()`之前会进行`pick_next_task()``pick_next_task()`会按照sched_class 优先级进行pick_next_task callback， sched_class 优先级如下：

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

### 2.1. CFS(Completely Fair Scheduler)
CFS 处理SCHED_NORMAL, SCHED_BATCH 级别的进程。

__权重__
CFS 中依据权重对进程分配运行时间。

`进程运行时间 = CPU 时间 x 进程权重 / 就绪队列中全部进程权重和`

以下是nice [-20 - 19] 对应权重, 以1024 对应0 最为基准，每降低1 nice，多获得10% CPU 时间，因此该表可以用公式近似求得`weight = 1024 / 1.25^nice`
```c
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

__调度延迟__
调度延迟就是保证每一个可运行进程都至少运行一次的时间间隔。__调度延迟是动态变化的__。如果固定的话，当系统中runable 进程越多时，每个进程分配的时间在减少，为了保证调度延迟不变，进程切换频率会增加，上下文切换时间开销变大。

因此，调度延迟在nr_running <= 8 时为6ms, 反之为 nr_running * 0.75ms。这个默认0.75 保证每个进程至少运行一定时间才让出CPU，这个至少一定时间就成为__最小粒度时间__。

```c
static unsigned int sched_nr_latency = 8;
/*
 * Minimal preemption granularity for CPU-bound tasks:
 * (default: 0.75 msec * (1 + ilog(ncpus)), units: nanoseconds)
 */
unsigned int sysctl_sched_min_granularity = 750000ULL;

/*
 * Targeted preemption latency for CPU-bound tasks:
 * (default: 6ms * (1 + ilog(ncpus)), units: nanoseconds)
 *
 * NOTE: this latency value is not the same as the concept of
 * 'timeslice length' - timeslices in CFS are of variable length
 * and have no persistent notion like in traditional, time-slice
 * based scheduling concepts.
 *
 * (to see the precise effective timeslice length of your workload,
 *  run vmstat and monitor the context-switches (cs) field)
 */
unsigned int sysctl_sched_latency = 6000000ULL;
static u64 __sched_period(unsigned long nr_running)
{
	if (unlikely(nr_running > sched_nr_latency))
		return nr_running * sysctl_sched_min_granularity;
	else
		return sysctl_sched_latency;
}
```
__虚拟时间__
CFS 与之前O(n/1) 相比没有固定的时间片的概念，而是动态调整。

例如，在6ms 的调度周期内，有两个进程A, B，权重分别为1024, 820。根据之前的公式：`进程运行时间 = CPU 时间 x 进程权重 / 就绪队列中全部进程权重和`

进程A 执行时间 = 6ms * 1024 / (1024 + 820) = 3.3ms
进程B 执行时间 = 6ms * 820 / (1024 + 820) = 2.7ms

但是这样进程A、B 实际执行的时间是不等的，__CFS 想保证每个进程运行时间相等，因此，引入了虚拟时间（vritual time）的概念。__ 

`virtual_time = wall_runtime * NICE_0_LOAD / weight`

virtual_time_process_A = 3.3ms * 1024 / 1024 = 3.3ms
virtual_time_process_B = 2.7ms * 1024 / 820 = 3.3ms

那为什么要引入virtual_time?
进程A,B 实际运行时间不等，我们不能按照之前的方式，利用优先级择优选择，这不是realtime scheduler等算法而是CFS。为了体现公平性，引入virtual_time 所有进程在执行完CPU 分配给他的时间后都一样，通过记录各个进程的virtual_time，我们就知道哪个进程完成了，哪个进程只完成了一部分。

我们可以理解成，进程A，B执行完CPU 分配时间都是100%，可能A 只完成了它的30%, B 完成了它的80%，那么算法就会优先选择只做了30%的进程A 执行。

虚拟时间(vritual time)，作为key 将进程管理在红黑树上。

kernel 中对计算公式进行变形：
```c
virtual_time = wall_runtime * NICE_0_LOAD / weight

virtual_time = (wall_runtime * NICE_0_LOAD * 2^32 / weight) >> 32

virtual_time = (wall_runtime * NICE_0_LOAD * inv_weight) >> 32

inv_weight = 2*32 / weight
```

kernel 中有对应的inv_weight 速查表：
```c
const u32 sched_prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```

在kernel 计算func 如下：
```c
/*
 * delta_exec * weight / lw.weight
 *   OR
 * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHIFT
 *
 * Either weight := NICE_0_LOAD and lw \e sched_prio_to_wmult[], in which case
 * we're guaranteed shift stays positive because inv_weight is guaranteed to
 * fit 32 bits, and NICE_0_LOAD gives another 10 bits; therefore shift >= 22.
 *
 * Or, weight =< lw.weight (because lw.weight is the runqueue weight), thus
 * weight/lw.weight <= 1, and therefore our shift will also be positive.
 */
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
	u64 fact = scale_load_down(weight);
	int shift = WMULT_SHIFT;

	__update_inv_weight(lw);

	if (unlikely(fact >> 32)) {
		while (fact >> 32) {
			fact >>= 1;
			shift--;
		}
	}

	/* hint to use a 32x32->64 mul */
	fact = (u64)(u32)fact * lw->inv_weight;

	while (fact >> 32) {
		fact >>= 1;
		shift--;
	}

	return mul_u64_u32_shr(delta_exec, fact, shift);
}
```

__就绪队列__
per-cpu 都有自己的runqueue。而每一个sched_class 也有自己的管理队列。在`struct rq` 中有`struct cfs_rq, rt_rq, dl_rq`, `struct cfs_rq` 对应CFS 调度类。

FixMe
![struct rq data structure]()

```c
/*
 * This is the main, per-CPU runqueue data structure.
 *
 * Locking rule: those places that want to lock multiple runqueues
 * (such as the load balancing or the thread migration code), lock
 * acquire operations must be ordered by ascending &runqueue.
 */
struct rq {
	unsigned int nr_running;

	/* capture load from *all* tasks on this cpu: */
	struct load_weight load;

	u64 nr_switches;

	struct cfs_rq cfs;
	struct rt_rq rt;
	struct dl_rq dl;
	...
};

struct rb_root {
	struct rb_node *rb_node;
};

struct cfs_rq {
	struct load_weight load;
	unsigned int nr_running;
	u64 min_vruntime;
	struct rb_root tasks_timeline;
	struct rb_node *rb_leftmost; /* 最左边叶子节点 */
}; 
```

CFS 以virtual_time 作为key 的红黑树。调度算法选择左边最需要CPU 运行的进程。

FixMe
![CFS redblack tree]()

```c
struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
	struct rb_node *left = cfs_rq->rb_leftmost;

	if (!left)
		return NULL;

	return rb_entry(left, struct sched_entity, run_node);
}
```

那什么时候将进程添加到cfs_rq 呢？ `在fork()`， set nice, cgroup attach 都会触发：

FixMe
![enqueu_task into cfs_rq]()

```c
static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
	update_rq_clock(rq);
	if (!(flags & ENQUEUE_RESTORE))
		sched_info_queued(rq, p);
	p->sched_class->enqueue_task(rq, p, flags);
}
```
最终调到各个具体的sched_class 中的callback.

strcut task_struct 数据结构关系如下：
FixMe
![struct task data structure]()

```c
struct sched_class {
	const struct sched_class *next;

	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*yield_task) (struct rq *rq);
	bool (*yield_to_task) (struct rq *rq, struct task_struct *p, bool preempt);

	struct task_struct * (*pick_next_task) (struct rq *rq,
						struct task_struct *prev,
						struct pin_cookie cookie);
	
	void (*set_curr_task) (struct rq *rq);
	void (*task_tick) (struct rq *rq, struct task_struct *p, int queued);
	void (*task_fork) (struct task_struct *p);
	void (*task_dead) (struct task_struct *p);

	void (*switched_from) (struct rq *this_rq, struct task_struct *task);
	void (*switched_to) (struct rq *this_rq, struct task_struct *task);
	void (*prio_changed) (struct rq *this_rq, struct task_struct *task,
			     int oldprio);

#ifdef CONFIG_FAIR_GROUP_SCHED
	void (*task_change_group) (struct task_struct *p, int type);
#endif
	...
};
```

`enqueue_entity()` 负责将task 放置到rb tree 上， `cfs_rq_throttled()` 
```c
static void
enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &p->se;

	if (p->in_iowait)
		cpufreq_update_this_cpu(rq, SCHED_CPUFREQ_IOWAIT);

	for_each_sched_entity(se) {
		if (se->on_rq)
			break;
		cfs_rq = cfs_rq_of(se);
		enqueue_entity(cfs_rq, se, flags);

		/*
		 * end evaluation on encountering a throttled cfs_rq
		 */
		if (cfs_rq_throttled(cfs_rq))
			break;
		cfs_rq->h_nr_running++;

		flags = ENQUEUE_WAKEUP;
	}
	
	if (!se)
		add_nr_running(rq, 1);

	hrtick_update(rq);
}

static void
enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
	bool renorm = !(flags & ENQUEUE_WAKEUP) || (flags & ENQUEUE_MIGRATED);
	bool curr = cfs_rq->curr == se;

	/*
	 * If we're the current task, we must renormalise before calling
	 * update_curr().
	 */
	if (renorm && curr)
		se->vruntime += cfs_rq->min_vruntime;

	update_curr(cfs_rq);

	account_entity_enqueue(cfs_rq, se);

	if (flags & ENQUEUE_WAKEUP)
		place_entity(cfs_rq, se, 0);

	if (!curr)
		__enqueue_entity(cfs_rq, se);

	se->on_rq = 1;
	...
}
```

### 组调度

### 带宽控制(对于组调度)
`CONFIG_CFS_BANDWIDTH` enable 时，CFS 带宽控制就会起作用。

### 2.2. Deadline Scheduler
处理SCHED_DEADLINE 级别进程。

## Reference
Base
[Linux系统如何标识进程？](http://www.wowotech.net/process_management/pid.html)
[Linux调度器：用户空间接口](http://www.wowotech.net/process_management/scheduler-API.html)
[Linux调度器：进程优先级](http://www.wowotech.net/process_management/process-priority.html)
[Linux内核分析——第四章 进程调度](https://www.cnblogs.com/20135235my/p/5398066.html)
[O(n)、O(1)和CFS调度器](http://www.wowotech.net/process_management/scheduler-history.html)

进程调度
[CFS调度器（1）-基本原理](http://www.wowotech.net/process_management/447.html)
[CFS调度器（2）-源码解析](http://www.wowotech.net/process_management/448.html)
[CFS调度器（3）-组调度](http://www.wowotech.net/process_management/449.html)
[CFS调度器（4）-PELT(per entity load tracking)](http://www.wowotech.net/process_management/450.html)
[CFS调度器（5）-带宽控制](http://www.wowotech.net/process_management/451.html)
[CFS调度器（6）-总结](http://www.wowotech.net/process_management/452.html)

[deadline调度器之（一）：原理](http://www.wowotech.net/process_management/deadline-scheduler-1.html)
[Deadline调度器之（二）：细节和使用方法](http://www.wowotech.net/process_management/dl-scheduler-2.html)

[Linux进程管理与调度-之-目录导航](https://blog.csdn.net/gatieme/article/details/51456569)