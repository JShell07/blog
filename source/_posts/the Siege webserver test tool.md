---
title: the Siege webserver performance test tool
date: 2017-06-22 10:56:27
tags: 
	- Siege
	- webserver
categories: tools
---

# 1. Backgroud
>Siege is an open source regression test and benchmark utility. It can stress test a single URL with a user defined number of simulated users, or it can read many URLs into memory and stress them simultaneously. The program reports the total number of hits recorded, bytes transferred, response time, concurrency, and return status. Siege supports HTTP/1.0 and 1.1 protocols, the GET and POST directives, cookies, transaction logging, and basic authentication. Its features are configurable on a per user basis.

Detail see  <https://github.com/JoeDog/siege>

# 2. Compile
Decompressing Siege source code archive, and run  
``` bash
./utils/bootstrap
./configure
make && make install
```
**Note:**  If we have ssl or zlib support, please point out when we make the configuration.   
```bash
--with-ssl=$(SSL_include_dir) \
--with-zlib=$(ZLIB_include_dir)
```
<!-- more -->
# 3. Run
Use "siege -C ' command to see current configuration.   
![](https://raw.githubusercontent.com/JShell07/images/master/siege/siege_config.png)

Modify /etc/url.txt file to test accessing url.  
![](https://raw.githubusercontent.com/JShell07/images/master/siege/siege_test_url.png)

Then we use siege command to test, and press "Ctrl + C" to stop.  
![](https://raw.githubusercontent.com/JShell07/images/master/siege/siege_run.png)

# 4. Result
After we stop Siege with "Ctrl + C", we will get result:  
![](https://raw.githubusercontent.com/JShell07/images/master/siege/siege_result.png)
