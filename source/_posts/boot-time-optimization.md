---
title: boot_time_optimization
date: 2020-04-09 17:26:48
tags: boot_time
categories: tools
---

从uboot 的引导到kenrel，以及之后的init 进程的运行，中间耗费的时间，当然越短越是理想。

<!--more-->

## 1. Method
常见的方法有：
- grabserial: add timestamps to all serial console output
- bootgraph: measuring time of kernel functions
- bootchart: measuring time of user-space processes

## 2. Kernel optimization
kernel 启动优化的建议, 主要是：越少的模块，打印更少。
- `quiet` boot argument
- slim down useless driers file system
- slim down device tree by removing hardware interfaces that not used

[bootgraph](https://jshell07.github.io/blog/2020/04/09/bootgraph/)

## 3. user-space processes optimization
用户空间优化建议：
- apps startup or running order(应用执行顺序，进程之间可能有等待资源情况)
- build optimization e.g. compile flags
- library optimizations to reduce load time

[bootchart](https://jshell07.github.io/blog/2020/04/09/bootchart/)

## Reference
[A Pragmatic Guide to Boot-Time Optimization - Chris Simmonds, Consultant](https://www.bilibili.com/video/BV1y4411X7e2)

[LPC_2019_kernel_fastboot_on_the_way.pdf](https://linuxplumbersconf.org/event/4/contributions/281/attachments/216/617/LPC_2019_kernel_fastboot_on_the_way.pdf)

[Linux kernel fastboot on the way](https://linuxplumbersconf.org/event/4/contributions/281/)