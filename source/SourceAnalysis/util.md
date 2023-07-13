# 系统依赖组件

## Logging日志库

本系统的logging系统，是google的glog的封装，支持四种不同级别的日志输出：
+ INFO
+ WARNING
+ ERROR
+ FATAL
修改了出错行为，当错误等级高于指定等级时，不终止应用而是直接返回false,输出“check failure”并继续运行程序。

## C++ boost库

boost库中的program_options主要用于命令行参数的读取。构造一个vector名为`commands`，接受命令行参数字符串并运行对应的函数。

## sqlite数据库

SQLite是一个进程内的库，实现了自给自足的、无服务器的、零配置的、事务性的 SQL 数据库引擎。它是一个零配置的数据库，这意味着与其他数据库不一样，不需要在系统中配置。
就像其他数据库，SQLite 引擎不是一个独立的进程，可以按应用程序需求进行静态或动态连接。SQLite 直接访问其存储文件。

特点：

+ 不需要一个单独的服务器进程或操作的系统（无服务器的）。
+ SQLite 不需要配置，这意味着不需要安装或管理。
+ 一个完整的 SQLite 数据库是存储在一个单一的跨平台的磁盘文件。
+ SQLite 是非常小的，是轻量级的，完全配置时小于 400KiB，省略可选功能配置时小于250KiB。
+ SQLite 是自给自足的，这意味着不需要任何外部的依赖。
+ SQLite 事务是完全兼容 ACID 的，允许从多个进程或线程安全访问。
+ SQLite 支持 SQL92（SQL2）标准的大多数查询语言的功能。
+ SQLite 使用 ANSI-C 编写的，并提供了简单和易于使用的 API。
+ SQLite 可在 UNIX（Linux, Mac OS-X, Android, iOS）和 Windows（Win32, WinCE, WinRT）中运行。

本系统包装了`LoadData`方法，用于从图片集中读取图片并存入数据库。


## rpclib

rpclib是一个开源的c++RPC（Remote Procedure Call）库。目的是在master机上可以调用worker机的函数。它构建了一个典型的C/S模型，本系统中worker作为service提供响应，master作为client负责调用worker。它在worker进程中构建service对象，并bind一个字符串作为调用记号。然后在master进程中构建clinet对象，用call+字符串的方法调用worker进程中对应的函数。

## ceres-solver运算库

google开源的c++运算库，用于求解优化问题。

## PBA（Parallel BA）库

并行BA（捆绑调整）库。论文参见[*Multicore Bundle Adjustment*](https://ieeexplore.ieee.org/abstract/document/5995552)