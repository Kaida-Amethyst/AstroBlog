---
title: 限制C++中的模板模板参数
description: C++的模板参数比较好限制，但是模板模板参数似乎有一点难，这里做一个记录
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/3.jpg
category: Programming
published: 2023-04-02
tags: [C++]
---

## Motivation

在模板编程的时候，我们可能会希望限定模板参数具备某些条件，例如拥有某些成员变量，或者某些成员函数。例如，我们写一个函数，这个函数接受一个容器类为模板参数，返回一个装有一个元素`(int)0`​的容器，可以这么写：

```cpp
template<typename ResultType>
ReusltType one_element_container() {
  ResultType res;
  res.reserve(1);
  res.push_back(1);
  return res;
}
```

上面的函数如果使用`one_element_container<std::vector<int>>()`​，是没有问题的，但是这种写法，至少存在以下的三个问题：

1. 函数体内使用了`reserve`​和`push_back`​，而传入的`ResultType`​未必有这两个成员函数。
2. 没有限定`ResultType`​就一定是一个容器类，只要`ResultType`​有`reserve`​和`push_back`​两个成员函数，这个函数就能够成功编译。
3. 我们未必希望明示容器里的数据类型，换句话说，我们可能会希望我们只需要指定容器的类型是`vector`​或者是`set`​，不需要制定容器里的数据的类型是什么。

---

## template template 参数

先来解决后两个问题，如果我们只需要指定容器类型，需要使用这种模板模板参数，也就是模板参数本身又是一个模板，具体的来看下面的代码：

```cpp
template <template <typename E> typename Container>
auto one_element_container() {
  Container<int> res;
  res.reserve(1);
  res.push_back(1);
  return res;
}
```

这个函数就只需要一个容器模板参数，`one_element_container<std::vector>`​即可返回一个`std::vector<int>`​，而无需指定数据类型。

对于模板`template <template <typename E> typename Container>`​，这一行要这么来看，首先外层的`template<...>`​声明了这是一个模板，里面的`template...`​说明这个模板参数本身又是一个模板。这个里层的模板是`<typename E> typename Container`​，后面的`typename Container`​是说`Container`​是一个类型，然后这个类型又挂靠着一个`<typename E>`​，在主函数内部需要指明`Container`​内的数据类型。

> 注意：这种使用auto作为返回类型的写法到C++14才支持，如果在C++11下，需要使用`Container<int>`​作为返回类型。另外，`Container`​前面的`typename`​在C++17之前都需要使用`class`​。

---

## allocator

我们希望传入的模板参数是一个容器类。一般而言，容器类的标志是使用`allocator`​，那么前一章的代码添加一个对`Container`​模板参数类型的限定即可：

```cpp
template <template <typename E, typename Allocator=std::allocator<E>> typename Container>
auto one_element_container() {
  Container<int> res;
  res.reserve(1);
  res.push_back(1);
  return res;
}
```

这个办法可以保证在大多数情况下（注意，仍然有可能绕过去），能够保证`Container`​是一个容器。另外，在C++11下，如果要传入`vector`​之类的容器，这个`allocator`​的信息是必须的。

---

## 限定模板参数

然后，我们需要限定模板参数，需要限定模板参数具有`reserve`​和`push_back`​成员函数。

> 注：这很重要，因为在更广泛的模板编程中，我们通常会让不同的模板参数采取不同的动作，而不仅仅只是为了触发编译报错。例如如果模板参数是`std::vector`​，我们可能需要`reserve`​和`push_back`​，如果是`std::list`​，我们就不需要`reserve`​，如果是`std::set`​我们又必须把`push_back`​换成`add`​。如果仅仅只是针对本文所要解决的问题，自然有更巧妙的办法。但是这里要讨论的是更通用的解决方案。

这个问题事实上是一个比较复杂的问题，不同的C++版本有不同的解决方案。

### SFINAE in C++11

SFINAE，全称Substitution Failure Is Not An Error，代入失败并非错误。它指的是当模板参数代入错误的时候，C++编译器不会认为这是一个错误，而是会转向下一个可能符合的模板。一个比较简单的例子：

```cpp
template<typename T, typename = typename std::is_same_v<T, int>>
auto foo() {
  T x = 1;
  return T;
}

template<typename T, typename = typename std::is_same_v<T, double>>
auto foo() {
  T x = 2.0;
  return T;
}
```

在实现上面两个函数的基础上，如果使用了`foo<double>()`​，那么这个模板参数代入到第一个模板函数时，会出现代入失败，但是编译器不会报错，而是会去寻找下一个候选者，代入到第二个模板函数里。对于`foo<long>()`​，会分别代入两个模板函数，直到两次代入都失败，编译器才会报出一个无候选者的错误。

利用这个特性，我们可以在模板参数上来限定Container具有某些成员函数。看下面的代码：

```cpp
template <typename Container> 
struct has_reserve_and_push_back {

  template <typename C, typename = decltype(std::declval<C>().reserve(1))>
  static std::true_type test_reserve(int);

  template <typename C> 
  static std::false_type test_reserve(...);

  template <typename C, typename = decltype(std::declval<C>().push_back(
                            std::declval<typename C::value_type>()))>
  static std::true_type test_push_back(int);

  template <typename C> 
  static std::false_type test_push_back(...);

  static constexpr bool value = decltype(test_reserve<Container>(0))::value 
                             && decltype(test_push_back<Container>(0))::value;
};
```

这个代码比较复杂，一点一点来解释：

1. ​`std::false_type`​和`std::true_type`​是`STL`​下个一个`struct`​，用于在编译期恒定返回一个`true`​或者`flase`​。
2. ​`std::declval<T>`​这个用于在编译期生成一个伪的T类型的对象，然后通过这个对象可以调用或者查看T类型的成员函数或者成员变量。
3. ​`decltype`​用于解析所跟随的表达式的类型。
4. ​`...`​参数是可变模板参数，填充的参数的数量是不一定的。

所以以`test_reserve`​为例，代码：

```cpp
  template <typename C, typename = decltype(std::declval<C>().reserve(1))>
  static std::true_type test_reserve(int);

  template <typename C> 
  static std::false_type test_reserve(...);
```

对于第一个函数，`typename = decltype(std::declval<C>().reserve(1))`​的意思，就是编译器会尝试在编译期去生成一个C的对象，然后尝试调用这个对象的`reserve`​成员函数，如果有这个成员函数，那么整体的`decltype(std::declval<C>().reserve(1))`​就代入成功，那么就不会再去带入第二个函数。否则，它会代入失败，并且尝试代入第二个函数。之所以第二个函数的参数是`...`​，是因为已经对第二个函数的参数是什么已经无所谓了，反正都会返回`std::false_type`​，因此干脆使用`...`​。对于`test_push_back`​也是同理。

最后使用一个`constexpr`​来在编译期计算出是否拥有`reserve`​和`push_back`​这两个函数。

使用的时候，只需要继续在模板参数中对`Container`​进行限制：

```cpp
template <
    template <typename E, typename Alloc = std::allocator<int>> class Container,
    typename = std::enable_if<has_reserve_and_push_back<Container<int>>::value>>
Container<int> one_ele() {
  Container<int> res;
  res.reserve(1);
  res.push_back(1);
  return res;
}
```

这里的`typename std::enable_if<has_reserve_and_push_back<Container>::value>::type`​​，就完成了对`Container`​​成员函数的限制。

### 利用decltype

除了上面这种利用struct和enable_if的，还有一种稍微简单的写法，使用`decltype`​和`declval`​:

```cpp
template <template <typename E, typename Alloc = std::allocator<int>>
          class Container>
auto one_ele()
    -> decltype(std::declval<Container<int>>().reserve(1),
                std::declval<Container<int>>().push_back(1), Container<int>()) {
  Container<int> res;
  res.reserve(1);
  res.push_back(1);
  return res;
}


```

这里的`decltype(std::declval<Container<int>>().reserve(1), std::declval<Container<int>>().push_back(1), Container<int>())`​里面是一个逗号表达式，相当与在编译期会尝试调用`reserve`​和`push_back`​这两个函数，然后把`Container<int>`​给返回作为返回类型。

## C++17下的`if constexpr`​

到了C++17，就可以直接使用`if constexpr`​，从而避免使用冗长的SFINAE技巧。

```cpp
template <template <typename E, typename Alloc = std::allocator<int>>
          class Container>
auto one_ele() {
  Container<int> res;
  if constexpr (std::is_same_v<decltype(res.reserve(1)), void>) {
    res.reserve(1);
  }
  if constexpr (std::is_same_v<decltype(res.push_back(1)), void>) {
    res.push_back(1);
  }
  return res;
}
```

## C++20下的concept

C++17下的`if constexpr`​已经很不错了，但是仍然有一些问题，有的是否可能比较难以模块化，例如，我可能希望把这种限制进行拆分。C++20引入了concept：

```cpp
template <typename Container>
concept ReserveAndPushBack = requires(Container c) {
  c.reserve(1);
  c.push_back(1);
};
```

声明好这种`concept`​之后，像这样去使用：

```cpp
template <template <typename E, typename Alloc = std::allocator<int>>
          class Container>
  requires ReserveAndPushBack<Container<int>>
auto one_ele() {
  Container<int> res;
  res.reserve(1);
  res.push_back(1);
  return res;
}
```

还有另一种更简便的写法，结合`if-constexpr`​和`requires`​，在函数体内进行限制：

```cpp
template <template <typename E, typename Alloc = std::allocator<int>>
          class Container>
auto one_ele() {
  Container<int> res;
  if constexpr (requires(Container<int> c) {
                  c.reserve(1);
                  c.push_back(1);
                }) {
    res.reserve(1);
    res.push_back(1);
  }
  return res;
}
```
