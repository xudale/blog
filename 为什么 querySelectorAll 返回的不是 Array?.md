# 为什么 querySelectorAll 返回的不是 Array?

## 非源码角度

这个问题很简单，其实不需要看源码，querySelectorAll 和 Array 属于不同的标准。

querySelectorAll 是 [Dom](https://dom.spec.whatwg.org/) 规范中的内容，Dom 规范是平台中立的，笔者在 Dom 规范中全局搜索 V8，没有结果。全局搜索 JavaScript，有 4 个结果。以下摘自规范：

> DOM defines a platform-neutral model for events, aborting activities, and node trees.


[Dom](https://dom.spec.whatwg.org/#parentnode) 规范和 querySelectorAll 相关的内容如下：

![querySelectorIDL](https://raw.githubusercontent.com/xudale/blog/master/assets/querySelectorIDL.png)
![querySelectorDesc](https://raw.githubusercontent.com/xudale/blog/master/assets/querySelectorDesc.png)


Array 的定义来自 [ECMA](https://tc39.es/ecma262/#sec-array-constructor)

![arrayecma](https://raw.githubusercontent.com/xudale/blog/master/assets/arrayecma.png)


既然 querySelectorAll 和 Array 不属于同一个规范，dom 规范还要平台中立，JavaScript 还要运行在服务端，甚至嵌入式。querySelectorAll 没有返回 Array 也算可以理解。


## 源码角度

如果不考虑任何边界条件，尽可能模仿 V8 FastArrayfind 的实现逻辑，find 方法可用 Javascript 实现如下：

```Javascript
function find(callback, thisArg) {
  const array = this
  const len = array.length
  for (i = 0; i < len; i++) {
    if (callback.call(thisArg, array[i], i, array)) {
      return array[i]
    }
  }
}
```

## 参考文献

[ecma262:sec-array.prototype.find](https://tc39.es/ecma262/#sec-array.prototype.find)

[mdn:Array.prototype.find](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/find)


















