---
title: filesystem
date: 2021-03-13 07:31:21
tags: filesystem
categories: linux
---

## 文件系统：

### 关于信息记载和存储的历史，现状和未来展望
+ 本能信息存储：在原始时期，信息被基因记载，并向下传递，个体发生基因突变后，存活下来的个体，能将突变的基因信息保存下来，传递下来，从而进化；
+ 口口相传：人类发明语言或者是类似的东西，然后将经验和信息一代代传递下来，通过人本身作为记录和存储；
+ 文字的出现：直到文字和记载载体出现，比如帛书，纸等，文字以语句，文章，数字等被记录成竹简或书本，保存下来；
+ 磁带记录：磁带的出现，提高了信息记载的便利性；<!-- more -->
+ 硬盘存储：硬盘的出现使得载体变多，而记录的形式也增强了，有文件系统，数据库等存储不同类型的媒介，而媒介也多种多样：文字，图片，视频等等；
+ 现在出现了很多类型的文件系统，存储也不再是单机，有云，网络，使得存储的性能和效率得到进一步的提升；
+ 未来：一方面随着信息的爆炸，往快，久，量大的方向继续进化；另一方面，针对不同的媒介有不同的存储支持；畅想，未来，存储可能会
变成像手机或其他智能硬件一样，成为人类通过外部工具进行进化的一种手段，而人类，可能也不再受限于记忆；
+ PS: 人类的意识存在，前世今生，和记忆存储关联密切，对失忆的人来讲，就像重新活过一样；

### 现代信息记载和存储分类：数据库和文件系统
现代信息记载，效率高的主要是数据库和文件系统，前者提高了数据存储的效率，以及支持数据以某个形式方式组织，更有利于有效存取；
而后者，是比较原始的，支持的信息媒介更多，也支持各种系统，限制比较小，但是不利于数据的查阅；

#### 文件系统的分类：
+ 单机文件系统
+ 网络文件系统
+ 分布式文件系统
+ 并行文件系统等等；

### 文件系统的关注点：性能数据，IO，进程并发等；
+ IO:
+ 磁盘分配单位
+ 碎片处理机制
+ 带宽利用率
+ 对多进程并发读写和交互式访问的性能，以及聚合读写带宽的能力；
+ 文件的组织形式和访问方式：比如硬盘如何分区，文件如何组织等等，和存储硬件本身限制的关联，会影响硬件操作性能；  
比如说  
候选的磁盘分区方案：  
    方案一： 255个盘面，C盘是0-100盘面， D盘是101-200个盘面,……  
    方案二：3263个柱面，C盘0-1000个柱面，D盘1001-20001个柱面,……


### 硬盘的组成和读写机制
####  硬盘简介
1、硬盘的物理结构一般由磁头与碟片、电动机、主控芯片与排线等部件组成；当主电动机带动碟片旋转时，副电动机带动一组（磁头）到相对应的碟片上并确定读取正面还是反面的碟面，磁头悬浮在碟面上画出一个与碟片同心的圆形轨道（磁轨或称柱面），这时由磁头的磁感线圈感应碟面上的磁性与使用硬盘厂商指定的读取时间或数据间隔定位扇区，从而得到该扇区的数据内容。所有的盘片都固定在一个旋转轴上，这个轴即盘片主轴。而所有盘片之间是绝对平行的，在每个盘片的存储面上都有一个磁头，磁头与盘片之间的距离比头发 丝的直径还小。所有的磁头连在一个磁头控制器上，由磁头控制器负责各个磁头的运动。磁头可沿盘片的半径方向动作，而盘片以每分钟数千转到上万转的速度在高 速旋转，这样磁头就能对盘片上的指定位置进行数据的读写操作。

{% asset_img disk1.png disk1 about %}

#### 磁道（Track）
当磁盘旋转时，磁头若保持在一个位置上，则每个磁头都会在磁盘表面划出一个圆形轨迹，这些圆形轨迹就叫做磁道（Track）。信息以脉冲串的形式记录在这些轨迹中，这些同心圆不是连续记录数据，而是被划分成一段段的圆弧（扇区），这些圆弧 的角速度一样。

#### 柱面 （Cylinder）
在有多个盘片构成的盘组中，由不同盘片的面，但处于同一半径圆的多个磁道组成的一个圆柱面（Cylinder）。所有盘面上的同一磁道构成一个圆柱，通常称做柱面（Cylinder），每个圆柱上的磁头由上而下从“0”开始编号。数据的读/写按柱面进行，即磁 头读/写数据时首先在同一柱面内从“0”磁头开始进行操作，依次向下在同一柱面的不同盘面即磁头上进行操作，只在同一柱面所有的磁头全部读/写完毕后磁头 才转移到下一柱面，因为选取磁头只需通过电子切换即可，而选取柱面则必须通过机械切换。电子切换相当快，比在机械上磁头向邻近磁道移动快得多，所以，数据 的读/写按柱面进行，而不按盘面进行。也就是说，一个磁道写满数据后，就在同一柱面的下一个盘面来写，一个柱面写满后，才移到下一个磁道开始写数据。读数 据也按照这种方式进行，这样就提高了硬盘的读/写效率。

#### 扇区（Sector）
磁盘上的每个磁道被等分为若干个弧段，这些弧段便是硬盘的扇区（Sector）。**硬盘的第一个扇区，叫做引导扇区**。操作系统以扇区（Sector）形式将信息存储在硬盘上，每个扇区包括512个字节的数据和一些其他信息。注意扇区不是指的扇形那个区域，而是磁道的一段弧线；由下计算存储空间大小公式可知；

#### 磁头（Head）
在硬盘系 统中，硬盘的每一个盘片都有两个盘面（Side），即上、下盘面，一般每个盘面都会利 用，都可以存储数据。盘面号又叫磁头号，因为每一个有效盘面都有一个对应的读写磁头。
{% asset_img disk2.png disk2 about %}

#### 如何计算存储大小：
硬盘的大小是使用"磁头数 x 柱面数 x 扇区数 x 每个扇区的大小"这样的公式来计算的。其中，磁头数（Heads）表示硬盘共有几个磁头，也可以理解为硬盘有几个盘面，然后乘以 2；柱面数（Cylinders）表示硬盘每面盘片有几条磁道；扇区数（Sectors）表示每条磁道上有几个扇区；每个扇区的大小一般是 512Byte。
举个例子：在ubuntu上

```
root@ksance-ThinkPad-X201:~# fdisk -l

Disk /dev/sda: 320.1 GB, 320072933376 bytes
255 heads, 63 sectors/track, 38913 cylinders, total 625142448 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000e6df5

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048   617170943   308584448   83  Linux
/dev/sda2       617172990   625141759     3984385    5  Extended
/dev/sda5       617172992   625141759     3984384   82  Linux swap / Solaris
该磁盘有255个heads，也就是说共有255个盘面。38913个柱面（cylinders），也就是说每个盘面上都有38913个磁道， 63 sectors/track说的是每个磁道上共有63个扇区。命令结果也给出了Sector size的值是512bytes。
255盘面  * 38913柱面 * 63扇区 * 每个扇区512bytes = 320072933376bytes
```

#### 磁盘的读写原理：
系统将文件存储到磁盘上时，按柱面、磁头、扇区的方式进行，即最先是第1磁道的第一磁头下（也就是第1盘面的第一磁道）的所有扇区，然后，是同一柱面的下一磁头，……，一个柱面存储满后就推进到下一个柱面，直到把文件内容全部写入磁盘。  
   
系统也以相同的顺序读出数据。读出数据时通过告诉磁盘控制器要读出扇区所在的柱面号、磁头号和扇区号（物理地址的三个组成部分）进行。磁盘控制器则直接使磁头部件步进到相应的柱面，选通相应的磁头，等待要求的扇区移动到磁头下。在扇区到来时，磁盘控制器读出每个扇区的头标，把这些头标中的地址信息与 期待检出的磁头和柱面号做比较（即寻道），然后，寻找要求的扇区号。待磁盘控制器找到该扇区头标时，根据其任务是写扇区还是读扇区，来决定是转换写电路， 还是读出数据和尾部记录。找到扇区后，磁盘控制器必须在继续寻找下一个扇区之前对该扇区的信息进行后处理。如果是读数据，控制器计算此数据的ECC码，然 后，把ECC码与已记录的ECC码相比较。如果是写数据，控制器计算出此数据的ECC码，与数据一起存储。在控制器对此扇区中的数据进行必要处理期间，磁 盘继续旋转。其实我们的文件大多数的时候都是破碎的，在文件没有破碎的时候,摇臂只需要寻找1次磁道并由磁头进行读取,只需要1次就可以成功读取;但是如果文件破碎成11处,那么摇臂要来回寻找11次磁道磁头进行11次读取才能完整的读取这个文件,读取时间相对没有破碎的时候就变得冗长。   
   
因此，磁盘IO时的过程包括：
  
    第一步，首先是磁头径向移动来寻找数据所在的磁道。这部分时间叫寻道时间。  
    第二步，找到目标磁道后通过盘面旋转，将目标扇区移动到磁头的正下方。  
    第三步，向目标扇区读取或者写入数据。到此为止，一次磁盘IO完成，故：  
  
所以，单次磁盘IO时间 = 寻道时间 + 旋转延迟 + 存取时间。   
  
    对于旋转延时，现在主流服务器上经常使用的是1W转/分钟的磁盘，每旋转一周所需的时间为60*1000/10000=6ms，故其旋转延迟为（0-6ms）。
    对于存取时间，一般耗时较短，为零点几ms。
    对于寻道时间，现代磁盘大概在3-15ms，其中寻道时间大小主要受磁头当前所在位置和目标磁道所在位置相对距离的影响。

#### 如何分区：
候选的磁盘分区方案：  

    方案一： 255个盘面，C盘是0-100盘面， D盘是101-200个盘面,……
    方案二：3263个柱面，C盘0-1000个柱面，D盘1001-20001个柱面,……

 
其实采用哪一种，最主要看的是那种方式性能更快。因为同一分区下的数据经常会一起读取，假如采用第一种，那么这样磁头就需要在3000多个track间不停地跳来跳去，这样磁盘的寻道时间就会翻倍，磁盘性能就会下降。而对于方案二，假如对于磁盘C，只需要在磁头在1-1000个磁道间移动就可以了，大大降低了寻道时间。（实际上分区并不是从0开始的，磁盘的第一个磁道对应的柱面会被用来安装引导加载程序以及磁盘分区表）。所以，方案二的分区方式可以降低磁盘IO时间中的寻道时间部分，所以所有的操作系统采用的都是方案二，没有用方案一的。

#### 几类硬盘
几类硬盘：  
磁盘种类 	最大IOPS 	最大响应时间
ATA/IDE 	70 	15ms
FC/SAS 	140~160 	10ms
SSD/EFD 	2500 	1ms
即IDE-->SCSI即目前的机械硬盘->ssd即固态硬盘

### 系统如何分区，格式化文件系统
Linux 下磁盘命名和分区   
在为主机添加硬盘前，首先要了解Linux系统下对硬盘和分区的命名方法。   
#### 1 磁盘命名  
在Linux下对 SCSI 和 SATA 设备是以 sd 命名的，第一个 scsi 设备是 sda，第二个是 sdb，依此类推。一般主板上有两个SCSI接口，因此一共可以安装四个SCSI设备。主 SCSI 上的两个设备分别对应 sda 和 sdb，第二个 SCSI 口上的两个设备对应 sdc 和 sdd。一般硬盘安装在主 SCSI 的主接口上，所以是 sda 或者 sdb，而光驱一般安装在第二个SCSI的主接口上，所以是 sdc。(IDE接口设备是用 hd 命名的，第一个设备是hda，第二个是hdb，依此类推。)

IDE 磁盘 	描述	配置
/dev/hda 	1st (Primary) IDE controller 	Master  
/dev/hdb 	1st (Primary) IDE controller 	Slave  
/dev/hdc 	2nd (Secondary) IDE controller 	Master   
/dev/hdd 	2nd (Secondary) IDE controller 	Slave  


#### 2磁盘分区
所谓的磁盘分区指的是告诉操作系统『我这颗磁盘在此分割槽可以存取的区域是由 A 磁柱到 B 磁柱之间的区块』， 如此一来操作系统就能够知道他可以在所指定的区块内进行文件数据的读/写/搜寻等动作了。 也就是说，磁盘分区意即指定分割槽的启始与结束磁柱就是了。  

分区表用于记录分区的信息
一个硬盘，有两种类型的分区表，一种传统流行的MBR，一种是GPT:  
ref:http://www.newskj.org/kjxx/2017062994604.html  
**MBR**  
（Master Boot Record，缩写：MBR），又叫做主引导扇区（主引导记录），是计算机开机后访问硬盘时所必须要读取的首个扇区，它在硬盘上的三维地址为（柱面，磁头，扇区）＝（0，0，1）。  
MBR是由分区程序（如Fdisk，Parted）所产生的，它不依赖任何操作系统，而且硬盘引导程序也是可以改变的，从而能够实现多系统引导.  
一个扇区是512字节，因此MBR的大小也是512字 节，其具体数据结构是：446个字节的引导代码、64个字节的分区表及2个字节的签名值"55AA"。  
由于MBR的分区表只有64个字节，这决定了它只能 存储4个分区记录。这就是为什么一块硬盘最多只能有4个“主分区"的原因。  

我们已经知道了MBR中的分表区只能存放4个分区（即4个主分区），那系统是如何划分出4个以上的分区的呢？一种直白而简单的思路就是把其中一个主分区再进 行细分，衍生出一个二级分区表。对的，这个被用来二次分区的主分区就是“扩展分区”，它下面的二级分区就是“逻辑分区”  

#### 3linux和Windows:
一个硬盘主分区至少有1个，最多4个，扩展分区可以没有，最多1个。且主分区+扩展分区总共不能超过4个。逻辑分区可以有若干个。  
扩展分区是我们对逻辑分区的总称，只是一中称呼，通过分区助手等工具可以看到如E,F盘为逻辑分区；

在windows下激活的主分区是硬盘的启动分区，他是独立的，也是硬盘的第一个分区，正常分的话就是C区。  
 在linux下主分区和逻辑分区都可以用来放系统，引导os开机，grub会兼容windows系统开机启动。  
分出主分区后，其余的部分可以分成扩展分区，一般是剩下的部分全部分成扩展分区，也可以不全分，那剩的部分就浪费了。  
但扩展分区是不能直接用的，他是以逻辑分区的方式来使用的，所以说扩展分区可分成若干逻辑分区。他们的关系是包含的关系，所有的逻辑分区都是扩展分区的一部分。  
在linux中第一块硬盘分区为hda分区，主分区编号为hda1-4，逻辑分区从5开始。  
硬盘的容量=主分区的容量+扩展分区的容量  
扩展分区的容量=各个逻辑分区的容量之和，1个扩展分区->24个逻辑分区  


#### 4linux下进行分区的方法
当我们买到一块硬盘，里面空空如也，或者通过虚拟机进行创建虚拟硬盘时，如何进行分区？  
当接入硬盘后：  
1、通过ls /dev/sd* 查看所有硬盘  找到sdb或其他如hd等（根据硬盘类型来）  
2、通过fdisk -l (root权限下) 查看未进行分区的硬盘  
3、通过fdisk /dev/sdn(对应未分区的硬盘名）进行分区，依次选择后，完成；  
4、格式化分区：mkfs.ext4 -L disk2 /dev/sdb 即对该分区创建文件系统；  
5、挂载该分区 sudo mount -t ext4 /dev/sdb1/ /mnt/  

过程总结： 
    创建：以某种方式格式化磁盘的过程就是在其之上建立一个文件系统的过程。创建文件系统时，会在磁盘的特定位置写入关于该文件系统的控制信息。  
    注册：向内核报到，声明自己能被内核支持。一般在编译内核的时侯注册；也可以加载模块的方式手动注册。注册过程实 际上是将表示各实际文件系统的数据结构struct file_system_type 实例化。  
    安装：也就是我们熟悉的mount操作，将文件系统加入到 Linux 的根文件系统的目录树结构上；这样文件系统才能被访问。  
开机直接挂载  
编辑 /etc/fstab 文件，添加：/dev/sda1 /test ext3 defaults 1 1，重启则发选已经挂载上去。  


#### 5 linux和windows文件系统和分区关系
每块硬盘都分为若干个分区，每个分区都有自己的文件系统。Windows为这些文件系统各自指定了一个字母。不过 GNU/Linux 使用唯一的树形结构来管理文件，而每个文件系统都挂载于树形结构的某个位置。  
正如 Windows 需要有 C: 驱动器一样，GNU/Linux 必须能够将根文件系统挂载于文件树的根(/)上。当根挂载完成之后，您就可以将其它文件系统挂载于树形结构各种挂载点上。根结构下的任何目录都可以作为挂载点，而您也可以将同一文件系统同时挂载于不同的挂载点上。挂载点实际上就是linux中的磁盘文件系统的入口目录：  

总结：从上面看，分区创建后需要格式化为某个类型的文件系统，所以分区和文件系统是一一对应的关系；


### linux文件系统架构：
#### 文件系统的概念和功能
##### wiki:
计算机的文件系统是一种存储和组织计算机数据的方法，它使得对其访问和查找变得容易，文件系统使用文件和树形目录的抽象逻辑概念代替了硬盘和光盘等物理设备使用数据块的概念，用户使用文件系统来保存数据不必关心数据实际保存在硬盘（或者光盘）的地址为多少的数据块上，只需要记住这个文件的所属目录和文件名。在写入新数据之前，用户不必关心硬盘上的那个块地址没有被使用，硬盘上的存储空间管理（分配和释放）功能由文件系统自动完成，用户只需要记住数据被写入到了哪个文件中。
严格地说，文件系统是一套实现了数据的存储、分级组织、访问和获取等操作的抽象数据类型（Abstract data type）

##### 总结文件系统的功能：
1、文件是放在硬盘中的，由物理作用写入的东西，而存放的东西需要管理，在使用时通过内存读入管理信息就能知道存放的内容，并进行进一步使用，
这个“管理” 就是文件系统；  
2、文件系统提供：  
     1) 硬盘文件的基本信息，文件基本信息，包括属性等；  
     2) 提供管理文件接口，如删除，增加文件等等；  

3、从功能上可以这样分：从底层到用户层：  
  1) 高速缓冲区的管理程序  --对硬盘等块设备的数据高速存取函数  driver层  
  2) 文件系统低层通用函数 --对文件索引节点的管理，磁盘数据块的分配和释放以及文件名和i节点的转换算法   
  3) 对文件中数据进行读写操作，包括对字符设备，管道，块读写文件中数据的访问  
  4) 文件系统调用接口的实现；--涉及文件打开，关闭，创建以及有关文件目录操作等的系统调用； VFS  

##### 查看：
我们在使用文件系统时，用到的大多是目录，文件，并不会去考虑实际的文件系统类型等，只有当比如插u盘的时候因为系统不支持u盘的文件系统显示不出来，才会去考虑这个问题；
那文件系统，和目录，文件是什么关系和区别？linux支持多种文件系统，比如ext系列，以及内存文件系统如procfs,它挂载在/proc下，也就是说，/proc这个路径和/usr这个路径是不同的文件系统；
比如通过如下命令，可以知道路径对应的文件系统

```
think@think-VirtualBox:~$ mount | column -t
/dev/sda1   on  /                         type  ext4             (rw,errors=remount-ro)
proc        on  /proc                     type  proc             (rw,noexec,nosuid,nodev)
sysfs       on  /sys                      type  sysfs            (rw,noexec,nosuid,nodev)
none        on  /sys/fs/cgroup            type  tmpfs            (rw)
none        on  /sys/fs/fuse/connections  type  fusectl          (rw)
none        on  /sys/kernel/debug         type  debugfs          (rw)
none        on  /sys/kernel/security      type  securityfs       (rw)
udev        on  /dev                      type  devtmpfs         (rw,mode=0755)
devpts      on  /dev/pts                  type  devpts           (rw,noexec,nosuid,gid=5,mode=0620)
tmpfs       on  /run                      type  tmpfs            (rw,noexec,nosuid,size=10%,mode=0755)
none        on  /run/lock                 type  tmpfs            (rw,noexec,nosuid,nodev,size=5242880)
none        on  /run/shm                  type  tmpfs            (rw,nosuid,nodev)
none        on  /run/user                 type  tmpfs            (rw,noexec,nosuid,nodev,size=104857600,mode=0755)
none        on  /sys/fs/pstore            type  pstore           (rw)
systemd     on  /sys/fs/cgroup/systemd    type  cgroup           (rw,noexec,nosuid,nodev,none,name=systemd)
gvfsd-fuse  on  /run/user/1000/gvfs       type  fuse.gvfsd-fuse  (rw,nosuid,nodev,user=think)

以及：
think@think-VirtualBox:~$ df -H
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G  4.1k  2.0G   1% /dev
tmpfs           415M  926k  414M   1% /run
/dev/sda1        28G   25G  1.6G  95% /
none            4.1k     0  4.1k   0% /sys/fs/cgroup
none            5.3M     0  5.3M   0% /run/lock
none            2.1G   78k  2.1G   1% /run/shm
none            105M   66k  105M   1% /run/user
```


#### 文件系统架构
{% asset_img vfs0.png vfs0 about %}

VFS本质是什么？   
linux为了支持各种文件系统，且在同时允许访问其他操作系统文件，linux内核在用户进程(或c库）和文件系统实现之间引入了一个抽象层，这个抽象层称为VFS;  
VFS的任务：  
一方面，提供一种操作文件，目录和其他对象的统一方法，另一方面，它必须能够与各种方法给出的具体文件系统达成妥协；
用户空间包含一些应用程序（例如，文件系统的使用者）和 GNU C 库（glibc），它们为文件系统调用（打开、读取、写和关闭）提供用户接口。系统调用接口的作用就像是交换器，它将系统调用从用户空间发送到内核空间中的适当端点。  

#### 文件系统的分类：
文件系统分3种：  
1) 基于磁盘的文件系统：如Ext2/3/4,Reiserfs,FAT,等，都是面向块的介质，需解决：如何将文件内容与结构信息存于目录层次结构上；从文件系统角度看，底层设备无非是存储块组成的一个列表，文件系统相当于对该列表实施一个适当的组织方案；  
2) 虚拟文件系统：内核中生成，允许用户程序和内核信息通信的方法：/proc等；是内核在内存上建立的一个层次化的文件结构，文件大小为0；  
3) 网络文件系统；这种文件系统允许访问另一个计算机的数据；数据存储于其他计算机上，内核无须关注文件存取等细节，而是通过命令和特定的协议如FTP等，而由于VFS抽象层，所以用户进程体会不到区别；  
#### 文件系统的组成（通用文件模型）概述：
+ 概述：  
VFS提供了一种结构模型，包含了一个强大文件系统所应具备的所有组件，该模型只存在于虚拟中，必须使用各种对象和函数指针与每种文件系统适配；
+ 文件的概念：  
在处理文件时，内核空间和用户空间使用的主要对象不同，用户程序而言，是一个fd,在所有关于文件操作中该整数作为标识文件的参数。文件描述符是在打开文件时由内核分配，只
在一个进程内部有效；两个不同的进程可以用同样的文件描述符，但是二者不是指向同一个文件。所以基于同一个fd来共享文件是不可能的；
对应到进程结构，是task_struct->file struct来表示一个文件结构的；
+ 内核空间的文件：inode,每个文件(和目录)都有且只有一个对应的inode;inode包含很多文件信息除了文件名；  
inode结构中的成员分类：
(1）描述文件状态的元数据，如访问权限或上次修改的日期  
(2) 保存实际文件内容的数据段，用于保存文件  
一个例子：/usr/bin/emacs的inode查找：  
(1)setp1：根目录的inode/,其数据段不保存文件数据而是根目录下的各个目录项，表示文件或目录，每个目录项由inode编号(唯一）和文件或目录名称构成；  
(2)step2: 查找子目录usr的inode,即在/对应的inode的数据段查找各个目录项，匹配usr,找到后在usr重复此过程，不过查找的是bin  
(3)step3: 查找bin的目录项(目录项可能对应的是文件或目录),直到匹配上emacs,到此结束，emacs的inode的数据段指向的是文件内容；  

{% asset_img inode1.png inode1 about %}  

+ 链接和编程接口  
链接：软链接和硬链接：  
   软链接：即符号链接，删除链接文件不会影响源文件，软链接文件用一个独立的inode，数据段保存一个字符串给出链接目标路径；  
   硬链接：无法区分哪个文件先创建，因为创建的目录项使用了一个现存的ino de编号；假设A是B的硬链接文件，删除A时，会删除A和B共享的inode,导致B也不能用了，当然，在inode中加入一个计数器即可防止这种情况；++直到计数器为0才删除inode  
编程接口：  
   open/write  
   struct file和文件描述符  
+ 万物皆文件：  
#### VFS的结构组成：
+ 结构概观：VFS由两个部分组成：文件和文件系统；  
文件的表示和其他关联结构如下：包含结构和操作函数指针集合；  
图：
{% asset_img vfs.png vfs1 about %}

结构分类：inode,超级块，目录，file

##### inode:表示一个具体的文件
inode操作：创建链接，文件重命名，目录中生存新文件，删除文件
前面有提到inode表示文件，即一个文件用一个inode表示，而文件关联的数据块信息，在inode中保存；

```cpp
inode:
    i_op:指向inode的操作函数
    i_fop:指向inode对应的file的操作集合；
    i_mapping:指向
    i_sb:执行超级块
    i_list:可以将inode存于一个链表中，总共3种inode状态：
                   1-inode未关联到任何文件也为活动；对应inode_unuserd全局链表
                   2-inode所有使用但未改变得inode : inode_in_use全局链表
                   3-inode已经使用，且被改变，即与存储介质上的内容不同个，即脏inode:保存于一个特定于超级块的链表中；s_inodes;
```

1) inode操作函数集合：  
i_op: 即inode_operations:即用于负责管理结构性的操作，比如删除一个文件和文件相关的元数据，如属性；  
i_fop: 和file结构，都是指向一个file_operations结构的指针，用于操作文件中包含的数据，比如读数据，写数据  

2) 在VFS中，内核通过三个全局链表来管理所有的inode:  
(1) 每个inode都有一个i_list成员，可以将inode存储在一个链表中，根据inode的状态，他可能有3种主要的情况：    
inode存在于内存中，未关联到任何文件，也不处于活动使用状态；  
inode结构在内存中，正在由一个或多个进程使用，通常表示一个文件，两个计数器(i_count和i_nlink)都必须大于0；即上次与存储介质同步后，该inode没有改变过；  
inode处于活动状态，其数据内容已经改变，与介质内容不同，即是脏的；  

(2) 在fs/inode.c中有两个全局变量做表头：  
inode_unused,即上述第一类  
inode_in_use:即第二类  
而脏的第三类，在一个特定于超级块的链表中；  

(3) 还有一种情况就是，当移动介质改变时，此时所有的inode都无效了；无效的inode存在一个本地的链表中；  

(4) 除了上面的和状态有关的全局结构，还有一个散列表，用来支持根据inode编号和超级块快速访问inode; fs/inode.c: inode_hashtable;  
    另外在超级表中，还有一个s_inodes表头的，i_sb_list做链表元素的链表；

总结来讲，就是每个文件有个关联的inode,vfs内核将所有的inode通过链表和散列表管理起来；通过inode编号等可以找到对应的inode

##### file：表示一个与进程相关联的已打开的文件，和inode区别在，不同进程共享一个inode;但是不同进程各自维护file对象
fd用于在一个进程内唯一表示打开的文件；而在进程结构中，包含了一个file结构，来对应这个fd;
```cpp
/* filesystem information */
	struct fs_struct *fs;
/* open file information */
	struct files_struct *files;
/*
 * Open file table structure
 */
struct files_struct {
  /*
   * read mostly part
   */
	atomic_t count;
	struct fdtable __rcu *fdt;
	struct fdtable fdtab;
  /*
   * written part on a separate cache line in SMP
   */
	spinlock_t file_lock ____cacheline_aligned_in_smp;
	int next_fd; //下一次打开新文件时使用的文件描述符
	unsigned long close_on_exec_init[1];
	unsigned long open_fds_init[1];
	struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};

struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path		f_path;//封装了文件名和inode之间的关联，以及文件所在文件系统的信息
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;//指向了所有的文件操作接口，如打开文件，读写文件，其中的open用于打开一个文件，相当于将一个file对象关联到inode

	/*
	 * Protects f_ep_links, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	atomic_long_t		f_count;
	unsigned int 		f_flags;
	fmode_t			f_mode;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct list_head	f_ep_links;
	struct list_head	f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
} __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */

```
files包含当前进程的各个文件描述符   
file_operations	*f_op;//指向了所有的文件操作接口，如打开文件，读写文件，其中的open用于打开一个文件，相当于将一个file对象关联到inode,
不同的文件系统，设备，用不同的操作函数更新如：  
fs/block_dev.c:
```cpp
const struct file_operations def_blk_fops = {
	.open		= blkdev_open,
	.release	= blkdev_close,
	.llseek		= block_llseek,
	.read_iter	= blkdev_read_iter,
	.write_iter	= blkdev_write_iter,
	.mmap		= generic_file_mmap,
	.fsync		= blkdev_fsync,
	.unlocked_ioctl	= block_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl	= compat_blkdev_ioctl,
#endif
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
};
fs/ext3/file.c:
是不同的初始化函数，具体不赘述；

```

##### 关于超级块：superblock对象： 表示一个具体的可封装的文件系统，相当于书的封面
整个文件系统的第一块空间，包含整个文件系统的基本信息，如块大小，指向空间的inode和数据块的指针等相关信息  
VFS支持的文件系统类型通过一种特殊的内核对象连接进来，该对象提供了一种读取超级块的方法；  
超级块信息：文件系统关键信息如块长度，最大文件长，读，写，操作inode的函数指针；  
内核建立了一个链表，存储所有活动文件系统的超级块实例  
超级块还有一个重要成员列表，包括相关文件系统中所有修改过的inode；  

##### 关于目录：目录信息，目录项缓存，命名空间，dentry: 表示一个目录条目，或路径中的一个分量；
（1）在进程结构task_struct中，还有一个成员，用来维护和目录相关的信息：  
```cpp
struct fs_struct
{
    struct dentry *root,pwd,altroot...//指定了相关进程的根目录,当前工作目录和特性目录
    struct vfsmount *rootmnt，pwdmnt,.://指定了相关进程的文件系统；当前工作目录的文件系统的vfsmount结构
}
```
（2）VFS命名空间：  
是为容器实现提供了可能，即每个容器分别跟踪装载的文件系统。VFS命名空间是所有已装载，构成某个容器目录树的文件系统的集合
对应的struct task_struct中的nsproxy  
(3) 目录项缓存；  
因为在inode查找时，即使设备数据已经在页缓存中，仍然会每次都重复整个查找操作，所以用目录项缓存(dentry缓存)来快速访问此前的查找结果；  
     创建：内核在读取一个目录项（文件或目录）的数据后，就会创建一个dentry实例，以缓存找到的数据；可以从上图看到，file结构和inode结构都有一个相关的指针指向此结构；  
     dentry结构：  
     dentry缓存的组织：一个散列表：dentry_hashtable包含了所有的dentry对象  
     dentry的操作函数集合：dentry_operations;  
     dentry的一些标准函数操作  

##### 文件系统中的文件操作，如何将上述结构关联：
+ open文件： int open(const char *pathname, int flags, mode_t mode);
最重要的参数是pathname,而返回的fd是后续执行的关键：
在fs/open.c:
```cpp
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
	if (force_o_largefile())
		flags |= O_LARGEFILE;

	return do_sys_open(AT_FDCWD, filename, flags, mode);
}
long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
	struct open_flags op;
	int fd = build_open_flags(flags, mode, &op);
	struct filename *tmp;

	if (fd)
		return fd;
    //使用相关函数从用户空间得到文件名
	tmp = getname(filename);
	if (IS_ERR(tmp))
		return PTR_ERR(tmp);
    //分配一个fd，并做相关初始化
	fd = get_unused_fd_flags(flags);
	if (fd >= 0) {
		struct file *f = do_filp_open(dfd, tmp, &op);
        //1 获取分配struct file,
        //2 查找文件名对应的inode：path_lookup,link_path_walk,do_follow_link
        // 内核试图在dentry缓存中找inode，并检查缓存是否有效，主要是操作dentry;
        //3 调用f_op->open函数,从而执行到设备层面的打开
        //4 返回file结构
		if (IS_ERR(f)) {
			put_unused_fd(fd);
			fd = PTR_ERR(f);
		} else {
			fsnotify_open(f);
			fd_install(fd, f);
		}
	}
	putname(tmp);
	return fd;
}
```
中可以看到，open的基本过程
1 根据文件名查找inode, 若文件存在，则通过文件系统找到对应的inode，若文件不存在，则创建文件和关联；
2 初始化fd等结构，   
3 执行底层的open函数；  
注意：inode是初始化时就创建了，而创建文件，是将文件和inode关联起来；  
```
inode也会消耗硬盘空间，所以硬盘格式化的时候，操作系统自动将硬盘分成两个区域。一个是数据区，存放文件数据；另一个是inode区（inode table），存放inode所包含的信息。
每个inode节点的大小，一般是128字节或256字节。inode节点的总数，在格式化时就给定，一般是每1KB或每2KB就设置一个inode。假定在一块1GB的硬盘中，每个inode节点的大小为128字节，每1KB就设置一个inode，那么inode table的大小就会达到128MB，占整块硬盘的12.8%。
查看每个硬盘分区的inode总数和已经使用的数量，可以使用df命令。
由于每个文件都必须有一个inode，因此有可能发生inode已经用光，但是硬盘还未存满的情况。这时，就无法在硬盘上创建新文件。
```

```
think@think-VirtualBox:~$ df -i
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
udev            482138    475  481663    1% /dev
tmpfs           505829    437  505392    1% /run
/dev/sda1      1703936 481526 1222410   29% /
none            505829      2  505827    1% /sys/fs/cgroup
none            505829      3  505826    1% /run/lock
none            505829      4  505825    1% /run/shm
none            505829     28  505801    1% /run/user
```
+ read文件
+ write文件：


##### VFS对象的操作： 即关于创建和操作文件系统，注册，挂载等
在创建文件系统和挂在文件系统时，主要涉及以下结构和操作：
+ 结构：
```cpp
file_system_type:用于描述文件系统的结构
struct file_system_type {
	const char *name;//文件系统名：eg: procfs
	int fs_flags;//使用标志，如只读庄子，禁止setuid等
#define FS_REQUIRES_DEV		1 
#define FS_BINARY_MOUNTDATA	2
#define FS_HAS_SUBTYPE		4
#define FS_USERNS_MOUNT		8	/* Can be mounted by userns root */
#define FS_USERNS_DEV_MOUNT	16 /* A userns mount does not imply MNT_NODEV */
#define FS_USERNS_VISIBLE	32	/* FS must already be visible */
#define FS_RENAME_DOES_D_MOVE	32768	/* FS will handle d_move() during rename() internally. */
	struct dentry *(*mount) (struct file_system_type *, int,
		       const char *, void *);//挂载时使用
	void (*kill_sb) (struct super_block *);//不再需要此文件系统时的清理
	struct module *owner;//以模块加载时使用
	struct file_system_type * next;//各个可用的文件系统通过这个链接起来
	struct hlist_head fs_supers;同一文件系统类型可能对应多个超级块结构，所以通过这个结构聚集在一个链表中；这个是表头

	struct lock_class_key s_lock_key;
	struct lock_class_key s_umount_key;
	struct lock_class_key s_vfs_rename_key;
	struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];

	struct lock_class_key i_lock_key;
	struct lock_class_key i_mutex_key;
	struct lock_class_key i_mutex_dir_key;
};
```
vfsmount: 维护挂载点相关信息等
super_block: 超级块以及它的操作函数集合，主要是对inode的管理，注意inode是fs层面的，和进程本身没有太大关系；  
+ 操作：
注册文件系统: register_filesystem;  
装载文件系统：mount  
卸载文件系统：umount  

+ 所以，若想创建一个新的文件系统类型，需要自己初始化上面结构的值，以及对应的操作函数集合ops,同时调用注册，装载，等函数；
这里新的是，如何管理和分配，读写inode的接口，新的超级块，挂载结构vfsmount和文件系统结构file_system_type 实例；
这里不变的，是inode，dentry等文件相关的结构及其设备相关的操作集合；若想变，需要做更大的支持和改动；

##### 从inode到具体的设备操作函数：
+ 块设备和硬盘，分区的对应关系：  
将一个磁盘添加到系统中时，内核将读取并分析底层块设备上的分区信息，但并不会对各个分区创建block_device(块设备结构)实例。为此，内核使用以下数据结构，对已经分区的
硬盘提供了一种表示：
<genhd.h> struct gendisk{
    int major: 主设备号
}
块设备，通用硬盘和分区之间的关联,todo 

+ 块设备的操作和请求队列等相关结构  
虚拟文件系统中的每个文件都关联到恰好一个inode,用于管理文件的属性等；inode结构有设备相关的成员，如i_rdev,i_mode,i_fops,i_bdev,i_cdev等；  
inode中有f_ops结构，被初始化为设备对应的操作函数集合，不同的设备，有不同的ops；  
而inode中也有设备相关的字段，主要是主设备号和从设备号，通过这两个可以确定操作的文件对应的设备，并通过这两个设备号  
拿到对应的设备操作函数集合，从而正确的操作；
inode,cdev,file,ops之间的关系；  
+ 向系统添加磁盘和分区--其实就是对上面gendisk及其关联结构的创建和操作等 TD  

#### 创建一个小的文件系统：
上面有提到，用户接口如何初始化一个分区格式为一个文件系统：  
这里是介绍如何实现一个小的文件系统；  
https://github.com/accelazh/hellofs  
这里有一个github上实现的简单的文件系统，但是和笔者想的不太一样，这个不免粗暴了些；而且有些重要的接口反而没有实现，比如alloc_inode等inode管理的
函数；  
待有时间再实现；  

#### 文件系统的内核接口  
参考：网络socketfs等等实现；  
http://reader.epubee.com/books/mobile/eb/eba6bf6a9f551ecdf3cf131e59937e41/text00037.html  
#### 文件系统的用户接口  
略，详细可以查相关资料；  
