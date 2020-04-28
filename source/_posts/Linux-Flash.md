---
title: Linux_Flash
date: 2018-08-22 22:20:13
tags:
    - flash
    - kernel
categories:
    - drivers
---

## 1. Flash
闪存（Flash Memory）是非易失性长寿命存储器，是电子可擦除只读存储器（EEPROM）的变种，而闪存的大部分芯片需要块擦除。

![MOS管插图](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/MOS%E7%AE%A1.png)

<!--more-->

注：
>N型半导体即自由电子浓度远大于空穴浓度的杂质半导体。(对于锗、硅类半导体材料，掺杂Ⅴ族元素(磷、砷等)
P型半导体即空穴浓度远大于自由电子浓度的杂质半导体。在纯净的硅晶体中掺入三价元素（如硼）

常见的Flash有:  
- NOR Flash
- NAND Flash

|                | NOR Flash                     | NAND Flash                               |
| :------------: | :---------------------------- | :--------------------------------------- |
|     读速度     | 快                            | 较快                                     |
| 单字节编程时间 | 快                            | 慢                                       |
| 多字节编程时间 | 慢                            | 快                                       |
|      寿命      | 十万次                        | 百万次                                   |
|    I/O 端口    | SRAM 接口，有足够地址引脚寻址 | 8个复用PIN 脚传送控制、地址、数据        |
|      功耗      | 高                            | 低，需要额外RAM                          |
|    应用市场    | 1~16MB                        | 大容量                                   |
|      特点      | 程序可在芯片内运行            | 成本低，高存储密度，较快的写入和擦除速度 |

两者更详细的比较可以通过Flash 厂商的datasheet 查看。

`Large Page Nand Flash`
![Large Page Nand](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/Large_Page_Nand.png)

注：Spare Area 常常可以存储BBT(Bad Block Table), ECC, 逻辑映射表等信息。

Nand Flash 硬件特性：
- R/W 最小单位，Page
- 擦除最小单位，Block
- 擦除的含义是将整块都擦除为0xFF
- 写操作前，必须先擦除，然后再写

## 2. Linux MTD

![MTD Structure](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/Linux_MTD_Structure.png)

常见MTD 分区为：

![](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/MTD_Common_Partitions.png)

## 3. Operation
Flash 常见的操作有Read, Erase, Write, WP(Write Protect)

`MTD Read Flow`
![MTD_Read_Flow](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/MTD_Read_Flow.png)

使用多并口时， SPI 会用到WP#， HOLD# 等管脚。
Standard SPI: SCLK, CS#, SI, SO
Dual SPI: SCLK, CS#, IO0(SI), IO1(SO)
Quad SPI: SCLK, CS#, IO0(SI), IO1(SO), IO2(WP#), IO3(HOLD#)

在SPI 接口上可能会有如下形式的Read：
- Page Read
- Page Read to cache
- page from cache
- fast read from cache
- read from cache X2
- read from cache X4
- read from cache dual IO
- read from cache quad IO

SPI Nand Read 时序一般为：
1. 13H(page read to cache)
2. 0FH(get features coomand to read the status)
3. 03H/OBH (read from cache), 3BH/6BH(read from cache x2/x4), BBH/EBH(Read from cache dual/quad IO)

spi_nand_read_from_cache
![spi_nand_read_from_cache](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/spi_nand_read_from_cache.png)

spi_nand_read_to_cache
![spi_nand_read_to_cache](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/spi_nand_read_to_cache.png)

spi_nand_read_from_cache_x4
![spi_nand_read_from_cache_x4](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/spi_nand_read_from_cache_x4.png)

spi_nand_read_from_cache_quad_io
![spi_nand_read_from_cache_quad_io](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/spi_nand_read_from_cache_quad_io.png)
<br>

`MTD Write Flow`
![MTD_Write_Flow](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/MTD_Write_Flow.png)

在SPI Nand Flash 上的时序图流程一般为：
1. 02H (PROGRAM LOAD)/32H (PROGRAM LOAD x4)
2. 06H (WRITE ENABLE)
3. 10H (PROGRAM EXECUTE)
4. 0FH (GET FEATURE command to read the status)

program load 是设定addr， 并将write data 加载到cache register（一般超过PAGE SIZE 如2K+oob 的数据会被ignore），打开WE#，在program execute 命令才执行。

`WP(Write Protect)`
一般Flash 还具备block 相关指令。

|    指令     | 说明                                 |
| :---------: | :----------------------------------- |
|    block    | 对某一区域Flash 进行写保护，不能写入 |
|  un-block   | 解除block 限制                       |
| block down  |
| lock status | 查看状态                             |

可能每个Flash 芯片厂商有所不同, 例如Giga SPI Nand Flash block, 通过SET Features，来控制block
![giga_spi_nand_block](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/giga_spi_nand_block.png)

`Erase`
Nand Flash 的erase 只能是Block(常见为64/128K) Erase。而Nor Flash 可以是Sector（一般4K） 或Block（32/64K） Erase。

## 4. Bad Block
Nand Flash 工艺不能保证Memory Cell在其生命周期中保持性能的可靠性，使用寿命一般100万次。

坏块一般分为：

- 固有坏块：生产中就有（芯片原厂将每个坏块第一个Page的spare area某个字节标记为非0xFF）
- 使用坏块：存储寿命（program/erase 错误，将block标记为坏块）

当然，也有伪坏块。芯片在才做过程中可能由于电压不稳定等因素导致Program、Erase错误，标记的坏块也可能是好的。解决办法是，retry 一次，然后查看状态。

`坏块的特性`

Page Program与Block Erase操作失败，会反映到Status Register。

`避免坏块策略`

**Wear-Leveling(损益均衡)技术**
![](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/Wear_Leveling.png)

**Buffer Cache**
类似于UBIFS 都会带有buffer cache，以及压缩，以此来尽量降低对I/O的操作。

`应对坏块`

为了保证数据正确性的R/W，我们使用BBT（Bad Block Table）。
![](https://raw.githubusercontent.com/JShell07/images/master/Linux_Flash/BBT.png)

## Reference
[gd5f1gq4xfxxh_v1.2_20190510](https://www.gigadevice.com/datasheet/gd5f1gq4xfxxh/)
[gd25b127d_v1.4_20180725](https://www.gigadevice.com/datasheet/gd25b127d/)


