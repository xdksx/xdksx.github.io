---
title: tcpip_ARP
date: 2018-05-28 00:19:57
tags: tcpip_mac
categories: tcpip
---
### ARP 协议
#### ARP协议的作用
+ ARP是一种地址解析协议，ARP的存在是为了能在端到端通信时的寻找到端地址；ARP由设备的ip地址获取到设备的mac硬件地址，从而能跟该设备通信；
+ 在网络层，有ip地址和路由作为寻路的导向，即通过封装源地址和目的地址，以及路由，一个ip包在不考虑mac层的时候大概是这样传输的：  
  端设备手机pc等-->(交换机)路由器--->下一个路由器--->下一个终端设备  <!--more-->
  端设备默认路由到网关-->网关收到包后，将源地址改为网关的外网地址--->下一个路由-->....->设备地址为目的地址时，接收而不转发
+ 考虑一个问题：在上述的网络上发送时，如pc发给路由器时，只知道它的ip地址，怎么发给它：  
  即端到端的发送：是借助设备唯一的mac地址来发送的  
  在有线网中。利用了交换机的端口和mac地址关系，转发  
  在wifi中，能接收到所有的包，但是会在驱动中将mac目的地址非本地的包丢弃，除非开启了混杂监听模式
+ 但是路由器怎么知道设备如手机的mac地址呢？
  通过arp协议来获取，arp是依赖mac和ip的"映射"
+ tcpip卷1中4.2举了一个完整的例子，可以去看
#### ARP协议的交互过程
+ 基本的交互方式：  
例如ping网关：  
station  ---ARP request---->   AP  ARP请求，广播帧    
station  <---ARP response ---  AP  ARP应答 ,单播帧  
通过上述的方式，station得到了AP的mac地址，之后将AP的MAC地址存在缓存中，在下次数据包等包的时候就可以将该mac地址填在mac头中，即知道了AP的地址后，包就可以顺利的被AP接收了
+ AP(路由器)也会主动的向station发送arp request，当它不知道station的mac地址（或缓存过期），例如，AP收到一个外网的包，要转发给对应192.168.0.4的staton,但是没有他的mac地址，或者过期了或者其他情况，则向该station发出arp请求。从而得打station的mac地址
+ 什么时候会触发arp请求？  
  1.在ping的时候  
  2.在发送tcp，ip包的时候  
  3.在缓存过期时主动发出，这个由arp状态机中实现
#### ARP代理和免费ARP
+ 当我们通过ping百度，或者想访问百度的时候，怎么在mac头中填写目的mac地址呢？即怎么找到百度?（另一个网络)
+ 是通过在mac头目的mac地址填写网关的mac地址实现的，借助w网关的来帮我们寻找，这个叫做ARP代理,其实欺骗了主机
+ tcpip卷对此有较详细解说4.6
+ 免费ARP是用于检测是否周围有使用相同的ip地址的设备，即通过ping自己，寻找有自己的ip的主机
#### ARP协议的包封装格式和抓包分析
+ 分组格式：  ()为字节数
mac头：以太网目的地址(6) mac源地址(6) 帧类型(2)(0x0806)   
28byte arp帧：硬件类型(2)|协议类型(2)|硬件地址长度(1)|协议地址长度(1)|op(请求或应答或rarp 2)|发送端mac地址(6)|发送端ip(4)|接收端mac地址(6)|接收端ip(4)  
+ 请求广播帧：目的mac地址为全1，电缆上所有的以太网接口都要接收广播帧,硬件类型为1表示以太网地址，协议类型为要映射的协议地址类型0x0800,表示为ip地址，op（arp请求为1，答应为2，rarp请求为3，rarp答应为4）
+ arp填充：以太网帧最小值为60，mac头14，所以=46，要填充：46-28=18个字节
+ arp抓包，可以用wireshark,Omnipeek - Savvius,tcpdump,或者自己写一个程序来抓～  
 tcpdump : sudo tcpdump -vv arp  
 ```c
 tcpdump: listening on wlp2s0, link-type EN10MB (Ethernet), capture size 262144 bytes
21:51:21.134575 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.0.101 tell 192.168.0.1, length 28 //ap ask me
21:51:21.135404 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.0.110 tell 192.168.0.1, length 28
21:51:21.135413 ARP, Ethernet (len 6), IPv4 (len 4), Reply 192.168.0.110 is-at 48:5a:b6:6e:c9:5f (oui Unknown), length 28// i reply ap
21:51:32.081916 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.0.106 tell 192.168.0.1, length 28 //ask other
```

```c 
22:01:27.149947 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 192.168.0.107 tell 192.168.0.1, length 28
	0x0000:  0001 0800 0604 0001 206b e70f 1b42 c0a8  .........k...B..
	0x0010:  0001 0000 0000 0000 c0a8 006b            ...........k
    
    22:04:17.114593 ARP, Ethernet (len 6), IPv4 (len 4), Reply 192.168.0.110 is-at 48:5a:b6:6e:c9:5f (oui Unknown), length 28
	0x0000:  0001 0800 0604 0002 485a b66e c95f c0a8  ........HZ.n._..
	0x0010:  006e 206b e70f 1b42 c0a8 0001            .n.k...B....
```

#### ARP协议的常用命令和调试分析
+ 查看ARP缓存：即现在保存的arp映射表:  
```
arp -a
? (192.168.0.101) at 4c:32:75:3a:09:b3 [ether] on wlp2s0
? (192.168.0.1) at 20:6b:e7:0f:1b:42 [ether] on wlp2s0
? (192.168.0.108) at 94:d0:29:9d:74:dd [ether] on wlp2s0
? (192.168.0.107) at 94:65:2d:ab:88:8b [ether] on wlp2s0
```
```
arp 
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.0.101            ether   4c:32:75:3a:09:b3   C                     wlp2s0
192.168.0.1              ether   20:6b:e7:0f:1b:42   C                     wlp2s0
```
```c
ip neigh
192.168.0.101 dev wlp2s0 lladdr 4c:32:75:3a:09:b3 STALE
192.168.0.1 dev wlp2s0 lladdr 20:6b:e7:0f:1b:42 STALE
192.168.0.108 dev wlp2s0 lladdr 94:d0:29:9d:74:dd STALE
192.168.0.107 dev wlp2s0 lladdr 94:65:2d:ab:88:8b STALE
192.168.0.104 dev wlp2s0 lladdr e4:9a:dc:b0:a5:36 STALE
```
+ arping命令：  
http://man.linuxde.net/arping
+ arp 命令
man arp 包括删除arp表项等，有问题，找男人~
#### ARP协议内核状态机
+ 对不存在的主机，arp请求的超时机制  
+ arp缓存和老化时间：
http://www.jb51.net/LINUXjishu/65693.html：
改变老化时间：在Linux上如果你想设置你的ARP缓存老化时间，那么执行sysctl -w net.ipv4.neigh.ethX=Y即
#### ARP协议的编程
+ 在PF_PACKET中发出ARP包
+ 直接贴例子，具体可以看博客的PF_PACKET文章：
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <linux/if.h>
#include <sys/ioctl.h>
#include <linux/if_packet.h>
#include <linux/if_ether.h>
```

```cpp
//获取硬件网卡的相应信息
void GetEthInfor(char *name , char *MAC_addr , struct in_addr * IP_addr);
//arp包的结构定义
struct ARP_PACKET
{
       //以太网首部
 unsigned char dest_mac[6]; //6字节
 unsigned char sorce_mac[6];//6字节
 unsigned short type;       //2字节
  //arp——内容
 unsigned short hw_type;   //2字节：硬件地址类型     0x0001 表示mac地址
 unsigned short pro_type;  //2字节：软件地址类型    0x0806 表示IPV4地址
 unsigned char hw_len;     //1字节：硬件地址长度  
 unsigned char pro_len;    //1字节：软件地址长度
 unsigned short op;        //2字节：操作类型           0x0001表示ARP请求；0x0002表示ARP应答
 unsigned char from_mac[6];//6字节
 unsigned char from_ip[4]; //4字节
 unsigned char to_mac[6];  //6字节
 unsigned char to_ip[4];   //4字节
 unsigned char padding[18];//18字节：填充字节，因为以太网数据最少要46字节
};
//主函数
int main()
{
 int i = 0;
 int fd = 0;
 int num=0;
 unsigned char MAC_ADDR[6];
 struct in_addr IP_ADDR;//结构体的定义在/netinet/in.h用来表示一个32位的IPv4地址.
 struct ARP_PACKET arp_pk={0};
 struct sockaddr_ll eth_info;//结构体的定义//在/linux/if_packet.h中， 表示设备无关的物理层地址结构
 //第一步：获取指定网卡的信息（MAC地址和IP地址）
 GetEthInfor("wlp2s0",MAC_ADDR,&IP_ADDR);
   /*printf("The MAC_addr is:");
 for(i =0 ;i<6;i++)
    printf("%4X",MAC_ADDR[i]); 
 printf("\n");
    printf("the IP is:%s\n",inet_ntoa(IP_ADDR));*/
      
    //第二步：填充ARP数据包的内容
 for(i=0;i<6;i++)                 //填充以太网首部的目的mac地址
 {
  arp_pk.dest_mac[i]=0XFF;      
 }
  
 for(i=0;i<6;i++)                 //填充以太网首部的源mac地址
 {
  arp_pk.sorce_mac[i]=MAC_ADDR[i];
 }
   arp_pk.type = htons(0x0806);    //填充以太网首部的侦类型
   arp_pk.hw_type = htons(0x0001); //填充硬件地址类型：0x0001表示的是MAC地址
   arp_pk.pro_type = htons(0x0800);//填充协议地址类型：0x0800表示的是IP地址
   arp_pk.hw_len = 6;              //填充硬件地址长度
   arp_pk.pro_len = 4;             //填充协议地址长度
   arp_pk.op = htons(0x0001);      //填充操作类型：0x0001表示ARP请求

   for(i=0;i<6;i++)                 //填充源mac地址
   {
  arp_pk.from_mac[i]=MAC_ADDR[i];
   }
   in_addr_t ipaddr=inet_network(inet_ntoa(IP_ADDR));
   for(i=3;i>=0;i--)                 //填充源IP地址
   {
  arp_pk.from_ip[i]=(unsigned char)ipaddr&0xFF;
   ipaddr=ipaddr>>8;
  printf("-%d-",arp_pk.from_ip[i]);
   }
 /* arp_pk.from_ip[0]=192;
  arp_pk.from_ip[1]=168;
  arp_pk.from_ip[2]=199;
  arp_pk.from_ip[3]=145;*/
  
   for(i=0;i<6;i++)                 //填充欲获取的目的mac地址
   {
    arp_pk.to_mac[i]=0X00;
   }
  
   arp_pk.to_ip[0]=0X0B;        //填充想要装换为MAC地址的IP地址。可以使用命令行参数来做
   arp_pk.to_ip[1]=0X40;
   arp_pk.to_ip[2]=0X39;
   arp_pk.to_ip[3]=0X0A;
 
 //第三步：填充sockaddr_ll eth_info结构
    eth_info.sll_family = PF_PACKET;
 eth_info.sll_ifindex = if_nametoindex("wlp2s0");//返回输入的接口名称的索引值
 //printf("number is:%d\n",eth_info.sll_family);
 
 //第四步：创建原始套接字
 fd = socket(PF_PACKET,SOCK_RAW,htons(ETH_P_ALL));  //
 if(fd<0)
 {
  printf("socket SOCK_RAW failed!\n");
  exit(1);
 }

 

 //第五步：发送ARP数据包
 num = sendto(fd , &arp_pk , sizeof(struct ARP_PACKET) , 0 ,(struct sockaddr*)(&eth_info),sizeof(eth_info));
 if(num<0)
 {
  printf("sendto failed!\n");
  exit(1);
 }
        //第六步：接受ARP应答
 num = recvfrom(fd , &arp_pk , sizeof(struct ARP_PACKET) ,0,NULL,0);

 if(num<0)
 {
  printf("rcvfrom failed!\n");
  exit(1);
 }
 else
 {
  printf("I receive %d bytes!\n",num);
  printf("the mac  is:");
  for(i=0;i<6;i++)
  {
   printf("%4X ",arp_pk.from_mac[i]);
  }
  printf("op:%d\n",arp_pk.op);
 for(i=0;i<4;i++)
  {
   printf("%d. ",arp_pk.to_ip[i]);
  }
  printf("\n");
 }

 close(fd);
 return 0;
}

void GetEthInfor(char *name ,  char *MAC_addr , struct in_addr * IP_addr)
{
 struct ifreq  eth;  //够结构用于存放最初多获取的接口信息
//该结构存放在：/net/if.h,详细字段表示在头文件中
   int fd;             //用于创建套接字
 int temp=0;         //用于验证接口调用
 int i=0;            //用于循环
 
 strncpy(eth.ifr_name,name,sizeof(struct ifreq)-1);
// fd = socket(AF_INET,SOCK_DGRAM,0);
 // fd=socket(AF_INET,SOCK_STREAM,0); 
    fd=socket(AF_PACKET,SOCK_DGRAM,htons(ETH_P_ALL));
if(fd<0)
 {
   printf("socket failed!\n");
   exit(1);
 }
 //获取并且保存和打印指定的物理接口MAC地址信息
   temp = ioctl(fd,SIOCGIFHWADDR,&eth);
 if(temp<0)
 {
  printf("ioctl--get hardware addr failed!\n");
  exit(1);
        }
 strncpy(MAC_addr,eth.ifr_hwaddr.sa_data,6);
 printf("The MAC_addr is:");
 for(i =0 ;i<6;i++)
  printf("%4X",(unsigned char)eth.ifr_hwaddr.sa_data[i]); 
 printf("\n");
       
        //获取并且保存和打印指定的物理接口IP地址信息
   temp = ioctl(fd,SIOCGIFADDR,&eth);
 if(temp<0)
 {
  printf("ioctl--get hardware addr failed!\n");
  exit(1);
        }
 memcpy(IP_addr ,&(((struct sockaddr_in *)(&eth.ifr_addr))->sin_addr),4);
 //关闭套接口
  printf("got ipaddr:%s\n",inet_ntoa(*IP_addr));
/*i=0;
printf("get the MAC_ADDR:\n");
for(i;i<6;i++)
  printf("%.2X:",MAC_addr[i]&0xFF);*/
   close(fd);
}
//第二步填充arp数据包的时候，填充的数据和类型根据需要进行。
//这里的arp数据包结构体是自己定义的，也可以是利用linux的相关结构：
/*eg:
typedef struct _tagARP_PACKET{    
    struct ether_header  eh;    ///net/ethernet.h
    struct ether_arp arp;    
}ARP_PACKET_OBJ, *ARP_PACKET_HANDLE; /netinet/if_ether.h;/net/if_arp.h
各个字段的填充见头文件*/

/*第三步：填充sockaddr_ll eth_info结构,关键是填充网络接口索引值，最直接的方法是上面的函数，
还有另外的方法:利用ioctl函数获取struct ifreq结构的信息，该结构放置了很多网络接口的相关信息*/
/*  struct sockaddr_ll{
    unsigned short sll_family; //总是 AF_PACKET 
    unsigned short sll_protocol; // 物理层的协议 
    int sll_ifindex; //接口号 
    unsigned short sll_hatype; // 报头类型 
    unsigned char sll_pkttype; // 分组类型 
    unsigned char sll_halen; // 地址长度 
    unsigned char sll_addr[8]; // 物理层地址 
};
eg:
*     struct sockaddr_ll peer_addr;  
*    memset(&peer_addr, 0, sizeof(peer_addr));    
        peer_addr.sll_family = AF_PACKET;    
        struct ifreq req;  
    bzero(&req, sizeof(struct ifreq));  
        strcpy(req.ifr_name, "eth0");    
        if(ioctl(sockfd, SIOCGIFINDEX, &req) != 0)  
        perror("ioctl()");    
        peer_addr.sll_ifindex = req.ifr_ifindex;    
        peer_addr.sll_protocol = htons(ETH_P_ARP);  

*/
/*第四步，创建套结字的时候，有以下的组合：更多见：man packet
 * 利用AF_PACKET 套接字发送一个任意的以太网帧。针对linux.
 * 第二个参数： 2）套接字类型：
          SOCK_DGRAM----以太网头已经构造好了
          SOCK_RAW------自己构造以太头 
          * 一种为SOCK_RAW,它是包含了MAC层头部信息的原始分组，当然这种类型的套接字
          * 在发送的时候需要自己加上一个MAC头部（其类型定义在linux/if_ether.h中，ethhdr），
          * 另一种是SOCK_DGRAM类型，它是已经进行了MAC层头部处理的，即收上的帧已经去掉了头部，
          * 而发送时也无须用户添加头部字段。
   第三个参数表示协议类型:链路层承载的协议IP ,ARP,RARP
   * 以太网帧中的以太网类型指定了它包含的负载的类型。有很多途径可以获取以太网类型：
   1）头文件Linux/if_ether.h 定义了大多数常用的以太网类型。包括以太网协议的ETH_P_IP(0x8000)、arp的ETH_P_ARP(0x0806)
  和IEEE 802.1Q VLAN tags的ETH_P_8021Q(0x8100)
    2)IEEE维护的注册以太网类型列表
     3）半官方的列表由IANA维护
      ETH_P_ALL允许任何在没有使用多个套接字的情况下接受所有以太网类型的报文。
      0x88b5和0x88b6是保留以太网类型，供实验或私人使用。 
      */
   /*第五步发送和接收套接字的函数可以是read write操作套结字，也可以是上文的UDP无连接用的函数*/
```

处理的时候可能需要获取网关的ip地址：
```cpp
/*proc方法获取网关地址*/
void GetGateWayIP(uint8 *ip_addr)
{
      char inf[100];
      FILE *file_fd;
      uint8 high=0,low=0,value;
      int i;
      file_fd = fopen("/proc/net/route","r");
      if(file_fd==NULL)
      {
            printf("can not open /proc/net/route\n");
      }
      else
     {
             while(!feof(file_fd))
             {
                     memset(inf,0,sizeof(inf));
                     fgets(inf,100,file_fd);
                     if(inf[5]=='0'&&inf[6]=='0'&&inf[7]=='0'&&inf[8]=='0'&&inf[9]=='0'&&inf[10]=='0'&&inf[11]=='0'&&inf[12]=='0')
                     {
                              for(i=20;i>=14;i-=2)
                              {
                                         if(inf[i]>=65)
                                                 high = inf[i]-55;
                                         else
                                                 high = inf[i]-48;
                                         if(inf[i+1]>=65)
                                                 low = inf[i+1]-55;
                                         else
                                                 low = inf[i+1]-48;
                                        value = high*16+low;
                                       ip_addr[10-i/2] = value;
                              }
                              break;
                       }
              }
      }
}```
#### ARP攻击
+ ARP包可以直接发送给对端不经过路由器，有趣
在一个局域网中，客户端的几乎所有的包都是发往路由器的，这归结于默认路由设置，而在无线网，网卡其实可以接收所有数据，毕竟空气介质，所以，局域网中的设备可以直接通信  
实验：在文件全能王中开启wifi文件传输，和它在同一个局域网的sta能访问该地址，抓包，发现目的地址和源地址都为同一个局域网的不同ip,且不为路由器网关ip，证明其两个设备是可以直接通信的
+ 再来做一个实验：通过pf_packet  
1、在电脑A上写一个程序，mac目的地址为B,源地址为A,协议类型随机，数据随意，发送给B  
   同时在B端进行接收，过滤掉其他mac地址，看是否可以接收到，在连接ap和不连接ap的条件下  
2、在A和B设置静态ip地址，且不同网段，都不连接AP ,让A向B 发送ARP请求包，B接收到后回复arp响应包，接着做ping或其他操作，看是否能互相通信
+ 编程可以参考上述程序，改一下地址什么的就可以，或者参考其他：
http://www.freebuf.com/articles/system/5157.html
这篇文章讲的很具体，个人也喜欢这个网站，emm,实验过了，还可以，can also make your own program,具体程序不贴了  
+ arp欺骗的基本原理：   
在局域网中，设备A,B连接同一个路由器，他们之间可以直接相互通信，因此可以直接向对方发送arp包；   
设备A ,B都会定期和AP交互arp请求和回复，相互知道对方的mac地址，这样，就能给ap发送请求，ap也能知道回复给谁；    
arp欺骗的基本原理在于，如B是攻击方，则B向A发送ARP回复自己是AP ,即在ARP回复包中带上自己的mac地址，ap的ip地址；
同时向AP回复自己是A，即带上A的ip地址，自己的mac地址；   
这样A就会以为B是AP，就会把请求发给B ,同时AP以为B是A，会把回复传给B,但是同时得配置B的转发功能，  
echo 1 >> /proc/sys/net/ipv4/ip_forwardroot  
该博文中有一处错误，见评论
