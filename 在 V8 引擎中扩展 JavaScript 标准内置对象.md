# 在 V8 引擎中扩展 JavaScript 标准内置对象
## 摘要
本文会以在 Math 对象上添加一个新方法为例，介绍如何在 V8 引擎中扩展 JavaScript 对象，并分析相关源码。
## 为 Math 对象添加 times10 方法
在 V8 中为 Math 对象添加 times10 方法，times10 方法的作用是将入参乘 10 后返回。可分为 3 步：

- 实现 times10 方法的功能；
- 生成 Code 对象；
- 为 Math 对象添加 times10 属性；

### 实现 times10 方法的功能

这里参考[CodeStubAssembler builtins](https://v8.dev/docs/csa-builtins)，仿照 [Math.imul](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Math/imul) 方法，在[src/builtins/builtins-math-gen.cc](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-math-gen.cc#179)文件内定义如下：

```c++
// ES6 #sec-math.imul
TF_BUILTIN(MathImul, CodeStubAssembler) {
  Node* context = Parameter(Descriptor::kContext);
  Node* x = Parameter(Descriptor::kX);
  Node* y = Parameter(Descriptor::kY);
  Node* x_value = TruncateTaggedToWord32(context, x);
  Node* y_value = TruncateTaggedToWord32(context, y);
  Node* value = Int32Mul(x_value, y_value);
  Node* result = ChangeInt32ToTagged(value);
  Return(result);
}
```

```c++
TF_BUILTIN(MathTimes10, CodeStubAssembler) {
  Node* context = Parameter(Descriptor::kContext);
  Node* x = Parameter(Descriptor::kX);
  Node* x_value = TruncateTaggedToFloat64(context, x);
  Node* y_value = Float64Constant(10.0);
  Node* value = Float64Mul(x_value, y_value);
  Node* result = ChangeFloat64ToTagged(value);
  Return(result);
}
```

### 生成 Code 对象

在 [src/builtins/builtins-definitions.h](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-definitions.h#34) 的宏 BUILTIN_LIST_BASE 下，新增一行：

```c++
#define BUILTIN_LIST_BASE(CPP, TFJ, TFC, TFS, TFH, ASM)      \
  /* GC write barrirer */                                    \
  // 前面源码太长，略                                           
  TFJ(MathTimes10, 1, kReceiver, kX)                         \
  // 后面源码太长，略
```

### 为 Math 对象添加 times10 属性

在[src/init/bootstrapper.cc](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/init/bootstrapper.cc#2705)文件中的[Genesis::InitializeGlobal](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/init/bootstrapper.cc#1386) Math 对象相关，增加一行

```c++
SimpleInstallFunction(isolate_, math, "exp", Builtins::kMathExp, 1, true);
Handle<JSFunction> math_floor = SimpleInstallFunction(
  isolate_, math, "floor", Builtins::kMathFloor, 1, true);
native_context()->set_math_floor(*math_floor);
// 下面一行为新增
SimpleInstallFunction(isolate_, math, "times10", Builtins::kMathTimes10, 1, true);
```

一共修改了 3 处代码，编译 V8，运行 D8：

```c++
./tools/dev/gm.py x64.debug
./out/x64.debug/d8
```

结果如下：

![运行结果](https://raw.githubusercontent.com/xudale/blog/master/assets/times10.png)

## 源码分析

### 实现 times10 方法的功能

首先回顾一下实现代码：

```c++
// ES6 #sec-math.imul
TF_BUILTIN(MathImul, CodeStubAssembler) {
  Node* context = Parameter(Descriptor::kContext);
  Node* x = Parameter(Descriptor::kX);
  Node* y = Parameter(Descriptor::kY);
  Node* x_value = TruncateTaggedToWord32(context, x);
  Node* y_value = TruncateTaggedToWord32(context, y);
  Node* value = Int32Mul(x_value, y_value);
  Node* result = ChangeInt32ToTagged(value);
  Return(result);
}
```

[TF_BUILTIN](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-utils-gen.h#29)是 C++ 定义的宏，源码如下：

```c++
#define TF_BUILTIN(Name, AssemblerBase)                                 \
  class Name##Assembler : public AssemblerBase {                        \
   public:                                                              \
    using Descriptor = Builtin_##Name##_InterfaceDescriptor;            \
                                                                        \
    explicit Name##Assembler(compiler::CodeAssemblerState* state)       \
        : AssemblerBase(state) {}                                       \
    void Generate##Name##Impl();                                        \
                                                                        \
    Node* Parameter(Descriptor::ParameterIndices index) {               \
      return CodeAssembler::Parameter(static_cast<int>(index));         \
    }                                                                   \
  };                                                                    \
  void Builtins::Generate_##Name(compiler::CodeAssemblerState* state) { \
    Name##Assembler assembler(state);                                   \
    state->SetInitialDebugInformation(#Name, __FILE__, __LINE__);       \
    if (Builtins::KindOf(Builtins::k##Name) == Builtins::TFJ) {         \
      assembler.PerformStackCheck(assembler.GetJSContextParameter());   \
    }                                                                   \
    assembler.Generate##Name##Impl();                                   \
  }                                                                     \
  void Name##Assembler::Generate##Name##Impl()
```

将上两段代码放在一个文件中，使用 g++ 宏扩展命令：

```c++
g++ xxxx.cpp -E xxxx.out
```

得到生成后的文件如下：

```c++
class MathImulAssembler : public CodeStubAssembler { 
  public: 
    using Descriptor = Builtin_MathImul_InterfaceDescriptor;
    explicit MathImulAssembler(compiler::CodeAssemblerState* state) 
      : CodeStubAssembler(state) {} 
  void GenerateMathImulImpl(); 
  Node* Parameter(Descriptor::ParameterIndices index) { 
    return CodeAssembler::Parameter(static_cast<int>(index)); 
  } 
}; 
void Builtins::Generate_MathImul(compiler::CodeAssemblerState* state) { 
  MathImulAssembler assembler(state); 
  state->SetInitialDebugInformation("MathImul", "macro.cpp", 25); 
  if (Builtins::KindOf(Builtins::kMathImul) == Builtins::TFJ) { 
    assembler.PerformStackCheck(assembler.GetJSContextParameter()); 
  } 
  assembler.GenerateMathImulImpl(); 
} 
void MathImulAssembler::GenerateMathImulImpl() {
  Node* context = Parameter(Descriptor::kContext);
  Node* x = Parameter(Descriptor::kX);
  Node* y = Parameter(Descriptor::kY);
  Node* x_value = TruncateTaggedToWord32(context, x);
  Node* y_value = TruncateTaggedToWord32(context, y);
  Node* value = Int32Mul(x_value, y_value);
  Node* result = ChangeInt32ToTagged(value);
  Return(result);
}
```

可见 [TF_BUILTIN](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-utils-gen.h#29)宏最终会把实现代码扩展成一个 C++ 类，V8 有很多奇技淫巧，上面只是入门级的。

### 生成 Code 对象

Code 是 V8 中的一个类，用于描述可执行代码。每个 JavaScript 函数在 V8 中都有一个与之关联的 Code 对象，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/objects/js-objects.h#932)：

```c++
class JSFunction : public JSObject {
 public:
  // 源码在长，略
  // [code]: The generated code object for this function.  Executed
  // when the function is invoked, e.g. foo() or new foo(). See
  // [[Call]] and [[Construct]] description in ECMA-262, section
  // 8.6.2, page 27.
  inline Code code() const; 
  inline void set_code(Code code);
  // Get the abstract code associated with the function, which will either be
  // a Code object or a BytecodeArray.
  inline AbstractCode abstract_code();
}
```

JSFunction 对应 JavaScript 的函数，JSObject 对应 JavaScript 的对象。相关内容可参考[从 V8 源码理解 Javascript 函数是一等公民](https://zhuanlan.zhihu.com/p/101132637)。

Code 对象[声明如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/objects/code.h#31)

```c++
// 源码太长，只截取部分
// Code describes objects with on-the-fly generated machine code.
class Code : public HeapObject {
 public:
  // [instruction_size]: Size of the native instructions, including embedded
  // data such as the safepoints table.
  inline int raw_instruction_size() const;
  inline void set_raw_instruction_size(int value);
  inline int InstructionSize() const;
  V8_EXPORT_PRIVATE int OffHeapInstructionSize() const;
  inline bool is_optimized_code() const;
  inline bool is_wasm_code() const;
  // [is_turbofanned]: For kind STUB or OPTIMIZED_FUNCTION, tells whether the
  // code object was generated by the TurboFan optimizing compiler.
  inline bool is_turbofanned() const;
}
```

上文中，我们在 [src/builtins/builtins-definitions.h](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-definitions.h#34) 的宏 BUILTIN_LIST_BASE 下，新增一行：

```c++
#define BUILTIN_LIST_BASE(CPP, TFJ, TFC, TFS, TFH, ASM)      \
  /* GC write barrirer */                                    \
  // 前面源码太长，略                                           
  TFJ(MathTimes10, 1, kReceiver, kX)                         \
  // 后面源码太长，略
```

在 V8 源码中全局搜索 BUILTIN_LIST_BASE，发现 [BUILTIN_LIST](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-definitions.h#1321)，有调用 BUILTIN_LIST_BASE。

```c++
#define BUILTIN_LIST(CPP, TFJ, TFC, TFS, TFH, BCH, ASM)  \
  BUILTIN_LIST_BASE(CPP, TFJ, TFC, TFS, TFH, ASM)        \
  // 专注重点，后面略
```

在 V8 源码中全局搜索 BUILTIN_LIST，发现 [SetupIsolateDelegate::SetupBuiltinsInternal](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/setup-builtins-internal.cc#286)有调用 BUILTIN_LIST

```c++
void SetupIsolateDelegate::SetupBuiltinsInternal(Isolate* isolate) {
  Builtins* builtins = isolate->builtins();
  int index = 0;
  Code code;
// 源码太长，前面略
#define BUILD_TFJ(Name, Argc, ...)                              \
  code = BuildWithCodeStubAssemblerJS(                          \
      isolate, index, &Builtins::Generate_##Name, Argc, #Name); \
  AddBuiltin(builtins, index++, code);
// 源码太长，中间略
  BUILTIN_LIST(BUILD_CPP, BUILD_TFJ, BUILD_TFC, BUILD_TFS, BUILD_TFH, BUILD_BCH, BUILD_ASM);
// 源码太长，后面略 
}
```

这种宏嵌套宏，并且不同的宏定义相隔甚选的写法，笔者是第一次见，当初耗费很长时间才看懂这段代码。其实，SetupIsolateDelegate::SetupBuiltinsInternal 主要做了两件事情：获取 Code 对象，将 Code 对象存入 builtins 数组。

获取 Code 对象的代码宏扩展展开后：

```c++
code = BuildWithCodeStubAssemblerJS(                          
      isolate, index, &Builtins::Generate_MathTimes10, Argc, "MathTimes10"); 
```

将 Code 对象存入 builtins 数组：

```c++
  AddBuiltin(builtins, index++, code); 
```

顺着 AddBuiltin 的源码一直追下去，发现所有的 Code 对象都存在 [builtins_](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/isolate-data.h#162)中。

```c++
Address builtins_[Builtins::builtin_count] = {};
```

让我们看宏 [BUILTIN_LIST](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins.h#49) 的另一处调用：

```c++
class Builtins {
  public:
  // 源码太长，前面略
  enum Name : int32_t {
#define DEF_ENUM(Name, ...) k##Name,
    BUILTIN_LIST(DEF_ENUM, DEF_ENUM, DEF_ENUM, DEF_ENUM, DEF_ENUM, DEF_ENUM,
                 DEF_ENUM)
#undef DEF_ENUM
        builtin_count
  };
  // 源码太长，后面略
}
```

![generateCode](https://raw.githubusercontent.com/xudale/blog/master/assets/generateCode.png)

### 为 Math 对象添加 times10 属性

这一步代码最简单，实际只有一行：

```c++
SimpleInstallFunction(isolate_, math, "times10", Builtins::kMathTimes10, 1, true);
```

参数 math 实际上就是 JavaScript 的 Math 对象，参数 "times10" 是我们要添加的方法的名字，Builtins::kMathTimes10 是方法的索引，至此，大功告成。本节内容总结如下图：

![getCode](https://raw.githubusercontent.com/xudale/blog/master/assets/getCode.png)

## 一些感想

### V8 还是比较安全的

笔者是正经人，从未想过攻击哪个网站。从内置函数相关的源码看下来，攻击 V8 还是很难的。如果攻击 V8，首先要绕过词法分析、语法分析和 AST 树 3 座大山，V8 的内置函数都存在数组 [builtins_](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/isolate-data.h#162)中，但这个数组离 JavaScript 层面实在太远了，笔者认为很难在 JavaScript 层面改变这个数组。

### 谈谈 Bootstrap 

在芯片或嵌入式领域，Bootstrap 有启动程序和引导程序的含义。这里谈的 [Bootstrap](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/init/bootstrapper.cc) 是 V8 里的一个文件。之所文章最后提一下这个文件，是因为这个文件注册了所有 JavaScript 标准所规定的函数，和前端的关系最为密切，因为某些原因，上面的链接可能打不开，笔者索性就粘贴一段 Bootstrap 文件中的数组相关的代码：

```c++
// Set up %ArrayPrototype%.
// The %ArrayPrototype% has TERMINAL_FAST_ELEMENTS_KIND in order to ensure
// that constant functions stay constant after turning prototype to setup
// mode and back.
Handle<JSArray> proto = factory->NewJSArray(0, TERMINAL_FAST_ELEMENTS_KIND,
                                            AllocationType::kOld);
JSFunction::SetPrototype(array_function, proto);
native_context()->set_initial_array_prototype(*proto);

Handle<JSFunction> is_arraylike = SimpleInstallFunction(
    isolate_, array_function, "isArray", Builtins::kArrayIsArray, 1, true);
native_context()->set_is_arraylike(*is_arraylike);

SimpleInstallFunction(isolate_, array_function, "from",
                      Builtins::kArrayFrom, 1, false);
SimpleInstallFunction(isolate_, array_function, "of", Builtins::kArrayOf, 0,
                      false);

JSObject::AddProperty(isolate_, proto, factory->constructor_string(),
                      array_function, DONT_ENUM);

SimpleInstallFunction(isolate_, proto, "concat", Builtins::kArrayConcat, 1,
                      false);
SimpleInstallFunction(isolate_, proto, "copyWithin",
                      Builtins::kArrayPrototypeCopyWithin, 2, false);
SimpleInstallFunction(isolate_, proto, "fill",
                      Builtins::kArrayPrototypeFill, 1, false);
SimpleInstallFunction(isolate_, proto, "find",
                      Builtins::kArrayPrototypeFind, 1, false);
SimpleInstallFunction(isolate_, proto, "findIndex",
                      Builtins::kArrayPrototypeFindIndex, 1, false);
SimpleInstallFunction(isolate_, proto, "lastIndexOf",
                      Builtins::kArrayPrototypeLastIndexOf, 1, false);
SimpleInstallFunction(isolate_, proto, "pop", Builtins::kArrayPrototypePop,
                      0, false);
SimpleInstallFunction(isolate_, proto, "push",
                      Builtins::kArrayPrototypePush, 1, false);
SimpleInstallFunction(isolate_, proto, "reverse",
                      Builtins::kArrayPrototypeReverse, 0, false);
SimpleInstallFunction(isolate_, proto, "shift",
                      Builtins::kArrayPrototypeShift, 0, false);
SimpleInstallFunction(isolate_, proto, "unshift",
                      Builtins::kArrayPrototypeUnshift, 1, false);
SimpleInstallFunction(isolate_, proto, "slice",
                      Builtins::kArrayPrototypeSlice, 2, false);
SimpleInstallFunction(isolate_, proto, "sort",
                      Builtins::kArrayPrototypeSort, 1, false);
SimpleInstallFunction(isolate_, proto, "splice",
                      Builtins::kArrayPrototypeSplice, 2, false);
SimpleInstallFunction(isolate_, proto, "includes", Builtins::kArrayIncludes,
                      1, false);
SimpleInstallFunction(isolate_, proto, "indexOf", Builtins::kArrayIndexOf,
                      1, false);
SimpleInstallFunction(isolate_, proto, "join",
                      Builtins::kArrayPrototypeJoin, 1, false);
```

上面的代码先是设置了 JavaScript Array 的 Prototype，然后为 JavaScript Array 添加了 from、of 静态方法，最后为 JavaScript Array 的 Prototype 添加了 concat、pop、push 等方法。

从上面的代码，还可以看到 JavaScript 做为一门动态语言的所具有的特点。比如 JavaScript Array 原型上的 concat、pop、push等方法，方法的名字做为一个字符串，客观的存在于 V8 中。如果运行时想要使用 pop 方法，只要JavaScript Array 原型上有一个名为 pop 的方法就可以，pop 方法可以由 V8 提供，也可以由第 3 方提供。总之，只要该方法挂载在JavaScript Array 原型上，并且名称为 pop 就可以。这种特性为 JavaScript 运行时的动态加载提供了基础。

而静态类型语言，如 C 语言。C 语言的变量和函数在编译后都直接对应地址，变量名和函数名在编译后都不复存在，这里可以参考笔者的另一篇文章[从 V8 源码理解 Javascript 函数是一等公民](https://zhuanlan.zhihu.com/p/101132637)。通常，静态语言都没有类似 JavaScript 动态加载的特性。

## 参考文献

[CodeStubAssembler builtins](https://v8.dev/docs/csa-builtins)

[Taming architecture complexity in V8 — the CodeStubAssembler](https://v8.dev/blog/csa)













