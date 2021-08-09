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

run_mvs.sh 字符串拼接方式传入三个参数 斜杠添加问题
因为原始join方法只能拼接路径，因此不能
auto_dense_reconstruction.sh

RunImageUndistorter

undistorter.start()

threadpool

UndistortReconstruction
UndistortCamera
write sparse

model_converter 输出格式

interfaceColmap
-i: input file
-o: output file

denseifyPointCloud

pointCloutFilter 去除不可见点？
八叉树加快搜索
大于阈值的都去除

DenseReconstruction
1. depth map computation
准备图片 筛选图片（未标定、无效的）
缩放一次图片（降采样）
读取分辨率合适的图片

对每个图片，选择所有有用的相邻图片
两步，1. 图片文件夹中的第一个图片并选择重建深度图的最佳视角，并提取所有相邻图片可见的3D点。
2.为相关视角选择 nNumViews 个对应视图（如果nNumViews为1，每张图片只选择最佳的视图。构造一个图，顶点是视角，如果互为邻居就用一条边连接起来

选择+筛选


构造一个处理队列
为图片选择一个视角去重建深度图

估计步骤：
首先，对稀疏点插值得到近似
从左上到右下
//对于每个像素，如果NCC分数更好，则首先将当前深度估计替换为其相邻估计。

//其次，通过尝试围绕当前深度和正常值的随机估计来改进估计，保持得分最佳的估计。

//估计可以在任何点停止，通常2-3次迭代就足以收敛。

//对于每个像素，深度和法线通过计算参考图像中的面片和目标图像中的包裹面片之间的NCC分数进行评分，该分数由待估计的当前值定义的单应矩阵决定。

//为了在局部估计每个像素时确保一定的平滑度，如果该像素的估计值与相邻像素的估计值接近，则会向NCC分数中添加额外的值。

//可选地，可以通过将所述迭代扩展到目标图像并移除在两个视图中不具有相似值的估计来检测被遮挡像素。 

1. fuse all depth-maps
fuse all valid depth-maps in the same 3D point cloud; join points very likely to represent the same 3D point and filter out points blocking the view

find best connected images
fuse all depth-maps, processing the best connected images first

立体对选择

从可用图像集中选择合适的立体对对于精确的深度图计算过程和整体重建至关重要。该算法使用SfM副产品计算可见点和相机中心之间的角度平均值。同样，计算两个光学中心之间的距离。对这些值（角度和距离）应用阈值后，剩余对的标量乘积的最小值就是被选为邻居的那个。对每个未失真图像（称为参考图像）重复此过程，以计算其相应的邻居图像（称为目标图像）。

深度图计算

使用PatchMatch算法计算每个立体对的深度贴图，可以细分为以下步骤。

使用PatchMatch算法计算每个立体对的深度贴图，可以细分为以下步骤。

随机初始化

参考图像中的每个像素用像素观察光线中的随机深度值和相机光学中心球坐标中的法线初始化。

注意：对于目标图像像素，此初始化不必是随机的。一旦参考图像像素的深度贴图计算完成，深度和法线可以扭曲到目标图像以计算初始估计。

单应与成本计算

对于参考图像中的每个像素p，放置固定大小的方形窗口/面片，以p为中心。

同样，对于每个面片中的每个像素，使用单应性在目标图像中找到对应的像素。然后，通过将每个面片中的像素的归一化互相关（NCC）代价相加，可以计算每个面片的匹配代价。

空间传播与随机分配

在多次迭代中，通过降低NCC分数，处理参考图像中的像素以改善与每个像素相关联的3D平面（即，其法线）。这种改进是通过两种操作实现的：空间传播和随机分配。

在空间传播中，具有较低NCC代价的相邻像素的平面被传播到正在处理的像素。

在空间传播之后，通过对法线的深度和球面参数进行微小的随机更改，进一步降低了匹配成本。

应用阈值

在上述两种操作之后，过滤出深度图中聚合匹配代价高于某个阈值的不可靠点。

深度映射细化

对于计算出的深度贴图中的每个像素，如果深度值在相邻图像中一致，则将其视为可靠的场景点，否则将从深度贴图中丢弃。

深度图合并

最后，合并在前面步骤中优化的深度贴图。此过程还包括冗余消除，因为深度贴图容易在多个贴图上具有相同的点。 

网格重建

delaunay四面体化，可见权重+1
然后就是文章重点讲的第二阶段做图割，文章重点介绍了一种计算每个点的权重的方法，主要是基于可视性的变化。大概的过程是，如果空间中一块区域（一个四面体？）有一个摄像头可以看到点云中的点，权重就加一，注意不是说这块区域能被一个摄像头看到就加一，而是视线穿过这块区域能看到点云中的点才加一，而哪个点能被哪几个摄像头看到在之前做三角化的时候就确定了，这里就不多说了。有了点的权重就可以计算四面体的权重，主要就是四面体内的点的权重相加，文章也考虑了点云的分布，点的权重也不是简单的一个摄像头就加一，但你可以把这看成一种优化，不影响过程的理解。有了四面体的权重就可以计算视线（摄像头中心到点云某点的射线）在经过该点前后一定距离内可视性的变化，文章主要考虑了三个权重，绝对变化，相对变化和一个啥。构成一个目标函数计算这个点是不是模型表面点。得到这些表面点这些点所构成的三角网格就是模型表面的网格。


mesh refinement
High Accuracy and Visibility-Consistent Dense Multiview Stereo
之前得到的网格细化

材质
纹理提取
Let There Be Color! - Large-Scale Texturing of 3D Reconstructions 





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

并查集 小于100忽略

### 3.分割图N-cut

### 4.运行（sfm+MVS)

每三秒 找一个空闲worker



SfMDataContainer 继承了 TaskDataContainer
TaskDataContainer 中的虚函数DistributeTask 在 SfMDataContainer 中被重写





### 5.再次三角化

### 6.全局BA

### 7.提取颜色


