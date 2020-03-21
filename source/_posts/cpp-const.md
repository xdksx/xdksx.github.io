---
title: cpp_const
date: 2018-06-08 22:07:46
tags: cpp_keyword
categories: c&cpp
---
### c++关键词之const
#### const介绍，用法，原理，注意点等
##### const 介绍：
const是用于标示不可修改的变量，对象或函数的。  
在其前面添加const就无法在之后做修改  
##### 用法 <!--more-->
const总结起来有以下用法：
+ const 修饰基本类型的变量  
 + const int xx=4; const double xd=3.3;   
 + const int array[3]={3,4,5};
 + const　修饰指针和引用
+ const 指针： 
 + const int *p=&value;   
  //无法改变指针指向的值，但是可以改变指针,  
  value可以是int value;/const int value;

```cpp
          int va=4;
    6         const int *p=&va ;
    7         va=5;
>>  8         *p=6;//error
    9         cout<<*p<<endl;//5
   10         return 0;
```

  + int *const p=&value; //无法改变指针的值，但是可以改变指针指向的值
  + const int *const p=&value;//指针的值和指针指向的值　都不可以改变
+ const 引用：
 + const int &ref=value;

##### const  c&cpp差异
+ 不同于CPP，c将const存入的内存中设为只读，还是在栈中，而取其值时需要从内存中获取，所以从这个点看其效率比cpp差
why?see follow 
```c
    c code:
    const int cc=8;
0x4004da movl   $0x8,-0x8(%rbp)
		int ccc=cc;
0x4004e1 mov    -0x8(%rbp),%eax
0x4004e4 mov    %eax,-0x4(%rbp)
```
##### const和类
+ const 对象 const对象不能调用非const成员函数，也不能改变成员
```cpp　　　　　　　　
       class constobj{
          public:
            int ax;
            int bx;
            constonj(int a,int b):ax(a),bx(b){};
            int getax() const {return ax;}
            int getbx() const {return bx;}
            void setax(int a){ax=a;}
        }
        int main ()
        {
            constobj cobj;
            const constobj ccobj;
            ccobj.setax(3);//error
            return 0;
            }```
+ const 成员函数（只有成员函数能被声明为const )，它不能改变成员
+ 不能在const成员函数中修改成员变量，但是可以修改其他变量。
+ 非const对象可以调用const成员函数
+ 一个灵活使用const成员函数的例子：

```cpp
class Something
{
private:
    std::string m_value;
public:
    Something(const std::string &value="") { m_value= value; }
    const std::string& getValue() const { return m_value; } // getValue() for const objects
    std::string& getValue() { return m_value; } // getValue() for non-const objects
};
int main()
{
	Something something;
	something.getValue() = "Hi"; // calls non-const getValue();
 
	const Something something2;
	something2.getValue(); // calls const getValue();
 
	return 0;
}
```

##### c++11中的添加的新内容
+ constexp，cv限定  
##### c++ const内存和原理
+ 基本变量  
const变量是在栈中分配内存的，但是仅限于定义的时候，之后其他地方使用这个const变量都直接是使用该值
从汇编中可以看到

```
    const int  co1=3;
0x4008ed movl   $0x3,-0x14(%rbp)
		 int  nor=co1;
0x4008f4 movl   $0x3,-0x10(%rbp)```

所以无法当进行co1=4;的赋值操作时会被编译器拦截，想想，编译后会变成3=4;是不对的； 
* 这也解释了为什么一开始就要给const的变量赋值 *


+ 数组:save in stack

```c 
   const int a[3]={2,3,5};
0x4008fb movl   $0x2,-0x20(%rbp)
0x400902 movl   $0x3,-0x1c(%rbp)
0x400909 movl   $0x5,-0x18(%rbp)```


+ 指针：  
const指针结合，其并不像基本类型那样贴出值，而是每次还是得去内存中取值，它的限定改变值，由编译器控制报错  

指针并不是像前面那样，如
```c
    const int *p=&value;
    int xx=*p;//被替换成值?不是的，而是去内存中先取出指针的值，再以此为地址去取值；
     int pv=4;
0x400a12 movl   $0x4,-0x58(%rbp)
      const int *p=&pv;
0x400a19 lea    -0x58(%rbp),%rax
0x400a1d mov    %rax,-0x40(%rbp)

	  int px=*p;
0x400a69 mov    -0x40(%rbp),%rax
0x400a6d mov    (%rax),%eax
0x400a6f mov    %eax,-0x48(%rbp)
```

```c
　　　   int pv=4;
0x400a12 movl   $0x4,-0x70(%rbp)//put 4 in mem(statck)
		 const int *p=&pv;
0x400a19 lea    -0x70(%rbp),%rax//get its addr
0x400a1d mov    %rax,-0x58(%rbp)//addr to p
         int *const pp=&pv; 
0x400a21 lea    -0x70(%rbp),%rax
0x400a25 mov    %rax,-0x50(%rbp)//adddr to pp
         int  *ppp=pp;
0x400a29 mov    -0x50(%rbp),%rax//get pp num
0x400a2d mov    %rax,-0x48(%rbp)//to ppp
```

+ 引用：
引用同指针，是会去内存中取值的,same to pointer
```c       
follow above
const int &ref=pv;
0x400aba lea    -0x7c(%rbp),%rax//get pv
0x400abe mov    %rax,-0x38(%rbp)//to ref
		 int cs=ref;
0x400ac2 mov    -0x38(%rbp),%rax//get ref
0x400ac6 mov    (%rax),%eax//get *ref
0x400ac8 mov    %eax,-0x64(%rbp)//to cs
```


+ const类对象：  
 + const对象为什么不能更改成员：  
   成员是存在对象中的，如int,char等成员，存在栈中
   const对象的成员也是放在栈中的，它们之所以不能赋值，是在编译期间确定，更具体的细节未知；  　
+ const对象为什么不能调用非const成员函数：
 + 一个成员函数如何被调用：   　　
 + 其实成员函数也是全局函数，所以它能被调用，  
　eg:   
     ```c
     obj.show();--->实际上被转换为：
     show(&obj)  --传递给this指针：
    所以当：const OB obj();
   obj.show();时，翻译为　const OB *this;
   在传递给OB *this时会出现不能将this指针
   从const OB转换为OB &的错误```

可以做个实验试试
+ conclude:   
if it can use non const func ,it will change member by non const this pointer;

+ 为什么可以调用const函数：  
const函数：void show() const转换为：show(const OB *this);所以匹配上了，就可以调用了

注意：有最小权限原则，非const对象可以调用const成员函数，就像可以赋值一样。非const变量可以用const赋值
