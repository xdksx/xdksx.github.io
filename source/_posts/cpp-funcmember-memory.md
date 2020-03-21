---
title: cpp_funcmember_memory
date: 2018-06-09 16:21:29
tags: cpp_class
categories: c&cpp
---
### c++ function语意学
something:  
恩，这部分记录自己对c++中的函数特殊性的理解，本想深入函数本质，之前接触过，记得在linux内核剖析那本书讲的很好。
 可以参考；<!--more-->
 
 + 实现类的成员函数和类之间的关联，即通过对象能调用成员函数。这种实现，可以是以下几种方式：
 1）struct+函数指针：结构体中，一个成员函数对应一个指针，增加空间 如何操作成员？
 2）函数+this指针，普通函数，this指针一个，不添加额外多的空间，this 操作成员
 。。。
 
+ c++的类主要是将成员和函数封装起来。那么成员的封装就像struct一样，而函数呢？成员函数如何和类，对象联系起来？--this指针


 + c++中的成员函数和普通函数一样，只是多了个this指针，这个指针指向当前调用这个函数的对象，并以形参传入，这样，成员函数就可以使用对象成员了，通过
 this指针；
 
+  这里也是根据深入探索c++模型中第四章，function语义学总结的：
#### 引入这个问题：
 通过对象和对象指针来调用成员函数的不同：  
   Point3d obj;  
   Point3d *p=&obj;  
   两者效率有何不同？
   
   通过上面分析：当这个成员函数为普通的成员函数时，则直接调用函数，传入this指针，所以基本没有大的区别  
 #### 以下分为几种函数讨论：
##### 非静态成员函数：
   为了支持this指针等构成成员函数，c++做了如下步骤：  
  +  a;改写函数原型：安插了一个this参数
  +  b；对对象成员的操作，通过this
  +  c;对成员函数写成外部函数，并将其名称进行mangling处理，成独一无二～类似：函数名___类名参数名 等等
 + 如何命名：成员函数和成员都会被名称特殊化处理：一般加上base class &derived class名
      对重载函数而言如何区分：加上参数链表；
      当extern C时，会压抑这种特殊命名化
      具体编译器实现不同，可以通过汇编等。gdb等看
   
   
+  鉴于此：看一个例子：  
   当你需要构造一个（拷贝构造函数）或者改变一个类的成员： 伪代码：  
  ```cpp
  void normalize__Point(register const Point3d *const this,Point3d &_result)
   {
      _reuslt.Point3d::Point3d() //默认构造函数：
      _result._x=this->__x/2;
      ...
      return ;
   }
   那么，以下这种方式：更好：
   Point3d Point3d::normmalize() const{
       return  Point3d(_x/2 ...)直接构建会更快） 
    =》转换为return Point3d(this->_x/2,...)   
  ```
 ##### virtual func
 若normilaze是虚拟函数，则  
 ptr->normilaze()=》 （*ptr->vptr[1])(ptr);  
 可以看到效率较低，所以当确认不用多态机制的时候，最好抑制它，直接使用：Point3d::xxx  这种方式调用函数
 而若被写成内连函数会更优--原因待探索：
 
 
##### 静态成员函数：
 显然这种能被类直接调用的函数，没有this指针，从而也就无法用成员  
 这种函数的出现，是因为一种需求：即不用每个成员函数都会使用成员，不一定需要this  
 不用声明一个对象就可以调用函数，当非要对象不可，又不用成员时：   
 ```cpp
 ((Point3d*)0)->object_count() ;```
 有了static后，就不用上述方式了  
 所以static的特性完全来源它的原理：  
+ 它不能直接存取non static 成员
+ 它不能被声明为const volatile virtual
+ 能直接被类调用  
 静态成员函数和普通函数更像，因为它没有this指针，也就不是这种类型：~ unsigned int(*) ();
 所以更可以和类之外的元素沟通，比如回调函数
 
 ##### 虚拟成员函数
 + a 为了支持多态，需要在执行期间能判断到底调用哪个函数？即pz->z()这个函数，pz为基类指针，而能调用子类函数
 + b 基类指针或引用参考到的（指向的）子类，，子类中的重写父类的函数，在调用时并不知道传入哪个this,即表现为调用哪个函数
 +  c　那么为了支持通过基类能调用它，通过子类也能调用该函数，则需要基类和子类共享某个东西，他能正确决议出要调用的函数
 
+  带来：额外的空间，和c的兼容性
##### 积极多态的概念：
（１）被指出的对象真正被使用；（２）dynamic_cast  
 那么哪些函数需要支持这样的特性－－－》由virtual标志来指出  
 如何在执行期间调用真正的函数？－－－》ptr所指的对象真实类型＋函数的位置　真实类型放在vtable[0]

+ 编译期间做的：  
在每个对象中加入：一个字符串或数字来表示class　类型＋一个指针，指向表格vtable,它带有程序的virtual func执行期地址
确定表格中的函数指针，执行期无法改变表格一个类对应一个表格，每个virtual func被指定一个固定的索引值
+ 执行期间做的：
 为vptr分配内存地址。它的值在编译期间确定，类似于x=3;  指向vtable
调用函数时激活　。编译器已经为其转换语义为xxx->vptr[n](this).. 
+ 注意，当一个子类继承基类时，vptr继承过来，当子类改写virtual函数时，则改变表中的指针指向子类的；当子类添加一个新的virtual func时，则在表中加一个slot
+ 唯一在执行期间才知道的：slot(n)到底指向哪个函数实体
 细想一下：  
      derived de; //编译期间，符号表确认要分配的对象类型,执行期间分配内存和调用构造函数等，构造函数中会初始化vptr,    
      base *p=&de;//编译期间，类似于int x=3;,执行期间分
      配内存给x,并将３给x,确定p所指的真正对象，从而才能取得vptr,到调用到最终的函数
      p->xx();//xx为virtual (*p->vptr[1])(p)
     
##### some question     
 //关于执行期和编译期确定的东西，有些困惑，为啥编译器不能在p->xx()的时候指定调用子类的xx()?
 //在编译器扫描代码的时候，知道定义了一个derived类型的变量，知道，定义了一个base指针，并指向的变量类型为derived,后面调用时也能
 确定函数名，为啥不能根据已存的这些信息。来得到哦。这时p指向的是derive类型的。所以调用derived的函数？
 不过也是，编译器从做词汇扫描，到语义扫描，出符号表，到语法检查，语义规则等
 
 
##### 多重继承下面的virtual func
 考虑以下例子：
 ```cpp
 class base1{
    public :
     int a;
 　　　virtual int a(){return a;}
    virtual base1* clone() const;
   }
 class base2{
  public:
    int b;
    virtual int b(){return b;
    virtual base2* clone() const;
    }
 class derive:public base1,public base2
    int c;
    virtual int c(){return c;}
    virtual derive* clone() const;
    }
    ```
 对于和derived有相同起点的base1不用考虑太多，关键是base2,;有三个问题要解决：
```
  　(1)virtual destructor (记得之前是逐层调用）
 　（２）被继承下来的b()
    (3）一组clone函数 
  （a)   做base2 *pbase2=new derive;
  　　　　＆＆＆编译期间确定：＆＆＆
   =>  derived *tmp=new derive;
   　　　base2 *pbase2=tmp?tmp+sizeof(base1):0;```
      
 为了使pbase2能访问到　b 即pbase２->b, 多重继承下使用 base *p=de;时，注意此时需要调整，让p指向的是de中的base对应部分
+ 当通过delete pbase2时，此时需要删除完整的对象，所以pbase２指针需要调整到指出完整对象的起始点  
  他也要通过上述a类似的加法，以及调用virtual destructor函数  
```cpp 
   如base2 *pbase2=new derive;  
  delete pbase2;//invoke derive class's destructor (virtual )```
  
 　首先这个调用要通过vptr,其次，传入的this指针需要调整
  ＆＆＆执行期间确定＆＆＆
+ 注意：然而，这种offset加法无法在编译期间直接设定。pbase2所指的真正对象只有在执行期间才能确定  
//如何调整；早期的想法是：将vtable中的slot改为一个slot放置函数指针faddr和this的offset  
 //则：（*pbase2->vptr[1])(pbase2);  
 //改为　（*pbase2->vptr[1].faddr)(pbase2+pbase2-  >vptr[1].offset);但是连带处罚了其他形式virtual func调用，
  
 + 那如何处理？  
 + [１]方法１：thunk
  　　　～：this+=sizeof(base1)
          Derived::~Derived(this);//只有汇编才有效率
   如何使能？在virtual table slot继续内含一个简单的指针，slot中的地址可以指向一个virtual func或者一个thunk(若需要调整this)
  其实多重继承来讲，每个继承的基类，都需要一个virtual table;,这样派生类就会维护多个基类的vtable
  + 1)经由derived或第一个base class)调用，不需要调整this
  + 2)经由>=2的base class 调用，同一种函数在virtual table中可能需要多笔对应的slot
        base1 *pbase1 =new derived;
        base2 *pbase2=new derived;
        delete pbase1//不需要调整this,virtual table slot放置正真的destructor地址
        
        delete pbase2//需要调整this ,放置thunk
        vptr和vtable命名也会被特殊化
        参考图在书中，这里不放
+ [２]方法２　：  
   因为动态链接器的原因，使得符号链接变得缓慢
   为了调整执行期链接器的效率，sun编译器将多个vtable 连锁为一个，指向次要表格的指针可以由主要表格加offset
　  其他类似。
例子：
```cpp
　　　　　　　base2 *ptr =new derived;
       delete ptr;//此时调用derived::~deirved,故ptr需要被调整，若为virtual，需要更多
       
       derived *pder=new derived;
       pder->b();//注意b没有被改写，所以需要调整pder指向base2 subobj
       
       base2 *pb1=new derived;
       base2 *pb2=pb1->clone()//注意是调用derived:clone,所以调用后返回值要改变器指向base2 subobj
       
       当函数被认为足够小　，sun编译器会采用某种算法优化，split。。，否则不会，具体待了解。
       所以virtual func的通常大小为８行```
+ [３]IBM:  
函数若支持多重进入点，将不必有许多thunk;将thunk操作放在virtual func中，里面判断是否需要调整this指针
     
     
##### 虚拟继承下的virtual func
这种情况相当复杂了。需要探索上述的这些问题：point2d.point3d的对象不再相符，即使只继承一个，即单一虚拟继承，也需要调整this指针
```cpp
class point2d{
  public:
  point2d()
  virtual ~point2d()
  virtual void mumble()
  virutal  float z()
  protected:
  float _x,_y;
  } 
  class point3d:public virtual point2d{
    public:
    point3d()
    ~point3d
    protected:
    float _z;
    }```
    可以尝试下写出例子比较point2d和point3d指针看指向是否相同
    当virtual base class中包含virtual func和nonstatic member时，各个厂商实现不同，复杂，作者不建议这么做
    
    －－－－－－－－－－－－－－－－－－－
#### 函数的效能：
  这部分可以通过函数的调用方式感性的了解，或者根据不同的编译器。测试对比几种调用的时间，以及是否已经开启优化等
  （如编译器将被视为不变的表达式提到循环之外）
  （通过消除局部对象的使用可以消除对constructor的调用）
  
#### 指向memeber　func的指针：
  取一个nonstatic member func的地址，若为nonvirtual 则为其在内存中的真正地址。
  但需要this参数。　
  指向member func的指针：double (Point::*pmf)();类似
  定义：double (point::*coord)() =&point::x;
       赋值:coord=&point::y
       调用：（origin.*coord)()/(ptr->*coord)()
       转换为：(coord)(&origin)/(coord)(ptr)
       
##### 指向virtual memeber func指针
  在g++中
  ```cpp
  #include<iostream>
#include<stdio.h>
using namespace std;
class Base1 {
		public:
				virtual float getp() { return mp;}
				void setp(float mmp){mp=mmp;}
				virtual int test(){return 3;}
				virtual int test1(){return 4;}
		private:
				float mp;
}; 
int main()
{  
		float (Base1::*pmf)()=&Base1::getp;
		Base1 *ptr=new Base1;
		ptr->setp(3.2);
		cout<<ptr->getp()<<endl;//3.2
		cout<<(ptr->*pmf)()<<endl;//3.2　　会被内部转换：（*ptr->vptr[(int)pmf])(ptr)
		printf("%p\n",&Base1::getp);//1　索引值
		printf("%p\n",&Base1::setp);//40xxx真实地址
		printf("%p\n",&Base1::test);// 9，为什么是９不清楚
		printf("%p\n",&Base1::test1);//11
		
		return  0;
}```
　　所以，对于虚拟函数也是可以通过函数指针来访问的，只不过得到的地址是索引值，但不影响访问
或者对以下
```cpp
#include<iostream>
#include<stdio.h>
using namespace std;
class Base1 {
		public:
				virtual float getp() { return mp;}
				void setp(float mmp){mp=mmp;}
			        int test(){return 3;}
				virtual int test1(){return 4;}
		private:
				float mp;
}; 
int main()
{  
		float (Base1::*pmf)()=&Base1::getp;
		Base1 *ptr=new Base1;
		ptr->setp(3.2);
		cout<<ptr->getp()<<endl;
		cout<<(ptr->*pmf)()<<endl;
		printf("%p\n",&Base1::getp);
		printf("%p\n",&Base1::setp);
		printf("%p\n",&Base1::test);
		printf("%p\n",&Base1::test1);
                  
                int (Base1::*pmi)()=&Base1::test1;//or test　可以指向两种，编译器如何区分呢？cfront２　通过判断是索引（may <127)还是函数地址来区分
                Base1 *ptr2=new Base1;
                cout<<(ptr2->*pmi)()<<endl;
                delete ptr;
                delete ptr2;
		
		return  0;
}```
##### 多重继承下指向member func指针
为了让mem func point能支持多重继承和虚拟继承：
噢这种情况很复杂了。你得知道它指向的是virtual 还是non virtual
书中并没有探讨太多，野比较复杂，这里举一个例子，探索细节可以看编译器，gdb等结果：
```cpp
#include<iostream>
#include<stdio.h>
using namespace std;
class Base1 {
		public:
				virtual float getp() { return mp;}
				void setp(float mmp){mp=mmp;}
			        int test(){return 3;}
				virtual int test1(){return 4;}
		private:
				float mp;
};
class Base2 {
		public:
				virtual float get2p(){return m2p;}
				virtual void set2p(float m2pp){m2p=m2pp;}
				int test22() {return 5;}
				virtual int test21(){return 6;}
		private:
				float m2p;
};
class Der:public Base1,public Base2{
		public:
				virtual float get3p(){return m3p;}
		private:
				float m3p;

};
int main()
{  
		float (Base1::*pmf)()=&Base1::getp;//这后面的调用就发挥想象把，想怎么尝试都行
		Base1 *ptr=new Der;
		ptr->setp(3.2);
		cout<<ptr->getp()<<endl;
		cout<<(ptr->*pmf)()<<endl;
		printf("%p\n",&Der::getp);
		printf("%p\n",&Base1::setp);
		printf("%p\n",&Base1::test);
		printf("%p\n",&Base1::test1);
                  
                int (Base1::*pmi)()=&Base1::test1;//or test
                Base1 *ptr2=new Base1;
                cout<<(ptr2->*pmi)()<<endl;
                delete ptr;
                delete ptr2;
		
		return  0;
}```
##### 效能
##### inline func:
首先得直到，内联函数实际上表现为什么？函数本质上被经过栈调用，而内联函数是直接　在主函数中铺开为表达式，所以调用内联函数　能提高效率，但是响应的
源代码会变大，而且有参数的限制
应用：在操作符函数中，指定set,get类函数，使得操作符函数能被写成普通的函数。　而set get 写成inline函数，会减少效率降低
inline关键词只是请求，并不会所有都inline展开，编译器会做判断权衡
具体看书，不是很细
内联函数两个注意点：
形式参数：inline扩展时，每一个形式参数会被对应的实际参数代替：
```cpp
　　　　　　　　inline int min (int i,int j){
                   return i<j?i:j;
                   }
         三个调用：
         inline int bar() {
            int minval;
            int val1=1024;
            int val2=2048;
            minval=min(val1,val2); 参数直接替换val1<val2?val1:val2;
            minval=min(1024,2048);替换后直接使用常量：1024
            minval=min(fool(),bar()+1) 引发参数副作用，需要导入一个临时对象，以避免重复求值：
            　　　　     int t1,t2; minval=(t1=foo()),(t2=bar()+1),t1<t2?...)
           
           
            return minval}```
 局部变量:在内联函数中加入局部变量，会导致因为维护局部变量带来的成本；  
 内联函数在替代c宏，是比较安全的；但是若宏中参数有副作用的话，如上，会导致大的扩展码以维护局部变量或者生成临时的变量
            
