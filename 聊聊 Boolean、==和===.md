# 聊聊 Boolean、==和===
业务开发，踩坑多次，本文将从 V8 源码分析 Boolean、==和===。
## Boolean
在 JavaScript 中，Boolean 函数有两种调用方式，一种是函数式调用：

```JavaScript
Boolean('test') // true
```
一种是构造函数式调用:

```JavaScript
new Boolean('test') // Boolean {true}
```
无论是哪种调用方式，在 V8 中都是由同一个函数处理，Boolean [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/boolean.tq#22)：

```c++
BooleanConstructor(context: Context, receiver: Object, ...arguments): Object {
  const value = SelectBooleanConstant(ToBoolean(arguments[0])); // 将参数转为 true 或 false
  const newTarget = Parameter(NEW_TARGET_INDEX); // 如果是构造函数式调用 new Boolean()，newTarget 一定有值
  if (newTarget == Undefined) {
    // 如果是函数式调用，此处返回
    return value;
  }
  // 如果是构造函数式调用，执行下面逻辑，建议不看
  const target = UnsafeCast<JSFunction>(Parameter(TARGET_INDEX));
  const map = GetDerivedMap(target, UnsafeCast<JSReceiver>(newTarget));
  let properties = kEmptyFixedArray;
  if (IsDictionaryMap(map)) {
    properties = AllocateNameDictionary(kNameDictionaryInitialCapacity);
  }
  const obj = UnsafeCast<JSValue>(AllocateJSObjectFromMap(
      map, properties, kEmptyFixedArray, kNone, kWithSlackTracking));
  obj.value = value;
  return obj;
}
```

BooleanConstructor 函数的逻辑很简单，通过 ToBoolean(arguments[0]) 将参数转为 true 或 false，如果是函数式调用，立刻返回，这也是日常开发中常见的情况。




## ==
## ===




[V8 源码](https://cs.chromium.org/chromium/src/v8/?g=0)可在浏览器中查看，这个网站在代码浏览与检索方面的功能十分强大，可以快速的查看 C++ 变量的定义和引用。缺点是不能查看 V8 某个版本或某个 git tag 的源码，但依然强烈推荐。如果想要查看 V8 某个 tag 的源码，可以访问 [v8.git](https://chromium.googlesource.com/v8/v8.git) 。如果想要在本地编译 V8 源码，在参考 [V8 官方文档](https://v8.dev/docs/build-gn)的基础上，还要注意墙的问题，浏览器能访问 google 不代表终端也能访问 google，终端也需要设置代理。








