---
title: 几个常用的C/C++builtin函数
description: 介绍几个工作中好用且常用的几个C和C++的builtin函数
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/1.jpg
category: Programming
published: 2023-04-02
tags: [C, C++]
---

### `__builtin_expect`

```cpp
long __builtin_expect(long exp, long n);
```

用于指示exp的值很可能等于n，在`[[likely]]`属性出现之前，经常用有程序使用类似于这样的宏：

```cpp
#define likely(ptr) __builtin_expect(!!(ptr), 0)`
```

以上的宏代表`ptr`并非空指针，这种宏可以用于分支预测从而提升性能。C++20之后可以使用属性`[[likely]]`​。

### `__builtin_bit_cast`

用于将数原位转换为另一种类型，例如，将`0x3f800000`按位转换为`float`的1.0.

```cpp
float x = __builtin_bit_cast(float, 0x3f800000);
// same as float x = 1.0f;
```

如果不使用这种功能，就只能这样：

```cpp
float x = ({unsigned _f = 0x3f800000; *(float*)&_f; });  // In C

consteval float _F(unsigned i) {return *(float*)&i;}
float x = _F(0x3f800000);  // In C++20
```

这样一来可能需要优化支持。

### `__builtin_constant_p`


```cpp
bool __builtin_constant_p(exp);
```

用于断定一个参数或者表达式是否可以在编译期进行求值，注意，**通常用来观察经过优化后是否可以在编译期求值**​

```cpp
void foo(int a) {
  std::cout << std::boolalpha << 
               __builtin_constant_p(a)
            << std::endl;
}

int main(int argc, char *argv[]) {
  /* do something */
  foo(1);
  foo(argc);

  return 0;
}
```

开启`-O3`优化后，可以看见`true`和`false`。

### `__builtin_dump_struct` (clang only)

可以用来打印结构体的值：

```cpp
struct A {
  int x;
  float y;
};

int main() {
  A a {1, 2.0};
  __builtin_dump_struct(&a, printf);
  return 0;
}
```

编译运行之后：

```plaintext
A {
  int x = 1
  float y = 2.000000
}
```

注意，这个函数是可以作为编译期函数的：

```cpp
#include <string>
struct T { int a, b; };
constexpr void constexpr_sprintf(std::string &out, const char *format,
                                 auto ...args) {
  // ...
}
constexpr std::string dump_struct(auto &x) {
  std::string s;
  __builtin_dump_struct(&x, constexpr_sprintf, s);
  return s;
}
static_assert(dump_struct(T{1, 2}) == R"(struct T {
  int a = 1
  int b = 2
}
)");
```
