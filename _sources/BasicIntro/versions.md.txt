# 系统版本

系统源码目前托管在两处，分别是章焕学长的私有github仓库和王泽斌学长搭建的gitlab仓库。

以托管在gitlab上的源码为基准，目前共有三个不同版本，分别为

+ master分支版本
+ dev分支版本
+ 90服务器版本

## [master分支版本](http://8.140.146.21/youweikang/dist_i23d/-/tree/main)

master分支版本为稳定可用的i23d分布式三维重建系统，与章焕学长的github私有仓库代码一致。版本提交于2020-09-04

## [dev分支版本](http://8.140.146.21/youweikang/dist_i23d/-/tree/dev)

dev分支版本为稳定可用的i23d分布式三维重建系统，变更如下：

+ 集成了D2HC-RMVSNet（可替换原始改造的分布式版openMVS，默认仍然使用openMVS）
+ 增加sampling_based匹配和BA算法（徐军学长的毕业论文）
+ 增加sobel算子的头文件
+ 修改了log日志系统的生成路径

此分支版本额外上传了build文件夹，即已编译后的运行文件。使用时需要删除build文件夹并重新build。

## 90服务器版本

90服务器版本为最新测试版的i23d分布式三维重建系统，同样可用，对入口参数做了优化调整增加鲁棒性。
