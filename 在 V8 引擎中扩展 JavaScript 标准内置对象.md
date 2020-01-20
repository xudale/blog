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

### C 语言函数的底层表示






