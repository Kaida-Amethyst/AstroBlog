---
title: 在C++中编写Cached函数
description: python里面有一个超好用的cache关键字，可以用来提升性能，C++虽然没有这样的东西，但是我们可以来自己实现
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Math/4.jpg
category: Programming
published: 2023-04-02
tags: [C++]
---
------

## Python Cache装饰器

在python里面，有一个装饰器：

```python
from functools import cache

@cache
def fact(n):
  if n == 1:
    return 1
  return n * fact(n)
```

这个装饰器，可以存储已经计算过的值，从而避免重复计算。

```python
def main():
  fact(3)
  fact(5)

if __name__ == "__main__":
  main()
```

那么在C++里面，我们如何来实现这样的函数？

---

## Cached_func

C++不像Python那样有装饰器，因此这种功能只能自己来实现。这篇博客来一步一步地实现一个相对通用的cached函数。

首先，假设我们有这样一个函数：

```cpp
int fib(int n) {
  if (n == 1 || n == 2)
    return 1;
  return fib(n-1) + fib(n-2);
}
```

很显然，这样函数包含了相当多的重复计算。那么我们的Cached函数，可以这样写，就是我们有一个`call_func_cached`​，这个函数的第一个参数是函数，其余的参数是这个函数的参数，在这个`call_func_cached`​内部，维护一个map，然后通过传入的参数在map中寻找是否已经计算过值，如果已经计算过，那么就直接使用map中的值，如果没有计算过，那么再进行函数调用计算。

---

### Version 1

```cpp
template <typename Func, typename Arg>
auto call_func_cached(Func func, Arg arg) {
  using result_type = decltype(func(arg));
  static std::map<Arg, result_type> cached_values;

  auto it = cached_values.find(arg);

  if (it != cached_values.end()) {
    return it->second;
  }

  return cached_values.insert({arg, func(arg)}).first->second;
}
```

上面的函数实现完成后，调用的时候，要显式地使用`call_func_cached`​来进行调用：

```cpp
int cached_fib(int n) {
  if (n == 1 || n == 2) {
    return 1;
  }
  return call_func_cached(cached_fib, n - 1) +
         call_func_cached(cached_fib, n - 2);
}
```

不过，上面的实现过于初级，一个问题就是函数可能不止一个参数，因此我们需要加入不定参数模板。

----

### Version 2

使用不定参数后，函数可以支持多个参数了，但是这样一来，map的key就需要改用tuple。另外，注意最后的return，我们使用了`std::move(key)`​。

```cpp
template <typename Func, typename... Args>
auto call_func_cached(Func func, Args... args) {
  using result_type = decltype(func(args...));
  using param_set = std::tuple<Args...>;
  
  param_set key{args...};

  static std::map<param_set, result_type> cached_values;

  auto iter = cached_values.find(param_set(args...));

  if (iter != cached_values.end()) {
    return iter->second;
  }

  return cached_values.insert({std::move(key), func(args...)})
      .first->second;
}
```

但是仍然有问题，我们的目标是写出一个相对通用的cached function，而一个函数的参数，可能有传引用等情况，而上面的函数是传值的。

-----

### Version 3

为了避免这种全部传值的写法，我们把`Args... args`​修改成`Args && ... args`​，就成了`forward reference`​，注意这不是右值引用。

```cpp
template <typename Func, typename... Args>
auto call_func_cached(Func func, Args &&...args) {
  using result_type = decltype(func(std::forward<Args>(args)...));
  using param_set =
      std::tuple<std::decay_t<Args>...>;

  param_set key{args...};
  static std::map<param_set, result_type> cached_values;

  auto iter = cached_values.find(key);

  if (iter != cached_values.end()) {
    return iter->second;
  }

  return cached_values.insert({std::move(key), func(std::forward<Args>(args)...)})
      .first->second;
}
```

----

### Version 4

更加现代一点的写法：

```cpp
template <typename Func, typename... Params>
auto call_func_cached(Func fun, Params &&...params) {
  using param_set = std::tuple<std::remove_cvref_t<Params>...>;

  param_set key{params...};

  using result_type =
      std::remove_cvref_t<std::invoke_result_t<Func, decltype(params)...>>;

  // Notice that it's not thread safe
  static std::map<param_set, result_type> cached_values;
  using value_type = decltype(cached_values)::value_type;

  auto iter = cached_values.find(key);

  if (iter != cached_values.end()) {
    return iter->second;
  }

  return cached_values
      .insert(value_type{std::move(key), fun(std::forward<Params>(params)...)})
      .first->second;
}
```
