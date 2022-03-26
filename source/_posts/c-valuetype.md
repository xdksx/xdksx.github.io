---
title: c++_valuetype
date: 2021-05-23 01:35:24
tags: c++_valuetype
categories: c++
---



### 彻底理解c++中的五种值，临时对象，移动语意，引用指针，const等的概念和关联
#### c++的引用和指针
##### 引用：
引用，是变量或对象的别名，这么解释其实还是有点抽象的；
如果是指针，则好理解，是对象的虚拟内存地址；所以在赋值拷贝等容易有实体理解；而引用，在这些常见下，又是什么样的实际操作？
考虑如下例子：<!--more-->
```cpp
为了理解引用是什么，来看这个例子：

leaq是只移动地址，或者说直接值，而movq是移动值作为地址的内存的存放的值，或者寄存器存放的值
258     xorl    %eax, %eax                                                    |14 class A
259     //为x1分配栈地址，将地址放到寄存器rax                                 |15 {
260     leaq    -44(%rbp), %rax                                               |16 public:
261     //将rax寄存器的值，即x1地址给到rdi(调用函数的参数0)                   |17    int a;
262     movq    %rax, %rdi                                                    |18    A() { std::cout<< "A con" << std::endl;}
263 .LEHB3:                                                                   |19    A& operator=(const A& other)
264     //调用A构造函数                                                       |20    {
265     call    _ZN1AC1Ev                                                     |21     ¦  a = other.a;
266 .LEHE3:                                                                   |22     ¦  std::cout<< "use operator =" << std::endl;                                     
267     //将34赋值给x1的内存地址                                              |23     ¦  return *this;
268     movl    $34, -44(%rbp)                                                |24    }
269     //将x1地址给到rax寄存器                                               |25    A(const A& aa) { a=aa.a; std::cout<< "copy con" << std::endl;}
270     leaq    -44(%rbp), %rax
        //将rax寄存器的值即x1的地址给到-32(%rbp)即x2                          |26    ~A() { std::cout<< "A dec" << std::endl;}
271     movq    %rax, -32(%rbp)                                               |27 };
272     //将x1的a取值赋值给寄存器                                             |28 
273     movl    -44(%rbp), %eax                                               |29 int main()
274     //将值赋值给dds，dds的内存地址就是-36(%rbp)                           |30 {
275     movl    %eax, -36(%rbp)                                               |31     A x1;
276     leaq    -44(%rbp), %rdx                                               |32     x1.a = 34;
277     leaq    -40(%rbp), %rax                                               |33     A &x2 = x1;
278     movq    %rdx, %rsi                                                    |34     int dds = x1.a;
279     movq    %rax, %rdi                                                    |35     A x3 = x1;
280 .LEHB4:                                                                   |36     std::cout<< "x1:" << x1.a << " addr:"<< &x1 <<std::endl;
281     call    _ZN1AC1ERKS_                                                  |37     std::cout << "x2:" <<x2.a << " addr:" << &x2  <<std::endl;
282 .LEHE4:                                                                   |38     x2.a = 321;
283     leaq    .LC4(%rip), %rsi                                              |39 
284     leaq    _ZSt4cout(%rip), %rdi                                         |40 
285 .LEHB5:                                                                   |41 
286     call    _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@PLT   |42     return 0;
...
//前面知道x2就在-32(%rbp)的位置，所以这里是取这块内存的值放寄存器
220     movq    -32(%rbp), %rax
//将321放到上面的值为地址的值，即x1的值被改了；
221     movl    $321, (%rax)
 //这里 x1在-44(%rbp),这个地址存放它的值，而x2是在-32(%rbp)上存放x1的地址；
//所以通过x2能找到x1,且修改其值时，编译器会转换为修改其值指向的地址上值，
//这样x1也会发生变化
//所以其实引用就是在一个内存位置上放一个相同的地址值，指向所引用的对象；

```

所以对于汇编来讲，引用和指针本质上是一样的，都是存放变量的地址，只是编译器对引用做了封装和限制，不允许像操作指针那样操作引用，而实际上
修改引用的值，是修改对应位置的值，所以会影响源对象，而将引用赋值给另一个变量，是将其值赋值过去，而不是将地址赋值过去；


##### c++的左值引用和右值引用
左值引用，和指针没啥太大本质差别，给程序员的权限比指针少了些，但是也是很多的；而右值引用，是编译器对临时对象的权限的释放，即允许程序员能有一定的控制临时对象的权利；
所以先来看临时对象；可能一切都是由临时对象惹得锅；

#### c++临时对象
我们知道，如果是c语言，是很少有临时对象这些说法的，因为c没有类和对象的概念，即使是struct，在赋值，构造的时候，也不需要调用构造函数，一般是映射过去；
而c++不一样，c++有了类和对象，和基础类型不同的是，对赋值传值等场景下，需要调用拷贝函数等，而且拷贝函数的参数等不能像基础变量那样，直接传递值到寄存器，往往涉及到
要用内存作为跳板，比如c=a+b，当它们是基础类型如int时，a+b的值甚至可以存在寄存器中，再赋值到c对应的内存中；而如果是对象，那a+b时，要调用operator+ 函数，结果
要构造一个临时对象来赋值给c，这个时候要调用赋值函数；

而临时对象的出现，伴随着即时的构造和销毁，会带来相应的开支，效率降低；

在一些场景下，对临时对象的某个操作可以提高效率，减少开支；如在赋值时，不是去做将临时对象拷贝到新对象上，而是将临时对象的地址赋值到新对象上，称为移动；
这样会少一次拷贝，提升效率；此为移动语意；

伴随着这种提升的出现；面临着一个问题，就是哪些可以用来移动，哪些只是临时对象，完全由编译器去控制？ 编译器可能不够清楚，需要程序员去提示它需要移动，不能立刻销毁；
要延长寿命； 传统的左值右值无法精确区分，所以为了用户明确表示它要移动，编译器制定了规则，用来区分，于是就有了下面的五种值；

#### c++的五种值：lvalue,xvalue,rvalue,gvalue,prvalue
             expression
          /       \
    glvalue       rvalue
       /      \      /      \
lvalue         xvalue        prvalue
所以实际上是三种值，只是又将三种值分为两类，为了简单，直接看这三种值：

主值类别对应于表达式的两个属性：
具有标识 ：可以通过比较对象的地址或它们标识的功能（直接或间接获得）来确定该表达式是否与另一个表达式引用相同的实体；
可以从以下位置移动：移动构造函数，移动赋值运算符，或实现移动语义的另一个函数重载可以绑定到表达式。

表示为：
具有身份并且不能被移走的被称为左值表达式 ; lvalue，即编译器不会自动帮你销毁它，除非到达了作用域结尾；
具有身份并可以移出的称为xvalue 表达式 ; xvalue,编译器以此知道它不能立刻被销毁，直到你移动了它；所以它叫expiring值，你移动了它，它就亡了
没有身份且可以移出的被称为prvalue 表达式 ; prvalue，完全由编译器去控制，用户几乎无感知它的生命周期；
没有身份并且无法从中移走的人不会被使用。

#### c++右值引用和五种值的关系
右值引用，可以说是对应了xvalue了，左值引用是lvalue,因为修改它，就像修改左值一样；而临时对象prvalue,没有引用之说，你不能引用它，除非你用它构造一个右值引用或左值引用；
它就是一个单纯的值，只在内存存在很短的时间；

#### c++临时对象，右值引用，和五种值的关系
到这里，我们可以知道临时对象，是五种值来源的关键，而右值引用，是对临时对象带来的问题的解救；



#### const &和& 的区别和使用
看下面这个例子，你会发现对汇编来讲，其实是一样的，但是编译器会检查你的行为，直接在编译阶段过滤掉你尝试对const的改动；

```cpp
| 28     movl    $3, -28(%rbp)                                                              |    19 int main()
| 29     leaq    -28(%rbp), %rax                                                            |    20 {
| 30     movq    %rax, -24(%rbp)                                                            |-   21     A x1;
| 31     leaq    -28(%rbp), %rax                                                            ||   22     x1.a = 3;
| 32     movq    %rax, -16(%rbp)                                                            ||   23 
| 33     leaq    .LC0(%rip), %rsi                                                           ||-- 24     const A &x2 = x1;
| 34     leaq    _ZSt4cout(%rip), %rdi                                                      ||-- 25     A &x3 = x1;                                                                       
| 35     call    _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@PLT                ||   26     std::cout<< "x1.a " << x1.a << std::endl;
| 36     movq    %rax, %rdx                                                                 ||   27 
| 37     movl    -28(%rbp), %eax                                                            ||   28     return 0;
                                                        
```

#### move移动语意
看下面的例子就知道了，move只是对封装了下，和const &的引用类似；
```cpp
 49     movl    $3, -20(%rbp)                                                              |     9 int main()
| 50     leaq    -20(%rbp), %rax                                                            |    10 { 
| 51     movq    %rax, %rdi                                                                 |-   11     A x1;
| 52     call    _ZSt4moveIR1AEONSt16remove_referenceIT_E4typeEOS3_                         ||   12     x1.a = 3;
| 53     movq    %rax, -16(%rbp)                                                            ||   13                                                                                       
| 54     leaq    .LC0(%rip), %rsi                                                           ||-- 14     A &&x2 = std::move(x1);
| 55     leaq    _ZSt4cout(%rip), %rdi                                                      ||   15     std::cout<< "x1.a " << x1.a << std::endl;
| 56     call    _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc@PLT                ||   16 

```


#### 一些例子和基础使用：
+ 左值引用可以被修改： 很明显就像指针；
+ 右值引用不能被修改？ 为什么?
我们来看一个例子：

```cpp

class A
{
public:
    int a;
};
                                                                                            ||   22     A x1;
                                                                                            ||   23     x1.a = 3;
| 24     subq    $32, %rsp                                                                  ||-- 24     A&& x2 = A();                                                                     
| 25     movq    %fs:40, %rax                                                               ||   25     //A&& x2 = std::move(x1);
| 26     movq    %rax, -8(%rbp)                                                             ||   26 
| 27     xorl    %eax, %eax                                                                 ||   27     std::cout<< "x1.a " << x1.a << std::endl;
| 28     movl    $3, -24(%rbp)                                                              ||   28 
| 29     movl    $0, -20(%rbp)                                                              ||   29     //A x3;
| 30     leaq    -20(%rbp), %rax                                                            ||   30     //x3 = x1;//会默认将x1弄为const &传递进去
| 31     movq    %rax, -16(%rbp)                                                            ||   31     return 0;
| 32     leaq    .LC0(%rip), %rsi                                                           |    32 }

从上面汇编可以看到，3被直接赋值到-24(%rbp)  即是x1.a=3,这里没有显示定义构造函数，所以没有调用，所以，如果可以，就不要声明构造函数了，这样减少开销；
接着将0 给到-20(%rbp),相当于在构造临时对象A(),接着将A的地址给到-16(%rbp)，即就是延长了A()临时对象的寿命，后面可以通过右值引用使用A()(不可写)
所以右值引用，是一个临时对象的地址；当我们将左值赋给右值引用时，会出现问题，显然编译器不允许这样操作；

这里好像还是没有解释为什么右值引用不能被修改： 这里明显是编译器的行为了;实际上它也是内存中的一块，但是编译器不允许你写；当检测到有写的操作时，会报错；
即使是间接的，比如试图把const指针赋值给指针，企图拿A()的地址等；
```

+ 其他的理解：
1) 对vector的emplace_back，有如下几种情况：
(1) emplace_back(A()) //将A的内存地址直接move；A()会被清理
(2) emplace_back(x) //x是基础变量，则x的地址被拷贝；
(3) emplace_back(3) //和push_back(int{3})没有区别，都需要再开辟一个内存地址,注意参数的不同
所以emplace_back要比push_back快一倍几乎；

+ 各种例子：
```cpp
struct X { int n; };
extern X x;

4;                   // prvalue: does not have an identity
x;                   // lvalue
x.n;                 // lvalue
std::move(x);        // xvalue
std::forward<X&>(x); // lvalue
X{4};                // prvalue: does not have an identity
X{4}.n;              // xvalue: does have an identity and denotes resources
                     // that can be reused
```
其他例子：ref:
https://www.geeksforgeeks.org/understanding-lvalues-prvalues-and-xvalues-in-ccwith-examples/
https://en.cppreference.com/w/cpp/language/value_category



