---
title: procsysfs
date: 2021-02-26 23:52:48
tags: linux_procsysfs
categories: linux
---

### linux 用户空间与内核的接口---内核信息的输出和修改
+ linux用户与内核的接口有多种,配置内核的netlink,ioctl,等特殊系统调用，其他系统调用;用户空间指令:ifconfig,route,
iptable等;以及内核输出信息的procfs,sysfs和修改内核参数的sysctl接口;这里介绍最后一种;<!-- more -->

+ 用户空间程序常常需要知道内核一些当前状态信息，且有时为了系统的稳定性和性能等，需要改变一些内核参数信息；
linux提供了这样的机制，用来读取内核输出信息的内存文件系统和修改的系统调用；
除了常规的系统调用外，还提供了sysctl系统调用和（/proc(procfs虚拟文件系统), /sys(sysfs虚拟文件系统）

### procfs介绍
+ 介绍：  
    procfs:是挂载在/proc的虚拟文件系统，允许内核以文件的形式向用户输出内部信息；这些信息存于内存中，可以通过cat或more及>重定向输出输入  
	它不能被编译为一个模块，配置菜单中相关内核选项为：“Filesystems->pseudo filesystems->/proc file system support"  

    procfs目前分为/proc/sys和/proc/其他的，前者可以通过sysctl系统调用来进行写，从而改变内核的一些参数和配置，后者只读；  

    如果是只读的，且是涉及更为复杂的数据结构而且需要特殊格式等如缓存和统计数据,则考虑用proc/非sys的，否则若只是简单变量，则应该使用/proc/sys  
+ 实践：
   1）用户空间使用： 用户可以通过cat /proc/...   ls /proc等方式来进行读取/procfs的值；对于/proc/sys的，有单独的说明见下；  
   2)  解释：关于/proc/下的各个文件目录的含义，可以查看man https://man7.org/linux/man-pages/man5/procfs.5.html  对网络代码注册的文件，一般位于/proc/net目录下，在该目录下存在不少文件，有一种比较特殊的文件，如tcp,udp有固定的格式，就像数据库的记录一样，称为综合文件(synthetic files).所以说proc文件系统适用于它，提供了对此类文件框架的支持：

```cpp
	cat  /proc/net/udp:
    sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode ref pointer drops             
    6591: 00000000:0044 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 14983 2 0000000000000000 0         
    6646: 26B4450A:007B 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 28063 2 0000000000000000 0         
    6646: 0100007F:007B 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 28061 2 0000000000000000 0         
    6646: 00000000:007B 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 28057 2 0000000000000000 0         
    6660: FFFF12AC:0089 00000000:0000 07 00000000:00000000 00:00000000 00000000     0        0 72326708 2 0000000000000000 0 
```

3) 内核中如何添加/proc/目录等信息：  
eg: 大多数网络功能在初始化时，都会在/proc/net中注册一个或多个文件，无论初始化动作发生在系统启动时还是模块加载时。当用户读取某个文件时，内核会调用一组内核函数来输出相应信息：
代码中如何创建：
     proc_mkdir:创建/proc中的目录
     create_proc_entry/remove_proc_entry:创建和除名文件
     可以用包裹函数如：
     proc_net_fops_create/proc_net_remove:/proc/net/中文件的注册和除名
     现在的版本和之前的版本不同，一些接口比如底层的create_proc_entry已经没有了，所以现在一些书本的例子不能用，需要参考内核代码，等看到时，再总结出例子来更新文章  

### sysctl和 /proc/sys
#### 介绍
+ sysctl:这个接口允许用户空间读取或修改内核变量的值；不能用此接口对每个内核变量进行操作，内核应明确指出哪些变量从此接口是可见的；
  从用户空间，你可以用两种方式访问sysctl输出的变量：一种是系统调用sysctl(man sysctl),一种则是依赖procfs(因为内核会在/proc中添加一个特殊目录/proc/sys,为每个sysctl所输出的内核变量引入一个文件
+ 内核配置选项：General setup->sysctl support
+ sysctl的信息大多可写，但是只有超级用户可写，一个简单内核变量或数据结构相关的一些文件

#### 命令行工具sysctl和运维
有时候需要查内核相关参数，并做调整来改变内核行为，以支持更多功能或者恢复手段等；这个时候可以使用sysctl指令：  
+ 展示：
```
$ sysctl -a 展示所有的sysctl参数
$ sysctl vm.swapiness 展示这个的值，会打印：
vm.swappiness = 60
sysctl命令实际上就是读取/proc/sys目录下的内容；它是一个虚拟目录，只包含当前内核参数值
sysctl vm.swapiness 和 cat /proc/sys/vm/swapiness 是一样的效果
```


+ 修改：
``` 
$ sysctl -w paramter=value  临时改变参数值，重启会恢复默认值
eg： sysctl -w net.ipv4.ip_forward=1
如果值包括空格或特殊字符，请用双引号括起来；
如果想要重启后也用这个值，可以：
$sysctl -w net.ipv4.ip_forward=1 >> /etc/sysctl.conf
其他临时写的方式：
$ echo 1 > /proc/sys/net/ipv4/ip_forward
```

+ 加载值：
```
$ sysctl -p /etc/sysctl.d/file_name.conf
当没有指定文件名：使用/etc/sysctl.conf文件；
```

#### 用户程序如何系统调用sysctl:
```cpp
https://man7.org/linux/man-pages/man2/sysctl.2.html
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscall.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <linux/sysctl.h>

int _sysctl(struct __sysctl_args *args );

#define OSNAMESZ 100

int
main(void)
{
    struct __sysctl_args args;
    char osname[OSNAMESZ];
    size_t osnamelth;
    int name[] = { CTL_KERN, KERN_OSTYPE };

   memset(&args, 0, sizeof(struct __sysctl_args));
    args.name = name;
    args.nlen = sizeof(name)/sizeof(name[0]);
    args.oldval = osname;
    args.oldlenp = &osnamelth;

   osnamelth = sizeof(osname);

   if (syscall(SYS__sysctl, &args) == -1) {
        perror("_sysctl");
        exit(EXIT_FAILURE);
    }
    printf("This machine is running %*s\n", osnamelth, osname);
    exit(EXIT_SUCCESS);
}
```

#### 内核中如何添加/proc/sys下的目录和文件
（代码中如何呈现：
   用户在/proc/sys下看到的一个文件，实际上是一个内核变量；就每个变量而言，内核可以定义：
   A:要将其放在/proc/sys的何处，与相同内核组件或功能相关联的变量，，通常位于一个目录中，如/proc/sys/net/ipv4
   B:命名：一般文件名和相关联的变量名相同
   C:权限：一般所有可读，超级可写
）
+ 简单例子1：
```cpp

 
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>
#include <linux/sysctl.h>
 
static int sysctl_hello_data = 1024;
 
static int hello_callback(struct ctl_table *table, int write,
        void __user *buffer, size_t *lenp, loff_t *ppos)
{
    int rc;
    int *data = table->data;
 
    printk(KERN_INFO "original value = %d\n", *data);
 
    rc = proc_dointvec(table, write, buffer, lenp, ppos);
    if (write)
        printk(KERN_INFO "this is write operation, current value = %d\n", *
data);
 
    return rc;
}
 
static struct ctl_table hello_ctl_table[] = {
    {
        .procname       = "helloctl",
        .data           = &sysctl_hello_data,
        .maxlen         = sizeof(int),
        .mode           = 0644,
        .proc_handler   = hello_callback,
    },
    {
        /* sentinel */ //哨兵的作用，见数据结构
    },
};
 
static struct ctl_table_header *sysctl_header;
 
static int __init sysctl_example_init(void)
{
    sysctl_header = register_sysctl_table(hello_ctl_table);
    if (sysctl_header == NULL) {
        printk(KERN_INFO "ERR: register_sysctl_table!");
        return -1;
    }
 
    printk(KERN_INFO "sysctl register success.\n");
    return 0;
 
}
 
static void __exit sysctl_example_exit(void)
{
    unregister_sysctl_table(sysctl_header);
    printk(KERN_INFO "sysctl unregister success.\n");
}
 
module_init(sysctl_example_init);
module_exit(sysctl_example_exit);
MODULE_LICENSE("GPL");
```

+ 复杂例子：
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
 
 
static int min_virdev_frags = 3;
static int max_virdev_frags = 10;
static int virdev_sum1=1;
static int virdev_array[4];
static int sysctl_max_virdev_frags=4;

 
static int set_default_something(struct ctl_table *table, int write,
        void __user *buffer, size_t *lenp, loff_t *ppos)
{
      printk(KERN_INFO "set default something value =\n");
      //you can do other ，可以自己实现类似proc_dointvec功能的函数，其实就是赋值；参考sysctl_net_core.c
      return 0;
}

static struct ctl_table virdev_table1[] = {
    { 
      .procname     = "virdev_sum1",
      .data         = &virdev_sum1,
      .maxlen       = sizeof(virdev_sum1),
      .mode         = 0644,
      .proc_handler = &proc_dointvec 
    },
    {
		.procname	= "max_virdev_frags",
		.data		= &sysctl_max_virdev_frags,
		.maxlen		= sizeof(int),
		.mode		= 0644,
		.proc_handler	= proc_dointvec_minmax, //若设置的值不在extra1-extra2之间，则设置失败
		.extra1		= &min_virdev_frags,
		.extra2		= &max_virdev_frags,
	},
    //只是回调来设置到某个值中或者做其他事，不需要data,但回调函数也必须符合格式
    {
		.procname	= "default_something",
		.mode		= 0644,
		.maxlen		= 16,
		.proc_handler	= set_default_something
	},    
    { }
};

static struct ctl_table virdev_table2[] = {
    { 
      .procname     = "virdev_array",
      .data         = &virdev_array,
      .maxlen       = sizeof(int), //是指data的大小，而data其实是void *,不管怎样其实就是指针大小
      .mode         = 0644,
      .proc_handler = &proc_dostring   //会将接收到的用户传入的buff数据，解析并，写到data中，具体见实际例子；
    },
    { }
};

static struct ctl_table virdev_dir_table[] = {
    {
        .procname = "virsub1",
        .mode     = 0555,
        .child    = virdev_table1
    },
    {
        .procname = "virsub2",
        .mode     = 0555,
        .child    = virdev_table2
    },
    { }
};


static struct ctl_table virdev_root_table[] = {
    {
        .procname       = "virdev",
        .mode           = 0555,
        .child          = virdev_dir_table
    },
    {
        /* sentinel */
    }, //多出的一个应该是哨兵的作用，参考数据结构设计
};
 
static struct ctl_table_header *sysctl_header;
 
static int __init sysctl_example_init(void)
{
    sysctl_header = register_sysctl_table(virdev_root_table);
    if (sysctl_header == NULL) {
        printk(KERN_INFO "ERR: register_sysctl_table!");
        return -1;
    }
 
    printk(KERN_INFO "sysctl register success.\n");
    return 0;
 
}
 
static void __exit sysctl_example_exit(void)
{
    unregister_sysctl_table(sysctl_header);
    printk(KERN_INFO "sysctl unregister success.\n");
}
 
module_init(sysctl_example_init);
module_exit(sysctl_example_exit);
MODULE_LICENSE("GPL");

```

+ 接口解释：
1) 结构体ctl_table
每一个sysctl条目对应一个 struct ctl_table 结构，在该结构体定义在文件/include/linux/sysctl.h中，定义及解释如下：
```cpp

struct ctl_table
{
    const char *procname; /* Text ID for /proc/sys, or zero */ /* 表示在proc/sys/下显示的文件名称 */
    void *data; /* 表示对应于内核中的变量名称    */ //是一个指针，如上，可以是int,数组等
    int maxlen;//  /* 表示条目允许的最大长度         */
    mode_t mode;/* 条目在proc文件系统下的访问权限 *///见 stat.h
    struct ctl_table *child; //子目录的结构，见上面的例子，但内核代码注释不建议这样用，甚至弃用；
    proc_handler *proc_handler; /* Callback for text         formatting *///当sysctl或echo等方式写时，触发这个回调函数将值写到注册的ctl_table中
    void *extra1;//proc_dointvec_minmax等有范围的限定用，下限
    void *extra2;//上限
}
maxlen: 它主要用于字符串内核变量，以便在对该条目设置时，对超过该最大长度的字符串截掉后面超长的部分.
```
2) 注册和卸载：
注册register_sysctl_table
注册sysctl条目使用函数register_sysctl_table，函数原型如下：
struct ctl_table_header *register_sysctl_table(struct ctl_table *table)
第一个参数为定义的struct ctl_table结构的sysctl条目或条目数组指针；
  
卸载unregister_sysctl_table
当模块卸载时，需要使用函数unregister_sysctl_table，其原型：
void unregister_sysctl_table(struct ctl_table_header * header)
其中struct ctl_table_header是通过函数register_sysctl_table
注册时返回的结构体指针。

3) 关于信息和参考，如linux版本更新后，接口变化：
内核源码例子：/proc/sys/net/core的添加
socket.c :sock_init
net/core/sysctl_net_core.c
源码：
proc_dointvec_minmax系列函数定义：kernel/sysctl.c sysctl.h
也可以参考：其中的说明
http://www.embeddedlinux.org.cn/linux_net/0596002556/understandlni-CHP-3-SECT-2.html

### sysfs
 sysfs:是挂载在/sys下的虚拟文件系统：它不仅可以把设备和驱动程序的信息从内核空间导到用户空间，也可以对设备和驱动进行配置；
目的是将一些原本在procfs中的设备独立出来，以设备树的形式呈现给用户；最初，sysfs名driverfs; 而后来对其他子系统也有用所以就更名；
sysfs在用户空间使用和procfs类似，对内核中的实现和使用，和设备模型相关，等总结完设备模型后，再总结；

 

