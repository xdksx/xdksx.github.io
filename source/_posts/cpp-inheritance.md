---
title: cpp_inheritance
date: 2018-06-09 11:44:54
tags: cpp_class
categories: c&cpp
---
### c++ 继承：
#### 继承是什么能做什么
继承是创建复杂类的另一种方式，它展现出来俩个obj 的is-a的概念（前一种方式是加入复杂的函数和成员等has-a　概念）
继承体现的是两个对象（类）之间的联系，子类继承父类的成员和方法，然后特殊化于扩展自己的成员和方法，正如苹果和梨都是水果，它们拥有水果的一般特性，等边三角形是特殊的等腰三角形<!--more-->
几个概念引英文词： the class being inherited from is called the parent class, base class, or superclass, and the class doing the inheriting is called the child class, derived class, or subclass.

#### 继承怎么使用，分为什么
继承的使用通过例子来学习，分为单继承和多继承等
多继承的问题：  
１）两个基类中存在相同的方法，子类调用该方法－－－》解决，可以调用的方法前加上类限定符base::  
2)俩个类继承自同一个类，接着另一个类继承这两个类
懒得自己写，从learncpp拷贝
```cpp
#include <iostream>
#include <string>
 ```
 
 ```cpp
class Person
{
public:
    std::string m_name;
    int m_age;
 
    Person(std::string name = "", int age = 0)
        : m_name(name), m_age(age)
    {
    }
 
    std::string getName() const { return m_name; }
    int getAge() const { return m_age; }
 
};
 
// BaseballPlayer publicly inheriting Person
class BaseballPlayer : public Person
{
public:
    double m_battingAverage;
    int m_homeRuns;
 
    BaseballPlayer(double battingAverage = 0.0, int homeRuns = 0)
       : m_battingAverage(battingAverage), m_homeRuns(homeRuns)
    {
    }
};
 
int main()
{
    // Create a new BaseballPlayer object
    BaseballPlayer joe;
    // Assign it a name (we can do this directly because m_name is public)
    joe.m_name = "Joe";
    // Print out the name
    std::cout << joe.getName() << '\n'; // use the getName() function we've acquired from the Person base class
 
    return 0;
}```
#### 继承的方式，访问控制
A child class inherits both behaviors (member functions) and properties (member variables) from the parent 但是受继承方式的限制
继承的方式有public等，受限制，这种可以通俗的理解为，你和你妈，你姨妈，你表舅等的亲密度和能访问的内容是有等级差别的

|Access specifier in base class|when inherited publicly|when　inherited privately|when inherited protectedly|
|:-|:-|:-|:-|
|Public|Public|Private|Protected|
|Private|Inaccessible|Inaccessible|Inaccessible|
|Protected|Protected|Private|Protected|

#### 继承的内存
+ 对单纯的类对象来说，没有继承时，其对象内存是成员变量，存储在栈中
，当继承了基类后，基类的成员变量（即使是private不能访问的）会添加到子类的对象内存中，而且一般是放在低地址（前面）
对访问控制是在编译期间去做限制的

#### 编译器对继承做了什么？
+ 构造函数  
+ 首先，构造函数顺序：Because Derived inherits functions and variables from Base, 
you may assume that the members of Base are copied into Derived. 
However, this is not true. Instead, we can consider Derived as a two part class: 
one part Derived, and one part Base.
，从构造函数可以看出，继承的类，对象先调用继承连最上层的类构造函数，最后调用它自己的构造函数
+ 其次，基类构造函数被子类调用：//弥补了子类不能　初始化父类成员的缺陷－－－为什么子类不能在初始化队列中初始化父类成员？若为const就出错了，初始化两次

```cpp
class Derived: public Base
{
private: // our member is now private
    double m_cost;
 
public:
    Derived(double cost=0.0, int id=0)
        : Base(id), // Call Base(int) constructor with value id!
            m_cost(cost)
    {
    }
    ```
    
+ 析构函数：调用顺序和构造函数相反
+ 子类加入自己的函数和ovrridewirte父类函数  
 + 策略：
When a member function is called with a derived class object, 
the compiler first looks to see if that member exists in the derived class.
 If not, it begins walking up the inheritance chain and checking whether the member 
 has been defined in any of the parent classes. It uses the first one it finds.
 
 + 在父类中被声明为private 的函数经过子类重写后可能会变成public:  
 
 ```cpp
 class Base
{
private:
	void print()
	{
		std::cout << "Base";
	}
};
class Derived : public Base
{
public:
	void print()
	{
		std::cout << "Derived ";
	}
};
int main()
{
	Derived derived;
	derived.print(); // calls derived::print(), which is public
	return 0;
}
```

+ 保留父类函数的方法:

```cpp
class Base
{
protected:
    int m_value;
 
public:
    Base(int value)
        : m_value(value)
    {
    }
 
    void identify() { std::cout << "I am a Base\n"; }
};

class Derived: public Base
{
public:
    Derived(int value)
        : Base(value)
    {
    }
 
    int GetValue() { return m_value; }
 
    void identify()
    {
        Base::identify(); // call Base::identify() first
        std::cout << "I am a Derived\n"; // then identify ourselves
    }
};
```
+ c++11新：  
将base类中的保护函数，在子类中声明为public:

```cpp
class Base
{
private:
    int m_value;
 
public:
    Base(int value)
        : m_value(value)
    {
    }
 
protected:
    void printValue() { std::cout << m_value; }
};
class Derived: public Base
{
public:
    Derived(int value)
        : Base(value)
    {
    }
    // Base::printValue was inherited as protected, so the public has no access
    // But we're changing it to public via a using declaration
    using Base::printValue; // note: no parenthesis here  //c++11
};

int main()
{
    Derived derived(7);
 
    // printValue is public in Derived, so this is okay
    derived.printValue(); // prints 7
    return 0;
}
```

子类中将父类的方法隐藏：
```cpp
class Base
{
private:
	int m_value;
 
public:
	Base(int value)
		: m_value(value)
	{
	}
 
	int getValue() { return m_value; }
};
 
class Derived : public Base
{
public:
	Derived(int value)
		: Base(value)
	{
	}
 
 
	int getValue() = delete; // mark this function as inaccessible
};
 
int main()
{
	Derived derived(7);
 
	// The following won't work because getValue() has been deleted!
	std::cout << derived.getValue();
 
	return 0;
}
```