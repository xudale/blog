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

### 为 Math 对象添加 times10 属性

这一步代码最简单，实际只有一行：

```c++
SimpleInstallFunction(isolate_, math, "times10", Builtins::kMathTimes10, 1, true);
```

参数 math 实际上就是 JavaScript 的 Math 对象，参数 "times10" 是我们要添加的方法的名字，Builtins::kMathTimes10 是方法的索引，至此，大功告成。





