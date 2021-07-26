# i23d分布式docker安装指南(旧版) 
## Bare System Environment 
System: Ubuntu 18.04

gcc/g++: 6.5.0 || 7.5.0

nvidia-driver: 450

cuda: 10.0 || 11.0

graphics card: >= 1080
## Pipeline 
### 1. install nvidia-driver and cuda 
```
https://zhuanlan.zhihu.com/p/59618999
https://blog.csdn.net/sinat_34686158/article/details/106845208
```
### 2. install docker 
#### 卸载原有的docker 
```
sudo apt-get remove docker docker-engine docker.io containerd runc
```
#### 安装docker 
```
sudo apt-get update
# 安装依赖包
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
# 添加 Docker 的官方 GPG 密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# 验证您现在是否拥有带有指纹的密钥
sudo apt-key fingerprint 0EBFCD88
# 设置稳定版仓库
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
#### 安装 Docker-ce
```
# 更新
sudo apt-get update
# 列出可用版本
apt-cache madison docker-ce
# 安装合适版本的Docker-ce（比如我这个是18.06.3~ce~3-0~ubuntu） 
sudo apt-get install docker-ce=18.06.3~ce~3-0~ubuntu
# 启动
sudo systemctl enable docker
sudo systemctl start docker
```
#### 测试
```
sudo docker run hello-world
```
![Alt text](https://img2018.cnblogs.com/i-beta/1661179/201911/1661179-20191118102729705-1522389123.png "nvidia-docker")

### 3. install nvidia-docker
```
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -

curl -s -L https://nvidia.github.io/nvidia-docker/ubuntu18.04/nvidia-docker.list |   sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update

# 安装nvidia-docker2软件包并重新加载docker守护程序配置
sudo apt-get install nvidia-docker2
sudo pkill -SIGHUP dockerd

# 选择合适的版本
sudo nvidia-docker run --rm nvidia/cuda:10.1-devel nvidia-smi
```
### 4. install i23d in nvidia-docker (commit docker image if needed)
```
Make sure you have installed the NVIDIA driver and Docker engine for your Linux distribution Note that you do not need to install the CUDA Toolkit on the host system, but the NVIDIA driver needs to be installed
```

![Alt text](https://cloud.githubusercontent.com/assets/3028125/12213714/5b208976-b632-11e5-8406-38d379ec46aa.png "nvidia-docker")

## Usage
### 1. pull docker image from docker hub
``` 
docker pull keyepoch/dist_i23d_1804
```
### 2. run docker image with bash
```
nvidia-docker run -it -v ${LOCAL_DIR}:${DOCKER_MOUNT_DIR} ${IMAGE_ID} bash 
(eg: nvidia-docker run -it -v /data4/mydocker/:/data1/ 341d5b37f6e1 bash)
```
## Basic Requirements
```sh
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
##### [ceres-solver](https://github.com/ceres-solver/ceres-solver)
```sh
sudo apt-get install libatlas-base-dev libsuitesparse-dev
git clone https://github.com/ceres-solver/ceres-solver
cd ceres-solver
git checkout 1.14.x
mkdir build
cd build
cmake .. -DBUILD_TESTING=OFF -DBUILD_EXAMPLES=OFF
make
sudo make install
```
##### [igraph](https://igraph.org)
[igraph](https://github.com/igraph/igraph) is used for `Community Detection` and graph visualization.
```sh
sudo apt-get install build-essential libxml2-dev
wget https://igraph.org/nightly/get/c/igraph-0.7.1.tar.gz
tar -xvf igraph-0.7.1.tar.gz
cd igraph-0.7.1
./configure
make
make check
sudo make install
```
##### [rpclib](https://github.com/qchateau/rpclib)
```sh
git clone https://github.com/qchateau/rpclib
cd rpclib
mkdir build && cd build
cmake ..
make -j8
sudo make install
```
#### 2.2 Build GraphSfM
```sh
cd GraphSfM
mkdir build && cd build
cmake .. && make -j8
```
##### 注意事项
如果apt-get Eigen3编译不过：
```sh
# 移除apt-get版本
sudo apt-get autoremove libeigen3-dev
#github 有个mirror,版本3.3.4 from 2017
git clone https://github.com/eigenteam/eigen-git-mirror
#安装
cd eigen-git-mirror
mkdir build
cd build
cmake ..
sudo make install
```
如果apt-get安装boost编译不通过，则尝试手动编译安装
```sh
wget http://sourceforge.net/projects/boost/files/boost/1.58.0/boost_1_58_0.tar.gz
tar -zxvf boost_1_58_0.tar.gz
cd boost_1_58_0
# 使用bootstrap来生成编译工具b2
sudo ./bootstrap.sh
# 使用b2安装
sudo ./b2 install
```
如果gtest报错，则尝试手动编译安装
```sh
sudo apt-get install libgtest-dev
sudo apt-get install cmake # install cmake
cd /usr/src/gtest
sudo cmake CMakeLists.txt
sudo make
 
#copy or symlink libgtest.a and libgtest_main.a to your /usr/lib folder
sudo cp *.a /usr/lib
sudo cp *.a /usr/local/lib
```

