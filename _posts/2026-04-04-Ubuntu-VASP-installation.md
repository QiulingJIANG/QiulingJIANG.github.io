---
layout: post
title: "Ubuntu—VASP installation"
date: 04/04/2026
categories: work-notes
tags: linux, ubuntu, vasp, openmpi
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

## Prepare

1. Prepare the "toolbox" (install dependencies: gcc, g++, gfortran, make etc.):

   ```bash
	sudo apt update
	sudo apt install -y build-essential libstdc++-12-dev rsync python3-venv wget
   ```

2. Intel OneAPI installation

|**套件名称**|**对应名**|**核心作用**|
|---|---|---|
|**基础套件**|**Intel® oneAPI Base Toolkit**|提供 **MKL (数学库)**，这是 VASP 算得快的核心。|
|**HPC套件**|**Intel® oneAPI HPC Toolkit**|提供 **Fortran 编译器** 和 **MPI (并行库)**，这是编译 VASP 的工具。|

```bash
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/163da1aa-791a-424d-82db-9806da09a962/l_BaseKit_p_2025.3.1.36.sh
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/757f6362-e64e-41a4-9e8c-8977f6b9c970/l_HPCKit_p_2025.3.1.55.sh
```
> Note: 
> 	1.  网络经常不稳定，所以推荐本地下载之后再上传到单机上
> 	2. 先安装base-toolkit，再安装HPC-kit
> 	3. base-toolkit只需要安装: Intel® oneAPI Math Kernel Library (MKL)即可，其余都不需要
> 	4. hpc-toolkit安装: Intel® Fortran Compiler (这是编译 VASP 的核心) + Intel® oneAPI MPI Library (这是并行的核心) + Intel® oneAPI DPC++/C++ Compiler
> 	5. 激活环境与check编译器: 
>``` bash
> echo "source /opt/intel/oneapi/setvars.sh" >> ~/.bashrc
> source ~/.bashrc	
> ifx --version #检查Fortran编译器 
> icpx --version #检查 C++ 编译器
> mpirun --version #检查并行库
> ```

## Vasp installation

1. 修改make.include文件
	- 这里修改的文件是make.include.intel：
	  - `make.include.intel`是intel下**最纯粹、兼容性最好**的基准模板
	  - `intel_omp` (OpenMP)，**混合并行**（MPI + OpenMP），往往用于超算中心跨节点的大规模计算，单机跑反而容易因为线程调度导致性能下降。
	  - `intel_serial` (Serial)，**单核串行版**
	- make.include.intel重命名为make.include，并修改以下内容，核心逻辑是**将旧版 Intel 编译器指令替换为 oneAPI 现代指令集**（**记得打补丁的代码对应修改**，这里只是对于vasp源代码需要修改的make.include内容）:
	- 编译器指令 (Compilers)：
		```bash
		FC = mpiifort # old 
		FC = mpiifx # new
		
		CC_LIB = icc (或 `icpx`) # old 
		CC_LIB = icx # new
		
		CXX_PARS = icpc # old 
		CXX_PARS = icpx # new
		```
	> note: Intel 已正式弃用经典的 `ifort` 和 `icc`。`ifx` 是基于 LLVM 的新一代 Fortran 编译器，性能更好。特别注意 `CC_LIB` 必须用 **`icx`**（C编译器）而不是 `icpx`（C++编译器），否则在编译底层 C 代码（如 `timing_.c`）时会报语法冲突错误。

	- 数学库链接 (MKL & Scalapack):
		```bash
		FCL += -mkl=sequential  # old 
		FCL += -qmkl=cluster # new
		
		# hash原有的 LLIBS 中关于 lmkl_scalapack_lp64 等长字符串
		# hash原来的MKLROOT那一行
		```
	> note: **`sequential`（串行）：** 只调用 MKL 的单核数学函数；**`cluster`（集群/并行）：** 告诉编译器，我要跑的是 MPI 并行任务，请自动帮我链接 **ScaLAPACK**（并行线性代数库）和 **BLACS**（通信库）；只要编译的是并行版 VASP（不管是单机多核，还是超算跨节点），都应该用 **`cluster`**。
	> 之所以hash MKLROOT，因为之前运行了 `source /opt/intel/oneapi/setvars.sh`。这个脚本会在 Linux 系统环境变量里**自动设置**好一个名为 `$MKLROOT` 的变量。
2. make
- make之前，先make veryclean
- 我之前是直接make all的，这次尝试了 `make DEPS=1 -j[核心数] all`，速度确实快很多！

|参数|作用|结果|
|---|---|---|
|`all`|编译 `std`, `gam`, `ncl` 三个版本。|默认一个一个文件编，极慢。|
|`-j16`|**并行编译**。同时启动 16 个进程编译不同的源码。|编译时间从 30 分钟缩短到 3 分钟。|
|`DEPS=1`|**依赖检查**。确保底层模块（如 `base.f90`）先于高层模块编译。|防止因为并行编译太快，导致“地基还没打好就开始盖楼”而报错。|
