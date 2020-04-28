---
title: kernel_tickless_idle
date: 2020-04-15 11:15:18
tags:
    - timer
categories:
    - drivers
---

在嵌入式设备中对于高功耗都避之若浼。IOT 物联网，手机等更是看中设备的电池使用时间。省电节约功耗基本从考虑降低频率（手机CPU 的大小核），关闭暂未使用模块，睡眠等方向考虑。 在kernel 中就有tickless timer，通过在OS IDLE 时减少scheduling-clock ticks，节省功耗。下面主要分析kernel-4.9.198 Idle dynticks system(tickless idle)。

<!--more-->
## 1. Base
kernel Timer system 中常见有如下方法管理：
- 周期时钟 (schedule-clock interrupts, CONFIG_HZ_PERIODIC=y)  
- idle 时忽略timer tick (tickless idle， CONFIG_NO_HZ_IDLE=y or CONFIG_NO_HZ=y, kernel default choose)
- idle 时或者只有一个task run 时不需要调度，忽略CPU 的调度时钟滴答(CONFIG_NO_HZ_FULL=y)

在kernel/time/Kconfig 可以看见其配置信息。
```kconfig
config NO_HZ_IDLE
        bool "Idle dynticks system (tickless idle)"
        select NO_HZ_COMMON
        help
          This option enables a tickless idle system: timer interrupts
          will only trigger on an as-needed basis when the system is idle.
          This is usually interesting for energy saving.

          Most of the time you want to say Y here.

config NO_HZ_FULL
        bool "Full dynticks system (tickless)"
        # We need at least one periodic CPU for timekeeping
        select NO_HZ_COMMON
        select RCU_NOCB_CPU
        select VIRT_CPU_ACCOUNTING_GEN
        select IRQ_WORK
        help
         Adaptively try to shutdown the tick whenever possible, even when
         the CPU is running tasks. Typically this requires running a single
         task on the CPU. Chances for running tickless are maximized when
         the task mostly runs in userspace and has few kernel activity.
```

![config_tickless](https://raw.githubusercontent.com/JShell07/images/master/kernel_timer_system/tickless_idle/config_tickless_idle.png)

![config_tickless_help](https://raw.githubusercontent.com/JShell07/images/master/kernel_timer_system/tickless_idle/config_tickless_idle_help.png)

Tick，即周期性产生的 timer 中断事件，可用于系统时间管理、进程信息统计、低精度 timer 处理等等。
`低分辨率定时器`
低分辨率定时器是<font color=red>基于 HZ 来实现</font>的，精度为 1/HZ。对定时器精度要求不高的内核模块还在大量使用低分辨率定时器，例如 CPU DVFS，CPU Hotplug 等。内核通过 time_list 结构体来描述低分辨率定时器。

`高精度定时器`
高精度定时器可以提供<font color=red>纳秒级别的定时精度</font>，以满足对时间精度要求严格的内核模块，例如音频模块，内核通过 hrtimer 结构体来描述高精度定时器。在系统启动的开始阶段，高精度定时器只能工作在低精度周期模式，在条件满足之后的某个阶段就会切换到高精度单触发模式。在系统切换为高精度timer 后，通过one_shot 模拟周期timer，为系统提供周期tick。

`tickless`
动态时钟，并不是真正没有tick， 只是在idle 时停掉tick 一段时间。

## 2. tickless 何时启动？

通过搜索tickless 关键函数`tick_nohz_stop_sched_tick()` 的调用， tickless 启动主要有两个时机点：
- cpu idle 时
- irq exit 时，尝试继续tickless

```c
/* sched tick emulation and no idle tick control/stats*/
struct tick_sched {
	struct hrtimer			sched_timer;
	unsigned long			check_clocks;
	enum tick_nohz_mode		nohz_mode;
	ktime_t				last_tick;
	int				inidle;
	int				tick_stopped;
	unsigned long			idle_jiffies;
	unsigned long			idle_calls;
	unsigned long			idle_sleeps;
	int				idle_active;
	ktime_t				idle_entrytime;
	ktime_t				idle_waketime;
	ktime_t				idle_exittime;
	ktime_t				idle_sleeptime;
	ktime_t				iowait_sleeptime;
	ktime_t				sleep_length;
	unsigned long			last_jiffies;
	u64				next_timer;
	ktime_t				idle_expires;
	int				do_timer_last; /* CPU was the last one doing do_timer before going idle*/
	atomic_t			tick_dep_mask;
};
```

### 2.1. idle 时启动tickless
kernel 总所周知有一个pid=1 的init 进程， 其实还有一个pid=0 的idle 进程。当CPU 进入此低优先级的idle 进程后，我们可以认定此时CPU 是处于idle 状态。 因此，在该进程中可以尝试停掉一段时间的tick。callstack 如下：
cpu_idle_loop -> 
tick_nohz_idle_enter -> 
__tick_nohz_idle_enter -> 
tick_nohz_stop_sched_tick

从`cpu_idle_loop()` 可以看出进入idle 进程 尝试开启tickless， 之后让CPU 进入低功耗`cpuidle_idle_call()`, 之后若有调度事件 则退出tickless, `tick_nohz_idle_exit()`
```c
/* kernel-4.9.198, remove some code */
static void cpuidle_idle_call(void)
{
   /* default idle */
   __asm__  volatile ("wfi');
}

static void cpu_idle_loop(void)
{
	int cpu = smp_processor_id();

	while (1) {
		tick_nohz_idle_enter();

		while (!need_resched()) {
			local_irq_disable();
			arch_cpu_idle_enter();
			cpuidle_idle_call();
			arch_cpu_idle_exit();
		}
		tick_nohz_idle_exit();
	}
}
```

在`__tick_nohz_idle_enter()` 中设定idle_entrytime, idle_active 状态， 并且判断是否可以stop idle tick， 其依据为：
- cpu online
- no need re-schedule
- no local softirq pending
- disable nohz full, or do timer cpu is not current cpu

```c
static ktime_t tick_nohz_start_idle(struct tick_sched *ts)
{
	ktime_t now = ktime_get();
	ts->idle_entrytime = now;
	ts->idle_active = 1;
	return now;
}

static void __tick_nohz_idle_enter(struct tick_sched *ts)
{
	ktime_t now, expires;
	int cpu = smp_processor_id();

	now = tick_nohz_start_idle(ts);

	if (can_stop_idle_tick(cpu, ts)) {
		int was_stopped = ts->tick_stopped;

		ts->idle_calls++;

		expires = tick_nohz_stop_sched_tick(ts, now, cpu);
		if (expires.tv64 > 0LL) {
			ts->idle_sleeps++;
			ts->idle_expires = expires;
		}

		if (!was_stopped && ts->tick_stopped)
			ts->idle_jiffies = ts->last_jiffies;
	}
}
```
tickless 核心关键函数落在`tick_nohz_stop_sched_tick()`，其基本流程是是否能进入tickless。若可以，则计算出合适的ticks 并调用`h`rtimer_start()` 或`tick_program_event()`对timer 重新编程。在后面小节**tickless 停掉tick 数确定** 会较详细分析该函数。

### 2.2. irq_exit 时尝试启动tickless
当有one shot timer中断到来时，我们在 tick_nohz_handler() intr servive 中设定下一次tick唤醒，这样在没有其他中断，进程执行时，也要被唤醒，这样我们的timer 又变成了周期timer。最好的应该是kernel 进行检查是否需要继续进行tickless 操作，因此在irq_exit 中有如下操作：
irq_exit -> 
tick_irq_exit -> 
tick_nohz_irq_exit->
tick_nohz_stop_sched_tick

补充：
irq 在arm 处理流程大致为：
```c
void handle_IRQ(unsigned int irq, struct pt_regs *regs)
{
	__handle_domain_irq(NULL, irq, false, regs);
}

int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
			bool lookup, struct pt_regs *regs)
{
	irq_enter();
    ...
    /* call irq intr func */
	generic_handle_irq(irq);
	
	irq_exit();
}
```
irq_enter(), irq_exit() 就会去做我们的软中断，generic_handle_irq() 实作HW 中断。

## 3. tickless 停掉tick 数确定
tickless 能停掉的ticks 怎样确认了？ 

在只有idle 进程运行时，cpu 需要处理的还有不定的HW 中断等，但是发生时间不能预测。<font color=red>但是第一个即将到期的中断时间是可以知道的，在这个时间到期之前都可以停掉 tick，由此得到需要停掉的tick 数</font>。

另外，停掉 tick 的时间不能超过 clock_event_device 的 max_delta_ns，不然可能会造成 clocksource 的溢出。
如果停掉的ticks 是在下一个tick_period 则不用stop tick。

```c
static ktime_t tick_nohz_stop_sched_tick(struct tick_sched *ts,
					 ktime_t now, int cpu)
{
	/*
	 * Keep the periodic tick, when RCU, architecture or irq_work
	 * requests it.
	 * Aside of that check whether the local timer softirq is
	 * pending. If so its a bad idea to call get_next_timer_interrupt()
	 * because there is an already expired timer, so it will request
	 * immeditate expiry, which rearms the hardware timer with a
	 * minimal delta which brings us back to this place
	 * immediately. Lather, rinse and repeat...
	 */
	if (rcu_needs_cpu(basemono, &next_rcu) || arch_needs_cpu() ||
	    irq_work_needs_cpu() || local_timer_softirq_pending()) {
		next_tick = basemono + TICK_NSEC;
	} else {
		next_tmr = get_next_timer_interrupt(basejiff, basemono);
		ts->next_timer = next_tmr;
		next_tick = next_rcu < next_tmr ? next_rcu : next_tmr;
	}

	/*
	 * If the tick is due in the next period, keep it ticking or
	 * force prod the timer.
	 */
	delta = next_tick - basemono;
	if (delta <= (u64)TICK_NSEC) {
		tick.tv64 = 0;
		timer_clear_idle();
		goto out;
	}

	/*
	 * If this CPU is the one which updates jiffies, then give up
	 * the assignment and let it be taken by the CPU which runs
	 * the tick timer next, which might be this CPU as well. If we
	 * don't drop this here the jiffies might be stale and
	 * do_timer() never invoked. Keep track of the fact that it
	 * was the one which had the do_timer() duty last. If this CPU
	 * is the one which had the do_timer() duty last, we limit the
	 * sleep time to the timekeeping max_deferment value.
	 * Otherwise we can sleep as long as we want.
	 */
	delta = timekeeping_max_deferment();
	if (cpu == tick_do_timer_cpu) {
		tick_do_timer_cpu = TICK_DO_TIMER_NONE;
		ts->do_timer_last = 1;
	} else if (tick_do_timer_cpu != TICK_DO_TIMER_NONE) {
		delta = KTIME_MAX;
		ts->do_timer_last = 0;
	} else if (!ts->do_timer_last) {
		delta = KTIME_MAX;
	}

	/* Calculate the next expiry time */
	if (delta < (KTIME_MAX - basemono))
		expires = basemono + delta;
	else
		expires = KTIME_MAX;

	expires = min_t(u64, expires, next_tick);
	tick.tv64 = expires;

	/*
	 * nohz_stop_sched_tick can be called several times before
	 * the nohz_restart_sched_tick is called. This happens when
	 * interrupts arrive which do not cause a reschedule. In the
	 * first call we save the current tick time, so we can restart
	 * the scheduler tick in nohz_restart_sched_tick.
	 */
	if (!ts->tick_stopped) {
		nohz_balance_enter_idle(cpu);
		calc_load_enter_idle();
		cpu_load_update_nohz_start();

		ts->last_tick = hrtimer_get_expires(&ts->sched_timer);
		ts->tick_stopped = 1;
		trace_tick_stop(1, TICK_DEP_MASK_NONE);
	}

	if (ts->nohz_mode == NOHZ_MODE_HIGHRES)
		hrtimer_start(&ts->sched_timer, tick, HRTIMER_MODE_ABS_PINNED);
	else
		tick_program_event(tick, 1);
out:
	/* Update the estimated sleep length */
	ts->sleep_length = ktime_sub(dev->next_event, now);
	return tick;
}
```

## 4. tickless 何时停止？
在cpu_idle_loop 循环中判断有新的进程被唤醒时退出tickless 模式，可参见前面`cpu_idle_loop()`函数，恢复到tick_period。
tick_nohz_idle_exit->
tick_nohz_restart 

在函数中，会判断高低分辨 timer，若是high resolution timer 则设定one shot 下一次中断来的expires, 反之低分辨率timer 则使用`tick_program_event()` 重新设定clockevent 。
```c
static void tick_nohz_restart(struct tick_sched *ts, ktime_t now)
{
	hrtimer_cancel(&ts->sched_timer);
	hrtimer_set_expires(&ts->sched_timer, ts->last_tick);

	/* Forward the time to expire in the future */
	hrtimer_forward(&ts->sched_timer, now, tick_period);

	if (ts->nohz_mode == NOHZ_MODE_HIGHRES)
		hrtimer_start_expires(&ts->sched_timer, HRTIMER_MODE_ABS_PINNED);
	else
		tick_program_event(hrtimer_get_expires(&ts->sched_timer), 1);
}
```

## 5. tickless 对中断的影响
由于动态时钟出现，jiffies 是滞后的，其一般是在恢复周期时钟时更新。如果进入中断时需要访问jiffies，那么数据是不准确的。因此，进入中断irq_enter, tick_check_idle() 被调用，以此来更新jiffies值。

```c
static void tick_nohz_stop_idle(struct tick_sched *ts, ktime_t now)
{
	update_ts_time_stats(smp_processor_id(), ts, now, NULL);
	ts->idle_active = 0;

	sched_clock_idle_wakeup_event(0);
}

void tick_irq_enter(void)
{
	tick_check_oneshot_broadcast_this_cpu();
	tick_nohz_irq_enter();
}

void irq_enter(void)
{
	rcu_irq_enter();
	if (is_idle_task(current) && !in_interrupt()) {
		/*
		 * Prevent raise_softirq from needlessly waking up ksoftirqd
		 * here, as softirq will be serviced on return from interrupt.
		 */
		local_bh_disable();
		tick_irq_enter();
		_local_bh_enable();
	}
	__irq_enter();
}
```

## Reference 
[Linux Tick 和 Tickless](http://kernel.meizu.com/linux-tick-and-tickless.html)

[Linux时间子系统之八：动态时钟框架（CONFIG_NO_HZ、tickless）](https://blog.csdn.net/DroidPhone/article/details/8112948)

[linux动态时钟探索](http://blog.chinaunix.net/uid-25942458-id-3412358.html)

[动静结合学内核：linux idle进程和init进程浅析](https://www.cnblogs.com/mfrbuaa/p/5152800.html)

[Linux时间子系统之（十三）：Tick Device layer综述](http://www.wowotech.net/timer_subsystem/tick-device-layer.html)

[NO_HZ: 减少调度时钟的滴答](https://blog.csdn.net/zhoudawei/article/details/86427101)
[kernel/Documentation/timers/NO_HZ.txt](https://lwn.net/Articles/549593/)