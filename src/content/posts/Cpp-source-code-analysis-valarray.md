---
title: C++标准库源码剖析 - valarray
description: C++标准库中的valarray是一个支持运算符操作的数组，这里来剖析一下它的源码
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/2.jpg
category: 编程实践
published: 2023-07-12
tags: [C++, Source Code Analysis]
---

-----

## Overview

valarray是C++标准库中的"变长数组"，但是与vector不一样的是，`vector`事实上并没有那么像"向量"，反而是这个valarray比vector更像一个向量。

```cpp
using namespace std;

vector<int> x;
x += 10; // Error!

valarray<int> y;
y += 10; // OK!
```

valarray可以像标量那样去做一些运算，而vector是不可以这样做的，因此valarray在外观上比vector更加像一个向量。

## How to implement?

如果我们自己想写一个vector，我们会发现其实相对来说比较容易，因为它基本上就是变长数组。但是valarray实现起来，实际上并不太容易。例如，我们考虑下面的语句：

```cpp
valarray<int> x, y, z;
x = y + z;
```

C++本身是支持运算符重载的，但问题是，如果我们这样写：

```cpp
valarray& valarray::operator=(const valarray& other) {
  assert(this->size == other.size):
  for(int i = 0; i < this->size; ++i) {
    _data[i] = other[i];
  }
  return *this;
}

valarray operator+(const valarray& lhs, const valarray& rhs) {
  assert(lhs.size == rhs.size):
  valarray result(lhs.size);
  for(int i = 0; i < this->size; ++i) {
    result[i] = lhs[i] + rhs[i];
  }
  return result;
}
```

虽然这样能够运行，但是毫无疑问它的性能较差，因为`x=y+z`这个动作会暗含两个循环，首先是两个向量的相加，而且相加的结果是在另一块内存上，然后是内存的移动。

那么自然我们就要想，能不能把这两重循环合并起来？

## Proxy

要知道，我们不能事事指望编译器为我们做某种优化，事实上，在编译器看来，我们就是有两个循环，编译器很可能也没办法帮助我们推断这个额外多出来的一块内存到底需不需要废弃掉。因此，我们可能需要另外想个办法来，看看这个`=`和`+`能不能合并。

实际上是有办法的，一个最核心的想法就是惰性求值，也就是对于`y+z`，生成一个记录，去记录加法，左操作数是`y`右操作树是`z`，然后在`=`赋值时，我再进行加法操作。也就是下面的思想：

```cpp
class AddProxy {
  Valarray& lhs;
  Valarray& rhs;
  AddProxy(const Valarray& l, const Valarray& r) : lhs{l}, rhs{r} {}
};

AddProxy operator+(const valarray& lhs, const valarray& rhs) {
  assert(lhs.size == rhs.size):
  return AddProxy{lhs, rhs};
}

valarray& valarray::operator=(const AddProxy & addproxy) {
  assert(addproxy.lhs.size == this->size):
  for(int i = 0; i < this->size; ++i) {
    _data[i] = addproxy.lhs[i] + addproxy.rhs[i] 
  }
  return *this;
}
```

这个就是核心思想，对于一般的`operator`，不进行计算，只返回一个`Proxy`，在需要进行赋值时再进行计算。

## Difficulty

这个核心思想虽然很容易理解，但是，事实上并不容易实现，基本上有以下几个难点：

* C++中涉及的运算比较多，除了加减乘除还有位运算，比较运算，而且还会遇到函数，这些怎么办？
* 连续的表达式，怎么办？例如`x = a + b - c * -(~d)`。
* 与标量运算怎么做？`x = y + 1`怎么处理？标量与向量的类型不一致怎么办？

这些基本就是`valarray`遇到的难点，事实上，这些问题相当不好解决。对上面的问题如果细想，就会发现它指向的问题实际上可能是一个类型论的问题。实践当中的`valarray`很少有人用，一个比较可能的原因也是`valarry`可能也没有非常好地解决上面的问题，但是无论如何，`valarry`都做了很好的尝试，因此有必要对它的源码进行一些阅读和分析。对`valarray`的分析也有利于我们之后对`simd`的实现与分析。

## Minimum Operator

我们先来研究两个最简单的运算符，取负和加法，注意我们这里尽管只研究了两个，但是我们仍然假设我们已经实现了很多个运算符了。

```cpp
struct __negate;
struct __plus;
```

## Closure and Expression

把标准库的`valarry`头文件打开，我们能看到以下的一些信息，为了方便阅读，这里消去了部分下划线

```cpp
template <class Clos, typename T> class _Expr;

template <class Oper, template<class, class> class Meta, typename Dom> class _UnClos;

template <class Oper,
          template<class, class> class Meta1
          template<class, class> class Meta2,
          class Dom1, class Dom2>
class _BinClos;
```

先了解一下整体思路。

对于每一个完整的表达式，都会返回一个`_Expr`，这个`_Expr`里面的`Clos`包裹着两个类型，一个是闭包运算，另一个是它本身的类型，用于向上推导。

假设我们已经有一个`valarray<int> x`，那么对于`-x`，首先得到闭包，`_UnClos<__negate, _Valarray, int>`，接着对这个表达式的类型进行推导，`int`取负仍然是`int`（注意有些运算是会改变类型的，例如`!`运算的结果类型就是`bool`）。那么也就是说对于`-x`这个表达式的类型是：`_Expr<_UnClos<__negate, _Valarray, int>, int>`。

注意这里出现了一个`_Valarray`，它的定义如下：

```cpp
template <typename, typename> class _Valarray;
```

这个`_Valarray`只是为了指示闭包的下一层是一个`valarray`，一个闭包的下一层也有可能是`_Expr`，因为闭包的下一层的`Meta`它的形式要求是`template<class, class> class Meta`所以`_Valarray`也相应地是这种模板模板，而不是普通的模板。

对于`-x`能够明白的话，那么对于`-(-x)`呢？对于外层的`-`号，毫无疑问的是，闭包是`_UnClos<__negate, _Expr, ...>`，主要是第三个参数，实际上我们只需要内部`-x`的闭包，因此整体的`-(-x)`的类型就应当是：`_UnClos<__negate, _Expr, _UnClos<__negate, _Valarray, int>>`，而对于这个表达式的类型，应该从内部`-x`的类型进一步推断出来应该是`int`，所以整体的`-(-x)`的类型就应该是：

```cpp
_Expr<_UnClos<__negate, _Expr, _UnClos<__negate, _Valarray, int>>
```

## Computing

如果`-(-x)`接着与其它的`_Expr`或者`valarray`进行运算，那么会接着生成新的`_Expr`类型，但我们现在需要关心一下的是，这个`_Expr`如果进行了赋值操作，那么它是怎样进行运算的？

对于`valarray`中的`operator=`，它的内部是这样：

```cpp
valarray& operator=(const _Expr<...> & expr) {
  for (...) {
    _data[i] = expr[i];
  }
}
```

也就是说，通过重载`_Expr`的索引。

更具体的流程是，对于一个`_Expr`，它的索引应该是去取它闭包的索引，一个闭包的索引应该是去取它下级的闭包或者是下级的valarray的索引。

以前面的`-(-x)`为例，它的类型是：

```cpp
_Expr<_UnClos<__negate, _Expr, _UnClos<__negate, _Valarray, int>>, int>
```

这个类型的索引，应该是它的闭包的索引，也就是`_UnClos<__negate, _Expr, _UnClos<__negate, _Valarray, int>>`的索引，而这个闭包的索引，就应该是它的下级闭包的索引，也就是`_UnClos<__negate, _Valarray, int>`的索引，而这个闭包的索引，就应该是它的下级`valarray`的索引。

注意这里就出现了一个类型选择的问题你，对于`_Expr`，没有类型选择的问题，它的索引就是它的闭包的索引，但是对于闭包的索引，要根据它的下级是另一个闭包还是一个valarray要做不同的判断。仔细观察上面的闭包：

```cpp
_UnClos<__negate,
        _Expr, 
        _UnClos<...>>  // 闭包1
_UnClos<__negate, _Valarray, int> // 闭包2
```

这里是有差别的，需要在实现当中特别注意。

## Implementation

现在我们就需要来实现上面的思路。

### Operator-

首先对于`valarray`当中的`operator-`，实现方式如下：

```cpp
template <T>
class valarray{;
/* ... */
  
  _Expr<_UnClos<__negate, _Valarray, T>, T>
  operator-() const {
    using Clos = _UnClos<__negate, _Valarray, T>;
    using RT   = T;
    return _Expr<Clos, RT>(Clos(*this));
  }

};
```

这里的实现的主要问题在于对于RT的处理是人为的，而不是通过某种机制进行推理的，因为我们事实上预设了这里的`T`就是基本类型，但是`valarray`在设计的时候并不是专门为了基本类型设计的。因此类型上应该需要通过某种机制进行推断。

然后，我们这里还没有实现`_Expr`和`Closure`，需要进行实现。

### _Expr

`_Expr`比较简单，就是获取闭包，然后有一个索引，索引直接指向这个闭包的索引即可，然后需要有一个`operator-`。

特别需要注意的是，`_Expr`的第二个参数`T`，这个`T`实际上有两种情况，基本类型或者是另一个闭包。

然后，额外注意这里`operator-`的写法。（注意这里没有对RT的推导，可能是`valarray`不好的地方）。

```cpp
template <class Closure, typename T> struct _Expr {
  const Closure M_closure;

  _Expr(const Closure &closure) : M_closure(closure) {}

  RT operator[](int i) const { return M_closure[i]; }

  _Expr<_UnClos<__negate, _Expr, Closure>, T>
  operaror-() const {
    using Clos = _UnClos<__negate, _Expr, Closure>;
    return _Expr<Clos, T>(Clos(this->M_closure));
  }

  /* ... */
};
```

### _UnClos

实际上闭包应该有很多种，至少应该包括unary和binary的闭包，`valarry`只实现了这两种，现在来实现`_UnClos`。

注意，根据前面的讨论`_UnClos`应该是有两种特化实现的，一种是它的下一层直接是一个`_Valarray`，另一种是它的下一层是另一个闭包。区别在于，对于`_UnClos<Op, _Expr, Dom>`，它的索引应该是`Dom`的索引，但是对于`_UnClos<Op, _Valarray, T>`，它的索引应该是`valarray<T>`的索引。

所以，这里首先实现了一个`_ValArrayRef<T>`，它的作用是，要么返回T，要么返回`valarray<T>&`。

```cpp
template <typename T>
struct _ValArrayRef {
  using type = const T;
};

// use real reference for valarray
template <typename T>
struct _ValArrayRef<valarray<T>> {
  using type = const valarray<T> &;
};
```

然后实现一个UnBase类，注意这里的`Arg`，有两种情况，一种是`valarray`，另一种是另一个闭包。

```cpp
template <class Oper, class Arg>
class UnBase {
private:
  typename _ValArrayRef<Arg>::type M_expr;

public:
  using Vt = typename Arg::value_type;
  using value_type = typename __fun<Oper, Vt>::result_type;

  UnBase(const Arg &e) : M_expr(e) {}

  value_type operator[](size_t i) const { return Oper()(M_expr[i]); }
};
```

然后，对`_UnClos`进行实现，去特化这个`UnBase`：

第一种情况，对于`_UnClos`直接下连一个`valarray`的情况，特化`UnBase<Op, valarray<T>>`

```cpp
template <class Oper, typename T>
struct UnClos<Oper, _ValArray, T> : UnBase<Oper, valarray<T>> {
  using Arg = valarray<T>;
  using _Base = UnBase<Oper, Arg>;
  using value_type = typename _Base::value_type;

  UnClos(const Arg &e) : _Base(e) {}
};
```

第二种情况，对于`_UnClos`下是一个`_Expr`的情况，特化`UnBase<Op, Dom>`，强调这里的`Dom`是另一个闭包。

```cpp
template <class Oper, class Dom>
struct UnClos<Oper, _Expr, Dom> : UnBase<Oper, Dom> {
  using Arg = Dom;
  using _Base = UnBase<Oper, Dom>;
  using value_type = typename _Base::value_type;

  UnClos(const Arg &e) : _Base(e) {}
};
```

实现完上述的步骤之后，应该就能够顺利完成`y = -(-x)`这样的动作了。
