---
title: cpp_conandde
date: 2018-06-09 11:03:35
tags: cpp_class
categories: c&cpp
---
## c++ class constructor and destructor
### 构造函数
#### 构造函数表现
##### 构造函数的作用：
构造函数主要是用来初始化对象的－－－一般是成员，函数不用
--所以它需要在构建对象时就执行<!--more-->
##### 构造函数如何写：
```cpp
public:　classname(arg...):member(arg),mem2(arg),..{xxx}
    classname(..){..}
```        
##### 默认构造函数的生成规则（深入对象模型）
+ 下面四个情况下会生成默认构造函数（编译器控制）(nontrivial default constructor)  
 + 带有default constructor的member class object  
 即在类中带有对象成员，该对象成员所属的类有构造函数
如：　
```cpp
class Foo {public :Foo(),Foo(int)...}
class Bar{public:Foo foo;char *str};
                          void funv()
                          {   Bar bar;....
                          }```
这种情况下若程序员没有实现构造函数，则编译器会自己生成构造函数，来调用Foo的构造函数
但是不初始化str,这个得由程序员来做。
如类似于：
```cpp
Bar::Bar(){
   foo.Foo::Foo();
 ```                  
 若程序员自己写了构造函数但是不包括初始化foo则，编译器会在程序眼写的构造函数中加入上述，另外若有多个对象
则按照声明顺序进行调用他们的构造函数；
 + 带有Default constructor 的base class  
 同理，当继承的基类带有构造函数时就会加入，若程序员的构造函数没有初始化他们时就会扩展  
//以上见例子class_constructor.cpp，
 + 和vcirtual相关need to create vptr  
    带有一个virtual　func的class  
     1)class　声明或继承一个virtua func  
     2)class派生自一个继承琏，其中有virtual base classes       这个比较容易理解
                
 + 和virtual相关 need to init vptr
   带有一个virtual　base class 的class  
   如：
   ```cpp
       class X{  public int i;}  
       class A :public virtual X {///
       class B :public virtual X {
       class C: public A,public B ```
       

##### 构造函数何时被执行  
-在对象定义时若有构造函数，则会执行
##### 实践：
1:5构造函数的使用：重载，初始化式

#### 构造函数表现的原理
##### 构造函数在静态代码块中的位置和符号体现
如果有定义构造函数，它本身也是一个函数，放在代码段中，可以通过kdbg查看
```c
origin(int ax=3,int bx=4):a(ax),b(bx){}
0x400bae push   %rbp
0x400baf mov    %rsp,%rbp
0x400bb2 mov    %rdi,-0x8(%rbp)
0x400bb6 mov    %esi,-0xc(%rbp)
0x400bb9 mov    %edx,-0x10(%rbp)```
##### 构造函数在动态执行时，放在哪个内存段中，如何被引用，使用
+ 动态执行时，在代码段中，通过this引用
+ 构造函数总结：任何c++ class只有在以上四中情况下才会产生默认构造函数，且产生的默认构造函数不会去初始化成员的值
### 拷贝构造函数
#### 拷贝的动作发生了什么
拷贝的本质，为什么需要拷贝构造函数？
+ 首先对基本的类型而言，拷贝是把值赋给另一个同类型的变量，而数租不能做=赋值拷贝
+ 对对象而言，也是把值赋给同类型的对象，而不是共享一个指针，所以这个过程需要将对象中的成员也复制过去（复制的是当前对象在此时时成员的值　）
+ c++中的拷贝构造函数针对三种行为：＝　，f(T t)和　f(){T t;return t}返回对象－－这三种情况都针对左值的　　　
+ =:注意这个是在定义时做的，如origin or1=or2;此时会调用"拷贝构造函数"（　同or1(or2))
　or2=or3;此时不会调用拷贝构造函数
（当程序员自己写了拷贝构造函数时，默认调用该构造函数，而不会区做成员复制，所以若在其中没有做复制，可能得到不想要的值，见例子)

#### 拷贝构造函数的作用和使用
##### 什么情况下会生成默认的拷贝构造函数？
类似于构造函数，在以下情形会生成默认的拷贝构造函数－
+ 当需要调用别的拷贝构造函数时，如：当类内含有一个member object而后者的class声明有一个copy　constructor时；
+ 当类继承自一个base class而后者有拷贝构造函数
+ 当类声明了一个或多个virtual functions时
+ 当class 派生自一个继承　串琏，而其中有一个或多个virtual base classes 时

#### 拷贝构造函数和编译器－－－汇编，转换：
分三种情况讨论：
+ 初始化拷贝构造：
```cpp
                   X x1(x0);
                   X x2=x0;
                   X x3=X(x0);```
   上述三种都是定义一个类，即定义的本质会在内存中开辟空间
上述三个都会执行拷贝构造函数，如何执行？  
会被转换为：伪代码
        X x1;
        x1.X::X(x0);
        会调用X::X(const X& xx) 
        x2,x3也是这样，将拷贝方作为函数参数传入
        这样就可以解释为什么拷贝构造函数的定义是      
        classname (const classname &obj)
+ 参数的初始化
即传入一个参数给函数：
       foo(X x)
如：
       X xx;
       //,..
       foo(xx);
       则会产生一个临时的对象：
 伪代码
        X __temp0;
        _temp0.X::X(xx);//use copy construtor
         foo(__temp0)
这里因为它是临时的，所以则定义的时候需要用引用foo(X &x)//这里不是很清楚为什么用引用～！，是否后面的操作直接面向该对象，需要做实际的修改，所以。。
所以，为了对付临时的对象，在执行完函数后，需要调用该对象的析构函数
+ 返回值的初始化：  
如
       X bar(){
                      X xx;
                       //...
                       return xx;
            }
         如何做X xxx=bar();如何拷贝的？双阶段初始化：
         a 增加一个额外的引用参数给函数，如void bar(X＆　_result)
         b 在return 前插入一个copy constructor 
             void bar(X &__result){
                       X xx;
                       xx.X::X();
                       __result.X::XX(xx);
                       return ;
               }
        所以上述会被转化为：
        Ｘ　xx=bar()  --->  X xx ;//注意这里不会执行默认构造函数　　bar(xx);
            ex:bar().memfunc()--->X __temp0;(bar(__temp0),__temp0).memfunc();
            X (*pf)();pf=bar;--->void (*pf)(X&);pf=bar;
            
##### 关于上述三种情况的优化：
１)在使用者层面上优化；２）在编译器层面的优化（ＮＲＶ）具体见深入模型（书）
##### 关于该不该编写copy  constructor: 
除非需要大量的memberwise 初始化操作，如传值给object以便做NRV优化，否则不需要
//上述情况的检验可以通过代码，或者去看编译器的生成代码～    
##### 拷贝构造函数的内存
放在代码段，
### 初始化队列
即构造函数的一种形式如：X(int f):a(ax),b(bx)..{....}
+ 问题：什么时候用初始化列表？它和初始化赋值有什么不同？
        有以下四种情况需要使用初始化列表：
        １）当初始化一个reference member时
        ２）当初始化一个const member时
        ３）当调用一个base class的constructor,当它拥有一组参数时
        ４）当调用一个member　class 的consructor，而它有一组参数时；
如
``cpp
   class world {
          String _nhame;
          int cnt;
          public :world(){_nhame=0;cnt=0;}}```
//此时，对_nhame这个成员，需要做较多的事，如产生临时对像，调用operator=....
   －－－－所以想到用初始化列表：
```cpp 
          world::world:nhame(0){
            cnt=0;
            }
            这样只会调用nhame的构造函数
            会被转换为：world::world{ //伪代码
             _nhame.String::String(0);
              cnt=0;
              }```
+ 更具体的，初始化列表如何进行：顺序如何：如前，初始化列表最终会被转换为初始化表达式到构造函数体中，那是什么顺序呢？  
注意这里的初始化顺序是根据成员在类中的定义（声明）顺序来的，而不是初始化列表中的顺序
看这个例子：
```cpp 
       class X{
                    int i; 
                    int j;
                   public:
                     X(int cal):i(j){}
                     ...
                     此时，因为i先初始化,再j,出错，i需要j```
                     
  ---->可以改善为：X::X(int cal):j(cal){i=j;}   
   ----为什么这样可以？  
   因为初始化列表的内容会放到初始化体中的表达式前面，这样j就会先定义
        －－－－》另一个可能出错的例子：
        　　　X::X(int cal):i(xfoo(cal)),j(cal){}
        -->转换为：X::X(/*this pointer*/ int cal){
                      i=this->xfoo(cal);
                      j=cal;
                      }
            这个情况在这里是对的，但是当X 继承自某个类，在xfoo是基类的函数，，则会出错this->xfoo

#### 几个问题：
+ 较为简单的例子见文件中的例子
+ 当程序员定义了拷贝构造函数时，还会执行memberwise initialization吗　－－会执行拷贝构造函数
参考：深入c++对象模型和http://en.cppreference.com/w/cpp/language/copy_constructor
