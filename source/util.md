# 系统基础组件
## Logging日志库

本系统改造了colmap的logging系统，而colmap的logging是google的glog的封装，支持四种不同级别的日志输出：
+ INFO
+ WARNING
+ ERROR
+ FATAL
colmap封装时修改了出错行为，当错误等级高于指定等级时，不终止应用而是直接返回false,输出“check failure”并继续运行程序。

## test
test