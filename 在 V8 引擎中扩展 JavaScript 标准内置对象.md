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
    0x100000f31 <+1>:  movq   %rsp, %rbp
    0x100000f34 <+4>:  movl   %edi, -0x4(%rbp)
    0x100000f37 <+7>:  imull  $0xa, -0x4(%rbp), %eax
    0x100000f3b <+11>: popq   %rbp
    0x100000f3c <+12>: retq 
```

C 语言是编译型语言，生成这样的汇编也没什么奇怪的，笔者想要表达的意思是 C 语言的函数在编译后最终会生成一个汇编语言的函数，或都说 C 语言的函数最终会编译为机器码，C 语言的函数在底层对应的是在虚拟内存中一片连续的机器码。

### JavaScript 语言函数的底层表示