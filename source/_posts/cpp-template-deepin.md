---
title: cpp_template_deepin
date: 2021-01-10 00:23:38
tags: cpp_template
categories: c&cpp
---


### 模板是什么，为什么要引入模板：

模板是用来生成代码的，通过模板可以定义一组类型的共同行为；
为什么要引入模板：
继承和组合是实现重用代码的方法，而容器也是，为了实现能承载不同类型的容器，java等其他语言用所有类都继承于根类型等方式，<!-- more -->
而c++这里为了减少不必要的开支，和冗余，采用预定义等方式，在预处理和编译时，将T替换为实际类型参数，并生成对应的类型；
来从而实现了容器；
所以说：模板的引入，是为了实现容器的需求；

那么容器呢？为什么容器被需要？
栈的内存管理依赖于函数本身，或者说操作系统，在函数的调用结束后，会回收相关的存于栈的结构，所以不用我们去考虑清理的事情；
但是当我们在堆上使用时，malloc/new后，往往需要free/delete,在传统的c中，malloc后会需要进行free，否则程序运行时会出现内存泄漏；
容器的真正需求，是在这种情景下，减轻程序员的负担，担负起自动new和清理的工作；

c++是怎么做的？ c++标准容器，用new创建需要的对象，将其指针放入容器中，实际使用时取出并处理，这种方法创建的只是对象，
清理时依赖析构函数时进行合理的free；不过需要注意，当存储的对象是指针时，此时的指针需要程序员自己去new和释放；
同时为了支持承载多种类型的对象，所以模板就被创建出来；

### 模板的基本原理：
为了解决多类型：有几个方法，模板采用的是第三个：
1) c方法复制粘贴代码
2) 继承来实现代码重用，但是需要学习基础类库
3) 实现类似宏替换的逻辑，并放到编译器中，编译器识别到类似声明，就进行替换，从而重新生成类定义等，也取消类型的指定；而容器的实现则是
    以堆来存放一组特定类型对象。类似对象数组等；

### 模板是怎么工作的，工程上，内部结构等，编译器的作用；
    
#### 1）工程文件上，如何预编译，编译，链接等；
 模板中分为函数模板和类模板：
类模板：定义和成员函数实现都是写在头文件中
函数模板：定义声明等都是写在头文件中；
模板编译模型：
模板的完整定义都是放在每个编译单元中；例如完全放在单个文件程序中，或者放在文件程序的头文件中；和传统的编程方式背道而驰；
##### (1) 传统的为什么要这么做？---分离模型：
不要放置分配存储空间的任何东西(这条规定是为了防止在链接期间的多重定义错误)，编译期间是单个文件的，此时不会出现，但是链接的时候是多个实现文件，若是头文件里也定义了，就会导致链接的时候多重定义，而编译器对此并没有去重；

头文件中的非内联函数体会导致多函数的定义，从而导致链接错误；
隐藏来自客户有益函数实现，减少了编译时链接；
隐藏代码，代码所有权；
头文件越小，编译时间越短；
##### (2) 模板时包含模型，那这样客户代码想隐藏怎么办？
在template<..>后的任何东西都意味着编译器在当时不为它分配存储空间，而是一直处于等待状态直到一个模板示例告知，而此时是编译器在碰到模板示例时，往往是编译期间，然后生成对应的类，然后在运行时，才分配对象的空间；在编译器和连接器中有机制能去掉同一模板的多重定义；所以为了使用方便，几乎总是在头文件中放置全部的模板声明和定义；

模板代码本质上只是产生代码的指令，不是真正的代码，只有实例化了才是，一个编译器在编译期间看到模板的完整定义后，在同一个翻译单元中碰到模板实例化点时，也会在其他翻译单元碰到同样的实例化点，这样就会重复生成实例化代码；而编译器和连接器需要解决这个重复定义；
这种有两个缺点：
a  编译时间增加  b 无法隐藏实现代码；
如何处理？ 如何实现分离？
一种是显示实例化，一种是导出模板：
显示实例化：
	```cpp
	eg:
ourMin.h :
#ifndef OURMIN_H
#define OURMIN_H
template<typename T> const T& min(const T&,const T&);
#endif

ourMin.cpp
#include "ourMin.h"
template<typename T> const T& min(const T& a,const T& b)
{
     return (a<b)?a:b;
}

UseMin1.cpp:
#include<iostream>
#include"outMin.h"
void usemin1()
{
    std::cout<<min(1,2) << std::endl;
}

UseMin2.cpp:
#include<iostream>
#Include "outMin.h"
void usemin2()
{
   std::cout<<min(3.2 ,4.3) <<std::endl;
}

main.cpp;
void usemin1();
void usemin2();
int main()
{
    usemin1();
    usemin2();
    return ;
}
	```

建立这个程序式，连接器报告有未解析的min<int>() 和min<double>()的外部引用； 因为编译器在min的特化时，只有min的声明可见，定义不可见，编译器认为它可能来自于其他单元，所以即没有实例化，问题留给了连接器，连接器无法找到；
所以可用加一个显示实例化来进行，即显示特化：
```cpp
Minstances.cpp:
#Include "ourMin.cpp"//因为编译器需要模板定义来实例化；
template const int& min<min>(const int&, const int&);
template const double& min<double> (const double &,const double&);
```

导出模板：
export关键字： 

 ```cpp
 export template<typename T> const T& min(const T&,const T&);
```

#### 2)模板定义的几种方式：
声明和内联函数的形式
```cpp
template<class T>
class AA
{
   public :
       AA (){};
       void add() {...};
};


声明和非内联函数的形式；
      template<class T>
      class Array {
          enum { size = 1000};
          T A[size];
          public:
               T& operator[](int index);
    };
       template<class T>
      T& Array<T>::operator[](int index) {
               .....
              return A[index];
     }
```

声明在头文件，定义在cpp的显示实例化，注意需要cpp加显示实例声明，见上面2

#### 3）模板的一些使用技巧： 在stl源码解析中：
     涉及以下几种： 类型萃取，迭代器，智能指针(引用释放等) ,泛型算法等等；



### 模板的使用细节：
 
#### 1) 模板参数： 类型(基础或用户自定义), 编译时常数值(整数，指针或某些静态实体的引用，通常作为无类型模板参数，支持默认参数),其他模板；
模板类型参数：
```cpp
template<class T>
class Array{
     ....
};
template<class T, template<class> class Seq>
class Container
{
    Seq<T> seq;//通知编译器，Seq是一个模板；本例子中Seq代表Array
    ...
};
```
```cpp
使用：Container<int,Array> container;
还可以支持标准库中的容器：
template<class T, template<class U,class = allocator<U>>  class Seq>
class Container
{
    Seq<T> seq;
    ....
}
```

使用： Container<int ,vector> xxx;实际上容器适配器就是用类似的方法实现的，如stack
typename关键字用法；
```cpp
当在模板中用T::id这种类型时，编译器会默认解析为T类中的静态成员id,而不会认为这个是一个内部类，所以，当用法为：
T::id i ;这种定义变量的方式时，会出错，此时需要向编译器说明这个是一个嵌套类；
所以typename在这里的作用： 1) 声明是一个类型，2）可以替换class template< typename T> class X{};
typename并不能起到定义新类型的作用，可以用typedef typename Seq<it>::iterator It;类似的 
```

template关键字的作用：
1）声明模板
2）模板中遇到> <等和模板的> <混合时，用template声明；

#### 2)成员模板；
就是在类模板中定义一个新的内部类模板：
eg:
```cpp
template<class T> class Outer{
     public :
         template<class R> class inter{
            public:
                 void g();
       };
};
template<class T> template<class R>
void Outer<T>::inter<R>::g() {
      ..
}
使用：
 Outer<int>::inter<bool> interr;
```

#### 3) 有关函数模板的内容
1) 函数模板定义了一簇函数； ---函数模板参数的类型如何推断：涉及一些参数可以省略的问题
2) 函数模板重载：其实是直接定义了普通函数，这样若是符合普通函数的类型则调用的是定义的普通函数，否则是模板生成；
3）以一个已生成的函数模板地址作为参数：

```cpp
template<typename T> void f(T*) {}
void h(void (*pf)(int*)) {}
template<typename T>void g(void (*pf)(T*)) {}

int main
{
    h(&f<int>);
    h(&f);
    g<int>(&f<int>);
    g<int>(&f);
}
```

4)将函数用到stl序列容器中：TODO
5）函数模板的半有序： 即T, T*,const T*的区分，优先匹配特化程度最高的那个模板；他们的特化程度逐渐递增；

#### 4)模板特化相关
显示特化：
```cpp
template<> 
const char* const& min<const char*>(.....)
```

半特化：
比如有两个参数，只限定了其中一个类型；
防止代码膨胀-- TODO

#### 5)模板中的名称查找问题
编译器解析模板定义，并寻找明显的语法错误，还要对其所能解析的所有名称进行解析；对于不依赖模板参数的名称，编译器使用普通名称查找解析他们，不能解析的就是关联名称，只有等到实例化才知道；
模板和友元：  --TODO

#### 6)模板编程中的习语:
1) 特征：将与某种类型相关联的所有声明绑定在一起的实现方式；
2）策略： 其实就是类似萃取类型；
3）神奇的递归模板，在编译期间就算出来值了；运行时只需要读取即可

#### 7)模板元编程：
1） 编译时编程 :模板中编译时循环，循环分解，编译时选择，编译时断言---即利用模板，在编译时就算出值，减少运行开销哈
2)  表达式模板：

#### 8)模板与继承： 模板实例也可以作为被继承方；
```cpp
class xx: public tes<A> {};
模板其他资料；见书：c++ templates,the complete guide
```

### 使用技巧

懒惰初始化，使用时才分配空间，读写时分配
存放指针对象时，为了避免多重释放，可以实现所有权函数，拥有所有权的才有释放的权利 owns函数

### ref:

c++编程思想；


