# 系统运行流程

![Alt text](http://colmap.github.io/_images/incremental-sfm.png "算法流程")

根据提供的worker机数量，以及数据集中的照片数量，可选择运行并行分布式或单机串行版系统。

## worker进程

系统首先调用worker脚本启动worker进程，worker脚本如下：

```sh
output_path=/home/keyepoch/Downloads/car/worker/images/
sfm_path=/home/keyepoch/Downloads/i23d/dist_i23d
 
./colmap local_sfm_worker --output_path=$output_path     \
	--worker_output_path=$output_path \
	--run_mvs_path=$sfm_path/run_mvs.sh \
        --worker_cluster_path=$output_path \
        --SfM_path=$sfm_path

```

调用了文件colmap.cc中的`RunLocalSfMWorker`函数，该函数读取完命令行参数后，首先设定日志目录`FLAGS_log_dir`参数。随后构建类IncrementalMapperController，这个类不仅是执行SfM过程，也执行MVS过程。实例化一个`mapper`对象后，将其作为rpc中的server。

这个函数被rpc框架与`“RunSfM”`字符串关联，由master决定在什么时候调用这个函数。

这个`mapper`对象包含的功能非常多，基本可以认为是整个流程中，除去特征提取和词汇树匹配以外的全部内容。它执行`start`方法后会调用重写的`run`函数，执行Sfm重建工作和mvs模块。

`run`函数内部首先是Reconstruct函数，构建一个IncrementalMapper类，对象同样命名为`mapper`（注意不要和上面的mapper搞混）。这个mapper读入参数后执行`BeginRestruction`方法，开始初始化重建工作。

首先先读入数据库中的内容，构建匹配图（注意，特征提取和词汇树匹配都是之前已完成的工作，worker进程从重建开始）。构造一个两视图几何的类，保存在属性`prev_init_two_view_geometry_`里。随后查找初始图像对（如果没有提供初始匹配对），用`EstimateInitialTwoViewGeometry`函数评估是否合适，具体是通过`FindCorrespondencesBetweenImages`函数返回匹配的特征数量，并且也判断初始相机是否满足要求。找到初始的两张图片对后，需要注册初始图片对，也就是`RegisterInitialImagePair`函数。注册同样需要用`EstimateInitialTwoViewGeometry`函数评估是否合适，利用对极几何生成初始的相机参数，然后构造一个四元数作为姿态。注册就是将图片标记入track的意思。同时三角化新的点，重建track。
此时调用结束重建函数`EndReconstruction`清除数据。此时做一次全局BA，对图片和点还要做一次滤波。

随后开始增量式匹配重建。在while循环内调用`FindNextImages`函数不断搜索下一个图片，并将其注册，直到注册失败。如果注册成功则进行三角化新的点，并做本地BA（不是全局）。如果这次选择的初始对失败了，则放弃这个初始对并重新选择一对。如果重叠超过预设值也将跳出循环。如果没有图片可以被注册，则进行一次全局的迭代BA并再次注册。如果仍然失败则退出增量式匹配。在中途是不进行全局BA的，所以如果最后一次不是全局BA，则再最后做一次全局BA。

此时SfM流程结束。这个是函数Reconstruct的流程。如果失败会将条件限制放宽并再次重建。重建完后将bin文件输出到worker机指定路径。

MVS


## master进程

在master进程
