---
title: memory_overiew
date: 2021-03-07 04:35:25
tags: memory
categories: linux
---

### linux内存系统概述
linux的内存系统是一个很复杂的系统，需要从几个角度去分析，里面包含多种机制，比如虚拟地址和物理地址的转换等等；本文是一个系统性的大局观来从几个方面分析内存系统的记录：<!--more-->
1 linux内存系统的地址空间：主要解释虚拟内存空间的机制和地址转换关系；  
2 liunx内存系统的物理内存相关结构和虚拟内存相关结构：主要解释内核用什么样的数据结构来管理内存  
3 linux内存初始化  
4 linux内存系统的分配路线：从kernel和用户进程两路分配出发，最后都用到伙伴系统分配内存页；  
5 linux内存的回收机制简单分析；  
6 linux下内存的查看等：  
其中涉及的调试方法和模块代码例子等等，包含在各个子节中；  

### linux内存系统的地址空间：
#### 简介：
一般来说，linux系统在使用时，只要一块物理内存，或者说，即使外部插入多个内存，最后再使用时，也是一个物理内存的地址空间;最开始的系统，使用内存都是直接通过取物理地址写入读取的;
而演变为采用虚拟地址空间，并通过MMU地址转换为物理地址，再访问真正的物理内存，其作用，一方面是进一步保护内存防止滥用和增加控制，另一方面，也对每个进程之间起到隔离作用，加强保护；
另外，还可以避免因为直接操作物理内存的情况下，不同进程操作时地址的不连续性带来更高的复杂度，更多作用，我想通过进一步学习，可以加深理解；
#### 虚拟地址空间：
   + 简介：  
	一般来说，linux内核把处理器的虚拟地址空间分为两个部分，底部比较大的部分用于用户进程，顶部则是专用于内核；所以在两个用户进程上下文切换的时候，只会改变下半部分；
而内核部分总是不变；所以不同的用户进程，有不同的虚拟地址空间，也就是说，两个不同的进程中的两个变量可以有相同的虚拟地址，但是实际物理内存确是不同的；
划分比如4G的内存，1G给到内核，3G给到用户空间；当然可以通过配置修改这种比例；  
   + 举例  
   内核和用户进程，在操作时，变量等结构，函数等打印出来的指针(地址)，是虚拟地址，而实际写入和读取的数据是放在物理内存的，是有唯一的物理内存地址的，那虚拟地址是如何对应到物理内存地址的呢？  
   这个涉及到虚拟地址到物理地址的转换--MMU,借助MMU，linux可以把虚拟地址通过某种映射转换为物理地址；
   + MMU(内存管理单元):   
   是计算机系统的一个物理部件，通常是CPU的一部分(但不一定)， linux的MMU是一个很复杂的模块，本文暂时不会进行详细的分析，等后面有时间再深入研究：
  关于MMU是如何将虚拟地址转换为物理地址的过程，可以参考下图：
{% asset_img MMU.png MMU about %}

### linux内存系统的相关结构：
主要分为这几个部分：
+ 基础的，物理内存：
+ 和MMU相关的
+ 和虚拟内存相关的：和内核相关的，和用户进程相关的；
#### 基础的，物理内存：
+ NUMA概念：VM中流行的第一个主要概念是非均匀内存访问(NUMA)。在大型机器中，内存可能被根据与处理器的“距离”而产生不同的存取成本，划分为多个部分(结点)。例如，可能有一个内存库分配给每个CPU或一个内存库非常适合DMA近设备卡。每个部分称为一个node,由挂在同一个CPU下的一片连续的物理内存组成，在内核中使用pg_data_t进行抽象.
可以从linux/mmzone.h来找到相关结构：
{% asset_img NUMA.png NUMA about %}


+ NUMA的结点node和相关的结构：zone:  
{% asset_img numa_zone.png numa_zone about %}
由于一些特殊的应用场景，导致只能分配特定地址范围内的内存（比如老式的ISA设备DMA时只能使用前16M内存；比如kmalloc只能分配低端内存，而不能分配高端内存），因此在node中又将内存细分为zone。  
zone 有以下几种类型，由zone结构中的flags标识；  
   1) ZONE_DMA：定义适合DMA的内存域，该区域的长度依赖于处理器类型。比如ARM所有地址都可以进行DMA，所以该值可以很大，或者干脆不定义DMA类型的内存域。而在IA-32的处理器上，一般定义为16M。  
   2) ZONE_DMA32：只在64位系统上有效，为一些32位外设DMA时分配内存。如果物理内存大于4G，该值为4G，否则与实际的物理内存大小相同。  
   3) ZONE_NORMAL：定义可直接映射到内核空间的普通内存域。在64位系统上，如果物理内存小于4G，该内存域为空。而在32位系统上，该值最大为896M。  
   4) ZONE_HIGHMEM：只在32位系统上有效，标记超过896M范围的内存。在64位系统上，由于地址空间巨大，超过4G的内存都分布在ZONE_NORMA内存域。     
   5) ZONE_MOVABLE：伪内存域，为了实现减小内存碎片的机制。


+ zone结构下的free_area,free_list和伙伴系统：
简单的说，就是zone下维护的free_area,是维护2^n个page的内存块的链表的数据结构，用来方便分配指定大小的内存页以及进行合并，拆分等，解决页外碎片的机制：
更详细可以参考：http://blog.chinaunix.net/uid-30282771-id-5185451.html
或者翻阅资料；
```cpp
cat /proc/buddyinfo 
Node 0, zone      DMA      0      0      0      0      2      1      1      0      1      1      3 
Node 0, zone    DMA32      4      2      3     28    149     21     13      8      1      2    178 
Node 0, zone   Normal      2     84     46     23     13     13     15     21      1      0      0 
```
+ page：即物理页，物理内存是按页为单元进行分配管理的，一页通常是4k
+ 以下是在linux4.8中以上各个数据结构的定义：  
linux/mmzone.h pg_data_t:  
```cpp
typedef struct pglist_data {
	struct zone node_zones[MAX_NR_ZONES];
	struct zonelist node_zonelists[MAX_ZONELISTS];
	int nr_zones;
#ifdef CONFIG_FLAT_NODE_MEM_MAP	/* means !SPARSEMEM */
	struct page *node_mem_map;
#ifdef CONFIG_PAGE_EXTENSION
	struct page_ext *node_page_ext;
#endif
#endif
#ifndef CONFIG_NO_BOOTMEM
	struct bootmem_data *bdata;
#endif
#ifdef CONFIG_MEMORY_HOTPLUG
	/*
	 * Must be held any time you expect node_start_pfn, node_present_pages
	 * or node_spanned_pages stay constant.  Holding this will also
	 * guarantee that any pfn_valid() stays that way.
	 *
	 * pgdat_resize_lock() and pgdat_resize_unlock() are provided to
	 * manipulate node_size_lock without checking for CONFIG_MEMORY_HOTPLUG.
	 *
	 * Nests above zone->lock and zone->span_seqlock
	 */
	spinlock_t node_size_lock;
#endif
	unsigned long node_start_pfn;
	unsigned long node_present_pages; /* total number of physical pages */
	unsigned long node_spanned_pages; /* total size of physical page
					     range, including holes */
	int node_id;
	wait_queue_head_t kswapd_wait;
	wait_queue_head_t pfmemalloc_wait;
	struct task_struct *kswapd;	/* Protected by
					   mem_hotplug_begin/end() */
	int kswapd_order;
	enum zone_type kswapd_classzone_idx;

#ifdef CONFIG_COMPACTION
	int kcompactd_max_order;
	enum zone_type kcompactd_classzone_idx;
	wait_queue_head_t kcompactd_wait;
	struct task_struct *kcompactd;
#endif
#ifdef CONFIG_NUMA_BALANCING
	/* Lock serializing the migrate rate limiting window */
	spinlock_t numabalancing_migrate_lock;

	/* Rate limiting time interval */
	unsigned long numabalancing_migrate_next_window;

	/* Number of pages migrated during the rate limiting time interval */
	unsigned long numabalancing_migrate_nr_pages;
#endif
	/*
	 * This is a per-node reserve of pages that are not available
	 * to userspace allocations.
	 */
	unsigned long		totalreserve_pages;

#ifdef CONFIG_NUMA
	/*
	 * zone reclaim becomes active if more unmapped pages exist.
	 */
	unsigned long		min_unmapped_pages;
	unsigned long		min_slab_pages;
#endif /* CONFIG_NUMA */

	/* Write-intensive fields used by page reclaim */
	ZONE_PADDING(_pad1_)
	spinlock_t		lru_lock;

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
	/*
	 * If memory initialisation on large machines is deferred then this
	 * is the first PFN that needs to be initialised.
	 */
	unsigned long first_deferred_pfn;
#endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	spinlock_t split_queue_lock;
	struct list_head split_queue;
	unsigned long split_queue_len;
#endif

	/* Fields commonly accessed by the page reclaim scanner */
	struct lruvec		lruvec;

	/*
	 * The target ratio of ACTIVE_ANON to INACTIVE_ANON pages on
	 * this node's LRU.  Maintained by the pageout code.
	 */
	unsigned int inactive_ratio;

	unsigned long		flags;

	ZONE_PADDING(_pad2_)

	/* Per-node vmstats */
	struct per_cpu_nodestat __percpu *per_cpu_nodestats;
	atomic_long_t		vm_stat[NR_VM_NODE_STAT_ITEMS];
} pg_data_t;
pg_data_t中描述了node的所有基本元素：
    1) node_zones: 该node包含的内存域zone
    2) node_zonelists：该node的备选节点及内存域列表，后面会详细说明。
    3) node_mem_map：linux为每个物理页分配了一个struct page的管理结构体，并形成了一个结构体数组，node_mem_map即为数组的指针；pfn_to_page和page_to_pfn都借助该数组实现。
    4) node_start_pfn：该node中内存的起始页帧号
    5) node_present_pages：该node中所有的物理page页数
    6) node_spanned_pages：该node地址范围内的所有page页数，包括空洞；目前还不清楚什么情况导致与node_present_pages不同。
    7) kswapd：负责回收该node内存的内核线程，每个node对应一个内核线程kswapd。
```

zone:  
```cpp
struct zone {
	/* Read-mostly fields */

	/* zone watermarks, access with *_wmark_pages(zone) macros */
	unsigned long watermark[NR_WMARK];

	/*
	 * We don't know if the memory that we're going to allocate will be freeable
	 * or/and it will be released eventually, so to avoid totally wasting several
	 * GB of ram we must reserve some of the lower zone memory (otherwise we risk
	 * to run OOM on the lower zones despite there's tons of freeable ram
	 * on the higher zones). This array is recalculated at runtime if the
	 * sysctl_lowmem_reserve_ratio sysctl changes.
	 */
	long lowmem_reserve[MAX_NR_ZONES];

#ifdef CONFIG_NUMA
	int node;
#endif

	/*
	 * The target ratio of ACTIVE_ANON to INACTIVE_ANON pages on
	 * this zone's LRU.  Maintained by the pageout code.
	 */
	unsigned int inactive_ratio;

	struct pglist_data	*zone_pgdat;
	struct per_cpu_pageset __percpu *pageset;

	/*
	 * This is a per-zone reserve of pages that should not be
	 * considered dirtyable memory.
	 */
	unsigned long		dirty_balance_reserve;

#ifndef CONFIG_SPARSEMEM
	/*
	 * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
	 * In SPARSEMEM, this map is stored in struct mem_section
	 */
	unsigned long		*pageblock_flags;
#endif /* CONFIG_SPARSEMEM */

#ifdef CONFIG_NUMA
	/*
	 * zone reclaim becomes active if more unmapped pages exist.
	 */
	unsigned long		min_unmapped_pages;
	unsigned long		min_slab_pages;
#endif /* CONFIG_NUMA */

	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	unsigned long		zone_start_pfn;

	/*
	 * spanned_pages is the total pages spanned by the zone, including
	 * holes, which is calculated as:
	 * 	spanned_pages = zone_end_pfn - zone_start_pfn;
	 *
	 * present_pages is physical pages existing within the zone, which
	 * is calculated as:
	 *	present_pages = spanned_pages - absent_pages(pages in holes);
	 *
	 * managed_pages is present pages managed by the buddy system, which
	 * is calculated as (reserved_pages includes pages allocated by the
	 * bootmem allocator):
	 *	managed_pages = present_pages - reserved_pages;
	 *
	 * So present_pages may be used by memory hotplug or memory power
	 * management logic to figure out unmanaged pages by checking
	 * (present_pages - managed_pages). And managed_pages should be used
	 * by page allocator and vm scanner to calculate all kinds of watermarks
	 * and thresholds.
	 *
	 * Locking rules:
	 *
	 * zone_start_pfn and spanned_pages are protected by span_seqlock.
	 * It is a seqlock because it has to be read outside of zone->lock,
	 * and it is done in the main allocator path.  But, it is written
	 * quite infrequently.
	 *
	 * The span_seq lock is declared along with zone->lock because it is
	 * frequently read in proximity to zone->lock.  It's good to
	 * give them a chance of being in the same cacheline.
	 *
	 * Write access to present_pages at runtime should be protected by
	 * mem_hotplug_begin/end(). Any reader who can't tolerant drift of
	 * present_pages should get_online_mems() to get a stable value.
	 *
	 * Read access to managed_pages should be safe because it's unsigned
	 * long. Write access to zone->managed_pages and totalram_pages are
	 * protected by managed_page_count_lock at runtime. Idealy only
	 * adjust_managed_page_count() should be used instead of directly
	 * touching zone->managed_pages and totalram_pages.
	 */
	unsigned long		managed_pages;
	unsigned long		spanned_pages;
	unsigned long		present_pages;

	const char		*name;

	/*
	 * Number of MIGRATE_RESERVE page block. To maintain for just
	 * optimization. Protected by zone->lock.
	 */
	int			nr_migrate_reserve_block;

#ifdef CONFIG_MEMORY_ISOLATION
	/*
	 * Number of isolated pageblock. It is used to solve incorrect
	 * freepage counting problem due to racy retrieving migratetype
	 * of pageblock. Protected by zone->lock.
	 */
	unsigned long		nr_isolate_pageblock;
#endif

#ifdef CONFIG_MEMORY_HOTPLUG
	/* see spanned/present_pages for more description */
	seqlock_t		span_seqlock;
#endif

	/*
	 * wait_table		-- the array holding the hash table
	 * wait_table_hash_nr_entries	-- the size of the hash table array
	 * wait_table_bits	-- wait_table_size == (1 << wait_table_bits)
	 *
	 * The purpose of all these is to keep track of the people
	 * waiting for a page to become available and make them
	 * runnable again when possible. The trouble is that this
	 * consumes a lot of space, especially when so few things
	 * wait on pages at a given time. So instead of using
	 * per-page waitqueues, we use a waitqueue hash table.
	 *
	 * The bucket discipline is to sleep on the same queue when
	 * colliding and wake all in that wait queue when removing.
	 * When something wakes, it must check to be sure its page is
	 * truly available, a la thundering herd. The cost of a
	 * collision is great, but given the expected load of the
	 * table, they should be so rare as to be outweighed by the
	 * benefits from the saved space.
	 *
	 * __wait_on_page_locked() and unlock_page() in mm/filemap.c, are the
	 * primary users of these fields, and in mm/page_alloc.c
	 * free_area_init_core() performs the initialization of them.
	 */
	wait_queue_head_t	*wait_table;
	unsigned long		wait_table_hash_nr_entries;
	unsigned long		wait_table_bits;

	ZONE_PADDING(_pad1_)
	/* free areas of different sizes */
	struct free_area	free_area[MAX_ORDER];

	/* zone flags, see below */
	unsigned long		flags;

	/* Write-intensive fields used from the page allocator */
	spinlock_t		lock;

	ZONE_PADDING(_pad2_)

	/* Write-intensive fields used by page reclaim */

	/* Fields commonly accessed by the page reclaim scanner */
	spinlock_t		lru_lock;
	struct lruvec		lruvec;

	/* Evictions & activations on the inactive file list */
	atomic_long_t		inactive_age;

	/*
	 * When free pages are below this point, additional steps are taken
	 * when reading the number of free pages to avoid per-cpu counter
	 * drift allowing watermarks to be breached
	 */
	unsigned long percpu_drift_mark;

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* pfn where compaction free scanner should start */
	unsigned long		compact_cached_free_pfn;
	/* pfn where async and sync compaction migration scanner should start */
	unsigned long		compact_cached_migrate_pfn[2];
#endif

#ifdef CONFIG_COMPACTION
	/*
	 * On compaction failure, 1<<compact_defer_shift compactions
	 * are skipped before trying again. The number attempted since
	 * last failure is tracked with compact_considered.
	 */
	unsigned int		compact_considered;
	unsigned int		compact_defer_shift;
	int			compact_order_failed;
#endif

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* Set to true when the PG_migrate_skip bits should be cleared */
	bool			compact_blockskip_flush;
#endif

	ZONE_PADDING(_pad3_)
	/* Zone statistics */
	atomic_long_t		vm_stat[NR_VM_ZONE_STAT_ITEMS];
} ____cacheline_internodealigned_in_smp;
```
free_area:注意在zone定义中数组大小只有11，意味着是只支持到2^10个页大小的内存块
```cpp
struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;
};
```  

/include/linux/mm_types.h
```cpp
/*
 * Each physical page in the system has a struct page associated with
 * it to keep track of whatever it is we are using the page for at the
 * moment. Note that we have no way to track which tasks are using
 * a page, though if it is a pagecache page, rmap structures can tell us
 * who is mapping it.
 *
 * The objects in struct page are organized in double word blocks in
 * order to allows us to use atomic double word operations on portions
 * of struct page. That is currently only used by slub but the arrangement
 * allows the use of atomic double word operations on the flags/mapping
 * and lru list pointers also.
 */
struct page {
	/* First double word block */
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	union {
		struct address_space *mapping;	/* If low bit clear, points to
						 * inode address_space, or NULL.
						 * If page mapped as anonymous
						 * memory, low bit is set, and
						 * it points to anon_vma object:
						 * see PAGE_MAPPING_ANON below.
						 */
		void *s_mem;			/* slab first object */
		atomic_t compound_mapcount;	/* first tail page */
		/* page_deferred_list().next	 -- second tail page */
	};

	/* Second double word */
	struct {
		union {
			pgoff_t index;		/* Our offset within mapping. */
			void *freelist;		/* sl[aou]b first free object */
			/* page_deferred_list().prev	-- second tail page */
		};

		union {
#if defined(CONFIG_HAVE_CMPXCHG_DOUBLE) && \
	defined(CONFIG_HAVE_ALIGNED_STRUCT_PAGE)
			/* Used for cmpxchg_double in slub */
			unsigned long counters;
#else
			/*
			 * Keep _count separate from slub cmpxchg_double data.
			 * As the rest of the double word is protected by
			 * slab_lock but _count is not.
			 */
			unsigned counters;
#endif

			struct {

				union {
					/*
					 * Count of ptes mapped in mms, to show
					 * when page is mapped & limit reverse
					 * map searches.
					 */
					atomic_t _mapcount;

					struct { /* SLUB */
						unsigned inuse:16;
						unsigned objects:15;
						unsigned frozen:1;
					};
					int units;	/* SLOB */
				};
				atomic_t _count;		/* Usage count, see below. */
			};
			unsigned int active;	/* SLAB */
		};
	};

	/*
	 * Third double word block
	 *
	 * WARNING: bit 0 of the first word encode PageTail(). That means
	 * the rest users of the storage space MUST NOT use the bit to
	 * avoid collision and false-positive PageTail().
	 */
	union {
		struct list_head lru;	/* Pageout list, eg. active_list
					 * protected by zone->lru_lock !
					 * Can be used as a generic list
					 * by the page owner.
					 */
		struct dev_pagemap *pgmap; /* ZONE_DEVICE pages are never on an
					    * lru or handled by a slab
					    * allocator, this points to the
					    * hosting device page map.
					    */
		struct {		/* slub per cpu partial pages */
			struct page *next;	/* Next partial slab */
#ifdef CONFIG_64BIT
			int pages;	/* Nr of partial slabs left */
			int pobjects;	/* Approximate # of objects */
#else
			short int pages;
			short int pobjects;
#endif
		};

		struct rcu_head rcu_head;	/* Used by SLAB
						 * when destroying via RCU
						 */
		/* Tail pages of compound page */
		struct {
			unsigned long compound_head; /* If bit zero is set */

			/* First tail page only */
#ifdef CONFIG_64BIT
			/*
			 * On 64 bit system we have enough space in struct page
			 * to encode compound_dtor and compound_order with
			 * unsigned int. It can help compiler generate better or
			 * smaller code on some archtectures.
			 */
			unsigned int compound_dtor;
			unsigned int compound_order;
#else
			unsigned short int compound_dtor;
			unsigned short int compound_order;
#endif
		};

#if defined(CONFIG_TRANSPARENT_HUGEPAGE) && USE_SPLIT_PMD_PTLOCKS
		struct {
			unsigned long __pad;	/* do not overlay pmd_huge_pte
						 * with compound_head to avoid
						 * possible bit 0 collision.
						 */
			pgtable_t pmd_huge_pte; /* protected by page->ptl */
		};
#endif
	};

	/* Remainder is not double word aligned */
	union {
		unsigned long private;		/* Mapping-private opaque data:
					 	 * usually used for buffer_heads
						 * if PagePrivate set; used for
						 * swp_entry_t if PageSwapCache;
						 * indicates order in the buddy
						 * system if PG_buddy is set.
						 */
#if USE_SPLIT_PTE_PTLOCKS
#if ALLOC_SPLIT_PTLOCKS
		spinlock_t *ptl;
#else
		spinlock_t ptl;
#endif
#endif
		struct kmem_cache *slab_cache;	/* SL[AU]B: Pointer to slab */
	};

#ifdef CONFIG_MEMCG
	struct mem_cgroup *mem_cgroup;
#endif

	/*
	 * On machines where all RAM is mapped into kernel address space,
	 * we can simply calculate the virtual address. On machines with
	 * highmem some memory is mapped into kernel virtual memory
	 * dynamically, so we need a place to store that address.
	 * Note that this field could be 16 bits on x86 ... ;)
	 *
	 * Architectures with slow multiplication can define
	 * WANT_PAGE_VIRTUAL in asm/page.h
	 */
#if defined(WANT_PAGE_VIRTUAL)
	void *virtual;			/* Kernel virtual address (NULL if
					   not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */

#ifdef CONFIG_KMEMCHECK
	/*
	 * kmemcheck wants to track the status of each byte in a page; this
	 * is a pointer to such a status block. NULL if not tracked.
	 */
	void *shadow;
#endif

#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
	int _last_cpupid;
#endif
}
/*
 * The struct page can be forced to be double word aligned so that atomic ops
 * on double words work. The SLUB allocator can make use of such a feature.
 */
#ifdef CONFIG_HAVE_ALIGNED_STRUCT_PAGE
	__aligned(2 * sizeof(unsigned long))
#endif
;
```
##### 实践：操作zone等：
```cpp
/**********************************************
  * Author: ksx
  * File name: sysctl_example.c
  * Description: sysctl example
  * Date: 2021-02-26
  *********************************************/
 
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/sysctl.h>
#include <linux/swap.h>
 #include<linux/topology.h>
#include<linux/gfp.h>
#include<linux/mm_types.h>
#include<linux/mm.h>

struct page *pages = NULL;
static int __init sysctl_example_init(void)
{

    pages = alloc_pages(GFP_KERNEL,8);
    if(!pages) 
    { 
        return -ENOMEM; 
    } 
    else 
    { 
        printk("alloc_pages Successfully!\n"); 
        printk("page_address(pages) = 0x%lx\n",(unsigned long)page_address(pages));
    }
    struct zone *arr = NODE_DATA(numa_node_id())->node_zones;
   // printk(KERN_INFO,"zone: some:%ld\n",z1.managed_pages);
    struct zonelist *zoneli = node_zonelist(numa_node_id(),GFP_KERNEL);
    unsigned int free_areasize = sizeof(arr->free_area)/sizeof(arr->free_area[0]);
    struct free_area *fa = arr->free_area;
    int freearea0_nrfree = arr->free_area[0].nr_free;

    printk(KERN_INFO "sysctl registercess %ld num_node_id:%d freeareasize :%u freelist:: %ld nr_free:%d\n",nr_free_buffer_pages(),numa_node_id(),free_areasize,sizeof(fa->free_list)/sizeof(fa->free_list[0]),freearea0_nrfree);
    return 0;
 
}
 
static void __exit sysctl_example_exit(void)
{
    printk(KERN_INFO "sysctl unregister success.\n");
    if(pages) 
    {
        __free_pages(pages,8);   // 释放所分配的

 
        printk("__free_pages ok!\n"); 
    }
}
 
module_init(sysctl_example_init);
module_exit(sysctl_example_exit);
MODULE_LICENSE("GPL");

```
#### 和MMU相关的：
+ TLB:
地址转换需要几次内存访问，内存访问相对CPU速度较慢。为了避免在地址转换上花费宝贵的处理器周期，cpu维护了一个名为翻译后备缓冲区(translation Lookaside Buffer, TLB)的此类转换缓存。
+ Huge Pages:Linux中有两种机制支持将物理内存映射到大页面。第一个是HugeTLB文件系统，或hugetlbfs。它是一个使用RAM作为后备存储的伪文件系统。对于在该文件系统中创建的文件，数据驻留在内存中，并使用大页进行映射。
+ page tables:页表,页表是用来建立用户进程虚拟地址空间和系统物理内存(页帧等)之间的关联：即通过上述图1可以看到线性(虚拟)地址和页表的关系，linux用三级甚至4级页表来管理内存页
通过页表，每个进程提供一致的虚拟地址空间，应用程序看到的地址空间是一个连续的内存区，该表也将虚拟内存页映射到物理内存，进而支持共享内存的实现，即几个进程共享的内存；
相关结构： /asm-arch/page.h  asm-arch/pgtable.h和具体的体系架构相关；  

每个进程都有一个指针(mm_struct→pgd)指向它自己的页全局目录(pgd)， pgd是一个物理页帧。这个框架包含一个类型为pgd_t的数组，pgd_t是在<asm/page.h>中定义的架构特定类型。根据架构的不同，页表的加载方式也不同。在x86上，通过将mm_struct→pgd复制到cr3寄存器来加载进程页表，这有冲洗TLB的副作用。事实上，这就是函数__flush_tlb()在依赖于体系结构的代码中实现的方式。
PGD表中的每个活动条目指向一个包含pmd_t类型的页中间目录(PMD)项数组的页框架，该页框架又指向一个包含pte_t类型的页表项(PTE)的页框架，pte_t类型的页表项最终指向包含实际用户数据的页框架。如果页面被交换到后备存储，交换项将存储在PTE中，并在页面故障期间由do_swap_page()使用，以查找包含页面数据的交换项。  

页表项的一些操作函数:pgd_alloc(如 /arch/arm64/mm/pgd.c),pte_page,pud_alloc等等  
```cpp
页表：层次化的页表用于支持对大地址空间的快速，高效管理；
用于建立用户进程的虚拟地址空间和系统物理内存（内存，页帧）之间的关联
页表也用于向每个进程提供一致的虚拟地址空间，应用程序看到的地址空间是一个连续的内存区，该表也将虚拟内存页映射到物理内存，因而支持共享内存的实现；
内核内存管理使用四级页表，不管底层处理器，一个页表有1024项，每项对应一个页(4K)
内存虚拟地址的分解：
页表格式：   /*
   * These are used to make use of C type-checking..
   */
  typedef struct { unsigned long pte; } pte_t;
  typedef struct { unsigned long pmd; } pmd_t;
#if CONFIG_PGTABLE_LEVELS == 4
  typedef struct { unsigned long pud; } pud_t;
#endif
  typedef struct { unsigned long pgd; } pgd_t;
  typedef struct { unsigned long pgprot; } pgprot_t;
  typedef struct page *pgtable_t;

用于处理内存页的体系结构相关状态的函数以及页表项操作函数
```
{% asset_img pgtable.png pgtable  about %}
{% asset_img pgtable2.png pgtable  about %}
更多参考：https://www.kernel.org/doc/gorman/html/understand/understand006.html

+ 和虚拟内存相关的：
内核相关：  
内核内存分配器：slab/slub/slob,具体使用场景和机制见下 内核的内存分配  
eg:
```cpp
struct slab {
	struct list_head list;		//slab描述符的三个双向循环链表中的一个
	unsigned long colouroff;	//slab中第一个对象
	void *s_mem;			//slab中第一个对象的地址
	unsigned int inuse;		//当前正在使用的slab中的对象的个数
	kmem_bufctl_t free;		//slab中第一个空闲对象的下标。
	unsigned short nodeid;
}
```

用户进程相关：具体怎么用见下 用户进程的内存分配  
/include/linux/mm_types.h vm_area_struct  
在 task_struct->mm_struct->vm_area_struct* 链表  
```cpp
/*
 * This struct defines a memory VMM memory area. There is one of these
 * per VM-area/task.  A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */

	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;

	/*
	 * Largest free memory gap in bytes to the left of this VMA.
	 * Either between this VMA and vma->vm_prev, or between one of the
	 * VMAs below us in the VMA rbtree and its ->vm_prev. This helps
	 * get_unmapped_area find a free area of the right size.
	 */
	unsigned long rb_subtree_gap;

	/* Second cache line starts here. */

	struct mm_struct *vm_mm;	/* The address space we belong to. */
	pgprot_t vm_page_prot;		/* Access permissions of this VMA. */
	unsigned long vm_flags;		/* Flags, see mm.h. */

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap interval tree.
	 */
	struct {
		struct rb_node rb;
		unsigned long rb_subtree_last;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_chain; /* Serialized by mmap_sem &
					  * page_table_lock */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */

	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units */
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* was vm_pte (shared mem) */

#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
	struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
};
```

### linux内存初始化简介：
+ 概述： 
初始化大都是建立全局数据结构并做初始化，设置系统状态，寄存器等等，linux内存也是大致如此；
linux内存初始化有个过程比较特别，就是在建立起这个内存管理之前，也要内存，但是此时是不能用内存管理接口的，那在这期间，是使用了一个额外的简化形式的内存管理模块(bootmem)
在初始化完成内存管理后，再丢弃掉；
linux内存初始化主要是体系架构相关的初始化和机器无关的初始化：前者主要是统计系统可用内存总数，以及cpu模型，结点node数量和内存域之间的分配情况，后者主要是建立起前面介绍的各个必要结构，比如pg_data_t;前者完成后，才能完成后者；
我们可以通过一个宏 mmzone.h: #define NODE_DATA(nid)		(&contig_page_data) 来获取到对应node编号的 pg_data_t实例，进而进行操作；  

+ 系统启动时内存的的初始化流程：
在初始化过程中主要包含以下几个步骤，在如下源文件中：
依然是从：  
```cpp
 arch/ia64/kernel/head.S, line 413
    arch/arm64/kernel/head.S, line 453
    等都能找到其调用start_kernel，这个函数就是用来启动内核；
    注意这个头文件：include/linux/start_kernel.h, line 11 (as a prototype)
    和这个实现文件：init/main.c, line 849 (as a function)
```c
    head.S ->start_kernel(init/main.c)
      lock_kernel();//首先让lock_depth变量自增，然后判断结果是否为0，如果是，则进行对信号量的自减操作，类似于PV操作
   start_kernel()函数会做很多init初始化，里面自然也有内存相关的初始化：
       setup_arch 体系结构相关的初始化
       setup_per_cpu_areas 每个cpu的初始化
       build_all_zonelists 初始化建立结点和内存区域
       mem_init  特定于体系结构的，用于停用bootmem分配器并迁移到内存管理函数
       kmem_cache_init:初始化内核内部小块内存区的分配器
       setup_per_cpu_pageset：zone下为pageset数组的第一个数组元素分配内存，分配第一个数组元素；
关于结点和内存域初始化：
```cpp
在page_alloc.c 中
void __init build_all_zonelists(void)遍历所有结点进行初始化
{
	int i;

	for(i = 0 ; i < numnodes ; i++)
		build_zonelists(NODE_DATA(i));//通过这个结构可以拿到所有的node_data[] pg_data_t
	printk("Built %i zonelists\n", numnodes);
}
free_area: mmzone.h 
struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;
};
```
通过模块类似的获取机制，来获取当前系统的页详情，以及分配时的动态变化；
build_zonelists：这个函数任务为：在当前处理的结点的内存域和其他结点的内存域之间建立一个等级次序，并下来按这种次序分配内存；
等级次序例子：内存要分配高端内存，则首先企图在当前结点的高端内存中找一个大小合适的空闲段，若失败，则查看该结点的普通内存域，若还是失败，则看
该结点的DMA内存域进行分配，还失败就去找其他结点；备选结点要尽量靠近主结点；
层次结构：
首先分配高端内存：廉价，在于内核没有任何部分依赖该内存域分配的内存；
接着普通内存：许多内核数据结构必须保存在该区域，而不能放置在高端内存域；普通内存用尽就紧急；
最后是DMA内存域，它用于外设和系统的数据传输；
没有再考虑其他内存结点 ；备选结点也有一个等级次序；
```
在初始化后，物理内存大致是这样分布的：
{% asset_img memlayout.png memlayout %}
我们也可以在procfs中查看：
```cpp
sudo cat /proc/iomem
00000000-00000fff : reserved
00001000-0009fbff : System RAM
0009fc00-0009ffff : reserved
000a0000-000bffff : PCI Bus 0000:00
000c0000-000c7fff : Video ROM
000e2000-000ef3ff : Adapter ROM
000f0000-000fffff : reserved
  000f0000-000fffff : System ROM
00100000-dffeffff : System RAM
  01000000-0182e000 : Kernel code
  0182e001-01f4dfff : Kernel data
  020cd000-02207fff : Kernel bss
dfff0000-dfffffff : ACPI Tables
e0000000-fdffffff : PCI Bus 0000:00
  e0000000-e0ffffff : 0000:00:02.0
    e0000000-e0ffffff : vmwgfx probe
  f0000000-f01fffff : 0000:00:02.0
    f0000000-f01fffff : vmwgfx probe
  f0200000-f021ffff : 0000:00:03.0
    f0200000-f021ffff : e1000
  f0400000-f07fffff : 0000:00:04.0
    f0400000-f07fffff : vboxguest
  f0800000-f0803fff : 0000:00:04.0
  f0804000-f0804fff : 0000:00:06.0
    f0804000-f0804fff : ohci_hcd
  f0806000-f0807fff : 0000:00:0d.0
    f0806000-f0807fff : ahci
fec00000-fec00fff : reserved
  fec00000-fec003ff : IOAPIC 0
fee00000-fee00fff : Local APIC
  fee00000-fee00fff : reserved
fffc0000-ffffffff : reserved
100000000-11fffffff : System RAM

如果想看BIOS等占用的物理内存情况，也可以
 dmesg| grep BIOS
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x00000000dffeffff] usable
[    0.000000] BIOS-e820: [mem 0x00000000dfff0000-0x00000000dfffffff] ACPI data
[    0.000000] BIOS-e820: [mem 0x00000000fec00000-0x00000000fec00fff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fee00000-0x00000000fee00fff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000100000000-0x000000011fffffff] usable
[    0.000000] SMBIOS 2.5 present.
[    0.000000] DMI: innotek GmbH VirtualBox/VirtualBox, BIOS VirtualBox 12/01/2006
[    0.000000] ACPI: DSDT 0x00000000DFFF0470 002325 (v02 VBOX   VBOXBIOS 00000002 INTL 20100528)
[    0.000000] Calgary: detecting Calgary via BIOS EBDA area
[    5.020947] BIOS EDD facility v0.16 2004-Jun-25, 0 devices found

```
更多见源码和其他文章参考：eg: https://www.kernel.org/doc/gorman/html/understand/understand008.html

### linux内存分配：
#### linux内存分配图谱：
{% asset_img allalloc.png allalloc%}

#### linux内存用户进程分配:
##### 用于进程内存相关机制：
 A:简介：
+ 进程的虚拟地址空间技术使得不同进程可以同时运行，而不会干扰到其他进程的内存；而所有进程的关联，在于物理内存中的页帧和所有进程虚拟地址空间中的页之间的关联：逆向映射，从虚拟内存页追踪物理页，缺页处理：从块设备读取数据填充虚拟地址空间；
+ 在巨大的线性地址空间中有很少的段可以用于各个用户空间进程，这些段需要被内核管理
+ 内核信任自身，不信任用户进程，所以用户进程在操作地址时需要接受权限等的检查
+ fork-exec模型：写时复制 

B:进程虚拟地址空间
+ 用户进程：0--TASK_SIZE-1 ,其上是内核地址 (虚拟地址：内核1G,进程3G：32位）
进程地址空间布局：text段，动态库代码,全局变量和动态产生数据的堆，局部变量和实现函数的栈，环境变量和命令行参数的段，将文件内容映射到虚拟地址空间中的内存映射；（mm_type.h: 见mm_struct的定义)
+ 建立内存空间布局：
(1) 输入./xxx文件运行时，由exec系统调用执行，并通过load_elf_binary函数装载一个ELF二进制文件，elf装载涉及太复杂流程，只看重要的；
(2) randomsize_via_space设置，用于启动地址随机化，可能会拖慢cpu速度
(3) 布局工作由arch_pick_mmap_layout完成，和体系结构相关；用户通过/proc/sys/kernel/legacy_va_layout输出得到指示，是执行新布局还是旧布局，无非是栈的上界确定和mmap的上下界确定等，最后确定栈的开始位置，mmap开始位置和堆的开始位置等等，这样便完成了布局
到这个时候，就会建立在task结构中的mm结构中，初始化各个段的位置，比如text段，堆栈段开始结束等；也会建立起vm_area_struct;
这里可能会有个疑问，就是vm_area_struct和进程中的数据段，代码段和堆栈段的关系：可能会认为一个vm_area_struct就是一个段，可能是代码段和数据段，但是实际实践发现不是；
vm_area_struct是进程中的内存区域，或者说是内存单元：其本身结构中的字段决定了它的大小，通常是页的整数倍，而数据段代码段等，mm_struct中已经有字段来表明边界；
```cpp

#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include<linux/sched.h>
#include<linux/pid.h>

static int __init sysctl_example_init(void)
{

    struct pid *tpid = find_get_pid(12473);
    struct task_struct *task = pid_task(tpid,PIDTYPE_PID);
    printk("the state of the task is:%d\n",task->state);     // 显示任务当前所处的状态
    printk("the pid of the task is:%d\n",task->pid);         // 显示任务的进程号
    printk("the pid of current thread is:%d\n",current->pid);// 显示当前进程的进程号
    struct mm_struct * mm_task=get_task_mm(task);             // 获取任务的内存描述符   
     printk("the mm_users of the mm_struct is:%d\n",mm_task->mm_users);
    printk("the mm_count of the mm_struct is:%d\n total vm :%ld exec_vm:%ld",mm_task->mm_count,mm_task->total_vm,mm_task->exec_vm); 
    struct vm_area_struct *vm= mm_task->mmap;
    printk("mmap base:%016x ,task_size:%016x,start_code:%016x,end_code:%016x,start_data:%016x, end_data:%016x,start_brk:%016x,endbrk:%016x, start_stack:%016x\n",mm_task->mmap_base,mm_task->task_size,mm_task->start_code,mm_task->end_code,mm_task->start_data,mm_task->end_data,mm_task->start_brk,mm_task->brk,mm_task->start_stack);
    printk("the first vmarea:start:%016x ,end:%016x\n",vm->vm_start,vm->vm_end);
    vm++;
    printk("the  vmarea:start:%016x ,end:%016x\n",vm->vm_start,vm->vm_end);
    vm++;
    printk("the first vmarea:start:%016x ,end:%016x\n",vm->vm_start,vm->vm_end);
    vm++;
    printk("the first vmarea:start:%016x ,end:%016x\n",vm->vm_start,vm->vm_end);
    vm++;
    printk("the first vmarea:start:%016x ,end:%016x\n",vm->vm_start,vm->vm_end);
    vm++;
    printk("the first vmarea:start:%016x ,end:%016x\n",vm->vm_start,vm->vm_end);
    return 0;
 
}
 
static void __exit sysctl_example_exit(void)
{
    printk(KERN_INFO "sysctl unregister success.\n");
}
 
module_init(sysctl_example_init);
module_exit(sysctl_example_exit);
MODULE_LICENSE("GPL");
结果举例：dmesg
[41928.340021] the state of the task is:0
[41928.340023] the pid of the task is:12473
[41928.340024] the pid of current thread is:14907
[41928.340025] the mm_users of the mm_struct is:6
[41928.340026] the mm_count of the mm_struct is:1
[41928.340026]  total vm :1083 exec_vm:478mmap base:00000000007de000 ,task_size:00000000fffff000,start_code:0000000000400000,end_code:0000000000400af4,start_data:0000000000600e10, end_data:0000000000601054,start_brk:00000000014b2000,endbrk:00000000014d3000, start_stack:000000005a04e590
[41928.340029] the first vmarea:start:0000000000400000 ,end:0000000000401000
[41928.340030] the  vmarea:start:00000000005b2000 ,end:00000000005b4000
[41928.340031] the first vmarea:start:0000000000600000 ,end:0000000000601000
[41928.340033] the first vmarea:start:00000000005b4000 ,end:00000000005b9000
[41928.340034] the first vmarea:start:000000005a061000 ,end:000000005a063000
[41928.340035] the first vmarea:start:00000000007db000 ,end:00000000007dc000

```
C:内存映射原理：
+ 前言：实际上进程此时用的还是虚拟内存，那么怎么和物理页关联？实际上物理页有限，虚拟空间大，所以，只有最常用的部分才和物理页关联；大多程序也只占一小部分内存；
+ 进程经常要从文件中读取数据，文件存放于磁盘上，于是，将磁盘上的文件映射到内存则很关键，涉及文件读写和磁盘文件的更新；而实际的映射过程，只是用了几页来存储文件结尾的数据，而文件开始处，则内核只需在地址空间保存相关信息如：数据位置，数据如何读取即可(PS:这些不是保存在进程中的结构吗？ ),text段也类似，只有在需要时才加载；
+ 用户进程访问一个内存地址的过程：
虚拟地址-->页表-->确定物理页(有则读取)-->无，出发缺页异常发送到内核->内核检查进程并找到适当的后备存储器(和文件系统相关),分配物理页并填充来自后备存储器的数据-->借助页表将物理页并入用户进程地址空间，进程恢复执行；
-- >     malloc->do_brk-->写时复制，缺页异常--->到vm_area_struct找，vma_fault-->page--->没有做alloc_page ，这样的流程；

D:详细阐述：数据结构mm_struct：vmarea_struct mmap虚拟内存区域列表 ; rb_root mm_rb;  vm_area_struct *mmap_cache上一个find_vma结果(上次处理的区域）
树和链表：每个区域都通过一个vmarea_struct实例描述，进程的各区域(如text段等)按两种方式排序：在一个单链表上(开始于mm_struct->mmap)/在一个红黑树中，根位于mm_rb; (task_struct->mm)即一个进程只有一个mm;
{% asset_img vmareatree.png vmareatree%}
而其实多个mm会连接起来形成全局链表：
{% asset_img moremm.png moremm%}

虚拟内存区域 vmarea_struct详解：
优先查找树结构：一个文件往往关联了多个进程：为了建立一个文件中的一个区域与该区域映射到的所有虚拟地址空间的关联，使用了优先查找树；
1) struct file: 每个打开的文件(和每个块设备，因为也可以和内存映射)都会表示为struct file的一个实例，该结构包含了一个指向地址空间对象struct address_space的指针(和后备存储器相关),关联是优先查找树的基础；其文件区间与其映射到的地址空间之间的关联即通过优先树建立；stuct file 和struct address_space见源码
2) 另外，每个文件和块设备都表示为struct inode的一个实例，struct file是通过open调用时打开的文件抽象，而inode 则是文件系统自身中的对象；
struct inode: address_space *i_mapping; 注意每个file是特定于给定进程的，也就是每个进程都有其file成员；
{% asset_img mm_file.png moremm%}
E 对区域的操作：
```cpp
将虚拟地址关联到区域:通过虚拟地址，借助find_vma函数可以查找用户地址空间中结束地址在给定地址后的第一个区域；
struct vm_area_struct * find_vma(struct mm_struct * mm, unsigned long addr)
{
	struct vm_area_struct *vma = NULL;

	if (mm) {
		/* Check the cache first. */ //先检查缓存，也就是上次使用的内存；
		/* (Cache hit rate is typically around 35%.) */
		vma = mm->mmap_cache;
		if (!(vma && vma->vm_end > addr && vma->vm_start <= addr)) {//不行再查找区域红黑树；
			struct rb_node * rb_node;

			rb_node = mm->mm_rb.rb_node;
			vma = NULL;

			while (rb_node) {
				struct vm_area_struct * vma_tmp;

				vma_tmp = rb_entry(rb_node,
						struct vm_area_struct, vm_rb);

				if (vma_tmp->vm_end > addr) {
					vma = vma_tmp;
					if (vma_tmp->vm_start <= addr)
						break;
					rb_node = rb_node->rb_left;
				} else
					rb_node = rb_node->rb_right;
			}
			if (vma)
				mm->mmap_cache = vma;
		}
	}
	return vma;
}
```
区域合并：当新区域被加到进程的地址空间时，内核会检查它是否可以与一个或多个现存区域合并：无非检查地址是否相邻等等
插入区域： insert_vm_struct函数，实际工作：将区域插入红黑树；
创建区域：需要检查虚拟地址空间中是否还有足够的空闲空间来插入新区域；get_unmapped_area  

F:地址空间：文件和进程的内存映射：可以看是文件系统对于的地址空间和用户进程虚拟地址空间的关联映射；
vm_operations_struct结构：用于建立两个地址空间的关联和通信；
##### 用户进程内存分配的流程：
malloc--> chunks
  __brk:/__map-->
    sys_brk/sys_mmap_pgoff: 找vm_area_struct
       无：缺页异常 ，操作vm_area_struct:->ops->vm_fault->page
         --伙伴系统 alloc_page

{% asset_img malloc.png malloc %}

#### linux内存内核分配：
总的来说，有如下几种分配：
+ kmalloc/kzmalloc: 小内存的分配
+ kmem_cache_alloc: 特定内存大小的内存分配
+ vmalloc:不连续的大内存分配，多个页
+ alloc_page:底层分配，分配多个页；
##### slab机制

针对页内碎片的slab分配技术：将页拆分，分配小单元内存  
slab分配器：分配比4K更小的内存；提供要经常分配的缓存结构如struct fs_struct;slab着色；slab分配器命名由来
PS：slab分配器负责完成与伙伴系统的交互，来分配所需的页

备选分配器：slob分配器；slub分配器

slab分配的原理：
下图：缓存即高速缓存kmem_cache 结构，也叫slab缓存由两部分组成：保存管理性数据的缓存对象(kmem_cache)和保存被管理对象的各个slab
(1)每个缓存只负责一种对象类型(如struct unix_sock实例，通过/proc/slabinfo可看到),或提供一般性缓冲区(kmalloc分配时用);各个缓存中slab数目各有不同
(2)各个缓存都保持在一个双向链表中cache_chain,内核有机会遍历他们
{% asset_img slab.png slab %}


+ 详细缓存结构(kmem_cache)
下图：重要成员1：
         array_cache :保存了各个cpu最后释放的对象：在分配和释放这些对象时，采用后进先出原理，内核假定刚释放的对象仍然处于CPU高速缓存中，会尽快再次分配它
                            仅当per-CPU缓存为空时，才会用slab中的空闲对象重新填充它们
         由此形成三级的分配体系：按照分配成本和操作对CPU高速缓存和TLB的负面影响逐级升高：
         (1) 仍然处于CPU高速缓存中的per-CPU对象
         (2) 现存于slab中的未使用对象
         (3)刚使用伙伴系统分配的新slab中未使用的对象
        重要成员2：kmem_list3:每个内存结点都对应3个表头，用于组织slab链表；完全用尽的slab,部分空闲的slab和空闲的slab
{% asset_img slaball.png slaball %}
+ slab结构解释：
 (1)对象在slab中非连续排列，用于每个对象的长度非确切大小，舍入和对齐了；
 (2)slab创建时有两种方案对齐：1可使用标志SLAB_HWCACHE_ALIGN，slab用户可以按照要求按硬件缓存行对齐，cache_line_size返回值进行；2 不按照硬件对齐
 (3)填充字节情况；链接了第一个对象，所以slab首部可以不用和slab对象在一起
 (4)另外 page结构包含了一个链表元素，用于管理各种链表中的页，但是slab缓存不用，用于：   
 page->lru.next指向页驻留的缓存的管理结构  kmem_cache
 page->lru.prev指向保存该页的slab的管理结构 slab
 设置或读取slab信息分别由set_page_slab和get_page_slab函数完成，带有__cache后缀的函数则处理缓存信息的设置和读取：
mm/slab.c:2.6版本：
page_set_cache 
*page_get_cache
page_set_slab
*page_get_slab
此外，内核还对分配给slab分配器的各个物理页都设置标志PG_SLAB
{% asset_img slablist.png slablist %}

+ kmem_cache_系列实现：
```
(1)数据结构详情：主要是对slab中涉及的kmem_cache,slab,kmem_list3等中的成员做详细的介绍；略
(2)初始化slab系统：
    一个问题：为初始化slab数据结构，内核需要若干远小于一页的内存块（slab中的object要分配内存，不然为空指针), 这些由kmalloc分配，但是kmaloc需要在slab启动后才能用；
   解决：涉及kmalloc中的per-CPU缓存的初始化：需要一些技巧：
       A: kmem_cache_init初始化slab分配器：在伙伴系统启动后调用：
           创建系统中第一个slab缓存，以便为kmem_cache的实例提供内存，内核此时使用的主要是在编译时创建的静态数据，一个静态数据结构(initarray_cache)用作per-CPU数组，缓存名为cache_cache;
           接着，初始化一般性缓存，用于kmalloc内存来源：调用kmem_cache_create，存在cache_cache需要借助kmalloc，但是kmalloc又需要借助它的问题：
                     内核使用g_cpucache_up变量解决，类似状态机；在合适的时候才用更大的缓存
           最后，将kmalloc动态分配的版本替换为slab对象的数据结构
(3)创建缓存：创建新的slab缓存必须调用kmem_cache_create；调用后可读的name会出现在/proc/slabinfo中；
(4)分配对象：kmem_cache_alloc:用于从特定的高速缓存kmem_cache获取对象，类似于所有的malloc函数；
(5)释放对象：kmem_cache_free
(6)销毁缓存：kmem_cache_destroy
```
+ 通用缓存:kamlloc:
```cpp
通用缓存：若不涉及特殊对象如struct unix_sock,而是传统的分配/释放内存，则必须调用kmalloc和kfree函数，这两个函数相当于c的malloc和free
          需要注意的是：这里的内存分配采用的所有可用的长度：2的幂次：
struct cache_sizes malloc_sizes[] = {
#define CACHE(x) {.cs_size = (x) },
#include 
/*
#if (PAGE_SIZE == 4096)
	CACHE(32)
#endif
	CACHE(64)
#if L1_CACHE_BYTES < 64	// L1_CACHE_BYTES = 128
	CACHE(96)
#endif
	CACHE(128)
#if L1_CACHE_BYTES < 128
	CACHE(192)
#endif
	CACHE(256)
	CACHE(512)
	CACHE(1024)
	CACHE(2048)
	CACHE(4096)
	CACHE(8192)
	CACHE(16384)
	CACHE(32768)
	CACHE(65536)
	CACHE(131072)
#if KMALLOC_MAX_SIZE >= 262144
	CACHE(262144)
#endif
#if KMALLOC_MAX_SIZE >= 524288
	CACHE(524288)
#endif
#if KMALLOC_MAX_SIZE >= 1048576
	CACHE(1048576)
#endif
#if KMALLOC_MAX_SIZE >= 2097152
	CACHE(2097152)
#endif
#if KMALLOC_MAX_SIZE >= 4194304
	CACHE(4194304)
#endif
#if KMALLOC_MAX_SIZE >= 8388608
	CACHE(8388608)
#endif
#if KMALLOC_MAX_SIZE >= 16777216
	CACHE(16777216)
#endif
#if KMALLOC_MAX_SIZE >= 33554432
	CACHE(33554432)
#endif
*/
	CACHE(ULONG_MAX);
#undef CACHE
}
```
##### linux内存内核分配的选择：
+ 概述：
分配小的chunks 使用 kmalloc或kmem_cache_alloc 家族函数
分配大的虚拟的连续区域，使用vmalloc和它的变种函数，或者你可以直接请求页，从页的分配器 with alloc_pages;分配器也可能使用的是更专业的比如 cma_alloc或 zs_malloc

许多内存分配器都使用GFP flags来表达内存应该怎么被分配。 这个GFP标志是给到get free pages的，是底层的分配函数
分配api的多样性和众多的GFP标志使得“我应该如何分配内存?”这并不容易回答，尽管你很可能会用
kzalloc(<size>, GFP_KERNEL);

+ 关于Get Free Page FLags:
GFP(get free page)标志，控制分配器的行为，他们告知什么内存zones被使用，分配器应该如何努力寻找空闲内存，内存是否可以被用户空间访问等
内存管理api提供了GFP标志及其组合的参考文档，这里我们简要概述了它们的推荐用法:
1) 大多数情况下，GFP_KERNEL正是您所需要的。用于内核数据结构的内存、DMAable内存、索引节点缓存，所有这些以及许多其他分配类型都可以使用GFP_KERNEL。注意，使用GFP_KERNEL意味着GFP_RECLAIM，这意味着直接回收可能会在内存压力下触发;必须允许调用上下文休眠。

2)如果分配从一个原子上下文执行，例如中断处理程序，使用GFP_NOWAIT。这个标志防止直接回收IO或文件系统操作。因此，在内存压力下，GFP_NOWAIT分配很可能失败。有合理回退的分配应该使用GFP_NOWARN。

3)如果你认为你访问的内存储备是合理的，且除非内核分配成功，否则会给内核带来压力的话，可以用GFP_ATOMIC;

4)从用户空间触发的不受信任的分配，应该被当成一个分配统计对象，并且有__GFP_ACCOUNT这个标志被设置；即GFP_KERNEL_ACCOUNT;

5)用户空间分配应该使用GFP_USER、GFP_HIGHUSER或 GFP_HIGHUSER_MOVABLE  ;标志名越长，限制越小；
GFP_HIGHUSER_MOVABLE不要求内核可以直接访问已分配的内存，这意味着数据是可移动的。
GFP_HIGHUSER表示所分配的内存是不可移动的，但是内核不需要直接访问它。一个例子可能是硬件分配，它将数据直接映射到用户空间，但没有寻址限制。
GFP_USER表示所分配的内存是不可移动的，必须由内核直接访问。

6)您可能注意到，现有代码中有相当多的分配指定了GFP_NOIO或GFP_NOFS。历史上，它们被用来防止由于直接内存回收调用FS或IO路径和阻塞已经占用的资源而引起的递归死锁。从4.12开始，解决这个问题的首选方法是使用FS/IO上下文中使用的GFP掩码中描述的新的作用域api。
7)其他遗留的GFP标志是GFP_DMA和GFP_DMA32。它们用于确保具有有限寻址能力的硬件可以访问所分配的内存。所以，除非你正在为一个有这样限制的设备编写驱动程序，否则不要使用这些标志。即使是有限制的硬件，使用dma_alloc* api也是可取的。

+ GFP flags和它的回收内存行为，即多个和一起：
1)GFP_KERNEL & ~__GFP_RECLAIM -乐观分配，根本没有尝试释放内存。最轻的重量模式，甚至不触发后台回收。应该谨慎使用，因为它可能会耗尽内存，下一个用户可能会更主动地回收内存。
2)GFP_KERNEL & ~__GFP_DIRECT_RECLAIM(或GFP_NOWAIT)-乐观分配，不尝试从当前上下文释放内存，但可以唤醒kswapd回收内存，如果该区域低于低水位。可以在原子上下文中使用，也可以在请求进行性能优化时使用，对于慢路径有另一种退路。
3)(GFP_KERNEL|__GFP_HIGH) & ~__GFP_DIRECT_RECLAIM(又名GFP_ATOMIC) -非睡眠分配，具有昂贵的回退，因此它可以访问部分内存储备。通常从中断/下半部分上下文使用一个昂贵的慢路径回退。

4)GFP_KERNEL -允许后台和直接回收，并使用默认的页面分配器行为。这意味着不昂贵的分配请求基本上是不会失败的，但没有这种行为的保证，所以失败必须由调用者正确地检查(例如，OOM killer 异常是
允许失败的）
5)GFP_KERNEL | __GFP_NORETRY -覆盖默认的分配器行为，所有的分配请求在早期失败，而不是导致中断回收(在这个实现中是一轮回收)。OOM杀手没有被调用。
6)GFP_KERNEL | __GFP_RETRY_MAYFAIL -覆盖默认的分配器行为和所有的分配请求尝试真正努力。如果回收不能取得任何进展，请求将失败。OOM杀手不会被触发。

7)GFP_KERNEL | __GFP_NOFAIL -覆盖默认的分配器行为，所有的分配请求将不断循环，直到它们成功。这可能真的很危险，特别是对大订单来说。



+ 如何选择正确的内存分配器：
 1 最直接的方式就是用 kmalloc，当然，安全的是初始化内存为0，如kzalloc(),若想分配给一个数组，则有kmalloc_array(),和kcalloc() ;辅助函数：struct_size(),array_size(),array3_size();用来防止溢出；
不过，用kmalloc分配的chunk的最大长度是被限制的，实际限制依赖于硬件和内核配置，但是好的实践是，分配的对象比页的大小小；用kmalloc分配的块的地址至少对齐到ARCH_KMALLOC_MINALIGN字节
对于2的幂次大小，对齐也保证至少是各自的大小。用kmalloc()分配的块可以用krealloc()调整大小。与kmalloc_array()类似:以krealloc_array()的形式提供了调整数组大小的帮助器。

2 对于大的分配来说，可以用vmalloc和vzalloc，或直接用页分配器；用vmalloc和相关联的函数分配可能不是物理上连续的内存；
如果您不确定分配大小对于kmalloc来说是否太大，则可以使用kvmalloc()及其衍生物。它将尝试使用kmalloc分配内存，如果分配失败，它将使用vmalloc重试。对于哪些GFP标志可以用于kvmalloc有一些限制;请参阅kvmalloc_node()参考文档。注意，kvmalloc可能返回的内存不是物理上连续的。

3 如果你需要分配许多相同的对象，你可以使用slab缓存分配器。在使用缓存之前，应该使用kmem_cache_create()或kmem_cache_create_usercopy()来设置缓存。如果缓存的一部分可能被复制到用户空间，则应该使用第二个函数。创建缓存之后，kmem_cache_alloc()及其方便的包装器可以从缓存中分配内存。
当分配的内存不再需要时，必须释放它。可以对kmalloc、vmalloc和kvmalloc分配的内存使用kvfree()。slab缓存应该使用kmem_cache_free()来释放。不要忘记使用kmem_cache_destroy()来销毁缓存。
##### kmalloc和kmem_cache_系列的区别：
来源于网上比较好的回答：
Here is the brief description about how kernel manages memory.
In order to manage small sized physical memory allocation, kernel uses slab
allocator. Slab allocator maintains two types of caches

1. Generalized Caches of memory pools
2. Specialized caches of memory pools.

Generalized caches contains small memory objects of sizes 8, 16, 32,
64,....512, 1024, 2048, 4096, 8192 bytes. These are named as kmalloc'ed
caches because kernel allocates memory from these caches when kmalloc is
used to to allocate memory.
These caches are created at the boot initialization phase. Ref
/proc/slabinfo for the list of kmalloc-ed caches.

Not always the user of the kernel will use the objects of the size
maintained by the generalized cache. For Eg. In case of Ethernet driver, the
driver has to allocate memory for the size 1500 bytes.
In this case, if the driver has to allocate memory using kmalloc, it has to
allocate a minimum of 2048 bytes per packet, thus wasting almost 550 bytes
for every allocation of memory per packet.
Hence kernel allows for the driver to create a specialized cache which
contains the memory objects of the size specified by user.
i.e. In the above example the driver can create his own cache of memory
objects of size 1500 bytes by using following KPIs

1. kmem_cache_create - Creates a specialized cache
2. kmem_cache_alloc - allocates memory from a specialized cache
3. kmem_cache_free - frees memory to the speicialized cache
4. kmem_cache_destroy - destroys the specialized cache.

Hence kmem_cache_alloc is used when the user of the kernel needs to allocate
memory from the specialized cache and kmalloc is used to allocate memory
from the generalized caches.

不过看实现，貌似后来的版本，这两个函数都基于slab/slub/slob:具体用哪个分配器，需要根据内核的默认配置； 只是前者kmalloc是可能两个模块共用一个cache，而后者则不是；
关于kmalloc和kmem_cache_alloc的区别：
kmalloc: It uses the generic slab caches available to any kernel code. so your module will share slab cache with other components in kernel.

kmem_cache_alloc: It will allocate objects from a dedicated slab cache created by kmem_cache_create. If you specifically want a better slab cache management dedicated to your module only, that too for a specific type of objects, use kmem_cache_create followed by kmem_cache_alloc. USB/SCSI drivers use this. kmem_cache_create takes sizeof your object you want to create slab of, a name which appears in /proc/slabinfo and flags to govern behavior of your slab cache.

### linux下内存的查看和几个问题：
#### linux下物理内存统计等
ref:http://linuxperf.com/?cat=7
系统当前的内存情况：
物理内存总数：和可用的物理内存数：
 dmesg | grep Memory
[    0.000000] Memory: 3857076K/4193848K available (8375K kernel code, 1336K rwdata, 3944K rodata, 1492K init, 1260K bss, 336772K reserved, 0K cma-reserved)
4193848K 表示此系统物理内存大小
3857076K 表示在初始化时，可供kernel分配的free memory的大小，注意这个值在初始化后，实际可用会变大，因为之后还会释放一些bootmem等用完的内存；
后面括号的是内核的代码大小占用等等；
所以：一个物理内存划分：BIOS|kernel code|initdata| totalavail
系统启动后，totalavail为：注意是物理内存
$ free  执行结果的total
             total       used       free     shared    buffers     cached
Mem:       4046636    2568888    1477748      17884      70888    1764708
-/+ buffers/cache:     733292    3313344
Swap:      4191228          0    4191228
或：下面指令的memtotal
$ cat /proc/meminfo
MemTotal:        4046636 kB
MemFree:         1477676 kB
MemAvailable:    3113840 kB
Buffers:           70896 kB
Cached:          1764708 kB
SwapCached:            0 kB
Active:           827164 kB
Inactive:        1596816 kB
Active(anon):     589152 kB
Inactive(anon):    17104 kB
Active(file):     238012 kB
Inactive(file):  1579712 kB
Unevictable:          16 kB
Mlocked:              16 kB
SwapTotal:       4191228 kB
SwapFree:        4191228 kB
Dirty:                 0 kB
Writeback:             0 kB
AnonPages:        588396 kB
Mapped:           163956 kB
Shmem:             17884 kB
Slab:              61964 kB
SReclaimable:      42112 kB
SUnreclaim:        19852 kB
KernelStack:        5888 kB
PageTables:        24916 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     6214544 kB
Committed_AS:    2746044 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:    407552 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:       96192 kB
DirectMap2M:     4098048 kB

更多内核内存的查看，见ref
进程的物理内存使用：
cat /proc/{pid}/status
VmRSS 或 VmHWM
这个怎么来的：追了下，它定义在：task_mem in task_mmu.c (fs\proc) : 		"VmRSS:\t%8lu kB\n"
 get_mm_rss(mm);其实就是task_struct->&mm->rss_stat.count[member]
 看下这个结构的解释：（看了下代码，就是在缺页等实际分配的时候做累加）
 struct mm_rss_stat rss_stat - A set of statistics contained in struct mm_rss_stat relating to Resident Set Size (RSS), i.e. memory that has been faulted in. The structure is simply an array of atomic_long_t counts for each of: MM_FILEPAGES - number of resident pages mapping files, MM_ANONPAGES - number of resident anonymous pages, MM_SWAPENTS - number of resident swap entries, and MM_SHMEMPAGES - number of resident shared pages. Note that in the usual case where the SPLIT_RSS_COUNTING constant is set, these statistics are only updated every TASK_RSS_EVENTS_THRESH page faults (hardcoded to 64.)

进程的虚拟内存大小：
```cpp
$ sudo cat /proc/12473/status
Name:	task_test2
State:	R (running)
Tgid:	12473
Ngid:	0
Pid:	12473
PPid:	10096
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
FDSize:	256
Groups:	4 24 27 30 46 108 124 1000 
NStgid:	12473
NSpid:	12473
NSpgid:	12473
NSsid:	10096
VmPeak:	    4332 kB
VmSize:	    4332 kB  是虚拟内存的大小，即task_struct->mm->total_vm个页，一个页是4k;
VmLck:	       0 kB
VmPin:	       0 kB
VmHWM:	     660 kB
VmRSS:	     660 kB 
VmData:	     200 kB
VmStk:	     132 kB
VmExe:	       4 kB
VmLib:	    1908 kB
VmPTE:	      32 kB
VmPMD:	      12 kB
VmSwap:	       0 kB
HugetlbPages:	       0 kB
Threads:	1
SigQ:	0/15066
SigPnd:	0000000000000000
ShdPnd:	0000000000000000
SigBlk:	0000000000000000
SigIgn:	0000000000000000
SigCgt:	0000000000000000
CapInh:	0000000000000000
CapPrm:	0000000000000000
CapEff:	0000000000000000
CapBnd:	0000003fffffffff
CapAmb:	0000000000000000
Seccomp:	0
Speculation_Store_Bypass:	vulnerable
Cpus_allowed:	3
Cpus_allowed_list:	0-1
Mems_allowed:	00000000,00000001
Mems_allowed_list:	0
voluntary_ctxt_switches:	0
nonvoluntary_ctxt_switches:	167165
```
#### 几个问题：
+ 通过malloc分配的最小物理内存单位是页吗，就是最小的话，一定会分配一个页，那这个页是不是这个进程独占，还是可以和其他进程共享？
+ 内存映射和普通的内存分配操作有什么区别？
+ 通过alloc_page分配内存页的时候，是不是就是即分配物理页，而不会再通过写入才分配？
是，即分配物理页，可以通过模块中alloc_page接口的调用，观察free指令下内存的变化；模块例子可以在本文中搜 alloc_page,在insmod模块后观察free下的值，最好指定分配的页多些，因为free下的空闲内存在一个范围变动；

#### 参考：
深入linux内核架构等

