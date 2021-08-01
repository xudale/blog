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

Chromium 源码版本 90.0。

querySelectorAll 源码位于 [blink](https://chromium.googlesource.com/chromium/src/+/refs/tags/90.0.4430.8/third_party/blink/)。[querySelectorAll](https://chromium.googlesource.com/chromium/src/+/refs/tags/90.0.4430.8/third_party/blink/renderer/core/dom/parent_node.h#91) 方法源码如下：



```C++
static StaticElementList* querySelectorAll(ContainerNode& node,
                                          const AtomicString& selectors,
                                          ExceptionState& exception_state) {
  return node.QuerySelectorAll(selectors, exception_state);
}
```

第一个参数 node 表示当前 dom 节点，第二个参数 selectors 是一个选择器对象，基本是用一个 C++ 对象把选择器字符串，如 ".xxx > div" 包了一层。从 blink 源码来看，querySelectorAll 返回的不是 JavaScript 数组，而是 StaticElementList，[源码如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/90.0.4430.8/third_party/blink/renderer/core/dom/static_node_list.h#40)：


```C++
using StaticElementList = StaticNodeTypeList<Element>;

template <typename NodeType>
class StaticNodeTypeList final : public NodeList {
 public:
  static StaticNodeTypeList* Adopt(HeapVector<Member<NodeType>>& nodes);

  ~StaticNodeTypeList() override;

  unsigned length() const override;

 private:
  HeapVector<Member<NodeType>> nodes_;
};
```

如果在浏览器中执行下面 JavaScript 代码：

```JavaScript
const divList = document.querySelectorAll('div')
divList.length
```

JavaScript 代码 divList.length，底层调用的是 StaticNodeTypeList<NodeType>::length，[源码如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/90.0.4430.8/third_party/blink/renderer/core/dom/static_node_list.h#69)：

```C++
template <typename NodeType>
unsigned StaticNodeTypeList<NodeType>::length() const {
  return nodes_.size();
}
```

JavaScript 数组 



## 参考文献

[ecma262:sec-array.prototype.find](https://tc39.es/ecma262/#sec-array.prototype.find)

[mdn:Array.prototype.find](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/find)


















