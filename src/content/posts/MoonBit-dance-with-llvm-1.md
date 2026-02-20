---
title: MoonBit与LLVM共舞（上）- 实现语法前端
description: 这个系列的文章来教你使用MoonBit来实现一个自己的编译器，这是第一部分，我们使用MoonBit来实现语法解析工具。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/3.jpg
category: 编程实践
published: 2025-07-11
tags: [MoonBit, Compiler]
---

## 引言

编程语言设计与编译器实现历来被视为计算机科学领域中最具挑战性的课题之一。传统的编译器教学路径往往要求学生首先掌握复杂的理论基础：

- **自动机理论**：有限状态自动机与正则表达式
- **类型理论**：λ演算与类型系统的数学基础
- **计算机体系结构**：从汇编语言到机器码的底层实现

然而，Moonbit作为一门专为现代开发环境设计的函数式编程语言，为我们提供了一个全新的视角。它不仅具备严谨的类型系统和卓越的内存安全保障，更重要的是，其丰富的语法特性和为AI时代量身定制的工具链，使得Moonbit成为学习和实现编译器的理想选择。

> **系列概述**
> 本系列文章将通过构建一个名为**TinyMoonbit**的小型编程语言编译器，深入探讨现代编译器实现的核心概念和最佳实践。
>
> - **上篇**：聚焦语言前端的实现，包括词法分析、语法解析和类型检查，最终生成带有完整类型标记的抽象语法树
> - **下篇**：深入代码生成阶段，利用Moonbit官方的`llvm.mbt`绑定库，将语法树转换为LLVM中间表示，并最终生成RISC-V汇编代码

---

## TinyMoonbit 语言设计

TinyMoonbit是一种系统级编程语言，其抽象层次与C语言相当。虽然在语法设计上大量借鉴了Moonbit的特性，但TinyMoonbit实际并非Moonbit语言的子集，而是一个为测试`llvm.mbt`功能完备性兼具教学作用的简化版本。

> 注：由于篇幅限制，本系列文章所提到的TinyMoonbit实现比真正的TinyMoonbit更加简单，完整版本请参考 [TinyMoonbitLLVM](https://github.com/Kaida-Amethyst/TinyMoonbitLLVM)。

### 核心特性

TinyMoonbit提供了现代系统编程所需的基础功能：

- ✅ **底层内存操作**：直接的指针操作和内存管理
- ✅ **控制流结构**：条件分支、循环和函数调用
- ✅ **类型安全**：静态类型检查和明确的类型声明
- ❌ **简化设计**：为降低实现复杂度，不支持类型推导和闭包等高级特性

### 语法示例

让我们通过一个经典的斐波那契数列实现来展示TinyMoonbit的语法：

```moonbit
extern fn print_int(x : Int) -> Unit;

// 递归实现斐波那契数列
fn fib(n : Int) -> Int {
  if n <= 1 {
    return n;
  }
  return fib(n - 1) + fib(n - 2);
}

fn main() -> Unit {
  print_int(fib(10));
}
```

### 编译目标

经过完整的编译流程后，上述代码将生成如下的LLVM中间表示：

```llvm
; ModuleID = 'tinymoonbit'
source_filename = "tinymoonbit"

define i32 @fib(i32 %0) {
entry:
  %1 = alloca i32, align 4
  store i32 %0, ptr %1, align 4
  %2 = load i32, ptr %1, align 4
  %3 = icmp sle i32 %2, 1
  br i1 %3, label %4, label %6

4:                                                ; preds = %entry
  %5 = load i32, ptr %1, align 4
  ret i32 %5

6:                                                ; preds = %4, %entry
  %7 = load i32, ptr %1, align 4
  %8 = sub i32 %7, 1
  %9 = call i32 @fib(i32 %8)
  %10 = load i32, ptr %1, align 4
  %11 = sub i32 %10, 2
  %12 = call i32 @fib(i32 %11)
  %13 = add i32 %9, %12
  ret i32 %13
}

define void @main() {
entry:
  %0 = call i32 @fib(i32 10)
  call void @print_int(i32 %0)
}

declare void @print_int(i32 %0)
```

---

## 第二章：词法分析

**词法分析**（Lexical Analysis）构成了编译过程的第一道关卡，其核心使命是将连续的字符流转换为具有语义意义的**词法单元**（Tokens）序列。这个看似简单的转换过程，实际上是整个编译器流水线的基石。

### 从字符到符号：Token的设计与实现

考虑以下代码片段：

```moonbit
let x : Int = 5;
```

经过词法分析器处理后，将产生如下的Token序列：

```
(Keyword "let") → (Identifier "x") → (Symbol ":") → 
(Type "Int") → (Operator "=") → (IntLiteral 5) → (Symbol ";")
```

这个转换过程需要处理多种复杂情况：

1. **空白符过滤**：跳过空格、制表符和换行符
2. **关键字识别**：区分保留字与用户定义标识符
3. **数值解析**：正确识别整数、浮点数的边界
4. **运算符处理**：区分单字符和多字符运算符

### Token类型系统设计

基于TinyMoonbit的语法规范，我们将所有可能的符号分类为以下Token类型：

```moonbit
pub enum Token {
  Bool(Bool)       // 布尔值：true, false
  Int(Int)         // 整数：1, 2, 3, ...
  Double(Double)   // 浮点数：1.0, 2.5, 3.14, ...
  Keyword(String)  // 保留字：let, if, while, fn, return
  Upper(String)    // 类型标识符：首字母大写，如 Int, Double, Bool
  Lower(String)    // 变量标识符：首字母小写，如 x, y, result
  Symbol(String)   // 运算符和标点：+, -, *, :, ;, ->
  Bracket(Char)    // 括号类：(, ), [, ], {, }
  EOF              // 文件结束标记
} derive(Show, Eq)
```

### 利用模式匹配

Moonbit的强大模式匹配能力使我们能够以一种前所未有的优雅方式实现词法分析器。与传统的有限状态自动机方法相比，这种基于模式匹配的实现更加直观和易于理解。

#### 核心分析函数

```moonbit
pub fn lex(code: String) -> Array[Token] {
  let tokens = Array::new()
  
  loop code[:] {
    // 跳过空白字符
    [' ' | '\n' | '\r' | '\t', ..rest] => 
      continue rest

    // 处理单行注释
    [.."//", ..rest] =>
      continue loop rest {
        ['\n' | '\r', ..rest_str] => break rest_str
        [_, ..rest_str] => continue rest_str
        [] as rest_str => break rest_str
      }
    
    // 识别多字符运算符（顺序很重要！）
    [.."->", ..rest] => { tokens.push(Symbol("->")); continue rest }
    [.."==", ..rest] => { tokens.push(Symbol("==")); continue rest }
    [.."!=", ..rest] => { tokens.push(Symbol("!=")); continue rest }
    [.."<=", ..rest] => { tokens.push(Symbol("<=")); continue rest }
    [..">=", ..rest] => { tokens.push(Symbol(">=")); continue rest }
    
    // 识别单字符运算符和标点符号
    [':' | '.' | ',' | ';' | '+' | '-' | '*' | 
     '/' | '%' | '>' | '<' | '=' as c, ..rest] => {
      tokens.push(Symbol("\{c}"))
      continue rest
    }
    
    // 识别括号
    ['(' | ')' | '[' | ']' | '{' | '}' as c, ..rest] => {
      tokens.push(Bracket(c))
      continue rest
    }
    
    // 识别标识符和字面量
    ['a'..='z', ..] as code => {
      let (tok, rest) = lex_ident(code);
      tokens.push(tok)
      continue rest
    }

    ['A'..='Z', ..] => { ... }
    ['0'..='9', ..] => { ... }
    
    // 到达文件末尾
    [] => { tokens.push(EOF); break tokens }
  }
}
```

#### 关键字识别策略

标识符解析需要特别处理关键字的识别：

```moonbit
pub fn let_ident(rest: @string.View) -> (Token, @string.View) {
  // 预定义关键字映射表
  let keyword_map = Map.from_array([
    ("let", Token::Keyword("let")),
    ("fn", Token::Keyword("fn")),
    ("if", Token::Keyword("if")),
    ("else", Token::Keyword("else")),
    ("while", Token::Keyword("while")),
    ("return", Token::Keyword("return")),
    ("extern", Token::Keyword("extern")),
    ("true", Token::Bool(true)),
    ("false", Token::Bool(false)),
  ])
  
  let identifier_chars = Array::new()
  let remaining = loop rest {
    ['a'..='z' | 'A'..='Z' | '0'..='9' | '_' as c, ..rest_str] => {
      identifier_chars.push(c)
      continue rest_str
    }
    _ as rest_str => break rest_str
  }
  
  let ident = String::from_array(identifier_chars)
  let token = keyword_map.get(identifier).or_else(() => Token::Lower(ident))
  
  (token, remaining)
}
```

### 💡 Moonbit语法特性深度解析

上述词法分析器的实现充分展示了Moonbit在编译器开发中的几个突出优势：

1. 函数式循环构造

```moonbit
loop initial_value {
  pattern1 => continue new_value1
  pattern2 => continue new_value2
  pattern3 => break final_value
}
```

`loop`并非传统意义上的循环结构，而是一种**函数式循环**：

- 接受一个初始参数作为循环状态
- 通过模式匹配处理不同情况
- `continue`传递新状态到下一次迭代
- `break`终止循环并返回最终值

2. 字符串视图与模式匹配

Moonbit的字符串模式匹配功能极大简化了文本处理：

```moonbit
// 匹配单个字符
['a', ..rest] => // 以字符'a'开头

// 匹配字符范围  
['a'..='z' as c, ..rest] => // 小写字母，绑定到变量c

// 匹配字符串字面量
[.."hello", ..rest] => // 等价于 ['h','e','l','l','o', ..rest]

// 匹配多个可能的字符
[' ' | '\t' | '\n', ..rest] => // 任意空白字符
```

3. 模式匹配优先级的重要性

> ⚠️ **重要提醒：匹配顺序至关重要**
>
> 在编写模式匹配规则时，必须将更具体的模式放在更一般的模式之前。例如：
>
> ```moonbit
> // ✅ 正确的顺序
> loop code[:] {
>   [.."->", ..rest] => { ... }     // 先匹配多字符运算符
>   ['-' | '>' as c, ..rest] => { ... }  // 再匹配单字符
> }
>
> // ❌ 错误的顺序 - "->"将永远无法被匹配
> loop code[:] {
>   ['-' | '>' as c, ..rest] => { ... }
>   [.."->", ..rest] => { ... }     // 永远不会执行
> }
> ```

通过这种基于模式匹配的方法，我们不仅避免了复杂的状态机实现，还获得了更清晰、更容易维护的代码结构。

---

## 第三章：语法分析与抽象语法树构建

**语法分析**（Syntactic Analysis）是编译器的第二个核心阶段，其任务是将词法分析产生的Token序列重新组织为具有层次结构的**抽象语法树**（Abstract Syntax Tree, AST）。这个过程不仅要验证程序是否符合语言的语法规则，更要为后续的语义分析和代码生成提供结构化的数据表示。

### 抽象语法树设计：程序的结构化表示

在构建语法分析器之前，我们需要精心设计AST的结构。这个设计决定了如何表示程序的语法结构，以及后续编译阶段如何处理这些结构。

#### 1. 核心类型系统

首先，我们定义TinyMoonbit类型系统在AST中的表示：

```moonbit
pub enum Type {
  Unit    // 单位类型，表示无返回值
  Bool    // 布尔类型：true, false
  Int     // 32位有符号整数
  Double  // 64位双精度浮点数
} derive(Show, Eq, ToJson)

pub fn parse_type(type_name: String) -> Type {
  match type_name {
    "Unit" => Type::Unit
    "Bool" => Type::Bool  
    "Int" => Type::Int
    "Double" => Type::Double
    _ => abort("Unknown type: \{type_name}")
  }
}
```

#### 2. 分层的AST节点设计

我们采用分层设计来清晰地表示程序的不同抽象层次：

1. 原子表达式（AtomExpr）
   代表不可再分解的基本表达式单元：

```moonbit
pub enum AtomExpr {
  Bool(Bool)                                    // 布尔字面量
  Int(Int)                                      // 整数字面量  
  Double(Double)                                // 浮点数字面量
  Var(String, mut ty~ : Type?)                  // 变量引用
  Paren(Expr, mut ty~ : Type?)                  // 括号表达式
  Call(String, Array[Expr], mut ty~ : Type?)    // 函数调用
} derive(Show, Eq, ToJson)
```

2. 复合表达式（Expr）
   可以包含运算符和多个子表达式的更复杂结构：

```moonbit
pub enum Expr {
  AtomExpr(AtomExpr, mut ty~ : Type?)          // 原子表达式包装
  Unary(String, Expr, mut ty~ : Type?)         // 一元运算：-, !
  Binary(String, Expr, Expr, mut ty~ : Type?)  // 二元运算：+, -, *, /, ==, !=, 等
} derive(Show, Eq, ToJson)
```

3. 语句（Stmt）
   代表程序中的可执行单元：

```moonbit
pub enum Stmt {
  Let(String, Type, Expr)                      // 变量声明：let x : Int = 5;
  Assign(String, Expr)                         // 赋值语句：x = 10;
  If(Expr, Array[Stmt], Array[Stmt])           // 条件分支：if-else
  While(Expr, Array[Stmt])                     // 循环语句：while
  Return(Expr?)                                // 返回语句：return expr;
  Expr(Expr)                                   // 单表达式语句
} derive(Show, Eq, ToJson)
```

4. 顶层结构
   函数定义和完整程序：

```moonbit
pub struct Function {
  name : String                     // 函数名
  params : Array[(String, Type)]    // 参数列表：[(参数名, 类型)]
  ret_ty : Type                     // 返回类型
  body : Array[Stmt]                // 函数体语句序列
} derive(Show, Eq, ToJson)

// 程序定义为函数名到函数定义的映射
pub type Program Map[String, Function]
```

> **设计要点：类型标记的可变性**
>
> 注意到每个表达式节点都包含一个 `mut ty~ : Type?` 字段。这个设计允许我们在类型检查阶段填充类型信息，而不需要重新构建整个AST。

### 递归下降解析：自顶向下的构建策略

**递归下降**（Recursive Descent）是一种自顶向下的语法分析方法，其核心思想是为每个语法规则编写一个对应的解析函数。在Moonbit中，模式匹配使这种方法的实现变得异常优雅。

#### 解析原子表达式

```moonbit
pub fn parse_atom_expr(
  tokens: ArrayView[Token]
) -> (AtomExpr, ArrayView[Token]) raise {
  match tokens {
    // 解析字面量
    [Bool(b), ..rest] => (AtomExpr::Bool(b), rest)
    [Int(i), ..rest] => (AtomExpr::Int(i), rest)
    [Double(d), ..rest] => (AtomExpr::Double(d), rest)
    
    // 解析函数调用：func_name(arg1, arg2, ...)
    [Lower(func_name), Bracket('('), ..rest] => {
      let (args, rest) = parse_argument_list(rest)
      match rest {
        [Bracket(')'), ..remaining] => 
          (AtomExpr::Call(func_name, args, ty=None), remaining)
        _ => raise SyntaxError("Expected ')' after function arguments")
      }
    }
    
    // 解析变量引用
    [Lower(var_name), ..rest] => 
      (AtomExpr::Var(var_name, ty=None), rest)
    
    // 解析括号表达式：(expression)
    [Bracket('('), ..rest] => {
      let (expr, rest) = parse_expression(rest)
      match rest {
        [Bracket(')'), ..remaining] => 
          (AtomExpr::Paren(expr, ty=None), remaining)
        _ => raise SyntaxError("Expected ')' after expression")
      }
    }
    
    _ => raise SyntaxError("Expected atomic expression")
  }
}
```

#### 解析语句

语句解析需要根据开头的关键字分发到不同的处理函数：

```moonbit
pub fn parse_stmt(tokens : ArrayView[Token]) -> (Stmt, ArrayView[Token]) {
  match tokens {
    // 解析let语句
    [Keyword("let"), Lower(var_name), Symbol(":"), ..] => { /* ... */ }
    
    // 解析if/while/return语句
    [Keyword("if"), .. rest] => parse_if_stmt(rest)
    [Keyword("while"), .. rest] => parse_while_stmt(rest)
    [Keyword("return"), .. rest] => { /* ... */ }
    
    // 解析赋值语句
    [Lower(_), Symbol("="), .. rest] => parse_assign_stmt(tokens)

    // 解析单表达式语句
    [Lower(_), Symbol("="), .. rest] => parse_single_expr_stmt(tokens)
    
    _ => { /* 错误处理 */ }
  }
}
```

> **难点**：处理运算符优先级：
>
> 表达式解析中最复杂的部分是处理运算符优先级，我们需要确保1 + 2 * 3被正确解析为1 + (2 * 3)而不是(1 + 2) * 3。

### 💡 Moonbit高级特性应用

#### 自动派生功能

```moonbit
pub enum Expr {
  // ...
} derive(Show, Eq, ToJson)
```

Moonbit的 `derive` 功能自动为类型生成常用的实现，这里我们使用三个：

- **`Show`**：提供调试输出功能
- **`Eq`**：支持相等性比较
- **`ToJson`**：序列化为JSON格式，便于调试和持久化

这些自动生成的功能在编译器开发中极为有用，特别是在调试和测试阶段。

#### 错误处理机制

```moonbit
pub fn parse_expression(tokens: ArrayView[Token]) -> (Expr, ArrayView[Token]) raise {
  // raise关键字表示此函数可能抛出异常
}
```

Moonbit的 `raise` 机制提供了结构化的错误处理，使得语法错误能够被准确定位和报告。

通过这种分层设计和递归下降的解析策略，我们构建了一个既灵活又高效的语法分析器，为后续的类型检查阶段奠定了坚实的基础。

---

## 第四章：类型检查与语义分析

**语义分析**是编译器设计中承上启下的关键阶段。虽然语法分析确保了程序结构的正确性，但这并不意味着程序在语义上是有效的。**类型检查**作为语义分析的核心组成部分，负责验证程序中所有操作的类型一致性，确保类型安全和运行时的正确性。

### 作用域管理：构建环境链

类型检查面临的首要挑战是正确处理变量的**作用域**（Scope）。在程序的不同层次（全局、函数、块级别），同一个变量名可能指向不同的实体。我们采用**环境链**（Environment Chain）的经典设计来解决这个问题：

```moonbit
pub struct TypeEnv[K, V] {
  parent : TypeEnv[K, V]?     // 指向父环境的引用
  data : Map[K, V]            // 当前环境的变量绑定
}
```

环境链的核心是变量查找算法，它遵循**词法作用域**的规则：

```moonbit
pub fn TypeEnv::get[K : Eq + Hash, V](self : Self[K, V], key : K) -> V? {
  match self.data.get(key) {
    Some(value) => Some(value)    // 在当前环境中找到
    None =>
      match self.parent {
        Some(parent_env) => parent_env.get(key)  // 递归查找父环境
        None => None              // 到达顶层环境，变量未定义
      }
  }
}
```

> **设计原则：词法作用域**
>
> 这种设计确保了变量的查找遵循词法作用域规则：
>
> 1. 首先在当前作用域中查找
> 2. 如果未找到，向上层作用域递归查找
> 3. 直到找到变量或到达全局作用域

### 类型检查器架构

单纯的环境管理还不足以完成类型检查任务。某些操作（如函数调用）需要访问全局的程序信息。因此，我们设计了一个综合的类型检查器：

```moonbit
pub struct TypeChecker {
  local_env : TypeEnv[String, Type]    // 本地变量环境
  current_func : Function              // 当前检查的函数
  program : Program                    // 完整的程序信息
}
```

### 部分节点类型检查的实现

类型检查器的核心是对不同AST节点应用相应的类型规则。以下是表达式类型检查的实现：

```moonbit
pub fn Expr::check_type(
  self : Self, 
  env : TypeEnv[String, Type]
) -> Type raise {
  match self {
    // 原子表达式的类型检查
    AtomExpr(atom_expr, ..) as node => {
      let ty = atom_expr.check_type(env)
      node.ty = Some(ty)  // 填充类型信息
      ty
    }
    
    // 一元运算的类型检查
    Unary("-", expr, ..) as node => {
      let ty = expr.check_type(env)
      node.ty = Some(ty)
      ty
    }
    
    // 二元运算的类型检查
    Binary(""+, lhs, rhs, ..) as node => {
      let lhs_type = lhs.check_type(env)
      let rhs_type = rhs.check_type(env)
      
      // 确保操作数类型一致
      guard lhs_type == rhs_type else {
        raise TypeCheckError(
          "Binary operation requires matching types, got \{lhs_type} and \{rhs_type}"
        )
      }
      
      let result_type = match op {
        // 比较运算符总是返回布尔值
        "==" | "!=" | "<" | "<=" | ">" | ">=" => Type::Bool
        
        // 算术运算符等保持操作数类型
        _ => lhs_type
      }
      
      node.ty = Some(result_type)
      result_type
    }
  }
}
```

** 💡 Moonbit枚举修改技巧 **

在类型检查过程中，我们需要为AST节点填充类型信息。Moonbit提供了一种优雅的方式来修改枚举变体的可变字段：

```moonbit
pub enum Expr {
  AtomExpr(AtomExpr, mut ty~ : Type?)               
  Unary(String, Expr, mut ty~ : Type?)          
  Binary(String, Expr, Expr, mut ty~ : Type?)   
} derive(Show, Eq, ToJson)
```

通过在模式匹配中使用 `as` 绑定，我们可以获得对枚举变体的引用并修改其可变字段：

```moonbit
match expr {
  AtomExpr(atom_expr, ..) as node => {
    let ty = atom_expr.check_type(env)
    node.ty = Some(ty)  // 修改可变字段
    ty
  }
  // ...
}
```

这种设计避免了重新构建整个AST的开销，同时保持了函数式编程的风格。

---

## 完整编译流程展示

经过词法分析、语法分析和类型检查三个阶段，我们的编译器前端已经能够将源代码转换为完全类型化的抽象语法树。让我们通过一个简单的例子来展示完整的过程：

### 源代码示例

```moonbit
fn add(x: Int, y: Int) -> Int { 
  return x + y; 
}
```

### 编译输出：类型化AST

利用 `derive(ToJson)` 功能，我们可以将最终的AST输出为JSON格式进行查看：

```json
{
  "functions": {
    "add": {
      "name": "add",
      "params": [
        ["x", {"$tag": "Int"}],
        ["y", {"$tag": "Int"}]
      ],
      "ret_ty": {"$tag": "Int"},
      "body": [
        {
          "$tag": "Return",
          "0": {
            "$tag": "Binary",
            "0": "+",
            "1": {
              "$tag": "AtomExpr",
              "0": {
                "$tag": "Var",
                "0": "x",
                "ty": {"$tag": "Int"}
              },
              "ty": {"$tag": "Int"}
            },
            "2": {
              "$tag": "AtomExpr", 
              "0": {
                "$tag": "Var",
                "0": "y",
                "ty": {"$tag": "Int"}
              },
              "ty": {"$tag": "Int"}
            },
            "ty": {"$tag": "Int"}
          }
        }
      ]
    }
  }
}
```

从这个JSON输出中，我们可以清楚地看到：

1. **完整的函数签名**：包括参数列表和返回类型
2. **类型标记的AST节点**：每个表达式都携带了类型信息
3. **结构化的程序表示**：为后续的代码生成阶段提供了清晰的数据结构

---

## 结语

通过本篇文章，我们深入探讨了编译器前端的完整实现流程。从字符流到类型化的抽象语法树，我们见证了Moonbit语言在编译器构建中的独特优势：

### 核心收获

1. **模式匹配的威力**：Moonbit的字符串模式匹配和结构化模式匹配极大简化了词法分析和语法分析的实现
2. **函数式编程范式**：`loop`构造、环境链和不可变数据结构的结合，提供了既优雅又高效的解决方案
3. **类型系统的表达力**：通过枚举的可变字段和trait对象，我们能够构建既类型安全又灵活的数据结构
4. **工程化特性**：`derive`功能、结构化错误处理和JSON序列化等特性，大大提升了开发效率

### 展望下篇

在掌握了语法前端的实现之后，下篇文章将引导我们进入更加激动人心的代码生成阶段。我们将：

- 深入了解LLVM中间表示的设计哲学
- 探索Moonbit官方`llvm.mbt`绑定库的使用方法
- 实现从AST到LLVM IR的完整转换
- 生成可执行的RISC-V汇编代码

编译器的构建是一个复杂而富有挑战性的过程，但正如我们在本篇中所展示的，Moonbit为这个过程提供了强大而优雅的工具。让我们在下篇中继续这段令人兴奋的编译器构建之旅。

> **资源推荐**
>
> - [TinyMoonbit完整项目](https://github.com/Kaida-Amethyst/TinyMoonbitLLVM)
> - [Moonbit官方文档](https://www.moonbitlang.com/docs/)
> - [llvm.mbt文档](https://mooncakes.io/docs/Kaida-Amethyst/llvm)
> - [llvm.mbt项目](https://github.com/moonbitlang/llvm.mbt)
> - [LLVM官方教程](https://llvm.org/docs/tutorial/)

---

