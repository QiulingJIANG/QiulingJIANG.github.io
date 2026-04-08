---
layout: post
title: "Ubuntu-CP2K installation"
date: 08/04/2026
categories: work-notes
tags: linux, ubuntu, cp2k
---

## Configurations

1. CPU: Intel Xeon E5-2696 v4
2. Socket: 2
3. Cores: 44
4. CPUs: 88
5. Flags: AVX2, FMA3
6. RAM: 64GB
7. SSD: 500GB
8. Ubuntu: 24.04.4

## Installation

1. 这个比较简单了，前面安装vasp的时候安装了oneAPI，已经有了 oneAPI 环境，首先确保环境变量已加载。

   ```bash
   source /opt/intel/oneapi/setvars.sh # 加载 oneAPI 环境变量（如果尚未加载）
   which ifx # 确认编译器路径
   which mpiifx # 确认编译器路径
   ```

   > note: 后续编译报错，发现缺少`gfortran`，之前系统只有`gcc`和`g++`，所以这里还是按照老规矩，先更新系统包列表，再安装`gfortran`以及一些 cp2k github 上推荐的一些基础包
   >
   > ```bash
   > sudo apt update
   > sudo apt install -y cmake gfortran libstdc++-12-dev build-essential wget python3 python3-dev pkg-config libtool autoconf automake # 安装 gfortran 以及 CP2K toolchain 依赖的一些基础开发包
   > ```

2. 获取cp2k源码

   ```bash
   # 这里准备安装2025.2的版本
   git clone --recursive -b v2025.2 https://github.com/cp2k/cp2k.git cp2k_2025
   ```

3. 编译 Toolchain（这个最简单，我之前超算不能联网，一个一个配置库非常麻烦）

   ```bash
   cd ~/Software/cp2k_2025/tools/toolchain
   source /opt/intel/oneapi/setvars.sh # 确保只加载了 oneAPI 环境，没有其他冲突的库路径

   # 再执行
   ./install_cp2k_toolchain.sh \
       --mpi-mode=intelmpi \
       --math-mode=mkl \
       --with-gcc=system \
       --with-cmake=system \
       --with-intelmpi=system \
       --with-mkl=system \
       -j 20
   ```

   > note:
   > - cp2k 的 toolchain 脚本可能还在找 `ifort`，导致和我这里单机的 Intel oneAPI 2025 的 `ifx`（彻底移除了经典的 `ifort` 编译器）一直报错，然后创建 `ifort` 符号链接指向 `ifx`，也就是：
   > - ```bash
   >   # 检查一下 toolchain 脚本里具体在找什么
   >   grep -n "ifort" ~/Software/cp2k_2025/tools/toolchain/scripts/stage0/install_intel.sh
   >   ```
   > - 我这里发现脚本只是用 `check_command` 检查 `ifort` 是否存在，然后把路径赋给 `FC` 变量。
   > - 所以重新创建一个符号链接即可：
   > - ```bash
   >   # 创建符号链接
   >   sudo ln -s /opt/intel/oneapi/compiler/2025.3/bin/ifx /opt/intel/oneapi/compiler/2025.3/bin/ifort
   >
   >   # check 一下是否生效
   >   which ifort
   >   ifort --version
   >   ```
   > - 这里还出现一个问题是 **DBCSR 编译失败**（因为 `ifort` 符号链接指向 `ifx` 后，CMake 识别编译器 ID 时产生了预处理器参数不匹配的问题，`fypp` 预处理没正确执行）。我查询到 cp2k 内置了 DBCSR，所以重新编译 toolchain 的时候，跳过了 DBCSR 的单独编译，添加了 `--with-dbcsr=no`。
   > - 之后再进行 toolchain 编译即可。

   |参数|含义|
   |---|---|
   |`--mpi-mode=intelmpi`|使用 Intel MPI 作为并行通信库（而非 OpenMPI）|
   |`--math-mode=mkl`|用 MKL 提供 BLAS、LAPACK、ScaLAPACK 数学库|
   |`--with-gcc=system`|用系统已装的 GCC 13.3，不重新编译|
   |`--with-cmake=system`|用系统已装的 CMake 3.28，不重新编译|
   |`--with-intelmpi=system`|用系统已装的 Intel MPI，不重新编译|
   |`--with-mkl=system`|用系统已装的 MKL，不重新编译|
   |`-j 20`|编译时用 20 个核并行，加快速度|

4. 修改和编译编译CP2K psmp

   ```bash
   # source 环境
   source /opt/intel/oneapi/setvars.sh
   source /home/jql24/Software/cp2k_2025/tools/toolchain/install/setup

   # 复制 arch 文件
   cp /home/jql24/Software/cp2k_2025/tools/toolchain/install/arch/* /home/jql24/Software/cp2k_2025/arch/

   # 查看可用的 arch 文件
   ls /home/jql24/Software/cp2k_2025/arch/local*

   # 开始编译
   make -j 20 ARCH=local VERSION=psmp
   ```

   > note:
   > - 需要修改 psmp 文件，修改了一下 Fortran 接口库，以及删除了一些不存在的路径。
   > - 编译过程中有报错，大多数是因为 `ifort` 与 `ifx` 的识别问题（我这里遇到的基本上是），有些文件需要手动复制一下，这个只有见招拆招了，每次不同机器上编译的错误都不太一样。

5. 确认一下可执行文件

   ```bash
   ls -la ~/Software/cp2k_2025/exe/local/
   ~/Software/cp2k_2025/exe/local/cp2k.psmp --version
   ```

6. 作业测试即可
