---
title: tcpip_tc
date: 2021-05-22 19:35:21
tags: tcpip_tc
categories: tcpip
---

### 流量控制概述
linux下通过tc traffic control 框架及系列实现和工具来实现对出口，甚至入口流量的控制，所谓的控制，就是进行包延迟传输，
丢包，包损坏，带宽限制，针对某个ip规则进行限制等等，来达到模拟网络异常状况，包优先级传输，或者更多功能；<!-- more -->
从手册上看：主要提供一下几种控制：
SHAPING: 平滑突发流量，如限制传输速率，小于有效带宽，作用于出口
       When traffic is shaped, its rate of transmission is under
       control. Shaping may be more than lowering the available
       bandwidth - it is also used to smooth out bursts in
       traffic for better network behaviour. Shaping occurs on
       egress.
SCHEDULING : 作用于出口，调度数据包的传输，比如优先级等
       By scheduling the transmission of packets it is possible
       to improve interactivity for traffic that needs it while
       still guaranteeing bandwidth to bulk transfers. Reordering
       is also called prioritizing, and happens only on egress.
POLICING： 作用于入口流量
       Whereas shaping deals with transmission of traffic,
       policing pertains to traffic arriving. Policing thus
       occurs on ingress.
DROPPING： 当流量超过阈值，丢弃数据包，作用于入口和出口；
       Traffic exceeding a set bandwidth may also be dropped
       forthwith, both on ingress and on egress.
<!--more-->
#### 例子：
ref:  https://netbeez.net/blog/how-to-use-the-linux-traffic-control/
```cpp
查看：当前只有默认的先入先出规则
think@think-VirtualBox:~$ tc qdisc
qdisc noqueue 0: dev lo root refcnt 2 
qdisc pfifo_fast 0: dev eth0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1

准备设置延迟：
think@think-VirtualBox:~$ ping www.baidu.com
PING www.baidu.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39: icmp_seq=1 ttl=56 time=9.13 ms
64 bytes from 14.215.177.39: icmp_seq=2 ttl=56 time=9.38 ms
64 bytes from 14.215.177.39: icmp_seq=3 ttl=56 time=9.09 ms
64 bytes from 14.215.177.39: icmp_seq=4 ttl=56 time=9.68 ms
^C
--- www.baidu.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 7048ms
rtt min/avg/max/mdev = 9.093/9.324/9.687/0.256 ms

添加延迟规则：
think@think-VirtualBox:~$ sudo tc qdisc add dev eth0 root netem delay 200ms

think@think-VirtualBox:~$ ping www.baidu.com
PING www.baidu.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39: icmp_seq=1 ttl=56 time=209 ms
64 bytes from 14.215.177.39: icmp_seq=2 ttl=56 time=212 ms
64 bytes from 14.215.177.39: icmp_seq=3 ttl=56 time=211 ms

删除规则： sudo tc qdisc del dev eth0 root netem delay 200ms
/ sudo tc qdisc del dev eth0 root 

设置丢包规则：
think@think-VirtualBox:~$ sudo tc qdisc add dev eth0 root netem loss 20%
[sudo] password for think: 
think@think-VirtualBox:~$ ping www.baidu.com -c 100
PING www.baidu.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39: icmp_seq=1 ttl=56 time=9.68 ms
64 bytes from 14.215.177.39: icmp_seq=2 ttl=56 time=8.92 ms
64 bytes from 14.215.177.39: icmp_seq=3 ttl=56 time=8.92 ms
。。。
64 bytes from 14.215.177.39: icmp_seq=100 ttl=56 time=10.1 ms

--- www.baidu.com ping statistics ---
100 packets transmitted, 80 received, 20% packet loss, time 103846ms
rtt min/avg/max/mdev = 8.510/9.537/23.505/1.655 ms
删除规则：
think@think-VirtualBox:~$ sudo tc qdisc del dev eth0 root netem loss 20%

设置带宽限制规则：
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 400ms

tbf: use the token buffer filter to manipulate traffic rates
rate: sustained maximum rate
burst: maximum allowed burst
latency: packets with higher latency get dropped

通过iperf测试：
iperf -c 172.31.0.142
------------------------------------------------------------
Client connecting to 172.31.0.142, TCP port 5001
TCP window size: 85.3 KByte (default)
------------------------------------------------------------
[  3] local 172.31.0.25 port 40233 connected with 172.31.0.142 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0-10.0 sec   113 MBytes  95.0 Mbits/sec ---> 1.1Mbits/sec
```

+ 指令解释：
qdisc: modify the scheduler (aka queuing discipline) 即实际的使用是依赖的qidsc机制
add: add a new rule 添加一个排队规则
dev eth0: rules will be applied on device eth0 排队规则作用对象一般是网卡
root: modify the outbound traffic scheduler (aka known as the egress qdisc) 修改出口流量调度程序
netem: use the network emulator to emulate a WAN property 使用wan网络模拟器
delay: the network property that is modified
200ms: introduce delay of 200 ms

tc是系统如linux提供的用户层操作指令，这里用的是shell指令：
更多  https://man7.org/linux/man-pages/man8/tc.8.html 


### 流量控制的基本实现原理
在linux内核中，流量控制用Qos实现，实际上使用了qdisc队列；主要是出口队列；（egress)
在链路层，每个数据包通过邻居子系统后，或者说离开协议栈后，都会由dev_queue_xmit(dev.c)来进一步调用相关设备驱动的发送函数
来发送出去； 而qdisc队列，和相关的排队规则即作用在dev_queue_xmit之后，设备驱动发送函数之前；


### 流量控制的实现和基本流程：
#### 相关代码：
```cpp
sch_generic.c
sch_generic.h
sch_api.c
pkt_sched.h
net/core/dev.c

```
#### 流程
在内核中的整体处理流程，及位置：
```cpp
net/core/dev.c
int dev_queue_xmit(struct sk_buff *skb) --发送函数
    // 1 从设备中拿到设备下的qdisc结构，每个设备net_device结构都有
  	txq = netdev_pick_tx(dev, skb, accel_priv);
	q = rcu_dereference_bh(txq->qdisc);

	trace_net_dev_queue(skb);
	if (q->enqueue) {
		rc = __dev_xmit_skb(skb, q, dev, txq);
		goto out;
	}
    
	//2 插入根排队规则中，并启动运行：
	rc = q->enqueue(skb, q, &to_free) & NET_XMIT_MASK; //插入根排队规则
		if (qdisc_run_begin(q)) {
			if (unlikely(contended)) {
				spin_unlock(&q->busylock);
				contended = false;
			}
			__qdisc_run(q);//会走netif_schedule来调度数据包输出软中断 net_tx_action,然后在合适的时机，比如延迟200ms，发送数据包；
			在调度前会检查网络设备的启动情况，以及是否已经在调度了；
			void __qdisc_run(struct Qdisc *q)
              {
              	int quota = weight_p;
              	int packets;
              
              	while (qdisc_restart(q, &packets)) {
              		/*
              		 * Ordered by possible occurrence: Postpone processing if
              		 * 1. we've exceeded packet quota
              		 * 2. another process needs the CPU;
              		 */
              		quota -= packets;
              		if (quota <= 0 || need_resched()) {
              			__netif_schedule(q);
              			break;
              		}
              	}
              
              	qdisc_run_end(q);
              }
		}

    dev.c:
	net_tx_action:
	smp_mb__before_atomic();
			clear_bit(__QDISC_STATE_SCHED, &q->state);
			qdisc_run(q);
			spin_unlock(root_lock);

	
	//3 qdisc_run/ __qdisc_run都会调用qdisc_restart，当可以发送时
	//这个restart函数，将包从根排队规则中取出，然后调用设备发送函数发送：
	static inline int qdisc_restart(struct Qdisc *q, int *packets)
    {
    	struct netdev_queue *txq;
    	struct net_device *dev;
    	spinlock_t *root_lock;
    	struct sk_buff *skb;
    	bool validate;
    
    	/* Dequeue packet */
    	skb = dequeue_skb(q, &validate, packets);
    	if (unlikely(!skb))
    		return 0;
    
    	root_lock = qdisc_lock(q);
    	dev = qdisc_dev(q);
    	txq = skb_get_tx_queue(dev, skb);
         //4 发送
    	return sch_direct_xmit(skb, q, dev, txq, root_lock, validate);---->ops->ndo_start_xmit(skb, dev);
    }
    
```

#### 流量控制的结构：
构成流量控制的基本元素有三种： 排队规则，类和过滤器
```

----------排队规则1:0--------------  -->根排队规则，提供两个外部接口，enqueue和dequeue
---过滤器1---过滤器2----过滤器3-----
---|-------------|------|---------
--类1:1---------(类  1:2  )-------
--|-----------------|-------------
--排队规则2:0-----排队规则3:0------- -->内部规则
```

+ 排队规则：
    在启用了流量控制的情况下，每个网络设备至少会配置一个排队规则；排队规则包括简单的fifo缓冲和令牌桶等，而精确的排队规则通常需要管理多个队列；
常见的排队规则由 fifo,令牌桶tbf(token bucket filter)等；
+ 排队规则的分类：
排队规则至少有一个队列，可能简单，如fifo排队规则，也有复杂如令牌桶；通常排队规则分无类和有类两种，无类规则简单，内部不能包含可配置的子类及内部规则
而有类则可包含多个类，如上图，且每个类又可以包含一个排队规则，这里的排队规则叫内部规则，可以是有类和无类的；
无类规则不可被用户配置，而有类的可以；
```
    分为可分类的qdisc和不可分类的qdisc实现：
	不可分类：pfifo ,pfifo_fast,red,sfq,tbf
	可分类：cbq,htb,prio
```
如默认：
```cpp
qdisc pfifo_fast 0: dev eth0 root refcnt 2 bands 3 priomap  1 2 2 2 1 2 0 0 1 1 1 1 1 1 1 1
```
在tc指令中，如下的，其中结尾的 qdisc [qdisc specific parameters] 就是指定具体的排队规则类型；
```cpp
创建qdisc 规则
tc [ OPTIONS ] qdisc [ add | change | replace | link | delete ]
       dev DEV [ parent qdisc-id | root ] [ handle qdisc-id ] [
       ingress_block BLOCK_INDEX ] [ egress_block BLOCK_INDEX ] qdisc [
       qdisc specific parameters ]
eg:

目前支持的qdisc: 无类的：
The classless qdiscs are:

       choke  CHOKe (CHOose and Keep for responsive flows, CHOose and
              Kill for unresponsive flows) is a classless qdisc designed
              to both identify and penalize flows that monopolize the
              queue. CHOKe is a variation of RED, and the configuration
              is similar to RED.

       codel  CoDel (pronounced "coddle") is an adaptive "no-knobs"
              active queue management algorithm (AQM) scheme that was
              developed to address the shortcomings of RED and its
              variants.

       [p|b]fifo
              Simplest usable qdisc, pure First In, First Out behaviour.
              Limited in packets or in bytes.

       fq     Fair Queue Scheduler realises TCP pacing and scales to
              millions of concurrent flows per qdisc.

       fq_codel
              Fair Queuing Controlled Delay is queuing discipline that
              combines Fair Queuing with the CoDel AQM scheme. FQ_Codel
              uses a stochastic model to classify incoming packets into
              different flows and is used to provide a fair share of the
              bandwidth to all the flows using the queue. Each such flow
              is managed by the CoDel queuing discipline. Reordering
              within a flow is avoided since Codel internally uses a
              FIFO queue.

       fq_pie FQ-PIE (Flow Queuing with Proportional Integral controller
              Enhanced) is a queuing discipline that combines Flow
              Queuing with the PIE AQM scheme. FQ-PIE uses a Jenkins
              hash function to classify incoming packets into different
              flows and is used to provide a fair share of the bandwidth
              to all the flows using the qdisc. Each such flow is
              managed by the PIE algorithm.

       gred   Generalized Random Early Detection combines multiple RED
              queues in order to achieve multiple drop priorities. This
              is required to realize Assured Forwarding (RFC 2597).

       hhf    Heavy-Hitter Filter differentiates between small flows and
              the opposite, heavy-hitters. The goal is to catch the
              heavy-hitters and move them to a separate queue with less
              priority so that bulk traffic does not affect the latency
              of critical traffic.

       ingress
              This is a special qdisc as it applies to incoming traffic
              on an interface, allowing for it to be filtered and
              policed.

       mqprio The Multiqueue Priority Qdisc is a simple queuing
              discipline that allows mapping traffic flows to hardware
              queue ranges using priorities and a configurable priority
              to traffic class mapping. A traffic class in this context
              is a set of contiguous qdisc classes which map 1:1 to a
              set of hardware exposed queues.

       multiq Multiqueue is a qdisc optimized for devices with multiple
              Tx queues. It has been added for hardware that wishes to
              avoid head-of-line blocking.  It will cycle though the
              bands and verify that the hardware queue associated with
              the band is not stopped prior to dequeuing a packet.

       netem  Network Emulator is an enhancement of the Linux traffic
              control facilities that allow to add delay, packet loss,
              duplication and more other characteristics to packets
              outgoing from a selected network interface.

       pfifo_fast
              Standard qdisc for 'Advanced Router' enabled kernels.
              Consists of a three-band queue which honors Type of
              Service flags, as well as the priority that may be
              assigned to a packet.

       pie    Proportional Integral controller-Enhanced (PIE) is a
              control theoretic active queue management scheme. It is
              based on the proportional integral controller but aims to
              control delay.

       red    Random Early Detection simulates physical congestion by
              randomly dropping packets when nearing configured
              bandwidth allocation. Well suited to very large bandwidth
              applications.

       rr     Round-Robin qdisc with support for multiqueue network
              devices. Removed from Linux since kernel version 2.6.27.

       sfb    Stochastic Fair Blue is a classless qdisc to manage
              congestion based on packet loss and link utilization
              history while trying to prevent non-responsive flows (i.e.
              flows that do not react to congestion marking or dropped
              packets) from impacting performance of responsive flows.
              Unlike RED, where the marking probability has to be
              configured, BLUE tries to determine the ideal marking
              probability automatically.

       sfq    Stochastic Fairness Queueing reorders queued traffic so
              each 'session' gets to send a packet in turn.

       tbf    The Token Bucket Filter is suited for slowing traffic down
              to a precisely configured rate. Scales well to large
              bandwidths.
无类的，在添加规则时需要注意：
In the absence of classful qdiscs, classless qdiscs can only be
       attached at the root of a device. Full syntax:
       tc qdisc add dev DEV root QDISC QDISC-PARAMETERS
To remove, issue
       tc qdisc del dev DEV root
 The pfifo_fast qdisc is the automatic default in the absence of a
       configured qdisc.
有类的：

       ATM    Map flows to virtual circuits of an underlying
              asynchronous transfer mode device.

       CBQ    Class Based Queueing implements a rich linksharing
              hierarchy of classes.  It contains shaping elements as
              well as prioritizing capabilities. Shaping is performed
              using link idle time calculations based on average packet
              size and underlying link bandwidth. The latter may be ill-
              defined for some interfaces.

       DRR    The Deficit Round Robin Scheduler is a more flexible
              replacement for Stochastic Fairness Queuing. Unlike SFQ,
              there are no built-in queues -- you need to add classes
              and then set up filters to classify packets accordingly.
              This can be useful e.g. for using RED qdiscs with
              different settings for particular traffic. There is no
              default class -- if a packet cannot be classified, it is
              dropped.

       DSMARK Classify packets based on TOS field, change TOS field of
              packets based on classification.

       ETS    The ETS qdisc is a queuing discipline that merges
              functionality of PRIO and DRR qdiscs in one scheduler. ETS
              makes it easy to configure a set of strict and bandwidth-
              sharing bands to implement the transmission selection
              described in 802.1Qaz.

       HFSC   Hierarchical Fair Service Curve guarantees precise
              bandwidth and delay allocation for leaf classes and
              allocates excess bandwidth fairly. Unlike HTB, it makes
              use of packet dropping to achieve low delays which
              interactive sessions benefit from.

       HTB    The Hierarchy Token Bucket implements a rich linksharing
              hierarchy of classes with an emphasis on conforming to
              existing practices. HTB facilitates guaranteeing bandwidth
              to classes, while also allowing specification of upper
              limits to inter-class sharing. It contains shaping
              elements, based on TBF and can prioritize classes.

       PRIO   The PRIO qdisc is a non-shaping container for a
              configurable number of classes which are dequeued in
              order. This allows for easy prioritization of traffic,
              where lower classes are only able to send if higher ones
              have no packets available. To facilitate configuration,
              Type Of Service bits are honored by default.

       QFQ    Quick Fair Queueing is an O(1) scheduler that provides
              near-optimal guarantees, and is the first to achieve that
              goal with a constant cost also with respect to the number
              of groups and the packet length. The QFQ algorithm has no
              loops, and uses very simple instructions and data
              structures that lend themselves very well to a hardware
              implementation.
			  
#创建规则：
tc qdisc add dev eth0 root handle 1:0 htb default 1 
#添加一个tbf规则，绑定到eth0上，命名为1:0 ，默认归类为1
#handle：为规则命名或指定某规则
```
排队规则在内核中的表示结构：
```cpp
描述排队规则的结构：
struct Qdisc {
	int 			(*enqueue)(struct sk_buff *skb,   --上面提到的两个函数
					   struct Qdisc *sch,
					   struct sk_buff **to_free);
	struct sk_buff *	(*dequeue)(struct Qdisc *sch);
	unsigned int		flags;
	const struct Qdisc_ops	*ops;//队列操作的接口，每个排队规则都必须实现该接口，如pfifo,tbf
	struct qdisc_size_table	__rcu *stab;
	struct list_head	list;
	u32			handle; //和tc的handle对应 句柄，排队规则，类和过滤器都有一个32位的句柄标识；
	u32			parent; //父句柄
	void			*u32_node;

	struct netdev_queue	*dev_queue;//和netdevice挂钩
	struct sk_buff_head	q;//队列当前的数据包数
    ..
}

struct Qdisc_ops {
	struct Qdisc_ops	*next;//用于链接已注册的各种排队规则的操作接口
	const struct Qdisc_class_ops	*cl_ops;//所在规则提供的类操作接口
	char			id[IFNAMSIZ];
	int			priv_size;

	int 			(*enqueue)(struct sk_buff *skb, //将数据包加入排队规则的函数
					   struct Qdisc *sch,
					   struct sk_buff **to_free);
	struct sk_buff *	(*dequeue)(struct Qdisc *);
	struct sk_buff *	(*peek)(struct Qdisc *);

	int			(*init)(struct Qdisc *, struct nlattr *arg);//排队规则的初始化
	void			(*reset)(struct Qdisc *);
	void			(*destroy)(struct Qdisc *);
	int			(*change)(struct Qdisc *, struct nlattr *arg);
	void			(*attach)(struct Qdisc *);

	int			(*dump)(struct Qdisc *, struct sk_buff *);
	int			(*dump_stats)(struct Qdisc *, struct gnet_dump *);

	struct module		*owner;
};
```

+ 类：
类： 定义在排队规则中，报文通过过滤器，过滤，分配到不同的类中；排队规则可以没有类，如fifo先进先出，也可以有多个类
类中也可以有内部的排队规则，包被过滤器过滤为某个类后，在这个类中通过fifo的排队规则出去，或者其他规则，这里的规则就是内部规则；
创建类：
```cpp
 tc [ OPTIONS ] class [ add | change | replace | delete ] dev DEV
       parent qdisc-id [ classid class-id ] qdisc [ qdisc specific
       parameters ]
eg:
#创建分类
tc class add dev eth0 parent 1:0 classid 1:1 htb rate 10Mbit burst 15k
#为eth0下的root队列1:0添加一个分类并命名为1:1，类型为htb，带宽为10M
#rate: 是一个类保证得到的带宽值.如果有不只一个类,请保证所有子类总和是小于或等于父类.
#ceil: ceil是一个类最大能得到的带宽值.
```
类的表示：
```cpp
在linux中，以xxx_class来表示，如htb:
struct htb_class {
	struct Qdisc_class_common common;
	struct psched_ratecfg	rate;
	struct psched_ratecfg	ceil;
	s64			buffer, cbuffer;/* token bucket depth/rate */
	s64			mbuffer;	/* max wait time */
	u32			prio;		/* these two are used only by leaves... */
	int			quantum;	/* but stored for parent-to-leaf return */

	struct tcf_proto __rcu	*filter_list;	/* class attached filters */ 类的过滤器链
	int			filter_cnt;
	int			refcnt;		/* usage count of this class */

	int			level;		/* our level (see above) */
	unsigned int		children;
	struct htb_class	*parent;	/* parent class */

	struct gnet_stats_rate_est64 rate_est;

	/*
	 * Written often fields
	 */
	struct gnet_stats_basic_packed bstats;
	struct tc_htb_xstats	xstats;	/* our special stats */
```


+ 过滤器
过滤器： 具体的过滤规则，用来分类；包含若干个匹配条件，如果符合条件的包，被分类到具体的类中；

一个类至少有一个过滤器，可能有多个过滤器，
tc指令：
```cpp
tc [ OPTIONS ] filter [ add | change | replace | delete | get ]
       dev DEV [ parent qdisc-id | root ] [ handle filter-id ] protocol
       protocol prio priority filtertype [ filtertype specific
       parameters ] flowid flow-id
内核目前提供的过滤器有：
 The available filters are:

       basic  Filter packets based on an ematch expression. See
              tc-ematch(8) for details.

       bpf    Filter packets using (e)BPF, see tc-bpf(8) for details.

       cgroup Filter packets based on the control group of their
              process. See tc-cgroup(8) for details.

       flow, flower
              Flow-based classifiers, filtering packets based on their
              flow (identified by selectable keys). See tc-flow(8) and
              tc-flower(8) for details.

       fw     Filter based on fwmark. Directly maps fwmark value to
              traffic class. See tc-fw(8).

       route  Filter packets based on routing table. See tc-route(8) for
              details.

       rsvp   Match Resource Reservation Protocol (RSVP) packets.

       tcindex
              Filter packets based on traffic control index. See
              tc-tcindex(8).

       u32    Generic filtering on arbitrary packet data, assisted by
              syntax to abstract common operations. See tc-u32(8) for
              details.
    matchall
              Traffic control filter that matches every packet. See
              tc-matchall(8) for details.
eg:
#使用u32创建过滤器
tc filter add dev eth0 protocol ip parent 1:0 prio 1 u32 match ip sport 22 flowid 1:10
```
在内核中的结构：
```cpp
struct tcf_proto {
	/* Fast access part */
	struct tcf_proto __rcu	*next;//将多个过滤器连接起来
	void __rcu		*root;
	int			(*classify)(struct sk_buff *,
					    const struct tcf_proto *,
					    struct tcf_result *); //报文分类函数--> tcf_proto_ops上
	__be16			protocol;

	/* All the rest */
	u32			prio;
	u32			classid;
	struct Qdisc		*q;
	void			*data;
	const struct tcf_proto_ops	*ops;//过滤器操作函数接口
	struct rcu_head		rcu;
};
```
### qdisc的例子： pfifo ,ftb等
通过fifo学习如何实现一个规则；
默认情况下是pfifo，这个通过dev_open挂到设备上；如果需要其他的，通过tc后->netlink再操作到dev结构等上；
```cpp
struct Qdisc noop_qdisc = {
	.enqueue	=	noop_enqueue,
	.dequeue	=	noop_dequeue,
	.flags		=	TCQ_F_BUILTIN,
	.ops		=	&noop_qdisc_ops,
	.list		=	LIST_HEAD_INIT(noop_qdisc.list),
	.q.lock		=	__SPIN_LOCK_UNLOCKED(noop_qdisc.q.lock),
	.dev_queue	=	&noop_netdev_queue,
	.running	=	SEQCNT_ZERO(noop_qdisc.running),
	.busylock	=	__SPIN_LOCK_UNLOCKED(noop_qdisc.busylock),
};

struct Qdisc_ops pfifo_fast_ops __read_mostly = {
	.id		=	"pfifo_fast",
	.priv_size	=	sizeof(struct pfifo_fast_priv),
	.enqueue	=	pfifo_fast_enqueue,
	.dequeue	=	pfifo_fast_dequeue,
	.peek		=	pfifo_fast_peek,
	.init		=	pfifo_fast_init,
	.reset		=	pfifo_fast_reset,
	.dump		=	pfifo_fast_dump,
	.owner		=	THIS_MODULE,
};
```

### tc工具的netlink接口
定义在sch_api.c，主要操作排队规则中的类和过滤器；
tc是通过netlink向内核通信，从而实现创建，修改qos等功能

本文只是给了一个流程和具体认知，通过本文来知道tc大致原理和框架，从而为进一步提供便利和查找依据；
