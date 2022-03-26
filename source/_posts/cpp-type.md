---
title: cpp_type
date: 2021-04-04 02:32:54
tags: cpp_type
categories: c&cpp
---

c++数据类型：

### 数据类型概览
1) 类型无处不在，注意表达式也有类型，一些语句本身也是某个类型的值，以下这些都有类型属性
    对象的类型和基本类型
    引用的类型
    函数的类型(返回值类型)
    函数模板特化
    表达式的值类型<!--more-->
2) 类型如何分类以及复合类型
    内建类型：
    void,
    nullptr_t(c++), 
    arithmetic 类型(整数数值)
    浮点数
    整数： 
       bool 
       字符
       有符号整数
       无符号整数
    复合类型：引用，指针，类的成员指针，数组，函数，枚举，类类型  
3) 关于类型的命名，类型在使用时，并不是严格的按照既定名称来的，比如vector<int> 也可以作为一个类型来使用（c++)
    eg:
```cpp
int* p;               // declaration of a pointer to int
static_cast<int*>(p); // type-id is "int*"
 
int a[3];   // declaration of an array of 3 int
new int[3]; // type-id is "int[3]" (called new-type-id)
```
4) 静态类型： 编译期确定，运行时无法改变； 动态类型：运行时动态指定，比如多态（c++）；
5) 未完成的类型，一些本身就是未完成的类型如void，还有是类类型中没有定义所有的函数(c++)；

### 类型的分类：
#### 基本类型
##### void:
+ 1) 概念简介：void类型不能定义变量，因为是一种未完成的类型；同样不能定义数组，引用，但是可以定义void 指针以及函数返回类型也可以是void *；
+ 2) 用法注意：
A: void 可以是指针类型，可以将void *类型的指针转换为任意类型的指针；  
B: 既然void可以定义指针类型，也就说明这个指针是任意类型的指针，在内存中分配了8字节/4字节的地址，但是没有指定类型，也就是说其可以访问的内存范围无法确定，也就是说不能对void指针做取值操作如void *a; *a=3;/printf(*a);
void* 类型只能在转型后才能访问指向的内存地址；  
```cpp
#include<stdio.h>
int main()
{
        void *a;
        a=3;//这里对a赋值没问题
        int *b=0;
        a=b;//这里是可以的；这样将a赋值为int 类型的

        int c=32;
        b=&c;
        a=b;
        printf("%d\n",a);//访问a就不需要转型，毕竟指针都是固
定字节的；
        printf("%d\n",*(int*)a);//访问*a就需要转型，即使前面
赋值也没用；

       /*b=3;
       *b=32;
       printf("%d\n",*b);
       *///这么用回出现段错误
        a = (char*)a;
        char *p="aaaaa";
        a=p;
        a=(void*)a;
        //printf("%c",*a);//这里就不行，因为*操作不知道拿多
少内存
        printf("%s",a);//这种方式倒是可以成功输出,因为不是对void *做取值，而是取其指针值
        return 0;
}
```
+ 3)函数返回值void *意味着它可以接受任何类型的返回指针类型；毕竟只是返回一个指针值；
+ 4)一些库函数，参数是void* 的，意味着可以不受限制传入任意的指定类型，但是实际上内部实现是需要做类型转换的，比如转为char* 除非只是对指针做透传；

##### nullptr_t
std::nullptr_t (c++11)
Defined in header <cstddef>
typedef decltype(nullptr) nullptr_t;
nullptr_t是空指针字面量nullptr的类型。它是一种特别的类型，本身不是指针类型或指向成员类型的指针。
nullptr_t is available in the global namespace when <stddef.h> is included, even if it is not a part of C.
```cpp
#include <cstddef>
#include <iostream>
 
void f(int*)
{
   std::cout << "Pointer to integer overload\n";
}
 
void f(double*)
{
   std::cout << "Pointer to double overload\n";
}
 
void f(std::nullptr_t)
{
   std::cout << "null pointer overload\n";
}
 
int main()
{
    int* pi {}; double* pd {};
 
    f(pi);
    f(pd);
    f(nullptr); // would be ambiguous without void f(nullptr_t)
    // f(0);    // ambiguous call: all three functions are candidates
    // f(NULL); // ambiguous if NULL is an integral null pointer constant 
                // (as is the case in most implementations)
}
```

#### 数学类型：数值：
##### bool布尔类型：
布尔类型只有两个值，true和false;
实际例子：
```cpp
       int bbo = (bool)true;
       cout<< bbo << endl;//1
       int testnum = 100;
       cout << (bool)testnum << endl;//1
       bool tt = testnum;
       cout<< tt << endl;//1
 
       bool tye = true;
       cout << boolalpha << tye << endl;//true   //noboolalpha 按数值打印 
```
##### 字符类型:char 系列：char *,signed char,unsigned char
+ 1) char:char是八位一个字节的类型，
     本身赋值可以容纳任何的八位二进制而不管符号；
     但是输出时则对应输出ASCII码，当不匹配任何ASCII码时，则输出乱码；
     内存中的值：8位值；
```cpp
#include<stdio.h>
int main()
{
    //实际存放的值，在小于等于2的八次方 255(全1）时，按补码方式存放，即正数时为原码，负数时为除符号位外取反加1；
    char a=-43;//实际存放：
    char b=0;//实际存放：00000000
    char c=0b11111111;//11111111//255
    char d=127;//0111 1111
    char e=0b10000000;//1000 0000//128
    char f=-128;//
    printf("%c\n%c\n%c\n%c\n%c\n%c\n",a,b,c,d,e,f);
    printf("------------------------------\n");
    //读取的时候，则按补码方式读，即如10000000，则判断符号位
1，则为负数，再取反加1则为-128；
    printf("%d\n%d\n%d\n%d\n%d\n%d\n",a,b,c,d,e,f);//
    return 0;
}
输出：
-43
0
-1
127
-128
-128
```

+ signed和unsigned关键字含义：
2) signed char:其实内部存储和读取同char,可用于表示单字节有符号数；
3) unsigned char:即存储时还是按照补码存储，但是读取时是原码无符号取；所以存负数时，读取为正数；可用于存储单字节的无符号数；
```cpp
    unsigned char ua=-21;
    unsigned char ub=23;
    unsigned char uc =0;
    unsigned char ud = 255;
    unsigned char ue = 127;
    unsigned char uf = 128;
    printf("%d\n%d\n%d\n%d\n%d\n%d\n",ua,ub,uc,ud,ue,uf);
    return 0;
    输出：
       235 (存补码：-23->(1001 0101->1110 1011)->读出时按无符号：11101011==235）
23
0
255
127
128
```

+ 4) char *:char指针若是初始化为字符串，则这个字符串是存放在常量数据区，不是在栈区，所以不能够改动，但是可以改动指针的值；
```cpp
char *p ="ddd";
*(p+1)='3';编译报错；
      char *p = "abcdef";
      char *p2 = p;
      printf("%s\n",p2);
      char p3[]="dddd";
      p=p3;
      printf("%s\n",p3);
      printf("%s\n",p);
```
+ 5）关于c++的 宽char类型： char16_t char32_t w_char

##### 整数类型:short ,int ,long ,等
+ short(short int)，short和short int是一样的类型，size也一样
+ short,int，long，long long 默认是有符号的，若想更清晰的使用，可以带unsigned这种；
不同的位数系统下，整数类型的size也不相同，有如下系统情况：
32 bit systems:  
LP32 or 2/4/4 (int is 16-bit, long and pointer are 32-bit)  
Win16 API  
ILP32 or 4/4/4 (int, long, and pointer are 32-bit);  
Win32 API  
Unix and Unix-like systems (Linux, macOS)  
   
64 bit systems:  
LLP64 or 4/4/8 (int and long are 32-bit, pointer is 64-bit)  
Win64 API  
LP64 or 4/8/8 (int is 32-bit, long and pointer are 64-bit)   
Unix and Unix-like systems (Linux, macOS)  

Other models are very rare. For example, ILP64 (8/8/8: int, long, and pointer are 64-bit) only appeared in some early 64-bit Unix systems (e.g. UNICOS on Cray).

{% asset_img typeint.png type %}

##### 浮点数类型：float,double
+ float: 单精度浮点数
+ double :双精度浮点数
+ long double: 扩展精度的浮点数类型；
浮点数的一些特性：  
1) 正数负数的无穷数  
2) 负0.0  
3) NaN not a number  
数值范围表参考：

{% asset_img type1.png type %}
#### 复合类型：

##### 引用类型：一个已存在的对象或函数的别名
+ 初始化：一个引用要求被一个有效的对象或函数初始化；  
引用的存储：引用不是对象，所以没必要性占用存储，尽管编译器在必要时可以分配存储空间来实现所需的语义(例如，引用类型的非静态数据成员通常会增加类的大小，以满足存储内存地址的需要)。
哪些没有引用：因为引用不是对象，所以没有void的引用，也没有引用的引用，没有指向引用的指针，也没有引用的数组；  
```cpp
int& a[3]; // error
int&* p;   // error
int& &r;   // error
这些全错；
```

引用类型不能在最高级别上符合cv要求;在声明中没有这方面的语法，如果将限定符添加到typedef-name或decltype说明符或类型模板参数中，则会忽略它

+ 引用折叠(缩写)
允许通过模板或typedefs中的类型操作形成对引用的引用，这种情况下引用折叠规则适用:  
```cpp
typedef int&  lref;
typedef int&& rref;
int n;
lref&  r1 = n; // type of r1 is int&
lref&& r2 = n; // type of r2 is int&
rref&  r3 = n; // type of r3 is int&
rref&& r4 = 1; // type of r4 is int&&
```
这一点，以及在函数模板中使用T&&时用于模板实参推导的特殊规则，形成了使std::forward成为可能的规则

+ 左引用
1） 左引用可以使用在一个已存在的对象，可选不同的cv限定
```cpp
eg:
#include <iostream>
#include <string>
 
int main() {
    std::string s = "Ex";
    std::string& r1 = s;
    const std::string& r2 = s;
 
    r1 += "ample";           // modifies s
//  r2 += "!";               // error: cannot modify through reference to const
    std::cout << r2 << '\n'; // prints s, which now holds "Example"
}
```

2）它也可以使用在函数参数上：
```cpp
#include <iostream>
#include <string>
 
void double_string(std::string& s) {
    s += s; // 's' is the same object as main()'s 'str'
}
 
int main() {
    std::string str = "Test";
    double_string(str);
    std::cout << str << '\n';
}
```
3）当函数的返回值是左引用时，则函数调用表达式也可以成为表达式的左值：
```cpp
#include <iostream>
#include <string>
 
char& char_number(std::string& s, std::size_t n) {
    return s.at(n); // string::at() returns a reference to char
}
 
int main() {
    std::string str = "Test";
    char_number(str, 1) = 'a'; // the function call is lvalue, can be assigned to
    std::cout << str << '\n';
}
//结果： Tast
```
+ 右引用
1) 右值引用是用来扩展临时对象的生命周期的；而const类型的左值引用也可以扩展临时对象的声明周期，但是不能被改变；
```cpp
eg:
#include <iostream>
#include <string>
 
int main() {
    std::string s1 = "Test";
//  std::string&& r1 = s1;           // error: can't bind to lvalue
 
    const std::string& r2 = s1 + s1; // okay: lvalue reference to const extends lifetime
//  r2 += "Test";                    // error: can't modify through reference to const
 
    std::string&& r3 = s1 + s1;      // okay: rvalue reference extends lifetime
    r3 += "Test";                    // okay: can modify through reference to non-const
    std::cout << r3 << '\n';
}
```
2)更重要的是，当一个函数同时具有右值引用和左值引用时，右值引用重载绑定到右值(包括prvalues和xvalues)，而左值引用重载绑定到左值:
```cpp
#include <iostream>
#include <utility>
 
void f(int& x) {
    std::cout << "lvalue reference overload f(" << x << ")\n";
}
 
void f(const int& x) {
    std::cout << "lvalue reference to const overload f(" << x << ")\n";
}
 
void f(int&& x) {
    std::cout << "rvalue reference overload f(" << x << ")\n";
}
 
int main() {
    int i = 1;
    const int ci = 2;
    f(i);  // calls f(int&)
    f(ci); // calls f(const int&)
    f(3);  // calls f(int&&)
           // would call f(const int&) if f(int&&) overload wasn't provided
    f(std::move(i)); // calls f(int&&)
 
    // rvalue reference variables are lvalues when used in expressions
    int&& x = 1;
    f(x);            // calls f(int& x)
    f(std::move(x)); // calls f(int&& x)
}
```
3）这允许在合适的时候自动选择移动构造函数、移动赋值操作符和其他支持移动的函数(例如std::vector::push_back())。
```cpp
std::vector<int> v{1,2,3,4,5};
std::vector<int> v2(std::move(v)); // binds an rvalue reference to v
assert(v.empty());
```
+ fowarding references
转发引用是一种特殊类型的引用，它保留了函数参数的值类别，使得可以通过std::forward转发它。转发引用可以是:  
1)函数模板的函数形参声明为对同一函数模板cv- qualified类型的右值引用:  
```cpp
template<class T>
int f(T&& x) {                    // x is a forwarding reference
    return g(std::forward<T>(x)); // and so can be forwarded
}
 
int main() {
    int i;
    f(i); // argument is lvalue, calls f<int&>(int&), std::forward<int&>(x) is lvalue
    f(0); // argument is rvalue, calls f<int>(int&&), std::forward<int>(x) is rvalue
}
 
template<class T>
int g(const T&& x); // x is not a forwarding reference: const T is not cv-unqualified
 
template<class T> struct A {
    template<class U>
    A(T&& x, U&& y, int* p); // x is not a forwarding reference: T is not a
                             // type template parameter of the constructor,
                             // but y is a forwarding reference
};

2)auto&& except when deduced from a brace-enclosed initializer list:
auto&& vec = foo();       // foo() may be lvalue or rvalue, vec is a forwarding reference
auto i = std::begin(vec); // works either way
(*i)++;                   // works either way
g(std::forward<decltype(vec)>(vec)); // forwards, preserving value category
 
for (auto&& x: f()) {
  // x is a forwarding reference; this is the safest way to use range for loops
}
auto&& z = {1, 2, 3}; // *not* a forwarding reference (special case for initializer lists)
```
+ Dangling references 悬挂引用：
尽管引用一旦初始化，总是引用有效的对象或函数，但是可以创建一个被引用对象的生命周期结束但引用仍可访问的程序(悬空)。访问这样的引用是未定义的行为。一个常见的例子是函数返回对自动变量(局部)的引用:
```cpp
std::string& f()
{
    std::string s = "Example";
    return s; // exits the scope of s:
              // its destructor is called and its storage deallocated
}
std::string& r = f(); // dangling reference
std::cout << r;       // undefined behavior: reads from a dangling reference
std::string s = f();  // undefined behavior: copy-initializes from a dangling reference
```


##### 指针类型:32位操作系统为4字节，64位操作系统为8字节
注意指针也是一个变量，那么也可以传值和返回值；  
一个指针的值：指向对象或函数，或一个对象的结尾(即对象位置的下一个位置) ,或空指针，或一个无效指针；  
对多字节的类型，指向它的指针，是指向它的第一个字节，即第一个字节的位置；而指向一个对象的结尾，是指指向这个对象结尾后的第一个字节；  
Note that two pointers that represent the same address may nonetheless have different values.  
```cpp
eg:
struct C
  6 {
  7     int x,y;
  8 } c;
  9 int main()
 10 {
 11     int *px = &c.x;
 12     int *pxe = px+1;
 13     int *py = &c.y;
 14     assert(pxe == py);
 15     *pxe = 1;//// undefined behavior even if the assertion does not fire 最好不要这么操作；
 16     cout<< *pxe<< endl;//1
 17     cout << c.y <<endl;//1
 18     cout << c.x << endl;//0
 19     cout << *(px+1) << endl;//1
 20     return 0;
 21 }
```



 
+ 1) 指向普通对象的类型 ，和指针的引用：
(1) 语法: S* D;
```cpp
int n;
int* np = &n; // pointer to int
int* const* npp = &np; // non-const pointer to const pointer to non-const int
 
int a[2];
int (*ap)[2] = &a; // pointer to array of int
 
struct S { int n; };
S s = {1};
int* sp = &s.n; // pointer to the int that is a member of s

int n;
int* p = &n;     // pointer to n
int& r = *p;     // reference is bound to the lvalue expression that identifies n ，注意这里是p指向的值的引用
r = 7;           // stores the int 7 in n
std::cout << *p; // lvalue-to-rvalue implicit conversion reads the value from n 输出为7
```
(2) 指针的引用: int n;int *p = &n;int *&r = p; 
```cpp
eg:
int m_value = 1;

void func(int *&p)
{
    p = &m_value;

    // 也可以根据你的需求分配内存
    p = new int;
    *p = 5;
}

int main(int argc, char *argv[])
{
    int n = 3;
    int *p = &n;
    int *&r = p;
    r=new int;
    std::cout<< *p << endl;//0 
 
    int n = 2;
    int *pn = &n;
    cout << *pn << endl;
    func(pn);
    cout << *pn <<endl;
    return 0;
}
```


(3) 一些说明：
指向数组的第一个成员的指针可以是用数组初始化，含隐式转换；
```cpp
int a[2];
int* p1 = a; // pointer to the first element a[0] (an int) of the array a
 
int b[6][3][8];
int (*p2)[3][8] = b; // pointer to the first element b[0] of the array b,
                     // which is an array of 3 arrays of 8 ints
同样的，基类和派生类也是：支持多态
struct Base {};
struct Derived : Base {};
 
Derived d;
Base* p = &d;
```

(4) 指向void的指针：
指向任何类型对象的指针都可以隐式转换为指向void的指针(可选cv限定);指针值没有改变。反向转换需要static_cast或显式类型转换，会产生原始指针值:
```cpp
int n = 1;
int* p1 = &n;
void* pv = p1;
int* p2 = static_cast<int*>(pv);
std::cout << *p2 << '\n'; // prints 1
```
如果原始指针指向某个多态类型对象中的基类子对象，dynamic_cast可用于获得指向派生类型的完整对象的void*。
指向void的指针用于传递未知类型的对象，这在C接口中很常见:pthread_create期望用户提供的回调函数接受并返回void*。在所有情况下，调用者都有责任在使用之前将指针转换为正确的类型。



+ 2) 指向函数的类型:
(1) 函数指针可以用非成员函数或静态成员函数的地址进行初始化。由于函数到指针的隐式转换，寻址操作符是可选的:
```cpp
    void f(int);
    void (*p1)(int) = &f;
    void (*p2)(int) = f; // same as &f
```
不像函数和函数的引用，函数指针是对象，因此可以进行拷贝，存储在数组和分配等  

(2)函数指针的一些例子
```cpp
Syntax	meaning
int (* f)()	f is a pointer to a function with prototype int func().
int (* f())()	f is a function taking no inputs and returning a pointer to a function with prototype int func().
int * f()	f is a function returning a pointer-to-int.
int (* a[])()	a is an array of pointers to functions each with prototype int func().
int (* f())[]	f is a function returning a pointer to an array.
int (f[])()	Not allowed. Can't have an array of functions.
int * const *(*g)(float)	g is pointer to a function with prototype int * const * func(float) where its return value int * const * is a pointer to a read-only pointer-to-int.
```
(3)模板和函数指针：
```cpp
    template<typename T> T f(T n) { return n; }
double f(double n) { return n; }
 
int main()
{
    int (*p)(int) = f; // instantiates and selects f<int>
}
```

+ 3) 指向类成员变量的类型  
(1) 指向非静态成员对象m的指针是C类的成员，可以用表达式&C::m进行初始化。像&(C::m)或&m这样的表达式在C的成员函数中不构成指向成员的指针。
这样的指针可以用作指针到成员访问操作符的右操作数。*操作符- > *:
```cpp
语法：S C::* D; //C是类类型
class C //or struct ,same
    {
    public:
        int m;
    } ;
    int main()
    {
        int C::* p= &C::m;//按照对指针的理解，初看下，是指p是int C::m 这种类型的指针，但是在使用时需要赋值
        C c = {7};
        std::cout << c.*p << '\n';//但是实际上不是，因为可以直接输出，而且可以用c.的形式，说明是它的成员；
    
        C b = {9};
        std::cout<< b.*p << endl;//而不同对象输出不同，说明这个指针是对象持有
    
        C* cp = &c;
        cp->m= 10 ;
        std::cout <<cp->*p << '\n';
       // std::cout <<sizeof(struct C) << endl;//4 not include *p
        std::cout <<sizeof(c) << endl;//4 not include *p ,size的时候，还是4，说明这里的形式可能不是简单的塞在结构里
        return 0;
    }
```

(2)指向可访问的无二义性非虚基类数据成员的指针可以隐式转换为指向派生类相同数据成员的指针:
```cpp
     struct Base { int m; };
 struct Derived : Base {};
 
 int main()
 {
     int Base::* bp = &Base::m;//先定义一个指向Base::m的指针，这个作为了Base的"成员"
     int Derived::* dp = bp;//赋值后，Derived的"成员"指针，也指向了bp,Base::m
     Derived d;
     d.m = 1;//对m赋值后
     std::cout << d.*dp << ' ' << d.*bp << '\n'; // prints 1 1 ,取出的两个都是实际值是m
     return 0;
 }
```
(3)在相反的方向转化,从一个指向派生类的数据成员指针明确非虚拟基类的数据成员,允许static_cast和显式类型转换,即使基类没有成员(但最终派生类,当指针用于访问):
```cpp
struct Base {};
struct Derived : Base { int m; };//派生类有这个成员，但是基类没有
 
int main()
{
    int Derived::* dp = &Derived::m; //派生类的指针
    int Base::* bp = static_cast<int Base::*>(dp);//基类上要动态转换，但是即使这样
 
    Derived d;
    d.m = 7;
    std::cout << d.*bp << '\n'; // okay: prints 7
 
    Base b;
    std::cout << b.*bp << '\n'; // undefined behavior，也无法访问到派生类"成员指针"
    return 0;
}
```

(4)指向成员的指针类型可以是指向成员的指针本身:指向成员的指针可以是多级的，在每一级都可以有不同的cv限定。也允许指针和指针到成员的混合多层次组合: 这代码看着贼恶心了
```cpp
struct A
{
    int m;
    // const pointer to non-const member
    int A::* const p;
};
 
int main()
{
    // non-const pointer to data member which is a const pointer to non-const member
    int A::* const A::* p1 = &A::p;
 
    const A a = {1, &A::m};
    std::cout << a.*(a.*p1) << '\n'; // prints 1
 
    // regular non-const pointer to a const pointer-to-member
    int A::* const* p2 = &a.p;
    std::cout << a.**p2 << '\n'; // prints 1
}
```

+ 4) 指向类成员函数的类型：
指向非静态成员函数f的指针是类C的成员，可以完全用表达式&C::f进行初始化。C成员函数中的&(C::f)或&f这样的表达式不构成指向成员函数的指针。
这样的指针可以用作指针到成员访问操作符的右操作数。- > *和. *。结果表达式只能用作函数调用操作符的左操作数:
```cpp
struct C
{
    void f(int n) { std::cout << n << '\n'; }
};
 
int main()
{
    void (C::* p)(int) = &C::f; // pointer to member function f of class C
    C c;
    (c.*p)(1);                  // prints 1
    C* cp = &c;
    (cp->*p)(2);                // prints 2
}
```
指向基类成员函数的指针可以隐式转换为指向派生类相同成员函数的指针:
```cpp
struct Base
{
    void f(int n) { std::cout << n << '\n'; }
};
struct Derived : Base {};
 
int main()
{
    void (Base::* bp)(int) = &Base::f;
    void (Derived::* dp)(int) = bp;
    Derived d;
    (d.*dp)(1);
    (d.*bp)(2);
}
```
在相反的方向转化,从一个指向派生类的成员函数指针明确非虚拟基类的成员函数,允许static_cast和显式类型转换,即使基类没有成员函数(但最终派生类,当指针用于访问):就是未定义行为：
```cpp
struct Base {};
struct Derived : Base
{
    void f(int n) { std::cout << n << '\n'; }
};
 
int main()
{
    void (Derived::* dp)(int) = &Derived::f;
    void (Base::* bp)(int) = static_cast<void (Base::*)(int)>(dp);
 
    Derived d;
    (d.*bp)(1); // okay: prints 1
 
    Base b;
    (b.*bp)(2); // undefined behavior
}
```
成员函数的指针可以用作回调函数或函数对象，通常在应用了std::mem_fn或std::bind之后使用。

+ 5) 关于空指针的值和空指针的作用
(1) 值和初始化：每种类型的指针都有一个特殊的值，称为该类型的空指针值。值为null的指针不指向对象或函数(对空指针进行解引用是未定义的行为)，它与值为null的所有相同类型的指针进行比较。
   要初始化指向null的指针或将null值赋给现有指针，可以使用空指针字面量nullptr、空指针常量null或从整数值0的隐式转换。零初始化和值初始化也初始化指向空值的指针。  
(2) 作用：空指针可以用来表示没有对象(例如function::target())，或者作为其他错误条件指示器(例如dynamic_cast)。通常，接收指针参数的函数几乎总是需要检查值是否为空，并以不同的方式处理这种情况(例如，当传递空指针时，delete表达式不做任何操作)。

+ 6) 关于cv修饰指针：
If cv appears before * in the pointer declaration, it is part of decl-specifier-seq and applies to the pointed-to object. 在*前面，则为修饰变量  
If cv appears after * in the pointer declaration, it is part of declarator and applies to the pointer that's being declared. 在*后面，则为修饰指针  


Syntax           meaning
const T*         pointer to constant object
T const*         pointer to constant object
T* const         constant pointer to object
const T* const   constant pointer to constant object
T const* const   constant pointer to constant object

```cpp
eg：
// pc is a non-const pointer to const int
// cpc is a const pointer to const int
// ppc is a non-const pointer to non-const pointer to const int
const int ci = 10, *pc = &ci, *const cpc = pc, **ppc;
// p is a non-const pointer to non-const int
// cp is a const pointer to non-const int
int i, *p, *const cp = &i;
 
i = ci;    // okay: value of const int copied into non-const int
*cp = ci;  // okay: non-const int (pointed-to by const pointer) can be changed
pc++;      // okay: non-const pointer (to const int) can be changed
pc = cpc;  // okay: non-const pointer (to const int) can be changed
pc = p;    // okay: non-const pointer (to const int) can be changed
ppc = &pc; // okay: address of pointer to const int is pointer to pointer to const int
 
ci = 1;    // error: const int cannot be changed
ci++;      // error: const int cannot be changed
*pc = 2;   // error: pointed-to const int cannot be changed
cp = &ci;  // error: const pointer (to non-const int) cannot be changed
cpc++;     // error: const pointer (to const int) cannot be changed
p = pc;    // error: pointer to non-const int cannot point to const int
ppc = &p;  // error: pointer to pointer to const int cannot point to
           // pointer to non-const int
```

+ 7) 关于智能指针
+ 8) 关于stl中的迭代器，替代指针的方案；
+ 9) 扩展：关于c++的cv概念：https://en.cppreference.com/w/cpp/language/cv 即const 和volatile
+ 10）为了辅助指针的类型判断，提供了这类函数：std::is_member_pointer
https://en.cppreference.com/w/cpp/types/is_member_pointer


##### 数组类型： 数组可以容纳所有内建类型，字符，指针，int等数字类型；
+ 1）数组索引，数组元素支持和不支持，cv限定，内存情况：存放内建类型，连续物理地址存放；访问时++索引或地址即可；
数组索引：声明的形式T [N];,声明一个数组对象,由连续N 个T类型的对象，分配一个数组的元素编号0,…,N - 1,和可能访问下标运算符[],在[0],…,(N - 1)。  
数组元素类型：数组可以从任何基本类型(void除外)、指针、指向成员的指针、类、枚举，或者从其他已知边界的数组(在这种情况下，该数组被称为多维数组)构造。换句话说，
除了未知范围的数组类型之外，只有对象类型才能成为数组类型的元素类型。不完整元素类型的数组类型也是不完整类型。 不支持引用的数组和函数的数组；  
cv限定：将cv限定符应用于数组类型(通过类型定义或模板类型操作)将限定符应用于元素类型，但是任何元素属于cv限定类型的数组类型都被认为具有相同的cv限定符  
+ 2)  数组的赋值：
数组类型的对象不能作为一个整体被修改:即使它们是左值(例如数组的地址可以被取走)，它们也不能出现在赋值操作符的左边
```cpp
int a[3] = {1, 2, 3}, b[3] = {4, 5, 6};
int (*p)[3] = &a; // okay: address of a can be taken
a = b;            // error: a is an array
struct { int c[3]; } s1, s2 = {3, 4, 5};
s1 = s2; // okay: implicity-defined copy assignment operator
         // can assign data members of array type
```




+ 3）数组和指针之间的隐式转换：
```cpp
eg:
#include <iostream>
#include <numeric>
#include <iterator>
 
void g(int (&a)[3])
{
    std::cout << a[0] << '\n';
}
 
void f(int* p)
{
    std::cout << *p << '\n';
}
 
int main()
{
    int a[3] = {1, 2, 3};
    int* p = a;
 
    std::cout << sizeof a << '\n'  // prints size of array
              << sizeof p << '\n'; // prints size of a pointer
 
    // where arrays are acceptable, but pointers aren't, only arrays may be used
    g(a); // okay: function takes an array by reference
//  g(p); // error
 
    for(int n: a)              // okay: arrays can be used in range-for loops
        std::cout << n << ' '; // prints elements of the array
//  for(int n: p)              // error
//      std::cout << n << ' ';
 
    std::iota(std::begin(a), std::end(a), 7); // okay: begin and end take arrays
//  std::iota(std::begin(p), std::end(p), 7); // error
 
    // where pointers are acceptable, but arrays aren't, both may be used:
    f(a); // okay: function takes a pointer
    f(p); // okay: function takes a pointer
 
    std::cout << *a << '\n' // prints the first element
              << *p << '\n' // same
              << *(a + 1) << ' ' << a[1] << '\n'  // prints the second element
              << *(p + 1) << ' ' << p[1] << '\n'; // same
}
```

+ 4）数组的数组，数组存放struct
数组的数组：多维数组其实在内存中的存放也是连续的，就是数组的每个元素是一个数组，这样：
```cpp
#include<stdio.h>
int main()
{
    int dualarray[2][5]={{1,2,3,4,5},{11,22,33,44,55}};
    int *pp=&dualarray[0][0];//指向数组元素的指针
    int (*p)[5]=dualarray; //使用行指针，指向二维数组的行；

    int n=0;
    for(n=0;n<10;n++)
    {
            printf("%d\n",*pp);
            pp++;
    }
    //printf("%d \n",**p);

    return 0;
}
数组存放的是struct:在内存中实际上也是连续存储的，但是在取值的时候，往往因为类型不同而不能直接取；
 struct cc x[3];
    struct cc sa;
    sa.a=1;
    sa.b=2;
    struct cc sb;
    sb.a=3;
    sb.b=4;
    struct cc sc;
    sc.a=5;
    sc.b=6;
    x[0]=sa;
    x[1]=sb;
    x[2]=sc;
    //struct cc *sp = &x[0];
    int *sp=&x[0].a;
    for(n=0;n<7;n++)
    {
            printf("%d\n",*sp);
            sp++;
    }
    return 0;
        输出：1 2 3 4 5 6 
```

+ 5) 未知边界的数组：
如果在数组的声明中省略了维度值，则声明的类型为“未知维度T的数组”，这是一种不完全类型，除非在带有初始化式的声明中使用 
```cpp
extern int x[];      // the type of x is "array of unknown bound of int"
int a[] = {1, 2, 3}; // the type of a is "array of 3 int"
```
因为数组元素不能是未知边界的数组，所以多维数组除了第一个维度之外，不能有未知边界:  
```cpp
extern int a[][2]; // okay: array of unknown bound of arrays of 2 int
extern int b[2][]; // error: array has incomplete element type
```
指向未知边界数组的指针不能参与指针算术，也不能在下标操作符的左侧使用，但可以解引用。指向未知边界数组的指针和引用不能用于函数形参。
数组和指针，引用的隐式转换：
```cpp
extern int a1[];
int (&r1)[] = a1;  // okay
int (*p1)[] = &a1; // okay
int (*q)[2] = &a1; // error (but okay in C)
 
int a2[] = {1, 2, 3};
int (&r2)[] = a2;  // error
int (*p2)[] = &a2; // error (but okay in C)
```
+ 6)  指针数组：数组元素是指针
+ 7)  数组和右值：
```cpp
#include <iostream>
#include <type_traits>
#include <utility>
 
void f(int (&&x)[2][3])
{
    std::cout << sizeof x << '\n';
}
 
struct X
{
    int i[2][3];
} x;
 
template<typename T> using identity = T;
 
int main()
{
    std::cout << sizeof X().i << '\n';           // size of the array
    f(X().i);                                    // okay: binds to xvalue
//  f(x.i);                                      // error: cannot bind to lvalue
 
    int a[2][3];
    f(std::move(a));                             // okay: binds to xvalue
 
    using arr_t = int[2][3];
    f(arr_t{});                                  // okay: binds to prvalue
    f(identity<int[][3]>{{1, 2, 3}, {4, 5, 6}}); // okay: binds to prvalue
 
}
```

##### 函数类型：
+ 1) 函数的声明语法
一个函数的声明是介绍函数的名字和类型；函数声明可能出现在任何的scope.在类中的声明是表明一个类成员函数(友元函数除外);  
函数的类型由返回值类型表明；  
c++或高版本的c在传统的函数声明上加了其他语法，所以对c++ 来说：  
(1)传统的函数声明语法  
(2)扩展的返回值声明：这个字段被放在最外面在这种情况下，decl-specifier-seq必须包含关键字auto  

```cpp
noptr-declarator ( parameter-list ) cv(optional) ref(optional) except(optional) attr(optional)	(1)	
noptr-declarator ( parameter-list ) cv(optional) ref(optional) except(optional) attr(optional) -> trailing	(2)	(since C++11)
一个函数声明由以下几部分组成：
noptr-declarator - 任何有效的声明，但是若开始于*,&,或&& ,则必须用小括号括起来：int（*pf)(double x); /int (*pf)(double); --这里应该指的是函数指针的声明，因为其他带&这几个符号的没见需要小括号的；TODO;
parameter-list:  - 参数列表，可能为空；
attr(c++11): 可选的属性列表，它作用于函数的类型而不是函数本身，它标识于函数声明的最后，和函数声明的开头相关联；
cv: --const/volatile: 只作用于非静态成员函数的声明；
ref:-- 引用声明，只作用于非静态成员函数的声明；
eg:
struct test{
  void f() &{ std::cout << "lvalue object\n"; }
  void f() &&{ std::cout << "rvalue object\n"; }
  void g() &&{std::cout<< "g rvalue  boject\n"; }
};

except:不属于函数类型的一部分，是异常抛出相关字段；
trailing(c++11): 扩展的返回类型：当返回类型依赖于参数名时很有用：eg:template <class T, class U> auto add(T t, U u) -> decltype(t + u);
(PS: c++20 中还支持： requires true在后面，具体见参考文档)
```

用法：函数的声明可以和其他声明混合在一起如：
```cpp
// declares an int, an int*, a function, and a pointer to a function
int a = 1, *p = NULL, f(), (*pf)(double);
// decl-specifier-seq is int
// declarator f() declares (but doesn't define)
//                a function taking no arguments and returning int
 
struct S
{
    virtual int f(char) const, g(int) &&; // declares two non-static member functions
    virtual int f(char), x; // compile-time error: virtual (in decl-specifier-seq)
                            // is only allowed in declarations of non-static
                            // member functions
};
```
            
+ 2) 函数的返回值
函数的返回值不能是一个函数或者数组，但是可以是一个指针或者引用指向他们
(1) 使用auto c++11 类型推论
文档描述：
If the decl-specifier-seq of the function declaration contains the keyword auto, trailing return type may be omitted, and will be deduced by the compiler from the type of the expression used in the return statement. If the return type does not use decltype(auto), the deduction follows the rules of template argument deduction.
```cpp
int x = 1;
auto f() { return x; }        // return type is int
const auto& f() { return x; } // return type is const int&
```
当返回值类型有多种情况时，会被归结为一种类型：
```cpp
auto f(bool val)
{
    if (val) return 123; // deduces return type int
    else return 3.14f;   // error: deduces return type float 编译失败
}
```
当返回值无类型，即为void时，则auto也可以用，decltype(auto)也可以用，但是注意以下情况：
```cpp
auto f() {}              // returns void
auto g() { return f(); } // returns void
auto* x() {}             // error: cannot deduce auto* from void
```
当返回值能被推算是归结到相同类型，则允许：
```cpp
auto sum(int i)
{
    if (i == 1)
        return i;              // sum’s return type is int
    else
        return sum(i - 1) + i; // okay: sum’s return type is already known
}
```
当返回值类型是初始化列表时，不允许：
```cpp
auto func () { return {1, 2, 3}; } // error
虚函数和协程不允许使用auto推论( c++14/c++20)
struct F
{
    virtual auto f() { return 2; } // error
};
```
当一个函数已经使用auto 类型推论，则不能使用确定类型的声明和decltype，即使是相同类型：
```cpp
auto f();               // declared, not yet defined
auto f() { return 42; } // defined, return type is int
int f();                // error: cannot use the deduced type
decltype(auto) f();     // error: different kind of deduction
auto f();               // okay: re-declared

template<typename T>
struct A { friend T frf(T); };
auto frf(int i) { return i; } // not a friend of A<int>
                当函数模板使用时：
                template<class T> auto f(T t) { return t; }
typedef decltype(f(1)) fint_t;    // instantiates f<int> to deduce return type
template<class T> auto f(T* t) { return *t; }
void g() { int (*p)(int*) = &f; } // instantiates both fs to determine return types,
                                  // chooses second template overload
```
当函数模板的特化使用时，必须和返回值相同类型：
```cpp
template<typename T> auto g(T t) { return t; } // #1
template auto g(int);      // okay: return type is int
//template char g(char);     // error: no matching template
 
template<> auto g(double); // okay: forward declaration with unknown return type
template<typename T> T g(T t) { return t; } // okay: not equivalent to #1
template char g(char);     // okay: now there is a matching template
template auto g(float);    // still matches #1
//void h() { return g(42); } // error: ambiguous
```


(2) 使用decltype c++11  用于从参数判断类型 
int x = 1;
decltype(auto) f() { return x; }  // return type is int, same as decltype(x)
decltype(auto) f() { return(x); } // return type is int&, same as decltype((x))

+ 3) 函数的参数列表
(1) 有如下5种语法情况：
```cpp
attr(optional) decl-specifier-seq declarator	(1)	int f(int a, int *p, int (*(*x)(double))[3]);
attr(optional) decl-specifier-seq declarator = initializer	(2)	int f(int a = 7, int *p = nullptr, int (*(*x)(double))[3] = nullptr);
attr(optional) decl-specifier-seq abstract-declarator(optional)	(3)	int f(int, int *, int (*(*)(double))[3]);
attr(optional) decl-specifier-seq abstract-declarator(optional) = initializer	(4)	int f(int = 7, int * = nullptr, int (*(*)(double))[3] = nullptr);
void	(5)	空参数，int f(void)和int f()一样，但是不能用cv描述，
int f(void, int);(不应该只有int吗) and int f(const void); 都是错误的,void*则可以，在函数模板中，则不能被实例为T=void
函数可选参数列表可以看：variadic function:手册有实例；
```
(2) 关于新特性 
c++20：如果函数的任何形参使用了占位符(auto或概念类型)，则函数声明将改为缩写的函数模板声明:

(3) 关于函数参数名：仅仅是为了文档化，自我说明；参数被这样解析：
首先，类型确定：若是T数组或未只边界的T，则被指向T的指针替代，若是函数类型F，则被指向函数F的指针替代；而对于CV限定，则只会影响函数类型，不会影响参数类型(因为函数只是传递值)：
因此： int f(const int p,decltype(p)*);和int f(int,const int*);是相同的声明；
以下声明是一样的：
```cpp
int f(char s[3]);
int f(char[]);
int f(char* s);
int f(char* const);
int f(char* volatile s);
这两个也一样：
int f(int());
int f(int (*g)());
```
形参类型不能是包含引用或指向未知绑定数组的指针(包括此类类型的多层次指针/数组)的类型，也不能是指向形参为此类类型的函数的指针。

而指示可变参数的省略号不必在前面加逗号，即使它跟在指示参数包扩展的省略号后面，因此以下函数模板是完全相同的:
```cpp
template<typename ...Args> void f(Args..., ...);
template<typename ...Args> void f(Args... ...);
template<typename ...Args> void f(Args......);
```
使用这种声明的一个例子是std::is_function的实现
+ 4) 函数的定义
非成员函数定义只能出现在命名空间作用域中(没有嵌套函数)。成员函数定义也可以出现在类定义体中。它们的语法如下:
```cpp
attr(optional) decl-specifier-seq(optional) declarator virt-specifier-seq(optional) function-body
where function-body is one of the following

ctor-initializer(optional) compound-statement	(1)(初始化式）	
function-try-block	(2)	
= delete ;	(3)	(since C++11)
= default ;	(4)	(since C++11)
即：
1) regular function body
2) function-try-block (which is a regular function body wrapped in a try/catch block)
3) explicitly deleted function definition
4) explicitly defaulted function definition, only allowed for special member functions and comparison operator functions (since C++20)
attr(C++11)	-	optional list of attributes. These attributes are combined with the attributes after the identifier in the declarator (see top of this page), if any.
decl-specifier-seq	-	the return type with specifiers, as in the declaration grammar
declarator	-	function declarator, same as in the function declaration grammar above. as with function declaration, it may be followed by a requires-clause (since C++20)
virt-specifier-seq(C++11)	-	override, final, or their combination in any order (only allowed for non-static member functions)  notice
ctor-initializer	-	member initializer list, only allowed in constructors
compound-statement	-	the brace-enclosed sequence of statements that constututes the body of a function
```
eg:
```cpp
int max(int a, int b, int c)
{
    int m = (a > b)? a : b;
    return (m > c)? m : c;
}
// decl-specifier-seq is "int"
// declarator is "max(int a, int b, int c)"
// body is { ... }函数体是一个复合语句(由0条或多条语句组成的由一对花括号括起来的序列)，在进行函数调用时执行。
说明：
形参类型以及函数定义的返回类型不能是不完整的类类型，除非函数定义为deleted(自c++ 11以来)。完整性检查是在函数体的上下文中进行的，这允许成员函数返回定义它们的类(或其外围类)，即使它在定义点是不完整的(在函数体中是完整的)。在函数定义的声明符中声明的参数在函数体的作用域中。如果在函数体中没有使用参数，则不需要对其命名(使用抽象声明符就足够了)
void print(int a, int) // second parameter is not used
{
    std::printf("a = %d\n",a);
}
尽管形参上的顶级cv限定符在函数声明中被丢弃，但它们将形参的类型修改为函数体中可见的类型:
void f(const int n) // declares function of type void(int)
{
    // but in the body, the type of n is const int
}这样就防止一些函数体内的修改(主要是引用等时，定义者来根据这个判断)
```


+ 5) 删除的函数：
If, instead of a function body, the special syntax = delete ; is used, the function is defined as deleted. Any use of a deleted function is ill-formed (the program will not compile). This includes calls, both explicit (with a function call operator) and implicit (a call to deleted overloaded operator, special member function, allocation function etc), constructing a pointer or pointer-to-member to a deleted function, and even the use of a deleted function in an unevaluated expression. However, implicit ODR-use of a non-pure virtual member function that happens to be deleted is allowed.

If the function is overloaded, overload resolution takes place first, and the program is only ill-formed if the deleted function was selected.  
```cpp
struct sometype
{
    void* operator new(std::size_t) = delete;
    void* operator new[](std::size_t) = delete;
};
sometype* p = new sometype; // error: attempts to call deleted sometype::operator new

删除的函数定义必须是翻译单元中的第一个声明:之前声明的函数不能被重新声明为已删除:
struct sometype { sometype(); };
sometype::sometype() = delete; // error: must be deleted on the first declaration
```
+ 6) __func__
```cpp
Within the function body, the function-local predefined variable __func__ is defined as if by
k@VM-180-38-ubuntu:~$ g++ -o test test.cpp 
test.cpp:8:22: warning: ‘__func__’ is not defined outside of function scope
    8 | void f(const char* s=__func__);
      |                      ^~~~~~~~
k@VM-180-38-ubuntu:~$ ./test 
s:top level
k@VM-180-38-ubuntu:~$ cat test.cpp 
#include<iostream>
#include<stdio.h>

using namespace std;

//static const char __func__[]="functime name";

void f(const char* s=__func__);
void f(const char* s)
{
	printf("s:%s\n",s);
}


int main()
{
    f();
    return 0;
}
```


##### 枚举类型：
枚举类型是一种特殊的类型，他的值被限定在一个范围内，可能会包含几个显示的命名常量表，常量的值是整数的类型
定义语法：
```cpp
enum-key attr(optional) enum-name(optional) enum-base(optional)(C++11) { enumerator-list(optional) }
enum-key attr(optional) enum-name enum-base(optional) ; since c++11
enum-key: 枚举关键字： enum ,enum class 或 enum struct其中的一个，后两者since c++11
attr(c++11) : 可选，任意数量属性的序列号
enum-name: 枚举名称，

+ 大类型1：unscoped enum: 即是enum修饰的enum
```
+ 1）语法
```cpp
enum name(optional) { enumerator = constexpr , enumerator = constexpr , ... }	(1)	
enum name(optional) : type { enumerator = constexpr , enumerator = constexpr , ... }	(2)	(since C++11)
enum name : type ;	(3)	(since C++11)

enum Color { red, green, blue };
Color r = red;
switch(r)
{
    case red  : std::cout << "red\n";   break;
    case green: std::cout << "green\n"; break;
    case blue : std::cout << "blue\n";  break;
}
```
+ 2)  枚举中的值，隐式和显式指定
```cpp
enum Foo { a, b, c = 10, d, e = 1, f, g = f + c };
//a = 0, b = 1, c = 10, d = 11, e = 1, f = 2, g = 12
而且名字可以省略：
enum { a, b, c = 0, d = a + 2 }; // defines a = 0, b = 1, c = 0, d = 2
```

+ 3)  和int的隐试转换
```cpp
enum color { red, yellow, green = 20, blue };
color col = red;
int n = blue; // n == 21
```

+ 4）枚举和其他类型的显示转换：
整型、浮点型和枚举类型的值可以通过static_cast或显式cast转换为任何枚举类型。如果基础类型不是固定的，并且源值超出范围，则结果是未指定的(直到c++ 17)undefined(因为c++ 17)。(如果源值能够容纳足够容纳目标枚举的所有枚举数的最小位字段，则源值在范围内，如果源值为浮点数则转换为枚举的基础类型。)否则，其结果与隐式转换到底层类型的结果相同。
```cpp
enum access_t { read = 1, write = 2, exec = 4 }; // enumerators: 1, 2, 4 range: 0..7
access_t rwe = static_cast<access_t>(7);
assert((rwe & read) && (rwe & write) && (rwe & exec));
 
access_t x = static_cast<access_t>(8.0); // undefined behavior since C++17
access_t y = static_cast<access_t>(8); // undefined behavior since C++17
 
enum foo { a = 0, b = UINT_MAX }; // range: [0, UINT_MAX]
foo x= foo(-1); // undefined behavior since C++17, even if foo's underlying type is unsigned int
```
+ 5)  类中枚举的使用：
```cpp
struct X
{
    enum direction { left = 'l', right = 'r' };
};
X x;
X* p = &x;
 
int a = X::direction::left; // allowed only in C++11 and later
int b = X::left;
int c = x.left;
int d = p->left;
```

+ 大类型2：scoped enum:即是enum class 或enum struct修饰的enum
和unscoped enum的区别在于，修饰符号不同+不能和int做隐试转换+可以使用类的一些用法：
+ 1) 语法
```cpp
enum struct|class name { enumerator = constexpr , enumerator = constexpr , ... }	(1)	
enum struct|class name : type { enumerator = constexpr , enumerator = constexpr , ... }	(2)	
enum struct|class name ;	(3)	
enum struct|class name : type ;	(4)	
```
+ 2) 不能和int做隐式转换
```cpp
enum class Color { red, green = 20, blue };
Color r = Color::blue;
switch(r)
{
    case Color::red  : std::cout << "red\n";   break;
    case Color::green: std::cout << "green\n"; break;
    case Color::blue : std::cout << "blue\n";  break;
}
// int n = r; // error: no scoped enum to int conversion
int n = static_cast<int>(r); // OK, n = 21
```
+ 3) 如果满足以下条件，则unscoped和scoped都可以用以下初始化式：
初始化是直接列表初始化，  
初始化器列表只有一个元素，  
unscoped或scoped的枚举是固定的基础类型，  
转换是非缩小的  
```cpp
enum byte : unsigned char {}; // byte is a new integer type
byte b { 42 }; // OK as of C++17 (direct-list-initialization)
byte c = { 42 }; // error
byte d = byte{ 42 }; // OK as of C++17; same value as b
byte e { -1 }; // error
 
struct A { byte b; };
A a1 = { { 42 } }; // error (copy-list-initialization of a constructor parameter)
A a2 = { byte{ 42 } }; // OK as of C++17
 
void f(byte);
f({ 42 }); // error (copy-list-initialization of a function parameter)
 
enum class Handle : std::uint32_t { Invalid = 0 };
Handle h { 42 }; // OK as of C++17
```

+ 4) 作为一种类型，它也可以这样使用：
```cpp
std::ostream& operator<<(std::ostream& os, const Direction& rhs) {
  std::string dir;
  switch (rhs) {
    case Direction::NORTH: dir = "NORTH"; break;
    case Direction::EAST:  dir = "EAST";  break;
    case Direction::SOUTH: dir = "SOUTH"; break;
    case Direction::WEST:  dir = "WEST";  break;
  }
  return os << dir;
}
```
+ c++20开始，可以用using-enum-declaration:
```cpp
using enum nested-name-specifier(optional) name ;
eg:
enum class fruit { orange, apple };
struct S {
  using enum fruit; // OK: introduces orange and apple into S
};
void f()
{
    S s;
    s.orange;  // OK: names fruit::orange
    S::orange; // OK: names fruit::orange
}
名字冲突导致无法使用：
enum class fruit { orange, apple };
enum class color { red, orange };
void f()
{
    using enum fruit; // OK
    // using enum color; // error: color::orange and fruit::orange conflict
}
```

##### 联合：
共用体占用的内存等于最长的成员占用的内存。共用体使用了内存覆盖技术，同一时刻只能保存一个成员的值，如果对新的成员赋值，就会把原来成员的值覆盖掉。
eg:
```cpp
#include <iostream>
#include <cstdint>
union S
{
    std::int32_t n;     // occupies 4 bytes
    std::uint16_t s[2]; // occupies 4 bytes
    std::uint8_t c;     // occupies 1 byte
};                      // the whole union occupies 4 bytes
 
int main()
{
    S s = {0x12345678}; // initializes the first member, s.n is now the active member
    // at this point, reading from s.s or s.c is undefined behavior
    std::cout << std::hex << "s.n = " << s.n << '\n';
    s.s[0] = 0x0011; // s.s is now the active member
    // at this point, reading from n or c is UB but most compilers define it
    std::cout << "s.c is now " << +s.c << '\n' // 11 or 00, depending on platform
              << "s.n is now " << s.n << '\n'; // 12340011 or 00115678
}
output:
s.n = 12345678
s.c is now 0
s.n is now 115678
```
类和结构类型struct：和类相关，初始化，位运算

#### 更多类型：
c++的类型远不止这些，c++可以允许用户定义自己的类型；以及一些声明中出现的临时类型，总的来说：
c++允许这样定义类型：
```cpp
class declaration;
union declaration;
enum declaration;
typedef declaration;
type alias declaration.
还有一些在声明中出现的，c++把它们称为type-id:
在c++程序中，通常需要引用没有名称的类型;其语法称为type-id。id类型名称类型T的语法就是声明一个变量或函数的语法类型的T,标识符中,除了decl-specifier-seq type-specifier-seq声明语法的限制,并且可以定义新类型只有在id类型出现在右边non-template类型别名声明。
int* p;               // declaration of a pointer to int
static_cast<int*>(p); // type-id is "int*"
 
int a[3];   // declaration of an array of 3 int
new int[3]; // type-id is "int[3]" (called new-type-id)
 
int (*(*x[2])())[3];      // declaration of an array of 2 pointers to functions
                          // returning pointer to array of 3 int
new (int (*(*[2])())[3]); // type-id is "int (*(*[2])())[3]"
 
void f(int);                    // declaration of a function taking int and returning void
std::function<void(int)> x = f; // type template parameter is a type-id "void(int)"
std::function<auto(int) -> void> y = f; // same
 
std::vector<int> v;       // declaration of a vector of int
sizeof(std::vector<int>); // type-id is "std::vector<int>"
 
struct { int x; } b;         // creates a new type and declares an object b of that type
sizeof(struct{ int x; });    // error: cannot define new types in a sizeof expression
using t = struct { int x; }; // creates a new type and declares t as an alias of that type
 
sizeof(static int); // error: storage class specifiers not part of type-specifier-seq
std::function<inline void(int)> f; // error: neither are function specifiers
声明语法中删除名称的声明器部分被称为抽象声明器。
Type-id可用于以下情况:
to specify the target type in cast expressions;
as arguments to sizeof, alignof, alignas, new, and typeid;
on the right-hand side of a type alias declaration;
as the trailing return type of a function declaration;
as the default argument of a template type parameter;
as the template argument for a template type parameter;
in dynamic exception specification.
```
其他： 关于静态类型，动态类型和未完成的类型；
#### 其他：
c位段结构：
位段结构中位段的定义格式为：
```cpp
unsigned <成员名>:<二进制位数>
例如：
struct bytedata
{unsigned a:2;   /*位段a，占2位*/
 unsigned:6;  /*无名位段，占6位，但不能访问*/
 unsigned:0;     /*无名位段，占0位，表下一位段从下一字边界开始*/
 unsigned b:10;  /*位段b，占10位*/
 int i;          /*成员i，从下一字边界开始*/
}data;
```
位段数据的引用:
同结构体成员中的数据引用一样，但应注意位段的最大取值范围不要超出二进制位数定的范围，否则超出部分会丢弃。
例如：data.a=2;   但  data.a=10;就超出范围（a占2位，最大3）
应用：tcpip头等的应用 

