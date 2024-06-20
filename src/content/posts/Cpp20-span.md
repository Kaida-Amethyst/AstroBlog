---
title: C++ - 用span代替指针+数据范围
description: C++20引入了一个`std::span`，允许将指针+数据范围的写法封装成一个span，这种特性对C++编程提供了一定程度的便携性，但请注意它并没有完全解决安全性问题。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Javascript/5.jpg
category: 编程实践
published: 2024-01-29
tags: [C++]
---

-----------------

## Overview

传统的C++ main函数带参数的签名：

```cpp
int main(int argc, char* argv[]);
```

这种签名延续了C的传统，即如果要将一个数据结构传入一个函数中进行处理，那我们需要传入一个数据结构的指针和长度。

但是这种方式显然并不安全，因为它并不会阻止函数进行越界访问或者修改。

到了C++中，有的时候我们也有一个需求，就是修改一个数组当中的一个部分，而并非全部（最典型的就是递归式地原地归并排序），这种情况下我们也只能传入数组的指针和长度，依旧无法阻止函数的越界访问和修改。

C++20提供了`std::span`，它的基本思想就是封装指针和数据范围，同时提供容器的API，使得span可以像一般的容器那样进行操作，并且保证安全性。

## Usage

### main

对于main函数而言，在C++20没有引入span之前，通常我们这样写：

```cpp
#include <span>
#include <iostream>

int main(int argc, char* argv[]) {
	for (int i = 0; i < argc; ++i) {
		// do something for argv[i]
	}
    return 0;
}
```

这种方式有时可能不够安全，C++20有了span之后，为了更加安全地访问数组，同时也更好地利用容器类，可以使用span：

```cpp
#include <span>
#include <iostream>

int main(int argc, char* argv[]) {
    std::span<char*> args(argv, argc);
    for (auto arg : args) {
        // do something for arg
    }
    return 0;
}
```

### normal functions

对于日常的开发，如果需要传入容器的一部分进入到函数，可以使用span。例如，假设我只想累加一个容器的子集的所有元素，那么可以这样写：

```cpp
int span_sum(std::span<int> elements) {
    return std::accumulate(elements.begin(), elements.end(), 0);
}

int main(){
    std::array<int, 10> arr = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::span<int> first_five = std::span(arr).subspan(0, 5);
    int sum_pre_five = span_sum(first_five);
    std::cout << "Sum of first five elements: " << sum_pre_five << std::endl;
    return 0;
}
```

如此一来在设计函数的时候，就无需过多考虑函数参数的问题，从而提升开发效率。

## Dynamic and Static Span

需要注意的是，span有两种，一种是动态的span，可以动态调整大小，类似于vector那样，另一种是静态的span，只有静态的大小，类似array，它们之间的区别就在于是否传入了模板size参数。

```cpp
std::span<int> // Dynamic span
std::span<int, 5>  // Static Span
```

并且，需要注意的是，这两种span是不能相互转换的。例如下面的代码会出现错误：

```cpp
int span_sum(std::span<int, 5> elements) {
    return std::accumulate(elements.begin(), elements.end(), 0);
}

int main(){
    std::array<int, 10> arr = /*...*/;
    std::span<int, 5> first_five = std::span(arr).subspan(0, 5);
    int sum_pre_five = span_sum(first_five);  
    std::cout << "Sum of first five elements: " << sum_pre_five << std::endl;
    return 0;
}
```

虽然静态span和动态span都会有subspan接口，但是返回类型不同。静态span的subspan接口仍然会返回静态span，动态span的subspan接口仍然会返回动态span。

## span cannot stop error

最后需要注意的问题是，span仅仅是提供了一个便捷的编程工具，它本身仍然不具备阻止错误的能力：

```cpp
#include <iostream>
#include <vector>
#include <numeric>

int main() {
  std::vector<int> nums = {1, 2, 3};
  std::span<int, 5> sp (nums);
  std::cout << std::accumulate(sp.begin(), sp.end(), 0) << std::endl;
  return 0;
}
```

上面的代码中，nums只有3个元素，span却掌握5个元素，C++并不会阻止这种情况，导致最后的结果可能有多种。

## API

### first

用于获取span中的前几个元素：

```cpp
std::span<int, 5> sp (arr);
std::span<int, 2> f2 = sp.first<2>();

std::span<int> sp2 (vec);
std::span<int> d2 = sp2.first<2>();
```

### last

用于获取span中后几个元素

```cpp
std::span<int, 5> sp (arr);
std::span<int, 2> f2 = sp.last<2>();

std::span<int> sp2 (vec);
std::span<int> d2 = sp2.last<2>();
```

### subspan

用于获取span中的一个子集

```cpp
std::span<int, 5> sp (arr);
std::span<int, 2> f2 = sp.subspan(1, 2);

std::span<int> sp2 (vec);
std::span<int> d2 = sp2.subspan(3, 4);
```

### at (C++26)

用于获取span中特定位置的元素

```cpp
std::span<int> sp2 (vec);
int e = sp2.at(0);
```

