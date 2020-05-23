# typeof 与 Javascript 类型源码分析.md
本文先分析 microtask 队列，后分析 async/await，版本为 V8 7.7.1。
## typeof 源码分析


### 基础功能


```c++
Address* ring_buffer_ = nullptr;
```





## 为什么 1 + 1 = 2，1 + '1' = '11'？

![microtaskflow](https://raw.githubusercontent.com/xudale/blog/master/assets/microtaskflow.png)












