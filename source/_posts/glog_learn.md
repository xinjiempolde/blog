# glog

glog是谷歌开源的C++日志库，这里对glog进行一个简单的学习记录。



<!--more-->c

# 在CMake中添加glog

CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(glog_learn VERSION 1.0)

set(CMAKE_CXX_STANDARD 14)

add_executable(glog_main main.cpp)

# 使用FetchContent添加github外部库
include(FetchContent)

FetchContent_Declare(glog
        GIT_REPOSITORY https://github.com/google/glog
        GIT_TAG v0.6.0)
FetchContent_MakeAvailable(glog)

# 添加外部库
target_link_libraries(glog_main glog::glog)
```



main.cpp

```c++
#include <iostream>
#include <glog/logging.h>

int main() {
  FLAGS_log_dir = "~/project/cpp_project/glog_learn/build/log";
  google::InitGoogleLogging("glog_learn");
  LOG(INFO) << "Starting";
  printf("hello world\n");
  return 0;
}
```



更多详细信息见[官网](https://github.com/google/glog)

# 注意事项

不能在conda环境中编译，否则会报错：`/usr/bin/ld: CMakeFiles/cleanup_with_absolute_prefix_unittest.dir/src/cleanup_with_absolute_prefix_unittest.cc.oo:(.data.rel.ro._ZTI47CleanImmediatelyWithAbsolutePrefix_logging_Test[_ZTI47CleanImmediatelyWithAbsolutePrefix_logging_Test]+0x10): undefined reference totypeinfo for testing::Test`。

退出conda环境:

```shell
conda deactivate
```

