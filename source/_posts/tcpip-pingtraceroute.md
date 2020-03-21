---
title: tcpip_pingtraceroute
date: 2018-06-24 22:42:49
tags: tcpip_ip
categories: tcpip
---
### ping and traceroute
#### ping
ping是ICMP中的回显报文类型：
+ ping 对应的icmp，type字段为0/8,code字段为0
+ ICMP回显请求和应答报文格式：<!--more-->

```cpp
类型(0/8）       code(0)       检验和 --4B
标示符（unix系统中为进程pid)   序号   --4B
           选项数据
 ```

+ 最常见，回显时间，得到往返时间：  
通过在icmp数据报文中存放发送请求的时间值来计算往返时间，当应答返回时，用当前时间减去存放在ICMP报文中的时间值，即是往返时间

```cpp
类型(0/8）       code(0)       检验和 --4B
标示符（unix系统中为进程pid)   序号   --4B
           struct timeval tv;         --8B
```

共16Byte,除开ip头
+ 放抓包的图片，ping回显示时间
+ ping程序代码

```cpp
int addr_conv(char *address,struct in_addr *inaddr)
{
	struct hostent *he;
	if(inet_aton(address,inaddr)==1)
		return (1);
	he=gethostbyname(address);
	if(he!=NULL)
	{
		*inaddr=*((struct in_addr *)he->h_addr_list[0]);
		return (1);
	}
	return 0;
}
void send_icmp(int sockfd,sockaddr_in send_addr);
void recv_icmp(int sockfd,sockaddr_in send_addr);
int main(int argc,char **argv)
{
    int sockfd;
	sockfd=socket(AF_INET,SOCK_RAW,IPPROTO_ICMP);
    if(sockfd<0)
 	{
		cout<<"creat socket error"<<endl;
		return 1;
	}
	sockaddr_in send_addr;
	bzero(&send_addr,sizeof(send_addr));
	send_addr.sin_family=AF_INET;
	addr_conv(argv[1],&send_addr.sin_addr);
//send_addr.sin_addr.s_addr=inet_addr("192.168.0.110");
	for(int i=0;i<3;i++)
 	{
		send_icmp(sockfd,send_addr);
		recv_icmp(sockfd,send_addr);
		sleep(1);
	}
	return 0;
}
unsigned short checksum(unsigned short *addr,int len)
{
	int nleft=len;
	int sum=0;
	unsigned short *w=addr;
	unsigned short answer=0;
	while(nleft>1)
	{
		sum+=*w++;
		nleft-=2;
	}
	if(nleft==1)
	{
		*(unsigned char *)(&answer)=*(unsigned char *)w;
		sum+=answer;
	}
	sum=(sum>>16)+(sum&0xffff);
	sum+=(sum>>16);
	answer=~sum;
	//answer=(unsigned short)sum&0xffff;
	return answer;
}
void send_icmp(int sockfd,sockaddr_in send_addr)
{
	static short int seq=10;
	char  buf[8+8];
	struct icmphdr *icmp=(struct icmphdr *)buf;
	//填充icmp首部
	icmp->type=ICMP_ECHO;//类型
	icmp->code=0;//和编码共同决定是回显报文
	icmp->checksum=0;//头部包含校验和
	icmp->un.echo.id=getpid();//标识符，唯一性，一般填充pid，这样即使几个程序发送ping也能识别，写成其他时接收失败
	icmp->un.echo.sequence=seq++;//顺序号，回显时，显示，一般无规定，但是会在此基础上加１，接着加１，，
	//填充icmp数据(时间)//这里报文数据只有时间戳
	struct timeval tv;
	//tv=(struct timeval*)icmp->icmp_data;
	gettimeofday(&tv,NULL);
	memcpy(buf+8,&tv,sizeof(tv));
	int buflen=sizeof(struct icmphdr)+sizeof(struct timeval);
	//计算校验和
	icmp->checksum=checksum((unsigned short *)buf,buflen);
	//发送icmp数据包
	int len=sendto(sockfd,buf,buflen,0,(struct sockaddr *)&send_addr,sizeof(send_addr));
	if(len<0)
		cout<<"send icmp error"<<endl;
	else cout<<"senmd ok"<<endl;
}
void recv_icmp(int sockfd,sockaddr_in send_addr)
{
	char buf[256];
	struct icmphdr *icmp;
	struct ip *ip;
	int ipheadlen;
	int icmplen;
	//接收icmp响应
	for(;;)
	{
		int n=recvfrom(sockfd,buf,sizeof(buf),0,NULL,NULL);
		if(n<0)
		{
			cout<<"recv error"<<endl;
			continue;
		}
		ip=(struct ip *)buf;	
		ipheadlen=ip->ip_hl<<2;
		icmplen=n-ipheadlen;
		if(icmplen<16)
			continue;
		icmp=(struct icmphdr *)(buf+ipheadlen);
		if(icmp->type==ICMP_ECHOREPLY&&icmp->un.echo.id==getpid())
			break;
	}
	//计算时间差
	struct timeval recv_tv;
	gettimeofday(&recv_tv,NULL);
	struct timeval send_tv;
	memcpy(&send_tv,icmp+1,sizeof(send_tv));
	recv_tv.tv_sec-=send_tv.tv_sec;
	recv_tv.tv_usec+=recv_tv.tv_sec*1000000L;
	long interval=recv_tv.tv_usec-send_tv.tv_usec;
	//输出信息
	cout<<icmplen<< " bytes fromfdfd "<<inet_ntoa(send_addr.sin_addr);
	cout<<" icmp_seq="<<icmp->un.echo.sequence<<" bytes="<<icmplen<<" ttl="<<(int)ip->ip_ttl;
	cout<<" time="<<(float)interval/1000.0<<"ms"<<endl;
}
```

+ ip 记录路由选项：  
利用ip头，ip头长度字段为4bit,最长的ip首部为15x32bit=60字节，所以除去头20字节，选项字段最大40字节，除去3字节的code,len.ptr三个字节，剩下37个字节，可以容纳9个ip地址
windows下可以通过ping -r ip来尝试
+ ip时间戳选项

#### traceroute
##### traceroute主要两点
+ 利用ttl,ttl是一个有限值，每经过一个路由器减1，这样放置循环发包。
+ 利用icmp报文，当路由器接收到ttl为0或1时，不转发，而是丢弃，并给信源回复一个icmp超时信息，且包含该路由器的ip
+ 具体流程：  
traceroute发送一份ttl字段为1的ip数据报给目的主机，处理这个数据报的第一个路由器将ttl值减去1，丢弃该数据报，并发送一份超时icmp报文，这样就得到了该路径的第一个路由器的地址；  
以此类推，发送ttl为2的数据报，则得到第二个路由器的地址；直到到达目的地址；  
怎么知道到达了目的地?  
traceroute发送了一份udp数据包给目的主机，但它选择了一个不可能的值作为udp端口号（大于30000)，使目的主机的任何一个应用程序都不可能使用该端口号，则当包到达目的主机时，目的主机的udp模块会产生一个端口不可达的错误icmp报文。traceroute要做的是区分icmp是超时还是端口不可达，以判断什么时候结束

##### traceroute命令

```
traceroute www.baidu.com
traceroute to www.baidu.com (119.75.216.20), 30 hops max, 60 byte packets//ttl字段为30跳，每个数据包为60字节（20ip头等）
 1  192.168.0.1 (192.168.0.1)  2.762 ms  3.485 ms  3.477 ms/发到网关1,针对每个ttl值发送三份包，分别在2.762,3.485,3.477收到
 2  192.168.1.1 (192.168.1.1)  3.466 ms  3.453 ms  3.443 ms
 3  101.232.192.1 (101.232.192.1)  6.807 ms  6.813 ms  7.412 ms
 4  10.144.11.37 (10.144.11.37)  7.405 ms  7.393 ms  7.381 ms
 5  10.144.14.138 (10.144.14.138)  7.369 ms  7.362 ms  7.340 ms
 6  * * 14.197.242.145 (14.197.242.145)  10.329 ms
 7  14.197.218.173 (14.197.218.173)  7.240 ms 14.197.248.253 (14.197.248.253)  6.855 ms  7.212 ms
 8  14.197.240.249 (14.197.240.249)  44.799 ms 14.197.252.189 (14.197.252.189)  42.107 ms 14.197.253.145 (14.197.253.145)  50.051 ms
 9  14.197.252.54 (14.197.252.54)  49.394 ms  49.414 ms 14.197.248.94 (14.197.248.94)  49.406 ms
10  14.197.149.178 (14.197.149.178)  49.406 ms  49.383 ms 14.197.178.102 (14.197.178.102)  49.382 ms
11  182.61.253.119 (182.61.253.119)  49.912 ms 182.61.253.117 (182.61.253.117)  50.916 ms 182.61.253.119 (182.61.253.119)  50.554 ms
12  182.61.253.126 (182.61.253.126)  47.975 ms *  50.625 ms//5s未收到时打印一个*号并发送下一份数据包
13  * * *
14  * * *
15  * * *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * *
27  * * *
28  * * *
29  * * *
30  * * *
```

tcpdump输出：

```
traceroute www.baidu.com(119.75.213.61)
192.168.0.110.39650 > 119.75.213.61.33434: [udp sum ok] UDP, length 32
20:21:58.009075 IP (tos 0x0, ttl 1, id 5354, offset 0, flags [none], proto UDP (17), length 60)//ttl=1
192.168.0.1 > 192.168.0.110: ICMP time exceeded in-transit, length 68//网关回复icmp超时
	IP (tos 0x0, ttl 1, id 941, offset 0, flags [none], proto UDP (17), length 60)
    192.168.0.110.48912 > 119.75.213.61.33435: [udp sum ok] UDP, length 32
20:21:58.009114 IP (tos 0x0, ttl 1, id 5355, offset 0, flags [none], proto UDP (17), length 60)
    192.168.0.110.43061 > 119.75.213.61.33436: [udp sum ok] UDP, length 32
20:21:58.009148 IP (tos 0x0, ttl 2, id 5356, offset 0, flags [none], proto UDP (17), length 60)//ttl=2
 192.168.1.1 > 192.168.0.110: ICMP time exceeded in-transit, length 68
	IP (tos 0x0, ttl 1, id 942, offset 0, flags [none], proto UDP (17), length 60)
//第2个路由器回复超时
    192.168.0.110.52554 > 119.75.213.61.33437: [udp sum ok] UDP, length 32
20:21:58.009189 IP (tos 0x0, ttl 2, id 5357, offset 0, flags [none], proto UDP (17), length 60)
    192.168.0.110.51967 > 119.75.213.61.33438: [udp sum ok] UDP, length 32
20:21:58.009243 IP (tos 0x0, ttl 2, id 5358, offset 0, flags [none], proto UDP (17), length 60)
    192.168.0.110.45922 > 119.75.213.61.33439: [udp sum ok] UDP, length 32
20:21:58.009281 IP (tos 0x0, ttl 3, id 5359, offset 0, flags [none], proto UDP (17), length 60)//ttl==3
    192.168.0.110.34392 > 119.75.213.61.33440: [udp sum ok] UDP, length 32  
    101.232.192.1 > 192.168.0.110: ICMP time exceeded in-transit, length 60
	IP (tos 0x0, id 945, offset 0, flags [none], proto UDP (17), length 60)
    192.168.0.110.57724 > 119.75.216.20.33440: UDP, length 32
//第三个路由器回复icmpc超时
20:21:59.304604 IP (tos 0x0, ttl 59, id 0, offset 0, flags [DF], proto UDP (17), length 161)
```

从上面包的情况可以看到:
+ 设备首先发送ucp包。若用更详细的打印看到端口号为源55050 目的端口:33443别的设备可能不同，但是都很大，这样到达时会发出端口不可达；ttl字段从1递增，源地址目的地址不变
+ 路由器根据ttl的值，回复icmp超时包，回复给设备，从源地址可以看到，经过的路由器；
+ 该icmp的包格式：
类型11  code 0/1  校验和  
ip首部（包括选项)+原始ip数据报中数据的前8个字节
+ 注意：每一次的路由都可能不一样

##### 关于ip源站选路选项
+ ip通常都是动态的，即每个路由都要判断数据报下面该转发到哪个路由器，一般不对此进行控制；这里提供的思想是：
+ 由源站发送者指定路由，即经过哪些ip
+ 分为严格的源路由选择和宽松的源站选路
+ 前者必须采用确切的路由，若发现失败则路由器返回源站选路失败的icmp报文；
+ 后者致命一个数据包经过的ip清单，但是数据包在清单上指明的任意两个地址之间可以通过其他路由器；
+ ip源站路由选项的格式：  
包含在ip头部的选项中，因长度有限只能包含9个ip:  
code(1) len(1) ptr(1) ip1(4) ip2(4)....
+ eg: traceroute -g 192.168.23.1 www.baidu.com

##### traceroute实现
+ 参考linux traceroute源码实现；
+ 主要就是发送一多个dp包设置ttl递增，并将接收到的icmp报文打印出来；直到收到端口不可达报文时退出；