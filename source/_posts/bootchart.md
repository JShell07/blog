---
title: bootchart
date: 2020-04-09 17:35:24
tags: optimization
categories: tools
---

bootchart 主要用于量度user-space processess 启动顺序及时间， 但是它的时间单位粒度其实有点大。

<!--more-->

## 1. Prepare
### 1.1. 平台支持bootchartd 命令
配置busybox，使其支持bootchartd 命令，用于抓取log。

![busybox_config_bootchartd](https://raw.githubusercontent.com/JShell07/images/master/tools/boot_time/busybox_config_bootchartd.png)

### 1.2. 启动参数设定

```bash
bootargs = "init=/sbin/bootchartd"
```

### 1.3. Host tools
用于解析抓取到日志文件： [bootchart2-0.14.8.tar.bz2](https://github.com/xrmx/bootchart/releases/download/0.14.8/bootchart2-0.14.8.tar.bz2)

在PC 上编译并安装：
```bash
make install
```

## 2. Usage
在platform 正常启动后，我们可以在<font color=red>/var/log/bootlog.tgz</font> 找到bootchartd抓取到的LOG。

![bootchartd_log](https://raw.githubusercontent.com/JShell07/images/master/tools/boot_time/bootchartd_log.png)

将抓取到LOG 用host 上tools 运行， 并在当前路径下生成图片：

```bash
/bootchart2-0.14.8/pybootchartgui.py bootlog.tgz
```

## 3. Example
[bootlog.tgz](https://github.com/JShell07/jshell07.github.io/blob/master/images/tools/boot_time/bootlog.tgz)

![boot_chart_example](https://raw.githubusercontent.com/JShell07/images/master/tools/boot_time/boot_chart_example.png)

## Reference
[bootchart](https://elinux.org/Bootchart)
[bootchart2-tools](https://github.com/xrmx/bootchart/releases)