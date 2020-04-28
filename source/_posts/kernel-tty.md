---
title: kernel_tty
date: 2020-02-17 18:12:19
tags: tty
categories: drivers
---

## 1. Base
### 1.1. 常见术语
|      术语       | 解释                                                                                             |
| :-------------: | :----------------------------------------------------------------------------------------------- |
|    Terminal     | 终端，是一个电子（或电气）人机交互设备                                                           |
| serial Terminal | TTY 设备的一种，通过串口线连接                                                                   |
|     console     | 控制台，早期PC的键盘显示器，比TTY 终端拥有更多的权限，系统的运行日志、错误信息通常会输出到控制台 |

>第一个Unix终端是一个名字为ASR33的电传打字机，而电传打字机的英文单词为Teletype（或Teletypewritter），缩写为TTY。

<!--more-->

### 1.2. TTY 分类
tty 设备有几个分类：  
- console(通常指键盘，显示器)  
- serial tty(传统意义的HW，输入输出都在一个独立的硬件上，如peripheral uart)
- VT(virtual tty，传统控制台同一时刻，只能有一个终端使用，为满足多用户、应用，Unix/linux又虚拟出6个终端，可以通过键盘的组合键（CTRL+ALT+ F1~F6）将某一个虚拟终端调出来在屏幕上显示)
- PTY(Pseudo TTY) 伪终端由pts(pseudo terminal slave)和ptm(pseudo terminal master)组成。

>PTS: 模拟终端完成与shell等应用的TTY 输入、输出需求
>PTM: 将数据通过socket 等形式与真实设备之间进行通行
>
>shell <->  PTS <-> PTM <-> SSHD,XTERM

如果从应用上讲还有如下：
- 软件终端（PUTTY, SecureCRT, mobalxterm, xshell等， 由软件模拟终端） 
- USB, Ethernet终端（通过其他通信协议模拟终端，例如USB CDC的uart）   
- 图形终端（GUI图形也可以说成终端的一种，只是不再是TTY框架中了，TTY主要在字符设备范围）

```
设备号(主, 次)        字符设备                            备注
(5, 0)               /dev/tty                            控制终端（Controlling Terminal）
(5, 1)               /dev/console                        控制台终端（Console Terminal）
(4, 0)               /dev/vc/0 or /dev/tty0              虚拟终端（Virtual Terminal）
(4, 1)               /dev/vc/1 or /dev/tty1              同上
…                    …                                   …
(x, x)               /dev/ttyS0                          串口终端（名称和设备号由驱动自行决定）
…                    …                                   …
(x, x)               /dev/ttyUSB0                        USB转串口终端 
```

## 2. 软件架构

![tty framework structure](https://raw.githubusercontent.com/JShell07/images/master/kernel_tty/linux_tty_framework_structure.png)

### TTY Line Disciplines
line disciplines(线路规程), 可以把它看成设备驱动和应用接口之间的一个适配层, 按照一些定义的标准进行某些字符的转换。例如"\n", "\r" 之间的转换。在Linux 4.9.x中有这些：
```c
/* include/linux/tty.h, line disciplines */
#define N_TTY		0
#define N_SLIP		1
#define N_MOUSE		2
#define N_PPP		3
#define N_STRIP		4
#define N_AX25		5
#define N_X25		6	/* X.25 async */
#define N_6PACK		7
#define N_MASC		8	/* Reserved for Mobitex module <kaz@cafe.net> */
#define N_R3964		9	/* Reserved for Simatic R3964 module */
#define N_PROFIBUS_FDL	10	/* Reserved for Profibus */
#define N_IRDA		11	/* Linux IrDa - http://irda.sourceforge.net/ */
#define N_SMSBLOCK	12	/* SMS block mode - for talking to GSM data */
				/* cards about SMS messages */
#define N_HDLC		13	/* synchronous HDLC */
#define N_SYNC_PPP	14	/* synchronous PPP */
#define N_HCI		15	/* Bluetooth HCI UART */
#define N_GIGASET_M101	16	/* Siemens Gigaset M101 serial DECT adapter */
#define N_SLCAN		17	/* Serial / USB serial CAN Adaptors */
#define N_PPS		18	/* Pulse per Second */
#define N_V253		19	/* Codec control over voice modem */
#define N_CAIF		20      /* CAIF protocol for talking to modems */
#define N_GSM0710	21	/* GSM 0710 Mux */
#define N_TI_WL		22	/* for TI's WL BT, FM, GPS combo chips */
#define N_TRACESINK	23	/* Trace data routing for MIPI P1149.7 */
#define N_TRACEROUTER	24	/* Trace data routing for MIPI P1149.7 */
#define N_NCI		25	/* NFC NCI UART */
```
## 3. Data Structure

### 3.1. TTY Core Data Structure
![tty data structure](https://github.com/JShell07/jshell07.github.io/blob/master/images/kernel_tty/tty_data_structure.png?raw=true)

```c
struct tty_struct {
	struct device *dev;
	struct tty_driver *driver;
	const struct tty_operations *ops;
    struct tty_ldisc *ldisc;
	unsigned char *write_buf;
    ...
	struct tty_port *port;
}

struct ktermios {
	tcflag_t c_iflag;		/* input mode flags */
	tcflag_t c_oflag;		/* output mode flags */
	tcflag_t c_cflag;		/* control mode flags */
	tcflag_t c_lflag;		/* local mode flags */
	cc_t c_line;			/* line discipline */
	cc_t c_cc[NCCS];		/* control characters */
	speed_t c_ispeed;		/* input speed */
	speed_t c_ospeed;		/* output speed */
};

struct tty_ldisc {
	struct tty_ldisc_ops *ops;
	struct tty_struct *tty;
};

struct tty_port {
	struct tty_bufhead	buf;		/* Locked internally */
	struct tty_struct	*tty;		/* Back pointer */
	struct tty_struct	*itty;		/* internal back ptr */
	const struct tty_port_operations *ops;	/* Port operations */
	int			count;		/* Usage count */
    ...
	unsigned char		*xmit_buf;	/* Optional buffer */
};

struct tty_driver {
	struct cdev **cdevs;
	const char	*driver_name;
	const char	*name;
	int	major;		/* major device number */
	int	minor_start;	/* start of minor device number */
	unsigned int	num;	/* number of devices allocated */
	short	type;		/* type of tty driver */
	short	subtype;	/* subtype of tty driver */
	struct ktermios init_termios; /* Initial termios */
	unsigned long	flags;		/* tty driver flags */
	struct proc_dir_entry *proc_entry; /* /proc fs entry */

	/* Pointer to the tty data structures */
	struct tty_struct **ttys;
	struct tty_port **ports;
	struct ktermios **termios;
    ...
	const struct tty_operations *ops;
};

```

### 3.2. Serial Driver Data Structure
相关数据结构， `struct uart_driver` 用来联系`struct tty_driver`, `uart_port` 则包含了uart 的一些callback(其中大部分参数都是使用struct uart_port) 和固有属性例如:irq, baudrate, fifosize等。 

![serial data structure](https://raw.githubusercontent.com/JShell07/images/master/kernel_tty/serial_data_structure.png)

```c
struct uart_state {
	struct tty_port		port;
	struct circ_buf		xmit;
    ...
	struct uart_port	*uart_port;
};

struct uart_port {
	unsigned long	iobase;	/* in/out[bwl] */
	unsigned char __iomem	*membase; /* read/write[bwl] */
	
	unsigned int	irq;	/* irq number */
	unsigned long	irqflags;	/* irq flags  */
	unsigned int	uartclk; /* base uart clock */
	unsigned int	fifosize; /* tx fifo size */
	unsigned char	x_char;	/* xon/xoff char */
	unsigned char	regshift; /* reg offset shift */
	unsigned char	iotype;	/* io access style */

	struct uart_state	*state;	/* pointer to parent state */
	struct uart_icount	icount;	/* statistics */

	struct console	*cons;	/* struct console, if any */
	unsigned int	type;	/* port type */
	const struct uart ops	*ops;
	unsigned int line;	/* port index */
	struct device	*dev;	/* parent device */
};

struct uart_driver {
	struct module	*owner;
	const char	*driver_name;
	const char	*dev_name;
	int	major;
	int	minor;
	int nr;
	struct console *cons;

	struct uart_state *state;
	struct tty_driver *tty_driver;
};
```

## 4. Flow
### 4.1. serial 驱动初始化
串口驱动初始化，主要涉及两个函数:
```c
int uart_register_driver(struct uart_driver *drv);
int uart_add_one_port(struct uart_driver *drv, struct uart_port *uport);
```
`uart_register_driver()`将uart_driver 透过`serial_core`中调用`tty_register_driver()`函数注册`tty_driver`结构体，同时根据Major, Minor 注册char设备。 

之后调用`uart_add_one_port()` 将uart_port 绑定到uart_state上，并注册device_attribute,并最终调用`device_register()`,完成设备模型的注册，最终才会产生"/dev/ttyS0"等设备。

![serial initial flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_tty/serial_initial_flow.png)

### 4.1. serial Open Flow

在`uart_add_one_port()`->`tty_port_register_device_attr()`->`tty_cdev_add()`中注册了cdev设备的R/W/Open。
```c
static const struct file_operations tty_fops = {
	.read		= tty_read,
	.write		= tty_write,
	.unlocked_ioctl	= tty_ioctl,
	.open		= tty_open,
	.release	= tty_release,
};
```

因此，从数据结构上观察`struct file_operation`-> `struct tty_operation` -> `struct tty_port_operation` -> `struct uart_ops`，如下图：

![linux_tty_open](https://raw.githubusercontent.com/JShell07/images/master/kernel_tty/tty_open_flow.png)

### 4.2. serial Read Flow
Read 流程可以分为两个部分， 用户空间与硬件的RX 中断函数。可以理解成消费者与生产者的关系。

#### 4.2.1. Consumer
用户空间调用下来的read， 可以看作是消费者。

![read flow consumer](https://raw.githubusercontent.com/JShell07/images/master/kernel_tty/tty_read_flow_consumer.png)

#### 4.2.2. Provider
在中断服务程序中将收到的ch，扮演生产者的角色，放置到tty_port.tty_buffer, 如果空间不够则调用`__tty_buffer_request_room()` 最小分配256 Bytes.

```c
int __tty_insert_flip_char(struct tty_port *port, unsigned char ch, char flag)
{
	struct tty_buffer *tb;
	int flags = (flag == TTY_NORMAL) ? TTYB_NORMAL : 0;

	if (!__tty_buffer_request_room(port, 1, flags))
		return 0;

	tb = port->buf.tail;
	if (~tb->flags & TTYB_NORMAL)
		*flag_buf_ptr(tb, tb->used) = flag;
	*char_buf_ptr(tb, tb->used++) = ch;

	return 1;
}

void tty_flip_buffer_push(struct tty_port *port)
{
	tty_schedule_flip(port);
}
```
之后调用，`tty_flip_buffer_push()`唤醒work 即`tty_flip_buffer_push()`, 调用到`flush_to_ldisc()`工作队列函数，将tty_buf的数据拷贝到`struct n_tty_data.read_buf`，
`kill_fasync()`负责唤醒用户空间的异步进程，`wake_up_interruptible_poll()`唤醒在discipline read 时的读等待队列。

![read flow provider](https://raw.githubusercontent.com/JShell07/images/master/kernel_tty/tty_read_flow_provider.png)

### 4.3. serial Write Flow
数据流向是userspace data -> `tty_struct.write_buf`-> `uart_state.xmit`。并且在n_tty.c 中的line routine（discipline）中做了回显的操作。

在Write 的流程中涉及到了如下缓冲区域：
- tty_struct.write_buf (char *)
- uart_state.xmit (circ_buf)
- HW 的TX-FIFO

![tty write flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_tty/tty_write_flow.png)

## Reference
[Linux TTY framework(1)_基本概念](http://www.wowotech.net/tty_framework/tty_concept.html)

[Linux TTY framework(2)_软件架构](http://www.wowotech.net/tty_framework/tty_architecture.html)

[Linux TTY framework(3)_从应用的角度看TTY设备](http://www.wowotech.net/tty_framework/application_view.html)

[tty驱动分析](http://www.wowotech.net/tty_framework/435.html)