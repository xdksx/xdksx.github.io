---
title: c++_keyword
date: 2021-05-23 02:21:43
tags: c++keyword
categories: c++
---


### c++中的关键字和基本解释
##### alignas:<!--more-->
+ 语法：
alignas( expression ) 即expression是值为合法正数常量的表达式，可以是0或者其他1，2,4这种2的n次方值		
alignas( type-id )	即类型，比如alignas(int)，等价于alignas(alignof(type))  
alignas( pack ... ) 等价于应用于同一声明的多个alignas说明符，一个用于 parameter pack的每个成员，形参包可以是type parameter pack，也可以是non-type parameter pack  
+ 解释：
用于对齐：  
1) class/struct/union或enum类型的声明或定义中的字节对齐  
2) 类数据成员中的非位域类型的字节对齐  
3) 变量的声明的对齐，除了函数参数和exception 中catch中的参数；  
由这种声明声明的对象或类型的对齐要求将等于声明中使用的所有alignas说明符中最严格的(最大的)非零表达式，但它不会削弱类型的自然对齐，即最小是该类型的字节数对齐：  
如果声明上最严格(最大)的对齐比没有任何对齐说明符时的对齐要弱(也就是说，弱于它的自然对齐或弱于同一对象或类型的另一个声明上的对齐)，则程序是非法的:  
eg: 不允许   

```cpp

alignas(1) int a; //error
struct alignas(8) S {};
struct alignas(1) U { S s; }; // error: alignment of U would have been 8 without alignas(1)
无效值不能用于对齐：如alignas(3);,若是0 ，则被忽略
Note: c++11才有的关键字：
As of the ISO C11 standard, the C language has the _Alignas keyword and defines alignas as a preprocessor macro expanding to the keyword in the header <stdalign.h>, but in C++ this is a keyword, and the headers <stdalign.h> and <cstdalign> (until C++20) do not define such macro. They do, however, define the macro constant __alignas_is_defined.
例子：
// every object of type struct_float will be aligned to alignof(float) boundary
// (usually 4)
struct alignas(float) struct_float {
    // your definition here
};
 
// every object of type sse_t will be aligned to 256-byte boundary
struct alignas(256) sse_t
{
    float sse_data[4];
};
 
// the array "cacheline" will be aligned to 128-byte boundary
alignas(128) char cacheline[128];
#include <iostream>
int main()
{
    sse_t x, y, z;
    struct default_aligned { float data[4]; } a, b, c;
 
    std::cout << "alignof(struct_float) = " << alignof(struct_float) << '\n'
              << "alignof(sse_t) = " << alignof(sse_t) << '\n'
              << "alignof(alignas(128) char[128]) = " 
                  << alignof(alignas(128) char[128]) << "\n\n";
 
    std::cout << std::hex << std::showbase
              << "&x: " << &x << '\n'
              << "&y: " << &y << '\n'
              << "&z: " << &z << '\n'
              << "&a: " << &a << '\n'
              << "&b: " << &b << '\n'
              << "&c: " << &c << '\n';
}
alignof(struct_float) = 4
alignof(sse_t) = 256
alignof(alignas(128) char[128]) = 128
 
&x: 0x7fffd901bb00
&y: 0x7fffd901bc00
&z: 0x7fffd901bd00
&a: 0x7fffd901bad0
&b: 0x7fffd901bae0
&c: 0x7fffd901baf0
```

##### 关于字节对齐的解释：
对齐是指，访问的变量是按照指定的对齐(或类型本身大小)来寻址的，则比如int ，则按4的整数倍地址寻址，所以分给int这个变量的地址需要可以被4B整除；
放在struct中，体现为该成员的地址是可以被4整除，所以前面至少是4B，若前面是char 则填充3个字节；由此，还需要考虑到struct数组或单纯int数组的情况，前后之间需要对齐：
两个典型例子：

```cpp
struct B
{
    char a[9];
    alignas(16) int b;
    char c;
}
考虑最大的16，则a这里要填充7才能保证访问b的地址是16字节对齐的，按道理加完就是33，但是如果是数组，下个struct结构就不是16对齐的，或者说struct基地址本身也要是16对齐的，所以
应该是48；  
struct C
{
     long double a; 16B
     alignas(32) int b;
     int *c;
     int d;
}
这里考虑最大32，则a要填充16-> 32+4 ,到c是8字节对齐，所以又需要填充为8的整数倍，为32+4+4，接着最后的d，不用填充，而考虑到数组的情况，下个struct也要对齐32，
因为32+4+4+8+4 =52 ，52+32不是32整除的，所以需要填充到64，所以其大小为64  
eg2:
#include<iostream>

struct A
{
    char a;
    int b;
    short c;
};
struct A1
{
    char a;
    int b;
    short c;
    short d;
};

struct A3
{
    char a;
    int b;
   alignas(16) short c;
};
struct  A4
{
    long double a;
    alignas(32) int b;
    int *c;
    int d;
};

struct alignas(64) A5
{
    long double a;
    alignas(32) int b;
    int *c;
    int d;
};

int main()
{
    std::cout<<" A:"  << sizeof(struct A) << std::endl;
    std::cout<<" A1:"  << sizeof(struct A1) << std::endl;
    std::cout<<" A3:"  << sizeof(struct A3) << std::endl;
    A3 a3;
    std::cout<< "a3: " << &a3 << std::endl;
    A4 a4;
    std::cout<< "a4:" << sizeof(a4) << "addr:" << &a4 << std::endl;
    A5 a5;
    std::cout<< "a5:" << sizeof(a5) << "addr:" << &a5 << std::endl;
    return 0;
}

 A:12
 A1:12
 A3:32
a3: 0x7ffccf181a60
a4:64addr:0x7ffccf181ac0
a5:64addr:0x7ffccf181a80
```
##### alinof: 用来判断对齐数的：
alignof( type-id )：Returns a value of type std::size_t.
如果类型是引用类型，操作符返回被引用类型的对齐;如果类型是数组类型，则返回元素类型的对齐要求
eg:
```cpp
#include <iostream>
 
struct Foo {
    int   i;
    float f;
    char  c;
};
 
// Note: `alignas(alignof(long double))` below can be simplified to simply 
// `alignas(long double)` if desired.
struct alignas(alignof(long double)) Foo2 {
    // put your definition here
}; 
 
struct Empty {};
 
struct alignas(64) Empty64 {};
 
int main()
{
    std::cout << "Alignment of"  "\n"
        "- char             : " << alignof(char)    << "\n"
        "- pointer          : " << alignof(int*)    << "\n"
        "- class Foo        : " << alignof(Foo)     << "\n"
        "- class Foo2       : " << alignof(Foo2)    << "\n"
        "- empty class      : " << alignof(Empty)   << "\n"
        "- alignas(64) Empty: " << alignof(Empty64) << "\n";
}
Possible output:

Alignment of
- char             : 1
- pointer          : 8
- class Foo        : 4
- class Foo2       : 16
- empty class      : 1
- alignas(64) Empty: 64
```
一个例子：

```cpp 
 std::cout<<" more :"<< alignof( char[150]) << std::endl;//1
    std::cout<<" more :"<< alignof(alignas(256) char[150]) << std::endl;//1
    std::cout<<" more :"<< alignof(alignas(256) char[128]) << std::endl;//1
    std::cout<<" more :"<< alignof(alignas(128) char[150]) << std::endl;//1
    std::cout<<" more :"<< alignof(alignas(128) char[128]) << std::endl;//1
and
和&&一样的效果：
#include <iostream>
int n;
int main()
{
   if(n > 0 and n < 5)
   {
       std::cout << "n is small and positive\n";
   }
}
and_eq：和&=一样的效果，亦或的意思：
#include <iostream>
#include <bitset>
int main()
{
    std::bitset<4> mask("1100");
    std::bitset<4> val("0111");
    val and_eq mask;
    std::cout << val << '\n';
}
out:0100
```




##### asm
asm声明提供了在c++程序中嵌入汇编语言源代码的能力。该声明是有条件支持的，并且定义了实现，这意味着它可能不存在，即使是由实现提供，它也没有固定的含义。
语法：asm ( string_literal ) ;
string_literal通常是一个用汇编语言编写的短程序，每当执行此声明时就执行该程序。不同的c++编译器对于asm声明有不同的规则，对于与周围c++代码的交互有不同的约定。
与其他块声明一样，这个声明可以出现在块(函数体或另一个复合语句)中，也可以出现在块的外部
例如以下在linux x86_64下运行的：
```cpp
#include <iostream>
 
extern "C" int func();
// the definition of func is written in assembly language
// raw string literal could be very useful
asm(R"(
.globl func
    .type func, @function
    func:
    .cfi_startproc
    movl $7, %eax
    ret
    .cfi_endproc
)");
 
int main()
{
    int n = func();
    // extended inline assembly
    asm ("leal (%0,%0,4),%0"
         : "=r" (n)
         : "0" (n));
    std::cout << "7*5 = " << n << std::endl; // flush is intentional
 
    // standard inline assembly
    asm ("movq $60, %rax\n\t" // the exit syscall number on Linux
         "movq $2,  %rdi\n\t" // this program returns 2
         "syscall");
}
out:7*5 = 35
```
##### atomic系列
##### auto:编译期间的类型推导；
##### bitand:按位与：位操作，和&一样： 
```cpp
#include <iostream>
#include <bitset>
using bin = std::bitset<8>;
void show(bin z, const char* s, int n)
{
if (n == 0) std::cout << "┌────────────┬──────────┐\n";
if (n <= 2) std::cout << "│ " <<s<< " │ " <<z<<" │\n";
if (n == 2) std::cout << "└────────────┴──────────┘\n";
}
int main()
{
bin x{ "01011010" }; show(x, "x ", 0);
bin y{ "00111100" }; show(y, "y ", 1);
bin z = x bitand y; show(z, "x bitand y", 2);
}
┌────────────┬──────────┐
│ x │ 01011010 │
│ y │ 00111100 │
│ x bitand y │ 00011000 │
└────────────┴──────────┘

bitor: 和|等效：
#include <iostream>
#include <bitset>
 
using bin = std::bitset<8>;
 
void show(bin z, const char* s, int n)
{
if (n == 0) std::cout << "┌───────────┬──────────┐\n";
if (n <= 2) std::cout << "│ " <<s<< " │ " <<z<<" │\n";
if (n == 2) std::cout << "└───────────┴──────────┘\n";
}
 
int main()
{
bin x{ "01011010" }; show(x, "x ", 0);
bin y{ "00111100" }; show(y, "y ", 1);
bin z = x bitor y; show(z, "x bitor y", 2);
}
┌───────────┬──────────┐
│ x │ 01011010 │
│ y │ 00111100 │
│ x bitor y │ 01111110 │
└───────────┴──────────┘
```

##### bool: 布尔类型；
##### break:用在switch,for等封闭复合语句中
在此语句之后，控制权被转移到紧接在外围循环或开关之后的语句。与任何块退出一样，在封闭复合语句或循环/开关条件中声明的所有自动存储对象都将在封闭循环后的第一行执行之前销毁，按构造顺序相反。
##### case: 在swtich中使用 
##### catch: 在try-catch语句块中，在语句中一并解释；
##### char,char8_t,char16_t,char32_t,class,是类型标识符
##### compl: 和~一样的效果，是按位取反
##### concepts：c++20新特性，TD---和模板类型萃取合并解释
##### const和cv
##### consteval（since c++20）/constexpr/constinit 
##### const_cast:在具有不同cv限定符的类型之间转换
const_cast < new_type > ( expression ) 
Returns a value of type new_type.
解释： 只有以下转换可以用const_cast实现，通常，only const_cast may be used to cast away (remove) constness or volatility.  
1 同一个类型的两个可能是多级指针可以相互转换，而不管每个级别上的cv限定符是什么。  
2 任何类型T的左值都可以转换为相同类型T的左值或右值引用，或多或少符合cv要求。同样，类类型的prvalue或任何类型的xvalue都可以转换为符合cv要求的右值引用。引用const_cast的结果指向原对象if表达式为glvalue，以及实体化临时表达式(自c++ 17起)。  
3 同样的规则适用于可能指向数据成员的多级指针，也适用于可能指向已知和未知绑定的数组的多级指针(指向cv限定元素的数组本身被认为是cv限定的)(从c++ 17开始)  
4 空指针值可以转换为new_type的空指针值  
对于所有的cast表达式，结果可以是：  
1） 如果new_type是左值引用类型或函数类型的右值引用，则为左值;  
2）如果new_type是对象类型的右值引用，则为xvalue  
3）一个prvalue ，对其他情况   
note:指向函数和成员函数的指针不受const_cast约束  
const_cast使实际引用const对象的非const类型的引用或指针，或者实际引用volatile对象的非volatile类型的引用或指针成为可能。通过非const访问路径修改const对象和通过非volatile glvalue引用volatile对象会导致未定义的行为 
 
```cpp
#include <iostream>
struct type {
int i;
type(): i(3) {}
void f(int v) const {
// this->i = v; // compile error: this is a pointer to const
const_cast<type*>(this)->i = v; // OK as long as the type object isn't const
}
};
int main() 
{
int i = 3; // i is not declared const
const int& rci = i; 
const_cast<int&>(rci) = 4; // OK: modifies i
std::cout << "i = " << i << '\n';
type t; // if this was const type t, then t.f(4) would be undefined behavior
t.f(4);
std::cout << "type::i = " << t.i << '\n';
const int j = 3; // j is declared const
[[maybe_unused]]
int* pj = const_cast<int*>(&j);
// *pj = 4; // undefined behavior
[[maybe_unused]]
void (type::* pmf)(int) const = &type::f; // pointer to member function
// const_cast<void(type::*)(int)>(pmf); // compile error: const_cast does
// not work on function pointers
}
Output:
i = 4
type::i = 4
```



















##### continue
#### 以下三个是关于协程的，单独记录
##### co_await(since c++20)
##### co_return(since c++20)
##### co_yield(since c++20)
#### 继续
##### decltype:编译时类型推导，基本用法见下面例子：
```cpp
 int i = 4;
    int arr[5] = { 0 };
    int *ptr = arr;
    struct S{ double d; }s ;
    void Overloaded(int);
    void Overloaded(char);//重载的函数
    int && RvalRef();
    const bool Func(int);
 
    //规则一：推导为其类型
    decltype (arr) var1; //int 标记符表达式
 
    decltype (ptr) var2;//int *  标记符表达式
 
    decltype(s.d) var3;//doubel 成员访问表达式
 
    //decltype(Overloaded) var4;//重载函数。编译错误。
 
    //规则二：将亡值。推导为类型的右值引用。
 
    decltype (RvalRef()) var5 = 1;
 
    //规则三：左值，推导为类型的引用。
 
    decltype ((i))var6 = i;     //int&
 
    decltype (true ? i : i) var7 = i; //int&  条件表达式返回左值。
 
    decltype (++i) var8 = i; //int&  ++i返回i的左值。
 
    decltype(arr[5]) var9 = i;//int&. []操作返回左值
 
    decltype(*ptr)var10 = i;//int& *操作返回左值
 
    decltype("hello")var11 = "hello"; //const char(&)[9]  字符串字面常量为左值，且为const左值。

 
    //规则四：以上都不是，则推导为本类型
 
    decltype(1) var12;//const int
 
    decltype(Func(1)) var13=true;//const bool
 
    decltype(i++) var14 = i;//int i++返回右值
```
##### default，delete用于虚基类的函数声明，c++11
##### do(-while),
##### double类型
##### 动态类型转换：dynamic_cast 见类型，泛型总结
##### else,enum,
##### explicit:在C++中，explicit关键字用来修饰类的构造函数，被修饰的构造函数的类，不能发生相应的隐式类型转换，只能以显示的方式进行类型转换。
```cpp
struct A
{
    A(int) { }      // 转换构造函数
    A(int, int) { } // 转换构造函数 (C++11)
    operator bool() const { return true; }
};
 
struct B
{
    explicit B(int) { }
    explicit B(int, int) { }
    explicit operator bool() const { return true; }
};
 
int main()
{
    A a1 = 1;      // OK：复制初始化选择 A::A(int)
    A a2(2);       // OK：直接初始化选择 A::A(int)
    A a3 {4, 5};   // OK：直接列表初始化选择 A::A(int, int)
    A a4 = {4, 5}; // OK：复制列表初始化选择 A::A(int, int)
    A a5 = (A)1;   // OK：显式转型进行 static_cast
    if (a1) ;      // OK：A::operator bool()
    bool na1 = a1; // OK：复制初始化选择 A::operator bool()
    bool na2 = static_cast<bool>(a1); // OK：static_cast 进行直接初始化
 
//  B b1 = 1;      // 错误：复制初始化不考虑 B::B(int)
    B b2(2);       // OK：直接初始化选择 B::B(int)
    B b3 {4, 5};   // OK：直接列表初始化选择 B::B(int, int)
//  B b4 = {4, 5}; // 错误：复制列表初始化不考虑 B::B(int,int)
    B b5 = (B)1;   // OK：显式转型进行 static_cast
    if (b2) ;      // OK：B::operator bool()
//  bool nb1 = b2; // 错误：复制初始化不考虑 B::operator bool()
    bool nb2 = static_cast<bool>(b2); // OK：static_cast 进行直接初始化
}
```
#####  export:用于标注一个模板定义被导出，并允许在其他翻译单元中声明，但不定义同一模板，c++11前可用，c++11起，不使用，但保留该关键字；
通常情况下，你会在.h文件中声明函数和类，而将它们的定义放置在一个单独的.cpp文件中。但是在使用模板时，这种习惯性做法将变得不再有用，因为当实例化一个模板时，编译器必须看到模板确切的定义，而不仅仅是它的声明。 因此，最好的办法就是将模板的声明和定义都放置在同一个.h文件中。这就是为什么所有的STL头文件都包含模板定义的原因。
另外一个方法就是使用关键字“export”！你可以在.h文件中，声明模板类和模板函数；在.cpp文件中，使用关键字export来定义具体的模板类对象和模板函数；然后在其他用户代码文件中，包含声明头文件后，就可以使用该这些对象和函数了。例如：
```cpp
// output.h - 声明头文件  
template<class T> void output (const T& t);

// out.cpp - 定义代码文件   
#include <****>   
export template<class T>
void output (const T& t) {std::cerr << t;}

//main.cpp:用户代码文件
#include "output.h"   void main() // 使用output()   {   output(4);  
output("Hello");   }
大多数编译器不支持，如vs,gcc,不去使用

```


##### extern,false,float,for,friend,goto,if,inline,int 
已知略
##### long
已知略
##### mutable:
在C++中，mutable也是为了突破const的限制而设置的，被mutable修饰的变量将永远处于可变的状态。

mutable的作用有两点：
（1）保持常量对象中大部分数据成员仍然是“只读”的情况下，实现对个别数据成员的修改；
（2）使类的const函数可以修改对象的mutable数据成员。

使用mutable的注意事项：
（1）mutable只能作用于类的非静态和非常量数据成员。
（2）在一个类中，应尽量或者不用mutable，大量使用mutable表示程序设计存在缺陷。
```cpp
#include <iostream>
using namespace std;

//mutable int test;//编译出错

class Student
{
	string name;
	mutable int getNum;
	//mutable const int test;    //编译出错
	//mutable static int static1;//编译出错
	
public:
	Student(char* name)
	{
		this->name=name;
		getNum=0;
	}
	string getName() const
	{
		++getNum;
		return name;
	}
	void pintTimes() const
	{
		cout<<getNum<<endl;
	}
};

int main(int argc, char* argv[])
{
	const Student s("张三");
	cout<<s.getName().c_str()<<endl;
	s.pintTimes();
	return 0;
}
```

##### namespace,new
已知略
##### noexcept: 用于声明函数等是否有异常抛出，以及检测函数等是否有异常抛出；异常处理部分会涵盖

##### not
已知略
##### not_eq
已知略
##### nullptr
已知略
##### operator
已知略
##### or or_eq
已知略
##### private,protected,public;
已知略
##### reflexpr: 和反射相关，c++20
##### register：存储有关，c++17后弃用
##### reinterpret_cast： 类型转换，可以在任何指针类型转换，但是风险大，不安全，使用频率低，具体在泛型相关内容统一解释；
##### requires:c++20后，在模板的泛型中使用
##### return,short,signed,sizeof,static
##### static_assert(c++11);编译期间的断言检查
##### static_cast 类型转换，泛型中统一
##### struct, switch
已知略
##### synchronized 规划中
##### template
##### this
##### thread_local: 存储连接中统一解释：线程存储
##### throw:异常捕获相关
##### true
##### try
##### typedef
##### typeid:用于在运行时确定表达式的类型
##### typename
##### union,unsigned,using,virtual
##### volatile:  cv限定中的v
##### void
##### wchar_t,while
##### xor,xor_eq


