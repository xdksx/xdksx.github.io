---
title: cpp_runtime_semantis
date: 2018-07-29 17:54:07
tags: cpp_class
categories: c&cpp
---
### c++ 执行期语意学
#### 执行期和编译期的理解
+ 执行期：此时是对已经编译等生成的可执行文件装载到内存并调用cpu将其作为一个进程执行的过程，对c.c++来说程序的入口为main,即第一条指令是执行main函数开始的，而c++可能会加入一些额外的代码，所以实际写的第一条语句和执行的第一条语句可能有偏差；　　　<!--more-->
+ 在执行的过程，是程序代码运行的过程，可以想象为工厂开始生产，此时需要空间来运行，需要生产线，产品，工人，工人们走来走去搬运物品等；对程序的运行而言，此时主要的活动空间为栈和堆，即在栈和堆分配空间，而流程制式就是代码段，静态变量的数据区可以比喻为整个工厂共有的数据等；　这个过程中栈会被不断生成消失，堆也是，承载他们的是内存对应的区域；执行完后就从内存消失；
+ 　编译期，就像生孩子之前的扫描，检查，看看语法对不对，添加额外的内容(c++).生成对应的汇编代码和二进制代码，而程序员需要尽力　编译出高效，整洁，可读等特性的代码，就像这个过程中通过调理，吃合适的东西等；尽可能避开一些坑和耗时的行为le.
+ 　这部分的内容并不涉及太多执行期的，而更多的是编译器在编译代码时做了什么手脚
#### c++中的运算符函数和运算符语法糖
+ 运算符被编译器转换为运算符函数
操作符函数例子-->到临时对象的产生和销毁带来的效率问题
-->如何在程序中尽量避免产生临时变量和调用析构函数  
例子：  
```cpp
X xx;
Y yy;
if(yy==xx.getvalue()) ``` 
其中涉及到yy的==运算符函数和xx的getvalue函数，前者为：  
bool operator==(const Y&) const后者为X getvalue()  　　
yy==xx.getvalue()被转换为yy.operator==(xx.getvalue),显然类型不相符；  
而此时若X有函数operator Y()const;//conversion运算符，则进一步转换为：  
     yy.operator==(xx.getvalue.operator Y())  
 这行代码看上去是这样简单，但是实际上需要产生中间变量，转为伪代码:  
   X　tmp1=xx.getvalue;//放返回值  
   Y tmp2=tmp1.operator Y()//同上  
   int tmp3=yy.operator==(tmp2);//放置返回值  
   总共产生三个临时变量，而且还得析构，麻烦效率低  
   (注意，上述的为什么不能直接连锁调用?因为返回的是值而不是指针，思考this指针的连锁操作，cout的连锁操作，个人思考，应该是因为返回的是指针，上述返回的是值，所以无法用值调用下面的函数）


#### 对象的构造和解构
+ 构造函数在哪里被安插:  
构造函数在编译时，由编译器在合适的地方安插，一般情况下，正如我们想像的一样：在定义对象时会执行构造函数,解构在对象销毁时;
//c++伪代码  
```cpp
{
   Point point ;
   //point.Point::Point() 一般而言会被安插在这里
   ...
   //point.Point::~Point() 一般而言会被安插在这里
   }
 ```
+ 解构函数在哪里被安插:  
1、解构函数的安插需要考虑程序的退出时间（或者某个代码块的退出时间，在可能退出的地方都要加解构函数
```cpp
{
  Point point;
  //constructor here
  switch (int (point.x()) {
  case -1:
    //mumble;
    //destructor here
    return ;
  case 0:
    //mumble;
    //destructor here
    ....
    default :
      //mumble
      //destructor here
      return ;
     }
    destructor here
    }
  ```
  

+ 解构代码应该在任何可能退出代码块的地方，return等，switch,if，goto等都会使加上解构函数的调用以避免出现退出但是还没有执行析构函数的尴尬；
+ 而在如下例子：
```cpp
{
   if(cache)
      return 1;
   Point xx;
   if(xx.get()==1)
   return 1;
   }
   和
   {
     Point xx;
     if(cache）
       return 1;
       ...
   }
   ```
   前者不需要在if(cache）的return 前加解构和　构造函数，而后者需要，显然后者综合效率更差些；
   所以设计c++代码时候需要考虑，尽量在使用它的附近定义它
   
+ 对特殊情况的考虑--全局对象的构造函数和解构函数的安插，有特殊的处理    
"前面看到的是正常的局部情况，现在考虑的是全局对象，定义在main外面，它的构造函数被安插在哪里，什么时候执行?"
考虑以下例子：  
```
  Matrix identity;
   main ()
   {
      //identity 必须在这里被初始化
      Matrix m1=identity;
      ...
      return 0;
   }
   ```
   很明显，c++ 必须保证第一次用到identity把他构造出来而在main结束前销毁它；对全局对象而言，有构造函数和析构函数时，称为静态的初始化和内存释放操作；
   全局对象和全局变量一样被放在数据段(data segment),在c中，可以在编译期间给定全局变量常量值，而c++中的全局对象需要程序激活后才能执行构造函数给初值；相当于给全局对象做静态初始化；
```
  //例如,cfront在执行前，加入_main来初始化各种全局对象；
  //sti_xxx---static initialization
   int main()
   {
      _main();---(_sti_xxx(); _sti_xxx();,,,)
      ...
      ...
      _exit()---(_std_xxx()....)
      //而在结束时调用他们的析构函数
      }
      但是需要收集程序中各个对象文件的_sti函数和_std函数，此时可以用nm命令，即它会倾倒出符号表项目，nm会施加到.o文件上；搜寻_sti _std开头的函数；最后总结整理出来；
```

+ 对特殊情况的考虑--局部静态对象
考虑局部静态对象只会构造一次和销毁一次，却是可能调用多次包含定义局部静态对象的函数，如何保证只会构造一次和销毁一次呢?
``` 
   const Matrix&
   identify() {
      static Matrix mat_identify;
      //....
      return mat_identify;
   }
  //1\首先要保证在调用该函数才初始化局部静态变量，
  //2、其次，保证多次调用该函数不会重复初始化对象；
```
简单的说，解决方案就是用一个标志变量，当已经初始化一次局部静态变量就置为真；
+ 对象数组什么时候构造和解构?
考虑一下定义了一个对象数组，之后未做任何改动，要取其中的值，会在定义数组的时候也初始化数组中每一个对象（即调用构造函数)吗？
```
class Point{
   public:
      int a;
      Point (){a=3;}
 }
Point knot[10];
cout<<knot[3].a;//会打印3
}
```
当这种对象数组没有默认构造函数和析构函数，则定义时和内置类型相同，只需配置足够的内存保存即可；
而当对象有构造函数和析构函数时，编译器提供了vec_new() vec_delete()之类的函数来统一做构造和析构

```
void *vec_new(void *array,//数组起始地址
      size_t elem_size,int elem_count,//对象大小和数组对象个数
      void (*constructor)(*void)
      void(*destructor)(*void)
      而实际上调用时：
      Point knots[10]
      //可能是这样调用，delete类似
      vec_new(&knnots,sizeof(Point),10,&Point::Point,0)
```
而如果程序员额外调用了其中一些元素的构造函数，则：
```
Point knots[10]={Point(),Point(1.8,2.1,0.2),-1.9};
类似这样，则可能会明确的初始化前三个元素，后面的其他则用vec_new
```


+ 默认构造函数和数组

#### new和delete运算符
上述是针对对象的，new和delete是针对指针的；

+ 本质上调用malloc函数和free函数,编译器解析new，delete会安插构造函数和解构函数
new 的实际过程：

```cpp
对内建类型
int *pi=new int(5);
//1 调用函数库的new:_new 
//int *pi=_new(sizeof(int));
//2 设置初值：*pi=5;
//或加条件：
(int  *pi ; if(pi=_new(...))*pi=5)
//delete类似；
对对象：
Point3d *origin =new Point3d;
转换为：
Point3d *origin;
//伪代码
if(origin =_new(sizeof(Point3d))
origin=Pointed::Point3d(origin)//注意这里会调用构造函数

delete origin;
转换为：
if (!origin !=0) {
//伪代码
Point3d::~Point3d(origin);
_delete(origin);
}

而new 一般由malloc实现，delete由free实现；
extern void operator new(size_t)
{
  if (size =0)
    size=1;
  void *last_alloc
  while(!(last_alloc=malloc(size)))
  {
   if(_new_handle)
   (*_new_handle)();
   else 
      return 0;
   }
      return last_alloc
}
      
 ```


+ 针对数组的new和delete
实际上的new数组，若不存在构造函数，则只会做new的运算符函数：int *parray=(*int)_new(5*sizeof(int));
待续，略复杂。。。
+ placement operator new的语意-——new的重载；

#### 影响c++效率因素之一---临时性对象
+ 为什么需要临时性对象
临时性对象是影响程序效率和引入bug的来源之一；
  + 隐式的类型转换需要临时性对象：
  当用内建类型写下：
 ```cpp
    int a,b,c;
    ....
    a=b+c;//内建类型将算出的值赋给a
    //想象一下如果此时a,b,c都是对象，b+c返回一个对象呢？
    在c++中操作符运算本质上也是函数，则这里单纯靠b+c返回
    一个对象的值赋值给a已经不现实了，对象是一个结构性的
    变量；所以在这种情况下就需要构建一个临时性对象，并将
    此临时性对象（函数的结果）利用赋值函数拷贝给a;
```
 + 单纯传入对象作为函数参数时，即使不返回对象，也会产生临时性对象，为什么? 随意写了个例子：
 ```
     1 #include<iostream>
    2 class Point {
    3         public:
    4                 int a;
    5                 Point(){std::cout<<"constructor"<<std::endl; a=4;}
    6                 void printa() const { std::cout<<a<<32<<std::endl;}
    7                 ~Point(){
    8                         std::cout<<"~destrucot"<<std::endl;}
>>  9                Point(const Point& p)
   10                {
   11                                 std::cout << "Copy Constructor" << std::en      dl;
   12                }
   13 
>> 14               Point& operator=(const Point& p)
   15               {
   16                  std::cout << "Assign" << std::endl;
   17                  return *this;
   18               }
   19 
   20 };
   21 void getv(const Point pp)
   22 {
   23     //= pp.a=6;
   24         pp.printa();
   25         std::cout<<pp.a<<std::endl;
   26 }
   27 
   28 int main()
   29 {
   30         Point pplist;
   31         std::cout<<pplist.a<<std::endl;
   32         getv(pplist);//这里产生临时对象是调用拷贝构造函数
   33         return 0;
   34 }
```

  按照内建类型来看，传入函数参数，其实传入的是值，
  而对于对象来说，若传入的是引用或指针则不需要产生临时性对象，但是传入的若是值，则c++编译器需要产生一个临时性对象，在函数的栈中，供函数中对该对象调用函数和值等操作；
  + 函数返回对象；
```
 另一个例子：
    如何做X xxx=bar();如何拷贝的？双阶段初始化：
  a 增加一个额外的引用参数给函数，如void bar(X＆　_result)
  b 在return 前插入一个copy constructor 
      void bar(X &__result){
                X xx;
                xx.X::X();
                __result.X::XX(xx);
                return ;
        }
    X xxx=bar()
    //->bar(X &__result);
        xxx=_result;
```
  + 手动调用构造函数：此时也会生成临时性对象；
  + 其他如通过构造函数调用成员函数的：单纯一个表达式 a+b这种的
 
```cpp
   #include <iostream>
class tmpclass
{
public:
    int a;
    tmpclass(int i)
    {
        a = i;
    }

    tmpclass()
    {
        tmpclass(0);//手动调用构造函数会产生临时对象，临时对象的a=0,故最后结果show还是无初值
    }

    void show()
    {
        std::cout << "a = " << a << std::endl;
    }
};

int main(void)
{
    tmpclass c;
    c.show();
    return 0;
}
其他：
 tmpclass c1 = tmpclass(6);//赋值构造+构造函数
 tmpclass(6).show()//产生临时性对象；
```

+ 如何避免产生临时性对象
尽量不用上述的手法编程
+ 临时性对象的效率（迷思，测试)
临时性对象会造成效率低下，在不当的代码下容易产生很多临时性对象；不当的使用也会造成非预期的结果；
+ 临时性对象的生命周期：
可能是这个表达式的生命周期，具体可以通过构造类函数和析构函数调试；
