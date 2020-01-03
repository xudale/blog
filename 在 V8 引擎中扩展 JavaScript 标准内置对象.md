# 在 V8 引擎中扩展 JavaScript 标准内置对象
## 摘要
JavaScript 标准内置对象，如 Math、Array 和 Promise 等是每一个前端程序员日常工作的基础，这些内置对象在 JavaScript 层面上是函数或者对象。首先，本文会从底层的角度对比一下 C 语言的函数和 JavaScript 语言函数的区别；然后，举例说明如何在 V8 中为 JavaScript 的 Math 对象添加一个名为 times10 的方法，并分析相关源码。
## 函数的底层表示
### C 语言函数的底层表示
如以下代码，可复制粘贴然后在 [https://tool.lu/coderunner/](https://tool.lu/coderunner/) 运行。

```c
    #include <stdio.h>

    int times10 (int small) {
        return small * 10;
    }

    int main(int argc, const char * argv[]) {
        int small = 1;
        int large = times10(small);
        printf("result is %d\n", large);
        return 0;
    }
```

代码逻辑非常简单，定义了一个名为 times10 的函数，函数的功能是将入参乘 10 后返回。main 函数中调用 times10 函数，函数运行结果在 Xcode 中如下：

![运行结果](https://raw.githubusercontent.com/xudale/blog/master/assets/ctimes10.png)

times10 函数对应的 X64 汇编如下：

```assemble
    0x100000f30 <+0>:  pushq  %rbp
    0x100000f31 <+1>:  movq   %rsp, %rbp  ; 前两行是套路，保存栈寄存器
    0x100000f34 <+4>:  movl   %edi, -0x4(%rbp) ; edi 寄存器存的是入参
    0x100000f37 <+7>:  imull  $0xa, -0x4(%rbp), %eax ; 乘 10 后的结果存入 eax 
    0x100000f3b <+11>: popq   %rbp ; 最后两行也是套路，恢复栈寄存器
    0x100000f3c <+12>: retq 
```

C 语言是编译型语言，C 语言的函数体在编译后会生成一个汇编语言的函数，或者说 C 语言的函数体最终会编译为机器码，C 语言的函数体在底层对应的是在虚拟内存中一片连续的机器码。

main 函数对应的 X64 汇编如下：

```assemble
    0x100000f40 <+0>:  pushq  %rbp
    0x100000f41 <+1>:  movq   %rsp, %rbp
    0x100000f44 <+4>:  subq   $0x20, %rsp
    0x100000f48 <+8>:  movl   $0x0, -0x4(%rbp)
    0x100000f4f <+15>: movl   %edi, -0x8(%rbp)
    0x100000f52 <+18>: movq   %rsi, -0x10(%rbp)
    0x100000f56 <+22>: movl   $0x1, -0x14(%rbp)
    0x100000f5d <+29>: movl   -0x14(%rbp), %edi
    0x100000f60 <+32>: callq  0x100000f30         ; 调用 times10
    0x100000f65 <+37>: leaq   0x3a(%rip), %rdi          
    0x100000f6c <+44>: movl   %eax, -0x18(%rbp)
    0x100000f6f <+47>: movl   -0x18(%rbp), %esi
    0x100000f72 <+50>: movb   $0x0, %al
    0x100000f74 <+52>: callq  0x100000f86         ; 调用 printf     
    0x100000f79 <+57>: xorl   %esi, %esi
    0x100000f7b <+59>: movl   %eax, -0x1c(%rbp)
    0x100000f7e <+62>: movl   %esi, %eax
    0x100000f80 <+64>: addq   $0x20, %rsp
    0x100000f84 <+68>: popq   %rbp
    0x100000f85 <+69>: retq
```
    
在地址为 0x100000f60 的这行汇编里

```assemble
    0x100000f60 <+32>: callq  0x100000f30         ; 调用 times10
```

main 函数调用 times10 函数，从生成的汇编来看，编译完成后 times10 这个函数名对应的是地址，也就是说 C 语言的函数名对应的是地址。
看到这里，可以看出 C 语言函数和 JavaScript 函数的一个区别，由于 C 语言函数体对应机器码，函数名称对应地址，所以 C 语言不支持为函数添加属性。
### JavaScript 语言函数的底层表示

在 V8 中，JavaScript 的函数在底层对应的是一个 C++ 对象，[声明代码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/objects/js-objects.h#932)：

```c++
    // JSFunction describes JavaScript functions.
    class JSFunction : public JSObject {
        public:
        // [prototype_or_initial_map]:
        DECL_ACCESSORS(prototype_or_initial_map, Object)
        // [shared]: The information about the function that
        // can be shared by instances.
        DECL_ACCESSORS(shared, SharedFunctionInfo)
        static const int kLengthDescriptorIndex = 0;
        static const int kNameDescriptorIndex = 1;
        // Home object descriptor index when function has a [[HomeObject]] slot.
        static const int kMaybeHomeObjectDescriptorIndex = 2;
        // [context]: The context for this function.
        inline Context context();
        inline bool has_context() const;
        inline void set_context(Object context);
        inline JSGlobalProxy global_proxy();
        inline NativeContext native_context();
        inline int length();
        static Handle<Object> GetName(Isolate* isolate, Handle<JSFunction> function);
        static Handle<NativeContext> GetFunctionRealm(Handle<JSFunction> function);
        // [code]: The generated code object for this function.  Executed
        // when the function is invoked, e.g. foo() or new foo(). See
        // [[Call]] and [[Construct]] description in ECMA-262, section
        // 8.6.2, page 27.
        inline Code code() const;
        inline void set_code(Code code);
        inline void set_code_no_write_barrier(Code code);
        // 源码太长，复制粘贴到此结束
    }
```

从代码的第一行注释

```c++
    // JSFunction describes JavaScript functions.
```

可知，JavaScript 的函数在 V8 里是一个 JSFunction 的实例，JSFunction 源码很长，这里举两个例子来佐证 JavaScript 函数是 V8 中的一个 C++ 对象。

JavaScript 的函数有一个名为 [toString](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/toString) 的方法，可以输出一个函数的字符串表示。比如任意自定义函数一个函数，然后调用这个函数的 toString 方法，如下：

```JavaScript
    a = _ => console.log(_)
    a.toString() // 输出 "_ => console.log(_)"
```

可以获得函数的实现代码，但当对内置对象的方法调用 toSring 时，比如：

```JavaScript
    Math.max.toString() // 输出 "function max() { [native code] }"
```

并没有输出函数的实现代码，而且输出的字符串 native code 是从哪里来的？这个问题困扰了笔者 3 年，下面，我们一起看下 JavaScript 函数的 toString 方法在 V8 中的实现。










