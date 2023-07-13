# 系统使用

## 1. 修改脚本命令路径

默认配置下有两个脚本文件和一个config文件需要修改

#### 1. dist_run.sh

需要将Dataset_dir修改为实际数据集目录，并保证下一级有images这一文件夹，装着所有待重建图片。最新版本的系统不需要保证有images文件夹。

**注意：路径最后不需要加上斜杠“/”**
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
#example
#add this to build/src/exe/
mkdir -p /dependency/dist_i23d/example 

output_path=/worker/elephant/images/
sfm_path=/dependency/dist_i23d
 
./colmap local_sfm_worker --output_path=$output_path \
	--worker_output_path=$output_path \
	--run_mvs_path=$sfm_path/run_mvs.sh \
        --worker_cluster_path=$output_path \
        --SfM_path=$sfm_path \
        --multiple_models=true

```
其中output_path应与config中的dir_images_worker保持一致，**路径最后同样要添加斜杠“/”**。
sfm_path即为dist_i23d文件目录

#### 4. auto_dense_reconstruction.sh(已弃用)

此为使用OpenMVS时需要修改的脚本文件。
主要修改如下路径：
+ mvs所在路径
mvs_path=./openMVS_build/bin
+ colmap（i23d)所在路径（即dist_i23d/build/src/exe）
colmap_path=./dist_i23d/build/src/exe

#### 5. scripts/shell/auto_dense_reconstruction_deep_learning.sh

此为老流程的稠密重建流程。主要包含图像去畸变、MVS、网格重建、网格简化、纹理映射等。
内容基本固定，不需要修改。其在worker内生效，即不受master控制。
可根据流程需要每个worker独立控制参数细节，此处不再赘述。

## 2. 使用tmux创建worker会话

### 安装tmux

```
sudo apt-get install tmux
```

### 创建新的worker会话

```
tmux new -s worker
```

进入tmux会话窗口后启动worker进程（**一台主机只能启动一个worker会话**）
```
sudo sh run_worker.sh
```

### 创建新的master会话

```
tmux new -s master
```

进入tmux会话窗口后启动master进程
```
sudo sh dist_run.sh
```

等待运行完毕即可。

如果在docker内运行则不需要sudo。

## 3. 重建结果查看

重建结果为obj文件，docker内的路径为/data1/xxx/sparse，可放入MeshLab中查看具体的重建效果。
对于分块后的模型，在对应的模型文件夹内。未分块模型需要手动将模型移入模型文件夹后打开。

