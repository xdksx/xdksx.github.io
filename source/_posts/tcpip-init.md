---
title: tcpip_init
date: 2021-03-21 07:11:45
tags: tcpip_init
categories: tcpip
---

### linux内核网络子系统的初始化介绍
基于linux4.8,版本其实现在已经是5.12了，但是因为我装的linux源码版本是4.8，为方便调试，都在这个版本分析，差别不会很大；
以下的内核都指的这个linux4.8的内核；环境都在这个上的；<!--more-->
相关地址：
源码查阅：https://elixir.bootlin.com/linux/latest/source
#### linux内核网络子系统初始化组成
+ 内核的初始化和各种init函数集合
+ 网络文件系统等初始化：sock_init
+ 协议栈及相关函数结构初始化：inet_init
+ 设备相关初始化

#### 内核的初始化过程整体和init函数集合
内核的初始化，其实就是对应的启动linux的时候，各种数据结构，设备等等初始化的过程；更具体的涉及更多复杂细节，这里只展示关键部分；
内核启动时，会进入到head.S/head.c,这个和架构有关，每种linux支持的架构都会有对应的实现文件：
在源码查阅中可以看到：
```cpp
/arch/alpha/kernel/head.S
/arch/arm/kernel/head.S
...
```
在这些文件中，都能找到其调用start_kernel，这个函数就是用来启动内核；
arch/x86/kernel/head64.c, line 195为例：
可以看到调用启动内核的函数：
```cpp
start_kernel();
进而找到：
    init/main.c
    简单分析start_kernel函数：截取其中的片段，可以看到其调用各种关键系统的初始化函数：
    setup_log_buf(0);
	pidhash_init();
	vfs_caches_init_early();
	sort_main_extable();
	trap_init();
	mm_init();
    ...
    cred_init();
	fork_init();
	proc_caches_init();
	buffer_init();
	key_init();
	security_init();
	dbg_late_init();
	vfs_caches_init(totalram_pages);
	signals_init();
    ...
    rest_init();
    从名字大概可以知道是初始化哪部分内容，这里每部分深入进去都是很长的内容，这里只去找网络相关的，看起来在rest_init部分了；
    rest_init:
    {
        .... //可以看到转门新建了一个内核线程来初始化；
        kernel_thread(kernel_init, NULL, CLONE_FS);
	    numa_default_policy();
	    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	    rcu_read_lock();

        kernel_init
        {
            //初始化各种init函数
            kernel_init_freeable();
               ->do_basic_setup
                 ->driver_init();
                  -->do_initcalls(); //这里会通过依赖编译器的方式，寻找到内核中所有的_init 表识的函数，如sock_init,inet_init函数等
                  ‘static int __init sock_init(void)’ 
                     从而能初始化各种init函数，这种机制，涉及到ld链接脚本，gcc编译器本身的机制，以及elf文件结构，内容较多，等有机会再具体写写
                    这里截取一段：通过objdump vmlinux后看到的能找到的init函数
                    ffffffff820a48e8 l     O .init.data	0000000000000008 __initcall_sock_init1 关键函数1
                    ffffffff820a4e18 l     O .init.data	0000000000000008 __initcall_proto_init4
                    ffffffff820a48f0 l     O .init.data	0000000000000008 __initcall_net_inuse_init1
                    ffffffff820a4788 l     O .init.data	0000000000000008 __initcall_net_ns_init0
                    ffffffff820a48f8 l     O .init.data	0000000000000008 __initcall_net_defaults_init1
                    ffffffff820a4900 l     O .init.data	0000000000000008 __initcall_init_default_flow_dissectors1
                    ffffffff820a5020 l     O .init.data	0000000000000008 __initcall_sysctl_core_init5
                    ffffffff820a4e20 l     O .init.data	0000000000000008 __initcall_net_dev_init4 关键函数2
                    ffffffff820a4e28 l     O .init.data	0000000000000008 __initcall_neigh_init4
                    ffffffff820a5030 l     O .init.data	0000000000000008 __initcall_inet_init5 关键函数3
            //拉取init进程
            if (execute_command) {
		    ret = run_init_process(execute_command);
		    if (!ret)
		    	return 0;
		    panic("Requested init %s failed (error %d).",
		          execute_command, ret);
	        }
	    if (!try_to_run_init_process("/sbin/init") ||
	        !try_to_run_init_process("/etc/init") ||
	        !try_to_run_init_process("/bin/init") ||
	        !try_to_run_init_process("/bin/sh"))
	    	return 0;
            }
       }
    }
```


#### 网络文件系统等初始化：sock_init
##### 代码流程分析
###### 整体代码：

```cpp
static int __init sock_init(void)
{
	int err;
	/*
	 *      Initialize the network sysctl infrastructure.
	 */
	err = net_sysctl_init();
    //初始化网络相关的/proc/sys/下的目录：
    /*
    /* Avoid limitations in the sysctl implementation by
	 * registering "/proc/sys/net" as an empty directory not in a
	 * network namespace.
	 
	net_header = register_sysctl("net", empty);
    */
    //初始化网络空间相关操作等，网络空间用于docker时，每个docker容器之间的网络隔离，类似的还有文件系统隔离等带来的文件系统空间等等；
    //ret = register_pernet_subsys(&sysctl_pernet_ops);
    //register_sysctl_root(&net_sysctl_root);
	if (err)
		goto out;

	/*
	 *      Initialize skbuff SLAB cache
	 */
	skb_init();
    //其实就是调用了kmem_cache_create初始化了skbuff,方便之后直接用kmem_cache_alloc分配skbuff
    //关于kmem_cache_create使用：https://docs.oracle.com/cd/E36784_01/html/E36886/kmem-cache-alloc-9f.html
	/*
	 *      Initialize the protocols module.
	 */
    
	init_inodecache();
    //初始化：kmem_cache_create: socket_alloc:
    /*sock_inode_cachep = kmem_cache_create("sock_inode_cache",
					      sizeof(struct socket_alloc),
					      0,
					      (SLAB_HWCACHE_ALIGN |
					       SLAB_RECLAIM_ACCOUNT |
					       SLAB_MEM_SPREAD | SLAB_ACCOUNT),
					      init_once);
    可以看到这个结构体，是包含了 socket和inode:所有操作网络的接口fd可以通过write等文件系统的函数和socket的特定函数send等
    struct socket_alloc {
	struct socket socket;
	struct inode vfs_inode;
    };
*/
	err = register_filesystem(&sock_fs_type);//注册socketfs网络文件系统
	if (err)
		goto out_fs;
	sock_mnt = kern_mount(&sock_fs_type);//挂载网络文件系统，主要调用了通用的接口：vfs_kern_mount(type, MS_KERNMOUNT, type->name, data);
	if (IS_ERR(sock_mnt)) {
		err = PTR_ERR(sock_mnt);
		goto out_mount;
	}

	/* The real protocol initialization is performed in later initcalls.
	 */

#ifdef CONFIG_NETFILTER
	err = netfilter_init();//若配置了netfilter，需要进一步初始化
	if (err)
		goto out;
#endif

	ptp_classifier_init();

out:
	return err;

out_mount:
	unregister_filesystem(&sock_fs_type);
out_fs:
	goto out;
}
```

要真正理解上面的代码，需要了解：  
1 内存管理相关的，理解kmem_cache高速缓存  --已独立文章分析  
2 文件系统相关的，理解socketfs是什么存在，以及如何管理sockinode; --有独立文章分析文件系统，待分析socketfs  
这里简单分析：socketfs是一种特殊的文件系统  
特殊文件系统：通常是Linux为了方便计算机管理或者提供某些服务而编写。典型的有proc、tmpfs、pipefs、sockfs等。  
所以sockfs就是为了套接字而设计的伪文件系统，在socket.c中  

```c
static struct file_system_type sock_fs_type = {
         .name    =  "sockfs",
         .mount   =  sockfs_mount,
         .kill_sb =  kill_anon_super,
};

```
从平常对套接字的使用，如可以用协议栈相关函数connect,bind accept,还可以用read,write进行发送和接收等，很明显后者是读写文件的函数；
从这里可以看到socket套接字兼顾两种特性；  sockfs是一个文件系统，自然支持inode，同时socket本身的结构支持协议栈函数；  

使用socket函数创建socket时：  
socket结构初始化：从alloc_inode ->调用sock_alloc_inode:创建如下结构体：  

```c
struct socket_alloc {
         struct socket socket;
         struct inode vfs_inode;
};
```
可以看到包含socket,和inode,同时socket结构包含file结构

```c
struct socket {
        …
        struct file *file;
        struct sock *sk;
        const struct proto_ops *ops;
        …
};
```
socket 成员面向的是协议栈，而 vfs_inode 成员面向的是文件系统，这体现了套接字的双重属性。struct file 则是将两者联系起来的枢纽，其面向的是进程。这样就把内核中的套接字，抽象成为一个简单的文件描述符，提供给用户空间使用。  

struct inode由于代表了文件系统中一个实际的文件在内存中的反映，已经不属于进程的范畴，所以 struct inode 不会有上面所谓的共享问题。但是 struct inode 只会在必要的时候创建，在允许的情况下销毁。  

3 sysctl相关的，如何 /proc/net/目录 

##### 数据结构图
这部分涉及到几个数据结构：  
1 skbuff: kmem_cache_create后产生的两个头：skbuff_head_cache，skbuff_fclone_cache
为后面分配skbuff做准备；  
2 sysctl:  
```cpp
proc_sysctl.c
static struct ctl_table_root sysctl_table_root = {
	.default_set.dir.header = {
		{{.count = 1,
		  .nreg = 1,
		  .ctl_table = root_table }},
		.ctl_table_arg = root_table,
		.root = &sysctl_table_root,
		.set = &sysctl_table_root.default_set,
	},
};
```
3 socketfs相关：  
sock_fs_type


#### 协议栈及相关函数结构初始化：inet_init
用户层通过socket函数创建socket,会传入三个参数：
```cpp
int socket(int domain, int type, int protocol);
domain:
指定协议簇，ipv4/ipv6..
AF_INET      IPv4 Internet protocols 

type: 
指定udp/tcp
SOCK_STREAM     Provides sequenced, reliable, two-way, connection-based byte streams.  
                An out-of-band data transmission mechanism may be supported.
SOCK_DGRAM      Supports datagrams (connectionless, unreliable messages of a fixed maximum length).

protocol:
通常只存在一个协议来支持 给定协议族中的特定套接字类型，在这种情况下，protocol可以指定为0。
```
然后会返回一个fd来唯一标识这个socket，那内核如何根据传入的三个参数，来选择正确的匹配协议和传输协议类型的相关函数呢？
在收到包后，如何根据收到的包，找到匹配的协议相关函数呢？  
在下面的inet_init初始化后，就搭建好了这个基本的协议栈函数；  

##### 代码结构分析
```cpp
static int __init inet_init(void)
{
	struct inet_protosw *q;
	struct list_head *r;
	int rc = -EINVAL;

	sock_skb_cb_check_size(sizeof(struct inet_skb_parm));

    //将tcp_prot,udp_prot，注册到(添加到)prot_list链表中；
    //list_add(&prot->node, &proto_list); 将prot结构挂到链表上；
	rc = proto_register(&tcp_prot, 1);
	if (rc)
		goto out;

	rc = proto_register(&udp_prot, 1);
	if (rc)
		goto out_unregister_tcp_proto;

	rc = proto_register(&raw_prot, 1);
	if (rc)
		goto out_unregister_udp_proto;

	rc = proto_register(&ping_prot, 1);
	if (rc)
		goto out_unregister_raw_proto;

	/*
	 *	Tell SOCKET that we are alive...
	 */
     
	(void)sock_register(&inet_family_ops);
    /*
    //将inet_family_ops注册到地址簇列表中
	(void)sock_register(&inet_family_ops);
    rcu_assign_pointer(net_families[ops->family], ops); //其实就是把ops放到对应的表net_family中，这个是全局变量，下面会解释
    */
#ifdef CONFIG_SYSCTL
	ip_static_sysctl_init();
#endif

	/*
	 *	Add all the base protocols.
	 */
    //下面填充 inet_protos[protocol]=struct net_protocol结构；
	if (inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) < 0)
		pr_crit("%s: Cannot add ICMP protocol\n", __func__);
	if (inet_add_protocol(&udp_protocol, IPPROTO_UDP) < 0)
		pr_crit("%s: Cannot add UDP protocol\n", __func__);
	if (inet_add_protocol(&tcp_protocol, IPPROTO_TCP) < 0)
		pr_crit("%s: Cannot add TCP protocol\n", __func__);
#ifdef CONFIG_IP_MULTICAST
	if (inet_add_protocol(&igmp_protocol, IPPROTO_IGMP) < 0)
		pr_crit("%s: Cannot add IGMP protocol\n", __func__);
#endif
    //初始化和注册 inetsw[]
	/* Register the socket-side information for inet_create. */
	for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
		INIT_LIST_HEAD(r);

	for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
		inet_register_protosw(q);
    /# 这个函数展开看看,主要是将inetws_array中的元素添加到全局静态链表中inetsw，注意linux特殊的链表连接方式，是结构中的成员为一个结点；
    void inet_register_protosw(struct inet_protosw *p)
    {
	    struct list_head *lh;
	    struct inet_protosw *answer;
	    int protocol = p->protocol;
	    struct list_head *last_perm;
    
	    spin_lock_bh(&inetsw_lock);
    
	    if (p->type >= SOCK_MAX)
	    	goto out_illegal;
    
	    /* If we are trying to override a permanent protocol, bail. */
	    last_perm = &inetsw[p->type];//取出结构
	    list_for_each(lh, &inetsw[p->type]) {
	    	answer = list_entry(lh, struct inet_protosw, list);
	    	/* Check only the non-wild match. */
	    	if ((INET_PROTOSW_PERMANENT & answer->flags) == 0)
	    		break;
	    	if (protocol == answer->protocol)
	    		goto out_permanent;
	    	last_perm = lh;
	    }
    
	    /* Add the new entry after the last permanent entry if any, so that
	     * the new entry does not override a permanent entry when matched with
	     * a wild-card protocol. But it is allowed to override any existing
	     * non-permanent entry.  This means that when we remove this entry, the
	     * system automatically returns to the old behavior.
	     */
	    list_add_rcu(&p->list, last_perm);//将p->list加到取出的结构中
        ...
    }
    #/
	/*
	 *	Set the ARP module up
	 */
    //初始化arp和arp_packet_type注册：dev_add_pack(&arp_packet_type);
	arp_init();

	/*
	 *	Set the IP module up
	 */
    //ip路由表初始化
	ip_init();
    //tcp hashinfo相关初始化
	tcp_v4_init();

	/* Setup TCP slab cache for open requests. */
	tcp_init();

	/* Setup UDP memory threshold */
	udp_init();

	/* Add UDP-Lite (RFC 3828) */
	udplite4_register();
    
    //ping_table相关初始化
	ping_init();

	/*
	 *	Set the ICMP layer up
	 */

	if (icmp_init() < 0)
		panic("Failed to create the ICMP control socket.\n");

	/*
	 *	Initialise the multicast router
	 */
#if defined(CONFIG_IP_MROUTE)
	if (ip_mr_init())
		pr_crit("%s: Cannot init ipv4 mroute\n", __func__);
#endif

	if (init_inet_pernet_ops())
		pr_crit("%s: Cannot init ipv4 inet pernet ops\n", __func__);
	/*
	 *	Initialise per-cpu ipv4 mibs
	 */

	if (init_ipv4_mibs())
		pr_crit("%s: Cannot init ipv4 mibs\n", __func__);

	ipv4_proc_init();

	ipfrag_init();
    //注册ip_packet_type
	dev_add_pack(&ip_packet_type);

	rc = 0;
out:
	return rc;
out_unregister_raw_proto:
	proto_unregister(&raw_prot);
out_unregister_udp_proto:
	proto_unregister(&udp_prot);
out_unregister_tcp_proto:
	proto_unregister(&tcp_prot);
	goto out;
}
```
##### 数据结构图
int socket(int domain, int type, int protocol);//先放着参考 协议簇，协议类型，协议号
上述涉及几个结构，从发送和接收的方向来解释：
发送方向：

+ net_families[]:
  net_families[PF_INET]=inet_family_ops 上面主要初始化了这个
  
```cpp
  static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,
	.create = inet_create,
	.owner	= THIS_MODULE,
};
```
  但是其实还有如：
  /* Protocol families, same as address families. */
    #define PF_UNSPEC	AF_UNSPEC
    #define PF_UNIX		AF_UNIX
    #define PF_LOCAL	AF_LOCAL
    #define PF_INET		AF_INET
    #define PF_AX25		AF_AX25
    #define PF_IPX		AF_IPX
    int socket(int domain, int type, int protocol);
    ...
    还有这些协议簇，其实对应了上面socket函数传入的参数1；
    当我们调用socket创建socket套接字时：

```cpp
    int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)

     {
         ...
         pf = rcu_dereference(net_families[family]);//通过传入的第一个参数找到对应的ops
         err = pf->create(net, sock, protocol, kern);//调用ops的create函数来创建socket
         进而调用到套接口层的create函数创建套接口；
     }
```
+ inetsw，inetsw_array:
定义：
```cpp
static struct list_head inetsw[SOCK_MAX];
/* This is used to register socket interfaces for IP protocols.  */
struct inet_protosw {
	struct list_head list;

        /* These two fields form the lookup key.  */
	unsigned short	 type;	   /* This is the 2nd argument to socket(2). */
	unsigned short	 protocol; /* This is the L4 protocol number.  */

	struct proto	 *prot;
	const struct proto_ops *ops;
  
	unsigned char	 flags;      /* See INET_PROTOSW_* below.  */
};
```

后者初始化了一个静态的写死的表：
af_inet.c:
```cpp
/* Upon startup we insert all the elements in inetsw_array[] into
 * the linked list inetsw.
 */
static struct inet_protosw inetsw_array[] =
{
	{
		.type =       SOCK_STREAM,
		.protocol =   IPPROTO_TCP,
		.prot =       &tcp_prot,
		.ops =        &inet_stream_ops,
		.flags =      INET_PROTOSW_PERMANENT |
			      INET_PROTOSW_ICSK,
	},

	{
		.type =       SOCK_DGRAM,
		.protocol =   IPPROTO_UDP,
		.prot =       &udp_prot,
		.ops =        &inet_dgram_ops,
		.flags =      INET_PROTOSW_PERMANENT,
       },

       {
		.type =       SOCK_DGRAM,
		.protocol =   IPPROTO_ICMP,
		.prot =       &ping_prot,
		.ops =        &inet_dgram_ops,
		.flags =      INET_PROTOSW_REUSE,
       },

       {
	       .type =       SOCK_RAW,
	       .protocol =   IPPROTO_IP,	/* wild card */
	       .prot =       &raw_prot,
	       .ops =        &inet_sockraw_ops,
	       .flags =      INET_PROTOSW_REUSE,
       }
};
以udp为例：
const struct proto_ops inet_dgram_ops = {
	.family		   = PF_INET,
	.owner		   = THIS_MODULE,
	.release	   = inet_release,
	.bind		   = inet_bind,
	.connect	   = inet_dgram_connect,
	.socketpair	   = sock_no_socketpair,
	.accept		   = sock_no_accept,
	.getname	   = inet_getname,
	.poll		   = udp_poll,
    ....
}
struct proto udp_prot = {
	.name		   = "UDP",
	.owner		   = THIS_MODULE,
	.close		   = udp_lib_close,
	.connect	   = ip4_datagram_connect,
	.disconnect	   = udp_disconnect,
	.ioctl		   = udp_ioctl,
	.destroy	   = udp_destroy_sock,
	.setsockopt	   = udp_setsockopt,
	.getsockopt	   = udp_getsockopt,
	.sendmsg	   = udp_sendmsg,
	.recvmsg	   = udp_recvmsg,
    ....
}


```
结构的作用，是对传入的参数2： type 协议类型进行函数匹配：
继续上面的，调用到create，其实是：inet_create函数；
这个函数会创建sock结构，并将从 inetsw_array匹配到的 SOCK_DGRAM/SOCK_STREAM/..对应的ops，进行赋值等；
```cpp
static int inet_create(struct net *net, struct socket *sock, int protocol,
		       int kern)
{
	struct sock *sk;
	struct inet_protosw *answer;
	struct inet_sock *inet;
	struct proto *answer_prot;
	unsigned char answer_flags;
    ...
    list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {

		err = 0;
		/* Check the non-wild match. */
		if (protocol == answer->protocol) {
			if (protocol != IPPROTO_IP)
				break;
		} else {
			/* Check for the two wild cases. */
			if (IPPROTO_IP == protocol) {
				protocol = answer->protocol;
				break;
			}
			if (IPPROTO_IP == answer->protocol)
				break;
		}
		err = -EPROTONOSUPPORT;
	}
    sock->ops = answer->ops;//关键步骤，所以之后的函数，可以通过ops找到
	answer_prot = answer->prot;//关键步骤，所以之后的函数，可以通过prot找到
	answer_flags = answer->flags;
    sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot);
```

接收方向：
+ ptype_base,ptype_all,arp_packet_type,ip_packet_type
在接收到skbuff后确定是哪种三层包，ip还是arp;,从而找到对应的接收函数；
```cpp
static struct packet_type arp_packet_type __read_mostly = {
	.type =	cpu_to_be16(ETH_P_ARP),
	.func =	arp_rcv,
};
dev_add_pack(&ip_packet_type);
static struct packet_type ip_packet_type __read_mostly = {
	.type = cpu_to_be16(ETH_P_IP),
	.func = ip_rcv,
};
```
主要是：全局变量：
static struct list_head ptype_base[16];	/* 16 way hashed list */  dev.c 
是以下十六个三层协议的列表，每种协议都由，packet_type数据结构表示，列表中的每个元素指向这个结构，
形成一个hash表，在dev_add_pack函数时添加到列表中
```cpp
/*
 *		0800	IP
 *		8100    802.1Q VLAN
 *		0001	802.3
 *		0002	AX.25
 *		0004	802.2
 *		8035	RARP
 *		0005	SNAP
 *		0805	X.25
 *		0806	ARP
 *		8137	IPX
 *		0009	Localtalk
 *		86DD	IPv6
 */
而  static struct list_head ptype_all;		/* Taps */ 对应了ETH_P_ALL
在skbuff向上传递时；
dev.c
static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
{
...
type = skb->protocol;
/* deliver only exact match when indicated */
	if (likely(!deliver_exact)) {
		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
				       &ptype_base[ntohs(type) &
						   PTYPE_HASH_MASK]);
....
基于此能传递到正确的三层处理函数；  

```

+ inet_protos
extern const struct net_protocol __rcu *inet_protos[MAX_INET_PROTOS];
inet_add_protocol(&udp_protocol, IPPROTO_UDP) //通过类似函数添加；
inet_protos[IPPROTO_UDP]=udp_protocol
```cpp
static const struct net_protocol udp_protocol = {
	.early_demux =	udp_v4_early_demux,
	.handler =	udp_rcv,
	.err_handler =	udp_err,
	.no_policy =	1,
	.netns_ok =	1,
};
```
进而能找到对应的四层函数：
```cpp
举例：
static int ip_local_deliver_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	__skb_pull(skb, skb_network_header_len(skb));

	rcu_read_lock();
	{
		int protocol = ip_hdr(skb)->protocol;
		const struct net_protocol *ipprot;
		int raw;

	resubmit:
		raw = raw_local_deliver(skb, protocol);

		ipprot = rcu_dereference(inet_protos[protocol]);//这里去找；
		if (ipprot) {
```

其他：
+ prot_list:
  这个结构原先以为会也是类似net_family的作用，但是查了下引用的位置，这个全局的静态链表，inet域支持的所有协议全部在这个链表中，它只是用于在/proc/net/protocols文件中输出当前系统所支持的所有协议。没有其他功能；
```cpp
sock.c
#ifdef CONFIG_PROC_FS
下面是proc相关的函数；
static void *proto_seq_start(struct seq_file *seq, loff_t *pos)
	__acquires(proto_list_mutex)
{
	mutex_lock(&proto_list_mutex);
	return seq_list_start_head(&proto_list, *pos);
}

static void *proto_seq_next(struct seq_file *seq, void *v, loff_t *pos)
{
	return seq_list_next(v, &proto_list, pos);
}
```
一个问题抛出来：
接收方向skbuff如何准确传递到正确的socket进而传递数据给对应的用户？
关于路由表相关和具体协议相关的结构，等到时候再分析；

#### 设备相关初始化
设备相关的初始化，是比较复杂的内容，linux很大一部分代码就是各种设备驱动程序代码；在2.6后，采用了通用设备模型来管理设备；
网络设备时比较特殊的设备之一；  
涉及以下几个方面：  
1 网络设备的表示，一个网络设备通过net_device结构实例来表示，内核通过net_device管理和控制，连接到实际网络设备驱动程序，进而控制设备；  
2 网络设备的驱动程序；网络设备多种多样，有wifi网卡，有线网卡等，而且跟具体芯片型号等有关，不同种类的设备驱动程序不同；而发送和接收都需要通过驱动程序  
  处理，并进而交互到设备的固件firmware，从而发送和接收；网络设备驱动程序本质上就是分配和初始化net_device的过程，并提供了各种操作设备的函数；
3 网络设备初始化的基本过程：  
  （1）注册：一个网络设备可用，就必须被内核认可，并且关联正确的驱动程序；驱动程序把驱动设备所需要的所有信息存储在私有数据结构中，然后与其他需要此设备的内核组件交互；  
  （2）网络设备驱动程序如何分配 设备与内核通信的资源  
  网络设备驱动程序如何分配 建立设备/内核通信所需要的资源：  
    即主要是包括：
    1) IRQ线初始化，虚拟设备不需要  
        网络设备NIC必须被分配一个IRQ:然后再必要时如rx,提醒内核； --涉及请求和释放irq线，从/proc/interrupts文件可知当前分配状态；  
    2)IO端口和内存注册：设备程序将其设备的一个内存区域如其配置寄存器映射到系统内存，这样驱动程序的读写可以通过系统内存进行；  
        映射函数：request_region,release_region   
    3)关键知识：  
      内核和设备之间的交互：  
      中断或轮询  
硬件中断：  
       每一个中断都会运行一个中断处理程序，这些中断响应程序都是设备驱动为设备量身定做的。一般而言，当设备注册一个NIC时，它首先会请求并分配一个IRQ，然后要为IRQ注册（如果设备被卸载了，则需要注销）一个IRQ响应程序。相应的内核代码在kernel/irq/manage.c和arch/XXX/kernel/irq.c。（其中XXX为处理器架构）
```cpp
int request_threaded_irq(unsigned int irq, irq_handler_t handler,                                            
             irq_handler_t thread_fn, unsigned long irqflags,
             const char *devname, void *dev_id)
void free_irq(unsigned int irq, void *dev_id)
    注意：irq的注册和释放函数都带有参数dev_id。因为IRQ是可以共享的，因此需要IRQ number和dev_id共同来唯一表示中断。
    另，在注册IRQ时，必须保证IRQ还未有设备请求，除非所有设备都支持IRQ共享。
    内核接收到一个中断信号时，会通过IRQ number调用关联的中断响应程序。IRQ number与中断响应程序以表的形式保存。由于多个设备可能共享IRQ的关系，IRQ number与中断响应程序的关系可能是一对多的。
    中断类型：接收到数据帧、帧传输失败、DMA传输已成功完成、设备已经有足够内存来创建新的传输会话（可用NIC可用内存达到一定数值<一般为设备MTU>时产生一个中断）
    为了防止内核在设备内存不足时多次提交传输请求，设备驱动可以关闭内核出口队列，待到资源足够是才重启。下面是一个范例：
   static netdev_tx_t
el3_start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    ……
    netif_stop_queue (dev);
    ……
    dev->trans_start = jiffies;
    if (inw(ioaddr + TX_FREE) > 1536)
        netif_start_queue(dev);
    else
        /* Interrupt us when the FIFO has room for max-sized packet. */
        outw(SetTxThreshold + 1536, ioaddr + EL3_CMD);
    ……
}
 IRQ共享：IRQ线是很有限的资源，为了让一个系统能支持更多的设备，只能让多个设备共享IRQ线。IRQ共享的机制是这样的，内核收到中断请求，然后调用所有与该中断相关联的响应例程，然后有各个响应例程自行判断过滤是否对这个中断进行处理。（注意，IRQ与响应程序是一对多的，发生一个IRQ，哪些响应程序要处理，哪些不需要不是有内核去判断，而是各个中断响应程序自己判断，内核则是调用所有的响应程序。）
IRQ与IRQ响应程序的组织：用全局的vector：irq_desc来组织，irq_desc包含所有IRQ，每个IRQ对应自己的链表，链表中是该IRQ关联的所有响应程序。只有IRQ共享时，IRQ链表的节点才会超过一个。
```

  （3）网络设备结构net_device的分配和初始化
      module_init和probe
     注册和初始化任务的一部分由内核负责（module_init)，其他部分则由设备驱动程序负责(pci扫描到的具体的probe函数)；
     部分由内核完成，部分由驱动程序完成（决定如何分配建立设备/内核通信所需资源：irq,IO端口）
           1) 通过一组函数指针和驱动设备函数交互
           2) 这个结构的初始化部分由内核完成，部分由设备驱动函数完成

4 网络设备相关的初始化相关重要函数：  
  (1)subsys_init(net_dev_init) 对应抽象设备层(核心模块)  
  下面的(2)(3)对应特定设备驱动程序
  (2)device_init(module_init)   
  (3)pci_scan(probe) 真正的和真实设备挂钩的初始化，设备模型 kobject pci，这样管理所有设备，并调用xxx_probe-->xxx_setup初始化net_device或其他设备实例；  

  关于抽象设备层解释：  
  这一层主要提供一些设备无关的处理流程，也提供一些公用的函数给底层的 device driver 调用。
它为网络协议提供统一的发送、接收接口。这主要是通过 net_device 结构。是上层的、与设备无关的，
这部分根据输入输出请求，通过特定设备驱动程序接口，来与设备进行通信。
subsys_initcall(net_dev_init);
这个宏定义请参见前面说的 init.h，它被定义为： define_initcall("4",fn)
所以它是在 core_initcall 和 fs_initcall 之后被调用的。 
  
  关于特定设备驱动程序解释：  
 是一种下层的、与设备有关的，常称为设备驱动程序，它直接与相应设备打
交道，并且向上层提供一组访问接口； 当一个网络设备的初始化程序被调用时，它返回一个状态指
示它所驱动的控制器是否有一个实例。 
主要包括module_init和probe函数，即最终都是对net_deivce结构的初始化  



##### 代码结构分析 -net_dev_init
```cpp
/*
 *       This is called single threaded during boot, so no need
 *       to take the rtnl semaphore.
 */
static int __init net_dev_init(void)
{
	int i, rc = -ENOMEM;

	BUG_ON(!dev_boot_phase);

	if (dev_proc_init())//在/proc/net目录下创建四个proc条目（分别为dev、softnet_stat、ptype和wireless） 
		goto out;

	if (netdev_kobject_init())//暂时不太清除
		goto out;

	INIT_LIST_HEAD(&ptype_all);
	for (i = 0; i < PTYPE_HASH_SIZE; i++)
		INIT_LIST_HEAD(&ptype_base[i]);//前面有提到，不赘述

	INIT_LIST_HEAD(&offload_base);()
	if (register_pernet_subsys(&netdev_net_ops))//将全局变量netdev_net_ops注册到链表(static struct list_head *first_device = &pernet_list;)上
		goto out;

	/*
	 *	Initialise the packet receive queues.
	 */
    //对于多CPU的系统来说，每个CPU都有一个各自的接收队列，并且使用各自的softnet_data结构体变量*sd来管理网络数据包的收发流量（通过per_cpu函数）。
	//初始化两个sk_buff_head结构体变量process_queue和input_pkt_queue。
    for_each_possible_cpu(i) {
		struct softnet_data *sd = &per_cpu(softnet_data, i);

		skb_queue_head_init(&sd->input_pkt_queue);
		skb_queue_head_init(&sd->process_queue);
		INIT_LIST_HEAD(&sd->poll_list);
		sd->output_queue_tailp = &sd->output_queue;
#ifdef CONFIG_RPS
		sd->csd.func = rps_trigger_softirq;
		sd->csd.info = sd;
		sd->cpu = i;
#endif

		sd->backlog.poll = process_backlog;
		sd->backlog.weight = weight_p;
	}

	dev_boot_phase = 0;

	/* The loopback device is special if any other network devices
	 * is present in a network namespace the loopback device must
	 * be present. Since we now dynamically allocate and free the
	 * loopback device ensure this invariant is maintained by
	 * keeping the loopback device as the first device on the
	 * list of network devices.  Ensuring the loopback devices
	 * is the first device that appears and the last network device
	 * that disappears.
	 */
     //注册网络命令空间设备，确保loopback设备在所有网络设备中最先出现和最后消失
	if (register_pernet_device(&loopback_net_ops))
		goto out;

	if (register_pernet_device(&default_device_ops))
		goto out;
    //分别注册网络设备数据包接收和发送的软中断处理程序 
	open_softirq(NET_TX_SOFTIRQ, net_tx_action);
	open_softirq(NET_RX_SOFTIRQ, net_rx_action);
    //注册回调处理函数dev_cpu_callback 
	hotcpu_notifier(dev_cpu_callback, 0);
	dst_init();//和通知链相关的初始化
	rc = 0;
out:
	return rc;
}
```
先备知识：  
1) 中断相关，软中断和硬中断  
2) 通知链相关  
3) 设备模型  
4) softnet_data相关，这里先不解释；  
5) other:用户空间辅助程序：  
    /sbin/modprobe 在内核需要加载某个模块时调用，判断内核传递的模块是不是/etc/modprobe.conf文件中定义的别名  
    /sbin/hotplug 在内核检测到一个新设备插入或拔出系统时调用，它的任务是根据设备标识加载正确的驱动  
##### 代码结构分析 -module_init:
module_init是一大类的函数，几乎所有的设备都有实现这个函数：xxx_init_module为module_init类型；
凡是被 module_init()“修饰”过的函数只能在两种 情况下被调用：一种是被 do_initcalls 调用，一种是
在模块插入到系统中时被调用（如果它是模块方式）。每个模块只有一个被 module_init 修饰的函数入口  
eg: 78. #define module_init(x) __initcall(x);  
对第一种情况：  
开机流程初始化有init_call机制，会做几乎所有的init,会包含一个初始化所有设备的module_init,自然就包括了net_device在module_init中的初始化；
eg:对回环设备：  
```cpp
think@think-VirtualBox:~/source_linux/linux-lts-xenial-4.4.0$ objdump  -t vmlinux| grep loopback
0000000000000000 l    df *ABS*	0000000000000000 loopback.c
ffffffff815ec3a0 l     F .text	00000000000000a8 loopback_setup
ffffffff81aa91c0 l     O .rodata	0000000000000198 loopback_ethtool_ops
ffffffff81aa8f80 l     O .rodata	0000000000000238 loopback_ops
ffffffff815ec450 l     F .text	0000000000000036 loopback_dev_free
ffffffff815ec490 l     F .text	0000000000000081 loopback_get_stats64
ffffffff815ec520 l     F .text	000000000000009e loopback_xmit
ffffffff815ec5c0 l     F .text	000000000000007d loopback_dev_init
ffffffff815ec640 l     F .text	000000000000009d loopback_net_init //这个函数
```
module_init只是对设备结构等做一个基本的初始化，这个时候还不能使用；或者说linux支持很多种设备，在开机初始化时做的module_init只是一个对支持的设备的初始化，对这个机器是否插入这个硬件设备不依赖，也不代表就可以用了；而要等到pci扫描设备后，对真正存在的设备调用对应的probe函数，读取和初始化真正设备，之后才能使用；  
具体过程如下：
##### 代码结构分析 -module_init:
+ pci子系统的基本介绍：
内核中的PCI子系统（PCI层）提供各种PCI设备驱动程序共同的所有通用功能；PCI电源管理和网络唤醒  
1、几个数据结构：/include/linux/mod_devicetable.h  
A:pci_device_id 设备标示符；  
B:pci_dev 每个pci上的设备都会分配一个pci_dev实例，如同网络设备会被分配net_device一样，这个结构由内核使用，以引用一个PCi设备；  
C:pci_driver:定义pci层和设备驱动程序之间的接口；由函数指针构成， 所有pci上的设备都会使用这个结构；  
D:char *name:驱动程序名称；总线上的；  
E:const struct pci_deivce_id *id_table:Id向量，内核用于把一些设备关联到此驱动程序；  
F:int (*probe)当pci层发现他正在搜寻驱动程序设备id和前面的id_table匹配，就会调用此函数。来做类似开启硬件，分配net_deivce结构，初始化并注册新设备；  
   void (*remove)和上面相反；  
G: suspend resume函数和电源管理有关；  

2、PCI NIC 设备驱动程序的注册；  
```cpp
struct pci_device_id {
	__u32 vendor, device;		/* Vendor and device ID or PCI_ANY_ID*/
	__u32 subvendor, subdevice;	/* Subsystem ID's or PCI_ANY_ID */
	__u32 class, class_mask;	/* (class,subclass,prog-if) triplet */
	kernel_ulong_t driver_data;	/* Data private to the driver */
};
```
用于独一无二识别pci设备；
 每个设备驱动程序都会把一个pci_device_id实例的向量注册到内核；这个实例向量列出了其所能处理的设备ID
3、pci设备本身的注册等；  
4、pci的探测分静态和动态  

+ pci总线系统在开机引导期间：  
例如linux现在支持PCI域，每个PCI域可以占用多大256个总线，每个总线占用32个设备；  
1) 系统引导时会建立一种数据库，把每个总线都关联到一份已侦测到而使用该总线的设备列表；如一个总线上挂了多个设备；  
2)总线上的设备如何注册  
   当设备驱动程序A（对应总线上的设备A）被加载时，会调用pci_register_driver并提供pci_driver实例而与pci层注册。  
   pci_driver结构中内含一个此驱动程序能驱动的pci设备id的向量（ *id_table),。  
   接着，pci层使用这个表去查看已侦测到的pci上设备列表中与哪些设备匹配；于是就会建立该驱动程序的设备列表；  
   接着，对每个匹配的设备，pci层会调用相匹配的驱动程序中的pci_driver结构中提供的probe函数；  
  接着，probe函数会建立并注册相关联的网络设备；接着调用到xx_setup函数；  
3) 总线上设备除名：  
  当驱动程序稍后卸载时，该模块的module_exit函数会调用pci_unregister_driver，接着由于其数据库，使得pci能遍历所有与该驱动程序相关联的设备；并
   启动该设备的remove函数；从网络设备除名；  

##### 几个问题：
0、module_init和probe函数中都会对net_device结果做更新和初始化，他们怎么调用的？  
   开机流程初始化有init_call机制，会做几乎所有的init,会包含一个初始化所有设备的module_init,自然就包括了net_device在module_init中的初始化；
而开机会做pci扫描，这个时候配置好设备后，会调用对应的probe,过程太过复杂，待研究，只学习较理论方面，见 linux设备模型中的设备驱动模型例子_PCI总线为例，通过扫描pci，接着注册后会调用到各个pci设备的probe函数，进而也调用到了网络设备的probe函数  
  
1、module_init修饰的函数什么时候调用？  
凡是被 module_init()“修饰”过的函数只能在两种 情况下被调用：一种是被 do_initcalls 调用，一种是
在模块插入到系统中时被调用（如果它是模块方式）。每个模块只有一个被 module_init 修饰的函数入口
78. #define module_init(x) __initcall(x);  

所以真正的初始化是 probe函数调用时；  
##### 数据结构图
略
net_device结构介绍：ref 深入理解linux网络内幕
1、介绍：
net_device数据结构存储着特定网络设备的所有信息，每个网络设备对应一个net_device结构实例，无论是真实设备(如NIC)或者是虚拟设备(Bonding或VLAN)，并由驱动程序调用相关内核函数进行分配和注册，内核做一些初始化；  
驱动程序，对这个设备的ops函数做初始化，并定义了相关函数赋值，这样内核通过统一的函数接口就可以掉用到对应的驱动函数；
可以说net_device是内核和驱动的桥梁；  
所有设备的net_device结构放在一个全局变量 dev_base所指的全局列表中，在include/linux/netdevice.h中定义
网络设备可以分成几种类型，如Ethernet卡和Token Ring卡；对同一类型的所有设备，会设置某些字段为相同的值，有些则根据设备模型做不同的设置；
为了改善性能，驱动程序还可以改写一些已由内核初始化过的字段；  
1_2: 结构组成：  
net_device结构的字段可以分成以下几种类型：  
标识符：net_device有三个标识符，不能搞混：  
    int ifindex: 独一无二的ID,当设备以dev_new_index注册时分派给每个设备；  
    int iflink: 这个字段主要是由(虚拟)隧道设备使用，可用于标识抵达隧道另一端的真实设备；  
    unsigned short dev_id: 目前在zSeries OSA NIC上由IPv6使用，此字段用于区别可由不同OS同时共享的同一种设备的诸多设虚拟实例；见net/ipv6/addrconf.c的注释；  

配置  
统计数据  
设备状态  
列表管理  
流量管理  
功能专用  
通用  
函数指针  

2、ref 深入理解linux网络内幕：设备注册和初始化：net_device  
(1) 网络设备何时以及如何在内核注册：静态，热插拔  
(2) 网络设备如何利用网络设备数据库注册，并指派一个net_device结构的实例  
(3) net_device结构如何组织到hash表和列表，以便于做各种查询；  
(4) net_device实例如何初始化，一部分由内核核心函数完成，一部分由其设备驱动程序完成，如ops结构中的函数指针；  
(5）就注册而言，虚拟设备和真实设备有和区别；  

3、正文：  
nic（网卡）可用之前，其相关联的net_deivce数据结构必须先初始化，添加到内核网络设备数据库，配置并开启；
注册/除名 和开启/关闭 是不同的，前者可以理解为注册和加载驱动/卸载驱动，后者可以理解为开启和关闭网络设备，即关闭相关进程；  
(1)网络设备注册之时：  
   A:开机时，加载nic设备驱动程序，insmod类似动作或者调用module_init这种；或者在运行时进行insmod动态加载；  
           这种情况下，可能会是通过总线设备驱动程序进行pci_driver->probe的调用，来负责设备注册；
  **这里详细说明下：通过pci检测设备并调用probe函数，这个函数是由驱动提供，通过.probe=xxx_probe来实现，并做与module_init相同的事情；**
  **那么什么时候会用probe函数什么时候用module_init函数？前者是自动检测是用的，后者是运行时动态加载时使用的；**--这个是查询的，需要再确认；
** 可以看到 若调用probe函数，则为XX_probe-->alloc_dev-->xxx_setup**  
例子：net/wireless/airo.c
   B:插入热插拔设备；此时内核会通知其驱动程序，驱动程序再注册该设备；   
  网络设备除名之时：pci_driver->remove
   1:关机时或者运行时做rmmod，即卸载设备驱动  
   2:拔出热插拔设备，此时做删除等动作；  

(2)分配net_deivce结构：  
使用在net/core/dev.c中的alloc_netdev分配，而一般由设备驱动程序进行调用；  
这个函数的三个参数：由驱动程序提供实参  
  私有数据结构大小:是驱动程序会使用的私有结构大小 
 ，设备名称，一个字符串，类似于eth0,eth1
设置函数：初始化函数指针，用来设置net_device结构的剩余部分；
返回值是指向已分配的net_device结构指针；
此外，内核也提供一组内含alloc_netdev的包裹函数，可用于为一组通用设备类型提供正确参数给alloc_netdev
好几个  




(3)Nic注册和除名的架构（设备驱动程序加载和卸载相关联）  
依赖于总线；  
其余部分则是网络设备驱动程序的架构；  
    结构：私有数据结构+net_deivce+ops
    初始化：alloc_dev+register net_device and ops中的函数集合+setup函数以及其他初始化函数；  
    卸载清理函数；  
    module_init,module_exit;/xxx_probe xxx_remove_one  
例子见上；注意一下几点：  
A:驱动程序可能会使用包裹函数，并只提供其私有数据区大小；  
B:包裹函数会使用驱动程序提供的参数（包括设备名加初始化函数）来调用alloc_netdev  
C:alloc_netdev所分配的内存块大小包括net_device结构和驱动程序私有数据块以及强制对齐所补的空白空间；  
D:有些驱动程序会调用netdev_boot_setup_check函数减产加载内核时用户是否提供了任何引导期间参数；  
E:新的net_device实例会利用register_netdevice来插入设备数据库；  

(4)设备初始化，包含xxx_setup函数，驱动程序传入的指针对于的函数，会对net_device等做一些初始化；  
设备初始化包含以下三类：
   设备驱动程序的初始化，包括但不止xxx_setup（init函数里面可能也会做一些初始化)，  
   设备类型：由xxx_setup函数负责;  
   各种功能：可选和强制功能也必须初始化如队列规则；  

注意：xxx_setup是设备驱动程序传给内核的，但会被设备模型（总线）通过xxx_probe调用到
xxx_setup也会做类似于设置mtu,macaddr,hard_head_len,等值；

(5) net_deivce 结构组织介绍：仅说明一些点；  
 览:net_device数据结构插在一个全局列表和两张hash表中；这些结构可以让内核按需求浏览和查询net_deivce数据库；
 A: dev_base是一个指针，指向net_deivce链表的头，net_device中的next结果将整个全部设备串起来，这样内核通过这个结构可以轻易浏览设备；取得关键数据等；  
 B: dev_name_head:是一张hash表，以设备名为索引，可以让类似ioctl进行操作；  
 C:dev_index_head:是一张hash表，以设备ID:dev->ifindex为索引，指向net_device结构指针；如ip,netlink时就是通过dev->ifindex来索引的；  
上面两张表，就可以提供获取设备的接口如：dev_get_by_name()和dev_get_by_index; 也可能根据设备类型和mac地址搜寻net_device，此时用的是dev_base
    上述三个结构通过dev_base_lock锁保护，所有查询函数在net/core/dev.c中  

(6)设备状态：  
net_device中有各种字段可以定义设备当前状态；
如： flags:开启或关闭,reg_state:注册状态；state:用于队列规则；
队列规则状态：  
每个网络设备都会分配一种队列规则，流量控制并以此来实现其Qos机制；
state就是用于这个，是位域，在include/linux/netdeivce.h中；
```c
enum netdev_state_t
{
	__LINK_STATE_XOFF=0,
	__LINK_STATE_START, 设备开启，此标示可以由netif_running检查
	__LINK_STATE_PRESENT,
	__LINK_STATE_SCHED,
	__LINK_STATE_NOCARRIER,
	__LINK_STATE_RX_SCHED,
	__LINK_STATE_LINKWATCH_PENDING
};
```
注册状态：  
reg_state:
```c
	/* register/unregister state machine */
	enum { NETREG_UNINITIALIZED=0,
	       NETREG_REGISTERING,	/* called register_netdevice */
	       NETREG_REGISTERED,	/* completed register todo */
	       NETREG_UNREGISTERING,	/* called unregister_netdevice */
	       NETREG_UNREGISTERED,	/* completed unregister todo */
	       NETREG_RELEASED,		/* called free_netdev */
	} reg_state;
```

(7)注册函数和除名函数过程：  
net/core/dev.c中：  
调用netdev_run_todo处理；书里面将的调用情况，先忽略
A:注册详情：即注册时做了什么  
  注册不仅仅是将net_device插入上述三个结构中，而还有其他动作：如一般有；  
   1) net_device的一些字段初始化；  
   2) 内核支持divert功能时，分配配置；  
   3) 执行xxx_setup函数  
   4)执行dev_new_index分配一个独一无二的标识码给设备；  
   5)将net_device加入上面三个结构中；  
   6)检查，设置状态和标识，初始化队列规则等；  
   7)调用通知链相关函数；  

B:除名详情，即反注册时做了什么：  
上面的反过程

(8)设备注册状态的通知：  
内核组件和用户空间应用程序可能都想知道何时发生网络设备注册，除名，关闭或者开启之事，这类事件的通知通过两种通道传送：
  A:netdev_chain:注册通知链；  
  B:Netlink的RTMGRP_LINK多播群组；  
A:所有的netdev_chain报告的NETDEV_XXX事件都列在include/linux/notifier.h中；通知链怎么用见前文  
   如对上面NETDEV_XXX事件感兴趣的可以通过register_netdevice_notifier和unregister_netdevice_notifier注册和反注册，就可以监听到消息；
   目前如：路由，防火墙，协议代码，虚拟设备等内核组件都在netdev_chain注册了；

(9) 引用计数：  
net_device结构无法释放除非所有的引用都已释放；dev->refcnt中；  
netdev_wait_allrefs;  


(10)开启和关闭网络设备；  
设备一旦注册就可以用，但是除非用户或用户应用程序明确开启，否则还是无法传输接收数据
net/core/dev.c中dev_open负责；
开启设备：
   A：dev->open;  
  B:设置dev->state  
  C:设置flag  
  D:初始化流量控制队列规则等；  
  E:通知链调用；  
关闭设备：  
相反


(11)和电源管理之间的交互：  
  当内核支持电源管理时，若进入挂起模式，则NIC设备驱动程序会接到通知，通过pci pci_driver结构的suspend,resume；
会影响到网络设备，如net->state,并调用相关函数挂起设备或重新继续；netif_device_detach/netif_device_attach
其他：  

(12)链路状态变更检测：  
A：驱动程序检测载波或者信号是否存在；  netif_carrier_on/netif_carrier_off;  
B:  调度并处理链路状态变更事件；  
C:链接监看标示  

(13)从用户空间配置设备相关信息；  
ifconfig,ethool,ip link;通过ioctl或netlink下去；  
媒介独立接口：MII  
(14)虚拟设备注意  
(15)上锁  
##### probe等函数例子：
driver/net/xxx/xxx 具体设备相关文件；  
##### 以wifi芯片举例：
       (data数据网络：ap侧：手机cpu modem侧：modem芯片)  
       (wifi网络：ap侧：固件wifi芯片  host端：手机cpu）  
       网卡NIC如无线网卡，内部是由程序在运行，在网卡中运行的程序叫做固件，对wifi而言也称为AP侧，而在PC端运行的用于控制和相应网卡中断的称为driver（驱动）或host侧；  
      驱动程序往往以ko的形式，被内核加载和运行  
      固件以bin或其他形式(mtk:bin,qcom:mpb)，被push到设备指定目录中，并最后push到设备内存中运行；  

###### ldd3上一个虚拟设备的例子：贴过来仅供参考；
```cpp
/*
 * snull.c --  the Simple Network Utility
 *
 * Copyright (C) 2001 Alessandro Rubini and Jonathan Corbet
 * Copyright (C) 2001 O'Reilly & Associates
 *
 * The source code in this file can be freely used, adapted,
 * and redistributed in source or binary form, so long as an
 * acknowledgment appears in derived source files.  The citation
 * should list that the code comes from the book "Linux Device
 * Drivers" by Alessandro Rubini and Jonathan Corbet, published
 * by O'Reilly & Associates.   No warranty is attached;
 * we cannot take responsibility for errors or fitness for use.
 *
 * $Id: snull.c,v 1.21 2004/11/05 02:36:03 rubini Exp $
 */

#include <linux/config.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/moduleparam.h>

#include <linux/sched.h>
#include <linux/kernel.h> /* printk() */
#include <linux/slab.h> /* kmalloc() */
#include <linux/errno.h>  /* error codes */
#include <linux/types.h>  /* size_t */
#include <linux/interrupt.h> /* mark_bh */

#include <linux/in.h>
#include <linux/netdevice.h>   /* struct device, and other headers */
#include <linux/etherdevice.h> /* eth_type_trans */
#include <linux/ip.h>          /* struct iphdr */
#include <linux/tcp.h>         /* struct tcphdr */
#include <linux/skbuff.h>

#include "snull.h"

#include <linux/in6.h>
#include <asm/checksum.h>

MODULE_AUTHOR("Alessandro Rubini, Jonathan Corbet");
MODULE_LICENSE("Dual BSD/GPL");


/*
 * Transmitter lockup simulation, normally disabled.
 */
static int lockup = 0;
module_param(lockup, int, 0);

static int timeout = SNULL_TIMEOUT;
module_param(timeout, int, 0);

/*
 * Do we run in NAPI mode?
 */
static int use_napi = 0;
module_param(use_napi, int, 0);


/*
 * A structure representing an in-flight packet.
 */
struct snull_packet {
	struct snull_packet *next;
	struct net_device *dev;
	int	datalen;
	u8 data[ETH_DATA_LEN];
};

int pool_size = 8;
module_param(pool_size, int, 0);

/*
 * This structure is private to each device. It is used to pass
 * packets in and out, so there is place for a packet
 */

struct snull_priv {//这个网络设备的私有数据结构
	struct net_device_stats stats;
	int status;
	struct snull_packet *ppool;
	struct snull_packet *rx_queue;  /* List of incoming packets */
	int rx_int_enabled;
	int tx_packetlen;
	u8 *tx_packetdata;
	struct sk_buff *skb;
	spinlock_t lock;
};
//一些net_deivce需要的函数
static void snull_tx_timeout(struct net_device *dev);
static void (*snull_interrupt)(int, void *, struct pt_regs *);

/*
 * Set up a device's packet pool.
 */
void snull_setup_pool(struct net_device *dev)
{
	struct snull_priv *priv = netdev_priv(dev);
	int i;
	struct snull_packet *pkt;

	priv->ppool = NULL;
	for (i = 0; i < pool_size; i++) {
		pkt = kmalloc (sizeof (struct snull_packet), GFP_KERNEL);
		if (pkt == NULL) {
			printk (KERN_NOTICE "Ran out of memory allocating packet pool\n");
			return;
		}
		pkt->dev = dev;
		pkt->next = priv->ppool;
		priv->ppool = pkt;
	}
}

void snull_teardown_pool(struct net_device *dev)
{
	struct snull_priv *priv = netdev_priv(dev);
	struct snull_packet *pkt;
    
	while ((pkt = priv->ppool)) {
		priv->ppool = pkt->next;
		kfree (pkt);
		/* FIXME - in-flight packets ? */
	}
}    

/*
 * Buffer/pool management.
 */
struct snull_packet *snull_get_tx_buffer(struct net_device *dev)
{
	struct snull_priv *priv = netdev_priv(dev);
	unsigned long flags;
	struct snull_packet *pkt;
    
	spin_lock_irqsave(&priv->lock, flags);
	pkt = priv->ppool;
	priv->ppool = pkt->next;
	if (priv->ppool == NULL) {
		printk (KERN_INFO "Pool empty\n");
		netif_stop_queue(dev);
	}
	spin_unlock_irqrestore(&priv->lock, flags);
	return pkt;
}


void snull_release_buffer(struct snull_packet *pkt)
{
	unsigned long flags;
	struct snull_priv *priv = netdev_priv(pkt->dev);
	
	spin_lock_irqsave(&priv->lock, flags);
	pkt->next = priv->ppool;
	priv->ppool = pkt;
	spin_unlock_irqrestore(&priv->lock, flags);
	if (netif_queue_stopped(pkt->dev) && pkt->next == NULL)
		netif_wake_queue(pkt->dev);
}

void snull_enqueue_buf(struct net_device *dev, struct snull_packet *pkt)
{
	unsigned long flags;
	struct snull_priv *priv = netdev_priv(dev);

	spin_lock_irqsave(&priv->lock, flags);
	pkt->next = priv->rx_queue;  /* FIXME - misorders packets */
	priv->rx_queue = pkt;
	spin_unlock_irqrestore(&priv->lock, flags);
}

struct snull_packet *snull_dequeue_buf(struct net_device *dev)
{
	struct snull_priv *priv = netdev_priv(dev);
	struct snull_packet *pkt;
	unsigned long flags;

	spin_lock_irqsave(&priv->lock, flags);
	pkt = priv->rx_queue;
	if (pkt != NULL)
		priv->rx_queue = pkt->next;
	spin_unlock_irqrestore(&priv->lock, flags);
	return pkt;
}

/*
 * Enable and disable receive interrupts.
 */
static void snull_rx_ints(struct net_device *dev, int enable)
{
	struct snull_priv *priv = netdev_priv(dev);
	priv->rx_int_enabled = enable;
}

    
/*
 * Open and close
 */

int snull_open(struct net_device *dev)
{
	/* request_region(), request_irq(), ....  (like fops->open) */

	/* 
	 * Assign the hardware address of the board: use "\0SNULx", where
	 * x is 0 or 1. The first byte is '\0' to avoid being a multicast
	 * address (the first byte of multicast addrs is odd).
	 */
	memcpy(dev->dev_addr, "\0SNUL0", ETH_ALEN);
	if (dev == snull_devs[1])
		dev->dev_addr[ETH_ALEN-1]++; /* \0SNUL1 */
	netif_start_queue(dev);
	return 0;
}

int snull_release(struct net_device *dev)
{
    /* release ports, irq and such -- like fops->close */

	netif_stop_queue(dev); /* can't transmit any more */
	return 0;
}

/*
 * Configuration changes (passed on by ifconfig)
 */
int snull_config(struct net_device *dev, struct ifmap *map)
{
	if (dev->flags & IFF_UP) /* can't act on a running interface */
		return -EBUSY;

	/* Don't allow changing the I/O address */
	if (map->base_addr != dev->base_addr) {
		printk(KERN_WARNING "snull: Can't change I/O address\n");
		return -EOPNOTSUPP;
	}

	/* Allow changing the IRQ */
	if (map->irq != dev->irq) {
		dev->irq = map->irq;
        	/* request_irq() is delayed to open-time */
	}

	/* ignore other fields */
	return 0;
}

/*
 * Receive a packet: retrieve, encapsulate and pass over to upper levels
 */
void snull_rx(struct net_device *dev, struct snull_packet *pkt)
{
	struct sk_buff *skb;
	struct snull_priv *priv = netdev_priv(dev);

	/*
	 * The packet has been retrieved from the transmission
	 * medium. Build an skb around it, so upper layers can handle it
	 */
	skb = dev_alloc_skb(pkt->datalen + 2);
	if (!skb) {
		if (printk_ratelimit())
			printk(KERN_NOTICE "snull rx: low on mem - packet dropped\n");
		priv->stats.rx_dropped++;
		goto out;
	}
	skb_reserve(skb, 2); /* align IP on 16B boundary */  
	memcpy(skb_put(skb, pkt->datalen), pkt->data, pkt->datalen);

	/* Write metadata, and then pass to the receive level */
	skb->dev = dev;
	skb->protocol = eth_type_trans(skb, dev);
	skb->ip_summed = CHECKSUM_UNNECESSARY; /* don't check it */
	priv->stats.rx_packets++;
	priv->stats.rx_bytes += pkt->datalen;
	netif_rx(skb);
  out:
	return;
}
    

/*
 * The poll implementation.
 */
static int snull_poll(struct net_device *dev, int *budget)
{
	int npackets = 0, quota = min(dev->quota, *budget);
	struct sk_buff *skb;
	struct snull_priv *priv = netdev_priv(dev);
	struct snull_packet *pkt;
    
	while (npackets < quota && priv->rx_queue) {
		pkt = snull_dequeue_buf(dev);
		skb = dev_alloc_skb(pkt->datalen + 2);
		if (! skb) {
			if (printk_ratelimit())
				printk(KERN_NOTICE "snull: packet dropped\n");
			priv->stats.rx_dropped++;
			snull_release_buffer(pkt);
			continue;
		}
		skb_reserve(skb, 2); /* align IP on 16B boundary */  
		memcpy(skb_put(skb, pkt->datalen), pkt->data, pkt->datalen);
		skb->dev = dev;
		skb->protocol = eth_type_trans(skb, dev);
		skb->ip_summed = CHECKSUM_UNNECESSARY; /* don't check it */
		netif_receive_skb(skb);
		
        	/* Maintain stats */
		npackets++;
		priv->stats.rx_packets++;
		priv->stats.rx_bytes += pkt->datalen;
		snull_release_buffer(pkt);
	}
	/* If we processed all packets, we're done; tell the kernel and reenable ints */
	*budget -= npackets;
	dev->quota -= npackets;
	if (! priv->rx_queue) {
		netif_rx_complete(dev);
		snull_rx_ints(dev, 1);
		return 0;
	}
	/* We couldn't process everything. */
	return 1;
}
	    
        
/*
 * The typical interrupt entry point
 */
static void snull_regular_interrupt(int irq, void *dev_id, struct pt_regs *regs)
{
	int statusword;
	struct snull_priv *priv;
	struct snull_packet *pkt = NULL;
	/*
	 * As usual, check the "device" pointer to be sure it is
	 * really interrupting.
	 * Then assign "struct device *dev"
	 */
	struct net_device *dev = (struct net_device *)dev_id;
	/* ... and check with hw if it's really ours */

	/* paranoid */
	if (!dev)
		return;

	/* Lock the device */
	priv = netdev_priv(dev);
	spin_lock(&priv->lock);

	/* retrieve statusword: real netdevices use I/O instructions */
	statusword = priv->status;
	priv->status = 0;
	if (statusword & SNULL_RX_INTR) {
		/* send it to snull_rx for handling */
		pkt = priv->rx_queue;
		if (pkt) {
			priv->rx_queue = pkt->next;
			snull_rx(dev, pkt);
		}
	}
	if (statusword & SNULL_TX_INTR) {
		/* a transmission is over: free the skb */
		priv->stats.tx_packets++;
		priv->stats.tx_bytes += priv->tx_packetlen;
		dev_kfree_skb(priv->skb);
	}

	/* Unlock the device and we are done */
	spin_unlock(&priv->lock);
	if (pkt) snull_release_buffer(pkt); /* Do this outside the lock! */
	return;
}

/*
 * A NAPI interrupt handler.
 */
static void snull_napi_interrupt(int irq, void *dev_id, struct pt_regs *regs)
{
	int statusword;
	struct snull_priv *priv;

	/*
	 * As usual, check the "device" pointer for shared handlers.
	 * Then assign "struct device *dev"
	 */
	struct net_device *dev = (struct net_device *)dev_id;
	/* ... and check with hw if it's really ours */

	/* paranoid */
	if (!dev)
		return;

	/* Lock the device */
	priv = netdev_priv(dev);
	spin_lock(&priv->lock);

	/* retrieve statusword: real netdevices use I/O instructions */
	statusword = priv->status;
	priv->status = 0;
	if (statusword & SNULL_RX_INTR) {
		snull_rx_ints(dev, 0);  /* Disable further interrupts */
		netif_rx_schedule(dev);
	}
	if (statusword & SNULL_TX_INTR) {
        	/* a transmission is over: free the skb */
		priv->stats.tx_packets++;
		priv->stats.tx_bytes += priv->tx_packetlen;
		dev_kfree_skb(priv->skb);
	}

	/* Unlock the device and we are done */
	spin_unlock(&priv->lock);
	return;
}



/*
 * Transmit a packet (low level interface)
 */
static void snull_hw_tx(char *buf, int len, struct net_device *dev)
{
	/*
	 * This function deals with hw details. This interface loops
	 * back the packet to the other snull interface (if any).
	 * In other words, this function implements the snull behaviour,
	 * while all other procedures are rather device-independent
	 */
	struct iphdr *ih;
	struct net_device *dest;
	struct snull_priv *priv;
	u32 *saddr, *daddr;
	struct snull_packet *tx_buffer;
    
	/* I am paranoid. Ain't I? */
	if (len < sizeof(struct ethhdr) + sizeof(struct iphdr)) {
		printk("snull: Hmm... packet too short (%i octets)\n",
				len);
		return;
	}

	if (0) { /* enable this conditional to look at the data */
		int i;
		PDEBUG("len is %i\n" KERN_DEBUG "data:",len);
		for (i=14 ; i<len; i++)
			printk(" %02x",buf[i]&0xff);
		printk("\n");
	}
	/*
	 * Ethhdr is 14 bytes, but the kernel arranges for iphdr
	 * to be aligned (i.e., ethhdr is unaligned)
	 */
	ih = (struct iphdr *)(buf+sizeof(struct ethhdr));
	saddr = &ih->saddr;
	daddr = &ih->daddr;

	((u8 *)saddr)[2] ^= 1; /* change the third octet (class C) */
	((u8 *)daddr)[2] ^= 1;

	ih->check = 0;         /* and rebuild the checksum (ip needs it) */
	ih->check = ip_fast_csum((unsigned char *)ih,ih->ihl);

	if (dev == snull_devs[0])
		PDEBUGG("%08x:%05i --> %08x:%05i\n",
				ntohl(ih->saddr),ntohs(((struct tcphdr *)(ih+1))->source),
				ntohl(ih->daddr),ntohs(((struct tcphdr *)(ih+1))->dest));
	else
		PDEBUGG("%08x:%05i <-- %08x:%05i\n",
				ntohl(ih->daddr),ntohs(((struct tcphdr *)(ih+1))->dest),
				ntohl(ih->saddr),ntohs(((struct tcphdr *)(ih+1))->source));

	/*
	 * Ok, now the packet is ready for transmission: first simulate a
	 * receive interrupt on the twin device, then  a
	 * transmission-done on the transmitting device
	 */
	dest = snull_devs[dev == snull_devs[0] ? 1 : 0];
	priv = netdev_priv(dest);
	tx_buffer = snull_get_tx_buffer(dev);
	tx_buffer->datalen = len;
	memcpy(tx_buffer->data, buf, len);
	snull_enqueue_buf(dest, tx_buffer);
	if (priv->rx_int_enabled) {
		priv->status |= SNULL_RX_INTR;
		snull_interrupt(0, dest, NULL);
	}

	priv = netdev_priv(dev);
	priv->tx_packetlen = len;
	priv->tx_packetdata = buf;
	priv->status |= SNULL_TX_INTR;
	if (lockup && ((priv->stats.tx_packets + 1) % lockup) == 0) {
        	/* Simulate a dropped transmit interrupt */
		netif_stop_queue(dev);
		PDEBUG("Simulate lockup at %ld, txp %ld\n", jiffies,
				(unsigned long) priv->stats.tx_packets);
	}
	else
		snull_interrupt(0, dev, NULL);
}

/*
 * Transmit a packet (called by the kernel)
 */
int snull_tx(struct sk_buff *skb, struct net_device *dev)
{
	int len;
	char *data, shortpkt[ETH_ZLEN];
	struct snull_priv *priv = netdev_priv(dev);
	
	data = skb->data;
	len = skb->len;
	if (len < ETH_ZLEN) {
		memset(shortpkt, 0, ETH_ZLEN);
		memcpy(shortpkt, skb->data, skb->len);
		len = ETH_ZLEN;
		data = shortpkt;
	}
	dev->trans_start = jiffies; /* save the timestamp */

	/* Remember the skb, so we can free it at interrupt time */
	priv->skb = skb;

	/* actual deliver of data is device-specific, and not shown here */
	snull_hw_tx(data, len, dev);

	return 0; /* Our simple device can not fail */
}

/*
 * Deal with a transmit timeout.
 */
void snull_tx_timeout (struct net_device *dev)
{
	struct snull_priv *priv = netdev_priv(dev);

	PDEBUG("Transmit timeout at %ld, latency %ld\n", jiffies,
			jiffies - dev->trans_start);
        /* Simulate a transmission interrupt to get things moving */
	priv->status = SNULL_TX_INTR;
	snull_interrupt(0, dev, NULL);
	priv->stats.tx_errors++;
	netif_wake_queue(dev);
	return;
}



/*
 * Ioctl commands 
 */
int snull_ioctl(struct net_device *dev, struct ifreq *rq, int cmd)
{
	PDEBUG("ioctl\n");
	return 0;
}

/*
 * Return statistics to the caller
 */
struct net_device_stats *snull_stats(struct net_device *dev)
{
	struct snull_priv *priv = netdev_priv(dev);
	return &priv->stats;
}

/*
 * This function is called to fill up an eth header, since arp is not
 * available on the interface
 */
int snull_rebuild_header(struct sk_buff *skb)
{
	struct ethhdr *eth = (struct ethhdr *) skb->data;
	struct net_device *dev = skb->dev;
    
	memcpy(eth->h_source, dev->dev_addr, dev->addr_len);
	memcpy(eth->h_dest, dev->dev_addr, dev->addr_len);
	eth->h_dest[ETH_ALEN-1]   ^= 0x01;   /* dest is us xor 1 */
	return 0;
}


int snull_header(struct sk_buff *skb, struct net_device *dev,
                unsigned short type, void *daddr, void *saddr,
                unsigned int len)
{
	struct ethhdr *eth = (struct ethhdr *)skb_push(skb,ETH_HLEN);

	eth->h_proto = htons(type);
	memcpy(eth->h_source, saddr ? saddr : dev->dev_addr, dev->addr_len);
	memcpy(eth->h_dest,   daddr ? daddr : dev->dev_addr, dev->addr_len);
	eth->h_dest[ETH_ALEN-1]   ^= 0x01;   /* dest is us xor 1 */
	return (dev->hard_header_len);
}





/*
 * The "change_mtu" method is usually not needed.
 * If you need it, it must be like this.
 */
int snull_change_mtu(struct net_device *dev, int new_mtu)
{
	unsigned long flags;
	struct snull_priv *priv = netdev_priv(dev);
	spinlock_t *lock = &priv->lock;
    
	/* check ranges */
	if ((new_mtu < 68) || (new_mtu > 1500))
		return -EINVAL;
	/*
	 * Do anything you need, and the accept the value
	 */
	spin_lock_irqsave(lock, flags);
	dev->mtu = new_mtu;
	spin_unlock_irqrestore(lock, flags);
	return 0; /* success */
}

/*
 * The init function (sometimes called probe).
 * It is invoked by register_netdev()
 */
//这个是setup函数，用于初始化net_device部分结构，并作为alloc_netdev的第三个参数传入
void snull_init(struct net_device *dev)
{
	struct snull_priv *priv;
#if 0
    	/*
	 * Make the usual checks: check_region(), probe irq, ...  -ENODEV
	 * should be returned if no device found.  No resource should be
	 * grabbed: this is done on open(). 
	 */
#endif

    	/* 
	 * Then, assign other fields in dev, using ether_setup() and some
	 * hand assignments
	 */
	ether_setup(dev); /* assign some of the fields */

	dev->open            = snull_open;
	dev->stop            = snull_release;
	dev->set_config      = snull_config;
	dev->hard_start_xmit = snull_tx;
	dev->do_ioctl        = snull_ioctl;
	dev->get_stats       = snull_stats;
	dev->change_mtu      = snull_change_mtu;  
	dev->rebuild_header  = snull_rebuild_header;
	dev->hard_header     = snull_header;
	dev->tx_timeout      = snull_tx_timeout;
	dev->watchdog_timeo = timeout;
	if (use_napi) {
		dev->poll        = snull_poll;
		dev->weight      = 2;
	}
	/* keep the default flags, just add NOARP */
	dev->flags           |= IFF_NOARP;
	dev->features        |= NETIF_F_NO_CSUM;
	dev->hard_header_cache = NULL;      /* Disable caching */

	/*
	 * Then, initialize the priv field. This encloses the statistics
	 * and a few private fields.
	 */
	priv = netdev_priv(dev);
	memset(priv, 0, sizeof(struct snull_priv));
	spin_lock_init(&priv->lock);
	snull_rx_ints(dev, 1);		/* enable receive interrupts */
	snull_setup_pool(dev);
}

/*
 * The devices
 */

struct net_device *snull_devs[2];



/*
 * Finally, the module stuff
 */
//驱动卸载时调用的函数
void snull_cleanup(void)
{
	int i;
    
	for (i = 0; i < 2;  i++) {
		if (snull_devs[i]) {
			unregister_netdev(snull_devs[i]);//反注册net_deivce
			snull_teardown_pool(snull_devs[i]);
			free_netdev(snull_devs[i]);
		}
	}
	return;
}



//驱动加载时的初始化函数
int snull_init_module(void)
{
	int result, i, ret = -ENOMEM;

	snull_interrupt = use_napi ? snull_napi_interrupt : snull_regular_interrupt;

	/* Allocate the devices */
	snull_devs[0] = alloc_netdev(sizeof(struct snull_priv), "sn%d",//调用的分配net_deivce函数
			snull_init);
	snull_devs[1] = alloc_netdev(sizeof(struct snull_priv), "sn%d",//这里"sn%d"内核会使用dev_alloc_name以完成该名字，次函数会把%d换成该设备类型中头一个未分配的数字
			snull_init);
	if (snull_devs[0] == NULL || snull_devs[1] == NULL)
		goto out;

	ret = -ENODEV;
	for (i = 0; i < 2;  i++)
		if ((result = register_netdev(snull_devs[i])))//注册net_device
			printk("snull: error %i registering device \"%s\"\n",
					result, snull_devs[i]->name);
		else
			ret = 0;
   out:
	if (ret) 
		snull_cleanup();
	return ret;
}


module_init(snull_init_module);
module_exit(snull_cleanup);

```

#### 其他部分，如子系统的初始化，等分析到了再写
