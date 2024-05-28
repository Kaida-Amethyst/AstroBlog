---
title: C语言根据宏参数个数不同采取不同动作
description: 有的时候我们希望当宏参数不同时，C语言能够采取不同的动作，例如只有一个参数时，扩展成一个if，当有两个参数时，扩展成一个if和一个else if。实际上使用一些技巧完全可以让C宏做到这件事。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/5.jpg
category: 编程实践
published: 2023-03-11
tags: [C]
---

我们有时会需要实现这样的功能，需要X或者等于1，或者等于2，或者等于3，等等。

当然在高级语言中可以通过哈希表来实现，不过这一次来尝试用宏实现一个`UX_IN_NUMS`，具体点说，下面的代码：

```c
UX_IN_NUMS(X, 1, 2, 3)
```

会扩展成

```c
*(unsigned*)&X == 1 || *(unsigned*)&X == 2 || *(unsigned*)&X == 3
```

下面就来尝试实现这样的宏。

## 做法

```cpp
// NARG的宏参数不能超过7个，如果超过7个的话需要修改宏，在ARG_N
// 和RSEQ_N宏后面添加即可
#define NARG(...) NARG_(0, ##__VA_ARGS__, RSEQ_N())
#define NARG_(...) ARG_N(__VA_ARGS__)
#define ARG_N(_1, _2, _3, _4, _5, _6, _7, _8, N,...) N
#define RSEQ_N() 7,6,5,4,3,2,1,0

// 在代码中下面是直接变成具体数字的
// NARG() --> 0
// NARG(a) --> 1
// NARG(a, b) --> 2
```

连等

```cpp

#define Concat(a, b) a ## b
#define XConcat(a, b) Concat(a, b)

#define UEQ(X, a) $U(X) == a
#define UX_IN_NUMS1(X, a)                UEQ(X, a)
#define UX_IN_NUMS2(X, a, b)             UEQ(X, a) || UX_IN_NUMS1(X, b)
#define UX_IN_NUMS3(X, a, b, c)          UEQ(X, a) || UX_IN_NUMS2(X, b, c)
#define UX_IN_NUMS4(X, a, b, c, d)       UEQ(X, a) || UX_IN_NUMS3(X, b, c, d)
#define UX_IN_NUMS5(X, a, b, c, d, e)    UEQ(X, a) || UX_IN_NUMS4(X, b, c, d, e)
#define UX_IN_NUMS6(X, a, b, c, d, e, f) UEQ(X, a) || UX_IN_NUMS5(X, b, c, d, e, f)
#define DUMMY_UX_IN_NUMS(M, X, ...) M(X, __VA_ARGS__)
#define UX_IN_NUMS(X, ...) DUMMY_UX_IN_NUMS(XConcat(UX_IN_NUMS, NARG(__VA_ARGS__)), X, __VA_ARGS__)

// UX_IN_NUMS(X, 1, 2, 3)扩展成
// DUMMY_UX_IN_NUMS(XConcat(UX_IN_NUMS, NARG(__VA_ARGS__)), X, __VA_ARGS__)
// 里面的XConcat(UX_IN_NUMS, NARG(__VA_ARGS__))
// 扩展成XConcat(UX_IN_NUMS, 3)  --> UX_IN_NUMS3
// 然后DUMMY_UX_IN_NUMS(UEQ3, X, 1,2,3)变成UX_IN_NUMS3(X, 1, 2, 3)
// 然后再扩展UX_IN_NUMS3
```
