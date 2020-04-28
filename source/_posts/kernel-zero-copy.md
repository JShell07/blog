---
title: kernel-zero-copy
date: 2019-08-21 15:35:06
tags: 
    - memory
    - kernel
categories: memory
---

在看openssl 1.1.1c 版本源码时，看到有一个zero copy 的字样。这里zero copy(零拷贝)主要指Kernel space 与user space 之间的拷贝过程。


<!--more-->

### 1.Normal R/W

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);

#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
```

在read/write 时，需要将userspace 的数据copy 到kernel space。kerne 中用到的函数就有：
>copy_from_user()
copy_to_user()

![normal read/write image](https://raw.githubusercontent.com/JShell07/images/master/kernel_zero_copy/normal%20read%20write.png)

mmap() 能减少read 的copy 动作，直接映射kernel空间到用户空间，但是在write时， 还是需要将write_data_buffer 拷贝到kernel space.

![mmap read/write image](https://raw.githubusercontent.com/JShell07/images/master/kernel_zero_copy/mmap%20read%20write.png)

在拷贝文件时，我们可以这样减少copy 的次数。
```c
size_t filesize = stat_buf.st_size;
source = mmap(0, filesize, PROT_READ, MAP_SHARED, f_in, 0);
target = mmap(0, filesize, PROT_WRITE, MAP_SHARED, f_out, 0);
memcpy(target, source, filesize);
```

### 2.zero copy
常见的zero copy 涉及到的函数有：
- sendfile
- vmsplice, splice
- tee

除vmsplice（） 是映射函数外，其他借助管道实现，而管道有众所周知的空间限制问题，超过了限制就会hang住，所以每次写入管道的数据量好严格控制，保守的建议值是一个内存页大小，即PAGE_SIZE, 常见为4k。

splice用于在两个文件间移动数据，而无需内核态和用户态的内存拷贝，但需要借助管道（pipe）实现。<font color=red>大概原理就是通过pipe buffer实现一组内核内存页（pages of kernel memory）的引用计数指针（reference-counted pointers），数据拷贝过程中并不真正拷贝数据，而是创建一个新的指向内存页的指针。也就是说拷贝过程实质是指针的拷贝.</font>

![zero copy image](https://raw.githubusercontent.com/JShell07/images/master/kernel_zero_copy/splice%20read%20write.png)

```c
#include <sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);


#include <fcntl.h>
#include <sys/uio.h>
ssize_t vmsplice(int fd, const struct iovec *iov,unsigned long nr_segs, unsigned int flags);


#include <fcntl.h>
ssize_t splice(int fd_in, loff_t *off_in, int fd_out,loff_t *off_out, size_t len, unsigned int flags);


#include <fcntl.h>
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);                      
```

system api | remarks
:-: | :-
sendfile() | sendfile的in_fd必须指向支持mmap的文件，也就是真实存在的文件，而不能是socket、管道等文件
splice() | splice()函数可以在两个文件描述符之间移动数据，且 __其中一个描述符必须是管道描述符__
tee() | 仅支持在<font color=red>两个管道描述符之间</font>复制数据


splice() 通过pipe 零拷贝文件的用例。

>file_in -> pipe[1] (write end) -> pipe[0](read end) -> file_out

```c
int pipefd[2], off_in = 0, off_out = 0;
int file_in, file_out, size, flags;

file_in = open("input_file", O_RDONLY);
file_out = open("output_file", O_RDWR);

pipe(pipefd);

splice(file_in, &off_in, pipefd[1], NULL, size, flags);
splice(pipefd[0], NULL, file_out, &off_out, size, flags);

close(file_in);
close(file_out);
close(pipefd[0]);
close(pipefd[1]);
```

### 参看资料

[零复制(zero copy)技术](https://www.cnblogs.com/f-ck-need-u/p/7615914.html)

[linux网络编程：splice函数和tee( )函数高效的零拷贝](https://www.cnblogs.com/kex1n/p/7446291.html)

[Linux 中的零拷贝技术 splice](http://abcdxyzk.github.io/blog/2015/05/07/kernel-mm-splice/)

![Linux 中的零拷贝技术，第 2 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy2/)

[Linux下提高性能的系统调用sendfile，splice和tee](https://blog.csdn.net/wzjking0929/article/details/51831478)

[splice and pipes](https://www.kernel.org/doc/html/latest/filesystems/splice.html)