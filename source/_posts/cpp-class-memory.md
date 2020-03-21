---
title: cpp_class_memory
date: 2018-06-09 10:44:41
tags: cpp_memory
categories: c&cpp
---
### c++类内存布局：
#### 静态
下编译后的“内存布局”，此时还未分配内存，不能算是内存，只能说是文件布局<!--more-->
+ two question:  
 多少内存能表现一个ob:?  
 多少内存能表现一个class?--编译期，如　int 大小为４ 
+ 静态下，编译后同C一样分为代码段等；而类的特性有些类似于struct,定义类的文件中包含了成员变量，成员函数。
+ 类：
        非静态成员变量：存于对象中
        vptr指针：存于对象中
        vt表:vptr指向这个表，这个表在构造函数中默认隐士初始化，将对应的虚拟函数地址赋给表中的各个值（表本身with"类"）
        obj:         class:
        _vptr --->   table: ptr1 -->virtual func1
                            ptr2 -->virtual func2
        静态变量: 存入数据段中
        成员函数:代码段,通过this和成员变量建立联系
        静态函数：　存于代码段中
        全局函数
        main函数
        全局变量和静态变量
        局部变量：栈
        something extra depend on compiler~
+ some rules
   + 每个类产生一堆指向virtual func的指针，放在表格中，表称为：vtbl;   
   + 每个obj被添加了一个指针，指向相关的tirtual table ,为vptr;
   +   vptr的设定和重置由constructor，destructor,copy assignment运算服自动完成）　
   + 注意每一个class所关联的type info object(用于支持runtime type identification  )也经由virtual table指出，放在vptr[0]处）
   +  虚函数有可能被转换为：(*px->vtbl[1])(px)
          具体见深入c++模型书
              
#### 多少内存能表现一个ob:?
+ non static data members
+ padding
+ virtual---vptr
#### 多少内存能表现一个class? 
见datamember_memory  
```cpp
最小是１　  size  
class T{ };   ---1 一个char 表示这个类型  
class X :public virtual T{};　　--指针大小，指针指向T    virtual base class subobject  
class Y :public virtual T{};  --指针大小
class A:public X,public Y {};　--两个指针大小　　```
more seee datamember_memory


#### 运行时
的内存布局，即作为进程运行时，其内存是如何的；

+ 运行时，涉及类的对象模型，对象的内存部分只包含非静态成员变量和虚拟函数指针表；
+ 可以通过gdb 或kgdb进行查看类对象的地址或者类函数的地址：
 ```cpp
	Circle c12;
		Circle c1(1.2,"red");
printf("getRadius:%x\n",&Circle::getRadius);
printf("%x, %x\n",&c12,c12);
 void *cc;
        cc=(Circle*)(&c12);
		cout<<*((double*)cc)<<endl;
```		
    		
所以c++的对象带来的开销在于操作多态时的vptr等效率低）		

一个例子
```cpp
#include<iostream>
using namespace std;
class A
{
		public:
				    virtual const char* getName() { return "A"; }
					virtual int getage(){ return 3;}
};
 
class B: public A
{
		public:
				    virtual const char* getName() { return "B"; }
					virtual int getage(){return 5;}
};
 
class C: public B
{
		public:
				    const char* getName() { return "C"; }
					int getage(){return 6;}
};
 
class D: public C
{
		public:
				    virtual const char* getName() { return "D"; }
0x400ae8 push   %rbp
0x400ae9 mov    %rsp,%rbp
0x400aec mov    %rdi,-0x8(%rbp)
					int getage() {return 8;}
0x400af8 push   %rbp
0x400af9 mov    %rsp,%rbp
0x400afc mov    %rdi,-0x8(%rbp)
};
 
int main()
{ 
		    A aa;
			aa.getName();
		    D d;
			d.getName();
			A &rBase = d;
			rBase.getName();
0x400986 mov    -0x28(%rbp),%rax
0x40098a mov    (%rax),%rax
0x40098d mov    (%rax),%rax
0x400990 mov    -0x28(%rbp),%rdx
0x400994 mov    %rdx,%rdi
0x400997 callq  *%rax
            rBase.getage();
0x400999 mov    -0x28(%rbp),%rax
0x40099d mov    (%rax),%rax
0x4009a0 add    $0x8,%rax
0x4009a4 mov    (%rax),%rax
0x4009a7 mov    -0x28(%rbp),%rdx
0x4009ab mov    %rdx,%rdi
0x4009ae callq  *%rax
		    std::cout << "rBase is a " << rBase.getName() << '\n';
					 
			D d2;
            A &rBase2 =d2;
0x4009f9 lea    -0x30(%rbp),%rax
0x4009fd mov    %rax,-0x20(%rbp)
		   rBase2.getName();	
0x400a01 mov    -0x20(%rbp),%rax
0x400a05 mov    (%rax),%rax
0x400a08 mov    (%rax),%rax
0x400a0b mov    -0x20(%rbp),%rdx
0x400a0f mov    %rdx,%rdi
0x400a12 callq  *%rax


		    return 0;
}		
```
		
		
		
		
		
		
		

something else:  
+ 成员函数也是只有一份被存于代码区中，而对象并没有指向成员函数的指针，对象只存成员和虚拟表　；那在对象.成员函数使用时，
+ 是如何确定该函数中使用的成员是该对象的成员?
这里是this指针的作用
从汇编看，是否是成员函数在取的成员变量时，会做一些动作，这时的成员变量是在栈区；；
或者从调用成员函数时，是否有传入或控制函数的一些动作，让它知道从哪里取得对应的成员返回；
+ 成员函数最终被编译成与对象无关的普通函数，除了成员变量，会丢失所有信息，所以编译时要在成员函数中添加一个额外的参数，把当前对象的首地址传入，以此来关联成员函数和成员变量。这个额外的参数，实际上就是 this，它是成员函数和成员变量关联的桥梁

