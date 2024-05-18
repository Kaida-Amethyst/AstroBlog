---
title: C++编译期快速排序
description: 实现一个快速排序，不过要在编译期完成
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Javascript/2.jpg
category: 编程实践
published: 2024-05-09
tags: [C++, Meta Programming]
---

-------

著名的网络段子，说某个人精通C++，之后去面试遭到市场教育，其中的一道题就是用C++实现一个编译期排序。这篇博客就来挑战一下这项任务。

## Algorithm

快速排序跟很多其它的排序算法不太一样的是，快速排序很好用数学形式表达，这一点看Haskell代码就可以看出来。

```haskell
quicksort [] = []
quicksort (x:xs) = 
     quicksort [y | y <- xs, y <= x] ++ [x] ++ quicksort [y | y <- xs, y > x]
```

它的思想就是，把首元素拿出，再把剩下的元素分成小于这个元素的，和大于这个元素的，递归调用快速排序之后，再合并即可。

而C++的模板元编程，它基本上是函数式编程的范式，所以C++编译期快速排序，实际上要比实现其它的排序算法，相对来说要稍微简单一点。（当然，也没简单到哪里去）

## Implementation

### 列表及其算法

首先是一个列表，这个比较简单，为了方便，这里的数据类型都使用`int`，在看懂整篇博客的前提下，拓展一个数据类型的模板参数并非难事。

```cpp
template <int ... N>
struct NumberList;
```

接下来是求一个列表的首元素：

```cpp
// Get Front Element
template <typename List>
struct FrontElementT;

template <int Head, int ... Tail>
struct FrontElementT<NumberList<Head, Tail...>> {
    static const int value = Head;
};

template <typename List>
static const int FrontElement = FrontElementT<List>::value;
```

注意这里我们没有考虑空列表的情况，所以如果列表为空，是直接编译报错的。

然后是求一个列表里，除首元素以外的剩下元素`PopFront`:

```cpp
// Pop Front
template <typename List>
struct PopFrontT;

template <int Head, int ... Tail>
struct PopFrontT<NumberList<Head, Tail...>> {
    using type = NumberList<Tail...>;
};

template <typename List>
using PopFront = typename PopFrontT<List>::type;
```

注意这里也没有考虑空列表的情况，主要是因为不需要。

对于快速排序而言，最重要的是`Partition`，我们可以试着想一下，这中间应该需要不断地往一个列表里面加元素，所以这里我们添加一个`PushFront`，用来向一个列表的头部添加元素：

```cpp
// Push Front
template <int Element, typename List>
struct PushFrontT;

template <int Element, int ... N>
struct PushFrontT<Element, NumberList<N...>> {
    using type = NumberList<Element, N...>;
};

template <int Element, typename List>
using PushFront = typename PushFrontT<Element, List>::type;
```

> 注：也可以依葫芦画瓢实现一个PushBack，无非是改变一下上面NumberList的N...和Element的顺序。但实际上，这篇博客里面实现的QuickSort没有用到PushBack，所以就不实现了。


### Partition

列表的添加与删除实现完毕之后，紧接着就是QuickSort的关键一步，就是Partition。这一步的思想是，我需要把一个列表，对比一个Pivot元素分成两拨，一波是小于Pivot的，另一波是大于等于Pivot的。

注意一般来说，这种比较运算我们可能会倾向于提取出来，所以我们首先把序关系单独作为一个功能：

```cpp
template<int L, int R>
struct Less {
    static constexpr bool value = L < R;
};
```

然后我们来实现`Partition`，分割会分割成两个部分，首先演示的是满足序关系的元素集（或者说小于Pivot的）：

```cpp
// Partition Left
template <int Pivot, typename List, template <int, int> typename Compare>
struct PartitionLeftT;

template <int Pivot, int Head, int ... Tail, template<int, int> typename Compare>
struct PartitionLeftT<Pivot, NumberList<Head, Tail...>, Compare> {
    using type = typename std::conditional_t<Compare<Head, Pivot>::value,
                                             PushFront<Head, typename PartitionLeftT<Pivot, NumberList<Tail...>, Compare>::type>,
                                             typename PartitionLeftT<Pivot, NumberList<Tail...>, Compare>::type>;
};

template <int Pivot, template<int, int> typename Compare>
struct PartitionLeftT<Pivot, NumberList<>, Compare> {
    using type = NumberList<>;
};

template <int Pivot, typename List, template<int, int> typename Compare>
using PartitionLeft = typename PartitionLeftT<Pivot, List, Compare>::type;
```

重点是中间的`std::conditional_t`的部分，当满足序关系时，进行PushFront，注意这里一定是PushFront而不是PushBack，因为这里是递归，新列表的生成是首先添加后面的元素，再添加前面的元素。

弄明白`PartitionLest`之后，剩下的`PartitionRight`就很好动了，只需要改变`Compare`的符号即可。

```cpp
// Partition Right
template <int Pivot, typename List, template<int, int> typename Compare>
struct PartitionRightT;

template <int Pivot, int Head, int ... Tail, template<int, int> typename Compare>
struct PartitionRightT<Pivot, NumberList<Head, Tail...>, Compare> {
    using type = typename std::conditional_t<!Compare<Head, Pivot>::value,
                                             PushFront<Head, typename PartitionRightT<Pivot, NumberList<Tail...>, Compare>::type>,
                                             typename PartitionRightT<Pivot, NumberList<Tail...>, Compare>::type>;
};

template <int Pivot, template<int, int> typename Compare>
struct PartitionRightT<Pivot, NumberList<>, Compare> {
    using type = NumberList<>;
};

template <int Pivot, typename List, template<int, int> typename Compare>
using PartitionRight = typename PartitionRightT<Pivot, List, Compare>::type;
```

按照QuickSort的算法，接下来就是分别对两个分割出来的子集调用递归进行快速排序，然后再合并，这里先把合并的代码写上，非常简单：

```cpp
// Concatenate
template <typename LList, int M, typename RList>
struct ConcatenateT;

template <int ... L, int M, int ... R>
struct ConcatenateT<NumberList<L...>, M, NumberList<R...>> {
    using type = NumberList<L..., M, R...>;
};

template <typename LList, int M, typename RList>
using Concatenate = typename ConcatenateT<LList, M, RList>::type;
```

## Quick Sort

最后，就是快速排序的部分了，核心就是下面的代码。

```cpp
// Quick sort
template <typename List, template<int, int> typename Compare>
struct QuickSortT {
  using Result = Concatenate<typename QuickSortT<PartitionLeft<FrontElement<List>, PopFront<List>, Compare>, Compare>::Result,
                             FrontElement<List>,
                             typename QuickSortT<PartitionRight<FrontElement<List>, PopFront<List>, Compare>, Compare>::Result>;
};

```

对`PartitionLeft`和`PartitionRight`各调用一次QuickSort，再合并起来。

最后进行递归终点的处理，以及做一次类型规整。

```cpp

template <template<int, int> typename Compare>
struct QuickSortT<NumberList<>, Compare> {
  using Result = NumberList<>;
};

template <int N, template<int, int> typename Compare>
struct QuickSortT<NumberList<N>, Compare> {
  using Result = NumberList<N>;
};

template <typename List, template<int, int> typename Compare>
using QuickSort = typename QuickSortT<List, Compare>::Result;
```

这样就完成了整个的编译期快速排序。

## 完整代码

```cpp
// Implement quick sort, but with meta programming.
// For better understanding, using Haskell code rather than runtime C code to represent the algorithm.
// Haskell code for quick sort:
// quicksort [] = []
// quicksort (x:xs) = 
//      quicksort [y | y <- xs, y <= x] ++ [x] ++ quicksort [y | y <- xs, y > x]

#include <iostream>

template <int ... N>
struct NumberList;

// Get Front Element
template <typename List>
struct FrontElementT;

template <int Head, int ... Tail>
struct FrontElementT<NumberList<Head, Tail...>> {
    static const int value = Head;
};

template <typename List>
static const int FrontElement = FrontElementT<List>::value;

// Push Front
template <int Element, typename List>
struct PushFrontT;

template <int Element, int ... N>
struct PushFrontT<Element, NumberList<N...>> {
    using type = NumberList<Element, N...>;
};

template <int Element, typename List>
using PushFront = typename PushFrontT<Element, List>::type;

// Pop Front
template <typename List>
struct PopFrontT;

template <int Head, int ... Tail>
struct PopFrontT<NumberList<Head, Tail...>> {
    using type = NumberList<Tail...>;
};

template <typename List>
using PopFront = typename PopFrontT<List>::type;

// Compare
template<int L, int R>
struct Less {
    static constexpr bool value = L < R;
};

// Partition Left
template <int Pivot, typename List, template <int, int> typename Compare>
struct PartitionLeftT;

template <int Pivot, int Head, int ... Tail, template<int, int> typename Compare>
struct PartitionLeftT<Pivot, NumberList<Head, Tail...>, Compare> {
    using type = typename std::conditional_t<Compare<Head, Pivot>::value,
                                             PushFront<Head, typename PartitionLeftT<Pivot, NumberList<Tail...>, Compare>::type>,
                                             typename PartitionLeftT<Pivot, NumberList<Tail...>, Compare>::type>;
};

template <int Pivot, template<int, int> typename Compare>
struct PartitionLeftT<Pivot, NumberList<>, Compare> {
    using type = NumberList<>;
};

template <int Pivot, typename List, template<int, int> typename Compare>
using PartitionLeft = typename PartitionLeftT<Pivot, List, Compare>::type;

// Partition Right
template <int Pivot, typename List, template<int, int> typename Compare>
struct PartitionRightT;

template <int Pivot, int Head, int ... Tail, template<int, int> typename Compare>
struct PartitionRightT<Pivot, NumberList<Head, Tail...>, Compare> {
    using type = typename std::conditional_t<!Compare<Head, Pivot>::value,
                                             PushFront<Head, typename PartitionRightT<Pivot, NumberList<Tail...>, Compare>::type>,
                                             typename PartitionRightT<Pivot, NumberList<Tail...>, Compare>::type>;
};

template <int Pivot, template<int, int> typename Compare>
struct PartitionRightT<Pivot, NumberList<>, Compare> {
    using type = NumberList<>;
};

template <int Pivot, typename List, template<int, int> typename Compare>
using PartitionRight = typename PartitionRightT<Pivot, List, Compare>::type;

// Concatenate
template <typename LList, int M, typename RList>
struct ConcatenateT;

template <int ... L, int M, int ... R>
struct ConcatenateT<NumberList<L...>, M, NumberList<R...>> {
    using type = NumberList<L..., M, R...>;
};

template <typename LList, int M, typename RList>
using Concatenate = typename ConcatenateT<LList, M, RList>::type;

// Quick sort
template <typename List, template<int, int> typename Compare>
struct QuickSortT {
  using Result = Concatenate<typename QuickSortT<PartitionLeft<FrontElement<List>, PopFront<List>, Compare>, Compare>::Result,
                             FrontElement<List>,
                             typename QuickSortT<PartitionRight<FrontElement<List>, PopFront<List>, Compare>, Compare>::Result>;
};

template <template<int, int> typename Compare>
struct QuickSortT<NumberList<>, Compare> {
  using Result = NumberList<>;
};

template <int N, template<int, int> typename Compare>
struct QuickSortT<NumberList<N>, Compare> {
  using Result = NumberList<N>;
};

template <typename List, template<int, int> typename Compare>
using QuickSort = typename QuickSortT<List, Compare>::Result;


template<typename T>
void printArg() {
  std::cout << __PRETTY_FUNCTION__ << std::endl;
}

int main() {
    using List = NumberList<1,2,6,3, 8>;
    printArg<PopFront<List>>();
    printArg<PartitionLeft<5, List, Less>>();
    printArg<PartitionRight<5, List, Less>>();
    printArg<ConcatenateT<NumberList<1,2,3>, 0, NumberList<4,5,6>>::type>();
    printArg<PartitionLeft<FrontElement<List>, PopFront<List>, Less>>();
    printArg<QuickSort<List, Less>>();
    return 0;
}

```
