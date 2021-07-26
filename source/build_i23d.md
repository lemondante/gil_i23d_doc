# 系统安装
**推荐选择dev分支版本的源码进行系统构建**

## 1. clone位于gitlab上dev分支的源码
gitlab与github在clone的行为上有所不同，github可以clone指定分支，但是gitlab一般默认只clone master分支。
因此clone代码后需要额外拉取dev分支的源码。
```
git clone git@8.140.146.21:root/dist_i23d.git
git pull origin dev:dev
git checkout dev
```
## 2.修改日志输出目录
日志输出位置主要存在于./src/exe/colmap.cc文件中，变量FLAGS_log_dir的值。
共有三处需要修改。它会被编译进可执行文件中，因此必须在编译前确定完毕。
## 3. 构建dist_i23d系统
由于dev分支里已经存在构建完成的build文件夹，因此首先删除build文件夹重新构建
重新新建build文件夹，执行cmake和make命令
命令类似如下：
```
cd dist_i23d
rm -r build
mkdir build && cd build
cmake .. && make -j8
```
执行完成后即构建完成
## 4. 修改脚本命令路径
默认配置下有三个脚本和一个config文件需要修改
#### 1. dist_run.sh
需要将Dataset_dir修改为实际数据集目录，并保证下一级有images这一文件夹，装着所有待重建图片
同时修改CONFIG_FILE_PATH为实际config.txt所在位置

#### 2. config.txt
将每台电脑的ip及端口写入“config.txt”文件，格式如下：
```
server_num
ip1 port1 dir_images_worker
ip2 port2 dir_images_worker
... ...
```
其中，第一项为各个worker机器的ip地址，第二项为worker的端口号，第三项为worker机器保存图片数据的目录（也就是master上的图片传到worker的这个目录下）。这个worker目录必须提前建立，否则程序直接退出。
**注意：dir_images_worker路径最后必须加上斜杠“/”，否则openMVS会无法读取到sfm生成的bin文件**

#### 3. run_worker.sh
原始的build文件夹内没有这个脚本文件，可以自行新建，放入dist_i23d/build/src/exe文件夹中，内容类似如下：
```
output_path=/data3/Dataset/elephant/worker/images/
sfm_path=../../../../dist_i23d
 
./colmap local_sfm_worker --output_path=$output_path     \
	--worker_output_path=$output_path \
	--run_mvs_path=$sfm_path/run_mvs.sh \
        --worker_cluster_path=$output_path \
        --SfM_path=$sfm_path

```
其中output_path应与config中的dir_images_worker保持一致，**路径最后同样要添加斜杠“/”**。
sfm_path即为dist_i23d文件目录

#### 4. auto_dense_reconstruction.sh
此为使用OpenMVS时需要修改的脚本文件。
主要修改如下路径：
+ mvs所在路径
mvs_path=/data1/dependency/openMVS_build/bin
+ colmap（i23d)所在路径（即dist_i23d/build/src/exe）
colmap_path=/data1/dependency/GraphSfM_test/xj_repo/dist_i23d/build/src/exe
