---
title: C++消除warning
description: 我们写的程序出现的warning可能无法避免，此时，我们可以在代码中加入一些内容消除这些warning。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/5.jpg
category: Programming
published: 2023-04-02
tags: [C++]
---

偶尔，我们写的程序出现的warning可能无法避免，此时，我们可以在代码中加入一些内容消除这些warning的产生。

例如，在ISO C++中，对于用户自定义的literal，必须有一个下划线，例如，如果我们想定义一个`XF`​后缀，把前面的数按原位解释成浮点数：

```cpp
#include <string>

consteval float operator""_XF(unsigned long long input) {
  return __builtin_bit_cast(float, static_cast<int>(input));
}

int main() {
  float x = 0x3f800000_XF;
}
```

这样写当然没有什么问题，但是有时我们可能会觉得这个`_`非常碍眼，如果直接不使用`_`，编译器通常会报一个warning。对于GCC编译器来说，会比较宽容，尽管GCC编译器会报warnning，但你真的这么做了GCC是允许的，只是会报warning。（但是，Clang会直接禁止你这么做。）

由于GCC只是报warning，那么我们可以在代码中消去这个warning，我们可以先直接实现代码，看GCC报出了什么样的warning:

```cpp
// main.cc
#include <string>

consteval float operator"" XF(unsigned long long input) {
  return __builtin_bit_cast(float, static_cast<int>(input));
}

int main() {
  float x = 0x3f800000XF;
}
```

使用`g++ main.cc -std=c++20`，我们可以看到warning:

```plaintext
main.cc:4:17: warning: literal operator suffixes not preceded by ‘_’ are reserved for future standardization [-Wliteral-suffix]
  4 | consteval float operator"" XF(unsigned long long x) {
      | 
```

看到这里的warning选项是`-Wliteral-suffix`，意味着我们可以在GCC编译选项中使用：`-Wno-literal-suffix`​来忽略这个warning。而在代码中，我们可以这么做：

```cpp
// main.cc
#include <string>

#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wliteral-suffix"
consteval float operator"" XF(unsigned long long input) {
  return __builtin_bit_cast(float, static_cast<int>(input));
}
#pragma GCC diagnostic pop

int main() {
  float x = 0x3f800000XF;
}
```

这样一来，直接使用`g++ main.cc -std=c++20`命令来编译，也不会出现warning提示信息了。
