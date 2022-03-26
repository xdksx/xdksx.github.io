---
title: sk_buff
date: 2021-02-21 01:33:29
tags: tcpip_code
categories: tcpip
---


### 关于skb
skb是网络协议栈中对包的底层操作结构，它需要满足以下特性：
1) 能方便的处理可变长缓存，因为发送和接收的数据长时不固定的
2）能容易实现头尾部增加和移除数据，因为这些缓存区需要在不同网络层次间进行传递
3）添加和移除数据能尽量避免数据的复制<!-- more -->
sk_buffer定义：
include/linux/skbuff.h
net/core/skbuff.c
在下载的linux源码中能找到(下来看的或者是ubuntu中下来编译内核的)

ref:<linux内核源码剖析--TCP/IP实现>
### sk_buff的结构
#### 大体结构：
分类：
1) 与skb组织相关的成员变量
2) 通用成员变量
3) 标志性变量
4) 与特性相关的成员变量
   这些与特性相关的成员，往往用#ifdef来限制，若要使用他们，需要系统定义这些宏，而这个需要在编译内核时开启，比如：
   编译时选中 Networking->Networking options->QoS and/or fair queueing->action，选中包分类器的功能，对应宏：
   CONFIG_NET_CLS_ACT
   注意的是：若某个内核模块包含了使用未定义的宏限制的变量，则无法被内核使用；

skb被哪些网络层次处理？
二层的mac或者其他链路层协议，三层的ip协议，四层的tcp和udp协议，某些成员会在层次间传递的时候发生改变，如四层向三层传递时会加一个ip头，
三层向二层传递时会加一个mac头等，反之则删除，而传递时只增加头部可以提高效率；这个需要复杂的指针操作，内核提供了一个函数： skb_reserve();
起到在向下传递skb前，在数据缓存区头部预留空间的作用；

#### skb相关的两个链表
在正式看skb的结构之前，先看看组织skb的两个链表，来看skb的全局：
在数据被传递进来时，触发中断，中断处理程序会将数据传递进内核，接着一步步拷贝到skb；skb由链表组织：
在sk_buff中有两个成员结构：
struct sk_buff *next;
struct sk_buff *prev;
以此构成skb_buff双向链表；
为了能使每个skb都能被整个链表的头部快速找到，在第一个skb结点前加了一个辅助的sk_buff_head结构的头结点，就像链表头一样，作为
链表的头结点，可以由此遍历skb链表；

```cpp
struct sk_buff_head {
	/* These two members must be first. */
	struct sk_buff	*next;
	struct sk_buff	*prev;

	__u32		qlen;
	spinlock_t	lock;
};
```

结构组织：
<-> sk_buff_head <-> skb1 <-> skb2 ...<-> 链接到sk_buff_head形成环形双向链表

#### skb结构：
skb结构可以被大致分为 描述符(skb本身) 和数据缓冲区 (head等成员指针指向的数据)
以下是内核4.4.0中的include/linux/定义的sk_buff结构：
```cpp

/** 
 *	struct sk_buff - socket buffer
 *	@next: Next buffer in list 
 *	@prev: Previous buffer in list
 *	@tstamp: Time we arrived/left 接收或发送时间戳，在网络设备收到一个数据包后，由netif_receive_skb()或netif_rx()调用net_timestamp()设置
 *	@rbnode: RB tree node, alternative to next/prev for netem/tcp
 *	@sk: Socket we are owned by //skb的宿主，当源ip和目的ip都不是本机时，为NULL,否则在数据发送接收时通过这个结构传递给到上层和应有程序挂钩；
 *	@dev: Device we arrived on/are leaving by 从网卡接收数据时，需要分配skb接收缓存队列，这个时候分配成功，就会设置这个结构为对应网卡，发送类似，复杂些；
 *	@cb: Control buffer. Free for use by every layer. Put private vars here
 *	@_skb_refdst: destination entry (with norefcount bit) 目的路由缓存；
 *	@sp: the security path, used for xfrm ipsec协议用来跟踪传输信息
 *	@len: Length of actual data  skb实际数据部分长度，包括线性缓存区中数据长度和(data->)和SG类型的聚合分散io的数据和FRAGLIST类型的聚合分散io数据，以及传递时增加或减去的协议首部；
 *	@data_len: Data length SG类型的聚合分散io的数据和FRAGLIST类型的聚合分散io数据长
 *	@mac_len: Length of link layer header
 *	@hdr_len: writable header length of cloned skb
 *	@csum: Checksum (must include start/offset pair)
 *	@csum_start: Offset from skb->head where checksumming should start
 *	@csum_offset: Offset from csum_start where checksum should be stored
 *	@priority: Packet queueing priority 发送或转发数据包的Qos类别，若包本地生成，则套接口层会设置该字段；
 *	@ignore_df: allow local fragmentation 允许本地分片
 *	@cloned: Head may be cloned (check refcnt to be sure) 标记所属的skb是否已克隆
 *	@ip_summed: Driver fed us an IP checksum  用于标记传输层校验和的状态：CHECKSUM_NONE/CHECKSUM_PARTIAL/..
 *	@nohdr: Payload reference only, must not modify header payload是否被单独引用，不存在协议首部。若被单独引用，则不能修改首部，也不能由data访问首部
 *	@nfctinfo: Relationship of this skb to the connection 
 *	@pkt_type: Packet class 帧类型：PACKET_HOST/PACKET_BROADCAST/PACKET_MULTICAST/PACKET_OTHERHOST/...
 *	@fclone: skbuff clone status 当前克隆状态：SKB_FCLONE_UNAVAILABLE/SKB_FCLONE_ORIG/SKB_FCLONE_CLONE:
 *	@ipvs_property: skbuff is owned by ipvs
 *	@peeked: this packet has been seen already, so stats have been
 *		done for it, don't do them again
 *	@nf_trace: netfilter packet trace flag 
 *	@protocol: Packet protocol from driver，重要的字段，即链路层承载的三层协议类型，如ip ipv6 arp,在if_ether.h中有定义，这个用来确定收包时传给哪个协议处理函数，因此必须提前初始化；
 *	@destructor: Destruct function  析构函数
 *	@nfct: Associated connection, if any
 *	@nf_bridge: Saved data about a bridged frame - see br_netfilter.c
 *	@skb_iif: ifindex of device we arrived on
 *	@tc_index: Traffic control index  输入流量控制
 *	@tc_verd: traffic control verdict  输入流量控制
 *	@hash: the packet hash
 *	@queue_mapping: Queue mapping for multiqueue devices
 *	@xmit_more: More SKBs are pending for this queue
 *	@pfmemalloc: skbuff was allocated from PFMEMALLOC reserves
 *	@ndisc_nodetype: router type (from link layer)
 *	@ooo_okay: allow the mapping of a socket to a queue to be changed
 *	@l4_hash: indicate hash is a canonical 4-tuple hash over transport
 *		ports.
 *	@sw_hash: indicates hash was computed in software stack
 *	@wifi_acked_valid: wifi_acked was set
 *	@wifi_acked: whether frame was acked on wifi or not
 *	@no_fcs:  Request NIC to treat last 4 bytes as Ethernet FCS
 *	@dst_pending_confirm: need to confirm neighbour
  *	@napi_id: id of the NAPI struct this skb came from
 *	@secmark: security marking
 *	@offload_fwd_mark: fwding offload mark
 *	@mark: Generic packet mark
 *	@vlan_proto: vlan encapsulation protocol
 *	@vlan_tci: vlan tag control information
 *	@inner_protocol: Protocol (encapsulation)
 *	@inner_transport_header: Inner transport layer header (encapsulation) 传输层首部
 *	@inner_network_header: Network layer header (encapsulation) 网络层首部
 *	@inner_mac_header: Link layer header (encapsulation) 链路层首部，这几个首部，在层与层传输中，data会发生指向的改变；
 *	@transport_header: Transport layer header
 *	@network_header: Network layer header
 *	@mac_header: Link layer header 
 *	@tail: Tail pointer
 *	@end: End pointer
 *	@head: Head of buffer
 *	@data: Data head pointer
 *	@truesize: Buffer size  整个缓存区长度 len+sizeof(sk_buff)
 *	@users: User count - see {datagram,tcp}.c 引用计数，为0时才释放此结构，否则只是单纯的递增递减，skb_get(),kfree_skb()等操作函数；
 */

struct sk_buff {
	union {
		struct {
			/* These two members must be first. */
			struct sk_buff		*next;
			struct sk_buff		*prev;

			union {
				ktime_t		tstamp;
				struct skb_mstamp skb_mstamp;
			};
		};
		struct rb_node		rbnode; /* used in netem, ip4 defrag, and tcp stack */
	};

	union {
		struct sock		*sk; 
		int			ip_defrag_offset;
	};

	struct net_device	*dev;

	/*
	 * This is the control buffer. It is free to use for every
	 * layer. Please put your private variables there. If you
	 * want to keep them across layers you have to do a skb_clone()
	 * first. This is owned by whoever has the skb queued ATM.
	 */
	char			cb[48] __aligned(8);

	unsigned long		_skb_refdst;
	void			(*destructor)(struct sk_buff *skb);
#ifdef CONFIG_XFRM
	struct	sec_path	*sp;
#endif
#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
	struct nf_conntrack	*nfct;
#endif
#if IS_ENABLED(CONFIG_BRIDGE_NETFILTER)
	struct nf_bridge_info	*nf_bridge;
#endif
	unsigned int		len,
				data_len;
	__u16			mac_len,
				hdr_len;

	/* Following fields are _not_ copied in __copy_skb_header()
	 * Note that queue_mapping is here mostly to fill a hole.
	 */
	kmemcheck_bitfield_begin(flags1);
	__u16			queue_mapping;
	__u8			cloned:1,
				nohdr:1,
				fclone:2,
				peeked:1,
				head_frag:1,
				xmit_more:1,
				pfmemalloc:1;
	kmemcheck_bitfield_end(flags1);

	/* fields enclosed in headers_start/headers_end are copied
	 * using a single memcpy() in __copy_skb_header()
	 */
	/* private: */
	__u32			headers_start[0];
	/* public: */

/* if you move pkt_type around you also must adapt those constants */
#ifdef __BIG_ENDIAN_BITFIELD
#define PKT_TYPE_MAX	(7 << 5)
#else
#define PKT_TYPE_MAX	7
#endif
#define PKT_TYPE_OFFSET()	offsetof(struct sk_buff, __pkt_type_offset)

	__u8			__pkt_type_offset[0];
	__u8			pkt_type:3;
	__u8			ignore_df:1;
	__u8			nfctinfo:3;
	__u8			nf_trace:1;

	__u8			ip_summed:2;
	__u8			ooo_okay:1;
	__u8			l4_hash:1;
	__u8			sw_hash:1;
	__u8			wifi_acked_valid:1;
	__u8			wifi_acked:1;
	__u8			no_fcs:1;

	/* Indicates the inner headers are valid in the skbuff. */
	__u8			encapsulation:1;
	__u8			encap_hdr_csum:1;
	__u8			csum_valid:1;
	__u8			csum_complete_sw:1;
	__u8			csum_level:2;
	__u8			csum_bad:1;
	__u8			dst_pending_confirm:1;
#ifdef CONFIG_IPV6_NDISC_NODETYPE
	__u8			ndisc_nodetype:2;
#endif
	__u8			ipvs_property:1;

	__u8			inner_protocol_type:1;
	__u8			remcsum_offload:1;
	/* 3 or 5 bit hole */

#ifdef CONFIG_NET_SCHED
	__u16			tc_index;	/* traffic control index */
#ifdef CONFIG_NET_CLS_ACT
	__u16			tc_verd;	/* traffic control verdict */
#endif
#endif

	union {
		__wsum		csum;
		struct {
			__u16	csum_start;
			__u16	csum_offset;
		};
	};
	__u32			priority;
	int			skb_iif;
	__u32			hash;
	__be16			vlan_proto;
	__u16			vlan_tci;
#if defined(CONFIG_NET_RX_BUSY_POLL) || defined(CONFIG_XPS)
	union {
		unsigned int	napi_id;
		unsigned int	sender_cpu;
	};
#endif
	union {
#ifdef CONFIG_NETWORK_SECMARK
		__u32		secmark;
#endif
#ifdef CONFIG_NET_SWITCHDEV
		__u32		offload_fwd_mark;
#endif
	};

	union {
		__u32		mark;
		__u32		reserved_tailroom;
	};

	union {
		__be16		inner_protocol;
		__u8		inner_ipproto;
	};

	__u16			inner_transport_header;
	__u16			inner_network_header;
	__u16			inner_mac_header;

	__be16			protocol;
	__u16			transport_header;
	__u16			network_header;
	__u16			mac_header;

	/* private: */
	__u32			headers_end[0];
	/* public: */

	/* These elements must be at the end, see alloc_skb() for details.  */
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head,
				*data;
	unsigned int		truesize;
	atomic_t		users;
};
```

一张图片来表示skb的线性缓存区：发送和接收时会动态变化；
struct sk_buff
...
head --->headroom起点
data --->data起点
tail --->tailroom起点
end  --->skb_shared_info起点
|headroom|data|tailroom|skb_shared_info|

### skb_shared_info结构
#### 基本结构
```cpp
/* This data is invariant across clones and lives at
 * the end of the header data, ie. at skb->end.
 */ //即end指针指向的
struct skb_shared_info {
	unsigned char	nr_frags;
	__u8		tx_flags;
	unsigned short	gso_size;
	/* Warning: this field is not always filled in (UFO)! */
	unsigned short	gso_segs;
	unsigned short  gso_type;
	struct sk_buff	*frag_list;
	struct skb_shared_hwtstamps hwtstamps;
	u32		tskey;
	__be32          ip6_frag_id;

	/*
	 * Warning : all fields before dataref are cleared in __alloc_skb()
	 */
	atomic_t	dataref;//引用计数器：当一个数据缓存区被多个skb描述符引用时，就会设置相应的计数，如克隆一个skb

	/* Intermediate layers must ensure that destructor_arg
	 * remains valid until skb destructor */
	void *		destructor_arg;

	/* must be last field, see pskb_expand_head() */
	skb_frag_t	frags[MAX_SKB_FRAGS];
};
```

#### 通过网络接口在两个本地文件间传输数据下，如何避免四次拷贝
在数据发送到接收时，往往send/write ->recv/read 这期间会涉及四次拷贝：
发送app用户进程空间缓存-->内核态缓存-（DMA复制)->硬件驱动 ----发送----接收方硬件网络模块-(DMA复制)->内核态缓存-->用户空间缓存；
如果是本地文件--网络接口---本地文件，这种也是四次拷贝，就有点耗费时间了；（或者是收到数据原封不动传输出去,这个貌似还没实现）
调用方式改变：从read/write====>open/sendfile
实际路径改变：接收硬件网络模块(DMA复制)-->内核态缓存---> 发送硬件驱动


#### 什么是聚合分散io数据以及skb_shared_info中对它支持的结构；
这里的聚合分散IO相关数据成员：
struct sk_buff *frag_list; //FRAGLIST类型的IP分片相关结构
unsigned short nr_frags; //聚合分散IO分片数量
skb_frag_t frags[MAX_SKB_FRAGS];//聚合分散IO page相关结构指针；
先理解下网络发送接收和组合分片相关的流程：
发送：用户数据-->四层tcp/udp-->IP:此时包需要一层层传递组合各层头，需要进行比较多的拷贝，此时是组合；在内核传递发送报文给hard_start_xmit()之前，需要判断网络是否
支持NETIF_F_SG,否则只能线性化处理，是则检查nr_frags的值，确定片段数，之后进行分片聚合；
接受：IP分片--IP分片重组-->udp/tcp-->用户；
```cpp
/* To allow 64K frame to be packed as single skb without frag_list we
 * require 64K/PAGE_SIZE pages plus 1 additional page to allow for
 * buffers which do not start on a page boundary.
 *
 * Since GRO uses frags we allocate at least 16 regardless of page
 * size.
 */
#if (65536/PAGE_SIZE + 1) < 16
#define MAX_SKB_FRAGS 16UL
#else
#define MAX_SKB_FRAGS (65536/PAGE_SIZE + 1)
#endif
//所有最多支持64K个分片
typedef struct skb_frag_struct skb_frag_t;

struct skb_frag_struct {
	struct {
		struct page *p;
	} page;
#if (BITS_PER_LONG > 32) || (PAGE_SIZE >= 65536)
	__u32 page_offset;
	__u32 size;
#else
	__u16 page_offset;
	__u16 size;
#endif
};
```
+ frag_list的使用：
1) 在接受分片组后链接多个分片，组成一个完整的IP数据报；
2) UDP数据输出时，将待分片的SKB链接到一个SKB中，然后在输出时快速分片；
3) 用于存放FRAGLIST类型的聚合分散IO数据包；
+ nr_frag SG类型的聚合分散IO的使用：
1) 在输出时需要判断nr_frag=0? 且frag_list==NULL，则没有分片，若nr_frag不为0且frag_list为NULL,则是聚合分散IO;
nr_frag表示数量，内容由frags数组指出：eg:
nr_frag =2; frags[0].page=p1, frags[0].page_offset=0,size=S1; frags[1].page=p1,frags[1].page_offset=S1,size=S2;
2) 不同的skb实例中的page指向相同的内存，即共享分片结构（共享内存),这个需要两方都不去修改它；
+ FRAGLIST类型的聚合分散IO
 frag_lsit直接指向了一个skb结构的实例；

#### 关于GSO/GRO/TSO/TRO的基本概念；
#### 如何访问skb_shared_info结构：
可以借助skb_info宏来访问此结构： 它其实就是简单的返回sk_buff结构的end指针的类型转换结果；
#define skb_shinfo(SKB) ((struct skb_shared_info*) ((SKB)->end))
eg: skb_shinfo(skb)->dataref++;

### skb的相关管理函数：
内核提供了很多小函数来操作skb变量和链表等，这些函数都有两个版本：do_something和__do_something(),前者是在调用后者的情况下加上
锁和检查等；一般使用用前者的；这些函数定义在skbuff.h和skbuff.c中，我们称为skb的管理操作函数；
以下介绍每个类型的函数，并给出一些简单的例子，可以是demo或者是内核中的相关调用；
#### SKB的缓存池介绍
在网络模块中，用告诉缓存来为skb分配空间，在初始化skb_init()中，创建了两个用来分配skb的高速缓存：
```cpp
void __init skb_init(void)
{
	skbuff_head_cache = kmem_cache_create("skbuff_head_cache",
					      sizeof(struct sk_buff),
					      0,
					      SLAB_HWCACHE_ALIGN|SLAB_PANIC,
					      NULL);
	skbuff_fclone_cache = kmem_cache_create("skbuff_fclone_cache",
						sizeof(struct sk_buff_fclones),
						0,
						SLAB_HWCACHE_ALIGN|SLAB_PANIC,
						NULL);
}
/* Layout of fast clones : [skb1][skb2][fclone_ref] */
struct sk_buff_fclones {
	struct sk_buff	skb1;

	struct sk_buff	skb2;

	atomic_t	fclone_ref;
};

```
一般情况下用第一个，当在分配skb的时候知道可能被克隆，则用第二个，因为第二个中会同时分配一个后备的skb，在克隆的时候直接用后备的skb就可以，不用重新分配；
可以看到第二个的单位内存区域是2*size+atomic_t,后者是用来做引用计数的；引用计数取0,1,2分别代表两个都没有被引用，1表示引用了其中一个，2表示两个都被引用；
#### 如何分配SKB
1) alloc_skb()
skb的数据缓存区和skb本身描述符是两个不同的实体，所以在分配一个skb时，实际上需要分配两块内存：
一个是skb描述符，一个是数据缓存区；在4.8的内核版本可以看到：
```cpp
static inline struct sk_buff *alloc_skb(unsigned int size,
					gfp_t priority)
{
	return __alloc_skb(size, priority, 0, NUMA_NO_NODE);
}
```

函数传入一个size和priority,返回一个sk_buff指针；
而__alloc_skb()则有四个参数：
```cpp
/**
 *	__alloc_skb	-	allocate a network buffer
 *	@size: size to allocate 分配的长度
 *	@gfp_mask: allocation mask  分配内存的方式
 *	@flags: If SKB_ALLOC_FCLONE is set, allocate from fclone cache
 *		instead of head cache and allocate a cloned (child) skb.
 *		If SKB_ALLOC_RX is set, __GFP_MEMALLOC will be used for
 *		allocations in case the data is required for writeback 
        预测是否会克隆，以此判断是从哪个告诉缓存中分配
        若 SKB_ALLOC_RX is set。。。
 *	@node: numa node to allocate memory on 当支持NUMA时，用于确定从哪个内存区中分配SKB
 *
 *	Allocate a new &sk_buff. The returned buffer has no headroom and a
 *	tail room of at least size bytes. The object has a reference count
 *	of one. The return is the buffer. On a failure the return is %NULL.
 *
 *	Buffers may only be allocated from interrupts using a @gfp_mask of
 *	%GFP_ATOMIC.
 */
struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask,
			    int flags, int node)
{
	struct kmem_cache *cache;
	struct skb_shared_info *shinfo;
	struct sk_buff *skb;
	u8 *data;
	bool pfmemalloc;

	cache = (flags & SKB_ALLOC_FCLONE)
		? skbuff_fclone_cache : skbuff_head_cache;//判断是用哪种高速缓存，上面已解释

	if (sk_memalloc_socks() && (flags & SKB_ALLOC_RX))//判断是否可以用紧急的缓存
		gfp_mask |= __GFP_MEMALLOC;

	/* Get the HEAD */
	skb = kmem_cache_alloc_node(cache, gfp_mask & ~__GFP_DMA, node);//从选定的告诉缓存中分配一个SKB描述符,一般在slab分配
	if (!skb)
		goto out;
	prefetchw(skb);

	/* We do our best to align skb_shared_info on a separate cache
	 * line. It usually works because kmalloc(X > SMP_CACHE_BYTES) gives
	 * aligned memory blocks, unless SLUB/SLAB debug is enabled.
	 * Both skb->head and skb_shared_info are cache line aligned.
	 */
	size = SKB_DATA_ALIGN(size);//给size先做下对齐
	size += SKB_DATA_ALIGN(sizeof(struct skb_shared_info));//加上缓存区尾部的skb_shared_info结构后
	data = kmalloc_reserve(size, gfp_mask, node, &pfmemalloc);//分配数据缓存区
	if (!data)
		goto nodata;
	/* kmalloc(size) might give us more room than requested.
	 * Put skb_shared_info exactly at the end of allocated zone,
	 * to allow max possible filling before reallocation.
	 */
	size = SKB_WITH_OVERHEAD(ksize(data));
	prefetchw(data + size);

	/*
	 * Only clear those fields we need to clear, not those that we will
	 * actually initialise below. Hence, don't put any more fields after
	 * the tail pointer in struct sk_buff!
	 */
     //初始化
	memset(skb, 0, offsetof(struct sk_buff, tail));
	/* Account for allocated memory : skb + skb->head */
	skb->truesize = SKB_TRUESIZE(size);
	skb->pfmemalloc = pfmemalloc;
	atomic_set(&skb->users, 1);
	skb->head = data;
	skb->data = data;
	skb_reset_tail_pointer(skb);
	skb->end = skb->tail + size;
	skb->mac_header = (typeof(skb->mac_header))~0U;
	skb->transport_header = (typeof(skb->transport_header))~0U;

	/* make sure we initialize shinfo sequentially */
	shinfo = skb_shinfo(skb);
	memset(shinfo, 0, offsetof(struct skb_shared_info, dataref));
	atomic_set(&shinfo->dataref, 1);
	kmemcheck_annotate_variable(shinfo->destructor_arg);
    //置克隆相关位
	if (flags & SKB_ALLOC_FCLONE) {
		struct sk_buff_fclones *fclones;

		fclones = container_of(skb, struct sk_buff_fclones, skb1);

		kmemcheck_annotate_bitfield(&fclones->skb2, flags1);
		skb->fclone = SKB_FCLONE_ORIG;
		atomic_set(&fclones->fclone_ref, 1);

		fclones->skb2.fclone = SKB_FCLONE_CLONE;
		fclones->skb2.pfmemalloc = pfmemalloc;
	}
out:
	return skb;
nodata:
	kmem_cache_free(cache, skb);
	skb = NULL;
	goto out;
}
EXPORT_SYMBOL(__alloc_skb);
```
注意：__alloc_skb通常不被直接调用，而是封装调用，被封装在__netdev_alloc_skb(),alloc_skb(),alloc_skb_fclone()等；
dev_alloc_skb()也是一个缓存区分配函数，也是返回sk_buff*,通常在设备驱动中断上下文中，是一个alloc_skb()的封装函数，因为在中断处理函数中被调用，所以要求
原子操作(GFP_ATOMIC),4.8版本已经变了，封装的是别的函数，但是类似；

#### 如何释放SKB
dev_kfree_skb()和kfree_skb()都可以用来释放skb，把它返回给高速缓存；dev_kfree_skb()只是简单的封装kfree_skb();
调用时只是简单的递减skb->users的值，直到减完为0才真的释放内存；
具体见skbuff.h/skbuff.c



#### 数据预留和对齐
主要是通过skb_reserve(),skb_put,skb_push,skb_pull,等函数，对数据缓存区相关指针进行更新来做预留空间的操作；
具体怎么移动，要看是接收方向还是发送方向：
1) skb_reserve()
skb_reserve()通常用来在数据缓存区头部预留一定的空间，比如插入协议首部或者在某个边界上对齐，而预留操作主要是移动data和tail指针；
注意只能用于空的skb，所以通常在分配后就会调用该函数；此时tail和data一同指向数据区的起始位置；
例如：某个以太网设备驱动接收函数在分配skb后，在向skb缓存区填充数据前，会通过skb_reserve(skb,2)来预留两个字节，因为以太网首部是14B,加了后正好16B对齐；此时是data指针往下移动两个字节；
当数据从上往下传递时，则每层将skb->data指针往上移动，然后复制本层首部，更新len;
2) skb_push()
在数据缓存区前头增加一块数据。也是只移动data和tail指针，和reserve类似
3) skb_put()
修改指向末尾的tail指针，使之向下移动len字节，然后更新len
4) skb_pull()
data指针向下移动；从而在数据区首部忽略len字节长度，用于收到包后往上传递忽略首部；
#### 克隆和复制SKB
1) skb_clone() 用来克隆skb，克隆时只复制skb描述符，而对数据缓存区则，引用计数+1；
2) pskb_copy() 当一个函数不仅要修改skb描述符，还要修改数据缓存区的时候，就需要同时复制数据缓存区；如果要修改的数据在head-end之间，就可以用这个函数，
不然若还要修改聚合分散IO中的数据，则用skb_copy()做完全的拷贝；

#### SKB链表的管理函数
在skb链表操作中，为了防止被中断，则必须先获得自旋锁；然后才能访问队列中的元素；
skb_queue_head_init() :初始化skb链表头结点，创建一个空的skb链表；
skb_queue_head()/skb_queue_tail(),加入队列头/尾部，
skb_dequeue和skb_dequeue_tail，从首部和尾部取下一个skb;
skb_queue_purge() ,清空skb链表；
skb_queue_walk() 循环遍历skb链表中每个元素的宏；
#### 添加和删除尾部数据
注意这里指的尾部数据，是data指向数据的结尾，而tail指向的是结尾的空间部分，一般是空的；
skb_add_data() tail指针下移，data尾部增加用户空间传递的数据，len加加；
skb_trim()和上面相反；
pskb_trim() 也处理到聚合分散iO;

#### 拆分数据
就是把一个skb拆成两个skb:
skb_split();分两种情况，一种是拆分的len小于线性缓存区长(即不包含聚合分散IO的)，另一种是大于，即分拆点在聚合分散IO中的某个位置；
#### 重新分配SKB的线性数据区；
pskb_expand_head(),可以理解为realloc,就是扩展空间，将原数据复制过去；
#### 其他函数；
pskb_may_pull: 检测skb中的数据是否有指定长度
skb_queue_empty:检测skb队列是否为空
skb_realloc_headroom: 根据指定的skb得到一个新的skb,并确保新的skb存在指定的headroom空间；
skb_get() :引用并返回一个skb;
skb_shared() :检测指定的skb是否被多次引用；
skb_share_check():检测并返回skb,当被检测的skb被引用多次时，则克隆此skb,并返回克隆得到的skb;
skb_unshare():检测并返回skb,当被检查的skb被克隆时，则复制此skb,并返回复制得到的skb;
skb_orphan(): 使得此skb不属于任何传输控制块；
skb_cow(): 确保skb存在指定的head空间，若不足，则会重新分配
skb_pagelen(): 获得skb中线性数据区和SG类型聚合分散io分片中的数据的长度；
#### 关于虚拟设备和物理设备：
linux支持多种形式的虚拟网络设备，并通过一个虚拟网络设备驱动管理。当这个虚拟设备被使用时，dev指针指向该虚拟设备的net_device结构。
在输出时，虚拟设备驱动会在一组设备中选择其中的某个合适的设备，并将dev指针修改为指向这个设备的net_device结构；
在输入时，当原始网络设备收到报文后，根据某种算法选择某个合适的虚拟网络设备，并将dev指针修改为指向这个虚拟设备的net_device结构。
因此，在某些情况下，指向传输设备的指针会在包处理过程中改变；









