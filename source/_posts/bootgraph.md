---
title: bootgraph
date: 2020-04-09 17:35:35
tags: optimization
categories: tools
---

bootgraph 主要用于量度kernel init function 的占用时间。

<!--more-->
## 1. Prepare
### 1.1. 配置Kernel
Enabel kernel 如下配置： CONFIG_PRINTK_TIME & CONFIG_KALLSYMS
![CONFIG_PRINTK_TIME](https://raw.githubusercontent.com/JShell07/images/master/tools/boot_time/CONFIG_PRINTK_TIME.png)

![CONFIG_KALLSYMS](https://raw.githubusercontent.com/JShell07/images/master/tools/boot_time/CONFIG_KALLSYMS.png)

### 1.2. 启动参数设定
```bash
bootargs = "initcall_debug"
```

## 2. Usage
### 2.1. platform
平台正常启动后， 使用如下命令收集LOG：

```bash
dmesg > boot.log
```

### 2.2. host
在Linux src 下使用脚本生成图片：
```bash
linux/scripts/bootgraph.pl boot.log > boot.svg
```

## 3. Example
[boot_log](https://github.com/JShell07/jshell07.github.io/blob/master/images/tools/boot_time/boot.log)
[bootgraph_boot_svg](https://github.com/JShell07/jshell07.github.io/blob/master/images/tools/boot_time/boot.svg)

![bootgraph_example](https://raw.githubusercontent.com/JShell07/images/master/tools/boot_time/bootgraph_example.png)

## Reference
[A Pragmatic Guide to Boot-Time Optimization - Chris Simmonds, Consultant](https://www.bilibili.com/video/BV1y4411X7e2)