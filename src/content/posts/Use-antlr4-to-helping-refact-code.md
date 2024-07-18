---
title: 利用Antlr4辅助重构代码
description: 重构代码一直是一件消耗人力的事，不过如果把代码重构理解为语言语法树的重构，那么很多问题上会省力很多。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Python/5.jpg
category: 编程实践
published: 2024-07-08
tags: [C++, Compiler]
---
------

## Overview

如果有一天，代码库添加了一个非常强力的特性，因为这个特性的添加，导致剩余的代码库需要做大量的重构，这个时候应该怎么办？

大多数团队的做法是组织人力进行修改，每个人分配一部分代码库，然后每个程序员对各自负责的代码库进行修改。

但其实，对这样的问题我们可以使用另一种方式进行思考，就是说，我们可以将重构代码这件事，从另一个角度去理解，将它理解为“语法树的改造”。

现在看下面的一个重构代码的例子，假设我们在代码中，频繁地使用这样形式的表达式：

```cpp
float a, b, c, d;
// ....
d = a * b + c;
```

这种先乘后加的形式，我们可能希望使用融合乘加的操作，也就是修改成这样：`d = fma(a, b, c)`。

咋一看似乎不难，似乎可以使用正则表达式，但是实际上，这个问题要复杂地多，看下面的例子：

```cpp
e1 = a * b + c * d;
e2 = c + a * b;
e3 = *c + +a * b + ++c;
e4 = *d + *(*a + b) * c;
e5 = *a++ * *b(c) + func(*c * +d); 
```

上面的例子全部都可以尝试去做出fma的修改，但是很显然没办法通过一个简简单单的正则表达式去进行修改。

实际上还有更加复杂的情况：

```cpp
int a, b, c, d1, d2, d3;
float fa, fb, fc;

d1 = a * b + c;
d2 = fa * fb + fc;

d3 = fa * fb;
d3 += fc;
```

上面的例子中，第一个表达式`d1 = a * b + c`实际上是不太好用fma去进行替换的，因为它是一个整数运算，而第二个表达式`d2 = fa * fb + fc`却可以。也就是说，是否能够替换还牵涉到类型的问题。至于d3就更不必说了，直接涉及到两条语句了。所以很难有一个非常简单的替换规则就可以做替换。

---

从上面的例子就可以看出来，即便是一个很简单的重构规则，往往需要一整个团队来进行大规模地工作，花费大量的时间和精力去解决这样的问题。

> 能否使用GPT来做呢？GPT的工作还算是靠谱的，但问题是，GPT的准确性有的时候会出问题，最终生成的代码还是需要人来把关，这样的话实际上还是会消耗人力。

不过，就像本博客在前面提到的那样，你可以去把重构代码这件事，理解为语法树的改造，

尽管语言上的形式千奇百怪，但是本质上，就是将语法树上的一个特殊的加法节点替换成函数调用节点。

所以实际上，可以想办法写一个语言的Parser，然后再使用这个Parser去构建语言的语法树，再接着对语法树进行改造，再打印出改造后的语法树即可。

## Antlr4

目前已经有很多的前端分析工具，例如flex & bison，tree-sitter，clang。使用的工具并不重要，只要能够分析出语法构建出语法树，都可以完成本博客中所描述的操作。对于本博客而言，使用的是Antlr4。

Antlr4本身是Java构建，但也支持使用其它语言，这里用C++来操控Antlr4。

先来介绍一下antlr4的安装方式。

### Antlr4 Installation

antlr4本身是java写的，因此首先需要下载安装java。

首先运行`javac --version`，先看一下是否已经下载安装好了java，如果没有的话，需要下载安装java。

### Java下载

对于适当的linux发行版来说，可以直接使用命令，注意java分成两个版本：JRE和JDK，如果只需要运行java程序，下载JRE，如果还希望做java开发，下载JDK。如果不确定，就无脑下载JDK。

* Debian系（Ubuntu，deepin等）：`sudo apt-get install default-jre`或`sudo apt-get install default-jdk`
* Arch系：`sudo pacman -S jre-openjdk`或者`sudo pacman -S jdk-openjdk`

如果以上的命令行不通，可以直接从官网上下载安装最新版。

* JRE下载：https://www.java.com/en/download/
* JDK下载：https://www.oracle.com/java/technologies/jdk-script-friendly-urls/

将下载下来的压缩包解压，放在特定位置下。然后在`/usr/bin`目录下做`java`和`javac`的软链接。确认在命令行下使用`java -version`或者`javac -version`有正确的输出即可。

### Antlr下载

可以直接在ANTLR官网下载jar包：

https://www.antlr.org/download.html

不过官网下载基本是最新版本，有时会出现所需要的JVM版本过高的情况，此时可以选择旧版本，在github上下载：

https://github.com/antlr/website-antlr4/tree/gh-pages/download

### ANTLR安装

将下载下来的jar包推荐放在`/usr/local/lib`下，然后在`.bashrc`或者`.bash_profile`下设定CLASSPATH（假设下载的是4.12的jar包）：

```shell
export CLASSPATH=".:/usr/local/lib/antlr-4.12-complete.jar:$CLASSPATH"
```

然后就可以通过java来启动ANTLR，有两种方式：

```shell
$ java -jar /usr/local/lib/antlr-4.12-complete.jar
```

或者：

```shell
$ java org.antlr.v4.Tool
```

一般而言会出现如下的信息：

```plaintext
ANTLR Parser Generator  Version 4.10
 -o ___              specify output directory where all output is generated
 -lib ___            specify location of grammars, tokens files
 -atn                generate rule augmented transition network diagrams
 -encoding ___       specify grammar file encoding; e.g., euc-jp
 -message-format ___ specify output style for messages in antlr, gnu, vs2005
 -long-messages      show exception details when available for errors and warnings
 -listener           generate parse tree listener (default)
 -no-listener        don't generate parse tree listener
 -visitor            generate parse tree visitor
 -no-visitor         don't generate parse tree visitor (default)
 -package ___        specify a package/namespace for the generated code
 -depend             generate file dependencies
 -D<option>=value    set/override a grammar-level option
 -Werror             treat warnings as errors
 -XdbgST             launch StringTemplate visualizer on generated code
 -XdbgSTWait         wait for STViz to close before continuing
 -Xforce-atn         use the ATN simulator for all predictions
 -Xlog               dump lots of logging info to antlr-timestamp.log
```

每次都要输入这么一长串命令显然很不方便，最好设定一个别名，在`.bashrc`和`.bash_profile`下设定：

```shell
$ alias antlr4='java -jar /usr/local/lib/antlr-4.12-complete.jar'
```

这样之后就可以直接使用`antlr4`命令来启动antlr了。

另外，由于我们主要是在C++下进行开发，因此可以添加一个新的别名缩写：

```shell
$ alias antlr4cpp='java -jar /usr/local/lib/antlr-4.12-complete.jar -Dlanguage=Cpp'
```

这样，之后使用`antlr4cpp`就可以直接生成C++的代码文件。

### Antlr4 cpp runtime

上面的antlr4是一个代码生成工具，它的作用是把语法文件生成我们所希望的用特定语言写成的词法分析器和语法分析器。但是所生成的词法和语法分析器仍然需要antlr4的cpp runtime工具来支持。简单地说，就是所生成的词法和语法分析工具引入了一系列的库和头文件，需要单独安装。

到antlr4的官网下载antlr4-cpp runtime：https://www.antlr.org/download.html，注意这个runtime需要单独编译。确保你的机器上可以使用`g++`和`cmake`命令。

```shell
$ ls
antlr4-cpp-runtime-4.12.0-source
$ mv antlr4-cpp-runtime-4.12.0-source antlr4-source-code
$ cd antlr4-source-code
antlr4-source-code$ ls
cmake  CMakeLists.txt  demo  deploy-macos.sh  deploy-source.sh  deploy-windows.cmd  dist  LICENSE.txt  README.md  runtime  VERSION
antlr4-source-code$ 
```

注意这里有一个`CMakeLists.txt`。我们使用cmake来构建即可。

```shell
antlr4-source-code$ cmake -B build && cmake --build build
```

或者：

```shell
antlr4-source-code$ mkdir build && cd build && cmake .. && make -j6
```

最后使用`sudo make install`即可把库和库文件安装到`/usr/local/`下了。

```shell
antlr4-source-code$ sudo make insall
```

安装好后，在`/usr/local/`下，可以看到`antlr`的runtime库，主要是`/usr/local/include/antlr4-runtime`和`/usr/local/lib/`下的antlr4的静态库和动态库。

最后需要额外记住的是，在​.bashrc​里面，务必要将它们加入到`include`的`library`的路径里面。

```bash
export CPLUS_INCLUDE_PATH=/usr/local/include/:$CPLUS_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=/usr/local/include/antlr4-runtime:$CPLUS_INCLUDE_PATH
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
```

然后，在写C++的时候，就可以通过`#include "antlr4-runtime.h"`来使用`antlr4`的runtime库了。

这里要注意一下，不能仅仅写`export CPLUS_INCLUDE_PATH=/usr/local/include/:$CPLUS_INCLUDE_PATH`，因为`antlr4`生成的那些词法与语法文件都是`#include "antlr4-runtime.h"`。

## Parser

对于在前面提到的例子而言，我们主要修改的是C代码，所以首先需要获取C语言的Parser。

Antlr4有自己的语法文件格式，以`.g4`为结尾。并且开源了大量的现成的很多语言的语法格式文件，在github上。

在github上可以找到C语言的语法文件`C.g4`，将其下载，然后就可以使用antlr4生成解析文件。

```shell
$ antlr4cpp C.g4 -vistor -listener
```

上面的这条命令会在当前目录下生成`.h`文件和`.cpp`文件，以及一些其它的辅助文件。

我们新创建一个`main.cc`，用来作为我们程序的入口。

在main.cc中写入下面的代码：

```cpp
#include "ANTLRInputStream.h"
#include "CommonTokenStream.h"
#include "CBaseListener.h"
#include "CBaseVisitor.h"
#include "CLexer.h"

#include <iostream>
#include <string_view>
#include <format>
#include <algorithm>
#include <set>

using namespace antlr4;

int main(int argc, char* argv[]) {
  std::string_view code = "float func() {retrun 1.0f * 2.0f + 3.0f;}";

  ANTLRInputStream input(code);

  CLexer lexer(&input);
  CommonTokenStream tokens(&lexer);
  tokens.fill();

  CParser parser(&tokens);
  tree::ParseTree *tree = parser.functionDefinition();

  //...
}
```

上面的代码非常好懂，`ANTLRInputStream input(code);`读取代码，然后进行词法和语法的分析，接着进行语法分析。

但是要稍微注意一下`tree::ParseTree *tree = parser.functionDefinition();`，这个代码中使用了`functionDefinition`这个成员函数，务必注意这个成员函数是从`C.g4`文件中找到的，因为我们输入的code是一个函数，看一下`C.g4`中对于函数定义的描述：

```plaintext
functionDefinition
    : declarationSpecifiers? declarator declarationList? compoundStatement
    ;
```

因为`C.g4`中有`functionDefinition`这个语法节点，因此在生成语法解析文件的时候，Parser类才会有这个成员函数。

当然`C.g4`中也描述了很多其它的语法节点，这个需要根据要解析的代码进行适当的选择。

## listener

Parser语法解析结束之后，我们的语法树就已经生成，Antlr4提供了两种模式来访问整个语法树，访问者模式和观察者模式。

这里先介绍访问者模式，在antlr4中，我们可以通过listener来自上而下访问语法树。

```cpp
class Listener : public CBaseListener {

  virtual void enterAdditiveExpression(CParser::AdditiveExpressionContext * ctx/*ctx*/) override {
    std::cout << "AdditiveExpresstion: " << ctx->getText() << std::endl;
  }
};
```

写好上面的Listener类之后，接下来就可以去用这个访问者去调用遍历语法树了：

```cpp
Listener listener;

tree::ParseTreeWalker::DEFAULT.walk(&listener, tree);
```

在遍历语法树时，当访问到Additive节点的时候，就会打印出节点所对应的内容：

```plaintext
AdditiveExpresstion: 1.0f * 2.0f + 3.0f
```

这里是一个简单的打印，但是从这个简单的例子当中，我们已经可以看出思路。

对于我们先前给出的例子来说，我们可以这样做，在进入到Additive节点的时候，看一下是否有多个Multiplicative节点，如果只有1个，那么不需要修改代码。如果有多个，那么看是否满足一个Multiplicative有两个CastExpression，而另一个Multiplicative只有一个CastExpress，这样的情况，如果有，那我们可以进行改写。

```cpp
std::vector<std::tuple<int, int, std::string>> substrs;

class Listener : public CBaseListener {

  virtual void enterAdditiveExpression(CParser::AdditiveExpressionContext * ctx/*ctx*/) override {
    // print it, if ctx has more than one multiplicativeExpression
    // Check the if it is mul + Num or Num + mul
    // std::cout << ctx->getText() << std::endl;
    auto mulExprs = ctx->multiplicativeExpression();
    if (mulExprs.size() < 2)
      return;

    auto *left = ctx->multiplicativeExpression(0);
    auto *right = ctx->multiplicativeExpression(1);

    const auto & leftCasts = left->castExpression();
    const auto & rightCasts = right->castExpression();

    if (leftCasts.size() == 1 && rightCasts.size() == 1) {
      return;
    }
    if (leftCasts.size() > 2 || rightCasts.size() > 2) {
      return;
    }
    if (leftCasts.size() == 2 && left->Star().empty()) {
      return;
    }
    if (rightCasts.size() == 2 && right->Star().empty()) {
      return;
    }

    // get ctx token Index
    int s = left->getStart()->getStartIndex();
    int e = right->getStop()->getStopIndex();

    // check additive is '+' or '-'
    bool isSub = ctx->Plus().empty();

    // rewrite a*b + c to fma(a, b, c)
    std::string a, b, c;

    std::string new_cod
    if (leftCasts.size() > 1 && rightCasts.size() == 1) {
      a = leftCasts.at(0)->getText(); 
      b = leftCasts.at(1)->getText();
      c = rightCasts.at(0)->getText();
      if (!isSub) {
        new_code = std::format("fma({}, {}, {})", a, b, c);
      } else {
        new_code = std::format("fma({}, {}, -{})", a, b, c);
      }
    }

    if (leftCasts.size() == 1 && rightCasts.size() > 1) {
      a = rightCasts.at(0)->getText();
      b = rightCasts.at(1)->getText();
      c = leftCasts.at(0)->getText();
      if (!isSub) {
        new_code = std::format("fma({}, {}, {})", a, b, c);
      } else {
        new_code = std::format("-fma({}, {}, -{})", a, b, c);
      }
    }

    substrs.push_back({s, e, new_code});
  }
};

```

我们用一个vector来保存修改，需要保存的是代码中，需要修改的位置和需要改成的代码。

这段代码不能完全解决问题，它没有类型上的考虑，也没有语句间的考虑，但已经足够展示出我们的思路。

再获取所有需要修改的地方之后，首先需要做一个排序，从后往前重新排，然后再把修改应用到源代码即可。

```cpp
// sort substrs by start index, from large to small
std::sort(substrs.begin(), substrs.end(), [](const auto &a, const auto &b) {
  return std::get<0>(a) > std::get<0>(b);
});

for(auto &[s, e, code_snippet] : substrs) {
  new_code.replace(s, e - s + 1, code_snippet);
}
```

