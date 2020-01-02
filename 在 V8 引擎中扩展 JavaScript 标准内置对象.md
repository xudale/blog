# 在 V8 引擎中扩展 JavaScript 标准内置对象
## 摘要
JavaScript 标准内置对象，如 Math、Array 和 Promise 等是每一个前端程序员日常工作的基础，这些内置对象在 JavaScript 层面上是函数或者对象。首先，本文会从底层的角度对比一下 C 语言的函数和 JavaScript 的函数的区别；然后，举例说明如何在 V8 中为 JavaScript 的 Math 对象添加一个名为 times10 的方法，并分析相关源码。
## 函数的底层表示
### C 语言函数的底层表示
### JavaScript 语言函数的底层表示