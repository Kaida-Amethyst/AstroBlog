---
title: C++为什么析构函数是虚函数
description: 解释一下为什么析构函数经常是虚函数
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/5.jpg
category: 技术原理
published: 2022-11-12
tags: [C++]
---
------

先要澄清一点，析构函数是可以设定为虚函数，而不是必然会设定为虚函数。默认的类的析构函数不是虚函数。只有那些用到继承特性的类才需要虚析构函数。

这个问题其实只需要用一句话就可以解释，跟所有虚函数的使用情景相同，因为我们可能会用父类的指针去析构一个对象。

```cpp
// a.cpp
#include <iostream>

class Base {
public:
  Base() {
    std::cout << "Create Base!" << std::endl;
  }
  virtual ~Base() {
    std::cout << "Destory Base!" << std::endl;
  }
};

class Derived : public Base {
public:
  Derived() {
    std::cout << "Create Derived!" << std::endl;
  }
  virtual ~Derived() {
    std::cout << "Destory Derived!" << std::endl;
  }
};

int main() {
   auto a = new Base();
   auto b = new Derived();

   delete a;
   delete b;
   return 0;
}
```

编译运行一下：

```shell
$g++ a.cpp && ./a.out
Create Base!
Create Base!
Create Derived!
Destory Base!
Destory Derived!
Destory Base!
```

这个当然我们一点也不意外。

现在，做一点改动，把源代码第26行改成：

```cpp
Base* b = new Derived();
```

再次编译运行：

```shell
$g++ a.cpp && ./a.out
Create Base!
Create Base!
Create Derived!
Destory Base!
Destory Derived!
Destory Base!
```

注意这里的`b`是一个`Base`的指针，它正确地找到了所管理的类的析构函数。

接下来，我们取消`Base`和`Derived`的析构函数`virtual`，再次编译运行：

```shell
$g++ a.cpp && ./a.out
Create Base!
Create Base!
Create Derived!
Destory Base!
Destory Base!
```

这个时候，就没有`Destory Derived!`的信息了，这说明`delete b`调用的是父类的析构函数。

在这种情况下，非常容易有内存泄漏的问题，因此如果类需要继承，那么析构函数就需要虚函数。（父类的析构函数有virtual标记即可）。
