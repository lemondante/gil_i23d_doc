# 系统版本

系统源码主要包括三个存储位置，分别是gitlab代码仓库（服务器ip:8.140.146.21，苏迈负责管理）、东信云服务器（同步较慢）、服务器本地容器快照。

以托管在gitlab上的源码为基准，目前共有两个不同版本，分别为

+ master分支版本
+ dev分支版本

它们也分别对应**老流程**、**新流程**

老流程指的是在章焕与徐军二人构建的i23d分布式三维重建系统的基础上，集成实验室他人研究成果后的完整重建系统。其对于大部分拍摄质量较高的场景，如建筑、地标、百货等，理论上都能进行高质量的完整重建。对于高难度场景，例如森林、沙漠、水面、雪地等，则有一定的重建困难。可重建范围基本对标ContextCapture。新流程指在此基础上增加图片语义分割、全景图拼接、场景层次化分割的流程，在建筑物较少的场景内有更高的重建效率。

## [master分支版本](http://8.140.146.21/youweikang/dist_i23d/-/tree/main)

master分支版本为稳定可用的i23d分布式三维重建系统，系统结构如下：
![old](../_static/old.png)

## [dev分支版本](http://8.140.146.21/youweikang/dist_i23d/-/tree/dev)

dev分支版本为结合语义的i23d分布式层次化三维重建系统。系统结构如下：
![new](../_static/new.png)



<!-- 
dev分支版本为稳定可用的i23d分布式三维重建系统，变更如下：

+ 集成了D2HC-RMVSNet（可替换原始改造的分布式版openMVS，默认仍然使用openMVS）
+ 增加sampling_based匹配和BA算法（徐军学长的毕业论文）
+ 增加sobel算子的头文件
+ 修改了log日志系统的生成路径

此分支版本额外上传了build文件夹，即已编译后的运行文件。使用时需要删除build文件夹并重新build。

## 90服务器版本

90服务器版本为最新测试版的i23d分布式三维重建系统，同样可用，对入口参数做了优化调整增加鲁棒性。 -->
