---
title: cpp_template_func
date: 2021-01-10 00:50:01
tags: cpp_template
categories: c&cpp
---

### 概述：


为什么需要函数模板：类型限制了函数的通用性，参数换一种类型，即得重新再定义一个处理流程相同的函数；<!-- more -->
```cpp
eg：
int max(int x, int y)
{
    return (x > y) ? x : y;
}
double max(double x, double y)
{
    return (x > y) ? x : y;
}
char,ints等等，甚至实现了>运算符函数的类也是如此；
```

### 什么是函数模板：
In C++, function templates are functions that serve as a pattern for creating other similar functions.
在c++中，函数模板即是能作为一个模式来创建其他相似函数的一组函数；
在c++函数模板中，我们使用占位符来替代部分或全部的函数中具体类型的变量；
返回值，形参，以及函数内定义的局部变量等都能使用；
typename和class 的区别：http://www.cplusplus.com/forum/general/8027/

### 例子

```cpp
#include<iostream>                                                             
//using namespace std;导入这个为什么会出错？这里也定义了min函数，那么会冲突？                            
using  std::cout;                                                              
using  std::endl;                                                              
template <typename T>                                                          
   /* T min( T x, T y){ 
            T t=x;                                          
        return (x+t>y)?y:x;                                                      
}   */
T min( T x, T y){                                       
        return (x>y)?y:x;                                                      
}                                                                               
int main ()                                                                    
{                                                                              
    int i = min(3,4);                                                          
    cout<<i<<endl;                                                             
                                                                               
    double d=min(6.38,12.32);                                                  
    cout<<d<<endl;                                                             
                                                                               
    char ch=min('a','6');                                                      
    cout<<ch<<endl;                                                            
                                                                               
    return 0;                                                                  
}                                     
```
```cpp
建议，这里的模板定义加上const和使用引用会更好，使用引用的原因是传入的参数有可能是类类型(这样参数类型和返回值应为引用更通用），使用const是为了避免对传入参数的原变量造成影响；如下：
    template <typename T>
    const T& min(const T& x,const T& y){
        return (x>y)?y:x;
    }                           
```

    
#### 更多使用例子
 
用于类类型:注意类需要实现对应的运算符函数等；
模板类型和实际类型混合
指针类型：
```cpp
template <class T>
T average(T *array, int length)
{
    T sum = 0;
    for (int count=0; count < length; ++count)
        sum += array[count];
 
    sum /= length;
    return sum;
}
```
多个模板类型

### 假设这里我传入了int* 会怎么样？
```cpp
来看一个例子：指针的话会优先匹配指针
#include<iostream>
//using namespace std;导入这个为什么会出错？这里也定义了min函数，那么会冲突？                            
using  std::cout;
using  std::endl;
template <typename T>
   /* T min( T x, T y){ 
            T t=x;                                          
        return (x+t>y)?y:x;                                                      
}   */
T min( T* x, T* y){//这样的话，这里的x实际上是指针类型，比如传入int* ==> T* ，要返回int的话，得对x做*，如*x
      cout<<"real"<<endl;
      return (*x>*y)?*y:*x;
}
/*template<typename T> 模板只能匹配参数，不能匹配返回值，所以这里和上面的是重复的；编译会报错；
T* min(T* x,T* y)
{
     return (x>y)?y:x;
}*/
template <typename T>
T min(T x,T y)
{
    return x+y;
}
int main ()
{
    int i = min(3,4);
    cout<<i<<endl;

    double d=min(6.38,12.32);
    cout<<d<<endl;

    char ch=min('a','6');
    cout<<ch<<endl;
//以上匹配的是非指针的版本+

    cout<<"now pointer"<<endl;
    int m =3;
    int n = 4;
    int jj=min(&m,&n); //这里匹配的是指针的
    cout<<"m:"<<&m<<"n:"<<&n<<endl;
    cout<<min(&m,&n)<<endl;
    cout<<jj<<endl;
    return 0;
}
~                                    
```

### 函数模板的特化 什么时候需要

引入：有时候，针对具体的类型，我们想定义具体的函数，而不是用模板中对所有类型通用的函数时，可以定义一个函数模板的特化函数；
语法：
eg

```cpp
template <class T>
class Storage
{
private:
    T m_value;
public:
    Storage(T value)
    {
         m_value = value;
    }
 
    ~Storage()
    {
    }
 
    void print()
    {
        std::cout << m_value << '\n';
    }
};
int main()
{
    // Define some storage units
    Storage<int> nValue(5);
    Storage<double> dValue(6.7);
 
    // Print out some values
    nValue.print();
    dValue.print();
}
那么当我们想针对double有个特殊专属的print函数时，应该怎么做？
答案是针对其写一个函数：特化：
template <>
void Storage<double>::print()
{
    std::cout << std::scientific << m_value << '\n';
}
这样，当编译器实例化模板时，double时，就会使用这个特化的print;
The template <> tells the compiler that this is a template function, but that there are no template parameters (since in this case, we’re explicitly specifying all of the types). Some compilers may allow you to omit this, but it’s proper to include it.
```

### 类相关例子：
```cpp
类模板的构造函数也可以这样处理：
template <>
Storage<char*>::Storage(char* value)
{
    // Figure out how long the string in value is
    int length=0;
    while (value[length] != '\0')
        ++length;
    ++length; // +1 to account for null terminator
 
    // Allocate memory to hold the value string
    m_value = new char[length];
 
    // Copy the actual value string into the m_value memory we just allocated
    for (int count=0; count < length; ++count)
        m_value[count] = value[count];
}
```
