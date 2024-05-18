---
title: C++编译期插入排序
description: C++中实现插入排序，但是需要是在编译期完成这个动作
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Java/2.jpg
category: 编程实践
published: 2024-03-13
tags: [C++, Meta Programming]
---

-----

曾经有个段子，说某个人在简历上写到“精通C++”，于是这个人就被邀请参加了一轮面试，其中一道面试题是，使用C++模板元编程实现排序。

本篇博客就来挑战一下这项任务，排序的算法有很多种，这一篇博客打算实现的是插入排序，至于其它排序算法，留待以后的博客了。

## 插入排序

大学算法课上教的插入排序，是使用循环，先保证前k项有序，遍历到第k+1项时，把第k+1项插入到前k项里面。

不过如果要使用C++的元编程，这个思路是不行的，因为模板编程没有“循环”这种东西，C++的模板编程是一个近乎纯粹的函数式编程，因此就要想办法使用递归。

这样的话我们的思路就变成了这样，把数组分成第一项和剩下的项，剩下的项递归调用插入排序，接着再把第一项插入到剩下的项中就可以了。

运行时的代码就像这样：

```cpp
void insert_recursive(std::vector<int> &v, int n) {
  if (n == 0) {
    return;
  }

  if (v[n] < v[n - 1]) {
    std::swap(v[n], v[n - 1]);
    insert_recursive(v, n - 1);
  }

  return;
}

void insert_sort(std::vector<int> &v, int n) {
  if (n <= 1) {
    return;
  }

  insert_sort(v, n - 1);
  insert_recursive(v, n - 1);
}

void insert_sort(std::vector<int> &v) { insert_sort(v, v.size()); }
```

理顺这个思路之后，我们就可以来实现元编程版本的插入排序了。

## 编译期插入排序

核心的部分是四个，获取列表中的第一个元素，获取列表中除第一个元素以外剩下的元素，一个元素和一个列表合并起来得到的新的列表，一个选择器。有了这四样，我们就可以来实现我们的插入排序了。

### 编译期列表

通过不定模板参数就可以实现一个编译期列表了，为了方便，这里的数默认都使用int。

```cpp
template <int ... Number>
struct NumberList;
```

### FrontElement

接着，我们来实现`FrontElement`，也就是获取列表中的第一个元素。

```cpp
template <typename List>
struct FrontElementT;

template <int Head, int ... Tail>
struct FrontElementT<NumberList<Head, Tail...>> {
  static const int value = Head;
};
```

这里的思路比较清晰了，首先有一个general的实现，接着特化模板来让FrontElementT只能接收NumberList为参数。

另外注意一下，C++17之后，我们可以创建一个模板变量，像这样：

```cpp
template <typename List>
static const int FrontElement = FrontElementT<List>::value;
```

模板变量可以极大地简化代码的编写，如果没有这个，当元编程开始复杂时，你可能会各种困惑与`::`和`<>`的符号中。

注意这里我们没有考虑`List`为空的情况，实际上`List`为空的时候我们也没法知道应该是什么值，所以一旦遇到这样的情况，编译器会直接报错。而这也意味着在后面的代码中我们需要手动处理`List`为空的情况。

### PopFront

接下来是一个`PopFront`，用来获得一个列表中，除第一个元素外，剩下的元素。

其实如果理解了`FrontElement`，`PopFront`就很容易写出来了。

```cpp
template <typename List>
struct PopFrontT;

template <int Head, int ... Tail>
struct PopFrontT<NumberList<Head, Tail...>> {
    using List = NumberList<Tail...>;
};

template<typename List>
using PopFront = typename PopFrontT<List>::List;
```

还是一样，注意一下这里没有考虑列表为空的情况。

### PushFront

`PushFront`用来将一个数跟一个列表合并起来，那么它自然需要两个参数，一个是`int`类型，用来代表要添加的数，另一个是就是`List`。

```cpp
template <int Element, typename List>
struct PushFrontT;

template <int Element, int ... Tail>
struct PushFrontT<Element, NumberList<Tail...>> {
    using List = NumberList<Element, Tail...>;
};

template <int Element, typename List>
using PushFront = typename PushFrontT<Element, List>::List;
```

### IfThenElse

列表的操作完成后，接下来就是分支选择器，其实这里原本是不需要我们来实现的，因为C++的标准库type_traits，已经提供了一个`std::conditional`，我们这里也基本上是它的实现，只是为了不引入额外的标准库，这里我们实现一下。

原理也很简单，三个模板参数，第一个模板参数Condition，第二和第三个分别命名为Else和Then，当Condition为true时，结果为Then，为false时，结果为Else。

```cpp
template <bool Condition, typename Then, typename Else>
struct IfThenElseT;

template <typename Then, typename Else>
struct IfThenElseT<true, Then, Else> {
    using Result = Then;
};

template <typename Then, typename Else>
struct IfThenElseT<false, Then, Else> {
    using Result = Else;
};

template <bool Condition, typename Then, typename Else>
using IfThenElse = typename IfThenElseT<Condition, Then, Else>::Result;
```

接着实现两个重要判断功能，一个用来判断列表是否为空，另一个用来判断两个元素的序关系，这里为了简化，就仅仅是一个小于关系好了。

```cpp
template <typename List>
struct IsEmptyT {
    static const bool value = false;
};

template<>
struct IsEmptyT<NumberList<>> {
    static const bool value = true;
};

template <typename List>
static const bool IsEmpty = IsEmptyT<List>::value;

template <int L, int R>
struct Less {
    static const bool value = L < R;
};
```

注意一下这里的`Less`我们没有使用模板变量，是因为这里相对来说必要性不大。

## Insert

接下来就是这一轮算法里面的核心，就是将一个数插入到一个有序的列表里面，注意是**有序的列表**。

它的思路是这样，首先，把新元素跟列表中的第一个数作对比，如果新元素已经小于列表里的第一个数了，就直接调用`PushFront`。

如果新元素比列表里的数大，那么把第一个数通过`FrontElement`，先拿出来，再把新元素插入到剩下的列表（`PopFront`）里去，最后再把刚才拿出来的数给`PushFront`进去。

也就是像这样：

```cpp
template <int Element, typename List, template <int, int> class Compare>
struct InsertT {
  using Result = IfThenElse<
    Compare<Element, FrontElement<List>>::value,
    PushFront<Element, List>,
    PushFront<
      FrontElement<List>,
      typename InsertT<Element, PopFront<List>, Compare>::Result
    >
  >;
};
```

不过这个代码有一个问题，就是没有考虑过`List`为空的情况，这一点很重要，当我们将一个大数往列表里面插入的时候，我们很容易就会递归到末尾，因此我们必须考虑`List`为空的递归终点问题。

只需要多考虑一层列表为空的情况即可，进行模板特化即可。

```cpp
template <int Element, template <int, int> class Compare>
struct InsertT<Element, NumberList<>, Compare> {
  using Result = NumberList<Element>;
};
```

最后把InsertT使用模板类型，简化写法：

```cpp
template <int Element, typename List, template <int, int> class Compare>
using Insert = typename InsertT<Element, List, Compare>::Result;
```

## InsertSort

最后的InsertSort反而没有Insert那样难了。对于非空的列表而言，只需要把第一个元素取出，递归对剩下的元素调用InsertSort，再将刚刚取出的元素插入到排序好的列表里即可，当然，需要特化一下列表为空的情况，当列表为空时，直接返回一个空列表。

```cpp
template <typename List, template <int, int> class Compare>
struct InsertionSortT {
  using Result = Insert<
    FrontElement<List>,
    typename InsertionSortT<PopFront<List>, Compare>::Result,
    Compare
  >;
};

template <template <int, int> class Compare>
struct InsertionSortT<NumberList<>, Compare> {
  using Result = NumberList<>;
};

template <typename List, template <int, int> class Compare>
using InsertionSort = typename InsertionSortT<List, Compare>::Result;
```

这样，我们就实现了编译期插入排序。

## 测试

建议使用`__PRETTY_FUNCTION__`这个功能进行打印。

```cpp
#include <iostream>

/*
 * Code
 */
int main() {
  // Check whether our implementation is correct
  using List = NumberList<1>;
  printTemplateArg<List>();
  printTemplateArg<FrontElementT<List>>();
  std::cout << FrontElement<List> << std::endl;
  printTemplateArg<PopFront<List>>();
  printTemplateArg<PushFront<4, List>>();
  printTemplateArg<IfThenElse<false, int, bool>>();
  printTemplateArg<Insert<5, NumberList<2,7,8>, Less>>();
  printTemplateArg<InsertionSort<NumberList<2,7,3,1>, Less>>();

  return 0;
}

```

或者，利用static_assert搭配`std::is_same_v`来进行测试：

```cpp
int main() {
  // Check whether our implementation is correct
  using List = NumberList<1>;
  static_assert(std::is_same_v<List, NumberList<1>>, "List should be 1");
  static_assert(FrontElement<List> == 1, "FrontElement should be 1");
  static_assert(std::is_same_v<PopFront<List>, NumberList<>>, "PopFront should be empty");
  static_assert(std::is_same_v<PushFront<4, List>, NumberList<4, 1>>, "PushFront should be 4, 1");
  static_assert(std::is_same_v<IfThenElse<false, int, bool>, bool>, "IfThenElse should be bool");
  static_assert(std::is_same_v<Insert<5, NumberList<2,7,8>, Less>, NumberList<2,5,7,8>>, "Insert should be 2,5,7,8");
  static_assert(std::is_same_v<InsertionSort<NumberList<2,7,3,1>, Less>, NumberList<1,2,3,7>>, "InsertionSort should be 1,2,3,7");

  std::cout << "All tests passed!\n";
  return 0;
}
```

## 完整代码

```cpp
// Implement insert sort but using meta programming
#include <iostream>

template <int ... Number>
struct NumberList;

template <typename List>
struct FrontElementT;

template <int Head, int ... Tail>
struct FrontElementT<NumberList<Head, Tail...>> {
    static const int value = Head;
};

template <typename List>
static const int FrontElement = FrontElementT<List>::value;

template <typename List>
struct PopFrontT;

template <int Head, int ... Tail>
struct PopFrontT<NumberList<Head, Tail...>> {
    using List = NumberList<Tail...>;
};

template<typename List>
using PopFront = typename PopFrontT<List>::List;

template <int Element, typename List>
struct PushFrontT;

template <int Element, int ... Tail>
struct PushFrontT<Element, NumberList<Tail...>> {
    using List = NumberList<Element, Tail...>;
};

template <int Element, typename List>
using PushFront = typename PushFrontT<Element, List>::List;

template <int L, int R>
struct Less {
    static const bool value = L < R;
};

template <bool Condition, typename Then, typename Else>
struct IfThenElseT;

template <typename Then, typename Else>
struct IfThenElseT<true, Then, Else> {
    using Result = Then;
};

template <typename Then, typename Else>
struct IfThenElseT<false, Then, Else> {
    using Result = Else;
};

template <bool Condition, typename Then, typename Else>
using IfThenElse = typename IfThenElseT<Condition, Then, Else>::Result;

template <typename List>
struct IsEmptyT {
    static const bool value = false;
};

template<>
struct IsEmptyT<NumberList<>> {
    static const bool value = true;
};

template <typename List>
static const bool IsEmpty = IsEmptyT<List>::value;

template <int Element, typename List, template <int, int> class Compare>
struct InsertT {
  using Result = IfThenElse<
      Compare<Element, FrontElement<List>>::value,
      PushFront<Element, List>,
      PushFront<
        FrontElement<List>,
        typename InsertT<Element, PopFront<List>, Compare>::Result
      >
  >;
};

template <int Element, template <int, int> class Compare>
struct InsertT<Element, NumberList<>, Compare> {
  using Result = NumberList<Element>;
};

template <int Element, typename List, template <int, int> class Compare>
using Insert = typename InsertT<Element, List, Compare>::Result;

template <typename List, template <int, int> class Compare>
struct InsertionSortT {
  using Result = Insert<
    FrontElement<List>,
    typename InsertionSortT<PopFront<List>, Compare>::Result,
    Compare
  >;
};

template <template <int, int> class Compare>
struct InsertionSortT<NumberList<>, Compare> {
  using Result = NumberList<>;
};

template <typename List, template <int, int> class Compare>
using InsertionSort = typename InsertionSortT<List, Compare>::Result;

template <typename A>
void printTemplateArg() {
  std::cout << __PRETTY_FUNCTION__ << std::endl;
}

int main() {
  // Check whether our implementation is correct
  using List = NumberList<1>;
  printTemplateArg<List>();
  printTemplateArg<FrontElementT<List>>();
  std::cout << FrontElement<List> << std::endl;
  printTemplateArg<PopFront<List>>();
  printTemplateArg<PushFront<4, List>>();
  printTemplateArg<IfThenElse<false, int, bool>>();
  printTemplateArg<Insert<5, NumberList<2,7,8>, Less>>();
  printTemplateArg<InsertionSort<NumberList<2,7,3,1>, Less>>();

  static_assert(std::is_same_v<List, NumberList<1>>, "List should be 1");
  static_assert(FrontElement<List> == 1, "FrontElement should be 1");
  static_assert(std::is_same_v<PopFront<List>, NumberList<>>, "PopFront should be empty");
  static_assert(std::is_same_v<PushFront<4, List>, NumberList<4, 1>>, "PushFront should be 4, 1");
  static_assert(std::is_same_v<IfThenElse<false, int, bool>, bool>, "IfThenElse should be bool");
  static_assert(std::is_same_v<Insert<5, NumberList<2,7,8>, Less>,
                               NumberList<2,5,7,8>>,
							   "Insert should be 2,5,7,8");
  static_assert(std::is_same_v<InsertionSort<NumberList<2,7,3,1>, Less>,
                               NumberList<1,2,3,7>>,
							   "InsertionSort should be 1,2,3,7");

  std::cout << "All tests passed!\n";

  return 0;
}
```
