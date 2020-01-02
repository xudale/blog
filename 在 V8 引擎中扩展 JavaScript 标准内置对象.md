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

![运行结果](https://xudale.github.io/blog/assets/ctimes10.png)

### JavaScript 语言函数的底层表示