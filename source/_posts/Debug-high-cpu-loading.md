---
title: Debug_high_cpu_loading
date: 2018-05-26 15:28:50
tags: Debug
categories: debug
---

![](http://rdc.hundsun.com/portal/data/upload/201705/f_3c65934a804b2cd6ec6dab02ccb33457.png)

<!--more-->

追查问题，主要是在：

- 浮现问题
- 缩小范围
- 猜测，修正及验证

我们这里使用如下测试代码演示：

``` c
#include <stdio.h>
#include <sys/epoll.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <fcntl.h>
#include <unistd.h>
#define MAX_MONITOR_FILES 4

void main()
{
  int e_fd, fd;
  struct epoll_event event, rdy_event[MAX_MONITOR_FILES];

  if ((fd = socket(AF_UNIX, SOCK_STREAM, 0)) == -1) {
      perror("socket");
      goto out1;
  }

  if ((e_fd = epoll_create(MAX_MONITOR_FILES)) == -1) {
    perror("epoll_create");
    goto out2;
  }

  event.events = EPOLLIN;
  if((epoll_ctl(e_fd, EPOLL_CTL_ADD, fd, &event)) == -1) {
    perror("epoll_ctl");
    goto out3;
  }

  int n, j;
  while(1) {
    n = epoll_wait(e_fd, rdy_event, MAX_MONITOR_FILES, 0);
    if(n == -1) {
      perror("epoll_wait");
    }else if(n == 0) {
      //timeout, skip
    }

    for(j=0; j<n; j++) {
      //read it out
    }
  }

out3:
        close(e_fd);
out2:
        close(fd);
out1:
        return;
}
```

## 进程
我们通常可以使用top 命令查看到某一个进程占用了较高的CPU Loading。
![](https://raw.githubusercontent.com/JShell07/images/master/cpu_high_loading/top_process.png)

## 线程
可以通过`top -H -p <pid>` 命令具体查看`<pid>` 进程的子线程占用CPU Loading的情况。
![](https://raw.githubusercontent.com/JShell07/images/master/cpu_high_loading/top_thread.png)

这里是单独一个进程，是以这里只是单单出现一个。

## 函数
在定位到某一个线程之后，我们需要继续定位到某一个函数或者命令导致了CPU 占用过高。
`strace -p <pid>` 命令能帮助查找到具体的函数。
`pstack <pid>`, `trace -p <tid>`等命令也能给予我们一些启示。
![](https://raw.githubusercontent.com/JShell07/images/master/cpu_high_loading/strace_process.png)

## 参考资源
[Linux下某个进程CPU占用率高分析方法](http://www.linuxeye.com/Linux/1843.html)
[一次服务器CPU占用率高的定位分析](https://www.jianshu.com/p/e680d4e6d4ae)
[超全整理！Linux性能分析工具汇总合集](https://blog.csdn.net/tiantangyouzui/article/details/72231590)