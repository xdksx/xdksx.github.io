---
title: tcpip_PF_PACKET
date: 2018-05-27 20:28:40
tags: PF_PACKET
categories: tcpip
---
{% asset_img pf_packet.jpg it just picture %}
### PF_PACKET的使用：
### PF_PACKET简介：
是linux下的用于发送和接收二层(mac层)的套接字：
<!-- more -->
### PF_PACKET基本使用：
+ 基本的几个操作：  
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
#include <linux/if_ether.h>```

```cpp
//获取硬件网卡的相应信息
void GetEthInfor(char *name ,  char *MAC_addr , struct in_addr * IP_addr)//传入接口名，取回mac和ip
{
   struct ifreq  eth;  //结构用于存放最初获取的接口信息
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
      close(fd);
}
//取得网络接口的索引：int值,传入fd和接口名
int Get_IfaceIndex(int fd, const char* interfaceName)
{
struct ifreq ifr;
if (interfaceName == NULL)
{
   return -1;
}
memset(&ifr, 0, sizeof(ifr));
strcpy(ifr.ifr_name, interfaceName);
if (ioctl(fd, SIOCGIFINDEX, &ifr) == -1)
{
   printf("RED ioctl error\n");
   return -1;
}
return ifr.ifr_ifindex;
}
int set_Iface_promisc(int fd, int dev_id)//传入fd和index
{
struct packet_mreq mr;
memset(&mr,0,sizeof(mr));
mr.mr_ifindex = dev_id;
mr.mr_type = PACKET_MR_PROMISC;
if(setsockopt(fd, SOL_PACKET, PACKET_ADD_MEMBERSHIP,&mr,sizeof(mr))==-1)//退出混杂模式使用这个：PACKET_DROP_MEMBERSHIP
{
   fprintf(stderr,"GREEN set promisc failed! \n");
   return -1;
}
return 0;
}```
使用
```cpp
int main ()
{
	 unsigned char MAC_ADDR[6];
     struct in_addr IP_ADDR;//结构体的定义在/netinet/in.h用来表示一个32位的IPv4地址.
	 //第一步：获取指定网卡的信息（MAC地址和IP地址）
    GetEthInfor("wlp2s0",MAC_ADDR,&IP_ADDR);
    int fd;
     fd=socket(AF_PACKET,SOCK_DGRAM,htons(ETH_P_ARP));
    int index=Get_IfaceIndex(fd,"enp1s0");
    printf("index:%d\n",index);
    return 0;
   }
```

### PF_PACKET的接收：
简单说明：  
创建套结字的时候，有以下的组合：更多见：man packet
 * 利用AF_PACKET 套接字发送一个任意的以太网帧。针对linux.
 * 第二个参数： 套接字类型：
          SOCK_DGRAM----以太网头已经构造好了
          SOCK_RAW------自己构造以太头 
          * 一种为SOCK_RAW,它是包含了MAC层头部信息的原始分组，当然这种类型的套接字
          * 在发送的时候需要自己加上一个MAC头部（其类型定义在linux/if_ether.h中，ethhdr），
          * 另一种是SOCK_DGRAM类型，它是已经进行了MAC层头部处理的，即收上的帧已经去掉了头部，
          * 而发送时也无须用户添加头部字段。
 + 第三个参数表示协议类型:链路层承载的协议IP ,ARP,RARP
   * 以太网帧中的以太网类型指定了它包含的负载的类型。有很多途径可以获取以太网类型：  
   1）头文件Linux/if_ether.h 定义了大多数常用的以太网类型。包括以太网协议的ETH_P_IP(0x8000)、arp的ETH_P_ARP(0x0806)
  和IEEE 802.1Q VLAN tags的ETH_P_8021Q(0x8100)  
  2)  IEEE维护的注册以太网类型列表  
  3）半官方的列表由IANA维护  
      ETH_P_ALL允许任何在没有使用多个套接字的情况下接受所有以太网类型的报文。
      0x88b5和0x88b6是保留以太网类型，供实验或私人使用。   
(若指定socket_type为SOCK_RAW则意味着原始数据包包含了链路层头部，若指定为SOCK_DGRAM则将做一些处理以剥掉链路层头部。链路层头部信息的通用格式是一个sockaddr_ll结构。protocol填入网络字节序的IEEE 802.3协议号。参见<linux/if_ether.h>头文件了解允许的协议。如果protocol被设置为htons(ETH_P_ALL)，则意味着接受所有协议。所有接入的协议类型为protocol的数据包将在传递给内核之前传递到PACKET套接字。)
+ 收包可以使用的接口：
```cpp
  int readnum = recvfrom(rawsock, buffer,2048,0, (struct sockaddr*)(&eth_info),&leneth_info);
  int readnum = read(rawsock, buffer,2048);
  int readnum = recvfrom(rawsock, buffer,2048,0, NULL,NULL);```
        
+ 一个简单的接收包的例子：
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <linux/if_ether.h>
#include <linux/in.h>
#include <stdlib.h>
#include <linux/if_packet.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <linux/if.h>
#include <sys/ioctl.h>
#include <linux/if_packet.h>
#include <linux/if_ether.h>
#define BUFFER_MAX 2048```

```c
int main(int argc, char *argv[])
{
 if((rawsock=socket(PF_PACKET,SOCK_RAW,htons(ETH_P_ALL))) < 0)
   {
        printf("error: create raw socket!!!\n");
        exit(0);
    }
  while(1)
   {
        int readnum = read(rawsock, buffer,2048);// can read packet
        //printf("recv buffer:%s\n",buffer);
    }
    return 0;
   }```
  
+ 指定从某个接口接收数据：
```c
int main(int argc, char *argv[])
{
 struct sockaddr_ll eth_info;//结构体的定义在/linux/if_packet.h中， 表示设备无关的物理层地址结构
  eth_info.sll_family = PF_PACKET;  //PF_PACKET定义在sys/types.h中
  eth_info.sll_ifindex = if_nametoindex("lo");//返回输入的接口名称的索引值　　//次函数定义在net/if.h中
 if((rawsock=socket(PF_PACKET,SOCK_RAW,htons(ETH_P_ALL))) < 0)
   {
        printf("error: create raw socket!!!\n");
        exit(0);
    }
 if(bind(rawsock,(struct sockaddr *)(&eth_info),sizeof(eth_info))==-1)//绑定接口，从而只接收那个接口上的数据
{
       printf("error: bind!!\n");
        exit(0);
}
  while(1)
   {
        int readnum = read(rawsock, buffer,2048);// can read packet
        //printf("recv buffer:%s\n",buffer);
    }
    return 0;
   }```
  
+ 接收后的包如何读取：以包括mac头的形式来看：粗暴的形式
```c
int main(int argc, char *argv[])
{
    int rawsock;
    char buffer[BUFFER_MAX];
    char *ethhead;
    char *iphead;
    char *tcphead;
    char *udphead;
    char *icmphead;
    char *pHead;
 if((rawsock=socket(PF_PACKET,SOCK_RAW,htons(ETH_P_ALL))) < 0)
   {
        printf("error: create raw socket!!!\n");
        exit(0);
    }
  while(1)
   {
        int readnum = read(rawsock, buffer,2048);// can read packet
        //printf("recv buffer:%s\n",buffer);
         if(readnum < 42)
        {
            printf("error: Header is incomplete!!!\n");
            continue;
        }
      //  for(j;j<readnum;j++)
      //     printf("%.2X:",buffer[j]&0xFF);
        ethhead = (char *)buffer;
        pHead = ethhead;
        int ethernetmask = 0XFF;
        framecount++;
        printf("------------------Analysis   Packet [%d]---------------------\n",framecount);
       // printf("all:-----%s\n",ethhead);
        printf("MAC:");
        
        int i = 6;
        for(; i <=11; i++)
        {
            printf("%.2X:",pHead[i]&ethernetmask);
        }
        printf("---->");
        for(i = 0; i <=5; i++)
        {
            printf("%.2X:",pHead[i]&ethernetmask);
        }
        printf("\n");
        printf("proto: %.2x:",pHead[12]&ethernetmask);
        printf("proto2: %.2x:\n",pHead[13]&ethernetmask);        
        iphead = ethhead + 14;
        pHead = iphead + 14;
        printf("IP:");
        for(i = 0; i <=3; i++)
        {
            printf("%d",pHead[i]&ethernetmask);
            if(i != 3)
            printf(".");
        }
        printf("---->");
        for(i = 10; i <=13; i++)
        {
            printf("%d",pHead[i]&ethernetmask);
            if(i != 13)
            printf(".");
        }
        printf("\n");
        int prototype = (iphead + 9)[0];
     //   printf("Protocol: %.2X:",prototype);
        //int prototype = (iphead + 9)[0];
        pHead = iphead + 20;
        printf("Protocol: ");
        switch(prototype)
        {
            case IPPROTO_ICMP:
                printf("ICMP\n");
                break;
            case IPPROTO_IGMP:
                printf("IGMP\n");
                break;
            case IPPROTO_IPIP:
                printf("IP\n");
                break;
            case IPPROTO_TCP :
                printf("TCP | source port: %u | ",(pHead[0]<<8)&0XFF00 | pHead[1]&0XFF);
                printf("dest port: %u\n", (pHead[2]<<8)&0XFF00 | pHead[3]&0XFF);
                break;
            case IPPROTO_UDP :
                printf("UDP | source port: %u | ",(pHead[0]<<8)&0XFF00 | pHead[1]&0XFF);
                printf("dest port: %u\n", (pHead[2]<<8)&0XFF00 | pHead[3]&0XFF);
                break;
            case IPPROTO_RAW :
                printf("RAW\n");
                break;
            default:
                printf("Unkown\n");
        }
      printf("-------------------------end-----------------------\n");
    }
    return 0;
   }```
收包处理的方式，也可以把指针赋给内核的结构：struct iphdr
如：   
```cpp
 struct iphdr ip;
 ip = (struct iphdr *)(buffer + sizeof(struct ethhdr));
 ```
内核提供了很多的包结构，在发包或者收包的时候可以直接使用这些结构解析或者粗暴的用字符串解析

### PF_PACKET发送包： 
+ 发包和接收包类似：  
```cpp
num = sendto(rawsock, buffer,2048 , 0 ,(struct sockaddr*)(&eth_info),sizeof(eth_info));
 if(num<0)
 {
  printf("sendto failed!\n");
  exit(1);
 }
 printf("success:%d\n",num);```