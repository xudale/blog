# Chromium 线程与消息循环

Chromium 源码版本 91.0.4437.3，大约发布于 2021 年春节。

## Chromium 线程

Chromium 与线程相关的类有两个，[Thread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/thread.h#62) 类和 [PlatformThread](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/base/threading/platform_thread.h#121)。Thread 类是 Chromium 对线程的封装，PlatformThread 类封装了操作系统的线程相关函数，不同的操作系统有不同的实现。Thread 类基于 PlatformThread 类。我们从创建线程的源码开始。

### Thread

### PlatformThread



这个问题很简单，其实不需要看源码，querySelectorAll 和 Array 属于不同的规范。

querySelectorAll 是 [Dom](https://dom.spec.whatwg.org/) 规范中的内容，Dom 规范是平台中立的，笔者在 Dom 规范中全局搜索 V8，没有结果。全局搜索 JavaScript，有 4 个结果。以下摘自规范：

> DOM defines a platform-neutral model for events, aborting activities, and node trees.


[Dom](https://dom.spec.whatwg.org/#parentnode) 规范中与 querySelectorAll 相关的内容如下：

![querySelectorIDL](https://raw.githubusercontent.com/xudale/blog/master/assets/querySelectorIDL.png)
![querySelectorDesc](https://raw.githubusercontent.com/xudale/blog/master/assets/querySelectorDesc.png)

从上面截图中可见，querySelectorAll 返回的结果是 NodeList。


Array 的定义来自 [ECMAScript specification](https://tc39.es/ecma262/#sec-array-constructor)。

![arrayecma](https://raw.githubusercontent.com/xudale/blog/master/assets/arrayecma.png)


既然 querySelectorAll 和 Array 不属于同一个规范，Dom 规范要平台中立，JavaScript 要运行在服务端，甚至嵌入式系统。querySelectorAll 返回的结果不是 Array 类型， 也算正常。


## chromium 消息循环

Chromium 源码版本 91.0.4437.3。因为 querySelectorAll 的返回结果和 JavaScript 数组都有 length 属性，本小节比较下两者的代码。

querySelectorAll 源码位于 [blink](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/)。[querySelectorAll](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/dom/parent_node.h#91) 方法源码如下：



```C++
static StaticElementList* querySelectorAll(ContainerNode& node,
                                          const AtomicString& selectors,
                                          ExceptionState& exception_state) {
  return node.QuerySelectorAll(selectors, exception_state);
}
```

第一个参数 node 表示当前 dom 节点，第二个参数 selectors 是一个选择器对象，基本是用一个 C++ 对象把选择器字符串，如 ".classname > div" 包了一层。从 blink 源码来看，querySelectorAll 的返回结果不是 JavaScript 数组，而是 StaticElementList，StaticElementList 的定义[源码如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/dom/static_node_list.h#40)：


```C++
using StaticElementList = StaticNodeTypeList<Element>;

template <typename NodeType>
class StaticNodeTypeList final : public NodeList {
 public:
  static StaticNodeTypeList* Adopt(HeapVector<Member<NodeType>>& nodes);

  ~StaticNodeTypeList() override;

  unsigned length() const override;

 private:
  // nodes_ 保存所有选中结点
  HeapVector<Member<NodeType>> nodes_;
};
```

如果在浏览器中执行下面 JavaScript 代码：

```JavaScript
const divList = document.querySelectorAll('div')
divList.length
```

JavaScript 代码 divList.length，底层调用的是 StaticNodeTypeList<NodeType>::length，[源码如下](https://chromium.googlesource.com/chromium/src/+/refs/tags/91.0.4437.3/third_party/blink/renderer/core/dom/static_node_list.h#69)：

```C++
template <typename NodeType>
unsigned StaticNodeTypeList<NodeType>::length() const {
  return nodes_.size();
}
```

JavaScript 数组源码位于 V8，[声明如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.1-lkgr/src/objects/js-array.h#24)：

```C++
class JSArray : public JSObject {
 public:
  // [length]: The length property.
  DECL_ACCESSORS(length, Object)

  // Overload the length setter to skip write barrier when the length
  // is set to a smi. This matches the set function on FixedArray.
  inline void set_length(Smi length);

  static bool MayHaveReadOnlyLength(Map js_array_map);
  static bool HasReadOnlyLength(Handle<JSArray> array);
}

// length 表示方法名称
// kLengthOffset 表示 length 属性在数组对象上的偏移量
ACCESSORS(JSArray, length, Object, kLengthOffset)
```

ACCESSORS 为数组定义的 length 方法，顺着宏定义一路往下追，最终[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.1-lkgr/src/objects/object-macros.h#179)：

```C++
#define ACCESSORS_CHECKED2(holder, name, type, offset, get_condition, \
                           set_condition)                             \
  DEF_GETTER(holder, name, type) {                                    \
    type value = TaggedField<type, offset>::load(isolate, *this);     \
    DCHECK(get_condition);                                            \
    return value;                                                     \
  }                                                                   

#define ACCESSORS_CHECKED(holder, name, type, offset, condition) \
  ACCESSORS_CHECKED2(holder, name, type, offset, condition, condition)

#define ACCESSORS(holder, name, type, offset) \
  ACCESSORS_CHECKED(holder, name, type, offset, true)
```

可见获取数组 length 属性的逻辑是读取 JSArray 对象上，偏移量为 kLengthOffset 的内容，与 StaticNodeTypeList 的 length 逻辑完全不同。

> querySelectorAll 的源码位于 blink，JavaScript Array 的源码位于 V8

> blink 是 Chromium 渲染引擎，V8 是 JavaScript 引擎，二者是不同的项目




















