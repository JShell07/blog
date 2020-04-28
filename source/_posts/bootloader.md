---
title: bootloader
date: 2018-08-25 23:48:53
tags:
    - bootloader
    - u-boot
categories:
    - bootloader
---

## 1. BootLoader

>The computer first executes a relatively small program stored in read-only memory (ROM) along with a small amount of needed data, to access the nonvolatile device or devices from which the operating system programs and data can be loaded into RAM.

[bootloader -- wiki](https://en.wikipedia.org/wiki/Booting#BOOT-LOADER)

![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/bootloader.jpg)

<!--more-->

## 2. U-Boot

[Das U-Boot](https://en.wikipedia.org/wiki/Das_U-Boot) -- the Universal Boot Loader
![](https://gss2.bdstatic.com/-fo3dSag_xI4khGkpoWK1HF6hhy/baike/w%3D268/sign=3f0275e9828ba61edfeecf29793497cc/a5c27d1ed21b0ef495e2eb7dddc451da81cb3e8e.jpg)

其官网地址为：http://www.denx.de/wiki/U-Boot/WebHome

官网也很贴心的有[指导手册](http://www.denx.de/wiki/DULG/Manual)

>U-Boot是遵循GPL条款的开放源码项目。其源码目录、编译形式与Linux内核很相似，事实上，不少U-Boot源码就是根据相应的Linux内核源程序进行简化而形成的，尤其是一些设备的驱动程序，这从U-Boot源码的注释中能体现这一点。U-Boot不仅仅支持嵌入式Linux系统的引导，它还支持NetBS
D, VxWorks, QNX, RTEMS, ARTOS, LynxOS, android嵌入式操作系统。它还具有丰富的驱动代码及文件系统支持。（UART、Ethernet、SDRAM、Flash，RTC，ext[2|3|4]、FAT，JFFS2、Squashfs、Cramfs、UBIFS等）

[U-Boot GitHub](https://github.com/u-boot/u-boot)

U-Boot的主要作用是：
- 初始化硬件
- 建立内存空间映射

### 2.1. Bootup Stages
BootLoader的启动过程可以使单阶段(single stage)与多阶段(multi-stage)
一般选择多阶段启动提供更复杂的功能，以及更好的可移植性。

![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/Bootup%20Stages.jpg)

### 2.1.1. Stage1
常用汇编代码编写此段代码。例如`u-boot/arch/arm/xxx/start.S`。
Stage1 的主要作用：
- 设定异常向量(exception vector)
- 初始化硬件（CPU速度，时钟频率，关中断，Disable I/D Cache，u-boot毕竟很简单）
- 初始化内存控制器
- 拷贝Rom或者Flash 等上的程序到Ram中（u-boot运行代码）
- 初始化堆栈

![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/bootup_flow_1.jpg)

### 2.1.2. Stage2
主要是C语言代码，ARM体系一般是在lib_arm/board.c中start_armboot()。Stage2 主要完成：
- 该阶段使用到的外围硬件初始化（Flash、SD、Ethernet等）
- 检测RAM 映射
- 拷贝Kernel 镜像至RAM 中的加载地址
- 设定启动参数
- 启动OS

![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/bootup_flow_2.jpg)

### 2.2. U-Boot Directory
![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/directory.jpg)

| 目录 | 特性 | 备注 |
| :- | :- | :- |
| board | 平台依赖 | 目标板文件。Flash，Ram等驱动 |
| cpu | 平台依赖 | 与处理器相关 |
| lib_arm | 平台依赖 | 处理器体系相关 |
| post | 通用 | 上电自检文件目录 |
| fs | 通用 | 文件系统fat，jffs2,nfs,ubifs,ext[2-4]等 |
| driver | 通用 | 通用设备驱动 |
| net | 通用 | 网络 |
| common | 通用 | 内存检测，Nand等常用命令 |



### 2.3. U-Boot Memory Map
U-Boot 的内存主要分布图如下图：
![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/uboot_mem_map.jpg)


### 2.4. U-Boot Parameters

| 名称 | 默认值 | 备注 |
| :- | :- | :- |
| bootargs | noinitrd root=/dev/mtdblock4 init=/linuxrc console=ttySAC0,115200 | U-Boot 在启动内核之前的等待时间，单位：秒 |
| bootdelay | 1 | 与处理器相关 |
| ethaddr | 00:40:5C:26:0A:5B | 网卡的MAC 地址 |
| ipaddr | 172.20.223.235 | U-Boot 使用的IP 地址 |
| loadaddr | 0x40000000 | 下载二进制文件时的默认地址 |
| bootcmd | nand read C0008000 600000 500000; bootm C0008000 | 当使用boot 命令启动系统时所执行的脚本 |

**Parameters Store**

globa data的RAM地址一般是存放在ARM 的`R8` 通用寄存器中。定义见`include/asm-generial/global_data.h`。

注：**具体的位置可能因为U-Boot 代码的维护更新在变动，具体请参考实际的GitHub 版本**

```c
#define DECLARE_GLOBAL_DATA_PTR register volatile gd_t*d asm("r8")

gd = (gd_t *)(_armboot_start - CONFIG_SYS_MALLOC_LEN - sizeof(gd_t));

gd->bd = (bd_t *)((char*)gd - sizeof(btd_t));
```

```c
typedef struct global_data {
	bd_t *bd;
	unsigned long flags;
	unsigned int baudrate;
	unsigned long cpu_clk;		/* CPU clock in Hz!		*/
	unsigned long bus_clk;
	unsigned long pci_clk;
	unsigned long mem_clk;

#ifdef CONFIG_BOARD_TYPES
	unsigned long board_type;
#endif
	unsigned long env_addr;		/* Address  of Environment struct */

	unsigned long ram_base;		/* Base address of RAM used by U-Boot */
	unsigned long ram_top;		/* Top address of RAM used by U-Boot */
	unsigned long relocaddr;	/* Start address of U-Boot in RAM */
	phys_size_t ram_size;		/* RAM size */
	unsigned long irq_sp;		/* irq stack pointer */
	unsigned long start_addr_sp;	/* start_addr_stackpointer */
	unsigned long reloc_off;
	struct global_data *new_gd;	/* relocated global data */

	char env_buf[32];		/* buffer for env_get() before reloc. */

#if CONFIG_VAL(SYS_MALLOC_F_LEN)
	unsigned long malloc_base;	/* base address of early malloc() */
	unsigned long malloc_limit;	/* limit address */
	unsigned long malloc_ptr;	/* current address */
#endif

    ... ...
#ifdef CONFIG_BOOTSTAGE
	struct bootstage_data *bootstage;	/* Bootstage information */
	struct bootstage_data *new_bootstage;	/* Relocated bootstage info */
#endif
} gd_t;
```

bd_t 保存于板子相关的配置参数。具体可见`include/asm-generial/u-boot.h`。

```c
typedef struct bd_info {
	unsigned long	bi_memstart;	/* start of DRAM memory */
	phys_size_t	bi_memsize;	/* size	 of DRAM memory in bytes */
	unsigned long	bi_flashstart;	/* start of FLASH memory */
	unsigned long	bi_flashsize;	/* size	 of FLASH memory */
	unsigned long	bi_flashoffset; /* reserved area for startup monitor */
	unsigned long	bi_sramstart;	/* start of SRAM memory */
	unsigned long	bi_sramsize;	/* size	 of SRAM memory */
#ifdef CONFIG_ARM
	unsigned long	bi_arm_freq; /* arm frequency */
	unsigned long	bi_dsp_freq; /* dsp core frequency */
	unsigned long	bi_ddr_freq; /* ddr frequency */
#endif

	unsigned long	bi_bootflags;	/* boot / reboot flag (Unused) */
	unsigned long	bi_ip_addr;	/* IP Address */
	unsigned char	bi_enetaddr[6];	/* OLD: see README.enetaddr */
	unsigned short	bi_ethspeed;	/* Ethernet speed in Mbps */
	unsigned long	bi_intfreq;	/* Internal Freq, in MHz */
	unsigned long	bi_busfreq;	/* Bus Freq, in MHz */

	ulong	        bi_arch_number;	/* unique id for this board */
	ulong	        bi_boot_params;	/* where this board expects params */

    ... ...
#ifdef CONFIG_NR_DRAM_BANKS
	struct {			/* RAM configuration */
		phys_addr_t start;
		phys_size_t size;
	} bi_dram[CONFIG_NR_DRAM_BANKS];
#endif /* CONFIG_NR_DRAM_BANKS */
} bd_t;
```

在启动之后一般使用RAM 临时存放启动参数，在使用`saveenv` 命令后将会将之写到类似于Flash 存储介质上。

![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/bootup_parameters_store.jpg)

**Parameters Structure**
ARM 采用了自己体系定义的参数结构。它的定义可以参见`arch/arm/include/asm/setup.h`。

```c
struct tag {
	struct tag_header hdr;
	union {
		struct tag_core		core;
		struct tag_mem32	mem;
		struct tag_videotext	videotext;
		struct tag_ramdisk	ramdisk;
		struct tag_initrd	initrd;
		struct tag_serialnr	serialnr;
		struct tag_revision	revision;
		struct tag_videolfb	videolfb;
		struct tag_cmdline	cmdline;

		/*
		 * Acorn specific
		 */
		struct tag_acorn	acorn;

		/*
		 * DC21285 specific
		 */
		struct tag_memclk	memclk;
	} u;
};

#define tag_next(t)	((struct tag *)((u32 *)(t) + (t)->hdr.size))
#define tag_size(type) ((sizeof(struct tag_header) + sizeof(struct type)) >> 2)

/* The list ends with an ATAG_NONE node. */
#define ATAG_NONE	0x00000000

struct tag_header {
	u32 size;
	u32 tag;
};

/* The list must start with an ATAG_CORE node */
#define ATAG_CORE	0x54410001

struct tag_core {
	u32 flags;		/* bit 0 = read-only */
	u32 pagesize;
	u32 rootdev;
};

#define ATAG_MEM	0x54410002
#define ATAG_VIDEOTEXT	0x54410003
... ...
```

U-Boot就是将各种启动参数串联成类似下图结构给OS。
![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/bootup_parameters_structure.jpg)

注：**必须以ATAG_CORE, ATAG_NONE开头与结尾，Kernel启动时会依据此检测参数的有效性。**

### 2.5. Bootup OS

以ARM 体系启动Kernel需满足以下几点：
- R0=0， R1=[机器码], R2=[Kernel启动参数起始地址]
- Disable I/D Cache
- Disable IRQ，FIQ
- CPU in SVC 模式

注：SVC 模式为Supervisor Control 模式，操作系统使用的保护模式，此模式下CPU能访问的数据权限更大，局限小。

![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/bootup_os.jpg)

`Setting`
![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/bootup_os_need_setting.jpg)

`Parameters`
![](https://raw.githubusercontent.com/JShell07/images/master/bootloader/bootup_os_parameters.jpg)

## Reference
[BootLoader -- wiki](https://en.wikipedia.org/wiki/Booting#BOOT-LOADER)

[Das U-Boot -- wiki](https://en.wikipedia.org/wiki/Das_U-Boot)

[U-Boot 百度百科](https://baike.baidu.com/item/U-Boot/10377075)

[The DENX U-Boot and Linux Guide (DULG) for canyonlands](http://www.denx.de/wiki/DULG/Manual)

[u-boot的内存分布和全局数据结构](
https://blog.csdn.net/xiaoaid01/article/details/39700509)

[移植u-boot学习笔记2-----分析启动过程之内存分布](
https://blog.csdn.net/qingkongyeyue/article/details/52423350)

[u-boot 运行时内存的分配](
http://www.360doc.com/content/15/1104/09/6828497_510609834.shtml)
