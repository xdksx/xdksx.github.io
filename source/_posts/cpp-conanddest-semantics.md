---
title: cpp_conanddest_semantics
date: 2018-06-09 22:03:32
tags: cpp_class
categories: c&cpp
---

### 构造和析构函数语义学  
#### 1. 
即使是abstract base class也可能需要手动写constructor,de...,关键是看它有没有non satic data member  <!--more-->
例如:  
```cpp
  class Abstract_class {
      public:
         virtual ~Abstract_base() = 0;
         virtual void interface() const=0;
         virtual const char * mumble() const {
             return _mumble;
         }
      protected:
         char *_mumble;
    };```
  上述例子，该抽象类本身不能被实例化，但是可以被继承，被继承时在构造时必须初始化 “mumble”,此时就需要用它的默认构造函数，所以必须为这个抽象类写一个构造函数；   
  
#### 2. 纯虚函数的存在： 
```cpp
#include<iostream>
using namespace std;
class Abstra_class {
	public :
	   // virtual ~Abstra_class() =0;
		virtual ~Abstra_class() { delete _mumble;cout<<"use ~abstra_class";}
		//virtual void interface() const=0;
		virtual void interface() const { cout<<"use interface Abstra_class:";}
		virtual const char* mumble() const{
				return _mumble;
		}
		Abstra_class(){_mumble=new char[4];cout<<"construct:Abs..";}
	protected:
		char* _mumble;
};
class dev : public Abstra_class {
		public:
				~dev() {dd=0;}
				dev(){dd=3;}
				void useA (){
               Abstra_class::interface();//即使可以这样，但是链接失败，即因该函数的定义=0,重新加上定义使之不为pure就ok了
				}
				virtual void interface() const {if(dd==4)cout<<"re";}
		protected:
				int dd;
};
int main ()
{
	  dev ddd;
		ddd.useA();
		ddd.interface();
	  //  Abstra_class::interface();
		return 0;
}
```
输出：construct:Abs..use interface Abstra_class:use ~Abstra_class: 

　"可以定义和调用pure virtual func ,但是只能被静态的调用，而不能通过虚拟机制 "这个在g++上实验了下，发现：
```cpp
#include<iostream>
using namespace std;
class Abstra_class {
	public :
	   // virtual ~Abstra_class() =0;
		virtual ~Abstra_class() { delete _mumble;cout<<"use ~Abstra_class:";}
		virtual void interface() const=0;
		//virtual void interface() const { cout<<"use interface Abstra_class:";}
		virtual const char* mumble() const{
				return _mumble;
		}
		virtual void  ii() const=0;//ensure is a abstra class
		Abstra_class(){_mumble=new char[4];cout<<"construct:Abs..";}
	protected:
		char* _mumble;
};
inline void Abstra_class::interface() const {
}
class dev : public Abstra_class {
		public:
				~dev() {dd=0;}
				dev(){dd=3;}
				void useA (){
                Abstra_class::interface();//即使可以这样，但是链接失败，即因该函数的引用为0
		           dd=4;
				}
				virtual void interface() const {
						if(dd==4)cout<<"\nre\n";
						//Abstra_class::interface();
				}
				virtual void ii() const {cout<<"is implenment";}
		protected:
				int dd;
};
int main ()
{
	  dev ddd;
		ddd.useA();
		ddd.interface();
	Abstra_class *pt=&ddd;
	pt->interface();
	  //  Abstra_class::interface();
		return 0;
}
```
  输出了两次re,说明可以通过虚拟机制调用它；interface在这里被内联的定义了空的函数，它不算pure了，所以上述说法感觉存在问题。。。  
　注意，因为在每一个derived class　destructor会被扩展，以静态方式调用每一个virtual base class 和上一层的desturctor,
     所以只要缺少任何一个 base destructor定义则链接失败　，所以需要定义pure virtual destructor
      一个比较好的替代方式就是不要把vitual dect~定义为pure
      考虑到成本，不要把所有的函数都定以为virtual
      
3)虚拟规格的存在：
　　　　在virtual func要不要为const ,主要看要不要对date member做修改
所以不要随便定义为pure,virtual const,毕竟效率

#### 考虑几种情况下的构造情况：
##### 一、无继承情况下对象构造几种方式：  
```cpp
Point global;　//周期：程序的生命周期，exit前
 
Point foobar(){
   Point local;　//此函数的周期，调用默认构造函数但是不会初始化成员
   Point *heap=new Point;//delete 前，调用默认构造函数，但不会初始化成员
   *heap=local;//local若未调用用户初始化的构造函数，则warning,调用拷贝赋值函数，maybe 位搬移
   delete heap;//默认析构函数
   return local;//maybe拷贝构造或者位搬移
   }
 ```
+ 考虑这几个对象的声明周期
+ 在c中，global会被放入bss中，未初始化，但是c++中所有的全局对象被当做“初始化过的数据”，会调用constructor函数　可以测试一下
+ 一般情况下，c++默认为类生成：default constructor,destructor,copy constructor,copy assignment operator
```cpp
#include<iostream>
using namespace std;
class Point {
	public:
			Point (){cout<<"in constructor "<<endl;}
			~Point() {cout<<"in destructor"<<endl;}
};
Point glo;
int main ()
{
		return 0;
}```
显然在定义Point glo的时候构造函数是会调用的

１、抽象数据类型：
　　根据需要决定是否写constructor destructor或者默认的就足够了
　　　global类型的对象直到程序激活才调用构造函数
　　　显性的初始化列表比将构造函数扩展为inline效率更高，后者需要赋值等，看下面例子：
```cpp
#include<iostream>  
using namespace std;  
  
class Point{  
public:  
    Point(double x = 0.0, double y = 0.0, double z = 0.0) :_x(x), _y(y), _z(z){}  
    void print(){ cout << _x << endl << _y << endl << _z << endl; };  
private:  
    double _x, _y, _z;  
};  
  
int main(){  
    Point local1 = { 1.1, 1.2, 1.3 };//用g++ --std=c++11可以，若为double a=1.5; ..={a,...}变量形式则不行（c++11可以） 
    local1.print();  
  
    system("pause");  
    return 0;  
} ``` 
而在显性初始化列表（explicit initialization list)->xxx={yyy};使用时较快是如下原因：  
函数的activation record 被放进程序的堆栈时，initializatioin list 中的常量就可以被放进local1的内存中了;  
但是explicit initialization list带来三个缺点：
+ 只有当class member 都是public时才生效，这点实验private时也可以  
+ 只能在{ }中指定常量，因为在编译期间进行评估求值
+ 由于编译器为自动施行，所以失败的可能性更高  
看一下汇编代码：
```cpp
class Point{  
public:  
    Point(int x = 0, int  y = 0, int z = 0) :_x(x), _y(y), _z(z){}  
 ```
 ```
0x400938 push   %rbp
0x400939 mov    %rsp,%rbp
0x40093c mov    %rdi,-0x8(%rbp)
0x400940 mov    %esi,-0xc(%rbp)
0x400943 mov    %edx,-0x10(%rbp)
0x400946 mov    %ecx,-0x14(%rbp)
```

```cpp
    void print(){ cout << _x << endl << _y << endl << _z << endl; };  
private:  
    int _x, _y, _z;  
};   
  
int main(){ 
	int  a=1;	
    Point local1 = { a, 4, 5};```
    
```
0x4008a4 mov    -0x24(%rbp),%esi
0x4008a7 lea    -0x20(%rbp),%rax
0x4008ab mov    $0x5,%ecx
0x4008b0 mov    $0x4,%edx
0x4008b5 mov    %rax,%rdi
0x4008b8 callq  0x400938 <Point::Point(int, int, int)>
```
上述讲的activation record是：  
+ Locals to the callee
+ Return address to the caller
+ Parameters of the callee  
从上述的汇编代码可以看到，在做显示初始化列表时将常量值赋给ecx等两个寄存器，接着调用构造函数，构造函数将寄存器的内容赋给成员，而其实采用Point local2;这种直接调用构造函数的发式也是如此，且和在构造函数中赋值给成员时转换的汇编相同，所以不是很理解这块的区别，在g++ c++11是如此表现的    
##### ２、为继承做准备：  
继承可能用到多态，此时需要使用virtual ,那就会带来constructor等函数中加入对vptr的初始化（copy constructor等）
c++标准会要求编译器尽量延迟对nontrivial members的实际合成 操作，直到真正遇到其使用场合：这里trivial意思是无意义，这个trivial和non-trivial是对类的四种函数来说的：

    构造函数(ctor)  
    复制构造函数(copy)  
    赋值函数(assignment)  
    析构函数(dtor)  

如果至少满足下面3条里的一条：

    显式(explict)定义了这四种函数。
    类里有非静态非POD的数据成员。
    有基类。

那么上面的四种函数是non-trivial函数，比如叫non-trivial ctor、non-trivial copy…，也就是说有意义的函数，里面有一下必要的操作，比如类成员的初始化，释放内存等。   

##### 二、继承体系下的对象构造：  
constructor函数中的隐藏代码，  
1. 初始化列表  
2. member的默认构造函数,若该member未出现在初始化列表中
3. vptr，在1,2之前，指向vtable   
4. base class constructor  1,2,3之前，以声明顺序为顺序，若在member initialization list中，则应传递参数，否则在1,2,3前加入其默认构造函数。多继承时可能this指针调
5. virtual base class constructor，从左到右，从最深到最浅 ，同4，若在list中有则用，否则。。  

例子：
```cpp
 class Point{
   public:
    Point (float x=0.0,float y=0.0);  
    Point(const Point&);
    Point& operator=(const Point&);
    virtual ~Point();
    virtual float z() {return 0.0;}
   protected:
   float _x,_y;
   };
   class line {
    Point _begin,_end;
    public:
      Line(....);
      Line(...);
      };```
line的默认析构函数会被合成，但是无意义，当然它的member的默认析构函数会被加入。。。其他类似

##### 三、虚拟继承：
+ 考虑下面的例子：
```cpp
class Point3d:public virtual Point{
    public:
       Point3d(float x=0.0,float y=0.0,float z=0.0):Point(x,y),_z(z){}
       Point3d(const Point3d& rhs):point(rhs),_z(rhs._z){}
       ~Point3d();
       Point3d& operator=...
       //..
      proteced:
      float _z;
      }```
  传统的如上面的扩充构造函数：
  ```cpp
  this->Point(:Point(x,y);
  this->_vptr_Point3d = vtbl_Point3d;
  this->_vptr_point3d_point=_vtbl_point3d_point;
  this->_z=rhs_z;
  return this```
  但是在这里，虚拟继承这显然不够准确：　　
  考虑当出现菱形继承：　　
  ```cpp
  class Vertex:virtual public Point;
  class Vertex3d:public Point3d,public Vertex{;};
  class Pvertex:public Vertex3d{;};'```
  
  那作为被共享的Point subobject应该由最底层的如Vertex3d来初始化：
  而最底层显然也会初始化Point3d和vertex,并调用他们的构造函数；如果他们的构造函数不加限制，就会又调用Point的构造，导致错误；　　
  所以应该做限制，如下：  
  ```cpp
  Vertex3d::Vertex3d(Vertex3d *this,bool __most_derived,float x,float y,float z){
  if(__most_derived!=false)//判断是否为最底层
     this->Point::Point(x,y);//是则构造最上层的
     //调用上一层的base classes
     //设定__most_derived为false
  this->Point3d::Ponint3d(..);
  ...
  ```
而在   
```cpp
Point3d::Point3d(Point3d* this,...)
{
   if(___most_derived!=false)
      this->Point::Point(x,y);
      ....
      }```
所以最底层的构造函数等会限制中间层对最上层的构造
+ 思考：
+ 当obj只是某个完整的obj的子obj时，则不用调用virtual base class constructor:,如上，那就产生了一种提高效率的方式：
+ 将构造函数分割为obj和subobj,，前者为完整版本的obj，自然要调用所有，包括基类的构造，后者不用，这种分裂可以提高效率

#### vptr初始化语意
+ 题外，这本书总喜欢讨论特殊的情况，哭笑，下面也是，在构造函数中调用虚函数222
+ 主要讨论vptr什么时候初始化合适，以及为什么
+ constructor调用顺序：考虑：
  ```cpp
  class Vertex:virtual public Point;
  class Vertex3d:public Point3d,public Vertex{;};
  class Pvertex:public Vertex3d{;};'```
  当一个PVertex对象被构造时，构造函数顺序为：　　  
  Point    
  Point3d  
  Vertex  
  Vertex3d  
  Pvertex  
+ 假设这个继承体系中的每一个类都定义了一个virtual func size()，该函数负责传回class的大小；如果我们写：  
  Pvertex pv;  
  Point3d p3d;  
  Point  *pt=&pv;  
  那么这个调用pt->size()传回PVertex的大小，而  
  pt=&p3d; pt->size()则传回p3d的大小；
+ 更进一步，特殊情况：
当在构造函数中调用这个操作时，如在point3d构造函数中调用时，是选择哪个class的size,毕竟现在正在构造point3d,答案是point3d的
 + 考虑如何使得上述生效?
 + 静态调用Point3d::size()或者bnalalla
 + 最后，利用vptr虚拟机制，即要在调用size之前初始化好其class对应的vptr
 + 总结：在这种情况下，vptr在base class constructor 调用之后，在程序员代码之前（如size),或是初始化列表所列的member初始化操作前
 + 更好的，分割constructor为完整obj和subobj
 
 ### 对象复制语意学
 #### 复制函数什么时候会被合成和使用
+ 前言，当你设计一个类，并以一个对象复制给另一个对象时，有三种选择：  
 + 什么也不做，实行默认行为  
 + 提供一个显性拷贝函数
 + 拒绝，只需要把复制函数声明为private就可以 
+ 考虑默认的行为：
 + bitwise copy行为，当不出现一下情况时，甚至不会调用复制函数，而是做bitwise copy
 + 而当出现以下行为时，调用默认的copy assignment operator函数，并合成相关操作：　　
	1. 当class中有mem obj，这个obj有一个copy ass operaator
	2. 当类的基类有copy assi opera..
	3. 类带virtual func
	4. 继承自一个virtual base class  
+ 写一个显性的复制函数：  
  + 在发生上述情况时，编译器也会像合成构造函数一样在显性的复制函数中加东西
  + 而继上面的，Point共享基类，编译器如何能在Point3d和Vertex的copy assignment operator中压抑point的copy assignment operator呢?
  + 书中的后面有些难以理解，等后面再探索吧，哎时间有限。。
### 对象的功能
测试对象构造和拷贝的成本，在从简单形式，单一继承，到多重继承，虚拟继承等情况下的时间复杂度
### 解构语意学
+ 析构函数并不会总是被合成出来，更别提调用；
+ 只有在类内带的member object或者自己的base class带有destructor，编译器才会合成一个析构函数，否则被视为不需要或者不用合成调用
+ 析构函数没必要和构造函数对称
+ 析构函数一般有以下顺序：
  + 先调用最底层子类析构函数，接着往上，直到基类
  + 析构函数本身在被执行时，vprt会在程序员代码前被执行
  + 若class拥有member class object ,而后者拥有destructor,则他们会以其声明顺序相反顺序被调用
  + 如果object内带一个vptr,则首先重设相关的virtual table
  + 若有任何直接的非虚基类拥有析构函数，则同上
  + 若有虚基类，则按照构造顺序相反顺序调用
+ 类似于构造函数，可以分裂