---
title: cpp_polymorphism
date: 2018-06-09 14:06:57
tags: cpp_class
categories: c&cpp
---
### 多态：

#### 为什么需要多态？
引入几个点：  
　　继承的机制中，基类的指针和引用能指向派生类，（派生类对象中的基类成员是在内存前面的，即指向派生类对象的指针指向的是第一个基类的第一个成员）  
+ 指针的值本质上是内存地址，也就是说指向不同变量(int char.)，对象的指针本身的值都是地址的value)
+ 指针的类型限定了指针的操作内存范围和取值的方式，如通过int指针取值的时候，取到４byte,通过char指针取值时取得1byte;）<!--more-->
+ 因此，基类指针限定了，即使它指向派生类，它也只能操作基类自身的成员和方法；就像char形指针指向int形，取得的只是一部分）

```cpp
　　　#include<iostream>
  3 using namespace std;
  4 int main()
  5 {
  6     int in=2;
  7     char *pc=(char*)&in;
  8     cout<<*pc<<endl;//乱码
  9     return 0;
 10 }
```

所以基类指针可以指向子类，但是只能调用基类的函数，即使子类复写了父类的函数
```cpp
class Base
{
public:
    const char* getName() { return "Base"; }
};
 
class Derived: public Base
{
public:
    const char* getName() { return "Derived"; }
};
 
int main()
{
    Derived derived;
    Base &rBase = derived;
    std::cout << "rBase is a " << rBase.getName() << '\n';
}
```
This example prints the result:

rBase is a Base

/＝》简化：如何让下面的例子得到想要的结果？  
```cpp
Animal *animals[] = { &fred, &garbo, &misty, &pooky, &truffle, &zeke };
    for (int iii=0; iii < 6; iii++)
        std::cout << animals[iii]->getName() << " says " << animals[iii]->speak() << '\n';
 ```
+ 如何让父类指针指向子类对象，可以调用子类函数呢？
      －－－多态
+ 调用的子类函数要满足:  
  A derived function is considered a match if it has the same signature (name, parameter types, and whether it is const) and return type as the base version of the function. Such functions are called overrides.

#### 如何使用－－－例子
```cpp
class Base
{
public:
    virtual const char* getName() { return "Base"; } // note addition of virtual keyword
};
 
class Derived: public Base
{
public:
    virtual const char* getName() { return "Derived"; }
};
 
int main()
{
    Derived derived;
    Base &rBase = derived;
    std::cout << "rBase is a " << rBase.getName() << '\n';
 
    return 0;
}
```
This example prints the result:

rBase is a Derived
－－－－－－－－－－－－－－－－－－－－－－－  
再盗一个例子：
```cpp
#include <string>
class Animal
{
protected:
    std::string m_name;
 
    // We're making this constructor protected because
    // we don't want people creating Animal objects directly,
    // but we still want derived classes to be able to use it.
    Animal(std::string name)
        : m_name(name)
    {
    }
 
public:
    std::string getName() { return m_name; }
    virtual const char* speak() { return "???"; }
};
 
class Cat: public Animal
{
public:
    Cat(std::string name)
        : Animal(name)
    {
    }
 
    virtual const char* speak() { return "Meow"; }
};
 
class Dog: public Animal
{
public:
    Dog(std::string name)
        : Animal(name)
    {
    }
 
    virtual const char* speak() { return "Woof"; }
};
 
void report(Animal &animal)
{
    std::cout << animal.getName() << " says " << animal.speak() << '\n';
}
 
int main()
{
    Cat cat("Fred");
    Dog dog("Garbo");
 
    report(cat);
    report(dog);
}
```
#### 注意多态的方式：
调用基类指针指向子类，函数时，会调用到继承琏最底端，且该最低端的子类有该virtual函数的，否则只调用到最下且复写了该函数的。如下例子：
```cpp
  1 #include<iostream>
  2 using namespace std;
  3 class A
  4 {
  5         public:
  6                     virtual const char* getName() { return "A"; }
  7 };
  8 
  9 class B: public A
 10 {
 11         public:
 12                     virtual const char* getName() { return "B"; }
 13 };
 14 
 15 class C: public B
 16 {
 17         public:
 18                     virtual const char* getName() { return "C"; }
                        //or const char * getName(){..} and default virtual
 19 };
 20 
 21 class D: public C
 22 {
 23         public:
 24                    // virtual const char* getName() { return "D"; }
 25 };
 26 
 27 int main()
 28 {
 29             D d;
 30             A &rBase = d;
 31             std::cout << "rBase is a " << rBase.getName() << '\n';
 32 
 33             return 0;
 34 }```
     输出c
     
#### 注意点：
+ virtual关键字是否都需要写？：  
Only the most base class function needs to be tagged as virtual for all of the derived 
functions to work virtually. However, having the keyword virtual on the derived functions
 does not hurt, and it serves as a useful reminder that the function is a virtual function 
 rather than a normal one.
 
 
+ 不能在构造函数和西沟函数中调用它，因为此时子类对象还没有被构造
 
+ c++11引入override和final来防止避免不匹配的复写和阻止继承：  
 １）override:  
 出现错误的例子:
 ```cpp
 class A
{
public:
	virtual const char* getName1(int x) { return "A"; }
	virtual const char* getName2(int x) { return "A"; }
};
class B : public A
{
public:
	virtual const char* getName1(short int x) { return "B"; } // note: parameter is a short int
	virtual const char* getName2(int x) const { return "B"; } // note: function is const
}; 
int main()
{
	B b;
	A &rBase = b;
	std::cout << rBase.getName1(1) << '\n';
	std::cout << rBase.getName2(2) << '\n';
 
	return 0;
}
 ```
 
 由于参数返回值不匹配，所以编译器认为不是复写，结果：
 A
A
```cpp
加入override:
class A
{
public:
	virtual const char* getName1(int x) { return "A"; }
	virtual const char* getName2(int x) { return "A"; }
	virtual const char* getName3(int x) { return "A"; }
};
class B : public A
{
public:
	virtual const char* getName1(short int x) override { return "B"; } // compile error, function is not an override
	virtual const char* getName2(int x) const override { return "B"; } // compile error, function is not an override
	virtual const char* getName3(int x) override { return "B"; } // okay, function is an override of A::getName3(int) 
};
int main()
{
	return 0;
} 
Rule: Apply the override specifier to every intended override function you write.
 ```
 
+ final:
 ```cpp
 加了final的函数无法被复写：
 class A
{
public:
	virtual const char* getName() { return "A"; }
};
class B : public A
{
public:
	// note use of final specifier on following line -- that makes this function no longer overridable
	virtual const char* getName() override final { return "B"; } // okay, overrides A::getName()
}; 
class C : public B
{
public:
	virtual const char* getName() override { return "C"; } // compile error: overrides B::getName(), which is final
};
 加了final的类不能被继承：
 class A
{
public:
	virtual const char* getName() { return "A"; }
};
class B final : public A // note use of final specifier here
{
public:
	virtual const char* getName() override { return "B"; }
};
class C : public B // compile error: cannot inherit from final class
{
public:
	virtual const char* getName() override { return "C"; }
};
 ````
 
+ 对匹配返回值的一个“例外”：covariant return types:
 ```cpp
 class Base
{
public:
    // This version of getThis() returns a pointer to a Base class
    virtual Base* getThis() { return this; }
};
class Derived: public Base
{
    // Normally override functions have to return objects of the same type as the base function
    // However, because Derived is derived from Base, it's okay to return Derived* instead of Base*
    virtual Derived* getThis() { return this; }
};```

注意当base=&derive;
     base.getThis()----取得的任然是base
 
 
#### 当想基类指针指向派生类，虚函数但仍然想用基类虚函数时：
```cpp
#include <iostream>
int main()
{
    Derived derived;
    Base &base = derived;
    // Calls Base::GetName() instead of the virtualized Derived::GetName()
    std::cout << base.Base::getName() << std::endl;
}　　
```

 
### 虚析构函数　：
#### 为什么需要虚析构函数：
 当基类指针指向派生类时，当析构后，只会调用基类的析构函数：
```cpp
#include <iostream>
class Base
{
public:
    ~Base() // note: not virtual
    {
        std::cout << "Calling ~Base()" << std::endl;
    }
};
class Derived: public Base
{
private:
    int* m_array;
public:
    Derived(int length)
    {
        m_array = new int[length];
    }
    ~Derived() // note: not virtual
    {
        std::cout << "Calling ~Derived()" << std::endl;
        delete[] m_array;
    }
};
int main()
{
    Derived *derived = new Derived(5);
    Base *base = derived ;
    delete base;
    return 0;
}```
 只输出Calling ~Base()
 + 所以为了调用派生类的析构函数，需要定义为虚析构函数：
 ```cpp
 #include <iostream>
class Base
{
public:
    virtual ~Base() // note: virtual
    {
        std::cout << "Calling ~Base()" << std::endl;
    }
};
class Derived: public Base
{
private:
    int* m_array;
public:
    Derived(int length)
    {
        m_array = new int[length];
    }
    virtual ~Derived() // note: virtual
    {
        std::cout << "Calling ~Derived()" << std::endl;
        delete[] m_array;
    }
};
int main()
{
    Derived *derived = new Derived(5);
    Base *base = derived;
    delete base;
 
    return 0;
}
Now this program produces the following result:
Calling ~Derived()
Calling ~Base()
Rule: Whenever you are dealing with inheritance, you should make any explicit destructors virtual. ```
 
 
 
 
 ### 虚表：
 #### Early binding（静态绑定） and late binding（动态绑定）
 前者指的是直接在代码里调用函数，通过函数名，即编译期间就可以直到函数的地址的（偏移，非真实的内存地址）  
"" Early binding (also called static binding) means the compiler (or linker) is able to directly associate the identifier name (such as a function or variable name) with a machine address. 
 Remember that all functions have a unique address. So when the compiler (or linker) encounters a function call, 
  it replaces the function call with a machine language instruction that tells the CPU to jump to the address of the function.""
 ```cpp
 #include <iostream>
int add(int x, int y)
{
    return x + y;
}
int subtract(int x, int y)
{
    return x - y;
}
int multiply(int x, int y)
{
    return x * y;
}
int main()
{
    int x;
    std::cout << "Enter a number: ";
    std::cin >> x;
    int y;
    std::cout << "Enter another number: ";
    std::cin >> y;
    int op;
    do
    {
        std::cout << "Enter an operation (0=add, 1=subtract, 2=multiply): ";
        std::cin >> op;
    } while (op < 0 || op > 2);
    int result = 0;
    switch (op)
    {
        // call the target function directly using early binding
        case 0: result = add(x, y); break;
        case 1: result = subtract(x, y); break;
        case 2: result = multiply(x, y); break;
    }
    std::cout << "The answer is: " << result << std::endl;
    return 0;
}```
 后者指的是通过函数指针等方式，即在编译期间无法确定调用什么函数，需要通过指针来访问函数（函数指针），效率比前者低
 in some programs, it is not possible to know which function will be called until runtime (when the program is run). 
 This is known as late binding (or dynamic binding). In C++, one way to get late binding is to use function pointers. 
 ```cpp
 #include <iostream>
int add(int x, int y)
{
    return x + y;
}
int main()
{
    // Create a function pointer and make it point to the Add function
    int (*pFcn)(int, int) = add;
    std::cout << pFcn(5, 3) << std::endl; // add 5 + 3
    return 0;
}
 ```
#### 虚表
 为什么虚函数能成立－－基类指针指向派生类能调用派生类的函数：  
 实际上，在定义了虚函数的基类中维护了一个虚指针：它指向一张表，表中包含各个虚函数指针；在这个继承琏中的每一个类有自己的一张虚表，这张表由编译器在编译期间生成, 虚指针在包含虚函数的类的对象被创建时自动创建
```cpp
class Base
{
public:
    virtual void function1() {};
    virtual void function2() {};
};
class D1: public Base
{
public:
    virtual void function1() {};
};
class D2: public Base
{
public:
    virtual void function2() {};
}; 
 实际上为：
 class Base
{
public:
    FunctionPointer *__vptr;//虚指针
    virtual void function1() {};
    virtual void function2() {};
};
class D1: public Base
{
public:
    virtual void function1() {};
};
class D2: public Base
{
public:
    virtual void function2() {};
};
 base
    *__vptr;-------------------------->base vtable
   virtual function1()<----------------function1()
|->virtual function2()<----------------function2()
-----------------------------------------------------|
   D1:public base                                    |
   *__vptr,(inherited) ----------------D1 vtable     |
   virtual function1(); <--------------function1()   |
                                       function2()----
 D2类似D1
 ```
 
### 纯虚函数和纯虚类：
#### 什么是纯虚函数和纯虚类：
 + 没有定义函数体的虚成员函数成为纯虚函数：  
  virtual int getValue() = 0; // a pure virtual function  
  包含一个或多个纯虚函数的类成为纯虚类  
  虚基类是不能被对象化的，当对象化后调用纯虚函数，就会出现错误  
```cpp
int main()
{
    Base base; // We can't instantiate an abstract base class, but for the sake of example, pretend this was allowed
    base.getValue(); // what would this do?
}```
#### 为什么需要纯虚函数和纯虚类？
当子类继承了基类后，对虚函数来讲，需要复写基类的虚函数，而当忘记去写时，默认就会调用基类的虚函数；  
而如果将基类的虚函数定义为纯虚函数。就会通过编译报错来提醒我们要复写子类的虚函数：  
 例子：  
 ```cpp
 #include <string>
class Animal // This Animal is an abstract base class
{
protected:
    std::string m_name;
public:
    Animal(std::string name)
        : m_name(name)
    {
    }
    std::string getName() { return m_name; }
    virtual const char* speak() = 0; // note that speak is now a pure virtual function
};
#include <iostream>
class Cow: public Animal
{
public:
    Cow(std::string name)
        : Animal(name)
    {
    }
    // We forgot to redefine speak
};
int main()
{
    Cow cow("Betsy");
    std::cout << cow.getName() << " says " << cow.speak() << '\n';
}
```

```cpp
#include <iostream>
class Cow: public Animal
{
public:
    Cow(std::string name)
        : Animal(name)
    {
    }
    virtual const char* speak() { return "Moo"; }
};
int main()
{
    Cow cow("Betsy");
    std::cout << cow.getName() << " says " << cow.speak() << '\n';
}
```

+ 当想要使用虚基类的实现时，又同时想让虚基类不能对象化且子类能提示需要实现纯虚函数时，应该怎么做？
```cpp
#include <string>
#include <iostream> 
class Animal // This Animal is an abstract base class
{
protected:
    std::string m_name;
public:
    Animal(std::string name)
        : m_name(name)
    {
    }
    std::string getName() { return m_name; }
    virtual const char* speak() = 0; // note that speak is a pure virtual function
}; 
const char* Animal::speak()
{
    return "buzz"; // some default implementation
}
class Dragonfly: public Animal
{
public:
    Dragonfly(std::string name)
        : Animal(name)
    {
    }
    virtual const char* speak() // this class is no longer abstract because we defined this function
    {
        return Animal::speak(); // use Animal's default implementation
    }
};
int main()
{
    Dragonfly dfly("Sally");
    std::cout << dfly.getName() << " says " << dfly.speak() << '\n';
}
The above code prints:
Sally says buzz
```

#### 接口类：
接口类的类中没有成员变量。全部由纯虚函数构成；常命名为I开头的:如IErrorLog,类似于java中的接口
```cpp
class IErrorLog
{
public:
    virtual bool openLog(const char *filename) = 0;
    virtual bool closeLog() = 0;
    virtual bool writeError(const char *errorMessage) = 0;
    virtual ~IErrorLog() {}; // make a virtual destructor in case we delete an IErrorLog pointer, so the proper derived destructor is called
};```
#### virtual base class
当普通的继承，即一个子类继承自两个不同的父类，这两个父类继承自同一个基类时。就容易出现问题：
如：
```cpp
class PoweredDevice
{
public:
    PoweredDevice(int power)
    {
		cout << "PoweredDevice: " << power << '\n';
    }
};
class Scanner: public PoweredDevice
{
public:
    Scanner(int scanner, int power)
        : PoweredDevice(power)
    {
		cout << "Scanner: " << scanner << '\n';
    }
};
class Printer: public PoweredDevice
{
public:
    Printer(int printer, int power)
        : PoweredDevice(power)
    {
		cout << "Printer: " << printer << '\n';
    }
};
class Copier: public Scanner, public Printer
{
public:
    Copier(int scanner, int printer, int power)
        : Scanner(scanner, power), Printer(printer, power)
    {
    }
};
int main()
{
    Copier copier(1, 2, 3);
}
PoweredDevice: 3
Scanner: 1
PoweredDevice: 3
Printer: 2
```
从输出看，基类被构造了两次  
如何防止构造两个基类呢？使用virtual base class
```cpp
class PoweredDevice
{
};
class Scanner: virtual public PoweredDevice
{
};
class Printer: virtual public PoweredDevice
{
};
class Copier: public Scanner, public Printer
{
};
#include <iostream>
class PoweredDevice
{
public:
    PoweredDevice(int power)
    {
		std::cout << "PoweredDevice: " << power << '\n';
    }
};
class Scanner: virtual public PoweredDevice // note: PoweredDevice is now a virtual base class
{
public:
    Scanner(int scanner, int power)
        : PoweredDevice(power) // this line is required to create Scanner objects, but ignored in this case
    {
		std::cout << "Scanner: " << scanner << '\n';
    }
};
class Printer: virtual public PoweredDevice // note: PoweredDevice is now a virtual base class
{
public:
    Printer(int printer, int power)
        : PoweredDevice(power) // this line is required to create Printer objects, but ignored in this case
    {
		std::cout << "Printer: " << printer << '\n';
    }
};
class Copier: public Scanner, public Printer
{
public:
    Copier(int scanner, int printer, int power)
        : Scanner(scanner, power), Printer(printer, power),
        PoweredDevice(power) // PoweredDevice is constructed here
    {
    }
};
This time, our previous example:
int main()
{
    Copier copier(1, 2, 3);
}
produces the result:
PoweredDevice: 3
Scanner: 1
Printer: 2
```
这样的话，基类的构造交给了继承琏最底层的类
+ 注意：
+ virtual base class在子类对象之前就创建了
+ if Copier was singly inherited from Printer, and Printer was virtually inherited from PoweredDevice, Copier is still responsible for creating PoweredDevice
+ Fourth, a virtual base class is always considered a direct base of its most derived class 
(which is why the most derived class is responsible for its construction).
+ But classes inheriting the virtual base still need access to it. So in order to facilitate this, the compiler creates a virtual table for each class directly inheriting the virtual class (Printer and Scanner). These virtual tables point to the functions in the most derived class. Because the derived classes have a virtual table,  that also means they are now larger by a pointer (to the virtual table).

#### 对象分割：
当子类对象赋值给基类会发生什么？
子类对象的基类部分会给基类对象
```cpp
int main()
{
    Derived derived(5);
    Base base = derived; // what happens here?
    std::cout << "base is a " << base.getName() << " and has value " << base.getValue() << '\n';
    return 0;
}
传值给基类
void printName(const Base base) // note: base passed by value, not reference
{
    std::cout << "I am a " << base.getName() << '\n';
}
This is a pretty simple function with a const base object parameter that is passed by value. If we call this function like such:	
int main()
{
    Derived d(5);
    printName(d); // oops, didn't realize this was pass by value on the calling end 
    return 0;
}
```

vector<base>和vector<&base>和vector<base*>
第一种可以但是只能调用基类的部分
第二种不行：std::vector<Base&> v;
Unfortunately, this won’t compile. The elements of std::vector must be assignable, whereas references can’t be reassigned (only initialized).
第三种可以但是要做delete
```cpp
#include <vector>
int main()
{
	std::vector<Base*> v;
	v.push_back(new Base(5)); // add a Base object to our vector
	v.push_back(new Derived(6)); // add a Derived object to our vector
        // Print out all of the elements in our vector
	for (int count = 0; count < v.size(); ++count)
		std::cout << "I am a " << v[count]->getName() << " with value " << v[count]->getValue() << "\n";
	for (int count = 0; count < v.size(); ++count)
		delete v[count];
	return 0;
}
用智能指针可以避免：
#include <vector>
#include <functional> // for std::reference_wrapper
int main()
{
	std::vector<std::reference_wrapper<Base> > v; // our vector is a vector of std::reference_wrapper wrapped Base (not Base&)
	Base b(5); // b and d can't be anonymous objects
	Derived d(6);
	v.push_back(b); // add a Base object to our vector
	v.push_back(d); // add a Derived object to our vector
	// Print out all of the elements in our vector
	for (int count = 0; count < v.size(); ++count)
		std::cout << "I am a " << v[count].get().getName() << " with value " << v[count].get().getValue() << "\n"; // we use .get() to get our element from the wrapper
	return 0;
}```


一种极端情况：
```cpp
int main()
{
    Derived d1(5);
    Derived d2(6);
    Base &b = d2;//b为d2的引用
    b = d1; // this line is problematic　导致b的基类部分为d1,子类部分为d2,因为b是d2的引用，导致现在d2的基类部分是d1而子类部分是d2
    return 0;
}```

#### 总结多态的方式＋dynamic_cast:
+ 　shape *ps=new circle();
    经由virtual func:  ps->rotate()    //virtual func
    经由dynamic_cast:和type运算符：
    if(circle *pc=dynamic_cast<circle*>(ps))
    
    (理解了指针类型的本质就可以理解它和理解分割的内容了:指针的类型会教导编译器如何解释某个特定地址的内容和大小:
    所以一个void*的指针无法去确定涵盖多大的地址空间，所以它只能含有一个地址但是不能操作它指向的Ｏｂｊｅｃｔ
    所以转型只是一种编译器指令，不改变真正的地址，而是改变解读指针的方式让它指出其内存的大小和内容）
    更多见内存布局第一章图就能理解）

