---
title: linux+windows网编
date: 2023-07-23
categories: 
  - 编程
author: oxygen
---

# chap01 计算机网络基础概念

为什么要有计算机网络,为什么要联网呢,**主要是共享信息,使用远程资源,分布式处理(把数据处理分在服务器多台机器,比如云计算);**

## TCP/IP 协议族

为什么有这种协议,其实就是就在再接入网络的这两台计算机,数据必须遵循某种格式,约定,语言,规则被称为==**协议**==就是为了让这**两台节点成功地通信**

**而在internet中,最常见的 网络协议就是TCP/IP协议;**,而TCP/IP协议是一个**协议族**,他其实包含了很多子协议;

- IP协议(网络层开发)
- UDP TCP(传输层开发)
- 很多,http,FTP...DNS(域名服务)(应用层开发的)

### 网络分层模型

- OSI七层模型

![image-20230719192322645](./assets/image-20230719192322645.png)

**对于发送,是从上往下的,而对于接收方,则是从下往上;**对于OSI七层模型,每一层都有不同作用;**(下面的了解一下就行,目前用的最多的是TCP/IP四层模型)**

1. **物理层**：这一层负责实际的硬件连接和电信号的传输。这包括电缆、卡、光纤等物理设备以及它们如何将数据转换为电信号。
2. **数据链路层**：这一层负责在物理层上建立一个可靠的链接，它提供了错误检测和修复的功能，确保数据的正确传输。这一层还负责控制物理层如何使用网络设备和介质来发送和接收数据。
3. **网络层**：这一层负责数据包的发送和路由选择，包括 IP 地址处理和路由选择。如果数据包需要通过多个网络（例如从一个城市发送到另一个城市），网络层决定数据包的路径。
4. **传输层**：这一层负责端到端的通信。这包括确定数据包的顺序以及数据传输的速率控制。**常见的协议有 TCP 和 UDP。**
5. **会话层**：这一层负责在网络中的设备之间建立、管理和终止会话。会话层使不同的应用程序可以在同一时间进行多个会话。
6. **表示层**：这一层负责将数据转换为可以理解的格式。包括数据的加密、解密、压缩、解压缩等。
7. **应用层**：这是最接近用户的层级，负责处理特定的应用程序细节。例如，当你在浏览器中访问一个网页时，**HTTP 协议**就在应用层工作。

- TCP/IP四层模型

![image-20230719193103744](./assets/image-20230719193103744.png)

==**他和OSI七层模型思想差不多,只不过是简化了;**==

目前用的多的就是TCP/IP的四层模型;OSI可以了解下;

**网络编程的开发,就是针对每一层的**,每一层都有不同的协议,但是他们都属于`TCP/IP`协议;

![image-20230719193324760](./assets/image-20230719193324760.png)

就是比如在最上层你要传输接收数据,要按照Telent,http,FTP,DNS这种协议进行;

### 数据封装

我们知道,数据在网络的不同分层传输,会经过不同地协议,在动态地经过这么多层,会根据协议来进行组织数据;

![image-20230719195224436](./assets/image-20230719195224436.png)

数据包分为**头部**和**体部**,比如在应用层,打开一个网页,数据是从某个服务器返回来的;在应用层浏览器的数据都是http协议的;在应用层就会封装成HTTP头部+数据,然后传输层,加上TCP/UDP协议头部->网络层加IP头部..以此类推; 

在封装完之后,就会发送到对方节点,**然后一层层地解封**,最终到用户的应用层,把这个表格显示出来;

所以对于我们应用层来说,只需要接受和发送最原始的数据就行了;

### TCP/IP常用协议

- IP协议

就是**网络层的协议,**但是他是不可靠的,体现在不向发送者或者接收者包的状态,但是它可以路由到最优路径;

![image-20230719195935133](./assets/image-20230719195935133.png)

### Mac地址

**在网络层,有一种协议叫做ARP,可以将ip地址->MAC地址;mac地址这才是唯一的,ip不是唯一的！**

就是标识**网卡**的**唯一ID**,有6个字节,理论上所有的网卡的这个MAC地址都是不一样的;

![image-20230719180504729](./assets/image-20230719180504729.png)

mac地址主要是标识主机的id,这个id是一个**物理地址**;ip是随时变动的(比如换地址上午),但是网卡不是;必须有这个id,网络的数据包才会到达主机;笔记本一般有两个网卡,有线和无线;

### ip地址

**我们要把数据发送到目的地,肯定是需要IP地址的**,用于标识唯一地址的;IP协议就要求建立TCP/IP链接的时候,每台主机都得分配32位的地址(ipv6 128位),Ip地址实际上是分配给**网卡的**;

因为ipv4协议32位,而世界上网关远远大于2^32^,所以其实是不能唯一标识的,那怎么办呢,一般采取

- NAT

路由器使用一种叫做网络地址转换（NAT）的技术，使得所有连接到它的设备可以共享一个**公网IP地址**。每个设备有其自己的**私有IP地址**，但在与Internet通信时，他们的地址会被**转换成路由器的公网IP地址**。

- IPV6

为了解决IPv4地址耗尽的问题，已经开发出了新的IP地址版本IPv6，其地址数量远超IPv4，可以为每个Internet设备分配一个唯一的全球IP地址。



通过IP协议就可以把IP地址->MAC地址(唯一标识);**可以这样理解**,就是全世界的ip地址是被分配了,这个ip地址是真实的,叫做**公网ip**,很多人可能用的都是同一个公网ip**(路由器是一样的话**),而在这个环境内,分配给你的ip地址,比如`192.168.1.1`则是虚拟的,每一个链接在路由器上面的东西都是有自己的虚拟**ip**

这也就分出了==公有ip==和==私有ip==的概念;

![image-20230720085720893](./assets/image-20230720085720893.png)

![image-20230719200345493](./assets/image-20230719200345493.png)

**用的比较多的是C的,**

![image-20230719200442199](./assets/image-20230719200442199.png)

一个ip可以分为子网ID和主机ID,这两者要和子网掩码一起来看;

比如一个ip:192.168.11.23 子网掩码是255.255.255.0, 

> **子网id  ip中被子网掩码中1连续覆盖的位**
>
> **主机id  ip中被子网掩码中0连续覆盖的位**

![image-20230719181434393](./assets/image-20230719181434393.png)

上面的例子我们可以看到,一个ip地址,他是**192.168.1.2**,而**子网掩码是255.255.255.0**

**可以得到他的子网id是168.192.1,而主机id就是2**,**而网段地址是192.168.1.0**,**广播地址是192.168.0.255;**

电脑是不能设置网段地址,广播地址任何一个的;也就是在一个**网段（192.168.1.0）**,==也就是192.168.1.0一共有256个,但是只有254个可以分配给用户==;

再比如,一个ip地址是`192.168.1.2`,而子网掩码是`255.255.0.0`,那么子网id就是`192.168`,公网id是`1.2`,网段是`192.168.0.0`，在这个局域网或者说是网段内最多可以容纳2^16^-2个主机;

![image-20230719182540146](./assets/image-20230719182540146.png)

### 端口

前面可以通过ip定位到一个唯一的网卡,或者服务器,但是我们如何定位这个服务呢?

这个就是用到了**端口**,他是软件层面的,比如数据库就那些服务,他就绑定1521,3301端口;**或者ssh**,他的端口就是22

![image-20230720090531963](./assets/image-20230720090531963.png)

使用`sudo more /etc/services`查看端口占用情况

![image-20230720090834526](./assets/image-20230720090834526.png)

**可以FTP21,SSH22,Telnet 23....**

### TCP/UDP协议

这两个协议也是TCP/IP协议里面的东西,他是传输层的协议.

- TCP协议

他是一个**可靠的**,**面向链接**的**字节流服务**,所以首先两个节点要连接;它主要用于和网络上的节点进行数据交换;

而且他还有crc算法进行完整性查验;

![image-20230720091656006](./assets/image-20230720091656006.png)

- UDP协议

UDP协议也是TCP/IP协议的传输层的,**他是无连接不可靠的传输服务;**也就是通信双方不需要建立连接;

他比TCP更快,如果发送丢包啥的,也不会向发送方报告;也不提供发包的顺序;

![image-20230720092458436](./assets/image-20230720092458436.png)

### 路由器协议

就是网络层的协议,比如IP协议,ICMP协议,RIP,DNS协议(从机器的名字确定机器的数字地址)等

他就是确定最优传输路径的;

### ping和本地回环

win和linux都适用,用来测试两台主机的网络连通性;

私有ip 127.0.0.1代表你本机的ip,你使用这个ip,最后都会回到你自己的ip,这个一般用于测试自己的网络协议栈是否是好使的;

# chap02 TCP编程

Socket就是套接字,他本身就是一种通讯机制,它包含一套调用接口和数据结构定义;

**他给应用程序使用TCP/UDP那些协议进行网络通信的手段**;他们本身上是一种特殊的IO;

他还是通过read/write这些系统调用;他就提供**对应的文件描述符**

一个socket都包含**协议,本地地址,本地端口,远程地址(ip),远程端口**,这个叫做socket五元组;

## 创建socket

```C++
extern int socket (int __domain, int __type, int __protocol) __THROW;
```

- `int __domain`：这个参数指定了**所使用的协议族**（Protocol Family）。常用的值有 **`AF_INET`**（IPv4 网络协议）、**`AF_INET6`**（IPv6 网络协议）、`AF_UNIX`（UNIX 文件系统套接字）等。
- `int __type`：这个参数指定了**套接字的类型**。常用的值有 `SOCK_STREAM`（**提供序列化的、可靠的、双向的、基于连接的字节流。这种类型的套接字使用 TCP 协议进行网络通信**）、`SOCK_DGRAM`（**提供无序的、不可靠的、双向的、基于消息的连接。这种类型的套接字使用 UDP 协议进行网络通信**）等。
- `int __protocol`：这个参数指定了套接字所使用的协议。如果这个参数为 0，那么会自动选择 `__type` 参数对应的默认协议。

这个函数会返回一个整数，这个整数是这个新创建的套接字的文件描述符。如果创建套接字失败，那么这个函数会返回 -1，并设置相应的错误码。

执行完socket之后,会进行下面的流程

![image-20230720100133722](./assets/image-20230720100133722.png)

## 字节序

就是大端字节序和小端字节序,就是比如32位数据

0x01020304,如果是**小端存储**,那么字节序是

地址从低到高->0x04 0x03 0x02 0x01

如果是大端存储,那么字节序是

地址从低到高->0x01,0x02,0x3,0x04;最高位反而在内存地址最高处;

网络使用的字节序叫做**网络字节序**也就是大端字节序,但是一般AMD INTEL架构都是小端字节序;因此要进行一些字节序的转换函数

### 字节序转换函数

1. `htons()`：将短整数从主机字节序转换到网络字节序。
2. `htonl()`：将长整数从主机字节序转换到网络字节序。
3. `ntohs()`：将短整数从网络字节序转换到主机字节序。
4. `ntohl()`：将长整数从网络字节序转换到主机字节序。

h就是hots,主机,而n就是网络,其中to就是到,s和l就是s是short,而l就是long;

## socket相关结构

### sockaddr

这个是同样的socket地址,哪个协议都可以用

```C++
/* Structure describing a generic socket address.  */
struct sockaddr
  {
    __SOCKADDR_COMMON (sa_);	/* Common data: address family and length.  */
    char sa_data[14];		/* Address data.  */
  };
```

- `__SOCKADDR_COMMON (sa_)`：这是一个宏，用**于定义地址家族和地址长度**。地址家族决定了将要使用的协议（如 IPv4 或 IPv6），地址长度则表示地址的长度。对于 `sockaddr` 结构体，这部分通常包括一个 `sa_family_t` 类型的 `sa_family` 字段和一个 `sa_len` 字段（在某些系统中）。
- `char sa_data[14]`：这是一个字符数组，用于存储实际的地址信息和端口号。对于 `sockaddr` 结构体，我们通常不直接访问这个字段，而是通过特定版本的 `sockaddr` 结构体（如 `sockaddr_in` 或 `sockaddr_in6`）来访问。

这个主要一般不用,而是用于某些系统调用强转传进去;

### sockaddr_in

```C++
struct sockaddr_in {
    short            sin_family;   // e.g. AF_INET, AF_INET6
    unsigned short   sin_port;     // e.g. htons(3490)
    struct in_addr   sin_addr;     // see struct in_addr, below
    char             sin_zero[8];  // zero this if you want to
};

```

- `sin_family`：这是一个整数，用于指定地址家族。对于 IPv4，**这个字段应设置为 `AF_INET`。**
- `sin_port`：这是一个整数，用于指定端口号。注意，**端口号应该是网络字节序**，所以在设置这个字段时，通常会使用 `htons` 函数进行转换。
- `sin_addr`：这是一个 `struct in_addr` 结构体，**用于指定 IPv4 地址**。`struct in_addr` 结构体只有一个字段 `s_addr`，**它是一个无符号长整数，用于保存 IPv4 地址（==以网络字节序形式==）。**
- `sin_zero`：这是一个 8 字节长的字符数组，没有实际的用途，只是用于填充 `sockaddr_in` 结构体，使其大小与 `sockaddr` 结构体相同。在设置 `sockaddr_in` 结构体时，**通常将这个字段设置为零**。

在实际使用中，我们通常会创建一个 `sockaddr_in` 结构体，设置其 `sin_family`、`sin_port` 和 `sin_addr` 字段，然后将其强制转换为==**`sockaddr`** 结构==并传递给需要套接字地址的函数（如 `bind` 或 `connect`）。

### inet_ntop与inet_pton

**他们定义在<arp/inet.h>**

```C++
extern int inet_pton (int __af, const char *__restrict __cp,
		      void *__restrict __buf) __THROW;

/* Convert a Internet address in binary network format for interface
   type AF in buffer starting at CP to presentation form and place
   result in buffer of length LEN astarting at BUF.  */
extern const char *inet_ntop (int __af, const void *__restrict __cp,
			      char *__restrict __buf, socklen_t __len)
     __THROW;
```



- `inet_ntop`（network to presentation）：这个函数用于将网络格式的 IP 地址转换为展示格式。它接受一个网络格式的 IP 地址（`in_addr` 结构体或 `in6_addr` 结构体的指针），并将其转换为一个字符串。
- `inet_pton`（presentation to network）：这个函数用于将展示格式的 IP 地址转换为网络格式。它接受一个字符串（包含一个 IPv4 或 IPv6 地址），并将其转换为一个 `in_addr` 结构体或 `in6_addr` 结构体。

af就是地址组,比如AF_INET,addr就是32位地址,str就是点分十进制指针,size就是字符串大小;

```C++
#include <sys/socket.h>
#include <arpa/inet.h>
#include <iostream>
#include <string>
#include <string.h>
//int socket(int domain,int type,int protocol);
int main(){
    
    //inet_ntop
    const char* ipPoint="192.168.1.1";
    auto ip=0l;
    auto ret=inet_pton(AF_INET,ipPoint,&ip);
    if(ret==-1){

        perror("fuck");
        return -1;
    }

    std::cout<<"ip :"<<ip<<std::endl;
    
    char ipT[INET_ADDRSTRLEN]={0};
    auto p=inet_ntop(AF_INET,&ip,ipT,sizeof (ipT));
    if(p==nullptr){

        perror("fuck");
        return -1;
    }
    std::cout<<"trans ip :"<<ipT<<std::endl;

    return 0;
}
```

用法如下

![image-20230720103342026](./assets/image-20230720103342026.png)

## socket编程模型

![image-20230720105736469](./assets/image-20230720105736469.png)

服务器端和客户机是不太一样的;而需要对应的函数就是上面那些;

### 服务器编程

```C++
#include <iostream>
#include <netdb.h>
#include <sys/socket.h>
#include <unistd.h>
#include <string.h>
#include <string>
#include <memory.h>
#include <signal.h>
#include <time.h>
#include <arpa/inet.h>

int g_sockfd;//套接字文件描述符
void sig_handler(int signo){

    if(signo==SIGINT){
        std::cout<<"server close"<<std::endl;
        close(g_sockfd);
        exit(1);
    }
}

auto showAddr(sockaddr_in* client_addr)->void{


    auto  port=ntohs(client_addr->sin_port);
    char ip[INET_ADDRSTRLEN]={0};
    //网络字节序->点分字节序
    auto p=inet_ntop(AF_INET,&client_addr->sin_addr,ip,INET_ADDRSTRLEN);
    if(p==nullptr){

        perror("error ip\r\n");
        return;
    }

    std::cout<<"clent ip->"<<p<<"port->"<<port<<std::endl;
}


auto doService(int fd)->bool{
    long t=time(0);
    char* s=ctime(&t);
    size_t size=strlen(s)*sizeof(char);

    //写入客户端
    if(write(fd,s,size)!=size){
        perror("write error!\r\n");
        return false;
    }

    return true;

}
auto main(int argc,char** argv)->int{
    if(argc<2){
        std::cout<<"usage :"<<argv[0]<<"#port"<<std::endl;
        exit(1);
        

    }

    if(signal(SIGINT,sig_handler)==SIG_ERR){

        perror("failed to set sigint handler\r\n");
        exit(1);
    }
    /*步骤一 创建socket ipv4 tcp协议 */
    g_sockfd=socket(AF_INET,SOCK_STREAM,0);
    /*步骤二 调用bind 将socket与ip port绑定*/
    sockaddr_in serverAddr{0};
    serverAddr.sin_family=AF_INET;
    serverAddr.sin_port=htons(atoi(argv[1]));//必须是网络字节序
    //INADDR_ANY 可以表示会响应所有ip请求
    serverAddr.sin_addr.s_addr=INADDR_ANY;
    if(bind(g_sockfd,(sockaddr*)&serverAddr,sizeof serverAddr)==-1){

        perror("failed  to bind\r\n");
        close(g_sockfd);
        exit(1);
    }
    /*调用lisent函数启用监听 会在你绑定的端口进行监听 他进入
    操作系统,通知OS,负责接受客户端对应端口的请求
    第二个是请求队列（os内部维护）最大个数
    */
    if(listen(g_sockfd,10)==-1){

        perror("failed to lisent\r\n");
        close(g_sockfd);
        exit(1);
    }

    /*调用accept 来进行获取队列里面的一个客户端链接请求
    
    */
   struct sockaddr_in clientAddr{0};
   socklen_t clientAddrLen=sizeof clientAddr;
   while(1){
    //这里获取到的文件描述符其实是客户端描述符
    //获取fd 就可以用read write进行与客户端进行通信了
    //这里会出现多种描述符 accept和socket是不一样的
    //linux的服务器的文件描述符主要是用于监听 accept 返回用户的fd
        int sfd=accept(g_sockfd,(sockaddr*)&clientAddr,&clientAddrLen);
        if(sfd==-1){
            perror("accept error\r\n");
            continue;
        }

        //调用io函数 双向通信
        showAddr(&clientAddr);
        doService(sfd);
        //关闭用户的套接字
        close(sfd);
   }
    return 0;
}
```

大概过程就是`socket`->`bind`->`lisent`->`accept`->`close`,这就是服务器端的流程;

打开,发现进程陷入阻塞,然后使用linux自带的telnet进行链接

![image-20230720180145582](./assets/image-20230720180145582.png)

可以发现,使用回环地址,链接成功,而且返回了系统时间,同时客户端也有响应

![image-20230720180216997](./assets/image-20230720180216997.png)

telnet用到的协议是应用层的Tenlnet的远程登陆的协议,但是为什么可以用传输层的tcp的协议通信呢?

**其实就是telnet在传输数据的时候经过传输层,他是传输层之上的,最终会用到传输层的,因此在传输层也可以获取这些数据;**
同时,这里可以看到clientIp和tenlent链接的serverip是相同的,不要惊讶,这是因为我们把服务器搭建在了本机上;==**而客户端也有一个端口,这个是OS指定的,不用管!**==

再比如,我们不一定用telnet远程登陆协议,可以用http协议这种应用层的;

![image-20230720181357060](./assets/image-20230720181357060.png)

只不过是因为应用层的协议不一样,返回的数据是一样,但是他如何(应用层)解析是不一样的;

### 客户端编程

前面我们使用了一些用户层的协议,来进行

```C++
#include <iostream>
#include <sys/socket.h>
#include <string.h>
#include <memory>
#include <arpa/inet.h>
#include <unistd.h>
auto main(int argc,char** argv)->int{

    if(argc<3){
        std::cout<<"usage: "<<argv[0]<<"\tip"<<"\tport"<<std::endl;

        exit(1);
    }
    /*步骤1 :创建socket*/
    auto clientFd=socket(AF_INET,SOCK_STREAM,0);
    if(clientFd==-1){
        perror("failed to create sockfd\r\n");
        exit(1);
    }

    /**客户端调用connect链接服务器*/
    
    auto serverAddr=sockaddr_in{.sin_port=htons(atoi(argv[2])),.sin_addr=0};
    serverAddr.sin_family=AF_INET;

    auto ret=inet_pton(AF_INET,argv[1],&serverAddr.sin_addr.s_addr);
    if(ret==-1){

        perror("failed to trans p to n\r\n");
        exit(1);
    }
    
    if(connect(clientFd,(sockaddr*)&serverAddr,sizeof serverAddr)==-1){
        perror("failed to connect \r\n");
        exit(1);
    }

    /**/
    char buf[0x1000]={0};
    ret=read(clientFd,buf,sizeof buf);
    if(ret==-1){
        perror("failed to read \r\n");
        exit(1);
    }
    close(clientFd);
    std::cout<<"socket read ->"<<buf<<std::endl;

    return 0;
}
```

然后指定服务端的ip,还有端口,可以成功获取

![image-20230720190323909](./assets/image-20230720190323909.png)

我们不论是客户端还是服务端都是在**传输层采用TCP直接进行进行链接,可靠的通信**;

总的来说,客户端的比较简单,如果服务器开启监听和接受,那么客户端只需要`socket`->`connect`就行;

### 三次握手和四次挥手

客户端想要通过TCP协议建立链接和关闭,需要经历一下步骤

- 三次握手

三次握手是 TCP 建立连接的过程，它的步骤如下：

1. **SYN**：客户端发送一个 SYN （synchronize）报文到服务器，请求建立连接。这个报文中包含了客户端的初始序列号。让我们假设这个初始序列号是 x。
2. **SYN-ACK**：服务器接收到 SYN 报文后，返回一个 SYN-ACK （synchronize-acknowledgement）报文给客户端。这个报文中包含了**服务器的初始序列号（让我们假设是 y）**和对**客户端初始序列号的确认（x+1）**。
3. **ACK**：客户端接收到 **SYN-ACK 报文后，返回一个 ACK （acknowledgement）报文给服务器，确认服务器的初始序列号（y+1）。此时，TCP 连接建立完成，数据传输可以开始**。

- 四次挥手

四次挥手是 TCP 终止连接的过程，它的步骤如下：

1. **FIN**：当数据传输完成后，发起关闭连接的一方（让我们假设是客户端）发送一个 FIN （finish）报文给另一方（服务器）。
2. **ACK**：服务器接收到 FIN 报文后，返回一个 ACK （acknowledgement）报文给客户端，确认收到了 FIN 报文。此时，**客户端到服务器的连接被关闭，但是服务器到客户端的连接仍然开启**，**因为 TCP 连接是双向的。**
3. **FIN**：**当服务器也准备好关闭连接时，它会发送一个 FIN 报文给客户端**。
4. **ACK**：客户端接收到 FIN 报文后，返回一个 ACK 报文给服务器，确认收到了 FIN 报文。此时，服务器到客户端的连接也被关闭。

![image-20230720191233223](./assets/image-20230720191233223.png)

​	在 TCP 协议中，"关闭连接"指的是不能再发送数据。但是，即使客户端到服务器的连接关闭，客户端仍然可以发送控制报文（例如 ACK 和 FIN）到服务器。这是因为这些控制报文是 TCP 协议本身的一部分，而不是实际的数据。==**因此四次挥手的时候虽然TCP单向通道被关闭,客户端仍然可以向服务端发送ACK FIN这种报文,这是不需要链接的;**==

### 处理器并发处理

我们在服务端**加上读取客户端**的代码

```C++
auto doService(int fd)->bool{
    long t=time(0);
    char* s=ctime(&t);
    size_t size=strlen(s)*sizeof(char);

    //在前面加个读取客户端的操作
    char rdBuf[20];
    if(read(fd,rdBuf,sizeof rdBuf)!=-1){
        //perror("rd error!\r\n");

    }
    //写入客户端
    if(write(fd,s,size)!=size){
        perror("write error!\r\n");
        return false;
    }

    return true;

}
```

可以发现,这个时候无论客户端还有服务端都阻塞了...

![image-20230721100323714](./assets/image-20230721100323714.png)

![image-20230721100316400](./assets/image-20230721100316400.png)

这个时候客户端没给服务器发数据,但是服务端读取了,会阻塞,但是客户端也在读,服务端也没写入,**这个时候双双就会陷入read阻塞**;后面,当你的客户端链接,也会阻塞;这样也只会处理一个客户端的操作;

我们要处理很多客户**并发处理**,可以采用很多个模型

- 多进程

服务器端这个父进程不会进行读取,**而是创建子进程进行通信**;父进程还是循环accept;

对于服务端,我们可以这样改

```C++
while(1){
    //这里获取到的文件描述符其实是客户端描述符
    //获取fd 就可以用read write进行与客户端进行通信了
    //这里会出现多种描述符 accept和socket是不一样的
    
        int sfd=accept(g_sockfd,(sockaddr*)&clientAddr,&clientAddrLen);
        if(sfd==-1){
            perror("accept error\r\n");
            continue;
        }

  

        /*启动子进程调用IO函数*/
        auto pid=fork(); 
        if(pid==-1) continue;//创建进程失败
        else if(pid==0){
            //子进程
            //调用io函数 双向通信
            showAddr(&clientAddr);
            doService(sfd);
            close(sfd);
            //处理完毕 子进程销毁
            break;
        }else{
            //父进程
            close(sfd);//父进程不需要这个sfd

        }
        
        //关闭用户的套接字
        
   }
//在DoSerivice里面
auto doService(int fd)->void{
    
    //与客户端进程双向通信

    char buf[512]={0};
    while(1){
        std::cout<<"start read write"<<std::endl;
        auto rdSize=read(fd,buf,sizeof buf);
        if(rdSize==-1){
            perror("read error!\r\n");
            continue;
        }else if(rdSize==0){

            //客户端已关闭; 子进程终止
            break;
        }else{

            std::cout<<"read :"<<buf<<std::endl;
            if(std::string(buf)=="getTime"){

                //需要获取当前时间 向客户端写入
                long t=time(0);
                char* s=ctime(&t);
                size_t size=strlen(s)*sizeof(char);

                //还要进行判断 如果服务器正在写入 客户关闭 这个时候write的时候会返回一个信号
                //而且errno会设置成EPIPE 和管道一样
                auto wtSize=write(fd,s,size);
                if(wtSize==-1 && errno==EPIPE){
                    break;
                }else if(wtSize==-1){
                    perror("failed to send msg to client\r\n");
                    break;
                }
                
            }else{
                //未知请求
                const char* unkRequest="unkown Request!";
                auto wtSize=write(fd,unkRequest,strlen(unkRequest)+1);
                if(wtSize==-1 && errno==EPIPE){
                    break;
                }else if(wtSize==-1){
                    perror("failed to send msg to client\r\n");
                    break;
                }
            }

            continue;
        }
    }
}
```

而对于客户端,我们这样改

```C++
 const char* sendData="getTime";
    char recvData[512]={0};
    size_t rdSize=0,wtSize=0;
    while (1)
    {
        wtSize=write(clientFd,sendData,strlen(sendData)+1);

        if(wtSize==-1){
            perror("send error!\r\n");
            continue;
        }else{

            rdSize=read(clientFd,recvData,sizeof recvData);
            if(rdSize==-1){
                perror("recv error!\r\n");
                continue;
            }else{

                std::cout<<"recv data"<<recvData<<std::endl;
            }
        }
    }
```

最后,服务器客户端可以实现如下情况:

![image-20230721111524405](./assets/image-20230721111524405.png)

![](./assets/image-20230721111539566.png)

总结一下就是,客户端通过fork来创建多个进程,然后再子进程里面进行读写判断进行与客户端口通信

而客户端则是先写入,然后再去读取;来发送和接受客户端的数据;

# chap03 UDP编程

基于TCP链接的,需要客户端和服务端都需要进行建立链接,而且TCP是可靠的,crc校验,基本不会出现差错;

UDP是不需要可靠性很高的场合,而且不建立链接,不在乎是否达到了,UDP一般用于在线视频解析,网络点播等场景;

![image-20230721113418330](./assets/image-20230721113418330.png)

对于服务器 UPD和TCP使用同样都是需要创建socket套接字,然后也是bind,进行端口 ip绑定,然后这个时候不需要lisent和accept了;而是直接readfrom;

## 发送和接受数据

- 发送

```C++
extern ssize_t send (int __fd, const void *__buf, size_t __n, int __flags);
```

```C++
extern ssize_t sendto (int __fd, const void *__buf, size_t __n,
		       int __flags, __CONST_SOCKADDR_ARG __addr,
		       socklen_t __addr_len);
```

sendto指明了目的地的信息;

- 接受

```C++
extern ssize_t recv (int __fd, void *__buf, size_t __n, int __flags);
```

```C++
extern ssize_t recvfrom (int __fd, void *__restrict __buf, size_t __n,
			 int __flags, __SOCKADDR_ARG __addr,
			 socklen_t *__restrict __addr_len);
```

对于recvfrom,第五个参数就是保存了发送方的ip,端口等一系列信息;

## UDP服务器编程

### 服务端编程

```C++
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <iostream>
#include <signal.h>
#include <time.h>
#include <memory>
#include <arpa/inet.h>
int g_sockfd;
auto sig_handler(int signo)->void{
    if(signo==SIGINT){
        std::cout<<"server close"<<std::endl;
        close(g_sockfd);
    }

}

auto show_addr(sockaddr_in* addr)->void{
    char pIp[16]={0};

    auto p=inet_ntop(AF_INET,&addr->sin_addr,pIp,sizeof pIp);
    if(p==nullptr){

        perror("failed to get client ip!\r\n");
        return;
    }

    std::cout<<"client connected ip ->"<<p<<std::endl;
}
auto do_service(int fd)->void{
    
     const auto recvLen=1024;
    auto clientAddr=sockaddr_in{0};
    auto recvBuf=std::make_unique<char[]>(recvLen);
    socklen_t len=sizeof(sockaddr_in);
    /*4.recvfrom 获取*/
    auto recvRet=recvfrom(fd,recvBuf.get(),recvLen,0,(sockaddr*)&clientAddr,&len);
    if(recvRet==-1){

        perror("recv error");
        return;

    }
    std::cout<<"get info->\t"<<recvBuf.get()<<std::endl;
    /*4.调用doService通信*/
    show_addr(&clientAddr);
    long int t=time(0);
    char* p=ctime(&t);
    if(sendto(fd,p,strlen(p)+1,0,(sockaddr*)&clientAddr,sizeof clientAddr)<0){
        perror("send info err!");
        return;

    }


}
auto main(int argc,char** argv)->int{
    if(argc<2){
        std::cout<<"usage "<<argv[0]<<"\tport"<<std::endl;
        exit(1);
    }

    /*1.创建socket*/
    g_sockfd=socket(AF_INET,SOCK_DGRAM,0);
    if(g_sockfd<0){
        perror("socket error!\r\n");
        exit(1);
    }

    /*2.创建套接字选项 让端口关闭立刻就能打开*/
    int opt=1;
    auto setRet=setsockopt(g_sockfd,SOL_SOCKET,SO_REUSEADDR,&opt,sizeof opt);
    if(setRet==-1){

        perror("set socket opt error");
        close(g_sockfd);
        exit(1);
    }

    /*3.调用bind 绑定socket和地址*/
    auto serverAddr=sockaddr_in{.sin_family=AF_INET,.sin_port=htons(atoi(argv[1])),.sin_addr=INADDR_ANY,.sin_zero={0}};

    auto bindRet=bind(g_sockfd,(sockaddr*)&serverAddr,sizeof(sockaddr));
    if(bindRet==-1){
        perror("bind error\r\n");
        close(g_sockfd);
        exit(1);
    }

    while(1){
        do_service(g_sockfd);
    }
    return 0;
}
```

### 客户端编程

```C++
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <iostream>
#include <signal.h>
#include <time.h>
#include <memory>
#include <arpa/inet.h>

auto main(int argc,char** argv)->int{

    if(argc<3){
        std::cout<<"usage ip port"<<std::endl;
        exit(1);
    }

    /*1.创建socket*/
    auto sockfd=socket(AF_INET,SOCK_DGRAM,0);
    if(sockfd<0){
        std::cout<<"create socket error!"<<std::endl;
        exit(1);
    }

    /*调用recvfrom和sendto 双向通信*/
    auto serverAddr=sockaddr_in{.sin_family=AF_INET,.sin_port=htons(atoi(argv[2])),.sin_addr=0,.sin_zero={0}};
    inet_pton(AF_INET,argv[1],&serverAddr.sin_addr.s_addr);

    /*像服务器发送数据*/
    const auto sendLen=1024;
    auto sendData=std::make_unique<char[]>(sendLen);
    strcpy(sendData.get(),"hello udp");

    auto sendRet=sendto(sockfd,sendData.get(),sendLen,0,(sockaddr*)&serverAddr,sizeof serverAddr);
    if(sendRet==-1){

        perror("send error!");
        close(sockfd);
        exit(1);
    }

    /*客户端接受报文*/

    auto recvData=std::make_unique<char[]>(sendLen);
    //recvfrom和recv都行,客户端不需要服务端的信息
    //但是服务端recv肯定要recvfrom
    auto recvRet=recv(sockfd,recvData.get(),sendLen,0);
    if(recvRet==-1){

        perror("recv error");
        close(sockfd);
        exit(1);
    }

    std::cout<<"recv data->\t"<<recvData.get()<<std::endl;

    return 0;
}
```

可以看到,UDP其实要比TCP简单方便的多,因为它不需要确定链接,对于服务器来说,bind之后,直接进行接受(`recvfrom`)即可,而对于客户端来说,在创建完socket之后,直接进行`sendto`即可,也不需要`connect`了;最后结果如下。而这里不需要bind的原因是因为我们使用了`sendto`,使用`send`还是需要绑定的;这里说的`bind`其实就是`connect`;

![image-20230721170734862](./assets/image-20230721170734862.png)

![image-20230721170747420](./assets/image-20230721170747420.png)

同时,我们还可以发现UDP其实根本用不到判断客户端链接是否关闭,不像TCP那样,因为UDP是无连接的;

而且,其实也可以不用`sendto`,而是用`connect`,只不过UDP的`connect`没有经过三次握手,不过`connect`之后,就可以直接用`send`就行了,因为已经记录客户端链接了;

**调用connect的好处,就是recv的时候可以保证一定来源于服务器端;**

## 域名和ip

端口,只能被默认绑定一次,除非你在sockopt的时候设置,端口才可以被多次绑定;

但是一旦设置,只有是最后一次绑定端口的那个服务端才会接受数据;

我们在访问网页的时候,比如访问谷歌,会访问`www.google.com`,其实本质上还是访问谷歌的ip地址;

不过这个时候用到的是`DNS`来进行域名解析,把域名解析成谷歌的IP地址

![image-20230721180018810](./assets/image-20230721180018810.png)

DNS就是域名服务器,客户端访问的时候,可以不指定,而是指定域名;

### 域名解析函数

```C++
//<netdb.h>
struct hostent {
    char  *h_name;            /* official name of host */ 主机名
    char **h_aliases;         /* alias list */ 别名 是很多个别名
    int    h_addrtype;        /* host address type */ 协议类型
    int    h_length;          /* length of address */ 网络地址大小
    char **h_addr_list;       /* list of addresses */
};
//<netdb.h>
struct hostent* gethostent(void);
struct hostent* getthostbyname(const char* hostname);
void sethostent(int stayopen);
void endhostent(void);
```

而在本机，linux下,`/ect/hosts`就记录了ip地址到域名之间映射的关系

![image-20230721180153155](./assets/image-20230721180153155.png)

比如这里我们可以修改这个这个文件

![image-20230721180705202](./assets/image-20230721180705202.png)

这里,oxygen oxygen1 www.xxx都是域名 **第一个就是主机名**,而除了第一个之后，**都叫做别名**;

正对应上面`hostent`这个结构的内容;

- **gethostbyname**

然后我们写如下代码,比如使用`gethostbyname()`,随便一个ip地址的别名都可以获取到对应的结构体

如下代码

```C++
#include <netdb.h>
#include <sys/types.h>
#include <unistd.h>
#include <iostream>
#include <arpa/inet.h>
auto main(int argc,char** argv)->int{

    if(argc<2){

        std::cout<<"usage "<<argv[0]<<"host"<<std::endl;
        exit(1);
    }
    auto ent=gethostbyname(argv[1]);
    if(ent==nullptr){

        perror("failed to get host by name");
        exit(1);
    }

    std::cout<<"host name ->"<<ent->h_name;
    for(auto p=ent->h_aliases;*p!=nullptr;p++){

        std::cout<<"alias name\t"<<*p<<std::endl;
    }
    std::cout<<std::endl;
    
    std::cout<<"ip list->";

    std::cout<<(ent->h_addrtype==AF_INET?"ipv4":"ipv6")<<std::endl;
    for(auto p=ent->h_addr_list;*p!=nullptr;p++){
        //std::cout<<"\t"<<*p;
        char buf[INET_ADDRSTRLEN]={0};
        inet_ntop(ent->h_addrtype, p, buf, INET_ADDRSTRLEN);
        std::cout<<"\t"<<buf;
    }
    std::cout<<std::endl;


     return 0;
}
```

![image-20230721190632795](./assets/image-20230721190632795.png)

![image-20230721190730721](./assets/image-20230721190730721.png)

- gethostent

`hostent`就是获取/etc/hosts（**本地 DNS 缓存查找**）的相关域名和别名,如果没有就返回0；

有了这些,我们就可以通过这个解析对应的域名,来获取相关的ip地址;进而在TCP/UDP编程之中,加一些代码就可以将域名->ip进行转换;

## 广播

广播就是实现一对多通信的一种途径;他是通过向**广播地址**发送数据报文实现的;

比如QQ聊天,两个人聊天可以是TCP/UDP,但是群聊就是广播的实现;

广播地址是一种特殊的ip地址,只要向广播地址发送这个数据,所有都会

广播设置我们需要用到`getsockopt`和`setsockopt`来进行设置SO_BROADCAST;**广播是只能使用UDP协议才可以;**

对于广播地址,实际上是比较特殊的,广播的其实只在相同子网ID才会发送;

### 发送端

```C++
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <iostream>
#include <signal.h>
#include <time.h>
#include <memory>
#include <arpa/inet.h>

auto main(int argc,char** argv)->int{

    if(argc<3){
        std::cout<<"usage ip port"<<std::endl;
        exit(1);
    }

    /*1.创建socket*/
    auto sockfd=socket(AF_INET,SOCK_DGRAM,0);
    if(sockfd<0){
        std::cout<<"create socket error!"<<std::endl;
        exit(1);
    }


    /*设置socket opt 来进行使用广播方式进行发送*/
    int opt=1;
    setsockopt(sockfd,SOL_SOCKET,SO_BROADCAST,&opt,sizeof opt);
    /*调用recvfrom和sendto 双向通信*/
    auto serverAddr=sockaddr_in{.sin_family=AF_INET,.sin_port=htons(atoi(argv[2])),.sin_addr=0,.sin_zero={0}};
    inet_pton(AF_INET,argv[1],&serverAddr.sin_addr.s_addr);

    std::cout<<"broadcasting..."<<std::endl;
    /*像服务器发送数据*/
    const auto sendLen=1024;
    auto sendData=std::make_unique<char[]>(sendLen);
    strcpy(sendData.get(),"haha there is a broadcast!\r\n");

    auto sendRet=sendto(sockfd,sendData.get(),sendLen,0,(sockaddr*)&serverAddr,sizeof serverAddr);
    if(sendRet==-1){

        perror("send error!");
        close(sockfd);
        exit(1);
    }


    return 0;
}
```

### 接收端

```C++
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <iostream>
#include <signal.h>
#include <time.h>
#include <memory>
#include <arpa/inet.h>
int g_sockfd;
auto sig_handler(int signo)->void{
    if(signo==SIGINT){
        std::cout<<"server close"<<std::endl;
        close(g_sockfd);
    }

}

auto show_addr(sockaddr_in* addr)->void{
    char pIp[16]={0};

    auto p=inet_ntop(AF_INET,&addr->sin_addr,pIp,sizeof pIp);
    if(p==nullptr){

        perror("failed to get client ip!\r\n");
        return;
    }

    std::cout<<"client connected ip ->"<<p<<std::endl;
}
auto do_service(int fd)->void{
    
     const auto recvLen=1024;
    auto clientAddr=sockaddr_in{0};
    auto recvBuf=std::make_unique<char[]>(recvLen);
    socklen_t len=sizeof(sockaddr_in);
    /*4.recvfrom 获取*/
    auto recvRet=recvfrom(fd,recvBuf.get(),recvLen,0,(sockaddr*)&clientAddr,&len);
    if(recvRet==-1){

        perror("recv error");
        return;

    }
    std::cout<<"get info->\t"<<recvBuf.get()<<std::endl;
    /*4.调用doService通信*/
    show_addr(&clientAddr);
    long int t=time(0);
    char* p=ctime(&t);
    if(sendto(fd,p,strlen(p)+1,0,(sockaddr*)&clientAddr,sizeof clientAddr)<0){
        perror("send info err!");
        return;

    }


}
auto main(int argc,char** argv)->int{
    if(argc<2){
        std::cout<<"usage "<<argv[0]<<"\tport"<<std::endl;
        exit(1);
    }

    /*1.创建socket*/
    g_sockfd=socket(AF_INET,SOCK_DGRAM,0);
    if(g_sockfd<0){
        perror("socket error!\r\n");
        exit(1);
    }

    /*2.创建套接字选项 让端口关闭立刻就能打开*/
    int opt=1;
    auto setRet=setsockopt(g_sockfd,SOL_SOCKET,SO_REUSEADDR,&opt,sizeof opt);
    if(setRet==-1){

        perror("set socket opt error");
        close(g_sockfd);
        exit(1);
    }

    /*3.调用bind 绑定socket和地址*/
    auto serverAddr=sockaddr_in{.sin_family=AF_INET,.sin_port=htons(atoi(argv[1])),.sin_addr=INADDR_ANY,.sin_zero={0}};

    auto bindRet=bind(g_sockfd,(sockaddr*)&serverAddr,sizeof(sockaddr));
    if(bindRet==-1){
        perror("bind error\r\n");
        close(g_sockfd);
        exit(1);
    }
    const auto len=1024;
    auto buf=std::make_unique<char[]>(len);
    sockaddr_in clientAddr{0};
    socklen_t csize=sizeof clientAddr;
    while(1){
        //do_service(g_sockfd);
        if(recvfrom(g_sockfd,buf.get(),len,0,(sockaddr*)&clientAddr,&csize)<0){
            perror("failed  to recv from\r\n");
            exit(1);
        }else{

            char ip[16];
            auto p=inet_ntop(AF_INET,&clientAddr.sin_addr,ip,sizeof ip);
            if(p==nullptr){
                perror("failed to n to p");
                exit(1);
            }
            auto port=ntohs(clientAddr.sin_port);

            std::cout<<"recv suc! ip->"<<ip<<"port->"<<port<<std::endl;
        }
    }
    return 0;
}
```

其实不难发现,广播发送和UDP是基本一模一样,唯一区别就是广播发送需要指定发送端的socket必须是广播形式,而且还得输入广播的ip;

最后发送端

![image-20230721201023149](./assets/image-20230721201023149.png)

![image-20230721201150044](./assets/image-20230721201150044.png)

可以发现,接收者也能接受到了;

# chap04 windows网编

在windows上,很少人使用网络编程,归根结底还是因为服务器windows用的很少,但是由于都是TCP/IP协议

也是需要套接字,也是需要服务器客户端各有ip,服务器在端口上进行监听的;三次握手四次挥手,UDP,TCP都是和linux类似,**只不过是数据结构和API变了**(==极少数==);

## 套接字流程

对于win来说,引入头文件只需要包含`#include <WinSock2.h>`就行了;同时还要引用这个`#pragma comment(lib,"ws2_32.lib")`库

其次,对于socket创建,无论是客户端还是服务端,都必须先初始化**socket环境**;

```C++
WSADATA wsaData;
if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
    std::cerr << "Failed to initialize winsock." << std::endl;
    return 1;
}
```

这个目的主要是告诉windows你的socket版本,比如MAKEWORD(2,2)就代表winsock2.2版本;

`wsaData`将被函数填充以返回Winsock实现的详细信息。

`WSAStartup`调用成功返回0,否则返回错误值;因此初始化这样写即可->

```C++
auto initWinsockVersion() -> bool {
	auto v = MAKEWORD(2, 2);
	auto wsaData = WSADATA{ 0 };
	auto errorCode=WSAStartup(v, &wsaData);

	if (errorCode != 0) {

		std::cerr << "failed to init winsock ver" << "\t errCode->" << errorCode << std::endl;
		return false;
	}

	return true;

}
```

在初始化完环境之后,可以使用socket这个api来进行创建套接字了,这里win和linux基本一致,win也使用`socket`这个API,区别是参数和返回值有细微差别;这主要是win和linux设计差别导致的;

我们知道,win的设计思想一切皆对象,那么返回的毫无疑问,不是文件描述符,而是socket对象的句柄;

```C++
auto hSock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
```

上面是hSock其实是一个`SOCKET`结构,本质上是句柄,除此之外,上面是创建的是基于TCP协议的套接字,还可以创建UDP协议的套接字

```C++
auto hSock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
```

接下来,服务器和客户端的创建流程就有差别了,这一点其实和linux基本一样;

因为服务器要绑定端口,要开启监听,要接受消息;而客户端则只需要创建完socket,connect就行(TCP协议的套接字);

## 服务端

对于服务端,则需要绑定能接受的ip地址以及开启监听的端口;也是通过`bind`这个api;而且用到的结构也和linux相同;参数除了linux填文件描述符,win填socket之外其他完全一样;

```C++
#include <windows.h>
#include <iostream>
#include <WinSock2.h>//socket主要API
#include <memory>
#include <thread>
#include <WS2tcpip.h>//inet_ntop...
#include <chrono>//time

auto initWinsockVersion() -> bool {
	auto v = MAKEWORD(2, 2);
	auto wsaData = WSADATA{ 0 };
	auto errorCode=WSAStartup(v, &wsaData);

	if (errorCode != 0) {

		std::cerr << "failed to init winsock ver" << "\t errCode->" << errorCode << std::endl;
		return false;
	}

	return true;

}

auto createSocket() -> SOCKET {

	auto hSock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

	if (hSock == INVALID_SOCKET) std::cerr << "failed to create socket!" << std::endl;
	
	return hSock;
}


SOCKET g_socket;
auto main(int argc, char** argv) -> int {

	if (argc < 2) {

		std::cerr << "usage " << argv[0] << "\t port" << std::endl;
		exit(1);
	}

	/*1.初始化socket版本 创建socket*/
	auto b = initWinsockVersion();
	if (b == 0) exit(1);

	g_socket = createSocket();
	if (g_socket == INVALID_SOCKET) exit(1);

	/*绑定socket*/

	auto _sin = sockaddr_in{ .sin_family = AF_INET,.
		sin_port = htons(atoi(argv[1])),.sin_addr = INADDR_ANY,.sin_zero = {0} };

	if (bind(g_socket, (sockaddr*)&_sin, sizeof _sin) == SOCKET_ERROR) {

		std::cerr << "failed to bind socket" << std::endl;
		exit(1);
	}

	/*开启监听*/
	if (listen(g_socket, 10) == SOCKET_ERROR) {

		std::cerr << "failed to listen!" << std::endl;
		exit(1);
	}

	/*接受请求 考虑多线程*/
	while (1) {
		auto cSockAddr = sockaddr_in{ 0 };
		int cSockAddrSize = sizeof cSockAddr;
		auto cSocket = accept(g_socket, (sockaddr*)&cSockAddr,&cSockAddrSize);
		if (cSocket == INVALID_SOCKET) {

			std::cerr << "failed to accept!" << std::endl;
			continue;
		}

		std::thread do_service([cSocket, cSockAddr]() {

			//解析用户的sockaddr
			auto ip = std::make_unique<char[]>(INET_ADDRSTRLEN);

			auto p = inet_ntop(AF_INET, &cSockAddr.sin_addr, ip.get(), INET_ADDRSTRLEN);

			if (p == nullptr) {

				std::cerr << "failed to trans n to p" << std::endl;
				closesocket(cSocket);
				return;
			}

			auto port = ntohs(cSockAddr.sin_port);

			std::cout << "connect success client ip:" << p << "\tport:" << port << std::endl;

			while (1) {
				//开始读取 响应用户请求
				const char recvLen = 1024;
				auto recvBuf = std::make_unique<char[]>(recvLen);
				auto sendBuf = std::make_unique<char[]>(recvLen);

				auto recvBytes = recv(cSocket, recvBuf.get(), recvLen, 0);
				if (recvBytes > 0) {
					//正常接受
					std::cout << "recv success" << std::endl;

					if (std::string(recvBuf.get()) == "get") {

						strcpy(sendBuf.get(), "you get success! cur Time :");
						auto now=std::chrono::system_clock::now();
						auto now_time_t = std::chrono::system_clock::to_time_t(now);
						strcat(sendBuf.get(),std::ctime(&now_time_t));
					}
					else {
						strcpy(sendBuf.get(), "unkown request!");
					}
					
				}
				else if (recvBytes == 0) {
					//TCP套接字单向关闭
					std::cerr << "client socket close" << std::endl;
					closesocket(cSocket);
					return;
				}
				else {
					//错误
					std::cerr << "recv error,errcode:" << WSAGetLastError() << std::endl;
					continue;
				}


				//写入
				auto sendBytes = send(cSocket, sendBuf.get(), recvLen, 0);
				if (sendBytes == 0) {
					//TCP套接字单向关闭
					std::cerr << "client socket close" << std::endl;
					closesocket(cSocket);
					return;

				}else if (sendBytes > 0) {
					std::cout << "send to client success!" << std::endl;
				}
				else {

					std::cerr << "send error,errcode:" << WSAGetLastError() << std::endl;
					continue;
				}
			}



			});

		do_service.detach();


	}
	closesocket(g_socket);

	return 0;
}
```

## 客户端

最终实现效果如下

![image-20230722151408168](./assets/image-20230722151408168.png)

当然上面的

例子其实不太好,因为对于判断recv send那块服务端和客户端是否结束还有些逻辑问题；但是无论如何,win和linux的网络编程都是很相似的,学会其中任何一个基本可以无缝衔接另一个；

