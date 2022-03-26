---
title: cpp_datamember_memory
date: 2018-06-09 14:46:49
tags: cpp_class
categories: c&cpp
---
### c++ class  datamemory
详细介绍c++的成员布局，类本身的布局和在各种情况下的布局

#### "类"本身的大小：
+ the simplest 引入
+ 1.2 多少内存能表现一个class?  
最小是１　  size<!--more-->

```c
class T{ };   ---1 一个char 表示这个类型
class X :public virtual T{};　　--指针大小，指针指向T  virtual base class subobject
class Y :public virtual T{};  --指针大小
class A:public X,public Y {};　--两个指针大小　```  
－－从深入那本书中说有两种方式，体现class的大小，这里是优化的特殊的类型，这些好理解，但是需要类的空间的真正原因是什么？这T的一个char存储了什么？  
用来干嘛？那在X中指向T的又是为什么需要？  
最小１char?  
我们知道，当一个类中仅包含一个nonstatic member时，如int ,则对象为int大小，但是当类为空时，对象如何去分配内存？
如果同时定义了两个对象，那如何去区分这两个对象？让他们地址不相同？－－－所以这里需要一个char

+ 注意，类本身也是一个类型，像int，struct一样，它的大小为４，struct成员和对齐，则类也一样，sizeof是在编译期间（确定c89中）  
一个例子：
```cpp
#include<iostream>
#define _sizeof(T) ((size_t)((T*)0+1))
#include<stdio.h>
using namespace std;
class T {};
class X:public virtual T{};
class Y:public virtual T{};
class A :public X,public Y{};
class TT{public:int q;int x;};
int main()
{
		T t1,t2;
		int xx;
		if (&t1==&t2)
				cout<<"same"<<endl;
		printf("%x\n",&t1);
		printf("%x\n",&t2);
        printf("%x\n",&xx);
	//	int sie=(Y*)0+1;
	//	printf("%x\n",X{});
	    int s= _sizeof(TT);　８　编译期间确定，直接把８赋给内存
        cout<<_sizeof(T)<<endl;　１
		cout<<sizeof(TT)<<endl;　　８
		cout<<sizeof(T)<<endl;
		cout<<sizeof(X)<<endl;
		cout<<sizeof(Y)<<endl;
		cout<<sizeof(A)<<endl;
		return 0;
}```
自然t1,t2地址不同，相邻

#### 那么一个类大的方面需要这些：
１）类中定义的普通成员  
２）语言本身所造成的额外负担如virtual机制，virtual table ,virtual base class subobject  
３）Aligmnment带来的  
（编译器的优化会带来内存布局的影响）

上述例子：当编译器不做优化时，为１，8+1+3,8+1+3 ,1(X)+8+8+3(ali)

#### 总结datamember的布局
+ 对象只存非静态的成员变量，而无论中间放置了几个静态变量都不会影响他们之间的排布。
　　　大部分的编译器将先定义的非静态成员变量放在低地址，而后的放在高地址。最好用时先用kdbg或测试程序等看下
+ 静态成员的存取不通过对象，他们放在数据段中
+ vptr一般会放在哪里？　
　　对象的头或者尾巴
+ align:C++标准要求，在同一个access section(private,public,protect等区段），member的排列只需要符和较晚出现放在较高的地址“即可，
　　access sections的多少并不会带来额外的负担
+ 我在g++上做了测试:

```cpp
class TT{
 public:
    int a;
    char b;
    };
   sizeof{TT);   8
class TT{
 public:
    int a;
    char b;
 protect:
    int c;
    char d;
    };
   sizeof(TT)=16
class TT{
 public:
    int a;
    char b;
 protect:
    int c;
    char d;
 public:
    int e;
    char f;
    };
  sizeof(TT)=24 
class TT{
 public:
    int a;
    char b;
    int e;
    char f;
 protect:
    int c;
    char d;
    };
  sizeof(TT)=24
class TT{
 public:
    int a;
    int e;
    char f;
    char b;
 protect:
    int c;
    char d;
    };
  sizeof(TT)=20
 ```
  由此看来这个编译器是按着c的struct对齐来的啊， 


#### data member的存取：
+ 成本  
比较：
```cpp
        TT tt1;
		TT *tt2=&tt1;
0x400b87 lea    -0x20(%rbp),%rax
0x400b8b mov    %rax,-0x48(%rbp)
		int d=tt1.d;
0x400b8f mov    -0x1c(%rbp),%eax
0x400b92 mov    %eax,-0x68(%rbp)
		int f=tt2->d;
0x400b95 mov    -0x48(%rbp),%rax
0x400b99 mov    0x4(%rax),%eax
0x400b9c mov    %eax,-0x64(%rbp)
```

+ 对static成员啊而言，实际是在数据段中，所以不管什么形式的访问都是相同的
待测试
+ 通过成员函数
需要通过this 指针，则同上例子中的指针访问

### 总结几种情况下的的布局


#### 单一继承不含多态
##### 一个典型的例子如下
```cpp
class Point2d{
   public:
    Point2d( float x=0.0,float y=0.0):_x(x),_y(y){};
    float x() {return _x;}
    float y() {return _y;}
    void x(float newX) { _x=newX;}
    void y(float newY) { _y=newY;}
    void operator+= (const Point2d&  rhs) {
        _x+=rhs.x();
        _y+=rhs.y();
        }
    ...more member;
   protected:
      float _x,_y;
};
class  Point3d: public Point2d{
     public: 
        Point3d(float x=0.0,float y=0.0,float z=0.0):Point2d(x,y),_z(z){};
        float z(){return _z;}
        void z(float newZ){_z=newZ;} 
        void operator+=(const Point3d& rhs) {
           Point2d::operator+=(rhs);
           _z+=rhs.z();
           }
          ...more member               
     protected:
      float _z;
};
```
##### 单一继承
则　基类的成员被子类包含，放在较低的地址；相比之下，多了对齐的空间；
```cpp
class Concrete {
  public:
    ...
  private:
     int val;
     char c1;
     char c2;
     char c3;
     };```
则需要占用８bytes;

而当被继承实现时：
```cpp
  class Concrete1{
    public:
    private: 
       int val;
       char bit1;
   };
  class Concrete2：public Concrete1{
    public:
    private:
         char bit2;
   };
   class  Concrete3:public Concrete2{
   public:
   private:
      char bit3;
      };
      ```
  由此带来成本 8+4+4=16
  
  + 那为什么要这么做的？继承的时候不能挤在一起吗？
  （在深入c++对象模型中有图容易理解。这里仅说明：
  　　　若：　Concrete2  *pc2;
            Concrete1 *pc1_1,*pc1_2;
            *pc1_2=*pc1_1; -默认复制构造
            pc1_1 = pc2; //pc1_1指向pc2;
            *pc1_2=*pc1_1;//覆盖掉了，如果继承是成员挤在一起，而不是对齐来的
            
            
            



##### 单一继承含多态：
```cpp
 class Point2d{
   public:
    Point2d( float x=0.0,float y=0.0):_x(x),_y(y){};
    float x() {return _x;}
    float y() {return _y;}
    virtual float z(){return 0.0;} 
    virtual void z(float){}  
    void x(float newX) { _x=newX;}
    void y(float newY) { _y=newY;}
    virtual void operator+= (const Point2d&  rhs) {
        _x+=rhs.x();
        _y+=rhs.y();
        }
    ...more member;
   protected:
      float _x,_y;
};
class  Point3d: public Point2d{
     public: 
        Point3d(float x=0.0,float y=0.0,float z=0.0):Point2d(x,y),_z(z){};
        virtual float z(){return _z;}
        virtual void z(float newZ){_z=newZ;}
        
        virtual void operator+=(const Point3d& rhs) {
           Point2d::operator+=(rhs);
           _z+=rhs.z();
           }
          ...more member     
     protected:
      float _z;
};```
由此可以满足
```cpp
　　　void fool(Point2d &p1,Point2d &p2){
     p1+=p2;
     }```
可以是Point2d和Point3d 这种弹性，牺牲了时间和空间
加入了什么呢？  

       virtual table
       vptr
	   add constructor vptr setting
       add destructor vptr virtual table dele

所以需要视情况而定，如若只是涉及到2d&3d之间，则可以是
```cpp
  virtual void operator+=(const Point２d& rhs) {
           Point2d::operator+=(rhs);
           _z+=rhs.z();//此时＋０
           }
Point2d p2d(...)
Point3d p3d(,,,,);
p3d+=p2d
```
另外：对vptr的摆放位置，若放在最后面，则兼容c
但是损失了对继承的更好支持，所以现在放在最前面


##### 多重继承
多重继承考虑的问题较多？但从设计角度看，你可能会问？
对比单一继承，子类成员起始地址和基类相同，那多重继承，和哪个基类一样？其他基类怎么办，在向上转型如何处理，指针呢？

另外，在单一继承中，vptr放在最开始的地方，如果base class没有virtual func而derived　class有呢？则此时单一继承的自然多态被打破，
若此时把一个derived class 转换为base class则　需要编译器介入，在多重继承+虚拟继承下就更有必要了

考虑这个例子：
```cpp
　class Point2d{ 带virtual 接口
   public: 
   protected:
       float _x _y;
       };
 class Point3d:public Point2d{
    public:
    protectd:
       float _z;
       }'
  class Vertex {带virtual接口
      protected:
         Vertex *next;
    };    
  class Vertex3d:public Point3d,public Vertex {
    protexted:
      float mumble;
      } ```

对多重继承派生对象，若将其地址　指定给最左端的base class则情况同单一继承一样，因为二者指向相同的地址。但是对第二个及以上时
需要将地址修改，加上或减去　介于中间的base class subobject
```cpp
  eg:  Vertex3d v3d;
        Vertex *pv;
        Point２d *p2d
        POint3d  *p3d;  
        pv=&v3d
     则内部为：pv=(Vertex*)(((char*)&v3d)+sizeof(Point3d));     
     而对p2d=&v3d;
         p3d=&v3d则只需要简单的拷贝
```              
      若为Vertex3d　*v3d;  pv=v3d;则内部还要进行判断空。因为*v3d可能为空，
      而引用不用，因为引用不可能参考到无
对存取其第二个基类成员，也是做类似的offset操作
##### 虚拟继承
在多重继承加虚拟继承时，如ios istream ostream
前面的内容会好理解一些，这部分的内容不好理解，也有些繁琐，于是，我回到之前学习的方法，即为什么，出现的原因，如果是自己实现又如何，等思考：
+ 为什么需要虚拟继承？  
虚拟继承出现，是因为当基类2和3都继承了基类1，而基类4继承了2和3，则基类4会同时拥有两份基类1，而虚拟继承就是为了让基类4只包含一份基类1，形成菱形继承结构
```cpp
class ios{..}
class istream:public virtual ios{..}
class ostream:public virtual ios{..}
class iostream:
public istream, public ostream {..}```
那么，虚拟继承是如何做，使得类4能只包含1份基类1，而不影响其他功能呢：
+ 梳理下：  
上述例子：挑战在于，将istream 和ostream 各自维护的一个ios subobj ，折叠为一个由iostream维护的单一ios subobj ，
并且还可以保存base class 和derived class的指针（以及reference)之间的多态操作  
 一般的实现方式如下： class 如果内含一个或者多个virtual base class obj,像istream那样，将被分割为两部分，一个不变的局部和一个共享局部  
 不变的局部中的数据，不管后继如何演化，都总是拥有固定的offset(从obj头算起），这部分数据可以直接存取，共享局部，则是virtual base class subject，这部分数据位置会因为每次派生发生变化，所以他们只能被间接存取  
 
 所以就是：istream 中有自己的和ios相关的部分，这部分为共享部分
```cpp
  class Point2d _x _y
 class Point3d:public virtual Point2d  _z
 class Vertex:public virtual Point2d
 class Vertex3d:public Point3d public Vertex 
 ```
 那 istream 如何存取这部分共享的数据？和iostream共享？感觉有点像单例模式
+  cfront是这样实现的：（这语言没见过）： 会在每个derived class obj中安插一些指针，每个指针 指向一个virtual base class 存取继承得来的virtual base
class member;所以在存取时通过这个指针存取
在多层继承时间接存取次数多；每个对象需要为每个virtual base class 背负一个指针）
                     void Point3d::operator+=(const Point3d &rhs) {
                      _x+=rhs._x;
                      _y+=rhs._y;
                      _z+=rhs._z;
                      }
                     则在这里：被转为：伪代码：_vbcPoint2d->_x+=rhs.__vbcPoint2d->_x;//vbc==virtual base class
                     ....
                    
                    而Point2d *2d=3d;
                    Point2d *2d=3d? 3d->__vbcPoint2d:0;
                    
                    
 + microsoft ：引入类似于vbtable的方法，引入virtual base class table 。而在对象中放一个指针指向该表，表则放置那些指向virtual base class 指针
 Bjarne: g++等（现在可能变了，但是类似）：
           在虚函数表中放置virtual base class 的offset而不是地址。
           在这里，上面的例子：（this+__vbtr__point3d[-1])->_x+= (&rhs+rhs.__vptr__point3d[-1])->_x;
           ...
           Point2d *2d=3d?3d+3d->__vptr__point3d[-1]:0
           
 + 注意：若是基类1含有virtual func则会拥有一个vptr指向自己的vtable. 而虚拟继承它的基类2,3，基类2,3本身含有virtual func则不会通过继承基类1的方式
 继承它的vptr,所以可以看到在书中的图中point3d有两个指针vptr,以下例子通过kdbg调试可以看到这个两个vptr
 
+ 两个问题：
+ 基类1在继承连增加时位置如何变化？
+ 在基类自己有virtual func时为什么要自己独用一个vptr?
```cpp
#include<iostream>
#include<stdio.h>
using namespace std;
class Point2d {
                public: virtual float printx(){return _x;}
		protected:
				float _x,_y;
};
class Vertex:public virtual Point2d{
		protected:
				Vertex *next;
};
class Point3d:public virtual Point2d{
		protected:
				float _z;
};
class Vertex3d:public Vertex,public Point3d{
		protected:
				float mumble;
};
class PO{
		public:
		    // virtual ~PO();
			 static int origin;
			 float x,y,z;
};
int PO::origin =3;
int main()
{
		Point2d d2d;
		Point3d d3d;
		Vertex vx;
		Vertex3d v3x;
		PO po;
        printf("%d\n",& PO::z);
		printf("%d\n",&po);
		printf("%d\n",&po.x);
		printf("%d\n",&po.y);
//		printf("%d\n",&po.origin);
		float PO::*p1=0;
		float PO::*p2=&PO::x;
		if(p1==p2)
				cout<<"sma"<<endl;
		return 0;
}          
```
##### 使用gdb调试：
写完程序后：
编译时加-g
gdb 科执行程序名

+ 设置断点：break 行号
s向下执行
set p obj <on/off>: 在C++中，如果一个对象指针指向其派生类，如果打开这个选项，GDB会自动按照虚方法调用的规则显示输出，如果关闭这个选项的话，GDB就不管虚函数表了。这个选项默认是off。 使用show print object查看对象选项的设置。
set p pertty <on/off>: 按照层次打印结构体。可以从设置前后看到这个区别。on的确更容易阅读。
```c
set p obj on
set p pertty on
p 对象名
p /a ((void ***)d3d)[0]@18 //看到更详细的内存 @18为打印的成员数，[0]指从第0个开始打印--感觉不太对
p /a ((void **)vx)[0]@16//同上
(gdb) p b
$1 = {_vptr.Base = 0x400a60 <vtable for Base+16>}
(gdb) x/16x 0x400a60
0x400a60 <_ZTV4Base+16>:    0x0040094c  0x00000000  0x72654437  0x64657669
(gdb) x/16x 0x0040094c
0x40094c <Base::f()>:   0xe5894855  0x10ec8348  0xf87d8948  0x400a15be
0x40095c <Base::f()+16>:    0x10c0bf00  0xf9e80060  0xc9fffffd  0x485590c3
0x40096c <Derived::f()+2>:  0x8348e589  0x894810ec  0x1bbef87d  0xbf00400a
0x40097c <Derived::f()+18>: 0x006010c0  0xfffddbe8  0x66c3c9ff  0x00841f0f
  (gdb) set $i = 0
  (gdb) while $i < 10
     >print $i
     >p /a (*(void ***)obj)[$i]
     >set $i = $i + 1
     >end
Where "obj" is the object whose vtable you'd like to print, and 10 is the number of methods.
p /a (*(void ***)obj)[0]@10
info address _ZTV3Bar
```






#### 对象成员的效率
#### 指向对象成员变量的指针
可以用于测试底层布局，如vptr放在哪，access section 次序。等
例子：
```cpp
　　　class Point3d {
     public :
        virtual ~Point3d();
     protected:
        static Point3d origin；
        float x,y,z;
}
```

1)&Point3d::z  --得到z在class obj中的偏移量
需用printf
        
书中说的为了使得指向对象和对象成员的指针区分，而+1,在g++中没有怎么体现：
```cpp
#include<iostream>
#include<stdio.h>
using namespace std;
class Point3d{
		public:
	//			virtual ~Point3d(){;}
				static Point3d origin;
				float x,y,z;
};
Point3d Point3d::origin;
int main ()
{
        Point3d p3d;
		printf("&Point3d=%p\n",&p3d);
		printf("&Point3d=%p\n",&p3d.x);//这两个地址相同
		printf("&Point3d=%p\n",&p3d.y);
		printf("&Point3d::x=%p\n",&Point3d::x);//nil,若Point3d带virtual func,则为8
		printf("&Point3d::y=%p\n",&Point3d::y);
		printf("&Point3d::z=%p\n",&Point3d::z);
		if((float*)&p3d==(float*)&p3d.x)cout<<"yes"<<endl; //输出yes
		float Point3d::*p1=0;
		float Point3d::*p2=&Point3d::x;
		float Point3d::*p3=NULL;
		if(p1==p2)//未输出
		{
				cout<<"p1==p2"<<endl;//no output in g++
		}
        if(p2==p3)
		{
				cout<<"p2==p3"<<endl;//no output in g++
		}
		return 0;
}```
在这里若是加了virtual func则，x为8，说明是vptr是放在前面的
通过指针取得对象成员:  
```cpp
                    float *p=origin.z  
                    struct  Base1{int val1;}
                    struct Base2 (int val2;}
                    struct Derved:Base1,Base2{..}
                    void func1(int Derved::*bmp,Derved *pd)//传入offset等，多继承时易出错
                    {
                       pd->*dmp;.....```
