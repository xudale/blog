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





### JavaScript 语言函数的底层表示