---
title: opengauss中可能用到的命令
date: 2022-04-18 10:28:40
tags:
    - openGauss
    - 数据库
categories:
    - openGauss
---

# 0x00 编译protobuf

```shell
source ~/gauss_env
/home/singheart/binarylibs/dependency/centos7.6_x86_64/protobuf/comm/bin/protoc --cpp_out=. serialized_tuple.proto
/home/singheart/binarylibs/dependency/centos7.6_x86_64/protobuf/comm/bin/protoc --cpp_out=. counter.proto
cd /home/singheart/GeoGauss/src/gausskernel/storage/access/heap/
protoc --cpp_out=. counter.proto
protoc --cpp_out=. serialized_tuple.proto
cp /home/singheart/GeoGauss/src/gausskernel/storage/access/heap/counter.pb.h /home/singheart/GeoGauss/src/include/access/
g++ -std=c++14 data.pb.cc prototest.cc -I/home/singheart/binarylibs/dependency/centos7.6_x86_64/protobuf/comm/include -L/home/singheart/binarylibs/dependency/centos7.6_x86_64/protobuf/comm/lib -lpthread -lprotobuf -D_GLIBCXX_USE_CXX11_ABI=0
```
<!--more-->

# 0x01 YCSB测试

## 对openGauss进行初始化
```shell
gsql -d postgres -p 22222
alter role singheart password 'Test@123';
create user jack password 'Test@123';
grant usage on foreign server mot_server to jack;
GRANT ALL PRIVILEGES ON DATABASE postgres to jack;
ALTER ROLE jack CREATEDB;
GRANT ALL PRIVILEGES To jack;

gsql -d postgres -p 22222 -U jack -W Test@123
DROP foreign table if exists usertable;
CREATE  Foreign TABLE usertable (YCSB_KEY VARCHAR(250) PRIMARY KEY,FIELD0 VARCHAR(250), FIELD1 VARCHAR(250),FIELD2 VARCHAR(250), FIELD3 VARCHAR(250),FIELD4 VARCHAR(250), FIELD5 VARCHAR(250),FIELD6 VARCHAR(250), FIELD7 VARCHAR(250),FIELD8 VARCHAR(250), FIELD9 VARCHAR(250));
\q
```

## YCSB加载数据和运行
```shell
~/ycsb-0.17.0/bin/ycsb load jdbc -s -P workloads/workloadb -P jdbc-binding/conf/db.properties -cp jdbc-binding/lib/postgresql.jar -threads 1
~/ycsb-0.17.0/bin/ycsb run jdbc -s -P workloads/workloadb -P jdbc-binding/conf/db.properties -cp jdbc-binding/lib/postgresql.jar -threads 1


```

# 0x02 安装brpc

项目中使用到了`brpc`，对于服务器centos7.6(同时还得使用binarylibs中自带的gcc7.3.0)参考`brpc`的安装教程：[开始 | bRPC (apache.org)](https://brpc.apache.org/zh/docs/getting_started/#fedoracentos)

> 现在我终于明白了，centos7.6使用yum安装的库都是不带__cxx11的(默认gcc版本为4.8.5)，而binarylibs中的gcc7.3.3默认开启了），我已必定用不了使用yum安装的库，除非所有使用gcc7.3.3的时候吧abicxx11给关了。。。

## 2.1 依赖准备

CentOS一般需要安装EPEL，否则很多包都默认不可用。

```shell
sudo yum install epel-release
```

安装依赖：

```shell
sudo yum install git gcc-c++ make openssl-devel
# sudo yum install protobuf-devel protobuf-compiler
# sudo yum install gflags-devel
# sudo yum install leveldb-devel
```

如果你要在样例中启用cpu/heap的profiler：

```shell
sudo yum install gperftools-devel
```

如果你要运行测试，那么要安装ligtest-dev:

```shell
sudo yum install gtest-devel
```

## 2.2 安装protobuf 3.0

需要注意的是，centos7.6版本太老了，使用yum安装的protobuf版本为2.5，而brpc引入了protobuf3.0的语法，因此protobuf需要自己安装，而不能使用系统安装的protobuf。

```shell
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.19.5/protobuf-all-3.19.5.tar.gz
tar -zxvf protobuf-all-3.19.5.tar.gz
cd protobuf-all-3.19.5
./configure CXXFLAGS="-D_GLIBCXX_USE_CXX11_ABI=0" --prefix=~/local
make -sj && sudo make install -sj
```

## 2.3 安装gflags

使用yum安装的gflags在编译brpc的时候出错了：

```text
/home/singheart/cpl/brpc/src/bvar/variable.cpp:743: undefined reference to `gflags::GetCommandLineOption(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)'
/opt/rh/devtoolset-11/root/usr/bin/ld: /home/singheart/cpl/brpc/src/bvar/variable.cpp:747: undefined reference to `gflags::GetCommandLineOption(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)'
/opt/rh/devtoolset-11/root/usr/bin/ld: /home/singheart/cpl/brpc/src/bvar/variable.cpp:752: undefined reference to `gflags::GetCommandLineOption(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)'
/opt/rh/devtoolset-11/root/usr/bin/ld: /home/singheart/cpl/brpc/src/bvar/variable.cpp:757: undefined reference to `gflags::GetCommandLineOption(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)'
/opt/rh/devtoolset-11/root/usr/bin/ld: /home/singheart/cpl/brpc/src/bvar/variable.cpp:761: undefined reference to `gflags::GetCommandLineOption(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)'
/opt/rh/devtoolset-11/root/usr/bin/ld: libbrpc.a(variable.o):/home/singheart/cpl/brpc/src/bvar/variable.cpp:767: more undefined references to `gflags::GetCommandLineOption(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)' follow
/opt/rh/devtoolset-11/root/usr/bin/ld: libbrpc.a(variable.o): in function `_GLOBAL__sub_I_variable.cpp':
/home/singheart/cpl/brpc/src/bvar/variable.cpp:912: undefined reference to `gflags::RegisterFlagValidator(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const*, bool (*)(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&))'
/opt/rh/devtoolset-11/root/usr/bin/ld: libbrpc.a(variable.o): in function `__static_initialization_and_destruction_0':
/home/singheart/cpl/brpc/src/bvar/variable.cpp:914: undefined reference to `gflags::RegisterFlagValidator(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const*, bool (*)(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&))'
/opt/rh/devtoolset-11/root/usr/bin/ld: /home/singheart/cpl/brpc/src/bvar/variable.cpp:916: undefined reference to `gflags::RegisterFlagValidator(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const*, bool (*)(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&))'
/opt/rh/devtoolset-11/root/usr/bin/ld: /home/singheart/cpl/brpc/src/bvar/variable.cpp:918: undefined reference to `gflags::RegisterFlagValidator(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const*, bool (*)(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&))'
/opt/rh/devtoolset-11/root/usr/bin/ld: /home/singheart/cpl/brpc/src/bvar/variable.cpp:920: undefined reference to `gflags::RegisterFlagValidator(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const*, bool (*)(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&))'
/opt/rh/devtoolset-11/root/usr/bin/ld: libbrpc.a(variable.o):/home/singheart/cpl/brpc/src/bvar/variable.cpp:925: more undefined references to `gflags::RegisterFlagValidator(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const*, bool (*)(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&))' follow
/opt/rh/devtoolset-11/root/usr/bin/ld: libbrpc.a(gflag.o): in function `bvar::GFlag::get_value[abi:cxx11]() const':
/home/singheart/cpl/brpc/src/bvar/gflag.cpp:81: undefined reference to `gflags::GetCommandLineOption(char const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >*)'
/opt/rh/devtoolset-11/root/usr/bin/ld: libbrpc.a(gflag.o): in function `bvar::GFlag::set_value(char const*)':

```

手动安装gflags：

```shell
wget https://github.com/gflags/gflags/archive/refs/tags/v2.2.2.tar.gz
tar -zxvf v2.2.2.tar.gz
mv v2.2.2.tar.gz  gflagsv2.2.2.tar.gz
cd gflags-2.2.2/
mkdir build && cd build
cmake .. -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=0" -DCMAKE_INSTALL_PREFIX=~/local
make -sj && sudo make install -sj

# 查看安装的flags的符号
nm /usr/local/lib/libgflags.a | grep FlagRegisterer | c++filt
```

需要注意的是，需要把原来使用yum安装的gflags卸载掉，否则链接的时候如果优先链接到此库，也会导致符号未定义的问题。

## 2.4 编译安装leveldb

```shell
mv 1.23.tar.gz leveldb.1.23.tar.gz
tar -zxvf leveldb.1.23.tar.gz
cd leveldb-1.23
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_POSITION_INDEPENDENT_CODE=ON -DCMAKE_CXX_FLAGS="-D_GLIBCXX_USE_CXX11_ABI=0" -DCMAKE_INSTALL_PREFIX=~/local .. && cmake --build .
make -sj && sudo make install -sj
```



## 2.5 使用config_brpc.sh编译brpc

git克隆brpc，进入项目目录然后执行：

```shell
git clone https://github.com/apache/brpc.git
sh config_brpc.sh --headers="/home/singheart/local/include /usr/include /usr/local/include" --libs="/home/singheart/local/lib /home/singheart/local/lib64 /usr/lib64 /usr/bin /usr/local/lib /usr/local/lib64 /usr/local/bin"
# 运行完后生成Makefile文件，需要在CXXFLAGS中添加-D_GLIBCXX_USE_CXX11_ABI=0
make -sj
```

修改编译器为clang，添加选项`--cxx=clang++ --cc=clang`。

不想链接调试符号，添加选项`--nodebugsymbols` 然后编译将会得到更轻量的二进制文件。

想要让brpc使用glog，添加选项：`--with-glog`。

要启用 [thrift 支持](https://brpc.apache.org/zh/docs/server/serve-thrift/)，先安装thrift，然后添加选项：`--with-thrift`。

**运行样例**

```shell
$ cd example/echo_c++
$ make
$ ./echo_server &
$ ./echo_client
```

上述操作会链接brpc的静态库到样例中，如果你想链接brpc的共享库，请依次执行：`make clean`和`LINK_SO=1 make`

**运行测试**

```shell
$ cd test
$ make
$ sh run_tests.sh
```

# 0x03 插入数据

```sql
insert into stu values(40, 'make');
insert into stu values(41, 'server');
insert into stu values(42, 'client');
insert into stu values(43, 'static');

update stu set name = 'xoxi' where id = 43;
```

