# 环境配置

## 配置要求

<table>
<tr>
<td>环境</td>
<td>要求</td>
</tr>
<tr>
<td>系统环境</td>
<td>Ubuntu 18.04</td>
</tr>
<tr>
<td>GPU依赖</td>
<td>Titan Xp或更高，Cuda>=9.0</td>
</tr>
<tr>
<td>内存</td>
<td>16G+，推荐64G+</td>
</tr>
<tr>
<td>gcc,g++</td>
<td>6.0</td>
</tr>
<tr>
<td>OpenMVS</td>
<td>1.1</td>
</tr>
<tr>
<td>OpenCV</td>
<td>4.x</td>
</tr>
<tr>
<td>python</td>
<td>3.6(pyTorch), 2.7(Tensorflow)</td>
</tr>
<tr>
<td>pyTorch</td>
<td>>=1.0.0</td>
</tr>
<tr>
<td>Tensorflow</td>
<td>>=1.4.0</td>
</tr>
</table>

## 本系统依赖的环境如下：

### 基础环境
```
sudo apt-get install \
    git \
    cmake \
    build-essential \
    libboost-program-options-dev \
    libboost-filesystem-dev \
    libboost-graph-dev \
    libboost-regex-dev \
    libboost-system-dev \
    libboost-test-dev \
    libeigen3-dev \
    libsuitesparse-dev \
    libfreeimage-dev \
    libgoogle-glog-dev \
    libgflags-dev \
    libglew-dev \
    qtbase5-dev \
    libqt5opengl5-dev \
    libcgal-dev \
    libcgal-qt5-dev
```

### [rpclib](https://github.com/qchateau/rpclib)

```sh
git clone https://github.com/qchateau/rpclib.git
cd rpclib
mkdir build && cd build
cmake ..
make -j8
sudo make install
```

### [openMVS](https://github.com/cdcseacave/openMVS)

openMVS源码必须使用90服务器上改造后的源码，**原始github上的不能够读取分布式sfm的结果**。
安装openMVS依赖库包括：
+ Eigen
+ Boost
+ OpenCV
+ CGAL
+ VCGLib
+ Ceres
+ GLFW3
虽然源码不同，但是安装仍可参考它的build说明。
其中ceres-solver与build说明不同，不可编译默认分支的ceres-solver，否则报如下错误：
```
trust_region_minimizer.cc:91] Terminating: Number of consecutive invalid steps more than Solver::Options::max_num_
consecutive_invalid_steps: 5  
```
必须切换为1.14.x版本再进行编译。

**也可直接使用以下脚本命令一键部署环境。**

```
#!/bin/bash
#Prepare and empty machine for building:
sudo apt-get update -qq && sudo apt-get install -qq
sudo apt-get -y install git cmake libpng-dev libjpeg-dev libtiff-dev libglu1-mesa-dev
main_path=`pwd`
echo "main_path=$main_path"

#Eigen (Required)
git clone https://gitlab.com/libeigen/eigen.git --branch 3.2
mkdir eigen_build && cd eigen_build
cmake . ../eigen
make && sudo make install
cd ..

#Boost (Required)
sudo apt-get -y install libboost-iostreams-dev libboost-program-options-dev libboost-system-dev libboost-serialization-dev

#OpenCV (Required)
sudo apt-get -y install libopencv-dev

#CGAL (Required)
sudo apt-get -y install libcgal-dev libcgal-qt5-dev

#VCGLib (Required)
git clone https://github.com/cdcseacave/VCG.git vcglib

#Ceres (Optional)
sudo apt-get -y install libatlas-base-dev libsuitesparse-dev
git clone https://ceres-solver.googlesource.com/ceres-solver ceres-solver
cd ceres-solver
git checkout 1.14.x
cd ..
mkdir ceres_build && cd ceres_build
cmake . ../ceres-solver/ -DMINIGLOG=ON -DBUILD_TESTING=OFF -DBUILD_EXAMPLES=OFF
make -j2 && sudo make install
cd ..

#GLFW3 (Optional)
sudo apt-get -y install freeglut3-dev libglew-dev libglfw3-dev
```

**在同一级目录下放入90服务器上的openMVS代码，并执行如下编译操作：**

```
mkdir openMVS_build && cd openMVS_build
cmake . ../openMVS -DCMAKE_BUILD_TYPE=Release -DVCG_ROOT="$main_path/vcglib"
make -j2 && sudo make install
```
此时运行环境配置完成。