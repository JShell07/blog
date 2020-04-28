---
title: kernel_mtd
date: 2019-06-25 11:43:30
tags:
    - mtd
categories:
    - drivers
---

## 1. Flash 大致分类  
- Nor Flash (intel 开发)
- Nand Flash (Toshiba 开发)
- OneNand Flash(Samsung 开发)

<!--more-->

NAND Flash在容量、功耗、使用寿命、写速度快、芯片面积小、单元密度高、擦除速度快、成本低等方面的优势使其成为高数据存储密度的理想解决方案。
NOR Flash的传输效率很高，但写入和擦除速度较低；

OneNAND结合了NAND存储密度高、写入速度快和NOR读取速度快的优点，整体性能完全超越常规的NAND和NOR。
__OneNAND采用NAND逻辑结构的存储内核和NOR的控制接口，并直接在系统内整合一定容量SRAM静态随即存储器作为高速缓冲区。__

当OneNAND执行程序时，代码必须从OneNAND存储核心载入到SRAM，然后在SRAM上执行。由于SRAM的速度优势，数据载入动作几乎可以在瞬间完成，用户感觉不到迟滞现象，加上SRAM被直接封装在OneNAND芯片内部，外界看起来就好像是OneNAND也具备程序的本地执行功能。

 Flash | 读 | 写 | 擦除 | 坏块 | XIP(eXecute In Place) | 容量 | 寿命 | 成本 
:-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: 
Nor Flash | 100MBps | 0.5MBps | 0.3MBps | - | YES | 常见<= 32M | 10几万 | 较高
Nand Flash | 15MBps | 7MBps | 64MBps |有 | - | 大容量 | 100万 | 低
OneNand Flash | 100MBps | 10MBps | 64MBps | 有 | YES | 较大容量 |  100万 | 较低

### 1.1. 总线接口

#### 1.1.1. Nand Flash
Nand Flash 常见有：
- 总线接口Flash    
- SPI Flash  
总线flash需要你的MCU上有外部总线接口，SPI flash就是通过SPI口对flash进行读写。
速度上，总线flash比SPI的快，但是SPI 芯片封装布局具有优势。

如果Nand Flash 使用总线接口，一般pin 如下：

Pins | Function  
:-: | :-  
I/O 0-7(15) | Data input or output(16 bit buswidth chips are supported in kernel)  
/CE | Chip Enable  
/CLE | Command Latch Enable  
ALE | Address Latch Enable  
/RE | Read Enable  
/WE | Write Enable  
/WP | Write Protect  
/SE | Spare area Enable ( link to GND)  
R/B | Ready / Busy Output  

At the moment there are only a few filesystems which support NAND:
- JFFS2 and YAFFS for bare NAND Flash and SmartMediaCards  
- NTFL for DiskOnChip devices  
- TRUEFFS from M-Systems for DiskOnChip devices  
- SmartMedia DOS-FAT as defined by the SSFDC Forum  
- UBIFS for bare NAND flash


#### 1.1.2. Nor Flash
在通信方式上Nor Flash 分为两种类型：CFI Flash和 SPI Flash。
_CFI Flash_
英文全称是common flash interface,也就是公共闪存接口，是由存储芯片工业界定义的一种获取闪存芯片物理参数和结构参数的操作规程和标准。CFI有许多关于闪存芯片的规定，有利于嵌入式对FLASH的编程。现在的很多NOR FLASH 都支持CFI，但并不是所有的都支持。  

CFI接口，相对于串口的SPI来说，也被称为parallel接口，并行接口；另外，CFI接口是JEDEC定义的，所以，有的又成CFI接口为JEDEC接口。所以，可以简单理解为：对于Nor Flash来说，CFI接口＝JEDEC接口＝Parallel接口 = 并行接口

_SPI Flash_
serial peripheral interface串行外围设备接口,是一种常见的时钟同步串行通信接口。

_两者不同处_
CFI接口的的Nor Flash的针脚较多，芯片较大。之所有会有SPI接口, 可以减少针脚数目，减少芯片封装大小，采用了SPI后的Nor Flash，针脚只有8个。SPI容量都不是很大，读写速度慢，但是价格便宜，操作简单。而parallel接口速度快，容量上市场上已经有1Gmbit的容量，价格昂贵。

## 2. 延伸扩展  
![](https://gss2.bdstatic.com/-fo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike92%2C5%2C5%2C92%2C30/sign=47c8a31259ee3d6d36cb8f99227f0647/e1fe9925bc315c60225ddb5a8db1cb13485477be.jpg)

### 2.1. SSD [SATA](https://baike.baidu.com/item/SATA) 接口 （串行口）  
![](https://gss3.bdstatic.com/-Po3dSag_xI4khGkpoWK1HF6hhy/baike/w%3D268/sign=226f2bbbb13533faf5b6942890d2fdca/d53f8794a4c27d1ee9ac99711bd5ad6edcc438f8.jpg)

### 2.2. SSD [NVME](https://baike.baidu.com/item/NVMe/20293531) 接口 （PCIe 口）
![](https://gss0.bdstatic.com/94o3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike150%2C5%2C5%2C150%2C50/sign=61b517595866d0166a14967af642bf62/cefc1e178a82b90112fe57a7798da9773812efd7.jpg)

[SSD技术扫盲之：什么是NVMe？ NVMe SSD有什么特点？](http://www.chinastor.com/baike/ssd/04103A942017.html)

## 3. Kernel MTD Source code
![](https://raw.githubusercontent.com/JShell07/images/master/kernel_mtd/block%20device%20vs%20mtd%20device.png)

Kernel 中MTD 的源码如下图所示：
![](https://raw.githubusercontent.com/JShell07/images/master/kernel_mtd/mtd_source_code_tree.png)

MTD 组成的源代码框架如下：
![](https://raw.githubusercontent.com/JShell07/images/master/kernel_mtd/mtd_source_code_structure.png)

项目 | 说明 
:-: | :-
FTL | Flash Translation Layer, 需要PCMCIA 硬件授权专利
NFTL | Nand flash translation layer, 需要DiskOnChip 硬件授权专利
INFTL | Inverse Nand flash translation layer, 需要DiskOnChip 硬件授权专利
spi-nor | spi nor flash source code
chips, maps | CFI Nor Flash
nand | nand flash
ubi | unsorted block images, 基于raw flash 的卷管理系统

在mtd 目录下还有一些有意思的code, 他们分别是：
- mtdconcat.c 将多个MTD 设备组成一个MTD， 功能类似于rapid 磁盘阵列  
- cmdlinepart.c 提供解析启动参数中的MTD 信息  
- mtdswap.c  交换分区，用于wear leveling 记录erase counter， 但UBI 已经具备此功能
- mtdsuper.c  用于向fs/jffs2, fs/romfs 提供挂载接口

## 4. 源代码框架
Kernel MTD 在Kernel 中的结构如下：
![](https://raw.githubusercontent.com/JShell07/images/master/kernel_mtd/mtd%20softeware%20structure.png)

在MTD Sub-system 中的结构如下：
![](https://raw.githubusercontent.com/JShell07/images/master/kernel_mtd/mtd_structure.png)

mtdchar.c 向flash tools 或者用户层提供IOCTL 操作。
mtdblock.c 向kernel 提供block read/write sector访问。

他们两者的动态加载是通过mtdcore.c 中的add_mtd_device()函数。 例如在mtdblock.c 中：
```c
static struct mtd_notifier blktrans_notifier = {
	.add = blktrans_notify_add,
	.remove = blktrans_notify_remove,
};

int register_mtd_blktrans(struct mtd_blktrans_ops *tr)
{
    ...
    register_mtd_user(&blktrans_notifier);
}
```
在mtdcore.c 中
```c
int add_mtd_device(struct mtd_info *mtd)
{

    /* register char dev node */
	mtd->dev.type = &mtd_devtype;
	mtd->dev.class = &mtd_class;
	mtd->dev.devt = MTD_DEVT(i);
	dev_set_name(&mtd->dev, "mtd%d", i);
	dev_set_drvdata(&mtd->dev, mtd);
	if (device_register(&mtd->dev) != 0)
		goto fail_added;

    /* call mtd_notifiers, so it will call mtdblock.c blktrans_notify_add() */
	list_for_each_entry(not, &mtd_notifiers, list)
		not->add(mtd);
}
```
大致流程如下图所示：
![](https://raw.githubusercontent.com/JShell07/images/master/kernel_mtd/register%20partition.png)

注册相关数据结构：
```c
struct mtd_part {
	struct mtd_info mtd;
	struct mtd_info *master;
	uint64_t offset;
	struct list_head list;
};

struct mtd_partition {
	const char *name;		/* identifier string */
	uint64_t size;			/* partition size */
	uint64_t offset;		/* offset within the master MTD space */
	uint32_t mask_flags;		/* master MTD flags to mask out for this partition */
	struct nand_ecclayout *ecclayout;	/* out of band layout for this partition (NAND only) */
};
```
## 参看资源：  
[OneNAND](https://blog.csdn.net/programxiao/article/details/6214607)

[三星OneNAND技术](https://www.chinaflashmarket.com/Instructor/102570)

[Nand Flash，Nor Flash，CFI Flash，SPI Flash 之间的关系](https://www.cnblogs.com/zhangj95/p/5649518.html) 

[CFI与SPI flash区别](https://blog.csdn.net/pine222/article/details/47090041)

[MTD 官网](http://www.linux-mtd.infradead.org/doc/general.html)