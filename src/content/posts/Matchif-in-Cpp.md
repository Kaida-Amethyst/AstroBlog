---
title: C++模板实现MatchIf
description: 利用C++模板技巧，实现matchif语句。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/4.jpg
category: Programming
published: 2023-04-02
tags: [C++]
---

我们期待实现如下的效果：

```cpp
template <typename T, bool = MathIf<T, char, int, long>::value>
void foo() {
  //...
}
```

也就是这种MatchIf，只有当`T`是`char`，`int`，`long`的时候才能够实例化。

当然有很多种实现方式，这里介绍的是一种稍微复杂的实现.

# Typelist

首先实现一个`Typelist`，在这个`Typelist`的基础上实现`Front`和`PopFront`两个Type Function。

```cpp
template <typename... Types> struct Typelist {};

template <typename List> struct FrontT;

template <typename Head, typename... Tails> struct FrontT<Typelist<Head, Tails...>> {
  using Type = Head;
};

template <typename List> using Front = typename FrontT<List>::Type;

template <typename List> struct PopFrontT;

template <typename Head, typename... Tail>
struct PopFrontT<Typelist<Head, Tail...>> {
  using Type = Typelist<Tail...>;
};

template <typename List> using PopFront = typename PopFrontT<List>::Type;
```

# MatchIfT

接着我们实现`MatchIfT`这个Helper，这个的主要作用是，递归查看`T`与后面的类型是否匹配，如果匹配，那么返回一个`true_type`，否则返回一个`false_type`：

```cpp
template <typename T, typename List> struct MatchIfT;

template <typename T, typename... Types>
struct MatchIfT<T, Typelist<Types...>> {
  using First = Front<Typelist<Types...>>;
  using Last = PopFront<Typelist<Types...>>;
  using result_type =
      std::conditional_t<
        std::is_same_v<T, First>,
        std::true_type,
        typename MatchIfT<T, Last>::result_type>;
};

template <typename T> struct MatchIfT<T, Typelist<>> {
  using result_type = std::false_type;
};
```

但是注意这个`MatchIfT`还不能直接使用，因为我们的期待是`bool = MathIf<...>`这种形式，如果直接使用`MatchIfT`，这个无论如何都会有返回，只是值要么是`true`要么是`false`。我们需要做的SFINAE是当match成功时有`value`域，否则没有`value`域从而造成代入失败。

也就是说我们有一个`EnableMathIfV`，其定义是这样：

```cpp
template <bool> struct EnableMatchIfV {};

template <> struct EnableMatchIfV<true> {
  constexpr static bool value = true;
};
```

注意这里，对于`EnableMatchIfV<false>`是没有`value`域的。

然后，对于`true_type`和`false_type`，实际上是可以直接进行编译期求值的，然后我们将值进行求出，接着再代入到`EnableMathIfV`里面即可：

```cpp
template <typename T, typename... Types>
struct MatchIf {
  using result_type = typename MatchIfT<T, Typelist<Types...>>::result_type;
  constexpr static bool v = result_type();
  constexpr static bool value = EnableMatchIfV<v>::value;
};
```

这样的话，我们就完成了我们的目标，尝试下面的函数：

```cpp
template <typename T, bool = MatchIf<T, char, int, long>::value>
void baz() {
  std::cout << __PRETTY_FUNCTION__ << std::endl;
}

int main() {
  baz<char>();           // OK
  baz<int>();            // OK
  baz<unsigned int>();   // ERROR!
};
```
