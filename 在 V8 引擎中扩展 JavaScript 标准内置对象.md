# 在 V8 引擎中扩展 JavaScript 标准内置对象
## 摘要
本文会以在 Math 对象上添加一个新方法 times10 为例，介绍如何在 V8 引擎中扩展 JavaScript 对象，并分析相关源码。
## 为 Math 对象添加 times10 方法
times10 方法的逻辑很简单，就是将入参乘 10 后返回。在 V8 源码中为 Math 对象添加 times10 方法，可分为 3 步：

- 实现 times10 方法的功能；
- 生成并存储 Code 对象；
- 取出上一步生成的 Code 对象，添加至 Math 对象的 times10 属性上；

### 实现 times10 方法的功能

很多人都说 V8 是用 C++ 写的，其实不然。本文使用 V8 内部的编程语言 [CodeStubAssembler builtins](https://v8.dev/docs/csa-builtins) 来实现 times10 函数的功能。与 C++ 相比，CodeStubAssembler 运行效率更高，而且语法接近汇编。虽然网络上关于 CodeStubAssembler 的教程极少，但是 times10 的逻辑十分简单，参考 V8 中 [Math.imul](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Math/imul) 的 [源码](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-math-gen.cc#179)：

```c++
// ES6 #sec-math.imul
TF_BUILTIN(MathImul, CodeStubAssembler) {
  Node* context = Parameter(Descriptor::kContext);
  Node* x = Parameter(Descriptor::kX); // 取出第一个参数 x
  Node* y = Parameter(Descriptor::kY); // 取出第二个参数 y
  Node* x_value = TruncateTaggedToWord32(context, x); // x 转换为 32 位整型 x_value
  Node* y_value = TruncateTaggedToWord32(context, y); // y 转换为 32 位整型 y_value
  Node* value = Int32Mul(x_value, y_value); // value = x_value * y_value
  Node* result = ChangeInt32ToTagged(value);
  Return(result);
}
```

从 Math.imul 源码来看，CodeStubAssembler 的语法接近汇编，同时也是合法的 C++ 代码。[Math.imul](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Math/imul) 的实现逻辑很简单：取出两个参数 x 和 y，分别转换成 32 位整型 x_value 和 y_value，相乘后返回结果。

参考 Math.imul 源码，我们自定义的函数 times10 代码如下：

```c++
TF_BUILTIN(MathTimes10, CodeStubAssembler) {
  Node* context = Parameter(Descriptor::kContext);
  Node* x = Parameter(Descriptor::kX); // 取出参数 x
  Node* x_value = TruncateTaggedToFloat64(context, x); // 转换为浮点数
  Node* y_value = Float64Constant(10.0);
  Node* value = Float64Mul(x_value, y_value); // 两个浮点数相乘
  Node* result = ChangeFloat64ToTagged(value);
  Return(result);
}
```

### 生成并存储 Code 对象

在 [src/builtins/builtins-definitions.h](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-definitions.h#34) 的宏 BUILTIN_LIST_BASE 下，新增一行：

```c++
#define BUILTIN_LIST_BASE(CPP, TFJ, TFC, TFS, TFH, ASM)      \
  /* GC write barrirer */                                    \
  // 前面源码太长，略；下一行为新增                                          
  TFJ(MathTimes10, 1, kReceiver, kX)                         \
  // 后面源码太长，略
```

### 取出上一步生成的 Code 对象，添加至 Math 对象的 times10 属性上

在 [src/init/bootstrapper.cc](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/init/bootstrapper.cc#2705) 文件中的 [Genesis::InitializeGlobal](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/init/bootstrapper.cc#1386) 方法，找到初始化 Javascript Math 对象的代码，参考邻近代码，增加一行：

```c++
SimpleInstallFunction(isolate_, math, "exp", Builtins::kMathExp, 1, true);
// 下面一行为新增
SimpleInstallFunction(isolate_, math, "times10", Builtins::kMathTimes10, 1, true);
```

### 编译运行 V8

一共修改了 3 处代码，编译 V8：

```c++
./tools/dev/gm.py x64.debug
```

然后运行 D8：

```c++
./out/x64.debug/d8
```

在 d8 控制台，输入 Math.times10(10)，结果如下图。可见我们在 V8 中为 Math 对象添加的 times10 方法已经生效了。

![运行结果](https://raw.githubusercontent.com/xudale/blog/master/assets/times10.png)

## 源码分析

### 实现 times10 方法的功能

首先回顾一下实现代码：

```c++
TF_BUILTIN(MathTimes10, CodeStubAssembler) {
  Node* context = Parameter(Descriptor::kContext);
  Node* x = Parameter(Descriptor::kX); // 取出参数 x
  Node* x_value = TruncateTaggedToFloat64(context, x); // 转换为浮点数
  Node* y_value = Float64Constant(10.0);
  Node* value = Float64Mul(x_value, y_value); // 两个浮点数相乘
  Node* result = ChangeFloat64ToTagged(value);
  Return(result);
}
```

[TF_BUILTIN](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-utils-gen.h#29) 是 C++ 定义的宏，源码如下：

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
g++ -E file_name.cpp > file_name.i
```

打开 file_name.i，将 C++ 代码格式化后，文件内容如下：

```c++
class MathTimes10Assembler : public CodeStubAssembler { 
  public: 
    using Descriptor = Builtin_MathTimes10_InterfaceDescriptor;
    explicit MathTimes10Assembler(compiler::CodeAssemblerState* state) : CodeStubAssembler(state) {} 
    void GenerateMathTimes10Impl(); 
    Node* Parameter(Descriptor::ParameterIndices index) { 
      return CodeAssembler::Parameter(static_cast<int>(index)); 
    } 
  }; 
void Builtins::Generate_MathTimes10(compiler::CodeAssemblerState* state) { 
  MathTimes10Assembler assembler(state); 
  state->SetInitialDebugInformation("MathTimes10", "macro.cpp", 25); 
  if (Builtins::KindOf(Builtins::kMathTimes10) == Builtins::TFJ) { 
    assembler.PerformStackCheck(assembler.GetJSContextParameter()); 
  } 
  assembler.GenerateMathTimes10Impl(); 
} 
void MathTimes10Assembler::GenerateMathTimes10Impl() {
  Node* context = Parameter(Descriptor::kContext);
  Node* x = Parameter(Descriptor::kX);
  Node* x_value = TruncateTaggedToFloat64(context, x);
  Node* y_value = Float64Constant(10.0);
  Node* value = Float64Mul(x_value, y_value);
  Node* result = ChangeFloat64ToTagged(value);
  Return(result);
}
```

可见 [TF_BUILTIN](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-utils-gen.h#29) 宏主要做了两件事情。

- 生成 MathTimes10Assembler 类，MathTimes10Assembler 类继承自 CodeStubAssembler 类。并为 MathTimes10Assembler 类添加一个新方法 GenerateMathTimes10Impl，GenerateMathTimes10Impl 方法体就是我们自定义 times10 函数的实现代码。我们刚才在实现 times10 的过程中，使用的 Parameter，TruncateTaggedToFloat64 等函数，都是继承自父类。
- 为 Builtins 类添加方法 Generate_MathTimes10，该方法最终调用了 times10 的实现代码 MathTimes10Assembler::GenerateMathTimes10Impl；

### 生成并存储 Code 对象

本部分源码只改动增加一行：

```c++
#define BUILTIN_LIST_BASE(CPP, TFJ, TFC, TFS, TFH, ASM)    \
  TFJ(MathTimes10, 1, kReceiver, kX)                       \ 
```

主要做了三件事：

- 生成索引 Builtins::kMathTimes10；
- 生成 Code 对象；
- 存储 Code 对象；

首先简要介绍下 V8 中的 Code 类
#### Code 类简介
Code 是 V8 中的一个类，用于描述可执行代码。每个 JavaScript 函数在 V8 中都有一个与之关联的 Code 对象，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/objects/js-objects.h#932)：

```c++
class JSFunction : public JSObject {
 public:
  // 源码在长，略
  // [code]: The generated code object for this function.  Executed
  // when the function is invoked, e.g. foo() or new foo(). See
  inline Code code() const; // 这个就是本节要介绍的 Code
  inline void set_code(Code code);
  // Get the abstract code associated with the function, which will either be
  // a Code object or a BytecodeArray.
  inline AbstractCode abstract_code();
}
```

JSFunction 对应 JavaScript 的函数，JSObject 对应 JavaScript 的对象。JSFunction 继承自 JSObject，并增加了 code 和 set_code 等与可执行代码相关的字段或方法，JSObject 并没有可执行代码相关的字段，这一点与我们使用 JavaScript 的体验是一致的。在 V8 源码中 JavaScript 函数与对象的关系，可参考[从 V8 源码理解 Javascript 函数是一等公民](https://zhuanlan.zhihu.com/p/101132637)。

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

从 Code 的声明来看，Code 是描述可执行代码的类。

#### 生成索引 Builtins::kMathTimes10

回顾下我们对源码的改动：

```c++
#define BUILTIN_LIST_BASE(CPP, TFJ, TFC, TFS, TFH, ASM)    \
  TFJ(MathTimes10, 1, kReceiver, kX)                       \ 
```

在 V8 源码中全局搜索 BUILTIN_LIST_BASE，发现宏 [BUILTIN_LIST](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-definitions.h#1321)，有调用 BUILTIN_LIST_BASE。

```c++
#define BUILTIN_LIST(CPP, TFJ, TFC, TFS, TFH, BCH, ASM)  \
  BUILTIN_LIST_BASE(CPP, TFJ, TFC, TFS, TFH, ASM)        \
  // 专注重点，后面略
```

源码中全局搜索 BUILTIN_LIST，可以搜到多个结果，其中 [Builtins](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins.h#47) 调用了宏 BUILTIN_LIST：

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

Builtins 使用宏嵌套声明了枚举，类型为整型，最终效果相当于为 Builtins 类增加了多个常量，其中就有由于我们对 V8 源码的改动，新生成的常量 Builtins::kMathTimes10，其类型为 Builtins::Name。

梳理本节代码调用链路：Builtins 的声明 -> 宏 BUILTIN_LIST -> 宏 BUILTIN_LIST_BASE。其中传递给宏 BUILTIN_LIST_BASE 的所有参数都是类 Builtins 声明代码中定义的宏 DEF_ENUM：

```c++
  #define DEF_ENUM(Name, ...) k##Name,
```

索引 Builtins::kMathTimes10 中的 k 本质上来自于宏 DEF_ENUM。

#### 生成 Code 对象

BUILTIN_LIST 的另一处调用 [SetupIsolateDelegate::SetupBuiltinsInternal](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/setup-builtins-internal.cc#286)：

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

这时宏 BUILTIN_LIST_BASE 的所有参数都是 宏 BUILD_TFJ，宏 BUILD_TFJ 的第一个作用是获取了 Code 对象，宏替换后代码如下：

```c++
code = BuildWithCodeStubAssemblerJS(                          
      isolate, index, &Builtins::Generate_MathTimes10, Argc, "MathTimes10"); 
```

其中 Builtins::Generate_MathTimes10 是上文提到的，在实现 times10 功能的时候，在宏 TF_BUILTIN 的作用下，我们为类 Builtins 添加的新方法，它生成了 Code 对象。

#### 存储 Code 对象

将 Code 对象存入 builtins 数组，源码还是位于宏 BUILD_TFJ 中：

```c++
  AddBuiltin(builtins, index++, code); 
```

顺着 AddBuiltin 的源码一直追下去，发现所有的 Code 对象都存在 [builtins_](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/isolate-data.h#162) 数组中。

```c++
Address builtins_[Builtins::builtin_count] = {};
```

梳理本节代码调用链路：SetupIsolateDelegate::SetupBuiltinsInternal -> 宏 BUILTIN_LIST -> 宏 BUILTIN_LIST_BASE，其中宏 BUILTIN_LIST_BASE 中的形参 TFJ，接收到的实际参数是宏 BUILD_TFJ。

宏定义嵌套宏定义，还时不时传个参数的写法，在 V8 源码中经常出现。这种写法既难读又难解释，下图为目前为止本文内容的缩略版：

![generateCode](https://raw.githubusercontent.com/xudale/blog/master/assets/generateCode.png)

### 为 Math 对象添加 times10 属性

这一步代码最简单，实际只有一行：

```c++
SimpleInstallFunction(isolate_, math, "times10", Builtins::kMathTimes10, 1, true);
```

math 就是 JavaScript 的 math 对象，Builtins::kMathTimes10 是上一节生成的索引，通过索引可以从 [builtins_](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/isolate-data.h#162) 数组中找到 times10 对应的 Code 对象。[SimpleInstallFunction](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/init/bootstrapper.cc#463) 的源码如下：

```c++
V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
    Isolate* isolate, Handle<JSObject> base, const char* name,
    Builtins::Name call, int len, bool adapt,
    PropertyAttributes attrs = DONT_ENUM) {
  // Although function name does not have to be internalized the property name
  // will be internalized during property addition anyway, so do it here now.
  Handle<String> internalized_name =
      isolate->factory()->InternalizeUtf8String(name);
  Handle<JSFunction> fun =
      SimpleCreateFunction(isolate, internalized_name, call, len, adapt); // 获取函数
  JSObject::AddProperty(isolate, base, internalized_name, fun, attrs); // 添加属性
  return fun;
}
```

SimpleCreateFunction 实际调用链路很长，它最终会从 [builtins_](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/isolate-data.h#162) 数组中找到对应的 Code 对象，生成一个新的 JSFunction 的实例，本节内容总结如下图：

![getCode](https://raw.githubusercontent.com/xudale/blog/master/assets/getCode.png)

## 一些感想

### V8 还是比较安全的

从内置函数相关的源码看下来，攻击 V8 定义的 Javascript 标准内置函数还是很难的。如果要攻击，首先要绕过词法分析、语法分析和 AST 树 3 座大山，V8 的内置函数都存在数组 [builtins_](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/isolate-data.h#162)中，但这个数组离 JavaScript 层面实在太远了，很难攻击。

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

上面的代码先是设置了为 array_function(对应 JavaScript Array) 添加了 proto(对应 Array 的 Prototype)，然后为 array_function 添加了 from、of 等静态方法，最后为 proto 添加了 concat、pop、push 等原型链方法。

从上面的代码，还可以看到 JavaScript 做为一门动态语言的所具有的特点。比如 JavaScript Array 原型上的 concat、pop、push 等方法，方法的名字做为一个字符串，客观的存在于 V8 中。如果运行时想要使用 pop 方法，只要 JavaScript Array 原型上有一个名为 pop 的方法就可以，pop 方法可以由 V8 提供，也可以由第 3 方提供。总之，只要该方法挂载在 JavaScript Array 原型上，并且名称为 pop 就可以。这种特性为 JavaScript 运行时的动态加载提供了基础。

而静态类型语言，如 C 语言。C 语言的变量和函数在编译后都直接对应地址，变量名和函数名在编译后都不复存在，这里可以参考[从 V8 源码理解 Javascript 函数是一等公民](https://zhuanlan.zhihu.com/p/101132637)。通常，静态语言都没有类似 JavaScript 动态加载的特性。

### 关于定制化 V8

V8 最初只应用于 Chrome 浏览器，有很多兼容性的包袱。如果在服务端定制 V8，Bootstrap 里面的很多代码都可以删除，比如下图中有大拇指标记的部分，这部分 API 由于浏览器兼容性的原因，虽然日常开发中早已不再使用，但 V8 源码中继续存在。如果有一天需要在服务端定制 V8，根据本文介绍的内容，顺藤摸瓜，可以从 V8 源码中彻底删除函数定义、生成索引和获取 Code 对象等代码，以便减少运行时 V8 实例的体积。

![delete](https://raw.githubusercontent.com/xudale/blog/master/assets/delete.png)

## 参考文献

[CodeStubAssembler builtins](https://v8.dev/docs/csa-builtins)

[Taming architecture complexity in V8 — the CodeStubAssembler](https://v8.dev/blog/csa)













