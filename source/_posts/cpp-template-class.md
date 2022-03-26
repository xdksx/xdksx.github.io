---
title: cpp_template_class
date: 2021-01-10 01:07:05
tags: cpp_template
categories: c&cpp
---


### 类模板的基本概念
### 类模板的引入：<!-- more -->

```cpp
一个例子：
用类封装一个包含多操作的IntArray:
#ifndef INTARRAY_H
#define INTARRAY_H
 
#include <assert.h> // for assert()
 
class IntArray
{
private:
    int m_length;
    int *m_data;
 
public:
    IntArray()
    {
        m_length = 0;
        m_data = nullptr;
    }
 
    IntArray(int length)
    {
        assert(length > 0);
        m_data = new int[length];
        m_length = length;
    }
 
    ~IntArray()
    {
        delete[] m_data;
    }
 
    void Erase()
    {
        delete[] m_data;
        // We need to make sure we set m_data to 0 here, otherwise it will
        // be left pointing at deallocated memory!
        m_data = nullptr;
        m_length = 0;
    }
 
    int& operator[](int index)
    {
        assert(index >= 0 && index < m_length);
        return m_data[index];
    }
 
    int getLength() { return m_length; }
};
 
#endif

用类封装一个包含多操作的DoubleArray:
#ifndef DOUBLEARRAY_H
#define DOUBLEARRAY_H
 
#include <assert.h> // for assert()
 
class DoubleArray
{
private:
    int m_length;
    double *m_data;
 
public:
    DoubleArray()
    {
        m_length = 0;
        m_data = nullptr;
    }
 
    DoubleArray(int length)
    {
        assert(length > 0);
        m_data = new double[length];
        m_length = length;
    }
 
    ~DoubleArray()
    {
        delete[] m_data;
    }
 
    void Erase()
    {
        delete[] m_data;
        // We need to make sure we set m_data to 0 here, otherwise it will
        // be left pointing at deallocated memory!
        m_data = nullptr;
        m_length = 0;
    }
 
    double & operator[](int index)
    {
        assert(index >= 0 && index < m_length);
        return m_data[index];
    }
 
    int getLength() { return m_length; }
};
 
#endif
```
```cpp
以上两个定义内容只有里面的类型不同，那么为什么不忽略类型的差异，定义通用的容器？于是类模板出现：
类模板的定义和使用：
#ifndef ARRAY_H
#define ARRAY_H
 
#include <assert.h> // for assert()
 
template <class T> // This is a template class, the user will provide the data type for T
class Array
{
private:
    int m_length;//注意这里可以指定具体类型；
    T *m_data;
 
public:
    Array()
    {
        m_length = 0;
        m_data = nullptr;
    }
 
    Array(int length)
    {
        m_data = new T[length];
        m_length = length;
    }
 
    ~Array()
    {
        delete[] m_data;
    }
 
    void Erase()
    {
        delete[] m_data;
        // We need to make sure we set m_data to 0 here, otherwise it will
        // be left pointing at deallocated memory!
        m_data = nullptr;
        m_length = 0;
    }
 
 
    T& operator[](int index)
    {
        assert(index >= 0 && index < m_length);
        return m_data[index];
    }
 
    // The length of the array is always an integer
    // It does not depend on the data type of the array
    int getLength(); // templated getLength() function defined below
};
 
template <typename T> // member functions defined outside the class need their own template declaration
int Array<T>::getLength() { return m_length; } // note class name is Array<T>, not Array
 
#endif
使用：
#include <iostream>
#include "Array.h"
 
int main()
{
	Array<int> intArray(12);
	Array<double> doubleArray(12);
 
	for (int count = 0; count < intArray.getLength(); ++count)
	{
		intArray[count] = count;
		doubleArray[count] = count + 0.5;
	}
 
	for (int count = intArray.getLength()-1; count >= 0; --count)
		std::cout << intArray[count] << "\t" << doubleArray[count] << '\n';
 
	return 0;
}
```


### 类模板和标准库：
从上面的例子可以看到，类模板定义的Array和Vector容器很类似，而类似vector的容器就是类模板实现的；

### 模板类无法进行定义和函数实现的分离：

```cpp
如下例子，在链接时会出现问题：
Array.h:
#ifndef ARRAY_H
#define ARRAY_H
 
#include <assert.h> // for assert()
 
template <class T>
class Array
{
private:
    int m_length;
    T *m_data;
 
public:
    Array()
    {
        m_length = 0;
        m_data = nullptr;
    }
 
    Array(int length)
    {
        m_data = new T[length];
        m_length = length;
    }
 
    ~Array()
    {
        delete[] m_data;
    }
 
    void Erase()
    {
        delete[] m_data;
        // We need to make sure we set m_data to 0 here, otherwise it will
        // be left pointing at deallocated memory!
        m_data = nullptr;
        m_length = 0;
    }
 
 
    T& operator[](int index)
    {
        assert(index >= 0 && index < m_length);
        return m_data[index];
    }
 
    // The length of the array is always an integer
    // It does not depend on the data type of the array
    int getLength();
};
 
#endif
Array.cpp:
#include "Array.h"
 
template <typename T>
int Array<T>::getLength() { return m_length; }
使用时：链接时出错
main.cpp:
#include "Array.h"
 
int main()
{
	Array<int> intArray(12);
	Array<double> doubleArray(12);
 
	for (int count = 0; count < intArray.getLength(); ++count)
	{
		intArray[count] = count;
		doubleArray[count] = count + 0.5;
	}
 
	for (int count = intArray.getLength()-1; count >= 0; --count)
		std::cout << intArray[count] << "\t" << doubleArray[count] << '\n';
 
	return 0;
}
```cpp

为什么呢：？
实例化模板的必要条件：
   为了让编译器去使用一个模板，它必须看到包括模板定义(而不是只有一个声明),和用于实例化模板的模板类型如int等；
上述例子为什么不成功？
   且记住C++是单独编译文件的；当Array.h头文件被包含到main时，模板类定义被拷贝到main.cpp,当编译器看到我们需
要两个模板实例，Array<int>和Array<double>,它将实例化他们，且编译他们作为main.cpp中的一部分；
然而，当他单独的获取和编译Array.cpp时，他已经忘记我们需要一个Array<int>和Array<double>（单独编译，看不到main),所以Array.cpp中的模板函数不会被实例化；因此会因为找不到函数getLength()的定义而得到链接错误；

```cpp
   针对上述问题，有几种处理方式：
     1)把定义全放在.h头文件
  2)把 rename Array.cpp to Array.inl (.inl stands for inline), and then include Array.inl from the bottom of the Array.h header. That yields the same result as putting all the code in the header, but helps keep things a little cleaner.
 3）在main中include所有：
// Ensure the full Array template definition can be seen
#include "Array.h"
#include "Array.cpp" // we're breaking best practices here, but only in this one place
 
// #include other .h and .cpp template definitions you need here
 
template class Array<int>; // Explicitly instantiate template Array<int>
template class Array<double>; // Explicitly instantiate template Array<double>
 
// instantiate other templates here
```

### 模板类类参数多样性

```cpp

模板类参数可以是无类型，即某种特定的类型：
A value that has an integral type or enumeration
A pointer or reference to a class object
A pointer or reference to a function
A pointer or reference to a class member function
std::nullptr_t
例子：
#include <iostream>
 
template <class T, int size> // size is the non-type parameter
class StaticArray
{
private:
    // The non-type parameter controls the size of the array
    T m_array[size];
 
public:
    T* getArray();
	
    T& operator[](int index)
    {
        return m_array[index];
    }
};
 
// Showing how a function for a class with a non-type parameter is defined outside of the class
template <class T, int size>
T* StaticArray<T, size>::getArray()
{
    return m_array;
}
 
int main()
{
    // declare an integer array with room for 12 integers
    StaticArray<int, 12> intArray;
 
    // Fill it up in order, then print it backwards
    for (int count=0; count < 12; ++count)
        intArray[count] = count;
 
    for (int count=11; count >= 0; --count)
        std::cout << intArray[count] << " ";
    std::cout << '\n';
 
    // declare a double buffer with room for 4 doubles
    StaticArray<double, 4> doubleArray;
 
    for (int count=0; count < 4; ++count)
        doubleArray[count] = 4.4 + 0.1*count;
 
    for (int count=0; count < 4; ++count)
        std::cout << doubleArray[count] << ' ';
 
    return 0;
}

```

### 类模板的特化
什么是类模板的特化，为什么需要，什么时候使用？
类模板的特化和函数模板特化类似，当我们定义了一个类模板后，这个类模板可以实例成int ,bool,char等等类型，都是同样的处理逻辑；
当我们想针对bool特殊化，比如bool本身用一个bit就可以实现，节省空间，时，可以用类模板的特化，来为实例成bool时定义一个类模板的特化，这样，bool就和其他int等不同了；

### 例子

```cpp
template <class T>
class Storage8
{
private:
    T m_array[8];
 
public:
    void set(int index, const T &value)
    {
        m_array[index] = value;
    }
 
    const T& get(int index)
    {
        return m_array[index];
    }
};

特化bool:
template <> // the following is a template class with no templated parameters
class Storage8<bool> // we're specializing Storage8 for bool
{
// What follows is just standard class implementation details
private:
    unsigned char m_data;
 
public:
    Storage8() : m_data(0)
    {
    }
 
    void set(int index, bool value)
    {
        // Figure out which bit we're setting/unsetting
        // This will put a 1 in the bit we're interested in turning on/off
        unsigned char mask = 1 << index;
 
        if (value)  // If we're setting a bit
            m_data |= mask;  // Use bitwise-or to turn that bit on
        else  // if we're turning a bit off
            m_data &= ~mask;  // bitwise-and the inverse mask to turn that bit off
	}
	
    bool get(int index)
    {
        // Figure out which bit we're getting
        unsigned char mask = 1 << index;
        // bitwise-and to get the value of the bit we're interested in
        // Then implicit cast to boolean
        return (m_data & mask);
    }
};
int main()
{
    // Define a Storage8 for integers (instantiates Storage8<T>, where T = int)
    Storage8<int> intStorage;
 
    for (int count=0; count<8; ++count)
        intStorage.set(count, count);
 
    for (int count=0; count<8; ++count)
        std::cout << intStorage.get(count) << '\n';
 
    // Define a Storage8 for bool  (instantiates Storage8<bool> specialization)
    Storage8<bool> boolStorage;
    for (int count=0; count<8; ++count)
        boolStorage.set(count, count & 3);
 
    for (int count=0; count<8; ++count)
        std::cout << (boolStorage.get(count) ? "true" : "false") << '\n';
 
    return 0;
}
```

### 模板的偏特化
区别于前文说的特化，偏特化指的是部分的特化，比如两个模板类型，将其中一个特化了，或者是
特化为指针等等；见下面例子：

#### 例子

```cpp
1) 两个类型特化了其中一个
泛化：
template<class T,class Alloc=alloc>
class vector{
    ...
};
特化：
template <class Alloc>
class vector<bool,Alloc>//在类型化时，若传入的第一个参数为bool，则用以下版zf本；
{
   ...
}

```

#### 偏特化为指针
```cpp
泛化：
template <class T>
struct iterator{
    typedef typename int ittt;
};
偏特化：
template <class T*>
struct iterator{
    typedef typename double ittt;
};
```
