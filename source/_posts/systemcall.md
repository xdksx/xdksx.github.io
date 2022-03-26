---
title: systemcall
date: 2021-04-18 00:41:27
tags: systemcall
categories: linux
---





### 系统调用概述
内核对于用户进程来讲，是一个具备多功能的黑盒子，不仅管理设备，管理内存，管理进程等，还封装了提供进程调用的各种接口，比如打开文件，写入和读取文件等等，这些接口<!--more-->
不能在用户态运行，而需要由内核统一管理，在内核态运行；所以为了能让用户进程方便的调用，顺利的进入内核态处理，并适当的恢复用户进程运行和返回结果等等，系统调用因此而生；

系统调用的实现，主要是由内核提供各种内核函数，形成内核函数库，分散在各个功能目录；
而用户进程，一般是通过类似glibc等标准库来调用系统调用内核函数的，当然也可以不同过标准库而使用一些特殊的宏来调用，如_syscall;

因为用户态和内核态的不同，堆栈不同，虚拟地址空间等不同，控制权需要在用户进程和内核之间来回传递，而参数和返回值也需要，造成了一定的复杂度；
这里需要区分标准库和系统调用的区别，调用标准库函数不一定会触发调用系统调用，当调用标准库函数需要系统调用时，由标准库去进一步调用系统调用；

### linux系统调用的两种使用方式：c库函数和直接系统调用
#### c库函数使用例子和追踪：
一个调用c库函数的典型例子如下：
```cpp
#include<stdio.h>
int main()
{
	printf("hello");
	return 0;
}

```
从上面的函数可以看到调用了标准c库的printf函数；
编译后进行系统调用跟踪：
```cpp
strace  ./main

execve("./main", ["./main"], [/* 70 vars */]) = 0
brk(0)                                  = 0x2425000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7ff4be6f0000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=86757, ...}) = 0
mmap(NULL, 86757, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7ff4be6da000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P \2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1840928, ...}) = 0
mmap(NULL, 3949248, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7ff4be10b000
mprotect(0x7ff4be2c5000, 2097152, PROT_NONE) = 0
mmap(0x7ff4be4c5000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1ba000) = 0x7ff4be4c5000
mmap(0x7ff4be4cb000, 17088, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7ff4be4cb000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7ff4be6d9000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7ff4be6d7000
arch_prctl(ARCH_SET_FS, 0x7ff4be6d7740) = 0
mprotect(0x7ff4be4c5000, 16384, PROT_READ) = 0
mprotect(0x600000, 4096, PROT_READ)     = 0
mprotect(0x7ff4be6f2000, 4096, PROT_READ) = 0
munmap(0x7ff4be6da000, 86757)           = 0
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 5), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7ff4be6ef000
write(1, "hello", 5hello)                    = 5
exit_group(0)                           = ?
+++ exited with 0 +++

```
可以看到，除开前面的标准库调用的基本框架，最后是调用了系统调用write函数，来进行打印操作；
标准库执行的大体框架，需要加载so文件，并做内存映射，为可执行，然后调用系统调用；
而系统调用，就通过一种机制调用到内核函数了

{% asset_img MMU.png MMU about %}
#### 直接使用使用例子和追踪：
```cpp
#define_NR_testsyscall 191

_syscall0(int,testsyscall)

main()
{
    testsyscall();
}
```

### 系统调用的各种标准介绍：
posix标准
system V
BSD

### 系统调用的分类和分布位置目录：
可以从内核源码中看到所有的系统调用函数接口：
```cpp
linux 2.x:
arch/i386/kernel/entry.s中
.data
575 ENTRY(sys_call_table)
576         .long sys_restart_syscall       /* 0 - old "setup()" system call, used for restarting */
577         .long sys_exit
...

3.x之后：
arch/x86/entry/syscalls/syscall_64.tbl
# 64-bit system call numbers and entry vectors
#
# The format is:
# <number> <abi> <name> <entry point>
#
# The __x64_sys_*() stubs are created on-the-fly for sys_*() system calls
#
# The abi is "common", "64" or "x32" for this file.
#
0	common	read			sys_read
1	common	write			sys_write
2	common	open			sys_open
3	common	close			sys_close
4	common	stat			sys_newstat
5	common	fstat			sys_newfstat
6	common	lstat			sys_newlstat
7	common	poll			sys_poll
8	common	lseek			sys_lseek
9	common	mmap			sys_mmap
10	common	mprotect		sys_mprotect
11	common	munmap			sys_munmap
12	common	brk			sys_brk
13	64	rt_sigaction		sys_rt_sigaction
14	common	rt_sigprocmask		sys_rt_sigprocmask
15	64	rt_sigreturn		sys_rt_sigreturn
16	64	ioctl			sys_ioctl
17	common	pread64			sys_pread64
...

```
#### 进程管理
+ 创建进程：fork,vfork,clone
+ 结束进程：exit
+ 查询： getuid等
+ 程序执行环境：personality 
+ 跟踪系统调用：ptrace
+ 优先级设置：nice
+ 设置一定的资源限制：setrlimit, getrlimit,getrusage
#### 时间操作
+ 读取和设置基于时间的内核变量：adjtimex
+ 定时器： alarm,setitimer,getitimer
+ 获取和设置系统当前时间：gettimeofday ,settimeofday
+ 睡眠： sleep,nanosleep 
+ 返回时间戳： timer

#### 信号处理
+ 设置信号处理函数：signal,sigaction
+ 检查进程当前是否有待决信号被阻塞：sigpending
+ 将进程置于等待队列上，直至某个特定（一组信号中的一个）的信号到达 :sigsuspend
+ 启用信号的阻塞机制，而getmask返回所有当前阻塞信号的列表:setmask
+ 用于向一个进程发送任何信号:kill
+ 还有一组处理实时信号的系统调用，但其对应的函数名带有前缀rt_。例如， rt_sigaction
+ 设置一个实时信号处理程序，而rt_sigsuspend将进程置于等待状态，直至某个特定（一组
+ 信号中的一个）信号到达
#### 调度
+ setpriority和getpriority分别设置和获取进程的优先级，因而是用于调度目的的关键系统调用。
 
+ 请注意， Linux不仅支持不同的进程优先级，还提供了多种调度类，以适应应用程序在时间方
  面具体的行为和需求。 sched_setscheduler和sched_getscheduler分别设置和查询调度类。
 sched_setparam和sched_getparam分别设置和查询进程的附加调度参数（当前，只使用了
 实时优先级的参数）。

+ sched_yield自愿释放CPU的控制权，即使进程当前仍然有CPU时间可用。
#### 模块
+ init_module添加一个新模块。
+ delete_module从内核移除一个模块
#### 文件系统
 一些系统调用被用作用户空间中同名实用程序的直接基础，用来创建和修改目录结构： chdir、
mkdir、 rmdir、 rename、 symlink、 getcwd、 chroot、 umask和mknod。
+ 文件和目录属性可以用chown和chmod修改。
+ 下列实用程序用于处理文件内容，其实现在标准库中，与对应的系统调用同名： open、 close、
read与readv、 write与writev、 truncate和llseek。
+ readdir和getdents读取目录结构。
+ link、 symlink和unlink创建和删除链接（或文件，如果该文件是某个硬链接的最后一个成
员）。 readlink读取链接的内容。
+ mount和umount用于文件系统的装载和卸载。
+ poll和select用于等待某些事件。
+ execve装载一个新进程，替换旧的进程。在与fork联合使用时，它会启动一个新的程序。
#### 内存管理
+ 就动态内存管理而言，最重要的调用是brk，它修改进程数据段的长度。调用了malloc或相似
函数的程序（几乎所有非平凡的代码，都符合这个条件）会频繁使用该系统调用。

+ mmap、 mmap2、 munmap和mremap执行内存映射、解除映射和重新映射操作，而mprotect控制
对虚拟内存中特定区域的访问， madvice提出对特定虚拟内存区域的使用建议。
mmap和mmap2的参数稍有不同，更多细节请参考手册页。默认情况下， GNU C库使用mmap2；
现在mmap只是一个用户层包装器函数。
根据malloc的实现，它在内部可以使用mmap或mmap2。这是可行的，因为匿名映射允许建立
没有文件作为后备存储的映射。与使用brk相比，该方法更加灵活。

+ swapon和swapoff分别启用和禁用外存储器设备上（附加）的交换区
#### 进程间通信和网络功能
+ socketcall处理网络方面的问题，用于实现套接字抽象。它管理各种类型的连接和协议，总
共实现了17种功能， 通过SYS_ACCEPT、 SYS_SENDTO等常数来区分。参数必须以指针形式传递，
指向一个与函数类型相关的用户空间结构，其中保存了所需的数据。

+ ipc与socketcall相对应，用于处理计算机本地的连接，而不是通过网络建立的连接。因为
该系统调用“只”需要实现11种功能，它使用了固定数目的参数来从用户空间向内核空间传
递数据，总共是5个。
#### 系统信息和设置
+ syslog向系统日志写入消息，并允许设置不同的优先级（根据消息的优先级不同，用户空
间工具或者向持久性的日志文件发送消息，或者直接向控制台输出消息以通知用户某些关键
情况。

+ sysinfo返回有关系统状态的信息，特别有关内存使用的统计量（物理内存、缓冲区、交换区）。
+ sysctl用于“微调”内核参数。内核现在支持大量的动态可配置选项，可以使用proc文件系
统读取和修改

#### 系统安全和能力
LSM（ Linux security modules， Linux安全模块）子系统提供了一个通用接口，支持内核在
各个位置通过挂钩调用模块函数来执行安全检查。
+ capset和capget负责设置和查询进程的能力。
+ security是一个系统调用的多路分解器，用于实现LSM


### 系统调用的实现
#### 理论上
##### 系统调用的结构：
+ 处理函数的实现：内核中实现：sys_
  1） 每个函数的名称前缀都是sys_，将该函数唯一地标识为一个系统调用
  2） 所有的处理程序函数都最多接受5个参数
  3） 所有的系统调用都在核心态执行
  在内核将控制权转移给处理程序例程后，控制流就进入了平台中立的代码，即不依赖于特定的CPU或体系结构。有少量处理程序函数是针对各个平台分别实现的
+ 调用分派和参数传递：系统调用表
1）调用分派：
系统调用由内核分配的一个编号唯一标识，所有的系统调用都由一处中枢代码处理，根据调用编号和一个静态表，将调用
分派到具体的函数。传递的参数也由中枢代码处理，这样参数的传递独立于实际的系统调用。

为容许用户态和核心态之间的切换，用户进程必须通过一条专用的机器指令，引起处理器/内核
对该进程的关注，这需要C标准库的协助。
2）参数传递：
在所有平台上，系统调用参数都是通过寄存器直接传递的，对具体的处理程序函数而言，参数与寄存器之间的映射是精确定义的。还需要一
个寄存器来定义系统调用编号，将系统调用分派给匹配的处理程序函数。比如 x86:系统调用编号通过寄存器eax传递，
而参数通过寄存器ebx、 ecx、 edx、 esi和edi传递。
3）系统调用表：eg：sys_call_table
如果一个用户空间程序调用open系统调
用，传递的系统调用编号是5。 分配器例程将编号5加到sys_call_table的基地址，得到该数组的第6
项，其中保存了sys_open的地址，这是独立于处理器的处理程序函数。在将保存在寄存器中的参数值
复制到栈上之后，内核调用处理程序例程，并切换到系统调用处理中独立于处理器的部分
用户态和内核态是使用两个不同的栈
+ 返回用户态
返回值的语义
通常，系统调用的返回值有如下约定：负值表示错误，而正值（和0）表示成功结束
在include/asm-generic/errno-base.h和include/asm-generic/errno.h中定义的符号常数
##### 如何访问用户空间
有些情况下，内核代码必须访问用户应用程序的虚拟内存。
内核忙于同步执行应用程序指派的任务。因为如下两种原因，内核必须访问应用程序的地址空间。
 如果一个系统调用需要超过6个不同的参数，它们只能借助进程内存空间中的C结构实例来传
递。系统调用将借助寄存器，将指向该结构实例的一个指针传递给内核。。。

#### 实际实现：linux系统调用实现流程 基于linux3.x后
##### glibc标准库：
查询标准库：
think@think-VirtualBox:~/source_linux/linux-lts-xenial-4.4.0/arch$ uname -a
Linux think-VirtualBox 4.4.177 #1 SMP Fri Sep 18 21:23:25 CST 2020 x86_64 x86_64 x86_64 GNU/Linux
这样就是arch要选择x86相关的代码；
基于write系统调用，来跟踪：

```cpp
glibc/sysdeps/unix/sysv/linux/write.c
1 
/* Write NBYTES of BUF to FD.  Return the number written, or -1.  */
ssize_t
__libc_write (int fd, const void *buf, size_t nbytes)
{
  return SYSCALL_CANCEL (write, fd, buf, nbytes);
}

2 
跟着这个宏，最后是：这些宏定义在：sysdep.h

#define SYSCALL_CANCEL(...) \
  ({									     \
    long int sc_ret;							     \
    if (SINGLE_THREAD_P) 						     \
      sc_ret = INLINE_SYSCALL_CALL (__VA_ARGS__); 			     \
    else								     \
      {									     \
	int sc_cancel_oldtype = LIBC_CANCEL_ASYNC ();			     \
	sc_ret = INLINE_SYSCALL_CALL (__VA_ARGS__);			     \
        LIBC_CANCEL_RESET (sc_cancel_oldtype);				     \
      }									     \
    sc_ret;								     \
  })

#define INLINE_SYSCALL_CALL(...) \
  __INLINE_SYSCALL_DISP (__INLINE_SYSCALL, __VA_ARGS__)

#define __INLINE_SYSCALL_NARGS_X(a,b,c,d,e,f,g,h,n,...) n
#define __INLINE_SYSCALL_NARGS(...) \
  __INLINE_SYSCALL_NARGS_X (__VA_ARGS__,7,6,5,4,3,2,1,0,)
#define __INLINE_SYSCALL_DISP(b,...) \
  __SYSCALL_CONCAT (b,__INLINE_SYSCALL_NARGS(__VA_ARGS__))(__VA_ARGS__)

#define __SYSCALL_CONCAT_X(a,b)     a##b
#define __SYSCALL_CONCAT(a,b)       __SYSCALL_CONCAT_X (a, b)
展开为：
__INLINE_SYSCALL_N(xxxx);//这里N是参数个数：

#define __INLINE_SYSCALL0(name) \
  INLINE_SYSCALL (name, 0)
#define __INLINE_SYSCALL1(name, a1) \
  INLINE_SYSCALL (name, 1, a1)
#define __INLINE_SYSCALL2(name, a1, a2) \
  INLINE_SYSCALL (name, 2, a1, a2)
#define __INLINE_SYSCALL3(name, a1, a2, a3) \
  INLINE_SYSCALL (name, 3, a1, a2, a3)
#define __INLINE_SYSCALL4(name, a1, a2, a3, a4) \
  INLINE_SYSCALL (name, 4, a1, a2, a3, a4)
#define __INLINE_SYSCALL5(name, a1, a2, a3, a4, a5) \
  INLINE_SYSCALL (name, 5, a1, a2, a3, a4, a5)
#define __INLINE_SYSCALL6(name, a1, a2, a3, a4, a5, a6) \
  INLINE_SYSCALL (name, 6, a1, a2, a3, a4, a5, a6)
#define __INLINE_SYSCALL7(name, a1, a2, a3, a4, a5, a6, a7) \
  INLINE_SYSCALL (name, 7, a1, a2, a3, a4, a5, a6, a7)


3
进入平台相关函数：
选x86_64: sysdeps/unix/sysv/linux/x86_64

# define INLINE_SYSCALL(name, nr, args...) \
  ({									      \
    unsigned long int resultvar = INTERNAL_SYSCALL (name, , nr, args);	      \
    if (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (resultvar, )))	      \
      {									      \
	__set_errno (INTERNAL_SYSCALL_ERRNO (resultvar, ));		      \
	resultvar = (unsigned long int) -1;				      \
      }									      \
    (long int) resultvar; })
#define INTERNAL_SYSCALL(name, err, nr, args...)			\
	internal_syscall##nr (SYS_ify (name), err, args)
也是扩展为参数个数后：
#undef internal_syscall0
#define internal_syscall0(number, err, dummy...)			\
({									\
    unsigned long int resultvar;					\
    asm volatile (							\
    "syscall\n\t"							\
    : "=a" (resultvar)							\
    : "0" (number)							\
    : "memory", REGISTERS_CLOBBERED_BY_SYSCALL);			\
    (long int) resultvar;						\
})
。。。

4 
调用了syscall函数
在syscall.S中： sysdeps/unix/sys/linux/x86_64
	.text
ENTRY (syscall)
	movq %rdi, %rax		/* Syscall number -> rax.  */
	movq %rsi, %rdi		/* shift arg1 - arg5.  */
	movq %rdx, %rsi
	movq %rcx, %rdx
	movq %r8, %r10
	movq %r9, %r8
	movq 8(%rsp),%r9	/* arg6 is on the stack.  */
	syscall			/* Do the system call.  */
	cmpq $-4095, %rax	/* Check %rax for error.  */
	jae SYSCALL_ERROR_LABEL	/* Jump to error handler if error.  */
	ret			/* Return to caller.  */

PSEUDO_END (syscall)

```


##### 内核代码中的系统调用平台相关入口函数：
总之，上面的syscall指令跳转到存储在MSR_LSTAR模型特定寄存器(长系统目标地址寄存器)中的地址。内核负责提供自己的自定义函数来处理系统调用，并在系统启动时将这个处理函数的地址写入MSR_LSTAR寄存器。自定义函数是entry_SYSCALL_64，它在arch/x86/entry/entry_64.S中定义。在arch/x86/kernel/cpu/common.c中，这个系统调用处理函数的地址在启动时被写入MSR_LSTAR寄存器。在这个文件可以看到：linux/arch/x86/entry/entry_64.S
```cpp

ENTRY(entry_SYSCALL_64)
	/*
	 * Interrupts are off on entry.
	 * We do not frame this tiny irq-off block with TRACE_IRQS_OFF/ON,
	 * it is too small to ever cause noticeable irq latency.
	 */
	SWAPGS_UNSAFE_STACK
	SWITCH_KERNEL_CR3_NO_STACK
	/*
	 * A hypervisor implementation might want to use a label
	 * after the swapgs, so that it can do the swapgs
	 * for the guest and jump here on syscall.
	 */
GLOBAL(entry_SYSCALL_64_after_swapgs)

	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp

	/* Construct struct pt_regs on stack */
	pushq	$__USER_DS			/* pt_regs->ss */
	pushq	PER_CPU_VAR(rsp_scratch)	/* pt_regs->sp */
	/*
	 * Re-enable interrupts.
	 * We use 'rsp_scratch' as a scratch space, hence irq-off block above
	 * must execute atomically in the face of possible interrupt-driven
	 * task preemption. We must enable interrupts only after we're done
	 * with using rsp_scratch:
	 */
	ENABLE_INTERRUPTS(CLBR_NONE)
	pushq	%r11				/* pt_regs->flags */
	pushq	$__USER_CS			/* pt_regs->cs */
	pushq	%rcx				/* pt_regs->ip */
	pushq	%rax				/* pt_regs->orig_ax */
	pushq	%rdi				/* pt_regs->di */
	pushq	%rsi				/* pt_regs->si */
	pushq	%rdx				/* pt_regs->dx */
	pushq	%rcx				/* pt_regs->cx */
	pushq	$-ENOSYS			/* pt_regs->ax */
	pushq	%r8				/* pt_regs->r8 */
	pushq	%r9				/* pt_regs->r9 */
	pushq	%r10				/* pt_regs->r10 */
	pushq	%r11				/* pt_regs->r11 */
	sub	$(6*8), %rsp			/* pt_regs->bp, bx, r12-15 not saved */

	ENABLE_IBRS

	testl	$_TIF_WORK_SYSCALL_ENTRY, ASM_THREAD_INFO(TI_flags, %rsp, SIZEOF_PTREGS)
	jnz	tracesys
entry_SYSCALL_64_fastpath:
#if __SYSCALL_MASK == ~0
	cmpq	$NR_syscalls, %rax
#else
	andl	$__SYSCALL_MASK, %eax
	cmpl	$NR_syscalls, %eax
#endif
	jae	1f				/* return -ENOSYS (already in pt_regs->ax) */
	sbb	%rcx, %rcx			/* array_index_mask_nospec() */
	and	%rcx, %rax
	movq	%r10, %rcx
#ifdef CONFIG_RETPOLINE
	movq	sys_call_table(, %rax, 8), %rax
	call	__x86_indirect_thunk_rax
#else
	call	*sys_call_table(, %rax, 8) //这里选择到了系统调用表中的对应函数指针；
#endif

	movq	%rax, RAX(%rsp)
```
调用了*sys_call_table(, %rax, 8)，传入了系统调用号；


##### 内核代码平台无关函数和数组初始化；
在linux3.x之后，系统调用由sys_call_table函数指针数组管理：
初始化如下：
```cpp
实际上是通过一个数组维护的，在：
/arch/x86/entry/syscall_64.c sys_call_table array
如何初始化这个表：
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &sys_ni_syscall, 初始化的这个函数指针其实啥也没做：
    #include <asm/syscalls_64.h>
};
typedef void (*sys_call_ptr_t)(void);
asmlinkage long sys_ni_syscall(void)
{
    return -ENOSYS;
}
```
如何赋值这个表：
We can do it with a GCC compiler extension called - Designated Initializers. This extension allows us to initialize elements in non-fixed order. 
我们可以看到，在上面的初始化中，最后一行贴了#include<asm/syscalls_64.h>
就可以看到：
```cpp
/arch/x86/include/generated/asm$ vim syscalls_64.h
__SYSCALL_COMMON(0, sys_read, sys_read)
__SYSCALL_COMMON(1, sys_write, sys_write)
__SYSCALL_COMMON(2, sys_open, sys_open)
__SYSCALL_COMMON(3, sys_close, sys_close)
。。。
#define __SYSCALL_COMMON(nr, sym, compat) __SYSCALL_64(nr, sym, compat)
#define __SYSCALL_64(nr, sym, compat) [nr] = sym,
所以之后就初始化好了；
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &sys_ni_syscall,
    [0] = sys_read,
    [1] = sys_write,
    [2] = sys_open,

```
而这个文件 asm/syscalls_64.h:是在内核编译时根据syscalls目录中的脚本syscalltbl.sh和系统调用号定义文件syscall_64.tbl生成的。
syscallhdr.sh脚本用于生成unistd_64.h等文件

这里就可以看到，初始化和赋值了该系统调用数组，所以其实标准库就是最后找到了这个系统调用函数进行调用；

##### _syscall
/arch/parisc/include/asm/unistd.h 不太确定
可以看到在这个文件下：
```cpp
#define _syscall0(type,name)						\
type name(void)								\
{									\
    return K_INLINE_SYSCALL(name, 0);	                                \
}
...
#define _syscall1(type,name,type1,arg1)					\
type name(type1 arg1)							\
{									\
    return K_INLINE_SYSCALL(name, 1, arg1);	                        \
}

...
#undef K_INLINE_SYSCALL
#define K_INLINE_SYSCALL(name, nr, args...)	({			\
	long __sys_res;							\
	{								\
		register unsigned long __res __asm__("r28");		\
		K_LOAD_ARGS_##nr(args)					\
		/* FIXME: HACK stw/ldw r19 around syscall */		\
		__asm__ volatile(					\
			K_STW_ASM_PIC					\
			"	ble  0x100(%%sr2, %%r0)\n"		\
			"	ldi %1, %%r20\n"			\
			K_LDW_ASM_PIC					\
			: "=r" (__res)					\
			: "i" (SYS_ify(name)) K_ASM_ARGS_##nr   	\
			: "memory", K_CALL_CLOB_REGS K_CLOB_ARGS_##nr	\
		);							\
		__sys_res = (long)__res;				\
	}								\
	if ( (unsigned long)__sys_res >= (unsigned long)-4095 ){	\
		errno = -__sys_res;		        		\
		__sys_res = -1;						\
	}								\
	__sys_res;							\
})
```

经过这个系统调用接口，可以不用经过glibc等标准库：
eg:
https://man7.org/linux/man-pages/man2/_syscall.2.html
```cpp
 #include <stdio.h>
       #include <stdlib.h>
       #include <errno.h>
       #include <linux/unistd.h>       /* for _syscallX macros/related stuff */
       #include <linux/kernel.h>       /* for struct sysinfo */

       _syscall1(int, sysinfo, struct sysinfo *, info);

       int
       main(void)
       {
           struct sysinfo s_info;
           int error;

           error = sysinfo(&s_info);
           printf("code error = %d\n", error);
           printf("Uptime = %lds\nLoad: 1 min %lu / 5 min %lu / 15 min %lu\n"
                  "RAM: total %lu / free %lu / shared %lu\n"
                  "Memory in buffers = %lu\nSwap: total %lu / free %lu\n"
                  "Number of processes = %d\n",
                  s_info.uptime, s_info.loads[0],
                  s_info.loads[1], s_info.loads[2],
                  s_info.totalram, s_info.freeram,
                  s_info.sharedram, s_info.bufferram,
                  s_info.totalswap, s_info.freeswap,
                  s_info.procs);
           exit(EXIT_SUCCESS);
       }

   Sample output
       code error = 0
       uptime = 502034s
       Load: 1 min 13376 / 5 min 5504 / 15 min 1152
       RAM: total 15343616 / free 827392 / shared 8237056
       Memory in buffers = 5066752
       Swap: total 27881472 / free 24698880
       Number of processes = 40
```

### 实现一个系统调用；
#### 定义一个内核系统调用函数：可以通过模块传入：
```cpp
#inculde(linux/kernel.h)
#inculde(linux/module.h)
#inculde(linux/modversions.h)
#inculde(linux/sched.h)
#inculde(asm/uaccess.h)

#define _NR_testsyscall 191

extern viod *sys_call_table[];

asmlinkage int testsyscall()
{ 
    printf("hello world\n");
    return 0;
}

int init_module()
{ 
    sys_call_table[_NR_tsetsyscall]=testsyscall;
    printf("system call testsyscall() loaded success\n");
    return 0;
}

void cleanup_module()
{

}
```
所以这里也把数组初始化好了，如果不经过glibc库的话，这里已经可以用了：

```cpp
#include<linux/unistd.h>
#define_NR_testsyscall 191
_syscall0(int,testsyscall)

main()
{
    testsyscall();
}
```
如果想通过glibc调用，可以通过增加glibc代码，编译后替换所使用的glibc库：
通过ldd main可以知道链接了哪个glibc

在网上找对应的版本，修改和编译后替换，具体修改，这里暂时不提供，有兴趣自己探索；

### 追踪系统调用
#### 关于ptrace
#### 如何用ptrace追踪指定的系统调用
ptrace是一个内核提供的一个可以用来追踪进程运行情况，甚至控制进程运行行为的系统调用，并提供了对应的接口，strace和gdb就是利用ptrace制成的；
ptrace本质上是一个用于读取和修改进程地址空间中的值的工具，不能用于直接跟踪系统调用。只有从正确的位置提取出所需的信息，才能跟踪进
程并就进行的系统调用得出结论。

ptrace的系统调用： sys_ptrace ,实现：arch/arch/kernel/ptrace.c
内核源码中有四个参数：
```cpp
<syscalls.h>
asmlinkage long sys_ptrace(long request, long pid, long addr, long data);
pid标识了要追踪的进程，可以在进程开始时或者运行中；
addr,data则向内核传递了一个内存地址和附加信息；因选择的操作而不同；

request用于选择一个操作，由ptrace执行；在ptrace.h中有列出：
PTRACE_TRACEME：tracee表明自己想要被追踪，这会自动与父进程建立追踪关系，这也是唯一能被tracee使用的request，其他的request都由tracer指定。
PTRACE_ATTACH：tracer用来附着一个进程tracee，以建立追踪关系，并向其发送`SIGSTOP`信号使其暂停。
PTRACE_SEIZE：像PTRACE_ATTACH附着进程，但它不会让tracee暂停，addr参数须为0，data参数指定一位ptrace选项。
PTRACE_DETACH：解除追踪关系，tracee将继续运行。
PEEKTEXT、 PEEKDATA和PEEKUSR从进程地址空间读取数据。 PEEKUSR读取普通的CPU寄存
器和使用的任何其他调试寄存器①（当然，会根据标识符只读取一个寄存器的内容，而不是
读取整个寄存器集合的内容）。 PEEKEXT和PEEKDATA从进程的代码段和数据段读取任意字，可以向对应区域写入值，从而操作进程空间的内容；
PTRACE_SYSCALL，如果用该选项激活ptrace，那么内核将开始执行进程，直至调用一个系统调用。
。。。。

```
一个例子：
```cpp
#include <sys/ptrace.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <linux/user.h>   /* For constants
                                   ORIG_EAX etc */
int main()
{   pid_t child;
    long orig_eax;
    child = fork();
    if(child == 0) {
        ptrace(PTRACE_TRACEME, 0, NULL, NULL);
        execl("/bin/ls", "ls", NULL);
    }
    else {
        wait(NULL);
        orig_eax = ptrace(PTRACE_PEEKUSER,
                          child, 4 * ORIG_EAX,
                          NULL);
        printf("The child made a "
               "system call %ld\n", orig_eax);
        ptrace(PTRACE_CONT, child, NULL, NULL);
    }
    return 0;
}
```



### 其他：重启系统调用
当系统调用和信号发生冲突时，有问题：
以下内容，引用自 深入linux内核架构：

如果在一个进程执行系统调用时，向该进程发送一个信号，那么在处理时，二者的优先级如何分配呢？应该等到系统调用结束再处理信号，还是
中断系统调用，以便尽快将信号投递到该进程？第一种方案导致的问题显然比较少，也是比较简单的
方案。遗憾的是，只有在所有系统调用都能够快速结束、不会让进程等待太长时间的情况下，这个方
案才能正确运作（信号投递的时机，总是在进程处理完一个系统调用、返回到用户
态的时候）。情况不总是这样。系统调用不仅需要一定的执行时间，而且在最坏情况下，很可能使进
程睡眠（例如，没有数据可供读取时）。对同时发生的信号而言，这意味着信号投递的严重延迟。因
而，必须不惜任何代价防止这种情况

如果一个正在执行的系统调用被中断，内核应该向应用程序返回什么样的值？在通常的场景下，
只有两种情况：调用成功或者失败。在出错的情况下，将返回一个错误码，使用户进程能够确定错误
的原因，并适当地做出反应。倘若系统调用被中断，则发生了第三种情况：必须通知应用程序，如果
系统调用在执行期间没有被信号中断，那么系统调用已经成功结束。在这种情况下， Linux（和其他
System V变体）下将使用-EINTR常数

该过程的负面效应是很明显的。尽管该方案易于实现，但它迫使用户空间应用程序的程序员必须
明确检查所有系统调用的返回值，并在返回值为-EINTR的情况下，重新启动被中断的系统调用，直至
该调用不再被信号中断。用这种方法重启的系统调用称作可重启系统调用（ restartable system call），
该技术则称为重启（ restarting）。
该行为第一次引入是在System V UNIX中。该方案将新信号的快速投递和系统调用的中断组合起
来，但它并非是唯一的组合方式， BSD所采用的方法即可证实这一点。我们来考察BSD内核在系统调
用被信号中断时，会做出何种反应。
BSD内核将中断系统调用的执行并切换到用户态执行信号处理程序。在发生这种情况时，该系统
调用不会有返回值，内核在信号处理程序结束后将自动重启该调用。因为该行为对用户应用程序是透
明的，也不再需要重复实现对-EINTR返回值的检查和调用的重启，所以与System V方法相比，这种方
案更受程序员的欢迎。
Linux通过SA_RESTART标志支持BSD方案，可以在安装信号处理例程时按需对具体信号指定该标
志。 System V提议的机制用作默认方案，因为BSD机制偶尔会导致一些困难，如下列例子所示（取自
[ME02]第229页）。
```cpp

#include <signal.h>
#include <stdio.h>
#include <unistd.h>
volatile int signaled = 0;
void handler (int signum) {
printf("signaled called\n");
signaled = 1;
}
int main() {
char ch;
struct sigaction sigact;
sigact.sa_handler = handler;
sigact.sa_flags = SA_RESTART;
sigaction(SIGINT, &sigact, NULL);
while (read(STDIN_FILENO, &ch, 1) != 1 && !signaled);
}
```
这个简短的C程序在一个while循环中等待，直至用户通过标准输入键入了一个字符，或者程序
被SIGINT信号中断（可使用kill-INT发送该信号，也可以按键CTRL+C）。我们来考察其代码的控制
流。如果用户点了一个普通的按键，没有导致发送SIGINT，那么read将得到一个正的返回值，即读
取字符的数目。
要结束while循环，循环的控制条件必须在逻辑上为false。这里的控制条件是由逻辑与（ &&）
运算连接的两个表达式，要结束循环，需要二者之一为false，或全部为false，如下。
1 按下了一个键， read返回1，检查read返回不等于1的表达式，其值为false。
2 signaled变量设置为1，该变量的反（ !signaled）也将为false值。
这些条件意味着，程序要结束，或者需要等到键盘输入，或者需要SIGINT信号到达。
为在上述代码中应用Linux默认实现的System V行为，需要取消SA_RESTART标志的设置。换句话
说， sigact.sa_flags = SA_RESTART一行需要删除或注释掉。在这样做之后，程序将按上面的描述
运行，在按下一个键或接收到SIGINT时结束。
如果激活了BSD行为模式，而read被SIGINT信号中断，那么示例程序的情况将更为有趣。在这
种情况下，将调用信号处理程序，将signaled设置为1，并输出一个消息表示接收到了SIGINT，但程
序不会结束。为什么?在运行处理程序之后， BSD机制将重启read调用，并再次等待输入一个字符。
这种情况使得while循环控制条件中的!signaled部分无法进行求值，导致循环不能结束。因而该程
序不能通过向其发送SIGNIT信号结束，尽管在表面上，代码的语义确实如此。
