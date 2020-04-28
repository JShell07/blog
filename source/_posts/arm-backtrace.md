---
title: arm_backtrace
date: 2020-04-26 10:48:41
tags: backtrace
categories: tools
---

arm backtrace 的实现主要有两种方式：
1. APCS (逐步被淘汰) 向gcc 传递选项`-mapcs-frame` or `-mapcs`
2. unwind 向gcc 传递选项`-funwind-tables`


<!--more-->

## 1. APCS
APCS (ARM Procedure Call Standard) ARM过程调用标准规范了arm寄存器的使用、过程调用时 出栈和入栈的约定。每次函数调用都需要准从这一套规范，入栈规定的寄存器。

register | alias | remarks
:-: | :- | :-
r11 | FP | frame pointer
r12 | IP | The Intra-Procedure-call scratch register. （可简单的认为暂存SP）
r13 | SP | The Stack Pointer.
r14 | LR | The Link Register.
r15 | PC | The Program Counter.

注：thumb2 模式下有所不同，FP 为R7。

![apcs](https://raw.githubusercontent.com/JShell07/images/master/backtrace/apcs.png)

每个函数都有自己的栈空间，这一部分我们称为栈帧，在函数被调用的时候创建，在函数返回后销毁。stack frame的两个边界分别由FP和SP来限定，其中SP指向栈顶，FP指向栈基址， PC。 在被调用函数callee中Stack 保存了调用函数caller 的{FP, SP, LR, PC}，通过pop 操作就可以知道caller 函数对应的寄存器值。这样递归操作就能知道整个调用栈。

## 2. UNWIND
相较于APCS，消耗更大的栈空间，占用更多寄存器，对性能有影响，unwind 占用额外的段，但不影响性能。在GCC > 4.5 增加新特性。它的原理是`记录每个函数的 入栈指令(一般比APCS的入栈要少的多)到特殊的段.ARM.unwind_idx .ARM.unwind_tab`。那他是怎么产生的呢？GCC 编译时如果有`-funwind-tables` 选项则会生成。

在ld 链接脚本里，有如下定义：
```c
/*linux/arch/arm/kernel/vmlinux.lds */
#ifdef CONFIG_ARM_UNWIND
	/*
	 * Stack unwinding tables
	 */
	. = ALIGN(8);
	.ARM.unwind_idx : {
		__start_unwind_idx = .;
		*(.ARM.exidx*)
		__stop_unwind_idx = .;
	}
	.ARM.unwind_tab : {
		__start_unwind_tab = .;
		*(.ARM.extab*)
		__stop_unwind_tab = .;
	}
#endif
```

我们使用`arm-none-linux-readelf -u vmlinux` 命令读取vmlinux 中unwind 段，使用`arm-none-linux-objdump -D vmlinux` 命令反汇编vmlinux 中函数。 
下面是截取对应某个函数分析：
```asm
/* vmlinux unwind 部分段 */
0x81244d44 <proc_sys_write>: 0x80028400
  Compact model index: 0
  0x02      vsp = vsp + 12
  0x84 0x00 pop {r14}
```
0x81244d44 为函数虚拟地址。0x80028400 为编码，编码对应的函数栈操作的逆过程出栈的伪指令，这样编码的目的是为减少段空间的浪费。

它的结构类似于：
```c
struct unwind_idx {
	unsigned long addr_offset;
	unsigned long insn;
};
```

```asm
/* vmlnux 反汇编部分段 */
81244d44 <proc_sys_write>:
81244d44:	e52de004 	push	{lr}		; (str lr, [sp, #-4]!)
81244d48:	e24dd00c 	sub	sp, sp, #12
81244d4c:	e3a0c001 	mov	ip, #1
81244d50:	e58dc000 	str	ip, [sp]
81244d54:	ebffffc2 	bl	81244c64 <proc_sys_call_handler>
81244d58:	e28dd00c 	add	sp, sp, #12
81244d5c:	e49df004 	pop	{pc}		; (ldr pc, [sp], #4)
```

回溯时，根据PC 值在uwind 段中查询到函数栈操作的反向操作即出栈，进而知道上一次调用函数的地址。以此方法递归则可以知道整个函数调用栈。

__With the -funwind-tables flag:__
```c
Idx Name Size VMA LMA File off Algn
0 .text 0005a600 08000000 08000000 00004000 2**14
CONTENTS, ALLOC, LOAD, CODE  
1 .ARM.exidx 00003fd8 0805a600 0805a600 0005e600 2**2
CONTENTS, ALLOC, LOAD, READONLY, DATA  
2 .ARM.extab 000049d0 0805e5d8 0805e5d8 000625d8 2**2
CONTENTS, ALLOC, LOAD, READONLY, DATA  
3 .rodata 0003e380 08062fc0 08062fc0 00066fc0 2**5  
```

__No flag:__
```c
Sections:
Idx Name Size VMA LMA File off Algn
0 .text 00058b1c 08000000 08000000 00004000 2**14
CONTENTS, ALLOC, LOAD, CODE
1 .ARM.exidx 00000008 08058b1c 08058b1c 0005cb1c 2**2
CONTENTS, ALLOC, LOAD, READONLY, DATA
2 .rodata 0003e380 08058b40 08058b40 0005cb40 2**5
```

## 3. 实现
### 3.1. 在用户空间实现backtrace
通过下面参看链接 [Getting the saved instruction pointer address from a signal handler](https://stackoverflow.com/questions/5397041/getting-the-saved-instruction-pointer-address-from-a-signal-handler) 我们知道在用户空间实现backtrace的方法主要通过:
1. sigaction 获取到ucontext
2. dladdr 获取到符号表的信息
   
当`sigaction()` sa_flags == <font color=red>SA_SIGINFO</font>时，sa_sigaction() handler 第三个参数为`struct ucontext *`，从ucontext 可以得到fp, sp, lr, pc 等寄存器值，我们再借助dladdr() 得到函数函数名。

```c
#include <signal.h>
int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

struct sigaction {
    void     (*sa_handler)(int);
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t   sa_mask;
    int        sa_flags;
    void     (*sa_restorer)(void);
};

struct ucontext {
	unsigned long	  uc_flags;
	struct ucontext  *uc_link;
	stack_t		  uc_stack;
	struct sigcontext uc_mcontext;
	sigset_t	  uc_sigmask;
	/* Allow for uc_sigmask growth.  Glibc uses a 1024-bit sigset_t.  */
	int		  __unused[32 - (sizeof (sigset_t) / sizeof (int))];
	/* Last for extensibility.  Eight byte aligned because some
	   coprocessors require eight byte alignment.  */
 	unsigned long	  uc_regspace[128] __attribute__((__aligned__(8)));
};

struct sigcontext {
	unsigned long trap_no;
	unsigned long error_code;
	unsigned long oldmask;
	unsigned long arm_r0;
	unsigned long arm_r1;
	unsigned long arm_r2;
	unsigned long arm_r3;
	unsigned long arm_r4;
	unsigned long arm_r5;
	unsigned long arm_r6;
	unsigned long arm_r7;
	unsigned long arm_r8;
	unsigned long arm_r9;
	unsigned long arm_r10;
	unsigned long arm_fp;
	unsigned long arm_ip;
	unsigned long arm_sp;
	unsigned long arm_lr;
	unsigned long arm_pc;
	unsigned long arm_cpsr;
	unsigned long fault_address;
};
```

实现arm architecture 的user space 用例。
```c
#include <ucontext.h>
#include <signal.h>

#define sigsegv_outp(x, ...) 	fprintf(stderr, x"\n", ##__VA_ARGS__)
#define REGFORMAT   "%lx"

/* Structure containing information about object searched using
   `dladdr'.  */
typedef struct
{
  __const char *dli_fname;  /* File name of defining object.  */
  void *dli_fbase;      /* Load address of that object.  */
  __const char *dli_sname;  /* Name of nearest symbol.  */
  void *dli_saddr;      /* Exact value of nearest symbol.  */
} Dl_info;
extern int dladdr (__const void *__address, Dl_info *__info) __THROW __nonnull ((2));

static void print_reg(const ucontext_t *uc)
{
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 0, uc->uc_mcontext.arm_r0);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 1, uc->uc_mcontext.arm_r1);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 2, uc->uc_mcontext.arm_r2);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 3, uc->uc_mcontext.arm_r3);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 4, uc->uc_mcontext.arm_r4);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 5, uc->uc_mcontext.arm_r5);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 6, uc->uc_mcontext.arm_r6);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 7, uc->uc_mcontext.arm_r7);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 8, uc->uc_mcontext.arm_r8);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 9, uc->uc_mcontext.arm_r9);
    sigsegv_outp("reg[%02d]     = 0x"REGFORMAT, 10, uc->uc_mcontext.arm_r10);
    sigsegv_outp("FP        = 0x"REGFORMAT, uc->uc_mcontext.arm_fp);
    sigsegv_outp("IP        = 0x"REGFORMAT, uc->uc_mcontext.arm_ip);
    sigsegv_outp("SP        = 0x"REGFORMAT, uc->uc_mcontext.arm_sp);
    sigsegv_outp("LR        = 0x"REGFORMAT, uc->uc_mcontext.arm_lr);
    sigsegv_outp("PC        = 0x"REGFORMAT, uc->uc_mcontext.arm_pc);
    sigsegv_outp("CPSR      = 0x"REGFORMAT, uc->uc_mcontext.arm_cpsr);
    sigsegv_outp("Fault Address = 0x"REGFORMAT, uc->uc_mcontext.fault_address);
    sigsegv_outp("Trap no       = 0x"REGFORMAT, uc->uc_mcontext.trap_no);
    sigsegv_outp("Err Code  = 0x"REGFORMAT, uc->uc_mcontext.error_code);
    sigsegv_outp("Old Mask  = 0x"REGFORMAT, uc->uc_mcontext.oldmask);
}

static void print_call_link(const ucontext_t *uc)
{
	int i = 0;
	Dl_info	dl_info;

	const void **frame_pointer = (const void **)uc->uc_mcontext.arm_fp;
	const void *return_address = (const void *)uc->uc_mcontext.arm_pc;

	sigsegv_outp("\nStack trace:");
	while (return_address) {
		memset(&dl_info, 0, sizeof(Dl_info));
		if (!dladdr((void *)return_address, &dl_info))	break;
		const char *sname = dl_info.dli_sname;
		sigsegv_outp("%02d: %p <%s + %lu> (%s)", ++i, return_address, sname,
			(unsigned long)return_address - (unsigned long)dl_info.dli_saddr,
													dl_info.dli_fname);
		if (dl_info.dli_sname && !strcmp(dl_info.dli_sname, "main")) break;

		if (!frame_pointer)	break;
		return_address = frame_pointer[-1];
		frame_pointer = (const void **)frame_pointer[-3];
	}
	sigsegv_outp("Stack trace end.");
}

static void segv_handler(int signal_number, siginfo_t *info, void *context)
{
	sigsegv_outp("Segmentation Fault!");
	sigsegv_outp("info.si_signo = %d", signal_number);
	if (info) {
		sigsegv_outp("info.si_errno = %d", info->si_errno);
		sigsegv_outp("info.si_code  = %d (%s)", info->si_code,
			(info->si_code == SEGV_MAPERR) ? "SEGV_MAPERR" : "SEGV_ACCERR");
		sigsegv_outp("info.si_addr  = %p\n", info->si_addr);
	}

	if (context) {
		const ucontext_t *uc = (const ucontext_t *)context;

		print_reg(uc);
		print_call_link(uc);
	}

	_exit(0);
}

void setup_sigsegv(void)
{
	struct sigaction sa;
	memset(&sa, 0, sizeof(sa));
	sa.sa_sigaction = segv_handler;
    sa.sa_flags = SA_SIGINFO;
	sigaction(SIGSEGV, &sa, NULL);
}

```
### 3.2. kernel space 的backtrace
在内核中`dump_stack()` 函数对应于不同的arch 实现了backtrace 的功能。下面我们来简单分析一下。

```c
/* linux-4.9.198, linux/lib/dump_stack.c */
static void __dump_stack(void)
{
	dump_stack_print_info(KERN_DEFAULT);
	show_stack(NULL, NULL);
}

void dump_stack_print_info(const char *log_lvl)
{
	printk("%sCPU: %d PID: %d Comm: %.20s %s %s %.*s\n",
	       log_lvl, raw_smp_processor_id(), current->pid, current->comm,
	       print_tainted(), init_utsname()->release,
	       (int)strcspn(init_utsname()->version, " "),
	       init_utsname()->version);

	if (dump_stack_arch_desc_str[0] != '\0')
		printk("%sHardware name: %s\n",
		       log_lvl, dump_stack_arch_desc_str);

	print_worker_info(log_lvl, current);
}
```
`dump_stack_print_info()` 主要dump 当前进程pid，name 等。我们主要分析`show_stack()`

整体flow 参见下面调用图：

![dump_stack flow](https://raw.githubusercontent.com/JShell07/images/master/backtrace/unwind_flow.png)

```c
/* linux/arch/arm/kernel/traps.c */
void show_stack(struct task_struct *tsk, unsigned long *sp)
{
	dump_backtrace(NULL, tsk);
	barrier();
}

#ifdef CONFIG_ARM_UNWIND
/* UNWIND MODE*/
static inline void dump_backtrace(struct pt_regs *regs, struct task_struct *tsk)
{
	unwind_backtrace(regs, tsk);
}
#else
/* APCS MODE */
static void dump_backtrace(struct pt_regs *regs, struct task_struct *tsk)
{
	unsigned int fp, mode;
	int ok = 1;

	printk("Backtrace: ");

	if (!tsk)
		tsk = current;

	if (regs) {
		fp = frame_pointer(regs);
		mode = processor_mode(regs);
	} else if (tsk != current) {
		fp = thread_saved_fp(tsk);
		mode = 0x10;
	} else {
		asm("mov %0, fp" : "=r" (fp) : : "cc");
		mode = 0x10;
	}

	if (!fp) {
		pr_cont("no frame pointer");
		ok = 0;
	} else if (verify_stack(fp)) {
		pr_cont("invalid frame pointer 0x%08x", fp);
		ok = 0;
	} else if (fp < (unsigned long)end_of_stack(tsk))
		pr_cont("frame pointer underflow");
	pr_cont("\n");

	if (ok)
		c_backtrace(fp, mode);
}
#endif
```

ok， 我们接下来分析unwind_backtrace(), 在该函数中借助于gcc 的build_in 参数得到FP, LR 等，PC 就设定为当前unwind_backtrace()。得到我们想要的struct stackframe frame 数据后，在while (1) 中进行回溯打印整个调用栈。

__builtin_frame_address(0), __builtin_return_address(0)等为gcc 内建函数。可以参看如下文章：
[gcc 内建函数](https://zhuanlan.zhihu.com/p/55771717)

[__builtin_frame_address, __builtin_return_address](https://www.ibm.com/support/knowledgecenter/SSGH2K_12.1.0/com.ibm.xlc121.aix.doc/compiler_ref/bif_builtin_frame_address_builtin_return_address.html)

[Getting the Return or Frame Address of a Function](https://gcc.gnu.org/onlinedocs/gcc/Return-Address.html)

```c
register unsigned long current_stack_pointer asm ("sp");

struct stackframe {
	/*
	 * FP member should hold R7 when CONFIG_THUMB2_KERNEL is enabled
	 * and R11 otherwise.
	 */
	unsigned long fp;
	unsigned long sp;
	unsigned long lr;
	unsigned long pc;
};

void unwind_backtrace(struct pt_regs *regs, struct task_struct *tsk)
{
	struct stackframe frame;

	pr_debug("%s(regs = %p tsk = %p)\n", __func__, regs, tsk);

	if (!tsk)
		tsk = current;

	if (regs) {
		arm_get_current_stackframe(regs, &frame);
		/* PC might be corrupted, use LR in that case. */
		if (!kernel_text_address(regs->ARM_pc))
			frame.pc = regs->ARM_lr;
	} else if (tsk == current) {
		frame.fp = (unsigned long)__builtin_frame_address(0);
		frame.sp = current_stack_pointer;
		frame.lr = (unsigned long)__builtin_return_address(0);
		frame.pc = (unsigned long)unwind_backtrace;
	} else {
		/* task blocked in __switch_to */
		frame.fp = thread_saved_fp(tsk);
		frame.sp = thread_saved_sp(tsk);
		/*
		 * The function calling __switch_to cannot be a leaf function
		 * so LR is recovered from the stack.
		 */
		frame.lr = 0;
		frame.pc = thread_saved_pc(tsk);
	}

	while (1) {
		int urc;
		unsigned long where = frame.pc;

		urc = unwind_frame(&frame);
		if (urc < 0)
			break;
		dump_backtrace_entry(where, frame.pc, frame.sp - 4);
	}
}
```

unwind_frame() 主要的作用是利用`unwind_find_idx()` 函数在`.ARM.unwind_idx` 段中找到当前PC 对应的idx。借此也找到了`.ARM.unwind_tab` 对应的出栈操作，并用`unwind_exec_insn()`函数进行出栈操作， 出栈完成后，FP, SP, LR, PC 寄存器已经更新，在最后更新`struct stackframe frame` 以便下一次的递归回溯。

```c
int unwind_frame(struct stackframe *frame)
{
	unsigned long low;
	const struct unwind_idx *idx;
	struct unwind_ctrl_block ctrl;

	/* store the highest address on the stack to avoid crossing it*/
	low = frame->sp;
	ctrl.sp_high = ALIGN(low, THREAD_SIZE);

	pr_debug("%s(pc = %08lx lr = %08lx sp = %08lx)\n", __func__,
		 frame->pc, frame->lr, frame->sp);

	if (!kernel_text_address(frame->pc))
		return -URC_FAILURE;

	idx = unwind_find_idx(frame->pc);
	if (!idx) {
		pr_warn("unwind: Index not found %08lx\n", frame->pc);
		return -URC_FAILURE;
	}

	ctrl.vrs[FP] = frame->fp;
	ctrl.vrs[SP] = frame->sp;
	ctrl.vrs[LR] = frame->lr;
	ctrl.vrs[PC] = 0;

	if (idx->insn == 1)
		/* can't unwind */
		return -URC_FAILURE;
	else if ((idx->insn & 0x80000000) == 0)
		/* prel31 to the unwind table */
		ctrl.insn = (unsigned long *)prel31_to_addr(&idx->insn);
	else if ((idx->insn & 0xff000000) == 0x80000000)
		/* only personality routine 0 supported in the index */
		ctrl.insn = &idx->insn;
	else {
		pr_warn("unwind: Unsupported personality routine %08lx in the index at %p\n",
			idx->insn, idx);
		return -URC_FAILURE;
	}

	/* check the personality routine */
	if ((*ctrl.insn & 0xff000000) == 0x80000000) {
		ctrl.byte = 2;
		ctrl.entries = 1;
	} else if ((*ctrl.insn & 0xff000000) == 0x81000000) {
		ctrl.byte = 1;
		ctrl.entries = 1 + ((*ctrl.insn & 0x00ff0000) >> 16);
	} else {
		pr_warn("unwind: Unsupported personality routine %08lx at %p\n",
			*ctrl.insn, ctrl.insn);
		return -URC_FAILURE;
	}

	ctrl.check_each_pop = 0;

	while (ctrl.entries > 0) {
		int urc;
		if ((ctrl.sp_high - ctrl.vrs[SP]) < sizeof(ctrl.vrs))
			ctrl.check_each_pop = 1;
		urc = unwind_exec_insn(&ctrl);
		if (urc < 0)
			return urc;
		if (ctrl.vrs[SP] < low || ctrl.vrs[SP] >= ctrl.sp_high)
			return -URC_FAILURE;
	}

	if (ctrl.vrs[PC] == 0)
		ctrl.vrs[PC] = ctrl.vrs[LR];

	/* check for infinite loop */
	if (frame->pc == ctrl.vrs[PC])
		return -URC_FAILURE;

	frame->fp = ctrl.vrs[FP];
	frame->sp = ctrl.vrs[SP];
	frame->lr = ctrl.vrs[LR];
	frame->pc = ctrl.vrs[PC];

	return URC_OK;
}
```

## Reference
[How stack trace works on ARM](https://sudonull.com/post/10719-How-stack-trace-works-on-ARM)

[Stack backtrace 的实现](http://www.alivepea.me/prog/how-backtrace-work/)

[arm上backtrace的分析与实现原理](https://cloud.tencent.com/developer/article/1599605)

[ARM FP寄存器及frame pointer介绍](https://blog.csdn.net/cosmoslhf/article/details/38143505)

[ARM平台(海思)unwind栈回溯的实现](https://blog.csdn.net/Callon_H/article/details/104012933)

[谈谈Linux的栈回溯与妙用](https://www.sohu.com/a/256793414_467784)

在用户空间实现backtrace
[Getting the saved instruction pointer address from a signal handler](https://stackoverflow.com/questions/5397041/getting-the-saved-instruction-pointer-address-from-a-signal-handler)

[How to get fullstacktrace using _Unwind_Backtrace on SIGSEGV](https://stackoverflow.com/questions/6254058/how-to-get-fullstacktrace-using-unwind-backtrace-on-sigsegv)

kernel dump_stack分析
[ARM 架构 dump_stack 实现分析(2.0 调用时序)](https://www.veryarm.com/35743.html)
[内核性能调试–ftrace](http://www.alivepea.me/kernel/kernel-trace/)
[linux内核中打印栈回溯信息 - dump_stack()函数分析](https://blog.csdn.net/jasonchen_gbd/article/details/45585133)
[内核符号表的生成和查找过程](https://blog.csdn.net/jasonchen_gbd/article/details/44025681)