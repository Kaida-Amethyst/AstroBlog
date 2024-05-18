---
title: "现代C++实践技巧 - 打印枚举变量名"
published: 2023-03-08
excerpt: "使用硬编码来打印变量名可能会非常麻烦，这里有一个比较方便的方法。"
img: "https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/7.jpg"
category: 编程实践
tags: [C++]
draft: false
---
------

## Motivation

假如我们有一个枚举：

```cpp
enum Weekdays {
  Monday, Tuesday, Wednesday, Thursday,
  Friday, Saturday, Sunday
};
```

我们有时可能有需求将这个枚举的名称进行打印，在终端中输出这些枚举的变量名。一般来说，我们可能会这么做，使用switch：

```cpp
// Use Switch
Weekdays day;
switch (day) {
case Monday:
  std::cout << "Monday" << std::endl;
  break;
case Tuesday:
  std::cout << "Tuesday" << std::endl;
  break;
case Wednesday:
  std::cout << "Wednesday" << std::endl;
  break;
case Thursday:
  std::cout << "Thursday" << std::endl;
  break;
case Friday:
  std::cout << "Friday" << std::endl;
  break;
case Saturday:
  std::cout << "Saturday" << std::endl;
  break;
case Sunday:
  std::cout << "Sunday" << std::endl;
  break;
}
```

或者使用`Map`或者`Array`进行映射。

```cpp
std::map<Weekdays, std::string> weekday_map;
weekday_map.insert(std::make_pair(Monday, "Monday"));
weekday_map.insert(std::make_pair(Tuesday, "Tuesday"));
weekday_map.insert(std::make_pair(Wednesday, "Wednesday"));
weekday_map.insert(std::make_pair(Thursday, "Thursday"));
weekday_map.insert(std::make_pair(Friday, "Friday"));
weekday_map.insert(std::make_pair(Saturday, "Saturday"));
weekday_map.insert(std::make_pair(Sunday, "Sunday"));
```

但是无论是哪种方式，代码都可能比较冗长。特别是枚举的变量名比较多时，代码会非常复杂。这里介绍一个更简便通用的办法。

### `__PRETTY_FUNCTION__`宏

宏`__PRETTY_FUNCTION__`可以将其所在的函数的名称打印出来，注意C++中的一个函数的名称，不仅仅是函数名，还包括了它的参数类型以及返回类型。看下面的函数：

```cpp
template <Weekdays day> constexpr const char *singleEnum() {
  return __PRETTY_FUNCTION__;
}
```

在main函数中打印`singleEnum<Monday>()`，会得到如下的字符串：

```plaintext
constexpr const char* singleEnum() [with Weekdays day = Monday]
```

注意这里就出现了`Monday`的字符子串，我们就可以将其提取出来，我们另外写一个函数：

```cpp
template <Weekdays day> std::string getEnumName() {
  std::string str = singleEnum<day>();
  std::size_t equal_index = str.find("=");
  std::size_t len = str.find("]") - equal_index - 2;
  return str.substr(equal_index + 2, len);
}
```

如果运行`getEnumName<Monday>()`就可以得到`Monday`的`string`。

### make_integer_sequence

知道了如何生成单个枚举变量名，接下来的问题就是如何按序列举枚举名。C++中的`make_integer_sequance`可以帮助做到这一点，`std::make_integer_sequence<int, 7>`可以生成一个`integer_sequence<int, 1, 2, 3, 4, 5, 6, 7>`，而后面的`1,2,3,4,5,6,7`可以直接使用变参模板参数来代替，这样可以使用C++17的((20230425144233-u621kyf "Fold Expression"))技巧。

例如，如果我们仅仅是希望把所有枚举变量名打印出来，可以这么做：

```cpp
template <int... Values>
void outputEnum(const std::integer_sequence<int, Values...> &) {
  ((std::cout << getEnumName<static_cast<Weekdays>(Values)>() << std::endl),
   ...);
}
```

这样就会打印出：

```plaintext
Monday
Tuesday
Wednesday
Thursday
Friday
Saturday
Sunday
```

或者，我们也可以将其加入到`map`或者`vector`中：

```cpp
template <int... Values>
void insert_to_vector(std::vector<std::string> &vec,
                      const std::integer_sequence<int, Values...> &) {
  ((vec.push_back(getEnumName<static_cast<Weekdays>(Values)>())), ...);
}

template <int... Values>
void insert_to_map(std::map<Weekdays, std::string> &map,
                   const std::integer_sequence<int, Values...> &) {
  ((map.insert(std::make_pair(static_cast<Weekdays>(Values),
                              getEnumName<static_cast<Weekdays>(Values)>()))),
   ...);
}
```

### Full Implementation

```cpp
// It can be used for any enum
// Just need to change the enum name 
// From `Weekdays` to your enum
template <Weekdays day> constexpr const char *singleEnum() {
  return __PRETTY_FUNCTION__;
}

template <Weekdays day> std::string getEnumName() {
  std::string str = singleEnum<day>();
  std::size_t equal_index = str.find("=");
  std::size_t len = str.find("]") - equal_index - 2;
  return str.substr(equal_index + 2, len);
}

template <int... Values>
void outputEnum(const std::integer_sequence<int, Values...> &) {
  ((std::cout << getEnumName<static_cast<Weekdays>(Values)>() << std::endl),
   ...);
}

template <int... Values>
void insert_to_vector(std::vector<std::string> &vec,
                      const std::integer_sequence<int, Values...> &) {
  ((vec.push_back(getEnumName<static_cast<Weekdays>(Values)>())), ...);
}

template <int... Values>
void insert_to_map(std::map<Weekdays, std::string> &map,
                   const std::integer_sequence<int, Values...> &) {
  ((map.insert(std::make_pair(static_cast<Weekdays>(Values),
                              getEnumName<static_cast<Weekdays>(Values)>()))),
   ...);
}
```

## 推荐阅读

[1]: 《C++ Templete The Complete Guide》 Chapter 4 - 4.2 Fold Expressions
