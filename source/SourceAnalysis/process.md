# 系统运行流程

![Alt text](http://colmap.github.io/_images/incremental-sfm.png "算法流程")

根据提供的worker机数量，以及数据集中的照片数量，可选择运行并行分布式或单机串行版系统。

## worker进程

系统首先调用worker脚本`run_worker.sh`启动worker进程，脚本如下：

```sh
output_path=/home/keyepoch/Downloads/car/worker/images/
sfm_path=/home/keyepoch/Downloads/i23d/dist_i23d
 
./colmap local_sfm_worker --output_path=$output_path     \
	--worker_output_path=$output_path \
	--run_mvs_path=$sfm_path/run_mvs.sh \
        --worker_cluster_path=$output_path \
        --SfM_path=$sfm_path

```

此脚本调用了文件colmap.cc中的`RunLocalSfMWorker`函数，参数主要是各种路径。该函数读取完命令行参数后，首先设定日志目录`FLAGS_log_dir`参数。随后构建类IncrementalMapperController，这个类不仅是执行SfM过程，也执行MVS过程。实例化一个`mapper`对象后，将其作为rpc中的server。

这个函数被rpc框架与`“RunSfM”`字符串关联，由master决定在什么时候调用这个函数。

这个`mapper`对象包含的功能非常多，基本可以认为是整个流程中，除去特征提取和词汇树匹配以外的全部内容。它执行`start`方法后会调用重写的`run`函数，执行Sfm重建工作和mvs模块。

`run`函数内部首先是Reconstruct函数，构建一个IncrementalMapper类，对象同样命名为`mapper`（注意不要和上面的mapper搞混）。这个mapper读入参数后执行`BeginRestruction`方法，开始初始化重建工作。

首先先读入数据库中的内容，构建匹配图（注意，特征提取和词汇树匹配都是之前已完成的工作，worker进程从重建开始）。构造一个两视图几何的类，保存在属性`prev_init_two_view_geometry_`里。随后查找初始图像对（如果没有提供初始匹配对），用`EstimateInitialTwoViewGeometry`函数评估是否合适，具体是通过`FindCorrespondencesBetweenImages`函数返回匹配的特征数量，并且也判断初始相机是否满足要求。找到初始的两张图片对后，需要注册初始图片对，也就是`RegisterInitialImagePair`函数。注册同样需要用`EstimateInitialTwoViewGeometry`函数评估是否合适，利用对极几何生成初始的相机参数，然后构造一个四元数作为姿态。注册就是将图片标记入track的意思。同时三角化新的点，重建track。
此时调用结束重建函数`EndReconstruction`清除数据。此时做一次全局BA，对图片和点还要做一次滤波。

随后开始增量式匹配重建。在while循环内调用`FindNextImages`函数不断搜索下一个图片，并将其注册，直到注册失败。如果注册成功则进行三角化新的点，并做本地BA（不是全局）。如果这次选择的初始对失败了，则放弃这个初始对并重新选择一对。如果重叠超过预设值也将跳出循环。如果没有图片可以被注册，则进行一次全局的迭代BA并再次注册。如果仍然失败则退出增量式匹配。在中途是不进行全局BA的，所以如果最后一次不是全局BA，则再最后做一次全局BA。

此时SfM流程结束。这个是函数Reconstruct的流程。如果失败会将条件限制放宽并再次重建。重建完后将bin文件输出到worker机指定路径。

MVS


## master进程

在启动worker进程后，调用脚本`dist_run.sh`启动master进程。脚本内容如下：

```sh
#需要将Dataset_dir修改为实际数据集目录，并保证下一级有images这一文件夹，装着所有待重建图片#
Dataset_dir=/data1/GZDT
###################################################################################

CONFIG_FILE_PATH=/data1/dependency/GraphSfM_test/xj_repo/dist_i23d/config.txt

SfM_path=`pwd`

# sparse recon
$SfM_path/scripts/shell/zh_distributed_sfm.sh \
$Dataset_dir \
200 \
log \
$SfM_path/build/src/exe \
$SfM_path/vobtree/vocab_tree_flickr100K_words1M.bin \
$CONFIG_FILE_PATH
```

此脚本初始化路径信息后，调用了`zh_distributed_sfm.sh`脚本，参数主要也是路径（包括数据集路径、config文件路径、sfm运行路径、词汇树路径等）以及每堆张数限制。接下来是`zh_distributed_sfm.sh`脚本的内容：

```sh
DATASET_PATH=$1
num_images_ub=$2
log_folder=$3
colmap_path=$4
VOC_TREE_PATH=$5
CONFIG_FILE_PATH=$6
# completeness_ratio=$4
# VOC_TREE_PATH=$5
# image_overlap=$3
# max_num_cluster_pairs=$4

#mkdir -p $DATASET_PATH/$log_folder

${colmap_path}/colmap feature_extractor \
--database_path=$DATASET_PATH/database.db \
--image_path=$DATASET_PATH/images \
--SiftExtraction.num_threads=8 \
--SiftExtraction.use_gpu=1 \
--SiftExtraction.gpu_index=0

#  ${colmap_path}/colmap exhaustive_matcher \
#  --database_path=$DATASET_PATH/database.db \
# --SiftMatching.num_threads=8 \
# --SiftMatching.use_gpu=1 \
# --SiftMatching.gpu_index=0

# Or use vocabulary tree matcher
${colmap_path}/colmap vocab_tree_matcher \
--database_path=$DATASET_PATH/database.db \
--SiftMatching.num_threads=8 \
--SiftMatching.use_gpu=1 \
--SiftMatching.gpu_index=0 \
--VocabTreeMatching.num_images=30 \
--VocabTreeMatching.num_nearest_neighbors=5 \
--VocabTreeMatching.vocab_tree_path=$VOC_TREE_PATH

${colmap_path}/colmap distributed_mapper \
$DATASET_PATH/$log_folder \
--database_path=$DATASET_PATH/database.db \
--transfer_images_to_server=1 \
--image_path=$DATASET_PATH/images \
--output_path=$DATASET_PATH/sparse \
--config_file_name=$CONFIG_FILE_PATH \
--num_workers=8 \
--distributed=1 \
--repartition=0 \
--assign_cluster_id=1 \
--write_binary=0 \
--retriangulate=0 \
--final_ba=0 \
--select_tracks_for_bundle_adjustment=1 \
--long_track_length_threshold=10 \
--graph_dir=$DATASET_PATH/$log_folder \
--num_images_ub=$num_images_ub \
--completeness_ratio=0.7 \
--relax_ratio=1.3 \
--cluster_type=NCUT #SPECTRA
```

此脚本主要调用三个模块：sift特征提取模块、词汇树匹配模块和分布式映射模块。核心来看分布式映射模块。

`distributed_mapper`主要调用了文件colmap.cc中的`RunDistributedMapper`函数，此函数接收的参数较多，以下是参数的含义：


`RunDistributedMapper`函数接收了参数并存放到option中后，构建了两个类`ReconstructionManager`和`DistributedMapperController`，分别实例化了`reconstruction_manager`和`distributed_mapper`对象，其中`DistributedMapperController`类存放了控制信息，也就是option中的信息。随后启动`distributed_mapper`函数。

`distributed_mapper`函数主要有八个步骤。

### 1.读取数据

借助封装的函数`LoadData`用SQLite模块读取图片信息

### 2.提取最大联通分量

### 3.分割图N-cut

### 4.运行（sfm+MVS)

### 5.再次三角化

### 6.全局BA

### 7.提取颜色


