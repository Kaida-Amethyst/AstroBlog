---
title: C++的静态继承 - CRTP机制
description: 面向对象中的继承机制要依赖动态多态来实现，不过C++的CRTP机制允许我们进行静态继承
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/1.jpg
category: Programming
published: 2023-04-02
tags: [C++]
---
------

## 运行时多态

在面向对象中，我们可能会有一个父类，用来存放子类中共有的成员变量和方法，例如，假设我们有一个基类`Comparable`，然后它的诸多子类可以进行比较：

```cpp
struct Comparable {
  virtual bool operator==(const Comparable &rhs) const = 0;
  virtual bool operator!=(const Comparable &rhs) const = 0;
  /* ... */
};
```

然后，假设我们有一个自定义的类`MyStruct`，如果我们希望这个类是可比较的，我们就会让其继承这个父类

```cpp
struct MyStruct: Comparable {
  virtual bool operator==(const Comparable &rhs) const override {/* ... */};
  virtual bool operator!=(const Comparable &rhs) const override {/* ... */};
};
```

但是，这种方式可能有一些问题：

1. 虚函数本身会占用一定的类空间，会有一定程度上的性能损耗。
2. 虚函数是没有办法inline的。
3. 虚函数也不能是模板函数，因为虚函数必须是一个运行时概念。
4. 虚函数往往需要使用指针来管理，但是有时我们可能希望更多地利用引用。

由于虚函数是一个完全的运行时机制，很多静态的特性无法使用，而C++除了这种运行时多态外，还提供了一种静态多态的机制：

## CRTP

CRTP全称为：Curiously Recurring Template Pattern，它指的是下面的这种形式：

```cpp
struct MyStruct : SomeTemplate<MyStruct> ;
```

也就是类继承一个以自身为模板参数的模板，这个外模板里面去实现一些共有的成员变量和方法。按照前面一节的目的，利用CRTP，我们可以这样写：

```cpp
template <typename CRTP> struct Comparable {
  inline bool operator==(const Comparable &rhs) const {
    return !(static_cast<const CRTP &>(*this) <
                 static_cast<const CRTP &>(rhs) ||
             static_cast<const CRTP &>(*this) > static_cast<const CRTP &>(rhs));
  }
  inline bool operator!=(const Comparable &rhs) const {
    return !(static_cast<const CRTP &>(*this) ==
             static_cast<const CRTP &>(rhs));
  }
};
```

注意这里使用`>`和`<`，那么就会要求实例化的`Comparable`需要实现`operator>`和`operator<`，

```cpp
struct Mystruct : public Comparable<Mystruct> {
  int a;
  int b;
  Mystruct(int a, int b) : a(a), b(b) {}

  inline bool operator<(const Mystruct &rhs) const {
    return a < rhs.a || (a == rhs.a && b < rhs.b);
  }
  inline bool operator>(const Mystruct &rhs) const {
    return a > rhs.a || (a == rhs.a && b > rhs.b);
  }
};
```

尽管对于基类`Comparable`的接口实现略显复杂（只是看上去复杂），但是其性能会比虚函数这种运行时多态要高出不少。
