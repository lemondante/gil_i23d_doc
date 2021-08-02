# 系统依赖组件

## Logging日志库

本系统的logging系统，是google的glog的封装，支持四种不同级别的日志输出：
+ INFO
+ WARNING
+ ERROR
+ FATAL
修改了出错行为，当错误等级高于指定等级时，不终止应用而是直接返回false,输出“check failure”并继续运行程序。

## C++ boost库

## sqlite数据库

## rpclib

## ceres-solver运算库

test
