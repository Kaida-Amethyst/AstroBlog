---
title: C++ - 从valarry到simd
description: 很久以前，C++就有支持运算符操作的valarray了，C++23之后会有simd，实际上运算符的问题相当复杂，值得探讨一下
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/7.jpg
category: 编程实践
published: 2023-05-26
tags: [C++]
---
-----

## valarray

STL里面有一个古早的模块`std::valarray`，不难猜出就是变长数组：

```cpp
#include <iostream>
#include <valarray>

int main() {
  std::valarray<int> vec = {1,2,3,4,5};
  return 0;
}
```

不过，与常用的`vector`相比，`valarray`实际上是比`vector`更加像vector的。例如，在数学上，一个vector可以直接做运算，但是在C++里面，你必须通过循环或者函数式编程。而`valarray`却重载了一些操作符，这使得`valarray`比`vector`更像`vector`，看下面的代码：

```cpp
#include <iostream>
#include <vector>
#include <valarray>

int main() {
  std::valarray<int> vec = {1,2,3,4,5};
  
  vec += 3;

  std::vector<int> vec2 = {1,2,3,4,5};
  // vec2+=3 will throw an error
  for(auto &i : vec2) {
    i+=3;
  }
  return 0;
}
```

简单列举一些`valarray`支持的运算：

```cpp
#include <iostream>
#include <valarray>

int main() {
  std::valarray<int> vec = {1, 2, 3, 4, 5};

  vec += 3;
  vec = vec * 2;
  vec = vec << 1;

  for (auto &i : vec2) {
    std::cout << i << std::endl;
  }

  return 0;
}
```

普通的加减乘除，位运算都没有问题。除掉这种简单运算外，对于cmath中的复杂运算，STL也提供了实现：

```cpp
#include <iostream>
#include <valarray>

int main() {
  std::valarray<float> vec = {1, 2, 3, 4, 5};
  // For the following line,
  // do not use `auto` to replace
  // `std::valarray<float>` 
  std::valarray<float> vec2 = std::sin(vec);

  for (auto &i : vec2) {
    std::cout << i << std::endl;
  }

  return 0;
}
```

除掉单独的运算外，valarray也有mask的用法：

```c++
#include <iostream>
#include <valarray>

int main() {
  std::valarray<float> vec = {1, 2, 3, 4, 5};

  vec[vec > 3] = 2;  // only assign

  for (auto &i : vec2) {
    std::cout << i << std::endl;
  }

  return 0;
}
```

以上的例子可以看出，`valarray`实际上初步具有`vector`的功能，尽管功能并不完全。

但是，未来ISO C++未必会对`valarray`进一步支持，`valarray`的底层实现仍然是变长数组。尽管在`O3`的优化条件下，可以发现一些指令被优化成了simd指令，但是这不意味着`valarray`原生支持simd指令。

-----

## STL中的simd

如果要使用simd指令，还是需要期待c++26的`std::simd`。

不过，实际上在支持c++20的版本下，我们就已经可以看到`simd`的实验版本了，本文写成时，使用的是`gcc 13.0.1`，将下面的代码编译成x86汇编，使用O0优化，可以看到`xmm`这种simd寄存器已经出现了：

```cpp
#include <array>
#include <experimental/simd>
#include <iostream>

namespace stdx = std::experimental;

int main() {
  // Prepare Memory and data
  std::array<float, 4> a = {1, 2, 3, 4};
  // declare simd data structure
  // need two template parameters
  // datatype and abi
  stdx::simd<float, stdx::simd_abi::native<float>> v;
  // Then load data
  v.copy_from(a.data(), stdx::vector_aligned);

  for (std::size_t i{}; i < 4; ++i)
    std::cout << v[i] << ' ';
  return 0;
}
```

```x86asm
main:
.LFB11422:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	subq	$320, %rsp
	movss	.LC0(%rip), %xmm0
	movss	%xmm0, -272(%rbp)
	movss	.LC1(%rip), %xmm0
	movss	%xmm0, -268(%rbp)
	movss	.LC2(%rip), %xmm0
	movss	%xmm0, -264(%rbp)
	movss	.LC3(%rip), %xmm0
	movss	%xmm0, -260(%rbp)
	leaq	-272(%rbp), %rax
	movq	%rax, -16(%rbp)
.......
```

然后我们来稍微解释一下这个代码。这里的`stdx::simd<float, stdx::simd_abi::native<float>>`的模板参数有两个部分，一个是`float`，一个是`abi`可能与架构有关，基本上还是使用`native`比较多。但是也有其它的选项，根据当前的文档信息，有六种可选：

* `simd_abi::scalar`
* `simd_abi::native`
* `simd_abi::fixed_size`
* `simd_abi::compatible`
* `simd_abi::deduce`
* `simd_abi::max_fixed_size`

另外，`stdx::simd<float, stdx::simd_abi::native<float>>`的写法有些冗余，这个写法实际上有一个别名：`stdx::native_simd<float>`，下面的代码都使用这种写法。

-----

## Computing with simd

可以与标量或者另一个simd做普通运算，如：

```cpp
int main() {
  std::array<float, 4> a = {1, 2, 3, 4};
  stdx::native_simd<float> v;
  v.copy_from(a.data(), stdx::vector_aligned);

  v += 3;
  v *= 2;

  std::array<float, 4> b = {-1, -2, -3, -4};
  stdx::native_simd<float> w;
  w.copy_from(b.data(), stdx::vector_aligned);

  v += w;

  for (std::size_t i{}; i < 4; ++i)
    std::cout << v[i] << ' ';
  return 0;
}
```

比较关键的是，也支持mask运算，但这里需要使用where来支持，这个`where`​猜想在以后可能会成为一个关键字：

例如下面的代码，将大于等于2的数加1。

```cpp
  stdx::where(v >= 2, v) += 1;
```

当然，也支持使用一个显式的`mask_simd`​

```cpp
  stdx::native_simd_mask<float> mask;
  mask = v > 2.0f;

  stdx::where(mask, v) += 1.0f;
```

需要注意的是`stdx::native_simd_mask<float>`​，这里的模板参数确实是`float`​，如果使用`bool`​会报错，但是这里`simd_mask`​内部的数据类型的确是`bool`​类型，猜想可能是因为当前的实现还不够全面。

这个simd_mask也可以从内存中载入，没有问题：

```cpp
  std::array<bool, 4> b = {true, false, true, false};
  stdx::native_simd_mask<float> mask;
  mask.copy_from(b.data(), stdx::vector_aligned);
```

最后，尽管`simd`​数据结构可以直接索引打印，还是推荐使用`copy_to`​写回到某个稳定的数据结构中：

```cpp
v.copy_to(a.data(), stdx::vector_aligned);
```

## 推荐阅读

1. 《Extreme C》- Chapter 3 有关ABI的部分
2. 《A Tour of C++》，提到了有关valarray和vector的旧事
3. https://cppreference.com/ 有更全面的simd的文档，尽管因为是实验特性的原因，暂时还不够全面。
