---
title: iperf2 交叉编译 ARM64 static版本
date: 2025-04-25 11:01:12
tags: 程序人生
categories: 程序人生
toc: true
---
编译静态版本的iperf2，到嵌入式环境进行测试；

iperf2 可以利用多核多线程并行测试，iperf3的多核是运行在一个物理核上面，只是软件意义上的多核；


```patch
(base) root@mengxia01:~/code/iperf-2.0.4-RELEASE# diff config.h ../iperf-2.0.4-arm/config.h
258c258
< //#define bool int  
---
> #define bool int
```
```bash
#编译命令
./configure  --host=arm CC=aarch64-linux-gnu-gcc   CXX=aarch64-linux-gnu-g++   CFLAGS="-static -fPIE -fPIC"   LDFLAGS="-static -fPIE -fPIC"
```