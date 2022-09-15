---
title: std::mutex
date: 2022-05-31 18:41:25
tags:
    - zmq
    - protobuf
categories:
    - tutorial
---

# zmq + protobuf安装教程

## 环境

- centos 7.6

<!--more-->

## 环境预安装

```bash
sudo yum install -y wget gcc-c++ automake autoconf libtool make lsof vim nano boost git
sudo yum -y install openssl-devel bzip2-devel expat-devel gdbm-devel curl-devel boost-devel readline-devel sqlite-devel kernel-devel
```



## 安装libsodium

```bash
wget https://download.libsodium.org/libsodium/releases/libsodium-1.0.18-stable.tar.gz --no-check-certificate
tar -zxvf libsodium-1.0.18-stable.tar.gz
cd libsodium-stable/
./configure
make
make check
sudo make install
sudo ldconfig
```



## 安装libzmq

```bash
wget https://github.com/zeromq/libzmq/releases/download/v4.3.4/zeromq-4.3.4.tar.gz
tar -zxvf zeromq-4.3.4.tar.gz
cd zeromq-4.3.4
./configure --with-libsodium=/usr/local
make
sudo make install
```



## 安装zmqpp

```bash
git clone https://github.com/zeromq/zmqpp.git
cd zmqpp
make
sudo make install
```



## 测试zmq

***server.cc***

```c++
//  Hello World server

#include <zmqpp/zmqpp.hpp>
#include <string>
#include <iostream>
#include <chrono>
#include <thread>

using namespace std;

int main(int argc, char *argv[]) {
  const string endpoint = "tcp://*:5555";

  // initialize the 0MQ context
  zmqpp::context context;

  // generate a pull socket
  zmqpp::socket_type type = zmqpp::socket_type::reply;
  zmqpp::socket socket (context, type);

  // bind to the socket
  socket.bind(endpoint);
  while (1) {
    // receive the message
    zmqpp::message message;
    // decompose the message 
    socket.receive(message);
    string text;
    message >> text;

    //Do some 'work'
    std::this_thread::sleep_for(std::chrono::seconds(1));
    cout << "Received Hello" << endl;
    socket.send("World");
  }

}
```

编译：

```bash
g++ server.cc -o server.app -std=c++11 -lzmq -lzmqpp
```



***client.cc***

```c++
//  Hello World client
#include <zmqpp/zmqpp.hpp>
#include <string>
#include <iostream>

using namespace std;

int main(int argc, char *argv[]) {
  const string endpoint = "tcp://localhost:5555";

  // initialize the 0MQ context
  zmqpp::context context;

  // generate a push socket
  zmqpp::socket_type type = zmqpp::socket_type::req;
  zmqpp::socket socket (context, type);

  // open the connection
  cout << "Connecting to hello world server…" << endl;
  socket.connect(endpoint);
  int request_nbr;
  for (request_nbr = 0; request_nbr != 10; request_nbr++) {
    // send a message
    cout << "Sending Hello " << request_nbr <<"…" << endl;
    zmqpp::message message;
    // compose a message from a string and a number
    message << "Hello";
    socket.send(message);
    string buffer;
    socket.receive(buffer);
    
    cout << "Received World " << request_nbr << endl;
  }
}
```

编译：

```bash
g++ client.cc -o client.app -std=c++11 -lzmq -lzmqpp
```



## 安装protobuf3

```bash
wget https://github.com/protocolbuffers/protobuf/releases/download/v21.5/protobuf-cpp-3.21.5.tar.gz
tar -zxvf protobuf-cpp-3.21.5.tar.gz
cd protobuf-3.21.5/
./configure -prefix=/usr/local/
make
sudo make install
```



## protobuf3例子

> 内容转载自[https://www.jianshu.com/p/d29913998976](https://www.jianshu.com/p/d29913998976)

创建game.proto

```protobuf
syntax = "proto3";
package pt;

message req_login
{
    string username = 1;
    string password = 2;
}

message obj_user_info
{
    string nickname    = 1;    //昵称
    string icon        = 2;    //头像
    int64  coin        = 3;    //金币
    string location    = 4;    //所属地
}

//游戏数据统计
message obj_user_game_record
{
    string time = 1;
    int32 kill  = 2;        //击杀数
    int32 dead  = 3;        //死亡数
    int32 assist= 4;        //助攻数
}

message rsp_login
{
    enum RET {
        SUCCESS         = 0;
        ACCOUNT_NULL    = 1;    //账号不存在
        ACCOUNT_LOCK    = 2;    //账号锁定
        PASSWORD_ERROR  = 3;    //密码错误
        ERROR           = 10;
    }
    int32 ret = 1;
    obj_user_info user_info = 2;
    repeated obj_user_game_record record = 3;
}
```

生成`.h`和`.cc`文件：

```bash
protoc ./game.proto --cpp_out=./
```

创建main.cpp

```c++
#include <iostream>
#include <string>
#include "game.pb.h"

int main()
{
    pt::rsp_login rsp{};
    rsp.set_ret(pt::rsp_login_RET_SUCCESS);
    auto user_info = rsp.mutable_user_info();
    user_info->set_nickname("dsw");
    user_info->set_icon("345DS55GF34D774S");
    user_info->set_coin(2000);
    user_info->set_location("zh");

    for (int i = 0; i < 5; i++) {
        auto record = rsp.add_record();
        record->set_time("2017/4/13 12:22:11");
        record->set_kill(i * 4);
        record->set_dead(i * 2);
        record->set_assist(i * 5);
    }

    std::string buff{};
    rsp.SerializeToString(&buff);
    //------------------解析----------------------
    pt::rsp_login rsp2{};
    if (!rsp2.ParseFromString(buff)) {
        std::cout << "parse error\n";
    }
    
    auto temp_user_info = rsp2.user_info();
    std::cout << "nickname:" << temp_user_info.nickname() << std::endl;
    std::cout << "coin:" << temp_user_info.coin() << std::endl;
    for (int m = 0; m < rsp2.record_size(); m++) {
        auto temp_record = rsp2.record(m);
        std::cout << "time:" << temp_record.time() << " kill:" << temp_record.kill() << " dead:" << temp_record.dead() << " assist:" << temp_record.assist() << std::endl;
    }
}
```

编译，运行：

```bash
g++ -std=c++11 main.cpp game.pb.cc `pkg-config --cflags --libs protobuf` -lpthread -lprotobuf
./a.out
```