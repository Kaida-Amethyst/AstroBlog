---
title: C++黑魔法 - 编译期变量
description: 编译期变量目前实际上在C++里是不存在的概念，因为C++的模板编程是函数式范式，但是其实可以利用C++编译器的漏洞来间接做到，所以是黑魔法，这里仅仅做有趣的探讨。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/6.jpg
category: 技术原理
published: 2023-03-28
tags: [C++]
---

注意：本篇blog中的代码涉及到Undefined Behaviour，不建议在生产环境中使用。

## Question

我们想实现一个非常特殊的`constexpr`函数，使得：

```cpp
constexpr int f();

int main() {
  constexpr int a = f();
  constexpr int b = f();

  static_assert(a != b, "failed!");
  return 0;
}
```

注意在这一篇blog当中，我们只要求两次调用结果不一致，不需要更多次的调用结果不一致。

注意这个要求实际上非常不容易达成，因为这里的`f`本身就是`constexpr`，理论上来说，这种`constexpr`函数的返回值应该是确定的才是。这道题目要想解出，需要相当熟悉C++的编译器才行。以下是答案：

---

## Solution

```cpp
struct A {
  constexpr A() {}
  friend constexpr int adl_flag(A);
};

template <class Tag> struct writer {
  friend constexpr int adl_flag(Tag) { return 0; }
};

template <class Tag, int = adl_flag(Tag{})>
constexpr bool is_flag_usable(int) {
  return true;
}

template <class Tag>
constexpr bool is_flag_usable(...) { return false; }

template <bool B, class Tag = A> struct dependent_writer : writer<Tag> {};

template <class Tag = A, bool B = is_flag_usable<Tag>(0),
          int = sizeof(dependent_writer<B>)>
constexpr int f() {
  return B;
}

int main() {
  constexpr int a = f();
  constexpr int b = f();
  static_assert(a != b, "failed!");
  return 0;
}
```

以上的代码可以使用g++通过编译（但clang++无法通过）。

---

## Explanation

上面的答案并不好理解，一行一行来看。

首先，我们声明一个类，这个类当中有一个友元函数：

```cpp
struct A {
  constexpr A() {}
  friend constexpr int adl_flag(A); // Notice we don't implement this function
};
```

注意，这里我们特意没有去实现这个函数，这是刻意为之，后面会解释。

然后，我们先跳过中间的`Writer`类，看一下第一个`is_flag_usable`：

```cpp
template <class Tag, int = adl_flag(Tag{})>
constexpr bool is_flag_usable(int) {
  return true;
}
```

这个时候，这个`is_flag_usable`无论如何都是不能直接使用的，因为并不存在一个`adl_flag(Tag)`函数，包括`adl_flag(A)`。也就是说，不管怎么使用，这个`is_flag_usable`都会出现代入失败。

接着看下面的一行：

```cpp
template <class Tag>
constexpr bool is_flag_usable(...) { return false; }
```

这个的意思就是，因为前面返回`true`的`is_flag_usable`无论如何都会代入失败，而代入失败的函数都会经过这个函数。那么到这里，如果我们直接使用`is_flag_useable`，不管参数是什么，都将返回`false`。

`is_flag_useable`解释完之后，接着，我们来看`writer`类：

```cpp
template <class Tag> struct writer {
  friend constexpr int adl_flag(Tag) { return 0; }
};
```

而对于`writer`类，它是有一个`adl_flag`的实现的。

**那么，仔细想想，在前面的**`**A类的时候，**` `adl_flag`**`是没有实现的。但是到了`**`writer`**`类，假如这里的`**`Tag = A` **`，那么我们不就有了一个`**`adl_flag(A)`**`了吗？`** 

所以，对于`f`函数：

```cpp
template <class Tag = A, bool B = is_flag_usable<Tag>(0),
          int = sizeof(dependent_writer<B>)>
constexpr int f() {
  return B;
}
```

那么，在第一次调用`f`的时候，这里的`is_flag_usable`因为`adl_flag(A)`代入失败，因此恒定地返回了一个`false`值。但是，这个函数的第三个木板参数`int = sizeof(depent_writer<B>)`，这个模板参数本身的值没有意义，它的意义在于实例化了一个`dependent_writer`的对象：

```cpp
template <bool B, class Tag = A> struct dependent_writer : writer<Tag> {};
```

这个`dependent_writer`，是继承了writer，而这个类的一旦被实例化，`adl_flag(A)`就会被启动，从而在第二次调用`f()`函数的时候`f`的模板`is_flag_usable`会返回true。因而前后两次f函数的调用会不一致。

但是不仅于此，再仔细观察`f`的第三个模板参数：

```cpp
int = sizeof(dependent_writer<B>)
```

会发现，这里`dependent_writer<B>`，使用这个`B`非常关键，因为这样以来就是强制要求编译器必须在实例化`depent_writer`之前，先取得`B`的确切的值。如果这里的`B`换成其它的参数，同样也会失效。

所以，通过这个例子其实可以猜到编译器的一些内幕：

1. g++不会记录`constexpr`函数的值，而是遇到一次计算一次。而clang++可能会记录`constexpr`函数的值。

# More Detail

* [ ] https://web.archive.org/web/20161217033223/http://b.atch.se/posts/non-constant-constant-expressions/
