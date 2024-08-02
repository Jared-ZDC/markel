---
title: 交叉编译Openblas库
date: 2024-06-03 08:06:28
tags: 笔记
categories: 笔记
---
# 下载源码
```bash
git clone https://github.com/OpenMathLib/OpenBLAS.git
```
# 编译
```bash
gmake TARGET=ARMV8 BINARY=64 HOSTCC=gcc CC=aarch64-linux-gnu-gcc FC=aarch64-linux-gnu-gfortran

#可以查看Targetlist.txt看支持哪些目标平台
```
## 注意
![image](https://github.com/Jared-ZDC/markel/assets/17999499/bca51fc3-2640-463d-8cf8-2d5387540244)
出现这种错误，是因为可能存在windows跟unix的脚本转换错误，这里使用dos2unix将系统目录整体处理一边
```bash
dos2unix ./*
dos2unix exports/gensymbol
```
# 编译完成
```bash
 OpenBLAS build complete. (BLAS CBLAS LAPACK LAPACKE)

  OS               ... Linux
  Architecture     ... arm64
  BINARY           ... 64bit
  C compiler       ... GCC  (cmd & version : aarch64-linux-gnu-gcc (Ubuntu 10.5.0-1ubuntu1~22.04) 10.5.0)
  Fortran compiler ... GFORTRAN  (cmd & version : GNU Fortran (Ubuntu 10.5.0-1ubuntu1~22.04) 10.5.0)
  Library Name     ... libopenblas_armv8p-r0.3.27.dev.a (Multi-threading; Max num-threads is 20)

To install the library, you can run "make PREFIX=/path/to/your/installation install".
```

# 编译benchmark
```bash
cd benchmark
gmake TARGET=ARMV8 BINARY=64 HOSTCC=gcc CC=aarch64-linux-gnu-gcc FC=aarch64-linux-gnu-gfortran
```


# 测试
```bash

#设置循环次数
export OPENBLAS_LOOPS=10000000
export OPENBLAS_PARAM_M=200
export OPENBLAS_PARAM_N=200
export OPENBLAS_PARAM_K=200

#设置线程数量，对于某些测试，不一定生效
export OPENBLAS_NUM_THREADS=4

#测试
./sgemm.goto 

```
