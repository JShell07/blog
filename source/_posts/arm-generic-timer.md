---
title: arm_generic_timer
date: 2019-11-27 13:59:41
tags:
    - arm
categories:
    - arm
---

![](https://gss1.bdstatic.com/9vo3dSag_xI4khGkpoWK1HF6hhy/baike/c0%3Dbaike80%2C5%2C5%2C80%2C26/sign=f3e81b7a10d5ad6ebef46cb8e0a252be/9922720e0cf3d7ca5a381fc8f91fbe096a63a945.jpg)

通用定时器是基于累加计数硬件，可以用作schedule events或trigger interrupts. It provides:
- Generation of timer events as interrupt outputs.  
- Generation of event streams.  
- Support for Virtualization Extensions  

<!--more-->
## 1. Base  
![generic timer example](https://raw.githubusercontent.com/JShell07/images/master/arm-generic-timer/generic_timer.png)

The Generic Timer 提供如下几种counter与Timer：
- __system counter__, measures the passing of time in real-time. 
- __virtual counter__(cpu support for virtualization), measure the passing of time on a particular virtual machine
- __timer__, that can assert a timer output signal after a period of time has passed.

## 2. system counter
system counter规格如下

|       子项       | 需求                                                       |
| :--------------: | :--------------------------------------------------------- |
|      Width       | 至少56bits 宽度，返回64-bit 是以0 进行扩展得到             |
|    Frequency     | 典型范围 1-50MHz. 在power-saving 时可以更低，例如500-20KHz |
|    Roll-over     | 不少于40年                                                 |
| Accuracy（精度） | 24Hours 误差不超过1s                                       |
|     Start-up     | 从0 开始累计                                               |

可以通过CNTFRQ 设置/读取 frequency. 

## 3. physical counter
CNTPCT（64-bit） 寄存器存储了physical counter

访问权限：
- secure PL1 mode, Non-secure Hyp mode
- Non-secure PL1 mode only when CNTHCTL.PL1PCTEN=1

## 4. virtual counter
在分两种情况：
- 没有virtualization Extension时，virtual time 与physical time相同
- 具备virtualization Extension，virtual counter里面的值等于physical counter减去 64-bit virtual offset.(<font color=green>virtual_counter = physical_counter - virtual_offset</font>)

CNTVCT 寄存器包含了virtual counter。 在Secure PL1 mode, Non-secure PL1, PL2 mode 下能访问。

CNTVOFF 寄存器包含了virtual offset, 只有Hyp mode或在SCR.NS=1时，Monitor mode下访问。

## 5. Event streams
Generic Timer 可以使用system counter 产生一个或多个event streams.

Event stream 用处：
- 产生超时on a Wait For Event polling loop
- 确保expected event在一定时间内是否generated
  
## 6. Timer
__Security Extensions not implemented__
The implementation provides a physical timer and a virtual timer.

__Security Extensions implemented, Virtualization Extensions not implemented__
- A Non-secure physical timer.  
- A Secure physical timer.  
- A virtual timer.

__Virtualization Extensions implemented__
<font color=red>
- A Non-secure PL1 physical timer. 
- A Secure PL1 physical timer.  
- A Non-secure PL2 physical timer.  
- A virtual timer.

</font>

Each timer is implemented as three registers:
- A 64-bit __CompareValue register__, that provides a 64-bit unsigned upcounter.  
- A 32-bit __TimerValue register__, that provides a 32-bit signed down counter.  
- A 32-bit __Control register__.

![timer registers summary](https://raw.githubusercontent.com/JShell07/images/master/arm-generic-timer/timer_registers_summary.png)

Accessing timer registers

|       timer        | remarks                                                                                                              |
| :----------------: | :------------------------------------------------------------------------------------------------------------------- |
| PL1 physical timer | - secure mode<br>- Non-secure Hyp mode<br>- CNTHCTL.PL1PCEN=1, Non-secure PL1 mode<br> - CNTKCTL.PL0PTEN=1, PL0 mode |
|   virtual timer    | secure, Non-secure PL1 mode, Hyp mode                                                                                |
| PL2 physical timer | Non-secure Hyp mode, 或SCR.NS=1时 Secure Monitor                                                                     |

在中断号上分配如下：

| IRQ NUM | Remarks                               |
| :-----: | :------------------------------------ |
|   29    | physical timer in secure PL1 mode     |
|   30    | physical timer in Non-secure PL1 mode |
|   27    | virtual timer in Non-secure PL1 mode  |
|   26    | physical timer in hyp mode            |

### 6.1. CompareValue
CompareValue 的操作可以看作是unsigned 64-bit up counter.

<font color=red>Eventtriggered = (Counter[63:0] - Offset[63:0]) - CompareValue[63:0] >= 0 </font>

|     术语     | 说明                                                                 |
| :----------: | :------------------------------------------------------------------- |
|   Counter    | physical counter, 读取CNTPCT，CNTVCT 寄存器                          |
|    Offset    | 如果没有Virtualization Extension, offset=0, 反之，offset = <CNTVOFF> |
| CompareValue | 比较值寄存器，CNTP_CVAL, CNTHP_CVAL, or CNTV_CVAL.                   |

### 6.2. TimerValue
TimerValue 的操作可以看作是signed 32-bit down counter.

<font color=red>Eventtriggered = (TimerValue <= 0) </font>

TimerValue 值来源于CNTP_TVAL, CNTHP_TVAL, or CNTV_TVAL.

## 7. Generic Timer registers
![generic timer registers](https://raw.githubusercontent.com/JShell07/images/master/arm-generic-timer/generic_timer_registers.png)


## Reference
cortex_a7_mpcore_r0p5_trm(arm_trm).pdf
arm_architecture_reference_manual(arm_arm).pdf

[Linux时间子系统之（十七）：ARM generic timer 驱动分析](http://www.wowotech.net/timer_subsystem/armgeneraltimer.html)