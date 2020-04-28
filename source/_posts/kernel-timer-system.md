---
title: kernel-timer-system
date: 2019-11-28 14:54:31
tags:
    - timer
categories:
    - drivers
---
## 1. Kernel Timer 软件架构

![Kernel timer structure](https://raw.githubusercontent.com/JShell07/images/master/kernel_timer_system/kernel_timer_structure.png)

<!--more-->

|      术语       | 说明                                                                                                                                                                                         |
| :-------------: | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Global Counter  | free running system counter, 可以参看arm_arm 手册里的Generic Timer->system counter。Rollover 时间至少40年。他提供了一个基础的timeline, 无线延伸                                              |
| CPU local Timer | CPU local timer 可以是Peripheral HW Timer, 或者是CPU 如ARM 含有的Generic Timer -> timer.                                                                                                     |
|   clock event   | 通过timer硬件的 __中断处理函数__ 完成的，在此基础上可以构建tick模块。clock event 是在timeline 上指定点产生event。                                                                            |
|    tick 模块    | 维护了系统的tick，各个 __进程的时间统计__ 也是基于tick的，内核的 __调度器__ 根据这些信息进行调度。__System Load和Kernel Profiling模块__ 也是基于tick的，用于计算系统负荷和进行内核性能剖析。 |
| timekeeping模块 | 系统时间， 每tick 的发生，其值增加。高进度的值，可以来源于clocksource                                                                                                                        |
|    timer lib    | 用户空间需求：1.获取系统时间，time, stime, gettimeofday, 2.定时器功能，settimer, alarm等                                                                                                     |

一个CPU 可以有多个local Clock Event， 但是会选择一个适合的作为tick device。

tick device 工作模式：
- one shot mode(提供高精度的clock event)
- periodic mode

一般有多少个cpu，就会有多少个tick device - local tick device, 在所有device 中会选取一个做global tick device, 负责维护整个系统的jiffies，更新wall clock，计算全局负荷等。

当系统处于高精度timer的时候（tick device处于one shot mode），系统会setup一个特别的高精度timer（可以称之sched timer），该高精度timer会周期性的触发，从而模拟的传统的periodic tick，从而推动了传统低精度timer的运转。因此，一些传统的内核模块仍然可以调用经典的低精度timer模块的接口。 

![Tick Device Layer structure](https://raw.githubusercontent.com/JShell07/images/master/kernel_timer_system/kernel_tick_device_structure.png)

## 2. file structure

|                         文件                         | 说明                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| :--------------------------------------------------: | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|                 time.c<br>timeconv.c                 | 用户空间函数,time, stime, gettimeofday，alarm等，以及转换函数                                                                                                                                                                                                                                                                                                                                                                                                           |
|             time_list.c<br>time_status.c             | 向用户空间提供的调试接口。在用户空间，可以通过/proc/timer_list接口可以获得内核中的时间子系统的相关信息。                                                                                                                                                                                                                                                                                                                                                                |
| posix-timer.c<br>posix-cpu-timers.c<br>posix-clock.c | POSIX timer， clock模块                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|                     alrmtimer.c                      | alarmtimer 模块                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|                        ntp.c                         | NTP 模块                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
|        timerkeeping.c<br>timerkeeping_debug.c        | timerkeeping.c模块                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
|    ick-common.c<br>tick-oneshot.c<br>tick-sched.c    | tick device layer模块。<br>tick-common.c文件是periodic tick模块，用于管理周期性tick事件。<br>tick-oneshot.c文件是for高精度timer的，用于管理高精度tick时间。<br> tick-sched.c是用于dynamic tick的。                                                                                                                                                                                                                                                                      |
|     tick-broadcast.c<br>tick-broadcast-hrtimer.c     | broadcast tick模块。                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
|                    sched_clock.c                     | 通用sched clock模块。这个模块主要是提供一个sched_clock的接口函数，可以获取当前时间点到系统启动之间的纳秒值。底层的HW counter其实是千差万别的，有些平台可以提供64-bit的HW counter，我们可以不使用这个通用sched clock模块（不配置CONFIG_GENERIC_SCHED_CLOCK这个内核选项），而在自己的clock source chip driver中直接提供sched_clock接口。使用通用sched clock模块的好处是：该模块扩展了64-bit的counter，即使底层的HW counter比特数目不足（有些平台HW counter只有32个bit）。 |
|              clocksource.c<br>jiffies.c              | clocksource.c是通用clocksource driver。其实也可以把system tick也看成一个特定的clocksource，其代码在jiffies.c文件中                                                                                                                                                                                                                                                                                                                                                      |
|                     clockevnet.c                     | clockevent 模块                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|                       timer.c                        | 传统的低精度timer 模块， 基本tick                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|                       htimer.c                       | 高精度timer                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

## 3. clocksource
时间其实可以抽象成一条直线, timeline. clock source就是用来抽象一个在指定输入频率的clock下工作的一个counter。输入频率可以确定以什么样的精度来划分timeline(假设输入counter的频率是1GHz，那么一个cycle就是1ns)

clocksource 数据结构如下：
```c
/* linux-4.9.198 code */
/**
 * struct clocksource - hardware abstraction for a free running counter
 *	Provides mostly state-free accessors to the underlying hardware.
 *	This is the structure used for system time.
 *
 * @name:		ptr to clocksource name
 * @list:		list head for registration
 * @rating:		rating value for selection (higher is better)
 *			To avoid rating inflation the following
 *			list should give you a guide as to how
 *			to assign your clocksource a rating
 *			1-99: Unfit for real use
 *				Only available for bootup and testing purposes.
 *			100-199: Base level usability.
 *				Functional for real use, but not desired.
 *			200-299: Good.
 *				A correct and usable clocksource.
 *			300-399: Desired.
 *				A reasonably fast and accurate clocksource.
 *			400-499: Perfect
 *				The ideal clocksource. A must-use where
 *				available.
 * @read:		returns a cycle value, passes clocksource as argument
 * @enable:		optional function to enable the clocksource
 * @disable:		optional function to disable the clocksource
 * @mask:		bitmask for two's complement
 *			subtraction of non 64 bit counters
 * @mult:		cycle to nanosecond multiplier
 * @shift:		cycle to nanosecond divisor (power of two)
 * @max_idle_ns:	max idle time permitted by the clocksource (nsecs)
 * @maxadj:		maximum adjustment value to mult (~11%)
 * @max_cycles:		maximum safe cycle value which won't overflow on multiplication
 * @flags:		flags describing special properties
 * @archdata:		arch-specific data
 * @suspend:		suspend function for the clocksource, if necessary
 * @resume:		resume function for the clocksource, if necessary
 * @owner:		module reference, must be set by clocksource in modules
 *
 * Note: This struct is not used in hotpathes of the timekeeping code
 * because the timekeeper caches the hot path fields in its own data
 * structure, so no line cache alignment is required,
 *
 * The pointer to the clocksource itself is handed to the read
 * callback. If you need extra information there you can wrap struct
 * clocksource into your own struct. Depending on the amount of
 * information you need you should consider to cache line align that
 * structure.
 */
struct clocksource {
	cycle_t (*read)(struct clocksource *cs);
	cycle_t mask;
	u32 mult;
	u32 shift;
	u64 max_idle_ns;
	u32 maxadj;
#ifdef CONFIG_ARCH_CLOCKSOURCE_DATA
	struct arch_clocksource_data archdata;
#endif
	u64 max_cycles;
	const char *name;
	struct list_head list;
	int rating;
	int (*enable)(struct clocksource *cs);
	void (*disable)(struct clocksource *cs);
	unsigned long flags;
	void (*suspend)(struct clocksource *cs);
	void (*resume)(struct clocksource *cs);

	/* private: */
#ifdef CONFIG_CLOCKSOURCE_WATCHDOG
	/* Watchdog related data, used by the framework */
	struct list_head wd_list;
	cycle_t cs_last;
	cycle_t wd_last;
#endif
	struct module *owner;
};
```
### 3.1. register/unregister clocksource

系统中使用cycle_t cycles 进行计数， 为了人们的方便会转换成年月日方式，所以就会用mult, shift（乘法，除法系数）进行转换。

```c
static inline s64 clocksource_cyc2ns(cycle_t cycles, u32 mult, u32 shift)
{
    return ((u64) cycles * mult) >> shift;
} 
```

```c
/* need to calculate mult, shift by caller */
int clocksource_register(struct clocksource *cs);

/* os will calculate mult, shift */
static inline int clocksource_register_hz(struct clocksource *cs,  u32 hz) ;
static inline int clocksource_register_khz(struct clocksource *cs,  u32 khz);

static int clocksource_unbind(struct clocksource *cs);
```

### 3.2. OS选择clock source
主要参看的因素有两个：
- best rate （分辨率越高的）
- 用户空间的选择
```c
static void __clocksource_select(bool skipcur)
```

### 3.3. timecounter 与 cyclecounter
具体来讲， cyclecounter 表示free running system counter 绝对的时间点, 而timecounter 表示counter， 但是表达的是ns 时间单位。

```c
/**
 * struct cyclecounter - hardware abstraction for a free running counter
 *	Provides completely state-free accessors to the underlying hardware.
 *	Depending on which hardware it reads, the cycle counter may wrap
 *	around quickly. Locking rules (if necessary) have to be defined
 *	by the implementor and user of specific instances of this API.
 *
 * @read:		returns the current cycle value
 * @mask:		bitmask for two's complement
 *			subtraction of non 64 bit counters,
 *			see CYCLECOUNTER_MASK() helper macro
 * @mult:		cycle to nanosecond multiplier
 * @shift:		cycle to nanosecond divisor (power of two)
 */
struct cyclecounter {
	cycle_t (*read)(const struct cyclecounter *cc);
	cycle_t mask;
	u32 mult;
	u32 shift;
};

/**
 * struct timecounter - layer above a %struct cyclecounter which counts nanoseconds
 *	Contains the state needed by timecounter_read() to detect
 *	cycle counter wrap around. Initialize with
 *	timecounter_init(). Also used to convert cycle counts into the
 *	corresponding nanosecond counts with timecounter_cyc2time(). Users
 *	of this code are responsible for initializing the underlying
 *	cycle counter hardware, locking issues and reading the time
 *	more often than the cycle counter wraps around. The nanosecond
 *	counter will only wrap around after ~585 years.
 *
 * @cc:			the cycle counter used by this instance
 * @cycle_last:		most recent cycle counter value seen by
 *			timecounter_read()
 * @nsec:		continuously increasing count
 * @mask:		bit mask for maintaining the 'frac' field
 * @frac:		accumulated fractional nanoseconds
 */
struct timecounter {
	const struct cyclecounter *cc;
	cycle_t cycle_last;
	u64 nsec;
	u64 mask;
	u64 frac;
};
```

## 4. clockevent
clockevent 数据结构如下

```c
/**
 * struct clock_event_device - clock event device descriptor
 * @event_handler:	Assigned by the framework to be called by the low
 *			level handler of the event source
 * @set_next_event:	set next event function using a clocksource delta
 * @set_next_ktime:	set next event function using a direct ktime value
 * @next_event:		local storage for the next event in oneshot mode
 * @max_delta_ns:	maximum delta value in ns
 * @min_delta_ns:	minimum delta value in ns
 * @mult:		nanosecond to cycles multiplier
 * @shift:		nanoseconds to cycles divisor (power of two)
 * @state_use_accessors:current state of the device, assigned by the core code
 * @features:		features
 * @retries:		number of forced programming retries
 * @set_state_periodic:	switch state to periodic
 * @set_state_oneshot:	switch state to oneshot
 * @set_state_oneshot_stopped: switch state to oneshot_stopped
 * @set_state_shutdown:	switch state to shutdown
 * @tick_resume:	resume clkevt device
 * @broadcast:		function to broadcast events
 * @min_delta_ticks:	minimum delta value in ticks stored for reconfiguration
 * @max_delta_ticks:	maximum delta value in ticks stored for reconfiguration
 * @name:		ptr to clock event name
 * @rating:		variable to rate clock event devices
 * @irq:		IRQ number (only for non CPU local devices)
 * @bound_on:		Bound on CPU
 * @cpumask:		cpumask to indicate for which CPUs this device works
 * @list:		list head for the management code
 * @owner:		module reference
 */
struct clock_event_device {
	void			(*event_handler)(struct clock_event_device *);
	int			(*set_next_event)(unsigned long evt, struct clock_event_device *);
	int			(*set_next_ktime)(ktime_t expires, struct clock_event_device *);
	ktime_t			next_event;
	u64			max_delta_ns;
	u64			min_delta_ns;
	u32			mult;
	u32			shift;
	enum clock_event_state	state_use_accessors;
	unsigned int		features;
	unsigned long		retries;

	int			(*set_state_periodic)(struct clock_event_device *);
	int			(*set_state_oneshot)(struct clock_event_device *);
	int			(*set_state_oneshot_stopped)(struct clock_event_device *);
	int			(*set_state_shutdown)(struct clock_event_device *);
	int			(*tick_resume)(struct clock_event_device *);

	void			(*broadcast)(const struct cpumask *mask);
	void			(*suspend)(struct clock_event_device *);
	void			(*resume)(struct clock_event_device *);
	unsigned long		min_delta_ticks;
	unsigned long		max_delta_ticks;

	const char		*name;
	int			rating;
	int			irq;
	int			bound_on;
	const struct cpumask	*cpumask;
	struct list_head	list;
	struct module		*owner;
} ____cacheline_aligned;
```

上面重要的有
```c
    /* set next event interrupt by cyclecounter or ktime */
    int			(*set_next_event)(unsigned long evt, struct clock_event_device *);
	int			(*set_next_ktime)(ktime_t expires, struct clock_event_device *);

    /* switch event mode: periodic or oneshot */
	int			(*set_state_periodic)(struct clock_event_device *);
	int			(*set_state_oneshot)(struct clock_event_device *);    
```

### 4.1. 常用函数

```c
extern void clockevents_config_and_register(struct clock_event_device *dev,
					    u32 freq, unsigned long min_delta,
					    unsigned long max_delta);

extern void clockevents_config(struct clock_event_device *dev, u32 freq);
extern void clockevents_register_device(struct clock_event_device *dev);

extern int clockevents_unbind_device(struct clock_event_device *ced, int cpu);
```

在我们register clockevnet 后， 上层的tick layer 可能会考虑替换当前的clockevent device.
替换参考的依据是：
- 首先是CPU local device
- 具备更好的rate

如果current clockevent device 是broadcast device(broadcost clockevent device 主要用于在CPU sleep 时，其他CPU local timer 已经睡眠，在resume时我们可以通过broadcast device 唤醒其他CPU)，需要先close.

```c
/* linux-4.9.198 code */
/*
 * Check, if the new registered device should be used. Called with
 * clockevents_lock held and interrupts disabled.
 */
void tick_check_new_device(struct clock_event_device *newdev)
{
	struct clock_event_device *curdev;
	struct tick_device *td;
	int cpu;

	cpu = smp_processor_id();
	td = &per_cpu(tick_cpu_device, cpu);
	curdev = td->evtdev;

	/* cpu local device ? */
	if (!tick_check_percpu(curdev, newdev, cpu))
		goto out_bc;

	/* Preference decision */
	if (!tick_check_preferred(curdev, newdev))
		goto out_bc;

	if (!try_module_get(newdev->owner))
		return;

	/*
	 * Replace the eventually existing device by the new
	 * device. If the current device is the broadcast device, do
	 * not give it back to the clockevents layer !
	 */
	if (tick_is_broadcast_device(curdev)) {
		clockevents_shutdown(curdev);
		curdev = NULL;
	}
	clockevents_exchange_device(curdev, newdev);
	tick_setup_device(td, newdev, cpu, cpumask_of(cpu));
	if (newdev->features & CLOCK_EVT_FEAT_ONESHOT)
		tick_oneshot_notify();
	return;

out_bc:
	/*
	 * Can the new device be used as a broadcast device ?
	 */
	tick_install_broadcast_device(newdev);
}
```
## Reference
[Linux时间子系统之（二）：软件架构](http://www.wowotech.net/timer_subsystem/time-subsyste-architecture.html)

[Linux时间子系统之（十三）：Tick Device layer 综述](http://www.wowotech.net/timer_subsystem/tick-device-layer.html)

[Linux时间子系统之（十五）：clocksource](http://www.wowotech.net/timer_subsystem/clocksource.html)

[Linux时间子系统之（十六）：clockevent](http://www.wowotech.net/timer_subsystem/clock-event.html)