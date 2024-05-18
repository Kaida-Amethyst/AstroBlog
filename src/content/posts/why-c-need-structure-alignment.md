---
title: 为什么C语言需要结构体对齐
description: 提供一个例子，解释为什么C语言需要结构体对齐
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Python/1.jpg
category: 技术原理
published: 2022-07-16
tags: [C]
---

C语言中的struct会做内存对齐，这个已经是被广为人知的，但是网上对于内存对齐的原因的解释却比较少，很多资料仅仅是说为了“提高读写效率"，这个理由其实还是很难有说服力。

这里提供一个新的解释思路，就是我们要知道`struct`是可以直接做赋值的，利用`=`符号可以直接做深拷贝。

```c
typedef struct Foo Foo;
struct Foo {
  int a;
  short b;
  char c;
};

int main () {
  Foo X = {1, 2, 'a'};
  Foo Y = X;
  Y.c = 'c';
  printf("X.c = %c, Y.c = %c\n", X.c, Y.c);
  // X.c = a, Y.c = c
  return 0;
}
```

然后我们来思考一下，这个深拷贝怎么去实现。

**如果有内存对齐**:

这个`struct Foo`，根据内存对齐的知识，我们知道`sizeof(Foo) = 8`，而x86上的寄存器位宽最大就是8，因此我们只需要读写一次就可以完成`Foo Y = X`这样的深拷贝了。

我们把上面的代码的汇编打印出来，可以看到下面的x86汇编：

```x86asm
        movl    $1, -8(%rbp)
        movw    $2, -4(%rbp)
        movb    $97, -2(%rbp)
        movq    -8(%rbp), %rax
        movq    %rax, -16(%rbp)
```

上面的三行代码对应的是`Foo X = {1, 2, 'a'}`，最后两行代码就是`Foo Y = X;`，可以看到在汇编中第4行直接使用`%rax`寄存器访存，然后再向`-16(%rbp)`的地址处写入8字节。

**如果没有内存对齐**

实际上，我们也可以要求C语言不做内存对齐的，将上面的`struct Foo`修改一下：

```c
struct __attribute__((__packed__)) 
Foo {
  int a;
  short b;
  char c;
};
```

这个时候要完成一次深拷贝要稍微复杂一点，不能直接利用一个`%rax`寄存器，因此此时`sizeof(Foo) = 7`，而`%rax`的位宽是8，像上面的汇编代码那样做，会直接写越界。为了保证不会越界写，编译器就能一个变量一个变量地去写，有几个变量就要做几次读写，也就是说，需要做三次读写。

我们再次编译上面的代码来看看：

```x86asm
        movl    $1, -7(%rbp)
        movw    $2, -3(%rbp)
        movb    $97, -1(%rbp)
        movl    -7(%rbp), %eax
        movl    %eax, -14(%rbp)
        movzwl  -3(%rbp), %eax
        movw    %ax, -10(%rbp)
        movzbl  -1(%rbp), %eax
        movb    %al, -8(%rbp)
```

头三行仍然是对应`Foo X = {1, 2, 'a'}`，但是可以看出`Foo Y = X;`多了不少代码，第4,5两行实际上是`Y.a = X.a`，第6,7行实际上就是`Y.b = X.b`，紧接着第8,9两行就是`Y.c = X.c`。

可以看出，如果没有内存对齐，`struct`做深拷贝的效率将会降低不少。
