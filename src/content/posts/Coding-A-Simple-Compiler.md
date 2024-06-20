---
title: 用LLVM写一个简单的编译器
description: 用LLVM来写一个非常简单的编译器，只能用来处理简单的运算。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Java/1.jpg
category: 编程实践
published: 2023-09-12
tags: [C++, LLVM, Compiler]
---

## Overview

本篇博客中，我们用LLVM来实现一个极其简单的计算程序。它从标准输入中获取一个类似下面的表达式：

```
with a,b : a * (4+b)
```

然后通过标准输入，给定a,b的值，可以计算出结果。

用EBNF来描述上面的语法规则：

```
calc   : ("with" ident ("," ident)* ":")? expr;
expr   : term (("+"|"-") term)* ;
term   : factor (("*"|"/") factor)* ;
factor : ident | number | "(" expr ")" ;
ident  : ([a-zA-Z])+;
number : ([0-9])+ ;
```

> 这个number的设计仅仅是一种简化，在这个例子当中，number只能是整数，且不能是负数。负数都必须用$0-x$来表示。

我们可以用LLVM来实现词法分析，语法分析，中间代码生成，以及最终的代码生成。

## Lexical Analysis

```cpp
#ifndef LEXER_H
#define LEXER_H

#include "llvm/ADT/StringRef.h"
#include "llvm/Support/MemoryBuffer.h"

class Lexer;

class Token {

  friend class Lexer;

  public:
    // 定义Token，这里eoi是end of input的意思，代表输入的结束
    // unknown 用来作错误处理
    enum TokenKind : unsigned short {
        eoi, unknown, ident, number, comma, colon, plus,
        minus, star, slash, l_paren, r_paren, KW_with
    };
  
  private:
    // 对于每一个token而言，用一个kind来标记token的类型
    // 再用一个Text来指明这个token的文本内容
    // 这里的llvm::StringRef是一个C字符串的指针加上一个文本长度的组合
    TokenKind Kind;
    llvm::StringRef Text;
  
  public:
    // 注意这里代码在参数表后面有一个const，代表函数体内不对成员变量做修改
    TokenKind getKind() const {return Kind;}
    llvm::StringRef getText() const {return Text;}
    bool is(TokenKind K) const {return Kind == K; }
    bool isOneOf(TokenKind K1, TokenKind K2) const {
        return is(K1) || is(K2);
    }
    // 这里使用了可变参模板技巧
    template <typename... Ts>
    bool isOneOf(TokenKind K1, TokenKind K2, Ts... Ks) const {
        return is(K1) || isOneOf(K2, Ks...);  
    }
};

// 词法分析器
class Lexer {
  // 两个指针，用来扫描buffer
  const char * BufferStart;
  const char * BufferPtr;

  public:
    Lexer(const llvm::StringRef &Buffer) {
        BufferStart = Buffer.begin();
        BufferPtr = BufferStart;
    }
    // 实际上Lexer只需要一个next函数，用来返回下一个token
    void next(Token &token);

  private:
    // next函数会调用内部的formToken
    void formToken(Token &Result, const char *TokEnd, Token::TokenKind Kind);
};

#endif
```

然后是Lexer.cpp：

```cpp
#include "Lexer.h"

// 首先定义了三个用于辅助的函数
namespace charinfo {
  LLVM_READNONE inline bool isWhitespace(char c) {
    return c == ' '  || c == '\t' || c == '\f' ||
           c == '\v' || c == '\r' || c == '\n';
  }

  LLVM_READNONE inline bool isDigit(char c) {
    return c >= '0' && c <= '9';
  }

  LLVM_READNONE inline bool isLetter(char c) {
    return (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z');
  }
}

// 然后实现Lexer类的next函数，就是后向扫描
// 不过我们的例子比较简单，所以也没有什么状态机这种复杂概念
// 就是扫描连续的字符，连续的数字，以及单字符，然后返回token
void Lexer::next(Token &token) {
    while (*BufferPtr && charinfo::isWhitespace(*BufferPtr)) {
        ++BufferPtr;
    }
    if (!*BufferPtr){
        token.Kind = Token.eoi;
        return;
    }
    if (charinfo::isLetter(*BufferPtr)){
        const char *end = BufferPtr+1;
        while (charinfo::isLetter(*end)) ++end;
        llvm::StringRef Name(BufferPtr, end - BufferPtr);
        Token::TokenKind kind = Name == "with" ? Token::KW_with : Token::ident;
        formToken(token, end, kind);
        return ;
    } else if (charinfo::isDigit(*BufferPtr)){
        const char *end = BufferPtr+1;
        while (charinfo::isDigit(*end)) ++end;
        formToken(token, end, Token::number);
        return ;
    } else {
    // 注意这里用了一个宏技巧
    // 这种技巧可以让我们更快地知道多个case的含义。
    // 另外，这里的define是写在代码内的，最后直接undef掉，也避免了写在别处造成的阅读困难。
        switch (*BufferPtr) {
            #define CASE(ch, tok) case ch: formToken(token, BufferPtr+1, tok); break
            CASE('+', Token::plus);
            CASE('-', Token::minus);
            CASE('*', Token::star);
            CASE('/', Token::slash);
            CASE('(', Token::l_paren);
            CASE(')', Token::r_paren);
            CASE(':', Token::colon);
            CASE(';', Token::comma);
            #undef CASE
            default:
                formToken(token, BufferPtr+1, Token::unknown);
        }
    }
}
// 最后是一个formoken的实现。
void Lexer::formToken(Token &Tok, const char * TokEnd, Token::TokenKind Kind) {
    Tok.Kind = Kind;
    Tok.Text = llvm::StringRef(BufferPtr, TokEnd - BufferPtr);
    BufferPtr = TokEnd;
}
```

然后是Parser.h，整体的思路还是递归下降。

```cpp
#ifndef PARSER_H
#define PARSER_H

#include "AST.h"
#include "Lexer.h"
#include "llvm/Support/raw_ostream.h"
// 注意这里，llvm项目规范中是禁止使用<iostream>的
// 需要输入输出的时候，需要使用llvm自带的输入输出功能

class Parser {
private:
  Lexer &Lex;
  Token Tok;
  bool HasError;
  // 用于错误处理
  void error() {
      llvm::errs() << "Unexpected: " << Tok.getText() << "\n";
      HasError = true;
  }
  // 读取下一个token
  void advance() {
      Lex.next(Tok);
  }
  // 看一下当前读取的Token是否是所期望的类型
  bool expect (Token::TokenKind Kind) {
      if (Tok.getKind() != Kind) {
          error();
          return false;
      }
      return true;
  }
  // consume的意思是，如果符合期望，就前进一个token。
  // 当前的token如果符合期望但是又不必加入到AST中就会使用了。
  bool consume(Token::TokenKind Kind) {
      if (!expect(Kind)) return false;
      advance();
      return true;
  }

  // AST的具体实现在后面，这里仅仅是一个声明
  AST*  parseCalc();
  Expr* parseExpr();
  Expr* parseTerm();
  Expr* parseFactor();

public:
  explicit Parser(Lexer &Lex) : Lex(lex), HasError(false) {advance();}
  bool hasError() {return HasError; }
  AST* parse();
};

#endif
```

和Parser.cpp

```cpp
#include "Parser.h"

// 总体的思路还是递归下降
AST* Parser::parse() {
    AST* Res = parseCalc();
    expect(Token::eoi);
    return Res;
}

AST* Parser::parseCalc() {
    llvm::SmallVector<llvm::StringRef, 8> Vars;
    if (Tok.is(Token::KW_with)) {
        do {
            advance();
            if (!expect(Token::ident)) goto _error;
            Vars.push_back(Tok.getText());
            advance();
        } while(Tok.is(Token::comma));
        if (consume(Token::colon)) goto _error;
    }
    Expr * E = parseExpr();
    if (Vars.empty()) return E;
    else return new WithDecl(Vars, E);

_error:
  while(!Tok.is(Token::eoi)) {
    advance();
    return nullptr;
  }
}

// Expr -> Term [+|-] Term
Expr* Parser::parseExpr() {
    Expr* Left = parseTerm();
    while (Tok.isOneOf(Token::plus, Token::minus)) {
        BinaryOp::Operator Op =  
           Tok.is(Token::plus) ? BinaryOp::Plus : 
                                 BinaryOp::Minus;
        advance();
        Expr* Right = parseTerm();
        Left = new BinaryOp(Op, Left, Right);
    }
    return Left;
}

// Term -> Factor [*|/] Factor
Expr* Parser::parseTerm() {
    Expr* Left = parseFactor();
    while (Token.isOneOf(Token::star, Token::slash)) {
        BinaryOp::Operator Op =  
           Tok.is(Token::star) ? BinaryOp::star : 
                                 BinaryOp::Div;
        advance();
        Expr* Right = parseFactor();
        Left = new BinaryOp(Op, Left, Right);
    }
    return Left;
}

// Factor -> num | ident | (Expr) 
Expr* Parser::parseFactor() {
    Expr* Res = nullptr;
    switch(Tok.getKind()) {
        case Token::number:
            Res = new Factor(Factor::Number, Tok.getText());
            advance(); break;
        case Token::ident:
            Res = new Factor(Factor::Ident, Tok.getText());
            advance(); break;
        case Token::l_paren:
            advance();
            Res = parseExpr();
            if (consume(Token::r_paren)) break;
        default:
            if (!Res) error();
            while (!Tok.isOneOf(Token::r_paren, Token::star,
                                Token::plus, Token::minus,
                                Token::slash, Token::eoi)){
                advance();
            }  
    }
    return Res;  // 这个return原先在switch里面，这里给提到外面去了，不过语义上无影响。
}
```

完成语法分析器之后，紧接着来设计语法树：这里面与前面设计的语法会稍有差别，表达式不管是乘除还是加减都是用一个`BinaryOp`来管理。`Expr`仅仅是一个父类，`Factor`继承`Expr`用来表示单节点树，`BinaryOp`继承`Expr`用来表示双节点树。

```cpp
#ifndef AST_H
#define AST_H

#include "llvm/ADT/SmallVector.h"
#include "llvm/ADT/StringRef.h"

class AST;
class Expr;
class Factor;
class BinaryOp;
class WithDecl;

// ASTVisitor用于后面的语义分析，这是一个纯虚类
class ASTVisitor {
public:
  virtual void visit(AST    &) {};
  virtual void visit(Expr   &) {};
  virtual void visit(Factor &) = 0;
  virtual void visit(BinaryOp &) = 0;
  virtual void visit(WithDecl &) = 0;
} ;

// AST父类，必须实现一个accept函数，用来进行语义检查
// 内部用一个accept函数，来召唤一个ASTVisitor来对传入的AST进行语义检查
// 个人认为这个accept函数的设计非常莫名其妙。直接将ASTVisitor作为友元类
// 不久行了？
class AST {
public:
  virtual ~AST() {} ;
  virtual void accept(ASTVisitor &V) = 0;
} ;

// Expr父类，这个类虽然自身没有纯虚函数，但是它没有实现父类的纯虚函数
// 因而这个函数也是不能实例化的
class Expr : public AST {
public:
  Expr() {}
} ;

// Factor继承Expr，用来表示AST的叶节点
class Factor : public Expr {
public:
  enum ValueKind { Ident, Number} ;
private:
  ValueKind Kind;
  llvm::StringRef Val;
public:
  Factor(ValueKind Kind, llvm::StringRef Val) :Kind{Kind}, Val(Val) {};
  ValueKind getKind() {return Kind; }
  llvm::StringRef getVal() {return Val; }
  virtual void accept (ASTVisitor &V) override { V.visit(*this) ;} 
}

// BinaryOp继承Expr，用来表示AST的非叶非根节点
class BinaryOp : public Expr {
public:
  enum Operator {Plus, Minus, Mul, Div } ;

private:
  Expr *Left;
  Expr *Right;
  Operator Op;

public:
  BinaryOp(Operator Op, Expr *L, Expr *R) : Op (Op), Left(L), Right(R) {};
  Expr *getLeft() {return Left; }
  Expr *getRight(){return Right;}
  Operator getOperator() {return Op;}
  virtual void accpet(ASTVisitor &V) override {V.visit(*this) ; }
} ;

// WithDecl作为整个AST的根节点，注意它继承的是AST
// 有两个成员变量，一是Vars，代表声明的变量，二就是E，也就是整个的表达式
// 注意这里它使用的是llvm里面的数组，而并非STL里面的array或者vector。
class WithDecl : public AST {
private:
  using VarVector = llvm::SmallVector<llvm::StringRef, 8>;
  VarVector Vars;
  Expr * E;
public:
  // 这里面还是用的llvm:SmallVector<llvm::StringRef, 8>而并非VarVector，个人认为是为了可读性？
  WithDecl(llvm:SmallVector<llvm::StringRef, 8> Vars, Expr *E) : Vars(Vars), E(E) {}
  // 由于Vars是私有成员变量，为了使用其迭代器，因而声明两个迭代器函数
  // 不过，个人觉得使用cbegin和cend比较好。最好再写成一个const函数。
  VarVector::const_iterator begin() { return Vars.begin(); }
  VarVector::const_iterator end()   { return Vars.end(); }
  // 获取表达式
  Expr *getExpr() { return E; }
  virtual void accept(ASTVisitor &V) override { V.visit(*this) ; }
};

#endif
```

AST设计完成后，接着进行语义分析（Sematic analysis），calc的语义分析很简单，就是两条，检查重复声明的变量，和检查表达式中未定义的变量。

Sema.h，里面只有一个Sema类，内置一个Semantic函数。

```cpp
#ifndef SEMA_H
#define SEMA_H

#include "AST.h"
#include "Lexer.h"

class Sema {
public:
  bool semantic(AST *Tree);
};

#endif
```

然后是Sema.cpp

```cpp
#include "Sema.h"
#include "llvm/ADT/StringSet.h"
// 这里把外面的namespace给去掉了
// DeclCheck会首先对变量表进行遍历，试图找出声明多次的变量
// 然后会遍历AST，试图找出未声明的变量
class DeclCheck : public ASTVistor {
public:
  llvm::StringSet<> Scope;
  bool HasError;
  
  // 两种错误类型，一是变量声明了两次，而是变量从未声明
  enum ErrorType {Twice, Not} ;
  
  void error (ErrorType ET, llvm::StringRef V) {
    llvm::errs() << "Variable " << V << " "
                 << (ET == Twice ? "already" : "not") 
                 << " declared!\n"
    HasError = true;
  }
public:
  DeclCheck() : HasError(false) {} 

  bool hasError() {  return HasError; }

  // 对Factor进行检查，寻找未定义的变量
  virtual void visit (Factor &Node) override {
    if (Node.getKind() == Factor::Ident) {
        // Scopde是一个StringSet，其实用contains成员函数可能更好
        if (Scope.find(Node.getVal()) == Scope.end()) {
            error(Not, Node.getVal()) ;
        }
    }
  }
  // 对BinaryOp进行检查，递归检查左右子树
  virtual void visit(BinaryOp &Node) override {
    if (Node.getLeft()) {
        Node.getLeft()->accept(*this);  // 为什么我觉得这个accept的设计不好，就是因为这个实际上是一个visit
    } else {                            // 它在这里面进行了一个递归，但是直接看代码却可能看不出来
        HasError = true;
    }
    if (Node.getRight()) {
        Node.getRight()->accept(*this);
    }else {
        HasError = true;
    }
  }

  // 然后是对WithDecl这个根节点的遍历，一是遍历变量表，找出重复声明的变量
  // 二是遍历AST
  virtual void visit(WithDecl &Node) override {
    // 这里面它采取的技巧是，用一个空的StringSet，然后遍历变量表，将变量加入到这个stringset中
    // 这个insert函数会返回一个pair，pair的第一个元素是一个迭代器，第二个元素是一个bool，
    // 代表是否顺利插入元素，如果set中已经有这个元素了，那么插入会失败，然后返回false。
    // 而一旦返回false，我们也就知道这里面是有重复声明的问题的。
    for(auto I = Node.begin(), E = Node.end(); I != E; ++I) {
        if (!Scope.insert(*I).second) {
            error(Twice, *I);
        }
    }
    // 然后就是一个递归
    if (Node.getExpr()) {
        Node.getExpr()->accept(*this);
    } else {
        HasError = true;
    }
  } 
};

// 最后是一个语义分析的入口函数
// 就是创建一个DeclCheck类
// 然后让AST去accpet
// 然后这个DeclCheck类再调用自己的visit函数去进行语义检查。
bool Sema::semantic (AST *Tree) {
  if (!Tree) {
    return false;
  }
  DeclCheck Check;
  Tree->accept(Check);
  return Check.hasError();
}
```

## LLVM IR

在语义分析完成之后，接下来的一个任务就是从AST去生成LLVM IR。先来简单介绍一下LLVM IR。

我们这里把这一章最后使用命令`calc "with a: a*3"`获取的IR来分析一下。

```llvm
declare i32 @calc_read(i8*)
declare void @calc_write(i32)
```

我们最后会生成一个可执行文件，但是仅仅用calc只会生成用于计算的，类似可重定位目标文件的llvm IR。如果我们还需要执行输出，就不能仅仅依靠calc。之后我们会在一个C文件当中写`calc_read`和`calc_write`两个函数，将这个C文件用编译成可重定位目标文件，再将其与calc生成的可重定位目标文件进行链接。

（关于可重定位目标文件：((20220302151824-fo1ixi2 '2. How does the linker work?'))）

这里的两行declare意思非常清晰，`i32`就是int，`@`符号可能标识这是一个具名元素（就是在源代码中出现的标识符，相对的概念是临时元素，指在源代码中未出现的，由编译器产生的标识符），`i8`是8位有符号整数，在C语言里面就是`char`。

```llvm
@a.str = private constant [2 x i8] c"a\00"
```

这个要结合着后面生成这段IR的命令来看，后面生成这段IR的命令是：

```shell
$./calc "with a:a*3"
```

这里面它会把`a`首先是当成一个字符串，`a`是一个字符，字符串必须是0结尾，所以还需要一个0字符，加在一起就是两个字符，这就是`[2 x i8]`的由来，`a.str`代表`a`是一个字符串，`"a\00`就是字符串内容了。

```llvm
define i32 @main(i32, i8**) {
```

定义main函数，类似于C语言中对函数的定义：

```llvm
entry:
```

IR是以基本块（Basic Block）组织起来的，每一个基本块都要有一个label。对于入口基本块，要设定`entry`。

```llvm
%2 = call i32 @calc_read(
    i8* getelementptr inbounds ([2 x i8], [2 x i8]* @a.str, i32 0, i32 0)
)
```

调用`calc_read`，将读取符号`a`的值，注意这不是把`a`这个字符串转换成数字。

```llvm
%3 = mul nsw i32 %2, 3
```

进行计算。

```llvm
  call void @calc_write(i32 %3)
  ret i32 0
```

然后把计算出来的值进行输入，而后返回。

## Generating the IR from the AST

CodeGen.h

```cpp
#ifndef CODEGEN_H
#define CODEGEN_H

#include "AST.h"

class CodeGen {
public:
 void compile(AST *Tree);
};

#endif
```

CodeGen.cpp

```cpp
#include "CodeGen.h"
#include "llvm/ADT/StrringMap.h"
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

namespace {
class ToIRVisitor : public ASTVisitor {
private:
  // 以下其实都是llvm::的前缀，但是因为使用了using namespace llvm
  // 因而就不需要再写这个前缀了
  // 这一大堆很多将来继续解释，主要的疑惑点在于Module
  Module *M;
  IRBuilder<> Builder;
  Type *VoidTy;
  Type *Int32Ty;
  Type *Int8PtrTy;       // char*，字符串
  Type *Int8PtrPtrTy;    // 实际上就是char**，给main函数用的
  constant *Int32Zero;
  Value *V;
  StringMap<Value*> nameNap;
public:
  // 一个构造函数，将上面全部初始化
  ToIRVisitor (Module *M) : M(M), Builder(M->getContext()) {
    VoidTy = Type::getVoidTy(M->getContext());
    Int32Ty = Type::getInt32Ty(M->getContext());
    Int8PtrTy = Type::getInt8PtrTy(M->getContext());
    Int8PtrPtrTy = Int8PtrTy->getPointerTo();
    Int32Zero = ConstantInt::get(Int32Ty, 0, true);
  }

  void run(AST *Tree) {
    // 首先生成一个main函数
    // MainFty是main函数的声明，第一个参数是返回值，这里是Int32Ty
    // 第二个参数接收一个列表，两个类型是Int32Ty和Int8PtrPtrTy，也就是int和char**
    FunctionType *MainFty = FunctionType::get(Int32Ty, {Int32Ty, Int8PtrPtrTy}, false);
    // 然后是Main函数的定义
    Function *MainFn = Function::Create(MainFty, GlobalValue::ExternalLinkage, "main", M);
    // 之后创建基本块BB，首先是entry基本块
    BasicBlock *BB = BasicBlock::Create(M->getContext(), "entry", MainFn);
    // IRbuilder去加载这个BB
    Builder.SetInsertPoint(BB);
    // 这个accept，内部会调用这个类的visit去生成IR，不是用ASTVisitor去生成IR的。
    // 会生成一系列的IR，详情见下面的代码。
    Tree->accept(*this);
    // 
    FunctionType *CalcWriteFnTy = FunctionType::get(VoidTy, {Int32Ty}, false);
    Function *CalcWriteFn = Function::Create(CalcWriteFnTy, GlobalValue::ExternalLinkage, "calc_write", M);
    Builder.CreateCall(CalcWriteFnTy, CalcWriteFn, {V});
    Builder.CreateRet(Int32Zero);
  }

  virtual void visit(WithDecl &Node) override {
    // 对于with，这是根节点，要先去调用calc_read
    // 但是对于calc_read，只需要有声明就可以了，它的定义是在后面用C语言去生成。
    FunctionType *ReadFty = 
      FunctionType::get(Int32Ty, {Int8PtrTy}, false);
    // 定义ReadFn，但是不会往里面加入BB，而是将其标注为external linkage，代表从外面来。
    Function *ReadFn = Function::Create (ReadFty, GlobalValue::ExternalLinkage, "calc_read", M);
    // 这里的遍历是针对WithDecl的遍历，是遍历符号表。
    for (auto I = Node.begin(), E = Node.end(); I != E; ++I) {
      // 获取变量的字符串形式
      StringRef Var = *I;
      Constant *StrText = ConstantDataArray::getString(M->getContext(), Var);
      // 声明全局变量
      GlobalVariable *Str = new GlobalVariable (
          *M, StrText->getType(), 
          /*isConstant=*/ true, 
          GlobalValue::PrivateLinkage, 
          StrText, Twine(Var).concat(.str)
      );
      // 这里生成一条IR，就是前面@a.str ... 的由来。
      Value *Ptr = Builder.CreateInBoundsGEP(Str, {Int32Zero, Int32Zero}, "Ptr");
      // 再次生成一条IR，是调用ReadFn的IR。
      CallInst *Call = Builder.CreateCall(ReadFty, ReadFn, {Ptr});
      nameMap[Var] = Call;
    }
    // 结束之后，再去遍历表达式部分。
    Node.getExpr()->accept(*this);
  }
  // 遍历到AST的叶节点时，仅仅是替换一个V，主要是为了下面的函数服务
  virtual void visit(Factor &Node) override {
    if (Node.getKind() == Factor::Ident) {
      V = nameMap[Node.getVal()];
    } else {
      int intval;
      Node.getVal().getAsInteger(10, intval);
      V = ConstantInt::get(Int32Ty, intval, true);
    }
  }
  // 意思很简单，不详述了
  virtual void visit(BinaryOp &Node) override {
    Node.getLeft()->accept(*this);
    Value *Left = V;
    Node.getRight()->accept(*this);
    Value *Right = V;
    switch (Node.getOperator() {
      case BinaryOp::Plus:
        V = Builder.CreateNSWAdd(Left, Right); break;
      case BinaryOp::Minus:
        V = Builder.CreateNSWSub(Left, Right); break;
      case BinaryOp::Mul:
        V = Builder.CreateNSWMul(Left, Right); break;
      case BinaryOp::Div:
        V = Builder.CreateSDiv(Left, Right); break;
    }
  }
  // 最后一个comile的实现
  void CodeGen::compile(AST *Tree) {
    LLVMContext Ctx;
    Module *M = new Module("clac.expr", Ctx);
    ToIRVisitor ToIR(M);
    ToIR.run(Tree);
    // 这里是进行了一个打印，而不是去生成一个IR的文件
    M->print(outs(), nullptr);
  }
};

}
```

## Driver and runtime

主体代码就已经完成了，接下来还需要驱动代码。对于一般的编译器而言，它需要能够读取一个文件。我们这里不需要读取一个文件，只需要读取命令行即可。

LLVM里面有专门的用于读取命令行的库。

Calc.cpp

```cpp
#include "CodeGen.h"
#include "Parser.h"
#include "Sema.h"
#include "llvm/Support/CommandLine.h"
#include "llvm/Support/InitLLVM.h"
#include "llvm/Support/raw_ostream.h"

// 这里是调用了一个构造函数，实例化了一个llvm::cl::opt的类，名称是Input
// cl应该是command line的意思
// opt可能是option
static llvm::cl::opt<std::string>
    Input(llvm::cl::Positional,
          llvm::cl::desc("<input expression>"),
          llvm::cl::init(""));
// 整个项目的main函数
int main(int argc, const char **argv) {
  llvm::InitLLVM X(argc, argv);
  // 解析命令行参数
  // 这里面第三个参数其实可以不填，是一个帮助信息
  llvm::cl::ParseCommandLineOptions(
      argc, argv, "calc - the expression compiler\n");
  // 词法分析和语法分析，目的是生成AST
  Lexer Lex(Input);
  Parser Parser(Lex);
  AST *Tree = Parser.parse();
  if (!Tree || Parser.hasError()) {
    llvm::errs() << "Syntax errors occured\n";
    return 1;
  }
  Sema Semantic;
  if (Semantic.semantic(Tree)) {
    llvm::errs() << "Semantic errors occured\n";
    return 1;
  }
  CodeGen CodeGenerator;
  // 进行编译，如果去运行这个calc的可执行文件，那么会打印出llvm的IR信息。
  CodeGenerator.compile(Tree);
  return 0;
}
```

## Calc_read 与 Calc_write

这里面我们调用了calc_read和calc_write两个函数，但是两个函数我们这里没法去单独实现，只能用C语言写，然后调用clang，生成可重定位目标文件。在calc生成IR后，用llc再编译成可重定位目标文件。最后用clang将两个可重定位目标文件进行链接生成可执行文件。

两个函数用C语言写，比较简单：

命名为rtcalc.c，注意这个不需要跟上面那些源代码一起编译，只需要用clang生成relocatable。

```c
#include <stdio.h>
#include <stdlib.h>

void calc_write(int v) {
    printf("The result is %d\n", v);
}


int calc_read(char *s)
{
  char buf[64];
  int val;
  printf("Enter a value for %s: ", s);
  fgets(buf, sizeof(buf), stdin);
  if (EOF == sscanf(buf, "%d", &val))
  {
    printf("Value %s is invalid\n", buf);
    exit(1);
  }
  return val;
}
```

## 组织

在进入到构建过程之前，先来看一下整个项目的组织结构：

```plaintext
.
├── build.sh
├── CMakeLists.txt
├── rtcalc.c
└── src
    ├── AST.h
    ├── Calc.cpp
    ├── CMakeLists.txt
    ├── CodeGen.cpp
    ├── CodeGen.h
    ├── Lexer.cpp
    ├── Lexer.h
    ├── Parser.cpp
    ├── Parser.h
    ├── Sema.cpp
    └── Sema.h

```

## 构建

直接使用CMake搭配build.sh就好。

calc的主目录下的CMakeLists.txt

```cmake
cmake_minimum_required (VERSION 3.8)

project ("calc")

find_package(LLVM REQUIRED CONFIG)
message("Found LLVM ${LLVM_PACKAGE_VERSION}, build type ${LLVM_BUILD_TYPE}")
list(APPEND CMAKE_MODULE_PATH ${LLVM_DIR})
include(DetermineGCCCompatible)
include(ChooseMSVCCRT)

add_definitions(${LLVM_DEFINITIONS})
include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
llvm_map_components_to_libnames(llvm_libs Core)

if(LLVM_COMPILER_IS_GCC_COMPATIBLE)
  if(NOT LLVM_ENABLE_RTTI)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
  endif()
  if(NOT LLVM_ENABLE_EH)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions")
  endif()
endif()

add_subdirectory ("src")
```

src下的CMakeLists.txt:

```cmake
add_executable (calc
  Calc.cpp
  CodeGen.cpp
  Lexer.cpp
  Parser.cpp
  Sema.cpp
  )
target_link_libraries(calc PRIVATE ${llvm_libs})
```

### build.sh

```bash
#!/bin/bash

BUILD_PATH="build"

if [ ! -d "$BUILD_PATH" ]; then
    mkdir "$BUILD_PATH"
fi

pushd ${BUILD_PATH}
rm -rf *
echo " --- building calc--- "

cmake -G Ninja -DCMAKE_CXX_STANDARD=14 \
               -DCMAKE_BUILD_TYPE=Release \
               -DLLVM_DIR=/data/Tinylang/llvm-12/lib/cmake/llvm \
               -DCMAKE_INSTALL_PREFIX=../calc ..

ninja
```

然后在shell里面进行build即可。

```shell
$./build.sh
```

## 运行

在build目录下的src目录中，可以看到calc可执行文件：

如果直接使用calc：

```shell
$calc "with a:a*3"
; ModuleID = 'calc.expr'
source_filename = "calc.expr"

@a.str = private constant [2 x i8] c"a\00"

define i32 @main(i32 %0, i8** %1) {
entry:
  %2 = call i32 @calc_read(i8* getelementptr inbounds ([2 x i8], [2 x i8]* @a.str, i32 0, i32 0))
  %3 = mul nsw i32 %2, 3
  call void @calc_write(i32 %3)
  ret i32 0
}

declare i32 @calc_read(i8*)

declare void @calc_write(i32)
```

会打印出IR，可以把这些IR通过llvm进一步去生成可重定位目标文件

```shell
$./calc "with a:a*3" | llc -filetype=obj -o=expr.o
```

这样就会把IR生成可重定位目标文件。

由于里面用到calc_read和calc_write，因此需要把rtcalc.c也生成可重定位目标文件，然后进行链接，这里直接使用clang（gcc也ok）即可：

```shell
$clang -c rtcalc.c -o rtcalc.o
```

（稍微注意一下rtcalc.c，它的位置可能不在build\src下，可能需要调整）这里是把rtcalc.c生成可重定位目标文件了。

然后再去生成可执行文件：

```shell
$clang expr.o rtcalc.o -o expr
```

来运行一下这个expr：

```shell
$./expr
Enter a value for a: 17
The result is 51
```
