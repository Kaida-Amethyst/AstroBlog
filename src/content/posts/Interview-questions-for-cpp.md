---
title: C++常见面试题（初阶版）
description: 记录一下初级C++程序员的常见面试题。
published: 2022-07-16
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Javascript/5.jpg
category: 技术原理
tags: [C++]
---

## C++编译过程

编译的四个阶段：

1. 预处理，处理以#为开头的指令
2. 编译，把C/C++代码编译成汇编代码(.s)
3. 汇编，把汇编代码翻译成机器指令，生成目标文件(.o)。
4. 链接，把代码中使用到的静态库动态库链接进来，最后生成可执行文件。

## 静态链接和动态链接的区别

静态链接就是把静态库中的代码拷贝到最终的可执行文件当中，当程序运行的时候，要使用的这些代码会出现在进程的虚拟地址空间中。

动态链接在链接时不会把整个要使用的代码都拷贝到可执行文件中，而是只会记录到共享对象的一些信息，在运行的时候，代码才会被装载到进程的虚拟地址空间中。

优缺点：如果使用静态链接，那么由这个静态库所衍生的所有可执行文件都将有这个目标文件的代码，这会很浪费空间。而且一旦对静态库进行改动，所有的可执行文件都需要重新编译，非常麻烦。好处是运行的时候速度会比较快。

如果使用动态链接，则每次执行都需要链接，会带来效率上的损失，但是可以节省空间，而且更新方便，只需要编译一次。

## C++内存管理

C++内存分区，由低地址向高地址包括：

1. 代码区（.text段）
2. 常量存储区（.data段）保存C++中的const常量。
3. 全局变量与静态变量区（.bss段和.data段）
4. 堆区（.heap）由程序员动态分配
5. unused 从堆到栈有一片空白的区域
6. 栈区（.stack）存放函数的局部变量，函数参数，返回地址等等。

## 堆和栈的区别

1. 堆由程序员动态申请，栈由编译器进行分配
2. 栈是一片连续的空间，分配的时候，如果剩余的空间大于申请的空间，则分配成功，否则出现stackoverflow栈溢出错误，堆的空间不是连续的，而是一片类似链表的数据结构，在程序员申请空间时，系统会首先寻找第一个剩余空间大于申请空间的节点，然后再分配给程序。
3. 栈分配由编译器控制，且是一片连续的空间，效率高。堆由程序员申请，空间不连续，效率较低，且容易产生碎片的问题。
4. 栈中存储的是局部变量，以及函数参数。堆中存放的内容则完全由程序员控制。

## 全局变量，静态全局变量，局部变量，静态局部变量的区别

1. 全局变量具有全局作用域，只需要在一个源文件中定义，所有的代码，文件均可以访问这个变量。在没有定义全局变量的文件中，需要使用extern来声明这是一个全局变量。
2. 静态全局变量仅在这个文件中发挥作用，在这个文件中定义的静态全局变量，在这个文件下的所有代码均可以访问这个变量，但是文件之外的代码和程序是不可以访问这个变量的。使用static来声明静态变量。
3. 局部变量的生命周期仅存在于一对包括了它声明和定义的花括号内，在程序结束这个花括号所存在的段代码时，这个局部变量也同时被销毁，内存会被回收。
4. 静态局部变量从被程序初始化开始到程序运行结束一直存在，但是只能在声明和定义它的函数内才可见。

## 头文件中定义全局变量有什么问题

如果头文件被include多次，就会造成全局变量被多次重复定义。不能使用#ifndef#endif，因为不同的.c文件都需要include这个头文件，从来在链接的时候出现重复定义的问题。需要使用extern来声明这是一个全局变量。

## 限制对象只能在栈上创建或者只能在堆上创建

### 只能在堆上创建

如果需要对象只能在堆上创建，就是要求对象的构造和释放都只能由程序员来决定，这个时候所采用的方法，就可以是把析构函数置为私有，这样编译器就会因为无法访问析构函数而不会在栈上给对象分配空间。

```cpp
class A {
public:
    A() {}
    void destory() { delete this;}
private:
    ~A(){}
};
```

但是只有这样做有一些缺点，第一是无法通过delete来释放对象空间，因为析构函数被置为私有，必须另外写一个destory函数在帮助销毁对象。第二是如果没法派生一个类，因为子类没法访问析构函数。因此，改进的策略就应该是把析构函数设置为protected。

```cpp
class A {
public:
    A() {}
    void destory() { delete this;}
protected:
    ~A(){}
};
```

当然，也可以把构造函数和析构函数都置为protected，然后使用一个专门的create函数进行构建。

```cpp
class A {
protected:
    A() {}
    ~A() {}
public:
    static A *create() {return new A();}
    void destory() {delete this;}
};
```

### 只能在栈上创建

那就禁用new就可以了，把new置为private即可，并且把delete也要置为private。

```cpp
class A {
private:
    void *operator new(size_t t) {}  
    void operator delete(void *ptr) {}
public:
    A() {}
    ~A() {}
};
```

## 为什么要有内存对齐

因为硬件的接口往往需要对象的地址必须是4或者8的倍数（有时也可以是2）。譬如如果有一个struct，第一个参数是int，第二个参数是char，第三个参数是int，这个时候，第一个参数占有4个字节，第二个参数占有一个字节，但是这个struct会多一个3字节的填充区域，然后再是第三个参数的4个字节。此时整个struct的大小就是12字节，是4的倍数。

## 类的大小

成员变量的大小遵循内存对齐，当存在虚函数时，多增加一个虚函数表的指针的8字节。如果是空类，就是1字节。

```cpp
#include <iostream>

using namespace std;

class A {
private:
    static int s_var; // 不影响类的大小
    const int c_var;  // 4 字节
    int var;          // 8 字节 4 + 4 (int) = 8
    char var1;        // 12 字节 8 + 1 (char) + 3 (填充) = 12
public:
    A(int temp) : c_var(temp) {} // 不影响类的大小
    ~A() {}                      // 不影响类的大小
    virtual void f() { cout << "A::f" << endl; }
    virtual void g() { cout << "A::g" << endl; }
    virtual void h() { cout << "A::h" << endl; } // 24 字节 12 + 4 (填充) + 8 (指向虚函数的指针) = 24
};

class B{
};

int main()
{
    A ex1(4);
    A *p;
    B ex2;
    cout << sizeof(p) << endl;   // 8 字节 注意：指针所占的空间和指针指向的数据类型无关
    cout << sizeof(ex1) << endl; // 24 字节
    cout << sizeof(ex2) << endl; // 1 字节
    return 0;
}

```

## 什么是内存泄漏

程序员在堆上分配内存后，因为疏忽或者程序中的bug导致没有及时地释放内存，造成堆上内存的失控。解决的办法就是排查错误，使用free或者new来释放内存。

## 怎样防止内存泄漏，以及如何检测内存泄漏

及时使用free和delete。

可以使用valgrind工具来进行检测。

## 什么是面向对象？面向对象的三大特性

所谓对象或者类，就是把客观世界里的事物进行分类和抽象。

三大类型：

1. 封装，把数据和实现方法封装在一个类中，对外只暴露接口，以此来降低耦合性。
2. 继承，允许一个类从另一个类中派生出来，使得抽象的父类，可以派生出一个更加具体的子类。
3. 多态，允许一个方法对不同的对象采取不同的动作。

## 什么是多态？多态如何实现？

多态就是允许同一个方法针对不同的对象采取不同的动作。C++中通过虚函数实现，在需要实现多态的函数前加上virtual关键字，在子类中给出具体实现即可。

## 智能指针有哪几种？智能指针的实现原理？

1. shared_ptr，共享指针。实现原理是引用计数，可以记录这个对象被多少个指针所指向。
2. unique_ptr，独占指针，要求对象只能被一个指针所指向。不能被赋值构造，只能被移动构造或者移动赋值。
3. weak_ptr，弱指针，接受share_ptr但是不增加引用计数，主要是解决shared_ptr所带来的循环引用的问题。

## 一个 unique_ptr 怎么赋值给另一个 unique_ptr 对象？

使用std::move进行移动构造或者移动赋值构造。

## 使用智能指针会出现什么问题？怎么解决？

主要是shared_ptr共享指针会出现问题，比如说循环引用，两个类，互相有指向对方的指针，这种情况下会因为引用计数始终不为0而导致对象无法被析构，使用把其中一些shared_ptr换成weak_ptr即可解决。

## sizeof 与 strlen的区别

1. strlen是cstring下的一个函数，sizeof是一个运算符。
2. strlen函数的返回结果是字符串的长度，sizeof的结果是为这个字符串所分配的大小。
3. 如果作为函数参数，sizeof测量的实际上是指针大小，strlen仍然是字符串长度。

## lambda表达式（匿名函数）的具体应用和使用场景

用法：

```cpp
[capture list]<template prarmeters>(parameter list) -> Type {body}
```

用途

常与标准库中algorithm联用，例如排序算法。

## explicit的作用

用在单参数的类构造函数，用来阻止传参时发生的隐式转换。

反例：

```cpp
class A{
private:
    size_t N;
public:
    A(size_t a):N{a}{};
    void print(){
        std::cout << N << std::endl;
    }
};

int main() {
    A a = -1;
    a.print();
    return 0;
}
```

```
18446744073709551615
```

加了explicit之后：

```cpp
class A{
private:
    size_t N;
public:
    explicit A(size_t a):N{a}{};
    void print(){
        std::cout << N << std::endl;
    }
};


int main() {
    A a = -1;
    a.print();
    return 0;
}
```

```
F:\C&C++ Project\SearchTrees\BinaryTree\main.cpp: In function 'int main()':
F:\C&C++ Project\SearchTrees\BinaryTree\main.cpp:15:12: error: conversion from 'int' to non-scalar type 'A' requested
     A a = -1;
            ^
mingw32-make.exe[3]: *** [CMakeFiles\BST.dir\build.make:62: CMakeFiles/BST.dir/main.cpp.obj] Error 1
mingw32-make.exe[2]: *** [CMakeFiles\Makefile2:75: CMakeFiles/BST.dir/all] Error 2
mingw32-make.exe[1]: *** [CMakeFiles\Makefile2:82: CMakeFiles/BST.dir/rule] Error 2
mingw32-make.exe: *** [Makefile:117: BST] Error 2
```

直接报错。

## static关键字的作用

定义局部静态变量，全局静态变量，静态函数，静态成员变量，静态成员函数。

局部静态变量：全局存在，但作用于局部。

全局静态变量：对于文件可见，文件外不能访问。

静态函数：对于文件可见，文件外不能访问。

静态成员变量，静态成员函数：所有成员均可访问相同的变量和成员函数。而且不定义类也可以使用它们。（静态成员函数不可以是虚函数，const和volatile）。

## static变量和全局变量的异同

全局变量具有全局作用域，static静态变量具有文件作用域。

## const的作用及用法

1. 定义常量
2. 修饰函数参数，使得函数参数不能被修改。
3. 修饰成员函数，使得该函数不能修改成员变量。注意const函数不可以调用非const的成员函数。
4. const成员变量在不同对象中可以不同。定值的时候，要么直接给值，要么通过列表初始化赋值。

## define和const的区别

1. define是编译预处理时进行替换，const是编译期时定值。
2. define是直接替换，它没有数据类型，不会进行类型检查，const的数据都是有类型的，会进行检查。
3. define出来的值，在程序中使用多少次就进行多少次替换，在内存中会有多个拷贝，位于代码段，const出来的值只有一份，在静态存储段。
4. define出来的值没法调试，const出来的值可以调试。

## define和typedef的区别

define只是简单地替换，它不作安全性检查。

## 用宏实现两数之间求最大最小

```c
#define MIN(X, Y) ((X)>(Y)?(Y):(X))
#define MAX(X, Y) ((X)>(Y)?(X):(Y))
```

## inline的作用

把函数直接展开，减少函数调用的开销。

## define和inline的区别

define是宏定义，展开函数是在预处理阶段就进行展开了，inline则是在编译阶段进行展开。

define不会进行函数参数的检查，而inline函数仍然是函数，它会进行函数参数检查。

## new的作用

C++中的关键字，用来动态分配内存。

## delete实现原理，delete和delete[]的区别

delete实现原理：

先调用对象的析构函数，然后调用标准库下的operator delete来释放内存空间。

delete释放单个对象，delete[]释放对象组。

## new和malloc如何判断是否成功分配内存

malloc分配失败没有异常发生，只会返回一个NULL的指针。

new分配失败会发生bad_alloc异常。

## new/delete和malloc/free的区别

new/delete是C++下的关键字，malloc/free是C下的stdlib的函数。

new无需指定分配空间的大小，分配空间的大小由编译器计算得到，malloc需要指定分配大小。

new返回对象，malloc返回指针

new返回的对象有类型，malloc返回的对象是void*，需要进行强制类型转换。

new失败抛出异常，malloc失败返回NULL指针。

## C++与C中的struct

C++ 中的struct基本就是类，它与class的唯一区别在于，struct如果不加private/public的话，默认都是public的，而class如果不加private/public的话，默认都是private的。C中的struct仅仅是用户自定义的数据类型，不能定义访问权限，不能定义成员函数。写法上也有区别，不使用tyepdef的话，就必须强调struct XXX xx，C++则不用。

## 为什么有了 class 还保留 struct？

为了兼容C语言。

## Struct和Union的区别

struct是集成多种数据，union是为多种数据提供不同的解释。为struct中的某一个变量赋值不会影响到其它变量。为Union中的某一个变量赋值会修改其它变量。

## volatile的作用，是否具有原子性

禁止编译器优化与该变量有关的代码段。不具有原子性。

## 什么情况下一定要用 volatile， 能否和 const 一起使用？

多线程编程时，某一个值是由多个线程共享的，此时需要考虑使用volatile。

const与volatile不冲突。

## 返回函数中静态变量的地址会发生什么？

不变。即使函数运行完毕也还存在。知道程序运行完毕才销毁。

## extern C的作用

指示编译器按照C的方式进行编译和链接，这样有利于C++调用C语言模块。

## sizeof(1==1) 在 C 和 C++ 中分别是什么结果？

C语言中跟sizeof(int)相同，4字节或者8字节，取决于操作系统。

C++中是1，因为C++有bool类型，占1字节。

## strcpy的缺陷

不检查缓冲区边界大小，直接赋值给一块连续的内存空间，可能会导致不安全的现象发生。

## 什么是虚函数？什么是纯虚函数？

使用virtual关键字修饰过的就是虚函数。纯虚函数是既加上了vritual，又加上了=0的成员函数。含有纯虚函数的类都是抽象类，均不可以被实例化。但是可以声明抽象类的指针和引用。

## 虚函数和纯虚函数的区别？

1. 虚函数可以直接使用，纯虚函数必须在派生类实现后使用。
2. 虚函数必须实现。
3. 虚函数只需要加virtual，纯虚函数还需要加=0；

## 虚函数的实现机制（可能需要拓展）

通过虚函数表，同一类的不同对象共用一个虚函数表。不同类使用不同的虚函数表。

## 如何禁用构造函数

在构造函数后面加一个=delete.

## 什么是类的默认构造函数？

由编译器自动生成的构造函数。可以不声明，或者声明了以后用=default来修饰。

## 构造函数、析构函数是否需要定义成虚函数？为什么？

构造函数不能定义为虚函数，因为对象要先被构造，然后接收虚函数表。

析构函数一般需要被定义为虚函数，因为我们经常是通过基类的指针来释放派生类的内存。

## 如何禁用对象拷贝

把拷贝构造函数和赋值构造函数设置成=delete。

```cpp
class noncopyable {
public:
    noncopyable(const noncopyable&) = delete;
    noncopyable& operator=(const noncopyable&) = delete;
};
```

## 如何减少构造函数开销？

使用初始化列表。

## 多继承出现的问题，以及如何解决

容易出现数据冲突

```cpp
#include <iostream>
using namespace std;

// 间接基类
class Base1 { public: int var1;};
// 直接基类
class Base2 : public Base1{ public: int var2;};
class Base3 : public Base1{ public: int var3;};

// 派生类
class Derive : public Base2, public Base3{
private:
    int var4;
public:
    void set_var1(int tmp) { var1 = tmp; } // 有错误，var1指的是base1的var1还是base2的var1?
    void set_var2(int tmp) { var2 = tmp; }
    void set_var3(int tmp) { var3 = tmp; }
    void set_var4(int tmp) { var4 = tmp; }
};

int main() {
    Derive d;
    return 0;
}
```

上面的这种继承方式就是菱形继承，这里的var1会发生冲突。

办法一：指明冲突数据的所属

```cpp
class Derive : public Base2, public Base3{
private:
    int var4;
public:
    void set_var1(int tmp) { Base1::var1 = tmp; } // ok
    void set_var2(int tmp) { var2 = tmp; }
    void set_var3(int tmp) { var3 = tmp; }
    void set_var4(int tmp) { var4 = tmp; }
};
```

办法二：虚继承

```cpp
class Base2 : virtual public Base1{ public: int var2;};
class Base3 : virtual public Base1{ public: int var3;};
```

## 空类占多少字节？空类定义时编译器会生成那些函数？

空类占1字节。

默认构造函数，默认析构函数，复制构造函数，赋值构造函数，取址运算符，const取址运算符。

## 为什么拷贝构造函数必须为引用？

避免无限递归。

## C++ 类对象的初始化顺序

先调用基类的构造函数，再调用成员变量的构造函数，最后调用自身的构造函数。

## 如何禁止一个类被实例化？

构造函数设置为delete。或者放在private底下。或者声明一个纯虚函数。

## 为什么用成员初始化列表会快一些？

因为成员变量的构造先于构造函数的调用。如果成员变量的初始化放在构造函数里，则相当于成员变量被构造了两次。

## 实例化一个对象需要哪几个阶段

1. 分配内存空间。
2. 初始化。（初始化成员函数，虚函数表）
3. 赋值。（调用构造函数）

## 什么是友元

友元函数或者友元类可以访问类的私有成员。

## 深拷贝和浅拷贝的区别

深拷贝：两个对象两块内存。

浅拷贝：两个对象共享一块内存。

## 编译时多态和运行时多态的区别

编译时多态指的是C++模板（泛型编程）和函数重载。

运行时多态指的是虚函数应用。

## 实现一个类成员函数，要求不允许修改类的成员变量？

加个const即可。

## 如何让类不能被继承？

1. 使用final关键字。
2. 或者，构造函数置为private。

## 左值和右值的区别？左值引用和右值引用的区别，如何将左值转换成右值？（是否能够结合汇编代码？）

左值：表达式结束后依然存在的对象

右值：表达式结束后不再存在的对象

左值引用不能引用表达式，字面量或者返回右值的表达式。右值则正好相反。

右值引用通过&&获得。

使用std::move（move语义）将一个左值强行转化为右值。

## std::move()的原理

本质上是一个强制类型转换，相当于static_cast<&&>

## 什么是野指针和悬空指针？

悬空指针指向一片已经被回收过内存的内存空间。野指针是不知道具体指向的指针，比如没有初始化化过的指针。

## nullptr与NULL的区别

NULL是C语言下的一个宏，本质上是一个数值0.

nullptr是C++11下的一个关键字，是一种特殊类型的字面值，可以被转换为任意类型。

nullptr是有类型的，它的类型就是nullptr_t。

## 指针和引用的区别

指针指向谁可以改变，引用一旦确定是不能改变的。

指针本身就占内存，引用不占内存。

指针可以为空，引用不能为空。

指针可以有多级，引用不能有多级。

## 常量指针（const int *）和指针常量（int * const）

常量指针（const int *）指的是，不能通过这个指针修改对象的值。指针常量（int * const）指的是，不能修改这个指针本身的值。

## 函数指针和指针函数的区别

函数指针：指向函数的指针。

指针函数：返回指针的函数。

## 强制类型转换有哪几种？（需要重点注意）

1. static_cast，用于基本数据类型的转换，或者基类和派生类之间指针或者引用的转换。
2. const_cast，多用于去除或添加指针或者引用的const属性。
3. reinterpret_cast，用于改变指针或者引用的类型。
4. dynamic_cast，程序运行时进行转换，用于带有虚函数基类或派生类之间的相互转换。

## 如何判断结构体是否相等？能否用 memcmp 函数判断结构体相等？

重载==操作符进行成员变量的一一比对。不可以用memcmp因为structure会有内存对齐，而填充部分的值不能保证相等。

## 参数传递时，值传递、引用传递、指针传递的区别？

值传递：形参是实参的拷贝，函数对形参的所有操作不会影响到实参。

引用传递：形参就是实参。或者说形参时实参的别名。

指针传递：本质上是值传递，只不过两个指针指向的是同一块内存，所以用指针也可以修改对象的值。

## 什么是模板特化？为什么特化？

当我们希望对于特殊的类型，采取不同于模板描述的方法时，需要进行特化。特化就是写一个我们想要特化类型的模板。

例子：

```cpp
template <class T>
bool compare(T t1, T t2) {
    cout << "通用版本：";
    return t1 == t2;
}

template <> //函数模板特化
bool compare(char *t1, char *t2) {
    cout << "特化版本：";
    return strcmp(t1, t2) == 0;
}
```

## 迭代器的作用？

在无需知道容器底层原理的情况下，提供一种遍历容器元素的工具。

# 什么是静态绑定和动态绑定？如何实现的？

静态绑定：对象的类型在编译期就能确定。

动态绑定：对象的类型在运行时确定。

实现方式：通过虚函数。
