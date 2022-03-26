---
title: cpp_static
date: 2018-06-08 22:44:43
tags: cpp_keyword
categories: c&cpp
---
### c++关键词之static
##### something share:
其实，一开始学习c++的时候，并没有想去了解它的语法原理。虽然学c的时候，附带学了汇编，也学到了内存段，和学编译原理的时候顺带了解了它的相关词法语法处理。　　
现在开始从原因，历史，内存，汇编等角度去看cpp，觉得清晰了一些。虽然没有去学习g++或者其他编译器<!--more-->
如何处理c++的语法词法。这个想着以后如果有时间再研究吧。应该也是有工具去看，毕竟，g++是开源的，可以直接看源码如何处理C++



#### static overiew 
+ static在c的时候就有，因存储的地方不同于局部变量(in static mem ,not in stack)，使得在函数代码块结束后不会被销毁。从而能持续整个程序.
+ 静态存储区(data)：内存在程序编译的时候就已经分配好，这块内存在程序的整个运行期间都存在。它主要存放静态数据、全局数据和常量。

#### 目录：
１、static概念和用法  
２、static内存存储和汇编  
３、static和类相关内容与原理  

#### static概念：
　static为静态修饰符。编译器遇到这个修饰符时将对应的变量存储在静态存储区(data)  　
+ 根据类型：static可以修饰变量和函数，修饰对象和成员函数  
+ 根据位置：静态局部变量和全局静态变量 -存于静态数据区(data)

##### c中的static:
+ 修饰变量：作用域是本程序文件，局部变量默认为auto,静态变量不管局部或者全局都存储于静态数据区，使得不会因为函数调用而销毁
+ 修饰函数：只能用于本程序文件，普通函数默认为extern ,能被其他文件引用，static则不行

" 举个例子：  
在stat.h中声明static int getstats()函数.  
并在stat.c中实现它，static int getstats(){return xxx;}  
在main中或者其他文件中使用这个函数  
编译时报错未能找到该函数（未定义该函数)  
c++中的static当和类无关时同c" 


#### static使用和内存与汇编：
##### static全局变量
```c
   static int global1=4;
   12 int main()
   13 {
>> 14         int loc1=global1;

_ZL7global1:
	.long	4
	.text
	.globl	main
	.type	main, @function

	movl	_ZL7global1(%rip), %eax
	movl	%eax, -4(%rbp)
//且可以通过kdbg看到在执行期，
static变量的内存位置约为：
(int *) 0x601048 <global1>　数据段地址
局部变量的位置约为：
(char **) 0x7ffff7a54530 <loc1>　栈地址```

##### 静态局部变量
```c 
int main()
   13 {
   14         static int locstatic1=5;
>> 15         int loc1=global1;
>> 16         int loc2=locstatic1;

	movl	_ZZ4mainE10locstatic1(%rip), %eax
	movl	%eax, -4(%rbp)

_ZZ4mainE10locstatic1:
	.long	5
执行期：
　　loca1 　(int *) 0x7fffffffd8a8
　　loca2 (int *) 0x7fffffffd8ac

  global1 (int *) 0x601048 <global1>
  locstatic:(int *) 0x60104c <main::locstatic1>
	```
##### static定义的变量和函数只能在本程序文件中使用
要知道这个的原因可能有些复杂，一时半会也说不清，得先直到函数为什么能被外部引用：存在“符号表”中，能被外部链接，静态函数不在符号表中所以不行，类似这样吧，
这块不太清楚，感觉是这个原因
##### static函数：
```c
 static  int getv()
    4 {
    5         int a=5;
    6         a++;
    7         cout<<a<<endl;
    8         return 4;
    9 } 

	.type	_ZL4getvv, @function
_ZL4getvv:
.LFB1021:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	subq	$16, %rsp
　　　。。。。
　
```
　	call	_ZL4getvv
从汇编代码看貌似跟普通函数没什么差别
```c
运行时
　　static  int getv()
{
0x400816 push   %rbp
0x400817 mov    %rsp,%rbp
0x40081a sub    $0x10,%rsp
		int a=5;
		a++;
		cout<<a<<endl;
		return 4;
} 

static int global1=4;
int main()
{
		static int locstatic1=5;
		int loc1=global1;
		int loc2=locstatic1;
//		cout<<getv()<<endl;
    	getv();
0x400866 callq  0x400816 <getv()>
```


#### static和类相关
##### static成员变量的使用
 static成员变量是类的所有对象共有的，但是只有一份，通过类也可以访问到
```cpp
class Something
{
public:
    static int s_value; // declares the static member variable
};
 
int Something::s_value = 1; // defines the static member variable (we'll discuss this section below)
 
int main()
{
    // note: we're not instantiating any objects of type Something
 
    Something::s_value = 2;
    std::cout << Something::s_value << '\n';
    return 0;
}
```
```c
_ZN9Something7s_valueE:
	.long	1
	.text
	.globl	main
	.type	main, @function

　　
  Something::s_value=3;
0x40081a movl   $0x3,0x20083c(%rip)        # 0x601060 <Something::s_value>
（	movl	_ZN9Something7s_valueE(%rip), %eax
	movl	%eax, %esi
）
可见类似于上述的，存在内存的数据段中
```

##### 类静态变量作用域
静态成员在多个文件中:
类中的机制，可以让定义在文件中的静态成员变量能被其他文件访问；加了作用域的声明，所以可以这样用
```cpp
在stati.h
 class Some{
  2         public:
  3                 static int s_v;
  4 };
  5 //static int s_vv;//error 错误

在stati.cpp
   #include"stati.h"
  2 int Some::s_v=4;
  3 //static int s_vv=5;error

在main
   #include "stati.h"

  int gets= Some::s_v;

other:
class Whatever
{
public:
    static const int s_value = 4; // a static const int can be declared and initialized directly
};
```
##### 静态成员函数的使用
+ 考虑一个问题，当静态成员被申明为private时，则无法使用它directly by class。  
but you can用非静态成员函数操作或者用静态成员函数操作to use it  
+ 静态成员函数也是可以用类直接调用，它没有this指针，所以不能修改和访问对象的成员； 
+ （非静态成员func,它可以定义在类的内部或者外部，外部不用加static,这点和static成员不同，成员得在外部初始化－－因为public等定义成员时，非const不能赋值，而
static不能修饰构造函数（对象相关），所以只能在外部初始化)

```cpp
class IDGenerator
{
private:
    static int s_nextID; // Here's the declaration for a static member
 
public:
     static int getNextID(); // Here's the declaration for a static function
};
 
// Here's the definition of the static member outside the class.  Note we don't use the static keyword here.
// We'll start generating IDs at 1
int IDGenerator::s_nextID = 1;
 
// Here's the definition of the static function outside of the class.  Note we don't use the static keyword here.
int IDGenerator::getNextID() { return s_nextID++; } 
 
int main()
{
    for (int count=0; count < 5; ++count)
        std::cout << "The next ID is: " << IDGenerator::getNextID() << '\n';
 
    return 0;
}
```