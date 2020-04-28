---
title: kernel_tlb
date: 2020-03-12 10:48:52
tags: tlb
categories: memory
---

TLB(Translation lookaside buffer) 本质上也是一种cache， 物理特性与D/I Cache 类似（SDRAM）。D/I Cache 中cache 的是数据或者指令，而TLB帮助cpu 取指或执行访问mem指令时将VA（Virtual Address）转换成PA（Physical Address）, 减少HW translation table walk 访问main mem 中的page tables，他在armv7 架构中的位置可以参看：

<!--more-->

![arm cortex-a7 structure](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/arm_cortex-a7_mpcore_cpu_structure.png)

TLB 一般还分为两大类：
- main tlb (常规的tlb)
- micro tlb (用于加速Data, Instructions的快速访问)

## 1. work flow
Armv7 ARM(architecture reference manual)中并没有规定TLB 具体的数据结构， 不过他主要包括：
- 物理地址（更准确的说是physical page number）。这是地址翻译的结果
- 虚拟地址（更准确的说是virtual page number）。用cache的术语来描述的话应该叫做Tag，进行匹配的时候就是对比Tag
- Memory attribute（例如：memory type，cache policies，access permissions）
- status bits（例如：Valid、dirty和reference bits
- 其他相关信息。例如ASID、VMID

当需要转换VA到PA的时候，首先在TLB中找是否有匹配的条目，如果有，那么TLB hit，这时候不需要再去访问页表来完成地址翻译。不过TLB始终是全部页表的一个子集，因此也有可能在TLB中找不到。如果没有在TLB中找到对应的item，那么称之TLB miss，那么就需要去访问memory中的page table来完成地址翻译，同时将翻译结果放入TLB。

![tlb work flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/tlb_work_flow.gif)

## 2. TLB Match
整个地址翻译过程并非简单的VA到PA的映射那么简单，系统中虚拟地址空间很多，每个地址空间都是独立的。可以考虑如下情况：
1. userspace 的进程都是独立虚拟空间，各个进程不同的虚拟地址空间中，相同的VA需翻译成不同的PA
2. 若支持virtual extension。guest os 之间相同的VA 也要通过过VA->IPA->PA 形式转换成不同的PA
3. secure, non-secure 虚拟的地址空间
4. Global 地址如OS TEXT段与Non Global 地址
   
因此，只有在满足一定的条件才能说TLB match：
1. 请求VA page number 与TLB TAG 中VA page number相等；
2. 请求VA 的memory space identifiler 与TLB entry中memory space identifiler 相同。space ID区分PL1,2 或secure, non-secure level；
3. 请求entry 为Non Global时，请求翻译的地址的ASID 与TLB 中相等。
4. 请求地址翻译的VMID 等于TLB entry 中VMID 相等。 

### 2.1. ASID(Address Space Identifier)
在进程切换的时候，将TLB中的所有内容全部flush掉（全部置为无效），这样的设计当然很清爽，但是性能会大打折扣。
一个比较好的方案是区分Global pages （内核地址空间）和Process-specific pages（参考页表描述符的nG的定义）。对于Global pages，地址翻译对所有操作系统中的进程都是一样的，因此，进程切换的时候，下一个进程仍然需要这些TLB entry，因而不需要flush掉。对于那些Process-specific pages对应的TLB entry，一旦发生切换，而TLB又不能识别的话，那么必须要flush掉上一个进程虚拟地址空间的TLB entry。如果支持了ASID，那么情况就不一样了：对于那些nG的地址映射，它会有一个ASID，对于TLB的entry而言，即便是保存多个相同虚拟地址到不同物理地址的映射也是OK的，只要他们有不同的ASID。 

### 2.2. VMID(Vitural Machine Identifier)
与ASID 存在意义类似， 在切换虚拟机的时候具有加速作用。

## 3.translation table
VMSAv7 支持两种格式translation table
`Short-descriptor format`
This is the original format defined in issue A of this Architecture Reference Manual, and is the only
format supported on implementations that do not include the Large Physical Address Extension. 
- Up to two levels of address lookup.
- 32-bit input addresses.Output addresses of up to 40 bits.
- Support for PAs of more than 32 bits by use of supersections, with 16MB granularity.
- 32-bit table entries.
  
`Long-descriptor format`
- Up to three levels of address lookup.
- Input addresses of up to 40 bits, when used for stage 2 translations.
- Output addresses of up to 40 bits.
- 4KB assignment granularity across the entire PA range.
- 64-bit table entries.
  
TTBCR.EAE 的配置决定采用何种格式转换表。

`Translation table base config`

|                    Config Register                    | Control Register | Remarks                                                                                                                 |
| :---------------------------------------------------: | :--------------: | :---------------------------------------------------------------------------------------------------------------------- |
|   HTTBR(Hypervisor Translation Tabel Base Register)   |       HTCR       | Non-secure PL2 stage 1 translation                                                                                      |
| VTTBR(Virtualization Translation Table Base Register) |       VTCR       | Non-secure PL1&0 stage 2 translation                                                                                    |
|                        TTBR0/1                        |      TTBCR       | secure/non-secure PL1&0 stage 1 translation,  TTBR0, TTBR1, and TTBCR are Banked between Secure and Non-secure versions |

### 3.1. short-descriptor translation table
支持四种section or pages：
- supersections, consist of 16MB blocks
- sections, consist of 1MB blocks
- large pages, consist of 64KB blocks
- small pages, consist of 4KB blocks

`First-level table`
Holds first-level descriptors that contain the base address and
• translation properties for a Section and Supersection
• translation properties and pointers to a second-level table for a Large page or a Small page.

`Second-level tables`
Hold second-level descriptors that contain the base address and translation properties for a Small
page or a Large page. With the Short-descriptor format, second-level tables can be referred to as
Page tables.
A second-level table requires 1KByte of memory.

#### 3.1.1. general view of address translation using short-descriptor
![general view of address translation using short-descriptor](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/general_view_using_short_descriptor.png)

#### 3.1.2. short descriptor 1st level format
![short descriptor 1st level format](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/short_descriptor_first_level.png)

#### 3.1.3. short descriptor 2nd level format
![short descriptor 2nd level format](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/short_descriptor_2nd_level.png)


|   Attribute    | Remarks                                                                     |
| :------------: | :-------------------------------------------------------------------------- |
| TEX[2:0]，C, B | mem 属性，cache, share等， These bits are not present in a Page table entry |
|       XN       | execute-never bit，CPU 是否能执行此地址指令                                 |
|       NS       | Non-secure bit                                                              |
|       AP       | Access permission， read/write 权限                                         |
|       S        | shareable bit                                                               |
|       nG       | not Global bit                                                              |

#### 3.1.4. TTBR0,1 之间的选择
当TTBCR.EAE=0 选择short-descriptor format，TTBCR.N 决定了TTBR0, TTBR1 的选择情况。
- N=0, only using TTBR0
- N>0, 若VA[31:32-N] bit 为0， 使用TTBR0, 其他使用TTBR1

TTBCR format when using short descriptor with security extension- 
![TTBCR format when using short descriptor with security extension](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/TTBCR%20format%20when%20using%20short%20descriptor%20with%20security%20extension.png)

TTBCR.N effect on address translation, short descriptor format
![TTBCR.N effect on address translation, short descriptor format](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/TTBCR.N%20effect%20on%20address%20translation%2C%20short%20descriptor%20format.png)

Example TTBCR.N effect on address translation, short descriptor format
![Example TTBCR.N effect on address translation, short descriptor format](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/example%20TTBCR.N%20effect%20on%20address%20translation%2C%20short%20descriptor%20format.png)

### 3.2. long-descriptor translation table
long-descriptor 在有virtualization extension的ARM 时，支持：
1. the Non-secure PL2 stage 1 translation
2. the Non-secure PL1&0 stage 2 translation
3. can be used for the Secure and Non-secure PL1&0 stage 1 translations

#### 3.2.1. general view of stage 1 address translation using long-descriptor
![general view of stage 1 address translation using long-descriptor](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/general_view_stage1_using_long_descriptor.png)

#### 3.2.2. general view of stage 2 address translation using long-descriptor
![general view of stage 2 address translation using long-descriptor](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/general_view_stage2_using_long_descriptor.png)

#### 3.2.3. long descriptor 1st, 2nd level format
![long descriptor 1st level format](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/long_descriptor_1st_2nd_level.png)

#### 3.2.4. long descriptor 3rd level format
![long descriptor 3rd level format](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/long_descriptor_3nd_level.png)


#### 3.2.5. TTBR0,1 之间的选择
与short descriptor 相似，但是查看TTBCR.T0SZ
TTBCR format when using long descriptor
![TTBCR format when using long descriptor](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/TTBCR%20format%20when%20using%20long%20descriptor.png)

## 4. TLB Maintenance
TLB 相关的操作可以参看如下图， 可以依照MVA(modified virtual address) 也可以依照ASID(Application space identifiler) 去维护TLB。
tlb maintenance operations
![tlb maintenance operations](https://raw.githubusercontent.com/JShell07/images/master/kernel_mm/tlb/tlb_maintenance_operations.png)

注：  
- MVA(modified virtual address)
- ASID (application space identifiler)

## Reference
[TLB flush操作](http://www.wowotech.net/memory_management/tlb-flush.html)
ARMv7 ARM - DDI0406C_C_arm_architecture_reference_manual