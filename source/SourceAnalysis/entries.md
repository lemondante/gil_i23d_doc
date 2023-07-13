# 系统功能

i23d系统围绕三维重建流程，包含众多丰富的功能函数。借助*colmap*优秀的结构化设计，i23d在截止SfM为止的大部分功能都被彻底的暴露在外，只需对`dist_i23d`文件夹内的`build/src/exe/colmap`文件进行`shell`调用即可执行。SfM后的流程，则由`shell`脚本调用外部模块完成。本章简述i23d系统可以直接外部调用的功能，分类为定位流程与重建流程，分别代表SfM之前与SfM之后。

## 定位流程

本节所描述的功能目前主要被`/dependency/dist_i23d/scripts/shell/zh_distributed_sfm.sh`文件所调用。其由master服务器进行控制。

### gui
对应函数`RunGraphicalUserInterface`。此函数主要为i23d的图形界面，与colmap功能类似。可能存在尚未更新完全的部分，谨慎使用。

### automatic_reconstructor
对应函数`RunAutomaticReconstructor`。Colmap的自动重建选项，i23d已弃用。

### bundle_adjuster
对应函数`RunBundleAdjuster`。Colmap的BA优化器的对外暴露接口。i23d较少使用。

### color_extractor
对应函数`RunColorExtractor`。i23d较少使用。

### database_creator
对应函数`RunDatabaseCreator`。用于创建数据库的对外暴露接口。i23d较少使用。

### database_merger
对应函数`RunDatabaseMerger`。用于合并数据库的对外暴露接口。i23d较少使用。

### distributed_mapper
对应函数`RunDistributedMapper`。用于Master进程的开启。i23d核心函数，由`dist_i23d.sh`调用`zh_distributed_sfm.sh`间接开启。其运行日志保存在变量`FLAGS_log_dir`指定的位置。

该函数接受的参数如下：

+ `database_path`：字符串变量，表示Sqlite数据库路径。可在`zh_distributed_sfm.sh`文件中修改。默认为图片images文件夹父路径。新版i23d取消了images文件夹限制，则默认为图片文件夹路径。数据库默认命名为`database.db`。

+ `image_path`：字符串变量，表示图片路径，可在`dist_i23d.sh`文件中修改，自动传递至`zh_distributed_sfm.sh`。早期版本的i23d与colmap类似，需要将输入图片的文件夹格式设置为类似“数据集名/images/具体图片.JPG”的格式。新版图片路径只需设置为数据集最外层文件夹名即可，系统会自动扫描文件夹内所有图片并保存图片名与图片相对路径于Sqlite数据库中便于查询。

+ `output_path`：字符串变量，表示输出路径，可在`dist_i23d.sh`文件中修改，自动传递至`zh_distributed_sfm.sh`。colmap中为输出场景的稀疏表示与相机信息的路径，即SfM结果，通常包括`cameras.txt/bin`、`images.txt/bin`、`points3d.txt/bin`三个或六个文件。i23d中会根据具体的分堆信息，将每个堆的SfM结果分别输出至此路径，并根据设置进行合并等工作。此外，i23d会将后续的稠密重建流程得到的三维模型与纹理贴图一齐传输至该路径。该路径通常包含**数字**、**obj_数字**、**partition数字**三个文件夹，与**model_数字.obj**文件。数字为分堆的堆号。**数字**文件夹中包含的是经过对齐是SfM结果，**obj_数字** *此处待修改*

+ `transfer_images_to_server`：布尔型变量，表示是否将图像从主服务器传递给从服务器。可在`zh_distributed_sfm.sh`文件中修改。通常设置为1，因为后续稠密重建流程依赖原始图像。在新流程中初期仅传输数据库内的图像特征信息与匹配结果，但不建议将该值改为0。

+ `num_workers`：uint型变量，表示Master服务器的最大线程数量，可在`zh_distributed_sfm.sh`文件中修改。理论上最佳设置为服务器的核心数或线程数，但区别不大。算法基本可以吃满服务器。

+ `distributed`：布尔型变量，表示是否采用分布式计算，可在`zh_distributed_sfm.sh`文件中修改。通常设置为1，即使图片数量较少以至于不需要分堆。当该项设置为1时，必须保证`config_file_name`项非空。

+ `config_file_name`：字符串变量，表示`config.txt`文件所在位置，可在`dist_i23d.sh`文件中修改，自动传递至`zh_distributed_sfm.sh`。通常该文件与`dist_i23d.sh`文件位于同一文件夹下，内容为从服务器的数量、ip地址与端口号。

+ `repartition`：布尔型变量，表示是否需要为MVS作重新分配，通常设置为0，可在`zh_distributed_sfm.sh`文件中修改。此为i23d旧版参数，与后续的新流程无关，建议忽视该参数。

+ `assign_cluster_id`：布尔型变量，表示是否给每个图片赋予一个cluster_id，通常设置为1，可在`zh_distributed_sfm.sh`文件中修改。分布式重建过程中图片需要获得分配至某一物理机的标识，故不建议修改为0。

+ `write_binary`：布尔型变量，表示是否用二进制输出SfM结果，通常设置为1，可在`zh_distributed_sfm.sh`文件中修改。由于二进制文件读写速度较快，且文件尺寸较小，故建议开启。文本格式的SfM结果自动保存，无需关闭。

+ `retriangulate`：布尔型变量，表示在SfM运算完成并合并后是否需要重三角化，通常设置为0，可在`zh_distributed_sfm.sh`文件中修改。由于重三角化需要大量的时间，且在老流程中分堆合并是在三维模型重建完后进行，故重三角化对输出结果没有增益，因此不需要开启。但在新流程中由于提前进行了分堆合并，故可以结合`final_ba`共同优化场景信息，提高后续三维重建的效果。

+ `final_ba`：布尔型变量，表示在SfM运算完成并合并后是否需要重新BA，通常设置为0，可在`zh_distributed_sfm.sh`文件中修改。与上一条参数类似，在新流程中可以考虑设置为1。

+ `select_tracks_for_bundle_adjustment`：布尔型变量，表示在重新BA时是否选择合适的track进行局部BA替代完整BA，通常设置为1，可在`zh_distributed_sfm.sh`文件中修改。本参数可以在几乎不降低BA效果的前提下减少BA的消耗时间。

+ `long_track_length_threshold`：uint型变量，表示选择track时的最大track长度，可在`zh_distributed_sfm.sh`文件中修改。默认设置为10。需结合`select_tracks_for_bundle_adjustment`参数使用。track长度过长可能会造成外点增多，故选择高质量track需要设置最长track值。

+ `image_grid_cell_size_pixels`：uint型变量，表示BA时选择的图像cell大小，可在`zh_distributed_sfm.sh`文件中修改。默认设置为100。使得图像的某个cell参与BA，能够降低匹配与BA的运算量。

+ `min_num_optimized_tracks_per_view`：uint型变量，表示每个视角的track下采样时的最小track数量，可在`zh_distributed_sfm.sh`文件中修改。默认设置为100。

+ `image_overlap`：uint型变量，表示每个子堆的共同图片数量，可在`zh_distributed_sfm.sh`文件中修改。默认设置为50。此设置效果存疑。

+ `num_images_ub`：uint型变量，表示每个子堆的图片数量上限，不包括后续补充的重叠图片，可在`dist_i23d.sh`文件中修改，自动传递至`zh_distributed_sfm.sh`。数据集被分割的堆数等于总数除以每堆图片上限，向上取整。通常建议设置为100-500之间的数，具体可根据数据集图片张数决定。例如当数据集中包含300张图片时，可直接设置为300确保不分堆，以减少重复图片带来的效率降低。

+ `completeness_ratio`：float型变量，表示每个子堆完整性比例，可在`zh_distributed_sfm.sh`文件中修改。通常建议设置为0.7效果较好。

+ `relax_ratio`：float型变量，表示每个子堆可扩展程度，可在`zh_distributed_sfm.sh`文件中修改。通常建议设置为1.3效果较好。

+ `max_num_cluster_pairs`：uint型变量，表示每个子堆之间的最大匹配数量，可在`zh_distributed_sfm.sh`文件中修改。默认设置为0，应该已被弃用。

+ `cluster_type`：字符串变量，表示分堆的算法名，可在`zh_distributed_sfm.sh`文件中修改。支持`NCut`和`Spectral`两种分割算法。默认采用`NCut`算法。`Spectral`准求度更高但运行较慢。

+ `graph_dir`：字符串变量，表示图像连接图的位置，可在`zh_distributed_sfm.sh`文件中修改。已经弃用，目前由某已弃用的日志文件目录替代。

以上为老流程中该模块接受参数的介绍。新流程中多了数个新参数，分别介绍如下：



### exhaustive_matcher
对应函数`RunExhaustiveMatcher`。colmap默认的暴力匹配模块，i23d不再使用。

### feature_extractor
对应函数`RunFeatureExtractor`。特征提取模块，采用SIFT算子对图像特征进行提取。主要参数为`image_list_path`，字符串变量，表示输入图像文件夹位置，可在`dist_i23d.sh`文件中修改，自动传递至`zh_distributed_sfm.sh`。内部包含`ImageReader`模块能够逐张读入图像并直接写入至数据库。`SiftFeatureExtractor`模块用于特征提取，可以通过阅读`SiftFeatureExtractorThread`类的内部实现理解特征的提取过程。

### feature_importer
对应函数`RunFeatureImporter`。用于读取已保存至数据库的特征的接口。

### hierarchical_mapper
对应函数`RunHierarchicalMapper`。colmap的层次化重建模块。i23d未启用。

### local_sfm_worker
对应函数`RunLocalSfMWorker`。用于Worker进程的开启。i23d核心函数，由`run_worker.sh`调用直接开启。其运行日志保存在变量`FLAGS_log_dir`指定的位置。

该函数接受的参数如下：

+ `output_path`：字符串变量，表示worker服务器的中间过程的输出目录，通常设置为`/worker/xxx/images/`，注意最后斜杠需要保留防止路径拼接错误。此变量可能是多余的。

+ `worker_output_path`：字符串变量，表示worker服务器的中间过程的输出目录，通常设置为`/worker/xxx/images/`，注意最后斜杠需要保留防止路径拼接错误。与上一个变量一致，但会被调用。

+ `run_mvs_path`：字符串变量，为colmap调用`run_mvs.sh`文件的路径，通常设置为`/dependency/dist_i23d/run_mvs.sh`。

+ `worker_cluster_path`：字符串变量，表示worker服务器的每个堆的输出目录，通常设置为`/worker/xxx/images/`，注意最后斜杠需要保留防止路径拼接错误。此变量会被调用，但可以被认为与`worker_output_path`一致。

+ `SfM_path`：字符串变量，表示i23d系统所在路径，通常设置为`/dependency/dist_i23d`。

+ `multiple_models`：bool型变量，表示在一个堆中若产生了多个联通分量，是否重建所有的子堆而不是只选择一堆。通常设置为true。



### sfm_aligner
对应函数`RunSfMAligner`。此为分堆对齐模块，可以对齐分堆后的SfM结果至同一坐标系。核心函数为`sfm_aligner.Align()`，主要用于分堆debug。目前其依赖手动输入已有分堆数，无法自动检测所有分堆，故需要保持分堆文件夹内没有其他文件干扰。

### point_cloud_segmenter
对应函数`RunPointCloudSegmenter`。colmap自带函数，几乎不使用。

### image_deleter
对应函数`RunImageDeleter`。colmap自带函数，几乎不使用。

### image_filterer
对应函数`RunImageFilterer`。colmap自带函数，几乎不使用。

### image_rectifier
对应函数`RunImageRectifier`。colmap自带函数，几乎不使用。

### image_registrator
对应函数`RunImageRegistrator`。colmap自带函数，几乎不使用。

### image_undistorter
对应函数`RunImageUndistorter`。colmap自带函数，用于对图像去畸变并将simple_redial格式的相机变为pinhole。其有bug，min_scale设置为0.8，否则会过度剪裁。

### mapper
对应函数`RunMapper`。colmap自带函数，几乎不使用。

### matches_importer
对应函数`RunMatchesImporter`。colmap自带函数，几乎不使用。

### model_aligner
对应函数`RunModelAligner`。colmap自带函数，几乎不使用。

### model_analyzer
对应函数`RunModelAnalyzer`。colmap自带函数，几乎不使用。

### model_converter
对应函数`RunModelConverter`。i23d的输出格式转换模块。i23d可以在`bin`、`txt`、`nvm`、`bundler`、`ply`、`vrml`等格式之间转换。接收的参数如下：

+ `input_path`：字符串型变量。表示输入的稀疏场景的文件路径。

+ `output_path`：字符串型变量。表示输入的稀疏场景的文件路径。
  
+ `output_type`： 字符串型变量。表示输入的稀疏场景的文件路径。


### model_merger
对应函数`RunModelMerger`。colmap自带函数，几乎不使用。


### model_orientation_aligner
对应函数`RunModelOrientationAligner`。colmap自带函数，几乎不使用。

### point_filtering
对应函数`RunPointFiltering`。colmap自带函数，几乎不使用。

### point_triangulator
对应函数`RunPointTriangulator`。colmap自带函数，几乎不使用。

### project_generator
对应函数`RunProjectGenerator`。colmap自带函数，几乎不使用。

### rig_bundle_adjuster
对应函数`RunRigBundleAdjuster`。colmap自带函数，几乎不使用。

### sequential_matcher
对应函数`RunSequentialMatcher`。colmap自带函数，效果提升不明显故几乎不使用。

### stochastic_matcher
对应函数`RunStochasticMatcher`。i23d独有的采样式匹配模块，用于在进行粗略的词汇树匹配后增强匹配数量。算法细节参考徐军的《一种基于采样的增量式运动恢复结构方法》。其日志输出目录为`/dependency/dist_i23d/logs/match_logs/`。接收的参数如下：

+ `extra_info_dir`：字符串变量，`extro_info`文件夹路径。可在`dist_i23d.sh`文件中修改，自动传递至`zh_distributed_sfm.sh`。默认路径为`/dependency/dist_i23d/extra_info`。
+ `database_path`：字符串变量，数据库路径，保存图片信息与特征信息。可在`dist_i23d.sh`文件中修改，自动传递至`zh_distributed_sfm.sh`。
+ `SiftMatching.num_threads`：整型变量，表示匹配时的最大线程数，通常与CPU线程数保持一致。
+ `SiftMatching.use_gpu`：布尔型变量，表示是否采用GPU加速。通常设置为1。
+ `SiftMatching.gpu_index`：整型变量，表示使用的GPU编号。单卡通常设置为0。

**注意**：在进行特征匹配的时候，是根据已有的匹配关系继续进行迭代的；若需要从头开始，切记删除`database.db`文件后进行。

### spatial_matcher
对应函数`RunSpatialMatcher`。colmap自带函数，效果提升不明显故几乎不使用。

### transitive_matcher
对应函数`RunTransitiveMatcher`。colmap自带函数，效果提升不明显故几乎不使用。

### vocab_tree_builder
对应函数`RunVocabTreeBuilder`。colmap自带函数，训练一棵词汇树。通常使用预训练的词汇树进行匹配，故不常使用。

### vocab_tree_matcher
对应函数`RunVocabTreeMatcher`。colmap自带函数，根据提取的特征构建一棵词汇树进行匹配。通常作为`stochastic_matcher`的预匹配过程。与`stochastic_matcher`模块相比额外的接收参数如下：

+ `VocabTreeMatching.num_images`：整型变量，表示每张图片检索的最近的图片数。当仅使用词汇树作为匹配手段时推荐设置为100-150。当结合`stochastic_matcher`模块一起工作时推荐设置为30-50。此参数对匹配效率影响较大。
+ `VocabTreeMatching.num_nearest_neighbors`：整型变量，表示每个特征检索的最近的视觉词汇数量。
+ `VocabTreeMatching.vocab_tree_path`：字符串变量，表示预训练的词汇树路径。


### vocab_tree_retriever
对应函数`RunVocabTreeRetriever`。colmap自带函数，几乎不使用。

## 重建流程

本节所描述的功能主要被`/dependency/dist_i23d/scripts/shell/auto_dense_reconstruction_deep_learning.sh`文件所调用。其由worker端控制。

### MVS

i23d系统的MVS主要支持两种方法，分别为基于Patch Match算法的传统MVS与基于深度学习的H-RMVSNet。后者的点云密度显著高于前者，运行时间相仿。故推荐采用后者。其接收参数分别为去畸变后pinhole格式的相机内外参与稀疏点云、去畸变后的待重建图像、输出结果目录。具体算法原理参见魏子庄博士的《复杂场景的鲁棒高效三维稠密点云重建与点云配准》。

MVS的参数调整主要位于`/dependency/i23d_mvs/densify_learning.sh`文件内，包含的主要参数如下：

+ max_d：整型变量，表示深度分层的最大值。通常设为256，一般不需要修改。
+ view_num：整型变量，表示参考视角的数量。通常设置为5或者10。
+ max_h/max_w：整型变量，表示降采样后的图片高度/宽度。通常根据原始图片的比例进行缩放，可以填入720/480或800/600等参数。
+ photo_threshold：浮点型变量，表示通过图片一致性对点云进行噪点过滤的设定阈值。对早期的AA-RMVSNet有效，对H-RMVSNet应该不再生效。通常置为0.15。

生成的点云文件名为`point_cloud.ply`。

### 网格生成

i23d系统的网格生成主要基于vis2mesh方法，算法理论参见《Vis2Mesh: Efficient Mesh Reconstruction from Unstructured Point Clouds of Large Scenes with Learned Virtual View Visibility》。代码位于`/dependency/vis2mesh`，其参数主要为输入输出，此处不再赘述。生成的网格文件名为`model_dense_vis2mesh.ply`。

vis2mesh需要对点云进行虚拟拍照，渲染结果保存在`/dependency/dist_i23d/example`文件夹内，debug时可以查看。但每次运行需要主动删除example文件夹内的内容（脚本文件已包含此操作）。

### 网格简化

i23d系统的网格简化方法理论参见姚洋的《城市场景的表面网格重建与优化》，代码与可执行文件位于`/dependency/MeshPolyRefinement`，主要可调参数如下：

+ planar_score：浮点型变量，取值介于0到1之间，代表平面性分数。数值越接近1，简化力度越小。若需要保留小物体的精细结构，可以选择0.96等取值。对于大规模建筑群可以取0.8上下。
+ plane_size：整型变量，取值大于等于1。表示考察的平面包含的三角面片数量的最小值。1表示对每个三角面片都进行考察。通常设置为1。设置的越大则简化力度越小。
+ option：整型变量。表示是否进行简化。若不简化则只做平面性检查与涂色。0表示简化，非0值表示不简化。通常城市场景设置为0，小场景设置为其他值。

生成的简化网格文件名为`model_dense_vis2mesh_simplified.ply`。

### 纹理映射

i23d系统的纹理映射方法理论参见王成的《三维重建网格表面的纹理》。代码与可执行文件位于`/dependency/TexturingPipeline`。本模块依赖nvm文件作为相机信息的输入，而nvm文件只能由simple_redail格式的相机文件转化而来（也可手动将pinhole文件增加值为0的畸变参数强行改为simple_redail）。因此对应的输入图像也得是未经去畸变的图片。主要可调参数如下：

+ sparse_model：布尔型变量，表示输入网格是否为经过简化的网格。对于经过简化的网格，本模块会先进行网格加密，确保每个三角面都能找到对应的照片进行纹理映射。此步骤会显著增加运行时间与存储开销，对于未经简化的稠密网格不需要此步骤。通常城市场景设置为true。
+ texture_quality：浮点型变量，取值介于0到1之间，代表纹理采样时的质量分数。数值越接近1则花费的时间越长，纹理质量越高。通常设置为0.5-0.8之间。


