---
title: C++编译期数组
description: C++中的数组是运行期的，但是我们可以通过模板元编程来实现编译期数组
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/3.jpg
category: Programming
published: 2023-04-02
tags: [C++]
---


C++中经常需要使用数组，无论是使用原生数组，或者STL提供的数据结构。它们都属于运行时数组。

不过，有的时候我们可能会遇到这样的场景，我们可能不需要修改数组里面元素的值，那这样的话，如果我们使用运行时数组，有可能会有一定的浪费。

我们可以使用编译期数组来节省空间。

## The Simplest Way

---

最简单的方法，是使用下面的形式，直接用一个模板`struct`，然后将索引特化：

```cpp
template <int idx> struct CompileTimeArray ;
template <> struct CompileTimeArray<0> { static const int value = 0x123; };
template <> struct CompileTimeArray<1> { static const int value = 0x456; };
template <> struct CompileTimeArray<2> { static const int value = 0x789; };
template <> struct CompileTimeArray<3> { static const int value = 0x987; };
template <> struct CompileTimeArray<4> { static const int value = 0x654; };
```

在C++17之后，当然也可以直接使用模板变量，但是无论是哪一种，都会有一个问题，就是它比较冗长，需要写的冗余代码比较多。如果我们的开发需要进行一定的修改的话，可能会造成一定的麻烦。

我们来尝试使用模板编程来解决这个问题。

## 模板元编程

---

我们期望的是达到如下的效果：

```cpp
using MyArray = CompileTimeArray<int, 0, 1, 2, 3, 4>;
std::cout << NthElement<MyArray, 3>  << std::endl;  // 输出3
```

首先我们来实现`CompileTimeArray`的定义：

```cpp
template <typename T, T... N>
struct CompileTimeArray;
```

接下来的难点是，我们怎么去获取里面的元素，也就是`NthElement`的实现。

我们首先把`CompileTimeArray`分割成第一个元素，和第一个元素以外剩下的元素，也就是`Front`和`PopFront`。

```cpp
template<typename List> struct Front;
template<typename List> struct PopFront;
```

接着，去特化这两个，让这两个去接收`CompileTimeArray`：

```cpp
template<typename T, T Head, T... Tail>
struct Front<CompileTimeArray<T, Head, Tail...>> {
  static const T value = Head;
};

template<typename T, T Head, T... Tail>
struct PopFront<CompileTimeArray<T, Head, Tail...>> {
  using type = CompileTimeArray<T, Tail...>;
};
```

实现好这两个之后，我们就可以反应过来。要得到`CompileTimeArray`的第3个元素，相当于得到对`CompileTimeArry`进行`PopFront`之后，再求其第2个元素，然后再`PopFront`，再求它的第1个元素，也就是`Front`元素了。

最后我们来实现`NthElement`：

```cpp
template<typename List, int idx>
struct NthElement : 
  public NthElement<typename PopFront<List>::type, idx - 1> {};
```

上面是递归的查找，最后补全一下递归终点，不过需要注意一下这里需要额外添加一个对`CompileTimeArray`里面元素类型的推导。

```cpp
template<typename List> struct ElementTypeOfList;
template<typename T, T Head, T ...Tail>
struct ElementTypeOfList<CompileTimeArray<T, Head, Tail...>> {
  using type = T;
};
```

最后来完成`NthElement`的递归终点。

```cpp
template<typename List>
struct NthElement<List, 0>  {
  using ElementType = typename ElementTypeOfList<List>::type;
  static const ElementType value = Front<List>::value;
};
```

## 完整代码

---

```cpp
template <typename T, T... N>
struct CompileTimeArray;

template<typename List> struct ElementTypeOfList;
template<typename List> struct Front;
template<typename List> struct PopFront;

template<typename T, T Head, T ...Tail>
struct ElementTypeOfList<CompileTimeArray<T, Head, Tail...>> {
  using type = T;
};

template<typename T, T Head, T... Tail>
struct Front<CompileTimeArray<T, Head, Tail...>> {
  static const T value = Head;
};

template<typename T, T Head, T... Tail>
struct PopFront<CompileTimeArray<T, Head, Tail...>> {
  using type = CompileTimeArray<T, Tail...>;
};

template<typename List, int idx>
struct NthElement : 
  public NthElement<typename PopFront<List>::type, idx - 1> {};

template<typename List>
struct NthElement<List, 0>  {
  using ElementType = typename ElementTypeOfList<List>::type;
  static const ElementType value = Front<List>::value;
};
```

## 测试代码

---

```cpp
int main() {
  using MyArray = CompileTimeArray<uint32_t, 123, 456, 789, 135, 246, 379>;
  std::cout << NthElement<MyArray, 0>::value << std::endl;
  std::cout << NthElement<MyArray, 1>::value << std::endl;
  std::cout << NthElement<MyArray, 2>::value << std::endl;
  std::cout << NthElement<MyArray, 3>::value << std::endl;
  std::cout << NthElement<MyArray, 4>::value << std::endl;
  std::cout << NthElement<MyArray, 5>::value << std::endl;
}
```
