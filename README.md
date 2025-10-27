# 从零开始：在 macOS 上运行湿大气斜压波算例（GRIST）

本教程以 macOS 系统为示例，介绍如何在类 Unix 环境下从零开始配置并运行 GRIST 模式 的湿大气斜压波算例（图 1）。教程旨在帮助用户在本地笔记本电脑上完成模式的编译、运行与可视化的完整流程。总体步骤同样适用于 Ubuntu Linux 等系统。Windows 用户可通过安装虚拟机实现运行。通过该案例，读者将了解一个具有完整复杂度的大气数值模型的基本结构与运行机制。

本文档仅限教学用途。

![Figure 1](https://github.com/GRIST-Dev/GRIST-RunOnUrLapTop/blob/main/doc/dcmip-bw.png)
*图1. 本算例所计算的湿大气斜压波算例展示的相对涡度场（注：为高分辨率设置，教程演示为低分辨率设置）*

---

## 演示系统信息

- **操作系统**：macOS Tahoe (Darwin, arm64)
- **模式代码仓库**：[https://github.com/grist-dev/GRIST]
- **所需数据**：GRIST 网格文件（本目录inputdata下）

## 模式设置信息

- **动力求解器**：湿大气-静力动力内核
- **传输算法**：水平两步保形平流（TSPAS）+垂直自适应半隐式平流
- **水平分辨率**：g4 (480km), g5 (240km)
- **垂直分辨率**：标准31层设置（模式顶约为40km）

---

## 一、安装基础依赖

### 1. 安装 Homebrew

Homebrew 是 macOS 下的包管理工具，类似于 Ubuntu 的 `apt`。  
如果尚未安装，请执行：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. 安装 GNU 编译器和autoconf, automake

macOS 自带的编译器为 `clang`，系统默认名称也叫 `gcc`，但并非真正的 GNU Compiler。  
这里，GRIST推荐使用 **GNU 12 编译器套件**：

```bash
brew install gcc@12
brew install autoconf
brew install automake
```

安装完成后，可使用以下命令：

```bash
gcc-12
g++-12
gfortran-12
autoconf
automake
aclocal
```

> 注意：**请使用版本号后缀 `-12` 的gcc-12, gfortran-12等可执行文件，以避免系统无法理解“gcc"是什么。**

---

## 二、安装辅助工具（推荐）

为便于结果后处理(如，水平插值)和可视化，建议安装以下工具：

```bash
brew install cdo
brew install nco
```

- **CDO**：气象数据运算与格式转换工具  
- **NCO**：NetCDF 文件处理与变量计算工具  

此外，建议安装 **NASA Panoply** 用于查看 `.nc` 文件图像。  
下载地址：[https://www.giss.nasa.gov/tools/panoply/]

---

## 三、编译模式依赖库

> 不推荐使用 `brew install` 安装模式依赖库，因为 Homebrew 的库通常基于 Clang 编译，会与 GNU 编译器不兼容。

请使用 GNU 12 全套编译器（`gcc-12`, `g++-12`, `gfortran-12`）从源码编译以下依赖：

| 库名 | 版本 | 说明 |
|------|------|------|
| METIS | 5.1.0 | 网格划分库 |
| LAPACK | 3.6.0 | 线性代数库 |
| OpenMPI | 4.1.2 | 并行通信框架 |
| PnetCDF | 1.12.2 | 并行 NetCDF 支持 |
| NetCDF-C / NetCDF-Fortran | 4.6.1/4.5.4 |netCDF数据格式支持 |

说明：
- 前四个库一般可顺利安装。
- `netcdf-c` 通常较稳定安装。
- `netcdf-fortran` 可能会遇到语法或版本兼容问题，建议多尝试几个版本。

其他平台（如 Ubuntu Linux）步骤类似，但路径和依赖版本可能不同。Linux发行版如自带标准 GNU 编译器，则采用其包管理器（如Ubuntu的apt）可直接安装与GNU编译器兼容的模式依赖库，无需教程第三步（编译模式依赖库）。

---

## 四、编译 GRIST 模式

安装完成依赖库后，进入 GRIST 源码目录：

```bash
cd /Users/zhangyi/model/GRIST/GRIST_kernel/build
```

创建可执行文件输出目录（如果不存在）：

```bash
mkdir -p /Users/zhangyi/model/GRIST/GRIST_kernel/bin/
```

编辑 `build.sh` 文件，根据本地环境修改：

示例片段：

```bash
model=$1
PREFIX=/Users/zhangyi/model/GRIST/GRIST_kernel/bin/

export NETCDF_PATH=/Users/zhangyi/softwares/netcdf-fortran-4.5.4/
export PNETCDF_PATH=/Users/zhangyi/softwares/pnetcdf-1.12.2/
export LAPACK_PATH=/Users/zhangyi/softwares/lapack-3.6.0/
export METIS_LIB_PATH=/Users/zhangyi/softwares/metis-5.1.0/lib/
export LIBTORCH_PATH=/fs2/home/zhangyi/wangym/model/test/libtorch/FTorch-intel   #本教程不需要设置

export FFLAGS="-ffree-line-length-512 -fallow-argument-mismatch -fallow-invalid-boz -m64"
export Fortran_Compiler="/Users/zhangyi/softwares/openmpi-4.1.2/bin/mpif90 ${FFLAGS}"
export CXX_Compiler=/Users/zhangyi/softwares/openmpi-4.1.2/bin/mpicxx
export netcdf_version="4"  #this is to distinguish netcdf code version(before 4.1 set 3; otherwise set 4)
export use_ftorch="no"    #
```

执行编译：

```bash
./build.sh gcm
```

编译完成后，`${PREFIX}` 目录下将生成 GRIST 可执行文件ParGRIST_gcm.exe（图2）。

![Figure 2](https://github.com/GRIST-Dev/GRIST-RunOnUrLapTop/blob/main/doc/Figure2_AfterCompile.png)
*图2. 模式编译成功后效果图*

---

## 五、运行湿大气斜压波算例

修改grist.nml文件里的: 

gridFilePath           = '/Users/zhangyi/model/inputdata/grid/' #设置为实际目录。

执行run.sh脚本即可运行模式。（可根据本地环境需要，修改metis库位置）

```bash
./run.sh
```

运行完毕后，界面如图3所示。

![Figure 3](https://github.com/GRIST-Dev/GRIST-RunOnUrLapTop/blob/main/doc/Figure3_AfterRun.png)
*图3. 模式运行成功后效果图*

---

## 六、结果分析与可视化

GRIST是非结构网格模式，输出文件包括 1d 2d 3d 文件，在文件名中有标识。其中，1d文件为球面2维信息，可通过cdo插值到经纬度网格

```bash
cdo -f nc remapdis,global_4 GRIST.ATM.G4UR.dtp.2020-06-09-00000.1d.h1.nc GRIST.ATM.G4UR.dtp.2020-06-09-00000.grid.h1.nc
```

使用 **Panoply** 打开 `GRIST.ATM.G4UR.dtp.2020-06-09-00000.grid.h1.nc` 查看模拟结果分布和演变（如ps表面气压）。


---

