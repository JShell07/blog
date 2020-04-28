---
title: measure cpu loading by /proc/stat
date: 2020-04-10 10:38:59
tags: cpu_loading
categories: tools
---

常见的测量CPU loading 的工具有:
- sar
- top
- iostat
- mpstat
- cat /proc/stat

<!--more-->

我们参看busybox 中src code，可以发现top，iostat, mpstat 都是使用到/proc/stat, 或者/proc/<pid>stat 文件并进行解析呈现。

`iostat`
![iostat](https://raw.githubusercontent.com/JShell07/images/master/tools/proc_stat/iostat.png)

`mpstat`
![mpstat](https://raw.githubusercontent.com/JShell07/images/master/tools/proc_stat/mpstat.png)

`cat /proc/1/stat`
![proc_pid_stat](https://raw.githubusercontent.com/JShell07/images/master/tools/proc_stat/proc_pid_stat.png)

## 1. /proc/stat

`CPU time = user + nice + system + idle + iowait + irq + softirq + Steal `

|     item     | remarks                             |
| :----------: | :---------------------------------- |
|  user time   | 普通用户进程占用时间                |
|  nice time   | 高优先级用户进程占用时间            |
| system time  | OS 中运行时间                       |
|  idle time   | CPU 空闲时间                        |
| iowait time  | I/O 等待时间                        |
|   irq time   | 硬中断处理时间                      |
| softirq time | 软中断处理时间                      |
|  steal time  | 类似于guest os 切换等未统计到的时间 |

![proc_stat](https://raw.githubusercontent.com/JShell07/images/master/tools/proc_stat/proc_stat.png)

### 1.1. 原理

```c
/* kernel/fs/proc/stat.c, kernel-4.9.198 */
static int stat_open(struct inode *inode, struct file *file)
{
	size_t size = 1024 + 128 * num_online_cpus();

	/* minimum size to display an interrupt count : 2 bytes */
	size += 2 * nr_irqs;
	return single_open_size(file, show_stat, NULL, size);
}

static int show_stat(struct seq_file *p, void *v)
{
	int i, j;
	u64 user, nice, system, idle, iowait, irq, softirq, steal;
	u64 guest, guest_nice;

	user = nice = system = idle = iowait =
		irq = softirq = steal = 0;
	guest = guest_nice = 0;

	getboottime64(&boottime);

	for_each_possible_cpu(i) {
		user += kcpustat_cpu(i).cpustat[CPUTIME_USER];
		nice += kcpustat_cpu(i).cpustat[CPUTIME_NICE];
		system += kcpustat_cpu(i).cpustat[CPUTIME_SYSTEM];
		idle += get_idle_time(i);
		iowait += get_iowait_time(i);
		irq += kcpustat_cpu(i).cpustat[CPUTIME_IRQ];
		softirq += kcpustat_cpu(i).cpustat[CPUTIME_SOFTIRQ];
		steal += kcpustat_cpu(i).cpustat[CPUTIME_STEAL];
		guest += kcpustat_cpu(i).cpustat[CPUTIME_GUEST];
		guest_nice += kcpustat_cpu(i).cpustat[CPUTIME_GUEST_NICE];
		sum += kstat_cpu_irqs_sum(i);
		sum += arch_irq_stat_cpu(i);
	}
}
```
在show_stat() 中<font color=red>kcpustat_cpu(i).cpustat[CPUTIME_USER]</font> 这个变量时一个关键全局变量。per_cpu 的用法大致是在kernel init 时拷贝CPU NUM 份变量到不同的内存空间，访问时加上CPU NUM(i) 的偏移量。

```c
struct kernel_cpustat {
	u64 cpustat[NR_STATS];
};

#define kstat_cpu(cpu) per_cpu(kstat, cpu)
#define kcpustat_cpu(cpu) per_cpu(kernel_cpustat, cpu)
```

### 1.2. 何时更新
那kernel_cpustat 是在什么时候更新的呢？答案是在Timer 的中断函数中进行更新。

我们可以使用`dump_stack()`函数打印调用栈。在`clockevents_config_and_register()` 进行clock event 注册时有如下关系：

```c
clockevents_config_and_register() ->
    tick_check_new_device() -> 
        tick_setup_device() -> 
            tick_setup_periodic() -> 
                tick_set_periodic_handle()
```

那之后timer 将会在1/HZ 时raise 中断， 则有如下调用关系起来：

```c
tick_handle_periodic()->
    update_process_times() ->
        account_process_tick() ->
            account_system_time()
```

在`account_system_time()` 函数中会进行分类统计CPU 占用时间。
```c
/* linux/kernel/sched/cputime.c, kernel-4.9.18 */
void account_process_tick(struct task_struct *p, int user_tick)
{
	cputime_t cputime, scaled, steal;
	struct rq *rq = this_rq();

	if (vtime_accounting_cpu_enabled())
		return;

	if (sched_clock_irqtime) {
		irqtime_account_process_tick(p, user_tick, rq, 1);
		return;
	}

	cputime = cputime_one_jiffy;
	steal = steal_account_process_time(ULONG_MAX);

	if (steal >= cputime)
		return;

	cputime -= steal;
	scaled = cputime_to_scaled(cputime);

	if (user_tick)
		account_user_time(p, cputime, scaled);
	else if ((p != rq->idle) || (irq_count() != HARDIRQ_OFFSET))
		account_system_time(p, HARDIRQ_OFFSET, cputime, scaled);
	else
		account_idle_time(cputime);
}
```

### 1.3. 怎么分类
接下来的问题是我们怎么知道何时是user, system, idle 等呢？

#### 1.3.1 user or system?

ARM CPU 可以从CPSR reg 得到当前的运行态，下面函数大概也是基于此思想：
```c
/*
tick_handle_periodic() ->
    tick_periodic()
*/
static void tick_periodic(int cpu)
{
	if (tick_do_timer_cpu == cpu) {
		write_seqlock(&jiffies_lock);

		/* Keep track of the next tick event */
		tick_next_period = ktime_add(tick_next_period, tick_period);

		do_timer(1);
		write_sequnlock(&jiffies_lock);
		update_wall_time();
	}

	update_process_times(user_mode(get_irq_regs()));
	profile_tick(CPU_PROFILING);
}

#define user_mode(regs)	\
	(((regs)->ARM_cpsr & 0xf) == 0)

static inline struct pt_regs *get_irq_regs(void)
{
	return __this_cpu_read(__irq_regs);
}  
```

#### 1.3.2. system, irq, softirq ?

__判断idle or system__
主要通过`struct rq -> runqueue` 运行队列上状态判断是否是IDLE。
```c
void account_process_tick(struct task_struct *p, int user_tick)
{
	struct rq *rq = this_rq();
    ...
	if (user_tick)
		account_user_time(p, cputime, scaled);
	else if ((p != rq->idle) || (irq_count() != HARDIRQ_OFFSET))
		account_system_time(p, HARDIRQ_OFFSET, cputime, scaled);
	else
		account_idle_time(cputime);
}
```

__判断system, irq, softirq__

kernel 判断通过thread_info 中<font color=red>preempt_count</font> 进行判断。 
在进入中断时，preempt_count 会进行设定。

```c
#define __irq_enter()					\
	do {						\
		account_irq_enter_time(current);	\
		preempt_count_add(HARDIRQ_OFFSET);	\
		trace_hardirq_enter();			\
	} while (0)

#define in_irq()		        (hardirq_count())
#define in_softirq()		    (softirq_count())
#define in_interrupt()		    (irq_count())
#define in_serving_softirq()	(softirq_count() & SOFTIRQ_OFFSET)

#define hardirq_count()	(preempt_count() & HARDIRQ_MASK)
#define softirq_count()	(preempt_count() & SOFTIRQ_MASK)

static __always_inline int preempt_count(void)
{
	return READ_ONCE(current_thread_info()->preempt_count);
}


void account_system_time(struct task_struct *p, int hardirq_offset,
			 cputime_t cputime, cputime_t cputime_scaled)
{
	int index;

	if ((p->flags & PF_VCPU) && (irq_count() - hardirq_offset == 0)) {
		account_guest_time(p, cputime, cputime_scaled);
		return;
	}

	if (hardirq_count() - hardirq_offset)
		index = CPUTIME_IRQ;
	else if (in_serving_softirq())
		index = CPUTIME_SOFTIRQ;
	else
		index = CPUTIME_SYSTEM;

	__account_system_time(p, cputime, cputime_scaled, index);
}
```

### 1.4. 准确性
通过上面的分析，我们知道数据更新频率是1/HZ。 如果在一个timer 中断周期内有进程的调度，那么我们在timer 周期中断函数统计就可能漏掉了调度前进程占用CPU 的时间。这就最终与我们的Kernel HZ 的配置有一定的关系， 不过一般情况下kenrel 进程切换的频率并没有达到如此频繁程度。

如下图所示， 在前一个Timer 周期内，Process A, Process B 在调度，那么在中断时，我们只统计到了process B， 我们就漏了Process A 占用时间。

![proc_stat_precision](https://raw.githubusercontent.com/JShell07/images/master/tools/proc_stat/proc_stat_precision.png)

引起进程调度的常见原因有：
- 进程调用sleep(), exit() 等函数
- 进程时间片耗尽
- driver 中主动调用schedule()
- 从中断等异常，系统调用返回用户态

我们可以通过如下方式得到当前OS 调度程度 `watch -d -n 1 'cat /proc/sched_debug | grep nr_switches'` 
![cpu_process_switch](https://raw.githubusercontent.com/JShell07/images/master/tools/proc_stat/cpu_process_switch.png)

## 2. 总结
- Linux CPU占用率是根据/proc/stat文件中的数据计算而来；
- /proc/stat中的数据精度为ticks，即1/HZ秒；
- 内核每个ticks会更新一次CPU使用信息；
- CPU 占用率的精度为1/HZ秒, 数据信息单位是ticks

## Reference
[linux cpu usage analysis](http://www.ilinuxkernel.com/files/Linux_CPU_Usage_Analysis.pdf)

[理解 CPU 利用率](https://www.jianshu.com/p/f595ee986b55?from=singlemessage)

[Linux系统中的CPU利用率](https://blog.csdn.net/lihualoveyou/article/details/78392229)

[借助perf工具分析CPU使用率](http://linuxperf.com/?p=36)

[我是如何把CPU使用率从70%降到25%的](https://www.jianshu.com/p/919c75dbc420)

[solaris上应该如何监控CPU使用情况](https://www.iteye.com/blog/sunrise-king-1697486)

[kernel per_cpu](https://blog.csdn.net/dayancn/article/details/51169241 )