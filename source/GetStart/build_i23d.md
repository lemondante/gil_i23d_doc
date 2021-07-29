# 系统构建

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

