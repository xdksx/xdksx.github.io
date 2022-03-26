---
title: linux_neighbor
date: 2021-05-22 18:42:26
tags: neighbor
categories: linux
---

#### linux下邻居子系统的实现：

##### 什么是邻居子系统：
在linux下，是一个用来管理网络中，二层的数据正确发送的系统，维护着二层地址相关结构，以及和三层地址映射等结构，
并提供更新，查询等缓存和接口；形成一个系统；供在发送数据时查询并正确发送到指定的机器上；
<!-- more -->
##### 邻居子系统的系统参数分类和总结：
+ 参数：
```cpp
base_reachable_time_ms: 邻居项有效期初始值，这个值会被来自更高协议正反馈延长；
delay_first_probe_time: 邻居项过期后发送第一个探测之前的延迟时间，默认5s
gc_stale_time: 确定多久检测一次邻居项过期，默认60s
gc_interval:垃圾回收处理邻居项的时间间隔，默认30s

echo 30 > /proc/sys/net/ipv4/neigh/default/gc_stale_time
echo 175 > /proc/sys/net/ipv4/route/gc_timeout
echo 20000 > /proc/sys/net/ipv4/neigh/default/base_reachable_time_ms
echo 30 > /proc/sys/net/ipv4/route/gc_interval

pherricoxide@midigaurd:~$ ip -s neighbor list
192.168.42.1 dev eth0 lladdr 00:25:90:7d:7e:cd ref 2 used 184/184/139 probes 4 STALE
192.168.10.2 dev eth0 lladdr 00:1c:23:cf:0b:6a ref 3 used 33/28/0 probes 1 REACHABLE
192.168.10.1 dev eth0 lladdr 00:17:c5:d8:90:a4 ref 219 used 275/4/121 probes 1 REACHABLE
```
+ 参数定义和使用的位置：
```cpp
---- base_reachable_time_ms Matches (8 in 4 files) ----
ndisc_ifinfo_sysctl_change in ndisc.c (net\ipv6) : 		 (strcmp(ctl->procname, "base_reachable_time_ms") == 0))
neigh_proc_base_reachable_time in neighbour.c (net\core) : 	else if (strcmp(ctl->procname, "base_reachable_time_ms") == 0)
neighbour.c (net\core) line 3076 : 		NEIGH_SYSCTL_MS_JIFFIES_REUSED_ENTRY(BASE_REACHABLE_TIME_MS, BASE_REACHABLE_TIME, "base_reachable_time_ms"),
neigh_sysctl_register in neighbour.c (net\core) : 		t->neigh_vars[NEIGH_VAR_BASE_REACHABLE_TIME_MS].proc_handler = handler;
neigh_sysctl_register in neighbour.c (net\core) : 		t->neigh_vars[NEIGH_VAR_BASE_REACHABLE_TIME_MS].proc_handler =
neighbour.h (include\net) line 59 : 	NEIGH_VAR_BASE_REACHABLE_TIME_MS, /* same data as NEIGH_VAR_BASE_REACHABLE_TIME */
sysctl_binary.c (kernel) line 285 : 	{ CTL_INT,	NET_NEIGH_REACHABLE_TIME_MS,	"base_reachable_time_ms" },
```

+ 数据结构：
```cpp
static struct neigh_table *neigh_tables[NEIGH_NR_TABLES] __read_mostly;
neigh_table_init(int index, struct neigh_table *tbl)
neigh_tables[index] = tbl;

一个邻居协议，对应一个neigh_table表实例：多个邻居协议实例，放在上面的数组中，
struct neigh_table {
	int			family;//邻居协议所属的地址簇，如ARP为ipv4 ,AF_INET
	int			entry_size;//邻居表中有多个邻居项，这个是邻居项结构的大小，对arp_tbl来说，初始化为sizeof(neighbour+4);
                                                        //因为在ARP中neighbour结构的最后一个成员primary_key，实际指向一个ipv4地址，4为ipv4地址长度；
	int			key_len;//哈希函数用的key长度，key一般是三层协议地址，所以一般是4
	__be16			protocol;
	__u32			(*hash)(const void *pkey,  //哈希函数，用来计算哈希值，arp为arp_hash()
					const struct net_device *dev,
					__u32 *hash_rnd);
	bool			(*key_eq)(const struct neighbour *, const void *pkey);
	int			(*constructor)(struct neighbour *);//邻居项初始化函数，用于初始化一个新的neighbour结构中的相关字段，arp
                                                                                //为arp_constructor
	int			(*pconstructor)(struct pneigh_entry *);
	void			(*pdestructor)(struct pneigh_entry *); //创建和释放一个代理项时被调用，ipv4没用，ipv6用了
	void			(*proxy_redo)(struct sk_buff *skb);//用来处理neigh_table->proxy_queue缓存队列中的代理arp报文
	char			*id;// 用来分配neighbour实例的缓冲池名字符串，arp为arp_cache
	struct neigh_parms	parms;   //存储一些和协议相关的可调节参数，如重传超时时间，proxy_queue队列长
	struct list_head	parms_list;
	int			gc_interval;//垃圾回收时钟gc_timer的到期间隔时间，即到期时触发一次垃圾回收，初始值是30s
	int			gc_thresh1;
	int			gc_thresh2;
	int			gc_thresh3; //这三个阈值对应内存对邻居项作垃圾回收处理的不同级别，若缓存邻居项数少于gc_thresh1,则不执行删除，超过gc_thresh2则在新建邻居项时若超过5s为刷新，则立即刷新，并做强制回收处理，超过gc_thresh3则，新建邻居项时，立即刷新并强制垃圾回收
	unsigned long		last_flush;//记录最近一次调用neigh_forced_gc强制刷新邻居表的时间；用于判断是否回收
	struct delayed_work	gc_work;//垃圾回收的相关结构，包括垃圾回收定时器
	struct timer_list 	proxy_timer; //处理proxy_queue队列的定时器，当proxy_queue为空，则第一个arp报文加入队列时，会启动这个定时器；在neigh_table_init中初始化，处理在neigh_proxy_process中处理；
	struct sk_buff_head	proxy_queue;//对于接收到需要进行代理的arp报文，需要先缓存到这个队列，在定时器处理函数中再处理它；
	atomic_t		entries; //整个邻居表中邻居项的数目
	rwlock_t		lock; //用于控制邻居表的读写锁 如look_up只需要读，而neigh_periodic_timer需要写
	unsigned long		last_rand; //用于记录neigh_params结构中的reachable_time成员最近一次被更新的时间
	struct neigh_statistics	__percpu *stats; //各类统计数据
	struct neigh_hash_table __rcu *nht; // 见下
	struct pneigh_entry	**phash_buckets;//见下
};

enum {
	NEIGH_ARP_TABLE = 0,
	NEIGH_ND_TABLE = 1,
	NEIGH_DN_TABLE = 2,
	NEIGH_NR_TABLES,
	NEIGH_LINK_TABLE = NEIGH_NR_TABLES /* Pseudo table for neigh_xmit */
};
/*
 *	neighbour table manipulation
 */

#define NEIGH_NUM_HASH_RND	4

struct neigh_hash_table {
	struct neighbour __rcu	**hash_buckets; //用于存储邻居项的散列表，改散列表在分配邻居项时，若邻居项数量超过散列表容量，可以动态扩容
	unsigned int		hash_shift;
	__u32			hash_rnd[NEIGH_NUM_HASH_RND];//随机数，用来在hash_buckets散列表扩容时计算关键字，以避免受到arp攻击
	struct rcu_head		rcu;
};

struct pneigh_entry { //存储arp代理三层协议地址的散列表，在neigh_table_init_no_netlink中完成初始化
	struct pneigh_entry	*next;
	possible_net_t		net;
	struct net_device	*dev;
	u8			flags;
	u8			key[0];
};

邻居表中一个邻居项的表示的结构体：
邻居项存储了邻居相关信息，包括状态，二层，三层协议地址，提供给三层协议的函数指针，定时器和缓存的二层首部等；注意一个邻居不代表一个主机，而是一个三层协议地址，因为对于配置了多接口的主机，一个主机将对应多个三层地址;
struct neighbour {
	struct neighbour __rcu	*next; //通过next串起来邻居表的所有邻居项
	struct neigh_table	*tbl;//指向所属的邻居表的指针
	struct neigh_parms	*parms; //用于调节邻居协议的参数，在创建邻居项的neigh_create中会调用alloc分配，其中会进行初始化，接着调constructor进行设置
	unsigned long		confirmed;//记录最近一次确认该邻居可达性的时间，用来描述邻居可达性；接收到回复时更新；传输层通过neigh_confirm来更新，而邻居子系统则通过neigh_update来更新
	unsigned long		updated;//记录最近一次被neigh_update更新的时间；和confirm不同；他们针对不同的特性
	rwlock_t		lock; //用来控制访问邻居项的读写锁
	atomic_t		refcnt;//引用计数
	struct sk_buff_head	arp_queue;//当邻居项状态处于无效时，用来缓存要发送的报文。若处于INCOMPLETE时，发送第一个要更新的；。。。
	unsigned int		arp_queue_len_bytes;
	struct timer_list	timer;//管理多种超时情况的定时器
	unsigned long		used;//最近一次被使用时间，不同状态下被不同函数更新，如neigh_event_send()和gc_timer
	atomic_t		probes; //尝试发送请求报文而未能得到应答的次数；在定时器中被检测，当达到阈值时 ，进入NUD_FAILED状态
	__u8			flags; //记录一些标志和特性
	__u8			nud_state; //邻居项状态
	__u8			type; //邻居地址的类型：对arp: RTN_UNICAST，RIN_LOCAL等
	__u8			dead;//生存标志，为1则表示正被删除
	seqlock_t		ha_lock;
	unsigned char		ha[ALIGN(MAX_ADDR_LEN, sizeof(unsigned long))]; //与存储在primary_key中的三层地址对应的二进制二层硬件地址；6B,其他链路可能更长，以太网是6B
	struct hh_cache		hh;//缓存的二层协议首部hh_cache
	int			(*output)(struct neighbour *, struct sk_buff *);//输出函数，用来将报文输出到该邻居；在邻居项生命周期中，由于其状态不断变化，所以该函数指针会指向不同的输出函数，如可达时调用neigh_connect()将output置为 neigh_ops-->connected_output
	const struct neigh_ops	*ops;//指向邻居项函数指针实例
	struct rcu_head		rcu;
	struct net_device	*dev;  //通过此网络设备可访问到该邻居；对每个邻居而言，只有一个
	u8			primary_key[0];//存储哈希函数使用的三层协议地址；
};

struct neigh_ops {
	int			family;
	void			(*solicit)(struct neighbour *, struct sk_buff *);//发送请求报文函数，在发送第一个报文时，需要新的邻居项，发送的报文会被缓存到arp_queue中，然后调用solicit()发送请求报文
	void			(*error_report)(struct neighbour *, struct sk_buff *);//当邻居项缓存着未发送的数据报文时，而该邻居项又不可达，被调用来向三层报告错误的函数，如arp_error_report(),最后会向报文发送方发送一个主机不可达的icmp差错报文
	int			(*output)(struct neighbour *, struct sk_buff *);//最通用的output函数，可用于所有情况；实现了完整的输出过程，此函数消耗资源。注意不要将neigh_ops->output和neighbour->output混淆
	int			(*connected_output)(struct neighbour *, struct sk_buff *);//邻居可达时，即状态为NUD_CONNECTED时；只是简单的加二层头，所以比上面的output快
};


struct neigh_parms {//邻居协议参数配置块，用来存储可调节的邻居协议参数，如重传超时时间，proxy_queueu长度等；
	possible_net_t net;
	struct net_device *dev;
	struct list_head list;
	int	(*neigh_setup)(struct neighbour *);//提供给老式接口设备的初始化和销毁接口，注意区分net_device中的setup成员函数
	void	(*neigh_cleanup)(struct neighbour *);
	struct neigh_table *tbl;

	void	*sysctl_table;//邻居表的sysctl表 ，arp_init中初始化，这样用户可以通过proc来读写

	int dead;
	atomic_t refcnt;
	struct rcu_head rcu_head;

	int	reachable_time;
	int	data[NEIGH_VAR_DATA_MAX];
	DECLARE_BITMAP(data_state, NEIGH_VAR_DATA_MAX);
};

struct pneigh_entry {//用来保存允许代理的条件，只有和结构中的接收设备以及目标地址相匹配才能代理，保存在phash_buckets散列表中，称为代理项，可以通过ip neigh add proxy命令添加；
	struct pneigh_entry	*next;
	possible_net_t		net;
	struct net_device	*dev;//通过该设备接收到的arp请求报文才能代理
	u8			flags;
	u8			key[0];
};


//统计数据：一个该结构实例对应一个网络设备上的一种邻居协议
struct neigh_statistics {
	unsigned long allocs;		/* number of allocated neighs */
	unsigned long destroys;		/* number of destroyed neighs */
	unsigned long hash_grows;	/* number of hash resizes */

	unsigned long res_failed;	/* number of failed resolutions */

	unsigned long lookups;		/* number of lookups */
	unsigned long hits;		/* number of hits (among lookups) */

	unsigned long rcv_probes_mcast;	/* number of received mcast ipv6 */
	unsigned long rcv_probes_ucast; /* number of received ucast ipv6 */

	unsigned long periodic_gc_runs;	/* number of periodic GC runs */
	unsigned long forced_gc_runs;	/* number of forced GC runs */

	unsigned long unres_discards;	/* number of unresolved drops */
	unsigned long table_fulls;      /* times even gc couldn't help */
};

hh_cache定义在netdevice.h中，是用来缓存二层首部的，这样可以复制而不是逐个设置，加快输出报文
struct hh_cache {
	u16		hh_len;
	u16		__pad;
	seqlock_t	hh_lock;

	/* cached hardware header; allow for machine alignment needs.        */
#define HH_DATA_MOD	16
#define HH_DATA_OFF(__len) \
	(HH_DATA_MOD - (((__len - 1) & (HH_DATA_MOD - 1)) + 1))
#define HH_DATA_ALIGN(__len) \
	(((__len)+(HH_DATA_MOD-1))&~(HH_DATA_MOD - 1))
	unsigned long	hh_data[HH_DATA_ALIGN(LL_MAX_HEADER) / sizeof(long)];
};

```

+ 邻居表的初始化：

```cpp
void neigh_table_init(int index, struct neigh_table *tbl)
{
	unsigned long now = jiffies;
	unsigned long phsize;
           //就是将表中的字段复制到项中
	INIT_LIST_HEAD(&tbl->parms_list);
	list_add(&tbl->parms.list, &tbl->parms_list);
	write_pnet(&tbl->parms.net, &init_net);
	atomic_set(&tbl->parms.refcnt, 1);
	tbl->parms.reachable_time =
			  neigh_rand_reach_time(NEIGH_VAR(&tbl->parms, BASE_REACHABLE_TIME));

	tbl->stats = alloc_percpu(struct neigh_statistics);
	if (!tbl->stats)
		panic("cannot create neighbour cache statistics");

#ifdef CONFIG_PROC_FS
	if (!proc_create_data(tbl->id, 0, init_net.proc_net_stat,
			      &neigh_stat_seq_fops, tbl))
		panic("cannot create neighbour proc dir entry");
#endif

	RCU_INIT_POINTER(tbl->nht, neigh_hash_alloc(3));

	phsize = (PNEIGH_HASHMASK + 1) * sizeof(struct pneigh_entry *);
	tbl->phash_buckets = kzalloc(phsize, GFP_KERNEL);

	if (!tbl->nht || !tbl->phash_buckets)
		panic("cannot allocate neighbour cache hashes");

	if (!tbl->entry_size)
		tbl->entry_size = ALIGN(offsetof(struct neighbour, primary_key) +
					tbl->key_len, NEIGH_PRIV_ALIGN);
	else
		WARN_ON(tbl->entry_size % NEIGH_PRIV_ALIGN);
           //建立定时器，并初始化定时器，包括两个定时器，老化(垃圾回收)定时器和proxy代理定时器
	rwlock_init(&tbl->lock);
	INIT_DEFERRABLE_WORK(&tbl->gc_work, neigh_periodic_work);//老化定时器对应的处理函数是neigh_periodic_work
	queue_delayed_work(system_power_efficient_wq, &tbl->gc_work,
			tbl->parms.reachable_time);
	setup_timer(&tbl->proxy_timer, neigh_proxy_process, (unsigned long)tbl);//proxy_timer对应的处理函数是neigh_proxy_process
	skb_queue_head_init_class(&tbl->proxy_queue,
			&neigh_table_proxy_queue_class);

	tbl->last_flush = now;
	tbl->last_rand	= now + tbl->parms.reachable_time * 20;
           //放到全局数组中
	neigh_tables[index] = tbl;
}
```

+ 邻居表的老化定时器和对应的状态机：
邻居表项有一个对于管理和维护邻居表来说很重要的成员： nud_state; 表示邻居项当前的状态；7种；
邻居表老化定时器定时扫描所有邻居表项，然后进行如下的状态迁移和处理
图+各个状态解释


+ 邻居项的创建： 分配和创建初始化；
neigh_create: neigh_alloc


+ 邻居表项扩容：
static struct neigh_hash_table *neigh_hash_grow(struct neigh_table *tbl,
						unsigned long new_shift)

+ 邻居表的各种操作：
邻居项的查找： neigh_look_up 查找频繁，添加和删除等都需要先查找  
邻居项的更新： neigh_update: 更新内容：硬件地址和状态，并根据状态更新输出函数  
垃圾回收：  
同步回收： 在创建新的邻居项时，达到条件触发；  
异步回收： 定时检测并看是否触发回收
外部事件的通知和处理：当网络设备NETDEV_UNREGISTER事件发生时，若邻居协议模块对此感兴趣，会调用此接口：neigh_ifdown;  
如arp: arp_ifdown->neigh_ifdown; 另外还有neigh_changeaddr  

+ 每个邻居项的定时函数：
邻居项的状态中，有些属于定时状态； 每个邻居项都有一个定时器，在创建邻居项时被初始化

+ 关于代理项：phash_buckets
查找： pneigh_lookup  删除 pneigh_delete
延时处理代理的请求报文：pneigh_enqueue

+ 关于输出函数：
丢弃： neigh_blackhole();
慢速发送：neigh_resolve_output
快速发送： dst_output后--> neigh_hh_output / neigh_connected_output

+ 关于邻居代理，arp代理：
arp代理：
对一个二层协议，在一个局域网内，则使用二层地址寻找，当不在此局域网了，则通过路由器网关转发出去；
但是可能在一个局域网内有多个子网，通过路由器或交换机转发，但是主机不知道，所以这个时候依然用相同的方式去发送arp请求，当处于不同子网的两个主机arp时，
则此时需要依赖路由器打开arp代理转发功能，若开启，则会在收到arp请求时，返回自己的mac地址，这样相当于转发功能，会收到数据，转发给对方主机，然后从对方主机收到后又进行转发；


注意到整个流程：
   arp会缓存数据包，然后等到arp邻居项可用了，再把数据包发出去；


+ arp外部事件通知：
   当一个设备的ip地址发生变化，arp模块通过注册到通知链中arp_netdev_notifier收到通知，调用arp_netdev_event来处理NETDEV_CHANGEADDR事件；删除释放与禁用网络设备相关的邻居项；等


+ 路由表项与邻居项的绑定：
    在路由模块中，每当添加一条输出路由或是单播转发路由时，会尝试将 该路由与该路由目的地址相对应的邻居项绑定；
arp_bind_neighbour 实现了路由表项与邻居绑定的功能；在绑定过程中，若对应的邻居项不存在，则新建一个邻居项后绑定；之后 再输出报文时就可以通过路由缓存找到
输出函数了；


#### 内核中的arp实现：
+ arp收包：
```cpp
/*
 *	Receive an arp request from the device layer.
 */

static int arp_rcv(struct sk_buff *skb, struct net_device *dev,
		   struct packet_type *pt, struct net_device *orig_dev)
{
	const struct arphdr *arp;

	/* do not tweak dropwatch on an ARP we will ignore */
	if (dev->flags & IFF_NOARP ||
	    skb->pkt_type == PACKET_OTHERHOST ||
	    skb->pkt_type == PACKET_LOOPBACK)
		goto consumeskb;

	skb = skb_share_check(skb, GFP_ATOMIC);//检测并返回skb,当被检测的skb被引用多次时，则克隆此skb,并返回克隆得到的skb;
	if (!skb)
		goto out_of_mem;

	/* ARP header, plus 2 device addresses, plus 2 IP addresses.  */
	if (!pskb_may_pull(skb, arp_hdr_len(dev)))// 在skb中加4个地址？
		goto freeskb;
           /这个函数的解释： pskb_may_pull(..)
             /* Moves tail of skb head forward, copying data from fragmented part,
 * when it is necessary.
 * 1. It may fail due to malloc failure.
 * 2. It may change skb pointers.
 *
 * It is pretty complicated. Luckily, it is called only in exceptional cases.
 */
       /

	arp = arp_hdr(skb);  //从skb中拿arp头，然后下面做检测
	if (arp->ar_hln != dev->addr_len || arp->ar_pln != 4)
		goto freeskb;

	memset(NEIGH_CB(skb), 0, sizeof(struct neighbour_cb));//把skb中的cb字段清空，见下的解释 control buffer

	return NF_HOOK(NFPROTO_ARP, NF_ARP_IN,
		       dev_net(dev), NULL, skb, dev, NULL,
		       arp_process); //通过netfilter 处理后再转到arp_process

consumeskb:
	consume_skb(skb);
	return NET_RX_SUCCESS;
freeskb:
	kfree_skb(skb);
out_of_mem:
	return NET_RX_DROP;
}

The control buffer is an e­junkyard that can be used as a scratch pad during processing by a given
layer of the protocol.   Its main use is by the IP layer to compile header options.
 265 /*
 266 * This is the control buffer. It is free to use for every
 267 * layer. Please put your private variables there. If you
 268 * want to keep them across layers you have to skb_clone()
 269 * first. This is owned by whoever has the skb queued ATM.
 270 */
 271 char cb[48];
 272

//接下来进入到arp_process，主要做arp包的头的校验及：
/*
 *	Process an arp request.
 */

static int arp_process(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct net_device *dev = skb->dev;
	struct in_device *in_dev = __in_dev_get_rcu(dev);
	struct arphdr *arp;
	unsigned char *arp_ptr;
	struct rtable *rt;
	unsigned char *sha;
	__be32 sip, tip;
	u16 dev_type = dev->type;
	int addr_type;
	struct neighbour *n;
	struct dst_entry *reply_dst = NULL;
	bool is_garp = false;

	/* arp_rcv below verifies the ARP header and verifies the device
	 * is ARP'able.
	 */

	if (!in_dev)
		goto out_free_skb;
           //拿到arp的头
	arp = arp_hdr(skb);

	switch (dev_type) {
	default:    //三层一样的，所以先做检测三层协议格式是不是ip,以及二层的和dev的是不是一样
		if (arp->ar_pro != htons(ETH_P_IP) ||
		    htons(dev_type) != arp->ar_hrd)
			goto out_free_skb;
		break;
            //检查二层的格式
	case ARPHRD_ETHER:
	case ARPHRD_FDDI:
	case ARPHRD_IEEE802:
		/*
		 * ETHERNET, and Fibre Channel (which are IEEE 802
		 * devices, according to RFC 2625) devices will accept ARP
		 * hardware types of either 1 (Ethernet) or 6 (IEEE 802.2).
		 * This is the case also of FDDI, where the RFC 1390 says that
		 * FDDI devices should accept ARP hardware of (1) Ethernet,
		 * however, to be more robust, we'll accept both 1 (Ethernet)
		 * or 6 (IEEE 802.2)
		 */
		if ((arp->ar_hrd != htons(ARPHRD_ETHER) &&
		     arp->ar_hrd != htons(ARPHRD_IEEE802)) ||
		    arp->ar_pro != htons(ETH_P_IP))
			goto out_free_skb;
		break;
	case ARPHRD_AX25:
		if (arp->ar_pro != htons(AX25_P_IP) ||
		    arp->ar_hrd != htons(ARPHRD_AX25))
			goto out_free_skb;
		break;
	case ARPHRD_NETROM:
		if (arp->ar_pro != htons(AX25_P_IP) ||
		    arp->ar_hrd != htons(ARPHRD_NETROM))
			goto out_free_skb;
		break;
	}

	/* Understand only these message types */
            //这里只会处理arp请求和回复
	if (arp->ar_op != htons(ARPOP_REPLY) &&
	    arp->ar_op != htons(ARPOP_REQUEST))
		goto out_free_skb;

/*
 *	Extract fields
 */
	arp_ptr = (unsigned char *)(arp + 1);
	sha	= arp_ptr;
	arp_ptr += dev->addr_len;
	memcpy(&sip, arp_ptr, 4);
	arp_ptr += 4;
	switch (dev_type) {
#if IS_ENABLED(CONFIG_FIREWIRE_NET)
	case ARPHRD_IEEE1394:
		break;
#endif
	default:
		arp_ptr += dev->addr_len;
	}
	memcpy(&tip, arp_ptr, 4);
/*
 *	Check for bad requests for 127.x.x.x and requests for multicast
 *	addresses.  If this is one such, delete it.
 */         //对本地环回的不需要处理？
	if (ipv4_is_multicast(tip) ||
	    (!IN_DEV_ROUTE_LOCALNET(in_dev) && ipv4_is_loopback(tip)))
		goto out_free_skb;

 /*
  *	For some 802.11 wireless deployments (and possibly other networks),
  *	there will be an ARP proxy and gratuitous ARP frames are attacks
  *	and thus should not be accepted.
  */        //对80211无线网络，可能是arp攻击，所以也不接受
	if (sip == tip && IN_DEV_ORCONF(in_dev, DROP_GRATUITOUS_ARP))
		goto out_free_skb;

/*
 *     Special case: We must set Frame Relay source Q.922 address
 */
	if (dev_type == ARPHRD_DLCI)
		sha = dev->broadcast;

/*
 *  Process entry.  The idea here is we want to send a reply if it is a
 *  request for us or if it is a request for someone else that we hold
 *  a proxy for.  We want to add an entry to our cache if it is a reply
 *  to us or if it is a request for our address.
 *  (The assumption for this last is that if someone is requesting our
 *  address, they are probably intending to talk to us, so it saves time
 *  if we cache their address.  Their address is also probably not in
 *  our cache, since ours is not in their cache.)
 *
 *  Putting this another way, we only care about replies if they are to
 *  us, in which case we add them to the cache.  For requests, we care
 *  about those for us and those for our proxies.  We reply to both,
 *  and in the case of requests for us we add the requester to the arp
 *  cache.
 */
           //对arp请求的处理
	if (arp->ar_op == htons(ARPOP_REQUEST) && skb_metadata_dst(skb))
		reply_dst = (struct dst_entry *)
			    iptunnel_metadata_reply(skb_metadata_dst(skb),
						    GFP_ATOMIC);

	/* Special case: IPv4 duplicate address detection packet (RFC2131) */
	if (sip == 0) {//请求报文的源ip为0，则该arp报文是用来检测ipv4地址冲突的 rfc2131,因此在确定请求报文的目标ip地址为本地ip后，以该ip地址为源地址及目标地址发送arp应答报文
		if (arp->ar_op == htons(ARPOP_REQUEST) &&
		    inet_addr_type_dev_table(net, dev, tip) == RTN_LOCAL &&
		    !arp_ignore(in_dev, sip, tip))
			arp_send_dst(ARPOP_REPLY, ETH_P_ARP, sip, dev, tip,
				     sha, dev->dev_addr, sha, reply_dst);
		goto out_consume_skb;
	}

	if (arp->ar_op == htons(ARPOP_REQUEST) && //若为arp请求报文且能找到目的ip地址tip对应的路由
	    ip_route_input_noref(skb, tip, sip, 0, dev) == 0) {

		rt = skb_rtable(skb);
		addr_type = rt->rt_type;

		if (addr_type == RTN_LOCAL) {//若是本地接收的
			int dont_send;

			dont_send = arp_ignore(in_dev, sip, tip);
			if (!dont_send && IN_DEV_ARPFILTER(in_dev))
				dont_send = arp_filter(sip, tip, dev);//过滤
			if (!dont_send) {
				n = neigh_event_ns(&arp_tbl, sha, &sip, dev);//调用来更新邻居项
				if (n) {
					arp_send_dst(ARPOP_REPLY, ETH_P_ARP,//发送arp应答报文
						     sip, dev, tip, sha,
						     dev->dev_addr, sha,
						     reply_dst);
					neigh_release(n);
				}
			}
			goto out_consume_skb;
		} else if (IN_DEV_FORWARD(in_dev)) {//若不是发送给本地的，看看是否要做代理缓存
			if (addr_type == RTN_UNICAST  &&
			    (arp_fwd_proxy(in_dev, dev, rt) ||
			     arp_fwd_pvlan(in_dev, dev, rt, sip, tip) ||
			     (rt->dst.dev != dev &&
			      pneigh_lookup(&arp_tbl, net, &tip, dev, 0)))) {
				n = neigh_event_ns(&arp_tbl, sha, &sip, dev);
				if (n)
					neigh_release(n);

				if (NEIGH_CB(skb)->flags & LOCALLY_ENQUEUED ||
				    skb->pkt_type == PACKET_HOST ||
				    NEIGH_VAR(in_dev->arp_parms, PROXY_DELAY) == 0) {
					arp_send_dst(ARPOP_REPLY, ETH_P_ARP,
						     sip, dev, tip, sha,
						     dev->dev_addr, sha,
						     reply_dst);
				} else {
					pneigh_enqueue(&arp_tbl,
						       in_dev->arp_parms, skb);
					goto out_free_dst;
				}
				goto out_consume_skb;
			}
		}
	}

	/* Update our ARP tables */
            //下面做更新arp表的操作
	n = __neigh_lookup(&arp_tbl, &sip, dev, 0);

	if (IN_DEV_ARP_ACCEPT(in_dev)) {
		unsigned int addr_type = inet_addr_type_dev_table(net, dev, sip);

		/* Unsolicited ARP is not accepted by default.
		   It is possible, that this option should be enabled for some
		   devices (strip is candidate)
		 */
		is_garp = arp->ar_op == htons(ARPOP_REQUEST) && tip == sip &&
			  addr_type == RTN_UNICAST;

		if (!n &&
		    ((arp->ar_op == htons(ARPOP_REPLY)  &&
				addr_type == RTN_UNICAST) || is_garp))
			n = __neigh_lookup(&arp_tbl, &sip, dev, 1);
	}

	if (n) {
		int state = NUD_REACHABLE;
		int override;

		/* If several different ARP replies follows back-to-back,
		   use the FIRST one. It is possible, if several proxy
		   agents are active. Taking the first reply prevents
		   arp trashing and chooses the fastest router.
		 */
		override = time_after(jiffies,
				      n->updated +
				      NEIGH_VAR(n->parms, LOCKTIME)) ||
			   is_garp;

		/* Broadcast replies and request packets
		   do not assert neighbour reachability.
		 */
		if (arp->ar_op != htons(ARPOP_REPLY) ||
		    skb->pkt_type != PACKET_HOST)
			state = NUD_STALE;
		neigh_update(n, sha, state,
			     override ? NEIGH_UPDATE_F_OVERRIDE : 0);
		neigh_release(n);
	}

out_consume_skb:
	consume_skb(skb);

out_free_dst:
	dst_release(reply_dst);
	return NET_RX_SUCCESS;

out_free_skb:
	kfree_skb(skb);
	return NET_RX_DROP;
}

```
+ arp发送时和发送接口：
1) 一些发送接口；注意是发送arp请求的，不是数据包
```cpp
void arp_send(int type, int ptype, __be32 dest_ip,
	      struct net_device *dev, __be32 src_ip,
	      const unsigned char *dest_hw, const unsigned char *src_hw,
	      const unsigned char *target_hw)
{
	arp_send_dst(type, ptype, dest_ip, dev, src_ip, dest_hw, src_hw,  --有三个接口会调用它：arp_send/arp_solicit/arp_process
		     target_hw, NULL); ---会调用arp_create创建+arp_xmit发送
}
创建arp报文：
/*
 *	Create an arp packet. If dest_hw is not set, we create a broadcast
 *	message.
 */
struct sk_buff *arp_create(int type, int ptype, __be32 dest_ip,
```
2）三层发送数据包时：
三层发送需要时会把数据包缓存起来，arp邻居项状态达到connected时发送；



+ arp老化定时器：结合图
+ 邻居表老化定时器
neigh_periodic_work

+ 邻居项定时器：
neigh_timer_handler

+ 代理项定时器
neigh_proxy_process






