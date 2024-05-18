---
title: 现代C++模板 - contain all none any
description: 用模板实现不定参数的contain all none和any
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/6.jpg
category: 编程实践
published: 2022-02-27
tags: [C++, Meta Programming]
---

-----

## Problem

实现`contain_all`，`contain_any`和`contain_none`函数，效果如下：

```cpp
std::vector<int> nums {1,2,3,4,5,6};

std::cout << contain_all(nums, 3, 4) << std::endl;     // true
std::cout << contain_all(nums, 3, 4, 7) << std::endl;  // false
std::cout << contain_any(nums, 7) << std::endl;        // true
std::cout << contain_any(nums, 0, 11, 7, 21) << std::endl;  // false
std::cout << contain_none(nums, 0, 11, 7, 21) << std::endl;  // false
```

但是注意，需要使用模板来实现。

---

## Implementation

利用"Variadic Parameters"以及"Fold Expression"

```cpp
template<typename Container, typename T>
bool contains(const Container& C, T&& v) {
  return C.end() != std::find(C.begin(), C.end(), v);
}

template<typename Container, typename... Values>
bool contain_any(const Container& C, Values &&... values) {
  return (... || contains(C, values));
}

template<typename Container, typename... Values>
bool contain_all(const Container& C, Values &&... values) {
  return (... && contains(C, values));
}

template<typename Container, typename... Values>
bool contain_none(const Container& C, Values &&... values) {
  return !(... || contains(C, values));
}
```

另外，`contain_none`也可以利用`contain_any`来实现，但要注意我们的参数使用了转发引用，因此需要forward。

```cpp
template<typename Container, typename... Values>
bool contain_none(const Container& C, Values&&... values) {
  return !contain_any(C, std::forward<Values>(values)...)
}
```
