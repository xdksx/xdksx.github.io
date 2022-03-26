---
title: cpp_this
date: 2018-06-09 10:13:46
tags: cpp_keyword
categories: c&cpp
---
### c++关键字之this
#### this指针是什么
this是一个指向当前正在使用的对象的指针，它是一个指针；
成员函数通过它来使用对象的值，通过它知道要对哪个对象的成员操作
如：<!--more-->
```cpp
class Simple
{
private:
    int m_id;
public:
    Simple(int id)
    {
        setID(id);
    }
    void setID(int id) { m_id = id; }
    int getID() { return m_id; }
};
int main()
{
    Simple simple(1);
    simple.setID(2);
    std::cout << simple.getID() << '\n';
    return 0;
}
```
```cpp
simple.setID(2);--->
setID(&simple, 2); // note that simple has been changed from an object prefix to a function argument!
  void setID(int id) { m_id = id; }--->
  void setID(Simple* const this, int id) { this->m_id = id; }
  ```
#### this指针用法
关键词 this 是一个纯右值表达式，其值是在调用成员函数时的对象地址。它能出现于下列语境：
+  在任何非静态成员函数体内，含成员初始化列表
+  在非静态成员函数的声明内任何(可选) cv 限定符序列后处，包含动态异常规定(弃用)、 noexcept 规定(C++11)及尾随返回类型(C++11 起)
+ 在默认成员初始化中 (C++11 起)
+ 在一个对象的构造期间，若通过不直接或间接从构造函数的 this 指针获得的泛左值，访问对象的值或任何其子对象，则如此获得的对象或子对象的值是未指定的。换言之，构造函数中 this 指针不能别名使用：

```cpp
extern struct D d;
struct D {
D(int a) : a(a), b(d.a) {} // a(a)will change to this->a(a),but b(d.a)-->this->b(d.a),and will get random value,but b(a) 或 b(this->a) 是正确的
    int a, b;
};
D d = D(1);   // 因为 b(d.a) 不通过 this 访问， d.b 现在是未指定的,i try in g++.find it d.b==1
```

#### this指针于内存哪里？  
this 指针是不占类的内存的，它是传入成员函数的参数，所以它在栈中  
#### this 指针总是指向正在操作的对象：
```cpp
int main()
{
    Simple A(1); // *this = &A inside the Simple constructor
    Simple B(2); // *this = &B inside the Simple constructor
    A.setID(3); // *this = &A inside member function setID
    B.setID(4); // *this = &B inside member function setID
 
    return 0;
}
``
+ this指针的连锁使用：  
由this指针理解cout<<xxx<<<xxx<<<xxxxx....
对上述的表达式，cout是一个类，<<是该类的操作符函数，则<<函数返回this，若返回空，则无法进行：  
```cpp
std::cout << "Hello, " << userName;
(std::cout << "Hello, ") << userName;

(void) << userName;　错误
(std::cout) << userName;正确
如何写？
class Calc
{
private:
    int m_value;
 
public:
    Calc() { m_value = 0; }
 
    Calc& add(int value) { m_value += value; return *this; }
    Calc& sub(int value) { m_value -= value; return *this; }
    Calc& mult(int value) { m_value *= value; return *this; }
 
    int getValue() { return m_value; }
};

#include <iostream>
int main()
{
    Calc calc;
    calc.add(5).sub(3).mult(4);
 
    std::cout << calc.getValue() << '\n';
    return 0;
}```

注意Calc& 和return *this
#### this指针到对象名代表的是什么
由above和以下例子：来看对象的地址等
```cpp
#include<iostream>
using namespace std;
class set
{
    int m_i;
public:
    set()
    {
        m_i = 0;
    }
    set add(int i)
    {
        m_i += i;
        return *this;
    }
    int  getval()
    {
        return m_i;
    }
};
int main()
{
    set s;
    s.add(2).add(2);
    cout<<s.getval();
}```
//结果是２　因为函数返回的*this是一个值，它是set对象的值：
```cpp
class set
    5 {   
    6     int m_e;
    7     int m_i;
    8  public:
    9     set()
   10     {
   11         m_e=0;
   12         m_i=0;
   13     }
   14     set add(int i){
   15             m_i+=i;
   16             return *this;
   17     }
   18     int getval() { 
   19             return m_i;
   20     }
   21 };

int main()
   33 {
   34         set s;
   35         set s2;
>> 36         printf("%x\n",s.add(2));--输出0,返回的是*this，为s的值，m_e是其第一个成员
>> 37         printf("%x\n",s);－－输出0
   38         s2=s.add(2);--s2被赋值了s，s此时的m_i=2+2
   39         s2.add(2);--s2的m_i=6
>> 40         printf("%d\n",s2);//输出的是０－－－ m_e=0
   41         cout<<s2.getval()<<endl;-输出6，因为
   42         return 0;
   43 }
 由此可以看出this->object  this->s
            *this==s
            *this==s的内容
　　　　　　取对象的地址　&s
```
ref:  
http://zh.cppreference.com/w/cpp/language/this
http://www.learncpp.com/cpp-tutorial/8-8-the-hidden-this-pointer/
