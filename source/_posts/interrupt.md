---
title: interrupt
date: 2021-04-18 00:42:52
tags: interrupt
categories: linux
---





### 中断概述
除了系统调用外，中断也是一种内核活动；系统调用不是在用户态和系统状态之间切换的唯一途径
中断的处理往往会涉及到汇编等和硬件相关的代码，但是中断处理部分随着时间的演化，已经达到了这样一种状态：高层代码和底层的硬件交互代码，已经尽可能有效而干净
地分隔开了。<!--more-->

### 中断框架：
老版本的linux，中断代码比较杂乱，新版本，因为其中引入了一个用于中断和IRQ的通用框架。各个平台现在只负责在最低层次上与硬件交互。所有其他功能都由通用代码提供
#### 中断类型：
同步中断和异步中断，两者都是通过中断服务程序（ISR中断处理程序）来处理的
##### 同步中断和异常
由cpu产生，针对当前执行的程序，比如除0，内核可能通过信号机制通知进程，进程做默认或注册行为比如输出错误信息，或者直接结束；
异常情况也可能是如缺页异常，这种由内核自动修复，进程往往无感知；

##### 异步中断
经典的中断，由外部设备产生，可能发生在任意时间。不同于同步中断，
异步中断并不与特定进程关联。它们可能发生在任何时间，而不牵涉系统当前执行的活动。 ①
网卡通过发出一个相关的中断来报告新分组的到达。因为数据可能在任意时刻到达系统，所
以当前执行的很可能是与数据无关的某个进程或其他东西。为避免损害该进程，内核必须确
保中断能够尽快处理完毕（通过缓冲数据），使得CPU时间能够返还给当前进程。这也是内核
需要延期操作机制的原因

##### 中断的一些其他逻辑：
+ 禁用中断，应尽量避免；
+ 设备共享中断编号--中断共享

### 硬件中断
由系统自身和与之连接的外设自动产生。它们用于支持更高效地实现设备驱动程序，也用于引起处理器自身对异常或错误的关注，这些是需要与内核代码进行交互的。
中断不能由处理器外部的外设直接产生，而必须借助于一个称为中断控制器(interrupt controller)的标准组件来请求，该组件存在于每个系统中。
/proc/interrupts 这个文件可以查看系统的中断编号和对应中断；
```cpp
think@think-VirtualBox:~/c++/ptrace$ cat /proc/interrupts 
           CPU0       CPU1       
  0:         39          0   IO-APIC   2-edge      timer
  1:       3038          0   IO-APIC   1-edge      i8042
  8:          0          0   IO-APIC   8-edge      rtc0
  9:          0          0   IO-APIC   9-fasteoi   acpi
 12:       3144          0   IO-APIC  12-edge      i8042
 14:          0          0   IO-APIC  14-edge      ata_piix
 15:       9824          0   IO-APIC  15-edge      ata_piix
 18:          0          0   IO-APIC  18-fasteoi   vmwgfx
 19:        129       7562   IO-APIC  19-fasteoi   eth0
 20:      18055          0   IO-APIC  20-fasteoi   vboxguest
 21:      46875          0   IO-APIC  21-fasteoi   0000:00:0d.0, snd_intel8x0
 22:         27          0   IO-APIC  22-fasteoi   ohci_hcd:usb1
NMI:          0          0   Non-maskable interrupts
LOC:     257382     487413   Local timer interrupts
SPU:          0          0   Spurious interrupts
PMI:          0          0   Performance monitoring interrupts
IWI:          0          0   IRQ work interrupts
RTR:          0          0   APIC ICR read retries
RES:     299362     323826   Rescheduling interrupts
CAL:       2173      26988   Function call interrupts
TLB:       1760       1871   TLB shootdowns
TRM:          0          0   Thermal event interrupts
THR:          0          0   Threshold APIC interrupts
DFR:          0          0   Deferred Error APIC interrupts
MCE:          0          0   Machine check exceptions
MCP:         35         35   Machine check polls
ERR:          0
MIS:          0
PIN:          0          0   Posted-interrupt notification event
PIW:          0          0   Posted-interrupt wakeup event
```

#### 硬件中断的处理
##### 进入和退出任务：
首先，必须建立一个适当的环境，使得处理程序函数能够在其中执行，接下来调用处理程序自身，最后将系统复原（在当前程序看来）到中断之前的状态。
进入是从用户态切到核心太，退出是从核心态到用户态，而且需要通过调度器是否选择新进程替代旧进程；中断到达时也可能处于核心态，所以这个时候需要判断；

##### 中断处理程序
+ 特点：
  可以禁用中断来防止被打断，但是只能短时间使用，处理过程短暂；
  可以在其他ISR执行期间调用的中断处理程序，不能彼此干扰
  可延期的处理，不用在中断处理程序中实现，可以放到类似下半部的软中断；或者tasklet

#### 中断相关数据结构：
+ 总的数组：
在kernel/irq/handle.c,kernel/irq/irqdesc.c:可以看到初始化的中断处理数组；每个都被处理为handle_bad_irq;此时还没赋值；
赋值发生在驱动程序或模块初始化时调用request_irq等类似函数进行注册；
```cpp
struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {
	[0 ... NR_IRQS-1] = {
		.handle_irq	= handle_bad_irq,
		.depth		= 1,
		.lock		= __RAW_SPIN_LOCK_UNLOCKED(irq_desc->lock),
	}
};
```
但IRQ的最大可能数目是通过一个平台相关的常数NR_IRQS指定的。大多数体系结构下，该常数定义在处理器相关的头文件include/asm-generic/irq.h中
不同处理器可能支持的不同；
```cpp
#ifndef NR_IRQS
#define NR_IRQS 64
#endif
```

+ 数组每个元素：
看下中断处理数组中每个元素的结构：每个irq都可以由该结构完全描述
```cpp
/**
 * struct irq_desc - interrupt descriptor
 * @irq_common_data:	per irq and chip data passed down to chip functions
 * @kstat_irqs:		irq stats per cpu
 * @handle_irq:		highlevel irq-events handler
 电流层ISR由handle_irq提供。 handler_data可以指向任意数据，该数据可以是特定于IRQ
或处理程序的。每当发生中断时，特定于体系结构的代码都会调用handle_irq。该函数负责
使用chip中提供的特定于控制器的方法，进行处理中断所必需的一些底层操作。用于不同中
断类型的默认函数由内核提供。 
 * @preflow_handler:	handler called before the flow handler (currently used by sparc)
 * @action:		the irq action chain    irq操作列表
 action提供了一个操作链，需要在中断发生时执行。由中断通知的设备驱动程序，可以将与
之相关的处理程序函数放置在此处。有一个专门的数据结构用于表示这些操作
 * @status:		status information      irq状态
 
/*
 * Bit masks for irq_common_data.state_use_accessors
 *
 * IRQD_TRIGGER_MASK		- Mask for the trigger type bits
 * IRQD_SETAFFINITY_PENDING	- Affinity setting is pending
 * IRQD_NO_BALANCING		- Balancing disabled for this IRQ
 * IRQD_PER_CPU			- Interrupt is per cpu
 * IRQD_AFFINITY_SET		- Interrupt affinity was set
 * IRQD_LEVEL			- Interrupt is level triggered
 * IRQD_WAKEUP_STATE		- Interrupt is configured for wakeup
 *				  from suspend
 * IRDQ_MOVE_PCNTXT		- Interrupt can be moved in process
 *				  context
 * IRQD_IRQ_DISABLED		- Disabled state of the interrupt
 * IRQD_IRQ_MASKED		- Masked state of the interrupt
 * IRQD_IRQ_INPROGRESS		- In progress state of the interrupt
 * IRQD_WAKEUP_ARMED		- Wakeup mode armed
 */
 /*
enum {
	IRQD_TRIGGER_MASK		= 0xf,
	IRQD_SETAFFINITY_PENDING	= (1 <<  8),
	IRQD_NO_BALANCING		= (1 << 10),
	IRQD_PER_CPU			= (1 << 11),
	IRQD_AFFINITY_SET		= (1 << 12),
	IRQD_LEVEL			= (1 << 13),
	IRQD_WAKEUP_STATE		= (1 << 14),
	IRQD_MOVE_PCNTXT		= (1 << 15),
	IRQD_IRQ_DISABLED		= (1 << 16),
	IRQD_IRQ_MASKED			= (1 << 17),
	IRQD_IRQ_INPROGRESS		= (1 << 18),
	IRQD_WAKEUP_ARMED		= (1 << 19),
};*/

/* @core_internal_state__do_not_mess_with_it: core internal status information
 * @depth:		disable-depth, for nested irq_disable() calls
 * @wake_depth:		enable depth, for multiple irq_set_irq_wake() callers
 * @irq_count:		stats field to detect stalled irqs
 * @last_unhandled:	aging timer for unhandled count
 * @irqs_unhandled:	stats field for spurious unhandled interrupts
 * @threads_handled:	stats field for deferred spurious detection of threaded handlers
 * @threads_handled_last: comparator field for deferred spurious detection of theraded handlers
 * @lock:		locking for SMP
 * @affinity_hint:	hint to user space for preferred irq affinity
 * @affinity_notify:	context for notification of affinity changes
 * @pending_mask:	pending rebalanced interrupts
 * @threads_oneshot:	bitfield to handle shared oneshot threads
 * @threads_active:	number of irqaction threads currently running
 * @wait_for_threads:	wait queue for sync_irq to wait for threaded handlers
 * @nr_actions:		number of installed actions on this descriptor
 * @no_suspend_depth:	number of irqactions on a irq descriptor with
 *			IRQF_NO_SUSPEND set
 * @force_resume_depth:	number of irqactions on a irq descriptor with
 *			IRQF_FORCE_RESUME set
 * @dir:		/proc/irq/ procfs entry
 * @name:		flow handler name for /proc/interrupts output
  name指定了电流层处理程序的名称，将显示在/proc/interrupts中。对边沿触发中断，通常
是“edge”，对电平触发中断，通常是“level”。
 */
struct irq_desc {
	struct irq_common_data	irq_common_data;
	struct irq_data		irq_data; //其中包含了irq_chip成员：用于表示一个IRQ控制器抽象的具体特征；
//通常在一个系统上只有一种类型的中断控制器会占据支配地位（当然，并没有什么约束条件阻止多个控制器并存），所有irq_desc的 irq_data的chip成员都指向irq_chip的同一实例
	unsigned int __percpu	*kstat_irqs;
	irq_flow_handler_t	handle_irq;
#ifdef CONFIG_IRQ_PREFLOW_FASTEOI
	irq_preflow_handler_t	preflow_handler;
#endif
	struct irqaction	*action;	/* IRQ action list */
	unsigned int		status_use_accessors;
	unsigned int		core_internal_state__do_not_mess_with_it;
	unsigned int		depth;		/* nested irq disables */
	unsigned int		wake_depth;	/* nested wake enables */
	unsigned int		irq_count;	/* For detecting broken IRQs */
	unsigned long		last_unhandled;	/* Aging timer for unhandled count */
	unsigned int		irqs_unhandled;
	atomic_t		threads_handled;
	int			threads_handled_last;
	raw_spinlock_t		lock;
	struct cpumask		*percpu_enabled;
#ifdef CONFIG_SMP
	const struct cpumask	*affinity_hint;
	struct irq_affinity_notify *affinity_notify;
#ifdef CONFIG_GENERIC_PENDING_IRQ
	cpumask_var_t		pending_mask;
#endif
#endif
	unsigned long		threads_oneshot;
	atomic_t		threads_active;
	wait_queue_head_t       wait_for_threads;
#ifdef CONFIG_PM_SLEEP
	unsigned int		nr_actions;
	unsigned int		no_suspend_depth;
	unsigned int		cond_suspend_depth;
	unsigned int		force_resume_depth;
#endif
#ifdef CONFIG_PROC_FS
	struct proc_dir_entry	*dir;
#endif
	int			parent_irq;
	struct module		*owner;
	const char		*name;
} ____cacheline_internodealigned_in_smp;
```
一些结构的解释：

+ 	irq控制器抽象：
struct irq_data		irq_data; //其中包含了irq_chip成员：用于表示一个IRQ控制器抽象的具体特征；
```cpp
/**
 * struct irq_chip - hardware interrupt chip descriptor
 *
 * @name:		name for /proc/interrupts
 * @irq_startup:	start up the interrupt (defaults to ->enable if NULL)
 * @irq_shutdown:	shut down the interrupt (defaults to ->disable if NULL)
 * @irq_enable:		enable the interrupt (defaults to chip->unmask if NULL)
 * @irq_disable:	disable the interrupt
 * @irq_ack:		start of a new interrupt
 * @irq_mask:		mask an interrupt source
 * @irq_mask_ack:	ack and mask an interrupt source
 * @irq_unmask:		unmask an interrupt source
 * @irq_eoi:		end of interrupt
 * @irq_set_affinity:	set the CPU affinity on SMP machines
 * @irq_retrigger:	resend an IRQ to the CPU
 * @irq_set_type:	set the flow type (IRQ_TYPE_LEVEL/etc.) of an IRQ
 * @irq_set_wake:	enable/disable power-management wake-on of an IRQ
 * @irq_bus_lock:	function to lock access to slow bus (i2c) chips
 * @irq_bus_sync_unlock:function to sync and unlock slow bus (i2c) chips
 * @irq_cpu_online:	configure an interrupt source for a secondary CPU
 * @irq_cpu_offline:	un-configure an interrupt source for a secondary CPU
 * @irq_suspend:	function called from core code on suspend once per chip
 * @irq_resume:		function called from core code on resume once per chip
 * @irq_pm_shutdown:	function called from core code on shutdown once per chip
 * @irq_calc_mask:	Optional function to set irq_data.mask for special cases
 * @irq_print_chip:	optional to print special chip info in show_interrupts
 * @irq_request_resources:	optional to request resources before calling
 *				any other callback related to this irq
 * @irq_release_resources:	optional to release resources acquired with
 *				irq_request_resources
 * @irq_compose_msi_msg:	optional to compose message content for MSI
 * @irq_write_msi_msg:	optional to write message content for MSI
 * @irq_get_irqchip_state:	return the internal state of an interrupt
 * @irq_set_irqchip_state:	set the internal state of a interrupt
 * @flags:		chip specific flags
 */
 结构需要考虑内核中出现的各个IRQ实现的所有特性。因而，一个该结构的特定实例，通常只
定义所有可能方法的一个子集
struct irq_chip {
	const char	*name;//表示硬件控制器，
	unsigned int	(*irq_startup)(struct irq_data *data);//用于第一次初始化一个IRQ
	void		(*irq_shutdown)(struct irq_data *data);//完全关闭一个中断源
	void		(*irq_enable)(struct irq_data *data);//用于激活一个IRQ
	void		(*irq_disable)(struct irq_data *data);//禁用IRQ

	void		(*irq_ack)(struct irq_data *data);
    //在某些模型中， IRQ请求的到达（以及在处理器的对应中
断）必须显式确认，后续的请求才能进行处理。如果芯片组没有这样的要求，该指针可以指
向一个空函数，或NULL指针。 ack_and_mask确认一个中断，并在接下来屏蔽该中断。
	void		(*irq_mask)(struct irq_data *data);
	void		(*irq_mask_ack)(struct irq_data *data);
	void		(*irq_unmask)(struct irq_data *data);
	void		(*irq_eoi)(struct irq_data *data);//eoi表示end of interrupt，即中断结束

	int		(*irq_set_affinity)(struct irq_data *data, const struct cpumask *dest, bool force);
	int		(*irq_retrigger)(struct irq_data *data);
	int		(*irq_set_type)(struct irq_data *data, unsigned int flow_type);
	int		(*irq_set_wake)(struct irq_data *data, unsigned int on);

	void		(*irq_bus_lock)(struct irq_data *data);
	void		(*irq_bus_sync_unlock)(struct irq_data *data);

	void		(*irq_cpu_online)(struct irq_data *data);
	void		(*irq_cpu_offline)(struct irq_data *data);

	void		(*irq_suspend)(struct irq_data *data);
	void		(*irq_resume)(struct irq_data *data);
	void		(*irq_pm_shutdown)(struct irq_data *data);

	void		(*irq_calc_mask)(struct irq_data *data);

	void		(*irq_print_chip)(struct irq_data *data, struct seq_file *p);
	int		(*irq_request_resources)(struct irq_data *data);
	void		(*irq_release_resources)(struct irq_data *data);

	void		(*irq_compose_msi_msg)(struct irq_data *data, struct msi_msg *msg);
	void		(*irq_write_msi_msg)(struct irq_data *data, struct msi_msg *msg);

	int		(*irq_get_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool *state);
	int		(*irq_set_irqchip_state)(struct irq_data *data, enum irqchip_irq_state which, bool state);

	unsigned long	flags;
};
```

找一个例子看看：/arch/x86/kernel/i8295.c:
```cpp
struct irq_chip i8259A_chip = {
	.name		= "XT-PIC",
	.irq_mask	= disable_8259A_irq,
	.irq_disable	= disable_8259A_irq,
	.irq_unmask	= enable_8259A_irq,
	.irq_mask_ack	= mask_and_ack_8259A,
};

```

+ 处理程序函数的表示：
```cpp
/**
 * struct irqaction - per interrupt action descriptor
 * @handler:	interrupt handler function
 * @name:	name of the device
 * @dev_id:	cookie to identify the device
 * @percpu_dev_id:	cookie to identify the device
 * @next:	pointer to the next irqaction for shared interrupts
 * @irq:	interrupt number
 * @flags:	flags (see IRQF_* above)
 * @thread_fn:	interrupt handler function for threaded interrupts
 * @thread:	thread pointer for threaded interrupts
 * @thread_flags:	flags related to @thread
 * @thread_mask:	bitmask for keeping track of @thread activity
 * @dir:	pointer to the proc/irq/NN/name entry
 */
struct irqaction {
	irq_handler_t		handler;//函数指针，指向中断处理程序
	void			*dev_id;
	void __percpu		*percpu_dev_id;
	struct irqaction	*next;//用于实现共享的IRQ处理程序，几个irqaction实例聚集到一个链表中。链表的所有元素都
//必须处理同一IRQ编号（处理不同编号的实例，位于irq_desc数组中不同的位置）,在发生一个共享中断时，内核扫描该链表找出中断实际上的来源设备。
	irq_handler_t		thread_fn;
	struct task_struct	*thread;
	unsigned int		irq;
	unsigned int		flags;<interrupt.h>中定义了
	unsigned long		thread_flags;
	unsigned long		thread_mask;
	const char		*name;
	struct proc_dir_entry	*dir;
} ____cacheline_internodealigned_in_smp;
```
name和dev_id唯一地标识一个中断处理程序。 name是一个短字符串，用于标识设备（例如，
“e100”、“ncr53c8xx”，等等），而dev_id是一个指针，指向在所有内核数据结构中唯一标识了该设
备的数据结构实例，例如网卡的net_device实例。

上面的flag:
```cpp
/*
 * These flags used only by the kernel as part of the
 * irq handling routines.
 *
 * IRQF_SHARED - allow sharing the irq among several devices表示有多于一个设备使用该IRQ电路。
 * IRQF_PROBE_SHARED - set by callers when they expect sharing mismatches to occur
 * IRQF_TIMER - Flag to mark this interrupt as timer interrupt
 * IRQF_PERCPU - Interrupt is per cpu
 * IRQF_NOBALANCING - Flag to exclude this interrupt from irq balancing
 * IRQF_IRQPOLL - Interrupt is used for polling (only the interrupt that is
 *                registered first in an shared interrupt is considered for
 *                performance reasons)
 * IRQF_ONESHOT - Interrupt is not reenabled after the hardirq handler finished.
 *                Used by threaded interrupts which need to keep the
 *                irq line disabled until the threaded handler has been run.
 * IRQF_NO_SUSPEND - Do not disable this IRQ during suspend.  Does not guarantee
 *                   that this interrupt will wake the system from a suspended
 *                   state.  See Documentation/power/suspend-and-interrupts.txt
 * IRQF_FORCE_RESUME - Force enable it on resume even if IRQF_NO_SUSPEND is set
 * IRQF_NO_THREAD - Interrupt cannot be threaded
 * IRQF_EARLY_RESUME - Resume IRQ early during syscore instead of at device
 *                resume time.
 * IRQF_COND_SUSPEND - If the IRQ is shared with a NO_SUSPEND user, execute this
 *                interrupt handler after suspending interrupts. For system
 *                wakeup devices users need to implement wakeup detection in
 *                their interrupt handlers.
 */
#define IRQF_SHARED		0x00000080
#define IRQF_PROBE_SHARED	0x00000100
#define __IRQF_TIMER		0x00000200
#define IRQF_PERCPU		0x00000400
#define IRQF_NOBALANCING	0x00000800
#define IRQF_IRQPOLL		0x00001000
#define IRQF_ONESHOT		0x00002000
#define IRQF_NO_SUSPEND		0x00004000
#define IRQF_FORCE_RESUME	0x00008000
#define IRQF_NO_THREAD		0x00010000
#define IRQF_EARLY_RESUME	0x00020000
#define IRQF_COND_SUSPEND	0x00040000

#define IRQF_TIMER		(__IRQF_TIMER | IRQF_NO_SUSPEND | IRQF_NO_THREAD)

```


#### 中断处理的相关操作：
+ 设置控制器硬件：
irq.h:
```cpp
/* Set/get chip/data for an IRQ: */
extern int irq_set_chip(unsigned int irq, struct irq_chip *chip);
extern int irq_set_handler_data(unsigned int irq, void *data);
extern int irq_set_chip_data(unsigned int irq, void *data);
extern int irq_set_irq_type(unsigned int irq, unsigned int type);
extern int irq_set_msi_desc(unsigned int irq, struct msi_desc *entry);
extern int irq_set_msi_desc_off(unsigned int irq_base, unsigned int irq_offset,
				struct msi_desc *entry);
```
+ 初始化和分配irq:
1) 注册irq：
```cpp
kernel/irq/manage.c
int request_irq(unsigned int irq,
irqreturn_t handler,
unsigned long irqflags, const char *devname, void *dev_id)
往往在设备驱动程序初始化时会做这个request_irq操作把处理程序注册进去；
setup_irq
```
2）释放irq:
free_irq

3) 注册中断：

+ 中断到达后：找到正确的中断处理程序的通用函数：do_IRQ,此时若处理器在用户态，需要切换到核心态；
linux中do_IRQ是和体系结构相关的函数：
例如：
```cpp
AMD:
irq.c:
do_IRQ
  --> set_irq_regs
  --> irq_enter
  --> generic_handle_irq -->.... handle_irq
  --> irq_exit
  --> set_irq_regs
```
关键处理：
handle_irq_event
```cpp
irqreturn_t handle_irq_event(struct irq_desc *desc)
{
	struct irqaction *action = desc->action;
	irqreturn_t ret;

	desc->istate &= ~IRQS_PENDING;
	irqd_set(&desc->irq_data, IRQD_IRQ_INPROGRESS);
	raw_spin_unlock(&desc->lock);

	ret = handle_irq_event_percpu(desc, action);//--->res = action->handler(irq, action->dev_id);

	raw_spin_lock(&desc->lock);
	irqd_clear(&desc->irq_data, IRQD_IRQ_INPROGRESS);
	return ret;
}

```
+ IRQ栈：通过内核栈处理不够，所以又衍生了用于硬件IRQ处理的栈和用于软件IRQ处理的栈；
常规的内核栈对每个进程都会分配，而这两个额外的栈是针对各CPU分别分配的。在硬件中断发
生时（或处理软中断时），内核需要切换到适当的栈。
```cpp
下列数组提供了指向额外的栈的指针：
arch/x86/kernel/irq_32.c
static union irq_ctx *hardirq_ctx[NR_CPUS] __read_mostly;
static union irq_ctx *softirq_ctx[NR_CPUS] __read_mostly;

用作栈的数据结构并不复杂：
arch/x86/kernel/irq_32.c
union irq_ctx {
struct thread_info tinfo;
u32 stack[THREAD_SIZE/sizeof(u32)];
};
```

+ 关于中断处理程序：
1) 中断是异步执行的。换句话说，它们可以在任何时间发生。
2) 例如，对网络驱动程序来说，不能将接收的数据直接转发到等待的应用程序。毕竟，内核无法确定等待数据的应用程序此时是否在运行（事实上，这种可能性很低）。
3) 中断上下文中不能调用调度器。因而不能自愿地放弃控制权。
4) 处理程序例程不能进入睡眠状态。
中断处理程序类型：irqreturn_t (*handler)(int irq, void *dev_id, struct pt_regs *regs)
一个例子：
```cpp
一个例子：tg3.c这个有napi形式的，故用这个：
```c
module_init(tg3_init);
static int __init tg3_init(void)
{
	return pci_module_init(&tg3_driver);
}

static struct pci_driver tg3_driver = {
	.name		= DRV_MODULE_NAME,
	.id_table	= tg3_pci_tbl,
	.probe		= tg3_init_one,
	.remove		= __devexit_p(tg3_remove_one),
	.suspend	= tg3_suspend,
	.resume		= tg3_resume
};

static int __devinit tg3_init_one(struct pci_dev *pdev,
				  const struct pci_device_id *ent)
{
	static int tg3_version_printed = 0;
	unsigned long tg3reg_base, tg3reg_len;
	struct net_device *dev;
	struct tg3 *tp;
	int i, err, pci_using_dac, pm_cap;
                    ...
//在init中没有申请中断，直到open时才申请：
static int tg3_open(struct net_device *dev)
{
	struct tg3 *tp = dev->priv;
	int err;
                    ...
                    err = request_irq(dev->irq, tg3_interrupt,
			  SA_SHIRQ, dev->name, dev);
                   
//来看中断处理程序：
static irqreturn_t tg3_interrupt(int irq, void *dev_id, struct pt_regs *regs)
{
	struct net_device *dev = dev_id;
	struct tg3 *tp = dev->priv;
                    //这里使用新式接口napi:
	if (likely(tg3_has_work(dev, tp)))
	       netif_rx_schedule(dev);		/* schedule NAPI poll */
```

#### 重要的中断处理程序：
+ 电流处理： typedef void fastcall (*irq_flow_handler_t)(unsigned int irq,
struct irq_desc *desc);

+ 边沿触发中断
handle_edge_irq，chip.c

+ 电平触发中断：
handle_level_irq,



### 软件中断
用于有效实现内核中的延期操作。完全用软件来实现，但是运作方式和硬件中断类似；softIRQ;
不管是哪种体系结构的do_IRQ,其结尾都会去处理所有待决的软中断，确保软中断可以定期处理；

#### 软中断表
一个包含也许是32个softirq_action类型的数据表；
interrupt.h:
```cpp
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};
```

#### 软中断的流程：
+ 先注册，内核才能执行：
open_softirq函数用于注册： 就是往上述表中填入action
kernel/softirq.c:
```cpp
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```
各个软中断都有一个唯一的编号，这表明软中断是相对稀缺的资源，使用其必须谨慎，不能由各种设备驱动程序和内核组件随意使用。
这个数组也是被定义在softirq.c中；
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
可以看枚举知道先有的以这种方式注册的软中断：
```cpp
enum
{
	HI_SOFTIRQ=0,
	TIMER_SOFTIRQ,
	NET_TX_SOFTIRQ, 网络的发送
	NET_RX_SOFTIRQ, 网络接收
	BLOCK_SOFTIRQ,块处理
	BLOCK_IOPOLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,调度器
	HRTIMER_SOFTIRQ,
	RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

	NR_SOFTIRQS
};

```
只有重要的情况才能用软中断，其他可以使用tasklet,工作队列或内核定时器的方式来达到延期处理的目的；
+ 发起中断：
硬件中断需要通过硬件设备触发中断控制器向cpu发起中断，然后进入中断处理程序处理，软中断没有这个机制，需要通过软件的方式发起中断：
void raise_softirq(unsigned int nr) 用于引发一个软中断；
```cpp
void raise_softirq(unsigned int nr)
{
	unsigned long flags;

	local_irq_save(flags);
	raise_softirq_irqoff(nr);
	local_irq_restore(flags);
}
/*
 * This function must run with irqs disabled!
 */
inline void raise_softirq_irqoff(unsigned int nr)
{
	__raise_softirq_irqoff(nr);

	/*
	 * If we're in an interrupt or softirq, we're done
	 * (this also catches softirq-disabled code). We will
	 * actually run the softirq once we return from
	 * the irq or softirq.
	 *
	 * Otherwise we wake up ksoftirqd to make sure we
	 * schedule the softirq soon.
	 */
	if (!in_interrupt())
		wakeup_softirqd();
}

//如果不在中断上下文调用raise_softirq，则调用wakeup_softirqd来唤醒软中断守护进程，这
//是开启软中断处理的两个可选方法之一。 
```
该函数设置各CPU变量irq_stat[smp_processor_id].__softirq_pending中的对应比特位。
该函数将相应的软中断标记为执行，但这个执行是延期执行。通过使用特定于处理器的位图，内核确
保几个软中断（甚至是相同的）可以同时在不同的CPU上执行。



+ 流程：
在注册了软中断后，若引发软中断且处理软中断的handler时，就会匹配到对应的函数，进行处理；
软中断处理被归结到do_softirq函数：
```cpp
do_softirq
 -->local_softirq_pending
 -->__do_softirq
 -->h->action--循环处理
 --> local_softirq_pending且重启次数没有超过次数->重启软中断
```
因为软中断大部分是用来延迟硬中断的后续处理，所以需要一个软中断守护进程来进行异步执行软中断，一方面不和硬中断冲突，另一方面确保软中断的及时执行；
系统为每个处理器都分配了自身的守护进程： ksoftirqd，为软中断守护进程；
所以实际上raise_softirq是为了唤醒这个守护进程，进行处理软件中断，进行及时处理；


+ 软中断守护进程：ksoftirqd
软中断守护进程的任务是，与其余内核代码异步执行软中断。为此，系统中的每个处理器都分配了自身的守护进程，名为ksoftirqd
内核中有两处调用wakeup_softirqd唤醒了该守护进程。
1) 在do_softirq中，如前所述。
2) 在raise_softirq_irqoff末尾。该函数由raise_softirq在内部调用，如果内核当前停用了
中断，也可以直接使用。
唤醒函数本身只需要几行代码。首先，借助于一些宏，从一个各CPU变量读取指向当前CPU软中
断守护进程的task_struct的指针。如果该进程当前的状态不是TASK_RUNNING，则通过wake_up_
process将其放置到就绪进程的列表末尾（参见第2章）。尽管这并不会立即开始处理所有待决软中断，
但只要调度器没有更好的选择，就会选择该守护进程（优先级为19）来执行。
(1) 软中断守护进程的初始化和启动：
在系统启动时用initcall机制调用init后，就创建了系统的软中断守护进程，初始化后，各个守护进程进行无限循环：
```cpp
kernel/softirq.c
static int ksoftirqd(void * __bind_cpu)
...
while (!kthread_should_stop()) {
if (!local_softirq_pending()) {
schedule();
}
__set_current_state(TASK_RUNNING);
while (local_softirq_pending()) {
do_softirq();
cond_resched();
}
set_current_state(TASK_INTERRUPTIBLE);
}
...
}
```
### tasklet ,另一种轻量的软中断
软中断是将操作推迟到未来时刻执行的最有效的方法。但该延期机制处理起来非常复杂。因为多
个处理器可以同时且独立地处理软中断，同一个软中断的处理程序例程可以在几个CPU上同时运行。
对软中断的效率来说，这是一个关键，多处理器系统上的网络实现显然受惠于此。

但处理程序例程的设计必须是完全可重入且线程安全的。 另外， 临界区必须用自旋锁保护（或其他IPC机制），
而这需要大量审慎的考虑。

tasklet是“小进程”，执行一些迷你任务，对这些任务使用全功能进程可能比较浪费
#### 创建tasklet:
各个tasklet的中枢数据结构称作tasklet_struct，
```cpp
interrupt.h:
struct tasklet_struct
{
	struct tasklet_struct *next;//用于建立tasklet_struct实例的链表。这容许几个任务排队执行。
	unsigned long state;//state表示任务的当前状态，类似于真正的进程。但只有两个选项，分别由state中的一个比特
//位表示，这也是二者可以独立设置/清除的原因。
    /*
    在tasklet注册到内核，等待调度执行时，将设置TASKLET_STATE_SCHED。
    TASKLET_STATE_RUN表示tasklet当前正在执行。
    第二个状态只在SMP系统上有用。用于保护tasklet在多个处理器上并行执行。
    */
	atomic_t count;
    //原子计数器count用于禁用已经调度的tasklet。如果其值不等于0，在接下来执行所有待决的tasklet时，将忽略对应的tasklet。
	void (*func)(unsigned long);//执行函数地址
	unsigned long data;//data用作该函数执行时的参数。
};

```
tasklet_vec 是一个pcpu变量，也就是每一个cpu上上的tasklet组成一个list.
```cpp
struct tasklet_head {
    struct tasklet_struct *head;
    struct tasklet_struct **tail;
};
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
```

#### 注册tasklet
要使用它，首先必须注册它；
```cpp
void tasklet_init(struct tasklet_struct *t,
		  void (*func)(unsigned long), unsigned long data)
{
	t->next = NULL;
	t->state = 0;
	atomic_set(&t->count, 0);
	t->func = func;
	t->data = data;
}
```
#### 执行：
tasklet基于软中断实现，它们总是在处理软中断时执行。
tasklet关联到TASKLET_SOFTIRQ软中断。因而，调用raise_softirq(TASKLET_SOFTIRQ)，即可
在下一个适当的时机执行当前处理器的tasklet。
interrupt.h:
```cpp
static inline void tasklet_schedule(struct tasklet_struct *t)
{
	if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
		__tasklet_schedule(t);
}
void __tasklet_schedule(struct tasklet_struct *t)
{
	unsigned long flags;

	local_irq_save(flags);
	t->next = NULL;
	*__this_cpu_read(tasklet_vec.tail) = t;
	__this_cpu_write(tasklet_vec.tail, &(t->next));
	raise_softirq_irqoff(TASKLET_SOFTIRQ);
	local_irq_restore(flags);
}
```

内核使用tasklet_action作为该软中断的action函数
该函数首先确定特定于CPU的链表，其中保存了标记为将要执行的各个tasklet。它接下来将表头
重定向到函数局部的一个数据项，相当于从外部公开的链表删除了所有表项。接下来，函数在以下循
环中逐一处理各个tasklet：
```cpp
static void tasklet_action(struct softirq_action *a)
{
	struct tasklet_struct *list;

	local_irq_disable();
	list = __this_cpu_read(tasklet_vec.head);
	__this_cpu_write(tasklet_vec.head, NULL);
	__this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
	local_irq_enable();

	while (list) {
		struct tasklet_struct *t = list;

		list = list->next;

		if (tasklet_trylock(t)) {
			if (!atomic_read(&t->count)) {
				if (!test_and_clear_bit(TASKLET_STATE_SCHED,
							&t->state))
					BUG();
				t->func(t->data);
				tasklet_unlock(t);
				continue;
			}
			tasklet_unlock(t);
		}

		local_irq_disable();
		t->next = NULL;
		*__this_cpu_read(tasklet_vec.tail) = t;
		__this_cpu_write(tasklet_vec.tail, &(t->next));
		__raise_softirq_irqoff(TASKLET_SOFTIRQ);
		local_irq_enable();
	}
}
```
在while循环中执行tasklet，类似于处理软中断使用的机制。
因为一个tasklet只能在一个处理器上执行一次，但其他的tasklet可以并行运行，所以需要特定于
tasklet 的 锁 。 state 状 态 用 作 锁 变 量 。 在 执 行 一 个 tasklet 的 处 理 程 序 函 数 之 前 ， 内 核 使 用
tasklet_trylock检查tasklet的状态是否为TASKLET_STATE_RUN。换句话说，它是否已经在系
统的另一个处理器上运行：
```cpp
<interrupt.h>
static inline int tasklet_trylock(struct tasklet_struct *t)
{
return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}
```
除了普通的tasklet之外，内核还使用了另一种tasklet，它具有“较高”的优先级。除以下修改之
外，其实现与普通的tasklet完全相同。
1) 使用HI_SOFTIRQ作为软中断，而不是TASKLET_SOFTIRQ，相关的action函数是tasklet_
hi_action。
2) 注册的tasklet在CPU相关的变量tasklet_hi_vec中排队。这是使用tasklet_hi_schedule完
成的。
在这里，“较高优先级”是指该软中断的处理程序HI_SOFTIRQ在所有其他处理程序之前执行，
尤其是在构成了软中断活动主体的网络处理程序之前执行。
当前，大部分声卡驱动程序都利用了这一选项，因为操作延迟时间太长可能损害音频输出的音质。
而用于高速传输的网卡也可以得益于该机制




### 等待队列和完成量；
等待队列（ wait queue）用于使进程等待某一特定事件发生，而无须频繁轮询。进程在等待期间
睡眠，在事件发生时由内核自动唤醒。 完成量（ completion）机制基于等待队列，内核利用该机制等
待某一操作结束。这两种机制使用得都比较频繁，主要用于设备驱动程序，

#### 等待队列：
##### 数据结构：
每个等待队列都有一个队列头：
wait.h:
```cpp
struct __wait_queue_head {
	spinlock_t		lock;
	struct list_head	task_list;
};
typedef struct __wait_queue_head wait_queue_head_t;
```
因为等待队列也可以在中断时修改，在操作队列之前必须获得一个自旋锁lock。
task_list是一个双链表，用于实现双链表最擅长表示的结构，即队列。

队列中的成员：
```cpp
struct __wait_queue {
	unsigned int		flags; //表示等待进程想要被独占地唤醒：则用WQ_FLAG_EXCLUSIVE
	void			*private;//指向等待进程的task_struct实例；
	wait_queue_func_t	func;//调用func，唤醒等待进程
	struct list_head	task_list;//用作链表元素，将wait_queue_t实例放入等待队列中；
};
```
等待队列的使用分为如下两部分。
(1) 为使当前进程在一个等待队列中睡眠，需要调用wait_event函数（或某个等价函数，在下文
讨论）。进程进入睡眠，将控制权释放给调度器。
内核通常会在向块设备发出传输数据的请求后，调用该函数。因为传输不会立即发生，而在此期
间又没有其他事情可做，所以进程可以睡眠，将CPU时间让给系统中的其他进程。
(2) 在内核中另一处，就我们的例子而言，是来自块设备的数据到达后，必须调用wake_up函数（或
某个等价函数，将在下文讨论）来唤醒等待队列中的睡眠进程。
在使用wait_event使进程睡眠之后，必须确保在内核中另一处有一个对应的wake_up调用
##### 使进程睡眠：
add_wait_queue函数用于将一个进程增加到等待队列，该函数在获得必要的自旋锁后，将工作
委托给__add_wait_queue：
```cpp
<wait.h>
static inline void __add_wait_queue(wait_queue_head_t *head, wait_queue_t *new)
{
list_add(&new->task_list, &head->task_list);
}
```
在将新进程统计到等待队列时，除了使用标准的list_add链表函数，没有其他工作需要做。
内核还提供了add_wait_queue_exclusive函数。它的工作方式与add_wait_queue相同，但将
进程插入在队列尾部，并将其标志设置为WQ_EXCLUSIVE

另一种方法： prepare_to_wait也是用来使进程睡眠，比上述方法多一个参数，表示进程状态；

初始化一个动态分配的wait_queue_t实例：
```cpp
static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
{
	q->flags	= 0;
	q->private	= p;
	q->func		= default_wake_function;
}
```
或者DEFINE_WAIT创建wait_queue_t的静态实例，它可以自动初始化：

```cpp
#define DEFINE_WAIT_BIT(name, word, bit)				\
	struct wait_bit_queue name = {					\
		.key = __WAIT_BIT_KEY_INITIALIZER(word, bit),		\
		.wait	= {						\
			.private	= current,			\
			.func		= wake_bit_function,		\
			.task_list	=				\
				LIST_HEAD_INIT((name).wait.task_list),	\
		},							\
	}
```

使进程睡眠，通常不直接调用add_wait_queue,而是调用wait_event；
```cpp
#define __wait_event(wq, condition)					\
	(void)___wait_event(wq, condition, TASK_UNINTERRUPTIBLE, 0, 0,	\
			    schedule())#define ___wait_event(wq, condition, state, exclusive, ret, cmd)	\
({									\
	__label__ __out;						\
	wait_queue_t __wait;						\
	long __ret = ret;	/* explicit shadow */			\
									\
	INIT_LIST_HEAD(&__wait.task_list);				\
	if (exclusive)							\
		__wait.flags = WQ_FLAG_EXCLUSIVE;			\
	else								\
		__wait.flags = 0;					\
									\
	for (;;) {							\
		long __int = prepare_to_wait_event(&wq, &__wait, state);\
									\
		if (condition)						\
			break;						\
									\
		if (___wait_is_interruptible(state) && __int) {		\
			__ret = __int;					\
			if (exclusive) {				\
				abort_exclusive_wait(&wq, &__wait,	\
						     state, NULL);	\
				goto __out;				\
			}						\
			break;						\
		}							\
									\
		cmd;							\
	}								\
	finish_wait(&wq, &__wait);					\
__out:	__ret;								\
})
每次进程被唤醒时，内核都会检查指定的条件是否满足，如果条件满足则
退出无限循环。否则，将控制转交给调度器，进程再次睡眠。
```
在条件满足时， finish_wait将进程状态设置回TASK_RUNNING，并从等待队列的链表移除对应的项。 
除了wait_event之外，内核还定义了其他几个函数，可以将当前进程置于等待队列中。其实现实际上等同于sleep_on：
```cpp
<wait.h>
#define wait_event_interruptible(wq, condition)
#define wait_event_timeout(wq, condition, timeout) { ... }
#define wait_event_interruptible_timeout(wq, condition, timeout)
```
1) wait_event_interruptible使用的进程状态为TASK_INTERRUPTIBLE。因而睡眠进程可以通
过接收信号而唤醒。
2) wait_event_timeout等待满足指定的条件，但如果等待时间超过了指定的超时限制（按jiffies
指定）则停止。这防止了进程永远睡眠。
3) wait_event_interruptible_timeout使进程睡眠，但可以通过接收信号唤醒。它也注册了
一个超时限制。从内核采用的命名方式来看，一般不会有出人意料之处！


##### 唤醒进程：
```cpp
<wait.h>
#define wake_up(x) __wake_up(x, TASK_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE, 1, NULL)
#define wake_up_nr(x, nr) __wake_up(x, TASK_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE, nr, NULL)
#define wake_up_all(x) __wake_up(x, TASK_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE, 0, NULL)
#define wake_up_interruptible(x) __wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)
#define wake_up_interruptible_nr(x, nr) __wake_up(x, TASK_INTERRUPTIBLE, nr, NULL)
#define wake_up_interruptible_all(x) __wake_up(x, TASK_INTERRUPTIBLE, 0, NULL)
在获得了用于保护等待队列首部的锁之后， _wake_up将工作委托给_wake_up_common。

kernel/sched.c
static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
int nr_exclusive, int sync, void *key)
{
wait_queue_t *curr, *next;
...
q用于选定等待队列，而mode指定进程的状态，用于控制唤醒进程的条件。 nr_exclusive表示
将要唤醒的设置了WQ_FLAG_EXCLUSIVE标志的进程的数目。
内核接下来遍历睡眠进程，并调用其唤醒函数func：
kernel/sched.c
list_for_each_safe(curr, next, &q->task_list, task_list) {
unsigned flags = curr->flags;
if (curr->func(curr, mode, sync, key) &&
(flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
```

这里会反复扫描链表，直至没有更多进程需要唤醒，或已经唤醒的独占进程的数目达到了
nr_exclusive。该限制用于避免所谓的惊群（ thundering herd）问题。如果几个进程在等待独占访问
某一资源，那么同时唤醒所有等待进程是没有意义的，因为除了其中一个之外，其他进程都会再次睡
眠。 nr_exclusive推广了这一限制。
最常使用的wake_up函数将nr_exclusive设置为1，确保只唤醒一个独占访问的进程


#### 完成量
完成量和信号量有相似之处，但是都是基于等待队列实现的；
```cpp
<completion.h>
struct completion {
unsigned int done;
wait_queue_head_t wait;
};
```
init_completion初始化一个动态分配的completion实例，而DECLARE_COMPLETION宏用来建立
该数据结构的静态实例。
进程可以用wait_for_completion添加到等待队列，进程在其中等待（以独占睡眠状态），直至
请求被内核的某些部分处理。这函数需要一个completion实例作为参数：
```cpp
<completion.h>
void wait_for_completion(struct completion *);
int wait_for_completion_interruptible(struct completion *x);
unsigned long wait_for_completion_timeout(struct completion *x,
unsigned long timeout);
unsigned long wait_for_completion_interruptible_timeout(
struct completion *x, unsigned long timeout);
```
在请求由内核的另一部分处理之后，必须调用complete或complete_all来唤醒等待的进程。因
为每次调用只能从完成量的等待队列移除一个进程，对n个等待进程来说，必须调用该函数n次。 另一
方面， complete_all将唤醒所有等待该完成的进程。 complete_and_exit是一个小的包装器，首先
调用complete，接下来调用do_exit结束内核线程。
```cpp
<completion.h>
void complete(struct completion *);
void complete_all(struct completion *);
kernel/exit.c
NORET_TYPE void complete_and_exit(struct completion *comp, long code);
```

#### 工作队列：
工作队列是将操作延期执行的另一种手段：因为它们是通过守护进程在用户上下文执行，函数可
以睡眠任意长的时间，这与内核是无关的。在内核版本2.5开发期间，设计了工作队列，用以替换此前
使用的keventd机制。

每个工作队列都有一个数组，数组项的数目与系统中处理器的数目相同。每个数组项都列出了将
延期执行的任务。  
对每个工作队列来说，内核都会创建一个新的内核守护进程，延期任务使用上文描述的等待队列
机制，在该守护进程的上下文中执行。
+ 创建新工作队列：
  新的工作队列通过调用create_workqueue或create_workqueue_singlethread函数来创建。前
一个函数在所有CPU上都创建一个工作线程，而后者只在系统的第一个CPU上创建一个线程。
