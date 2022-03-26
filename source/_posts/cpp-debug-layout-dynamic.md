---
title: cpp_debug_layout_dynamic
date: 2018-05-20 18:33:09
tags: cpp_memory
categories: c&cpp
---

### c执行期内存布局和调试：

在生成目标文件后，还没被执行时还是一个静态文件，当被执行时，可能会进行，动态链接等
１、将目标文件装入: <!-- more -->
*１）重定位－－－　放在内存哪里
 2)  等待调度执行*
这里可以使用gdb进行调试查看


一个进程是一个运行着的程序段，一个程序有可能有多个进程在运行（程序中fork).
一个进程主要包括：
   在内存中申请的空间，代码（加载的程序段，包括代码段，数据段和bss).堆，栈，以及内核进程信息结构task_struct，打开的文件，上下文信息和挂起的信号等

>栈区　　高地址到低地址
>堆区　　低地址到高地址
>bss
>数据
>代码
     
     
     
 以下将从两个维度进行对一个程序被执行成进程时，内存的情况：
 １、各种段区的内存分布
 ２、gdb 调试程序执行时的过程。－－可能涉及到汇编
 －－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
#### 1、gdb　常用的命令和用法：
1)通过gcc 编译出带调试信息的程序：gcc -o gdbtest gdbtest.c -g 回车重复上一次命令
 2)设置断点：　
b 行号
b 函数名
b 行号　if 条件
eg:
break main  /  b main
  删除断点:delete 行号

3)列出代码
l /list

4)运行，start
  跳转到断点:c/continue   r/run
  until  行号　　　运行直到该行 
5)观察变量b和地址
watch b  若变量值发生变化，则程序停止


　p/print  b  看变量值
　p/print &b　看变量地址
 i  locals
  
  
 info registers 显示所有寄存器的值
 查看特定内存位置的值如：
 print/x $eax   显示为16进制 
 print/t  2进制，　
 print/d 十进制,
x/nyz  : n表示字段数，y为输出格式，z是字段长度
 
６）单步调试
 n/next   /   s/step　
 
 7) 保存断点：
 info b  查看断点信息
 save breakpoint fig.dp 保存断点
 读取断点文件：　gdb hello -x fig.dp
 
 退出quit
 
 http://bbs.chinaunix.net/thread-150524-1-1.html
 
#### 2 使用kdbg 
 界面版本gdb  在gcc ... -g后，用kdbg打开即可、
 查看程序运行时各个地址：
	```
	#include<stdio.h>
	#include<string.h>
	#include<sys/types.h>
	#include<stdlib.h>
	#include<unistd.h>
	#define SHW_ADR(ID,I) printf("the id %s \t is at adr:%8X\n",ID,&I);
	extern etext,edata,end;
	char *cptr="Hello World\n";
	char buffer1[25];
	int main(void)
	{
		void showit(char *);
		int i=0;
	   	printf("Adr etext:%8x\tAdr edata:%8x \t Adr end :%8x \n\n",&etext,&edata,&end);
		SHW_ADR("main",main);
		SHW_ADR("showit",showit);
		SHW_ADR("cptr",cptr);
		SHW_ADR("buffer1",buffer1);
		SHW_ADR("i",i);
		strcpy(buffer1,"A demonstration\n");
		write(1,buffer1,strlen(buffer1)+1);
		for(;i<1;++i)
			showit(cptr);
		return 0;
	}
	
	void showit(char *p)
	{
		char *buffer2;
		SHW_ADR("buffer2",buffer2);
		if((buffer2=(char *)malloc((unsigned)(strlen(p)+1)))!=NULL)
		{
			strcpy(buffer2,p);
			printf("%s",buffer2);
			free(buffer2);
		}
		else
		{
			printf("Allocation error.\n");
			exit(1);
		}
	}
	
	```
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
	```cpp
	#include <stdio.h>
	#include <malloc.h>
	#include <unistd.h>
	#include <alloca.h>
	
	extern void afunc(void);
	extern etext,edata,end;
	
	int bss_var;				//no init globel data must be in bss
	
	int data_var=42;			//init globel data must be in data
	
	#define SHW_ADR(ID,I) printf("the %8s\t is at adr:%8x\n",ID,&I);		//the macro to printf the addr
	
	int main(int argc,char *argv[])
	{
		char *p,*b,*nb;
	
		printf("Adr etext:%8x\t Adr edata %8x\t Adr end %8x\t\n",&etext,&edata,&end);
		
		printf("\ntext Location:\n");
		SHW_ADR("main",main);			//text section function
		SHW_ADR("afunc",afunc);			//text section function
	
		printf("\nbss Location:\n");
		SHW_ADR("bss_var",bss_var);		//bss section var
	
		printf("\ndata location:\n");
		SHW_ADR("data_var",data_var);	//data section var
		
		
		printf("\nStack Locations:\n");
		afunc();
		
		p=(char *)alloca(32);			//alloc memory from statck
		if(p!=NULL)
		{
			SHW_ADR("start",p);
			SHW_ADR("end",p+31);
		}
		
		b=(char *)malloc(32*sizeof(char));	//malloc memory from heap
		nb=(char *)malloc(16*sizeof(char));
		
		printf("\nHeap Locations:\n");	
		printf("the Heap start: %p\n",b);
		printf("the Heap end:%p\n",(nb+16*sizeof(char)));
		printf("\nb and nb in Stack\n");
		SHW_ADR("b",b);
		SHW_ADR("nb",nb);
		free(b);
		free(nb);
	}
	
	
	void afunc(void)
	{
		static int long level=0;	//data section static var
		int	 stack_var;				//temp var ,in stack section
		if(++level==5)
		{
			return;
		}
		SHW_ADR("stack_var in stack section",stack_var);
		SHW_ADR("Level in data section",level);
		afunc();
	}
	```
 
 
 
 
 
