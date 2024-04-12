# Changelog for package performance_test

tros_2.1.0rc1 (2024-04-12)
------------------
1. package.xml中`osrf_testing_tools_cpp`依赖项由test_depend类型变更为exec_depend，解决运行时未安装依赖的问题。

tros_2.1.0 (2024-04-11)
------------------
1. 适配ros2 humble零拷贝。
2. 只编译Array类型消息，降低编译时内存占用。

tros_2.0.1 (2023-07-07)
------------------
1. 修复独立打包导致零拷贝方式运行失败问题


tros_2.0.0 (2023-05-11)
------------------
1. 更新package.xml，支持应用独立打包
