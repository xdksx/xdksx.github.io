---
title: cpp_debug_layout_static
date: 2018-05-20 18:54:58
tags: cpp_memory
categories: c&cpp
---
###  c compile time memory layout
　　标题：从一个简单的程序开始，介绍几种间接或直接debug出c程序内存布局的方法：

　　概述：首先介绍几个概念：<!-- more -->
　　程序从源代码编译之后，即经过预处理，编译，汇编，连接的过程后生成一个可执行程序，在linux为elf,此时程序还是静态的，没有被执行，可以说是编译期或者静态的时候；
　　当程序在终端被运行，此时会将pc指向main开始执行程序，并一步一步根据程序需要分配内存，栈区，堆区等等，此时为动态期；成为进程或线程； 
      注意：c语言经过词法分析语法分析等，期间有符号表等的生成（可能存储着和struct，class和定义的类型基本类型等信息），之后生成汇编语言代码不会体现类型信息如无法在其中见到struct的定义代码，并且char定义的变量的值被转化为ASICC码　。汇编代码中体现struct的只有其变量的偏移等等；至于它如何知道，是通过符号表得到的，但是啊生成汇编后不需要符号表了。。汇编执行的过程中并无太多变量相关的内容，之后将变量的值存入某个内存位置的语句；
   

　　本文主要探索通过linux下的工具如size gdb等深入学习c程序静态和动态下在内存中的分布和执行情况，以进一步理解c

 

前言：
一个源代码通过编译后生成一个目标文件.o它是一个elf　relocatable文件,以下.o文件指此类，elf文件为executable文件
.o文件(elf)文件的结构： 通过objdump -h xx.o可以查看到

              other data  ...
              .comment    offset 0x000000c6.
              .rodata      ...　　　//const & str 常量
              .data
              .text
              .elf header

一个elf文件的结构是这样的：可以通过readelf来看：
	　　　　　　　　elfheader　　　　　　　　文件头包含了平台信息：/usr/include/elf.h
	        .text　　　　　　　　　　　　段表　各个段：应用程序也可以自己定义段，和指定变量在哪个段
	        .data
	        .bss
	        ..
	        other sections
	        section header table　　　　
	        string tables　　　　　　　　重定位表和字符串表
	        symbol tables　　　　　　　　 符号表是链接的接口
 符号表是链接的接口：符号修饰和函数有很大关系，涉及调用的，其中c的符号比较简单，c++为了冲突处理，采用了命名空间，又包含类，所以符号比较复杂
 可以通过readelf -s　xx.o查看；且加了签名机制，将函数名又做进一步转换（类似乱码），减少冲突
  强符号和弱符号（强引用和若引用），调试信息　－g
 扩展：
 １）比如应用程序可以增加非系统保留字的段如：添加一个music段：当elf运行起来后可以读取这个段播放音乐；但是自定义的段名不能使用.作为前缀；
 如objcopy工具将图片段加入：objcopy -I binary -O elf32-i386 -B  i386 image.jpg image.o
   objdump -ht image.o
   可以看到图片在内存中的起始地址等，可以在程序中直接声明后使用他们
   具体看文档
２）自定义段：__attribute__((section("FOO"))) int global=24;
           __attribute__((section("BAR"))) void  foo(){}  既可以把变量或函数放入该段中　段名：BAR FOO



正文：

#### 0、先从几个命令：
1)size filename:查看elf或.o文件中各个段大小：
  text	   data	    bss	    dec	    hex	filename
     74	      0	      0	     74	     4a	simplest.o
　　代码段　　　数据段　　
代码段：只读，可共享，代码中的常量数据在编译时在代码段中分配空间，如int x=3;中的3及const 声明的常量和字符串常量 ,其他数据只引用地址
数据段：为全局初始化数据区和静态数据区：已经初始化的全局变量和静态变量（全局和局部）
bss:为初始化数据取，存放为初始化的全局变量和静态变量，但在开始执行时会被内核赋值０或null

2)其他工具：
 readelf -a simplest.o
 可以看到更清楚的段信息
 objdump -t simplest
 objdump -h simplest.o   替代size可以看到更多信息
 
此时显示出来的一些地址并不是装载后的地址

objdump -s -d xx.o:　　-s 16进制，-d反汇编　　－－－查看代码段
objdump -s -x -d xx.o:   --查看数据段和rodata

3)查看目标文件文件属性如relocatable --.o/executable --elf/share object --.so..
 file xxx


#### １、从最简单的程序开始：simplest.c
	int main()
	{
	   return 0;
	}
	
将它进行预处理：gcc -E simplest.c 可以看到没有引用到其他头文件：
	# 1 "simplest.c"
	# 1 "<built-in>"
	# 1 "<command-line>"
	# 1 "/usr/include/stdc-predef.h" 1 3 4
	# 1 "<command-line>" 2
	# 1 "simplest.c"
	int main()
	 {
	  return 0;
	 }
	 
将它进行汇编：gcc -S simplest.c
		.file	"simplest.c"
		.text
		.globl	main
		.type	main, @function
	main:
	.LFB0:
		.cfi_startproc
		pushq	%rbp
		.cfi_def_cfa_offset 16
		.cfi_offset 6, -16
		movq	%rsp, %rbp
		.cfi_def_cfa_register 6
		movl	$0, %eax
		popq	%rbp
		.cfi_def_cfa 7, 8
		ret
		.cfi_endproc
	.LFE0:
		.size	main, .-main
		.ident	"GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.9) 5.4.0 20160609"
		.section	.note.GNU-stack,"",@progbits
	


生成.o文件：gcc -c simplest.c
并通过file xx.o查看类型
simplest.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped

通过size 查看各个段：
size simplest.o:  可以看到只有代码段，数据为空，bss为空，是比较纯净的

text	   data	    bss	    dec	    hex	filename
     67	      0	      0	     67	     43	simplest.o
     
接着编译成elf:gcc -o simplest simplest.c
file simplest
simplest: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, 
interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=
56beff50d6dfd6d435c9556630737121d59c1cc7, not stripped
可以看到连接器等信息

size simplest
  text	   data	    bss	    dec	    hex	filename
   1099	    544	      8	   1651	    673	simplest
注意这里的和.o的文件大小和分段不同，

#### ２、加入头文件和局部变量

	#include<stdio.h>
	int main()
	{
	  int locala;
	  int localb=3;
	  return 0;
	  }
进行gcc -E simplest.c会看到很多导入的部分，从输出可以看到头文件的位置和引入的函数
汇编，可以看到分配３到内存中
		.file	"simplest.c"
		.text
		.globl	main
		.type	main, @function
	main:
	.LFB0:
		.cfi_startproc
		pushq	%rbp
		.cfi_def_cfa_offset 16
		.cfi_offset 6, -16
		movq	%rsp, %rbp
		.cfi_def_cfa_register 6
		movl	$3, -4(%rbp)
		movl	$0, %eax
		popq	%rbp
		.cfi_def_cfa 7, 8
		ret
		.cfi_endproc
	.LFE0:
		.size	main, .-main
		.ident	"GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.9) 5.4.0 20160609"
		.section	.note.GNU-stack,"",@progbits

　　生成.o文件：可以看到未改变data和bss,但是代码段变大
  text	   data	    bss	    dec	    hex	filename
     74	      0	      0	     74	     4a	simplest.o


生成elf:数据区和bss未改变，代码段也未改变？
 text	   data	    bss	    dec	    hex	filename
   1099	    544	      8	   1651	    673	simplest



#### ３、加入已经初始化的局部静态变量：
	int main()
	{
	 static int statica=3;
	..
	}
看生成的汇编：
		.file	"simplest.c"
		.text
		.globl	main
		.type	main, @function
	main:
	.LFB0:
		.cfi_startproc
		pushq	%rbp
		.cfi_def_cfa_offset 16
		.cfi_offset 6, -16
		movq	%rsp, %rbp
		.cfi_def_cfa_register 6
		movl	$3, -4(%rbp)
		movl	$0, %eax
		popq	%rbp
		.cfi_def_cfa 7, 8
		ret
		.cfi_endproc
	.LFE0:
		.size	main, .-main
		.data
		.align 4
		.type	statica.2285, @object　//新加的段
		.size	statica.2285, 4
	statica.2285:
		.long	3
		.ident	"GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.9) 5.4.0 20160609"
		.section	.note.GNU-stack,"",@progbits
		
	
.o大小：在静态数据区增加了，４　１个int的长度
 text	   data	    bss	    dec	    hex	filename
     74	      4	      0	     78	     4e	simplest.o

elf:有点费解，。。
 text	   data	    bss	    dec	    hex	filename
   1099	    548	      4	   1651	    673	simplest



#### ４、加入已经初始化的全局变量和全局静态变量

	     int golbala=6;
	    　static long gs=12;
生成的汇编
		.file	"simplest.c"
		.globl	golbala
		.data
		.align 4
		.type	golbala, @object
		.size	golbala, 4
	golbala://变量名
		.long	6
		.align 8
		.type	gs, @object
		.size	gs, 8
	gs://变量名
		.quad	12
		.text
		.globl	main
		.type	main, @function
	main:
	.LFB0:
		.cfi_startproc
		pushq	%rbp
		.cfi_def_cfa_offset 16
		.cfi_offset 6, -16
		movq	%rsp, %rbp
		.cfi_def_cfa_register 6
		movl	$3, -4(%rbp)
		movl	$0, %eax
		popq	%rbp
		.cfi_def_cfa 7, 8
		ret
		.cfi_endproc
	.LFE0:
		.size	main, .-main
		.data
		.align 4
		.type	statica.2287, @object
		.size	statica.2287, 4
	statica.2287:
		.long	3
		.ident	"GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.9) 5.4.0 20160609"
		.section	.note.GNU-stack,"",@progbits
	
size : 从４变到20:一个long 和int 4+4齐（见汇编）+8（long)+4=20
      text	   data	    bss	    dec	    hex	filename
       74	     20	      0	     94	     5e	simplest.o

elf size:548-->564  16
  text	   data	    bss	    dec	    hex	filename
   1099	    564	      4	   1667	    683	simplest


####  5 将 　　int golbala=6;
    　static long gs=12;　　倒换位置！！！！！！！！！
 则对齐成：size x.o为：１６比原来小，可以用于节省内存：
   text	   data	    bss	    dec	    hex	filename
     74	     16	      0	     90	     5a	simplest.o
　　汇编：
	.file	"simplest.c"
	.data
	.align 8
	.type	gs, @object
	.size	gs, 8
gs:
	.quad	12
	.globl	golbala
	.align 4
	.type	golbala, @object
	.size	golbala, 4
golbala:
	.long	6
	.text
	.globl	main
	.type	main, @function
main:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	$3, -4(%rbp)
	movl	$0, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	main, .-main
	.data
	.align 4
	.type	statica.2287, @object
	.size	statica.2287, 4
statica.2287:
	.long	3
	.ident	"GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.9) 5.4.0 20160609"
	.section	.note.GNU-stack,"",@progbits
elf文件的也变小：但bss变大，费解。。
 text	   data	    bss	    dec	    hex	filename
   1099	    560	      8	   1667	    683	simplest

#### 6、加入未初始化的全局变量和全局与局部静态变量
	  1 #include<stdio.h>
	    2 static long gs=12;
	    3 int golbala=6;
	    4      
	    5     
	    6 static long gsl;
	    7 int gi;
	    8 int main()
	    9 {   
	   10      static int staticn;
	11      static int statica=3;
       12      int locala;
       13      int localb=3;
	   14      return 0;
	   15 }
汇编没有看到什么变化：                    
	.file	"simplest.c"
	.data
	.align 8
	.type	gs, @object
	.size	gs, 8
gs:
	.quad	12
	.globl	golbala
	.align 4
	.type	golbala, @object
	.size	golbala, 4
golbala:
	.long	6
	.local	gsl
	.comm	gsl,8,8
	.comm	gi,4,4
	.text
	.globl	main
	.type	main, @function
main:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	$3, -4(%rbp)
	movl	$0, %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	main, .-main
	.data
	.align 4
	.type	statica.2290, @object
	.size	statica.2290, 4
statica.2290:
	.long	3
	.local	staticn.2289
	.comm	staticn.2289,4,4
	.ident	"GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.9) 5.4.0 20160609"
	.section	.note.GNU-stack,"",@progbits


.o文件：+12 未包含未初始化的全局变量
text	   data	    bss	    dec	    hex	filename
     74	     16	     12	    102	     66	simplest.o


elf:+16 未包含未初始化的全局变量
text	   data	    bss	    dec	    hex	filename
   1099	    560	     24	   1683	    693	simplest

至此分析完c程序静态下各个段的大小。在执行时才会有堆和栈的出现；链接前后大小也不同


参考：程序员的自我修养
