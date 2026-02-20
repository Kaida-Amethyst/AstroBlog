---
title: MoonBit与LLVM共舞（下）- llvm后端生成
description: 这个系列的文章来教你使用MoonBit来实现一个自己的编译器，这是第二部分，在前端解析部分完成后，我们来借助llvm.mbt的能力生成后端。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/4.jpg
category: 编程实践
published: 2025-07-12
tags: [MoonBit, Compiler]
---

## 引言

在编程语言设计的过程中，**语法前端**负责理解和验证程序的结构与语义，而**编译器后端**则承担着将这些抽象概念转化为可执行机器代码的重任。后端的实现不仅需要对目标体系结构有深入的理解，更要掌握复杂的优化技术来生成高效的代码。

**LLVM**（Low Level Virtual Machine）作为现代编译器基础设施的集大成者，为我们提供了一个强大而灵活的解决方案。通过将程序转换为LLVM中间表示（Intermediate Representation, IR），我们可以利用LLVM成熟的工具链将代码编译到多种目标架构，包括RISC-V、ARM和x86等。

> **Moonbit的LLVM生态**
>
> Moonbit官方提供了两个重要的LLVM相关项目：
>
> - **[`llvm.mbt`](https://github.com/moonbitlang/llvm.mbt)**：原版LLVM的Moonbit语言绑定，提供对llvm-c接口的直接访问。需要安装完整的LLVM工具链，只能生成native后端，需要自行解决编译和链接的问题，但能够生成与原版LLVM完全兼容的IR。
> - **[`MoonLLVM`](https://github.com/moonbitlang/MoonLLVM)**：纯Moonbit实现的LLVM仿制版，无需外部依赖即可生成LLVM IR，支持JavaScript和WebAssembly后端
>
> 本文选择`llvm.mbt`作为我们的工具，其API设计参考了Rust生态中广受好评的inkwell库。

在上篇《Moonbit 与 LLVM 共舞：实现现代编译器（上篇）》中，我们已经完成了从源代码到类型化抽象语法树的转换。本篇将承接这一成果，重点阐述**代码生成**的核心技术和实现细节。

---

## 第一章：LLVM类型系统的Moonbit表示

在深入代码生成之前，我们需要首先理解`llvm.mbt`如何在Moonbit的类型系统中表示LLVM的各种概念。LLVM的类型系统相当复杂，包含基本类型、复合类型和函数类型等多个层次。

### Trait Object：类型的抽象表示

在`llvm.mbt`的API设计中，你会频繁遇到`&Type`这一核心概念。这并非一个具体的struct或enum，而是一个**Trait Object**——可以将其理解为面向对象编程中**抽象基类**的函数式对等物。

```moonbit
// &Type是一个trait object，代表任意LLVM类型
let some_type: &Type = context.i32_type()
```

#### 类型识别与转换

要确定一个`&Type`的具体类型，我们需要通过`as_type_enum`接口进行运行时类型检查：

```moonbit
pub fn identify_type(ty: &Type) -> String {
  match ty.as_type_enum() {
    IntType(int_ty) => "Integer type with \{int_ty.get_bit_width()} bits"
    FloatType(float_ty) => "Floating point type"
    PointerType(ptr_ty) => "Pointer type"
    FunctionType(func_ty) => "Function type"
    ArrayType(array_ty) => "Array type"
    StructType(struct_ty) => "Structure type"
    VectorType(vec_ty) => "Vector type"
    ScalableVectorType(svec_ty) => "Scalable vector type"
    MetadataType(meta_ty) => "Metadata type"
  }
}
```

#### 安全的类型转换策略

当我们确信某个`&Type`具有特定的类型时，有多种转换方式可供选择：

1. 直接转换（适用于确定性场景）

```moonbit
let ty: &Type = context.i32_type()
let i32_ty = ty.into_int_type()  // 直接转换，错误由llvm.mbt处理
let bit_width = i32_ty.get_bit_width()  // 调用IntType特有的方法
```

2. 防御性转换（推荐的生产环境做法）

```moonbit
let ty: &Type = get_some_type()  // 从某处获得的未知类型

guard ty.as_type_enum() is IntType(i32_ty) else {
  raise CodeGenError("Expected integer type, got \{ty}")
}

// 现在可以安全地使用i32_ty
let bit_width = i32_ty.get_bit_width()
```

### 复合类型的构造

LLVM支持多种复合类型，这些类型通常通过基本类型的方法来构造：

```moonbit
pub fn create_composite_types(context: @llvm.Context) -> Unit {
  let i32_ty = context.i32_type()
  let f64_ty = context.f64_type()
  
  // 数组类型：[16 x i32]
  let i32_array_ty = i32_ty.array_type(16)
  
  // 函数类型：i32 (i32, i32)
  let add_func_ty = i32_ty.fn_type([i32_ty, i32_ty])
  
  // 结构体类型：{i32, f64}
  let struct_ty = context.struct_type([i32_ty, f64_ty])
  
  // 指针类型（LLVM 18+中所有指针都是opaque）
  let ptr_ty = i32_ty.ptr_type()
  
  // 输出类型信息用于验证
  println("Array type: \{i32_array_ty}")      // [16 x i32]
  println("Function type: \{add_func_ty}")    // i32 (i32, i32)
  println("Struct type: \{struct_ty}")        // {i32, f64}
  println("Pointer type: \{ptr_ty}")          // ptr
}
```

> **重要提醒：Opaque指针**
>
> 自LLVM 18版本开始，所有指针类型都采用了**opaque指针**设计。这意味着无论指向什么类型，所有指针在IR中都表示为`ptr`，指向的具体类型信息在类型系统中不再可见。

---

## 第二章：LLVM值系统与BasicValue概念

相比类型系统，LLVM的值系统会复杂一些。`llvm.mbt`与inkwell一致，将值分为两个重要的抽象层次。`Value` 和 `BasicValue`。不同点在于在于区分值的创建来源和值的使用方式：

- **Value**：关注值是如何产生的（常量、指令结果等）
- **BasicValue**：关注值具有什么样的基本类型（整数、浮点数、指针等）

#### 实际应用示例

```moonbit
pub fn demonstrate_value_system(context: Context, builder: Builder) -> Unit {
  let i32_ty = context.i32_type()
  
  // 创建两个整数常量 - 这些直接就是IntValue
  let const1 = i32_ty.const_int(10)  // Value: IntValue, BasicValue: IntValue
  let const2 = i32_ty.const_int(20)  // Value: IntValue, BasicValue: IntValue
  
  // 执行加法运算 - 结果是一个指令InstructionValue
  let add_result = builder.build_int_add(const1, const2)
  
  // 在不同的上下文中，我们需要不同的视角：
  
  // 作为指令来检查其属性
  let instruction = add_result.as_instruction()
  println("Instruction opcode: \{instruction.get_opcode()}")
  
  // 作为基本值来获取其类型
  let basic_value = add_result.into_basic_value()
  println("Result type: \{basic_value.get_type()}")
  
  // 作为整数值来进行后续计算
  let int_value = add_result.into_int_value()
  let final_result = builder.build_int_mul(int_value, const1)
}
```

### 值类型的完整分类

1. ValueEnum：所有可能的值类型

```moonbit
pub enum ValueEnum {
  IntValue(IntValue)              // 整数值
  FloatValue(FloatValue)          // 浮点数值
  PointerValue(PointerValue)      // 指针值
  StructValue(StructValue)        // 结构体值
  FunctionValue(FunctionValue)    // 函数值
  ArrayValue(ArrayValue)          // 数组值
  VectorValue(VectorValue)        // 向量值
  PhiValue(PhiValue)             // Phi节点值
  ScalableVectorValue(ScalableVectorValue)  // 可伸缩向量值
  MetadataValue(MetadataValue)    // 元数据值
  CallSiteValue(CallSiteValue)    // 调用点值
  GlobalValue(GlobalValue)        // 全局值
  InstructionValue(InstructionValue)  // 指令值
} derive(Show)
```

2. BasicValueEnum：具有基本类型的值

```moonbit
pub enum BasicValueEnum {
  ArrayValue(ArrayValue)              // 数组值
  IntValue(IntValue)                  // 整数值
  FloatValue(FloatValue)              // 浮点数值
  PointerValue(PointerValue)          // 指针值
  StructValue(StructValue)            // 结构体值
  VectorValue(VectorValue)            // 向量值
  ScalableVectorValue(ScalableVectorValue)  // 可伸缩向量值
} derive(Show)
```

### 💡 值转换的最佳实践

在实际的代码生成过程中，我们经常需要在不同的值视角之间进行转换：

```moonbit
pub fn value_conversion_patterns(instruction_result: &Value) -> Unit {
  // 模式1：我知道这是什么类型，直接转换
  let int_val = instruction_result.into_int_value()
  
  // 模式2：我只需要一个基本值，不关心具体类型
  let basic_val = instruction_result.into_basic_value()
  
  // 模式3：防御性编程，检查后转换
  match instruction_result.as_value_enum() {
    // 处理整数值
    IntValue(int_val) => handle_integer(int_val)
    // 处理浮点值
    FloatValue(float_val) => handle_float(float_val)
    _ => raise CodeGenError("Unexpected value type")
  }
}
```

通过这种双层抽象，`llvm.mbt`既保持了LLVM值系统的完整性，又为Moonbit开发者提供了直观易用的接口。

---

## 第三章：LLVM IR生成实战

在理解了类型和值系统的基础上，让我们通过一个完整的示例来演示如何使用`llvm.mbt`生成LLVM IR。这个示例将实现一个简单的 `muladd` 函数，展示从初始化到指令生成的完整流程。

### 基础设施初始化

任何LLVM程序的开始都需要建立三个核心组件：

```moonbit
pub fn initialize_llvm() -> (Context, Module, Builder) {
  // 1. 创建LLVM上下文 - 所有LLVM对象的容器
  let context = @llvm.Context::create()
  
  // 2. 创建模块 - 函数和全局变量的容器
  let module = context.create_module("demo_module")
  
  // 3. 创建IR构建器 - 用于生成指令
  let builder = context.create_builder()
  
  (context, module, builder)
}
```

### 一个简单的函数生成示例

让我们实现一个计算 `(a * b) + c` 的函数：

```moonbit
pub fn generate_muladd_function() -> String {
  // 初始化LLVM基础设施
  let (context, module, builder) = initialize_llvm()
  
  // 定义函数签名
  let i32_ty = context.i32_type()
  let func_type = i32_ty.fn_type([i32_ty, i32_ty, i32_ty])
  let func_value = module.add_function("muladd", func_type)
  
  // 创建函数入口基本块
  let entry_block = context.append_basic_block(func_value, "entry")
  builder.position_at_end(entry_block)
  
  // 获取函数参数
  let arg_a = func_value.get_nth_param(0).unwrap().into_int_value()
  let arg_b = func_value.get_nth_param(1).unwrap().into_int_value()  
  let arg_c = func_value.get_nth_param(2).unwrap().into_int_value()
  
  // 生成计算指令
  let mul_result = builder.build_int_mul(arg_a, arg_b).into_int_value()
  let add_result = builder.build_int_add(mul_result, arg_c)
  
  // 生成返回指令
  let _ = builder.build_return(add_result)
  
  // 输出生成的IR
  module.dump()
}
```

### 生成的LLVM IR

运行上述代码将产生以下LLVM中间表示：

```llvm
; ModuleID = 'demo_module'
source_filename = "demo_module"

define i32 @muladd(i32 %0, i32 %1, i32 %2) {
entry:
  %3 = mul i32 %0, %1
  %4 = add i32 %3, %2
  ret i32 %4
}
```

### 💡 代码生成最佳实践

1. 命名约定

有返回值的指令，构建接口有一个`name`的label argument，可以给指令的结果添加名称。

```moonbit
let mul_result = builder.build_int_mul(lhs, rhs, name="temp_product")
let final_result = builder.build_int_add(mul_result, offset, name="final_sum")
```

2. 错误处理

使用raise而并非panic来进行错误处理，对不好直接确定的情况进行异常管理。

```moonbit
// 对可能失败的操作进行检查
match func_value.get_nth_param(index) {
  Some(param) => param.into_int_value()
  None => raise CodeGenError("Function parameter \{index} not found")
}
```

---

## 第四章：TinyMoonbit编译器实现

现在让我们将注意力转向真正的编译器实现，将上篇文章中构建的抽象语法树转换为LLVM IR。

### 类型映射：从Parser到LLVM

首先需要建立TinyMoonbit类型系统与LLVM类型系统之间的映射关系：

```moonbit
pub struct CodeGen {
  parser_program : Program                    // 源程序的AST表示
  llvm_context : @llvm.Context               // LLVM上下文
  llvm_module : @llvm.Module                 // LLVM模块
  builder : @llvm.Builder                    // IR构建器
  llvm_functions : Map[String, @llvm.FunctionValue]  // 函数映射表
}

pub fn convert_type(self : Self, parser_type : Type) -> &@llvm.Type raise {
  match parser_type {
    Type::Unit => self.llvm_context.void_type() as &@llvm.Type
    Type::Bool => self.llvm_context.bool_type()
    Type::Int => self.llvm_context.i32_type()  
    Type::Double => self.llvm_context.f64_type()
    // 可以根据需要扩展更多类型
  }
}
```

### 环境管理：变量到值的映射

在代码生成阶段，我们需要维护一个从变量名到LLVM值的映射关系：

```moonbit
pub struct Env {
  parent : Env?                        // 父环境引用
  symbols : Map[String, &@llvm.Value]        // 局部变量映射
  
  // 全局信息
  codegen : CodeGen                           // 代码生成器引用
  parser_function : Function                  // 当前函数的AST
  llvm_function : @llvm.FunctionValue         // 当前函数的LLVM表示
}

pub fn get_symbol(self : Self, name : String) -> &@llvm.Value? {
  match self.symbols.get(name) {
    Some(value) => Some(value)
    None => 
      match self.parent {
        Some(parent_env) => parent_env.get_symbol(name)
        None => None
      }
  }
}
```

### 变量处理：内存分配策略

TinyMoonbit作为一个系统级语言，支持变量的重新赋值。在LLVM IR的SSA（Static Single Assignment）形式中，我们需要采用**alloca + load/store**的模式来实现可变变量：

```moonbit
pub fn Stmt::emit(self : Self, env : Env) -> Unit raise {
  match self {
    // 变量声明：例如let x : Int = 5;
    Let(var_name, var_type, init_expr) => {
      // 转换类型并分配栈空间
      let llvm_type = env.codegen.convert_type(var_type)
      let alloca = env.codegen.builder.build_alloca(llvm_type, var_name)
      
      // 将分配的指针记录到符号表
      env.symbols.set(var_name, alloca as &@llvm.Value)
      
      // 计算初始化表达式的值
      let init_value = init_expr.emit(env).into_basic_value()
      
      // 将初始值存储到分配的内存
      let _ = env.codegen.builder.build_store(alloca, init_value)
    }
    
    // 变量赋值：x = 10;
    Assign(var_name, rhs_expr) => {
      // 从符号表获取变量的内存地址
      guard let Some(var_ptr) = env.get_symbol(var_name) else {
        raise CodeGenError("Undefined variable: \{var_name}")
      }
      
      // 计算右侧表达式的值
      let rhs_value = rhs_expr.emit(env).into_basic_value()
      
      // 存储新值到变量内存
      let _ = env.codegen.builder.build_store(var_ptr, rhs_value)
    }
    
    // 其他语句类型...
    _ => { /* 处理其他语句 */ }
  }
}
```

> **设计决策：为什么使用alloca？**
>
> 在函数式语言中，不可变变量可以直接映射为SSA值。但TinyMoonbit支持变量重新赋值，这与SSA的"每个变量只赋值一次"原则冲突。
>
> **alloca + load/store** 模式是处理可变变量的标准做法：
>
> - `alloca`：在栈上分配内存空间
> - `store`：将值写入内存
> - `load`：从内存读取值
>
> LLVM的优化过程会自动将简单的alloca转换回值形式（mem2reg优化）。

### 表达式代码生成

表达式的代码生成相对直观，主要是根据表达式类型调用相应的指令构建方法：

```moonbit
fn Expr::emit(self: Self, env: Env) -> &@llvm.Value raise {
  match self {
    AtomExpr(atom_expr, ..) => atom_expr.emit(env)
    Unary("-", expr, ty = Some(Int)) => {
      let value = expr.emit().into_int_value()
      let zero = env.gen.llvm_ctx.i32_type().const_zeor()
      env.gen.builder.build_int_sub(zero, value)
    }
    Unary("-", expr, ty = Some(Double)) => {
      let value = expr.emit().into_float_value()
      env.gen.builder.build_float_neg(value)
    }
    Binary("+", lhs, rhs, ty=Some(Int)) => {
      let lhs_val = lhs.emit().into_int_value()
      let rhs_val = rhs.emit().into_int_value()
      env.gen.builder.build_int_add(lhs_val, rhs_val)
    }
    // ... others
  }
}
```

> **技术细节：浮点数取负**
>
> 注意在处理浮点数取负时，我们使用 `build_float_neg` 而不是用零减去操作数。这是因为：
>
> 1. **IEEE 754标准**：浮点数有特殊值（如NaN、∞），简单的减法可能产生不正确的结果
> 2. **性能考虑**：专用的否定指令在现代处理器上通常更高效
> 3. **精度保证**：避免了不必要的舍入误差

---

## 第五章：控制流指令的实现

控制流是程序逻辑的骨架，包括条件分支和循环结构。在LLVM IR中，控制流通过**基本块**（Basic Blocks）和**分支指令**来实现。每个基本块代表一个没有内部跳转的指令序列，块与块之间通过分支指令连接。

### 条件分支：if-else语句的实现

条件分支需要创建多个基本块来表示不同的执行路径：

```moonbit
fn Stmt::emit(self: Self, env: Env) -> Unit raise {
  let ctx = env.gen.llvm_ctx
  let func = env.llvm_func
  let builder = env.gen.builder
  match self {
    If(cond, then_stmts, else_stmts) => {
      let cond_val = cond.emit(env).into_int_value()
      
      // 创建三个基本块
      let then_block = ctx.append_basic_block(llvm_func)
      let else_block = ctx.append_basic_block(llvm_func)
      let merge_block = ctx.append_basic_block(llvm_func)

      // 创建跳转指令
      let _ = builder.build_conditional_branch(
        cond_val, then_block, else_block,
      )

      // 生成then_block的代码
      builder.position_at_end(then_block)
      let then_env = self.subenv()
      then_stmts.each(s => s.emitStmt(then_env))
      let _ = builder.build_unconditional_branch(merge_block)

      // 生成else_block的代码
      builder.position_at_end(else_block)
      let else_env = self.subenv()
      else_stmts.each(s => s.emitStmt(else_env))
      let _ = builder.build_unconditional_branch(merge_block)

      // 代码生成完毕后，builder的位置要在merge_block上
      builder.position_at_end(merge_block)

    }
    // ...
  }
}
```

#### 生成的LLVM IR示例

对于以下TinyMoonbit代码：

```moonbit
if x > 0 {
  y = x + 1;
} else {
  y = x - 1; 
}
```

将生成类似这样的LLVM IR：

```llvm
  %1 = load i32, ptr %x, align 4
  %2 = icmp sgt i32 %1, 0
  br i1 %2, label %if.then, label %if.else

if.then:                                          ; preds = %0
  %3 = load i32, ptr %x, align 4
  %4 = add i32 %3, 1
  store i32 %4, ptr %y, align 4
  br label %if.end

if.else:                                          ; preds = %0
  %5 = load i32, ptr %x, align 4
  %6 = sub i32 %5, 1
  store i32 %6, ptr %y, align 4
  br label %if.end

if.end:                                           ; preds = %if.else, %if.then
  ; 后续代码...
```

### 循环结构：while语句的实现

循环的实现需要特别注意条件检查和循环体的正确连接：

```moonbit
fn Stmt::emit(self: Self, env: Env) -> Unit raise {
  let ctx = env.gen.llvm_ctx
  let func = env.llvm_func
  let builder = env.gen.builder
  match self {
    While(cond, body) => {
      // 生成三个块
      let cond_block = ctx.append_basic_block(.llvm_func)
      let body_block = ctx.append_basic_block(llvm_func)
      let merge_block = ctx.append_basic_block(llvm_func)

      // 首先无条件跳转到cond块
      let _ = builder.build_unconditional_branch(cond_block)
      builder.position_at_end(cond_block)

      // 在cond块内生成代码，以及条件跳转指令
      let cond_val = cond.emit().into_int_value()
      let _ = builder.build_conditional_branch(
        cond_val, body_block, merge_block,
      )
      builder.position_at_end(body_block)

      // 对body块生成代码，末尾需要一个无条件跳转指令，到cond块
      let body_env = self.subenv()
      body.each(s => s.emitStmt(body_env))
      let _ = builder.build_unconditional_branch(cond_block)

      // 代码生成结束以后，跳转到merge block
      builder.position_at_end(merge_block)
    }
    // ...
  }
}
```

#### 生成的LLVM IR示例

对于TinyMoonbit代码：

```moonbit
while i < 10 {
  i = i + 1;
}
```

将生成：

```llvm
  br label %while.cond

while.cond:                                       ; preds = %while.body, %0
  %1 = load i32, ptr %i, align 4
  %2 = icmp slt i32 %1, 10
  br i1 %2, label %while.body, label %while.end

while.body:                                       ; preds = %while.cond
  %3 = load i32, ptr %i, align 4
  %4 = add i32 %3, 1
  store i32 %4, ptr %i, align 4
  br label %while.cond

while.end:                                        ; preds = %while.cond
  ; 后续代码...
```

**💡 控制流设计要点 **

1. 基本块的命名策略

`append_basic_block` 函数同样有`name`这个label argument。

```moonbit
// 使用描述性的块名称，便于调试和理解
let then_block = context.append_basic_block(func, name="if.then")
let else_block = context.append_basic_block(func, name="if.else")
let merge_block = context.append_basic_block(func, name="if.end")
```

2. 作用域管理

```moonbit
// 为每个分支和循环体创建独立的作用域
let branch_env = env.sub_env()
branch_stmts.each( stmt => stmt.emit(branch_env) }
```

3. 构建器位置管理

末尾注意将指令构建器放到正确的基本块上。

```moonbit
// 始终确保构建器指向正确的基本块
builder.position_at_end(merge_block)
// 在这个块中生成指令...
```

---

## 第六章：从LLVM IR到机器代码

在生成完整的LLVM IR之后，我们需要将其转换为目标机器的汇编代码。虽然`llvm.mbt`提供了完整的目标机器配置API，但对于学习目的，我们可以使用更简便的方法。

### 使用llc工具链进行编译

最直接的方法是将生成的LLVM IR输出到文件，然后使用LLVM工具链进行编译：

调用`Module`的`dump`函数即可，也可以使用`println`函数。

```moonbit
let gen : CodeGen = ...
let prog = gen.llvm_prog
prog.dump() // 更建议使用dump，会比println快一点，效果相同

// or println(prog)
```

### 完整的编译流程示例

让我们看一个完整的从源代码到汇编代码的编译流程：

1. TinyMoonbit源代码

```moonbit
fn factorial(n: Int) -> Int {
  if n <= 1 {
    return 1;
  }
  return n * factorial(n - 1);
}

fn main() -> Unit {
  let result: Int = factorial(5);
  print_int(result);
}
```

2. 生成的LLVM IR

```llvm
; ModuleID = 'tinymoonbit'
source_filename = "tinymoonbit"

define i32 @factorial(i32 %0) {
entry:
  %1 = alloca i32, align 4
  store i32 %0, ptr %1, align 4
  %2 = load i32, ptr %1, align 4
  %3 = icmp sle i32 %2, 1
  br i1 %3, label %4, label %6

4:                                                ; preds = %entry
  ret i32 1

6:                                                ; preds = %entry
  %7 = load i32, ptr %1, align 4
  %8 = load i32, ptr %1, align 4
  %9 = sub i32 %8, 1
  %10 = call i32 @factorial(i32 %9)
  %11 = mul i32 %7, %10
  ret i32 %11
}

define void @main() {
entry:
  %0 = alloca i32, align 4
  %1 = call i32 @factorial(i32 5)
  store i32 %1, ptr %0, align 4
  %2 = load i32, ptr %0, align 4
  call void @print_int(i32 %2)
  ret void
}

declare void @print_int(i32 %0)
```

3. 使用LLC生成RISC-V汇编

```bash
# 生成llvm ir
moon run main --target native > fact.ll

# 生成RISC-V 64位汇编代码
llc -march=riscv64 -mattr=+m -o fact.s fact.ll
```

4. 生成的RISC-V汇编片段

```assembly
factorial:
.Lfunc_begin0:
	.cfi_startproc
	addi	sp, sp, -32
	.cfi_def_cfa_offset 32
	sd	ra, 24(sp)
	.cfi_offset ra, -8
	sd	s0, 16(sp)
	.cfi_offset s0, -16
	addi	s0, sp, 32
	.cfi_def_cfa s0, 0
	sw	a0, -20(s0)
	lw	a0, -20(s0)
	li	a1, 1
	blt	a1, a0, .LBB0_2
	li	a0, 1
	j	.LBB0_3
.LBB0_2:
	lw	a0, -20(s0)
	lw	a1, -20(s0)
	addi	a1, a1, -1
	sw	a0, -24(s0)
	mv	a0, a1
	call	factorial
	lw	a1, -24(s0)
	mul	a0, a1, a0
.LBB0_3:
	ld	ra, 24(sp)
	ld	s0, 16(sp)
	addi	sp, sp, 32
	ret
```

---

## 结语

通过本系列的两篇文章，我们完成了一个功能完整的编译器实现。尽管功能简单，但比较完整。从字符流的词法分析，到抽象语法树的构建，再到LLVM IR的生成和机器代码的输出。

### 回顾

**上篇**：

- 基于模式匹配的优雅词法分析器
- 递归下降语法分析器的实现
- 完整的类型检查系统
- 环境链作用域管理

**下篇**：

- LLVM类型和值系统的深入理解
- SSA形式下的变量管理策略
- 控制流指令的正确实现
- 完整的代码生成流水线

### Moonbit在编译器开发中的优势

通过这个实践项目，我们深刻体会到了Moonbit在编译器构建领域的独特价值：

1. **表达力强大的模式匹配**：极大简化了AST处理和类型分析的复杂度。
2. **函数式编程范式**：不可变数据结构和纯函数使得编译器逻辑更加清晰可靠。
3. **现代化的类型系统**：trait对象、泛型和错误处理机制提供了充分的抽象能力。
4. **优秀的工程特性**：derive功能、JSON序列化等特性显著提升了开发效率。

### 结语

编译器技术代表了计算机科学理论与工程实践的完美结合。通过Moonbit这一现代化的工具，我们能够以更加优雅和高效的方式探索这个古老而又充满活力的领域。

希望本系列文章能够为读者在编译器设计的道路上提供一个有力的帮助。

> **学习资源推荐**
>
> - [TinyMoonbit完整项目](https://github.com/Kaida-Amethyst/TinyMoonbitLLVM)
> - [Moonbit官方文档](https://www.moonbitlang.com/docs/)
> - [llvm.mbt文档](https://mooncakes.io/docs/Kaida-Amethyst/llvm)
> - [llvm.mbt项目](https://github.com/moonbitlang/llvm.mbt)
> - [LLVM官方教程](https://llvm.org/docs/tutorial/)

---
