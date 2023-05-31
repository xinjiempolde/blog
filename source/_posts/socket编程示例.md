---
title: socket编程示例
date: 2022-04-04 20:42:43
tags:
    - 计算机网络
    - socket
categories:
    - 计算机网络
---

> 本次使用的环境为Ubuntu 16.04。如果是linux操作系统能够运行本示例代码，如果是Windows操作系统，那么本篇文章不适用。

# 什么是socket

`socket`是应用层和传输层之间的API接口，通过socket为本地进程和远端进程提供通信服务，如TCP、UDP，是面向client-server架构的。需要注意的是，socket也能够实现与网络层的连接，不过用得很少，这里示意图不再标出。

<!--more-->

![img](http://img.singhe.art/FqAL28eXrS7gEZRG9Ej7UxaVu362)

# 相关数据结构和函数

本节介绍相关的数据结构以及相关的函数定义

## sockaddr_in

这里对其结构进行了简化，真实的头文件定义会更冗长一些。

```c
struct sockaddr_in {
    u_char sin_len;             /*地址长度*/
    u_char sin_family;          /*地址簇(TCP/IP: AF_INET)
    u_short sin_port;           /*端口号*/
    struct in_addr sin_addr;    /*IP地址*/
    char sin_zero[8];           /*占位(置0)*/
}
```

## socket

```
sd = socket(int protofamily, int type, int proto);
```

socket函数用于创建套接字，返回值类型为`int`，是socket的编号，或者叫socket描述符(socket descriptor)

- 第一个参数protofamily：代表协议簇。socket被设计用于多个协议，只不过大多数情况下被用于TCP/IP协议(PF_INET)

- 第二个参数type: 代表套接字类型，可选项有：

  ```
  SOCK_STREAM
  ```

  、

  ```
  SOCK_DGRAM
  ```

  和

  ```
  SOCK_RAW
  ```

  。

  - `SOCK_STREAM`：使用TCP套接字
  - `SOCK_DGRAM`：使用UDP套接字
  - `SOCK_RAW`：就是前面提到的与网络层的连接

- 第三个参数proto：0为默认值。

## bind

```c
int bind(int sd, struct sockaddr *localaddr, socklen_t addrlen);
```

bind通常用于服务器端绑定套接字的本地端点地址。

- 第一个参数sd：socket descriptor。
- 第二个参数localaddr：本地端点地址，结构为`*sockaddr`，也就是`*sockaddr_in`，需要类型转换一下。
- 第三个参数addrlen：一般为`sizeof(sockaddr_in)`

## listen

```c
int listen(int sd, int queuesize);
```

将服务器端的套接字置为监听状态，即监听套接字。

- 第一个参数sd：socket descriptor
- 第二个参数，设置请求队列的大小，客户端的连接请求都会在请求队列中进行排队。

## connect

```c
int connect(int sd, struct sockaddr *saddr, socklen_t saddrlen);
```

用于客户端向服务器端请求连接，连接成功返回0，失败返回-1

- 第一个参数sd：socket descriptor
- 第二个参数saddr：同`*sockaddr_in`结构
- 第三个参数saddrlen：一般为`sizeof(sockaddr_in)`

## accept

```c
int accept(int sd, struct sockaddr *addr, socklen_t addrlen);
```

从处于监听状态的套接字sd的客户连接请求队列中取出队首请求，并创建一个新的套接字。这里创建一个新的套接字来处理消息交互，便于进行并发操作，处于监听状态的套接字sd又可以为其他的请求服务，而不会被一直占用。

- 第一个参数sd：socket descr
- 第二个参数addr：同`sockaddr_in`
- 第三个参数addrlen：`sizeof(sockaddr_in)`

## write

```c
long write(int sd, const void* buf, unsigned long n);
```

发送消息，从buf开始的n个字节。若成功，返回写入字节数；若失败，返回-1。

## read

```c
long read(int sd, const void* buf, unsigned long n);
```

读取n字节个消息，写入buf中。若成功，返回读取字节数；若失败，返回-1。

# 示例代码

此节代码来源于https://www.thegeekstuff.com/2011/12/c-socket-programming/以及https://blog.csdn.net/weixin_41249411/article/details/89060985，对其进行了部分修改。

## 总体流程图

![img](http://img.singhe.art/Fjgm57yhgpaF7lqh3qIpQ8xogvqZ)

## 服务器端

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <time.h> 

int main(int argc, char *argv[])
{
    int listenfd = 0, connfd = 0;
    struct sockaddr_in serv_addr; 

    char sendBuff[1025];
	char readBuff[1025] = {0};
    time_t ticks; 

    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    memset(&serv_addr, '0', sizeof(serv_addr));
    memset(sendBuff, '0', sizeof(sendBuff)); 

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(1234); 

    bind(listenfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr)); 

    listen(listenfd, 10); 

    while(1)
    {
        connfd = accept(listenfd, (struct sockaddr*)NULL, NULL); 

        ticks = time(NULL);
		read(connfd, readBuff, sizeof(readBuff)-1);
		printf("received : %s\n", readBuff);

		memset(sendBuff, 0, sizeof(sendBuff));
		strcpy(sendBuff, "hello");
        write(connfd, sendBuff, strlen(sendBuff)); 

        close(connfd);
        sleep(1);
     }
}
```

## 客户端

```c
/*client_tcp.c*/
#include<stdio.h>
#include<string.h>
#include<stdlib.h>
#include<unistd.h>
#include<arpa/inet.h>
#include<sys/socket.h>

int main(){
	//创建套接字
	int sock = socket(AF_INET, SOCK_STREAM, 0);

	//服务器的ip为本地，端口号1234
	struct sockaddr_in serv_addr;
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family = AF_INET;
	inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr);
	serv_addr.sin_port = htons(1234);
	
	//向服务器发送连接请求
	connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
	//发送并接收数据
	char buffer[40];
	printf("Please write:");
	scanf("%s", buffer);
	write(sock, buffer, sizeof(buffer));

	memset(buffer, 0, sizeof(buffer));
	read(sock, buffer, sizeof(buffer) - 1);
	printf("Serve send: %s\n", buffer);

	//断开连接
	close(sock);

	return 0;
}
```



# Java socket代码

## client

```java
import java.io.*;
import java.net.Inet4Address;
import java.net.InetSocketAddress;
import java.net.Socket;
public class Client {
    //java基础类方法的入口
    public static void main(String[] args)throws IOException {
        Socket socket=new Socket();
        //读取流超时的时间设置为3000
        socket.setSoTimeout(3000);
        //连接本地，端口2000；超时时间3000ms
        // Inet4Address.getByName("39.98.27.6")
        socket.connect(new InetSocketAddress(Inet4Address.getLocalHost(), 2000),3000);
        System.out.println("发起服务器连接---------");
        System.out.println("客户端信息："+socket.getLocalAddress()+" P:"+socket.getLocalPort());//打印本地服务器地址和本地端口号
        System.out.println("服务端信息："+socket.getInetAddress()+" P:"+socket.getPort());
        try{
            //发送接收数据
            todo(socket);
        }catch (Exception e){
            System.out.println("出现异常关闭啦");
        }
        //释放资源
        socket.close();
        System.out.println("再见，客户端已退出");
    }
    //发送数据的方法
    private  static void todo(Socket client) throws IOException{
        //构建键盘输入流
        InputStream in=System.in;
        //把键盘输入流转换为BufferedReader
        BufferedReader input=new BufferedReader(new InputStreamReader(in,"UTF-8"));
        //得到Socket输出流（Client要发送出去给服务器的信息），并转换为打印流
        OutputStream outputStream = client.getOutputStream();
        PrintStream socketPrintStream=new PrintStream(outputStream);
        //得到Socket输入流（Server回复传入Client的信息）,并转换为BufferedReader
        InputStream inputStream = client.getInputStream();
        BufferedReader socketBufferedReader=new BufferedReader(new InputStreamReader(inputStream,"UTF-8"));
        //判断Server是否想要退出，回复“bye”时是他想要结束对话
        boolean flag=true;
        do {
            //键盘读取一行
            String str = input.readLine();
            //发送到服务器，（通俗就是显示在输入处，在键盘上输入什么，屏幕显示什么）
            //String str = "003099999920220614100000M1S1C0x0a";
            socketPrintStream.println(str);
            //从服务器读取一行，即Server传入回复给Client的信息
            String echo = socketBufferedReader.readLine();
            if("bye".equalsIgnoreCase(echo)){
                flag=false;
            } else{
                //打印到屏幕上，Server回复什么就显示什么
                System.out.println("客户端回复："+echo);
            }
        }while(flag);
        //资源释放，关闭对于socket资源
        socketPrintStream.close();
        socketBufferedReader.close();
    }
}
```



## server

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintStream;
import java.net.ServerSocket;
import java.net.Socket;
public class Server {
    public static void main(String[] args)throws IOException {
        ServerSocket server=new ServerSocket(2000);
        System.out.println("服务器准备就绪----------");
        System.out.println("服务器信息："+server.getInetAddress()+" P:"+server.getLocalPort());
        //等待多个客户端连接，循环异步线程
        for(;;) {
            //得到客户端
            Socket client = server.accept();
            //客户端构建异步线程
            ClientHandler clientHandler = new ClientHandler(client);
            //启动线程
            clientHandler.start();
        }
    }
    /**
     * 客户端消息处理
     */
    //多个客户端需要做异步操作，建立异步处理类
    private static class ClientHandler extends Thread{//线程
        private  Socket socket;//代表当前的一个连接
        private boolean flag=true;
        ClientHandler(Socket socket){
            this.socket=socket;
        }//构造方法
        //一旦Thead启动起来，就会运行run方法，代表线程启动的部分
        @Override
        public void run(){
            super.run();
            //打印客户端的信息
            System.out.println("新客户端发起连接："+socket.getInetAddress()+" P:"+socket.getPort());
            //在发送过程中会触发一个IO过程，所以需要捕获异常
            try {
                //得到打印流，用于数据输出，服务器回送数据使用，即在屏幕上显示Server要回复Client的信息
                PrintStream socketOutput=new PrintStream(socket.getOutputStream());
                //得到输入流，用于接收数据，得到Client回复服务器的信息
                BufferedReader sockeInput=new BufferedReader(new InputStreamReader(socket.getInputStream(),"UTF-8"));
                do {
                    //客户端回复一条数据
                    String str = sockeInput.readLine();
                    if("bye".equalsIgnoreCase(str)){
                        flag=false;
                        //回送
                        socketOutput.println("bye");
                    }else{
                        //打印到屏幕，并回送数据长度
                        System.out.println(str);
                        socketOutput.println("Server回答说：" +str.length());
                    }
                }while(flag);
                sockeInput.close();
                socketOutput.close();
            }catch (Exception e){
                //触发异常时打印一个异常信息
                System.out.println("连接异常断开！！！");
            }finally {
                //连接关闭
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            System.out.println("再见，客户端退出："+socket.getInetAddress()+" P:"+socket.getPort());
        }
    }
}
```

