---
title: openwrt-ssh 跳过密码验证
date: 2017-09-04 23:29:54
tags: 
	- ssh
	- dropbear
categories: openwrt
---

openwrt 一般采用dropbear 作为ssh 客户端/服务端。 但一般都是使用password  的形式登录ssh. 我们使用public/private key 的形式来跳过需要password的验证。

# 1. 客户端
使用如下命令，生成public keys
``` bash
dropbearkey  -y -f /etc/dropbear/dropbear_rsa_host_key
```
![](https://raw.githubusercontent.com/JShell07/images/master/openwrt-ssh/public_key.png)
将图中pulibc key 复制到服务端。
<!-- more -->
# 2. 服务端
dropbear 与openssh 有点区别在于，**authorized_keys 文件并不在~/.ssh/authorized_keys 文件中, 而是在/etc/dropbear/authorized_keys**

之后重启服务端dropbear service
``` bash
/etc/init.d/dropbear restart
```
dropbear 的配置如下：
![](https://raw.githubusercontent.com/JShell07/images/master/openwrt-ssh/dropbear_config.png)

# 3. 使用
在客服端使用如下命令登陆服务端
``` bash
ssh -i /etc/dropbear/dropbear_rsa_host_key root@172.28.52.151 -p 6350
```
需要使用 "-i"选项指明identify 文件， "-p 6350" 是指明端口号
![](https://raw.githubusercontent.com/JShell07/images/master/openwrt-ssh/ssh_help.png)
![](https://raw.githubusercontent.com/JShell07/images/master/openwrt-ssh/login_ssh_without_password.png)
当然我们可以不指明**-i** option 指定文件，直接登录。

作如下步骤：
``` bash
cp /etc/dropbear/dropbear_rsa_host_key ~/.ssh/id_dropbear
```

使用如下命令直接登陆：
``` bash
ssh 172.28.52.151 -l root -p 6350 
```
![](https://raw.githubusercontent.com/JShell07/images/master/openwrt-ssh/simple_ssh_login.png)

