# Array.prototype.findIndex 源码分析

Array.prototype.findIndex 与 Array.prototype.find 的源码几乎是相同的，如下图。

![findfindIndex](https://raw.githubusercontent.com/xudale/blog/master/assets/findfindIndex.png)

区别主要在于两行代码，已用蓝框标出：

|                                    | Array.prototype.find | Array.prototype.findIndex |
| -----------------------------------| -------------------- | -------------------- |
| 找到能使 callback 返回 true 的元素   | 返回元素 | 返回索引 k |
| 找不到能使 callback 返回 true 的元素 | 返回 undefined | 返回 -1 |

其它内容参考 [Array.prototype.find 源码分析](https://zhuanlan.zhihu.com/p/380442932)。
















