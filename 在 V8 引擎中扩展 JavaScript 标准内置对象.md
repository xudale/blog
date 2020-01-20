# 在 V8 引擎中扩展 JavaScript 标准内置对象
## 摘要
本文会以在 Math 对象上添加一个新方法为例，介绍如何在 V8 引擎中扩展 JavaScript 对象，并分析相关源码。
## 为 Math 对象添加 times10 方法
在 V8 中为 Math 对象添加 times10 方法，times10 方法的作用是将入参乘 10 后返回。可分为 3 步：

- 实现 times10 方法的功能；
- 生成 Code 对象；
- 为 Math 对象添加 times10 属性；

### 定义

### C 语言函数的底层表示


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



