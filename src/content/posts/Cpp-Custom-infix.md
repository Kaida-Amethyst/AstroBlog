---
title: C++自定义中缀运算符
description: C++编程中，借助运算符重载特性，可以自定义中缀运算符。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Java/4.jpg
category: 编程实践
published: 2024-06-02
tags: [C++]
---

-------

## Overview

看下面的代码：

```cpp
int main() {
  float x = 2.0;
  float y = 3.0;
  float z = x <pow> y;
  std::print("z = {}", z);
  return 0;
}
```

上面的代码中会打印出8.0，这个非常容易看出来。主要的问题是，其中的代码`float z = x <pow> y`非常有意思。它似乎定义了一种中缀运算符。

实际上这并不是C++23的新特性，而是一种很简单的trick。如果你使用clang-tidy这样的代码规整工具，就可以很容易看出来，因为clang-tidy可以将这一行代码，替换为`x < pow > y`。

如果你熟悉C++的运算符重载，此时你应该就能够大概猜出这里的玄机，这里的pow应该是某种类封装了pow函数，接着`x < pow`则是重载了`<`运算符，得到一个包含左操作数的闭包，接着这个闭包再与`y`做`>`运算，得到一个包含两个操作数的闭包，最后赋值，通过重载强制类型转换运算符得到最终的结果。

## The Simplest Implementation

按照这种思路，我们可以实现一个最简单的，让上面的代码可以运行的实现。

首先做一个Function类，保存函数，为了实现最简单的版本，这里是直接使用了STL的`std::function`，在实际开发的过程中，可能需要按照需求来对Function类做特殊处理。

```cpp
struct Function {
  std::function<float(float, float)> f;
  float operator()(float a, float b) {
    return f(a, b);
  }
}
```

接着，我们要实现一个`Closure_with_left`类，用来代表包含了左操作数的闭包：

```cpp
struct Closure_with_left {
  Function f;
  float left;
};
```

然后，重载`operator<`，让`x < pow`这样的表达式返回一个`Closure_with_left`。

```cpp
Closure_with_left operator<(float a, Function f) {
  return {f, a};
}
```

接着，对于整个`x <pow> y`来说，`x < pow`是一个`Closure_with_left`类，这个类与`y`再做`>`运算，应该得到一个包含两个操作数的闭包，我们将其命名为`Closure_with_both`：

```cpp
struct Closure_with_both {
  Function f;
  float left;
  float right;
};
```

然后再重载`>`运算符，让`Clousre_with_left`和`>`做运算可以得到`Closure_with_both`。

```cpp
Closure_with_both operator>(Closure_with_left a, float b) {
  return {a.f, a.left, b};
}
```

这样一来，`x <pow> y`的部分就完成了，`x <pow> y`将会得到一个`Closure_with_both`，接下来要处理的是，如何让它执行这个函数。

如果需要让`float z = x <pow> y`能够执行成功的话，只需要重载`float()`转换就可以了，也就是说，将`Closure_with_both`修改一下，变成下面的形式：

```cpp
struct Closure_with_both {
  Function f;
  float left;
  float right;

  operator float() {
    return f(left, right);
  }
};
```

这样就可以执行`float z = x <pow> y`了。

## More Problem

上面展示的是在C++里面设计中缀运算符的方法，而实际上，中缀运算符的问题并不那么简单，在实际的开发过程中，需要考虑下面的问题：

1. 类型问题，也就是Function，Closure中的各种类型问题。
2. 复合表达式运算符优先级的问题，也就是说，假设这样的表达式：`float r = x <mul> y <pow> z;`，这里面会有一个运算符优先级的问题需要解决。

## Full Code

```cpp
#include <iostream>
#include <functional>
#include <cmath>

struct Function {
  std::function<float(float, float)> f;
  float operator()(float a, float b) {
    return f(a, b);
  }
};
struct Closure_with_left {
  Function f;
  float left;
};

struct Closure_with_right {
  Function f;
  float right;
};

struct Closure_with_both {
  Function f;
  float left;
  float right;

  operator float() {
    return f(left, right);
  }
};

Closure_with_left operator<(float a, Function f) {
  return {f, a};
}

Closure_with_both operator>(Closure_with_left a, float b) {
  return {a.f, a.left, b};
}

int main() {
  Function pow = {[](float a, float b) { return std::pow(a, b); }};

  float x = 2.0;
  float y = 3.0;
  float z = x <pow> y;

  std::cout << z << std::endl;
}
```
