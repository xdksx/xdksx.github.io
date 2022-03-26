---
title: linux_netdevicelayer
date: 2021-05-22 18:59:09
tags: tcpip_device
categories: tcpip
---


### 目标：
#### 描述linux网络协议栈结构：
1) 常规的方式：传统的单机网络结构：
   一个服务器(server)上，运行一个linux系统，linux系统之上运行一个协议栈，支持相关上层应用；  
   服务器的下方(硬件设备)，接一个或多个网卡，代表这个系统可能支持多个ip，多个出口等；每个网卡NIC接不同的交换机(路由器),来连接到可能不同的  
   运营商物理网络，如下图：在这种情况下一个服务器为一个单点的物理机；<!-- more -->

2) 虚拟机虚拟网络架构：
   一个服务器，其实可能会是个多核比如32核的cpu,运算能力强，也配了一个或多个网卡，可以运行多个虚拟机(操作系统)，像vmware,virtualbox,kvm,qemu  
   等软件支持的虚拟机，可以运行多个不同操作系统的虚拟机，各个虚拟机之间相互隔离；  
   这个虚拟机架构需要网络架构上支持，称虚拟网络架构，每个虚拟机有各自的虚拟网卡，每个虚拟机之间的通信，通过将每个虚拟网卡连接到多个虚拟交换机上(虚拟交换机也是在这个服务器上),从而分成几个隔离的虚拟网络；最后需要经过物理网络出口入口时，由虚拟交换机接出；  
   虚拟机架构：Hypervisor: 一种模拟器，常见的实现有vmward,virtualbox,qemu,kvm,半虚拟化的virtio等等  
   虚拟交换机架构： open vSwitch  
   虚拟网卡： 有tap/tun的实现例子  
   虚拟lan: VLANS, macvlan,ipvlan等
   硬件加速：intel虚拟化技术： VT-d  

3) openstack：
   可以说是云计算的开源架构吧，它可以实现为一套软件，如目前的laaS(Infrastructure as a Service基础设施即服务)/Paas/SaaS  
   基础设施资源，主要包括三个方面：计算、存储、网络。说通俗点，就是CPU，硬盘，网卡。  
   OpenStack对资源进行管理，并且以服务的形式提供给上层应用或者用户去使用。  
   laaS: 厂商管理网络，存储，服务器，虚拟化，即厂商提供docker或其他的类型的虚拟云主机(容器云等)，由使用者用户管理虚拟机上的操作系统，中间件，运行时环境，数据以及应用程序  
   PaaS: 厂商管理网络，存储，服务器，虚拟化，操作系统，中间件和运行时环境，而用户只需要部署其数据和应用程序就好了；  
   SaaS:全部都由厂商来管理；

#### 这里先不讨论虚拟化的网络架构，先看传统的结构：
##### 1 描述这一层的位置，具体含义。
网络设备分为物理网络设备和虚拟网络设备，这里将网络设备都简化为只讨论网卡；
+ 对网卡而言：有它的特性：
带宽，速度： 即它是千兆网卡还是其他，这个可以通过指令拿到：

```cpp
sudo ethtool eth0
Settings for eth0:
	Supported ports: [ TP ]
	Supported link modes:   10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	Supported pause frame use: Symmetric
	Supports auto-negotiation: Yes
	Advertised link modes:  10baseT/Half 10baseT/Full 
	                        100baseT/Half 100baseT/Full 
	                        1000baseT/Full 
	Advertised pause frame use: Symmetric
	Advertised auto-negotiation: Yes
	Speed: 1000Mb/s  千兆网卡
	Duplex: Full
	Port: Twisted Pair
	PHYAD: 1
	Transceiver: internal
	Auto-negotiation: on
	MDI-X: on
	Supports Wake-on: pumbg
	Wake-on: g
	Current message level: 0x00000007 (7)
			       drv probe link
	Link detected: yes
```

其他特性： 网卡支持的工作模式，是否支持多队列等等；
其中多队列的特性比较重要，即意味着它支不支持RSS:
+ 关于RSS,RPS,RFS等：
+ RSS：
```c
    支持多队列的NIC的驱动程序通常提供一个内核模块参数来指定要配置的硬件队列的数量。例如，在bnx2x驱动程序中，这个参数被称为num_queues。典型的RSS配置是，如果设备支持足够的队列，那么每个CPU都有一个接收队列，或者至少每个内存域有一个接收队列，其中内存域是一组共享特定内存级别(L1、L2、NUMA节点等)的CPU。
    RSS设备的间接表(通过屏蔽散列解析队列)通常是由驱动程序在初始化时编写的。默认的映射是将队列均匀地分布在表中，但是可以在运行时使用ethtool命令(——show-rxfh-indir和——set-rxfh-indir)检索和修改间接表。可以通过修改间接表来为不同的队列指定不同的相对权重。
    RSS irq 配置
    每个接收队列都有一个与之相关联的独立IRQ。当新包到达给定队列时，NIC会触发此操作通知CPU。PCIe设备的信令路径使用消息信号中断(MSI-X)，它可以将每个中断路由到特定的CPU。队列到irq的活动映射可以从/proc/interrupts中确定。默认情况下，IRQ可以在任何CPU上处理。因为数据包处理中不可忽略的一部分发生在接收中断处理中，所以在cpu之间分散接收中断是有利的。手动调整每个interru的IRQ亲和力

    推荐配置，即中断打散：
    当考虑延迟或接收中断处理形成瓶颈时，应该启用RSS。在cpu之间分散负载会减少队列长度。对于低延迟网络，最佳设置是分配与系统中cpu数量相同的队列(如果更低，则为网卡最大值)。最有效的高速率配置可能是接收队列数量最少的配置，其中没有由于CPU饱和而导致的接收队列溢出，因为在启用中断合并的默认模式下，中断的聚合数量(因此工作)会随着每增加一个而增加
    可以使用mpstat实用程序观察每个CPU的负载，但是请注意，在具有超线程(HT)的处理器上，每个超线程都表示为一个单独的CPU。对于中断处理，HT在最初的测试中没有显示出任何好处，因此将队列的数量限制在系统中的CPU内核的数量。
```

内核代码实现中，将支持此种特性时，用MSI-X来表示，可以看相关函数名；
查看网卡队列：
```c
ethtool -l eth1
Channel parameters for eth1:
Pre-set maximums:
RX:		0
TX:		0
Other:		1
Combined:	32  //网卡支持的
Current hardware settings:
RX:		0
TX:		0
Other:		1
Combined:	20 //当前配置的
```

+ RPS：
对于单队列网卡，可以通过RPS (receive packet steering)或RFS (receive flow steering)两种方式模拟多队列模式，但效果不如启用RSS的多队列网卡
接收包转向(Receive packet steering, RPS)在cpu之间平衡软中断的负载.  网卡驱动程序通过使用一个四联体(SIP、SPORT、DIP和DPORT)计算每个流的哈希ID，中断处理程序将哈希ID分配给相应的CPU，从而充分利用多核能力
RPS通常是通过软件来模拟多队列网卡的功能。当网卡支持多队列时，RPS无效。RPS主要用于多cpu环境中的单队列网卡。如果一个网卡支持多个队列，可以通过配置SMP IRQ亲和性直接将硬中断绑定到cpu。

+ RFS:
RPS只是将数据包分配到不同的cpu上。当使用不同的CPU运行应用程序和处理软中断时，这会大大降低CPU缓存的利用率。在这种情况下，RFS确保使用一个CPU来运行应用程序和处理软中断，以充分利用CPU缓存。RPS和RFS通常一起使用以获得最好的结果。它们主要用于多cpu环境中的单队列网卡。
即RFS和应用绑定一起，确保该cpu上收到的包，就是被该cpu上运行的应用处理的，仔细看下面两个图，前面是RPS，data3->app8了，无关联，但是后面的是RFS，data3->app3，都是3

接收流导向(Receive flow steering, RFS)与RPS一起，将数据包插入指定CPU的backlog队列中，并唤醒CPU执行。
IRQbalance适用于大多数场景。但在对网络性能要求较高的场景下，建议手动绑定中断。
IRQbalance在运行过程中可能会出现以下问题:a)计算值有时不合适，无法实现cpu之间的负载均衡。(b)当系统处于空闲状态，irq处于省电模式时，IRQbalance将所有中断分配到第一个CPU，使其他空闲CPU休眠，降低能耗。当负载突然上升时，可能会由于调整滞后而导致性能下降。(c)指定处理中断的CPU频繁变化，导致更多的上下文切换。(d)启用IRQbalance但不生效，即没有指定处理中断的CPU。

more:https://01.org/linuxgraphics/gfx-docs/drm/networking/scaling.html


##### 2 关于网络设备层：网络收发时的总体流程，以及这一层的流程；并贴上大致的函数调用接口
Linux内核中有一个网络设备管理层，处于网络设备驱动和协议栈之间，负责衔接它们之间的数据交互。驱动不需要了解协议栈的细节，协议栈也不需要了解设备驱动的细节。
对于一个网络设备来说，就像一个管道（pipe）一样，有两端，从其中任意一端收到的数据将从另一端发送出去。
比如一个物理网卡eth0，它的两端分别是内核协议栈（通过内核网络设备管理模块间接的通信）和外面的物理网络，从物理网络收到的数据，会转发给内核协议栈，而应用程序从协议栈发过来的数据将会通过物理网络发送出去。

+ 网络设备层收包流程：
```c

                   +-----+
                   |     |                            Memroy
+--------+   1     |     |  2  DMA     +--------+--------+--------+--------+
| Packet |-------->| NIC |------------>| Packet | Packet | Packet | ...... |
+--------+         |     |             +--------+--------+--------+--------+
                   |     |<--------+
                   +-----+         |
                      |            +---------------+
                      |                            |
                    3 | Raise IRQ                  | Disable IRQ
                      |                          5 |
                      |                            |
                      ↓                            |
                   +-----+                   +------------+
                   |     |  Run IRQ handler  |            |
                   | CPU |------------------>| NIC Driver |
                   |     |       4           |            |
                   +-----+                   +------------+
                                                   |
                                                6  | Raise soft IRQ
                                                   |
                                                   ↓
//来源：https://segmentfault.com/a/1190000008836467
```
+ 具体步骤
0) 网卡初始化和启动：在开机时module_init初始化部分，然后在pci或其他总线发现设备时，调用probe进一步真正初始化；在类似ifconfig up的指令启动网卡后，
会调用到网卡的xxx_open函数，进行如中断注册，使能等操作；这个时候，设备就可以开始工作了；  

1) 数据包从外面的网络进入物理网卡。这个时候网卡芯片自身的逻辑(固件),会判断如果目的mac地址不是该网卡的，且该网卡没有开启混杂模式，该包会被网卡丢弃，这里发生在网卡固件逻辑上，没在驱动做处理。先代网卡分以太网卡，和wifi网卡，又根据接口不同，有pci接口的，和usb接口的，比如常见的usbwifi网卡；
   以太网的物理头和wifi物理头(802.11头)，不一样，或者说支持的协议不同，根源是物理环境不同；wifi的环境是空气；  

2) 网卡将数据包通过DMA的方式写入到指定的内存地址，该地址由网卡驱动分配并初始化。注： 老的网卡可能不支持DMA，不过新的网卡一般都支持。  
3) 网卡通过硬件中断（IRQ）通知CPU，告诉它有数据来了  
4）CPU根据中断表，调用已经注册的中断函数，这个中断函数会调到驱动程序（NIC Driver）中相应的函数  
5) 对NAPI模式： 驱动先禁用网卡的中断，表示驱动程序已经知道内存中有数据了，告诉网卡下次再收到数据包直接写内存就可以了，不要再通知CPU了，这样可以提高效率，避免CPU不停的被中断。  
6) 对非NAPI的模式，驱动是每次中断处理接收数据，然后写到内存的；这里NAPI的N是new的意思；  
7) 启动软中断。这步结束后，硬件中断处理函数就结束返回了。由于硬中断处理程序执行的过程中不能被中断，所以如果它执行时间过长，会导致CPU没法响应其它硬件的中断，于是内核引入软中断，这样可以将硬中断处理函数中耗时的部分移到软中断处理函数里面来慢慢处理。  

```c
                                                     +-----+
                                             17      |     |
                                        +----------->| NIC |
                                        |            |     |
                                        |Enable IRQ  +-----+
                                        |
                                        |
                                  +------------+                                      Memroy
                                  |            |        Read           +--------+--------+--------+--------+
                 +--------------->| NIC Driver |<--------------------- | Packet | Packet | Packet | ...... |
                 |                |            |          9            +--------+--------+--------+--------+
                 |                +------------+
                 |                      |    |        skb
            Poll | 8      Raise softIRQ | 6  +-----------------+
                 |                      |             10       |
                 |                      ↓                      ↓
         +---------------+  Call  +-----------+        +------------------+        +--------------------+  12  +---------------------+
         | net_rx_action |<-------| ksoftirqd |        | napi_gro_receive |------->| enqueue_to_backlog |----->| CPU input_pkt_queue |
         +---------------+   7    +-----------+        +------------------+   11   +--------------------+      +---------------------+
                                                               |                                                      | 13
                                                            14 |        + - - - - - - - - - - - - - - - - - - - - - - +
                                                               ↓        ↓
                                                    +--------------------------+    15      +------------------------+
                                                    | __netif_receive_skb_core |----------->| packet taps(AF_PACKET) |
                                                    +--------------------------+            +------------------------+
                                                               |
                                                               | 16
                                                               ↓
                                                      +-----------------+
                                                      | protocol layers |
                                                      +-----------------+
```

8) 内核中的ksoftirqd进程专门负责软中断的处理，当它收到软中断后，就会调用相应软中断所对应的处理函数，对于上面第7步中是网卡驱动模块抛出的软中断，ksoftirqd会调用网络模块的net_rx_action函数,这个函数是在初始化时，注册到指定的网络接收软中断上的；  
9) net_rx_action调用网卡驱动里的poll函数来一个一个的处理数据包  
10) 在poll函数中，驱动会一个接一个的读取网卡写到内存中的数据包，内存中数据包的格式只有驱动知道  
11) 驱动程序将内存中的数据包转换成内核网络模块能识别的skb格式，然后调用napi_gro_receive函数  
12) napi_gro_receive会处理GRO相关的内容，也就是将可以合并的数据包进行合并，这样就只需要调用一次协议栈。然后判断是否开启了RPS，如果开启了，将会调用enqueue_to_backlog  
13) 在enqueue_to_backlog函数中，会将数据包放入CPU的softnet_data结构体的input_pkt_queue中，然后返回，如果input_pkt_queue满了的话，该数据包将会被丢弃，queue的大小可以通过net.core.netdev_max_backlog来配置  
14) CPU会接着在自己的软中断上下文中处理自己input_pkt_queue里的网络数据（调用__netif_receive_skb_core）  
15） 如果没开启RPS，napi_gro_receive会直接调用__netif_receive_skb_core  
16)  看是不是有AF_PACKET类型的socket（也就是我们常说的原始套接字），如果有的话，拷贝一份数据给它。tcpdump抓包就是抓的这里的包。  
17) 调用协议栈相应的函数，将数据包交给协议栈处理。  
18) 待内存中的所有数据包被处理完成后（即poll函数执行完成），启用网卡的硬中断，这样下次网卡再收到数据的时候就会通知CPU
接着传递给协议栈；  

+ NAPI和非NAPI的区别：以下两篇写得不错，我就不重复写了；
http://www.hyuuhit.com/2018/07/25/receive-packet/
http://cxd2014.github.io/2017/10/15/linux-napi/


##### 网络设备层发包流程：
网络数据包通过协议栈后，经过邻居子系统，接着都会调用dev_queue_xmit 进入网络设备层进行发包；
期间经过流量控制系统tc,再到发包软中断等，最后在合适的时机，通过设备对应的驱动的发送函数发送出去；
整体流程如下：

```c
                          |
                          |
                          ↓
                   +----------------+
  +----------------| dev_queue_xmit |
  |                +----------------+
  |                       |
  |                       |
  |                       ↓
  |              +-----------------+
  |              | Traffic Control |
  |              +-----------------+
  | loopback              |
  |   or                  +--------------------------------------------------------------+
  | IP tunnels            ↓                                                              |
  |                       ↓                                                              |
  |            +---------------------+  Failed   +----------------------+         +---------------+
  +----------->| dev_hard_start_xmit |---------->| raise NET_TX_SOFTIRQ |- - - - >| net_tx_action |
               +---------------------+           +----------------------+         +---------------+
                          |
                          +----------------------------------+
                          |                                  |
                          ↓                                  ↓
                  +----------------+              +------------------------+
                  | ndo_start_xmit |              | packet taps(AF_PACKET) |
                  +----------------+              +------------------------+
```
explanation:
```cpp
dev_queue_xmit： netdevice子系统的入口函数，在该函数中，会先获取设备对应的qdisc，如果没有的话（如loopback或者IP tunnels），就直接调用dev_hard_start_xmit，否则数据包将经过Traffic Control模块进行处理
Traffic Control： 这里主要是进行一些过滤和优先级处理，在这里，如果队列满了的话，数据包会被丢掉，详情请参考文档，这步完成后也会走到dev_hard_start_xmit

dev_hard_start_xmit： 该函数中，首先是拷贝一份skb给“packet taps”，tcpdump就是从这里得到数据的，然后调用ndo_start_xmit。如果dev_hard_start_xmit返回错误的话（大部分情况可能是NETDEV_TX_BUSY），调用它的函数会把skb放到一个地方，然后抛出软中断NET_TX_SOFTIRQ，交给软中断处理程序net_tx_action稍后重试（如果是loopback或者IP tunnels的话，失败后不会有重试的逻辑）

ndo_start_xmit： 这是一个函数指针，会指向具体驱动发送数据的函数
```

#####  ethernet网卡，e1000,igb i350为例子解释
以简单的只有单队列的千兆网卡e1000为例：e1000是
e1000_main.c
+ pci_scan->probe: 网卡初始化：
```c
e1000_probe:
    结构： 
	struct net_device *netdev; //每个网络设备都有
	struct e1000_adapter *adapter;//网卡适配器结构，每个网卡有自己的adapter结构，包括napi结构，是否为多队列结构，tx,rx相关属性等；
	struct e1000_hw *hw;//表示一些硬件属性
    流程：主要是初始化以上三个结构
    /*e1000_probe initializes an adapter identified by a pci_dev structure.
 * The OS initialization, configuring of the adapter private structure,
 * and a hardware reset occur.*/
    	netif_napi_add(netdev, &adapter->napi, e1000_clean, 64); --初始化poll函数，为e1000_clean
```

+ 启动网卡：ifconfig up/call open func 
```c
 /* 启动设备，分配相关结构，向操作系统注册中断，看门狗启动，协议栈被通知接口已ready
 The open entry point is called when a network interface is made
 * active by the system (IFF_UP).  At this point all resources needed
 * for transmit and receive operations are allocated, the interrupt
 * handler is registered with the OS, the watchdog task is started,
 * and the stack is notified that the interface is ready.
 **//
```

函数不长，可以看看：

```cpp
int e1000_open(struct net_device *netdev)
{
	struct e1000_adapter *adapter = netdev_priv(netdev);
	struct e1000_hw *hw = &adapter->hw;
	int err;

	/* disallow open during test */
	if (test_bit(__E1000_TESTING, &adapter->flags))
		return -EBUSY;

	netif_carrier_off(netdev);

	/* allocate transmit descriptors */
	err = e1000_setup_all_tx_resources(adapter);
	if (err)
		goto err_setup_tx;

	/* allocate receive descriptors */
	err = e1000_setup_all_rx_resources(adapter);
	if (err)
		goto err_setup_rx;

	e1000_power_up_phy(adapter);

	adapter->mng_vlan_id = E1000_MNG_VLAN_NONE;
	if ((hw->mng_cookie.status &
			  E1000_MNG_DHCP_COOKIE_STATUS_VLAN_SUPPORT)) {
		e1000_update_mng_vlan(adapter);
	}

	/* before we allocate an interrupt, we must be ready to handle it.
	 * Setting DEBUG_SHIRQ in the kernel makes it fire an interrupt
	 * as soon as we call pci_request_irq, so we have to setup our
	 * clean_rx handler before we do so.
	 */
	e1000_configure(adapter);

	err = e1000_request_irq(adapter);
	if (err)
		goto err_req_irq;

	/* From here on the code is the same as e1000_up() */
	clear_bit(__E1000_DOWN, &adapter->flags);

	napi_enable(&adapter->napi);

	e1000_irq_enable(adapter);

	netif_start_queue(netdev);

	/* fire a link status change interrupt to start the watchdog */
	ew32(ICS, E1000_ICS_LSC);

	return E1000_SUCCESS;

err_req_irq:
	e1000_power_down_phy(adapter);
	e1000_free_all_rx_resources(adapter);
err_setup_rx:
	e1000_free_all_tx_resources(adapter);
err_setup_tx:
	e1000_reset(adapter);

	return err;
}
//从这里可以看到中断处理程序：e1000_intr
	irq_handler_t handler = e1000_intr;
	int irq_flags = IRQF_SHARED;
	int err;

	err = request_irq(adapter->pdev->irq, handler, irq_flags, netdev->name,
			  netdev);
```
+ 接收数据包的中断处理：

```c
	__napi_schedule(&adapter->napi); //这里是napi的，直接进入调度--->__raise_softirq_irqoff(NET_RX_SOFTIRQ);
	static void net_rx_action(struct softirq_action *h)
	-->budget -= napi_poll(n, &repoll);从设备拉数据
	  --->work = n->poll(n, weight);从设备拉数据
	    -->从上面可以看到poll为e1000_clean
		 -->napi_complete_done   ---dev.c: 通用设备无关函数
		  -->napi_gro_flush==>napi_gro_complete--> 处理gro
		   -->netif_receive_skb_internal --如果打开了RPS,放到对应的cpu队列：enqueue_to_backlog
		     -->__netif_receive_skb --传递到协议栈
```

+ 发送：
先看看定义的可供上层调用的函数接口：

```cpp
static const struct net_device_ops e1000_netdev_ops = {
	.ndo_open		= e1000_open,
	.ndo_stop		= e1000_close,
	.ndo_start_xmit		= e1000_xmit_frame,
	.ndo_get_stats		= e1000_get_stats,
	.ndo_set_rx_mode	= e1000_set_rx_mode,//设置接收模式，混杂模式或其他
	.ndo_set_mac_address	= e1000_set_mac,//设置mac地址
	.ndo_tx_timeout		= e1000_tx_timeout,
	.ndo_change_mtu		= e1000_change_mtu,
	.ndo_do_ioctl		= e1000_ioctl,
	.ndo_validate_addr	= eth_validate_addr,
	.ndo_vlan_rx_add_vid	= e1000_vlan_rx_add_vid,
	.ndo_vlan_rx_kill_vid	= e1000_vlan_rx_kill_vid,
#ifdef CONFIG_NET_POLL_CONTROLLER
	.ndo_poll_controller	= e1000_netpoll,
#endif
	.ndo_fix_features	= e1000_fix_features,
	.ndo_set_features	= e1000_set_features,
};

```

可以看到发送函数：e1000_xmit_frame
具体代码流程涉及较多设备相关的函数，暂不分析；


###### wifi网卡 RTL8180解释
wifi比较特殊，它有比较复杂的扫描和连接四次握手的流程，这块流程不属于数据包的处理，而是控制包的处理，linux内核将它交给了另一套流程，而上层应用用
wpa_supplicant来处理；

1) rtl8187 网卡驱动，是一个usb设备的；
它的一些相关函数定义在 drivers/...rtl8187/dev.c下
看下它支持的操作函数ops
```cpp
static const struct ieee80211_ops rtl8187_ops = {
	.tx			= rtl8187_tx,
	.start			= rtl8187_start,
	.stop			= rtl8187_stop,
	.add_interface		= rtl8187_add_interface,
	.remove_interface	= rtl8187_remove_interface,
	.config			= rtl8187_config,
	.bss_info_changed	= rtl8187_bss_info_changed,
	.prepare_multicast	= rtl8187_prepare_multicast,
	.configure_filter	= rtl8187_configure_filter,
	.conf_tx		= rtl8187_conf_tx,
	.rfkill_poll		= rtl8187_rfkill_poll,
	.get_tsf		= rtl8187_get_tsf,
};
```
start函数初始化了usb驱动需要的东西，并设置了接收数据的回调函数，可以理解为类似接收中断处理函数：

```cpp
rtl8187_start
    rtl8187_init_urbs(dev);
        while (skb_queue_len(&priv->rx_queue) < 32) {
		skb = __dev_alloc_skb(RTL8187_MAX_RX, GFP_KERNEL);
		if (!skb) {
			ret = -ENOMEM;
			goto err;
		}
		entry = usb_alloc_urb(0, GFP_KERNEL);
		if (!entry) {
			ret = -ENOMEM;
			goto err;
		}
		usb_fill_bulk_urb(entry, priv->udev,
				  usb_rcvbulkpipe(priv->udev,
				  priv->is_rtl8187b ? 3 : 1),
				  skb_tail_pointer(skb),
				  RTL8187_MAX_RX, rtl8187_rx_cb, skb);//接收的回调，这里接收到从usb输入的数据，可能是网络数据
		info = (struct rtl8187_rx_info *)skb->cb;
		info->urb = entry;
		info->dev = dev;
		skb_queue_tail(&priv->rx_queue, skb);
		usb_anchor_urb(entry, &priv->anchored);
		ret = usb_submit_urb(entry, GFP_KERNEL);//提交后，usb设备核心在数据到达后，会调用回调函数，rx_cb
		usb_put_urb(entry);
		if (ret) {
			skb_unlink(skb, &priv->rx_queue);
			usb_unanchor_urb(entry);
			goto err;
		}
	}
	return ret;
接收数据：

rtl8187_rx_cb 收到包后：
          skb = dev_alloc_skb(RTL8187_MAX_RX);//分配skb
          ieee80211_rx_irqsafe(dev, skb);
          tasklet_schedule(&local->tasklet);
这里用的是tasklet来处理下半部：
而这个是在ieee80211层处理的，在初始化这个模块时：
struct ieee80211_hw *ieee80211_alloc_hw_nm(size_t priv_data_len,
    初始化：赋值软中断中半部
tasklet_init(&local->tx_pending_tasklet, ieee80211_tx_pending,
		     (unsigned long)local);

	tasklet_init(&local->tasklet,
		     ieee80211_tasklet_handler,
		     (unsigned long) local);//初始化了tasklet的handler，这样在usb接收到数据后能进入tasklet任务
void ieee80211_rx_irqsafe(struct ieee80211_hw *hw, struct sk_buff *skb)
{
	struct ieee80211_local *local = hw_to_local(hw);

	BUILD_BUG_ON(sizeof(struct ieee80211_rx_status) > sizeof(skb->cb));

	skb->pkt_type = IEEE80211_RX_MSG;
	skb_queue_tail(&local->skb_queue, skb);
	tasklet_schedule(&local->tasklet);//这里会进入
}
--> 进入到这个处理的handler:
static void ieee80211_tasklet_handler(unsigned long data)
{
	struct ieee80211_local *local = (struct ieee80211_local *) data;
	struct sk_buff *skb;

	while ((skb = skb_dequeue(&local->skb_queue)) ||
	       (skb = skb_dequeue(&local->skb_queue_unreliable))) {
		switch (skb->pkt_type) {
		case IEEE80211_RX_MSG:
			/* Clear skb->pkt_type in order to not confuse kernel
			 * netstack. */
			skb->pkt_type = 0;
			ieee80211_rx(&local->hw, skb);
			break;
		case IEEE80211_TX_STATUS_MSG:
			skb->pkt_type = 0;
			ieee80211_tx_status(&local->hw, skb);
			break;
		default:
			WARN(1, "mac80211: Packet is of unknown type %d\n",
			     skb->pkt_type);
			dev_kfree_skb(skb);
			break;
		}
	}
}
调用流程：
ieee80211_tasklet_handler
    ieee80211_rx
调用流程： 
ieee80211_rx
ieee80211_rx_napi
__ieee80211_rx_handle_packet
ieee80211_prepare_and_rx_handle
ieee80211_invoke_rx_handlers
ieee80211_rx_handlers
ieee80211_rx_handlers_result
ieee80211_rx_cooked_monitor-->netif_receive_skb(skb);
```

##### rtl8180,pci接口的wifi网卡
```cpp
8180有中断处理：也一样；
static irqreturn_t rtl8180_interrupt(int irq, void *dev_id)
ieee80211_rx_irqsafe(dev, skb);

```
ref:
参考资料：
http://landley.net/kdocs/Documentation/DocBook/xhtml-nochunks/80211.html
https://zhuanlan.zhihu.com/p/68425080
https://www.zhihu.com/question/31878199
https://zhuanlan.zhihu.com/p/100616192


#### 虚拟网卡 tun例子
ref:
https://segmentfault.com/a/1190000009249039
实现：http://vtun.sourceforge.net/
ifb也是一个虚拟网卡
和tun一样，ifb也是在数据包来自的地方和去往的地方做文章。对于tun而言，数据包在xmit中发往字符设备，而从字符设备写下来的数据包则在tun网卡上模拟一个rx操作，对于ifb而言，情况和这类似。
       ifb驱动太简单，以至于很短的话就可以将其说清，然后上一幅全景图，最后留下一点如何使用它的技巧，本文就完了。
       ifb驱动模拟一块虚拟网卡，它可以被看作是一个只有TC过滤功能的虚拟网卡，说它只有过滤功能，是因为它并不改变数据包的方向，即对于往外发的数据包被重定向到ifb之后，经过ifb的TC过滤之后，依然是通过重定向之前的网卡发出去，对于一个网卡接收的数据包，被重定向到ifb之后，经过ifb的TC过滤之后，依然被重定向之前的网卡继续进行接收处理，不管是从一块网卡发送数据包还是从一块网卡接收数据包，重定向到ifb之后，都要经过一个经由ifb虚拟网卡的dev_queue_xmit操作。


### 如何调试linux收发内核问题；  
连通性：
step:  
1) 检查外部网络是否ok,比如其他机器是否正常等，本机物理网卡是否ok: 物理接口的连接，led灯,网线是否连接正常，wifi是否已完成扫描连接；  
2) 检查基本的配置，ip,mac地址，网关等配置，这些影响数据的基本输入  
3) 通过ping，上网工具如浏览器等，来判断整体网络连通情况，至此网络连通性检查基本完成，若不行，检查路由表，dns解析情况，尝试用纯ip进行访问；  
4) 如果3)的检查不行，则尝试抓包，这个时候数据不经过协议栈，可以排除协议栈的影响，而考虑驱动，物理网络的影响；可以从相关设备驱动层的配置和特性如gro等考虑；  

收：以igb为例：在通过基本常规上层手段排除不出来问题时，可以从以下方面入手：
step: 介绍数据从设备->用户的基本流程等，并提供排查的基本手段；  
1) 检查驱动程序是否已加载，初始化，start/open: 通过检查ifconfig，ethtool dev的状态，最极端的就是查日志；
2）检查硬件中断，软件中断是否运行正常，硬件中断查看cat /proc/interputs 软中断： /proc/softirqs
3) 查看ksoftirqd守护进程是否运行正常，每个cpu上都会运行一个，用来进行软中断poll数据；
4) 检查网卡多队列是否支持，若不支持，尽量提供RPS，RFS特性的启动；避免cpu高载导致网络收包延迟或丢包；尽量让收包处理分散在多个cpu上；
查看各网络接口的收发包情况：  

```c
cat /proc/net/dev
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
  eth0: 110346752214 597737500    0    2    0     0          0  20963860 990024805984 6066582604    0    0    0     0       0          0
    lo: 428349463836 1579868535    0    0    0     0          0         0 428349463836 1579868535    0    0    0     0       0          0
查看网卡设备驱动和多队列配置等：
ethtool -l eth0
ethtool eth0
ethtool -i eth0
以及调整 具体man ethtool
ethtool是设备驱动程序向上提供的接口，一般会有个单独的文件来实现，如e1000就有e1000_ethtool.c
查硬中断是否为多队列多中断号：
cat /proc/interrupts
            CPU0       CPU1       CPU2       CPU3
   0:         46          0          0          0 IR-IO-APIC-edge      timer
   1:          3          0          0          0 IR-IO-APIC-edge      i8042
  30: 3361234770          0          0          0 IR-IO-APIC-fasteoi   aacraid
  64:          0          0          0          0 DMAR_MSI-edge      dmar0
  65:          1          0          0          0 IR-PCI-MSI-edge      eth0
  66:  863649703          0          0          0 IR-PCI-MSI-edge      eth0-TxRx-0
  67:  986285573          0          0          0 IR-PCI-MSI-edge      eth0-TxRx-1
  68:         45          0          0          0 IR-PCI-MSI-edge      eth0-TxRx-2
  69:        394          0          0          0 IR-PCI-MSI-edge      eth0-TxRx-3
查软中断收包情况：
cat /proc/softirqs
                    CPU0       CPU1       CPU2       CPU3
          HI:          0          0          0          0
       TIMER: 2831512516 1337085411 1103326083 1423923272
      NET_TX:   15774435     779806     733217     749512
      NET_RX: 1671622615 1257853535 2088429526 2674732223
       BLOCK: 1800253852    1466177    1791366     634534
设置cpu亲和性：
eg:  sudo bash -c 'echo 1 > /proc/irq/8/smp_affinity'
设置包到net_rx_action时的budget值，会影响一次从设备中poll包的最大值
sudo sysctl -w net.core.netdev_budget=600
检查和调整GRO
ethtool -k eth0 | grep generic-receive-offload
generic-receive-offload: on

开启和关闭RPS/RFS/..
```


5) 包到达每个cpu的情况：
```c
 /proc/net/softnet_stat:
6dcad223 00000000 00000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

参数解释

```c
Each line of /proc/net/softnet_stat corresponds to a struct softnet_data structure, of which there is 1 per CPU.
The values are separated by a single space and are displayed in hexadecimal

The first value, sd->processed, is the number of network frames processed. This can be more than the total number of network frames received if you are using ethernet bonding. There are cases where the ethernet bonding driver will trigger network data to be re-processed, which would increment the sd->processed count more than once for the same packet.
第一列，是sd->processed,即处理的网络帧数；
 __netif_receive_skb_core
    __this_cpu_inc(softnet_data.processed);


The second value, sd->dropped, is the number of network frames dropped because there was no room on the processing queue. More on this later.
 enqueue_to_backlog(
	  sd->dropped++;



The third value, sd->time_squeeze, is (as we saw) the number of times the net_rx_action loop terminated because the budget was consumed or the time limit was reached, but more work could have been. Increasing the budget as explained earlier can help reduce this.
The next 5 values are always 0.
即在netif_rx_action过程中，异常中断的次数：
		/* If softirq window is exhausted then punt.
		 * Allow this to run for 2 jiffies since which will allow
		 * an average latency of 1.5/HZ.
		 */
		if (unlikely(budget <= 0 ||
			     time_after_eq(jiffies, time_limit))) {
			sd->time_squeeze++;
			break;
		}


The ninth value, sd->cpu_collision, is a count of the number of times a collision occurred when trying to obtain a device lock when transmitting packets. This article is about receive, so this statistic will not be seen below.
发送的时候，cpu异常？ 这个目前都设置为0 了，在最新的版本；

The tenth value, sd->received_rps, is a count of the number of times this CPU has been woken up to process packets via an Inter-processor Interrupt
通过rps方式，即软中断打散的次数：
/* Called from hardirq (IPI) context */
static void rps_trigger_softirq(void *data)
{
	struct softnet_data *sd = data;

	____napi_schedule(sd, &sd->backlog);
	sd->received_rps++;
}
The last value, flow_limit_count, is a count of the number of times the flow limit has been reached. Flow limiting is an optional Receive Packet Steering feature that will be examined shortly.超过flow最大限制的次数
```



6) 数据包到达协议栈，会通过路由子系统，netfilter框架(防火墙)，或者用户自己开发的防火墙模块，包可能在这里被丢掉，需要检查；
```
$ cat /proc/net/snmp
Ip: Forwarding DefaultTTL InReceives InHdrErrors InAddrErrors ForwDatagrams InUnknownProtos InDiscards InDelivers OutRequests OutDiscards OutNoRoutes ReasmTimeout ReasmReqds ReasmOKs ReasmFails FragOKs FragFails FragCreates
Ip: 1 64 25922988125 0 0 15771700 0 0 25898327616 22789396404 12987882 51 1 10129840 2196520 1 0 0 0
统计了几个协议层的数据情况
```
7) 包顺利进入到ip层；
ref:
https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/
收-图：
https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/
发
https://blog.packagecloud.io/eng/2017/02/06/monitoring-tuning-linux-networking-stack-sending-data/



### 关于TSO ,GSO ,GRO等
需要网卡支持
TSO(TCP Segmentation Offload)，是利用网卡对TCP数据包分片，减轻CPU负荷的一种技术，也有人叫 LSO (Large segment offload) ，TSO是针对TCP的，UFO是针对UDP的。如果硬件支持 TSO功能，同时也需要硬件支持的TCP校验计算和分散/聚集 (Scatter Gather) 功能。如果网卡支持TSO/GSO，可以把最多64K大小的TCP payload直接往下传给协议栈，此时IP层也不会进行segmentation，网卡会生成TCP/IP包头和帧头，这样可以offload很多协议栈上的内存操作，节省CPU资源，当然如果都是小包，那么功能基本就没啥用了。

GSO(Generic Segmentation Offload)，GSO是TSO的增强 ，GSO不只针对TCP，对任意协议。比TSO更通用，推迟数据分片直至发送到网卡驱动之前，此时会检查网卡是否支持分片功能（如TSO、UFO）,如果支持直接发送到网卡，如果不支持就进行分片后再发往网卡。

LRO(Large Receive Offload)，通过将接收到的多个TCP数据聚合成一个大的数据包，然后传递给网络协议栈处理，以减少上层协议栈处理 开销，提高系统接收TCP数据包的能力。

GRO(Generic Receive Offload)，跟LRO类似，克服了LRO的一些缺点，更通用。后续的驱动都使用GRO的接口，而不是LRO。
在系统中可以通过ethtool命令来进行查看，如下：

＃ethtool -k eth0

generic-segmentation-offload: on

generic-receive-offload: on

TSO、UFO、GSO是对应网络发送， LRO、GRO是在接收方向上
