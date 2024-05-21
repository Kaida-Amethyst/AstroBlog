---
title: C++编译期归并排序
description: C++中利用模板元编程在编译期实现归并排序
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Javascript/8.jpg
category: 编程实践
published: 2024-05-12
tags: [C++, Meta Programming]
---
中国的互联网上曾经有一个网络段子，说某个刚毕业的学生在简历上写“精通C++”，于是被某一家公司邀请面试，在面试过程中，面试官不断地向这个学生提出了一系列高难的题目，让这个不知天高地厚的学生非常尴尬。这里面其中的一道题就是用C++实现一个编译期快速排序。有关C++如何实现编译期快速排序的内容，在本博客网站里面已经有了，本篇博客要实现的另一种算法，归并排序。

## Algorithm

归并排序是一种经典的分治算法。它将一个数组分成两半，分别对这两部分进行排序，然后将它们合并在一起。这个过程是递归的，每一层递归都会将数组分割成更小的片段，直到这些片段的大小为1，即它们已经排好序了。

在Haskell中，归并排序可以被表达为：

```haskell
mergesort [] = []
mergesort [x] = [x]
mergesort xs = merge (mergesort ys) (mergesort zs)
    where (ys, zs) = splitAt (length xs `div` 2) xs
```

这种递归的自然形式在C++模板元编程中同样适用，尽管实现方式会因语言特性而有所不同。

## Implementation

### 数据结构

首先，我们定义一个基于模板的列表来表示序列：

```cpp
template<int ... N>
struct NumberList;
```

### 辅助操作

为了实现归并排序，我们需要实现一些基本操作。

首先是获取列表的长度`Length`：

```cpp
template <typename L>
struct LengthT;

template <int N, int ... Ns>
struct LengthT<NumberList<N, Ns...>> {
    static constexpr int value = 1 + LengthT<NumberList<Ns...>>::value;
};

template <>
struct LengthT<NumberList<>> {
    static constexpr int value = 0;
};

template <typename L>
static constexpr int Length = LengthT<L>::value;
```

接着是获取列表的首元素`Front`。

```cpp
// get Front, pop Front, push Front, push Back
template <typename L>
struct FrontT;

template <int N, int ... Ns>
struct FrontT<NumberList<N, Ns...>> {
  static constexpr int value = N;
};

template <typename L>
static constexpr int Front = FrontT<L>::value;
```

### 分割列表

为了进行归并排序，需要能够将列表分割为两部分。这里首先实现`FrontNT`，代表的是列表中的前N项。

```cpp
template <typename L, int Count>
struct FrontNT {
  using type = PushFront<Front<L>, typename FrontNT<PopFront<L>, Count - 1>::type>;
};

template <typename L>
struct FrontNT<L, 0> {
  using type = NumberList<>;
};

template <typename L, int Count>
using FrontN = typename FrontNT<L, Count>::type;
```

然后再实现一个`PopFrontN`，代表的是去掉前N项之后剩下的项。

```cpp
template <typename L, int Count>
struct PopFrontNT {
  using type = typename PopFrontNT<PopFront<L>, Count - 1>::type;
};

template <typename L>
struct PopFrontNT<L, 0> {
  using type = L;
};

template <typename L, int Count>
using PopFrontN = typename PopFrontNT<L, Count>::type;
```

### 合并操作

合并两个排序列表是归并排序的核心。我们定义一个模板，用于将两个已排序的列表合并为一个，注意这里有一个模板参数`Comp`用来定义两个数的序关系。

```cpp
template <typename L1, typename L2, template<int, int> class Comp>
struct MergeT;

template <int N, int ... Ns, int M, int ... Ms, template<int, int> class Comp>
struct MergeT<NumberList<N, Ns...>, NumberList<M, Ms...>, Comp> {
    using type = typename std::conditional_t<
        Comp<N, M>::value, 
        PushFront<N, typename MergeT<NumberList<Ns...>, NumberList<M, Ms...>, Comp>::type>,
        PushFront<M, typename MergeT<NumberList<N, Ns...>, NumberList<Ms...>, Comp>::type>>;
};
```

注意到Merge用了递归，我们必须为递归思考递归终点的问题，也就是说两个列表其中一个为空，或者两个都为空的情况。

```cpp
template <int N, int ... Ns, template<int, int> class Comp>
struct MergeT<NumberList<N, Ns...>, NumberList<>, Comp> {
    using type = NumberList<N, Ns...>;
};

template <int M, int ... Ms, template<int, int> class Comp>
struct MergeT<NumberList<>, NumberList<M, Ms...>, Comp> {
    using type = NumberList<M, Ms...>;
};

template <template<int, int> class Comp>
struct MergeT<NumberList<>, NumberList<>, Comp> {
    using type = NumberList<>;
};

template <typename L1, typename L2, template<int, int> class Comp>
using Merge = typename MergeT<L1, L2, Comp>::type;
```

### 归并排序算法

最后，归并排序算法的实现将使用上述工具和概念来递归地分割和合并列表，从而达到排序的目的：

```cpp
// MergeSort
template <typename L, template<int, int> class Comp>
struct MergeSortT {
  using Result = Merge<
      typename MergeSortT<FrontN<L, Length<L> / 2>, Comp>::Result,
      typename MergeSortT<PopFrontN<L, Length<L> /2>, Comp>::Result,
      Comp>;
};

template <int M, template<int, int> class Comp>
struct MergeSortT<NumberList<M>, Comp> {
  using Result = NumberList<M>;
};

template <template<int, int> class Comp>
struct MergeSortT<NumberList<>, Comp> {
  using Result = NumberList<>;
};

template <typename L, template<int, int> class Comp>
using MergeSort = typename MergeSortT<L, Comp>::Result;
```

这种方式通过递归地分割列表并在每个级别上合并，使得整个列表在编译期就被排序了。

## 完整代码

```cpp
// Implement MergeSort, but using metaprogramming

#include <type_traits>
#include <iostream>

template<int ... N>
struct NumberList;

template <typename L>
struct LengthT;

template <int N, int ... Ns>
struct LengthT<NumberList<N, Ns...>> {
    static constexpr int value = 1 + LengthT<NumberList<Ns...>>::value;
};

template <>
struct LengthT<NumberList<>> {
    static constexpr int value = 0;
};

template <typename L>
static constexpr int Length = LengthT<L>::value;

// get Front, pop Front, push Front, push Back
template <typename L>
struct FrontT;

template <int N, int ... Ns>
struct FrontT<NumberList<N, Ns...>> {
  static constexpr int value = N;
};

template <typename L>
static constexpr int Front = FrontT<L>::value;


template <typename L>
struct PopFrontT;

template <int N, int ... Ns>
struct PopFrontT<NumberList<N, Ns...>> {
    using type = NumberList<Ns...>;
};

template <typename L>
using PopFront = typename PopFrontT<L>::type;

template <int E, typename L>
struct PushFrontT;

template <int E, int ... Ns>
struct PushFrontT<E, NumberList<Ns...>> {
    using type = NumberList<E, Ns...>;
};

template <int E, typename L>
using PushFront = typename PushFrontT<E, L>::type;

template <int E, typename L>
struct PushBackT;

template <int E, int ... Ns>
struct PushBackT<E, NumberList<Ns...>> {
    using type = NumberList<Ns..., E>;
};

template <int E, typename L>
using PushBack = typename PushBackT<E, L>::type;

// Split
template <typename L, int Count>
struct FrontNT {
  using type = PushFront<Front<L>, typename FrontNT<PopFront<L>, Count - 1>::type>;
};

template <typename L>
struct FrontNT<L, 0> {
  using type = NumberList<>;
};

template <typename L, int Count>
using FrontN = typename FrontNT<L, Count>::type;

template <typename L, int Count>
struct PopFrontNT {
  using type = typename PopFrontNT<PopFront<L>, Count - 1>::type;
};

template <typename L>
struct PopFrontNT<L, 0> {
  using type = L;
};

template <typename L, int Count>
using PopFrontN = typename PopFrontNT<L, Count>::type;

// Compare
template <int N, int M>
struct Less {
    static constexpr bool value = N < M;
};

// Merge
template <typename L1, typename L2, template<int, int> class Comp>
struct MergeT;

template <int N, int ... Ns, int M, int ... Ms, template<int, int> class Comp>
struct MergeT<NumberList<N, Ns...>, NumberList<M, Ms...>, Comp> {
    using type = typename std::conditional_t<
        Comp<N, M>::value, 
        PushFront<N, typename MergeT<NumberList<Ns...>, NumberList<M, Ms...>, Comp>::type>,
        PushFront<M, typename MergeT<NumberList<N, Ns...>, NumberList<Ms...>, Comp>::type>>;
};

template <int N, int ... Ns, template<int, int> class Comp>
struct MergeT<NumberList<N, Ns...>, NumberList<>, Comp> {
    using type = NumberList<N, Ns...>;
};

template <int M, int ... Ms, template<int, int> class Comp>
struct MergeT<NumberList<>, NumberList<M, Ms...>, Comp> {
    using type = NumberList<M, Ms...>;
};

template <template<int, int> class Comp>
struct MergeT<NumberList<>, NumberList<>, Comp> {
    using type = NumberList<>;
};

template <typename L1, typename L2, template<int, int> class Comp>
using Merge = typename MergeT<L1, L2, Comp>::type;

// MergeSort
template <typename L, template<int, int> class Comp>
struct MergeSortT {
  using Result = Merge<
      typename MergeSortT<FrontN<L, Length<L> / 2>, Comp>::Result,
      typename MergeSortT<PopFrontN<L, Length<L> /2>, Comp>::Result,
      Comp>;
};

template <int M, template<int, int> class Comp>
struct MergeSortT<NumberList<M>, Comp> {
  using Result = NumberList<M>;
};

template <template<int, int> class Comp>
struct MergeSortT<NumberList<>, Comp> {
  using Result = NumberList<>;
};

template <typename L, template<int, int> class Comp>
using MergeSort = typename MergeSortT<L, Comp>::Result;


template <typename T>
void printArg() {
  std::cout << __PRETTY_FUNCTION__ << std::endl;
}

int main() {
    using List = NumberList<4, 2>;
    using NonSorted = NumberList<4, 2, 3, 1, 8, 9, 5, 0, 7>;
    using Sorted1 = NumberList<1, 3, 7>;
    using Sorted2 = NumberList<2, 8>;
    static_assert(Length<List> == 2,
                  "Length of List is not 8");
    static_assert(Front<List> == 4,
                  "Function `Front` is not working");
    static_assert(std::is_same_v<PopFront<List>,
                  NumberList<2>>,
                  "Function `PopFront` is not working");
    static_assert(std::is_same_v<PushFront<1, List>,
                  NumberList<1, 4, 2>>,
                  "Function `PushFront` is not working");
    static_assert(std::is_same_v<PushBack<3, List>,
                  NumberList<4, 2, 3>>,
                  "Function `PushBack` is not working");
    static_assert(std::is_same_v<FrontN<NonSorted, 4>,
                  NumberList<4, 2, 3, 1>>,
                  "Function `FrontN` is not working");
    static_assert(std::is_same_v<PopFrontN<NonSorted, 4>,
                  NumberList<8, 9, 5, 0, 7>>,
                  "Function `PopFrontN` is not working");
    static_assert(std::is_same_v<
                  Merge<Sorted1, Sorted2, Less>,
                  NumberList<1, 2, 3, 7, 8>>,
                  "Function `Merge` is not working");
    static_assert(std::is_same_v<
                  MergeSort<NonSorted, Less>,
                  NumberList<0, 1, 2, 3, 4, 5, 7, 8, 9>>,
                  "Function `MergeSort` is not working");
    printArg<MergeSort<NonSorted, Less>>();
    return 0;
}
```
