---
title: C++ ranges - Quick Start
description: 从C++20开始引入了ranges，允许更加灵活，更加安全可靠的编程方式。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Javascript/3.jpg
category: 编程实践
published: 2024-03-01
tags: [C++]
---

----------------

## A Problematic Code

看下面的代码：

```cpp
#include <vector>
#include <iostream>
#include <iterator>

std::vector<int> getVector() {
  std::vector<int> v {1,2,3};
  return v;
}

int main() {
  std::copy(getVector().begin(), getVector().end(), std::ostream_iterator<int>(std::cout, " "));
  return 0;
}

```

上面的代码，很显然是有问题的，因为`getVector`这个函数会创造一个临时的`vector`，这里调用了两次`getVector`，也就意味着出现两个临时的`vector`对象，那么前后两次使用迭代器`begin()`和`end()`，就并不是同一个对象的起点和终点。于是，这个代码就很容易出现未定义的行为。

本质上来说，这里的`copy`，跟下面的代码差不多：

```cpp
std::copy(a.begin(), b.end(), /*...*/)
```

但是麻烦的是，这样的代码却是可以编译通过的，无论是`g++`编译器还是`clang++`编译器，都没有提出错误或者警告信息。我们只有在运行的时候，才知道这里出现了问题。

而STL大量地使用这种范式，无论是这里copy或者transform，都需要接收两个迭代器参数，这样的情况下，编译器有时很难检测这两个迭代器是否来自于同一个对象，这样为整个程序实际上都带来了隐患。

因此从C++20开始，C++引入ranges，来避免使用迭代器模式，而是使用类似于函数式编程的范式来处理日常的工作。

## overview

日常我们可能会需要这样的任务，给定一个`vector`，我们需要每一个元素筛选出符合条件的元素，接着应用某一个函数再返回出一个新的`vector`。古老的做法可能是使用`for`循环，C++11之后使用`STL`可能会是使用接口`filter`和`transform`，对于C++20引入的ranges，我们可以这么做：

```cpp
std::vector<int> v;
auto view = std::views::all(v) 
		| std::views::filter([](int n) { /*...*/ })
		| std::views::transform([](int n){/*...*/});
		| std::ranges::to<std::vector<int>>();
		;
```

就像这样，ranges允许你使用这种管道和运算的组合来完成各种操作。这种函数式的写法比先前的for循环或者STL的写法要更加便捷和优雅，也因为使用这种函数式编程，它也更容易在底层进行优化。

> 注意：上面的代码中，可以去掉`std::views::all(v)`而直接使用`v`，这是C++标准库为STL数据结构提供的一个便利。

下面介绍一些常用的ranges API。

## 1. 通用ranges API

### 1.1 std::views::iota

`std::views::iota`生成一个从指定值开始的连续整数序列。例如，生成一个从0到9的整数序列：

```cpp
#include <iostream>
#include <ranges>
#include <iterator>

int main() {
    auto vec = std::views::iota(0, 10);
    std::copy(vec.begin(), vec.end(), std::ostream_iterator<float>(std::cout, " "));
    std::cout << '\n';
    // 0 1 2 3 4 5 6 7 8 9
    return 0;
}
```

### 1.2 std::views::take

`std::views::take`用于从序列的开始处获取指定数量的元素。例如，从上面的整数序列中取前5个数字：

```cpp
#include <iostream>
#include <ranges>
#include <iterator>

int main() {
    auto vec = std::views::iota(0, 10) | std::views::take(5);
    std::copy(vec.begin(), vec.end(), std::ostream_iterator<float>(std::cout, " "));
    std::cout << '\n';
    // 0 1 2 3 4
    return 0;
}
```

### 1.3 std::views::drop

`std::views::drop`用于从序列的开始处跳过指定数量的元素。例如，跳过前3个元素：

```cpp
#include <iostream>
#include <ranges>
#include <iterator>

int main() {
    auto vec = std::views::iota(0, 10) | std::views::drop(3);
    std::copy(vec.begin(), vec.end(), std::ostream_iterator<float>(std::cout, " "));
    std::cout << '\n';
    // 3 4 5 6 7 8 9
    return 0;
}
```

### 1.4 std::views::transform

`std::views::transform`应用一个函数到序列中的每个元素上。例如，将每个元素乘以2：

```cpp
#include <iostream>
#include <ranges>
#include <iterator>

int main() {
    auto vec = std::views::iota(0, 10)
             | std::views::transform([](int n) { return n * 2; });
    std::copy(vec.begin(), vec.end(), std::ostream_iterator<float>(std::cout, " "));
    std::cout << '\n';
    // 0 2 4 6 8 10 12 14 16 18
    return 0;
}
```

### 1.5 std::views::filter

`std::views::filter`根据给定的谓词函数过滤序列。例如，筛选出偶数：

```cpp
#include <iostream>
#include <ranges>
#include <iterator>

int main() {
    auto vec = std::views::iota(0, 10)
             | std::views::filter([](int n) { return n % 2 == 0; });
    std::copy(vec.begin(), vec.end(), std::ostream_iterator<float>(std::cout, " "));
    std::cout << '\n';
    // 0 2 4 6 8
    return 0;
}
```

### 1.6 std::views::reverse

`std::views::reverse`反转序列。例如，反转整数序列：

```cpp
#include <iostream>
#include <ranges>
#include <iterator>

int main() {
    auto vec = std::views::iota(0, 10)
             | std::views::reverse;
    std::copy(vec.begin(), vec.end(), std::ostream_iterator<float>(std::cout, " "));
    std::cout << '\n';
    // 9 8 7 6 5 4 3 2 1 0
    return 0;
}
```

### 1.7 std::views::repeat (C++23)

`std::views::repeat`用于创造重复的序列。

```cpp
#include <iostream>
#include <ranges>
#include <iterator>

int main() {
    auto vec = std::views::repeat(1, 10);
    std::copy(vec.begin(), vec.end(), std::ostream_iterator<float>(std::cout, " "));
    std::cout << '\n';
    // 1 1 1 1 1 1 1 1 1 1
    return 0;
}
```

### 1.8 std::views::all

`std::views::all`用于创建一个包含整个序列的视图。例如，简化对整个向量的访问，让向量里面的元素参与ranges运算。

```cpp
#include <iostream>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    auto all = std::views::all(vec);
	std::copy(all.begin(), all.end(), std::ostream_iterator<float>(std::cout, " "));
    std::cout << '\n';
	// 1 2 3 4 5
    return 0;
}
```

### 1.9 std::views::empty

`std::views::empty`用于生成一个空的视图。在一些特殊情况下可能会需要（想一想使用nullptr的场景）。

```cpp
#include <iostream>
#include <ranges>

int main() {
    auto empty_view = std::views::empty<int>;
    return 0;
}
```

### 1.10 std::views::single

`std::views::single`用于创建一个只包含单个元素的视图。例如：

```cpp
#include <iostream>
#include <ranges>

int main() {
    auto single_view = std::views::single(42);
    return 0;
}
```

## 2. 字符串ranges API

split操作和join操作虽然也可以用在别的容器上，但是最一般的情况，应该还是用在字符串操作上。

> 注意：使用string_view而不是string。

### 2.1 std::views::split

`std::views::split`用于根据指定的分隔符将字符串拆分成多个部分。例如，拆分一个字符串：

```cpp
#include <iostream>
#include <ranges>
#include <string_view>

using namespace std::literals;

int main() {
    std::string_view s = "this is a temp file"sv;
    auto words = std::views::split(s, " "sv);
    for (const auto & w: words) {
        std::cout << std::string_view(w) << '\n';
    }
	return 0;
}
```

### 2.2 std::views::join

`std::views::join`用于将多个序列连接成一个序列。例如，连接多个字符串：

```cpp
#include <iostream>
#include <vector>
#include <ranges>
#include <string_view>

using namespace std::literals;

int main() {
    std::vector<std::string_view> words {"Hello"sv, "this"sv, "is"sv, "a"sv, "simple"sv, "exmaple"sv};
    for(auto &w: words|std::views::join) {
        std::cout << w;
    }
    return 0;
}
```

### 2.3 std::views::join_with（C++23）

`std::views::join`用于将多个序列连接成一个序列，并且使用连接符号：

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <ranges>

int main() {
    std::vector<std::string> words = {"Hello", "World"};
    auto joined = words | std::views::join_with(", ");
    for (char c : joined) {
        std::cout << c;
    }
    return 0;
}
```

## 3. 映射ranges API

### 3.1 views::keys_view

`views::keys_view`用于从映射中提取键的视图。例如，从一个`std::map`中提

取键：

```cpp
#include <iostream>
#include <map>
#include <ranges>
#include <iterator>

int main() {
    std::map<std::string, int> m = {{"one", 1}, {"two", 2}, {"three", 3}};
    auto keys = m | std::views::keys;
	std::ranges::copy(keys, std::ostream_iterator<std::string>(std::cout, " "));
    std::cout << '\n';
    // one two three 
    return 0;
}
```

### 3.2 views::values_view

`views::values_view`用于从映射中提取值的视图。例如，从一个`std::map`中提取值：

```cpp
#include <iostream>
#include <map>
#include <ranges>
#include <iterator>

int main() {
    std::map<std::string, int> m = {{"one", 1}, {"two", 2}, {"three", 3}};
    auto values = m | std::views::values;
	std::ranges::copy(value, std::ostream_iterator<int>(std::cout, " "));
    std::cout << '\n';
    // 1 3 2
    return 0;
}
```

## 4. C++23 C++26 ranges API

以下介绍一些在C++23和C++26中才会出现的ranges操作，截止本博客写成之时，仍然有部分API没有实现，它真正的实现和用法可能会与以下的内容不同。未来博客会再次更新。

### 4.1 views::enumerate

`views::enumerate`将序列的元素和其对应的索引配对，生成一个包含索引和元素的视图。例如，枚举一个序列的元素：

```cpp
#include <iostream>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> vec = {10, 20, 30};
    auto enumerated = vec | std::views::enumerate;
    for (auto &[index, value] : enumerated) {
        std::cout << "Index: " << index << ", Value: " << value << "\n";
    }
    // Index: 0, Value: 10
    // Index: 1, Value: 20
    // Index: 2, Value: 30
    return 0;
}
```

### 4.2 views::zip

`views::zip`用于将多个序列的对应元素组合成一个新的序列。例如，将两个序列的元素组合：

```cpp
#include <iostream>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> vec1 = {1, 2, 3};
    std::vector<std::string> vec2 = {"one", "two", "three"};
    auto zipped = std::views::zip(vec1, vec2);
    for (auto &[num, str] : zipped) {
        std::cout << num << ": " << str << "\n";
    }
	// 1: one
	// 2: two
	// 3: three
    return 0;
}
```

### 4.3 views::adjacent

`views::adjacent`用于将序列中相邻的元素组合成一个新的序列。例如，将相邻元素配对：

```cpp
#include <iostream>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    auto adjacent_pairs = vec | std::views::adjacent(2);
    for (auto [a, b] : adjacent_pairs) {
        std::cout << a << ", " << b << "\n";
    }
	// 1, 2
	// 2, 3
	// 3, 4
	// 4, 5
    return 0;
}
```

### 4.4 views::slide

`views::slide`创建一个滑动窗口视图，其中每个窗口包含序列中的连续元素。例如，创建三个元素的滑动窗口：

```cpp
#include <vector>
#include <ranges>
#include <iostream>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6};
    auto sliding_windows = vec | std::views::slide(3);
    for (auto window : sliding_windows) {
        for (int x : window) {
            std::cout << x << " ";
        }
        std::cout << "\n";
    }
    return 0;
}
```

这里，每个输出的窗口包含序列中的三个连续元素，展示了如何通过滑动窗口捕获和处理数据的局部特征。

### 4.5 views::chunk

`views::chunk`将序列分割成指定大小的块。例如，将一个序列分割成每块包含两个元素的块：

```cpp
#include <iostream>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6};
    auto chunks = vec | std::views::chunk(2);
    for (auto chunk : chunks) {
        for (int x : chunk) {
            std::cout << x << " ";
        }
        std::cout << "\n";
    }
    return 0;
}
```

这个例子显示了如何将一个更大的序列分割成更小的、更易于管理的单元，便于进行并行处理或者更细粒度的数据分析。

### 4.6 views::stride

`views::stride`从序列中以指定的步长取元素，生成新的视图。例如，从一个整数序列中每隔一个元素取一个：

```cpp
#include <iostream>
#include <vector>
#include <ranges>

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5, 6};
    auto strided = vec | std::views::stride(2);
    for (int x : strided) {
        std::cout << x << " ";
    }
    return 0;
}
```

在这个例子中，输出为“1 3 5”，展示了如何跳过一定数量的元素来访问序列，这可以用于降低数据密度或者处理数据时跳过不必要的部分。

## Summary

这些例子展示了如何利用C++20和预期中的C++23中的ranges API来简化和增强数据处理的能力，通过提供更声明式、更易读且安全的方法来操作序列。ranges不仅提高了代码的可维护性，还降低了错误发生的概率，是现代C++中处理序列数据的强大工具。
